결론부터 말씀드리자면 
데이터를 소비하는 쪽에서 데이터를 요청하는 방식을 사용하면 async/await만 사용해도 상관 없습니다.
> 예를 들어, 뷰컨트롤러가 버튼을 누르면 issues를 레포지토리에 요청하고 반환 받아 쓰는 상황

하지만 데이터 변경되었을 때 소비되는 쪽에서 그걸 받아서 사용하려면 Combine을 이용해야 합니다.
> 예를 들어, 레포지토리의 issues들이 변경되었을 때 여러 뷰컨에서 table view를 리로딩 하는 상황

물론 저 룰을 꼭 지킬 필요는 없지만 Combine이 그걸 아주 쉽게 해결해주는 목적으로 만들어졌습니다.

가장 먼저  반응형 프로그래밍에 대해 설명하겠습니다.

# 패러다임의 변화

## 데이터 흐름의 방식: Pull 모델 (또는 Pull 방식)
기존의 프로그래밍은 데이터가 필요한 쪽에서 데이터를 요청했습니다.
예를 들면 ViewController에서 버튼이 눌리면 다음 코드를 실행할 수도 있었겠지요.

```swift
let issues = await repository.fetch()
```

이런 **데이터 흐름의 방식**을 **Pull 모델**이라고 합니다.

Pull 방식의 데이터 흐름이 바로 데이터가 필요한 **객체**가 해당 데이터가 **필요한 시점**에 **데이터를 제공하는 객체**에게 메시지(메서드)를 통해 요청하는 겁니다.
하지만 데이터가 필요한 시점을 알기 어려울 수 있습니다.
> 예를 들어서 뷰컨 A에서 fetch를 할 때 뷰컨 B도 fetch 해야하는지 뷰컨B입장에서 스스로 알기는 어렵습니다.

상황을 하나 가정해봅시다.
모든 issues가 들어있는 source of truth가 있을 수 있습니다.
이 issues 배열이 변경되는 시점이 뷰컨트롤러들이 데이터를 Pull 할 시점인데, 뷰컨트롤러들 입장에서 이를 감지하도록 코드를 짜는 것은 어렵습니다.

> 참고로 이때  NotificationCenter나 Pub/Sub 패턴을 사용해서 알아낼 수 있습니다.
> 제 생각에는 이를 감지하는 수고로움이 견딜만 하면
> 굳이 반응형 프로그래밍을 사용할 필요는 없다고 느껴집니다.
> 소비자가 요청할 시점만 잘 파악한다면 데이터를 요청한 후 받아서 선언형으로 처리해도 비슷하니까요!

이렇듯 기존에는 데이터의 소비자(consumer)가 주도적인 역할을 했습니다.
중요한 것은 데이터 원본 그 자체인데 말이죠.

반응형 프로그래밍에서는 
데이터의 생산자(producer)가 주도적인 데이터 흐름의 방식이 더 좋다고 이야기합니다.
이게 바로 Push 모델입니다.

# 데이터 흐름의 방식: Push 모델
Push 모델에서는 데이터의 생산자(producer)가 **먼저 주도적으로** 소비자에게 값을 전달합니다.
다시 말해서 아까처럼 소비자 쪽에서 먼저 데이터를 요청하지 않습니다.
생산자가 먼저 소비자에게 데이터를 전달합니다.

이제 생산자를 issues: [Issue] 를 프로퍼티로 가지는 Repository라고 가정합시다.
어떻게 소비자들인 ViewController들에게 전달할 수 있을까요?

> [!WARNING]
> 사실 Combine에서 producer의 역할은 Publisher입니다.
> 그리고 consumer은 Subscriber입니다.
> Repository나 ViewController는 각각 producer/consumer라기보다는 그걸 관리하는 쪽에 가깝다고 생각합니다.

물론 다음 코드처럼 레포지토리가 뷰컨트롤러들의 참조를 가지면 주도적으로 데이터 스트림을 시작할 수 있습니다.

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
            URLSession.shared.dataTask(with: url) { [weak self] _,_,_  in
                // Completion Handler
                self?.issues = 결과
            }
        }
    }
}
```

>  하지만 의존성 문제가 생기기에 이렇게 하지 않기 위해서 Loose Coupling을 목적으로 publisher - subscriber 패턴이 나왔습니다.

Combine에서는 생산자(레포지토리)가 주도적으로 소비자(뷰컨트롤러)에게 먼저 데이터를 보내야하지만, 참조를 가지지 않기 위해서 Publisher-Subscriber 패턴을 사용합니다.

그리고 소비자(뷰컨트롤러)가 선언형의 방식으로 생산자로부터 받은 데이터를 가공해서 사용하죠.
다음 코드처럼 말입니다.
```swift
var initialIssues =  [
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
            .map{ issues in // 데이터 스트림의 일부
                issues.map { $0.id }
            }
            .sink { _ in // 소비자(subscription)을 만드는 것
                // 받은 데이터 가공
            }
            .store(in: &subscriptions)
    }
}
```
이게 바로 push 방식의 생산자 주도형의 데이터 흐름입니다.

# 이 내용이 Combine과 async/await중 뭘 써야 하는 것과 무슨 상관?
방금까지 본 내용에서는 생산자(Publisher)로는 이미 있는 데이터가 쓰였습니다.
하지만 비동기 함수에 대한 반환값도 push 방식의 생산자로 쓰일 수 있어야 했습니다.
그래서 나온게 dataTaskPublisher입니다.

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


반환값 가지는 비동기 함수라는 점에서  async/await에서 말하는 `동기 코드처럼 보이는 비동기코드`와 닮아있죠. 
기존의 DispatchQueue와 completion handler를 이용한 비동기 함수는 반환값을 가질 수 없었으니까요.
함수형처럼 값을 변환해서 사용하기 위해서 비동기 함수가 값을 반환하게 되었습니다.

>Combine이 2019년, Swift Concurrency가 2021년에 나왔습니다.

실제로 Combine이 나온 시점에는 async/await이 없었기 때문에 저런 코드를 통해서 DispatchQueue나 completion handler를 사용하지 않고 비동기 함수의 반환값을 절차적으로 처리할 수 있게 되었습니다.

> 그 이후에 async/await이 나왔습니다.

async - await을 통해서 아주 복잡한 비동기 코드를 절차적으로 작성할 수 있는 것은 확실하지만 Combine을 통해서 해결하려고 한 문제와는 방향이 다르다고 느껴집니다.

async - await를 통해서는 consumer 주도의 pull 방식의 데이터 흐름 패러다임에서 벗어날 수 없습니다.
여전히 소비자(ex. viewController)에서 먼저 자신이 필요한 비동기 함수의 반환 값을 await으로 누군가(ex. 레포지토리)에게 요청해야하기 때문이죠.
# 결론
### 각 개념의 역할
Combine은 
- 데이터 생산자로부터
- 선언적으로 그 데이터를 가공해서
- 데이터 소비자에게 주도적으로 먼저 데이터를 전달하기 위한 프레임워크
async/await은 마스터 섹션에 배운 것처럼
- DispatchQueue와 completion handler를 사용하지 않고 
- 절차적으로 보이는 비동기 코드를 작성하는 방법론입니다.
- 이걸 통해서 그간 엄청 복잡하던 비동기 코드를 절차적으로 작성할 수 있게 되었습니다.
- (물론 이거 말고도 실행 중인 스레드를 양보해가며 스레드 점유를 안 하는 등 다양한 역할이 있다고 생각합니다.)


https://yozm.wishket.com/magazine/detail/1334/