Kickstarterはアプリの実装をオープンソースとして公開しており  
多くのstarを獲得しています。  
  
https://github.com/kickstarter/ios-oss  
  
開発の規模が大きかったり  
iOSチームとAndroidチームが協業しているという背景から  
スムーズに安全に開発していくための特徴的な仕組みが数多く導入され  
開発時の参考になる要素がたくさんあります。  
  
そこで今回はKickstarterで導入されている一部の特徴的な要素を  
見ていきたいと思います。  
  
# 構造の統一  
  
Kickstarterでは  
iOSとAndroidで同じ構造になるように設計されています。  
言語の違いがあるものの  
両方のエンジニアがお互いのコードを理解しやすいようになっています。  
  
そういった面もあり  
iOSアプリ内の構造も一貫している箇所が多くあり  
コードの可読性や理解のしやすさを向上させる仕組みがあります。  
  
構造に関してはドキュメントがあり  
今は少し違った形になっていますが  
記載にあるような形での統一が試みられているようです。  
https://github.com/kickstarter/native-docs  
  
## Model-View-ViewModel(MVVM)  
  
Kickstarterアプリの基本設計はMVVMです。  
  
ViewControllerでユーザの入力を受け付けると  
ViewModelで処理を実行し  
結果をViewControllerへ再び戻します。  
  
ロジックはViewModelが全て担う様にすることで  
- ViewControllerとの責務を明確に分離することができる  
- ViewControllerなしにテストを実行することができる  
などのメリットがあります。  
  
## ViewModelの入力Protocolと出力Protocol  
  
ViewModelは共通の役割と命名規則を持った  
Protocolが定義されています。  
  
```swift

public protocol 〇〇ViewModelInputs {
  func viewDidLoad()
}

public protocol 〇〇ViewModelOutputs {
}

public protocol 〇〇ViewModelType {
  var inputs: 〇〇ViewModelInputs { get }
  var outputs: 〇〇ViewModelOutputs { get }
}

public final class 〇〇ViewModel: 〇〇ViewModelType, 〇〇ViewModelInputs, 
   〇〇ViewModelOutputs {
}
```  
  
### Inputs  
  
InputsはViewControllerがユーザイベントを受け取ったときなどに  
ViewModelでロジックを実行させるためのメソッドを宣言します。  
  
ViewControllerとViewModelでメソッド名は同じにし  
ViewModelとViewControllerのメソッド関連づけがすぐにわかるようになっています。  
  
### Outputs  
  
OutputsはViewModelでロジックを実行したあとに  
ViewControllerへ結果を渡すためのプロパティを宣言します。  
  
ViewModelではReactiveSwiftを使って  
リアクティブプログラミングで書かれています。  
  
https://github.com/ReactiveCocoa/ReactiveSwift  
  
そのためOutputsのプロパティはSignalとなっており  
ViewControllerはそれとViewのプロパティをbind(紐付け)させることで  
変更を検知してViewを変更させます。  
  
ViewControllerのbindViewModelでViewModelのOutputsとViewを紐付けます。  
  
これはUIViewControllerのextensionで定義されており  
viewDidLoadの直後に呼ばれるようになっています。  
  
https://github.com/kickstarter/ios-oss/blob/master/Library/UIViewController-Preparation.swift  
  
Signalの２番目のErrorパラメータにはNeverを指定し  
Errorが発生してUIの更新が止まらないようにしています。  
  
### ViewModelType  
  
これはInputsとOutputsへのインターフェイスです。  
  
## メリット  
  
### 責務がはっきりしている  
  
ViewControllerからViewModelへ入ってくる値はInputs  
ViewModelから外へ出る値はOutputsに書くということが決まっているので  
どこに何を書けば良いのかに悩む必要がありません。  
これは同時に知りたい情報を探す時の時間の短縮にも繋がります。  
  
### テストがしやすい  
  
上記にも記載しましたが  
ViewModelがロジックを全て担っているため  
ViewControllerなしでテストを実行することが可能になります。  
  
ViewControllerをテストで使用する際は  
インスタンス化やライフサイクルの呼び出しなど  
本体のテストとは関係のない部分で時間がかかったり  
余計なコードが増えます。  
  
