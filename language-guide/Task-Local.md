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

 태스크-로컬은 작업 컨텍스트 간에 메타데이터를 안전하고 일관되게 전달합니다. 태스크-로컬 값은 작업 생성이 생성될 때 암시적으로 전달되며, 해당 작업이 생성하는 모든 하위 작업에서도 읽을 수 있습니다. 태스크-로컬 값을 특정 스코프에 바인딩하면, 해당 스코프 내의 동기 함수(Synchronous Function), 비동기 함수(Asynchronous Function), `Task`, 작업 그룹(TaskGroup) 그리고 `async-let` 구문에서도 바인딩된 값을 읽을 수 있습니다. 이 값은 바인딩된 스코프
  내의 작업에만 접근할 수 있으며, 다른 작업과 임의로 공유되지 않습니다.


{% hint style="info" %}
**Info** 

실제 코드에서는 태스크-로컬을 신중하게 사용해야 합니다. 단순히 매개변수로 전달할 수 있는 값을 태스크-로컬로 사용하는 것은 피해야 합니다. 태스크-로컬 값은 함수 호출의 결과에 직접적인 영향을 주지 않고, 부수적인 구성 정보에만 영향을 미치는 메타데이터에 한정하여 사용하는 것이 바람직합니다. 만약 어떤 값을 매개변수로 전달할지, 태스크-로컬로 전달할지 확신이 서지 않는 경우에는 매개변수로 직접 전달하는 방법을 선택하는 것이 좋습니다. 태스크-로컬은 주로 추적 ID, 인증 토큰과 같은 메타데이터를 전달하기 위해 설계되었다는 점을 기억하세요.
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

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}

