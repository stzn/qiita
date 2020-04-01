iOS13では  
`UISearchBar`に関連する下記のクラスが追加されました。  
  
**UISearchTextField**  
https://developer.apple.com/documentation/uikit/uisearchtextfield  
  
**UISearchToken**  
https://developer.apple.com/documentation/uikit/uisearchtoken  
  
**UISearchTextFieldDelegate**  
https://developer.apple.com/documentation/uikit/uisearchtextfielddelegate  
  
  
色々調べてみたのですが  
あまり情報が出てきませんでした。  
  
WWDC2019のセッションでもサンプルコードを出すと言っていたのですが  
現在まだ見つけることができていません。  
もし正式な情報の場所などご存知の方いらっしゃれば教えてください🙇🏻‍♂️  
  
WWDC2019のセッション動画  
https://developer.apple.com/videos/play/wwdc2019/224/?time=1276  
  
そこで  
今回は実際にコードを動かしながら  
どういう風に活用できるのかを検討した結果を記載したいと思います。  
  
# 各クラスの概要  
  
## UISearchTextField  
  
こちらは元々非公開のプロパティでしたが  
iOS13より公開され  
`UISearchBar`のテキストフィールドのカスタマイズが  
より簡単にできるようになりました。  
  
下記のように`searchTextField`からアクセスできます。  
  
```swift

var searchBar = UISearchBar()
searchBar.searchTextField.backgroundColor = .systemOrange
searchBar.searchTextField.textColor = .systemPurple
searchBar.searchTextField.font = UIFont(name: "American Typewriter", size: 18)

```  
  
このように設定すると下記の様にテキストフィールドの背景色とテキストの色を変更できます。  
  
![Simulator Screen Shot - iPhone 11 Pro Max - 2019-11-23 at 12.05.02.png](/image/c7a97478-24af-89fb-4a30-0bd5f16601f7.png)  
  
## UISearchToken  
  
こちらは検索キーワードを一個のタグのように  
取り扱うことができます。  
  
例えば下記のように`searchTextField`プロパティへトークンを追加すると  
下記の様に表示されます。  
  
```swift

let phoneToken = UISearchToken(icon: UIImage(systemName: "phone"), text: "電話")
let messageToken = UISearchToken(icon: UIImage(systemName: "message"), text: "メッセージ")
searchBar.searchTextField.insertToken(phoneToken, at: 0)
searchBar.searchTextField.insertToken(messageToken, at: 0)
searchBar.searchTextField.tokenBackgroundColor = .systemBlue
```  
  
![Simulator Screen Shot - iPhone 11 Pro Max - 2019-11-23 at 11.41.42.png](/image/78be9525-ec23-fa68-ce09-85ab8582e5b1.png)  
  
※ トークンの色は`searchTextField`のプロパティに設定しているので  
個々のトークンで色を決めることはできないようです。  
(あったら嬉しいかなとちょっと思ったりしました。)  
  
## UISearchTextFieldDelegate  
  
`searchTextField(_:itemProviderForCopying:)`というメソッドを一つ持つProtocolです。  
  
https://developer.apple.com/documentation/uikit/uisearchtextfielddelegate/3175446-searchtextfield  
  
引数から見るとコピー&ペーストで使用するように思われます。  
(WWDCでもコピー&ペーストと言っていました)  
  
こちら全然情報見つからなかったのですが  
検した結果を後ほど記載させていただきたいと思います。  
  
# 実装  
  
もう少し詳しく動作を確認するために  
`UISearchBar`を使って  
`UITableView`に表示されるデータの絞り込みをします。  
  
※ 動作の確認をするもので内容は適当ですので予めご了承ください。  
  
## 事前準備  
  
まず  
下記のような`Plan`というenumを用意します。  
3つのcaseがあって  
それぞれにitemsが存在します。  
  
`UISearchBar`に入力されたキーワードの最初の２文字が一致したら  
インスタンスを生成するようにinitを用意します。  
  
