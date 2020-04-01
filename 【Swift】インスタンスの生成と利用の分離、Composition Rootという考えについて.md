# はじめに  
文章の中で「クラス」という言葉を用いていますが  
これはSwiftの`class`だけを指しているのではなく  
`struct`や`enum`なども全て含めてクラスと呼称しています。  
  
---  
  
アプリを開発をしていると  
  
- 必要な機能をクラスに分けて開発する  
- さらにモジュールに分けて開発する  
- プロジェクトを超えて共通で利用できるモジュールを利用する  
  
など  
を行うことがあるかと思います。  
  
そうすると  
あるクラスの中で他のクラスを利用することがあるかと思いますが  
どこで利用するクラスを生成するかが  
問題になることがあります。  
  
例えば  
Aクラスの中で利用するBクラスのインスタンスを生成しているとします。  
  
仮にBクラスのインスタンス生成時に  
新しくCクラスを引数に渡す必要が出てきた場合  
本来Aクラスでは必要のないCクラスのインスタンスの生成も行うことになります。  
  
もしこの新しく追加したCクラスのインスタンス生成方法に変更があった場合  
Aクラスには全く必要のないCクラスに依存していることによって  
Aクラスが変更しなければならないという事態に陥ってしまいます。  
  
  
この問題を解決する方法として  
クラスの外部で必要なインスタンスの生成を行い  
それを渡してあげるようにする  
**DI(依存性注入)**を行う方法があります。  
  
DIにはいくつかの種類が存在し  
  
- インスタンス生成時に引数として渡す(コンストラクタインジェクション)  
- プロパティを用意してインスタンス生成後に変数に値を設定する(プロパティインジェクション)  
- メソッドの引数として渡す(メソッドインジェクション)  
  
があります。  
  
この中でも  
コンストラクタインジェクションを利用すると  
クラス同士の依存関係を明確にできたり  
値の設定忘れを防ぐことができるため  
より好ましいと言われています。  
  
DIを行うことで  
クラス同士が密接に結合することを防ぎ(疎結合)  
一つのクラスの変更の影響範囲を抑えることができますが  
DIを行う上で一つ気をつけなければいけないことがあります。  
  
それはDIをどこで行うかということです。  
  
例えば  
色々な場所でDIを行っていると  
仮にインスタンスの生成方法に変更が入った際  
複数ファイルに飛び散ったインスタンスの生成箇所を探す必要が出てくるかもしれません。  
  
コンストラクタインジェクションの場合は  
コンパイラが教えてくれるので良いかもしれませんが  
プロパティインジェクションの場合は  
どこかで設定漏れがあっても  
気がつくタイミングが遅れてしまう可能性も考えられます。  
  
こういったリスクを軽減する方法として  
  
**Composition Root**  
  
という考えを活用する方法があります。  
  
今回は  
  
- インスタンスの生成と利用の分離  
- Composition Root  
  
について見ていきたいと思います。  
  
# インスタンスの生成と利用を分離する  
  
  
例えば  
QiitaのAPIから投稿一覧を取得するアプリを考えてみます。  
  
![qiita.png](/image/7ca3333b-c6c3-8a97-66b5-014a404b7812.png)  
  
アプリは  
NetworkとUIモジュールに分かれているとします。  
  
そして  
各クラスの役割は  
  
---  
  
#### QiitaLoader   
一覧に表示する情報を外部ネットワークから取得するクラス  
  
#### QiitaUserImageLoader   
一覧情報に含まれたユーザ画像URLから画像データを外部ネットワークから取得するクラス  
  
#### QiitaListViewModel   
一覧に表示するデータや画面の状態を管理するクラス  
QiitaLoaderで取得したデータをQiitaListViewControllerに通知する  
(※ 今回はクロージャを経由して渡します。)  
  
#### QiitaCellViewModel   
一覧の各Cellに表示するデータや画面の状態を管理するクラス  
QiitaUserImageLoaderで取得したデータをQiitaCellViewControllerに通知する  
  
#### QiitaListViewController   
QiitaListViewModelとUITableViewを繋げるUIViewController  
QiitaCellViewControllerの配列を管理する  
  
#### QiitaCellViewController   
QiitaCellとQiitaCellViewModelを繋げるUIViewController  
  
#### QiitaCell   
各Qiitaの情報を表示するUITableViewCellのサブクラス  
  
---  
  
