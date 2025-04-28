---
description: 데이터 경합의 위험 없이 서로 다른 동시 컨텍스트(스레드)에서 안전하게 공유할 수 있는 타입
---

동시성 코드를 작성할 때 근본적으로 어려운 문제 중 하나는 데이터 경합(Data Race)을 피하는 것입니다. 데이터 경합은 두 개 이상의 스레드(또는 작업)가 동시에 동일한 공유 가변 상태에 접근하고, 그 중 하나가 쓰기 작업을 수행할 때 발생합니다.

데이터 경합은 클래스와 같은 참조 타입(Reference Type)에서 쉽게 발생할 수 있습니다. 참조 타입은 여러 위치에서 동일한 인스턴스를 공유할 수 있기 때문에, 한 스레드에서 데이터를 수정하면 그 변경 사항이 다른 스레드에도 영향을 미칠 수 있습니다. 이로 인해, 여러 스레드가 동시에 같은 데이터에 접근하거나 수정할 경우, 데이터 경합이 발생하기 쉽습니다. 반면, 값 타입(Value Type)은 복사 기반으로 동작하기 때문에, 이러한 동시성 문제에서 상대적으로 자유롭습니다. 한 스레드가 데이터를 수정하면 그 변경사항이 다른 스레드에 영향을 주지 않기 때문에, 병렬 환경에서 안전하게 사용할 수 있습니다. 이러한 특성 덕분에 값 타입은 동시성 환경에서 더욱 안정적이며, Swift가 일관되게 값 타입의 사용을 권장해온 이유이기도 합니다.

만약 구조체나 열거형과 같은 값 타입이나 동기화 메커니즘을 스스로 구현한 일부 클래스처럼 스레드 간 안전하게 공유할 수 있는 타입에 특별한 표식을 부여할 수 있다면 어떨까요? 그리고, 이 표식을 가진 타입만 서로 다른 스레드 간에 공유할 수 있도록 컴파일 타임에 강제할 수 있다면 어떨까요? 그렇게 하면, 이 표식이 없는 타입이 공유될 경우, 컴파일 타임에서 이를 감지하고 차단할 수 있습니다. 개발자는 복잡한 디버깅 없이도 데이터 경합이 발생할 수 있는 위험한 코드를 빠르게 찾아내고 수정할 수 있을 것입니다. 바로 이러한 아이디어에서 출발한 것이 `Sendable` 프로토콜입니다.


# Sendable Types for Safe Sharing Across Concurrency Domains

`Sendable`은 데이터 경합의 위험 없이 서로 다른 동시 컨텍스트(Concurrent Context) 간에 안전하게 값을 공유할 수 있는지를 나타내는 프로토콜입니다. Swift 컴파일러는 `Sendable`을 따르지 않는 값이 서로 다른 스레드 간에 전달되려는 시도를 감지하면 컴파일 타임에 오류를 발생시켜 안전하지 않은 동시성 코드를 사전에 차단합니다. `Sendable`은 요구하는 메서드나 프로퍼티가 없는 마커 프로토콜(Marker Protocol)이며, 단지 해당 타입이 동시성 환경에서 안전하게 사용될 수 있음을 명시하는 역할을 합니다.


## Unchecked Sendable for Disabling Concurrency Safety Checks

개발자가 특정 타입이 동시성 환경에서 안전하게 동작한다고 직접 보장할 수 있는 경우, `@unchecked Sendable`을 사용해 컴파일러의 동시성 검사를 비활성화할 수 있습니다. 이 속성을 사용하면 컴파일러가 헤딩 타입의 스레드 안전성을 더 이상 검증하지 않으며, 그 책임은 전적으로 개발자에게 전가됩니다. 부주의하게 사용할 경우, 데이터 경합과 같은 동시성 문제를 초래할 수 있습니다.

`@unchecked Sendable`은 주로 락(NSLock), 세마포어(DispatchSemaphore), 직렬 디스패치 큐(Serial Dispatch Queue)와 같은 동기화 메커니즘을 타입 내부에 직접 구현한 경우에 사용됩니다. 이러한 경우, 컴파일러는 내부 구현이 스레드에 안전한지 직접 판단할 수 없기 때문에, 해당 타입이 동시성 환경에서도 안전하다는 것을 보장한다는 의미로 `@unchecked Sendable`을 선언하게 됩니다.

```swift
final class Counter: @unchecked Sendable {
    private let lock = NSLock()
    var value = 0
    
    func increment() {
        lock.lock()
        value += 1
        lock.unlock()
    }
}
```



# Sendable Compliance Rules for Structs, Classes and Enums

모든 타입이 `Sendable`할 수 있는 건 아닙니다. 구조체나 열거형처럼 값이 복사되어 전달되는 값 타입은 몇 가지 간단한 요건만 충족하면 `Sendable`이 될 수 있습니다. 반면, 클래스처럼 참조를 기반으로 동작하는 참조 타입은 `Sendable`이 되기 위해 더 엄격한 조건을 충족해야 합니다. 이제 액터(Actor), 구조체, 클래스 등 타입별로 어떻게 `Sendable`이 될 수 있는지 살펴보겠습니다.


