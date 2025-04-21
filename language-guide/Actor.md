---
description: 동시성 문제를 원천적으로 차단하는 타입
---

동시성 코드를 작성할 때 근본적으로 어려운 문제 중 하나는 데이터 경합(data race)을 피하는 것입니다. 데이터 경합은 두 개 이상의 스레드(또는 작업)가 동시에 동일한 공유 가변 상태에 접근하고, 그 중 하나가 쓰기 작업을 수행할 때 발생합니다.

```swift
class Counter {
    var value = 0
    func increment() -> Int {
        value += 1
        return value
    }
}

let counter = Counter()
Task.detached { 
    counter.increment()
}
Task.detached {
    counter.increment()
}
```

두 작업에서 값을 증가시키려 한다고 가정해보겠습니다. 이는 데이터 경합이 발생하는 대표적인 예로, 실행 순서에 따라 결과값이 1과 1 혹은 2과 2가 나올 수 있습니다. 예컨대, 두 작업이 모두 초기값 0을 읽고 1을 더하거나, 이미 증가된 1을 읽고 또 1을 더하는 최악의 경우가 발생할 수 있습니다. 데이터 경합은 프로그램의 서로 다른 위치에서 발생할 수 있기 때문에, 이를 정확히 이해하려면 국지적인 판단이 아니라 전체적인 추론이 필요합니다. 게다가 운영체제의 스케줄러는 프로그램을 실행할 때마다 작업들을 다양한 방식으로 인터리빙(interleaving)할 수 있기 때문에, 이러한 문제는 디버깅이 매우 어렵습니다.

데이터 경합은 공유 가변 상태로 인해 발생합니다. 데이터가 변경되지 않거나, 여러 작업 간에 공유되지 않는 경우에는 데이터 경합이 발생할 수 없습니다. 데이터 경합을 방지하는 한 가지 방법은 값의 복사가 일어나는 값 타입(value type)을 사용하는 것입니다. 값 타입은 모든 변경이 해당 인스턴스 내부에서 국소적으로 이루어지기 때문에, 서로 다른 스레드에서 동시에 접근하더라도 안전합니다.

```swift
struct Counter {
    var value = 0
    mutating func increment() -> Int {
        value += 1
        return value
    }
}

let counter = Counter()
Task.detached { 
    var counter = counter
    counter.increment()
}
Task.detached {
    var counter = counter
    counter.increment()
}
```

`Counter` 클래스를 구조체로 바꾸어 동시성 문제를 해결해봅시다. 각 작업 내부에 새로운 변수를 선언하고, Counter 인스턴스를 복사하여 할당합니다. 각 작업은 `Counter` 인스턴스의 복사본을 갖고 있으므로, 서로 간에 영향을 주지 않습니다. 위 예제를 실행하면, 두 작업 모두 1을 출력하게 됩니다. 코드가 데이터 경합으로부터 안전해졌음에도 불구하고, 이는 우리가 의도한 동작은 아닙니다. 이로써 여전히 공유 가변 상태가 필요한 상황이 존재한다는 사실을 알 수 있습니다.

아울러, 락(Lock), 세마포어(DispatchSemaphore), 직렬 디스패치 큐(Serial Dispatch Queue)와 같은 동기화 도구를 사용해 우리가 직접 동기화 메커니즘을 구현할 수도 있습니다. 하지만, 이러한 도구들은 매번 정확하고 신중하게 사용하지 않으면, 동기화가 제대로 이루어지지 않아 다양한 문제가 발생할 수 있습니다. 이것이 바로 액터(Actor)가 등작한 배경입니다.

액터는 서로 다른 작업에서 동일한 데이터에 안전하게 동시에 접근할 수 있도록 설계된 새로운 타입입니다. 액터는 공유 가변 상태에 대한 데이터 경합 문제를 언어 차원에서 해결합니다. 클래스와 달리, 액터는 개발자가 직접 락이나 직렬 디스패치 큐 등의 동기화 메커니즘을 구현하지 않아도 자체적으로 동기화를 보장합니다.

```swift
actor Counter {
    let id = UUID()
    var value = 0
}
```

액터는 클래스와 유사한 점이 많습니다. 클래스와 마찬가지로, 액터는 힙(heap) 메모리 영역에 저장되는 참조 타입(reference type)입니다. 액터는 생성자(initalizer), 메서드, 프로퍼티, 서브스크립트(subscript)를 가질 수 있으며, 확장(extension)하거나 프로토콜을 준수할 수도 있습니다. 또한, 액터는 제네릭으로 정의되거나 제네릭 타입과 함께 사용할 수도 있습니다.

액터가 클래스와 구별되는 가장 큰 차이점은 공유 가변 상태를 보호한다는 점에 있습니다. 액터는 프로퍼티와 메서드를 프로그램의 나머지 부분으로부터 격리함으로써 상태를 안전하게 보호합니다. 이러한 보호는 모든 접근이 반드시 액터를 통해서만 이루어져야 한다는 원칙에 기반합니다.



# Isolation to Protect State from the Rest of the Program

모든 액터는 액터 격리(Actor Isolation)를 통해, 한 번에 단 하나의 작업만이 해당 액터의 데이터에 접근할 수 있도록 보장함으로써 자신만의 고유한 상태를 안전하게 보호합니다. 액터에 선언된 모든 저장 및 계산 프로퍼티, 메서드, 서브스크립트는 기본적으로 해당 액터에 격리됩니다. 

