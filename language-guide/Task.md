---
description: 비동기 작업을 수행하기 위한 기본 단위
---

`Task`는 비동기 작업을 실행하기 위한 기본 단위입니다. 

`Task`는 동작 방식 측면에서 글로벌 디스패치 큐(Global Dispatch Queue)와 유사하게 동작합니다. 격리되지 않은 `Task`는 특정 액터(Actor)에 속하지 않으며, 글로벌 스레드 풀(Global Thread Pool)에서 가용한 스레드를 할당받아 실행됩니다. `Task`는 비동기적으로 실행되며, 생성 직후 결과를 기다리지 않고 곧바로 다음 줄의 코드를 실행할 수 있습니다. 이러한 특성 덕분에 각 `Task`는 병렬로 실행될 수 있습니다. `Task` 내부의 코드는 작성된 순서대로 실행되며, 비동기로 실행된다고 해서 코드의 일부가 건너뛰어지거나 무시되는 일은 없습니다.

각 `Task`는 고유한 상태를 가지며, 자체적인 작업 컨텍스트(Task Context) 내에서 실행됩니다. 작업 컨텍스트란 `Task`가 실행되는 동안 내부적으로 유지되는 독립적인 실행 환경을 의미합니다. 이 컨텍스트에는 액터(Actor), 작업 우선순위(Priority), 취소 상태(Cancellation)나 태스크-로컬(Task-Local) 등 컨텍스트를 포함하고 있습니다. 이 컨텍스트는 다른 `Task`와 상태를 공유하지 않으므로, 각 `Task`는 서로에게 영향을 주지 않고 독립적으로 동작할 수 있습니다. 

`Task`는 동기 코드에서 비동기 함수를 사용할 수 있도록 해주는 가교 역할을 합니다. 즉, 기존의 동기 흐름 안에서 새로운 비동기 작업을 시작할 수 있도록 도와줍니다. `Task`를 생성하면 새로운 비동기 컨텍스트(Asynchronous Context)가 열리며, 이 컨텍스트 내부에서만 비동기 함수(Asynchronous Function)를 호출할 수 있습니다. 이 컨텍스트는 실행 흐름을 특정 시점에서 일시 중단(suspend)할 수 있으며, 적절한 시점에 다시 재개(resume)될 수 있도록 지원합니다.


# Creating a Concurrency Context to Call Asynchronous Functions

작업 컨텍스트를 생성하는 주요 방법으로 두 가지가 있습니다. 바로 `Task`와 `Detached Task`입니다. 이 두 방식은 구조화되지 않은 동시성(Unstructured Concurrency)에 해당하며, 구조화된 동시성보다 더 높은 유연성을 제공합니다. 

예를 들어, 비동기 코드를 실행해야 하지만 현재 컨텍스트가 동기적인 경우, 작업의 생명 주기가 현재 코드 블록의 범위를 벗어나야 하는 경우, 우선순위나 실행 컨텍스트를 명시적으로 지정해야 하는 경우, `Task`와 `Detached Task`로 새로운 비동기 컨텍스트를 생성하는 게 적절한 해결책이 됩니다.

그러나 이러한 유연성에는 대가가 따릅니다. `Task`와 `Detached Task`는 상위 작업(Parent Task)과 느슨하게 연결되어 있으며, 그 결과 구조화된 동시성에서 자동으로 제공되는 작업 간 취소 전파(Cancellation Propagation)가 이뤄지지 않습니다. 즉, 상위 작업이 취소되더라도 하위 작업도 자동으로 취소되지 않기 때문에, 취소 관리와 예외 처리는 개발자가  명시적으로 관리해야 합니다.


## Tasks That Inherit Context

`Task`를 생성할 때, 수행할 작업을 `operation` 클로저 형태로 전달하며, 생성과 동시에 작업이 즉시 시작됩니다. `URLSession`의 `resume()` 메서드처럼 별도의 실행 시작 메서드를 호출할 필요가 없습니다. 

작업이 생성되면, `Task` 인스턴스가 반환됩니다. 이 인스턴스를 통해 현재 진행 중인 작업과 상호작용할 수 있습니다. 예를 들어, 진행 중인 작업을 취소하거나, 작업의 결과를 받기 위해 기다리는 등의 동작이 가능합니다. 작업에 대한 참조를 유지하지 않더라도 작업은 계속 실행되지만, 참조가 없으면 작업의 결과를 기다리거나 취소하는 등의 제어를 할 수 없습니다.

