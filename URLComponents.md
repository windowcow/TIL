[URLComponents 공식 문서](https://developer.apple.com/documentation/foundation/urlcomponents)

> A structure that parses URLs into and constructs URLs from their constituent parts.

URL을 다룰 때 `URLComponents` 를 사용하는 것이 좋을 수 있다.
URL은 여러가지의 구성 요소로 나뉘고 그 구성 요소들을 추가하거나 변경해서 쓰는 경우가 많기 때문이다.
> 특히 base url을 두고 구성 요소를 추가하는 경우가 가장 많은 것 같다.

하지만 URL 타입 자체로 그 요소들을 다루는 데에는 한계가 있다.

예를 들어서 다음 URL을 살펴보자.
`https://www.example.com/path?key1=value1&key2=value2#section1`

URL은 `스킴://호스트:포트/경로?쿼리#프래그먼트`로 구성 요소를 구분할 수 있다.

• **스킴 (Scheme)**: https

• **호스트 (Host)**: www.example.com

• **경로 (Path)**: /path

• **쿼리 (Query)**: key1=value1&key2=value2

• **프래그먼트 (Fragment)**: section1

`URLComponents`를 사용하면 각각의 요소에 바로 접근해서 읽고 쓸 수 있다.
![](https://i.imgur.com/nguDx8m.png)
이걸 통해서 원하는 요소에 접근하고 수정해보자.
***
### URL 구성 요소 갈아끼우기
`URLComponents`를 사용하면 구성 요소 중 일부를 간단히 갈아 끼울 수 있다.

```swift
let url = URL(string: "https://www.example.com/path?key1=value1&key2=value2#section1")!

var components = URLComponents(url: url, resolvingAgainstBaseURL: false)

print(components?.url) // Optional(https://www.example.com/path?key1=value1&key2=value2#section1)

components?.path = "/newPath" 

print(components?.url) // Optional(https://www.example.com/newPath?key1=value1&key2=value2#section1)
```

***
### 쿼리 다루기
`URLComponents`를 추천하는 가장 큰 이유는 쿼리 파라미터를 다우기가 매우 좋기 때문이다.
관련된 구조체가 있는데 그게 바로 `URLQueryItem`이다.

[URLQueryItem 공식 문서](https://developer.apple.com/documentation/foundation/urlqueryitem)
>A single name-value pair from the query portion of a URL.

이걸 사용하면 URL에서 쿼리에 해당하는 문자열 매우 쉽게 만들 수 있다.

```swift
let url = URL(string: "https://www.example.com/path?key1=value1&key2=value2#section1")!

var components = URLComponents(url: url, resolvingAgainstBaseURL: false)

print(components?.url) // Optional(https://www.example.com/path?key1=value1&key2=value2#section1)

components?.queryItems = [URLQueryItem(name: "key1", value: "value1")]

print(components?.url) // Optional(https://www.example.com/path?key1=value1#section1)
```

-끝-