<!---->
<!--Swift의 Task 구현은 본질적으로 작업(Task)들을 연결한 연결 리스트(linked list) 형태로 되어 있으며,-->
<!--자식 작업이 자신의 부모 작업(parent task)을 조회할 수 있습니다.-->
<!--이 구조를 활용하여, Task-local 값을 자식 작업에 복사하지 않고도 필요한 의미론을 구현할 수 있습니다.-->
<!---->
<!--구체적으로, Task-local 값을 읽을 때의 동작은 다음과 같습니다:-->
<!--    •    먼저 현재 작업에 해당 키(key)가 존재하는지 확인합니다.-->
<!--    •    만약 존재하지 않으면, 부모 작업(parent) 에서 키를 조회합니다.-->
<!--    •    이 과정을 부모가 더 이상 없을 때까지 반복합니다.-->
<!--    •    끝까지 조회해도 키를 찾지 못하면, 최종적으로 Task-local 키에 정의된 기본값(default value) 을 반환합니다.-->
<!--    -->
<!--```-->
<!--[detached] ()-->
<!--  \ -->
<!--  |[child-task-1] (id:10)-->
<!--  |   \-->
<!--  |   |[child-task-1-1] (id:20)-->
<!--  |[child-task-2] (name: "alice")-->
<!--```-->
<!---->
<!--위 구조를 기준으로 설명하면 다음과 같습니다.-->
<!--    •    child-task-2에서 name을 조회할 경우,-->
<!--현재 작업에 "alice"가 직접 저장되어 있으므로 즉시 "alice"를 반환합니다.-->
<!---->
<!--    •    반면, child-task-1-1에서 name을 조회하려 하면,-->
<!--    1.    먼저 자신(child-task-1-1)을 확인하고,-->
<!--    2.    부모(child-task-1)를 확인하고,-->
<!--    3.    최상위 작업(detached)을 확인하게 됩니다.-->
<!--그러나 이들 어디에도 name 값이 설정되어 있지 않기 때문에, 최종적으로 기본값(default value) 을 반환하게 됩니다.-->
<!---->
<!--    •    child-task-1-1에서 id를 조회하는 경우에는,-->
<!--현재 작업(child-task-1-1)에 id: 20이 저장되어 있으므로, 곧바로 20을 반환합니다.-->
<!--이는 호출 체인(call chain)에서 더 구체적으로 설정된(more specific) 값을 우선 사용하는, 우리가 기대한 동작입니다.-->
<!---->
<!--또한, 실제로 많은 상황에서 다음과 같은 작업(Task) 체인이 형성되는 것을 볼 수 있습니다:-->
<!---->
<!--```-->
<!--[detached] ()-->
<!--  \ -->
<!--   [child-task-1] (requestID:10)-->
<!--    \-->
<!--    |[child-task-2] ()-->
<!--     \-->
<!--     |[child-task-3] ()-->
<!--      \-->
<!--      |[child-task-4] ()-->
<!--```-->
<!---->
<!--여기서 볼 수 있듯이, 여러 작업이 존재할 수 있지만, 이들 중 다수는 새로운 Task-local 값을 추가하지 않고 단순히 체인에 연결되어 있을 뿐입니다.-->
<!---->
<!--Task-local 값은 작업 생성 시점에 불변(immutable) 이기 때문에,-->
<!--한 번 설정된 값은 이후 변경되지 않는다는 사실을 보장할 수 있습니다.-->
<!--이 덕분에, 부모 작업이 새로운 Task-local 값을 추가하지 않는 경우,-->
<!--그 자식 작업들에서도 Task-local 값을 조회하는 과정을 최적화할 수 있습니다.-->
<!---->
<!--구체적으로 설명하면, 예를 들어 child-task-3을 생성할 때,-->
<!--부모 작업인 child-task-2가 Task-local 값을 가지고 있지 않음을 감지할 수 있습니다.-->
<!--따라서 child-task-3은 직접 상위(parent) 작업인 child-task-1을 가리키도록 최적화할 수 있습니다.-->
<!--(child-task-1은 실제로 Task-local 값을 가지고 있습니다.)-->
<!---->
<!--일반적으로 이 규칙은 다음과 같이 표현할 수 있습니다:-->
<!--    •    “Task-local 값을 실제로 가지고 있는 첫 번째 부모 작업을 가리킨다.”-->
<!---->
<!--이 최적화 덕분에, 예를 들어 child-task-4에서 requestID를 조회할 때,-->
<!--    •    단 한 번의 “점프(hop)“만으로 바로 child-task-1에 도달해 값을 가져올 수 있습니다.-->
<!--(child-task-1은 requestID를 정의하고 있으므로 바로 값을 반환할 수 있습니다.)-->
<!---->
<!--만약 child-task-1조차도 우리가 찾는 키를 가지고 있지 않았다면,-->
<!--    •    검색은 계속 진행되며, (Task-local 값이 없는 작업들은 건너뛰면서)-->
<!--    •    최종적으로 detached 작업에 도달할 때까지 이어집니다.-->
<!--    -->
<!--이 접근 방식은 Task-local 값이 사용되는 실제 용도에 맞추어 고도로 최적화되어 있습니다.-->
<!--특히, 다음과 같은 접근 패턴(Access Pattern) 을 전제로 설계되었습니다:-->
<!---->
<!--Task-local 값을 실제로 읽는 작업(Task)은 상대적으로 소수에 불과하다.-->
<!--    •    대개는 하나의 “루트 작업(root task)” 이 Task-local 정보를 설정하고,-->
<!--그 수명 동안 수백 개, 수천 개의 작은 자식 작업(child task) 이 생성되며,-->
<!--이들 중 일부는 값을 읽고, 일부는 읽지 않을 수도 있다.-->
<!--    •    대부분의 자식 작업은 Task-local 정보를 읽지 않는다.-->
<!--심지어 트레이싱(tracing) 같은 경우처럼 많은 작업이 값을 읽는 상황에서도,-->
<!--이는 전체 코드 실행의 일부에만 해당된다.-->
<!---->
<!--결론:  모든 자식 작업에 대해 값을 적극적으로 복사하는 것은 비용 대비 효율이 떨어진다.-->
<!--    •    조회할 때 약간의 성능 손해를 감수하는 편이 훨씬 합리적이다.-->
<!--    -->
<!---->
<!--값을 설정한 작업(Task)과 그 값을 읽는 작업 사이에는-->
<!--수많은 작업들이 ‘끼어 있을 수’ 있습니다.-->
<!---->
<!--또한, 프레임워크나 런타임이 제어 흐름(control flow)을 사용자 코드로 넘기기 전에,-->
<!--Task-local 값을 한 번만 설정하는 경우가 흔합니다.-->
<!--이후 사용자 코드는 대개 새로운 Task-local 값을 추가하지 않고,-->
<!--이미 설정된 값을 단순히 사용하는 정도에 그칩니다.-->
<!--(예: 로깅(logging)이나 트레이싱(tracing) 과정에서 기존 request ID를 참조하는 경우)-->
<!---->
<!--결론: Task-local 값을 가지고 있지 않은 ‘빈(empty) 작업’을 건너뛰는 최적화는 충분히 가치가 있다.-->
<!---->
<!---->
<!--작업(Task)은 Task-local 값에 대한 경합(race condition)을 걱정할 필요가 없어야 합니다.-->
<!---->
<!--작업은 언제든지 Lib.myValue를 호출했을 때-->
<!--예측 가능한(predictable) 값을 안정적으로 반환받을 수 있어야 합니다.-->
<!--이를 보장하기 위해, 작업은 자신의 Task-local 값을 변경(mutate)할 수 없어야 합니다.-->
<!---->
<!--만약 작업이 실행 중에 Task-local 값을 변경할 수 있다면,-->
<!--자식 작업(child task) 이 동시에 실행되면서 Lib.myValue를 두 번 호출할 때-->
<!--서로 다른 결과(conflicting results) 를 얻을 가능성이 생깁니다.-->
<!--이는 매우 혼란스러운 프로그래밍 모델을 초래할 수 있습니다.-->
<!---->
<!--결론: Task-local 저장소는 작업(Task) 생성 시점에만 초기화되어야 하며, 이후에는 값을 변경할 수 없어야 합니다. 새로운 값을 사용하고자 한다면, 새로운 범위(scope)나 작업(task)를 생성하여 값(binding)을 새로 설정해야 합니다.-->



