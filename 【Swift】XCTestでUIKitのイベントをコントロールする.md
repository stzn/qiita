XCTestを書く時に  
UIKitのイベントが関わってくると  
テストが複雑になることがあります。  
  
これはUIKit側から発生するイベントに応答してテストを書く必要があることが  
大きな要因の一つだと思います。  
  
そのため  
ViewControllerで受け取ったイベントを  
ViewModelなど他に寄せていくことで  
UIKitに触れずに確認できる範囲を広げていく  
などの方法が取られることもあります。  
  
それでも  
画面を操作するテストは必要であり  
XCUITestや手動での確認を行っているのではないかと思います。  
  
しかし  
XCUITestはXCTestに比べて準備や実行に時間がかかったり  
手動で確認する場合には時間に加えて人力も必要になってきます。  
  
今回は  
そういった確認する負担を軽減する方法を探していくために  
XCTest内でUIKitから発生するイベントを  
コントロールする方法を見ていきたいと思います。  
  
# 今回のサンプル  
  
サンプルとしては  
シンプルにTableViewにリモートから取得したデータの一覧を表示する  
だけです。  
  
# 確認したいことは  
  
以下のことを確認します。  
  
- 画面の表示時にデータを取得して正しく表示されているか(viewDidLoad)  
- データの取得中はローディングが表示され、読み込みが完了したら(成功失敗関わらず)非表示になるか  
- ユーザがリロード操作をした時にデータの読み込みが行われるか(PullToRefresh)  
- セルが表示される時に画像のURLからデータの取得を開始し、正しく表示されるか(cellForRowAt)  
- セルが非表示になった先に画像データの取得がキャンセルされるか(didEndDisplaying)  
- 画像データの取得中はセル内にローディングが表示され、読み込みが完了したら(成功失敗関わらず)非表示になるか  
- セルが表示される直前に画像データの取得をし始めるか(プリフェッチ)  
- セルが非表示になる直前に画像データの取得がキャンセルされるか(プリフェッチのキャンセル)  
  
# 事前準備  
  
今回の本題とは関係ありませんが  
サンプル内で使用しているものについて下記に記載します。  
  
## データ取得  
  
まずは取ってくるデータは下記のようになります。  
  
```swift

struct Item {
    let id: UUID
    let name: String
    let imageUrl: URL
}
```  
  
ネットワーク通信は下記のProtocolに対してモックを使用しています。  
どこかからデータを取得してくるだけです。  
  
```swift

protocol ItemLoader {
    func load(completion: @escaping (Result<[Item], Error>) -> Void)
}
```  
  
セル内に表示する画像を取得するProtocolです。  
  
```swift

protocol ItemImageLoaderTask {
    func cancel()
}

protocol ItemImageLoader {
    func load(url: URL, completion: @escaping (Result<Data, Error>) -> Void) -> ItemImageLoaderTask
}
```  
  
上記のProtocolに適合したSpyを定義します。  
メソッドが呼び出された際のリクエスト(Resultを受け取るコールバック)を保存しておき  
`complete...`メソッドを呼び出すことで  
そのコールバックを呼び出します。  
  
こうすることでリモート通信で戻ってくるデータの取得タイミングをコントロールします。  
  
