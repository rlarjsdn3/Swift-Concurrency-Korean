---
description: 일시 중단(suspend)하고, 다시 재개(resume)하기
---

비동기(Asynchronous) 코드는 실행 이후 결과가 반환되기까지 걸리는 시간을 예측할 수 없으며, 다음 줄의 코드 실행을 막지 않는 특성을 가집니다. 네트워크 요청처럼 외부 환경에 따라 응답 속도가 달라지거나, 파일 I/O처럼 무거운 작업은 대표적인 비동기 작업 예시입니다. 이러한 작업들을 메인 스레드에서 처리하면 UI 렌더링이나 사용자 이벤트 처리에 지장을 줄 수 있기 때문에, 비동기적으로 실행하여 메인 스레드의 부담을 덜어주는 것이 중요합니다.

비동기 작업에서 가장 중요한 것 중 하나는 작업이 언제 완료되는지를 아는 것입니다. 작업이 완료되면 그 결과를 바탕으로 UI를 갱신하거나, 이후의 후속 작업을 이어서 처리해야 합니다. Swift Concurrency가 도입되기 전에는 이러한 흐름을 콜백(Callback) 기반으로 처리해왔습니다. 하지만 콜백 기반의 비동기 함수는 코드가 장황해지기 쉬우며, 흐름이 복잡하게 얽히는 경우 구현이 어렵고 가독성도 떨어집니다. 특히 콜백 호출을 누락하거나, 메인 스레드로의 전환을 놓치는 등의 실수가 발생하기 쉬운데, 이러한 문제를 Swift 컴파일러가 사전에 잡아주지 못한다는 점이 큰 단점이었습니다.


```swift
func downloadImage(from url: String,
                   completion: @escaping (Result<UIImage, Error>) -> Void) {
    
    let task = URLSession.shared.dataTask(with: URL(string: url)!) { data, response, error in
        if let error = error {
            completion(.failure(error)) // 🟠 만약 콜백을 적는 걸 깜박한다면?
            return
        } else {
            guard let response = response as? HTTPURLResponse,
                  response.statusCode == 200 else {
                completion(.failure(URLError(.badServerResponse)))  // 🟠 만약 콜백을 적는 걸 깜박한다면?
                return
            }
            
            if let data = data,
               let image = UIImage(data: data) {
                
                prepareThumbnail(from: image) { image in
                    if let image = image {
                        DispatchQueue.main.async {
                            completion(.success(image)) // 🟠 만약 콜백을 적는 걸 깜박한다면?
                            return
                        }
                    }
                }
            } else {
                completion(.failure(URLError(.cannotDecodeContentData)))  // 🟠 만약 콜백을 적는 걸 깜박한다면?
                return
            }
        }
    }
    
    task.resume()
}

downloadImage(from: url) { result in
    if let image = try? result.get() {
        DispatchQueue.main.async { //  🟠 만약 메인 디스패치 큐를 적는 걸 깜박한다면?
            // imageView.image = image
            print("➡️ 이미지 다운로드 완료: \(image)")
        }
    }
}
```

위 예제에서는 콜백 호출 누락이나 스레드 전환 처리의 실수로 인해 잠재적인 버그가 발생할 수 있는 지점들이 곳곳에 숨어 있습니다. 이러한 요소 중 하나라도 놓치게 되면, 애플리케이션은 발생한 예외 상황에 대해 사용자에게 적절한 피드백을 전달하지 못할 수 있습니다. Swift Concurrency는 이러한 문제를 해결하고, 비동기 코드를 더 안전하고 직관적으로 구현할 수 있도록 도와줍니다. `async`와 `await`으로 구현한 비동기 함수는 코드가 직선적으로 구성되어 가독성이 훨씬 좋고, 흐름을 따라가기도 수월합니다. 또한 예외가 발생할 수 있는 위치에서 예외를 명시적으로 처리하지 않거나, 함수가 적절한 결과를 반환하지 않을 경우, Swift 컴파일러가 이를 감지하여 개발자에게 알려줍니다.


# Defining and Calling Asynchronous Functions

