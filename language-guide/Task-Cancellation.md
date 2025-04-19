---
description: 비동기 작업을 안전하게 종료하는 방법
---

작업 취소(Cancellation)는 사용자 경험(UX) 향상시키고, 시스템 자원을 효율적으로 관리하기 위해 꼭 필요한 기능 중 하나입니다. 

예를 들어, 사진 앱에서 사용자가 사진 목록을 스크롤하고 있다고 가정해봅시다. 사용자가 스크롤할 때, 호면에 보이는 셀에 해당하는 이미지를 네트워크를 통해 불러오게 됩니다. 그런데 사용자가 빠르게 스크롤하여 해당 셀이 화면에서 사라지면, 아직 로딩 중인 이미지 작업은 더 이상 필요하지 않게 됩니다. 이처럼 더 이상 사용자에게 보여지지 않는 셀의 이미지를 계속해서 불러오는 것은 비효율적이며, 시스템 자원을 불필요하게 낭비하게 됩니다.

Swift Concurrency가 도입되기 전까지는 이러한 비동기 작업을 안전하고 구조적으로 취소하는 것이 쉽지 않았습니다. 예를 들어, `DispatchQueue`를 사용할 경우, `DispatchWorkItem`을 생성한 뒤 `cancel()`을 호출함으로써 작업을 취소할 수는 있었지만, 취소 상태를 작업 내부에 효과적으로 전파하기 어렵고, 중첩된 작업이나 비동기 체인을 관리하기도 까다로웠습니다. 이러한 한계를 보완하기 위해 `OperationQueue`를 사용하는 방법도 있었지만, 구현이 복잡해지고 코드가 장황해진다는 단점이 있었습니다.

```swift
let queue = DispatchQueue.global()

var innerTask: DispatchWorkItem?
let outerTask = DispatchWorkItem {
    let task = DispatchWorkItem {
        for j in 0...10 {
            if let innerTask = innerTask, innerTask.isCancelled {
                print("🟠 내부 작업이 취소됨")
                break
            }
            sleep(1)
            print("✨ 내부 작업 수행 완료: \(j)")
        }
    }
    innerTask = task
    queue.async(execute: task)
    
    for i in 0...10 {
        if outerTask.isCancelled {
            innerTask?.cancel()
            print("🟠 외부 작업이 취소됨")
            break
        }
        sleep(1)
        print("➡️ 외부 작업 수행 완료: \(i)")
    }
}

queue.async(execute: outerTask)

queue.asyncAfter(deadline: .now() + 3) {
    outerTask.cancel()
}
```

위 예제는 `Grand Central Dispatch(GCD)`를. 이용한 비동기 처리에서 외부 작업이 취소될 경우, 해당 취소 신호를 내부 작업에 명시적으로 전파해야 함을 보여줍니다. `DispatchWorkItem`은 내부 작업에 대한 취소 전파를 자동으로 처리해주지 않기 때문에, 외부 작업이 취소될 때 내부 작업도 함께 취소되길 원한다면 이를 직접 코드로 구현해야 합니다. 이러한 방식은 복잡할 뿐 아니라, 작업 취소를 제대로 처리하지 못해 실수가 발생하기 쉬운 위험한 코드가 될 수 있습니다.

Swift Concurrency는 보다 구조화되고 이해하기 쉬운 방식으로 작업 취소를 지원합니다. 구조화된 동시성(Strucutred Concurrency)에서는 모든 작업이 작업 트리(Task Tree) 구조로 구성되며, 상위 작업에서 발생한 취소 신호는 자동으로 모든 하위 작업에 전파됩니다. 하위 작업들은 자신들의 작업 컨텍스트를 스스로 정리한 뒤 상위 작업에게 종료를 알리고, 모든 하위 작업이 완료되면 상위 작업도 종료됩니다.

Swift Concurrency는 협력적 취소(Cooperative Cancellation) 모델을 따릅니다. 이는 각 작업이 실행 도중 스스로 취소 여부를 확인하고, 그에 맞는 처리를 수행해야 함을 의미합니다. 상위 작업이 하위 작업에 취소 신호를 전파하더라도, 하위 작업이 즉시 종료되는 것은 아닙니다. 단지 `Task.isCancelled` 프로퍼티의 값이 `true`로 설정될 뿐이며, 실제 종료 여부는 하위 작업의 구현 방식에 따라 달라집니다. 예를 들어, 어떤 작업은 취소 시 중간 결과를 반환할 수 있고, 또 어떤 작업은 `CancellationError`를 던지며 즉시 종료되기도 합니다. 이처럼 작업마다 취소에 대한 반응이 다를 수 있기 때문에, 작업을 설계할 때는 항상 취소 가능성을 고려하여 설계해야 합니다.

