---
description: 특정 작업 컨텍스트 내에 바인딩하고 읽을 수 있는 값
---

Swift Concurrency가 도입되기 이전에는, 실행 환경 메타데이터를 각 스레드나 큐에서 독립적으로 실행되는 작업마다 고유하게 유지하기 위해 스레드-로컬(Thread-Local)이나 큐 전용 값(Queue-Specific Value)을 사용하는 방식이 일반적이었습니다. 그러나, Swift Concurrency의 도입과 함께 동시성 모델이 스레드 기반에서 작업 기반으로 전환되면서, 이러한 방법은 더 이상 적합하지 않게 되었습니다. Swift Concurrency에서는 액터(Actor)나 실행자(Executor)가 비동기 코드를 실행하지만, 해당 작업이 어떤 스레드나 큐에서 실행될지는 보장되지 않기 때문입니다.

```swift
let myQueue = DispatchQueue.global()
let traceID = DispatchSpecificKey<String>()
myQueue.setSpecific(key: traceID, value: "123456789")

myQueue.async {
    Task {
        if let queueName = DispatchQueue.getSpecific(key: traceID) {
            print("Currently on queue: \(queueName)")
        } else {
            print("Not on the expected queue")
        }
    }
}
// Prints "Not on the expected queue"
```

위 예제에서도 볼 수 있듯이, `DispatchQueue.getSpecific(key:)`를 호출하더라도 기대했던 큐 전용 값을 가져오지 못하는 상황이 발생할 수 있습니다. 이는 작업(Task)이 실행되는 스레드나 큐가 Swift 런타임에 의해 동적으로 관리되기 때문입니다. 즉, 기존의 스레드-로컬이나 큐 전용 값 방식으로는 메타데이터를 안정적으로 전달하는 데 한계가 있으며, 이를 보완하기 위해 태스크-로컬(Task-Local)이라는 새로운 메커니즘이 도입되었습니다.

작업은 우선순위(priority)나 취소 상태와 같은 작업 컨텍스트를 가지고 있으며, 상위 작업(Parent Task)의 컨텍스트는 하위 작업(Child Task)에게 자연스럽게 전파됩니다. 이처럼 작업은 기본적으로 컨텍스트를 상속하는 구조를 가지고 있으며, 이 구조를 확장하여 개발자가 추가적인 메타데이터도 함께 전달할 수 있도록 한 것이 바로 태스크-로컬입니다. 

 태스크-로컬은 작업 컨텍스트 간에 메타데이터(이하 '값')를 안전하고 일관되게 전달합니다. 태스크-로컬 값은 작업 생성이 생성될 때 암시적으로 전달되며, 해당 작업이 생성하는 모든 하위 작업에서도 읽을 수 있습니다. 태스크-로컬 값을 특정 스코프에 바인딩하면, 해당 스코프 내의 동기 함수(Synchronous Function), 비동기 함수(Asynchronous Function), `Task`, 작업 그룹(TaskGroup) 그리고 `async-let` 구문에서도 바인딩된 값을 읽을 수 있습니다. 이 값은 바인딩된 스코프 내의 작업에만 접근할 수 있으며, 다른 작업과 임의로 공유되지 않습니다.


{% hint style="info" %}
**Info** 

실제 코드에서는 태스크-로컬(task-local)을 신중하게 사용해야 합니다. 단순히 매개변수로 전달할 수 있는 값을 태스크-로컬로 사용하는 것은 피해야 합니다. 태스크-로컬 값은 함수 호출 결과에 직접적인 영향을 주지 않고, 부수적인 구성 정보에만 영향을 미치는 메타데이터에 한정해 사용하는 것이 바람직합니다.

만약 어떤 값을 매개변수로 직접 전달할지, 태스크-로컬로 전달할지 확신이 서지 않는 경우에는 매개변수로 전달하는 방법을 선택하는 것이 좋습니다. 태스크-로컬은 주로 추적 ID(trace ID), 인증 토큰(authentication token)과 같은 메타데이터를 전달하기 위해 설계되었다는 점을 기억하세요.

또한, 태스크-로컬을 매개변수 전달을 대체하는 수단으로 남용해서는 안 됩니다. 숨겨진 인자(hidden argument)를 통해 값을 전달하면 코드의 동작을 이해하고 추적하기 어려워질 수 있습니다. 명시적으로 매개변수를 전달하는 방식이 코드의 가독성과 유지보수성 측면에서 훨씬 더 우수합니다.
{% endhint %}


# Task-Local Declarations

태스크-로컬 값은 반드시 `@TaskLocal` 프로퍼티 래퍼를 적용한 정적 프로퍼티(static) 또는 전역 변수로 선언해야 합니다.

