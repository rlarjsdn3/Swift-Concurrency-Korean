---
description: 비동기 함수를 병렬로 실행하는 구조화된 동시성
---

컴퓨터가 처음 개발되어 사용되던 시기에는 애플리케이션이 명령어의 연속으로 작성되었기 때문에 코드의 흐름을 이해하기가 어려웠습니다. 제어 흐름이 코드 전반에 걸쳐 자유롭게 점프(goto)할 수 있었기 때문입니다. 하지만 오늘날에는 이러한 방식의 프로그래밍은 거의 찾아보기 어렵습니다. 이는 현대 프로그래밍 언어들이 구조적 프로그래밍(Structured Programming)을 채택함으로써 제어 흐름을 더 일관되고 예측 가능하게 만들었기 때문입니다.

```swift
#include <stdio.h>

int main() {
    int score = 75;

    if (score >= 60) {
        printf("Pass\n");
    } else {
        printf("Non-Pass\n")
    }

    return 0;
}
```

구조적 프로그래밍은 코드를 블록 단위로 명확하게 구성하고, 진입점과 종료점이 분명한 제어 흐름을 가지도록 하여 가독성과 유지보수성을 높이려는 프로그래밍 패러다임입니다. 대표적인 구조적 제어 흐름의 예시로는 `if-else` 문을 들 수 있습니다. 이 구문은 코드가 위에서 아래로 순차적으로 실행되는 흐름 속에서, 특정 조건을 만족하면 해당 코드 블록을 실행하고, 그렇지 않으면 `else` 블록의 코드를 실행합니다. 또한, 구조적 프로그래밍에서 코드 블록은 정적 스코핑(Static Ccoping)을 따릅니다. 즉, 변수는 해당 코드 블록이 속한 범위 내에서만 유효하며, 코드 블록을 벗어나면 사라집니다. 이러한 정적 스코핑을 기반으로 한 구조적 프로그래밍은 제어 흐름과 변수의 수명을 명확하게 파악할 수 있게 해줍니다. 

이러한 개념은 오늘날 개발자에게는 너무나도 자연스럽고 당연하게 느껴질 수 있습니다. 하지만 최근의 애플리케이션은 비동기(Asynchronous) 및 동시성(Concurrent) 코드를 포함하는 경우가 많아지면서, 기존의 구조적 프로그래밍 방식으로 이러한 흐름을 표현하기가 쉽지 않았습니다.

```swift
func fetch(with request: URLRequest,
           completionHandler: @escaping (Data?, URLResponse?, (any Error)?) -> Void) -> URLSessionDataTask {
    let task = URLSession.shared.dataTask(with: request, 
                                          completionHandler: completionHandler)
    task.resume()
    return task
}
```

콜백 기반의 비동기 API는 구조적 프로그래밍이 지향하는 흐름을 깨뜨리는 대표적인 예입니다. 비동기 작업이 완료된 후 결과값은 콜백 함수를 통해 전달되는데, 이로 인해 해당 결과값을 누가, 언제, 어떻게 처리할지는 현재 코드 블록 안에서는 파악하기 어렵습니다. 결과적으로, 비동기 처리가 코드 블록의 경계를 넘어 프로젝트 전반에 흩어지게 되며, 이는 유지보수를 어렵게 만들고 코드의 흐름을 직관적으로 이해하기 어렵게 만듭니다. 

이러한 문제를 해결하기 위해 Swift는 구조화된 동시성(Structured Concurrency)을 도입하여, 비동기 작업 역시 코드 블록의 구조 안에서 명확하게 관리할 수 있도록 했습니다. 즉, 변수와 마찬가지로 비동기 작업 역시 구조화된 코드 블록 안에 존재하도록 제한되며, 함수가 종료될 때 모든 하위 작업이 자동으로 정리됩니다. 이 덕분에 누락된 작업 처리 등 예기치 못한 동시성 문제에 대한 걱정 없이, 보다 안전하고 예측 가능한 방식으로 비동기 코드를 작성할 수 있게 되었습니다.


# Parallelism with Structured Concurrency

구조화된 동시성은 작업(Task)들을 계층적으로 구성하여, 각 작업의 생명 주기, 자원 상속, 오류 처리, 취소 전파가 상위 작업의 컨텍스트 안에서 일관되게 이루어질 수 있도록 돕는 동시성 모델입니다. 이 모델은 모든 하위 작업이 상위 작업의 범위 내에 존재하도록 제한함으로써, 작업의 시작과 종료 시점을 명확하게 관리할 수 있게 해줍니다. 

