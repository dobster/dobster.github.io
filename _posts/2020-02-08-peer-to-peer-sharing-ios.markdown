---
layout: post
title:  "Adopting Network.framework for iOS peer-to-peer connectivity"
date:   2020-02-08 18:24:35 +1100
categories: iOS iPadOS iPad Network.framework
---

![Image of the flight deck at 40,000 ft][flying-over-bondi]
> Not much internet connectivity up here!

<br/>

Apple introduced the [Multipeer Connectivity][multipeer-connectivity] framework back in 2013. This unheralded framework allows discovery of other nearby devices and exchange of data using peer-to-peer connectivity. For iOS, this would be via Wi-Fi networks, peer-to-peer Wi-Fi and Bluetooth. 

At Qantas, we have been using this framework for sharing of Navigation Log entries between the pilot iPads during flight. Without a reliable satellite connection, client-to-server syncing to exchange data between devices is not possible.

At WWDC 2019 Apple [presented a new way to do peer-to-peer sharing][wwdc-2019-advanced-networking] using the relatively fresh [Network][network-framework] framework. This blog post will go through how we implemented this.

<br/>

So why a new framework?
-----

The previous Multipeer Connectivity framework was not without its problems. As the app could be backgrounded or the device move temporarily out of range, re-discovering lost connections was a source of pain. We wanted to give pilots a _seamless experience_—they should not have to monitor connections and manually re-establish them. Unfortunately, the Multipeer Connectivity framework abstracted a little too much away, resulting in a lack of visibility over what was going on.

The new Network framework is being promoted by Apple as the recommended way of implementing your own networking—if URLSession is not enough for your needs. As URLSession has been updated to sit on top of Network framework, this makes sense. Perhaps one of the key motivators for this is a better security foundation, in addition to more visiblity to what's going on under the hood.

In contrast to the Multipeer Connectivity framework, the Network framework will limit you to communicating via Wi-Fi networks and peer-to-peer Wi-Fi, **but not Bluetooth**. For modern iPads this should not be a problem.

<br/>

Building Blocks
------

The Network framework provides us with two discovery classes:
- An [NWListener][nwlistener] object that listens for incoming network connections. By establishing a listener you also advertise yourself via Bonjour.
- An [NWBrowser][nwbrowser] object that lets you browse for advertised Bonjour services you may wish to connect to.

And finally
- An [NWConnection][nwconnection] object which is the connection you establish with the other device.

Let's go through how to set this up. We'll start by building wrappers around the NWListener and NWBrowser objects, as per the sample code from WWDC2019. The wrappers will let us manage the NWListener and NWBrowser objects which may need to be recreated under some situations.

We'll configure our listener with a bonjour service name (essentially our private peer-to-peer Wi-Fi network) and a pre-shared key for security that we'll cover later. 

We'll also need a UUID to uniquely identify each other when there are multiple peers around to connect to. This UUID will also be handy when establishing who initiates a connection.

{% highlight swift %}
class PeerListener {
    let bonjourService: String
    let presharedKey: String
    let myPeerID: String

    private var listener: NWListener?
    
    func startListening() {
        guard let listener = try? NWListener(using: NWParameters(passcode: presharedKey)) else {
            print("[listener] failed to create NWListener")
            return
        }
        self.listener = listener
        
        listener.service = NWListener.Service(name: myPeerID, type: bonjourService, domain: nil, txtRecord: nil)
         
        listener.newConnectionHandler = { [weak self] newConnection in
            guard let self = self else { return }
            self.delegate?.peerListener(self, didFindNewConnection: newConnection)
        }

        listener.stateUpdateHandler = { [weak self] newState in
            guard let self = self else { return }
            
            switch newState {
            case .failed:
                self.listener?.cancel()
                self.startListening()

            // ...
            }
        }
        
        listener.start(queue: .main)
    }

    func stopListening() {
        listener?.cancel()
        listener = nil
    }
}
{% endhighlight %}

Next lets create the browser to find advertised Bonjour services and initiate new connections. 

We'll limit ourselves to peer-to-peer Wi-Fi connections, for simplicity and security.

{% highlight swift %}
class PeerBrowser {
    let bonjourService: String