## Actor

액터는 서로 다른 작업 컨텍스트(Task Context)에서 동일한 데이터에 안전하게 동시에 접근할 수 있도록 설계된 새로운 타입입니다. 액터의 내부 상태는 프로그램의 나머지 부분으로부터 격리(Isolated)되며, 해당 상태에 대한 모든 접근은 반드시 액터를 통해서만 이루어져야 합니다. 이러한 격리 특성 덕분에, 액터 타입은 기본적으로 `Sendable`합니다.


## Struct And Enumeration

구조체나 열거형이 `Sendable`하려면, 모든 저장 프로퍼티나 연관 값(Associated Value)이 `Sendable`이어야 합니다. 구조체와 열거형이 `@frozen`으로 선언되어 있거나, 접근 수준이 `public`이 아니며 `@usableFromInline` 속성도 없다면, Swift 컴파일러는 해당 타입을 암시적으로 `Sendable`하다고 간주할 수 있습니다. 

```swift
struct User: Sendable {
    let id: Int
    var name: String
}
```

위 예제에서 `id`와 `name`은 각각 `Sendable`한 `Int`와 `String` 타입으로 구성되어 있으므로, `User` 구조체도 암시적으로 `Sendable`로 간주될 수 있습니다.

한편, 저장 프로퍼티나 연관 값 중에 `Sendable`하지 않는 타입이 포함된 구조체나 열거형에서 `Sendable`을 명시하면, 컴파일러는 오류를 발생시킵니다. 하지만, 해당 타입이 동시성 환경에서 안전하다는 것을 개발자가 직접 보장할 수 있는 경우, `@unchecked Sendable`을 명시하여 컴파일러의 동시성 검사를 비활성화할 수 있습니다.


## Class

대부분의 클래스는 기본적으로 `Sendable`하지 못합니다. 하지만, 아래와 같이 매우 제한적인 조건을 모두 충족하는 경우에 한해, 클래스에 `Sendable`하다고 명시할 수 있습니다.

- 클래스가 `final`로 선언되어 상속이 불가능해야 합니다.

- 모든 저장 프로퍼티가 `Sendable`이고, 동시에 불변(Immutable)이어야 합니다.

- 상위 클래스가 없거나, `NSObject`만을 상속받고 있어야 합니다.

클래스를 `final`로 선언해야 하는 이유는 외부에서 해당 클래스를 상속받아 새로운 저장 프로퍼티를 추가할 수 있기 때문입니다. 만약 상속이 허용된다면, 해당 타입이 스레드 간에 안전하게 공유될 수 있는지를 컴파일 타임에 정확히 예측할 수 없게 됩니다.

```swift
final class Author: Sendable {
    let name: String
}
```

위 조건을 충족하지 못하더라도 락(NSLock), 세마포어(DispatchSemaphore), 직렬 디스패치 큐(Serial Dispatch Queue)와 같은 동기화 메커니즘을 타입 내부에 직접 구현한 경우에는 `@unchecked Sendable`을 명시할 수 있습니다. 이 속성을 사용하면 컴파일러가 헤딩 타입의 스레드 안전성을 더 이상 검증하지 않으며, 그 책임은 전적으로 개발자에게 전가됩니다.

```swift
final class SafeDict<Key, Value>: @unchecked Sendable where Key: Hashable & Sendable, Value: Sendable {
    private let queue = DispatchQueue(label: "serial.queue")
    private var dict: [Key: Value] = [:]
    
    func set(_ key: Key, value: Value) {
        queue.async { self.dict[key] = value }
    }
    func get(_ key: Key) -> Value? {
        queue.sync { return self.dict[key] }
    }
}
```

한편, 메인 액터(MainActor)에 격리된 클래스는 객체의 내부 상태에 대한 모든 접근이 항상 메인 스레드에서 순차적으로 일어나도록 보장되기 때문에, 컴파일러는 해당 클래스를 암시적으로 `Sendable`하다고 간주할 수 있습니다. 일반적인 타입과는 달리, `Sendable`하지 않는 프로퍼티가 포함되어 있더라도, 컴파일러는 이를 예외적으로 허용합니다.

## Tuple

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}


## Generic

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}


## Metatypes

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}


# Capturing Rules for @Sendable Closures

클로저(함수)도 `Sendable`이 될 수 있습니다. 하지만, 클로저는 일반적인 타입과 달리, 직접적으로 프로토콜을 준수할 수 없기 때문에, Swift에서는 이를 위해 `@Sendable`이라는 특별한 속성을 제공합니다. `@Sendable`은 함수 타입 선언이나 클로저의 매개변수 앞에 붙이며, 해당 클로저가 캡처하는 모든 값이 동시성 환경에서도 안전하게 공유될 수 있음을 의미합니다.