```swift
enum MyLibrary {
  @TaskLocal
  static var requestID: String?
}

// Global task local properties are supported since Swift 6.0:
@TaskLocal
var contextualNumber: Int = 12
```

태스크-로컬 값을 선언할 때는 반드시 기본값을 할당해야 합니다. 만약 옵셔널 타입으로 선언한 경우에는 기본값으로 nil이 자동으로 할당됩니다.

각 태스크-로컬 선언은 독립적인 저장소를 나타냅니다. 동일해 보이는 선언이라 하더라도, 서로 다른 태스크-로컬 변수를 통해 저장된 값은 서로 간섭하거나 영향을 주지 않습니다. 태스크-로컬 값은 선언된 지점에 따라 고유하게 식별되며, 이를 통해 서로 다른 모듈 간의 의도치 않은 충돌을 방지할 수 있습니다. 또한, 태스크-로컬 값은 여러 작업에서 동시에 접근될 수 있으므로, 해당 타입은 반드시 `Sendable`해야 합니다. `Sendable`을 만족하지 않는 타입을 태스크-로컬 값으로 바인딩할 경우, 서로 다른 작업 간에 상태가 공유되어 예기치 않은 동작이 발생할 수 있기 때문입니다.

```swift
class Account { }

enum MyLibrary {
    @TaskLocal static var account: Account? // 🔴 error: 'account' is not concurrency-safe because non-'Sendable' type
}
```

모든 작업(Task)은 고유한 태스크-로컬(task-local) 스택(stack)을 가지고 있으며, 이 스택에 필요한 값을 저장합니다. 태스크-로컬에는 값 타입(value type)과 참조 타입(reference type) 모두 저장할 수 있으며, 각 타입에 따라 저장 방식이 다르게 동작합니다. 값 타입은 복사(copy)되어 작업의 태스크-로컬 스택에 저장됩니다. 반면, 참조 타입은 참조(retain)되어 저장되며, 값을 바인딩할 때마다 참조 횟수(retain count)가 증가합니다. 이 덕분에, 참조 인스턴스는 바인딩되어 있는 동안 의도치 않게 해제되지 않고, 지속적으로 유지되도록 합니다.


# Reading Task-Local Values 

태스크-로컬 값은 비동기 함수에서 호출되는 동기 함수를 포함하여, 작업 컨텍스트 내에서 실행되는 모든 함수에서 읽을 수 있습니다. 태스크-로컬 값을 읽을 때, 현재 작업-에 값이 바인딩되어 있으면 해당 바인딩된 값을 반환하고, 바인딩이 없는 경우에는 기본값을 반환합니다.

```swift
func syncFunction() {
    guard let requestID = MyLibrary.requestID else {
        // ...
    }
}
```

태스크-로컬 값은 일반적인 정적 프로퍼티를 읽는 것과 마찬가지로, 특별한 추가 작업 없이 간단하게 읽을 수 있습니다.

{% hint style="warning" %}
**Warning** 

태스크-로컬 값을 읽을 때는, 태스크-로컬 스택의 상단부터 하단까지 검색하여 값을 찾습니다. 값이 발견되지 않으면 스택 끝까지 탐색을 계속합니다. 이 과정은 일반적인 정적 프로퍼티를 읽는 것보다 더 높은 비용이 발생합니다. 따라서, 태스크-로컬 값은 꼭 필요한 경우에만 신중하게 사용해야 합니다. 예를 들어, `for` 문 안에서 동일한 값을 반복해서 읽는 대신, 루프 시작 전에 한 번만 읽어 저장한 후 사용하는 방식으로 최적화하는 것이 좋습니다.

```swift
func requestBookOnLoan(_ books: [Book]) async {
    for book in books {
        // 🟠 warning: 매 루프마다 requestID를 태스크-로컬에서 조회
        if let requestId = MyLibrary.requestID {
            // ...
        }
    }
}

```
{% endhint %}


## Optimizing Task-Local Value Reads

구조화된 동시성에서는 작업 트리(Task Tree)가 각 작업(Task)을 연결 리스트(Linked List) 형태로 연결하여 구성됩니다. 따라서, 하위 작업은 상위 작업을 참조할 수 있으며, 이를 통해 태스크-로컬 값을 명시적으로 복사하지 않고도 바인딩할 수 있습니다. 구조화된 동시성(Structured Concurrency)에서 태스크-로컬 값을 읽을 때는 다음과 같은 과정을 따릅니다.

1. 먼저 현재 작업에서 태스크-로컬 스택의 상단부터 하단까지 검색하여, 지정한 키가 존재하는지 확인합니다.

2. 키가 존재하지 않으면 상위 작업으로 이동하여 동일한 방법으로 키를 조회합니다.

