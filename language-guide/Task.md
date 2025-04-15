---
description: 비동기 작업을 수행하기 위한 기본 단위
---

`Task(작업)`은 Swift Concurrency에서 비동기 작업을 수행하기 위한 기본 실행 단위입니다. `Task`는 비동기 처리를 위한 자체적인 상태와 실행 자원을 관리하며, 각 작업은 서로 독립적인 작업 컨텍스트(Task Context)를 가지므로 다른 Task와 상태를 공유하지 않습니다.

`Task`는 동작 방식 면에서 `DispatchQueue.global().async`와 유사합니다. 전달된 작업은 특정 액터(@MainActor)에 속하지 않는 한, 글로벌 스레드 풀(Global Thread Pool)에서 사용 가능한 스레드를 할당받아 실행됩니다. 이 작업은 결과가 반환되기를 기다리지 않고, 곧바로 다음 줄의 코드를 실행합니다. 즉, 작업은 비동기적으로 처리됩니다. 각 `Task`는 서로 병렬로 수행될 수 있으며, 내부의 코드는 정해진 순서대로 순차적으로 실행됩니다. 즉, 비동기적으로 실행된다고 해서 중간 코드가 건너뛰어지거나 무시되는 일은 없습니다.

`Task`는 동기 코드에서 비동기 함수(Asynchronous Function)를 호출할 수 있는 가교 역할을 합니다. 비동기 함수는 `Task`가 제공하는 비동기 컨텍스트(Asynchronous Context)에서만 호출될 수 있으며, 이 컨텍스트는 특점 지점에서 코드의 실행을 일시 중단(suspend)했다가, 적절한 시점에 다시 재개(resume)될 수 있도록 지원합니다.


# Unstructured Concurrency

`Task`와 `Detached Task(독립적인 작업)`은 구조화되지 않은 동시성(Unstructured Concurrency)에 해당합니다. 구조화되지 않은 동시성은 구조화된 동시성(Structured Concurrency)와 달리 더 높은 유연성을 제공하지만, 작업 간 취소 전파나 자원(Task-Local, 우선순위 등) 상속에는 많은 제한이 따릅니다. 특히 동기 코드에서 비동기 작업을 실행해야 하는 상황에서는 상위 작업이 존재하지 않을 수 있습니다. 또한 작업의 생명 주기가 특정 코드 블록의 범위를 벗어나야 하는 경우도 있습니다. 이처럼 구조화된 동시성을 사용할 수 없거나 사용하기 어려운 경우, `Task`가 가장 적합한 선택지가 됩니다.


## Task

`Task`를 생성할 때는 수행할 작업을 클로저로 전달해야 합니다. `Task`는 생성되자마자 즉시 실행되며, `URLSession`의 `resume()` 메서드처럼 명시적으로 시작하거나 스케줄링할 필요가 없습니다. 작업을 생성한 후에는 해당 인스턴스를 통해 작업과 상호작용할 수 있습니다. 예를 들어, 작업의 결과를 기다리거나 작업을 취소할 수 있습니다. 작업에 대한 참조를 유지하지 않더라도 작업은 즉시 실행되지만, 참조를 유지하지 않으면 결과를 기다리거나 작업을 취소하는 등의 제어는 할 수 없습니다.


```swift
Task {
    print("✅ 작업 1 실행 시작")
    try? await Task.sleep(for: .seconds(1))
    print("✅ 작업 1 실행 종료")
}

let task: Task<Void, Never> = Task {
    print("💰 작업 2 실행 시작")
    await Task.yield()
    print("💰 작업 2 실행 종료")
}

Print "✅ 작업 1 실행 시작"
Print "💰 작업 2 실행 시작"
Print "💰 작업 2 실행 종료"
Print "✅ 작업 1 실행 종료"
```

`Task`는 외부 작업으로부터 일부 컨텍스트 자원(Task-Loca, 우선순위 등)을 상속받을 수 있습니다. 단, 작업 간 취소 전파는 불가능합니다.

```swift
Task(priority: .background) { 
    print("➡️ 외부 작업의 우선순위: \(Task.currentPriority)")
    
    Task { 
        print("➡️ 내부 작업의 우선순위: \(Task.currentPriority)")
    }
}

// Print "➡️ 외부 작업의 우선순위: TaskPriority.background"
// Print "➡️ 내부 작업의 우선순위: TaskPriority.background"
``` 