# Binding Task-Local Values

값을 바인딩(binding)하는 것은 태스크-로컬의 핵심적인 기능입니다. 태스크-로컬은 스레드-로컬과 달리, 값을 직접 설정(set)할 수 없습니다. 대신, 주어진 스코프 내에서만 값을 바인딩하며, 이 스코프 내에서 생성된 모든 (하위) 작업에서 바인딩된 값을 사용할 수 있도록 합니다. 스코프를 벗어나면 바인딩된 값에 더 이상 접근할 수 없으며, 기본값만을 읽을 수 있습니다. 이 메커니즘은 값이 설정된 후 방치되어 메모리 누수가 발생하거나, 예상치 못한 코드 경로에서 잘못된 값을 읽어 디버깅이 어려워지는 문제를 방지하기 위한 것입니다.

값의 생명 주기를 스코프의 생명 주기에 맞춰 제한함으로써, Swift는 태스크-로컬 값을 스택 기반으로 효율적으로 관리할 수 있습니다. 스코프가 종료되면, 해당 스코프 내에서 생성된 하위 작업도 함께 종료되고, 이에 따라 바인딩된 태스크-로컬 값 역시 자동으로 해제됩니다.

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


## Binding values for the duration of a child-task

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}

<!---->
<!--- 구조화된 동시성에서 하위 작업 생성 시, 태스크-로컬은 하위 작업들에게 태스크-로컬 값을 상속하고, 읽을 수 있게 함 (내용 보충)**** -->
<!---->
<!--```swift-->
<!--await MyLibrary.$requestID.withValue("1234-5678") {-->
<!--  await withTaskGroup(of: String.self) { group in -->
<!--    group.addTask { // add child task running this closure-->
<!--      MyLibrary.requestID // returns "1234-5678", which was bound by the parent task-->
<!--    }-->
<!--                                        -->
<!--    return await group.next()! // returns "1234-5678"-->
<!--  } // returns "1234-5678"-->
<!--}-->
<!--```-->
<!---->
<!--- The same operations also work and compose naturally with child tasks created by async let and any other future APIs that would allow creating child tasks.-->
<!---->
<!--```swift-->
<!--await Lib.$wasabiPreference.withValue(.withWasabi) {-->
<!--  async let firstMeal = cookDinner()-->
<!--  async let secondMeal = cookDinner()-->
<!--  async let noWasabiMeal = Lib.$wasabiPreference.withValue(.withoutWasabi) {-->
<!--    cookDinner()-->
<!--  }-->
<!--  await firstMeal, secondMeal, noWasabiMeal-->
<!--}-->
<!--```-->
<!---->
<!--- withValue(_:operation:) 스코프 안에서 설정된 Task-local 값은, 단순히 해당 클로저가 끝날 때 없어지는 게 아닙니다. 그 스코프 안에서 생성된 모든 자식 Task가 끝날 때까지 유지됩니다. withValue(_:operation:)을 통해 스코프를 만들고 그 범위 내에서만 값이 유효하게 됩니다 스코프가 끝나면 값도 자동으로 사라짐 → 안전, 예측 가능, 누수 없음-->
<!--    -->
<!--    이 구조 덕분에 자식 Task들이 부모의 값을 안정적으로 참조할 수 있어요. 중간에 값을 해제해버리면 예상치 못한 크래시나 버그가 생기기 때문이죠.-->
<!---->
<!--자식 Task가 부모 Task의 Task-Local 값을 변경하는 것은 불가능합니다. 자식 Task는 부모 Task로부터 Task-Local 값을 초기 복사(copy) 받을 수 있지만, 변경은 자기 복사본에 대해서만 할 수 있고, 부모 Task의 원본에는 절대 영향을 줄 수 없습니다.-->
<!---->
<!--즉, 자식 Task는 부모 Task로부터 받은 자기 복사본만 수정할 수 있고,-->
<!--부모 Task의 원본 Task-Local 값은 전혀 영향을 받지 않습니다.-->
<!--이로써 Task 간 데이터 격리가 엄격히 지켜집니다.-->