```swift

private class LoaderSpy: ItemLoader, ItemImageLoader {

    // ItemLoader

    private var itemRequests: [(Result<[Item], Error>) -> Void] = []
    
    var loadCallCount: Int {
        return itemRequests.count
    }
    
    func load(completion: @escaping (Result<[Item], Error>) -> Void) {
        itemRequests.append(completion)
    }
    
    func completeItemLoading(with items: [Item] = [], at index: Int = 0) {
        itemRequests[index](.success(items))
    }
    
    func completeItemLoadingWithError(_ error: Error = NSError(domain: "error", code: 0, userInfo: nil), at index: Int = 0) {
        itemRequests[index](.failure(error))
    }
    
    // ItemImageLoader
    
    private var imageRequests:[(url: URL, completion: (Result<Data, Error>) -> Void)] = []
    
    var requestedImageURLs: [URL] {
        return imageRequests.map { $0.url }
    }
    private(set) var canceledImageURLs: [URL] = []
    
    func load(url: URL, completion: @escaping (Result<Data, Error>) -> Void) ->
        ItemImageLoaderTask {
            imageRequests.append((url, completion))
            return TaskSpy { [weak self] in
                self?.canceledImageURLs.append(url)
            }
    }
    
    func completeImageLoading(with data: Data, at index: Int) {
        imageRequests[index].completion(.success(data))
    }

    func completeImageLoadingWithError(_ error: Error = NSError(domain: "error", code: 0, userInfo: nil), at index: Int = 0) {
        imageRequests[index].completion(.failure(error))
    }

    private struct TaskSpy: ItemImageLoaderTask {
        let cancelCallback: () -> Void
        func cancel() {
            cancelCallback()
        }
    }
}
```  
  
## 画面表示用のクラス  
  
以下は必要なUIを表示するクラスになります。  
  
※   
実際に画面への表示は想定しておらず  
テストを実行するための必要最低限の設定しかしていません。  
  
## ItemListViewController  
  
Itemの一覧を表示するUITableViewControllerのサブクラスです。  
  
```swift

final class ItemListViewController: UITableViewController, UITableViewDataSourcePrefetching {
    var cellControllers: [ItemCellController] = [] {
        didSet {
            tableView.reloadData()
        }
    }    
    private var refreshController: ItemRefreshController?

    convenience init(refreshController: ItemRefreshController) {
        self.init()
        self.refreshController = refreshController
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
        tableView.prefetchDataSource = self
        refreshControl = refreshController?.refreshControl
        refreshController?.refresh()
    }
    
    override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        return cellController(forRowAt: indexPath).view()
    }
    
    override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return cellControllers.count
    }
    
    override func numberOfSections(in tableView: UITableView) -> Int {
        return 1
    }
    
    override func tableView(_ tableView: UITableView, didEndDisplaying cell: UITableViewCell, forRowAt indexPath: IndexPath) {
        cancelImageLoad(forRowAt: indexPath)
    }
    
    func tableView(_ tableView: UITableView, prefetchRowsAt indexPaths: [IndexPath]) {
        indexPaths.forEach {
            cellController(forRowAt: $0).preloadImage()
        }
    }
    
    func tableView(_ tableView: UITableView, cancelPrefetchingForRowsAt indexPaths: [IndexPath]) {
        indexPaths.forEach(cancelImageLoad)
    }
    
    private func cellController(forRowAt indexPath: IndexPath) -> ItemCellController {
        return cellControllers[indexPath.row]
    }

    private func cancelImageLoad(forRowAt indexPath: IndexPath) {
        cellControllers[indexPath.row].cancelLoadImage()
    }
}

```  
  
## ItemRefreshController  
  
`UIRefreshControl`の表示と  
Refresh時のデータの再取得などを管理するControllerです。  
  
```swift

final class ItemRefreshController: NSObject {
    lazy var refreshControl: UIRefreshControl = {
        let refreshControl = UIRefreshControl()
        refreshControl.addTarget(self, action: #selector(refresh), for: .valueChanged)
        return refreshControl
    }(）
    var onRefresh: (([Item]) -> Void)?

    private let itemLoader: ItemLoader    

    init(itemLoader: ItemLoader) {
        self.itemLoader = itemLoader
    }

    @objc func refresh() {
        load()
    }
    
    private func load() {
        refreshControl.beginRefreshing()
        itemLoader.load { [weak self] result in
            guard let self = self else {
                return
            }
            if let items = try? result.get() {
                self.onRefresh?(items)
            }
            self.refreshControl.endRefreshing()
        }
    }
}
```  
  
## ItemCellController  
  
  
ItemCellに表示する画像データの取得や  
ItemCellの生成を管理するControllerです。  
  