'액터에 격리된다'는 액터의 프로퍼티나 메서드에 대한 모든 접근이 반드시 해당 액터를 통해서만 이루어져야 한다는 의미입니다. 즉, 액터 외부(격리되지 않은 코드)에서 액터 내부에 접근하려면 반드시 비동기(Asynchronous) 방식으로 접근해야 합니다. 액터 내부의 메서드는 별도로 `async` 키워드를 명시하지 않더라도, 기본적으로 비동기 메서드로 간주됩니다.

```swift
extension Counter {
    func incrment() -> Int {
        value += 1
        return value
    }
}
```  

위 예제는 서로 다른 작업에서 `increment()` 메서드를 비동기적으로 호출함으로써, 액터에 안전하게 접근하는 모습을 보여줍니다. 액터의 동기화 메커니즘은 한 작업이 `incrment()`를 호출해 완료되기 전까지, 다른 작업에서 동일한 메서드를 호출하지 못하도록 보장합니다. 즉, 한 작업이 이미 액터를 사용 중일 때 다른 작업이 액터에 접근을 시도하면, 해당 작업은 액터에 대한 접근을 예약(enqueue)하고, 이전 작업이 완료될 때까지 일시 중단(suspend)됩니다. 이전 작업이 액터에 대한 접근을 마친 후에야, 비로소 다음 작업이 접근을 시작할 수 있습니다. 결과적으로 결과값이 1과 2 또는 2와 1이 될 수는 있지만, 동일한 값을 두 번 읽거나 값을 건너뛰는 일은 발생하지 않습니다.

액터는 종종 메일박스(mailbox)에 비유됩니다. 비동기 함수 호출은 액터가 안전하게 실행할 수 있을 때 해당 작업을 수행하도록 요청하는 일종의 메시지로 볼 수 있습니다. 이 메시지들은 액터의 메일박스에 저장되며, 비동기 함수를 호출할 쪽은 액터가 해당 메시지를 처리할 때까지 일시 중단됩니다. 액터는 메일박스에 저장된 메시지를 한 번에 하나씩 처리하기 때문에, 동일학 액터에서 두 개의 요청(작업)이 동시에 실행되는 일이 절대 없습니다. 이는 액터에 격리된 상태에 대한 데이터 경합을 방지하는 핵심 메커니즘입니다.

액터에 격리된 메서드는 동일한 액터에 격리된 다른 메서드를 자유롭게 호출할 수 있습니다. 이러한 내부 접근은 모두 동기적(Synchronous)으로 처리되며, 호출 시 `await` 키워드를 사용할 필요가 없습니다.

```swift
extension Counter {
    func resetSlowly(_ newValue: Int) {
        value = 0
        for _ in 0..<newValue {
            incrment()
        }
        assert(value == newValue)
    }
}
```

액터 외부에서 액터의 프로퍼티에 접근할 경우, 오직 읽기 전용(read-only)으로만 접근할 수 있습니다. 즉, 액터 외부에서 액터의 프로퍼티에 접근해 직접 값을 할당할 수 없으며, 값을 변경하려면 반드시 액터 내부의 메서드를 통해서만 수정해야 합니다.

```swift
func increment(_ counter: Counter, newValue: Int) async {
    await counter.incrment() // ✅
    await counter.value += 1 // ❌ 오류: Mutation of this property is only permitted within the actor 
}
```  

{% hint style="info" %}
**Note** 
이는 원자성(atomicity)을 보장하기 위한 제한 사항 중 하나입니다. 비동기적으로 프로퍼티를 개별적으로 설정할 경우, 의도치 않게 불변 조건(invariants)이 깨질 수 있습니다. 예를 들어, 아래 예제처럼 은행 잔고(balance)와 거래 기록(transactionLog)를 각각 따로 비동기적으로 설정할 수 있다면, 이 두 연산 사이에 다른 작업이 끼어들 수 있어 원자성이 보장되지 않고, 경쟁 조건(race condition)으로 이어질 수 있습니다.

```swift
await account.balance = 2000
await account.transactionLog = ["Deposited 1000"]
```

이 경우, 잔고는 변경되었지만 거래 기록은 누락되는 등 논리적으로 함께 유지되어야 할 상태들 사이의 일관성이 깨질 위험이 생깁니다.
{% endhint %}

액터 내부의 `let` 프로퍼티에는 비동기 접근이 필요하지 않습니다. 하지만, 외부 모듈에 정의된 액터의 `let` 프로퍼티에 접근할 경우에는 비동기적으로 접근해야 합니다.

```swift
// 외부 모듈에서 Counter 액터의 identifier에 비동기적으로 접근하는 예시
```

이는 외부 모듈의 액터에서 해당 프로퍼티가 향후 `let`에서 `var`로 변경되더라도, 이를 사용하는 코드에 최소한의 수정만으로 대응할 수 있도록 하기 위한 설계입니다.


## Non-Isolated Declarations

액터의 내부 상태는 기본적으로 액터에 격리되지만, 프로퍼티나 메서드 앞에 `nonisolated` 키워드를 붙이면 격리되지 않도록 선언할 수 있습니다. 동시 접근으로 인해 데이터 경합이 발생할 가능성이 없는 `let` 프로퍼티나 격리된 상태에 접근하지 않는 메서드에 대해 매번 비동기적으로 접근하는 것은 비효율적입니다. 이러한 경우, 해당 선언에 `nonisolated`를 명시하여 액터 격리에서 제외시킬 수 있습니다.