これをViewModelの中で全て確認できるようにすることで  
開発のしやすさやスピードの向上に繋がります。  
  
下記のような方法でテストをすることが可能です。  
  
```swift

class ViewModelTests: XCTestCase {
    let vm: MyViewModelType = MyViewModel()
    let submitButtonEnabled = TestObserver<Bool, NoError>()
    
    override func setUp() {
        super.setUp()
        self.vm.outputs.submitButtonEnabled.observe(self.submitButtonEnabled.observer)
    }
}
```  
  
TestObserverはKickstarterで使われているクラスで  
リアクティブに流れてきたデータを配列で保持しており  
後々参照することでテストの判定に活用します。  
  
https://github.com/kickstarter/Kickstarter-ReactiveExtensions/blob/master/ReactiveExtensions/test-helpers/TestObserver.swift  
  
上記の例では  
ViewModelのoutputsのsubmitButtonEnabledプロパティに値が設定されると  
TestObserverのsubmitButtonEnabledのobserverプロパティに値が流れ  
内部で流れてきた値が保持されます。  
  
ViewControllerの動きは  
ViewModelのinputsメソッドを呼び出すことで同じ効果が期待できます。  
  
```swift

func testSubmitButtonEnabled() {
    self.vm.inputs.viewDidLoad()
    self.submitButtonEnabled.assertValues([false])
    
    self.vm.inputs.nameChanged(name: "Chris")
    self.submitButtonEnabled.assertValues([false])
}
```  
  
このようにInputsのメソッドを呼び出すことで  
ViewControllerのあらゆる動きをシミュレートできるので  
ViewControllerのテストはほとんど必要はありません。  
  
# シングルトンのstructのによる依存関係の管理  
  
DateやDateFormatやAPIへの接続など  
テストや開発時の環境に応じて設定を入れ替えたい依存項目に関しては  
Environmentというstructに集約されています。  
  
https://github.com/kickstarter/ios-oss/blob/master/Library/Environment.swift  
  
  
そしてAppEnvironmentの  
staticなcurrentという変数で管理されています。  
  
```swift

fileprivate static var stack: [Environment] = [Environment()]

public static var current: Environment! {
  return stack.last
}

```  
https://github.com/kickstarter/ios-oss/blob/master/Library/AppEnvironment.swift  
  
シングルトンを使用することは「一般的に」よくないこととされており  
色々と疑問を感じる方も多いかもしれません。  
  
そこでよく聞かれる疑問に対しての回答が  
下記のブログ記事に書かれています。  
https://www.pointfree.co/blog/posts/21-how-to-control-the-world  
  
※  
Pointfreeの2人は元々KickstarterのiOSエンジニアで  
Kickstarterアプリの仕組みの構築にも深く携わっていました。  
  
下記に簡単に回答について紹介します。  
  
## なぜシングルトンなのか？  
  
これを使うことでテスト時のモックの設定などが  
とても簡単にできます。  
  
下記はTestCaseクラスの中のsetUpメソッドです。  
テストではこのクラスを継承することで  
テスト用の環境に入れ替えています。  
  
```swift

override func setUp() {
  super.setUp()

  UIView.doBadSwizzleStuff()
  UIViewController.doBadSwizzleStuff()

  var calendar = Calendar(identifier: .gregorian)
  calendar.timeZone = TimeZone(identifier: "GMT")!

  AppEnvironment.pushEnvironment(
    apiService: self.apiService,
    apiDelayInterval: .seconds(0),
    application: UIApplication.shared,
    assetImageGeneratorType: AVAssetImageGenerator.self,
    cache: self.cache,
    calendar: calendar,
    config: self.config,
    cookieStorage: self.cookieStorage,
    countryCode: "US",
    currentUser: nil,
    dateType: self.dateType,
    debounceInterval: .seconds(0),
    device: MockDevice(),
    is1PasswordSupported: { true },
    isVoiceOverRunning: { false },
    koala: Koala(client: self.trackingClient, loggedInUser: nil),
    language: .en,
    launchedCountries: .init(),
    locale: .init(identifier: "en_US"),
    mainBundle: self.mainBundle,
    pushRegistrationType: MockPushRegistration.self,
    reachability: self.reachability.producer,
    scheduler: self.scheduler,
    ubiquitousStore: self.ubiquitousStore,
    userDefaults: self.userDefaults
  )
}
```  
https://github.com/kickstarter/ios-oss/blob/master/Library/TestHelpers/TestCase.swift  
  