    private var browser: NWBrowser?
    
    func startBrowsing() {
        let params = NWParameters()
        params.includePeerToPeer = true
        params.requiredInterfaceType = .wifi
        let browser = NWBrowser(for: .bonjour(type: bonjourService, domain: nil), using: params)
        self.browser = browser
        
        browser.browseResultsChangedHandler = { [weak self] results, changes in
            guard let self = self else { return }
            self.delegate?.peerBrowser(self, didUpdateResults: Array(results))
        }

        browser.stateUpdateHandler = { [weak self] newState in
            guard let self = self else { return }
            
            switch newState {
            case .failed:
                self.browser?.cancel()
                self.startBrowsing()

            //  ...          
            }
        }
      
        browser.start(queue: .main)
    }

    func stopBrowsing() {
        browser?.cancel()
        browser = nil
    }
}
{% endhighlight %}

Finally let's create a wrapper class around the NWConnection object.

{% highlight swift %}
class PeerConnection {
    
    var connection: NWConnection?
    
    func startConnection() {
        guard let connection = connection else { return }
        
        connection.stateUpdateHandler = { [weak self] newState in
            guard let self = self else { return }
            self.delegate?.peerConnection(self, didChangeState: newState)
            
            switch newState {
            case .ready:
                self.receiveNextMessage()
                
            case .failed:
                connection.cancel()
                
            case .waiting:
                // for seamless connectivity, we don't really want to wait for anything
                connection.cancel()

            // ...
            }
        }
        
        connection.start(queue: .main)
    }

    func cancel() {
        connection?.cancel()
        connection = nil
    }

    func sendMessage(type: UInt32, content: Data) {
        // TODO
    }
    
    func receiveNextMessage() {
        // TODO
    }
}
{% endhighlight %}

Using these wrapper classes enable a fairly straight forward session management class which will manage a list of "active peers" (as distinct from connections which may or may not be ready and communicating). This level of abstraction makes reasoning a lot easier for the rest of your app which shouldn't need to worry about the current connection state.

We'll also add callback closures at this level, which will help with unit testing later.

{% highlight swift %}
class MultipeerSession: PeerBrowserDelegate, PeerListenerDelegate {
    
    /// Callback when new peers are found or known peers are lost.
    var peersChangeHandler: (_ peers: [PeerInfo]) -> Void = { _ in }
    
    /// Callback when a message is received from a peer.
    var messageReceivedHandler: (_ peer: PeerInfo, _ data: Data) -> Void = { _, _, _ in }
        
    private var browser: PeerBrowser?
    private var listener: PeerListener?
    
    private var browserResults: [NWBrowser.Result] = []
    private var connections: [PeerConnection] = []
    private var activePeers: [PeerInfo] = []

    func startSharing() {
        self.browser = PeerBrowser(...)
        self.listener = PeerListener(...)
        
        self.listener?.startListening()
        self.browser?.startBrowsing()
    }
    
    func stopSharing() {
        connections.forEach { $0.cancel() }
        connections.removeAll()
        
        browser?.stopBrowsing()
        listener?.stopListening()
    }

    // MARK: - PeerBrowserDelegate
      
    func peerBrowser(_ browser: PeerBrowser, didUpdateResults results: [NWBrowser.Result]) {
        browserResults = results
        connections.removeAll(where: { $0.connection?.state == .cancelled || $0.connection == nil })
        updatePeerList()
    }
    
    // MARK: - PeerListenerDelegate
    
    func peerListener(_ listener: PeerListener, didFindNewConnection connection: NWConnection) {
        guard !connections.contains(where: { $0.connection?.endpoint == connection.endpoint }) else {
            connection.cancel()
            return
        }
        let newConnection = PeerConnection(connection: connection, delegate: self)
        connections.append(newConnection)
        updatePeerList()
    }

     // MARK: - Update Active Peers List
    
    private func updatePeerList() {
        let activePeers = connections.filter {
            $0.connection?.state == .ready
        }
        
        if activePeers != self.activePeers {
            self.activePeers = activePeers
            queue.async {
                self.peersChangeHandler(activePeers)
            }
        }
    }
}
{% endhighlight %}

Previously we glossed over the initialisation of the connection and listener objects, which take an NWParameter object. 