```swift
actor Person {
    nonisolated let name: String
    var address: String
}

let person = Person()
Task.detached { 
    _ = person.name
}
```

프로퍼티나 메서드가 `nonisolated`로 선언되면, 비록 액터 내부에 정의되어 있더라도 실제로는 액터 외부에 있는 것처럼 간주합니다. 이는 해당 프로퍼티나 메서드가 액터의 격리된 상태를 보호하지 않으며, 여러 작업에서 동기적으로 동시에 접근 가능하다는 것을 의미합니다. 예를 들어, `name` 프로퍼티나 `description()` 메서드는 액터 외부에서 동기적으로 호출해 값을 바로 반환받을 수 있습니다. 단, 비격리 프로퍼티는 반드시 `let`으로 선언되어야 합니다.

```swift
extension Person {
    nonisolated func description() async -> String {
        "\(name) lives in \(await address)"
    }
}
```

또한, 격리되지 않은 메서드는 액터의 내부 상태에 접근할 때는 반드시 비동기적으로 접근해야 합니다. 즉, `nonisolated` 메서드 내에서 액터의 격리된 프로퍼티나 메서드를 사용하려면 `await` 키워드를 사용해야 합니다.


### Closures That Do Not Inherit Actor Isolation

```swift
extension Person {
    func work(at company: String) -> String {
        Task.detached {
            "\(await self.name) works at \(company)"
        }
    }
}
```

`Detached Task`는 액터를 포함한 어떤 실행 컨텍스트도 상속하지 않으며, 완전히 독립적인 작업 컨텍스트를 갖습니다. 따라서, 액터 내부의 메서드에서 `Detached Task`를 생성하면, 이 작업은 액터 격리 경계 바깥에서 실행되며, 액터의 상태에 직접 접근할 수 없습니다. `Detached Task` 내부에서 액터에 격리된 상태에 접근하려면 반드시 비동기적으로 접근해야 합니다.


### Protocol Requirements That Must Be Implemented as Non-Isolated

다른 타입들과 마찬가지로, 액터도 프로토콜의 요구사항을 충족한다면 해당 프로토콜을 준수할 수 있습니다. 예를 들어, `Person` 액터가 `Equatable` 프로토콜을 따르도록 만들어 봅시다. 이때 정적(static) == 메서드는 계정 식별자(id)를 기준으로 두 인스턴스를 비교한 뒤, 그 결과를 불리언 값으로 반환합니다.

```swift
extension Person: Equatable {
    nonisolated static func == (lhs: Person, rhs: Person) -> Bool {
        lhs.name == rhs.name
    }
}
```

이 정적 메서드는 액터 인스턴스(self)와 무관하게 동작하며, 두 개의 `Person` 액터 타입 매개변수를 받아 처리할 뿐 어느 액터에도 속하지 않습니다. 또한, 메서드 내부에서는 오직 `let` 프로퍼티에만 접근하고 있으므로, 액터에 의해 격리되어 보호될 필요가 없습니다. 따라서, 이 정적 메서드를 `nonisolated`로 선언하여 액터 격리에서 제외하는 것이 적절합니다.

```swift
extension Person: Hashable {
    nonisolated func hash(into hasher: inout Hasher) {
        hasher.combine(name)
    }
}
```

이번에는 `Person` 액터가 `Hashable` 프로토콜을 준수하도록 예제를 확장해 보겠습니다. `Hashable`을 준수하기 위해서는 `hash(into:)` 메서드를 구현해야 합니다. 그러나, `Hashable`은 격리되지 않은 프로토콜이기 때문에, 해당 메서드를 액터 내부에 격리된 상태로 구현하면 컴파일 에러가 발생합니다. 또한, `hash(into:)`는 외부 모듈에서 해시 값을 계산할 때 동기적으로 호출되므로, 액터의 격리된 상태에 접근할 수 없습니다. 따라서, 이 메서드를 `nonisolated`로 선언해 액터에 격리되지 않은 프로퍼티에 동기적으로 접근하여 해시 값을 계산하도록 구현해야 합니다.


# Actor And Sendable Types

액터는 자체적인 동기화 메커니즘을 통해 내부의 공유 가변 상태를 보호하므로, 액터 인스턴스는 서로 다른 스레드(동시 컨텍스트) 간에 안전하게 공유될 수 있습니다. 이러한 특성 덕분에, 모든 액터 타입은 암시적으로 `Sendable` 프로토콜을 준수합니다.

```swift
class Person {
    let name: String
    var address: String
}
actor BankAccount {
    var owners: [Person] = []
    func primaryOwner() -> Person? {
        owners.first
    }
}
Task.detached {
    _ = await bankAccount.primaryOwner() 
    // 🔴 오류: Non-sendable result type 'Person?' cannot be sent from actor-isolated context in call to instance method 'primaryOwner()'
}
```

위 예제에서 `primaryOwner()`는 액터 외부에서 비동기적으로 호출될 수 있으며, `Sendable`을 따르지 않는 `Person` 인스턴스를 반환합니다. 이렇게 반환된 `Person` 인스턴스는 액터 외부의 임의의 작업에서 동시에 접근되거나 수정될 수 있기 때문에, 데이터 경합이나 예기치 않는 상태 변경과 같은 동시성 문제가 발생할 수 있습니다. 값을 변경항는 게 아니라 단순히 접근하더라도 문제가 될 수 있습니다. 예를 들어, 액터 내부에서 `address`가 수정되는 동안, 외부에서도 해당 값을 동시에 접근할 수 있기 때문입니다.

