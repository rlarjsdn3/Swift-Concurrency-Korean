---
description: 서로 다른 동시 컨텍스트(스레드)에서 안전하게 공유할 수 있는 타입
---

Swift Concurrency는 기존 동시성 프로그래밍 모델의 한계를 극복하고, 더욱 안전하고 효율적인 코드 작성을 목표로 설계되었습니다. 특히, 데이터 경합(Data Race)과 같은 동시성 문제를 컴파일 타임에 감지하고 예방함으로써, 개발자가 높은 성능의 동시성 코드를 보다 신뢰성있게 작성할 수 있도록 지원합니다.

데이터 경합(Data Race)은 여러 스레드가 동일한 공유 가변 상태(Shared Mutable State)에 접근하고, 그 중 하나 이상의 스레드가 쓰기 작업을 수행할 때 발생합니다. 특히, 동일한 인스턴스를 여러 곳에서 참조할 수 있는 클래스와 같은 참조 타입(Reference Type)은 어느 한 스레드가 데이터를 수정하면 다른 스레드에 영향을 미칠 수 있으므로 데이터 경합이 발생하기 쉬운 구조입니다. 반면, 값이 복사되는 구조체나 열거형과 같은 값 타입(Value Type)은 데이터 경합 문제에서 상대적으로 자유롭다고 볼 수 있습니다. 값 타입은 복사 기반으로 전달되기 때문에, 어느 한 스레드가 데이터를 수정하더라도 그 변경이 다른 스레드에 영향을 미치지 않습니다. 이러한 특성 덕분에 값 타입은 동시성 환경에서 훨씬 안전하며, Swift가 일관되게 값 타입의 사용을 권장해온 이유이기도 합니다.

만약 구조체나 열거형과 같은 값 타입, 혹은 자체적인 동기화 메커니즘이 구현된 일부 클래스처럼 스레드 간 안전하게 공유할 수 있는 타입에 특별한 표식을 부여하고, 서로 다른 스레드 간에는 이러한 표식을 가진 타입만 공유할 수 있도록 강제한다면 어떨까요? 그렇게 하면 컴파일 타임에 이 표식이 없는 타입의 공유 시도를 차단할 수 있고, 개발자는 복잡한 디버깅 과정 없이도 데이터 경합의 가능성이 높은 코드를 빠르게 식별하고 수정할 수 있을 것입니다. 바로 이 아이디어에서 출발해 등장한 것이 `Sendable` 프로토콜입니다.


# Sendable

`Sendable`은 서로 다른 동시 컨텍스트(Concurrent Context) 간에 데이터 경합의 위험 없이 안전하게 값을 공유할 수 있는지를 검증하는 마커 프로토콜(Marker Protocol)입니다. Swift 컴파일러는 `Sendable`을 따르지 않는 값이 서로 다른 동시 컨텍스트 간에 주고받으려는 시도를 감지하면, 컴파일 타임에 오류를 발생시켜 개발자에게 이를 즉시 알려줍니다. 이 프로토콜은 구현해야 할 메서드나 프로퍼티가 없으며, 해당 타입이 동시성 환경에서도 안전하게 사용될 수 있음을 나타내는 역할을 합니다.

## @unchecked Sendable

어떤 타입이 동시성 환경에서 안전하게 동작한다고 개발자가 판단하는 경우, `@unchecked Sendable`을 사용해 컴파일러의 동시성 검사를 비활성화할 수 있습니다. 이 속성은 컴파일러가 타입의 스레드 안전성을 검증하지 않기 때문에, 반드시 개발자의 책임 하에 제한적으로 사용해야 합니다. 부주의하게 사용할 경우, 데이터 경합 등 동시성 문제를 초래할 수 있으므로 주의가 필요합니다.

`@unchecked Sendable`은 주로 `NSLock`, `DispatchSemaphore`, 직렬 디스패치 큐와 같은 동기화 메커니즘을 타입 내부에 직접 구현한 경우에 사용됩니다.

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



# 타입별 Sendable 준수 방식

모든 타입이 `Sendable` 프로토콜을 준수할 수 있는 건 아닙니다. 값이 복사되는 구조체나 열거형과 같은 값 타입은 사소한 요건만 충족한다면 `Sendable`이 될 수 있지만, 참조의 복사가 일어나는 클래스와 같은 참조 타입이 `Sendable`이 되기 위해선 다소 까다로운 요건을 충족해야 합니다. 액터(Actor)부터 구조체, 클래스에 이르기까지 타입별로 `Sendable` 프로토콜의 준수 방식을 살펴보겠습니다. 


