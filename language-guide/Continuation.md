---
description: 콜백 기반 비동기 API를 Async-Await 기반 비동기 함수로 변환하는 방법
---

`async`와 `await`을 사용해 비동기 함수(Asynchronous Function)를 작성하면, 콜백 기반 비동기 API보다 가독성이 훨씬 높아지고, 코드의 안정성과 유지보수성 또한 크게 향상됩니다. 다만, 기존 프로젝트에서 이미 콜백 기반으로 구현된 모든 비동기 코드를 `async`, `await` 기반으로 전환하는 일은 결코 간단하지 않습니다. 이를 보다 안전하고 유연하게 전환할 수 있도록 Swift Concurrency는 `CheckedContinuation` API를 제공합니다.

이 API를 사용하면 비동기 코드의 실행을 일시 중단(suspend)하고, 개발자가 원하는 시점에 코드 흐름을 재개(resume)할 수 있습니다. `CheckedContinuation`은 시스템이 관리하는 특정 시점의 실행 상태를 캡처해 저장하는 객체인 `Continuation`에 좀 더 직접적으로 접근하고 제어할 수 있도록 도와줍니다.

# Bridging Callbacks and Async/Await with Continuation API

`Check(Unsafe)Continuation` API는 크게 `withCheckContinuation(_:)`, `withCheckedThrowingContinuation(_:)`, `withUnsafeContinuation(_:)` 그리고 `withUnsafeThrowingContinuation(_:)`으로 4가지 유형이 있습니다.

## withCheckedContinuation

`withCheckedContinuation(_:)`는 일반적인 상황에서 콜백 기반 비동기 API를 `async`, `await` 기반 비동기 함수로 안전하게 전환할 수 있도록 도와주는 도구입니다. 비동기 함수의 일시 중단과 재개를 수동으로 제어할 수 있게 해주는 핵심 도구 중 하나입니다. 

함수를 호출하면 _body_ 클로저가 즉시 실행되며, 이 클로저는 `CheckedContinuation<T, Never>` 인스턴스를 매개변수로 전달받습니다. 클로저 내부에서 `continuation.resume(returning:)`을 호출하면 중단된 작업이 다시 재개되면서, 결과값이 반환됩니다.

```swift
await withCheckedContinuation<String> { continuation in
    continuation.resume(returning: "Hello, Continuation!")
}
```

예외를 던질 수 있는 콜백 기반 API를 비동기 함수로 전환해야 하는 경우에는 `withCheckedThrowingContinuation(_:)`을 사용해야 합니다.

`resume()` 메서드는 실행 흐름 전체에서 반드시 단 한번만 호출되어야 합니다. 만약 `resume()`이 한 번도 호출되지 않으면, 해당 작업은 완료되지 않은 채 일시 중단 상태로 영구적으로 남게 되며, 시스템이 이를 해제할 수 없기 때문에 메모리 누수(leak)로 이어질 수 있습니다. `resume()`을 호출하지 않거나 중복 호출하는 것은 모두 심각한 오류로 간주됩니다.

`withCheckedContinuation(_:)`는 이러한 `Continuation`의 잘못된 사용을 검사합니다. `resume()`을 한 번도 호출하지 않으면 콘솔 로그에 `Continuation`이 잘못 사용되었다고 알려줍니다. 중복으로 호출하면 크래시가 나며 앱이 강제로 종료됩니다.

```
SWIFT TASK CONTINUATION MISUSE: downloadImage(from:) leaked its continuation without resuming it. This may cause tasks waiting on it to remain suspended forever.
```


## withUnsafeContinuation

`withUnsafeContinuation(_:)`은 앞서 살펴본 `withCheckedContinuation(_:)`과 기능적으로는 동일하지만, 안전성 검사가 생략된다는 차이가 있습니다. `withCheckedContinuation(_:)`은 `Continuation`이 올바르게 사용되지 않은 경우 런타임 경고를 통해 문제를 알려주지만, `withUnsafeContinuation(_:)`는 이러한 검사를 수행하지 않으며, `Continuation`을 잘못 사용할 경우 크래시가 나며 앱이 강제로 종료됩니다. 하지만 이러한 검사를 생략한 덕분에, `withCheckedContinuation(_:)`보다 더 나은 성능을 제공한다는 장점이 있습니다.

```swift
try await withUnsafeThrowingContinuation { continuation in
    continuation.resume(with: .success("Hello, Continuation!"))
}
```