`QiitaCellViewController`は一覧情報を取得した後に  
インスタンスを生成する必要があります。  
そして  
個々のQiita情報に対するユーザ画像を取得するために  
一つ一つの`QiitaCellViewController`に  
`QiitaUserImageLoader`のインスタンスを渡す必要があります。  
  
そのため  
`QiitaListViewController`では  
  
- `QiitaCellViewController`の配列  
- `QiitaLoader`  
- `QiitaUserImageLoader`  
  
への参照を保持する必要があります。  
  
この時の  
各インスタンスの生成場所  
について考えてみます。  
  
## QiitaListViewControllerの内部で生成する  
  
利用されるクラスの生成を  
利用するクラスで生成する場合を考えます。  
  
この場合  
デメリットがいくつか想定されます。  
  
- 全く関係のない変更の影響を受ける可能性がある  
- テストがしづらい場合がある  
  
### 全く関係のない変更の影響を受ける  
  
例えば  
それぞれのインスタンス生成時に  
新しく引数が渡す必要が出てきた場合  
`QiitaListViewController`でも引数を渡す変更を入れる必要があります。  
  
  
```swift

final class QiitaLoader {
    let newParameter: String
    init(newParameter: String) {
        self.newParameter = newParameter
    }
}

final class QiitaListViewModel {
    let loader: QiitaLoader
    init(loader: QiitaLoader) {
        self.loader = loader
    }
}

final class QiitaListViewController: UIViewController {
    // こっちでも変更が必要になる
    let viewModel = QiitaListViewModel(loader: QiitaLoader(newParameter: "new parameter"))
}
```  
  
`QiitaListViewController`では  
`QiitaListViewModel`のインスタンス生成時に  
`QiitaLoader`を渡しているだけですが  
変更の影響を受けてしまいます。  
  
上記はとてもシンプルな例ですが  
`QiitaLoader`の初期化処理が複雑になってくると  
直接関係のないコードも増え  
見通しもどんどん悪くなっていきます。  
  
  
### テストがしづらい場合がある  
  
例えば  
`QiitaLoader`で  
`URLSession.shared`を使ってネットワーク通信をしているとします。  
  
```swift

final class QiitaLoader {
    func load(from url: URL, completion: @escaping ([Qiita]) -> Void) {
        URLSession.shared.dataTask(url: url) {
            ...
        }
    }
}
```  
  
そうすると  
`ViewModel`や`ViewController`のテストをする際にも  
実際にネットワーク通信をする必要が出てきます。  
  
これはテストの実行速度を落とし  
もし接続先のサーバがメンテナンスなどで止まっていたら  
テストが失敗するという可能性もあります。  
  
例えば  
チームで開発をしていて  
masterブランチへのマージには  
テストに通過する必要があるなどの場合ですと  
サーバが利用可能になるまで待たなければならなかったり  
CIを再実行しなければならなくなります。  
  
※   
`URLProtocol`のサブクラスを作成して直接接続させない方法もありますが  
これも結構手間がかかります。  
  
また  
CIツールに障害が発生したというケースは今回考えていません。  
  
## DIでオブジェクトの作成と利用を分離する  
  
このような問題に対応する方法として  
まずDIによってオブジェクトの生成を分離します。  
  
これは外部から生成したオブジェクトを渡すようにします。  
  
下記の例では`UIViewController`なので  
Storyboardを利用していることを想定して  
プロパティインジェクションを利用しています。  
  
※ iOS13で登場した引数を渡せるコンストラクタについては  
今回は割愛させていただきます。  
  
https://developer.apple.com/documentation/uikit/uistoryboard/3213988-instantiateinitialviewcontroller  
  
  
```swift

final class QiitaListViewController: UIViewController {
    var viewModel: QiitaListViewModel!

}

final class SceneDelegate: UIResponder, UIWindowSceneDelegate {

    func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options connectionOptions: UIScene.ConnectionOptions) {
        ...

        let qiitaLoader = QiitaLoader(newParameter: "new parameter")
        let viewModel = QiitaListViewModel(loader: qiitaLoader)
        qiitaListViewController.viewModel = viewModel

        window?.rootViewController = qiitaListViewController
    }
}
```  
  
これで  
全く関係のない`QiitaLoader`変更の影響を受ける  
という問題は解消できます。  
  
## 抽象化でテストしやすくする  
  