## Actor

액터는 공유 가변 상태에 대한 동기화 메커니즘을 제공하는 특별한 참조 타입입니다. 액터의 내부 상태는 프로그램의 나머지 부분으로부터 *격리(Isolated)되며, 해당 상태에 대한 모든 접근은 반드시 액터를 통해서만 이루어져야 합니다. 이러한 격리 특성 덕분에, 액터 타입은 기본적으로 `Sendable` 프로토콜을 자동으로 준수합니다.


## Struct And Enumeration

구조체나 열거형이 `Sendable` 프로토콜 준수하려면, 모든 저장 프로퍼티나 연관 값(Associated Value)이 `Sendable`이어야 합니다. 구조체와 열거형이 `frozen` 상태이거나, 접근 제어자가 `public`이 아니며 `@usableFromInline` 속성이 없다면, Swift 컴파일러는 이 타입을 암시적으로 `Sendable`이라고 마킹합니다.

```swift
struct User: Sendable {
    let id: Int
    let name: String
}
```

위 예제에서 `Int`와 `String` 타입은 모두 `Sendable`을 준수하기 때문에, `User` 구조체도 별도의 구현 없이 암시적으로 `Sendable`로 간주됩니다.

한편, `Sendable`을 따르지 않는 저장 프로퍼티나 연관 값이 있는 구조체나 열거형이라면 컴파일러는 오류를 발생시킵니다. 하지만 개발자가 해당 타입이 동시성 환경에서 안전하다고 판단하는 경우, `@unchecked Sendable`을 명시적으로 마킹하여 컴파일 동시성 검사를 비활성화할 수 있습니다.


## Class

대부분의 클래스는 `Sendable`을 준수하지 못합니다. 다만, 아래와 같이 아주 제한적인 조건을 모두 충족하는 경우에 한해 명시적으로 `Sendable`을 채택할 수 있습니다.

- 클래스가 `final`로 선언되어 상속이 불가능한 경우

- 모든 저장 프로퍼티가 `Sendable`이고, 불변(Immutable)인 경우

- 상위 클래스가 없거나, `NSObject`만을 상속받고 있는 경우

클래스에 `final` 키워드를 사용하는 이유는 만약 외부에서 해당 클래스를 상속받아 새로운 저장 프로퍼티나 동작이 추가되면 원래의 `Sendable` 조건이 깨질 수 있기 때문입니다. 즉, 상속이 허용되면 해당 타입의 스레드 안전성을 컴파일 타임에 예측할 수 없게 되므로, `Sendable`을 안전하게 보장하려면 반드시 `final` 클래스로 제한해야 합니다.

```swift
final class Author: Sendable {
    let name: String
}
```

위 예제는

한편, `@MainActor`로 마킹된 클래스는 객체 상태에 대한 모든 접근이 메인 스레드에서 일어나도록 보장되기 때문에, 암시적으로 `Sendable`로 간주될 수 있습니다. 이 경우 해당 클래스가 `Sendable`을 따르지 않거나, 가변 저장 프로퍼티를 가지고 있어도 문제가 되지 않습니다.

위 조건을 충족하지 않는 클래스라도, 내부에 _직렬 디스패치 큐_, `NSLock`, `DispatchSemaphore` 등의 동기화 메커니즘을 직접 구현한 경우에는 (개발자가 타입의 스레드 안전성을 스스로 보장한다는 전제 하에) `@unchecked Sendable`을 명시적으로 마킹할 수 있습니다.

```swift
final class SafeDict<Key, Value>: @unchecked Sendable where Key: Hashable & Sendable, Value: Sendable {
    private let queue = DispatchQueue(label: "serial.queue")
    private var dict: [Key: Value] = [:]
    
    init() { }
    func set(_ key: Key, value: Value) {
        queue.async { self.dict[key] = value }
    }
    func get(_ key: Key) -> Value? {
        queue.sync { return self.dict[key] }
    }
}

```

## Tuple

{% hint style="info" %}
**Information** 이 섹션은 현재 작성 중입니다.
{% endhint %}


## Generic

{% hint style="info" %}
**Information** 이 섹션은 현재 작성 중입니다.
{% endhint %}



# @Sendable 클로저 속성

함수(또는 클로저)도 `Sendable`이 될 수 있습니다. 다만, 함수 타입은 일반적인 타입처럼 프로토콜을 채택할 수 없기 때문에, 이를 표현하기 위해 `@Sendable`이라는 특별한 속성이 도입되었습니다. `@Sendable`은 함수의 타입 어노테이션이나 클로저 매개변수 앞에 붙여, 해당 함수가 동시성 환경에서도 안전하게 실행될 수 있음을 나타냅니다.

