---
description: 특정 작업 컨텍스트 내에 바인딩하고 읽을 수 있는 값
---

태스크-로컬(Task-Local) 값은 작업(Task) 컨텍스트 내에서 바인딩하고 읽을 수 있는 값입니다. 이 값은 바인딩된 특정 범위 내에서 생성된 모든 작업에 암시적으로 전달되며, 해덩 작업이 생성하는 (_TaskGroup_이나 _async-let_에서 생성하는 작업처럼) 모든 하위 작업(Child Tasks)에서도 접근할 수 있습니다. 태스크-로컬은 특정 범위 내에서만 의미가 있는 값을 전달할 때 유용하며, 전역 상태를 사용하지 않고도 작업 컨텍스트에 따라 달라지는 값을 안전하게 전파할 수 있도록 도와줍니다.


# Task-Local Declarations

태스크-로컬은 반드시 정적(Static) 프로퍼티나 전역 프로퍼티로 선언되어야 합니다.

```swift
enum Example {
    @TaskLocal
    static var traceID: TraceID?
}

// Global task local properties are supported since Swift 6.0:
@TaskLocal
var contextualNumber: Int = 12
```

# Task-Local Default Values

태스크-로컬 값은 명시적으로 값을 바인딩하지 않은 상태에서 읽으면 기본값을 반환합니다. 예를 들어, 태스크-로컬이 `TraceID?`처럼 옵셔널 타입으로 선언되어 있다면, 기본값은 자동으로 `nil`이 됩니다. 하지만 아래와 같이 태스크-로컬 선언 시 기본값을 직접 지정할 수도 있습니다.

```swift
enum Example {
    @TaskLocal
    static var traceID: TraceID = TraceID.default
}
```

기본값은 현재 작업이나 상위 작업 컨텍스트에 바인딩된 값이 없거나, 호출 스택(Call Stack)에 어떤 비동기 함수도 없는 동기 함수에서 태스크-로컬 값을 읽으려 할 때 반환됩니다.


# Reading Task-Local Values 

태스크-로컬 값을 읽는 방법은 일반적인 정적 프로퍼티를 읽는 것과 동일하게 간단합니다.

```swift
guard let traceID = Example.traceID else {
    print("no trace id")
    return
}
print(traceID)
```

태스크-로컬 값은 비동기 함수뿐만 아니라 동기 함수에서도 읽을 수 있습니다. 다만, 작업 컨텍스트 외부의 동기 함수에서는 기본값이 반환된다는 점에 유의해야 합니다.


# Binding Task-Local Values

태스크-로컬 값은 직접 할당(set)할 수 없으며, 반드시 `withValue<R>(_:operation:)` 메서드를 통해 바인딩해야 합니다. 이 값은 바인딩된 범위 내에서만 유효하며, 해당 범위 내에에서 생성된 모든 (하위) 작업에서 접근할 수 있습니다. 

```swift
func read() {
    print("traceID: \(Example.traceID)")
}

Task {
    await Example.$traceID.withValue(1234) { // bind the value
        print("traceID: \(Example.traceID)") // traceID: 1234
        read() // traceID: 1234
        
        async let id = read() // async let child task, traceID: 1234
        
        await withDiscardingTaskGroup { group in
            group.addTask { read() } // task group child task, traceID: 1234
        }
        
        Task { // unstructured tasks do inherit task locals by copying
            read() // traceID: 1234
        }
        
        Task.detached { // detached tasks do not inherit task-local values
            read() // traceID: nil
        }
    }
}
```

독립적인 작업 컨텍스트를 생성하는 _Detached Task_는 태스크-로컬 값을 상속하지 않습니다. 반면, _Task_로 생성된 작업은 구조화되지 않은 작업(Unstructured Task)이더라도, 현재 작업 컨텍스트의 태스크-로컬 값을 복사하는 방식으로 상속합니다.


## Accessing Task-Local Values Outside of a Task

태스크-로컬 값은 작업 외부에서도 바인딩하고 읽을 수 있습니다.

이는 작업 내에서 호출된다고 보장되지 않는 동기 함수에서 특히 유용합니다. 작업 외부에서 태스크-로컬 값을 바인딩하는 경우, 런타임은 작업과 동일한 저장 메커니즘을 사용하는 스레드-로컬(Thread-Local)을 설정하여 동작합니다. 즉, 아래와 같은 예제처럼 호출 컨텍스트를 신경 쓰지 않고도, 태스크-로컬 값을 안정적으로 바인딩하고 읽을 수 있습니다.

```swift
func enter() {
    Example.$traceID.withValue(1234) { 
        print("traceID: \(Example.traceID)") // always "1234", regardless if enter() was called from inside a task or not:
    }
}

// 1) Call `enter` from non-Task code
//       e.g synchronous main() or non-Task thread (e.g. a plain pthread)
enter()

// 2) Call `enter` from Task
Task {
    enter()
}
```
