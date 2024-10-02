결론부터 말씀드리자면 데이터를 소비하는 쪽에서 데이터를 요청하는 방식이라면 `async/await`만 사용해도 상관없습니다.

> **예시**: 뷰 컨트롤러에서 버튼을 누르면 `issues`를 레포지토리에 요청하고 반환 받아 사용하는 상황.

하지만 데이터가 변경될 때 소비자가 그 변경을 받아서 사용하려면 **Combine**을 이용해야 합니다.

> **예시**: 레포지토리의 `issues`가 변경되었을 때 여러 뷰 컨트롤러에서 테이블 뷰를 리로드하는 상황.

물론 이 룰을 꼭 지킬 필요는 없지만, **Combine**은 이러한 문제를 아주 쉽게 해결해주는 목적으로 만들어졌습니다.

조금 더 깊은 이해를 위해  반응형 프로그래밍에 대해 설명하겠습니다.

---

## 데이터 흐름의 방식: Pull 모델

기존의 프로그래밍에서는 데이터가 필요한 쪽에서 데이터를 요청했습니다. 예를 들어, `ViewController`에서 버튼이 눌리면 다음과 같은 코드를 실행할 수 있습니다:

```swift
let issues = await repository.fetch()
```

이러한 **데이터 흐름 방식**을 **Pull 모델**이라고 합니다. Pull 방식에서는 데이터가 필요한 **객체**가 해당 데이터가 **필요한 시점**에 **데이터를 제공하는 객체**에게 메서드를 통해 요청합니다. 하지만 데이터가 필요한 시점을 알기 어려울 수 있습니다.

> **예시**: 뷰 컨트롤러 A에서 `fetch`를 할 때, 뷰 컨트롤러 B도 `fetch`를 해야 하는지 B 입장에서 스스로 알기 어렵습니다.

상황을 하나 가정해봅시다. 모든 `issues`가 들어있는 **source of truth**가 있습니다. 이 `issues` 배열이 변경되는 시점이 뷰 컨트롤러들이 데이터를 Pull 할 시점인데, 뷰 컨트롤러들 입장에서 이를 감지하도록 코드를 짜는 것은 어렵습니다.

> 참고로 이때 `NotificationCenter`나 Pub/Sub 패턴을 사용해서 알아낼 수 있습니다. 이를 감지하는 수고로움이 견딜만 하면 굳이 반응형 프로그래밍을 사용할 필요는 없다고 느껴집니다. 소비자가 요청할 시점만 잘 파악한다면 데이터를 요청한 후 받아서 선언형으로 처리해도 비슷하니까요!

이렇듯 기존에는 데이터의 **소비자(consumer)** 가 주도적인 역할을 했습니다. 중요한 것은 데이터 원본 그 자체인데 말이죠.

앞서 말한 소비자 주도의 데이터 흐름의 문제점을 해결하기 위해서 반응형 프로그래밍이 나온 면도 있습니다.
데이터 변경이 먼저고, 그 내용을 소비자가 받아서 처리하기 위해서죠.

---

## 데이터 흐름의 방식: Push 모델

**Push 모델**에서는 데이터의 **생산자(producer)** 가 **먼저 주도적으로** 소비자에게 값을 전달합니다. 
아까처럼 소비자 쪽에서 먼저 데이터를 요청하지 않습니다. 
생산자가 먼저 소비자에게 데이터를 전달합니다.

생산자를 `issues: [Issue]`를 프로퍼티로 가지는 `Repository`라고 가정합시다. 어떻게 소비자들인 `ViewController`들에게 전달할 수 있을까요?

> **주의**: 사실 Combine에서 producer의 역할은 `Publisher`이고 consumer은 `Subscriber`입니다. `Repository`나 `ViewController`는 각각 producer/consumer라기보다는 그걸 관리하는 쪽에 가깝다고 생각합니다.

물론 다음 코드처럼 레포지토리가 뷰 컨트롤러들의 참조를 가지면 주도적으로 데이터 스트림을 시작할 수 있습니다.

```swift
class Repository {
    var issues: [Issue] = [] {
        willSet {
            viewControllers.forEach { viewController in
                viewController.configure(newValue)
            }
        }
    }

    var viewControllers: [UIViewController] = [viewController1, viewController2]

    func fetch() {
        DispatchQueue.global().async {
            URLSession.shared.dataTask(with: url) { [weak self] _, _, _ in
                // Completion Handler
                self?.issues = 결과
            }
        }
    }
}
```

하지만 **의존성 문제**가 생기기에 Combine에서는 **Loose Coupling**을 목적으로 **Publisher-Subscriber 패턴** 을 사용합니다.

생산자(레포지토리)가 주도적으로 소비자(뷰 컨트롤러)에게 먼저 데이터를 보내고 소비자(뷰 컨트롤러)가 선언형의 방식으로 생산자로부터 받은 데이터를 가공해서 사용하죠. 다음 코드처럼 말입니다.