We'll want secure connections so we'll use TLS 1.2 over TCP using a pre-shared key. You'll have to decide how each app will know the pre-shared key. In the WWDC 2019 example, a four digit code is shared between players of the game. This isn't practical for seamless connectivity, so get that key onto the device in another secure way. Just don't send it over the wire!

The implementation here is pretty much a copy of the WWDC 2019 sample code. Yay for CryptoKit!

{% highlight swift %}

import Network
import CryptoKit

extension NWParameters {

    // Create parameters for use in PeerConnection and PeerListener.
    convenience init(identity: String, secret: String) {
        // Customize TCP options to enable keepalives.
        let tcpOptions = NWProtocolTCP.Options()
        tcpOptions.enableKeepalive = true
        tcpOptions.keepaliveIdle = 2
        
        let tlsOptions = NWProtocolTLS.Options()
        
        let authenticationKey = SymmetricKey(data: secret.data(using: .utf8)!)
        var authenticationCode = HMAC<SHA256>.authenticationCode(for: identity.data(using: .utf8)!, using: authenticationKey)
        let authenticationDispatchData = withUnsafeBytes(of: &authenticationCode) { (ptr: UnsafeRawBufferPointer) in
            DispatchData(bytes: ptr)
        }
        let psk = authenticationDispatchData as __DispatchData
        
        var identityData = identity.data(using: .unicode)!
        let identityDispatchData = withUnsafeBytes(of: &identityData) { (ptr: UnsafeRawBufferPointer) in
            DispatchData(bytes: ptr)
        }
        let psk_identity = identityDispatchData as __DispatchData
        
        sec_protocol_options_add_pre_shared_key(tlsOptions.securityProtocolOptions, psk, psk_identity)

        let ciphersuite = tls_ciphersuite_t(rawValue: TLS_PSK_WITH_AES_128_GCM_SHA256)!
        sec_protocol_options_append_tls_ciphersuite(tlsOptions.securityProtocolOptions, ciphersuite)

        // Create parameters with custom TLS and TCP options.
        self.init(tls: tlsOptions, tcp: tcpOptions)

        // Enable using a peer-to-peer link.
        self.includePeerToPeer = true

        let options = NWProtocolFramer.Options(definition: TLVMessageProtocol.definition)
        self.defaultProtocolStack.applicationProtocols.insert(options, at: 0)
    }
}
{% endhighlight %}

<br/>

Who connects to whom?
---

The browser will discover which of the other devices are advertising. 

image here...

However, if every device initiates a connection to the others, you'll end up with double the required connections!

This is where the unique peer ID helps. Who-ever has the highest UUID can take the initative to establish the connection.

{% highlight swift %}
 /// Should connect to a discovered service?
    private func shouldConnectTo(_ result: NWBrowser.Result) -> Bool {
        guard !connections.contains(where: { $0.connection?.endpoint == result.endpoint }) else {
            // already have an active connection
            return false
        }
        
        switch result.endpoint {
        case .service(let name, let type, _, _):
            // Only initiate a connection if our peerID > their peerID
            return type == config.bonjourService && name < config.myPeerInfo.peerID
        default:
            return false
        }
    }
{% endhighlight %}

In a scenario with two devices, you'll end up with a host and a service connection.

insert image

With three devices, you'll end up with one device hosting twice, one device connecting twice, and one doing both!

insert image

<br/>

Exchanging data
---

The Network framework is suprisingly primitive in dealing with transfer of data. The WWDC 2019 Advanced Network 2 presentation walks through creating an elaborate transfer protocol specific to the demoed "TicTacToe" application. We can simplify things by implementing a basic Type-Length-Message protocol as described, but letting the users of our MultipeerSession object to exchange Codable structs on their own terms. Much saner and simpler!

{% highlight swift %}

class PeerConnection {

  // ...

  func sendMessage(type: UInt32, content: Data) {
    guard connection?.state == .ready else { return }

    let framerMessage = NWProtocolFramer.Message(messageType: type)
    let context = NWConnection.ContentContext(identifier: "Message", metadata: [framerMessage])

    connection?.send(content: content, contentContext: context, isComplete: true, completion: .idempotent)
  }