`@Sendable`로 마킹된 함수나 클로저는 서로 다른 동시 컨텍스트 간에 전달될 수 있기 때문에, 오직 불변(immutable)한 값만 캡처할 수 있으며, 캡처된 모든 값 또한 `Sendable` 프로토콜을 준수해야 합니다. 이러한 제약은 `Sendable`하지 않은 값이 액터 경계(Actor Boundary)를 넘어 이동하거나 실행되는 것을 방지하여, 데이터 경합 등 예측 불가능한 동작을 사전에 차단하는 데 중요한 역할을 합니다.

```swift
@MainActor
final class MAIC {
    var mutableSendableType = SendableType()
    let immutableSendableType = SendableType()
    var mutableNonSendableType = NonSendableType()
    
    func getSendableClosure() -> (@Sendable () async -> Void) {
        return {
            _ = await self.mutableSendableType
            _ = self.immutableSendableType
            _ = await self.mutableNonSendableType // ❌ 오류: Non-sendable type 'NonSendableType' in implicitly asynchronous access to main actor-isolated property 'mutableNonSendableType' cannot cross actor boundary
        }
    }
}

```

`@Sendable` 클로저는 액터 경계를 넘어 실행될 수 있는 가능성이 있기 때문에, Swift는 해당 클로저를 자체적으로 격리된 실행 컨텍스트로 간주합니다. 다시 말해, `@MainActor`에 격리된 클래스 내부에서 정의된 `@Sendable` 클로저라고 하더라도, 그 클로저는 메인 액터(MainActor)에 격리되지 않으며, 메인 액터 외부에서 실행될 수 있는 독립적인 실행 컨텍스트로 취급됩니다. 이로 인해, `@Sendable` 클로저 내부에서 메인 액터에 격리된 프로퍼티에 접근하려면 반드시 비동기(await) 방식으로 접근해야 합니다. 그리고 만약 해당 프로퍼티가 `Sendable`을 따르지 않는 타입이라면, 액터 경계를 넘어 값을 전달할 수 없기 때문에 컴파일 오류가 발생하게 됩니다.


{% hint style="info" %}
**Note**

액터 외부에서 실행될 수 있는 동기적인 `@Sendable` 클로저는 액터 내부 상태를 캡처할 수 없습니다. 이는 액터와의 상호작용이 항상 비동기적으로 이루어져야 한다는 Swift Concurrency의 원칙에 기반한 제한입니다.

```swift
actor MyActor {
    var value = 0
    func getSyncClosure() -> (@Sendable () -> Void) {
        return {
            print(self.value) // ❌ 오류: Actor-isolated property 'value' can not be referenced from a Sendable closure
        }
    }
}
```

액터 내부 상태에 접근하려면 비동기 방식이 필요하기 때문에, 오직 비동기적인 `@Sendable` 클로저만이 액터 내부 상태를 안전하게 캡처할 수 있습니다.
{% endhint %}



지금까지 살펴본 대부분의 Swift Concurrency 예제는 사실상 `@Sendable`에 의존하고 있습니다. 작업(Task)이나 하위 작업(Child Task)을 생성할 때, Swift는 항상 `@Sendable` 클로저를 요구합니다.

작업이 실제로 병렬(Parallel)로 실행되는지 여부와 관계없이, `@Sendable` 클로저가 캡처하는 모든 값은 불변이며, 여러 동시 컨텍스트 간 안전하게 공유될 수 있습니다. 이러한 특성 덕분에, 서로 다른 동시 컨텍스트 간 데이터 경합이 발생할 위험이 없이 안전하게 작업을 수행할 수 있습니다.

```swift
final class Counter {
    var value = 0
    func increment() { value += 1}
}

func incrementParellel() async {
    let counter = Counter()
    await withDiscardingTaskGroup { group in
        group.addTask { // ❌ 오류: Passing closure as a 'sending' parameter risks causing data races between code in the current task and concurrent execution of the closure
            counter.increment()
        }
    }
}
```

{% hint style="info" %}
**Note** Swift 6.0부터는 `@Sendable` 키워드가 `sending`으로 변경되었습니다. ~~`sending`에 관한 자세한 내용은 [지역 기반 격리]()를 참조하세요.~~
{% endhint %}




[^1]: Marker Protocol
[^2]: @frozen
[^3]: @usableFromInline
