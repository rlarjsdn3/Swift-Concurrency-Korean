# Async/Await: Sequences

* 제안: [SE-0298](0298-asyncsequence.md)
* 저자: [Tony Parker](https://github.com/parkera), [Philippe Hausler](https://github.com/phausler)
* 리뷰 매니저: [Doug Gregor](https://github.com/DougGregor)
* 상태: **구현됨 (Swift 5.5)**
* 구현: [apple/swift#35224](https://github.com/apple/swift/pull/35224)
* 결정 기록: [Rationale](https://forums.swift.org/t/accepted-with-modification-se-0298-async-await-sequences/44231)
* 개정: Based on [forum discussion](https://forums.swift.org/t/pitch-clarify-end-of-iteration-behavior-for-asyncsequence/45548)

## Introduction

Swift의 [async/await](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0296-async-await.md) 기능은 (미래의 어느 시점에) 단일 값을 반환하는 비동기 함수를 직관적이고 내장된 방식으로 작성하고 사용할 수 있도록 도와줍니다. 우리는 이 기능을 기반으로 하여, 시간이 지남에 따라 여러 값을 반환하는 함수를 직관적이고 내장된 방식으로 작성하고 사용할 수 있는 기능을 제안합니다.

이 제안은 다음과 같은 요소로 이루어져 있습니다:

1. 비동기 시퀀스(asynchornous sequence)를 표현하는 프로토콜의 표준 라이브러리 정의

2. 비동기 시퀀스에 대해 `for...in` 구문을 사용할 수 있도록 하는 컴파일러 지원

3. 비동기 시퀀스를 다루는 데 자주 사용되는 함수들의 표준 라이브러리 구현

<details>

<summary>원문 보기</summary>

Swift's [async/await](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0296-async-await.md) feature provides an intuitive, built-in way to write and use functions that return a single value at some future point in time. We propose building on top of this feature to create an intuitive, built-in way to write and use functions that return many values over time.

This proposal is composed of the following pieces:

1. A standard library definition of a protocol that represents an asynchronous sequence of values
2. Compiler support to use `for...in` syntax on an asynchronous sequence of values
3. A standard library implementation of commonly needed functions that operate on an asynchronous sequence of values

</details>

## Motivation

비동기 시퀀스를 순회하는 작업은 동기 시퀀스를 순회하는 것만큼 간단하길 바랍니다. 예를 들어, 파일의 각 줄을 하나씩 읽는 경우를 생각해볼 수 있습니다:

```swift
for try await line in myFile.lines() {
  // Do something with each line
}
```

Swift 개발자에게 이미 익숙한 `for...in` 구문을 사용함으로써, 비동기 API를 다루는 진입 장벽을 낮출 수 있습니다. 다른 Swift 타입 및 개념들과의 일관성 역시 우리가 추구하는 가장 중요한 목표 중 하나입니다. 이 구문에서 `await` 키워드를 사용하도록 요구함으로써, 동기 시퀀스와의 차이를 분명히 할 수 있습니다.


<details>

<summary>원문 보기</summary>

We'd like iterating over asynchronous sequences of values to be as easy as iterating over synchronous sequences of values. An example use case is iterating over the lines in a file, like this:

```swift
for try await line in myFile.lines() {
  // Do something with each line
}
```

Using the `for...in` syntax that Swift developers are already familiar with will reduce the barrier to entry when working with asynchronous APIs. Consistency with other Swift types and concepts is therefore one of our most important goals. The requirement of using the `await` keyword in this loop will distinguish it from synchronous sequences.

</details>



### `for/in` Syntax

`for in` 구문을 사용할 수 있도록 하려면, `func lines()`의 반환 타입이 컴파일러가 순회 가능한 것으로 인식할 수 있는 타입이어야 합니다. 현재 Swift에는 `Sequence` 프로토콜이 존재합니다. 여기서 이를 한번 사용해보겠습니다:

```swift
extension URL {
  struct Lines: Sequence { /* ... */ }
  func lines() async -> Lines
}
```

불행히도, 위 함수는 실제로 **모든** 줄이 준비될 때까지 기다린 후에야 반환됩니다. 그러나 우리가 원했던 방식은 각 줄을 하나씩 기다린 후(await) 순차적으로 처리하는 것이었습니다. 물론 `lines` 함수를 수정해 다르게 동작하도록(예: 참조 타입으로 결과를 반환하게) 만드는 것도 상상해볼 수 있지만, 이러한 순회 작업을 최대한 단순하게 만들기 위해서는 새로운 프로토콜을 정의하는 것이 더 나은 방법입니다.

```swift
extension URL {
  struct Lines: AsyncSequence { /* ... */ }
  func lines() async -> Lines
}
```

`AsyncSequence`는 연관된 반복자(iterator) 타입에 비동기 `next()` 함수를 정의함으로써, 전체 결과가 아니라 각 요소를 하나씩 기다릴 수 있도록 합니다.

<details>

<summary>원문 보기</summary>

To enable the use of `for in`, we must define the return type from `func lines()` to be something that the compiler understands can be iterated. Today, we have the `Sequence` protocol. Let's try to use it here:

```swift
extension URL {
  struct Lines: Sequence { /* ... */ }
  func lines() async -> Lines
}
```

Unfortunately, what this function actually does is wait until *all* lines are available before returning. What we really wanted in this case was to await *each* line. While it is possible to imagine modifications to `lines` to behave differently (e.g., giving the result reference semantics), it would be better to define a new protocol to make this iteration behavior as simple as possible.

```swift
extension URL {
  struct Lines: AsyncSequence { /* ... */ }
  func lines() async -> Lines
}
```

`AsyncSequence` allows for waiting on each element instead of the entire result by defining an asynchronous `next()` function on its associated iterator type.

</details>

### Additional AsyncSequence functions

한 걸음 더 나아가서, 새로 만든 `lines` 함수를 다양한 곳에서 어떻게 사용할 수 있을지 상상해봅시다. 예를 들어, 특정 길이를 초과하는 줄을 만날 때까지만 줄을 처리하고 싶을 수 있습니다.

```swift
let longLine: String?
do {
  for try await line in myFile.lines() {
    if line.count > 80 {
      longLine = line
      break
    }
  }
} catch {
  longLine = nil // file didn't exist
}
```

또는, 작업을 시작하기 전에 파일의 모든 줄을 읽어오고 싶을 수도 있습니다.

```swift
var allLines: [String] = []
do {
  for try await line in myFile.lines() {
    allLines.append(line)
  }
} catch {
  allLines = []
}
```

위 코드는 아무런 문제가 없으며, 개발자가 이를 작성할 수 있어야 합니다. 하지만, 자주 사용될 수 있는 일반적인 작업치고는 보일러플레이트(boilerplate)가 많아 보입니다. 이를 해결하는 한 가지 방법은 `URL`에 더 많은 함수를 추가하는 것입니다:

```swift
extension URL {
  struct Lines : AsyncSequence { }

  func lines() -> Lines
  func firstLongLine() async throws -> String?
  func collectLines() async throws -> [String]
}
```

비슷한 작업이 필요한 다른 경우들도 쉽게 떠올릴 수 있습니다. 따라서, 이러한 함수들을 `URL`에 직접 추가하기보다는 (`Sequence`와 마찬가지로) 일반적으로 활용할 수 있도록 `AsyncSequence`에 제네릭 확장으로 정의하는 게 가장 적절하다고 생각합니다.

<details>

<summary>원문 보기</summary>

Going one step further, let's imagine how it might look to use our new `lines` function in more places. Perhaps we want to process lines until we reach one that is greater than a certain length.

```swift
let longLine: String?
do {
  for try await line in myFile.lines() {
    if line.count > 80 {
      longLine = line
      break
    }
  }
} catch {
  longLine = nil // file didn't exist
}
```

Or, perhaps we actually do want to read all lines in the file before starting our processing:

```swift
var allLines: [String] = []
do {
  for try await line in myFile.lines() {
    allLines.append(line)
  }
} catch {
  allLines = []
}
```

There's nothing wrong with the above code, and it must be possible for a developer to write it. However, it does seem like a lot of boilerplate for what might be a common operation. One way to solve this would be to add more functions to `URL`:

```swift
extension URL {
  struct Lines : AsyncSequence { }

  func lines() -> Lines
  func firstLongLine() async throws -> String?
  func collectLines() async throws -> [String]
}
```

It doesn't take much imagination to think of other places where we may want to do similar operations, though. Therefore, we believe the best place to put these functions is instead as an extension on `AsyncSequence` itself, specified generically -- just like `Sequence`.

</details>

## Proposed solution

표준 라이브러리에 다음과 같은 프로토콜을 정의할 예정입니다:

```swift
public protocol AsyncSequence {
  associatedtype AsyncIterator: AsyncIteratorProtocol where AsyncIterator.Element == Element
  associatedtype Element
  __consuming func makeAsyncIterator() -> AsyncIterator
}

public protocol AsyncIteratorProtocol {
  associatedtype Element
  mutating func next() async throws -> Element?
}
```

Swift 컴파일러는 `AsyncSequence`를 준수하는 모든 타입에 대해 `for in` 반복문을 사용할 수 있도록 코드를 자동으로 생성합니다. 표준 라이브러리 또한 이 프로토콜을 확장하여 개발자에게 익숙한 제네릭 알고리즘들을 제공합니다. 아래 예제는 `next` 내부에서 실제로 `async` 함수를 호출하지 않지만, 기본적인 형태를 보여주는 예제입니다:

```swift
struct Counter : AsyncSequence {
  let howHigh: Int

  struct AsyncIterator : AsyncIteratorProtocol {
    let howHigh: Int
    var current = 1
    mutating func next() async -> Int? {
      // We could use the `Task` API to check for cancellation here and return early.
      guard current <= howHigh else {
        return nil
      }

      let result = current
      current += 1
      return result
    }
  }

  func makeAsyncIterator() -> AsyncIterator {
    return AsyncIterator(howHigh: howHigh)
  }
}
```

호출 지점에서는 `Counter`를 다음과 같이 사용할 수 있습니다:

```swift
for await i in Counter(howHigh: 3) {
  print(i)
}

/* 
Prints the following, and finishes the loop:
1
2
3
*/


for await i in Counter(howHigh: 3) {
  print(i)
  if i == 2 { break }
}
/*
Prints the following:
1
2
*/
```

<details>

<summary>원문 보기</summary>

The standard library will define the following protocols:

```swift
public protocol AsyncSequence {
  associatedtype AsyncIterator: AsyncIteratorProtocol where AsyncIterator.Element == Element
  associatedtype Element
  __consuming func makeAsyncIterator() -> AsyncIterator
}

public protocol AsyncIteratorProtocol {
  associatedtype Element
  mutating func next() async throws -> Element?
}
```

The compiler will generate code to allow use of a `for in` loop on any type which conforms with `AsyncSequence`. The standard library will also extend the protocol to provide familiar generic algorithms. Here is an example which does not actually call an `async` function within its `next`, but shows the basic shape:

```swift
struct Counter : AsyncSequence {
  let howHigh: Int

  struct AsyncIterator : AsyncIteratorProtocol {
    let howHigh: Int
    var current = 1
    mutating func next() async -> Int? {
      // We could use the `Task` API to check for cancellation here and return early.
      guard current <= howHigh else {
        return nil
      }

      let result = current
      current += 1
      return result
    }
  }

  func makeAsyncIterator() -> AsyncIterator {
    return AsyncIterator(howHigh: howHigh)
  }
}
```

At the call site, using `Counter` would look like this:

```swift
for await i in Counter(howHigh: 3) {
  print(i)
}

/* 
Prints the following, and finishes the loop:
1
2
3
*/


for await i in Counter(howHigh: 3) {
  print(i)
  if i == 2 { break }
}
/*
Prints the following:
1
2
*/
```

</details>

## Detailed design

앞서 살펴본 예제로 돌아가서:

```swift
for try await line in myFile.lines() {
  // Do something with each line
}
```

Swift 컴파일러는 다음 코드와 동일한 결과를 생성합니다:

```swift
var it = myFile.lines().makeAsyncIterator()
while let line = try await it.next() {
  // Do something with each line
}
```

일반적인 예외 처리 규칙이 모두 적용됩니다. 예를 들어, 이 반복문은 예외를 처리하기 위해 `do/catch`로 감싸져야 하거나, `throws` 함수 내부에 있어야 합니다. 또한, `await`에 대한 일반적인 규칙도 모두 적용됩니다. 예를 들어, 이 반복문은 `await` 호출이 허용되는 `async` 함수와 같은 컨텍스트에서 호출되어야 합니다.

<details>

<summary>원문 보기</summary>

Returning to our earlier example:

```swift
for try await line in myFile.lines() {
  // Do something with each line
}
```

The compiler will emit the equivalent of the following code:

```swift
var it = myFile.lines().makeAsyncIterator()
while let line = try await it.next() {
  // Do something with each line
}
```

All of the usual rules about error handling apply. For example, this iteration must be surrounded by `do/catch`, or be inside a `throws` function to handle the error. All of the usual rules about `await` also apply. For example, this iteration must be inside a context in which calling `await` is allowed like an `async` function.

</details>

### Cancellation

`AsyncIteratorProtocol` 타입은 Swift의 `Task` API에서 제공하는 취소 프리미티브(cancellation primitives)를 사용해야 하며, 이는 [구조화된 동시성](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)의 일부입니다. 해당 문서에서 설명한 바와 같이, 반복자(iterator)는 취소에 어떻게 응답할지 선택할 수 있습니다. 가장 기본적인 동작은 `CancellationError`를 던지거나, 반복자에서 `nil`을 반환하는 것입니다.

`AsyncIteratorProtocol` 타입이 취소 시 수행해야 할 정리 작업이 있는 경우, 다음 두 곳에서 이를 처리할 수 있습니다.

1. `Task` API를 사용하여 취소 여부를 확인한 이후

2. 클래스 타입인 경우에는 해당 타입의 `deinit` 내에서

<details>

<summary>원문 보기</summary>

`AsyncIteratorProtocol` types should use the cancellation primitives provided by Swift's `Task` API, part of [structured concurrency](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md). As described there, the iterator can choose how it responds to cancellation. The most common behaviors will be either throwing `CancellationError` or returning `nil` from the iterator. 

If an `AsyncIteratorProtocol` type has cleanup to do upon cancellation, it can do it in two places:

1. After checking for cancellation using the `Task` API.
2. In its `deinit` (if it is a class type).

</details>

### Rethrows

이 제안은 [여기](https://forums.swift.org/t/pitch-rethrowing-protocol-conformances/42373)에서 제안된 프로토콜에 특수한 `rethrows` 적합성(conformance)을 추가하는 방안을 활용합니다. 해당 제안에서 제시된 `rethrows` 관련 변경 사항이 적용되면, 자체적으로 예외를 던지지 않는 `AsyncSequence`를 순회할 때 `try`를 사용할 필요가 없어집니다.

`await`은 항상 필요합니다. `AsyncIteratorProtocol` 타입의 정의 자체가 항상 비동기적이기 때문입니다.

<details>

<summary>원문 보기</summary>

This proposal will take advantage of a separate proposal to add specialized rethrows conformance in a protocol, pitched [here]((https://forums.swift.org/t/pitch-rethrowing-protocol-conformances/42373)). With the changes proposed there for rethrows, it will not be required to use try when iterating an AsyncSequence which does not itself throw.

The await is always required because the definition of the protocol is that it is always asynchronous.

</details>

### End of Iteration

`AsyncIteratorProtocol` 타입이 `next()` 메서드에서 `nil`을 반환하거나 예외를 던진 이후, 모든 `next()` 호출에서 반드시 `nil`을 반환해야 합니다. 이는 `IteratorProtocol` 타입의 동작과 일치하며, 반복이 종료되었는지를 확인할 수 있는 유일한 수단이 반복자의 `next()` 메서드를 호출하는 것이기 때문에 매우 중요합니다.

<details>

<summary>원문 보기</summary>

After an `AsyncIteratorProtocol` types returns `nil` or throws an error from its `next()` method, all future calls to `next()` must return `nil`. This matches the behavior of `IteratorProtocol` types and is important, since calling an iterator's `next()` method is the only way to determine whether iteration has finished.

</details>

## AsyncSequence Functions

표준 `AsyncSequence` 프로토콜의 존재는 이를 준수하는 모든 타입에 대해 제네릭 알고리즘을 작성할 수 있게 합니다. 이러한 함수들은 두 가지 범주로 나눌 수 있습니다. 하나는 단일 값을 반환하는 함수이며, `async`로 표시됩니다. 다른 하나는 `AsyncSequence`를 반환하는 함수이며, `async`로 표시되지 않습니다.

단일 값을 반환하는 함수들은 특히 흥미롭습니다. 반복문을 하나의 `await` 문으로 대체함으로써 사용성을 크게 향상시키기 때문입니다. 이 범주에 속하는 함수들로는 `first`, `contains`, `min`, `max`, `reduce` 등이 있습니다. 한편, 새로운 `AsyncSequence`를 반환하는 함수들로는 `filter`, `map`과 `compactMap` 등이 있습니다.

<details>

<summary>원문 보기</summary>

The existence of a standard `AsyncSequence` protocol allows us to write generic algorithms for any type that conforms to it. There are two categories of functions: those that return a single value (and are thus marked as `async`), and those that return a new `AsyncSequence` (and are not marked as `async` themselves).

The functions that return a single value are especially interesting because they increase usability by changing a loop into a single `await` line. Functions in this category are `first`, `contains`, `min`, `max`, `reduce`, and more. Functions that return a new `AsyncSequence` include `filter`, `map`, and `compactMap`.

</details>

### AsyncSequence to single value

반복문을 단 하나의 호출로 줄여주는 알고리즘은 코드의 가독성을 향상시킬 수 있습니다. 이러한 알고리즘은 반복문을 생성하고 순회하는 데 필요한 보일러플레이트 코드를 줄여줍니다.

예를 들어, `contains` 함수를 한번 살펴봅시다:

```swift
extension AsyncSequence where Element : Equatable {
  public func contains(_ value: Element) async rethrows -> Bool
}
```

이 확장을 사용하면, 앞서 언급했던 "첫 번째로 긴 줄 찾기" 예제를 다음과 같이 단순하게 표현할 수 있습니다:

```swift
let first = try? await myFile.lines().first(where: { $0.count > 80 })
```

또는, 시퀀스를 비동기적으로 처리하고 나중에 사용해야 하는 경우에는:

```swift
async let first = myFile.lines().first(where: { $0.count > 80 })

// later

warnAboutLongLine(try? await first)
```

다음과 같은 함수들이 `AsyncSequence`에 추가될 예정입니다:

| Function | Note |
| - | - |
| `contains(_ value: Element) async rethrows -> Bool` | Requires `Equatable` element |
| `contains(where: (Element) async throws -> Bool) async rethrows -> Bool` | The `async` on the closure allows optional async behavior, but does not require it |
| `allSatisfy(_ predicate: (Element) async throws -> Bool) async rethrows -> Bool` | |
| `first(where: (Element) async throws -> Bool) async rethrows -> Element?` | |
| `min() async rethrows -> Element?` | Requires `Comparable` element |
| `min(by: (Element, Element) async throws -> Bool) async rethrows -> Element?` | |
| `max() async rethrows -> Element?` | Requires `Comparable` element |
| `max(by: (Element, Element) async throws -> Bool) async rethrows -> Element?` | |
| `reduce<T>(_ initialResult: T, _ nextPartialResult: (T, Element) async throws -> T) async rethrows -> T` | |
| `reduce<T>(into initialResult: T, _ updateAccumulatingResult: (inout T, Element) async throws -> ()) async rethrows -> T` | |

<details>

<summary>원문 보기</summary>

Algorithms that reduce a for loop into a single call can improve readability of code. They remove the boilerplate required to set up and iterate a loop.

For example, here is the `contains` function:

```swift
extension AsyncSequence where Element : Equatable {
  public func contains(_ value: Element) async rethrows -> Bool
}
```

With this extension, our "first long line" example from earlier becomes simply:

```swift
let first = try? await myFile.lines().first(where: { $0.count > 80 })
```

Or, if the sequence should be processed asynchronously and used later:

```swift
async let first = myFile.lines().first(where: { $0.count > 80 })

// later

warnAboutLongLine(try? await first)
```

The following functions will be added to `AsyncSequence`:

| Function | Note |
| - | - |
| `contains(_ value: Element) async rethrows -> Bool` | Requires `Equatable` element |
| `contains(where: (Element) async throws -> Bool) async rethrows -> Bool` | The `async` on the closure allows optional async behavior, but does not require it |
| `allSatisfy(_ predicate: (Element) async throws -> Bool) async rethrows -> Bool` | |
| `first(where: (Element) async throws -> Bool) async rethrows -> Element?` | |
| `min() async rethrows -> Element?` | Requires `Comparable` element |
| `min(by: (Element, Element) async throws -> Bool) async rethrows -> Element?` | |
| `max() async rethrows -> Element?` | Requires `Comparable` element |
| `max(by: (Element, Element) async throws -> Bool) async rethrows -> Element?` | |
| `reduce<T>(_ initialResult: T, _ nextPartialResult: (T, Element) async throws -> T) async rethrows -> T` | |
| `reduce<T>(into initialResult: T, _ updateAccumulatingResult: (inout T, Element) async throws -> ()) async rethrows -> T` | |

</details>

### AsyncSequence to AsyncSequence

`AsyncSequence`의 이러한 함수들은 반환값 자체가 또 다른 `AsyncSequence`가 될 수 있습니다. `AsyncSequence`는 비동기적으로 동작하기 때문에, 이러한 동작 방식은 표준 라이브러리에 있는 `Lazy` 타입들과 여러 면에서 유사합니다. 이러한 함수들을 호출하더라도 시퀀스의 다음 값을 즉시 `await`하지 않으며, 실제로 언제 반복을 시작할지는 호출자에게 달려있습니다. 호출자가 반복을 시작하면, 그 시점에 비로소 작업이 수행됩니다. 

예를 들어, `map` 함수를 살펴보겠습니다:

```swift
extension AsyncSequence {
  public func map<Transformed>(
    _ transform: @escaping (Element) async throws -> Transformed
  ) -> AsyncMapSequence<Self, Transformed>
}

public struct AsyncMapSequence<Upstream: AsyncSequence, Transformed>: AsyncSequence {
  public let upstream: Upstream
  public let transform: (Upstream.Element) async throws -> Transformed
  public struct Iterator : AsyncIterator { 
    public mutating func next() async rethrows -> Transformed?
  }
}
```

먼저, 각 함수마다 `AsyncSequence` 프로토콜을 준수하는 타입을 정의합니다. 이 타입의 이름은 `LazyDropWhileCollection`이나 `LazyMapSequence`처럼 표준 라이브러리의 기존 `Sequence` 타입을 참고하여 설계되었습니다. 그런 다음, `AsyncSequence`에 대한 확장(extension)을 통해 해당 타입의 인스턴스를 생성하는 함수를 추가하고, 이 함수는 `self`를 상위 시퀀스(`upstream`)로 사용하여 새로운 타입을 반환합니다.

| Function |
| - |
| `map<T>(_ transform: (Element) async throws -> T) -> AsyncMapSequence` |
| `compactMap<T>(_ transform: (Element) async throws -> T?) -> AsyncCompactMapSequence` |
| `flatMap<SegmentOfResult: AsyncSequence>(_ transform: (Element) async throws -> SegmentOfResult) async rethrows -> AsyncFlatMapSequence` |
| `drop(while: (Element) async throws -> Bool) async rethrows -> AsyncDropWhileSequence` |
| `dropFirst(_ n: Int) async rethrows -> AsyncDropFirstSequence` |
| `prefix(while: (Element) async throws -> Bool) async rethrows -> AsyncPrefixWhileSequence` |
| `prefix(_ n: Int) async rethrows -> AsyncPrefixSequence` |
| `filter(_ predicate: (Element) async throws -> Bool) async rethrows -> AsyncFilterSequence` |

<details>

<summary>원문 보기</summary>

These functions on `AsyncSequence` return a result which is itself an `AsyncSequence`. Due to the asynchronous nature of `AsyncSequence`, the behavior is similar in many ways to the existing `Lazy` types in the standard library. Calling these functions does not eagerly `await` the next value in the sequence, leaving it up to the caller to decide when to start that work by simply starting iteration when they are ready.

As an example, let's look at `map`:


```swift
extension AsyncSequence {
  public func map<Transformed>(
    _ transform: @escaping (Element) async throws -> Transformed
  ) -> AsyncMapSequence<Self, Transformed>
}

public struct AsyncMapSequence<Upstream: AsyncSequence, Transformed>: AsyncSequence {
  public let upstream: Upstream
  public let transform: (Upstream.Element) async throws -> Transformed
  public struct Iterator : AsyncIterator { 
    public mutating func next() async rethrows -> Transformed?
  }
}
```

For each of these functions, we first define a type which conforms with the `AsyncSequence` protocol. The name is modeled after existing standard library `Sequence` types like `LazyDropWhileCollection` and `LazyMapSequence`. Then, we add a function in an extension on `AsyncSequence` which creates the new type (using `self` as the `upstream`) and returns it.

| Function |
| - |
| `map<T>(_ transform: (Element) async throws -> T) -> AsyncMapSequence` |
| `compactMap<T>(_ transform: (Element) async throws -> T?) -> AsyncCompactMapSequence` |
| `flatMap<SegmentOfResult: AsyncSequence>(_ transform: (Element) async throws -> SegmentOfResult) async rethrows -> AsyncFlatMapSequence` |
| `drop(while: (Element) async throws -> Bool) async rethrows -> AsyncDropWhileSequence` |
| `dropFirst(_ n: Int) async rethrows -> AsyncDropFirstSequence` |
| `prefix(while: (Element) async throws -> Bool) async rethrows -> AsyncPrefixWhileSequence` |
| `prefix(_ n: Int) async rethrows -> AsyncPrefixSequence` |
| `filter(_ predicate: (Element) async throws -> Bool) async rethrows -> AsyncFilterSequence` |

</details>

## Future Proposals

The following topics are things we consider important and worth discussion in future proposals:

### Additional `AsyncSequence` functions

We've aimed for parity with the most relevant `Sequence` functions. There may be others that are worth adding in a future proposal.

API which uses a time argument must be coordinated with the discussion about `Executor` as part of the [structured concurrency proposal](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md).

We would like a `first` property, but properties cannot currently be `async` or `throws`. Discussions are ongoing about adding a capability to the language to allow effects on properties. If those features become part of Swift then we should add a `first` property to `AsyncSequence`.

### AsyncSequence Builder

In the standard library we have not only the `Sequence` and `Collection` protocols, but concrete types which adopt them (for example, `Array`). We will need a similar API for `AsyncSequence` that makes it easy to construct a concrete instance when needed, without declaring a new type and adding protocol conformance.

## Source compatibility

This new functionality will be source compatible with existing Swift.

## Effect on ABI stability

This change is additive to the ABI.

## Effect on API resilience

This change is additive to API.

## Alternatives considered

### Explicit Cancellation

An earlier version of this proposal included an explicit `cancel` function. We removed it for the following reasons:

1. Reducing the requirements of implementing `AsyncIteratorProtocol` makes it simpler to use and easier to understand. The rules about when `cancel` would be called, while straightforward, would nevertheless be one additional thing for Swift developers to learn.
2. The structured concurrency proposal already includes a definition of cancellation that works well for `AsyncSequence`. We should consider the overall behavior of cancellation for asynchronous code as one concept.

### Asynchronous Cancellation

If we used explicit cancellation, the `cancel()` function on the iterator could be marked as `async`. However, this means that the implicit cancellation done when leaving a `for/in` loop would require an implicit `await` -- something we think is probably too much to hide from the developer. Most cancellation behavior is going to be as simple as setting a flag to check later, so we leave it as a synchronous function and encourage adopters to make cancellation fast and non-blocking.

### Opaque Types

Each `AsyncSequence`-to-`AsyncSequence` algorithm will define its own concrete type. We could attempt to hide these details behind a general purpose type eraser. We believe leaving the types exposed gives us (and the compiler) more optimization opportunities. A great future enhancement would be for the language to support `some AsyncSequence where Element=...`-style syntax, allowing hiding of concrete `AsyncSequence` types at API boundaries.

### Reusing Sequence

If the language supported a `reasync` concept, then it seems plausible that the `AsyncSequence` and `Sequence` APIs could be merged. However, we believe it is still valuable to consider these as two different types. The added complexity of a time dimension in asynchronous code means that some functions need more configuration options or more complex implementations. Some algorithms that are useful on asynchronous sequences are not meaningful on synchronous ones. We prefer not to complicate the API surface of the synchronous collection types in these cases.

### Naming

The names of the concrete `AsyncSequence` types is designed to mirror existing standard library API like `LazyMapSequence`. Another option is to introduce a new pattern with an empty enum or other namespacing mechanism.

We considered `AsyncGenerator` but would prefer to leave the `Generator` name for future language enhancements. `Stream` is a type in Foundation, so we did not reuse it here to avoid confusion.

### `await in`

We considered a shorter syntax of `await...in`. However, since the behavior here is fundamentally a loop, we feel it is important to use the existing `for` keyword as a strong signal of intent to readers of the code. Although there are a lot of keywords, each one has purpose and meaning to readers of the code.

### Add APIs to iterator instead of sequence

We discussed applying the fundamental API (`map`, `reduce`, etc.) to `AsyncIteratorProtocol` instead of `AsyncSequence`. There has been a long-standing (albeit deliberate) ambiguity in the `Sequence` API -- is it supposed to be single-pass or multi-pass? This new kind of iterator & sequence could provide an opportunity to define this more concretely.

While it is tempting to use this new API to right past wrongs, we maintain that the high level goal of consistency with existing Swift concepts is more important. 

For example, `for...in` cannot be used on an `IteratorProtocol` -- only a `Sequence`. If we chose to make `AsyncIteratorProtocol` use `for...in` as described here, that leaves us with the choice of either introducing an inconsistency between `AsyncIteratorProtocol` and `IteratorProtocol` or giving up on the familiar `for...in` syntax. Even if we decided to add `for...in` to `IteratorProtocol`, it would still be inconsistent because we would be required to leave `for...in` syntax on the existing `Sequence`.

Another point in favor of consistency is that implementing an `AsyncSequence` should feel familiar to anyone who knows how to implement a `Sequence`.

We are hoping for widespread adoption of the protocol in API which would normally have instead used a `Notification`, informational delegate pattern, or multi-callback closure argument. In many of these cases we feel like the API should return the 'factory type' (an `AsyncSequence`) so that it can be iterated again. It will still be up to the caller to be aware of any underlying cost of performing that operation, as with iteration of any `Sequence` today.