아래 표는 `Check(Unsafe)Continuation`가 제공하는 API 종류를 정리한 것입니다.

| 항목 | 특징 | 안전성 검사 |
| -   | -   | -      | 
| **withCheckedContinuation(_:)** | 예외를 던지지 않는 일반 비동기 작업을 위한 `Continuation` 생성 | ✅ |
| **withCheckedThrowingContinuation(_:)** | 예외를 던질 수 있는 비동기 작업을 위한 `Continuation` 생성 | ✅ |
| **withUnsafeContinuation(_:)** | 안전성 검사를 생략한 일반 비동기 작업을 위한 `Continuation` 생성 | ❌ |
| **withUnsafeThrowingContinuation(_:)** | 안전성 검사를 생략하고, 예외를 던질 수 있는 비동기 작업을 위한 `Continuation` 생성 | ❌ |



# Converting an Asynchronous Callback-Based API to Async-Await

아래 예제는 콜백 기반 API로 구현된 이미지를 다운로드하는 코드입니다.

```swift
func downloadImage(from url: String,
                   completion: @escaping (Result<UIImage, Error>) -> Void) {
    
    let task = URLSession.shared.dataTask(with: URL(string: url)!) { data, response, error in
        if let error = error {
            completion(.failure(error))
            return
        } else {
            guard let response = response as? HTTPURLResponse,
                  response.statusCode == 200 else {
                completion(.failure(URLError(.badServerResponse)))
                return
            }
            
            if let data = data,
               let image = UIImage(data: data) {
                
                prepareThumbnail(from: image) { image in
                    if let image = image {
                        DispatchQueue.main.async {
                            completion(.success(image))
                            return
                        }
                    }
                }
            } else {
                completion(.failure(URLError(.cannotDecodeContentData)))
                return
            }
        }
    }
    
    task.resume()
}
```

```swift
func downloadImage(from url: String) async throws -> UIImage {
    try await withCheckedThrowingContinuation { continuation in
        downloadImage(from: url) { result in
            switch result {
            case .success(let image):
                continuation.resume(returning: image)
            case .failure(let error):
                continuation.resume(throwing: error)
            }
        }
    }
}
```

먼저, 예외를 던질 수 있는 `downloadImage(from:)` 비동기 함수를 정의하고, 반환 타입을 `UIImage`로 지정합니다.

함수 내부에서 `withCheckedThrowingContinuation(_:)`을 호출하고, 그 클로저 내부에 기존의 콜백 기반 메서드인 `downloadImage(from:completion:)`을 실행합니다. 콜백이 완료되는 시점에 `continuation.resume(returning:)` 또는 `continuation.resume(throwing:)`을 호출하여, 일시 중단된 작업을 재개할 수 있습니다.

이와 같이 기존의 콜백 기반 API를 `async`, `await` 기반 비동기 함수로 자연스럽게 전환할 수 있습니다.


# Converting a Delegate-Based API to Async-Await

많은 API는 우리 앱에 중요한 이벤트나 알림을 전달하기 위해 델리게이트(Delegate)을 사용합니다. 이러한 델리게이트 기반 비동기 API를 `async`, `await` 기반 비동기 함수로 전환할 때는,`Continuation` 객체를 클로저 외부에 저장핻고, 실제 데이터가 생성되는 시점에 `continuation.resume(returning:)`를 호출하여, 일시 중단된 작업을 재개할 수 있습니다.

```swift
public class ViewController: UIViewController {
    private var activeContinuation: CheckedContinuation<[CLLocation], Error>?
    public func currentLocations() async throws ->[CLLocation] {
        try await withCheckedThrowingContinuation { continuation in
            self.activeContinuation = continuation
            self.locationManager.startUpdatingLocation()
        }
    }
}

extension ViewController: CLLocationManagerDelegate {
    public func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        self.activeContinuation?.resume(returning: locations)
        self.activeContinuation = nil // guard against multiple calls to resume
        self.manager.stopUpdatingLocation()
    }
    public func locationManager(_ manager: CLLocationManager, didFailWithError error: any Error) {
        self.activeContinuation?.resume(throwing: error)
        self.activeContinuation = nil // guard against multiple calls to resume
    }
}
```

`Checked(Unsafe)Continuation`의 API 규칙을 준수하기 위해 반드시 `Continuation`을 재개한 후, `nil`로 설정하여 한 번 이상 호출하는 실수를 방지해야 합니다.