```swift

final class ItemCellController {
    private var task: ItemImageLoaderTask?
    private let imageLoader: ItemImageLoader
    private let item: Item
    
    init(item: Item, imageLoader: ItemImageLoader) {
        self.item = item
        self.imageLoader = imageLoader
    }
    
    func view(in tableView: UITableView) -> ItemCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: ItemCell.identifier) as! ItemCell
        cell.configure(item: item)
        cell.imageLoadingIndicator.startAnimating()
        self.task = imageLoader.load(url: item.imageUrl) { [weak cell] result in
            let data = try? result.get()
            let image = data.map(UIImage.init) ?? nil
            cell?.thumbnailImageView.image = image
            cell?.imageLoadingIndicator.stopAnimating()
        }
        return cell
    }
    
    func preloadImage() {
        task = imageLoader.load(url: item.imageUrl) { _ in }
    }
    
    func cancelLoadImage() {
        task?.cancel()
    }
}
```  
  
## ItemCell  
  
画面に表示するCellです。  
今回は実際に表示しないので必要な部品を適当に初期化しています。  
  
```swift

final class ItemCell: UITableViewCell {
    static let identifier = "\(ItemCell.self)"
    
    let nameLabel = UILabel()
    let thumbnailImageView = UIImageView()
    let imageLoadingIndicator = UIActivityIndicatorView()
    
    func configure(item: Item) {
        thumbnailImageView.image = nil
        nameLabel.text = item.name
    }
}
```  
  
  
# UIKitの動きをシミュレートする  
  
ここからはUIKitの動きをシミュレートして  
テストの中でコントロールできるようにします。  
  
主に下記の2つをシミュレートします。  
  
- TableViewのイベント  
- ユーザ操作によるUIControlのイベント  
  
## TableViewのイベントをシミュレートする  
  
今回は一覧を表示するために`UITableView`を使用しています。  
TableViewのイベントをシミュレートするためには  
UITableViewが呼び出すメソッドを  
こちらで呼び出すようにします。  
  
※下記はテストクラスに追加しています。  
  
```swift

private extension ItemListViewController {
    // cellForRowAt
    func itemCell(at index: Int) -> UITableViewCell? {
        let dataSource = tableView.dataSource
        let indexPath = IndexPath(row: index, section: 0)
        return dataSource?.tableView(tableView, cellForRowAt: indexPath)
    }

    // itemCellを呼び出すことで表示させている
    @discardableResult
    func simulateItemCellVisible(at index: Int) -> ItemCell? {
        return itemCell(at: index) as? ItemCell
    }
    
    // numberOfRowsInSection
    func numberOfRenderedItemCells() -> Int {
        let dataSource = tableView.dataSource
        return dataSource?.tableView(tableView, numberOfRowsInSection: 0) ?? 0
    }
        
    // didEndDisplaying
    func simulateItemCellNotVisible(at index: Int) {
        // 非表示を表現するためにまず表示させている
        let view = simulateItemCellVisible(at: index)

        let delegate = tableView.delegate
        let indexPath = IndexPath(row: index, section: itemSection)
        delegate?.tableView?(tableView, didEndDisplaying: view!, forRowAt: indexPath)
    }
    
    // prefetchRowsAt
    func simulateItemCellNearVisible(at index: Int) {
        let prefetch = tableView.prefetchDataSource
        let indexPath = IndexPath(row: index, section: 0)
        prefetch?.tableView(tableView, prefetchRowsAt: [indexPath])
    }

    // cancelPrefetchingForRowsAt
    func simulateItemCellNotNearVisible(at index: Int) {        
        // 非表示を表現するためにまず表示させている
        simulateItemCellNearVisible(at: index)
        
        let prefetch = tableView.prefetchDataSource
        let indexPath = IndexPath(row: index, section: 0)
        prefetch?.tableView?(tableView, cancelPrefetchingForRowsAt: [indexPath])
    }
}

```  
  
## ユーザ操作によるUIControlのイベントをシミュレートをする  
  
次にユーザの操作をシミュレートをするために  
UIControlを拡張します。  
  
### UIControlの拡張  
  
```swift

extension UIControl {
    func simulate(event: UIControl.Event) {
        allTargets.forEach { target in
            actions(forTarget: target, forControlEvent: event)?.forEach {
                (target as NSObject).perform(Selector($0))
            }
        }
    }
}
```  
  