클로저는 액터의 격리 경계(Isolation Boundary)를 넘어 다양한 실행 컨텍스트에서 호출될 수 있습니다. 이는 클로저가 캡처한 값이 액터 외부로 전달되고, 여러 실행 컨텍스트에서 동시에 접근될 수 있음을 의미합니다. 이러한 경우, 동시 실행 시 스레드 안전성이 보장되지 않을 수 있으며, 데이터 경합과 같은 동시성 문제를 초래할 수 있습니다.

{% hint style="info" %}
**Note** 클로저가 캡처한 모든 값은 힙(heap) 메모리 영역에 저장됩니다.
{% endhint %}

 `@Sendable`한 클로저는 값 기반 캡처(By-Value Capture)만 허용합니다. 그리고, 캡처하는 모든 값은 `Sendable`해야 하며, 불변이어야 합니다. 
 
 이는 `Sendable`하지 않은 참조가 액터의 격리 경계를 넘어 전파되는 것을 방지하는데 중요한 역할을 합니다. 예를 들어, 참조 타입인 클래스는 기본적으로 `Sendable`하지 않기 때문에, `@Sendable` 클로저에서 이를 직접 캡처하지 못합니다. 이러한 제한이 존재하는 이유는, 참조가 서로 다른 실행 컨텍스트에서 동시에 접근될 경우, 데이터 경합과 같은 동시성 문제로 이어질 수 있기 때문입니다. 결과적으로, `@Sendable`은 클로저가 동시성 환경에서 안전하게 실행될 수 있도록 보장합니다. 

```swift
class Person { ... }

actor Dog {
    let name: String
    var degreeOfHappiness: Int
    var owner: Person?

    func getSendableClosure() -> (@Sendable () -> Void) {
        return {
            _ = self.name
            _ = self.degreeOfHappiness // 🔴 오류: Actor-isolated property 'degreeOfHappiness' can not be referenced from a Sendable closure
            _ = self.owner // 🔴 오류: Actor-isolated property 'owner' can not be referenced from a Sendable closure
        }
    }
}
```

위 예제에서 `@Sendable` 클로저는 `Sendable`하지만 가변 프로퍼티인인 `degreeOfHappiness`와 아예 `Sendable`하지 않는 타입인 `owner`를 캡처할 수 없습니다. 

아울러, `degreeOfHappiness`와 `owner`는 모두 액터에 격리되었으며, `@Sendable`한 클로저 내부에서 동기적으로 접근하려 할 경우 컴파일 오류가 발생합니다. 이는 `@Sendable`한 클로저가 액터의 격리 컨텍스트를 상속하지 않기 때문입니다. 즉, `Dog` 액터 내부에서 정의되었더라도, 해당 클로저는 액터와 무관한 독립적인 실행 컨텍스트에서 실행될 수 있는 코드로 취급됩니다. 따라서, 액터에 격리된 상태에 접근하려면 반드시 `await`을 통해 비동기적으로 접근해야 합니다. 

```swift
func getSendableClosure() -> (@Sendable () async -> Void) {
    return {
        _ = self.name
        _ = await self.degreeOfHappiness
        _ = await self.owner // 🔴 오류: Non-sendable type 'Person?' of property 'owner' cannot exit actor-isolated context
    }
}
```

위 예제와 같이 `Sendable`한 가변 프로퍼티인 `degreeOfHappiness`는 `await`을 통해 안전하게 접근할 수 있습니다. 그러나, `Sendable`하지 않는 타입인 `owner`는 `await`을 통해 접근하더라도 액터 외부로 값을 전달할 수 없기 때문에 컴파일 오류가 발생합니다.


지금까지 살펴본 대부분의 동시성 예제는 사실상 `@Sendable` 속성에 의존하고 있었습니다. 작업(Task)이나 하위 작업(Child Task)을 생성할 때, Swift는 항상 `@Sendable`한 클로저를 요구합니다.

아래 예제는 작업을 생성하는 대표적인 API들입니다.

```swift
Task.init(
  priority: TaskPriority? = nil,
  operation: @Sendable @escaping @isolated(any) () async -> Success
)

static func detached(
  priority: TaskPriority? = nil,
  operation: @Sendable @escaping @isolated(any) () async throws -> Success
) -> Task<Success, Failure>

mutating func addTask(
  priority: TaskPriority? = nil,
  operation: @Sendable @escaping @isolated(any) () async -> ChildTaskResult
)
```

이처럼 작업을 실행하는 클로저는 `@Sendable`하기 때문에, 클로저 내부에서는 항상 `Sendable`한 값만 캡처할 수 있습니다. 이로 인해 클로저가 서로 다른 작업 컨텍스트에서 동시에 실행되더라도, 데이터 경합과 같은 동시성 문제 없이 안전한 동시 실행을 가능해집니다.

{% hint style="info" %}
**Note** Swift 6.0부터는 `@Sendable` 키워드가 `sending`으로 변경되었습니다. ~~`sending`에 관한 자세한 내용은 [지역 기반 격리]()를 참조하세요.~~
{% endhint %}