상위 작업은 모든 하위 작업이 완료되기 전까지 종료될 수 없으며, 상위 작업이 취소되면 해당 하위 작업들도 함께 취소됩니다. 또한, 상위 작업의 자원(Task-Local 값, 우선 순위 등)은 하위 작업에 자동으로 상속됩니다. 만약 하위 작업의 우선 순위(Priority)가 상위 작업보다 더 높을 경우, 상위 작업의 우선 순위는 하위 작업의 수준에 맞게 자동으로 승격됩니다. 하위 작업으로 추가된 모든 작업은 병렬(Parellel)로 실행되며, 필요한 경우 `await`을 통해 그 결과를 취합하고 반환할 수 있습니다. 

```
📦 downloadImages(from:)  ← 상위 작업(Parent Task)
 ├── 🧵 downloadImage(from:)   ← 다른 작업의 상위 작업 역할도 하는 하위 작업
 │    ├── 🧵 fetchData(from:)     ← 하위 작업(Child Task)
 │    └── 🧵 fetchMetaData(from:) ← 하위 작업(Child Task)
 └── 🧵 downloadImage(from:)   ← 하위 작업(Child Task)
```

{% hint style="info" %}
**Info** 
구조화된 동시성에서 하위 작업은 상위 작업의 액터(Actor) 컨텍스트를 상속하지 않습니다. 이는 상위 액터가 제한된 수의 스레드에서만 실행되는 실행자(Executor)를 내장하고 있을 수 있기 때문입니다. 만약 하위 작업이 이러한 실행 환경을 그대로 상속받게 되면, 병렬성을 보장해야 하는 구조화된 동시성의 목적이 훼손될 수 있습니다. ~~액터에 관한 자세한 내용은 [Actor]()를 참조하세요. 액터 실행자(Actor Executor)에 관한 자세한 내용은 [Actor Executor]()를 참조하세요.~~
{% endhint %}


우리가 여러 이미지를 다운로드받는 상황이라 가정해봅시다. 지금까지 배운 지식을 총동원하면 아래 예제가 만들어질 수 있습니다.

```swift
func downloadImages(from urls: [String]) async -> [UIImage] {
    let firstImage = await downloadPhoto(from: urls[0])
    let secondImage = await downloadPhoto(from: urls[1])
    let thirdImage = await downloadPhoto(from: urls[2])

    let images = [firstImage, secondImage, thirdImage]
    return images
}
```

하지만, 위 예제는 이미지를 병렬로 다운로드하지 않습니다. `await` 키워드를 사용해 비동기 함수를 호출하면, 작업 자체는 비동기적으로 수행되더라도 작업의 실행은 순차적으로 진행됩니다. 앞선 작업이 완료되기 전에는 다음 작업이 시작되지 않기 때문에, 결국 세 작업이 하나씩 차례대로 처리되는 셈입니다. 그러나 이미지 다운로드 작업은 서로 독립적이므로 굳이 순차적으로 실행할 필요가 없습니다. 만약 세 작업을 병렬로 실행하고, 그 결과를 한 번에 모아 반환할 수 있다면 전체 성능은 훨씬 더 향상될 것입니다. 이제, 이 코드를 어떻게 개선할 수 있을지 살펴보겠습니다.



## Parallel Execution of a Fixed Number of Tasks with async-let

동시에 수행해야 할 작업의 수가 정해져 있다면 `async-let` 바인딩을 사용하는 것이 적합합니다. 이는 간단하게 `let` 키워드 앞에 `async`를 붙이고, 초기화 시점의 `await` 키워드를 생략하는 방식으로 구현할 수 있습니다.  

```swift
func downloadImages(from urls: [String]) async throws -> [UIImage] {
    async let firstImage = downloadImage(from: urls[0])
    async let secondImage = downloadImage(from: urls[1])
    async let thirdImage = downloadImage(from: urls[2])

    let images = await [firstImage, secondImage, thirdImage]
    return images
}
```

위 예제에서 `downloadImage(from:)` 함수는 이전 작업이 완료될 때까지 기다리지 않고, 서로 독립적으로 호출됩니다. 가용 가능한 시스템 자원이 충분하다면 세 작업은 동시에 병렬로 실행됩니다. 이처럼 `async-let`을 사용해 하위 작업을 생성할 때, 해당 함수의 결과를 즉시 기다리지 않기 때문에 `await` 키워드를 붙이지 않습니다. 대신, 해당 작업의 결과가 실제로 필요한 표현식에 도달했을 때, 모든 작업이 완료될 때까지 일시 중단됩니다.