비동기 함수(Asynchronous Function)는 함수 실행 도중 네트워크 요청이나 파일I/O처럼 완료까지 시간이 오래 걸리는 작업을 만나면, 일시적으로 중단(suspend)될 수 있는 특별한 유형의 함수입니다. 비동기 함수도 예외를 던지거나 값을 반환하거나, 혹은 아무 값도 반환하지 않으며 실행을 완료하는 등 동기 함수의 일반적인 특성을 그대로 가집니다. 단지 실행 중 일시 중단될 수 있다는 점만 다릅니다.

함수가 비동기적으로 동작함을 나타내기 위해서는 함수 선언부에서 매개변수 목록 뒤, 반환 화살표(->) 앞에 `async` 키워드를 명시해야 합니다. 이는 예외를 던질 수 있는 함수에 `throws` 키워드를 붙이는 것과 같은 맥락입니다. 비동기 함수가 값을 반환하는 경우에는 `async` 키워드 뒤에 일반적인 동기 함수처럼 반환 타입을 명시하면 됩니다.

```swift
func downloadImage(from url: String) async -> UIImage {
    let image = // ... 이미지 다운로드 코드
    return image
}
```

비동기 함수는 함수 실행 도중 네트워크 타임아웃(TimeOut) 등 다양한 예외 상황에 효과적으로 대응하기 위해 예외를 던질 수 있습니다. 비동기 함수가 예외를 던질 수 있음을 나타내기 위해서는 `async` 키워드 뒤에 `throws` 키워드를 함께 명시해야 합니다.

비동기 함수를 호출하면, 해당 함수가 값을 반환할 때까지 실행이 일시적으로 중단됩니다. 이때 함수 호출 코드 앞에 `await` 키워드를 명시해, 해당 함수가 일시 중단(suspend)될 수 있음을 나타내야 합니다. 이는 예외를 던질 수 있는 함수를 호출할 때 `try` 키워드를 사용하는 것과 같은 맥락입니다. 잠재적으로 일시 중단될 수 있는 모든 비동기 코드를 호출할 때는 반드시 `await` 키워드를 사용해야 하며, 이러한 일시 중단은 암시적으로나 자동으로 발생하지 않습니다. `await` 키워드를 명시함으로써, 코드의 흐름을 더 명확하게 하고 동시성 코드의 가독성과 이해도를 높일 수 있습니다.

```swift
Task { let image = await downloadImage(from: url) }
```

그렇다면 일시적으로 중단된 실행 흐름은 언제 다시 재개(resume)될까요? 비동기 함수가 결과값을 반환하는 시점에 코드 실행이 다시 재개됩니다. 시스템은 비동기 함수가 값을 반환하는 타이밍을 감지하고, 해당 시점이 실행을 이어가기 적절하다고 판단되면 중단된 지점부터 코드를 다시 실행합니다.

완성된 `downloadImage(from:)` 함수 코드를 통해, 실행 흐름이 어떻게 일시 중단되고 다시 재개되는지 살펴보며, 그 속에 숨겨진 동작 원리를 하나씩 파헤쳐보겠습니다.

```swift
func downloadImage(from url: String) async throws -> UIImage? {
    let (data, _) = try await URLSession.shared.data(from: URL(string: url)!) // 1️⃣
    let image = UIImage(data: data) // 2️⃣
    let thumbnail = await image?.byPreparingThumbnail(ofSize: CGSize(width: 50, height: 50)) // 3️⃣
    return thumbnail // 4️⃣
}
```