작업을 취소하려면 `Task` 인스턴스의 `cancel()` 함수를 호출합니다. 이 함수를 호출하면 `Task` 내의 호출된 비동기 함수(Asynchronous), `async-let` 바인딩 혹은 작업 그룹에 취소 신호가 전파됩니다.

작업의 취소 여부를 확인하는 방법은 아래와 같습니다.

* `[Task.checkCancellation()]()`: 작업 취소 여부를 확인하고, 취소되었을 경우 `CancellationError`를 던지며 즉시 실행을 중단합니다.

* `[Task.isCancelled]()`: 작업 취소 여부를 `Bool` 값으로 반환합니다. 개발자가 직접 중단 조건을 정의할 수 있으며, 네트워크 연결 해제, 임시 파일 삭제 등 자원 컨텍스트 정리 작업을 수행할 수 있습니다.


```swift
func downloadImage() async throws -> UIImage {
    try await Task.checkCancellation()
    // ...
    return image
}
```



# Task Cancellation in Structured Concurrency

구조화된 동시성에서는 모든 작업이 작업 트리(Task Tree) 구조로 구성됩니다. 이 구조에서는 상위 작업에서 취소 신호가 발생하면, 해당 신호가 모든 하위 작업에 자동으로 전파됩니다. 하위 작업들은 자신의 작업 컨텍스트를 스스로 정리한 뒤, 상위 작업에게 종료를 알립니다. 모든 하위 작업이 종료되면, 상위 작업도 뒤이어 함께 종료됩니다.

구조화된 동시성에서의 취소 전파는 다음 두 가지 방식으로 나눌 수 있습니다.

* **암시적(Implicit) 취소 전파:** 시스템이 자동으로 취소 신호를 전파하는 방식입니다. 예를 들어, 하위 작업 중 하나에서 예외가 발생하면, 그 시점부터 모든 다른 하위 작업에 취소 신호가 자동으로 전파됩니다.

* **명시적(explicit) 취소 전파:** 직접 `Task.cancel()`이나 `cancelAll()` 메서드를 호출해 취소 신호를 전파하는 방식입니다. 



## Implicit Cancellation Propagation

암시적 취소 전파는 병렬로 수행 중인 하위 작업 중 하나가 예외를 던졌을 때 발생합니다. 하위 작업이 예외를 발생시키면, 이를 수신한 상위 작업은 즉시 나머지 실행 중인 모든 하위 작업에게 취소 신호를 전파합니다. 

```
📦 withThrowingTaskGroup { .. }   ← 2️⃣ 하위 작업이 던진 예외 받음
 ├── 🧵 downloadImage(from: url)  ← 3️⃣ (실행 중인) 하위 작업에 취소 전파
 ├── 🧵 downloadImage(from: url)  ← 3️⃣ (실행 중인) 하위 작업에 취소 전파
 └── 🧵 downloadImage(from: url)  ← 1️⃣ 예외 발생 및 상위 작업으로 throw
```

아래 예제는 `TaskGroup`을 사용해 여러 이미지를 병렬로 다운로드하는 과정에서, 일부 하위 작업이 예외를 던지면 나머지 하위 작업들에 암시적으로 취소가 전파된다는 점을 보여줍니다.

```swift
func downloadImages(from urls: [String]) async throws -> [UIImage] {
    try await withThrowingTaskGroup(of: UIImage?.self) { group in
        group.addTask {   // 3️⃣ (실행 중인) 하위 작업에 취소 전파
            guard !Task.isCancelled else { 
                print("🟠 이미지 다운로드 작업 취소 - 1")
                return nil 
            }
            return try await downloadImage(from: urls[0])
        }
        group.addTask {   // 3️⃣ (실행 중인) 하위 작업에 취소 전파
            guard !Task.isCancelled else { 
                print("🟠 이미지 다운로드 작업 취소 - 2")
                return nil 
            }
            return try await downloadImage(from: urls[0])
        }
        group.addTask {   // 3️⃣ (실행 중인) 하위 작업에 취소 전파
            guard !Task.isCancelled else { 
                print("🟠 이미지 다운로드 작업 취소 - 3")
                return nil 
            }
            return try await downloadImage(from: urls[0])
        }
        group.addTask {   // 3️⃣ (실행 중인) 하위 작업에 취소 전파
            try? await Task.sleep(for: .seconds(5))
            guard !Task.isCancelled else { 
                print("🟠 이미지 다운로드 작업 취소 - 4")
                return nil 
            }
            return try await downloadImage(from: urls[0])
        }
        group.addTask {   // 3️⃣ (실행 중인) 하위 작업에 취소 전파
            try? await Task.sleep(for: .seconds(10))
            guard !Task.isCancelled else { 
                print("🟠 이미지 다운로드 작업 취소 - 5")
                return nil 
            }
            return try await downloadImage(from: urls[0])
        }
        group.addTask {   // 1️⃣ 예외 발생 및 상위 작업으로 throw
            try await Task.sleep(for: .seconds(5))
            throw CancellationError() 
        }
        
        var images: [UIImage] = []
        for try await image in group {   // 2️⃣ 하위 작업이 던진 예외 받음
            if let image = image {
                images.append(image)
            }
        }
        return images   // 4️⃣ 예외를 `downloadImages(from:)` 함수 밖으로 던짐
    }
}
Task {
    do {
        try await downloadImages(from: urls)
    } catch {
        print("💥 이미지 다운로드 작업 취소")
    }
}

// Prints "🟠 이미지 다운로드 작업 취소 - 5"
// Prints "🟠 이미지 다운로드 작업 취소 - 4"
// Prints "💥 이미지 다운로드 작업 취소"
```