{% hint style="info" %}
**Note** `Task`는 작업이 생성될 때 주변 컨텍스트를 자동으로 캡처하므로, 일반적으로 `self`를 명시적으로 캡처할 필요는 없습니다.
{% endhint %}


## Detached Task

`Detached Task` 역시 `Task`와 많은 특성을 공유합니다. 그러나 `Detached Task`는 외부 작업으로부터 어떤 컨텍스트 자원을 상속받지 않으며, 완전히 분리되어 독립적으로 실행됩니다. 

```swift
Task(priority: .background) { 
    print("➡️ 외부 작업의 우선순위: \(Task.currentPriority)")
    
    Task.detached { 
        print("➡️ 내부 작업의 우선순위: \(Task.currentPriority)")
    }
}

// Print "➡️ 외부 작업의 우선순위: TaskPriority.background"
// Print "➡️ 내부 작업의 우선순위: TaskPriority.medium"
```

`Detached Task`는 백그라운드에서 로깅, 캐시 저장 등 외부 작업의 컨텍스트 자원에 의존하지 않고 완전히 독립적으로 실행되어야 하는 작업에 적합합니다. 즉, 작업이 특정 작업 컨텍스트의 제약에서 벗어나 최대한의 실행 유연성이 요구되는 경우, `Detached Task`가 가장 알맞은 선택지가 됩니다. 

```swift
@MainActor
extension MyDelegate: UICollectionViewDelegate {
    public func collectionView(_ view: UICollectionView,
                               willDisplay cell: UICollectionViewCell,
                               forItemAt item: IndexPath) {
        let ids = getThumbnailIDs(for: item)
        thumbnailTasks[item] = Task {
            defer { thumbnailTasks[item] = nil}
            let thumbnails = await fetchThumbnails (for: ids)
            Task.detached(priority: .background) {
                self.writeToLocalCache(thumbnails)
            }
            display(thumbnails, in: cell)
        }
    }
    
    public func collectionView(_ collectionView: UICollectionView,
                               didEndDisplaying cell: UICollectionViewCell,
                               forItemAt indexPath: IndexPath) {
        thumbnailTasks[indexPath]?.cancel()
        thumbnailTasks[indexPath] = nil
    }
}
```

위 예제에서는 `collectionView(_:willDisplay:)` 메서드는 셀이 화면에 표시되기 직전에 해당 셀에 표시할 썸네일 이미지를 비동기적으로 불러옵니다. 이 썸네일 로딩 작업은 일반적인 `Task`를 통해 실행되며, 작업 인스턴스를 통해 필요 시 취소할 수 있습니다.

썸네일이 성공적으로 로딩되면, 해당 이미지를 디스크에 저장하는 작업을 `Task.detached`로 실행합니다. `writeToLocalCache(_:)`는 디스크 I/O와 같이 비교적 무거운 백그라운드 작업으로, UI 컨텍스트나 상위 작업의 우선순위에 영향을 받지 않아도 되는 독립적인 작업입니다. 따라서 `Task.detached(priority: .background)`를 사용해 외부 작업과 분리된 환경에서 처리합니다.

아래 표는 `Task`와 `Detached Task`를 비교한 것입니다.

|   | **Task** | **Detached Task** |
| - | - | - |
| 취소 전파 | ❌ | ❌ |
| 자원 상속(Task-Local, 우선순위 등) | ✅ | ❌ |
| 비고 | 동기 코드에서 비동기 작업을 실행하는 비동기 컨텍스트 제공, 작업의 생명 주기가 특정 코드 블록의 범위를 벗어나야 할 때 사용됨. | 외부 작업의 컨텍스트 자원에 의존하지 않고 완전히 독립적으로 실행되어야 하는 작업에 적합함. |


# Returning Results from a Task

`Task`는 작업을 완료하면 결과값을 반환할 수 잇으며, 이 결과값은 `value` 프로퍼티를 통해 접근할 수 있습니다. 결과값은 얻는 데 시간이 걸릴 수 있으므로, 해당 프로퍼티에 접근할 때는 `await` 키워드를 붙여야 합니다. 만약 `Task`가 예외를 던질 수 있다면, 예외 처리를 위해 `await` 키워드 앞에 `try` 키워드도 함께 작성해야 합니다. `Task`는 오직 `Sendable`한 값만 반환할 수 있습니다. 그리고 `value` 프로퍼티를 호출하면 (내부) `Task`의 우선순위는 (외부) `Task`의 우선순위 수준으로 일시적으로 승격됩니다.

