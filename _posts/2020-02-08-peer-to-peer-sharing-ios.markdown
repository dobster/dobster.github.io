---
layout: post
title:  "Sharing between iOS apps via peer wifi"
date:   2020-02-08 18:24:35 +1100
categories: iOS iPadOS iPad Network.framework
---
Adopting Network.framework for peer-to-peer device connectivity.

Apple introduced the [Multipeer Connectivity][multipeer-connectivity] framework way back in 2013. This unheralded framework allows discovery of other nearby devices and exchange of data using peer-to-peer connectivity. For iOS, this would be via Wi-Fi networks, peer-to-peer Wi-Fi and Bluetooth.

At Qantas we have been using this framework for peer-to-peer sharing of flight Nav Log entries between the pilot iPads during flight. 

In our use-case, during a flight pilots are required to make entries in the Nav Log, for both regulatory audit purposes and also to review progress and fuel consumption against the original flight plan. For long-haul flights, two pilots will be on duty at a time, and a shift change mid-flight will require all the Nav Log entries to be shared with the incoming crew.

Without a reliable satellite connection, client-to-server syncing to exchange data between devices is not possible.

*Problems with the Multipeer Connectivity framework*

Bloah




Jekyll also offers powerful support for code snippets:

{% highlight swift %}
class MultipeerSession {
  static let service = "YOHOO"
}
{% endhighlight %}

X.

[multipeer-connectivity]: https://developer.apple.com/documentation/multipeerconnectivity