```swift

enum Plan: String, CaseIterable {
    case business
    case travel
    case shopping

    init?(_ text: String) {
        if text.starts(with: "bu") {
            self = .business
        } else if text.starts(with: "tr") {
            self = .travel
        } else if text.starts(with: "sh") {
            self = .shopping
        } else {
            return nil
        }
    }

    var items: [String] {
        switch self {
        case .business:
            return ["会議", "商談", "プレゼン", "勉強会"]
        case .travel:
            return ["宿泊", "日帰り"]
        case .shopping:
            return ["スーパー", "デパート"]
        }
    }

    var iconName: String {
        switch self {
        case .business:
            return "bag"
        case .travel:
            return "airplane"
        case .shopping:
            return "cart"
        }
    }
}
```  
  
次に  
`UISearchBar`の設定です。  
  
```swift

class ViewController: UIViewController {
    @IBOutlet weak private var tableView: UITableView!

    private var searchController = UISearchController()

    private var searchBar: UISearchBar {
        searchController.searchBar
    }

   override func viewDidLoad() {
        super.viewDidLoad()
        setupSearchBar()
    }

    private func setupSearchBar() {
        searchController.searchResultsUpdater = self
        searchController.searchBar.placeholder = "検索したいキーワード"

        // UISearchTextField
        searchBar.searchTextField.backgroundColor = .systemOrange
        searchBar.searchTextField.textColor = .systemPurple
        searchBar.searchTextField.font = UIFont(name: "American Typewriter", size: 18)
        searchBar.searchTextField.tokenBackgroundColor = .systemBlue
    }
}
```  
  
ここは上記で見たように表示方法などの設定を行っています。  
  
  
全体は最後に記載させていただきますので  
ここからは要点だけ見ていきます。  
  
  
## トークンの作成  
  
最初にトークンを作成する実装をしていきます。  
下記のメソッドは`UISearchBar`に入力された時に  
トークンを作成するメソッドです。  
  
```swift

extension ViewController {

    private func setToken(from plan: Plan) {
        // 1
        let planToken = UISearchToken(icon: UIImage(systemName: plan.iconName), text: plan.rawValue)
        // 2
        planToken.representedObject = plan
        // 3
        let field = searchBar.searchTextField
        field.replaceTextualPortion(of: field.textualRange, with: planToken, at: field.tokens.count)
    }
}
```  
  
  
まず、1で`UISearchToken`のインスタンスを生成しています。  
  
2では`UISearchToken`が公開している`representedObject`プロパティに  
それぞれのトークンを識別するための値を設定できます。  
  
取得する際には下記のように`tokens`というプロパティから探します。  
  
```swift

private func extractSearchPlans() -> [Plan] {
    searchBar.searchTextField.tokens.compactMap { $0.representedObject as? Plan }
}
```  
  
3で入力した文字列をトークンに置き換えています。  
  
`replaceTextualPortion`ですが  
ソースコメントに下記のような説明が記載されています。  
  
```swift

Removes any text contained in the specified range, 
inserts the provided token at the specified index, 
and selects the newly-inserted token. 
Does not replace any tokens within the provided range. 
If the range intersects the marked text range, the marked text is committed.

This method is essentially a convenience wrapper 
around the more fundamental `text`, `tokens`, and `selectedTextRange` properties, 
providing the selection behavior the user will expect.

@note
Because this method does not remove any tokens in the provided range, 
the caller can pass the field’s selectedTextRange 
to convert the selected portion of the text 
into a token without first having to trim the range.
```  
  
メソッドの引数で指定している`textualRange`には  
下記のようなソースコメントが記載されています。  
  
```swift

The range that corresponds to the field’s text, exclusive of any tokens.

@see -[<UITextInput> positionWithinRange:atCharacterOffset:]

```  
  
トークン部分を除いたテキストの範囲が設定されているようです。  
  
  
### 動作  
  
このような設定をすると  
下記のような動きが実現できました。  
  
![UISearchBar720.gif](/image/e02c3f61-146d-2ac6-1068-38f4a6ce692c.gif)  
  
## コピー&ペースト  
  
ここからはコピー&ペーストの実装をしていきます。  
  
まず`searchTextField`に下記の設定を追加します。  
  
```swift

searchBar.searchTextField.allowsCopyingTokens = true

```  
  
どういう実装が必要なのかがわからなかったので  
`allowsCopyingTokens `のソースコードを見てみます。  
  