```swift
Task {
    do {
        print("Task 1 started")
        try await Task.sleep(for: .seconds(1))
        print("Task 1 ended")
    } catch { ... }
}

let task: Task<Void, Never> = Task {
    print("Task 2 started")
    print("Task 2 ended")
}

// Print "Task 1 started"
// Print "Task 2 started"
// Print "Task 2 ended"
// Print "Task 1 ended"
```

생성된 `Task`는 액터, 작업 우선순위나 태스크-로컬 값 등 외부 작업의 컨텍스트를 상속받을 수 있습니다. 

```swift
Task(priority: .background) { 
    print("Outer task priority: \(Task.currentPriority)")
    
    Task { 
        print("Inner task priority: \(Task.currentPriority)")
    }
}

// Print "Outer task priority: TaskPriority.background"
// Print "Inner task priority: TaskPriority.background"
``` 


## Fully Independent Detached Tasks

`Detached Task`는 현재 작업 컨텍스트와 완전히 분리된 최상위(top-level) 작업 컨텍스트를 생성합니다. 이 작업은 생성된 작업 컨텍스트로부터 어떠한 상태도 상속받지 않으며, 완전히 독립적인 환경에서 실행됩니다. 다시 말해, `Detached Task`는 생성 시점의 외부 `Task`로부터 액터, 작업 우선순위나 태스크-로컬 등 컨텍스트를 상속받지 않고 를  어느 컨텍스트로 상속받지 않으며, 완전히 분리(detached)되어 깨끗한 상태에서 시작하게 됩니다. 

```swift
Task(priority: .background) { 
    print("외부 작업의 우선순위: \(Task.currentPriority)")
    
    Task.detached { 
        print("내부 작업의 우선순위: \(Task.currentPriority)")
    }
}

// Print "외부 작업의 우선순위: TaskPriority.background"
// Print "내부 작업의 우선순위: TaskPriority.medium"
```

`Detached Task`는 명시적으로 작업 우선순위나 컨텍스트를 지정해야 하는 경우에 유용하게 사용됩니다. 이 작업은 외부 작업의 컨텍스트로부터 완전히 분리되어 독립적으로 실행되기 때문에, 로깅, 캐시 저장 등과 같이 외부 상태에 영향을 받지 않고 독립적으로 처리되어야 하는 작업에 적합합니다.
 

```swift
@MainActor
extension MyDelegate: UICollectionViewDelegate {
    public func collectionView(_ view: UICollectionView,
                               willDisplay cell: UICollectionViewCell,
                               forItemAt item: IndexPath) {
        let ids = getThumbnailIDs(for: item)
        thumbnailTasks[item] = Task {
            defer { thumbnailTasks[item] = nil }
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

위 예제에서 `collectionView(_:willDisplay:)` 메서드는 셀이 화면에 표시되기 직전에(willDisplay) 해당 셀에 보여줄 썸네일 이미지를 서버로부터 비동기적으로 불러옵니다. 썸네일을 불러오는 비동기 작업은 `Task`를 사용해 실행되며, 해당 작업에 대한 참조는 `thumbnailTasks` 딕셔너리에 저장되어, 셀이 화면에서 사라질 경우 필요에 따라 작업을 취소할 수 있습니다.

썸네일을 성공적으로 불러온 후, 해당 이미지를 로컬 캐시에 저장하는 작업은 `Detached Task`를 통해 실행됩니다. `writeToLocalCache(_:)`는 디스크 I/O와 같이 비교적 무거운 백그라운드 작업에 해당되며, UI 작업과 직접적인 연관되지 않기 때문에 별도의 작업 컨텍스트에서 독립적으로 처리하는 것이 적절합니다. 이를 `Detached Task`로 분리 실행하면, 메인 스레드의 작업 흐름을 방해하지 않으며서 앱의 전반적인 성능을 향상시킬 수 있습니다.

아래 표는 `Task`와 `Detached Task`를 비교한 것입니다.

|   | **Task** | **Detached Task** |
| - | - | - |
| 취소 전파 | ❌ | ❌ |
| 컨텍스트 상속(액터, 작업 우선순위, 태스크-로컬 등) | ✅ | ❌ |
| 활용 | - 동기 코드에서 비동기 코드를 실행할 수 있는 비동기 컨텍스트를 생성할 때<br>- 작업의 생명 주기가 특정 코드 블록의 범위를 벗어나야 할 때 | - 외부 작업의 컨텍스트에 의존하지 않고, 완전히 독립적으로 실행되어야 할 때 |

<!-- 여기서부터 다시 검토 -->
# Returning Results from a Task

`Task`는 작업이 완료되면 결과값을 반환할 수 있으며, 이 값은 `Task`의 `value` 프로퍼티를 통해 접근할 수 있습니다. 결과값을 비동기적으로 생성되기 때문에, `value` 프로퍼티를 사용할 때는 반드시 `await` 키워드를 함께 사용해야 합니다. 또한, 작업 도중 예외가 발생할 수 있는 경우에는 `try` 키워드도 함께 작성해야 합니다. 

```swift
let task = Task { () throws -> String in
    return try await fetchTitle()
}