작업 그룹에 5초 후 취소 예외를 던지는 하위 작업을 추가함으로써, 암시적 취소 전파가 어떻게 동작하는지 살펴보겠습니다. 1~3번째 하위 작업은 즉시 실행되어 이미지를 다운로드하고, 그 결과를 `images` 배열에 추가합니다. 4번째와 5번째 하위 작업은 각각 5초, 10초 뒤에 실행하도록 잠시 대기합니다. 그러나 6번째로 추가된 하위 작업이 5초 뒤 `CancellationError` 예외를 던지고, 이를 감지한 상위 작업은 아직 실행 중이거나 대기 중인 4번째와 5번째 하위 작업에 취소 신호를 전파합니다. 이후 모든 하위 작업이 정리되면, `withThrowingTaskGroup`은 발생한 예외를 `downloadImages(from:)` 함수의 호출자에게 던지고, 해당 작업은 종료됩니다.

이때, 이미 다운로드된 이미지들이 `images` 배열에 담겨 있더라도, 작업 전체가 실패했기 때문에 반환되지 않고 사라집니다. 코드를 약간 수정해, 작업 중 예외가 발생하더라도 지금까지 완료된 결과를 그대로 반환되도록 개선해 보겠습니다. 

```swift
func downloadImages(from urls: [String]) async throws -> [UIImage] {
    try await withThrowingTaskGroup(of: UIImage?.self) { group in
        // ...
        group.addTask { 
            try? await Task.sleep(for: .seconds(5))
            guard !Task.isCancelled else { 
                print("🟠 이미지 다운로드 작업 취소 - 4")
                return nil 
            }
            return try await downloadImage(from: urls[0])
        }
        group.addTask { 
            try? await Task.sleep(for: .seconds(10))
            guard !Task.isCancelled else { 
                print("🟠 이미지 다운로드 작업 취소 - 5")
                return nil 
            }
            return try await downloadImage(from: urls[0])
        }
        group.addTask {
            try await Task.sleep(for: .seconds(5))
            throw CancellationError()
        }
        
        var images: [UIImage] = []
        do {
            for try await image in group {
                if let image = image {
                    images.append(image)
                }
            }
        } catch {
            print("✨ 하위 작업에서 던진 예외를 곧바로 소비함") // 다른 하위 작업으로 취소 전파 안함
            //group.cancelAll()
        }
        return images
    }
}
Task {
    try await downloadImages(from: urls)
    print("🌱 이미지 다운로드 작업 완료")
}

// Prints "✨ 하위 작업에서 던진 예외를 곧바로 소비함"
// Prints "🌱 이미지 다운로드 작업 완료"
```

위 예제에서 주의 깊게 살펴봐야 할 점은 하위 작업이 예외를 던지더라도, 곧바로 나머지 하위 작업들에 취소 신호가 전파되는 것이 아니라는 점입니다. Swift Concurrency에서 암시적 취소 전파가 작동하려면, 예외가 하위 작업에서 던져졌을 뿐만 아니라, `withThrowingTaskGroup`이 해당 예외를 실제로 받아야 합니다. 즉, 단순히 하위 작업이 예외를 던지는 것만으로는 충분하지 않습니다.

위 예제에서는 하위 작업 중 하나가 `CancellationError`를 던지지만, 상위 작업이 `do-catch` 구문으로 해당 예외를 직접 처리(소비)하고 있습니다. 이 경우 `withThrowingTaskGroup`은 예외를 바깥으로 던지지 않기 때문에, 다른 하위 작업들에 취소 신호를 전파하지 않습니다. 그 결과, 이전 예제에서는 4번째와 5번째 하위 작업이 취소되며 "🟠 이미지 다운로드 작업 취소" 메시지가 출력되었지만, 이 예제에서는 그러한 로그가 출력되지 않는 걸 볼 수 있습니다.


