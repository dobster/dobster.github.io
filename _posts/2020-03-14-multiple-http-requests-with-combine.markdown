---
layout: post
title:  "Coordinating multiple HTTP requests with Combine framework"
date:   2020-03-14 07:00:00 +1100
categories: iOS iPadOS URLSession Combine.framework
---

<br/>

A common challenge for mobile development when faced with less-than-ideal backend API services is dealing with multiple endpoints that deliver heterogeneous data that you'd rather process as one single dataset.

This may arise for many reasons including
* you don't want to proceed unless you have all the data
* it will be harder to keep local database integrity if you process each response data individually

I would normally use a [DispatchGroup][dispatch-group] to achieve this. Launch multiple requests and use the semaphore controls to wait until all the responses have come back.

For example...

{% highlight swift %}
let dispatchGroup = DispatchGroup()
var errors = [Error]()
var image1: UIImage?
var image2: UIImage?

dispatchGroup.enter()
URLSession.shared.dataTask(with: url1) { (data, response, error) in
    if let data = data, let image = UIImage(data: data) {
        image1 = image
    } else {
        errors.append(error ?? ImageFetchError.somethingWrong)
    }
    dispatchGroup.leave()
}.resume()

dispatchGroup.enter()
URLSession.shared.dataTask(with: url2) { (data, response, error) in
    if let data = data, let image = UIImage(data: data) {
        image2 = image
    } else {
        errors.append(error ?? ImageFetchError.somethingWrong)
    }
    dispatchGroup.leave()
}.resume()

dispatchGroup.wait()

if !errors.isEmpty {
    print(errors)
} else {
    imageView1.image = image1
    imageView2.image = image2
}
{% endhighlight %}

Not too bad if we're just waiting for two requests, but the mess of juggling of responses and errors doesn't scale nicely when we inevitably end up with five or six endpoints we need to coordinate.

One of the highlights of WWDC 2019 was the introduction of the functional-reactive framework [Combine][combine-framework], and in the [Advanced Networking 1 session at WWDC2019][wwdc-2019-advanced-networking] session, it was introduced as..

> Networking is inherently asynchronous, that's why it's perfect to adopt Combine.

So, how would we use Combine for this problem?

<br/>

Coordinating multiple requests with Combine
-----

The Combine extension to URLSession is particularly neat as it handles the error cases very nicely.

{% highlight swift %}
let dataTask1 = URLSession.shared
    .dataTaskPublisher(for: url1)
    .map { data, _ in UIImage(data: data) ?? UIImage() }

let dataTask2 = URLSession.shared
    .dataTaskPublisher(for: url2)
    .map { data, _ in UIImage(data: data) ?? UIImage() }

dataTask1.zip(dataTask2)
    .receive(on: RunLoop.main)
    .sink(receiveCompletion: {
        if case .failure(let error) = $0 {
            print(error)
        }
    }, receiveValue: { [weak self] (image1, image2) in
        self?.imageView1.image = image1
        self?.imageView2.image = image2
    })
    .store(in: &subscriptions)
{% endhighlight %}

A simple `zip()` does the trick! 

This method scales well when you have many requests to coordinate.

Of course, it's no substitution for sorting out your backend APIs... 😀

[wwdc-2019-advanced-networking]: https://developer.apple.com/videos/play/wwdc2019/713/
[combine-framework]: https://developer.apple.com/documentation/combine
[dispatch-group]: https://developer.apple.com/documentation/dispatch/dispatchgroup