さらにもう一つのテストのしづらさの問題も考えてみます。  
上記でも少し触れましたが  
問題なのは直接ネットワークに接続しにいくことで  
テストの速度や成否に影響を与えてしまうことでした。  
  
そこでDIとして引数に渡すものを  
抽象的なものにすることで  
テスト時にモックなどへ入れ替えられるようにします。  
  
```swift

protocol QiitaLoaderProtocol {
    func load(from url: URL, completion: @escaping ([Qiita]) -> Void)
}

final class QiitaListViewModel {
    let loader: QiitaLoaderProtocol
    init(loader: QiitaLoaderProtocol) {
        self.loader = loader
    }
}
```  
  
```swift

final class QiitaLoaderStub: QiitaLoaderProtocol {
    func load(from url: URL, completion: @escaping ([Qiita]) -> Void) {
        completion([Qiita(title: "キータ!")])
    }
}

final class QiitaListViewModelTests: XCTestCase {
    func testLoadQiitaList() {
        let sut = QiitaListViewModel(loader: QiitaLoaderStub())
        ...
    }
}
```  
  
こうすることで直接外部のネットワークで接続する必要がなくなり  
テストの実行速度を高め  
外部の事情でテストが失敗するということも少なくなります。  
  
抽象化を行うということは  
テストに限らず  
今後変更が起こりうる可能性が箇所に導入することで  
変更の影響範囲を抑えることができ  
変更に強い設計にすることにも繋がります。  
  
# Composition Root  
  
上記でいくつかの問題が解決されましたが  
まだ似たような問題があります。  
  
`QiitaCellViewController`のインスタンスの生成に関してです。  
現在の設計ですと  
`QiitaCellViewController`は`QiitaListViewController`の内部で行われ  
`QiitaUserImageLoader`のインスタンスを渡すために  
参照を保持しています。  
  
```swift

final class QiitaListViewModel {
    let loader: QiitaLoaderProtocol
    init(loader: QiitaLoaderProtocol) {
        self.loader = loader
    }
    
    var onLoad: (([Qiita]) -> Void)?
    func load(from url: URL) {
        loader.load(from: url) { list in
            onLoad?(list)
        }
    }
}


final class QiitaCellViewModel {
    let qiita: Qiita 
    let loader: QiitaUserImageLoaderProtocol
    init(qiita: Qiita, loader: QiitaUserImageLoaderProtocol) {
        self.qiita = qiita
        self.loader = loader
    }
    
    var onLoad: (([Qiita]) -> Void)?
    func load(from url: URL) {
        loader.load(from: url) { list in
            onLoad?(list)
        }
    }
}

final class QiitaListViewController: UIViewController {
    var viewModel: QiitaListViewModel!
    var imageLoader: QiitaUserImageLoaderProtocol!
    var cellViewControllers: [QiitaCellViewController] = []

    override func viewDidLoad() {
        super.viewDidLoad()
        viewModel.onLoad = { [weak self] qiitaList in
            qiitaList.forEach { qiita in
                guard let self = self else { return }
                let cellViewController = QiitaCellViewController.fromStoryboard()
                cellViewController.viewModel = QiitaCellViewModel(
                                                   qiita: qiita, loader: self.imageLoader)
                self.cellViewControllers.append(cellController)        
            }
        }
    }

    func getList(from url: URL) {
        viewModel.load(from: url)
    }
}

class SceneDelegate: UIResponder, UIWindowSceneDelegate {

    func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options connectionOptions: UIScene.ConnectionOptions) {
        ...

        let qiitaLoader = QiitaLoader(newParameter: "new parameter")
        let viewModel = QiitaListViewModel(loader: qiitaLoader)
        qiitaListViewController.viewModel = viewModel
        qiitaListViewController.imageLoader = QiitaUserImageLoader()

        window?.rootViewController = qiitaListViewController
    }
}

```  
  
`QiitaCellViewController`を生成するためには  
一覧情報を取得する必要があるため  
`QiitaListViewController`と引き離すことができません。  
  
しかし  
`QiitaLoader`の時と同様に  
`QiitaUserImageLoader`や`QiitaCellViewController`の  
変更からの影響を受けてしまいます。  
  
  
そんな時に活用できる  
Composition Rootという考えについて  
次に見ていきたいと思います。  
  
## Composition Rootとは？  
  
**モジュール同士を繋げるアプリ上のできる限り単一の場所**  
  
