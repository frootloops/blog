---
title: Monitoring performance of screens in your iOS app
date: 2022-07-04 12:37:22
---

Today I want to tell you a story about performance in iOS applications. Since I’m working as an iOS developer on the trading product at Revolut sometimes I get reports like:

> some screen is laggy, can you please have a look?

Naturally I go to Xcode, open Profiler and I try to make a sense out of it. But honestly it’s hard to do so. Mainly because I get so much data and I don’t have a clear guidance where to look. I feel like I’m looking for a needle in a haystack.

![](https://miro.medium.com/max/1400/1*af5bUumJNu5RnStiQhhQhA.png)

But if you think about it, most of the time you have a blocking call somewhere inside viewDidLoad, viewWillAppear or viewDidAppear.

So when I’m researching such reports what I usually do is I go to the screen I need to check for performance issues and I write something like this:

``` swift
override func viewDidLoad() {
    super.viewDidLoad()
    let start = CFAbsoluteTimeGetCurrent()
    // actual code
    let diff = CFAbsoluteTimeGetCurrent() - start
    print("viewDidLoad took \(diff)ms")
}
```

This helps me to measure execution time but it doesn’t help me to navigate Profiler more easily. So I started to research and to watch WWDC videos to get ideas about what can help, and that led me to os_signpost.

What signpost does is basically the same as we did with CFAbsoluteTimeGetCurrent above. But instead of just printing a statement it passes this information to the system and we can reach this data via Profiler later. Let’s try it.

``` swift
override func viewDidLoad() {
    super.viewDidLoad()
    let log = OSLog(subsystem: "com.revolut", category: .pointsOfInterest)
    let signpostID = OSSignpostID(log: log)
    os_signpost(.begin, log: log, name: "viewDidLoad", signpostID: signpostID)
    // actual code
    os_signpost(.end, log: log, name: "viewDidLoad", signpostID: signpostID)
}
```

Let’s have a look at each line separately to better understand it. We start by defining the log variable and to define it we specify a subsystem which is pretty much just an identifier and category.

You can create your own category but it’s better to use an existing category created by Apple called pointsOfInterest. The reason why we want to use this category is that it automatically appears in Profiler (whereas if you create your own category you’d need to manually add it to the screen).

Signpost id helps the system to match events.

The os_signpost call tells the system that you’re about to begin a task with the given name (viewDidLoad) alongside signpostID. The second os_signpost call tells the system that you’re done and it can calculate execution time.

Now if we open Profiler and zoom in a bit we will see a bar that indicates execution time of a given viewDidLoad. If you double click to the bar it will focus on it, go to the Time Profiler section and now you can see what actually affects performance of the given viewDidLoad.

![](https://miro.medium.com/max/1400/1*2y7B4x5jon1fxTC1-_6gUA.png)

Doing this helped me to find and fix the issue that was reported but it got me thinking: ‘how can I make sure I don’t have to constantly write and delete this code?’ since you have to write this code inside each controller and delete it afterwards.

So, we need a better solution. In comes Method swizzling! We can replace viewDidLoad with our own copy and wrap all viewDidLoads into this measuring code, right?

I tried and I failed. Since you’re creating subclasses for view controller and you’re probably going to swizzle original UIViewController.viewDidLoad , what you’re going to measure is actually this

``` swift
let originalMethod = class_getInstanceMethod(UIViewController.self, #selector(viewDidLoad))
let swizzledMethod = class_getInstanceMethod(UIViewController.self, #selector(swizzledDidLoad))
method_exchangeImplementations(originalMethod, swizzledMethod)


override func viewDidLoad() {
    let log = OSLog(subsystem: "com.revolut", category: .pointsOfInterest)
    let signpostID = OSSignpostID(log: log)
    os_signpost(.begin, log: log, name: "viewDidLoad", signpostID: signpostID)
    super.viewDidLoad()
    os_signpost(.end, log: log, name: "viewDidLoad", signpostID: signpostID)
    
    // and not this
    // actual code
}
```

So you end up measuring only what happens inside super.viewDidLoad() but in reality we want to measure viewDidLoad of each subclass separately. And oh boy, this is tricky.

My initial approach to this problem was to traverse all subclasses of UIViewController at runtime and swizzle their viewDidLoad function. However, this operation takes time, plus it feels weird and unnecessary invasive. So instead I start thinking about how I can do this on demand, as I present screens. At this moment a classic interview question pops to my mind: a life cycle of view controllers. viewDidLoad, viewWillAppear… loadView! Eureka!

Now I can swizzle UIViewController.loadView and when somebody tries to show a screen I can swizzle viewDidLoad of this specific subclass right before we show it. Problem solved, right?

``` swift
func swizzledLoadView() {
    let originalMethod = class_getInstanceMethod(type(of: self), #selector(viewDidLoad))
    let swizzledMethod = class_getInstanceMethod(type(of: self), #selector(swizzledDidLoad))
    method_exchangeImplementations(originalMethod, swizzledMethod)
    swizzledLoadView()
}
```

If you try to run this code for let’s say ViewControllerA and then for ViewControllerB you’ll be surprised, shocked and probably will reconsider your life choices. In this case, instead of calling ViewControllerB’s viewDidLoad method you’ll see that ViewControllerA’s viewDidLoad is being called 🤯. But why?

It took me some time to figure it out. Let’s think about this together. When you swap ViewControllerA’s viewDidLoad with swizzledViewDidLoad what happens is that now swizzledViewDidLoad references to ViewControllerA’s viewDidLoad (that’s the reason why we call swizzledLoadView inside swizzledLoadView and this does not result in an endless loop).

Now we try to run the same code for ViewControllerB. By doing so we exchange ViewControllerA’s viewDidLoad with swizzledViewDidLoad again. But this time swizzledViewDidLoad is not a function that we created to measure performance of original viewDidLoad. Now this function is ViewControllerA’s viewDidLoad.

So, now we know this approach doesn’t work. What we actually want is to create a swizzledViewDidLoad method at runtime so every ViewController gets its own version of it. Sounds simple, right? But again it took me quite a bit of researching to get it done since the code is far from straightforward. Let’s dive in!

``` swift
func swizzleViewDidLoad(target controller: UIViewController.Type) {
    let selector = #selector(UIViewController.viewDidLoad)
    let viewDidLoadImp = class_getMethodImplementation(controller, selector)

    let swizzledViewDidLoadImp = imp_implementationWithBlock(
        { (receiver: UIViewController) in
            let viewDidLoad = unsafeBitCast(
                viewDidLoadImp,
                to: (@convention(c)(UIViewController, Selector) -> Void).self
            )

            let start = CFAbsoluteTimeGetCurrent()
            viewDidLoad(receiver, selector)
            let diff = CFAbsoluteTimeGetCurrent() - start
            print("viewDidLoad took \(diff)")
        } as @convention(block) (UIViewController) -> Void
    )

    let viewDidLoadMethod = class_getInstanceMethod(controller, selector)!

    method_setImplementation(
        viewDidLoadMethod,
        swizzledViewDidLoadImp
    )
}
```

I know this is scary code but bear with me. Once I tell you what each line does, you’ll get it.

We start simple. This is a selector that we store as a variable for convenience.

``` swift
let selector = #selector(UIViewController.viewDidLoad)
```

Next we take an existing legit implementation of viewDidLoad for this specific controller and store it in a variable too. This way we can execute the original implementation when needed.

``` swift
let viewDidLoadImp = class_getMethodImplementation(controller, selector)
```

Then we define a new method which is based on a closure. Note as @convention(block) (UIViewController) -> Void means that we pass this closure as an objective C closure rather than swift closure. Also, we expect UIViewController to be passed to this closure. This UIViewController is going to be an active instance of a view controller.

``` swift
let swizzledViewDidLoadImp = imp_implementationWithBlock(
    { (receiver: UIViewController) in
		    // ...
    } as @convention(block) (UIViewController) -> Void
)
```

Since class_getMethodImplementation returns an IMP we need to convert this type to a swift closure so we can call it. Note @convention(c) means a pointer to a C function.

``` swift
let viewDidLoad = unsafeBitCast(
    viewDidLoadImp,
    to: (@convention(c)(UIViewController, Selector) -> Void).self
)
```

Now we get to the fun part! Here we execute the original viewDidLoad and measure execution time. Please note that we’re using standard C calling convention here which means we pass self (an instance of view controller) as argument number one, a selector as an argument number two and actual parameters of a function after that, for example viewWillAppear(receiver, selector, animated).

``` swift
let start = CFAbsoluteTimeGetCurrent()
viewDidLoad(receiver, selector)
let diff = CFAbsoluteTimeGetCurrent() - start
print("viewDidLoad took \(diff)")
```

Here we obtain a reference to a method which we’ll need to replace the original implementation.

``` swift
let viewDidLoadMethod = class_getInstanceMethod(controller, selector)!
```

And the last step. We replace the original implementation with a newly created one.

``` swift
method_setImplementation(
    viewDidLoadMethod,
    swizzledViewDidLoadImp
)
```

Well done! Now you know how to dynamically create functions at runtime and use them to replace other methods.

Now let’s recap how it works. When we run an app we replace the original loadView method with our method. This replaces life cycle events with functions that measure execution time of the original methods.

Keep in mind that loadView is going to be called once per instance but we only want to swizzle methods once per type. So we’re going to define a global variable where we’ll store already swizzled types.

``` swift
var swizzled: [String] = []
```

And when we’re about to swizzle we’ll check that we haven’t done this before

``` swift
let classname = NSStringFromClass(type(of: self))
if !swizzled.contains(classname) &&
    !classname.starts(with: "_") &&
    "^[A-Z]{2}".range(of: classname, options: .regularExpression) == nil {
    // ....
}
swizzledLoadView()
```

We’re also going to make sure we don’t mess with private controllers (their names start with an underscore) nor with controllers from SDKs since they usually have two-letter prefixes.

That’s it. Now we have a tool that’s always on and will always report execution time, allowing us to stop problems earlier.

## Disclaimer

Using runtime magic can be dangerous and can lead to unexpected crashes. All code listed above only intended to be used during development. This code must not be distributed to actual users.