  func receiveNextMessage() {
    connection?.receiveMessage { (data, context, _, error) in
      if let message = context?.protocolMetadata(definition: TLVMessageProtocol.definition) as? NWProtocolFramer.Message {
          self.delegate?.peerConnection(self, didReceiveMessageType: message.messageType, data: data ?? Data())
      }

      if let error = error {
          print("[connection] \(error)")
          self.cancel()
      } else {
          self.receiveNextMessage()
      }
    }
  }
}
{% endhighlight %}

Using a struct wrapping an enum and a custom Codable implementation, we can exchange just one struct and get all the messages we need.

{% highlight swift %}

struct PeerShareData: Codable {
    enum Message {
        case navLog(navLog: NavLog)
        case flightPlan(flightPlan: FlightPlan)
        // add more as needed...
    }
    
    private enum MessageType: String, Codable {
        case navLog
        case flightPlan
    }
    
    let message: Message
    private let messageType: MessageType
    
    private enum CodingKeys: String, CodingKey {
        case messageType
        case navLog
        case flightPlan
    }
    
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        messageType = try container.decode(MessageType.self, forKey: .messageType)
        switch messageType {
        case .navLog:
            let response = try container.decode(NavLog.self, forKey: .navLog)
            message = .navLog(navLog: response)
        case .flightPlan:
            let response = try container.decode(FlightPlan.self, forKey: .flightPlan)
            message = .flightPlan(flightPlan: response)
        }
    }
    
    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(messageType, forKey: .messageType)
        switch message {
        case .navLog(let navLog):
            try container.encode(navLog, forKey: .navLog)
        case .flightPlan(let flightPlan):
            try container.encode(flightPlan, forKey: .flightPlan)
        }
    }
    
    init(from navLog: PeerNavLog) {
        messageType = .navLog
        message = .navLog(navLog: navLog)
    }
    
    init(from flightPlan: FlightPlan) {
        messageType = .flightPlan
        message = .flightPlan(flightPlan: flightPlan)
    }
}
{% endhighlight %}

<br/>

Seamless Connectivity
---

Several scenarios will cause havoc with peer-to-peer Wi-Fi, and this is where our own MultipeerSession object can come into its own.

The first messy situation is when the app is backgrounded and we want to shut down the connection. When the app resumes in the foreground we'll want to re-establish connections.

{% highlight swift %}
class MultipeerSession {
    
    // ...
    