`allTargets`は対象のUIControlに繋がっている`NSObject`の配列を返します。  
`addTarget`したものだと思っていただければ大丈夫です。  
https://developer.apple.com/documentation/uikit/uicontrol/1618207-alltargets  
  
`actions`はtargetの特定の`UIControl.Event`に対する`action`の配列を返します。  
`addTarget`時に`#selector(〇〇)`で指定した〇〇が文字列で返ってくると思ってください。  
https://developer.apple.com/documentation/uikit/uicontrol/1618251-actions  
  
`perform`で対象のメソッドを実行します。  
https://developer.apple.com/documentation/objectivec/nsobjectprotocol/1418867-perform  
  
ただしこれは通常の実装では使用しないように警告されていますので  
テスト目的以外では使用しない方が良いようです。  
  
```
Important

Because of the inherent lack of type safety, 
this API is not recommended for use in Swift 
unless your code specifically relies on the dynamic method resolution 
provided by the Objective-C run-time.
```  
  
これが定義できれば  
あとは個々のUIKitのクラスを拡張して  
動作をシミュレートします。  
  
今回使用する例としては`UIRefreshControl`の  
画面を下に引っ張るとリフレッシュするような動作をシミュレートします。  
上記の動作を行うと`UIControl.Event.valueChanged`が発生するので  
そのイベントを発生させます。  
  
### UIRefreshControlの拡張   
  
```swift

extension UIRefreshControl {
    func simulatePullToRefresh() {
        simulate(event: .valueChanged)
    }
}
```  
  
今回はこちらを下記のように活用しています。  
  
  
```swift

private extension ItemListViewController {
    func simulateUserReload() {
        refreshControl?.simulatePullToRefresh()
    }
}
```  
  
  
### UIButtonの拡張  
  
他にもよく使うものとして  
ボタンを押した操作などは下記のようになります。  
  
  
```swift

extension UIButton {
    func simulateTap() {
        simulate(event: .touchUpInside)
    }
}
```  
  
# テストを書いていく  
  
それではテストを書いていきます。  
  
## (補足)ヘルパーメソッド  
  
テストの対象やデータを作成するコードを  
毎回書くのは大変なので  
下記のヘルパーメソッドを使用しています。  
  
※  
ItemListComposeは  
ViewControllerの生成とLoaderのDIを行っています。  
  
```swift

private func makeTestTarget() -> (ItemListViewController, LoaderSpy) {
    let loader = LoaderSpy()
    let target = ItemListComposer.compose(itemLoader: loader, imageLoader: loader)
    return (target, loader)
}

private func makeItem(name: String = "", imageUrl: URL = URL(string: "https://any.com")!) -> Item {
    return Item(id: UUID(), name: name, imageUrl: imageUrl)
}

```  
  
他にも画面の要素にアクセスしやすくしたり  
可読性を高めるために  
下記のようなプロパティを用意しています。  
  
```swift

private extension ItemListViewController {
    var isShowIndicator: Bool {
        return refreshControl?.isRefreshing ?? false
    }
}


private extension ItemCell {
    var name: String? {
        return nameLabel.text
    }
    
    var isShowingImageLoadingIndicator: Bool {
        return imageLoadingIndicator.isAnimating
    }
    
    var renderedImageData: Data? {
        return thumbnailImageView.image?.pngData()
    }
}
```  
  
  
## データをロード時のアクション  
  