## Task-local value and tasks which outlive their scope

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}

<!---->
<!--- `Task`는 실행 컨텍스트, 작업 우선순위, 태스크-로컬 값 등 생성하는 작업의 모든 작업 컨텍스트를 상속받습니다. 비록 구조화되지 않은 작업(unstructured task)이라 하더라도, 새로운 비동기 작업으로 값을 복사하는 방식으로 태스크-로컬 값을 상속합니다.-->
<!---->
<!--때때로 작업의 결과를 기다리지 않고 “비동기적으로 작업을 계속 진행”해야 할 필요가 있을 수 있습니다.-->
<!---->
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
<!--필요한 경우, 수동으로 전파 과정을 처리하여 detached task가 특정 우선순위(priority), 실행자(executor) 선호 설정, 심지어 태스크-로컬 값(task-local value)까지도 전달받도록 만들 수 있습니다.-->
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
<!--이는 작업량이 많고 보일러플레이트 코드가 많이 발생하는 작업이지만, detached task가 이전 컨텍스트를 전혀 물려받지 않는 것은 의도된 설계입니다. 따라서 detached task가 어떤 정보를 전달받아야 하는 경우, 반드시 명시적으로 그렇게 해야 합니다.-->
<!---->
<!--한편, 새로 제안된 async 연산(이름은 아직 확정되지 않았으며, 아마 send가 될 수도 있습니다(?))은 생성한 작업의 다음 속성들을 모두 상속받습니다: 실행 컨텍스트(execution context), 태스크 우선순위(task priority), 그리고 태스크-로컬 값(task-local values).-->
<!---->
<!--async 연산은 별도로 제안(pitch)될 예정이지만, 이 제안에서는 async가 태스크-로컬 값을 어떻게 전파하는지만 집중하면 됩니다. 다음 코드 조각을 살펴봅시다.-->
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
<!--async 연산은 detach 연산과 유사하게, 생성한 작업보다 더 오래 살아남을 수 있다는 점에 주목해야 합니다. 즉, async로 생성된 작업은 자식 작업(child-task)이 아니기 때문에, 기존에 태스크-로컬 값이 태스크 트리(task tree)를 기반으로 저장되던 방식은 여기서는 사용할 수 없습니다.-->
<!---->
<!--구현은 이를 올바르게 처리하기 위해, async 작업을 생성하는 시점(위 예시의 3번째 줄)에서 모든 태스크-로컬 값 바인딩(task-local value bindings)을 새 작업으로 복사(copy)합니다. 이는 일반적인 자식 작업을 생성하는 것보다 약간 더 무거운 작업임을 의미합니다. 단순히 작업(Task)이 힙에 할당될 가능성이 높아질 뿐만 아니라, 생성한 작업의 모든 태스크-로컬 바인딩을 새 작업으로 복사해야 하기 때문입니다.-->
<!---->
<!--여기서 복사되는 것은 바인딩(binding)만이라는 점에 유의해야 합니다. 즉, 생성한 작업에서 withValue를 사용해 바인딩된 값이 참조 카운트(reference counted) 타입이라면, 새 작업으로 복사되는 것은 해당 객체의 “참조(reference)“입니다. 그리고 객체를 계속 살아있게 유지하기 위해 참조 카운트가 증가합니다.-->