{% hint style="Info" %}
**Note**
`URLSession`의 `data(from:)` 비동기 메서드는 작업 실행 중 취소 신호를 받으면, 네트워크 요청을 자동으로 중단하고 `URLError.cancelled` 예외를 발생시킵니다.
{% endhint %}



## Explicit Cancellation Propagation

명시적 취소 전파는 `Task`에서 명시적으로 `cancel()` 함수나, 작업 그룹에서 명시적으로 `cancelAll()` 함수를 호출할 때 발생합니다. 이 함수가 호출되면 지체없이 모든 하위작업들에 취소 신호를 전파합니다.

```
downloadImages()  ← 1️⃣ 취소 신호 발생 및 하위 작업에 취소 전파
 ├── 🧵 downloadImage(from: url)  ← 2️⃣ 취소 신호 수신 및 작업 정리
 ├── 🧵 downloadImage(from: url)  ← 2️⃣ 취소 신호 수신 및 작업 정리
 └── 🧵 downloadImage(from: url)  ← 2️⃣ 취소 신호 수신 및 작업 정리 
```

아래 예제는 `async-let` 바인딩으로 세 개의 이미지 다운로드 작업을 수행하다가, 1초 후 작업 취소가 일어나면 하위 작업들에 취소가 전파된다는 점을 보여줍니다.

```swift
func downloadImages(from urls: [String]) async throws -> [UIImage?] {
    async let firstImage = downloadImage(from: urls[0])
    async let secondImage = downloadImage(from: urls[0])
    async let thirdImage = downloadImage(from: urls[0])
    
    return try await [firstImage, secondImage, thirdImage] // 2️⃣ 취소 신호 수신 및 작업 정리
}
Task {
    let imageTask = Task { 
        do {
            try await downloadImages(from: urls)
        } catch {
            print("💥 이미지 다운로드 작업 취소")
        }
    }
//    try? await Task.sleep(for: .seconds(1))
    imageTask.cancel()  // 1️⃣ 취소 신호 발생 및 하위 작업에 취소 전파
}

// Prints "💥 이미지 다운로드 작업 취소"
```

작업그룹의 `cancelAll()`을 호출하여 비슷한 방식으로 하위 작업들에 취소 신호를 전파할 수 있습니다.

```swift
func downloadImages() async throws -> [UIImage] {
    try await withThrowingTaskGroup(of: UIImage?.self) { group in  // 2️⃣ 취소 신호 수신 및 작업 정리
        let random = Int.random(in: 1..<10)
        for _ in 0..<1_000 {
            group.addTask {  // 2️⃣ 취소 신호 수신 및 작업 정리
                return try await downloadImage(from: url) 
            }
        }
        
        var images: [UIImage] = []
        
        
        for try await image in group {
            if let image = image {
                images.append(image)
            }
            if images.count == random {
                group.cancelAll()  // 1️⃣ 취소 신호 발생 및 하위 작업에 취소 전파
                break
            }
        }
        return images
    }
}
Task { try? await downloadImages() }

```

위 예제는 특정 개수(임의의 정수값)가 다운로드되면, `cancelAll()` 메서드를 호출하여 나머지 하위 작업들을 명시적으로 취소합니다. 이때 작업그룹의 대기 중이거나 실행 중인 하위 작업들에 즉시 취소 신호를 전파하고, 하위 작업들은 곧바로 작업을 종료하게 됩니다. 결과적으로, 취소 시점까지 다운로드가 완료된 특정 개수의 이미지들만 `images` 배열에 담겨 반환됩니다.



# Cancellation Propagation in Unstructured Concurrency

구조화되지 않은 동시성(Unstructured Concurrency)는 취소 전파가 중첩된 내부 `Task`나 `Detached Task`에는 자동으로 취소 전파가 되지 않습니다. 이 구조에서는 각 `Task`에 수동으로 취소를 전파해야 합니다.

```swift
let outerTask = Task {
    try? await Task.sleep(for: .seconds(1))
    
    if Task.isCancelled {
        print("🟡 outerTask 작업 취소")
    }
    
    let innerTask = Task {
        if Task.isCancelled {
            print("🟡 innerTask 작업 취소")
            return
        }
        
        print("➡️ Task2 작업이 완료되었습니다.")
    }
}
outerTask.cancel()

// Prints "🟡 outerTask 작업 취소"
// Prints "➡️ Task2 작업이 완료되었습니다."
```