## シングルトンは悪いもの？  
  
一般的には「使わない方が良い」とされていますが  
これは「変更などがどこで起こるかがわからずコントロールできない」からです。  
例えばFileManager.defaultを使用しているとコードとファイルシステムが密接に結びついてしまい  
テストのために設定を変更するのは難しいです。  
  
一方でKickstarterのシングルトンに関しては  
全てのプロパティが上書き可能でコントロールすることができます。  
  
さらに外部のシングルトンへの呼び出しを統一することもでき  
コードの中でシングルトンへのアクセスや変更を散らばらせることもなくなります。  
  
## グローバル変数が変更可能なことは悪いこと？  
  
確かに変更可能にすることでバグを起こす可能性があったり  
追跡するのがかなり困難になってしまいます。  
  
しかし  
目的は開発時やテスト時の設定をできる限り簡単に入れ替えたいということであるので  
リリースビルドの際はletにすることで変更不可にしたり  
swiftlintで変更がされた場合にチェックをするなど  
不用意な変更ができないようにすることも可能です。  
  
※  
Kickstarterの実装では個々に`if DEBUG`を付けてMockと差し替えているようです。  
https://github.com/kickstarter/ios-oss/blob/27858bf856fc37434cbad4a2a8fe30e6826e56b4/KsApi/MockService.swift#L1  
@y__n さんありがとうございます  
  
## なぜprotocolではなくstruct？  
  
今回のケースで考えるとProtocolを使った時の方が  
ボイラープレートが増えてしまい  
設定の入れ替えがしづらいからです。  
  
Protocolを使った例を見てみたいと思います。  
  
APIのロジックを入れ替えたいためにまずProtocolを定義します。  
  
```swift

protocol APIClientProtocol {
  var token: String? { get set }
  func fetchCurrentUser(_ completionHandler: (Result<User, Error>) -> Void)
}
```  
  
```swift

extension APIClient: APIClientProtocol {}
```  
  
本番用のAPIClientがこれに適合するようにします。  
  
次にモック用のクラスを作成します。  
返したい結果を初期化時に渡せるように  
tokenとcurrentUserResultプロパティも定義します。  
  
  
```swift

class MockAPIClient: APIClientProtocol {
  var token: String?

  var currentUserResult: Result<User, Error>?
  func fetchCurrentUser(_ completionHandler: (Result<User, Error>) -> Void) {
    completionHandler(self.fetchCurrentUserResult!)
  }
}
```  
  
これをEnvironmentで差し替え可能にするためにプロパティを追加します。  
  
```swift

struct Environment {
  var api: APIClientProtocol = APIClient.shared
}
```  
  
これまでの実装をまとめると下記のようになります。  
  
```swift

protocol APIClientProtocol {
  func fetchItem(_ completionHandler: (Result<Item, Error>) -> Void) -> Void
}

extension APIClient: APIClientProtocol {}

class MockAPIClient: APIClientProtocol {
  var token: String?

  var itemResult: Result<Item, Error>?
  func fetchItem(_ completionHandler: (Result<Item, Error>) -> Void) {
    completionHandler(self.fetchItemResult!)
  }
}

struct Environment {
  var api: APIClientProtocol = APIClient.shared
}
```  
  
これをstructを使った場合を考えてみます。  
  
```swift

struct API {
  var fetchItem = APIClient.shared.fetchItem
}

struct Environment {
  var api = API()
}
```  
  
このようにかなり短い実装で実現することができます。  
  
さらにProtocolの場合は他のAPIが追加になった際は  
毎回上記のような特定の結果を返す様にプロパティを追加して  
メソッドを実装していくなどの手間がかかってしまいます。  
  
### structの実装によるデメリット  
  