이처럼, 액터에 격리된 상태의 동시 접근으로 인한 동시성 문제를 방지하기 위해 액터의 메서드의 매개변수와 반환값은 모두 `Sendable`이어야 합니다. 또한, `let`으로 선언된 프로퍼티에 접근하는 경우에도 해당 프로퍼티가 `Sendable`이어야 합니다. 액터는 오직 `Sebdable`한 값만 내보내보거나 들여오도록 강제함으로써, 공유 가변 상태에 대한 참조가 액터의 격리 경계를 넘나드는 일을 원천적으로 차단합니다.


# Protocol Conformances

모든 액터 타입은 암시적으로 `Actor` 프로토콜을 준수합니다.

```swift
protocol Actor: AnyObject, Sendable { }
```

`Actor` 프로토콜을 사용하면 모든 액터에 공통적으로 적용할 수 있는 프로퍼티나 메서드를 정의하거나, 새로운 기능을 확장할 수 있습니다. 또한, `Actor` 프로토콜의 요구사항(프로퍼티, 메서드, 서브스크립트 등)은 확장을 포함해 모두 해당 액터 인스턴스에 격리되어 실행됩니다.

```swift
protocol DataProcessible: Actor {  // only actor types can conform to this protocol
  var data: Data { get }           // actor-isolated to self
}

extension DataProcessible {
  func compressData() -> Data {    // actor-isolated to self
    // use data synchronously
  }
}

actor MyProcessor : DataProcessible {
  var data: Data                   // okay, actor-isolated to self
  
  func doSomething() {
    let newData = compressData()   // okay, calling actor-isolated method on self
    // use new data
  }
}
```

클래스, 열거형, 구조체와 같은 다른 유형의 타입은 상태를 격리시킬 수 없기 때문에, `Actor` 프로토콜을 준수할 수 없습니다. 액터는 비동기 요구사항이 포함된 프로토콜도 준수할 수 있습니다.

```swift
protocol Server {
  func send<Message: MessageType>(message: Message) async throws -> Message.Reply
}

actor MyActor: Server {
  func send<Message: MessageType>(message: Message) async throws -> Message.Reply { 
  }
}
```

반면, `Hashable`, `Identifiable`과 같이 동기 요구사항이 포함된 프로토콜은 기본적으로 액터에서 직접 준수할 수 없습니다. 이러한 프로토콜을 준수하려면, 메서드나 프로퍼티에 `nonisolated` 키워드를 붙여 비격리로 선언해야 합니다.



# Isolation Rules for Actor Initializers and Deinitializers

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}



## Non-Delegating Initializer

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}



### Initializers with isolated self

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}



### Initializers with non-isolated self   

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}



## Delegating Initializers

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}



## Deinitalizer 

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}



# Actors Synchronize Using a Serial Executor

실행자(executor)는 비동기 작업이 어떤 스레드(또는 큐)에서 실행될지, 어떤 우선순위를 가질지, 그리고 어떤 순서로 처리될지를 조율하는 실행 컨텍스트입니다. 실행자는 작업을 특정 스레드에 직접 할당하진 않지만, 어떤 실행 환경에서 작업이 실행되어야 하는지를 정의합니다. 또한, 주어진 우선순위에 따라 더 중요한 작업을 우선적으로 실행할 수 있도록 하며, 직렬 실행자의 경우에는 작업 간의 실행 순서를 제어하여, 데이터 충돌 없이 안전하게 실행되도록 합니다. Swift는 기본적으로 전역 동시 실행자(global concurrent executor)와 직렬 실행자(serial executor)를 제공합니다.

모든 액터 인스턴스는 자신만의 직렬 실행자를 기본 실행자(default executor)로 갖습니다. 직렬 실행자는 액터에 격리된 상태를 스레드에 안전하게 보호하는 핵심 메커니즘입니다. 직렬 실행자는 한 번에 하나의 부분 작업(parital task)만을 실행하며, 두 작업이 동시에 액터에 격리된 상태에 접근하지 못하도록 보장합니다. 직렬 실행자는 작업을 순서대로 실행한다는 직관을 주지만, 실제로는 큐에 등록된 순서와 실행 순서가 반드시 일치하지는 않습니다. Swift 런타임은 우선순위 역전(Priority Inversion)을 방지하기 위해, 우선순위 승격(Priority Escalation)과 같은 기법을 사용하여 작업 간 우선순위를 고려해 실행 순서를 유연하게 조정합니다. 이러한 스케줄링 전략은 엄격한 선입선출(FIFO) 순서로 작업을 실행하는 직렬 디스패치 큐와 구분되는 특징이며, 비동기 함수의 효율적인 실행을 가능하게 합니다. 

모든 액터는 기본적으로 협력형 스레드 풀(Cooperative Thread Pool)에서 작업을 실행합니다. 이 스레드 풀은 특정 작업을 어떤 스레드에서 실행할지 동적으로 결정하며, 작업을 항상 동일한 스레드에서 실행한다고 보장하지 않습니다. 다시 말해, 직렬 실행자는 각 작업을 (단일 스레드가 아닌) 서로 다른 스레드에서 실행하게 하며, 실행 시점엔 항상 한 번에 하나의 작업만 실행되록 직렬화합니다. 이는 동일한 스레드에서 작업이 순차적으로 처리되는 것처럼 보이게 하며, 개념적으로 ‘단일 스레드에서 직렬로 실행한다’고 이해해도 무방합니다. 중요한 건 액터의 격리된 상태가 동시에 여러 스레드에 노출되지 않는다는 점입니다.