```
Whether the user can copy tokens to the pasteboard or drag them out of the text field.
To support copying tokens, 
this property must be true and the delegate must provide an item provider 
for the tokens to be copied. 
UISearchTextField always enables the Copy command 
if any plain text is selected, 
even if the selection also includes tokens and this property is false. 
Defaults to true.

```  
  
ここからわかることとして  
  
```
the user can copy tokens to the pasteboard
```  
  
`UIPasteboard`を使用する  
  
ということと  
  
  
```
this property must be true 
and the delegate must provide an item provider 
for the tokens to be copied
```  
  
とあるのでdelegateの実装が必要になるようです。  
  
`UIPasteboard`の説明は割愛させていただきます。  
↓をご参照ください。  
https://developer.apple.com/documentation/uikit/uipasteboard  
  
  
### UISearchTextFieldDelegateの実装  
  
まず`UISearchTextFieldDelegate`の実装をしていきます。  
  
```swift

extension ViewController: UISearchTextFieldDelegate {
    func searchTextField(_ searchTextField: UISearchTextField, itemProviderForCopying token: UISearchToken) -> NSItemProvider {
        guard let plan = token.representedObject as? Plan else {
            return NSItemProvider()
        }
        return NSItemProvider(object: PastePlan(plan))
    }
}
```  
  
このメソッドはトークンのCopyをタップした際に呼ばれます。  
  
  
ここでは`UIPasteboard`に設定する`NSItemProvider`を用意しています。  
  
`NSItemProvider`の説明は割愛させていただきます。  
↓をご参照ください。  
https://developer.apple.com/documentation/foundation/nsitemprovider  
  
取得したい情報は  
上記のsetTokenメソッドの中で設定した  
`representedObject`から取得しています。  
  
delegateの設定も忘れないように行います。  
  
```swift

searchBar.searchTextField.delegate = self
```  
  
### UIPasteboardへ値の書き込みと読み込み  
  
次に`UIPasteboard`へ値の書き込みと読み込みをするための実装をしていきます。  
  
そのために`UITextPasteDelegate`を実装します。  
https://developer.apple.com/documentation/uikit/uitextpastedelegate  
  
使用するメソッドは`UITextPasteItem`が取得できる  
  
```swift

textPasteConfigurationSupporting(
    _ textPasteConfigurationSupporting: UITextPasteConfigurationSupporting, 
    transform item: UITextPasteItem)
```  
  
です。  
  
```swift

extension ViewController: UITextPasteDelegate {
    func textPasteConfigurationSupporting(_ textPasteConfigurationSupporting: UITextPasteConfigurationSupporting, transform item: UITextPasteItem) {
        guard let item = item as? UISearchTextFieldPasteItem else {
            return
        }

        item.itemProvider.loadObject(ofClass: PastePlan.self) {
            (pastePlan, error) in
            guard let plan = (pastePlan as? PastePlan)?.plan else {
                return
            }
            let token = UISearchToken(icon: UIImage(systemName: plan.iconName), text: plan.rawValue)
            item.setSearchTokenResult(token)
        }
    }
}
```  
  
`UISearchTextFieldPasteItem`はProtocolで  
`setSearchTokenResult(_ token: UISearchToken)`メソッドがあり  
ここでトークンを`UIPasteboard`に設定することができます。  
  
`UISearchTextFieldPasteItem`  
https://developer.apple.com/documentation/uikit/uisearchtextfieldpasteitem  
  
上記のコードで出てきている`PastePlan`は  
`NSItemProvider`に値を設定するために用意したクラスです。  
(コードは最後に記載しています。)  
内部で`Plan`を保持するようにしています。  
  
またJSONとして値を取得するために  
`Codable`にも適合するようにしています。  
  
delegateは  
`searchTextField`が適合する  
`UITextPasteConfigurationSupporting`の  
`pasteDelegate`への設定が必要になります。  
  
```swift

searchBar.searchTextField.pasteDelegate = self
```  
  
https://developer.apple.com/documentation/uikit/uitextpasteconfigurationsupporting/2887494-pastedelegate  
  
### UIPasteboardからの値の取得  
  
最後に`UIPasteboard`から値を取得します。  
  
```swift

private func extractPlanFromPasteboard() -> Plan? {
    guard let data = UIPasteboard.general.data(forPasteboardType: (kUTTypeJSON as String)) else {
        return nil
    }
    UIPasteboard.general.items = []
    return try? JSONDecoder().decode(PastePlan.self, from: data).plan
}
```  
  