let title = try await task.value
```

참고로, `Task`가 반환하는 값은 동시 컨텍스트에서 안전하게 전달될 수 있는 `Sendable`한 값이어야 합니다. 이는 데이터 경합(data race)없이 다른 스레드에 안전하게 값을 전달하기 위함입니다.

{% hint style="info" %}
**Info** `Sendable` 프로토콜은 데이터 경합의 위험 없이 서로 다른 동시 컨텍스트(스레드) 간에 안전하게 공유할 수 있는 타입을 나타냅니다. ~~자세한 내용은 [Sendable](programming-guide/Sendable.md) 문서를 참조하세요.~~
{% endhint %}


# Task Priority

`Task`에 우선순위를 설정할 수 있습니다. 아래 표는 사용할 수 있는 우선순위 값들을 보여줍니다.

| 종류 | 원시값 |
| - | - |
| `userInitiated` | 25 |
| `high` | 25 |
| `medium` | 21 |
| `low` | 17 |
| `utility` | 17 |
| `background` | 9 |

```swift
Task(priority: .userInitiated) { ... }
Task(priority: .high) { ... }
Task(priority: .medium) { ... }
Task(priority: .low) { ... }
Task(priority: .utility) { ... }
Task(priority: .background) { ... }
```

## Priority Escalation to Prevent Inversion

Swift 런타임은 작업 간 우선순위 역전(Priority Inversion)을 방지하기 위해, 일부 상황에서 작업의 우선순위를 일시적으로 승격시킬 수 있습니다. 

예를 들어, 높은 우선순위의 외부 `Task`에서 내부 `Task`의 결과값을 `value`를 통해 기다리는 경우, 런타임은 해당 결과를 계산 중인 내부 `Task`의 우선순위를 외부 `Task`의 우선순위 수준으로 일시적으로 승격시킵니다. 이를 통해 외부 `Task`가 빠르게 결과를 받을 수 있도록 하며, 우선순위 역전을 방지합니다.

```swift
let outerTask = Task(priority: .high) {
    print("Outer task priority: \(Task.currentPriority)")
    
    let innerTask = Task(priority: .low) {
        print("Inner task priority (before sleep): \(Task.currentPriority)")
        
        try? await Task.sleep(for: .seconds(1))
        
        print("Inner task priority (after sleep): \(Task.currentPriority)")
    }
    
    // The high-priority outer task waits for the result of the low-priority inner task.
    try? await Task.sleep(for: .seconds(0.5))
    _ = try? await innerTask.value
}

_ = try? await outerTask.value

Print "Outer task priority: TaskPriority.high"
Print "Inner task priority (before sleep): TaskPriority.low"
Print "Inner task priority (after sleep): TaskPriority.high"
```

또 다른 예로, 액터가 낮은 우선순위의 작업을 실행 중일 때, 더 높은 우선순위의 작업이 해당 액터에 접근을 시도하면, 현재 실행 중인 작업의 우선순위가 일시적으로 승격될 수 있습니다. 액터는 한 번에 하나의 작업만 처리할 수 있기 때문에, 낮은 우선순위의 작업이 액터의 큐(queue) 앞에 위치해 실행 중이라면, 이후에 도착한 높은 우선순위의 작업은 즉시 실행되지 못하고 대기하게 됩니다. 이러한 상황에서 런타임은 현재 실행 중인 작업의 우선순위를 일시적으로 승격시켜, 높은 우선순위의 작업이 불필요하게 지연되지 않도록 합니다.

{% hint style="info" %}
**Info** 액터는 재진입성(Re-Entrancy)을 통해 작업 간 우선순위를 보다 유연하게 조정할 수도 있습니다. ~~자세한 내용은 [Actor]() 문서를 참조하세요.~~
{% endhint %}


# Task does not require [weak self]

`Task`를 생성할 때, `operation` 클로저에서 `self`를 강하게 참조한다 하더라도 일반적으로 문제가 되지 않습니다. `Task` 인스턴스의 참조 여부와 별개로, 작업이 완료되면 해당 클로저에서 캡처된 모든 참조(`self`)들은 자동으로 해제됩니다. 이로 인해, `Task`의 `operation` 클로저 안에서 `self`를 `[weak self]`로 약하게 캡처할 필요는 없습니다.

```swift
struct Work: Sendable { }