## Binding task-local values from synchronous functions

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}

<!---->
<!--- 태스크-로컬 값을 읽거나 바인딩하는 작업은 동기 함수(synchronous functions)에서도 가능합니다.-->
<!---->
<!--- 동기 함수에서 값을 바인딩하거나 읽을 때도 동일한 API를 사용합니다. 다만, 동기 함수 안에서 withValue를 호출해 키에 특정 값을 바인딩할 경우, 전달하는 클로저는 비동기(async) 클로저일 수 없습니다. (이는 async 함수 호출 규칙과 같습니다.)-->
<!---->
<!--- 가끔, 동기 withValue 함수가 태스크(Task)가 존재하지 않는 컨텍스트에서 호출될 수 있습니다. 일반적인 Swift 프로그램에서는 모든 스레드와 호출이 어떤 초기 비동기 함수로부터 시작되므로 이런 상황은 드뭅니다. 그러나, 예를 들어 C 라이브러리나 자체적으로 스레드를 관리하는 다른 라이브러리에서 진입하는 경우, 사용할 수 있는 Task가 존재하지 않을 수 있습니다. 이러한 경우에도, 태스크-로컬 값 API는 정상적으로 작동합니다. 특별한 스레드-로컬(thread-local) 메커니즘을 사용해 가상의 태스크 스코프를 시뮬레이션하여 태스크-로컬 저장소를 기록하기 때문입니다.-->
<!---->
<!--- 이러한 설계 덕분에, 코드가 동기적(synchronous)으로 유지되기만 한다면, 태스크가 없는(task-less) 컨텍스트에서도 일반적인 태스크-로컬 작업을 계속해서 사용할 수 있습니다.-->
<!---->
<!---->
<!--```swift-->
<!--func enter() {-->
<!--    Example.$traceID.withValue(1234) { -->
<!--        print("traceID: \(Example.traceID)") // always "1234", regardless if enter() was called from inside a task or not:-->
<!--    }-->
<!--}-->
<!---->
<!--// 1) Call `enter` from non-Task code-->
<!--//       e.g synchronous main() or non-Task thread (e.g. a plain pthread)-->
<!--enter()-->
<!---->
<!--// 2) Call `enter` from Task-->
<!--Task {-->
<!--    enter()-->
<!--}-->
<!--```-->
<!---->
<!--- 만약 현재 실행 중인 코드가 Task 컨텍스트 바깥, 즉 비동기 작업이 전혀 시작되지 않은 상태라면 어떻게 될까요? Swift의 @TaskLocal은 비동기 함수가 없는 동기 함수에서도 사용할 수 있으며, 이러한 경우에는 스레드 로컬(thread-local) 저장소를 사용하여 값을 저장합니다. 즉, Task 컨텍스트 외부(예: 메인 스레드나 비동기 Task가 아닌 곳)에서 @TaskLocal 값을 설정하면, Swift 런타임은 해당 값을 스레드 로컬 저장소에 저장합니다. -->
<!---->
<!--```swift-->
<!---->
<!--```-->
<!---->
<!--- 이렇게 하면 호출 컨텍스트를 신경 쓰지 않고도 태스크 로컬 값을 안정적으로 바인딩하고 읽을 수 있습니다. 그러나 이러한 스레드 로컬 저장소는 새로운 비구조적 스레드(예: pthread)에서는 자동으로 전파되지 않습니다. 따라서 이러한 스레드에서는 @TaskLocal 값을 명시적으로 설정해야 합니다. ￼이러한 동작은 Swift의 구조화된 동시성 모델과 일관성을 유지하면서도, 비동기 컨텍스트가 아닌 경우에도 @TaskLocal API가 정상적으로 작동하도록 보장해 줍니다.-->
<!---->
<!--- 이는 작업 내에서 호출된다고 보장되지 않는 동기 함수에서 특히 유용합니다. Swift에서는 코드가 Task의 컨텍스트 밖—즉, 동기 함수나 Swift의 동시성 시스템에 의해 관리되지 않는 스레드—에서 실행될 경우, Task-Local 값은 스레드-로컬(thread-local) 저장소를 사용하도록 대체(fallback) 됩니다. 즉, 아래와 같은 예제처럼 호출 컨텍스트를 신경 쓰지 않고도, 태스크-로컬 값을 안정적으로 바인딩하고 읽을 수 있습니다.-->