```swift
Task(priority: .high) {
    let task: Task<Int, Never> = Task {
        await doSomething()
    }
    let birthDay = await task.value
    print("김소월의 생일은 \(birthDay)입니다.")
}
// Print "김소월의 생일은 19980321입니다."
```

{% hint style="info" %}
**Info** `Sendable` 프로토콜은 서로 다른 동시 컨텍스트(Concurrent Context)에서 안전하게 공유할 수 있는 타입입니다. 자세한 내용은 [Sendable](programming-guide/Sendable.md) 문서를 참조하세요.
{% endhint %}


# Task Priority

`Task`도 `Grand Central Dispatch(GCD)`와 마찬가지로 우선순위(priority)를 가질 수 있습니다. 아래 표는 사용 가능한 우선순위 값을 보여줍니다.

| 종류 | 원시값 |
| - | - |
| `userInitiated` | 25 |
| `high` | 25 |
| `medium` | 21 |
| `low` | 17 |
| `utility` | 17 |
| `background` | 9 |

시스템은 작업 간 우선순위 역전(Priority Inversion)을 방지하기 위해, 특정 상황에서 작업의 우선순위를 일시적으로 승격시킬 수 있습니다. 예를 들어, 우선순위가 높은 (외부) `Task`에서 `value`를 호출해 다른 `Task`의 결과를 기다리는 경우, 시스템은 해당 결과를 계산 중인 (내부) `Task`의 우선순위를 현재 `Task`의 우선순위 수준으로 일시적으로 승격시켜 결과를 빠르게 반환할 수 있도록 합니다.

```swift
let outerTask = Task(priority: .high) {
    print("➡️ 외부 작업의 우선순위: \(Task.currentPriority)")
    
    let innerTask = Task(priority: .low) {
        print("➡️ 내부 작업의 우선순위: \(Task.currentPriority)")
        
        try? await Task.sleep(for: .seconds(1))
        
        print("➡️ 내부 작업의 우선순위: \(Task.currentPriority)")
    }
    
    // 높은 우선순위 작업이 낮은 우선순위 작업이 종료될 때까지 기다립니다.
    try? await Task.sleep(for: .seconds(0.5))
    _ = try? await innerTask.value
}

// 전체 작업이 종료될 때까지 기다립니다.
_ = try? await outerTask.value

Print "➡️ 외부 작업의 우선순위: TaskPriority.high"
Print "➡️ 내부 작업의 우선순위: TaskPriority.low"
Print "➡️ 내부 작업의 우선순위: TaskPriority.high"
```

위 예제에서는 높은 우선순위의 `outerTask`가 낮은 우선순위인 `innerTask`의 결과를 `value`를 통해 기다립니다. 이때 시스템은 `innerTask`의 우선순위를 일시적으로 `outerTask`의 우선순위(`high`)와 동일하게 승격시켜 `outerTask`가 더 빠르게 결과를 받을 수 있도록 합니다. 

또 다른 예로, 액터(Actor)가 작업을 수행 중일 때, 더 높은 우선순위의 작업이 해당 액터의 대기열(Queue)에 새롭게 추가되면, 현재 실행 중인 작업의 우선순위가 일시적으로 승격될 수 있습니다. 액터는 한 번에 하나의 작업만 처리할 수 있기 때문에, 대기열 앞에 낮은 우선순위 작업이 실행 중이라면, 이후에 도착한 높은 우선순위 작업은 자신의 우선순위대로 즉시 실행되지 못하고 지연될 수 있습니다. 이러한 상황을 방지하기 위해, 시스템은 현재 실행 중인 작업의 우선순위를 일시적으로 높여, 높은 우선순위 작업이 불필요하게 대기하지 않도록 합니다.

{% hint style="info" %}
**Info** 액터는 재진입성(Re-Entrancy)이라는 특성을 통해 작업 간 우선순위를 보다 유연하게 조정할 수도 있습니다. ~~자세한 내용은 [Actor]() 문서를 참조하세요.~~
{% endhint %}