위 예제에서 생성된 하위 작업들은 작업 트리(Task Tree) 구조로 나타낼 수 있으며, `async-let`을 통해 생성된 각 작업은 `downloadImages(from:)` 함수의 하위 작업(Child Task)으로 구성합니다.

```
📦 downloadImages(from:)  ← 상위 작업(Parent Task)
 ├── 🧵 downloadImage(from:)   ← 하위 작업(Child Task)
 ├── 🧵 downloadImage(from:)   ← 하위 작업(Child Task)
 └── 🧵 downloadImage(from:)   ← 하위 작업(Child Task)
```

`downloadImages(from:)` 함수 내부에서 생성된 하위 작업들은 해당 함수의 실행 범위 내에서만 유효하며, 함수 외부의 다른 작업 컨텍스트에는 영향을 주지 않습니다. 만약 이러한 하위 작업의 결과를 `await`으로 기다리지 않은 채 함수가 종료되면 어떻게 될까요? `async-let` 구문을 만나는 즉시 Swift는 하위 작업을 생성하고 실행을 시작하지만, 결과를 명시적으로 기다리지 않고 함수가 끝나면 해당 하위 작업들은 모두 자동으로 취소(cancel)됩니다. 즉, `async-let`으로 시작된 하위 작업으로부터 결과값을 얻으려면 반드시 `await`을 통해 기다려야 합니다. 그렇지 않으면 작업이 완료되기 전에 함수가 종료되어 작업의 결과값을 얻지 못하게 됩니다.



## Parallel Execution of a Dynamic Number of Tasks with TaskGroup

고정된 수의 작업을 병렬로 수행하는 `async-let` 바인딩과 달리, 작업 그룹(TaskGroup)은 작업의 수를 알 수 없는, 동적으로 작업의 수가 변할 수 있는 환경에서 선택할 수 있는 가장 적합한 선택지입니다. 작업그룹을 생성하려면 `withTaskGroup(of:returning:body:)` 함수를 사용해야 합니다. 함수를 호출하면 _body_ 클로저가 즉시 실행되며, 이 클로저는 `TaskGroup<ChildTaskResult>` 인스턴스를 매개변수로 전달받습니다. 해당 인스턴스의 `addTask(priority:operation:)` 메서드로 병렬로 수행하고자 하는 하위 작업을 생성하고 실행할 수 있습니다.

```swift
func downloadImages(from urls: [String]) async throws -> [UIImage] {
    var images: [String: UIImage] = []
    await withTaskGroup(of: UIImage.self, returning: [UIImage].self) { group in
        for url in urls {
            group.addTask { 
                return (url, await downloadImage(from: url))
            }
        }    
        
        for await (url, image) in group {
            images[url] = image
        }
    }
    return images
}
```

`of` 매개변수에는 각 하위 작업이 반환하는 값의 타입을, `returning` 매개변수는 작업 그룹이 최종적으로 반환할 결과 타입을 명시합니다. 위 예제에서 생성된 하위 작업들은 작업 트리(Task Tree) 구조로 나타낼 수 있으며, Swift 컴파일러는 전달된 `urls` 배열의 요소 수만큼 작업을 생성하고, 이 작업들을 모두 해당 작업 그룹의 하위 작업으로 구성합니다. 

```
📦 withTaskGroup { .. }  ← 상위 작업(Parent Task)
 ├── 🧵 downloadImage(from:)  ← 하위 작업(Child Task)
 ├── 🧵 downloadImage(from:)  ← 하위 작업(Child Task)
 ├── 🧵 ...
 └── 🧵 downloadImage(from:)  ← 하위 작업(Child Task)
```

`withTaskGroup(of:returning:body:)` 함수 내부에서 생성된 하위 작업들은 해당 함수의 실행 범위 내에서만 유효하며, 함수 외부의 다른 작업 컨텍스트에는 영향을 주지 않습니다. 

{% hint style="info" %}
**Expriment** 
```swift
func downloadImages(from urls: [String]) async throws -> [String: UIImage] {
    var images: [String: UIImage] = [:]
    await withTaskGroup(of: Void.self) { group in
        for url in urls {
            group.addTask { 
                let image = await downloadImage(from: url)
                images[url] = image
            }
        }    
    }
    return images
}
```
`for-await-in` 반복문을 사용해 이미지 배열에 결과를 수집하는 것이 다소 번거로워 보일 수 있습니다. 그렇다면 왜 굳이 그렇게 하는 걸까요? 하위 작업 내에서 바로 `images` 배열에 이미지를 추가하면 안 되는 걸까요?
{% endhint %}