3. 이 과정을 상위 작업이 더 이상 없을 때까지 반복합니다.

4. 최상위 작업(Root Task)까지 검색해도 키를 찾지 못하면, 최종적으로 해당 키에 정의된 기본값(default value)을 반환합니다.

```
[Task.detached] ()
  \ 
  |[child-task-1] (id:10)
  |   \
  |   |[child-task-1-1] (id:20)
  |[child-task-2] (name: "alice")
```

예를 들어, `child-task-2` 작업에서 `name` 키를 조회하는 경우, 현재 작업 컨텍스트에 "alice"가 저장되어 있으므로 즉시 "alice"를 반환합니다. 반면, `child-task-1-1` 작업에서 `name` 키를 조회하려고 하면, 먼저 자신(`child-task-1-1`)의 태스크-로컬 스택 상단부터 하단까지 검색하여 지정한 키가 존재하는지 확인합니다. 키가 발견되지 않으면 상위 작업(`child-task-1`)으로 이동하여 다시 검색하고, 이후 최상위 작업(`Task.detached`)까지 검색을 이어갑니다. 그러나 이들 작업 어디에서도 `name` 키가 바인딩되어 있지 않기 때문에, 최종적으로 기본값이 반환됩니다.

더 나아가서, 바인딩된 태스크-로컬 값은 변경할 수 없는(immutable) 특성을 가지므로, 한 번 설정된 값은 이후에 생성된 하위 작업에서도 변경되지 않습니다. 이로 인해 상위 작업이 새로운 태스크-로컬 값을 바인딩하지 않는 경우, 하위 작업은 해당 상위 작업을 건너뛰고 그 위의 작업으로 바로 이동하여 키를 검색함으로써 검색 과정을 최적화할 수 있습니다.

```
[detached] ()
  \ 
   [child-task-1] (requestID:10)
    \
    |[child-task-2] ()
     \
     |[child-task-3] ()
      \
      |[child-task-4] ()
```

예를 들어, `child-task-3` 작업을 생성할 때, 상위 작업인 `child-task-2`가 태스크-로컬 스택을 가지고 있지 않음을 확인할 수 있습니다. 이 경우, `child-task-3`는 직접 태스크-로컬 값을 바인딩하고 있는 상위 작업인 `child-task-1`을 가리키도록 최적화할 수 있습니다. 그 결과, `child-task-3`가 `requestID` 키를 조회할 때 단 한 번의 점프(hop)만으로 `child-task-1`에 도달하여 값을 가져올 수 있습니다.

이처럼, 태스크-로컬은 필요한 값을 검색할 때, 태스크-로컬 스택 전체를 복사하는 방식 대신, 링크드 리스트(Linked List) 기반 검색 방식을 채택했습니다. 이는 태스크-로컬 값을 실제로 읽는 작업(Task)이 전체 작업 중 일부에 불과하기 때문입니다. 일반적으로 하나의 최상위 작업이 태스크-로컬 값을 바인딩하고, 그 생명 주기 동안 수백 개의 하위 작업이 생성됩니다. 이들 하위 작업 중 일부만 태스크-로컬 값에 접근하며, 많은 작업은 값을 읽지 않습니다.

이러한 특성으로 인해, 모든 하위 작업에 대해 태스크-로컬 스택을 복사하는 것은 메모리 사용량과 작업 생성 비용 측면에서 비효율적입니다. 결과적으로, 값 검색 시 약간의 비용을 감수하는 편이 전체 시스템 성능 측면에서 훨씬 유리합니다. 또한, 이러한 구조를 고려할 때, 태스크-로컬 값을 바인딩하지 않은 ’빈 작업(empty task)’을 검색 과정에서 건너뛰는 최적화는 검색 비용을 낮추고, 검색 성능을 더욱 향상시키는 데 실질적인 이점을 제공합니다.


{% hint style="info" %}
**Info**

구조화되지 않은 동시성(Unstructured Concurrency)은 작업 간에 상위-하위 관계를 명확히 구성하는 구조화된 동시성과 달리, 작업 간 계층 구조를 형성하지 않습니다. 구조화되지 않은 작업은 특정 스코프에 생명 주기가 제한되는 구조화된 작업과 달리, 스코프와 무관하게 독립적으로 실행되며, 생성한 작업보다 더 오래 살아남을 수 있습니다.

예를 들어, `Task` 또는 `Task.detached`를 통해 생성된 작업은 부모 작업의 하위 작업이 아니기 때문에, 기존의 링크드 리스트 기반 태스크-로컬 검색 방식은 사용할 수 없습니다. 대신, 구조화되지 않은 작업을 생성할 때는, 생성 시점의 태스크-로컬 스택이 새 작업으로 복사됩니다.