## Implementing a Custom Actor Executor

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}



# Actor Re-entrancy and Deadlock Avoidance

액터에 격리된 메서드는 재진입(re-entrancy)이 가능합니다. 이는 해당 메서드가 실행 도중 `await` 키워드를 만나 일시 중단되면, 그 중단된 시점에 다른 작업이 해당 액터에 진입하여 실행될 수 있음을 의미합니다.

이러한 재진입은 처음 액터가 `await` 지점에서 일시 중단된 사이에 다른 작업이 동일한 액터의 상태를 변경할 수 있음을 의미합니다. 그 결과, 처음 진입한 작업이 재개될 때는 `await` 앞뒤의 상태가 달라져서 의도치 않은 동작이나 불변 조건 깨짐으로 이어질 수 있습니다. 


```swift
(ImageDownload 액터 선언)
```


{% hint style="info" %}
**Not** 
이는 `await` 키워드 자체가 직접적인 실행 효과를 가지지 않더라도, 모든 잠재적인 일시 중단 지점 앞에 `await`을 명시해야 하는 이유입니다. `await` 키워드는 해당 지점을 기준으로 공유 가변 상태가 변경될 수 있음을 개발자에게 알려주는 일종의 경고 표시 역할을 하기 때문입니다. 따라서 `await` 이전에 읽은 상태가 `await` 이후에도 동일하다고 가정해서는 안됩니다.
{% endhint %}

`EmojiDownloader`는 주어진 URL에 대한 이모지(Emoji)가 캐시에 존재하는 경우, 해당 이모지를 반환합니다. 캐시에 이모지가 없다면 네트워크에서 이모지를 다운로드한 후, 이를 캐시에 저장하고 반환합니다. `cache`는 액터에 격리되어 있어 낮은 수준의 데이터 경합으로부터 보호받습니다. 하나의 작만이 액터에 진입하여 `cache`에 접근할 수 있으므로, 이미지를 다운로드하고 저장하거나 읽는 과정에서 캐시가 손상될 가능성은 없어 보입니다.

하지만, 정말 안전하다고 할 수 있을까요? 예를 들어, 두 개의 독립된 작업이 동시에 같은 이모지를 요청하는 상황을 가정해 봅시다.

```
actor ImageDownloader {
    private var cache: [URL: Image] = [:]

    func image(from url: URL) async throws -> Image? {
        if let cached = cache[url] {
            return cached
        }

        let image = try await downloadImage(from: url)

        cache[url] = image
        return image
    }
}
```

{% stepper %}
{% step %}
### Step 1

먼저, 1️⃣번째 작업이 캐시에 🔥 이모지가 없음을 확인하고, 서버에서 해당 이모지를 다운로드하기 시작합니다. 이 다운로드 작업은 시간이 소요되므로, 작업은 일시 중단됩니다.

{% endstep %}
{% step %}
### Step 2

그 사이, 동일한 URL에 ❄️ 이모지가 새롭게 업로드되었다고 가정해 봅시다. 이제 2️⃣번째 작업이 이 URL로부터 이모지를 가져오려 시도합니다.

{% endstep %}
{% step %}
### Step 3

2️⃣번째 작업 역시 캐시에 이모지가 없음을 확인합니다. 이는 1️⃣번째 작업의 다운로드가 아직 완료되지 않았기 때문입니다. 따라서 2️⃣번째 작업도 서버로부터 ❄️ 이모지를 다운로드하기 시작하고, 완료될 때까지 일시 중단됩니다.

{% endstep %}
{% step %}
### Step 4

시간이 지나 2️⃣번째 작업이 완료되어 실행을 재개하고, 받은 ❄️ 이모지를 캐시에 저장한 뒤 반환합니다.

{% endstep %}
{% step %}
### Step 5

이어서, 1️⃣번째 작업도 완료되어 실행을 재개합니다. 그러나, 이 작업은 🔥 이모지를 캐시에 덮어씁니다. 결과적으로, 최신 이모지인 ❄️가 🔥로 되돌아가는 상황이 발생합니다.

{% endstep %}
{% endstepper %}

1️⃣번째 작업은 2️⃣번째 작업이 동일한 URL에 대해 먼저 이모지를 캐시에 저장했음에도 불구하고, 다른 이모지로 이를 덮어씁니다. 우리는 일반적으로 한 번 캐시에 저장된 후에는 동일한 URL에 대해 항상 같은 이미지가 반환되기를 기대합니다. 그러나, 위 예제에서는 캐시에 저장된 이미지가 예상치 않게 변경되는 결과가 발생했습니다.

위 예제는 액터의 재진입성으로 인해 발생할 수 있는 미묘한 버그를 보여주는 대표적인 사례입니다. 실행이 인터리빙되더라도, 액터는 여전히 ‘단일 스레드에서 실행되는 것처럼 보이는 환상(single-threaded illusion)’을 유지합니다. 즉, 어떤 액터에서도 동시에 두 개의 작업이 실행되는 일은 결코 없습니다. 액터는 공유 가변 상태에 대한 저수준의 데이터 경합에는 안전하지만, 항상 고수준의 경쟁 조건까지 막아주는 것은 아닙니다.