    init(..., notificationCenter: NotificationCenter = NotificationCenter.default) {
        ...
        
        notificationCenter.addObserver(self, selector: #selector(appDidBecomeActive(_:)), name: UIApplication.didBecomeActiveNotification, object: nil)
        notificationCenter.addObserver(self, selector: #selector(appDidEnterBackground(_:)), name: UIApplication.didEnterBackgroundNotification, object: nil)
    }

    // MARK: - Monitor Application State
    
    @objc func appDidEnterBackground(_ notification: Notification) {
        browser?.stopBrowsing()
        connections.forEach { $0.cancel() }
        connections.removeAll()
        updatePeerList()
    }
    
    @objc func appDidBecomeActive(_ notification: Notification) {
        browser?.startBrowsing()
    }
{% endhighlight %}

The second scenario is failing or flaky connections. Despite configuring the TCP keep-alive to 2 seconds, observations suggest failed connections can linger for a while, which does not result in a good user experience.

One solution is to add a higher level "keep-alive", which periodically checks connections with a ping, and aggressively terminates failing connections, allowing them to be re-established. If someone out there can manage to achieve this level of reliability at the TCP level, please let me know!

{% highlight swift %}
class MultipeerSession {
    
    // ....
    
    private var timer: Timer?
       
    func startSharing() {
        // ....
        startReconnectionTimer()
    }
    
    func stopSharing() {
        stopReconnectionTimer()
        //....
    }
    
    // MARK: - Periodic Connectivity Checks
       
    private func startReconnectionTimer() {
        timer = Timer.scheduledTimer(withTimeInterval: config.connectivityCheckInterval, repeats: true, block: { [weak self] _ in
           guard let self = self else { return }
           self.pingExistingConnections()
           self.killFailingConnections()
           self.attemptNewConnections()
           self.updatePeerList()
       })
    }

    private func stopReconnectionTimer() {
       timer?.invalidate()
    }
       
    private func attemptNewConnections() {
        browserResults.forEach { result in
            if shouldConnectTo(result) {
                let networkConnection = NWConnection(to: result.endpoint, using: NWParameters(passcode: config.presharedKey))
                let peerConnection = PeerConnection(connection: networkConnection, delegate: self)
                connections.append(peerConnection)
                updatePeerList()
            }
        }
    }
    
    private func pingExistingConnections() {
        connections.forEach {
            $0.sendMessage(type: InternalMessageType.ping, content: Data())
        }
    }
    
    private func killFailingConnections() {
        connections.filter {
            $0.connection?.state == .preparing
            && Date().timeIntervalSince($0.created) > config.failedConnectionTimeout
        }.forEach {
            $0.cancel()
        }
        
        connections.filter {
            $0.connection?.state == .ready
            && Date().timeIntervalSince($0.lastPing) > config.failedConnectionTimeout
        }.forEach {
            $0.cancel()
        }
    }
}
{% endhighlight %}

This results in a fairly consistent "seamless connectivity" experience. No user intervention required!

demo video???

<br/>

Unit Testing
---

How do you unit test something that needs multiple devices to work?

Normally such unit testing would be done by mocking out the dependencies - in this case the Network framework. However, it's generally not a great idea to mock out objects that you don't own, simply because you really don't know how they work, and so your unit tests are only really testing against your understanding of how the framework works.

For this library, there is a better way! There is nothing stopping us from creating _multiple_ MultipeerSession objects, and communicating between them. That's possible thanks to the curious way that NWBrowser will return a list of discovered services.. including itself!

On with the unit tests...

{% highlight swift %}
import XCTest
@testable import p2pshare

class p2pshareTests: XCTestCase {

    var session1: MultipeerSession?
    var session2: MultipeerSession?
    
    override func setUp() {
        let myPeerInfo1 = PeerInfo(["name": "Tim"])
        let service = "_test1._tcp"
        let psk = "1234"
        let config1 = MultipeerSessionConfig(myPeerInfo: myPeerInfo1, bonjourService: service, presharedKey: psk)
        let browser1 = PeerBrowser(bonjourService: service)
        let listener1 = PeerListener(bonjourService: service, presharedKey: psk, myPeerID: myPeerInfo1.peerID)
        session1 = MultipeerSession(config: config1, browser: browser1, listener: listener1)

        let myPeerInfo2 = PeerInfo(["name": "Jony"])
        let config2 = MultipeerSessionConfig(myPeerInfo: myPeerInfo2, bonjourService: service, presharedKey: psk)
        let browser2 = PeerBrowser(bonjourService: service)
        let listener2 = PeerListener(bonjourService: service, presharedKey: psk, myPeerID: myPeerInfo2.peerID)
        session2 = MultipeerSession(config: config2, browser: browser2, listener: listener2)
    }

    override func tearDown() {
        session1 = nil
        session2 = nil
    }

    func testTwoSessionsCanConnect() {
        var result: [PeerInfo]?
        
        let exp = XCTestExpectation()
        session1!.peersChangeHandler = { peers in
            if peers.count > 0 {
                result = peers
                exp.fulfill()
            }
        }
        wait(for: [exp], timeout: 20.0)
        
        XCTAssertNotNil(result)
        XCTAssertEqual(result!.count, 1)
        XCTAssertEqual(result![0].info["name"], "Jony")
    }

    // and so on ..
}
{% endhighlight %}



[multipeer-connectivity]: https://developer.apple.com/documentation/multipeerconnectivity
[wwdc-2019-advanced-networking]: https://developer.apple.com/videos/play/wwdc2019/713/
[network-framework]: https://developer.apple.com/documentation/network
[flying-over-bondi]: /not-much-connectivity-here.jpg "Boeing 737 flight deck over Bondi, Sydney Australia"
[nwlistener]: https://developer.apple.com/documentation/network/nwlistener
[nwbrowser]: https://developer.apple.com/documentation/network/nwbrowser
[nwconnection]: https://developer.apple.com/documentation/network/nwconnection