구조화되지 않은 작업은 복사된 태스크-로컬 스택만을 참조하여 값을 검색합니다. 작업에서 태스크-로컬 값을 읽을 때는, 자신의 스택 상단부터 하단까지 순차적으로 키를 검색하고, 스택을 모두 탐색했음에도 값을 찾지 못하면 기본값을 반환합니다.
{% endhint %}



# Binding Task-Local Values

값을 바인딩(binding)하는 것은 태스크-로컬의 핵심적인 기능입니다. 태스크-로컬은 스레드-로컬과 달리, 값을 직접 설정(set)할 수 없습니다. 대신, 주어진 스코프 내에서만 값을 바인딩하며, 이 스코프 내에서 생성된 모든 (하위) 작업에서 바인딩된 값을 사용할 수 있도록 합니다. 스코프를 벗어나면 바인딩된 값에 더 이상 접근할 수 없으며, 기본값만을 읽을 수 있습니다. 이 메커니즘은 값이 설정된 후 방치되어 메모리 누수가 발생하거나, 예상치 못한 코드 경로에서 잘못된 값을 읽어 디버깅이 어려워지는 문제를 방지하기 위한 것입니다.

```swift
func asyncFunction() async {
    await MyLibrary.$requestID.withValue("123") {
        print("requestID:", MyLibrary.requestID) // 🔵 "123"
        syncFunction()
        
        async let loans = requestBooksOnLoan() // 🔵 "123"
        
        await withTaskGroup(of: Void.self) { group in
            group.addTask {
                await requestBooksOnLoan() // 🔵 "123"
            }
            return await group.next()!
        }
        
        Task {
            print("requestID:", MyLibrary.requestID) // 🔵 "123"
        }
        
        Task.detached {
            print("requestID:", MyLibrary.requestID) // 🔵 nil
        }
    }
}
```

동일한 작업 컨텍스트 내에서 동일한 키를 여러 번 중첩하여 바인딩하는 것도 가능합니다. 이 경우, 값을 조회할 때 현재 작업 컨텍스트에서 가장 가까운(가장 최근에 바인딩된) 스코프에서 값을 가져옵니다.

```swift
func asyncFunction() async {
    await MyLibrary.$requestID.withValue("123") {
        Task {
            await MyLibrary.$requestID.withValue("456") {
                await withTaskGroup(of: Void.self) { group in
                    group.addTask {
                        await requestBooksOnLoan() // 🔵 "456"
                    }
                    return await group.next()!
                }
            }
        }
    }
}
```

태스크-로컬 값은 바인딩이 중첩되어 있더라도, 동일한 작업 컨텍스트 내에서 호출되는 모든 함수에서 바인딩된 값을 읽을 수 있습니다. 예를 들어, 하나의 비동기 함수에서 태스크-로컬 값을 바인딩하고 다른 비동기 함수를 호출한 후, 해당 비동기 함수가 다시 동기 함수를 호출하는 경우에도, 호출 경로 상의 모든 함수에서 동일한 태스크-로컬 값을 읽을 수 있습니다.

```swift
func outer() async -> String? {
  await MyLibrary.$requestID.withValue("123") { 
    MyLibrary.requestID // 🔵 "123"
    return middle()
  }
}

func middle() async -> String? {
  MyLibrary.requestID // 🔵 "123"
  return inner()
}

func inner() -> String? { // synchronous function
  return MyLibrary.requestID // 🔵 "123"
}
```

이 예제에서는 `outer()` 함수에서 `requestID`를 “1234”로 바인딩한 이후, `middle()`과 `inner()` 함수에서도 동일한 `requestID` 값을 읽을 수 있습니다. 비동기 함수와 동기 함수가 혼합된 호출 경로에서도 바인딩된 태스크-로컬 값은 일관되게 유지됩니다.

태스크-로컬(task-local) 값에 아무런 값을 바인딩하지 않고 읽으면, 태스크-로컬에 설정된 기본값(default value)이 반환됩니다. 태스크-로컬을 옵셔널 타입(optional type)으로 선언한 경우에는 기본값으로 nil이 반환됩니다. 아울러, 기본값은 현재 작업이나 상위 작업 컨텍스트에서 해당 키에 대해 값이 바인딩되어 있지 않거나, 호출 스택(call stack) 상에서 비동기 함수가 없는 동기 함수에서 태스크-로컬 값을 읽으려 할 때 반환될 수 있습니다.

```swift
func asyncFunction() async {
    await MyLibrary.$requestID.withValue("123") {
    }
    
    print("requestID:", MyLibrary.requestID) // 🔵 nil
}
```