```swift
actor ImageDownloader {
    private var cache: [URL: Image] = [:]

    func image(from url: URL) async throws -> Image? {
        if let cached = cache[url] {
            return cached
        }

        let image = try await downloadImage(from: url)

        cache[url] = cache[url, default: image]
        return image
    }
}
```

하지만, 더 나은 해결책은 중복 다운로드 자체를 방지하는 것입니다. 아래 예제는 이보다 개선된 방식의 구현을 보여줍니다.

<details>

<summary>Check your assumptions after an await: A better solution</summary>

```swift
actor ImageDownloader {

   private enum CacheEntry {
       case inProgress(Task<Image, Error>)
       case ready(Image)
   }

   private var cache: [URL: CacheEntry] = [:]

   func image(from url: URL) async throws -> Image? {
       if let cached = cache[url] {
           switch cached {
           case .ready(let image):
               return image
           case .inProgress(let task):
               return try await task.value
           }
       }

       let task = Task {
           try await downloadImage(from: url)
       }

       cache[url] = .inProgress(task)

       do {
           let image = try await task.value
           cache[url] = .ready(image)
           return image
       } catch {
           cache[url] = nil
           throw error
       }
   }
}
```

</details>

액터의 재진입성은 두 액터가 서로의 응답을 기다리며 발생할 수 있는 교착 상태(deadlock)를 방지해줄 뿐만 아니라, 작업이 일시 중단된 동안 다른 작업이 액터에 진입할 수 있도록 하여 불필요한 차단(blocking)을 줄여줍니다. 또한, 우선순위가 더 높은 작업이 먼저 실행될 수 있는 기회를 제공함으로써 전체적인 처리 성능 향상에도 기여합니다.

{% hint style="info" %}
**Note**
액터의 재진입을 허용하지 않으면 어떻게 될까요?

재진입이 가능한 액터와 대비되는 개념은 재진입이 불가능한(non-reentrant) 액터입니다. 이는 한 작업이 액터에 격리된 메서드를 호출한 뒤 `await`을 만나 일시 중단되더라도, 다른 작업은 해당 메서드의 실행이 완전히 완료될 때까지 액터에 접근할 수 없다는 의미입니다. 결과적으로, 해당 메서드가 완료되기 전까지 액터 전체가 다른 어떤 작업도 처리할 수 없으며, 이후의 모든 호출은 대기 상태로 유지됩니다.

예를 들어, 아래 예제에서 `DecisionMaker`가 재진입이 불가능한 액터라고 가정해 보겠습니다. 이 경우, `DecisionMaker`는 `friend.tell(heldBy:)` 호출이 완료될 때까지 다른 어떤 작업도 액터에 진입할 수 없습니다. 즉, 해당 작업이 끝날 때까지 액터는 일시적으로 잠긴 상태(lock)에 놓이게 되며, 다른 모든 호출은 대기 상태로 유지됩니다.

```swift
// assume non-reentrant
actor DecisionMaker {
  let friend: DecisionMaker
  var opinion: Decision = .noIdea

  func thinkOfGoodIdea() async -> Decision {
    opinion = .goodIdea                                   
    await friend.tell(opinion, heldBy: self)
    return opinion // ✅ always .goodIdea
  }

  func thinkOfBadIdea() async -> Decision {
    opinion = .badIdea
    await friend.tell(opinion, heldBy: self)
    return opinion // ✅ always .badIdea
  }
}

extension DecisionMaker {
  func tell(_ opinion: Decision, heldBy friend: DecisionMaker) async {
    if opinion == .badIdea {
      await friend.convinceOtherwise(opinion)
    }
  }
}
```

Alice와 Bob이라는 두 친구가 의견을 주고받으며 티키타카를 하는 상황을 액터로 모델링해보겠습니다. 예를 들어, Alice가 Bob에게 `thinkOfGoodIdea()`를 호출하며 좋은 아이디어를 전달하면, Bob의 `tell(heldBy:)` 메서드는 아무런 추가 동작을 하지 않기 때문에, 이 경우에는 교착 상태가 발생하지 않습니다.

하지만, 두 친구가 `thinkOfBadIdea()`를 호출해 나쁜 의견을 주고 받는 상황에서는 문제가 발생합니다. Alice는 Bob에게 `tell(heldBy:)`를 호출하고, 그 응답을 기다리는 동안 일시 중단된 상태에 들어갑니다. 그런데 Bob은 이 의견이 마음에 들지 않아, `convinceOtherwise()`를 호출하며 다시 Alice에게 의견을 바꾸라고 설득하려고 합니다. 그라나, 이 시점에서 Alice는 아직 `tell(heldBy:)` 호출의 결과를 기다리고 있는 상태이므로, Bob이 Alice에게 재진입을 할 수 없습니다. 결과적으로, Alice는 Bob의 응답을, Bob은 Alice의 응답을 기다리며 서로가 서로의 작업을 막는 교착 상태에 빠지게 됩니다.

그렇다면, 만약 이 액터들이 재진입이 가능했다면 어떻게 됐을까요? Bob이 Alice에게 `convinceOtherwise()`를 호출하더라도, Alice는 재진입을 허용하기 때문에 해당 요청을 처리하고 응답을 반환할 수 있습니다. 그 결과, Bob의 `tell(heldBy:)`도 완료되며, Alice는 다시 Bob으로부터 응답을 받아 자신의 작업을 마무리하게 됩니다. 

즉, 재진입이 가능한 액터는 이런 순환 호출 상황에서도 교착 상태없이 자연스럽게 흐름을 이어갈 수 있게 해줍니다.
{% endhint %}