※  
モジュール同士となっていますが  
クラス同士でもDIが必要な場合は同じであると思っています。  
  
※   
「できる限り」とされているのは  
決して一つのクラスにしなければいけないという訳ではなく  
**同じモジュール内にある限り**は  
複数のクラスでもメソッドでも問題はないからだそうです。  
  
  
と定義されています。  
  
つまり  
DIする(インスタンスを生成する)場所を集約しよう  
ということです。  
  
こうすることで  
  
- インスタンスの管理がしやすくなる  
- クラスが疎結合になる(不要な変更の影響を受けなくなる)  
- 各クラス同士の関係が一箇所でわかる  
- 変更箇所を集約できる  
- プロパティインジェクションの設定忘れのリスクを減らせる  
  
などのメリットが挙げられます。  
  
## Composition Rootにする場所  
  
さらに  
Composition Rootとする場所としては  
  
**できる限りアプリのエントリーポイントに近いところが良い**  
  
とされています。  
  
この理由としては  
DIをエントリーポイントに近いところで行うと  
アプリの内部は具体的な実装について知る必要が少なくなり  
より変更に強い設計になると考えられています。  
(いわゆる**実装の判断を遅らせる**ということです。)  
  
iOSアプリだと  
`AppDelegate`や`SceneDelegate`がある  
**メインモジュール**にあるのが  
好ましいのではないかと思います。  
  
  
## Composition Rootを取り入れる  
  
それでは  
先ほどの例でComposition Rootを使用します。  
  
![qiita2.png](/image/daa891e8-86b8-e0a5-3d65-bd9d99afc943.png)  
  
  
```swift

struct QiitaListUIComposer {
    static func composeQiitaListViewController(qiitaLoader: QiitaLoaderProtocol, imageLoader: QiitaImageLoaderProtocol) -> QiitaListViewController {
        let viewController = QiitaListViewController.fromStroyboard()
        let viewModel = QiitaListViewModel(loader: qiitaLoader)
        viewModel.onLoad = { [weak viewController] items in
            let cellViewControllers: [QiitaCellViewController] = qiitas.map { qiita in
                QiitaCellViewController(
                    viewModel: QiitaListCellViewModel(
                        qiita: qiita,
                        loader: imageLoader)
            }
            viewController?.cellViewControllers.append(contentsOf: cellViewControllers)
            viewController?.updateTableView()
        }
        viewController.viewModel = viewModel

        return viewController
    }
}

class SceneDelegate: UIResponder, UIWindowSceneDelegate {

    func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options connectionOptions: UIScene.ConnectionOptions) {
        ...
        let viewController = QiitaListUIComposer.composeQiitaListViewController(
                                          qiitaLoader: QiitaLoader(newParameter: "new parameter"), 
                                          imageLoader: QiitaUserImageLoader())
        window?.rootViewController = viewController
    }
}

```  
  
こうすることで`QiitaListViewController`から  
`QiitaUserImageLoader`の変更の影響を受けることがなくなりました。  
  
さらに  
今回登場したクラスのインスタンス生成に将来的に変更が入る場合でも  
この`QiitaListUIComposer`に全てがまとまっているため  
どこを修正すればよいかに悩む要素も減ります。  
  
### Adapterパターン  
  
上記のComposerの中でもう一つ注目したい点として  
ComposerはAdapterパターンを適応しています。  
  
それが`viewModel.onLoad`の部分です。  
  
`QiitaListViewModel`から`[Qiita]`を受け取りますが  
`QiitaListViewController`では`[QiitaCellViewController]`が必要になります。  
  
それを`QiitaListUIComposer`が担うことで  
仮に`QiitaListViewModel`から受け取る値に変更があったとしても  
`QiitaListViewController`はただ必要な値を受け取るだけで  
変更を加える必要がなくなります。  
  
https://ja.wikipedia.org/wiki/Adapter_%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3  
  
今回は割愛しますが  
ここはかなり複雑になっているのでメソッドや別のクラスとして  
切り出した方が良いかもしれません。  
  
## Composition Rootをもっと活用する  
  
Composition Rootを活用することで  
インスタンスの管理しやすさや変更の影響を少なくするなどの  
メリットを見てきましたが  
その他でも有効活用できる方法を見ていきます。  
  
### Decoratorパターン  
  