작업에서 태스크-로컬 값을 바인딩하면, 해당 작업의 태스크-로컬 스택에 값이 푸시(push)됩니다. 바인딩이 중첩되면, 스택에 여러 값이 순차적으로 푸시될 수 있습니다. 작업이 태스크-로컬 값을 읽을 때는, 스택의 상단부터 하단까지 순차적으로 검색하여 지정한 키에 해당하는 값을 찾습니다. 또한, `withValue` 스코프를 벗어날 때, 해당 스코프 안에서 스택에 푸시된 값은 자동으로 팝(pop)되어 정리됩니다. 이처럼, 태스크-로컬은 `withValue` 스코프 내에서 생성된 모든 하위 작업이 완전히 종료되기 전까지 스코프를 벗어나지 않습니다. 이 덕분에, 태스크-로컬은 전역 저장 방식이 아니라 스택 기반 저장 방식으로 값을 안전하고 효율적으로 저장하고 검색할 수 있습니다.

```swift
func asyncFunction() async {
    await MyLibrary.$requestID.withValue("123") {
        // Binding: Push (requestID: "123") onto the task-local stack

        Task {
            // Copying: Inherit the current task-local stack [(requestID: "123")]

            await MyLibrary.$userID.withValue("abc") {
                // Binding: Push (userID: "abc") onto the task-local stack

                Task {
                    // Copying: Inherit the current task-local stack [(requestID: "123"), (userID: "abc")]
                    
                    let userID = MyLibraryID.userID       // 🔵 "abc"
                    let requestID = MyLibraryID.requestID // 🔵 "123"
                }
            }
        }
    }
}

Task {
    await asyncFunction()
}
```


{% hint style="info" %}
**Info**

태스크-로컬 값은 작업 실행 중에 변경할 수 없습니다. 이는 작업이 태스크-로컬 값에 대한 데이터 경합(data race)을 걱정할 필요 없이, 언제든지 예측 가능한 값을 안정적으로 읽을 수 있도록 보장하기 위함입니다. 작업이 실행 중에 태스크-로컬 값을 변경할 수 있다면, 동시에 실행되는 자식 작업들이 동일한 태스크-로컬 값을 읽을 때 서로 다른 결과를 얻을 수 있습니다. 따라서, 태스크-로컬 값은 작업 생성 시점에만 초기화되며, 이후에는 값을 변경할 수 없습니다. 새로운 값을 사용하고자 할 경우, 새로운 스코프에서 값을 다시 바인딩해야 합니다.
{% endhint %}



## Binding values for the duration of a child-task

`withValue(_:operation:)`을 통해 바인딩된 태스크-로컬 값은 단순히 클로저가 종료될 때 제거되는 것이 아닙니다. `withValue` 스코프 내에서 생성된 모든 하위 작업이 완료될 때까지 값이 유지됩니다.

구조화된 동시성에서는 작업의 생명 주기가 스코프에 명확히 국한되며, 모든 작업이 완료되어야만 해당 스코프를 벗어날 수 있습니다. 하위 작업 중 하나가 예외를 던지는 경우에도, Swift는 나머지 모든 하위 작업을 자동으로 취소하고, 이들이 종료될 때까지 기다린 후 스코프를 종료합니다. 이러한 구조 덕분에, 상위 작업이 하위 작업보다 먼저 종료하면서 태스크-로컬 값을 해제하는 일이 발생하지 않습니다. 따라서, 하위 작업은 상위 작업이 바인딩한 태스크-로컬 값을 안정적으로 읽고 사용할 수 있습니다.

즉, 구조화된 동시성에서는 하위 작업이 `withValue` 스코프에서 바인딩된 태스크-로컬 값을 안전하게 읽을 수 있으며, 상위 작업은 스코프 종료 전까지 해당 값을 유지합니다. `withValue` 스코프를 벗어나기 전에, 생성된 모든 하위 작업은 반드시 완료되어야 하며, 이 덕분에, 스코프를 벗어날 때는 해당 값을 읽는 작업이 존재하지 않음이 보장됩니다. 덕분에, Swift는 태스크-로컬 스택에서 값을 안전하게 제거(pop)할 수 있습니다.

이를 통해, 태스크-로컬 값을 안전하고 예측 가능하게 사용할 수 있으며, 값이 조기에 해제되거나 잘못된 값을 참조하는 위험 또한 효과적으로 방지할 수 있습니다.


## Task-local value and tasks which outlive their scope

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}