※   
最終的なトークンの設定や表示するデータのフィルターは  
`UISearchResultsUpdating`の  
`updateSearchResults(for searchController: UISearchController)`  
の中で行っています。  
  
### 動作  
  
このような実装をすることで  
下記のような動作が実現できました。  
  
![画面収録-2019-11-24-8.20.58.gif](/image/31032a65-f0e8-141f-f326-423c8ffa69f8.gif)  
  
  
# 今回使用したコード  
  
下記に記載もしていますが  
gistにもアップロードしました。  
  
https://gist.github.com/stzn/4e49e0afe3d4208a824e8eed53f24c61  
  
<details><summary>全体のコード</summary><div>  
  
```swift

import UIKit
import MobileCoreServices

class PastePlan: NSObject, NSItemProviderReading, NSItemProviderWriting, Codable {
    let plan: Plan
    init(_ plan: Plan) {
        self.plan = plan
    }
    static var readableTypeIdentifiersForItemProvider: [String] = [(kUTTypeJSON) as String]
    static var writableTypeIdentifiersForItemProvider: [String] = [(kUTTypeJSON) as String]

    static func object(withItemProviderData data: Data, typeIdentifier: String) throws -> Self {
        try JSONDecoder().decode(self, from: data)
    }

    func loadData(withTypeIdentifier typeIdentifier: String, forItemProviderCompletionHandler completionHandler: @escaping (Data?, Error?) -> Void) -> Progress? {
        let progress = Progress(totalUnitCount: 100)
        do {
            let data = try JSONEncoder().encode(self)
            progress.completedUnitCount = 100
            completionHandler(data, nil)
       } catch {
           completionHandler(nil, error)
       }

       return progress
    }
}

enum Plan: String, CaseIterable, Equatable, Codable {
    case business
    case travel
    case shopping

    init?(_ text: String) {
        if text.starts(with: "bu") {
            self = .business
        } else if text.starts(with: "tr") {
            self = .travel
        } else if text.starts(with: "sh") {
            self = .shopping
        } else {
            return nil
        }
    }

    var iconName: String {
        switch self {
        case .business:
            return "bag"
        case .travel:
            return "airplane"
        case .shopping:
            return "cart"
        }
    }

    var items: [String] {
        switch self {
        case .business:
            return ["会議", "商談", "プレゼン", "勉強会"]
        case .travel:
            return ["宿泊", "日帰り"]
        case .shopping:
            return ["スーパー", "デパート"]
        }
    }
}

class ViewController: UIViewController {
    @IBOutlet weak private var tableView: UITableView!

    private var searchController = UISearchController()

    private var filteredItems: [[String]] = []

    private var searchBar: UISearchBar {
        searchController.searchBar
    }

    private var isSearchBarEmpty: Bool {
        if !searchBar.searchTextField.tokens.isEmpty {
            return false
        }
        return searchBar.text?.isEmpty ?? true
    }

    override func viewDidLoad() {
        super.viewDidLoad()
        setupSearchBar()
        setupTableView()
    }

    private func setupSearchBar() {
        searchController.searchResultsUpdater = self
        searchController.obscuresBackgroundDuringPresentation = false
        searchBar.placeholder = "検索キーワード"
        definesPresentationContext = true

        // UISearchTextField
        searchBar.searchTextField.backgroundColor = .systemOrange
        searchBar.searchTextField.textColor = .systemPurple
        searchBar.searchTextField.font = UIFont(name: "American Typewriter", size: 18)
        searchBar.searchTextField.tokenBackgroundColor = .systemBlue

        // Delete
        searchBar.searchTextField.allowsDeletingTokens = true

        // Copy & Paste
        searchBar.searchTextField.allowsCopyingTokens = true
        searchBar.searchTextField.delegate = self
        searchBar.searchTextField.pasteDelegate = self
    }

    private func setupTableView() {
        tableView.register(UITableViewCell.self, forCellReuseIdentifier: "cell")
        tableView.dataSource = self
        tableView.tableHeaderView = searchController.searchBar
    }
}

extension ViewController: UISearchTextFieldDelegate {
    func searchTextField(_ searchTextField: UISearchTextField, itemProviderForCopying token: UISearchToken) -> NSItemProvider {
        guard let plan = token.representedObject as? Plan else {
            return NSItemProvider()
        }
        return NSItemProvider(object: PastePlan(plan))
    }
}

extension ViewController: UITextPasteDelegate {
    func textPasteConfigurationSupporting(_ textPasteConfigurationSupporting: UITextPasteConfigurationSupporting, transform item: UITextPasteItem) {
        guard let item = item as? UISearchTextFieldPasteItem else {
            return
        }

        item.itemProvider.loadObject(ofClass: PastePlan.self) {
            (pastePlan, error) in
            guard let plan = (pastePlan as? PastePlan)?.plan else {
                return
            }
            let token = UISearchToken(icon: UIImage(systemName: plan.iconName), text: plan.rawValue)
            item.setSearchTokenResult(token)
        }
    }
}

extension ViewController: UITableViewDataSource {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        if isSearchBarEmpty {
            return Plan.allCases[section].items.count
        }
        if !filteredItems.isEmpty {
            return filteredItems[section].count
        }
        return 0
    }

    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
        var item: String? = nil
        if isSearchBarEmpty {
            item = Plan.allCases[indexPath.section].items[indexPath.row]
        } else if !filteredItems.isEmpty {
            item = filteredItems[indexPath.section][indexPath.row]
        }
        cell.textLabel?.text = item
        return cell
    }
}

extension ViewController: UITableViewDelegate {
    func numberOfSections(in tableView: UITableView) -> Int {
        if isSearchBarEmpty {
            return Plan.allCases.count
        }

        if !filteredItems.isEmpty {
            return filteredItems.count
        }

        return 0
    }

    func tableView(_ tableView: UITableView, titleForHeaderInSection section: Int) -> String? {
        Plan.allCases[section].rawValue
    }
}

extension ViewController: UISearchResultsUpdating {
    func updateSearchResults(for searchController: UISearchController) {
        var text = searchBar.text!.lowercased().trimmingCharacters(in: .whitespacesAndNewlines)
        guard !isSearchBarEmpty else {
            return
        }
        
        var searchPlans: [Plan] = extractSearchPlans()
        if let plan = Plan(text) {
            setToken(from: plan)
            searchPlans.append(plan)
            text = ""
        } else if let plan = extractPlanFromPasteboard() {
            searchPlans.append(plan)
            text = ""
        }
        updateUI(plans: searchPlans, text: text)
    }

    private func extractPlanFromPasteboard() -> Plan? {
        guard let data = UIPasteboard.general.data(forPasteboardType: (kUTTypeJSON as String)) else {
            return nil
        }
        UIPasteboard.general.items = []
        return try? JSONDecoder().decode(PastePlan.self, from: data).plan
    }

    private func extractSearchPlans() -> [Plan] {
        searchBar.searchTextField.tokens.compactMap { $0.representedObject as? Plan }
    }

    private func setToken(from plan: Plan) {
        let planToken = UISearchToken(icon: UIImage(systemName: plan.iconName), text: plan.rawValue)
        planToken.representedObject = plan
        let field = searchBar.searchTextField
        field.replaceTextualPortion(of: field.textualRange, with: planToken, at: field.tokens.count)
    }

    private func updateUI(plans: [Plan], text: String) {
        filteredItems = []
        if !plans.isEmpty {
            filteredItems = Plan.allCases.filter { plans.contains($0) }
                .map {
                    $0.items.filter { $0.starts(with: text) }
            }
        } else {
            filteredItems = Plan.allCases.map {
                    $0.items.filter { $0.starts(with: text) } }
        }
        tableView.reloadData()
    }
}
```  
</div></details>  
  
# まとめ  
  
iOS13の`UISearchBar`の新機能について見ていきました。  
  
特に`UISearchToken`に関しては  
検索キーワードを一つのまとまりとして扱ったりすることができ  
使い道がたくさんあるのではないかと思います。  
  
ただ現在情報が見つからないため  
正式なサンプルやドキュメントなどが  
早く出てくると嬉しいですね😃  
  
もし間違いなどございましたら教えていただけますとうれしいです🙇🏻‍♂️  