{% stepper %}
{% step %}
### Step 1
`downloadImage(from:)` 함수가 호출되면, 곧바로 `URLSession` 인스턴스의 `data(from:)` 비동기 함수를 통해 서버로부터 이미지 데이터를 불러옵니다. 이 과정에서 결과가 반환될 때까지 실행 흐름은 일시적으로 중단되며, 해당 함수를 실행 중이던 스레드의 제어권을 시스템에게 양보합니다. 스레드의 제어권을 넘겨받은 시스템은 `downloadImage(from:)` 작업이 일시 중단되어 있는 동안, 그 스레드를 활용해 다른 유용한 작업(예: 파일 I/O 등)을 수행할 수 있습니다. 한편, 비동기 함수 실행 중 네트워크 타임아웃과 같은 예외가 발생하면, 코드 실행은 즉시 중단되며 호출자(caller)가 처리할 수 있도록 예외를 던지게 됩니다.
{% endstep %}
{% step %}
### Step 2
`data(from:)` 비동기 함수가 이미지 데이터를 반환하면, 다음 줄의 코드가 이어서 실행됩니다. `UIImage(data:)`는 동기 코드이므로 실행 흐름을 일시 중단하지 않습니다. 데이터 변환에 성공하면, 생성된 이미지가 변수에 할당되고 이후 줄의 코드가 순차적으로 실행됩니다.
{% endstep %}
{% step %}
### Step 3
비동기 함수인 `byPreparingThumbnail(ofSize:)`을 호출하면 1️⃣단계에서와 마찬가지로, 결과가 반환될 때까지 실행 흐름은 일시적으로 중단되며, 다른 비동기 작업이 실행될 기회를 얻게 됩니다.
{% endstep %}
{% step %}
### Step 4
호출자에게 썸네일 이미지를 반환합니다. 
{% endstep %}
{% endstepper %}

{% hint style="warning" %}
**Note**
비동기 함수가 `await` 키워드를 만나 실행을 일시 중단하고 스레드의 제어권을 포기한 후, 일정 시간이 흐른 뒤 다시 재개될 때 Swift는 어느 스레드가 해당 함수를 재개할지 보장하지 않습니다. `await` 이전에 실행되던 스레드와 이후에 실행되는 스레드는 서로 다를 수 있으므로, 스레드-로컬(Thread-Local)과 같이 스레드에 의존적인 데이터는 사용하지 않아야 합니다.
{% endhint %}



위 실행 흐름을 다이어그램으로 그려보면 아래와 같습니다.