## Actor Reprioritization

액터의 재진입성은 우선순위 역전을 방지하는 데에도 도움이 줍니다. 기존의 직렬 디스패치 큐는 모든 작업을 엄격한 선입선출(FIFO) 순서로 실행하는 반면, 액터의 기본 실행자인 직렬 실행자는 이보다 더 경량화되고 유연한 구조를 갖습니다. 이 덕분에, 액터는 다음에 실행할 작업을 선택할 때, 우선순위가 더 높은 작업을 우선적으로 실행함으로써, 낮은 우선순위 작업에 의한 우선순위 역전 현상을 효과적으로 방지할 수 있습니다.

앞서 살펴본 이모지 다운로드 예제에서는 작업 2️⃣가 작업 1️⃣보다 나중에 액터에 진입했음에도 불구하고 먼저 완료되는 모습을 확인할 수 있었습니다. 이처럼 액터가 재진입성을 지원하면, 작업들이 반드시 선입선출(FIFO) 순서로 실행되고 종료될 필요는 없습니다. 즉, 우선순위가 낮은 작업 1️⃣이 먼저 실행되었다 하더라도, Swift 런타임은 큐에 대기 중인 작업들을 분석하여, 작업 1️⃣이 `await` 지점에서 일시 중단되면, 우선순위가 더 높은 작업 2️⃣에게 실행 기회를 우선적으로 부여할 수 있습니다. 이러한 방식은 작업 간의 실행 순서와 자원 배분 비중을 동적으로 조정함으로써 전체적인 처리 효율을 높이는 데 기여합니다.

{% hint style="info" %}
**Note**
만약 액터에 격리된 메서드가 `await` 지점 없이 실행된다면, 우선순위는 어떻게 조정될 수 있을까요? 만약 액터에서 우선순위가 낮은 작업 🅐가 실행 중이고, 우선순위가 높은 작업 🅑가 접근을 시도한다면, Swift 런타임은 작업 🅐의 우선순위를 일시적으로 승격시켜 처리 속도를 높입니다. 이를 통해 작업 🅐가 보다 빠르게 종료되어, 작업 🅑가 지연 없이 실행될 수 있도록 돕습니다. ~~자세한 내용은 [Task]()를 참조하세요.~~
{% endhint %}


# Actor Hopping

액터에 격리된 메서드에서 다른 액터에 격리된 메서드를 호출하면서 실행 컨텍스트가 전환되는 현상을 액터 홉핑(Actor Hopping)이라고 합니다. 이 과정은 컨텍스트 스위칭(Context Switching)을 수반하며, 기존 작업이 일시 중단된 뒤 새로운 액터의 실행 컨텍스트에서 재개되는 방식으로 이루어집니다. 액터 홉핑이 자주 발생하면 실행 컨텍스트의 빈번한 전환으로 인해, 작업의 일시 중단과 재개, 작업 큐 간 전달, 스레드 간 전환 등 다양한 형태의 오버헤드가 발생할 수 있습니다. 따라서, 성능을 고려할 때는 불필요한 액터 홉핑을 줄이는 방향으로 액터를 설계해야 합니다.

{% hint style="info" %}
**Note**
Swift Concurrency에서 말하는 컨텍스트 스위칭은 일반적인 의미의 스레드 컨텍스트 스위칭(Thread Context Switching) — 즉, 스레드 문맥을 저장하고 복원하는 작업 — 을 의미하지 않습니다. Swift Concurrency는 협력형 스레드 풀 모델을 채택하고 있으며, 실행에 사용되는 스레드의 수는 물리적인 CPU 코어 수를 초과하지 않습니다. 따라서, 한 CPU 코어에서 여러 스레드가 짧은 시간 간격으로 번갈아 실행되며 발생하는 스레드 컨텍스트 스위칭은 일어나지 않습니다.
{% endhint %}

```
┌──────────────────────────────┐
│       프로그램의 나머지 부분       │
│  ┌────────────────────────┐  │
│  │  Task A                │  │
│  │  await counter.inc()   │ ─│───┐  
│  └────────────────────────┘  │   │      
│                              │   │ 👷🏼‍♀️ 작업 요청 전달!
│  ┌────────────────────────┐  │   │
│  │  Task B                │  │   │
│  │  await counter.get()   │ ─│─┐ │
│  └────────────────────────┘  │ │ │ 👷🏻 작업 요청 전달!
└──────────────────────────────┘ │ │
                                 ▼ ▼
┌─────────────────────────────────────┐
│            actor Counter            │
│─────────────────────────────────────│
│  (격리된 내부 상태)                     │
│  var value: Int                     │
│                                     │
│  func increment() -> Int            │
│                                     │
│  🔒 내부 상태는 액터만 직접 접근 가능       │
└─────────────────────────────────────┘
            ▼
┌──────────────────────┐
│    Serial Executor   │  ◀─ 작업들을 한 번에 하나씩 처리
└──────────────────────┘
```

UI 렌더링 및 이벤트 처리 코드는 모두 메인 액터(MainActor)에 격리되어 메인 스레드에서 실행됩니다. 반면, 그 외의 일반 액터는 메인 스레드와 분리된 협력형 스레드 풀에서 실행됩니다.