여기서 주목해야 할 또 하나의 포인트는 바로 `for-await-in` 구문입니다. 이 구문은 비동기 반복문으로, 작업 그룹에 추가된 하위 작업 중 완료된 순서대로 결과를 루프에 전달하여 실행됩니다. 이 비동기 반복문은 각 작업의 결과를 순차적으로 실행하기 때문에, 상위 작업이 데이터 경합(Data Race)의 위험없이 안전하게 각 키-값 쌍을 딕셔너리에 추가할 수 있습니다.

{% hint style="info" %}
**Info** ~~비동기 시퀀스(Asynchronous Sequence)와 비동기 스트림(Asynchronous Stream)에 관한 자세한 내용은 [AsyncSequence/AsyncStream]()를 참조하세요.~~
{% endhint %}

`async-let`과 작업 그룹은 하위 작업을 병렬로 실행한다는 공통점이 있지만, 중요한 차이점이 하나 있습니다. `async-let`은 하위 작업의 결과를 소비하지 않고 함수가 끝나면 해당 하위 작업들은 모두 암묵적으로 취소가 되지만, 작업 그룹에서는 하위 작업들이 단순히 대기될 뿐 취소되지 않습니다.  이는 작업 그룹이 포크-조인(fork-join) 패턴을 따르기 때문입니다. 포크된 작업은 조인 여부와 상관없이 독립적으로 실행되며, 상위 작업은 하위 작업이 모두 완료되어 종료될 때까지 대기합니다.


작업 그룹(TaskGroup)과 동시적 바인딩(async let)은 작업을 병렬로 실행한다는 공통점이 있지만, 중요한 차이점이 있습니다. `async let`의 경우, 하위 작업의 결과를 소비하지 않은 채 함수가 종료되면, 해당 작업이 암묵적으로 취소됩니다. 반면, 작업 그룹에서 생성된 하위 작업은 취소되지 않고 실행 중인 상태로 남아 있으며, 명시적으로 취소 신호를 전파하지 않는 이상 계속 실행됩니다. 이러한 차이를 보이는 이유는 작업 그룹이 포크-조인(Fork-Join) 패턴을 따르기 때문입니다. 포크된 하위 작업들은 (조인 여부와 상관없이) 서로 독립적으로 실행되며, 상위 작업은 하위 작업이 모두 완료될 때까지 대기합니다.


# Types of Task Groups

아래 표는 작업 그룹의 유형을 정리한 것입니다.

| **항목** | **특징** | **비고** |
| -       | -      | -        |
| `withTaskGroup(of:returning:body:)` | 하위 작업의 결과를 수집하여 반환하는 작업 그룹 생성 | 하위 작업이 `for-await-in` 반복문에서 소비되어야 메모리에서 해제 |
| `withThrowingTaskGroup(of:returning:body:)` | 예외를 던질 수 있으며, 하위 작업의 결과를 수집하여 반환하는 작업 그룹 생성 | 하위 작업이 `for-try-await-in` 반복문에서 소비되어야 메모리에서 해제 |
| `withDiscardingTaskGroup(returning:body:)` | 하위 작업의 결과를 수집할 필요가 없는 작업 그룹 생성 | 하위 작업이 완료되면 즉시 메모리에서 해제 |
| `withThrowingDiscardingTaskGroup(returning:body:)` | 예외를 던질 수 있으며, 하위 작업의 결과를 수집할 필요가 없는 작업 그룹 생성 | 하위 작업이 완료되면 즉시 메모리에서 해제 |

`withDiscardingTaskGroup(returning:body:)`과 `withThrowingDiscardingTaskGroup(returning:body:)` 함수는 결과를 반환할 필요가 없거나, 결과를 무시해도 되는 하위 작업을 병렬로 실행할 때 사용됩니다. 이 함수들은 하위 작업이 완료되는 즉시 해당 작업을 메모리에서 해제하도록 설계되어 있어, 일반 작업 그룹에 비해 메모리 사용 면에서 효율적이며 성능 또한 더 우수합니다.

```swift
func cacheImages(_ images: [UIImage]) async throws {
    try await withThrowingDiscardingTaskGroup(returning: Void.self) { group in
        for image in images {
            group.addTaskUnlessCancelled {
                try await cache(image)
            }
        }
    }
}
```