```swift
var initialIssues = [
    Issue(assignee: "Alice", body: "Fix login bug"),
    Issue(assignee: "Bob", body: "Update documentation"),
    Issue(assignee: "Charlie", body: "Improve UI design"),
    Issue(assignee: "Diana", body: "Optimize database queries"),
]

class Repository {
    static let shared = Repository()
    private init() {}

    let issueDataSource = CurrentValueSubject<[Issue], Never>(initialIssues)

    func fetch() async {
        try? await Task.sleep(for: .seconds(3))

        let currentIssues = issueDataSource.value

        issueDataSource.send(currentIssues.suffix(5))
    }
}

class MyViewController: UIViewController {
    var subscriptions: Set<AnyCancellable> = []

    override func viewDidLoad() {
        Repository.shared.issueDataSource
            .map { issues in // 데이터 스트림의 일부
                issues.map { $0.id }
            }
            .sink { _ in // 소비자(subscription)을 만드는 것
                // 받은 데이터 가공
            }
            .store(in: &subscriptions)
    }
}
```

이게 바로 **Push 방식의 생산자 주도형 데이터 흐름**입니다.

---

## 이 내용이 Combine과 async/await 중 무엇을 써야 하는 것과 무슨 상관?

방금까지 본 내용에서는 생산자(Publisher)로 이미 있는 데이터가 쓰였습니다. 하지만 아직은 없는, 비동기 함수에 대한 반환값도 Push 방식의 생산자로 쓰일 수 있어야 했습니다. 그래서 나온 게 `dataTaskPublisher`입니다. (사실 다른 것도 있음)

```swift
let url = URL(string: "https://api.example.com/data")!

let publisher = URLSession.shared.dataTaskPublisher(for: url)
    .map { $0.data }
    .decode(type: MyDataModel.self, decoder: JSONDecoder())

let cancellable = publisher
    .sink(
        receiveCompletion: { completion in
            switch completion {
            case .finished:
                print("데이터 수신 완료")
            case .failure(let error):
                print("에러 발생: \(error)")
            }
        },
        receiveValue: { dataModel in
            print("수신된 데이터 모델: \(dataModel)")
        }
    )
```

반환값을 가지는 비동기 함수라는 점에서 async/await에서 말하는 `동기 코드처럼 보이는 비동기 코드`와 닮아있습니다. 기존의 `DispatchQueue`와 completion handler를 이용한 비동기 함수는 반환값을 가질 수 없었으니까요.

> **참고**: Combine이 2019년에, Swift Concurrency가 2021년에 나왔습니다.

실제로 Combine이 나온 시점에는 async/await이 없었기 때문에, 저런 코드를 통해서 `DispatchQueue`나 completion handler를 사용하지 않고 비동기 함수의 반환값을 절차적으로 처리할 수 있게 되었습니다.

> 그 이후에 async/await이 나왔습니다.

`async/await`을 통해서 아주 복잡한 비동기 코드를 절차적으로 작성할 수 있는 것은 확실하지만, Combine을 통해서 해결하려고 한 문제와는 방향이 다르다고 느껴집니다.

Combine에서는 앞서 보았던 소비자 주도의 데이터 흐름의 문제를 해결하기 위해서 나왔습니다.
비동기 함수를 선언형으로 처리하는 것을 목적으로 만들어진 프레임워크는 아닙니다.
생산자 주도의 데이터 흐름을 위해서는 비동기 함수가 completion handler를 사용하는 대신에 반환 값을 가지는 것처럼 만들어서 절차적으로 처리할 수 있도록 만든 것이지요.

`async/await`을 통해서는 소비자 주도의 Pull 방식의 데이터 흐름 패러다임에서 벗어날 수 없습니다. 여전히 소비자(예: `ViewController`)에서 먼저 자신이 필요한 비동기 함수의 반환 값을 `await`으로 누군가(예: `Repository`)에게 요청해야 하기 때문이죠.

---

## 결론

### 각 개념의 역할

**Combine**은:

- 데이터 **생산자**로부터
- **선언적**으로 그 데이터를 가공해서
- 데이터 **소비자**에게 주도적으로 먼저 데이터를 전달하기 위한 프레임워크입니다.

**async/await**은:

- `DispatchQueue`와 completion handler를 사용하지 않고
- 절차적으로 보이는 비동기 코드를 작성하는 방법론입니다.
- 이를 통해 그간 엄청 복잡하던 비동기 코드를 절차적으로 작성할 수 있게 되었습니다.
- (물론 이 외에도 실행 중인 스레드를 양보해가며 스레드 점유를 하지 않는 등 많은 고급 기능들이 있지만 말입니다.)

---

## 참고 자료

- [반응형 프로그래밍](https://yozm.wishket.com/magazine/detail/1334/)

---