<!---->
<!--`Task`는 실행 컨텍스트, 작업 우선순위, 태스크-로컬 값 등 생성하는 작업의 모든 작업 컨텍스트를 상속받습니다. 비록 구조화되지 않은 작업(unstructured task)이라 하더라도, 새로운 비동기 작업으로 값을 복사하는 방식으로 태스크-로컬 값을 상속합니다.-->
<!---->
<!--때때로 작업의 결과를 기다리지 않고 “비동기적으로 작업을 계속 진행”해야 할 필요가 있을 수 있습니다.-->
<!--현재는 detach 연산이 존재하여, 구조화된 동시성(Structured Concurrency)의 영역을 완전히 벗어나 호출 스코프를 초월해 살아남을 수 있도록 합니다. 그러나 이는 태스크-로컬 값(task-local values)과 충돌하는 문제를 일으킬 수 있습니다. 태스크-로컬 값은 전적으로 자식 작업(child tasks)이라는 구조화된 개념에 맞추어 설계되고 최적화되어 있기 때문입니다. 또한, detached task의 목적 자체가 생성된 컨텍스트로부터 “완전히 분리(detach)“되어 “깨끗한 상태에서 시작하는 것”입니다. 다시 말해, detached task는 호출 컨텍스트로부터 태스크-로컬 값을 상속하지 않으며(!), 이는 detached task가 호출 컨텍스트의 실행 컨텍스트나 우선순위를 상속하지 않는 것과 마찬가지입니다.-->
<!---->
<!--```swift-->
<!--// priority == .background-->
<!--await Lib.$tea.withValue(.green) { -->
<!--  async { -->
<!--    await Task.sleep(10_000)-->
<!--    // assert(Task.currentPriority == .background) // inherited from creating task (!)-->
<!--    assert(Lib.tea == .green)                      // inherited from creating task-->
<!--    print("inside")-->
<!--  }-->
<!--}-->
<!---->
<!--print("outside")-->
<!--```-->
<!---->
<!--예상한 대로, detached task는 생성한 작업의 모든 컨텍스트 정보를 완전히 버리기 때문에, .sugar와 같은 선호 설정(preference)이 자동으로 전달되지 않습니다. 이는 detached task가 우선순위(priority) 또한 자동으로 상속받지 않는 것과 유사합니다.-->
<!---->
<!---->
<!--```swift-->
<!--await Lib.$sugar.withValue(.noSugar) { -->
<!--  assert(Lib.sugar == .noSugar)-->
<!--  -->
<!--  detach { // completely detaches from enclosing context!-->
<!--    assert(Lib.sugar == .noPreference) // no preference was inherited; it's a detached task!-->
<!--  }-->
<!--  -->
<!--  assert(Lib.sugar == .noSugar)-->
<!--} -->
<!--```-->
<!---->
<!--한편, 새로 제안된 async 연산(이름은 아직 확정되지 않았으며, 아마 send가 될 수도 있습니다(?))은 생성한 작업의 다음 속성들을 모두 상속받습니다: 실행 컨텍스트(execution context), 태스크 우선순위(task priority), 그리고 태스크-로컬 값(task-local values).-->
<!---->
<!--```swift-->
<!--// priority == .background-->
<!--await Lib.$tea.withValue(.green) { -->
<!--  async { -->
<!--    await Task.sleep(10_000)-->
<!--    // assert(Task.currentPriority == .background) // inherited from creating task (!)-->
<!--    assert(Lib.tea == .green)                      // inherited from creating task-->
<!--    print("inside")-->
<!--  }-->
<!--}-->
<!---->
<!--print("outside")-->
<!--```-->
<!---->



## Binding task-local values from synchronous functions

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}

태스크-로컬(Task-local) 값은 동기 함수(synchronous functions)에서도 정상적으로 읽거나 바인딩할 수 있습니다. 동기 함수에서 태스크-로컬 값을 사용할 때도 비동기 함수와 동일한 API를 사용하지만, 동기 함수 내에서 withValue를 호출할 경우, 전달하는 클로저는 반드시 동기 클로저여야 합니다. (이는 비동기 함수 호출 규칙과 일관됩니다.)

가끔 동기 withValue가 Task가 존재하지 않는 컨텍스트에서 호출될 수도 있습니다. 일반적인 Swift 프로그램에서는 대부분의 호출이 어떤 비동기 함수로부터 시작되지만, 예를 들어 C 라이브러리나 자체 스레드를 관리하는 라이브러리로부터 진입하는 경우, 사용할 수 있는 Task가 존재하지 않을 수 있습니다. 이러한 경우에도 태스크-로컬 API는 정상적으로 동작합니다.

Swift는 Task가 없는(task-less) 컨텍스트에서도 Task-local 값을 지원하기 위해, 특별한 스레드-로컬(thread-local) 메커니즘을 사용해 가상의 동적 스코프(dynamic scope) 를 시뮬레이션합니다. 즉, 현재 코드가 비동기 Task에 의해 관리되지 않는 스레드에서 실행되더라도, 일반적인 방식으로 태스크-로컬 값을 저장하고 읽을 수 있습니다.