QiitaLoaderやQiitaUserImageLoaderが  
外部からデータを取得する時に  
バックグラウンドスレッドで処理を実行する場合があります。  
`URLSession`もバックグラウンドで実行します。  
  
一方で  
UIKitには  
メインスレッドで処理を行わなければいけないという制約があります。  
  
```
Important: Use UIKit classes only from your app’s main thread or main dispatch queue, 
unless otherwise indicated. 
This restriction particularly applies to classes 
derived from UIResponder or that involve manipulating your app’s user interface in any way.
```  
  
この時にメインスレッドで処理行う場合に  
`DispatchQueue`を利用します。  
  
例えば  
`QiitaListViewController`でUITableViewを更新する時に  
  
```swift

final class QiitaListViewController: UIViewController {
    func updateTableView() {
        DispatchQueue.main.async { [weak self] in
            self?.tableView.reloadTableView()
        }
    }
}
```  
  
しかし  
こうしてしまうと  
`QiitaLoader`を利用するクラス内で  
繰り返し同じコードを記載する必要が出てきますし  
メインスレッドに戻して処理をするというのは  
Loaderの内部的な事情に影響を受けていることになり  
結合が不用意に強くなってしまっています。  
  
例えばLoaderがメモリからデータを取得する場合などは  
メインスレッドに戻す処理が必要ないかもしれません。  
  
この時にDecoratorパターンを活用することで  
この問題を解消できる場合があります。  
  
https://ja.wikipedia.org/wiki/Decorator_%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3  
  
Decoratorパターンは  
ざっくり言うと既にある機能の上にさらに機能を動的に追加するものです。  
  
Swiftでは`protocol`を使ってこれを簡単に実現できます。  
  
```swift

final class DispatchMainQueueDecorator<T> {
    private let decoratee: T

    init(_ decoratee: T) {
        self.decoratee = decoratee
    }

    func dispatch(completion: @escaping () -> Void) {
        guard Thread.isMainThread else {
            return DispatchQueue.main.async(execute: completion)
        }
        completion()
    }
}

extension DispatchMainQueueDecorator: QiitaLoaderProtocol where T == QiitaLoaderProtocol {
    func load(from url: URL, completion: @escaping ([Qiita]) -> Void) {
        decoratee.load(from: url) { [weak self] result in
            self?.dispatch { completion(result) }
        }
    }
}

struct QiitaListUIComposer {
    static func composeQiitaListViewController(qiitaLoader: QiitaLoaderProtocol, imageLoader: QiitaImageLoaderProtocol) -> QiitaListViewController {
        ...
        let viewModel = QiitaListViewModel(loader: DispatchMainQueueDecorator(qiitaLoader))
        ...
    }
}

```  
  
こうすることで  
UIモジュールの方では  
メインスレッドで処理をする必要があるかどうかを気にする必要がなくなり  
重複して`DispatchQueue.main.async`というコードを記載する必要もなくなります。  
  
### Proxyパターン  
  
似たような形ですが別のパターンを適用して  
別の問題を解決してみます。  
  
例えば  
`QiitaListViewModel`でデータを取得する前後で  
通信の開始と終了をDelegateパターンで伝達する機能を追加します。  
  
※   
現実的ではないかもしれませんがご了承ください。  
  
```swift

protocol QiitaListViewModelDelegate: AnyObject {
    func startLoading()
    func endLoading()
}

final class QiitaListViewModel {
    let loader: QiitaLoaderProtocol
    let delegate: QiitaListViewModelDelegate
    init(loader: QiitaLoaderProtocol, delegate: QiitaListViewModelDelegate) {
        self.loader = loader
        self.delegate = delegate
    }
    
    var onLoad: (([Qiita]) -> Void)?
    func load(from url: URL) {
        delegate.startLoading()
        loader.load(from: url) { list in
            onLoad?(list)
            delegate.endLoading()
        }
    }
}

```  
  
ここで`QiitaListViewController`で  
このDelegateを実装して  
通信中にインディケーターを表示するようにします。  
  
