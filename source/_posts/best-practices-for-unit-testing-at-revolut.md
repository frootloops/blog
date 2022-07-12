---
title: Best Practices For Unit Testing at Revolut
date: 2021-08-31 21:41:41
---

Testing is important. Unit testing is super important, I’m not questioning that. But sometimes, you write some code and you’d love to cover it with tests, but it feels like it would be so much effort. So you avoid writing tests, or you write less than you would like to. Sound familiar?

![](https://miro.medium.com/max/1400/1*NkaJoUvbW5GIKFV_Fm5KAQ.png)

I get it, but I also value unit tests and I think writing them should be fun. Otherwise, what’s the point? So at some point I started to analyse why writing unit tests feel like such a chore. I thought finding these showstoppers and streamlining them would make writing unit tests fun again. And make me more productive.

I did it. And I’m sharing it with you. Here’s the list of best practices I follow to keep writing unit tests easy.

## The Sample Function

Imagine you have a function that takes some business model, let’s say PriceAlert and the current price, and if the current price exceeds the target price the function returns true. You don’t care about all other properties of the PriceAlert, you care only about the target price. But here’s the friction point: you want a price alert with a specific target price but you have to fill all other properties.

``` swift 
struct PriceAlert: Equatable {
    let id: String
    let symbol: String
    let targetPrice: MoneyItem
    let createdAt: Date
}

// test
let priceAlert = PriceAlert(
  id: String = "42",
  symbol: String = "AAPL",
  targetPrice: MoneyItem = MoneyItem(amount: 1050, currency: "USD"),
  createdAt: Date = Date(timeIntervalSince1970: 1526202500)
)

check(
  priceAlert: priceAlert,
  currectPrice: MoneyItem(amount: 123, currency: "USD")
)
```

To solve that, every business model should be provided with a factory method that can create an instance with no input but, if you want to, you can override some properties. Sound familiar?

``` swift
extension PriceAlert {
    static func sample(
        id: String = "42",
        symbol: String = "AAPL",
        targetPrice: MoneyItem = MoneyItem(amount: 1050, currency: "USD"),
        createdAt: Date = Date(timeIntervalSince1970: 1526202500)
    ) -> PriceAlert {
        return PriceAlert(
            id: id,
            symbol: symbol,
            targetPrice: targetPrice,
            createdAt: createdAt
        )
    }
}
```

As you can see, we provide a default value for every property so if you really don’t care, you can just ask for an instance. In most cases, you want to provide one or two specific values and you can do that, too. Also, keep in mind that every sample function is a pure function, so you shouldn’t use any dynamic values like Date().

## The MockFunc Object

The second problem I noticed was the mocks. More specifically, the way we used to set them up. Originally, we used a closure-based generator to create mocks. Something like this:

``` swift
class AnimatorSpy: Animator {

  var invokedAnimate = false
  var invokedAnimateCount = 0
  var invokedAnimateParameters: (duration: TimeInterval, Void)?
  var invokedAnimateParametersList = [(duration: TimeInterval, Void)]()
  var shouldInvokeAnimateAnimations = false
  var stubbedAnimateCompletionResult: (Bool, Void)?
  var stubbedAnimateResult: Bool! = false

  func animate(duration: TimeInterval, animations: () -> (), completion: (Bool) -> ()) -> Bool {
    invokedAnimate = true
    invokedAnimateCount += 1
    invokedAnimateParameters = (duration, ())
    invokedAnimateParametersList.append((duration, ()))
    if shouldInvokeAnimateAnimations {
      animations()
    }
    if let result = stubbedAnimateCompletionResult {
      completion(result.0)
    }
    return stubbedAnimateResult
  }
}
```

But the problem with these mocks was that it was super verbose and it cluttered test files really quickly. So again, friction. To solve that problem, I created a simple object to represent the act of mocking, if you will.

``` swift
public struct MockFunc<Input, Output> {
    public var parameters: [Input] = []
    public var result: (Input) -> Output = { _ in fatalError() }

    public mutating func callAndReturn(_ input: Input) -> Output {
	parameters.append(input)
        return result(input)
    }

    public mutating func returns(_ value: Output) {
        result = { _ in value }
    }
}
```

So an actual mock looks like this:

``` swift
final class PriceAlertRepositoryMock: PriceAlertRepositoryProtocol {
    var cachedPriceAlertsMockFunc = MockFunc<String?, [PriceAlert]>()
    func cachedPriceAlerts(symbol: String?) -> [PriceAlert] {
        return cachedPriceAlertsMockFunc.callAndReturn(symbol)
    }
}
```

The mock object works very simply. It knows what you put in and what you get out of it. But, of course, it doesn’t know how to actually transform input to output. This is where you come in. In most cases, you just provide output no matter what input is. Like this:

``` swift
var priceAlertRepository = PriceAlertRepositoryMock()
priceAlertRepository.cachedPriceAlertsMockFunc.returns([.sample(id: "53")])
```

The MockFunc also has quite a few handy shortcuts:


``` swift
someMockFunc.succeeds(.sample()) // if ouput is Result<T>
someMockFunc.fails(error) // if ouput is Result<T>
someMockFunc.returnsNil() // if ouput is T?
someMockFunc.returns() // if ouput is Void
expect(someMockFunc).to(beCalled())
```

The full implementation of the MockFunc can be found [here](https://gist.github.com/frootloops/de256f714db0bdbde9499381e5dd83cd).

## The Builders

Another problem I noticed was in the first phase of any unit tests — preparation of the main actor. In this case, it’s more about readability but readability still helps you to write unit tests, because you can read existing ones faster. It counts, guys.

So, we used to prepare everything inside a test case like this:

``` swift
// preparation
var priceAlertApiService = PriceAlertApiServiceMock()
var priceAlertStorageService = PriceAlertStorageServiceMock()

let repository = PriceAlertRepository(
	network: priceAlertApiService,
	storage: priceAlertStorageService
)

// actual test
priceAlertStorageService.cachedPriceAlertsMockFunc.returns(
	[.sample(id: "1"), .sample(id: "2")]
)

let result = repository.cachedPriceAlerts(symbol: "AAPL")
expect(result.map { $0.id }) == ["1", "2"]
```

This simple example preparation takes the same amount of space as the actual test. Not good! But don’t worry, a builder saves the day:

``` swift
private class Builder {
    var priceAlertApiService = PriceAlertApiServiceMock()
    var priceAlertStorageService = PriceAlertStorageServiceMock()

    func makeRepository() -> PriceAlertRepository {
        let repository = PriceAlertRepository(
            network: priceAlertApiService,
            storage: priceAlertStorageService
        )
        return repository
    }
}
```

So, now we don’t need to clutter a test case itself and we can focus on the fun part — writing actual unit tests.

``` swift
// preparation
let builder = Builder()
let repository = builder.makeRepository()

// actual test
builder.priceAlertStorageService.cachedPriceAlertsMockFunc
    .returns([.sample(id: "1"), .sample(id: "2")])

let result = repository.cachedPriceAlerts(symbol: "AAPL")
expect(result.map { $0.id }) == ["1", "2"]
```

It helps to reduce the noise. Also it gives you the ability to share a common setup with other test cases. Let’s say we want to setup the repository with some existing cache, you could create a custom makeRepository and simplify the code above to:

``` swift
let repository = builder.makeRepository(
	with: [.sample(id: "1"), .sample(id: "2")]
)
```

## Conclusion

If you look at these best practices separately, they may look minor. But if you combine them, they really help us keep test coverage in check and deliver new updates every week with little to no regression. But we’re not done yet. Please share your best practices, I would love to add them to my collection!