```swift

func test_データをロード時のアクション_viewDidloadとユーザがリロードした時にアイテムを読み込むこと() {
    let (target, loader) = makeTestTarget()

    XCTAssertEqual(loader.loadCallCount, 1, "画面表示後にアイテムが１回読み込まれる")
    
    // ユーザがリロード操作を行った
    target.simulateUserReload()
    XCTAssertEqual(loader.loadCallCount, 2, "ユーザーがリロードした際にアイテムが1回読み込まれる")

    // ユーザがリロード操作を行った
    target.simulateUserReload()
    XCTAssertEqual(loader.loadCallCount, 3, "ユーザーが再度リロードした際にアイテムが1回読み込まれる")
}

func test_データをロード時のアクション_ロード中はIndicatorが表示され_ロード後は非表示になること() {
    let (target, loader) = makeTestTarget()
    
    XCTAssertTrue(target.isShowIndicator, "画面初期表示時のアイテム読み込み中で表示されている")

    // データの読み込みが完了した
    loader.completeItemLoading(at: 0)
    XCTAssertFalse(target.isShowIndicator, " アイテムの読み込みが完了したので表示はされていない")
    
    // ユーザがリロード操作を行った
    target.simulateUserReload()
    XCTAssertTrue(target.isShowIndicator, "リロード時のアイテム読み込み中で表示されている")

    // データの読み込みが完了した
    loader.completeItemLoadingWithError(at: 1)
    XCTAssertFalse(target.isShowIndicator, " エラーでもアイテムの読み込みが完了したので表示はされていない")
}

```  
  
## データ取得  
  
次にデータが正しく取得されているかを確認します。  
  
※  
下記で使用されているassertThatは  
正しい値が設定されているかどうかをチェックするためのメソッドです。  
中身は割愛します。  
  
```swift

func test_データ取得_正しく取得できていること() {
    let (target, loader) = makeTestTarget()

    let item1 = makeItem()
    let item2 = makeItem(name: "name1")
    let item3 = makeItem(name: "name2", imageUrl: URL(string: "https://item3-url")!)

    // まだデータの書き込みが終わっていないので何も保持していない
    assertThat(target: target, isRendering: [])

    // データを1件取得完了
    loader.completeItemLoading(with: [item1], at: 0)
    
    // 1件データを保持しているはず
    assertThat(target: target, isRendering: [item1])

    // ユーザがリロード操作を行った
    target.simulateUserReload()

    // データを2件取得完了
    loader.completeItemLoading(with: [item2, item3], at: 1)

    // 2件データを保持しているはず
    assertThat(target: target, isRendering: [item2, item3])
}

func test_データ取得_エラーが発生しても取得済みのデータを保持していること() {
    let (target, loader) = makeTestTarget()

    let item1 = makeItem()

    assertThat(target: target, isRendering: [])

    loader.completeItemLoading(with: [item1], at: 0)
    
    // 1件データが保持しているはず
    assertThat(target: target, isRendering: [item1])

    target.simulateUserReload()

    // エラーが発生した
    loader.completeItemLoadingWithError(at: 1)

    // 1件のデータは保持したままのはず
    assertThat(target: target, isRendering: [item1])
}

```  
  
## セル内の画像の読み込み  
  
```swift

func test_画像の読み込み_セルが表示されている時に画像がURLから読み込みが開始されていること() {
    let (target, loader) = makeTestTarget()

    let item1 = makeItem(name: "item1", imageUrl: URL(string: "https://item-1.com")!)
    let item2 = makeItem(name: "item2", imageUrl: URL(string: "https://item-2.com")!)

    // データを2件取得完了
    loader.completeItemLoading(with: [item1, item2], at: 0)
    XCTAssertEqual(loader.requestedImageURLs, [], "cellForRowAtはまだ呼ばれず画像の読み込みは行わない")

    // 1行目のセルが表示された
    target.simulateItemCellVisible(at: 0)

    XCTAssertEqual(loader.requestedImageURLs, [item1.imageUrl], "1行目のセルのcellForRowAtが呼ばれて画像の読み込みが行われる")

    // 2行目のセルが表示された
    target.simulateItemCellVisible(at: 1)
    
    XCTAssertEqual(loader.requestedImageURLs, [item1.imageUrl, item2.imageUrl], "2行目のセルのcellForRowAtが呼ばれて画像の読み込みが行われる")

}

func test_画像の読み込み_セルが非表示になった時_読み込みがキャンセルされること() {
    let (target, loader) = makeTestTarget()

    let item1 = makeItem(name: "item1", imageUrl: URL(string: "https://item-1.com")!)
    let item2 = makeItem(name: "item2", imageUrl: URL(string: "https://item-2.com")!)

    loader.completeItemLoading(with: [item1, item2], at: 0)
    XCTAssertEqual(loader.requestedImageURLs, [], "cellForRowAtはまだ呼ばれず画像の読み込みは行わない")
    XCTAssertEqual(loader.canceledImageURLs, [], "didEndDisplayingはまだ呼ばれず画像の読み込みは行わない")

    // 1行目のセルが表示された後に非表示になった
    target.simulateItemCellNotVisible(at: 0)

    XCTAssertEqual(loader.canceledImageURLs, [item1.imageUrl], "1行目のセルのdidEndDisplayingが呼ばれて画像の読み込みがキャンセルされる")

    // 2行目のセルが表示された後に非表示になった
    target.simulateItemCellNotVisible(at: 1)

    XCTAssertEqual(loader.canceledImageURLs, [item1.imageUrl, item2.imageUrl], "2行目のセルのdidEndDisplayingが呼ばれて画像の読み込みがキャンセルされる")
}
```  
## 画像の読み込み(prefetch)  
  