上記で見てきたようにProtocolを使うよりも全てが優っているように見えますが  
デメリットもあります。  
  
一番大きいものとしては引数ラベルの問題があります。  
  
クロージャには引数ラベルを定義することができないため、  
引数に値を渡す時に何にどの値を設定しているのかがわからなくなってしまいます。  
  
```swift

APIClient.shared.fetchItem(byId: Int) { result in
  ...
}
```  
  
structを使用する場合  
プロパティの方でラベルを使用しなければならず  
呼び出し側へ引数ラベルを渡すことができません。  
  
```swift

struct API {
  var fetchItemById = APIClient.shared.fetchItem(byId:)
  // …
}

AppEnvironment.current.fetchItemById(1)
// vs.
APIClient.shared.fetchItem(byId: 1)
```  
  
これが複数の引数を持つ場合は  
間違えるリスクがかなり高くなります。  
  
```swift

APIClient.shared.register(
  withId: 1,
  name: "orange",
  price: 100
) { result in
  ...
}
// vs.
AppEnvironment.current.register(1, "orange", 100)
```  
  
これの対策としてはプロパティの名前に  
引数ラベルを入れてしまうという方法があります。  
  
```swift

struct API {
  var registerWithIdNamePrice = APIClient.shared.register
  …
}
```  
  
しかし  
これだとメソッド名がかなり長くなってしまい  
呼び出す側としては結構煩わしくなります。  
  
そこで少しボイラープレートが増えますが  
下記のようにextensionで対応してみます。  
  
```swift

extension API {
  func register(
    withId id: Int,
    name: String,
    price: Int
    ) {

    self.registerWithIdNamePrice(
      id, name, price
    )
  }
}

AppEnvironment.current.register(withId: 1, name: "orange", price: 100)
```  
  
これは必要に応じて追加すればよく  
Protocolのように毎回定義する必要はありません。  
  
## DIは明示的に行った方が良いのでは？  
  
DIはグローバルなものを引数として渡すことになります。  
しかしこれもProtocolのように余計なボイラープレートを増やしてしまうケースが多々あります。  
  
まずプロパティを定義し  
それを受け取るためのイニシャライザを定義する必要があります。  
  
```swift

class ViewController: UIViewController {
  let api: APIClientProtocol
  let date: () -> Date
  let label = UILabel()

  init(_ api: APIClientProtocol, _ date: () -> Date) {
    self.api = api
    self.date = date
  }

  func explain() {
    self.api.fetchItem { result in
      if let item = result.success {
        self.label.text = "This item is, \(item.name)! Now \(self.date())."
      }
    }
  }
}
```  
  
またViewControllerでStoryboardを使用している場合は  
イニシャライザを使用せず  
プロパティに値を設定するケースが多くあります。  
  
他にも  
子のViewControllerでAPIを使う必要があるので  
親のViewControllerでもDIしなければならなくなる  
というケースなどもあります。  
  
  
```swift

class ViewController: UIViewController {
  var api: APIClientProtocol!
  var date: (() -> Date)!
  @IBOutlet var label: UILabel!

  func explain() {
    self.api.fetchItem { result in
      if let item = result.success {
        self.label.text = "This item is, \(item.name)! Now \(self.date())."
      }
    }
  }
}
```  
  
もし設定を忘れた場合はクラッシュします。  
  
このようにボイラープレートを書く煩わしさや  
クラッシュする可能性を生み出してしまいます。  
  
  
  
これがstructの場合ですと  
  
```swift

class ViewController: UIViewController {
  @IBOutlet var label: UILabel!

  func explain() {
    AppEnvironment.current.api.fetchItem { result in
      if let item = result.success {
        self.label.text = "This item is, \(item.name)! Now \(self.date())."
      }
    }
  }
}
```  
  
とボイラープレートを書く必要もなくなり  
クラッシュするリスクもほとんどなくなります。  
  
# 画面遷移を集約して管理  
  
上記のEnvironmentと同様に  
アプリの様々な場所から参照されるものを一箇所に集約しているケースが多く見られます。  
  
