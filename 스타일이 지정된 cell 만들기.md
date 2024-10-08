
> 이 내용은 `.value2` 와 같이 애플에서 제공해준 스타일이 지정된 셀을 만드는 내용입니다.
> 완전 커스텀 셀을 만드는 것에 대한 내용이 아닙니다.

![](https://i.imgur.com/F8gQVmV.png)

이런 스타일의 Cell은 많이 쓰이는 만큼 UIKit에 이미 구현이 되어있다.

이런 셀을 만들 때에는 3가지만 생각하면 된다.
1. **override** **init**(style: UITableViewCell.CellStyle, reuseIdentifier: String?)
2. **static** **var** identifier: String
3. **func** configure(title: String, description: String)

#### **override** **init**(style: UITableViewCell.CellStyle, reuseIdentifier: String?)
```swift
override init(style: UITableViewCell.CellStyle, reuseIdentifier: String?) {
	super.init(style: .value2, reuseIdentifier: reuseIdentifier)
}
```

스타일의 지정은 위의 코드처럼 생성자에서 일어난다. 다른 곳에서 바꿀 수 없다.

하지만 우리가 셀을 만들 때 항상 저 init을 명시적으로 호출해서 셀을 만들지는 않는다.

```swift
func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {

	let cell = tableView.dequeueReusableCell(withIdentifier: IssueCreateTableViewCell.identifier, for: indexPath)

	guard let cell = cell as? IssueCreateTableViewCell else {
		return IssueCreateTableViewCell(style: .value2, reuseIdentifier: IssueCreateTableViewCell.identifier)
	}

	return cell
}
```


>tableView.dequeueReusableCell(withIdentifier:for: ) 에 대한 설명
>
>Call this method only from the method of your table view data source object. This method returns an existing cell of the specified type, if one is available, or it creates and returns a new cell using the class or storyboard you provided earlier. 
>...
>When creating cells from a registered class, this method creates the cell and initializes it by calling its [`init(style:reuseIdentifier:)`](doc://com.apple.documentation/documentation/uikit/uitableviewcell/1623276-init)method. 

```swift
let cell = tableView.dequeueReusableCell(withIdentifier: IssueCreateTableViewCell.identifier, for: indexPath)
```

위의 코드에서 직접 생성자를 호출하는 코드를 작성하지 않지만 셀이 생성된다.
> 저 메서드는 셀이 없으면 셀을 생성해서 반환까지 한다.

```swift
override init(style: UITableViewCell.CellStyle, reuseIdentifier: String?) {
	super.init(style: .value2, reuseIdentifier: reuseIdentifier)
}
```

위에서 적은 것처럼 `override init` 안의 `super.init`에 스타일에 관한 인자를 넘겨줘야 `dequeueReusableCell`을 통해 원하는 스타일의 셀을 생성할 수 있다.

### 요약
- style 지정은 생성자에서만 가능하다.
- 하지만 생성자를 내가 명시적으로 호출하지 않는 상황이 많다.
- tableView.dequeueReusableCell(withIdentifier:for: ) 여기가 특히 그렇다.
- 그래서 override init을 통해서 런타임에 저 생성자를 제대로 호출하도록 해야한다.
- 만약 style: .value2로 안하고 style: style로 그대로 넣어주면 스타일이 제대로 지정되지 않는다. 마치 다음 코드처럼 하면 안된다는 뜻이다.
```swift
// 이렇게 하면 스타일 지정이 안됨
override init(style: UITableViewCell.CellStyle, reuseIdentifier: String?) {
	super.init(style: style, reuseIdentifier: reuseIdentifier)
}
```

#### 다음은 완성된 코드와 결과물이다.

```swift
// Tabl
final class IssueCreateTableViewCell: UITableViewCell {

    static var identifier: String {
        String(describing: Self.self)
    }

    override init(style: UITableViewCell.CellStyle, reuseIdentifier: String?) {
        super.init(style: .value2, reuseIdentifier: reuseIdentifier)
    }

    required init?(coder: NSCoder) {
        super.init(coder: coder)
    }

    func configure(title: String, description: String) {
        var contentConfiguration = self.defaultContentConfiguration()
        contentConfiguration.text = title
        contentConfiguration.secondaryText = description
        self.contentConfiguration = contentConfiguration
    }
}
```

```swift
func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {

	let cell = tableView.dequeueReusableCell(withIdentifier: IssueCreateTableViewCell.identifier, for: indexPath)

	guard let cell = cell as? IssueCreateTableViewCell else {

		return IssueCreateTableViewCell(style: .value2, reuseIdentifier: IssueCreateTableViewCell.identifier)

	}

	cell.configure(title: "123", description: "456")

	cell.accessoryType = .disclosureIndicator

	return cell

}
```

결과
![](https://i.imgur.com/rnfVhXp.png)