# Task Cancellation

`Task`는 작업을 취소할 수 있는 간단한 메서드를 제공합니다. `Task` 인스턴스에서 `cancel()` 메서드를 호출하면, 해당 `Task` 내부의 모든 비동기 함수, `async-let` 그리고 `TaskGroup`에 작업 취소가 전파됩니다. 단, 내부에 정의한 또 다른 `Task`나 `Detached Task`에는 취소가 전파되지 않습니다.

```swift
let outerTask = Task {
    try? await Task.sleep(for: .seconds(1))
    
    let innerTask = Task {
        print("➡️ 내부 작업의 취소 여부: \(Task.isCancelled)")
    }
    
     print("➡️ 외부 작업의 취소 여부: \(Task.isCancelled)")
}
outerTask.cancel()

// Print "➡️ 외부 작업의 취소 여부: true"
// Print "➡️ 내부 작업의 취소 여부: false"
```

`cacnel()` 메서드는 단순히 `Task`의 `isCancelled` 프로퍼티를 `false`에서 `true`로 변경할 뿐이며, 작업을 즉시 중단시키지 않습니다. 즉, `cancel()`은 취소를 알리는 신호만 보낼 뿐, 실제로 작업을 중단하는 로직은 구현되어 있지 않습니다. 따라서 각 작업은 실행 도중 스스로 취소 여부를 확인하고, 그에 맞게 적절한 방식으로 종료되어야 합니다. 이러한 방식을 협력적 취소(Cooperative Cancellation)라고 합니다.

협력적 취소가 필요한 이유는 작업마다 취소에 대응하는 방식이 다를 수 있기 때문입니다. 대부분의 작업은 취소가 전파되면 예외를 던지며 종료되지만, 일부 작업은 지금까지의 중간 결과를 반환하거나, 별도의 정리 작업을 수행해야 할 수도 있습니다.

작업 중단 시 단순히 예외를 던지면 되는 경우, `Task.checkCancellation()` 메서드를 사용해 취소 여부를 확인할 수 있습니다. 반대로, 중단 시 별도의 처리가 필요한 경우에는 `Task.isCancelled` 프로퍼티를 활용해 필요한 처리를 직접 구현해야 합니다.

```swift
func doTask() async {
    print("📡 작업 시작")
    guard !Task.isCancelled else {
        print("❌ 작업 취소")
        return
    }
    print("📡 작업 종료")
}
let task = Task {
    try? await Task.sleep(for: .seconds(1))
    await doTask()
}
task.cancel()

// Print "📡 작업 시작"
// Print "❌ 작업 취소"
```

{% hint style="info" %}
**Info** ~~작업 취소(Cancellation)에 관한 자세한 내용은 [Cancellation]() 문서를 참조하세요.~~
{% endhint %}


# Preferred Task Executor

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}

<!--{% hint style="info" %}-->
<!--**Info** ~~작업 실행자(Task Executor)에 관한 자세한 내용은 [Task Executor]() 문서를 참조하세요.~~-->
<!--{% endhint %}-->


# Task and [weak self] Capture

대부분의 경우, `Task`를 생성할 때 `[weak self]`와 같은 키워드를 사용해 현재 컨텍스트를 약하게 캡처할 필요가 없습니다. 이는 `Task`가 보유한 모든 참조가 작업이 완료된 직후 곧바로 해제되기 때문입니다.

예를 들어 아래 예제에서, `Task`는 액터를 약하게 캡처할 필요가 없습니다. 작업이 완료되면 `Task`가 액터에 대한 참조를 해제함으로써, 작업과 액터 간의 순환 참조(Reference Cycle)가 끊기기 때문입니다.

```swift
struct Work: Sendable { }

actor Worker {
    var work: Task<Void, Never>?
    var result: Work?
    
    deinit {
        // even though the task is still retained,
        // once it completes it no lognre causes a reference cycle with the actor
        
        print("deinit actor")
    }
    
    func start() {
        work = Task {
            print("start task work")
            try? await Task.sleep(for: .seconds(3))
            self.result = Work()
            print("completed task work")
            // but as the task completes, this reference is released
        }
        // we keep a strong reference to the task
    }
}
Task { await Worker().start() }
```