例えば  
下記のNavigationというenumで  
すべてのエントリーポイントを管理しており  
APIのパスやディープリンクのURLなど  
あらゆる入り口を管理しています。  
  
Navigation  
https://github.com/kickstarter/ios-oss/blob/master/Library/Navigation.swift  
  
  
ShortcutItemもenumで管理しています。  
  
Shortcut  
https://github.com/kickstarter/ios-oss/blob/master/Library/ShortcutItem.swift  
  
AppDelegateでディープリンクやショートカットからの起動のトリガーを受け取り  
AppDelegateViewModelで遷移をコントロールしています。  
  
```swift

func application(_ application: UIApplication,
                 open url: URL,
                 sourceApplication: String?,
                 annotation: Any) -> Bool {
    
    return self.viewModel.inputs.applicationOpenUrl(application: application,
                                                    url: url,
                                                    sourceApplication: sourceApplication,
                                                    annotation: annotation)
}

internal func application(_ application: UIApplication,
                          didReceiveRemoteNotification userInfo: [AnyHashable: Any]) {
    
    self.viewModel.inputs.didReceive(remoteNotification: userInfo,
                                     applicationIsActive: application.applicationState == .active)
}

internal func application(_ application: UIApplication,
                          performActionFor shortcutItem: UIApplicationShortcutItem,
                          completionHandler: @escaping (Bool) -> Void) {
    
    self.viewModel.inputs.applicationPerformActionForShortcutItem(shortcutItem)
    completionHandler(true)
}
```  
  
AppDelgate  
https://github.com/kickstarter/ios-oss/blob/1062a97a903fea9e90167011e923b0ff1281efa5/Kickstarter-iOS/AppDelegate.swift#L228  
  
Navagtion.match  
https://github.com/kickstarter/ios-oss/blob/0e2aa2f15e8c098e67a4a80ced833adf4e4194a0/Library/Navigation.swift#L216  
  
AppDelegateViewModelのnavigation(fromPushEnvelope:)  
https://github.com/kickstarter/ios-oss/blob/master/Kickstarter-iOS/ViewModels/AppDelegateViewModel.swift#L737  
  
AppDelegateViewModelのnavigation(fromShortcutItem:)  
https://github.com/kickstarter/ios-oss/blob/master/Kickstarter-iOS/ViewModels/AppDelegateViewModel.swift#L804  
  
## メリット  
  
### 探しやすく変更しやすい  
  
一箇所に集約しているので  
この場所を見にいけばどういう場合にどこへ遷移するのかがわかるため  
コード内を探し回る負担がなくなります。  
また同時にドキュメントとしての役割も果たします。  
  
変更時も一箇所の変更で完了するため  
他に与える影響などを心配する負担も減らすこともできます。  
  
### テストがしやすい  
  
AppDelegateはほとんど何もしていないためテストする必要がなくなり  
Navigationを独立してテストすることができるようになります。  
  
https://github.com/kickstarter/ios-oss/blob/master/Library/NavigationTests.swift  
  
# Playground駆動開発  
  
個々のViewControllerの最初のプロトタイプを行うのに  
Playgroundを活用しています。  
  
## メリット  
  
### フィードバックが即座に得られる  
  
これはすぐにフィードバックを得られることができ  
初期段階に画面を見ながらデザインの相談ができるなど  
開発スピードを向上させるのに大きな役割を果たしています。  
  
https://github.com/kickstarter/ios-oss/tree/master/Kickstarter-iOS.playground  
  
下記のように  
コードで基本的なレイアウトを組み立て  
  
```swift

import UIKit
import PlaygroundSupport

class MyViewController: UIViewController {
  let rootStackView = UIStackView()
  let submitButton = UIButton()

  override func viewDidLoad() {
    super.viewDidLoad()

    self.submitButton.setTitle("Submit", for: .normal)
    
    self.view.addSubview(self.rootStackView)
    self.rootStackView.addArrangedSubview(self.submitButton)

    NSLayoutConstraint.activate([
      self.rootStackView.leadingAnchor.constraint(equalTo: self.view.leadingAnchor),
      self.rootStackView.trailingAnchor.constraint(equalTo: self.view.trailingAnchor),
      self.rootStackView.topAnchor.constraint(equalTo: self.view.topAnchor),
      self.rootStackView.bottomAnchor.constraint(lessThanOrEqualTo: self.view.bottomAnchor),
      ])
  }
}
```  
  
