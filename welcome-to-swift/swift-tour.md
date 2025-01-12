---
icon: swift
description: 스위프트의 구문과 기능을 알아봅시다.
---

# A Swift Tour

새로운 언어를 배울 때 첫 번째로 작성하는 프로그램은 화면에 "Hello, world!"를 출력하는 것이 전통입니다. 스위프트에서는 이 작업을 한 줄로 할 수 있습니다.

```swift
print("Hello, world!")
// Prints "Hello, world!"
```

이 구문은 다른 언어을 알고 계신다고 익숙하게 보일 겁니다. 스위프트에서는 이 코드 한 줄이 완성된 프로그램입니다. 텍스트를 출력하거나 문자열을 처리하는 기능을 위해 별도의 라이브러리를 가져올 필요가 없습니다. 전역 범위에 작성된 코드는 프로그램의 진입점으로 사용되므로, `main()` 함수가 필요하지 않습니다. 또한, 각 구문의 끝에 세미콜론을 작성할 필요가 없습니다.

이 페이지는 다양한 프로그래밍 작업을 수행하는 방법을 보여주며, 스위프트에서 코딩을 시작하는 데 필요한 충분한 정보를 제공합니다. 이해하지 못하더라도 걱정하지 마세요. 이 페이지에서 소개되는 모든 내용은 이 책의 나머지 부분에서 자세히 설명됩니다.

## Simple Values

`let`을 사용해 상수를 만들고, `var`를 사용해 변수를 만드세요. 상수의 값은 컴파일 타임에 알 필요가 없지만, 반드시 한 번만 값을 상수에 할당해야 합니다. 이는 상수를 사용해 한 번 정한 값에 이름을 부여하고, 이를 여러 곳에서 사용할 수 있음을 의미합니다.

```swift
var myVariable = 42
myVariable = 42
let myConstant = 42
```

상수나 변수는 할당하려는 값과 반드시 동일한 타입이어야 합니다. 그러나 항상 타입을 명시적으로 적을 필요는 없습니다. 상수나 변수를 생성할 때 값을 제공하면, 컴파일러는 해당 값의 타입을 추론합니다. 위 예제에서 컴파일러는 _myVariable_의 초기 값이 정수이기 때문에 해당 변수를 정수로 추론합니다. 

만약 초기 값이 충분한 정보를 제공하지 않는다면(또는 초기 값이 없다면), 변수 이름 뒤에 콜론을 사용하여 타입을 명시해야 합니다. 

```swift
let implicitInteger = 70
let implicitDouble = 70.0
let explicitDouble: Double = 70
```

{% hint style="info" %}
**Experiment**
명시적으로 `Float` 타입으로 지정한 상수를 생성하고, 값으로 4를 할당해보세요.
{% endhint %}

값은 다른 타입으로 암시적으로 변환되는 일이 절대 없습니다. 값을 다른 타입으로 변환하려면, 변환하고자 하는 타입의 인스턴스를 명시적으로 생성하세요.

```swift
let label = "The width is "
let width = 94
let widthLabel = label + String(width)
```

{% hint style="info" %}
**Experiment**
마지막 줄에서 `String`으로 변환하는 코드를 제거해 보세요. 어떤 에러가 발생하나요?
{% endhint %}

문자열에 값을 포함하는 더 쉬운 방법이 있습니다. 값을 괄호 안에 적고, 괄호 앞에 백슬레시(\)를 작성하세요. 예를 들어,

```swift
let apples = 3
let oranges = 5
let appleSummary = "I have \(apples) apples."
let fruitSummary = "I have \(apples + oranges) pieces of fruit."
```

{% hint style="info" %}
**Experiment**
인사를 위해 누군가의 이름을 포함하거나, 문자열에 부동소수점 연산 결과를 포함하려면 _\()_을 사용하세요.
{% endhint %}

여러 줄로 구성된 문자열은 세 개의 쌍따옴표(""")를 사용하세요. 따옴표로 감싼 각 줄의 들여쓰기는 닫는 따옴표와 들여쓰기가 일치하면 제거됩니다. 예를 들어,

```swift
let quotation = """
    Even though there's whitespace to the left,
    the actual lines aren't indented.
        Except for this line.
    Double quotes (") can appear without being escaped.
    
    I still have \(apples + oranges) pieces of fruit.
    """
```

대괄호([])를 사용해 배열과 딕셔너리를 생성하고, 대괄호 안에 인덱스나 키를 적어 요소에 접근하세요. 마지막 요소 뒤에 콤마를 사용할 수 있습니다.

```swift
let fruits = ["strawberries", "limes", "tangerines"]
fruits[1] = "grapes"

var occupations = [
    "Malcolm": "Captain",
    "Kaylee": ""Mechanic",
]
occupations["Jayne"] = "Public Relations"
```

배열에 요소를 추가하면 자동으로 크기가 커집니다.

```swift
fruits.append("blueberries")
print(fruits)
// Prints "["strawberries, "grapes", "tangeries", "blueberries"]"
```

대괄호를 사용해 빈 배열이나 딕셔너리를 작성할 수 있습니다. 배열의 경우, []를 작성하고, 딕셔너리의 경우, [:]를 작성하세요.

```swift
fruits = []
occupations = [:]
```

만약 새로운 변수나 타입 정보가 없는 다른 곳에 빈 배열이나 딕셔너리를 할당한다면, 타입을 명시해야 합니다.

```swift
let emptyArray: [String] = []
let emptyDictionary: [String: Float] = [:]
```


{% hint style="info" %}
**제목** (내용)
{% endhint %}


{% hint style="info" %}
**제목** (내용)
{% endhint %}
