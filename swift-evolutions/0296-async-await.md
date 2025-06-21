# Async/await

* 제안: [SE-0296](0296-async-await.md)
* 저자: [John McCall](https://github.com/rjmccall), [Doug Gregor](https://github.com/DougGregor)
* 리뷰 매니저: [Ben Cohen](https://github.com/airspeedswift)
* 상태: **구현됨 (Swift 5.5)**
* 결정 기록: [Rationale](https://forums.swift.org/t/accepted-with-modification-se-0296-async-await/43318), [Amendment to allow overloading on `async`](https://forums.swift.org/t/accepted-amendment-to-se-0296-allow-overloads-that-differ-only-in-async/50117)

## Table of Contents

   * [Async/await](#asyncawait)
      * [Introduction](#introduction)
      * [Motivation: Completion handlers are suboptimal](#motivation-completion-handlers-are-suboptimal)
      * [Proposed solution: async/await](#proposed-solution-asyncawait)
         * [Suspension points](#suspension-points)
      * [Detailed design](#detailed-design)
         * [Asynchronous functions](#asynchronous-functions)
         * [Asynchronous function types](#asynchronous-function-types)
         * [Await expressions](#await-expressions)
         * [Closures](#closures)
         * [Overloading and overload resolution](#overloading-and-overload-resolution)
         * [Autoclosures](#autoclosures)
         * [Protocol conformance](#protocol-conformance)
      * [Source compatibility](#source-compatibility)
      * [Effect on ABI stability](#effect-on-abi-stability)
      * [Effect on API resilience](#effect-on-api-resilience)
      * [Future Directions](#future-directions)
         * [reasync](#reasync)
      * [Alternatives Considered](#alternatives-considered)
         * [Make await imply try](#make-await-imply-try)
         * [Launching async tasks](#launching-async-tasks)
         * [Await as syntactic sugar](#await-as-syntactic-sugar)
      * [Revision history](#revision-history)
      * [Related proposals](#related-proposals)
      * [Acknowledgments](#acknowledgments)

## Introduction

현대의 Swift 개발에서는 클로저와 완료 핸들러(completion handler)를 활용한 비동기 프로그래밍이 많이 사용되지만, 이러한 API는 사용하기 어렵다는 단점이 있습니다. 특히, 많은 비동기 작업을 사용할 때, 오류 처리가 필요할 때, 또는 비동기 호출 간의 제어 흐름이 복잡해질 때 문제가 더욱 심각해집니다. 이 제안서는 이러한 비동기 프로그래밍을 보다 자연스럽고 오류가 더 적게 발생하는 방식으로 개선하기 위한 언어 기능(language extension)을 설명합니다.

이 설계는 [코루틴 모델](https://en.wikipedia.org/wiki/Coroutine)에 기반합니다. 함수는 `async` 키워드를 사용하여 비동기 함수로 선언할 수 있으며, 이를 통해 개발자는 일반적인 제어 흐름 구조를 사용해 비동기 작업을 포함하는 복잡한 로직을 구성할 수 있습니다. 컴파일러는 이러한 비동기 함수를 적절한 클로저와 상태 머신(state machine)의 집합으로 변환하는 역할을 담당합니다.

이 제안서는 비동기 함수의 의미를 정의합니다. 그러나, 여기서는 동시성(concurrency)은 다루지 않습니다. 동시성은 구조화된 동시성(structured concurrency)을 도입하는 별도의 제안서에서 다루며, 해당 제안서에는 비동기 함수를 동시에 실행되는 작업(task)과 연결하고, 작업을 생성하고, 조회하며, 취소할 수 있는 API를 제공합니다.

Swift-Evolution 스레드: [Pitch #1](https://forums.swift.org/t/concurrency-asynchronous-functions/41619), [Pitch #2](https://forums.swift.org/t/pitch-2-async-await/42420)

<details>

<summary>원문 보기</summary>

Modern Swift development involves a lot of asynchronous (or "async") programming using closures and completion handlers, but these APIs are hard to use.  This gets particularly problematic when many asynchronous operations are used, error handling is required, or control flow between asynchronous calls gets complicated.  This proposal describes a language extension to make this a lot more natural and less error prone.

This design introduces a [coroutine model](https://en.wikipedia.org/wiki/Coroutine) to Swift. Functions can opt into being `async`, allowing the programmer to compose complex logic involving asynchronous operations using the normal control-flow mechanisms. The compiler is responsible for translating an asynchronous function into an appropriate set of closures and state machines.

This proposal defines the semantics of asynchronous functions. However, it does not provide concurrency: that is covered by a separate proposal to introduce structured concurrency, which associates asynchronous functions with concurrently-executing tasks and provides APIs for creating, querying, and cancelling tasks.

Swift-evolution thread: [Pitch #1](https://forums.swift.org/t/concurrency-asynchronous-functions/41619), [Pitch #2](https://forums.swift.org/t/pitch-2-async-await/42420)

</details>

## Motivation: Completion handlers are suboptimal

완료 핸들러를 사용하는 비동기 프로그래밍은 아래에서 살펴볼 것처럼 많은 문제를 가지고 있습니다. 우리는 이러한 문제를 해결하기 위해 언어에 비동기 함수(async function)를 도입할 것을 제안합니다. 비동기 함수는 비동기 코드를 마치 직선적인(동기적인) 코드로 작성할 수 있게 해줍니다. 또한, 비동기 함수는 코드가 어떤 순서로 실행되고, 어디에서 멈췄다가 다시 시작되는지를 컴파일러나 런타임이 쉽게 이해할 수 있도록 해줍니다. 이러한 시스템 덕분에 콜백을 더 효율적으로 실행할 수 있습니다.

<details>

<summary>원문 보기</summary>

Async programming with explicit callbacks (also called completion handlers) has many problems, which we’ll explore below.  We propose to address these problems by introducing async functions into the language.  Async functions allow asynchronous code to be written as straight-line code.  They also allow the implementation to directly reason about the execution pattern of the code, allowing callbacks to run far more efficiently.

</details>

#### Problem 1: Pyramid of doom

일련의 간단한 비동기 작업들은 종종 깊이 중첩된 클로저를 요구하게 됩니다. 아래는 이를 보여주는 예시입니다.

```swift
func processImageData1(completionBlock: (_ result: Image) -> Void) {
    loadWebResource("dataprofile.txt") { dataResource in
        loadWebResource("imagedata.dat") { imageResource in
            decodeImage(dataResource, imageResource) { imageTmp in
                dewarpAndCleanupImage(imageTmp) { imageResult in
                    completionBlock(imageResult)
                }
            }
        }
    }
}

processImageData1 { image in
    display(image)
}
```

이러한 '파멸의 피라미드(pyramid of doom)'는 코드를 읽기 어렵게 만들고, 현재 코드가 어디에서 실행되고 있는지를 파악하기 어렵게 만듭니다. 또한, 클로저를 여러 겹으로 쌓아 사용하는 방식은 뒤이어 논의할 많은 2차적인 문제를 초래합니다.

<details>

<summary>원문 보기</summary>

A sequence of simple asynchronous operations often requires deeply-nested closures. Here is a made-up example showing this:

```swift
func processImageData1(completionBlock: (_ result: Image) -> Void) {
    loadWebResource("dataprofile.txt") { dataResource in
        loadWebResource("imagedata.dat") { imageResource in
            decodeImage(dataResource, imageResource) { imageTmp in
                dewarpAndCleanupImage(imageTmp) { imageResult in
                    completionBlock(imageResult)
                }
            }
        }
    }
}

processImageData1 { image in
    display(image)
}
```

This "pyramid of doom" makes it difficult to read and keep track of where the code is running. In addition, having to use a stack of closures leads to many second order effects that we will discuss next.


</details>

#### Problem 2: Error handling

콜백은 오류 처리를 어렵고 장황하게 만듭니다. Swift 2에서는 동기 코드를 위한 오류 처리 모델이 도입되었지만, 콜백 기반 인터페이스는 이 모델로부터 아무런 이점을 누리지 못합니다.

```swift
// (2a) Using a `guard` statement for each callback:
func processImageData2a(completionBlock: (_ result: Image?, _ error: Error?) -> Void) {
    loadWebResource("dataprofile.txt") { dataResource, error in
        guard let dataResource = dataResource else {
            completionBlock(nil, error)
            return
        }
        loadWebResource("imagedata.dat") { imageResource, error in
            guard let imageResource = imageResource else {
                completionBlock(nil, error)
                return
            }
            decodeImage(dataResource, imageResource) { imageTmp, error in
                guard let imageTmp = imageTmp else {
                    completionBlock(nil, error)
                    return
                }
                dewarpAndCleanupImage(imageTmp) { imageResult, error in
                    guard let imageResult = imageResult else {
                        completionBlock(nil, error)
                        return
                    }
                    completionBlock(imageResult)
                }
            }
        }
    }
}

processImageData2a { image, error in
    guard let image = image else {
        display("No image today", error)
        return
    }
    display(image)
}
```

표준 라이브러리에 [`Result`](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0235-add-result.md)가 추가되면서 Swift API의 오류 처리 방식이 개선되었습니다. 특히, 비동기 API는 `Result` 타입이 도입된 [주요 동기](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0235-add-result.md#asynchronous-apis) 중 하나였습니다.

```swift
// (2b) Using a `do-catch` statement for each callback:
func processImageData2b(completionBlock: (Result<Image, Error>) -> Void) {
    loadWebResource("dataprofile.txt") { dataResourceResult in
        do {
            let dataResource = try dataResourceResult.get()
            loadWebResource("imagedata.dat") { imageResourceResult in
                do {
                    let imageResource = try imageResourceResult.get()
                    decodeImage(dataResource, imageResource) { imageTmpResult in
                        do {
                            let imageTmp = try imageTmpResult.get()
                            dewarpAndCleanupImage(imageTmp) { imageResult in
                                completionBlock(imageResult)
                            }
                        } catch {
                            completionBlock(.failure(error))
                        }
                    }
                } catch {
                    completionBlock(.failure(error))
                }
            }
        } catch {
            completionBlock(.failure(error))
        }
    }
}

processImageData2b { result in
    do {
        let image = try result.get()
        display(image)
    } catch {
        display("No image today", error)
    }
}
```

```swift
// (2c) Using a `switch` statement for each callback:
func processImageData2c(completionBlock: (Result<Image, Error>) -> Void) {
    loadWebResource("dataprofile.txt") { dataResourceResult in
        switch dataResourceResult {
        case .success(let dataResource):
            loadWebResource("imagedata.dat") { imageResourceResult in
                switch imageResourceResult {
                case .success(let imageResource):
                    decodeImage(dataResource, imageResource) { imageTmpResult in
                        switch imageTmpResult {
                        case .success(let imageTmp):
                            dewarpAndCleanupImage(imageTmp) { imageResult in
                                completionBlock(imageResult)
                            }
                        case .failure(let error):
                            completionBlock(.failure(error))
                        }
                    }
                case .failure(let error):
                    completionBlock(.failure(error))
                }
            }
        case .failure(let error):
            completionBlock(.failure(error))
        }
    }
}

processImageData2c { result in
    switch result {
    case .success(let image):
        display(image)
    case .failure(let error):
        display("No image today", error)
    }
}
```

`Result`를 사용하면 오류 처리가 더 쉬워지지만, 클로저 중첩 문제는 여전히 남아 있습니다.

<details>

<summary>원문 보기</summary>

Callbacks make error handling difficult and very verbose. Swift 2 introduced an error handling model for synchronous code, but callback-based interfaces do not derive any benefit from it:

```swift
// (2a) Using a `guard` statement for each callback:
func processImageData2a(completionBlock: (_ result: Image?, _ error: Error?) -> Void) {
    loadWebResource("dataprofile.txt") { dataResource, error in
        guard let dataResource = dataResource else {
            completionBlock(nil, error)
            return
        }
        loadWebResource("imagedata.dat") { imageResource, error in
            guard let imageResource = imageResource else {
                completionBlock(nil, error)
                return
            }
            decodeImage(dataResource, imageResource) { imageTmp, error in
                guard let imageTmp = imageTmp else {
                    completionBlock(nil, error)
                    return
                }
                dewarpAndCleanupImage(imageTmp) { imageResult, error in
                    guard let imageResult = imageResult else {
                        completionBlock(nil, error)
                        return
                    }
                    completionBlock(imageResult)
                }
            }
        }
    }
}

processImageData2a { image, error in
    guard let image = image else {
        display("No image today", error)
        return
    }
    display(image)
}
```

The addition of [`Result`](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0235-add-result.md) to the standard library improved on error handling for Swift APIs. Asynchronous APIs were one of the [main motivators](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0235-add-result.md#asynchronous-apis) for `Result`: 

```swift
// (2b) Using a `do-catch` statement for each callback:
func processImageData2b(completionBlock: (Result<Image, Error>) -> Void) {
    loadWebResource("dataprofile.txt") { dataResourceResult in
        do {
            let dataResource = try dataResourceResult.get()
            loadWebResource("imagedata.dat") { imageResourceResult in
                do {
                    let imageResource = try imageResourceResult.get()
                    decodeImage(dataResource, imageResource) { imageTmpResult in
                        do {
                            let imageTmp = try imageTmpResult.get()
                            dewarpAndCleanupImage(imageTmp) { imageResult in
                                completionBlock(imageResult)
                            }
                        } catch {
                            completionBlock(.failure(error))
                        }
                    }
                } catch {
                    completionBlock(.failure(error))
                }
            }
        } catch {
            completionBlock(.failure(error))
        }
    }
}

processImageData2b { result in
    do {
        let image = try result.get()
        display(image)
    } catch {
        display("No image today", error)
    }
}
```

```swift
// (2c) Using a `switch` statement for each callback:
func processImageData2c(completionBlock: (Result<Image, Error>) -> Void) {
    loadWebResource("dataprofile.txt") { dataResourceResult in
        switch dataResourceResult {
        case .success(let dataResource):
            loadWebResource("imagedata.dat") { imageResourceResult in
                switch imageResourceResult {
                case .success(let imageResource):
                    decodeImage(dataResource, imageResource) { imageTmpResult in
                        switch imageTmpResult {
                        case .success(let imageTmp):
                            dewarpAndCleanupImage(imageTmp) { imageResult in
                                completionBlock(imageResult)
                            }
                        case .failure(let error):
                            completionBlock(.failure(error))
                        }
                    }
                case .failure(let error):
                    completionBlock(.failure(error))
                }
            }
        case .failure(let error):
            completionBlock(.failure(error))
        }
    }
}

processImageData2c { result in
    switch result {
    case .success(let image):
        display(image)
    case .failure(let error):
        display("No image today", error)
    }
}
```

It's easier to handle errors when using `Result`, but the closure-nesting problem remains.


</details>


#### Problem 3: Conditional execution is hard and error-prone

조건부로 비동기 함수를 실행하는 것은 매우 번거로운 일입니다. 예를 들어, 이미지를 가져온 후 "색상 변환(swizzle)" 작업을 해야 한다고 가정해봅시다. 그러나, 경우에 따라 색상 변환 작업 전에 이미지를 비동기적으로 디코딩해야 할 수도 있습니다. 아마도 이 함수를 구현하는 가장 좋은 방법은 색상 변환 코드를 일명 "컨티뉴에이션(continuation)" 헬퍼 클로저에 작성한 다음, 해당 클로저를 조건에 따라 완료 핸들러 안에서 호출하는 것일 수 있습니다.

```swift
func processImageData3(recipient: Person, completionBlock: (_ result: Image) -> Void) {
    let swizzle: (_ contents: Image) -> Void = {
      // ... continuation closure that calls completionBlock eventually
    }
    if recipient.hasProfilePicture {
        swizzle(recipient.profilePicture)
    } else {
        decodeImage { image in
            swizzle(image)
        }
    }
}
```

이 패턴은 함수의 자연스러운 탑-다운(top-down) 구조를 뒤집어버립니다. 즉, 함수의 후반부에 실행되어야 할 코드가 전반부에 실행될 코드보다 반드시 **먼저** 등장하게 됩니다. 함수의 전체 구조를 재구성해야 할 뿐만 아니라, 이제는 컨티뉴에이션 클로저가 완료 핸들러 안에서 사용되기 때문에 캡처되는 값들에 대해서도 신중히 생각해야 합니다. 조건부로 실행되는 비동기 함수가 많아질수록 이 문제는 점점 심각해지며, 결국 뒤집힌 형태의 "파멸의 피라미드"가 만들어지게 됩니다.

<details>

<summary>원문 보기</summary>

Conditionally executing an asynchronous function is a huge pain. For example, suppose we need to "swizzle" an image after obtaining it. But, we sometimes have to make an asynchronous call to decode the image before we can swizzle. Perhaps the best approach to structuring this function is to write the swizzling code in a helper "continuation" closure that is conditionally captured in a completion handler, like this:

```swift
func processImageData3(recipient: Person, completionBlock: (_ result: Image) -> Void) {
    let swizzle: (_ contents: Image) -> Void = {
      // ... continuation closure that calls completionBlock eventually
    }
    if recipient.hasProfilePicture {
        swizzle(recipient.profilePicture)
    } else {
        decodeImage { image in
            swizzle(image)
        }
    }
}
```

This pattern inverts the natural top-down organization of the function: the code that will execute in the second half of the function must appear *before* the part that executes in the first half. In addition to restructuring the entire function, we must now think carefully about captures in the continuation closure, because the closure is used in a completion handler. The problem worsens as the number of conditionally-executed async functions grows, yielding what is essentially an inverted "pyramid of doom."

</details>

#### Problem 4: Many mistakes are easy to make

비동기 작업 도중 중간에 빠져나오는 것은 올바른 완료 핸들러 블럭을 호출하지 않고 단순히 반환(return)함으로써 쉽게 발생할 수 있습니다. 그런데 이 호출을 잊어버리면, 문제의 원인을 찾고 디버깅하는 것이 매우 어려워집니다.

It's quite easy to bail-out of the asynchronous operation early by simply returning without calling the correct completion-handler block. When forgotten, the issue is very hard to debug:

```swift
func processImageData4a(completionBlock: (_ result: Image?, _ error: Error?) -> Void) {
    loadWebResource("dataprofile.txt") { dataResource, error in
        guard let dataResource = dataResource else {
            return // <- forgot to call the block
        }
        loadWebResource("imagedata.dat") { imageResource, error in
            guard let imageResource = imageResource else {
                return // <- forgot to call the block
            }
            ...
        }
    }
}
```

완료 핸들러 블록을 적절히 호출하더라도, 그 다음 반환(return)하는 것을 잊어버릴 수 있습니다.

When you do remember to call the block, you can still forget to return after that:

```swift
func processImageData4b(recipient:Person, completionBlock: (_ result: Image?, _ error: Error?) -> Void) {
    if recipient.hasProfilePicture {
        if let image = recipient.profilePicture {
            completionBlock(image) // <- forgot to return after calling the block
        }
    }
    ...
}
```

다행히도 `guard` 구문은 반환을 깜박하는 실수를 어느 정도 방지해주지만, 항상 적절한 상황에서 사용할 수 있는 것은 아닙니다.

Thankfully the `guard` syntax protects against forgetting to return to some degree, but it's not always relevant.

<details>

<summary>원문 보기</summary>

It's quite easy to bail-out of the asynchronous operation early by simply returning without calling the correct completion-handler block. When forgotten, the issue is very hard to debug:

```swift
func processImageData4a(completionBlock: (_ result: Image?, _ error: Error?) -> Void) {
    loadWebResource("dataprofile.txt") { dataResource, error in
        guard let dataResource = dataResource else {
            return // <- forgot to call the block
        }
        loadWebResource("imagedata.dat") { imageResource, error in
            guard let imageResource = imageResource else {
                return // <- forgot to call the block
            }
            ...
        }
    }
}
```

When you do remember to call the block, you can still forget to return after that:

```swift
func processImageData4b(recipient:Person, completionBlock: (_ result: Image?, _ error: Error?) -> Void) {
    if recipient.hasProfilePicture {
        if let image = recipient.profilePicture {
            completionBlock(image) // <- forgot to return after calling the block
        }
    }
    ...
}
```

Thankfully the `guard` syntax protects against forgetting to return to some degree, but it's not always relevant.

</details>

#### Problem 5: Because completion handlers are awkward, too many APIs are defined synchronously

이는 수치화하기 어렵지만, 저자는 완료 핸들러를 이용한 비동기 API의 정의와 사용이 어색하고 불편하다는 점 때문에, 실제로는 블로킹(blocking)이 발생할 수 있음에도 불구하고 외형상 동기적인 함수처럼 보이도록 API를 설계해버린 경우가 많다고 보고 있습니다. 이러한 API는 UI 애플리케이션에서 성능 문제나 반응성 저하를 일으킬 수 있습니다. 또한, 대규모 처리를 위해 비동기 처리가 핵심인 서버 환경에서는 이러한 API를 아예 사용할 수 없는 경우도 생깁니다.

<details>

<summary>원문 보기</summary>

This is hard to quantify, but the authors believe that the awkwardness of defining and using asynchronous APIs (using completion handlers) has led to many APIs being defined with apparently synchronous behavior, even when they can block.  This can lead to problematic performance and responsiveness problems in UI applications, e.g. a spinning cursor.  It can also lead to the definition of APIs that cannot be used when asynchrony is critical to achieve scale, e.g. on the server.

</details>

## Proposed solution: async/await

async/await이라 알려진 비동기 함수는 비동기 코드를 마치 직선적인 동기 코드처럼 작성할 수 있게 해줍니다. 이는 개발자가 동기 코드에서 사용할 수 있는 동일한 언어 구조를 온전히 활용할 수 있도록 하여, 위에서 설명한 많은 문제를 해결합니다. 또한, async/await의 사용은 코드의 의미론적 구조를 자연스럽게 보존하며, 언어 전체에 걸쳐 영향을 미치는 다음과 같은 최소 3가지 중요한 개선을 가능하게 합니다: (1) 비동기 코드의 더 나은 성능 (2) 디버깅, 프로파일링, 코드 탐색 시 더 일관된 경험을 제공하는 더 나은 도구 (3) 작업 우선순위나 취소와 같은 향후 동시성 기능에 대한 기반. 아래 예제는 앞서 소개한 예제를 async/await 방식으로 다시 작성한 것으로, 비동기 코드를 얼마나 획기적으로 단순해질 수 있는지를 보여줍니다.  

```swift
func loadWebResource(_ path: String) async throws -> Resource
func decodeImage(_ r1: Resource, _ r2: Resource) async throws -> Image
func dewarpAndCleanupImage(_ i : Image) async throws -> Image

func processImageData() async throws -> Image {
  let dataResource  = try await loadWebResource("dataprofile.txt")
  let imageResource = try await loadWebResource("imagedata.dat")
  let imageTmp      = try await decodeImage(dataResource, imageResource)
  let imageResult   = try await dewarpAndCleanupImage(imageTmp)
  return imageResult
}
```

많은 async/await에 대한 설명에서는 이를 구현하는 일반적인 방식 중 하나로 컴파일러가 함수 코드를 여러 조각으로 나눠 처리하는 방식을 중심으로 설명합니다. 이러한 방식은 시스템이 어떻게 동작하는지를 저수준 관점에서 이해하는 데는 매우 중요하지만, 고수준 관점에서는 굳이 신경 쓸 필요는 없습니다. 대신, 비동기 함수는 '스레드를 포기(give up)할 수 있는 특별한 능력을 가진 일반 함수'라고 생각하시면 됩니다. 비동기 함수가 이 능력을 직접 사용하는 일은 드물고, 대신 다른 함수를 호출하는 과정에서 자연스럽게 스레드를 잠시 포기하고, 어떤 작업이 완료될 때까지 잠시 대기하게 됩니다. 그리고 그 작업이 완료되면, 비동기 함수는 멈췄던 지점부터 실행을 다시 재개합니다.

비동기 함수는 동기 함수와 매우 유사한 방식으로 동작합니다. 동기 함수는 다른 함수를 호출할 수 있고, 호출한 함수가 작업을 완료할 때까지 기다립니다. 호출된 함수가 완료되면, 제어 흐름은 다시 원래 함수로 돌아와 멈췄던 지점부터 계속 실행됩니다. 비동기 함수도 마찬가지입니다. 비동기 함수도 다른 함수를 호출할 수 있고, (일반적으로) 호출한 함수가 작업을 완료할 때까지 기다립니다. 호출이 완료되면, 제어 흐름은 다시 원래 비동기 함수로 돌아와 이전에 멈췄던 지점부터 실행을 이어갑니다. 유일한 차이점은 동기 함수는 실행되는 동안 스레드와 호출 스택의 일부를 온전히 활용하지만, **비동기 함수는 스레드와 호출 스택을 완전히 포기하고, 별도의 저장 공간을 사용**할 수 있다는 점입니다. 비동기 함수가 가지는 이러한 추가적인 기능은 많은 비용을 발생시키지만, 언어를 중심으로 전체적으로 잘 설계하면 그 비용을 상당히 줄일 수 있습니다.

비동기 함수는 자신이 점유한 스레드를 포기할 수 있어야 하지만, 동기 함수는 스레드를 포기할 수 없기 때문에, 일반적으로 동기 함수는 비동기 함수를 호출할 수 없습니다. 비동기 함수는 실행 중 일시 중단되면 자신이 점유하고 있던 스레드를 포기합니다. 그러나 동기 호출는 이르 일반적인 반환으로 잘못 인식하여, 반환값 없이 그 지점부터 실행을 계속하려고 시도하게 됩니다. 일반적으로 이 문제를 해결하는 방법은 비동기 함수가 다시 재개되어 완료될 때까지 전체 스레드를 차단하는 것이지만, 이는 비동기 함수의 목적을 완전히 무의미하게 만들 뿐만 아니라, 시스템 전반에 걸쳐 심각한 부작용을 초래할 수 있습니다.

반대로, 비동기 함수는 동기 함수든 비동기 함수든 모두 호출할 수 있습니다. 물론 동기 함수를 호출하는 동안에는 스레드를 포기할 수 없습니다. 사실, 비동기 함수는 임의로 스레드를 포기하지 않으며, 오직 일시 중단 지점(suspension point)에 도달했을 때만 스레드를 포기합니다. 일시 중단 지점은 함수 내부에서 직접 발생할 수 있고, 함수가 호출한 다른 비동기 함수 내부에서 발생할 수도 있습니다. 하지만 어느 경우든, 그 함수와 그것을 호출한 모든 비동기 함수들이 동시에 스레드를 포기하게 됩니다. (다만 실제로는 비동기 함수들이 호출 중에는 스레드에 의존하지 않도록 컴파일되기 때문에, 가장 안쪽의 함수만 추가적인 작업을 수행하게 됩니다.)

비동기 함수로 제어가 돌아오면, 중단되었던 지점에서 정확히 이어서 실행을 계속합니다. 하지만, 이는 반드시 일시 중단된 시점과 동일한 스레드에서 실행된다는 것을 의미하지는 않습니다. 언어 차원에서 중단 이후 동일한 스레드에서 실행될 것이라는 보장은 없기 때문입니다. 이 설계에서 스레드는 단지 동시성을 구현하기 위한 수단일 뿐이며, 개발자가 직접 다뤄야 하는 동시성의 핵심 개념이나 인터페이스는 아닙니다. 하지만, 많은 비동기 함수들은 단순히 비동기일 뿐만 아니라, 특정 액터(actor)와 연관되어 있습니다. 이러한 함수들은 항상 해당 액터의 일부로 실행되어야 합니다.

Swift는 이러한 함수들이 실행을 마치기 위해 자신이 속한 액터로 다시 돌아오도록 보장합니다. 따라서, 상태 격리(state isolation)를 위해 스레드를 직접 사용하는 라이브러리라면, Swift가 제공하는 기본적인 언어 수준의 보장이 제대로 직동하도록 하기 위해 그 스레드를 액터로 모델링하는 것이 바람직합니다. 

<details>

<summary>원문 보기</summary>

Asynchronous functions—often known as async/await—allow asynchronous code to be written as if it were straight-line, synchronous code.  This immediately addresses many of the problems described above by allowing programmers to make full use of the same language constructs that are available to synchronous code.  The use of async/await also naturally preserves the semantic structure of the code, providing information necessary for at least three cross-cutting improvements to the language: (1) better performance for asynchronous code; (2) better tooling to provide a more consistent experience while debugging, profiling, and exploring code; and (3) a foundation for future concurrency features like task priority and cancellation.  The example from the prior section demonstrates how async/await drastically simplifies asynchronous code:

```swift
func loadWebResource(_ path: String) async throws -> Resource
func decodeImage(_ r1: Resource, _ r2: Resource) async throws -> Image
func dewarpAndCleanupImage(_ i : Image) async throws -> Image

func processImageData() async throws -> Image {
  let dataResource  = try await loadWebResource("dataprofile.txt")
  let imageResource = try await loadWebResource("imagedata.dat")
  let imageTmp      = try await decodeImage(dataResource, imageResource)
  let imageResult   = try await dewarpAndCleanupImage(imageTmp)
  return imageResult
}
```

Many descriptions of async/await discuss it through a common implementation mechanism: a compiler pass which divides a function into multiple components.  This is important at a low level of abstraction in order to understand how the machine is operating, but at a high level we’d like to encourage you to ignore it.  Instead, think of an asynchronous function as an ordinary function that has the special power to give up its thread.  Asynchronous functions don’t typically use this power directly; instead, they make calls, and sometimes these calls will require them to give up their thread and wait for something to happen.  When that thing is complete, the function will resume executing again.

The analogy with synchronous functions is very strong.  A synchronous function can make a call; when it does, the function immediately waits for the call to complete. Once the call completes, control returns to the function and picks up where it left off. The same thing is true with an asynchronous function: it can make calls as usual; when it does, it (normally) immediately waits for the call to complete. Once the call completes, control returns to the function and it picks up where it was. The only difference is that synchronous functions get to take full advantage of (part of) their thread and its stack, whereas *asynchronous functions are able to completely give up that stack and use their own, separate storage*.  This additional power given to asynchronous functions has some implementation cost, but we can reduce that quite a bit by designing holistically around it.

Because asynchronous functions must be able to abandon their thread, and synchronous functions don’t know how to abandon a thread, a synchronous function can’t ordinarily call an asynchronous function: the asynchronous function would only be able to give up the part of the thread it occupied, and if it tried, its synchronous caller would treat it like a return and try to pick up where it was, only without a return value. The only way to make this work in general would be to block the entire thread until the asynchronous function was resumed and completed, and that would completely defeat the purpose of asynchronous functions, as well as having nasty systemic effects.

In contrast, an asynchronous function can call either synchronous or asynchronous functions.  While it’s calling a synchronous function, of course, it can’t give up its thread. In fact, asynchronous functions never just spontaneously give up their thread; they only give up their thread when they reach what’s called a suspension point. A suspension point can occur directly within a function, or it can occur within another asynchronous function that the function calls, but in either case the function and all of its asynchronous callers simultaneously abandon the thread. (In practice, asynchronous functions are compiled to not depend on the thread during an asynchronous call, so that only the innermost function needs to do any extra work.)

When control returns to an asynchronous function, it picks up exactly where it was. That doesn’t necessarily mean that it’ll be running on the exact same thread it was before, because the language doesn’t guarantee that after a suspension. In this design, threads are mostly an implementation mechanism, not a part of the intended interface to concurrency.  However, many asynchronous functions are not just asynchronous: they’re also associated with specific actors (which are the subject of a separate proposal), and they’re always supposed to run as part of that actor. Swift does guarantee that such functions will in fact return to their actor to finish executing. Accordingly, libraries that use threads directly for state isolation—for example, by creating their own threads and scheduling tasks sequentially onto them—should generally model those threads as actors in Swift in order to allow these basic language guarantees to function properly.

</details>

### Suspension points

일시 중단 지점은 비동기 함수의 실행 중 스레드를 포기해야 하는 시점을 말합니다. 일시 중단 지점은 항상 함수 내에서 결정 가능하고(deterministic), 문법적으로 명시되어 있습니다. 즉, 함수의 관점에서는 일시 중단 지점이 숨겨져 있거나, 예기치 않게 비동기로 작동하지 않습니다. 일시 중단 지점의 주요 형태는 다른 실행 컨텍스트에 속한 비동기 함수를 호출하는 것입니다.

일시 중단 지점은 항상 코드상에서 명확히 드러나는 동작을 통해 발생해야 합니다. 실제로 이는 너무 중요하기 때문에, 이 제안에서는 중단될 가능성이 **있는** 모든 호출은 반드시 `await` 표현식으로 감싸야 한다고 요구합니다. 이러한 호출은 **잠재적인 일시 중단 지점(potential suspension points)**이라고 불립니다. 그 이유는 해당 호출이 실제로 중단될지는 정적으로는 알 수 없기 때문입니다. 이는 호출 시점에서는 보이지 않는 코드, 예를 들어 호출된 함수가 비동기 입출력 작업에 의존할 수 있는 경우나, 실행 중에 결정되는 조건들, 예를 들어 그 비동기 입출력 작업이 실제로 대기해야 하는 상황인지 여부에 따라 달라집니다.

잠재적인 일시 중단 지점에 `await`을 요구하는 것은 오류를 던질 수 있는 함수 호출에 `try`를 명시해야 하는 기존 규칙을 따른 것입니다. 그리고 잠재적인 일시 중단 지점을 명시하는 것은 특히 중요한 이유는 **일시 중단이 실행의 원자성(atomicity)을 깨뜨릴 수 있기 때문** 입니다. 예를 들어, 비동기 함수가 직렬 큐(serial queue)로 보호되는 특정 컨텍스트 내에서 실행 중일 때, 일시 중단 지점에 도달하면 동일한 직렬 큐에서 다른 코드가 끼어들어(interleave) 실행될 수 있다는 것을 의미합니다. 원자성을 설명할 때 자주 등장하는 (다소 진부한) 예로는 은행을 들 수 있습니다. 예를 들어, 한 계좌에 입금이 반영된 후, 이에 상응하는 출금 처리가 이루어지기 전에 작업이 일시 중단되면, 그 사이의 틈에서 자금이 이중으로 인출될 수 있는 가능성이 생깁니다. 많은 Swift 개발자에게 더 익순한 예는 UI 스레드입니다. 일시 중단 지점은 사용자에게 UI가 표시될 수 있는 시점이기 때문에, UI의 일부만 구성된 상태에서 작업이 일시 중단되면, 깜박이거나 불완전한 UI가 사용자에게 노출될 위험이 있습니다. (일시 중단 지점은 명시적인 콜백을 사용하는 코드에서도 분명히 드러납니다. 이 경우 일시 중단은 바깥 함수가 반환된 이후, 콜백이 실행되기 전 사이에 발생합니다.) 모든 잠재적인 일시 중단 지점에 `await`을 명시하도록 요구하면, 개발자는 일시 중단 지점이 없는 코드는 원자적으로 동작한다고 안전하게 가정할 수 있습니다. 또한, 문제가 될 수 있는 비-원자적 패턴을 더 쉽게 인식할 수 있게 됩니다.

잠재적인 일시 중단 지점은 비동기 함수 내에서 명시적으로 표시된 지점에서만 나타날 수 있기 때문에, 긴 계산 작업은 여전히 스레드를 차단할 수 있습니다. 이런 상황은 많은 작업을 수행하는 동기 함수를 호출할 때 발생할 수 있으며, 비동기 함수 내부에 계산량이 많은 반복문이 작성되어 있을 때도 마찬가지로 발생할 수 있습니다. 어느 경우든, 이러한 계산이 수행되는 동안 스레드는 다른 코드를 끼워 넣을 수 없습니다. 이는 보통 올바른 동작을 보장하기 위한 적절한 선택이지만, 동시에 확장성 측면에서 문제가 될 수 있습니다. 집약적인 계산을 수행해야 하는 비동기 프로그램은 일반적으로 해당 작업을 별도의 컨텍스트에서 실행하는 것이 좋습니다. 그렇게 하는 게 어려운 경우에는, 다른 작업이 끼어들 수 있도록 인위적으로 일시 중단할 수 있는 라이브러리 기능이 제공될 수 있습니다.

비동기 함수는 스레드를 실제로 차단할 수 있는 함수를 호출하는 것을 피해야 합니다. 특히, 아직 실행되지 않았을 수도 있는 작업을 기다리느라 스레드를 차단하는 경우라면 더욱 피해야 합니다. 예를 들어, 뮤텍스(mutex)를 사용하는 경우에는 현재 실행 중인 다른 스레드가 해당 뮤텍스를 해제할 때까지 스레드를 일시적으로 차단합니다. 이는 경우에 따라 허용될 수 있지만, 교착 상태(deadlock)나 확장성 저하와 같은 문제를 일으킬 수 있으므로 신중하게 사용해야 합니다. 반면, 조건 변수(condition variable)를 사용하는 경우에는 임의의 작업이 스케줄되어 실행된 후 해당 조건을 시그널(signal)할 때까지 스레드가 차단될 수 있습니다. 이처럼 실행 시점이 불확실한 작업을 기다리며 스레드를 멈추게 만드는 방식은 권장 사항에 크게 어긋나므로 지양해야 합니다.

<details>

<summary>원문 보기</summary>

A suspension point is a point in the execution of an asynchronous function where it has to give up its thread. Suspension points are always associated with some deterministic, syntactically explicit event in the function; they’re never hidden or asynchronous from the function’s perspective. The primary form of suspension point is a call to an asynchronous function associated with a different execution context.

It is important that suspension points are only associated with explicit operations. In fact, it’s so important that this proposal requires that calls that *might* suspend be enclosed in an `await` expression. These calls are referred to as *potential suspension points*, because it is not known statically whether they will actually suspend: that depends both on code not visible at the call site (e.g., the callee might depend on asynchronous I/O) as well as dynamic conditions (e.g., whether that asynchronous I/O will have to wait to complete). 

The requirement for `await` on potential suspension points follows Swift's precedent of requiring `try` expressions to cover calls to functions that can throw errors. Marking potential suspension points is particularly important because *suspensions interrupt atomicity*. For example, if an asynchronous function is running within a given context that is protected by a serial queue, reaching a suspension point means that other code can be interleaved on that same serial queue. A classic but somewhat hackneyed example where this atomicity matters is modeling a bank: if a deposit is credited to one account, but the operation suspends before processing a matched withdrawal, it creates a window where those funds can be double-spent. A more germane example for many Swift programmers is a UI thread: the suspension points are the points where the UI can be shown to the user, so programs that build part of their UI and then suspend risk presenting a flickering, partially-constructed UI. (Note that suspension points are also called out explicitly in code using explicit callbacks: the suspension happens between the point where the outer function returns and the callback starts running.) Requiring that all potential suspension points are marked allows programmers to safely assume that places without potential suspension points will behave atomically, as well as to more easily recognize problematic non-atomic patterns.

Because potential suspension points can only appear at points explicitly marked within an asynchronous function, long computations can still block threads. This might happen when calling a synchronous function that just does a lot of work, or when encountering a particularly intense computational loop written directly in an asynchronous function. In either case, the thread cannot interleave code while these computations are running, which is usually the right choice for correctness, but can also become a scalability problem. Asynchronous programs that need to do intense computation should generally run it in a separate context. When that’s not feasible, there will be library facilities to artificially suspend and allow other operations to be interleaved.

Asynchronous functions should avoid calling functions that can actually block the thread, especially if they can block it waiting for work that’s not guaranteed to be currently running. For example, acquiring a mutex can only block until some currently-running thread gives up the mutex; this is sometimes acceptable but must be used carefully to avoid introducing deadlocks or artificial scalability problems. In contrast, waiting on a condition variable can block until some arbitrary other work gets scheduled that signals the variable; this pattern goes strongly against recommendation.

</details>

## Detailed design

### Asynchronous functions

함수 타입은 `async` 키워드를 명시적으로 붙여 비동기 함수임을 나타낼 수 있습니다.

```swift
func collect(function: () async -> Int) { ... }
```

함수나 이니셜라이저도 `async` 키워드를 사용해 명시적으로 비동기 함수로 선언할 수 있습니다.

```swift
class Teacher {
  init(hiringFrom: College) async throws {
    ...
  }
  
  private func raiseHand() async -> Bool {
    ...
  }
}
```

> **이유**: `async`는 단순히 함수가 비동기적으로 동작한다는 것을 선언하는 데 그치지 않고, 이 함수의 타입을 구성하는 중요한 부분이기 때문에 매개변수 목록 뒤에 위치합니다. 이는 `throws` 키워드의 사용 방식과 동일한 규칙을 따릅니다.

`async`로 선언된 함수나 이니셜라이저를 참조할 때, 그 참조 타입은 `async` 함수 타입이 됩니다. 만약 해당 참조가 인스턴스 메서드를 타입 이름을 통해(curried) 정적으로 참조한 것이라면, 일반적인 규칙에 따라 "내부(inner)" 함수 타입은 `async`가 됩니다.

`deinit`이나 프로퍼티와 서브스크립트의 getter와 setter와 같은 특수한 함수는 `async`로 선언할 수 없습니다.[^1]

[^1]: 

> **이유**: getter만 가지는 프로퍼티나 서브스크립트는 이론적으로 `async`로 선언할 수 있습니다. 그러나, setter까지 `async`로 선언하게 되면, 해당 값을 `inout` 방식으로 넘기고, 내부 속성(properties)에 접근하는 등의 상황에서 문제가 발생할 수 있습니다. setter는 본질적으로 즉시 실행되는 (동기적이고 오류를 던지지 않는) 동작이어야만 하며, 중간에 멈추거나 지연되면 안됩니다. 그래서 getter만 `async`를 허용하는 복잡한 규칙을 두기보다는, getter와 setter 모두에서 `async`를 금지하는 단순한 규칙을 채택한 것입니다.
 
함수가 `async`와 `throws`를 모두 사용하는 경우, 타입 선언에서 `async` 키워드는 반드시 `throws`보다 먼저 와야 합니다. `async`와 `rethrows`를 함께 사용할 때도 같은 규칙이 적용됩니다.

> **이유**: 이 순서 제한은 임의적이지만, 코드 스타일에 대한 불필요한 논쟁의 소지를 없애줍니다.

상위 클래스가 있는 클래스의 `async` 이니셜라이저에서 `super.init()`을 명시적으로 호출하지 않은 경우, 상위 클래스에 매개변수가 없는 동기 지정 이니셜라이저(desinated initializer)가 있을 때만 `super.init()`을 암시적으로 호출합니다.

> **이유**: 상위 클래스의 이니셜라이저가 `async`인 경우, 그 호출은 잠재적인 일시 중단 지점이 됩니다. 따라서, 해당 호출(그리고 필수적인 `await`)은 소스 코드에서 명시적으로 드러나야 합니다.

<details>

<summary>원문 보기</summary>

Function types can be marked explicitly as `async`, indicating that the function is asynchronous:

```swift
func collect(function: () async -> Int) { ... }
```

A function or initializer declaration can also be declared explicitly as `async`:

```swift
class Teacher {
  init(hiringFrom: College) async throws {
    ...
  }
  
  private func raiseHand() async -> Bool {
    ...
  }
}
```

> **Rationale**: The `async` follows the parameter list because it is part of the function's type as well as its declaration. This follows the precedent of `throws`.

The type of a reference to a function or initializer declared `async` is an `async` function type. If the reference is a “curried” static reference to an instance method, it is the "inner" function type that is `async`, consistent with the usual rules for such references.

Special functions like `deinit` and storage accessors (i.e., the getters and setters for properties and subscripts) cannot be `async`.

> **Rationale**: Properties and subscripts that only have a getter could potentially be `async`. However, properties and subscripts that also have an `async` setter imply the ability to pass the reference as `inout` and drill down into the properties of that property itself, which depends on the setter effectively being an "instantaneous" (synchronous, non-throwing) operation. Prohibiting `async` properties is a simpler rule than only allowing get-only `async` properties and subscripts.
 
If a function is both `async` and `throws`, then the `async` keyword must precede `throws` in the type declaration. This same rule applies if `async` and `rethrows`.

> **Rationale** : This order restriction is arbitrary, but it's not harmful, and it eliminates the potential for stylistic debates.

An `async` initializer of a class that has a superclass but lacks a call to a superclass initializer will get an implicit call to `super.init()` only if the superclass has a zero-argument, synchronous, designated initializer.

> **Rationale**: If the superclass initializer is `async`, the call to the asynchronous initializer is a potential suspension point and therefore the call (and required `await`) must be visible in the source.

</details>
 
### Asynchronous function types

비동기 함수 타입은 동기 함수 타입과 서로 다른 별개의 타입입니다. 하지만, 동기 함수 타입에서 상응하는 비동기 함수 타입으로 암시적 변환이 가능합니다. 이는 예외를 던지지 않는 함수에서 예외를 던질 수 있는 함수로 암시적 변환이 가능한 것과 유사하며, 비동기 함수의 암시적 변환과 함께 동작할 수도 있습니다. 예를 들어:

```swift
struct FunctionTypes {
  var syncNonThrowing: () -> Void
  var syncThrowing: () throws -> Void
  var asyncNonThrowing: () async -> Void
  var asyncThrowing: () async throws -> Void
  
  mutating func demonstrateConversions() {
    // Okay to add 'async' and/or 'throws'    
    asyncNonThrowing = syncNonThrowing
    asyncThrowing = syncThrowing
    syncThrowing = syncNonThrowing
    asyncThrowing = asyncNonThrowing
    
    // Error to remove 'async' or 'throws'
    syncNonThrowing = asyncNonThrowing // error
    syncThrowing = asyncThrowing       // error
    syncNonThrowing = syncThrowing     // error
    asyncNonThrowing = syncThrowing    // error
  }
}
```

<details>

<summary>원문 보기</summary>

Asynchronous function types are distinct from their synchronous counterparts. However, there is an implicit conversion from a synchronous function type to its corresponding asynchronous function type. This is similar to the implicit conversion from a non-throwing function to its throwing counterpart, which can also compose with the asynchronous function conversion. For example:

```swift
struct FunctionTypes {
  var syncNonThrowing: () -> Void
  var syncThrowing: () throws -> Void
  var asyncNonThrowing: () async -> Void
  var asyncThrowing: () async throws -> Void
  
  mutating func demonstrateConversions() {
    // Okay to add 'async' and/or 'throws'    
    asyncNonThrowing = syncNonThrowing
    asyncThrowing = syncThrowing
    syncThrowing = syncNonThrowing
    asyncThrowing = asyncNonThrowing
    
    // Error to remove 'async' or 'throws'
    syncNonThrowing = asyncNonThrowing // error
    syncThrowing = asyncThrowing       // error
    syncNonThrowing = syncThrowing     // error
    asyncNonThrowing = syncThrowing    // error
  }
}
```

</details>

### Await expressions

`async` 함수 타입의 값을 호출하는 경우(직접 `async` 함수를 호출하는 경우를 포함하여), 잠재적인 일시 중단 지점이 생깁니다. 이러한 잠재적인 일시 중단 지점은 반드시 비동기 컨텍스트(예: `async` 함수) 안에서 일어나야 하며, `await` 표현식의 피연산자로서 호출되어야 합니다.

다음 예제를 살펴봅시다:

```swift
// func redirectURL(for url: URL) async -> URL { ... }
// func dataTask(with: URL) async throws -> (Data, URLResponse) { ... }

let newURL = await server.redirectURL(for: url)
let (data, response) = try await session.dataTask(with: newURL)
```

이 코드에서 `redirectURL(for:)`와 `dataTask(with:)`는 모두 `async` 함수이므로, 이들을 호출하는 동안 작업이 일시 중단될 수 있습니다. 이러한 호출은 잠재적인 일시 중단 지점을 포함하고 있기 때문에, 반드시 `await` 표현식 안에서 호출되어야 합니다. `await` 표현식은 여러 개의 잠재적인 일시 중단 지점을 포함할 수 있습니다. 예를 들어, 위 코드를 다음과 같이 다시 작성하면, `await` 하나로 두 함수 호출을 모두 감쌀 수 있습니다.

```swift
let (data, response) = try await session.dataTask(with: server.redirectURL(for: url))
```

`await`은 추가적인 의미를 가지지 않습니다.`try`와 마찬가지로, 단지 비동기 호출이 이루어지고 있음을 나타내는 표시일 뿐입니다. `await` 표현식의 타입은 그 피연산자의 타입과 동일하며, 결과도 피연산자의 결과값과 같습니다. `await` 표현식의 피연산자에 잠재적인 일시 중단 지점이 없을 수 있으며, 이 경우 Swift 컴파일러는 `try` 표현식과 마찬가지로 경고를 발생시킵니다.

```swift
let x = await synchronous() // warning: no calls to 'async' functions occur within 'await' expression
```

> **이유**: 비동기 호출은 함수 내부에서 명확히 식별 가능해야 합니다. 이러한 호출은 일시 중단 지점을 만들어낼 수 있고, 이는 연산의 원자성(atmocity)을 깨드릴 수 있기 때문입니다. 일시 중단 지점은 해당 호출이 본질적으로 다른 실행자(executor)에서 실행되어야 하기 때문에 발생할 수도 있고, 단순히 호출되는 함수의 구현에 포함되어 있기 때문에 발생할 수 있습니다. 하지만, 어느 경우든 의미론적으로 중요하므로, 개발자가 이를 명시적으로 인지해야 합니다. `await` 표현식은 이러한 비동기 코드를 나타내는 지표 역햘을 하며, 이는 클로저의 타입 추론과도 상호작용합니다. 자세한 내용은 [클로저](#closures)를 참조하세요.

`async` 함수 타입이 아닌 오토클로저(autoclosure) 안에는 잠재적인 일시 중단 지점이 포함되어서는 안됩니다.

잠재적인 일시 중단 지점은 `defer` 블록 안에 포함되어서는 안됩니다.

`await`과 `try` 계열(try, try!, try?)이 동일한 하위 표현식에 함께 적용되는 경우, `await`은 반드시 `try`/`try!`/`try?` 뒤에 봐야 합니다.

```swift
let (data, response) = await try session.dataTask(with: server.redirectURL(for: url)) // error: must be `try await`
let (data, response) = await (try session.dataTask(with: server.redirectURL(for: url))) // okay due to parentheses
```

> **이유**: 이 제약은 임의적이지만, `async throws`의 순서를 제한하는 규칙과 마찬가지로 코드 스타일에 대한 불필요한 논쟁의 소지를 없애줍니다.

<details>

<summary>원문 보기</summary>

A call to a value of async function type (including a direct call to an async function) introduces a potential suspension point. Any potential suspension point must occur within an asynchronous context (e.g., an async function). Furthermore, it must occur within the operand of an await expression.

Consider the following example:

```swift
// func redirectURL(for url: URL) async -> URL { ... }
// func dataTask(with: URL) async throws -> (Data, URLResponse) { ... }

let newURL = await server.redirectURL(for: url)
let (data, response) = try await session.dataTask(with: newURL)
```

In this code example, a task suspension may happen during the calls to `redirectURL(for:)` and `dataTask(with:)` because they are async functions. Thus, both call expressions must be contained within some `await` expression, because each call contains a potential suspension point. An `await` operand may contain more than one potential suspension point. For example, we can use one `await` to cover both potential suspension points from our example by rewriting it as:

```swift
let (data, response) = try await session.dataTask(with: server.redirectURL(for: url))
```

The `await` has no additional semantics; like `try`, it merely marks that an asynchronous call is being made.  The type of the `await` expression is the type of its operand, and its result is the result of its operand.
An `await` operand may also have no potential suspension points, which will result in a warning from the Swift compiler, following the precedent of `try` expressions:

```swift
let x = await synchronous() // warning: no calls to 'async' functions occur within 'await' expression
```

> **Rationale**: It is important that asynchronous calls are clearly identifiable within the function because they may introduce suspension points, which break the atomicity of the operation.  The suspension points may be inherent to the call (because the asynchronous call must execute on a different executor) or simply be part of the implementation of the callee, but in either case it is semantically important and the programmer needs to acknowledge it. `await` expressions are also an indicator of asynchronous code, which interacts with inference in closures; see the section on [Closures](#closures) for more information.

A potential suspension point must not occur within an autoclosure that is not of `async` function type.

A potential suspension point must not occur within a `defer` block.

If both `await` and a variant of `try` (including `try!` and `try?`) are applied to the same subexpression, `await` must follow the `try`/`try!`/`try?`:

```swift
let (data, response) = await try session.dataTask(with: server.redirectURL(for: url)) // error: must be `try await`
let (data, response) = await (try session.dataTask(with: server.redirectURL(for: url))) // okay due to parentheses
```

> **Rationale**: this restriction is arbitrary, but follows the equally-arbitrary restriction on the ordering of `async throws` in preventing stylistic debates.

</details>
 
### Closures

클로저는 `async` 함수 타입을 가질 수 있습니다. 이러한 클로저는 다음과 같이 `async`로 명시적으로 표시할 수 있습니다.

```swift
{ () async -> Int in
  print("here")
  return await getInt()
}
```

익명 클로저 내부에 `await` 표현식이 포함되어 있다면, 해당 클로저는 자동으로 `async` 함수 타입으로 추론됩니다.

```swift
let closure = { await getInt() } // implicitly async

let closure2 = { () -> Int in     // implicitly async
  print("here")
  return await getInt()
}
```

클로저에서 `async`로의 추론은 해당 클로저 자체에만 적용되며, 이를 감싸고 있는 바깥 함수나 중첩된 다른 함수나 클로저에는 전파되지 않습니다. 이는 각 컨텍스트가 서로 독립저그올 동기 또는 비동기인지 판단하기 때문입니다. 예를 들어, 아래 상황에서는 오직 `closure6`만 `async`로 추론됩니다.

```swift
// func getInt() async -> Int { ... }

let closure5 = { () -> Int in       // not 'async'
  let closure6 = { () -> Int in     // implicitly async
    if randomBool() {
      print("there")
      return await getInt()
    } else {
      let closure7 = { () -> Int in 7 }  // not 'async'
      return 0
    }
  }
  
  print("here")
  return 5
}
```

<details>

<summary>원문 보기</summary>

A closure can have `async` function type. Such closures can be explicitly marked as `async` as follows:

```swift
{ () async -> Int in
  print("here")
  return await getInt()
}
```

An anonymous closure is inferred to have `async` function type if it contains an `await` expression.

```swift
let closure = { await getInt() } // implicitly async

let closure2 = { () -> Int in     // implicitly async
  print("here")
  return await getInt()
}
```

Note that inference of `async` on a closure does not propagate to its enclosing or nested functions or closures, because those contexts are separably asynchronous or synchronous. For example, only `closure6` is inferred to be `async` in this situation:

```swift
// func getInt() async -> Int { ... }

let closure5 = { () -> Int in       // not 'async'
  let closure6 = { () -> Int in     // implicitly async
    if randomBool() {
      print("there")
      return await getInt()
    } else {
      let closure7 = { () -> Int in 7 }  // not 'async'
      return 0
    }
  }
  
  print("here")
  return 5
}
```

</details>

### Overloading and overload resolution

기존의 Swift API들은 일반적으로 콜백 인터페이스를 통한 비동기 함수를 지원합니다.

```swift
func doSomething(completionHandler: ((String) -> Void)? = nil) { ... }
```

이러한 API 중 상당수는 다음과 같이 `async` 형태를 추가하여 업데이트될 가능성이 높습니다:

```swift
func doSomething() async -> String { ... }
```

이 두 함수는 동일한 이름을 가지고 있지만, 함수 시그니처(signature)는 서로 다릅니다. 그러나 기본값이 지정된 완료 핸들러 덕분에, 두 함수 모두 매개변수 없이 호출할 수 있으며, 따라서 기존 코드에서는 다음과 같은 문제가 발생할 수 있습니다.

```swift
doSomething() // problem: can call either, unmodified Swift rules prefer the `async` version
```

동일한 시그니처를 가진 동기와 비동기 버전의 함수를 모두 제공하게 되는 API에서도 비슷한 문제가 발생할 수 있습니다. 이러한 함수 쌍(pair)은 기존 API와의 호환성을 깨뜨리지 않으면서, Swift의 비동기 환경에 더 적합한 새로운 비동기 함수를 추가할 수 있도록 해줍니다. 예를 들어, 새로운 비동기 함수는 취소(cancellation)를 지원할 수 있습니다(이는 [구조적 동시성](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)에서 다룹니다).

```swift
// Existing synchronous API
func doSomethingElse() { ... }

// New and enhanced asynchronous API
func doSomethingElse() async { ... }
```

첫 번째 경우에서, Swift의 오버로딩 규칙은 기본 매개변수가 더 적은 함수를 우선적으로 호출합니다. 따라서 완료 핸들러 없이 `doSomething(completionHandler:)`를 호출하던 기존 코드는 `async` 함수가 추가되면 컴파일 오류가 발생할 수 있습니다. 이 오류 메시지는 다음과 유사할 수 있습니다.

```
error: `async` function cannot be called from non-asynchronous context
```

이는 코드의 발전에 있어 문제를 일으킬 수 있습니다. 기존의 비동기 라이브러리의 개발자는 호환성을 과감히 끊고(예: 메이저 업데이트) 나아가거나, 새롭게 추가되는 모든 `async` 버전에 대해 서로 다른 이름을 붙여야 하기 때문입니다. 후자의 경우, 이는 [C#에서 광범위하게 사용되는 `Async` 접미사](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/async/task-asynchronous-programming-model)와 같은 네이밍 방식으로 이어질 가능성이 높습니다. 

두 번째 경우처럼, 두 함수가 동일한 시그니처를 가지되 `async`만 다른 경우는 기존 Swift의 오버로딩 규칙에 따라 일반적으로 허용되지 않습니다. Swift의 오버로딩 규칙은 함수 간의 **효과(effect)**만으로 구분되는 것을 허용하지 않습니다. 예를 들어, `throws`만 다른 두 함수를 정의할 수 없는 것과 같은 맥락입니다.

```
// error: redeclaration of function `doSomethingElse()`.
```

이 또한 코드의 발전에 있어 문제를 일으킬 수 있습니다. 기존 라이브러리의 개발자들이 기존의 동기 API를 유지하면서, 동시에 새로운 비동기 기능을 지원하는 것이 불가능해질 수 있기 때문입니다.

대신, 우리는 함수 호출의 컨텍스트에 따라 적절한 함수는 선택하는 새로운 오버로드 해석 규칙(overload-resolution)을 제안합니다. 함수가 호출될 때, 동기 컨텍스트 내에서는 `async`가 아닌 함수를 우선적으로 선택합니다(이는 동기 컨텍스트에서는 `async` 함수를 호출할 수 없기 때문입니다). 반대로, 비동기 컨텍스트에서는 `async` 함수를 우선적으로 선택합니다(이는 비동기 컨텍스트에서는 비동기 모델을 벗어나 작업을 멈추게 하는(blocking) 동기 API를 호출하는 상황을 피해야 하기 때문입니다). 그리고 Swift 컴파일러가 `async` 함수를 선택한 경우라도, 해당 호출은 여전히 `await` 표현식 안에서 이루어져야 한다는 규칙을 따라야 합니다. 

오버로드 해석 규칙은 현재 컨텍스트가 동기인지 비동기인지에 따라 달라지며, 컴파일러는 이 규칙에 따라 하나의 오버로드만을 선택합니다. `async` 오버로드가 선택된 경우에는, 해당 호출이 잠재적인 일시 중단 지점이 될 수 있으므로 `await` 표현식 안에서 호출되어야 합니다. 

```swift
func f() async {
  // In an asynchronous context, the async overload is preferred:
  await doSomething()
  // Compiler error: Expression is 'async' but is not marked with 'await'
  doSomething()
}
```

In non-`async` functions, and closures without any `await` expression, the compiler selects the non-`async` overload:

```swift
func f() async {
  let f2 = {
    // In a synchronous context, the non-async overload is preferred:
    doSomething()
  }
  f2()
}
```

<details>

<summary>원문 보기</summary>

Existing Swift APIs generally support asynchronous functions via a callback interface, e.g.,

```swift
func doSomething(completionHandler: ((String) -> Void)? = nil) { ... }
```

Many such APIs are likely to be updated by adding an `async` form:

```swift
func doSomething() async -> String { ... }
```

These two functions have different names and signatures, even though they share the same base name. However, either of them can be called with no parameters (due to the defaulted completion handler), which would present a problem for existing code:

```swift
doSomething() // problem: can call either, unmodified Swift rules prefer the `async` version
```

A similar problem exists for APIs that evolve into providing both a synchronous and an asynchronous version of the same function, with the same signature. Such pairs allow APIs to provide a new asynchronous function which better fits in the Swift asynchronous landscape, without breaking backward compatibility. New asynchronous functions can support, for example, cancellation (covered in the [Structured Concurrency](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md) proposal).

```swift
// Existing synchronous API
func doSomethingElse() { ... }

// New and enhanced asynchronous API
func doSomethingElse() async { ... }
```

In the first case, Swift's overloading rules prefer to call a function with fewer default arguments, so the addition of the `async` function would break existing code that called the original `doSomething(completionHandler:)` with no completion handler. This would get an error along the lines of:

```
error: `async` function cannot be called from non-asynchronous context
```

This presents problems for code evolution, because developers of existing asynchronous libraries would have to either have a hard compatibility break (e.g, to a new major version) or would need have different names for all of the new `async` versions. The latter would likely result in a scheme such as [C#'s pervasive `Async` suffix](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/async/task-asynchronous-programming-model).

The second case, where both functions have the same signature and only differ in `async`, is normally rejected by existing Swift's overloading rules. Those do not allow two functions to differ only in their *effects*, and one can not define two functions that only differ in `throws`, for example.

```
// error: redeclaration of function `doSomethingElse()`.
```

This also presents a problem for code evolution, because developers of existing libraries just could not preserve their existing synchronous APIs, and support new asynchronous features.

Instead, we propose an overload-resolution rule to select the appropriate function based on the context of the call. Given a call, overload resolution prefers non-`async` functions within a synchronous context (because such contexts cannot contain a call to an `async` function).  Furthermore, overload resolution prefers `async` functions within an asynchronous context (because such contexts should avoid stepping out of the asynchronous model into blocking APIs). When overload resolution selects an `async` function, that call is still subject to the rule that it must occur within an `await` expression.

The overload-resolution rule depends on the synchronous or asynchronous context, in which the compiler selects one and only one overload. The selection of the async overload requires an `await` expression, as all introductions of a potential suspension point:

```swift
func f() async {
  // In an asynchronous context, the async overload is preferred:
  await doSomething()
  // Compiler error: Expression is 'async' but is not marked with 'await'
  doSomething()
}
```

In non-`async` functions, and closures without any `await` expression, the compiler selects the non-`async` overload:

```swift
func f() async {
  let f2 = {
    // In a synchronous context, the non-async overload is preferred:
    doSomething()
  }
  f2()
}
```

</details>


### Autoclosures

함수 자체가 `async`로 선언되지 않았다면, `async` 함수 타입의 오토클로저(autoclosure)를 매개변수로 받을 수 없습니다. 예를 들어, 아래와 같은 선언은 올바르지 않습니다.

```swift
// error: async autoclosure in a function that is not itself 'async'
func computeArgumentLater<T>(_ fn: @escaping @autoclosure () async -> T) { } 
```

이 제한은 여러 가지 이유로 존재합니다. 다음 예제를 살펴봅시다:

  ```swift
  // func getIntSlowly() async -> Int { ... }

  let closure = {
    computeArgumentLater(await getIntSlowly())
    print("hello")
  }
  ```

겉보기에는 `await` 표현식이 `computeArgumentLater(_:)` 호출 **이전**에 잠재적인 일시 중단 지점이 있는 것처럼 보입니다. 하지만 실제로는 그렇지 않습니다. 잠재적인 일시 중단 지점은 `computeArgumentLater(_:)`에 전달되어 본문(body)에서 사용되는 (오토)클로저 **내부**에 존재합니다. 이로 인해 몇 가지 문제가 발생합니다. 첫째, `await`이 함수 호출 앞에 위치해 있기 때문에, Swift 컴파일러는 `closure`를 `async` 함수 타입으로 잘못 추론하게 됩니다. 그러나 클로저 내부의 코드는 모두 동기적으므로 이는 올바르지 않습니다. 둘째, `await` 표현식의 피연산자는 내부 어딘가에 잠재적인 일시 중단 지점만 포함하고 있어도 되기 때문에, 이 호출은 다음과 같이 다시 작성할 수 있어야 합니다. 

```swift
await computeArgumentLater(getIntSlowly())
```

하지만, 인자(argument)는 오토클로저이기 때문에, 위와 같은 방식으로 다시 작성하면 의미가 달라져 더 이상 동작을 보존하지 않게 됩니다. 따라서 `async` 오토클로저 매개변수에 대한 제약은 이러한 문제들을 방지하기 위한 것이며, `async` 오토클로저 매개변수가 비동기 컨텍스트에서만 사용되도록 제한됩니다.

<details>

<summary>원문 보기</summary>

A function may not take an autoclosure parameter of `async` function type unless the function itself is `async`. For example, the following declaration is ill-formed:

```swift
// error: async autoclosure in a function that is not itself 'async'
func computeArgumentLater<T>(_ fn: @escaping @autoclosure () async -> T) { } 
```

This restriction exists for several reasons. Consider the following example:

  ```swift
  // func getIntSlowly() async -> Int { ... }

  let closure = {
    computeArgumentLater(await getIntSlowly())
    print("hello")
  }
  ```

At first glance, the `await` expression implies to the programmer that there is a potential suspension point *prior* to the call to `computeArgumentLater(_:)`, which is not actually the case: the potential suspension point is *within* the (auto)closure that is passed and used within the body of `computeArgumentLater(_:)`. This causes a few problems. First, the fact that `await` appears to be prior to the call means that `closure` would be inferred to have `async` function type, which is also incorrect: all of the code in `closure` is synchronous. Second, because an `await`'s operand only needs to contain a potential suspension point somewhere within it, an equivalent rewriting of the call should be:

```swift
await computeArgumentLater(getIntSlowly())
```

But, because the argument is an autoclosure, this rewriting is no longer semantics-preserving. Thus, the restriction on `async` autoclosure parameters avoids these problems by ensuring that `async` autoclosure parameters can only be used in asynchronous contexts.

</details>

### Protocol conformance

프로토콜 요구사항은 `async`로 선언할 수 있습니다. 이렇게 선언된 요구사항은 `async` 함수나 동기 함수로 충족할 수 있습니다. 그러나, 동기 함수로 선언된 프로토콜 요구사항은 `async` 함수로 충족할 수 없습니다. 예를 들어:

```swift
protocol Asynchronous {
  func f() async
}

protocol Synchronous {
  func g()
}

struct S1: Asynchronous {
  func f() async { } // okay, exactly matches
}

struct S2: Asynchronous {
  func f() { } // okay, synchronous function satisfying async requirement
}

struct S3: Synchronous {
  func g() { } // okay, exactly matches
}

struct S4: Synchronous {
  func g() async { } // error: cannot satisfy synchronous requirement with an async function
}
```

이 동작은 비동기 함수에 대한 서브타이핑(subtyping) 및 암시적 변환 규칙을 따르며, 이는 `throws` 키워드가 작동하는 방식과 같은 선례를 따릅니다.

<details>

<summary>원문 보기</summary>

A protocol requirement can be declared as `async`. Such a requirement can be satisfied by an `async` or synchronous function. However, a synchronous function requirement cannot be satisfied by an `async` function. For example:

```swift
protocol Asynchronous {
  func f() async
}

protocol Synchronous {
  func g()
}

struct S1: Asynchronous {
  func f() async { } // okay, exactly matches
}

struct S2: Asynchronous {
  func f() { } // okay, synchronous function satisfying async requirement
}

struct S3: Synchronous {
  func g() { } // okay, exactly matches
}

struct S4: Synchronous {
  func g() async { } // error: cannot satisfy synchronous requirement with an async function
}
```

This behavior follows the subtyping/implicit conversion rule for asynchronous functions, as is precedented by the behavior of `throws`.

</details>

## Source compatibility

이 제안은 전반적으로 기존 기능에 추가되는 형태(additive)입니다. 즉, 기존 코드는 새로운 기능(`async` 함수나 클로저 등)을 사용하지 않기 때문에 영향을 받지 않습니다. 다만, 이 제안에서는 두 개의 새로운 문맥적 키워드(contextual keyword)인 `async`와 `await`를 도입합니다.

`async`가 사용되는 새로운 위치(함수 선언과 함수 타입)는 문법적으로 안전한 위치이기 때문에, `async`를 문맥적 키워드로 처리해도 기존 소스 코드와의 호환성을 깨뜨리지 않습니다. 잘 구성된 코드에서는 사용자 정의 식별자 `async`가 해당 위치에 올 수 없기 때문입니다.

`await`는 표현식 내부에서 사용되는 문맥적 키워드이기 때문에, 이 부분은 조금 더 문제가 될 수 있습니다. 예를 들어, 현재의 Swift에서는 `await`라는 이름의 함수를 직접 정의할 수도 있습니다:

```swift
func await(_ x: Int, _ y: Int) -> Int { x + y }

let result = await(1, 2)
```

이 코드는 현재 기준으로는 정상적인 코드이며, `await`라는 이름의 함수를 호출하는 것으로 해석됩니다. 하지만 이 제안이 도입되면, 해당 코드는 `await` 표현식으로 해석되며, (1, 2)는 그 하위 표현식(subexpression)이 됩니다. 이로 인해 기존 Swift 프로그램에서는 컴파일 오류가 발생하게 됩니다. 왜냐하면 `await`는 비동기 컨텍스트 안에서만 사용 가능하며, 현재의 Swift 코드에는 그런 컨텍스트가 존재하지 않기 때문입니다. 다만, 이러한 방식으로 정의된 `await` 함수는 실제로 거의 사용되지 않는 경우가 많기 때문에, 이 정도의 소스 코드 변경(source break)은 async·await 도입을 위한 수용 가능한 수준의 변화라고 판단됩니다.

<details>

<summary>원문 보기</summary>

This proposal is generally additive: existing code does not use any of the new features (e.g., does not create `async` functions or closures) and will not be impacted. However, it introduces two new contextual keywords, `async` and `await`.

The positions of the new uses of `async` within the grammar (function declarations and function types) allows us to treat `async` as a contextual keyword without breaking source compatibility. A user-defined `async` cannot occur in those grammatical positions in well-formed code.

The `await` contextual keyword is more problematic, because it occurs within an expression. For example, one could define a function `await` in Swift today:

```swift
func await(_ x: Int, _ y: Int) -> Int { x + y }

let result = await(1, 2)
```

This is well-formed code today that is a call to the `await` function. With this proposal, this code becomes an `await` expression with the subexpression `(1, 2)`. This will manifest as a compile-time error for existing Swift programs, because `await` can only be used within an asynchronous context, and no existing Swift programs have such a context. Such functions do not appear to be common, so we believe this is an acceptable source break as part of the introduction of async/await.

</details>

## Effect on ABI stability

비동기 함수와 함수 타입은 추가적인 요소로 도입되는 것으로, 기존의 (동기) 함수나 함수 타입에는 변화가 없습니다. 따라서 ABI 안정성에는 영향을 주지 않습니다.

<details>

<summary>원문 보기</summary>

Asynchronous functions and function types are additive to the ABI, so there is no effect on ABI stability, because existing (synchronous) functions and function types are unchanged.

</details>

## Effect on API resilience

`async` 함수의 ABI는 동기 함수의 ABI와 완전히 다릅니다(예를 들어, 이들은 서로 호환되지 않는 호출 규약(calling convention)을 사용합니다). 따라서 함수나 함수 타입에 `async`를 추가하거나 제거하는 것은 견고한(resilience) 변화로 간주되지 않습니다.

<details>

<summary>원문 보기</summary>

The ABI for an `async` function is completely different from the ABI for a synchronous function (e.g., they have incompatible calling conventions), so the addition or removal of `async` from a function or type is not a resilient change.

</details>

## Future Directions

### `reasync`

Swift's `rethrows` is a mechanism for indicating that a particular function is throwing only when one of the arguments passed to it is a function that itself throws. For example, `Sequence.map` makes use of `rethrows` because the only way the operation can throw is if the transform itself throws:

```swift
extension Sequence {
  func map<Transformed>(transform: (Element) throws -> Transformed) rethrows -> [Transformed] {
    var result = [Transformed]()
    var iterator = self.makeIterator()
    while let element = iterator.next() {
      result.append(try transform(element))   // note: this is the only `try`!
    }
    return result
  }
}
```

Here are uses of `map` in practice:

```swift
_ = [1, 2, 3].map { String($0) }  // okay: map does not throw because the closure does not throw
_ = try ["1", "2", "3"].map { (string: String) -> Int in
  guard let result = Int(string) else { throw IntParseError(string) }
  return result
} // okay: map can throw because the closure can throw
```

The same notion could be applied to `async` functions. For example, we could imagine making `map` asynchronous when its argument is asynchronous with `reasync`:

```swift
extension Sequence {
  func map<Transformed>(transform: (Element) async throws -> Transformed) reasync rethrows -> [Transformed] {
    var result = [Transformed]()
    var iterator = self.makeIterator()
    while let element = iterator.next() {
      result.append(try await transform(element))   // note: this is the only `try` and only `await`!
    }
    return result
  }
}
```

*Conceptually*, this is fine: when provided with an `async` function, `map` will be treated as `async` (and you'll need to `await` the result), whereas providing it with a non-`async` function, `map` will be treated as synchronous (and won't require `await`).

*In practice*, there are a few problems here:

* This is probably not a very good implementation of an asynchronous `map` on a sequence. More likely, we would want a concurrent implementation that (say) processes up to number-of-cores elements concurrently.
* The ABI of throwing functions is intentionally designed to make it possible for a `rethrows` function to act as a non-throwing function, so a single ABI entry point suffices for both throwing and non-throwing calls. The same is not true of `async` functions, which have a radically different ABI that is necessarily less efficient than the ABI for synchronous functions.

For something like `Sequence.map` that might become concurrent, `reasync` is likely the wrong tool: overloading for `async` closures to provide a separate (concurrent) implementation is likely the better answer. So, `reasync` is likely to be much less generally applicable than `rethrows`.

There are undoubtedly some uses for `reasync`, such as the `??` operator for optionals, where the `async` implementation degrades nicely to a synchronous implementation:

```swift
func ??<T>(
    _ optValue: T?, _ defaultValue: @autoclosure () async throws -> T
) reasync rethrows -> T {
  if let value = optValue {
    return value
  }

  return try await defaultValue()
}
```

For such cases, the ABI concern described above can likely be addressed by emitting two entrypoints: one when the argument is `async` and one when it is not. However, the implementation is complex enough that the authors are not yet ready to commit to this design.

## Alternatives Considered

### Make `await` imply `try`

Many asynchronous APIs involve file I/O, networking, or other failable operations, and therefore will be both `async` and `throws`. At the call site, this means `try await` will be repeated many times. To reduce the boilerplate, `await` could imply `try`, so the following two lines would be equivalent:

```swift
let dataResource  = await loadWebResource("dataprofile.txt")
let dataResource  = try await loadWebResource("dataprofile.txt")
```

We chose not to make `await` imply `try` because they are expressing different kinds of concerns: `await` is about a potential suspension point, where other code might execute in between when you make the call and it when it returns, while `try` is about control flow out of the block.

One other motivation that has come up for making `await` imply `try` is related to task cancellation. If task cancellation were modeled as a thrown error, and every potential suspension point implicitly checked whether the task was cancelled, then every potential suspension point could throw: in such cases `await` might as well imply `try` because every `await` can potentially exit with an error.
Task cancellation is covered in the [Structured Concurrency](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md) proposal, and does *not* model cancellation solely as a thrown error nor does it introduce implicit cancellation checks at each potential suspension point.

### Launching async tasks

Because only `async` code can call other `async` code, this proposal provides no way to initiate asynchronous code. This is intentional: all asynchronous code runs within the context of a "task", a notion which is defined in the [Structured Concurrency](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md) proposal. That proposal provides the ability to define asynchronous entry points to the program via `@main`, e.g.,

```swift
@main
struct MyProgram {
  static func main() async { ... }
}
```

Additionally, top-level code is not considered an asynchronous context in this proposal, so the following program is ill-formed:

```swift
func f() async -> String { "hello, asynchronously" }

print(await f()) // error: cannot call asynchronous function in top-level code
```

This, too, will be addressed in a subsequent proposal that properly accounts for
top-level variables.

None of the concerns for top-level code affect the fundamental mechanisms of async/await as defined in this proposal.

### Await as syntactic sugar

This proposal makes `async` functions a core part of the Swift type system, distinct from synchronous functions. An alternative design would leave the type system unchanged, and instead make `async` and `await` syntactic sugar over some `Future<T, Error>` type, e.g.,

```swift
async func processImageData() throws -> Future<Image, Error> {
  let dataResource  = try loadWebResource("dataprofile.txt").await()
  let imageResource = try loadWebResource("imagedata.dat").await()
  let imageTmp      = try decodeImage(dataResource, imageResource).await()
  let imageResult   = try dewarpAndCleanupImage(imageTmp).await()
  return imageResult
}
```

This approach has a number of downsides vs. the proposed approach here:

* There is no universal `Future` type on which to build it in the Swift ecosystem. If the Swift ecosystem had mostly settled on a single future type already (e.g., if there were already one in the standard library), a syntactic-sugar approach like the above would codify existing practice. Lacking such a type, one would have to try to abstract over all of the different kinds of future types with some kind of `Futurable` protocol. This may be possible for some set of future types, but would give up any guarantees about the behavior or performance of asynchronous code.
* It is inconsistent with the design of `throws`. The result type of asynchronous functions in this model is the future type (or "any `Futurable` type"), rather than the actual returned value. They must always be `await`'ed immediately (hence the postfix syntax) or you'll end up working with futures when you actually care about the result of the asynchronous operation. This becomes a programming-with-futures model rather than an asynchronous-programming model, when many other aspects of the `async` design intentionally push away from thinking about the futures.
* Taking `async` out of the type system would eliminate the ability to do overloading based on `async`. See the prior section on the reasons for overloading on `async`.
* Futures are relatively heavyweight types, and forming one for every async operation has nontrivial costs in both code size and performance. In contrast, deep integration with the type system allows `async` functions to be purpose-built and optimized for efficient suspension. All levels of the Swift compiler and runtime can optimize `async` functions in a manner that would not be possible with future-returning functions.

## Revision history

* Post-review changes:
   * Replaced `await try` with `try await`.
   * Added syntactic-sugar alternative design.
   * Amended the proposal to allow [overloading on `async`](https://github.com/swiftlang/swift-evolution/pull/1392).
* Changes in the second pitch:
	* One can no longer directly overload `async` and non-`async` functions. Overload resolution support remains, however, with additional justification.
	* Added an implicit conversion from a synchronous function to an asynchronous function.
	* Added `await try` ordering restriction to match the `async throws` restriction.
	* Added support for `async` initializers.
	* Added support for synchronous functions satisfying an `async` protocol requirement.
	* Added discussion of `reasync`.
	* Added justification for `await` not implying `try`.
	* Added justification for `async` following the function parameter list.

* Original pitch ([document](https://github.com/DougGregor/swift-evolution/blob/092c05eebb48f6c0603cd268b7eaf455865c64af/proposals/nnnn-async-await.md) and [forum thread](https://forums.swift.org/t/concurrency-asynchronous-functions/41619)).

## Related proposals

In addition to this proposal, there are a number of related proposals covering different aspects of the Swift Concurrency model:

* [Concurrency Interoperability with Objective-C](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0297-concurrency-objc.md): Describes the interaction with Objective-C, especially the relationship between asynchronous Objective-C methods that accept completion handlers and `@objc async` Swift methods.
* [Structured Concurrency](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md): Describes the task structure used by asynchronous calls, the creation of both child tasks and detached tasks, cancellation, prioritization, and other task-management APIs.
* [Actors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md): Describes the actor model, which provides state isolation for concurrent programs

## Acknowledgments

The desire for async/await in Swift has been around for a long time. This proposal draws some inspiration (and most of the Motivation section) from an earlier proposal written by
[Chris Lattner](https://github.com/lattner) and [Joe Groff](https://github.com/jckarter), available [here](https://gist.github.com/lattner/429b9070918248274f25b714dcfc7619). That proposal itself is derived from a proposal written by [Oleg Andreev](https://github.com/oleganza), available [here](https://gist.github.com/oleganza/7342ed829bddd86f740a). It has been significantly rewritten (again), and many details have changed, but the core ideas of asynchronous functions have remained the same.

Efficient implementation is critical for the introduction of asynchronous functions, and Swift Concurrency as a whole. Nate Chandler, Erik Eckstein, Kavon Farvardin, Joe Groff, Chris Lattner, Slava Pestov, and Arnold Schwaighofer all made significant contributions to the implementation of this proposal.