actor Worker {
    var work: Task<Void, Never>?
    var result: Work?
    
    deinit {
        // even though the task is still retained,
        // once it completes it no longer causes a reference cycle with the actor
        
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

Print "start task work"
Print "completed task work"
Print "deinit actor"
```

예를 들어, 위 예제에서 `start()` 메서드에서 생성된 `Task`는 `Worker` 액터를 강하게 참조합니다. 동시에 `Worker` 액터도 `work` 프로퍼티를 통해 해당 `Task`를 강하게 참조하고 있습니다. 일반적으로 이러한 상호 참조는 순환 참조(retain cycle)을 유발할 수 있지만, 작업이 완료되면 `Task` 내부에서 캡처한 모든 참조는 자동으로 해제되므로 순환 참조가 발생하지 않습니다.

{% hint style="info" %}
**Note** `Task`는 생성 시점에 `self`를 암시적으로 캡처합니다. 따라서 `operation` 클로저 내부에서 `self.fetchData()`처럼 명시적으로 `self`를 작성할 필요가 없습니다.

```swift
class Manager {
    func load() {
        Task {
            fetchData() // 'self' is implicitly captured
        }
    }
}
```
{% endhint %}


# Isolating a Task to a Specific Actor

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}


# Controlling Task Execution Context with Executor Preferences

<!--실행자(executor)는 비동기 작업이 어떤 스레드(또는 큐)에서 실행될지, 어떤 우선순위를 가질지, 그리고 어떤 순서로 처리될지를 조율하는 실행 컨텍스트입니다. 실행자는 작업을 특정 스레드에 직접 할당하진 않지만, 어떤 실행 환경에서 작업이 실행되어야 하는지를 정의합니다. 또한, 주어진 우선순위에 따라 더 중요한 작업을 우선적으로 실행할 수 있도록 하며, 직렬 실행자의 경우에는 작업 간의 실행 순서를 제어하여, 데이터 충돌 없이 안전하게 실행되도록 합니다. Swift는 기본적으로 전역 동시 실행자(global concurrent executor)와 직렬 실행자(serial executor)를 제공합니다.-->

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}


## Implementing a Custom Task Executor

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}



---

# [Appendix] Terminology Comparison: Task, Async, and Execution Contexts

| 항목 | 정의 | 목적 | 예시 |
| :-: | :- | :-  | :- |
| 작업 컨텍스트(Task Context) | `Task`가 실행되는 동안 유지되는 작업의 내부 상태와 메타데이터를 포함한 실행 환경 | 작업의 우선순위, 취소 상태, Task-Local 값을 추적하고 전달 | `Task.currentPriority`, `Task.isCancelled`, `Task-Local` | 
| 실행 컨텍스트(Execution Context) | 코드가 실제로 실행되는 스레드 또는 실행자(executor) 환경 | 작업이 어떤 실행 환경에서 실행될지를 결정하고, 실행 순서와 우선순위를 조율 | `@MainActor`, `GlobalConcurrentExecutor`, `SerialExecutor` |
| 비동기 컨텍스트(Asynchronous Context) | `await` 키워드를 사용할 수 있는 코드 블록 또는 비동기 함수 내부 | 비동기 함수를 호출할 수 있도록 하는 문법적 환경을 제공 | 비동기 함수 내부, `Task`, `addTask { .. }` 등 |
| 동시 컨텍스트(Concurrency Context) | 여러 작업이 동시에 실행될 수 있도록 설정된 실행 환경 | 데이터에 동시에 접근할 수 있는 상황을 관리하고, 동시적 안전성을 확보 | 액터(Actor), 작업그룹(TaskGroup) 등 |

* **컨텍스트(Context):** 코드가 실행되는 동안 런타임이 제공하는 실행 환경 및 관련 정보들의 집합
{% endhint %}