```swift
func enter() {
    Example.$traceID.withValue(1234) {
        print("traceID: \(Example.traceID)") 
    }
}

// 1) Called from outside any Task
enter()

// 2) Called from inside a Task
Task {
    enter()
}
```

위 예제처럼, 호출이 Task 컨텍스트 안이든 밖이든 관계없이, 항상 태스크-로컬 값을 안정적으로 읽을 수 있습니다.

만약 현재 실행 중인 코드가 Task 컨텍스트 바깥이라면, Swift는 태스크-로컬 저장소를 스레드-로컬(thread-local) 스토리지에 기록합니다. 이 방식은 일반적인 비동기 컨텍스트에서는 사용되지 않지만, 동기 코드나 레거시 환경과의 상호 운용성(interoperability)을 위해 지원됩니다.

다만, 이러한 스레드-로컬 저장소는 새로 생성된 비구조적 스레드(예: pthread)로는 자동 전파되지 않기 때문에, 별도의 Task-local 값을 명시적으로 설정해야 할 수 있습니다.

또한, Task가 없는 컨텍스트에서 사용할 수 있는 Swift Concurrency API는 제한적입니다. 대표적으로 async {} 호출은 가능하며, 이때 현재 존재하는 Task-local 값 스냅샷이 복사되어 새로운 작업(Task)로 전달됩니다.

```swift
func synchronous() {
    withUnsafeCurrentTask { task in
        assert(task == nil) // No Task is available!
    }

    Example.$local.withValue(13) {
        other()
    }
}

func other() {
    print(Example.local) // 13
}
```

이러한 설계 덕분에, 코드가 Task 컨텍스트 안이든 밖이든, Swift 런타임은 일관된 방식으로 태스크-로컬 값을 관리할 수 있으며, 레거시 시스템과의 뛰어난 상호 운용성도 확보할 수 있습니다.



<!--- 만약 현재 실행 중인 코드가 Task 컨텍스트 바깥, 즉 비동기 작업이 전혀 시작되지 않은 상태라면 어떻게 될까요? Swift의 @TaskLocal은 비동기 함수가 없는 동기 함수에서도 사용할 수 있으며, 이러한 경우에는 스레드 로컬(thread-local) 저장소를 사용하여 값을 저장합니다. 즉, Task 컨텍스트 외부(예: 메인 스레드나 비동기 Task가 아닌 곳)에서 @TaskLocal 값을 설정하면, Swift 런타임은 해당 값을 스레드 로컬 저장소에 저장합니다. -->
<!---->
<!--```swift-->
<!---->
<!--```-->
<!---->
<!--- 이렇게 하면 호출 컨텍스트를 신경 쓰지 않고도 태스크 로컬 값을 안정적으로 바인딩하고 읽을 수 있습니다. 그러나 이러한 스레드 로컬 저장소는 새로운 비구조적 스레드(예: pthread)에서는 자동으로 전파되지 않습니다. 따라서 이러한 스레드에서는 @TaskLocal 값을 명시적으로 설정해야 합니다. ￼이러한 동작은 Swift의 구조화된 동시성 모델과 일관성을 유지하면서도, 비동기 컨텍스트가 아닌 경우에도 @TaskLocal API가 정상적으로 작동하도록 보장해 줍니다.-->
<!---->
<!--- 이는 작업 내에서 호출된다고 보장되지 않는 동기 함수에서 특히 유용합니다. Swift에서는 코드가 Task의 컨텍스트 밖—즉, 동기 함수나 Swift의 동시성 시스템에 의해 관리되지 않는 스레드—에서 실행될 경우, Task-Local 값은 스레드-로컬(thread-local) 저장소를 사용하도록 대체(fallback) 됩니다. 즉, 아래와 같은 예제처럼 호출 컨텍스트를 신경 쓰지 않고도, 태스크-로컬 값을 안정적으로 바인딩하고 읽을 수 있습니다.-->