デザインを変更したい場合は  
viewDidLoadの中でプロパティに値を設定していきます。  
  
```swift

self.submitButton.backgroundColor = .blue
self.submitButton.layer.cornerRadius = 6
self.submitButton.layer.masksToBounds = true
```  
  
### 端末依存や設定によって変化するデザインのテストが容易  
  
デバイスの向きやDynamicTypeの設定などによるデザインの変化を確認するには  
ターゲットのViewControllerの親ViewControllerを生成し  
そこからsetOverrideTraitCollectionメソッドを通してTraitCollectionを伝達させます。  
  
setOverrideTraitCollection  
https://developer.apple.com/documentation/uikit/uiviewcontroller/1621406-setoverridetraitcollection#  
  
```swift

public func playgroundControllers(
  device: Device = .phone4_7inch,
  orientation: Orientation = .portrait,
  child: UIViewController = UIViewController(),
  additionalTraits: UITraitCollection = .init()
)
  -> (parent: UIViewController, child: UIViewController) {
  let parent = UIViewController()
  parent.view.backgroundColor = .white

  child.view.backgroundColor = .white
  child.view.autoresizingMask = [.flexibleWidth, .flexibleHeight]

  parent.addChild(child)
  parent.view.addSubview(child.view)
  child.didMove(toParent: parent)

  child.view.frame = parent.view.frame

  let traits: UITraitCollection
  switch (device, orientation) {
  case (.phone5_5inch, .portrait):
    parent.view.frame = .init(x: 0, y: 0, width: 414, height: 736)
    traits = .init(traitsFrom: [
      .init(horizontalSizeClass: .compact),
      .init(verticalSizeClass: .regular),
      .init(userInterfaceIdiom: .phone)
    ])
    ...
  }
  let allTraits = UITraitCollection.init(traitsFrom: [traits, additionalTraits])
  parent.setOverrideTraitCollection(allTraits, forChild: child)

  return (parent, child)
}
```  
  
https://github.com/kickstarter/ios-oss/blob/master/Kickstarter-iOS.playground/Sources/playgroundController.swift  
  
  
下記のように使用します。  
  
```swift

let controller = RewardPledgeViewController
  .configuredWith(project: project, reward: project.rewards.last!)

let (parent, _) = playgroundControllers(device: .phone4_7inch, orientation: .portrait, child: controller)

let frame = parent.view.frame |> CGRect.lens.size.height .~ 800
PlaygroundPage.current.liveView = parent
parent.view.frame = frame
```  
  
playgroundControllersに設定する値を変更するだけで  
様々な環境のテストを実行し  
フィードバックを受け取ることができます。  
  
より詳しい内容などは下記で紹介されています。  
http://talk.objc.io/episodes/S01E51-playground-driven-development  
  
# まとめ  
  
Kickstarterでは  
開発を安全にスピーディーに進めていくための仕組みが  
たくさん存在することがわかりました。  
  
今回は特徴的な部分のみの紹介となりましたが  
他にも様々な点で開発を支える仕組みがあります。  
  
開発の規模が大きくなっていくにつれて  
少しずつ積み上げてきた結果ではあり  
今後自分の携わっている開発が大きくなってきた時に  
何が起きてどうすれば良いのかを考える際の  
大変参考になるなと感じました。  
  
また  
Kickstarterアプリはどんどんと進化をしており  
現在でも新しい仕組みを導入しようと試みています。  
  
例えば  
ViewModelをただの関数に変更することで  
状態を保持しないより安全な設計にできないかという試みをしていました。  
  
現在はうまく行かないエッジケースがあるようで一旦クローズしているようですが  
今後も検討はしていくようです。  
  
https://github.com/kickstarter/ios-oss/pull/504  
https://github.com/kickstarter/ios-oss/pull/612  
  
  
今後もどういう形で進化していくのか  
動向が気になりますね😃  