```swift

final class QiitaListViewController: UIViewController {
    @IBOutlet private(set) weak var indicator: UIActivityIndicatorView!

    private let viewModel: QiitaListViewModel
}


extension QiitaListViewController: QiitaListViewModelDelegate {
    func startLoading() {
        indicator.startAnimating()
    }

    func endLoading() {
        indicator.stopAnimating()
    }
}

struct QiitaListUIComposer {
    static func composeQiitaListViewController(qiitaLoader: QiitaLoaderProtocol,
                                               imageLoader: QiitaImageLoaderProtocol)
        -> QiitaListViewController {
            ...
            let viewController = QiitaListViewController.fromStoryboard()
            let viewModel = QiitaListViewModel(loader: DispatchMainQueueDecorator(qiitaLoader),
                                               delegate: viewController)
            viewController.viewModel = viewModel
            ...
    }
}
```  
  
ここで問題が発生します。  
  
`QiitaListViewController`は`QiitaListViewModel`を強参照していて  
`QiitaListViewModel`も`delegate`を経由して`QiitaListViewController`を強参照しています。  
**循環参照**が起きました。  
  
これを解消するためには`delegate`に`weak`を設定して弱参照にします。  
  
  
```swift

final class QiitaListViewModel {
    weak var delegate: QiitaListViewModelDelegate?
    init(loader: QiitaLoaderProtocol, delegate: QiitaListViewModelDelegate) {
        self.loader = loader
        self.delegate = delegate
    }
    
    var onLoad: (([Qiita]) -> Void)?
    func load(from url: URL) {
        delegate?.startLoading()
        loader.load(from: url) { list in
            onLoad?(list)
            delegate?.endLoading()
        }
    }
}
```  
  
しかし  
これはiOSのメモリ管理方法の事情であり  
Presenterとは関係がありません。  
  
PresenterをiOS以外で利用する場合に  
もしかしたらweakにする必要がないかもしれません。  
  
さらにこのweakが色々なところに指定されていると  
より大規模な開発では  
見えないところで問題を発生させることもあるかもしれません。  
  
そこで  
Proxyパターンを活用することを検討してみます。  
  
https://ja.wikipedia.org/wiki/Proxy_%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3  
  
Proxyパターンは  
使用したいオブジェクトの入れ物を用意して  
外からは中のオブジェクトにアクセスするのと同様に  
アクセスするようにする方法です。  
  
Swiftではstandard libraryの  
`AnySequence`などのTypeEraserの実装で利用されています。  
  
```swift

private final class WeakReferenceProxy<T: AnyObject> { |
    private weak var object: T?
    init(_ object: T) {
        self.object = object
    }
}

extension WeakReferenceProxy: QiitaListViewModelDelegate where T: QiitaListViewModelDelegate {
    func startLoading() {
        object?.startLoading()
    }

    func endLoading() {
        object?.endLoading()
    }
}

struct QiitaListUIComposer {
    static func composeQiitaListViewController(qiitaLoader: QiitaLoaderProtocol, imageLoader: QiitaImageLoaderProtocol) -> QiitaListViewController {
        ...
        let viewController = QiitaListViewController.fromStoryboard()
        let viewModel = QiitaListViewModel(loader: DispatchMainQueueDecorator(qiitaLoader))
        viewModel.delegate = WeakReferenceProxy(viewController)
        viewController.viewModel = viewModel
        ...
    }
}

```  
  
上記のように  
`WeakReferenceProxy`で  
`weak`の実装を隠しています。  
  
こうすることで  
  
- メモリ管理という内部事情を外に漏らさなくなる  
- weakで参照する必要があるインスタンスの管理を集約できる  
  
などができるようになります。  
  
# まとめ  
  
DIという方法を通して  
インスタンスの生成と使用を分離することで変更の影響を抑えるメリットや  
Composition Rootという考えを利用して  
クラス依存関係を簡潔に管理する方法を見てきました。  
  
開発の規模が大きくなればなるほど  
クラス間の依存関係は複雑になり  
余計なインスタンスの生成をしてメモリの使用量が増えたり  
違うインスタンスを参照してしまった結果  
予期せぬ動きをしてしまうことがあるかもしれません。  
  
開発規模によっては  
ここまで大きな仕組みは必要ないかもしれませんが  
Composition Rootという考えを知っておくことで  
余計な複雑性を持ち込むリスクを減らすことは  
できるのではないかと思います😃  
  
もし間違いなどございましたらご指摘していただけると嬉しいです🙇🏻‍♂️  
  
# 参考資料  
  
https://freecontent.manning.com/dependency-injection-in-net-2nd-edition-understanding-the-composition-root/  
https://blog.ploeh.dk/2011/07/28/CompositionRoot/  
  