```swift

func test_preload_セルが表示される直前に画像の読み込みが開始されること() {
    let (target, loader) = makeTestTarget()

    let item1 = makeItem(name: "item1", imageUrl: URL(string: "https://item-1.com")!)
    let item2 = makeItem(name: "item2", imageUrl: URL(string: "https://item-2.com")!)

    // データを2件取得完了
    loader.completeItemLoading(with: [item1, item2], at: 0)

    XCTAssertEqual(loader.requestedImageURLs, [], "prefetchRowsAtはまだ呼ばれず画像の読み込みは行われない")

    // 1行目のセルが表示される直前になった
    target.simulateItemCellNearVisible(at: 0)

    XCTAssertEqual(loader.requestedImageURLs, [item1.imageUrl], "1行目のセルのprefetchRowsAtが呼ばれ画像の読み込みが開始される")


    target.simulateItemCellNearVisible(at: 1)
    XCTAssertEqual(loader.requestedImageURLs, [item1.imageUrl, item2.imageUrl], "2行目のセルのprefetchRowsAtが呼ばれ画像の読み込みが開始される")
}

func test_preload_セルが非表示になる直前に画像の読み込みがキャンセルされること() {
    let (target, loader) = makeTestTarget()

    let item1 = makeItem(name: "item1", imageUrl: URL(string: "https://item-1.com")!)
    let item2 = makeItem(name: "item2", imageUrl: URL(string: "https://item-2.com")!)

    loader.completeItemLoading(with: [item1, item2], at: 0)
    XCTAssertEqual(loader.requestedImageURLs, [], "prefetchRowsAtは呼ばれず画像の読み込みは行わない")
    XCTAssertEqual(loader.canceledImageURLs, [], "cancelPrefetchingForRowsAtは呼ばれず画像の読み込みキャンセルは行わない")

    target.simulateItemCellNotNearVisible(at: 0)
    XCTAssertEqual(loader.canceledImageURLs, [item1.imageUrl], "1行目のセルのcancelPrefetchingForRowsAtは呼ばれ画像の読み込みがキャンセルされる")

    target.simulateItemCellNotNearVisible(at: 1)
    XCTAssertEqual(loader.canceledImageURLs, [item1.imageUrl, item2.imageUrl], "2行目のセルのcancelPrefetchingForRowsAtは呼ばれ画像の読み込みがキャンセルされる")
}

```  
  
## 画像のローディングの表示  
  
次に画像読み込み時のローディングの表示の確認をします。  
  