# Task-Local Value Lifecycle

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}
<!---->
<!---->
<!--태스크-로컬 값은 withValue 연산 스코프가 종료될 때까지 유지(retain)됩니다.-->
<!--실질적으로 이는, 해당 스코프 내에서 생성된 모든 자식 작업이 종료될 때까지 값이 살아있음을 의미합니다.-->
<!--이는 매우 중요한데, 자식 작업이 부모 작업의 특정 태스크-로컬 값을 참조하고 있을 수 있기 때문에,-->
<!--그보다 먼저 값을 해제할 수 없기 때문입니다.-->
<!---->
<!--태스크-로컬 저장소에는 값 타입(value type)과 참조 타입(reference type) 모두 저장할 수 있으며,-->
<!--각 타입에 따라 예상되는 방식으로 동작합니다:-->
<!---->
<!--* 값 타입은 복사(copy) 되어 태스크-로컬 저장소에 저장됩니다.-->
<!---->
<!--* 참조 타입은 참조(retain) 되어 태스크-로컬 저장소에 저장됩니다.-->
<!---->
<!--Task-local “항목(item)” 저장소는 효율적인 스택 규율(stack-discipline) 기반 할당기를 사용하여 메모리를 할당합니다. 이는 해당 항목들이 설정된 작업(Task)을 절대 초과하여 살아남을 수 없다는 사실을 기반으로 설계되었습니다. 덕분에, 이러한 방식으로 값을 저장하는 것은 전역 메모리 할당기를 거치는 것보다 약간 더 저렴하게 메모리를 할당할 수 있습니다.-->
<!---->
<!--다만, Task-local 저장소는 명시적인 매개변수 전달을 대신하기 위한 용도로 남용해서는 안 됩니다. 이렇게 “숨겨진 인자(hidden argument)“를 통해 값을 전달하면 코드의 동작을 이해하기 어려워질 수 있기 때문입니다. 명시적으로 함수 매개변수를 사용하는 전통적인 방식이 코드 가독성과 유지보수성 측면에서 훨씬 낫습니다.-->
<!---->
<!--Task-local 항목이 다른 작업(Task)으로 복사되는 경우,-->
<!--예를 들어 async {}를 사용해 새로운 비구조적 작업(unstructured task) 을 시작할 때,-->
<!--해당 항목은 새로 생성된 작업에 독립적으로 연결되며, 각각 별도의 수명(lifecycle) 을 가집니다.-->
<!--즉, async {}로 새 작업을 만들 때,-->
<!--Task-local 저장소에 보관된 참조 카운트(reference-counted) 타입은 유지(retain)될 수 있습니다.-->


