---
title: How to extract analytics data from your iOS application
date: 2017-09-25 12:40:12
---
As a data driven company, it’s essential that we understand how our users work the app in order to improve their experience. In this blog, I’m going to explain how we extract and use analytical data from the iOS version of the app.

![](https://miro.medium.com/max/1400/1*5AT76efFGNNcIIE_mNU2rg.jpeg)

Here at Revolut, we use two main platforms (Google Analytics and Amplitude) to aggregate insights from a multitude of data points. Today, we’re going to show you how to create an expandable, reliable and testable solution for mobile analytics.

## Implementation

We start by defining a protocol for tracking events that can occur in our application. Let’s call it **TrackingEventType**.

``` swift
protocol TrackingEventType {
    var identifier: String { get }
    var properties: [String: Any] { get }
}
```

This protocol has two attributes: A unique **identifier**, used to differentiate one event from another and **properties**, which we use to share data with the analytics platforms.

Next, we create an enum where each case represents a single event.

``` swift
enum UserEvents {
    case created(user: User)
    case openApp
    case bought(product: Product)
    case sentMessage
}
```

We now have to implement **TrackingEventType** to set up the relationship between each case and its identifiers and properties.

``` swift
extension UserEvents: TrackingEventType {
    var identifier: String {
        switch self {
        case .created:
            return "user.created"
        case .openApp:
            return "user.open-app"
        case .bought:
            return "user.bought"
        case .sentMessage:
            return "user.sent-message"
        }
    }

    var properties: [String: Any] {
        switch self {
        case let .created(user):
            return ["name": user.name, "age": user.age]
        case let .bought(product):
            return ["title": product.title, "price": product.price]
        default:
            return [:]
        }
    }
}
```

Great, so now we know what events to track but we don’t know how to actually track them. We need to define one or more providers; Provider is a class that represents an analytics system such as Google Analytics or Amplitude. Let’s have a look at a protocol that all providers must conform to:

``` swift
protocol TrackingProviderType {
    func track(event: TrackingEventType)
    func associate(with user: User)
    func disassociateUser()
    func setup()
}
```

A provider does 4 things:

* Sets itself up to track events
* Accepts events and is actively tracks them
* Associates tracking sessions with a given user
* Disassociates tracking session from a user

In our case, since we use both GA and Amplitude, we need to create separate providers for each analytics system. For tracking events in Amplitude, we create **AmplitudeProvider**. Let’s take a look at the implementation.

``` swift
final class AmplitudeProvider: TrackingProviderType {
    private let amplitude = Amplitude()

    func setup() {
        amplitude.initializeApiKey("XXXX-XXXX")
        amplitude.eventUploadPeriodSeconds = 120
    }

    func track(event: TrackingEventType) {
        amplitude.logEvent(event.identifier, withEventProperties: event.properties)
    }

    func associate(with user: User) {
        amplitude.setUserId(user.id)

        let identify = AMPIdentify()
        identify.set("name", value: user.name)
        identify.set("price", value: user.price)
        amplitude.identify(identify)
    }

    func disassociateUser() {
        amplitude.setUserId(nil)
    }
}
```

While the implementation is pretty straight-forward, it’s worth going through each step quickly.
* In the **setup** method, we’re setting up a connection with Amplitude. This usually means that we need to provide some kind of key for the analytics system.
* In **track(event: TrackingEventType)** we take an identifier along with the properties of an event and send it directly to Amplitude.
* In **associate(with user: User)** we set up user-specific variables for correct session logging. Note that you have to call this method right after a user logs in.
* **disassociateUser** is a where you must break the link between the current tracking session and the user. You must call it after the user logs out.

The last class we need is a client, which does two things. It stores all providers and it forwards events to the providers. Let’s have a look at the protocol:

``` swift
protocol TrackingClientType: TrackingProviderType {
    var providers: [TrackingProviderType] { get }
    func add(provider: TrackingProviderType)
}
```

**TrackingClientType** inherits from **TrackingProviderType** as it forwards all calls to a bunch of providers. Let’s take a look at how we can implement the client.

``` swift
class TrackingClient: TrackingClientType {
    var providers = [TrackingProviderType]()

    func add(provider: TrackingProviderType) {
        providers.append(provider)
    }

    func track(event: TrackingEventType) {
        for provider in providers {
            provider.track(event: event)
        }
    }
  
    func associate(with user: User) {
        for provider in providers {
            provider.associate(with: user)
        }
    }

    func disassociateUser() {
        for provider in providers {
            provider.disassociateUser()
        }
    }

    func setup() {
        for provider in providers {
            provider.setup()
        }
    }
}
```

As you can see, the client simply stores providers and forwards events to all of them. In the rest of the app you will be working almost exclusively with an instance of a client.

## Usage

The best way to deal with our analytics system is to create an instance of **TrackingClient** and feed it with all required providers. A good place to do so is **didFinishLaunchingWithOptions** method. Next, call the **setup()** method to allow providers to set themselves up properly.

``` swift 
func application(_ application: UIApplication,  didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey : Any]? = nil) -> Bool {
    App.trackingClient = TrackingClient()
    App.trackingClient.add(provider: AmplitudeProvider())
    App.trackingClient.add(provider: GoogleProvider())
    App.trackingClient.add(provider: FacebookProvider())
    App.trackingClient.setup()
  
    return true
}
````

When it’s time to start tracking events, you simply need to pass an event to the tracking client like this:

``` swift
let arsen = User(id: "1", name: "Arsen", age: 26)
App.trackingClient.track(event: UserEvents.created(user: arsen))
```

That’s it — you’re good to go!

## Bonus — Testability

You’re probably wondering why we’ve hidden all the classes behind protocols. We did it for a purpose — testability. Since we do not rely on a particular implementation, we can easily create a special implementation of **TrackingClientType**, giving us the ability to test our code.

Imagine you’re working on a page where users can buy your products and you want to be able to see all purchases made in Google Analytics.

To make this possible, we will create a dummy provider that will expose all tracked events.

``` swift
class TestTrackingClient: TrackingClientType {
    var providers = [TrackingProviderType]()
    var events = [TrackingEventType]()

    func track(event: TrackingEventType) {
        events.append(event)
    }
}
```

You can now replace a normal client with the dummy one and test away!

``` swift
App.trackingClient = TestTrackingClient()
let productViewController = ProductViewController(product: product)
productViewController.buy()
App.trackingClient.events[0] == UserEvents.bought(product: product)
```

Because **TestTrackingClient** exposes all tracked events via events property, it’s really easy to check what our code tracked.

## Conclusion

By now, you should be able to build your very own extendable, reliable and testable solution for your iOS application. The solution is based on protocol-oriented programming and enums with associated values — feel free to check out the [source code](https://gist.github.com/frootloops/6418491def7d6ef29667c90a4842a23a).

*As always, we welcome your feedback, comments and suggestions.*