동일한 협력형 스레드 풀 내에서 발생하는 일반 액터 간의 홉핑은 일반적으로 성능에 큰 영향을 주지 않습니다. 작업은 스케줄러에 의해 기존 스레드에서 그대로 이어질 수도 있고, 상황에 따라 다른 스레드에서 재개될 수도 있습니다. 따라서, 이 스레드 풀 내부에서의 액터 홉핑은 매우 빠르고 효율적으로 처리됩니다.

하지만 메인 액터와 일반 액터 간의 홉핑은 상황이 다릅니다. 메인 액터와 일반 액터의 실행 환경은 물리적으로 분리되어 있기 때문에, 이들 간의 홉핑은 필연적으로 실제 스레드 전환을 수반하게 되며, 이는 상대적으로 더 높은 비용을 초래하게 됩니다.

```swift
// on database actor
func loadArticle(with id: ID) async throws -> Article { ... }

@MainActor
func updateUI(for article: Article) { ... }

@MainActor
func updateArticles(for ids: [ID]) async throws {
    for id in ids {
        let article = try await loadArticle(with: id)  // 💥 컨텍스트 스위칭
        await updateUI(for: article)
    }
}
```

```
──────────────────────────────────────────────────────────────────
| loadArticle | 💥 | updateUI | 💥 | loadArticle | 💥 | updateUI | ・・・
──────────────────────────────────────────────────────────────────
💥: 컨텍스트 스위칭
```

메인 액터에 격리된 `updateArticles(for:)` 메서드는 데이터베이스 액터로부터 기사를 불러오고, 각 기사를 UI에 반영합니다. 이때 `for` 반복문의 각 루프마다 최소 두 번의 컨텍스트 스위칭이 발생합니다. 하나는 메인 액터에서 데이터베이스 액터로의 전환, 다른 하나는 다시 메인 액터로 복귀하는 전환입니다. 즉, 각 반복마다 두 개의 스레드가 짧은 시간 간격으로 교대로 실행되는 패턴이 반복됩니다. 루프 반복 횟수가 적고, 각 반복에서 수행하는 작업량이 충분히 클 경우에는 큰 문제가 되지 않을 수 있습니다. 그러나 메인 액터에서 벗어났다 다시 돌아오는 홉핑이 반복적으로 발생할 경우, 스레드 전환에 따른 누적 비용으로 인해 실행 성능이 저하될 수 있습니다.

```swift
 // on database actor
func loadArticles(with ids: [ID]) async throws -> Article { ... }

@MainActor
func updateUI(for articles: [Article]) { ... }

@MainActor
func updateArticles(for ids: [ID]) async throws {
    let articles = try await loadArticle(with: ids) 
    await updateUI(for: articles)
}
```

스레드 전환 비용을 줄이기 위해서는 메인 액터에서 처리할 작업을 가능한 한 묶어서 실행하는 것이 좋습니다. 예를 들어, 각 루프의 작업을 `loadArticles(with:)` 및 `updateUI(for:)`와 같은 메서드로 분리하여, 개별 항목이 아닌 배열 단위로 한 번에 처리하도록 리팩토링할 수 있습니다. 작업을 묶어 일괄 처리하면 컨텍스트 스위칭 횟수를 줄일 수 있으며, 메인 스레드와 다른 액터 간의 전환으로 인한 성능 오버헤드도 효과적으로 완화할 수 있습니다. 이처럼, 메인 액터와 다른 액터 간의 전환이 반복적으로 발생하지 않도록 주의하여 액터를 설계하는 것이 중요합니다.

{% hint style="info" %}
**Note**
일반 액터 간의 홉핑 동일한 협력형 스레드 풀 내에서 이루어진다고 하더라도, 가능하면 피하는 것이 좋습니다. 일반 액터 간의 홉핑은 메인 액터와 일반 액터 간의 홉핑보다 실제 컨텍스트 스위칭이 발생할 가능성은 낮지만, 완전히 배제할 수는 없습니다. 따라서 '어떤 홉핑은 비용이 적고, 어떤 홉핑은 크다'는 식의 이분법적인 구분은 실질적인 의미가 크지 않을 수 있으며, 모든 형태의 액터 간 홉핑은 불필요하게 반복되지 않도록 최소화하는 것이 바람직합니다.
{% endhint %}


# Actor Contentions

한 번에 하나의 작업만 순차적으로 실행한다는 액터의 이러한 특성은 병렬 처리의 이점을 활용하기 어렵다는 단점으로 이어질 수 있습니다. 따라서, 병렬성(Parellelism)이 중요한 작업이라면 가능한 부분은 액터 외부에서 처리하고, 정말 필요한 경우에만 액터에 접근하도록 하여 액터에 머무는 시간을 최소화해야 합니다. 액터에 대한 접근을 가능한 한 작은 단위로 분리하면, 전체적인 처리 성능을 높일 수 있습니다.

```swift
actor CompressionUtils {

    var logs: [String] = []

    nonisolated func compress(with file: FileStatus) async -> Data {
        await log(update: "🔴 압축 시작: \(file.name)")
        let compressedData = compressFile(
            for: file
        ) { size in
            Task { @MainActor in
                state.update(name: file.name, uncompressedSize: size)
            }
        } progressNotification: { progress in
            Task { @MainActor in
                state.update(name: file.name, progress: progress)
                await log(update: "🔵 압축 진행 중: \(progress)")
            }
        } finalNotification: { size in
            Task { @MainActor in
                state.update(name: file.name, compressedSize: size)
            }
        }
        await log(update: "🔵 압축 완료: \(file.name)")

        return compressedData
    }
}
```