※  
`isShowingImageLoadingIndicator`は  
セルのUIActivityIndicatorViewのisAnimatingプロパティにアクセスします。  
  
  
```swift

func test_画像のローディングの表示_画像読み込み中は表示され_読み込み後は非表示になること() {
    let (target, loader) = makeTestTarget()

    let item1 = makeItem(name: "item1", imageUrl: URL(string: "https://item-1.com")!)
    let item2 = makeItem(name: "item2", imageUrl: URL(string: "https://item-2.com")!)

    // データを2件取得完了
    loader.completeItemLoading(with: [item1, item2], at: 0)

    // セルが表示された
    let cell0 = target.simulateItemCellVisible(at: 0)
    let cell1 = target.simulateItemCellVisible(at: 1)

    // セルが表示されたので画像の読み込みが始まり、ローディングが表示されているはず
    XCTAssertEqual(cell0?.isShowingImageLoadingIndicator, true)
    XCTAssertEqual(cell1?.isShowingImageLoadingIndicator, true)
    
    // 1行目のセルの画像の読み込みが完了した
    loader.completeImageLoading(with: Data(), at: 0)

    // 1行目のセルが表示されたので画像の読み込みが完了し、ローディングは非表示になっているはず
    XCTAssertEqual(cell0?.isShowingImageLoadingIndicator, false)

    // 2行目のセルが表示されたので画像の読み込みが完了し、ローディングは非表示になっているはず
    XCTAssertEqual(cell1?.isShowingImageLoadingIndicator, true)
    
    // 2行目のセルの画像の読み込みが完了した
    loader.completeImageLoadingWithError(at: 1)

    // 1行目のセルのローディングは非表示のままのはず
    XCTAssertEqual(cell0?.isShowingImageLoadingIndicator, false)

    // 2行目のセルが表示されたので画像の読み込みが完了し、ローディングは非表示になっているはず
    XCTAssertEqual(cell1?.isShowingImageLoadingIndicator, false)
}

```  
  
## 画像の表示  
  
次に表示されている画像が正しいかを確認します。  
  
※  
`renderedImageData`は  
セルのUIImageViewのimageプロパティにアクセスします。  
  
`UIImage.make`は  
ダミーの小さい`UIImage`を作成するためのメソッドです。  
  
assetsにダミー画像を用意する方法もありますが  
こちらの方がファイルにアクセスする時間の削減やエラーのリスクの回避ができるので  
良いのかなと個人的には思っています。  
  
```swift

func test_画像の表示_画像読み込み後_画像が表示されていること() {
    let (target, loader) = makeTestTarget()

    let item1 = makeItem(name: "item1", imageUrl: URL(string: "https://item-1.com")!)
    let item2 = makeItem(name: "item2", imageUrl: URL(string: "https://item-2.com")!)

    // データを2件取得完了
    loader.completeItemLoading(with: [item1, item2], at: 0)

    // 2行のセルを表示
    let cell0 = target.simulateItemCellVisible(at: 0)
    let cell1 = target.simulateItemCellVisible(at: 1)

    // 画像データの取得が完了していないので画像は表示されていない
    XCTAssertEqual(cell0?.renderedImageData, nil)
    XCTAssertEqual(cell1?.renderedImageData, nil)

    let imageData0 = UIImage.make(withColor: .red).pngData()!

    // 1行目のセルの画像データの取得が完了
    loader.completeImageLoading(with: imageData0, at: 0)

    // 1行目のセルの画像が表示されているはず
    XCTAssertEqual(cell0?.renderedImageData, imageData0)

    let imageData1 = UIImage.make(withColor: .blue).pngData()!

    // 2行目のセルの画像データの取得が完了
    loader.completeImageLoading(with: imageData1, at: 1)
    
    // 2行目のセルの画像が表示されているはず
    XCTAssertEqual(cell1?.renderedImageData, imageData1)
}

```  
  
# まとめ  
  
今回はTableViewとUIControlをシミュレートした例を見ていきました。  
TableViewの例はCollectionViewにも応用できますし  
UIControlの拡張を使用することでどんなイベントも発生させることができます。  
  
またこの方法では  
モックではなく本物のUIKitを使用しているため  
実際にアプリを操作した時に似たような動作を再現することができます。  
  
アプリの動作確認は必ずどこかで必要になるとは思いますが  
普段の操作ではたどらないようなケースや  
再現が難しいケースなどの確認にも  
役に立つかもしれませんね。  
  
  
何か間違いやご指摘があれば教えていただけるとうれしいです🙇🏻‍♂️  