```
┌──────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ downloadImage(from:) │    💥 suspend    │   readFile(from:)   │   💨 resume   │ downloadImage(from:) │
└──────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

위 다이어그램은 `downloadImage(from:)` 함수가 실행되는 도중 `await` 키워드를 만나 일시적으로 중단되는 상황을 나타냅니다. 시스템은 그 사이에 `readFile(from:)`과 같은 다른 작업을 처리하며, 이미지 데이터가 준비되면 중단되었던 `downloadImage(from:)` 작업을 이어서 재가합니다. 이러한 흐름은 Swift Concurrency가 제공하는 비선점형(Non-Blocking) 비동기 처리의 핵심입니다. 

코드의 실행 흐름이 중간중간 점프(jump)하는 콜백 기반 비동기 함수와는 달리, `async`와 `await`을 사용한 비동기 함수는 코드가 직선적으로 구성되어 가독성이 훨씬 좋고, 흐름을 따라가기도 수월합니다. 또한 예외가 발생할 수 있는 위치에서 예외를 명시적으로 처리하지 않거나, 함수가 적절한 결과를 반환하지 않을 경우, Swift 컴파일러가 이를 감지하여 개발자에게 알려줍니다. 이러한 특성 덕분에 비동기 코드를 마치 동기 코드처럼 자연스럽게 작성할 수 있으며, 코드의 안정성과 유지보수성 또한 크게 향상됩니다.

결과를 반환하는 데 시간이 오래 걸리는 작업을 만나면 스레드를 점유(Blocking)해버리는 `GCD`와는 달리, Swift Concurrency가 제공하는 언어 수준의 동시성은 오래 걸리는 작업이 있을 경우 실행 흐름을 일시 중단하고 스레드의 제어권을 시스템에 반환하여, 해당 스레드가 다른 작업을 실행할 수 있도록 합니다. 이 덕분에 `GCD`에서 자주 발생하던 스레드 폭발(Thread Explosion)에 따른 메모리 및 스케줄링 오버헤드 문제를 피할 수 있으며, 스레드를 더욱 효율적으로 사용할 수 있습니다. 

{% hint style=“info” %}
**Note**
Swift Concurrency에서는 작업마다 새로운 스레드를 생성하지 않고, 물리적인 CPU 코어 수에 맞춰 제한된 수의 스레드만을 생성해 비동기 작업을 처리합니다. 이로 인해 하나의 코어가 여러 스레드를 번갈아 실행할 때 발생하는 스레드 컨텍스트 스위칭이 최소화되어, 보다 안정적이고 높은 성능을 얻을 수 있습니다. 이는 작업들이 스레드를 서로 양보하며 협력적으로 사용한다는 의미에서 협력적 스레드 풀(Cooperative Thread Pool)이라고 불립니다.
{% endhint %}


## Handy Static Methods for Async Function

`Task`는 비동기 컨텍스트와 상호작용하기 위한 다양한 정적(Static) 메서드를 제공합니다. 그중 대표적인 예가 `Task.sleep(for:tolerance:clock:)` 메서드입니다. 이 메서드는 지정된 시간만큼 작업을 일시 중단(suspend)하여, 비동기 작업의 흐름을 제어할 수 있게 해줍니다. 이 메서드는 실제 네트워크 지연이나 I/O 대기와 같은 비동기 상황을 시뮬레이션할 때 매우 유용합니다. 예를 들어, 아래와 같이 임의의 지연을 두어 Swift Concurrency의 동작을 테스트해볼 수 있습니다.

```swift
func downloadImage(from url: String) async throws -> UIImage {
    try await Task.sleep(for: .seconds(1))
    return UIImage()
}
```

또한 `Task.sleep(for:tolerance:clock:)`은 작업이 일시 중단된 상태에서 취소 신호가 전달되면 예외를 던질 수 있으므로, 호출 시 `try`와 함께 사용해야 합니다.

{% hint style=“info” %}
**Note**
비슷한 이름의 함수로 `sleep(_:)`이 있지만, 이 둘은 동작 방식이 근본적으로 다릅니다. `Task.sleep(for:)`는 비동기 함수로, 작업을 일시 중단(suspend)하면서 스레드의 제어권을 시스템에 양보합니다. 이로 인해 다른 작업이 해당 스레드를 사용할 수 있게 되어 효율적인 동시 실행이 가능합니다. 반면, `sleep(_:)`은 동기 함수이며, 스레드를 점유한 채 그대로 대기합니다.
{% endhint %}

`Task.yield()`는 수행 중인 작업을 명시적으로 일시 중단하고, 다른 동시 작업에게 실행 기회를 양보할 수 있도록 도와주는 함수입니다.

```swift
func show(_ images: [ImageStatus]) async throws -> Data {
    for image in images {
        let compressedImage = compress(for: image)
        await Task.yield()
    }
}
```

위 예제에서 `compress(for:)` 함수는 이미지를 압축하는 동기적인 작업이라고 가정해봅시다. 동기 코드는 내부에 일시 중단 지점(Suspension Point)이 존재하지 않기 때문에, 해당 루프가 실행되는 동안 다른 비동기 작업이 개입할 수 없습니다. 그러나 반복문 내부에서  `Task.yield()`를 호출하면, 작업 중간에 명시적으로 일시 중단 지점을 삽입하게 되어 다른 동시 작업에게 스레드를 양보할 수 있습니다. 이를 통해, 압축 작업이 길어질 경우에도 다른 동시 작업의 실행 가능성을 높일 수 있습니다.


# Valid Contexts for Calling Async Functions

비동기 컨텍스트는 실행에 필요한 액터(Actor), Task-Local, 우선순위 등의 메타데이터를 포함하고 있습니다. 이러한 정보들은 비동기 작업의 실행을 추적하고 제어하는 데 사용되므로, Swift에서는 모든 비동기 작업이 반드시 이러한 메타데이터를 가진 비동기 컨텍스트 내에서 수행되어야 합니다. 따라서 `await`이 필요한 비동기 함수는 동기 컨텍스트에서는 호출할 수 없으며, 반드시 비동기 컨텍스트 안에서 호출되어야 합니다. Swift에서 비동기 컨텍스트에 해당하는 대표적인 예시는 다음과 같습니다.

* 비동기 함수, 메서드 또는 프로퍼티의 구현부 내부

* @main 속성으로 지정된 구조체, 클래스, 또는 열거형의 main() 정적 메서드

* 구조화되지 않은 작업(Unstructured Task) 내부


# Understanding Task Suspension and Resumption

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}


## Cooperative Thread Pool

{% hint style="info" %}
**Info** 이 섹션은 현재 작성 중입니다.
{% endhint %}