<!---->
<!--Task-local은 Task가 존재하지 않는 문맥(context) 에서도 정상적으로 동작할 수 있습니다.-->
<!--이 경우, “동적 스코프(dynamic scope)” 처럼 동작하며,-->
<!--Task 체인을 형성하는 대신, 하나의 스레드-로컬(thread-local) 변수를 사용해 Task-local 저장소를 관리합니다.-->
<!---->
<!--```-->
<!--func synchronous() {-->
<!--  withUnsafeCurrentTask { task in -->
<!--    assert(task == nil) // no task is available!-->
<!--  }-->
<!--  -->
<!--  Example.$local.withValue(13) { -->
<!--    other()-->
<!--  }-->
<!--}-->
<!---->
<!--func other() {-->
<!--  print(Example.local) // 13, works as expected-->
<!--}-->
<!--```-->
<!---->
<!--이러한 Task가 없는 동기 함수(synchronous function) 에서 호출할 수 있는 유일한 Swift Concurrency API는 async{} 입니다.-->
<!--async{}는 호출 시점에 존재하는 Task-local 값들을 일반적인 방식대로 복사(copy) 하여 새로운 작업(Task)에 전달합니다.-->
<!---->
<!--이 덕분에, 레거시 라이브러리(legacy library) 와 함께 사용할 때도 뛰어난 상호 운용성(interoperability)을 확보할 수 있습니다.-->
<!--즉, 현재 코드가 Swift Concurrency 런타임이 소유한 스레드에서 실행되고 있는지 여부에 상관없이, 항상 예상한 대로 동작이 유지됩니다.-->






<!---->
<!---->
<!------->
<!---->
<!--1. 언제 태스크-로컬 스택을 복사하고, 복사하지 않는가?-->
<!---->
<!--✅ 핵심 결론부터 말하면:-->
<!--    •    “자식 Task” (예: async let, TaskGroup.addTask) ➔ 복사하지 않고, 링크드 리스트로 상위 작업을 참조하며 값을 조회합니다. (★ 복사 안 함)-->
<!--    •    “비구조화 Task” (예: Task {}) ➔ 복사해서 자신만의 태스크-로컬 저장소를 갖습니다. (★ 복사함)-->
<!--    -->
<!--    -->
<!--2. (구조화된 동시성에서) 태스크-로컬 스택에서 값을 찾을 수 없을 때 상위 작업-하위 작업 간, 링크드 리스트를 통해 상위 작업으로 거슬러 올라가는가?-->
<!---->
<!--✅ 네, 맞습니다.-->
<!--Task-local 값 조회 과정은 이렇게 작동합니다:-->
<!--    1.    **현재 작업(Task)**의 태스크-로컬 스택을 먼저 검색합니다.-->
<!--    2.    없으면, 상위 작업(parent task)을 따라 링크드 리스트로 계속 거슬러 올라갑니다.-->
<!--    3.    상위 작업에서도 못 찾으면, 또 그 상위 작업을 올라가고…-->
<!--    4.    최종 루트 작업(예: detached 작업)까지 가도 없으면 ➔ *기본값(default value)*을 반환합니다.-->
<!---->
<!-->     “링크드 리스트를 타고 상위 작업으로 거슬러 올라가는 동작은, 구조화된 동시성(Structured Concurrency) 안에서만 일어납니다.”-->
<!---->
<!---->
<!----------->
<!---->
<!--🧩 1. 구조화된 작업 (structured tasks) – 링크드 리스트를 타고 상위 작업으로 올라간다-->
<!---->
<!--```-->
<!--[parent task] (Task-local: requestID = 1234)-->
<!--    |-->
<!--    +-- [child task 1] (no new task-local)-->
<!--    |      |-->
<!--    |      +-- [grandchild task 1-1] (no new task-local)-->
<!--    |-->
<!--    +-- [child task 2] (Task-local: userID = "Alice")-->
<!--```-->
<!--    •    child task 1이 requestID를 읽으면?-->
<!--➔ 자신의 스택에 없으니 → 부모인 parent task에서 찾음 → 1234 반환-->
<!--    •    grandchild task 1-1이 requestID를 읽으면?-->
<!--➔ 자기 → 부모(child 1) → 조부모(parent task) 순으로 올라가면서 찾음-->
<!--    •    child task 2가 requestID를 읽으면?-->
<!--➔ 자기 → 부모(parent task)로 올라가서 찾음-->
<!--    •    child task 2가 userID를 읽으면?-->
<!--➔ 자기 스택에 직접 userID = "Alice"가 있으므로 바로 읽음-->
<!---->
<!--✅ 핵심:-->
<!--구조화된 작업은 항상 부모 작업을 가리키는 링크드 리스트가 있기 때문에,-->
<!--태스크-로컬을 찾을 때 스택을 따라 상위 작업으로 거슬러 올라가는 탐색이 가능합니다.-->
<!---->
<!---->
<!---->
<!---->
<!--🧩 2. 비구조화 작업 (detached tasks) – 상위 작업 링크가 없다-->
<!---->
<!--```-->
<!--[parent task] (Task-local: requestID = 1234)-->
<!--    |-->
<!--    +-- [detached task] (no parent link, own task-local copy)-->
<!--```-->

<!--이는 Swift 동시성이 지닌 구조화된 특성을 온전히 수용하기 때문입니다.-->