## Child task and value lifetimes

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}

<!---->
<!--또한 중요한 점은,-->
<!--Task-local 값 스택(stack)이나 연결 리스트(linked list) 의 내부 구현에서 추가적인 동기화(synchronization)가 필요하지 않다는 사실입니다.-->
<!--이는 Swift의 구조화된 동시성(structured concurrency) 이 제공하는 강력한 보장에 기반합니다.-->
<!---->
<!--구체적으로, Swift는 다음과 같은 보장을 활용합니다:-->
<!--    •    스코프(scope) 가 종료될 때까지-->
<!--자식 작업(child task) 은 반드시 완료되었거나,-->
<!--완료되지 않았다면 자동으로(await) 기다리게 됩니다.-->
<!--    •    만약 스코프가 오류(error)를 던지면서 종료될 경우,-->
<!--자식 작업은 자동으로 취소(cancel) 된 후, 기다려(await) 마무리됩니다.-->
<!---->
<!--이러한 구조화된 동시성의 특성 덕분에,-->
<!--Task-local 값의 내부 상태를 안전하게 관리할 수 있으며,-->
<!--별도의 락(lock)이나 복잡한 동기화 메커니즘을 추가할 필요가 없습니다.-->
<!---->
<!--이러한 보장 덕분에,-->
<!--자식 작업(child task) 은 부모 작업(parent) (또는 상위 부모(super-parent))의 Task-local 값 스택(stack)의 헤드(head) 를 직접 참조할 수 있습니다.-->
<!--그리고 이 참조를 위해 추가적인 관리 작업(house-keeping) 을 따로 구현할 필요가 없습니다.-->
<!---->
<!--Swift는 다음을 확실히 보장합니다:-->
<!--    •    자식 작업은,-->
<!--withValue 블록 내에서 정의된 Task-local 값들을 참조할 수 있으며,-->
<!--부모 작업이 해당 값을 보유한 상태로 유지됩니다.-->
<!--    •    또한,-->
<!--모든 자식 작업은 withValue가 반환되기 전에 반드시 완료됩니다.-->
<!---->
<!--이러한 구조를 활용하여,-->
<!--Swift는 withValue 함수가 반환될 때-->
<!--Task-local 값 스택에 바인딩된(bound) 값들을 자동으로 제거(pop) 합니다.-->
<!---->
<!--이 동작은 안전하게 보장됩니다.-->
<!--왜냐하면 그 시점에는 모든 자식 작업이 이미 완료되었기 때문에,-->
<!--Task-local 값을 참조하고 있는 작업은 더 이상 존재하지 않기 때문입니다.-->


## Task-locals in contexts where no Task is available

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}

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
