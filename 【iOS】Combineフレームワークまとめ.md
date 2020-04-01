  
- 2019/6/14 Combine in URLSessionについて追記しました。  
- 2019/6/18 Xcode11 Beta2より`＠Published`と`DataTaskPublisher`が補完されるようになりました。ドキュメントもできていましたのでリンクも追記しています。  
- 2019/6/24 Subscriberのreceive(_ input: Self.Input) -> Subscribers.Demandについて追記しました。  
- 2019/6/29 下記について追加、修正しました。   
    - catchがなぜ元のPublisherの出力を止めてしまうのかの理由について  
    - 処理を繋げた際の型とtype eraserについて  
    - Combineで事前に定義されているPublisherとSubscriberについて  
    - FRPについて  
    - assignメソッドとsinkメソッドをPublisherのextensionへ修正  
- 2019/7/3  
    - XCode11 Beta3よりFoundationと統合され、Combineをimportしなくても`URLSession`や`NotificationCenter`などでCombineを使用することができるようになりました。(`@Published`はまだ使えなかったです。)  
- 2019/7/15 デバッグについて追記  
- 2019/7/17 FutureとJustがPublishersから移動していたのでリンク先の修正など行いました。  
- 2019/7/18 tryCatchのドキュメントへのリンクを追加しました。またXcode11 Beta4より`Publisher`の`sink`の`Failure`が`Never`になりました(詳細は下記)  
- 2019/7/19 Xcode11 Beta4でBindableObjectがdidChange->willChange, didSet->willSetに変わったので一部修正しました。  
- 2019/7/30 Xcode11 Beta5でBindableObjectはdeprecatedになり`ObservableObject`を使うようになったので追記しました。`@ObjectBinding`も`@ObservedObject`に名前が変わりました。  
- 2019/7/31 `ObservableObject`がCombineに含まれること、`Identifiable`はStandard Libraryに移動したことについて記載しました。また`@Published`をつけるとwillChangeなどの記載が不要になりました。  
- 2019/8/1 Beta5だとプロパティのwillSetで値の変更通知していたロジックが`@Published`を使用しないと動かなくなっています。またCancellableのメモリ管理の挙動が変わりました。(詳細は下記に記載しています。)  
- 2019/8/6 `AnyCancellable`の`store`メソッドと`objectWillChange`について追記しました。  
- 2019/8/21 Beta6で`@Published`の`projectedValue`の型が`Published.Publisher`になったこと、`Binding`の`Collection`へのConditional Comformanceが廃止されたことについて追記しました。  
- 2019/9/21 Schedulerについて書いた記事のリンクを追加しました。  
- 2019/10/17 Demandについて間違いがございましたので修正しました。`Subscribers.Demand`を`enum`から`struct`に変更。`receive(_ input: Self.Input) -> Subscribers.Demand`は現在の`Demand`に対する調整値を設定するように説明を変更。  
  
# 2019/7/18追記 Xcode11 Beta4で起きている事象  
  
これまでできていた`sink`の使い方でコンパイルエラーになるものがありました。  
  
https://developer.apple.com/documentation/combine/publisher/3343979-sink  
  
他にもあるかもしれません。  
  
```swift

enum SomeError: Swift.Error {
    case somethingWentWrong
}

let subject = PassthroughSubject<String, SomeError>()

// error: referencing instance method 'sink(receiveValue:)' on 'Publisher' requires the types 'SomeError' and 'Never' be equivalent
let subscription = subject.sink { _ in }

// これも🙅🏻‍♂️
let subscription = subject.sink(receiveValue: { _ in })
```  
  
下記のようにするとコンパイルが通ります。  
  
```swift

// これは🙆🏻‍♂️
let subscription = subject.sink(receiveCompletion: { _ in }, receiveValue: { _ in })
```  
  
`sink(receiveValue:)`の`Failure`が`Never`になったようです。  
  
```swift

extension Publisher where Self.Failure == Never {
    public func sink(receiveValue: @escaping ((Self.Output) -> Void)) -> Subscribers.Sink<Self.Output, Self.Failure>
}

```  
  
```swift

// これは🙆🏻‍♂️
let subject = PassthroughSubject<String, Never>()
let subscription = subject.sink(receiveValue: { _ in })
```  
  
2019/8/7追記 Beta5でドキュメントへも記載がされていました。  
https://developer.apple.com/documentation/combine/anypublisher/3362546-sink  
  
`receiveValue`のみを引数に取る`sink`メソッドは  
`Failure`が`Never`のときのみ利用可能という記載に統一されています。  
  
# 2019/8/1追記 Xcode Beta5の変更の影響  
  
## willSetから値の変更の通知が動かなくなった？  
  
willSetでwillChangeのsendでデータの変更を通知していたものは  
動かなくなっていました。  
  
下記のようなコードでボタンのdisabledを制御していましたが  
Beta5では動かなくなっています。  
  
※ コードは一部省略しています。  
  
```swift


final class LoginBindableObject: ObservableObject  {
    var willChange = PassthroughSubject<Void, Never>()
    
    var id: String = "" {
        willSet {
            willChange.send(())
        }
    }
    var password: String = "" {
        willSet {
            willChange.send(())
        }
    }
    var isEnabled: Bool {
        return !id.isEmpty && !password.isEmpty
    }    
}

struct LoginView: View {
    @ObservedObject var viewModel: LoginBindableObject
    
    var body: some View {
        VStack(alignment: .center, spacing: 24) {
            HStack(spacing: 12) {
                Text("ID:")
                TextField("IDを入力してください", text: $viewModel.id)
                    .textFieldStyle(RoundedBorderTextFieldStyle())
            }
            HStack(spacing: 12) {
                Text("パスワード:")
                SecureField("パスワードを入力してください", text: $viewModel.password)
                    .textFieldStyle(RoundedBorderTextFieldStyle())
            }
            Button("ログイン"){
            }
            .disabled(!viewModel.isEnabled)
        }.padding()
    }
}
```   
  
こうすればうまく行きました。  
  
※ コードは一部省略しています。  
  
```swift

final class LoginBindableObject: ObservableObject  {
    @Published var id: String = ""
    @Published var password: String = ""
    var isEnabled: Bool {
        return !id.isEmpty && !password.isEmpty
    }    
}

```  
  
2019 8/7 追記 ドキュメントの例も更新されていました。  
  
```swift

struct MyFoo {
    @Published var bar: String
}
let foo = MyFoo(bar: "bar")
let barSink = foo.$bar
    .sink() {
        print("bar value: \($0)")
}
```  
  
### objectWillChange  
  
上記の`willChange`ですが`objectWillChange`というプロパティに変更になっているようでした。  
https://developer.apple.com/documentation/combine/observableobject/3362557-objectwillchange  
  
ただ`@Published`を使えば不要なので  
単に値の変更通知を行う場合にはあえて使う必要はないのかと思っています。  
  
  
  
## Cancellableのメモリ管理が変わった(直った？)  
  
↓にも記載があるのですが参照を保持していなくても  
参照が残っているような挙動をしていました。  
https://medium.com/better-programming/swift-5-1-and-combine-memory-management-a-problem-14a3eb49f7ae  
  
  
例えば下記の様なテストが以前は通っていました。  
  
  
※ コードは一部省略しています。  
  
```swift

func testApplyLogin() {
    let expectation = self.expectation(description: "testApplyLogin")
    let publisher = AuthPublisher()

　　 // 参照を保持しない
    _ = publisher.userStream.sink { user in
        XCTAssert(user == .mock)

        expectation.fulfill()
    }
    publisher.apply(.didLogin(.mock))
    waitForExpectations(timeout: 0.5)
}
```  
  
参照が解放されるはずなので  
これは失敗するような気がしますが通っていました。  
  
Beta5では失敗するようになり下記の様に参照を保持する必要が出てきました。  
  
※ コードは一部省略しています。  
  
```swift

// クラス変数
var cancellables: [Cancellable] = []

func testApplyLogin() {
    let expectation = self.expectation(description: "testApplyLogin")
    let publisher = AuthPublisher()

    // 参照を保持
    let userStream = publisher.userStream.sink { user in
        XCTAssert(user == .mock)

        expectation.fulfill()
    }
    publisher.apply(.didLogin(.mock))
    cancellables += [
        userStream
    ]

    waitForExpectations(timeout: 0.5)
}
```  
  
これはバグが修正されたのではないかと考えています。  
  
  
# 2019/8/6追記 AnyCancellableのstoreメソッド  
  
`AnyCancellable`にすると`store`メソッドが利用できます。  
  
`store(in:)`  
https://developer.apple.com/documentation/combine/anycancellable/3333294-store  
https://developer.apple.com/documentation/combine/anycancellable/3333295-store  
  
これを利用すると下記のように記載ができます。  
  
```swift

@Published var items: [QiitaItemViewModel] = []
@Published var searchText: String = ""
private var cancellables: Set<AnyCancellable> = []

$searchText
    .removeDuplicates()
    .debounce(for: 0.8, scheduler: DispatchQueue.main)
    .flatMap { keyword in
        // ネットワーク通信で情報を取得するなど
    }
    .receive(on: DispatchQueue.main)
    .assign(to: \.items, on: self)
    .store(in: &cancellables)
```  
  
  
# 2019/8/21追記 Beta6でのアップデート  
  
あまり大きな変更をなさそうですが`@Published`の`projectValue`の型がPublished.Publisherになりました。  
https://developer.apple.com/documentation/combine/published/publisher  
  
BindingのCollectionへのConditional Comformancegが廃止されました。  
https://developer.apple.com/documentation/swiftui/binding  
  
  
# 追記や更新について  
  
※~~まだベータ版ですので今後変更点などを追って更新していきたいと思います。  
ドキュメントへのリンクでメソッド名の前に番号が付いているものは変わることが多々あるため  
見つけ次第更新します。(もし気がついた方がいらっしゃれば教えてください🙇🏻‍♂️)~~~  
  
※ 9/12更新　  
GMがリリースされましたのが  
過去の経緯を追っていくためにも変更履歴は残しておきたいと思います。  
  
今後も更新があったものに関しては随時追っていきたいと思います。  
  
# Apple公式の非同期処理を扱うフレームワークの登場  
  
iOS13で新しく追加されたフレームワークの一つに  
**Combineフレームワーク**があります。  
  
これは非同期の処理などを扱うためのフレームワークで  
今まではサードライブラリで実装しているものが多くありましたが  
今回Appleの正式なものとして登場し注目を浴びています。  
  
私もRxSwiftを業務で使用しているので  
何となくそれっぽく使えそうだなと思っていましたが  
  
ちょっと見てみると似ているようで違う部分もあり  
今回は今後学んでいくための最初のきっかけとして  
WWDC2019の動画を中心にCombineフレームワークについてどういうものか調べてみました。  
  
  
WWDC2019動画 Introducing Combine  
https://developer.apple.com/videos/play/wwdc2019/722  
  
WWDC2019動画 Combine in Practice  
https://developer.apple.com/videos/play/wwdc2019/721  
  
まずは**Introducing Combine**から見ていきます。  
  
# ※Combineフレームワークとは？  
  
※ 以下、Combineと呼びます。  
  
発表の中では  
**時間とともに値を処理するための統一的で宣言的なAPI**  
と言っています。  
  
![スクリーンショット 2019-06-08 12.35.10.png](/image/19ea042d-fc2f-9722-0fff-9a43d0d0a098.png)  
  
  
ドキュメントでは  
**イベントを処理する※Operator(オペレーター)を組み合わせて**  
**非同期のイベント処理を扱いやすくしたもの**  
とあります。  
  
※ ここからOperatorと呼びます。  
  
https://developer.apple.com/documentation/combine  
  
※ 2019/6/29 追記  
  
また  
※**関数型リアクティブプログラミング(Functional reactive programming)**ライブラリ  
の一つであるとも言われています。  
  
※ ここからFRPと呼びます。  
  
強力な型を持ったSwiftの性質を活かして  
他の言語やライブラリに見られるのと  
同様の関数型リアクティブの概念を使用しています。  
  
# FRPとは？  
  
関数型プログラミングを基にして  
**データの流れ**※を扱うプログラミングです。  
  
※ ここからストリームと呼びます。  
  
関数型プログラミングで配列を操作するのと同様に  
FRPではデータの流れを操作します。  
`map`、`filter`、`reduce`といったメソッドは関数型プログラミングと  
同様の機能を持っています。  
  
さらに  
FRPではストリームの分割や統合、変換などの機能も有しています。  
  
イベントやデータの連続した断片などを非同期の情報の流れとして扱うことがあるかと思いますが  
これはオブジェクトを監視し変更があった際に通知がきて更新を行うという  
いわゆる**オブザーバー(Observer)パターン**で実装します。  
  
時間とともにこれが繰り返されることで  
ストリームとしてこの更新を見ることができます。  
  
FRPでは  
  
変化する一つ以上の要素の変化を一緒に見たい  
ストリームの途中データに操作を行いたい  
エラー場合にある処理を加えたい  
あるタイミングで特別な処理を実行したい  
  
など  
あらゆるイベントにシステムがどういうレスポンスをするのかをまとめます。  
  
FRPはユーザインターフェイスやAPIなど  
外部リソースのデータの変換や処理をするのに効果的です。  
  
  
# CombineとFRP  
  
Combineは  
FRPの特徴と強力な型付け言語であるSwiftの特徴を組み合わせて作成されています。  
  
Combineは組み合わせて使えるように構成されており  
いくつかのAppleの他のフレームワークに明示的にサポートされています。  
  
SwiftUIでは`Subscriber`も※データの送り手(`Publisher`)も使用しており  
もっとも注目されている例です。  
  
※ 以下、**Publisher**と呼びます。  
  
Foundationの`NotificationCenter`や`URLSession`も  
`Publisher`を返すメソッドが追加になりました。  
  
  
# 4つの特徴  
  
Combineには4つの特徴があります。  
  
## ジェネリック  
ボイラープレートを減らすことができると同時に  
一度処理を書いてしまえばあらゆる型に適応することができます。  
  
  
## 型安全  
コンパイラがエラーのチェックをしてくれるため  
ランタイムエラーのリスクを軽減できます。  
  
## Compositionファースト  
  
コア概念が小さくて非常にシンプルで理解しやすく  
それを組み合わせてより大きな処理を扱っていきます。  
  
## リクエスト駆動  
  
Combineではデータを受け取る側(リクエストをする側)から  
あらゆる処理の実行が開始されます。  
  
また  
データの流量制御(back pressure)の概念を含んでおり  
データを受け取る側が  
一度にどのくらいのデータが必要で、どのくらい処理する必要があるのか  
などのコントロールをすることができます。  
  
不要になった場合はキャンセルをすることも可能です。  
  
  
# 主要な3つコンセプト  
  
Combineには3つの主要なコンセプトがあります。  
  
- Publishers  
- Subscribers  
- Operators  
  
## Publishers  
  
データを提供します。  
`Output`と`Failure`の2つの`associatedtype`を持ちます。  
https://developer.apple.com/documentation/combine/publisher  
  
値型で`Subscriber`は`Publisher`に登録することができます。  
  
発表のスライドでは  
`Publisher`プロトコルは下記のように定義されています。  
  
![スクリーンショット 2019-06-08 12.58.09.png](/image/dd4d657f-1e42-a0e6-e225-4632d04c4792.png)  
  
### receive(subscriber:)とsubscribe(_:)  
  
ここがちょっとわかっていないのですが  
2019/6/8の時点ですと  
ドキュメント上やコードの定義では下記のメソッドが  
requiredになっています。  
  
`receive(subscriber:)`  
https://developer.apple.com/documentation/combine/publisher/3229093-receive  
  
しかし  
ドキュメントなどを見てみると  
  
```
This function is called 
to attach the specified Subscriber 
to this Publisher by subscribe(_:)
```  
  
と記載があり  
実際にAPIとして使用するものとしては  
  
`subscribe(_:)`  
  
https://developer.apple.com/documentation/combine/publisher/3204756-subscribe  
  
になります。  
  
`Output`が出力される値で  
`Failure`がエラー時に出力される型で  
`Output`は`Subscriber`のInputの型と  
`Failure`は`Subscriber`の`Failure`と一致する必要があります。  
  
```swift

Publisher     Subscriber
<Output>  --> <Input>
<Failure> --> <Failure>
```  
  
https://developer.apple.com/documentation/combine/publisher  
  
  
例として  
`NotificationCenter`の`extension`が紹介されており  
エラーが発生しないため`Failure`には`Never`が指定されています。  
  
```swift

extension NotificationCenter {
    struct Publisher: Combine.Publisher {
        typealias Output = Notification
        typealias Failure = Never
        init(center: NotificationCenter, name: Notification.Name, object: Any? = nil)
    }
}
```  
  
※ 上記の例はコンパイルが通らないのですが、  
このメソッド(struct)が結構他のセッションでも使われているのにも関わらず  
実装が見つからないので  
勝手に脳内補完してみると下記のようになりました。  
間違っていたらご指摘ください🙏🏻  
  
```swift
extension NotificationCenter {
    struct Publisher: Combine.Publisher {
        typealias Output = Notification
        typealias Failure = Never
        
        let center: NotificationCenter
        let name: Notification.Name
        let object: Any?
        
        init(center: NotificationCenter, name: Notification.Name, object: Any? = nil) {
            self.center = center
            self.name = name
            self.object = object
        }
        
        func receive<S>(subscriber: S) where S : Subscriber, Failure == S.Failure, Output == S.Input {
            subscribe(subscriber)
        }
    }
    
    func publisher(for name: Notification.Name, object: Any? = nil) -> Publisher {
        return Publisher(center: self, name: name, object: object)
    }
}
```  
  
https://developer.apple.com/documentation/foundation/notificationcenter/publisher  
https://developer.apple.com/documentation/foundation/notificationcenter/3329353-publisher  
  
※ 2019/6/29追記  
  
Combineは便利な`Publisher`を事前に提供しています。  
  
- ~~Publishers.Empty~~   
https://developer.apple.com/documentation/combine/publishers/empty  
  
- ~~Publishers.Fail~~   
https://developer.apple.com/documentation/combine/publishers/fail  
  
- ~~Publishers.Just~~   
~~https://developer.apple.com/documentation/combine/publishers/just~~  
  
- Publishers.Once   
https://developer.apple.com/documentation/combine/publishers/once  
  
- Publishers.Optional   
https://developer.apple.com/documentation/combine/publishers/optional  
  
- Publishers.Sequence   
https://developer.apple.com/documentation/combine/publishers/sequence  
  
- ~~Publishers.Deferred~~   
https://developer.apple.com/documentation/combine/publishers/deferred  
  
- ~~Publishers.Future~~   
~~https://developer.apple.com/documentation/combine/publishers/future~~  
  
※ 2019/7/31 追記 下記はPublishersから移動していました。  
https://developer.apple.com/documentation/combine/just  
https://developer.apple.com/documentation/combine/empty  
https://developer.apple.com/documentation/combine/deferred  
https://developer.apple.com/documentation/combine/future  
  
中身はテキトーですが  
下記のようにしてAnyPublisherに変換したりできます。  
  
```swift

extension Int {
    enum NumberConvertError: Error {
        case convert
    }
    
    func toAnyPublisher(string: String) -> AnyPublisher<Int, Error> {
        guard let number = Int(string) else {
            return Fail(error: NumberConvertError.convert)
            .eraseToAnyPublisher()
        }
        return Just(number)
            .setFailureType(to: Error.self)
            .eraseToAnyPublisher()
    }
}
```  
  
`setFailureType`は  
`Publisher`の`Failure`が`Never`の場合のみに使用でき  
指定した`Error` Protocolに適合した型を出力する`Publisher`に変換します。  
上記の例ですと`Just<Int, Never>`から`Just<Int, Error>`に変換しています。  
  
`setFailureType`  
https://developer.apple.com/documentation/combine/just/3343941-setfailuretype  
  
新しく`Record`というstructが追加されいましたが  
詳細がわかり次第追記します。  
https://developer.apple.com/documentation/combine/record  
  
他のAppleのAPIも`Publisher`を提供しているものはあります。  
  
- Timer.publish、Timer.TimerPublisher  
https://developer.apple.com/documentation/foundation/timer/timerpublisher  
  
- URLSession dataTaskPublisher  
https://developer.apple.com/documentation/foundation/urlsession/3329707-datataskpublisher  
  
さらに  
~~`Pulishers.Future`~~ `Future`を戻り値にすることで独自の`Publisher`を生成することもできます。  
これは`Promise`を返すクロージャを引数に渡すことで初期化されます。  
つまり、既存のAPIからもPublisherの生成が可能です。  
~~https://developer.apple.com/documentation/combine/publishers/future/promise~~  
  
Futureは  
１つ値を出力して終了するか  
エラーで失敗するか  
のいずれかのみを結果として出力します。  
  
また  
Subscribeされていなくても  
Futureインスタンスが生成されると同時に  
出力する値の計算を実行します。  
  
そして全てのsubscriberは同じ値を受け取ります。  
  
  
2019/7/17 追記 Futureの移動に伴いPromiseもPublishersから移動していたようです。  
https://developer.apple.com/documentation/combine/future/promise  
  
## Subscribers  
  
データの受け取り側です。  
  
`Publisher`から出力された値や  
出力の最後の完了イベントを受け取ります。  
  
`Subscriber`は参照型です。  
  
`Subscriber`プロトコルは下記のように定義されています。  
  
![スクリーンショット 2019-06-08 13.04.26.png](/image/fec39145-a664-cea5-7649-f72d2504eb03.png)  
  
https://developer.apple.com/documentation/combine/subscriber  
  
3つのメソッドは下記のようになっています。  
  
### receive(subscription:)  
  
`Subscriber`が  
`Publisher`への登録が成功して  
値をリクエストすることができることを伝えます。  
  
`receive(subscription:)`  
https://developer.apple.com/documentation/combine/subscriber/3213655-receive  
  
またこの中で`subscription`の`request(_:)`を呼ぶことで  
`Publisher`へ何回の出力した値を受け取るかを伝えます。  
  
`request(_:)`  
https://developer.apple.com/documentation/combine/subscription/3213720-request  
  
引数には`Subscribers.Demand`という`struct`を渡します。  
  
`Subscribers.Demand`  
https://developer.apple.com/documentation/combine/subscribers/demand  
  
staticな変数`unlimited`と`none`  
staticなメソッド`max(Int)`  
があります。  
  
`unlimited`で無制限値を受け取り  
`none`は値を受け取らないことを意味します。  
  
`max(Int)`では具体的な数字を指定することができ  
`max(0)`と`none`は同じ意味になります。  
  
  
### receive(_ input: Self.Input) -> Subscribers.Demand  
  
実際に`Publisher`から出力された値を受け取ります。  
  
`receive(_:)`  
https://developer.apple.com/documentation/combine/subscriber/3213653-receive  
  
この際に`Subscribers.Demand`を戻り値に指定しますが  
これは現在の`Demand`に対しての**調整値**を指定します。  
  
例えば  
  
```swift

func receive(_ input: Int) -> Subscribers.Demand { 
    return .none 
}
```  
  
とするとこれは  
  
今後値を受け取らないという意味**ではなく**  
  
**現在のDemandの状態を維持する**  
  
ということを意味します。  
  
他の例を見てみると  
  
```swift

func receive(subscription: Subscription) {
    subscription.request(.max(2)) 
}

func receive(_ input: String) -> Subscribers.Demand { 
    return input == "Add" ? .max(1) : .none
}
```  
  
この場合  
最初は`max(2)`が指定されていましたが  
Addという文字列を受け取った場合は  
`max(1)`は追加されているため  
結果として`max(3)`になります。  
  
ちょっと紛らわしいので注意が必要です。  
  
さらに`max(Int)`を使用する場合は  
０以上の値を指定しなければならず  
マイナス値を設定すると`fatalError`が発生します。  
  
### receive(completion:)  
  
`Publisher`の値の出力が完了(これ以上値を出力しない)ということを  
`Subscriber`に伝えます。  
  
これは正常な場合もエラーが起きた場合もこのメソッドが呼ばれます。  
  
`receive(completion:)`  
https://developer.apple.com/documentation/combine/subscriber/3213654-receive  
  
例として  
`Subscribers`の`extension`の`Assign`クラスが紹介されていました。  
これは`Root`の中の`Input`に`Publisher`から受け取った値を適用します。  
`keypath`はパイプラインが生成された時に設定されます。  
  
https://developer.apple.com/documentation/combine/subscribers/assign  
  
```swift

extension Subscribers {
    class Assign<Root, Input>: Subscriber, Cancellable {
        typealias Failure = Never
        init(object: Root, keyPath: ReferenceWritableKeyPath<Root, Input>)
    }
}
```  
  
※ 途中でassignというメソッドが出てきてますが  
こちらもNotificationCenterの時と同様にこちらも一部脳内補完をしました。  
  
※ 2019/6/29 修正  
  
ドキュメントを見つけたので一部修正しました。  
(`Subscriber`ではなく`Publisher`のextensionでした)  
  
https://developer.apple.com/documentation/combine/publisher/3235801-assign  
  
```swift

extension Publisher where Self.Failure == Never {
    func assign<Input>(to keyPath: ReferenceWritableKeyPath<Self, Self.Output>, on object: Root) -> AnyCancellable {
        return Subscribers.Assign<Self, Input>(object: object, keyPath: keyPath)
    }
}
```  
  
例えば下記のように使用します。  
  
```swift

assign(to: \.isEnabled, on: signupButton)
```  
  
`assign`はUIKitやAppKitでも使用することが可能です。  
  
  
※ 2019/6/29追記  
  
Combineは2つの`Subscriber`を事前に定義しています。  
どちらも`Cancellable`プロトコルに適合しています。  
https://developer.apple.com/documentation/combine/cancellable  
  
### Assign  
  
上記で紹介させていただきました。  
  
### Sink  
  
`Publisher`から受け取った値を引数にしたクロージャを受け取ります。  
`Sink`を使うと独自の実装でパイプラインを完了させることができます。  
  
https://developer.apple.com/documentation/combine/subscribers/sink  
  
また`sink`というメソッドが存在します。  
`Publisher`から受け取った値を引数にしたクロージャを受け取ります。  
`sink`を使うと独自の実装をすることができます。  
  
`sink`  
https://developer.apple.com/documentation/combine/publisher/3343978-sink  
https://developer.apple.com/documentation/combine/publisher/3235806-sink  
  
※ こちらも一部脳内補完しました。  
  
```swift

extension Publisher {
    
    public func sink(receiveCompletion: ((Subscribers.Completion<Self.Failure>) -> Void)? = nil, receiveValue: @escaping ((Self.Output) -> Void)) -> Subscribers.Sink<Self> {
        return Subscribers.Sink(receiveCompletion: receiveCompletion, receiveValue: receiveValue)
    }
}

```  
  
もしCompletionを取得するクロージャを含まない場合は  
エラーを受け取ることができません。  
  
一つ注意点として  
`Subscriber`はCombineの処理の実行させます。  
そのため`sink`で処理が終わっている場合は  
その時点で`Subscriber`が生成され、`Publisher`と接続している状態になります。  
つまり繋げた時点で暗黙的にsubscribeされ  
無限のデータのリクエストを要求したことになります。  
  
```swift

let _ = remoteDataPublisher.sink { value in
  print(".sink() received \(String(describing: value))")
}
```  
  
`Publisher`が`Completion`を送信するまで  
値が更新されるたびにクロージャ内の処理が実行されます。  
エラーや`Failure`が起きたあとはクロージャは実行されません。  
  
`sink`で`Subscriber`を生成した場合は繰り返し値を受け取ります。  
失敗を処理した場合は`Completion`を受け取るクロージャを渡す必要があります。  
この時は2つのクロージャを渡します。  
  
```swift

let _ = remoteDataPublisher.sink(receiveCompletion: { completion in
    switch completion {
    case .finished:
        // ここでは値を受け取れないが完了したことを伝えることはできる
        break
    case .failure(let anError):
        // エラーを受け取ることができる
        print("エラー: ", anError)
        break
    }
}, receiveValue: { someValue in
    // 値を受け取って処理することができる
    print(".sink() received \(someValue)")
})

```  
  
`receiveCompletion`では  
`enum`の`Subscribers.Completion`の受け取ります。  
https://developer.apple.com/documentation/combine/subscribers/completion  
  
`failure`ケースでは`Error`を`associatedValue`に持ち  
エラーの原因へアクセスすることができます。   
  
  
SwiftUIではほぼ全てのコントロールが`Subscriber`です。  
`onReceive(publisher)`は`sink`に似ており  
クロージャを受け入れてSwiftUIの`＠State`や`＠Bindings`に値を設定します。  
  
https://developer.apple.com/documentation/swiftui/view/3278619-onreceive  
  
  
## PublisherとSubscriberのライフサイクル  
  
![スクリーンショット 2019-06-08 13.29.46.png](/image/a3f4eee7-7ae2-52ff-224d-b2a6a06f3e8b.png)  
  
図にある通りですが  
  
1. `Subscriber`が`Publisher`に紐づく  
2. `Subscriber`が`Publisher`から登録完了のイベントを受け取る  
3. `Subscriber`が`Publisher`へリクエストを送りイベントを受け取りが開始される  
4. `Subscriber`が`Publisher`から値を受け取る(繰り返し)  
5. `Subscriber`が`Publisher`から出力終了のイベントを受け取る  
  
  
## Operators  
  
上記で示した2つの例を使ってOperatorsの説明をしています。  
  
![スクリーンショット 2019-06-08 13.53.11.png](/image/c569f430-8f13-906b-6bcc-d57609ee5369.png)  
  
`Publisher`側では`Notification`を出力しているにも関わらず  
`Subscriber`側では`Int`が来ることを想定しているため  
型が合わずにコンパイルエラーになります。  
  
※そもそもコンパイルエラーというのは気にせずにいきましょう。  
  
![スクリーンショット 2019-06-08 13.51.19.png](/image/8a34cbcb-ad10-7183-dae8-94e3970bc48e.png)  
  
ここの隙間を埋めるのがOperatorsです。  
  
※ Operatorsは特別な名前ではなく  
複数のOperatorを表す単語として使用されています。  
  
Operatorsは`Publisher`プロトコル適合にし  
上流の`Publisher`に登録して  
出力された値を受け取って変換した結果を  
下流の`Subscriber`に送ります。  
  
Operatorsは値型です。  
  
Operatorsは`Publishers`という`enum`の中で`struct`で定義されています。  
https://developer.apple.com/documentation/combine/publishers  
  
Combineの特徴でもある  
**Compositionファースト**を実現するためにも  
Operatorsを活用した組み合わせを行っていきます。  
  
![スクリーンショット 2019-06-08 14.11.25.png](/image/c85a4277-af1d-fef1-ac76-26f3b6e13004.png)  
  
  
さらにOperatorsは  
Swiftのよく利用されるメソッドにシンタックスに合わせた  
インスタンスメソッドも定義されております。  
https://developer.apple.com/documentation/combine/publisher  
  
  
図にすると下記の様な関係で  
Intのような一つの値を扱う型のメソッドは  
Combineの`Future`という型(あとで出てきます)に同じようなメソッドがあり  
  
`Array`のような複数の値を扱う型のメソッドが  
Combineの`Publisher`型にも存在するということだそうです。  
  
![スクリーンショット 2019-06-08 14.42.53.png](/image/1e02d2c0-3552-a4c6-c72c-ada37be82fe7.png)  
  
※ 2019/6/29追記  
  
tryというprefixがついた関数はErrorを返す可能性があることを示します。  
  
例えば、`map`には`tryMap`があります。  
`map`はどんな`Output`と`Failure`の組み合わせも可能ですが  
`tryMap`の場合どんな`Input`や`Output`と`Failure`の組み合わせも許可されていますが  
エラーがthrowされた場合に出力されるFailureの型は必ず`Error`になります。  
  
https://developer.apple.com/documentation/combine/publisher/3204772-trymap  
  
  
## 非同期処理のOperators  
  
次に非同期処理時に`Publisher`を組み合わせる  
Operatorsとして`Zip`と`CombineLatest`が紹介されていました。  
  
### Zip  
  
複数の`Publisher`の結果を待ってタプルとして一つにまとめます。  
すべての`Publisher`から出力結果を受け取る必要があります。  
  
![スクリーンショット 2019-06-08 14.19.25.png](/image/8fede034-5e11-d583-af3c-6ee93b095aaf.png)  
  
`Zip`  
https://developer.apple.com/documentation/combine/publishers/zip  
  
### CombineLatest  
  
複数の`Publisher`の出力を一つの値にします。  
どれかの新しい値を受け取ったら新しい結果を出力します。  
すべての`Publisher`から出力結果を受け取る必要があります。  
最後の値を保持します。  
  
`CombineLatest`  
https://developer.apple.com/documentation/combine/publishers/combinelatest  
  
  
次に**Combine in Practice**のセッションを見ていきます。  
  
# より詳細を見る  
  
こちらのセッションでは  
具体例を交えてCombineの使い方をより詳細に見ています。  
  
例は通知で受け取った`MagicTrick`というクラスの値から  
`name`という`String`のプロパティを取り出すというような話です。  
  
受け取る値はJSON形式で  
`JSONDecoder`で`decode`できるものとします。  
  
内容は特に関係なく  
出力される型に注目していきます。  
  
※ コンパイルは通りません  
  
## Publisher  
  
最初の通知で受け取る`Output`は`Notification`です。  
失敗はないので`Failure`は`Never`です。  
  
```swift

let trickNamePublisher = NotificationCenter.default.publisher(for: .newTrickDownloaded)

// Notification Output
// Never Failure

```  
  
次に`map`で`Output`を`Data`に変換します。  
`Failure`は`Never`のままです。  
  
```swift

let trickNamePublisher = NotificationCenter.default.publisher(for: .newTrickDownloaded)
    .map { notification in
        return notification.userInfo?["data"] as! Data
    }
// Data Output
// Never Failure

```  
  
次に`Output`を`MagicTrick`クラスに変換します。  
この際に`decode`でエラーが発生する可能性があるため  
`Failure`は`Error`になります。  
  
```swift
let trickNamePublisher = NotificationCenter.default.publisher(for: .newTrickDownloaded)
    .map { notification in
        return notification.userInfo?["data"] as! Data
    }
    .tryMap { data in
        let decoder = JSONDecoder()
        try decoder.decode(MagicTrick.self, from: data)
    }
    // MagicTrick
    // Error
```  
  
ちなみに`decode`は`Publisher`の`extension`として定義されているため  
下記のようにも書けます。  
  
https://developer.apple.com/documentation/combine/publisher/3204703-decode  
  
```swift

let trickNamePublisher = NotificationCenter.default.publisher(for: .newTrickDownloaded)
    .map { notification in
        return notification.userInfo?["data"] as! Data
    }
    .decode(MagicTrick.self, JSONDecoder())
    // MagicTrick
    // Never
```  
  
ここでエラーが発生する可能性があり  
エラー処理を扱うOperatorsが紹介されています。  
  
![スクリーンショット 2019-06-08 15.32.26.png](/image/19871504-8b9b-1e50-8f74-5e6b15cd5d6b.png)  
  
`assertNoFailure()`の場合はエラーの際にFatal errorが発生します。  
https://developer.apple.com/documentation/combine/publisher/3204686-assertnofailure  
  
![スクリーンショット 2019-06-08 15.36.17.png](/image/11bd21aa-2373-d445-66f8-42e924c0a804.png)  
  
`catch`の場合は自分でエラーの処理方法を決められます。  
今回のケースですと`Just`で新たに`Publisher`を返すようにしています。  
  
`catch`  
https://developer.apple.com/documentation/combine/publisher/3204690-catch  
  
`Just`  
~~https://developer.apple.com/documentation/combine/publishers/just~~  
  
2019/7/17 追記 JustはPublishersから移動していました。  
https://developer.apple.com/documentation/combine/just  
  
![スクリーンショット 2019-06-08 15.37.35.png](/image/84d2ebf8-2414-770b-4c50-60f101a73dc6.png)  
  
```swift

let trickNamePublisher = NotificationCenter.default.publisher(for: .newTrickDownloaded)
    .map { notification in
        return notification.userInfo?["data"] as! Data
    }
    .decode(MagicTrick.self, JSONDecoder()
    .catch {
        return Just(MagicTrick.placeholder)
    }

// MagicTrick
// Never
```  
  
これまでの流れが下記のようになっています。  
  
![スクリーンショット 2019-06-08 15.44.06.png](/image/1a094a8e-65fd-d229-4dc8-5e310bc47525.png)  
  
ここで一つ問題が出てきました。  
エラーが発生した場合に`Publisher`は出力を止めてしまいます。  
  
※ 2019/6/29追記  
  
なぜかというと  
`catch`は`Publisher`を別の`Publisher`へと置き換えて返却します。  
これによって`catch`よりも上位で実行されていた処理を効率的に実行させなくすることができます。  
一方で`catch`が発生した以降は元々の`Publisher`からの出力は受け取れなくなります。  
  
  
今回のケースでは  
エラーが発生しても出力を続けて欲しいとします。  
  
その場合は`flatMap`を用います。  
  
`flatMap`  
https://developer.apple.com/documentation/combine/publisher/3204712-flatmap  
  
`FlatMap`  
https://developer.apple.com/documentation/combine/publishers/flatmap  
  
`flatMap`は受け取った全ての値を  
既存のもしくは新しい一つの`Publisher`に変換します。  
  
`flatMap`を使うことでエラーが発生してとしても  
新しい`Publisher`を生成して出力を継続することが可能になります。  
  
`data`を`Just`で包み  
新しい`Publisher`を生成しています。  
  
```swift

let trickNamePublisher = NotificationCenter.default.publisher(for: .newTrickDownloaded)
    .map { notification in
        return notification.userInfo?["data"] as! Data
    }
    .flatMap { data in
        return Just(data)
            .decode(MagicTrick.self, JSONDecoder()
            .catch {
                return Just(MagicTrick.placeholder)
            }
    }

// MagicTrick
// Never
```  
  
こうすることでエラーが発生しないままになります。  
  
ではここから`name`を取り出します。  
  
  
```swift

let trickNamePublisher = NotificationCenter.default.publisher(for: .newTrickDownloaded)
    .map { notification in
        return notification.userInfo?["data"] as! Data
    }
    .flatMap { data in
        return Just(data)
            .decode(MagicTrick.self, JSONDecoder()
            .catch {
                return Just(MagicTrick.placeholder)
            }
    }
    .publisher(for: \.name)

// String
// Never
```  
  
次に`Subscriber`が値を受け取るタイミングを指定します。  
今回の場合はメインスレッドで受け取るようにしていますが  
UIの更新などはメインスレッドで行う必要があります。  
  
  
```swift

let trickNamePublisher = NotificationCenter.default.publisher(for: .newTrickDownloaded)
    .map { notification in
        return notification.userInfo?["data"] as! Data
    }
    .flatMap { data in
        return Just(data)
            .decode(MagicTrick.self, JSONDecoder()
            .catch {
                return Just(MagicTrick.placeholder)
            }
    }
    .publisher(for: \.name)
    .receive(on: RunLoop.main)
// String
// Never
```  
  
`receive(on:options:)`  
https://developer.apple.com/documentation/combine/publisher/3204743-receive  
  
  
これはドキュメントの例の方がイメージしやすいかもしれません。  
  
`Publisher`はバックグラウンドでJSONを処理し  
`Subscriber`はメインスレッドでUIの更新を行います。  
  
```swift

let jsonPublisher = MyJSONLoaderPublisher() // Some publisher.
let labelUpdater = MyLabelUpdateSubscriber() // Some subscriber that updates the UI.

jsonPublisher
    .subscribe(on: backgroundQueue)
    .receiveOn(on: RunLoop.main)
    .subscribe(labelUpdater)
```  
  
最終的な`Publisher`の処理の流れです。  
  
![スクリーンショット 2019-06-08 16.32.56.png](/image/8738d174-876e-3c5d-d786-bf247e2a61c1.png)  
  
## Subscriber  
  
次に受け取り側を見ていきます。  
  
前述しましたが  
`Subscriber`は主に3つのものを受け取ります。  
  
- 正確に一度きりの登録完了の通知  
- 出力された値(0回以上)  
- 出力の終了またはエラーの通知  
  
### Subscriberの種類  
  
下記のようなものが挙げられていました。  
  
- keyPath  
- Sinks  
- Subjects  
- SwiftUI  
  
  
`Publisher`の例の続きを見ていきます。  
  
```swift

let trickNamePublisher = ... // Publisher of <String, Never> 

let canceller = trickNamePublisher.assign(to: \.someProperty, on: someObject)

canceller.cancel()
```   
  
`assign`は`Publisher`から受け取った値を`keypath`で定義されたオブジェクトへ渡し  
`Subscriber`を生成します。  
  
`assign(to:on:)`  
https://developer.apple.com/documentation/combine/publisher/3235801-assign  
  
注目点としては  
戻り値の`canceller`で`Cancellable`プロトコルに適合したインスタンスを返却します。  
※ Subscriberだと思ってもらえば大丈夫です。  
  
`Cancellable`  
https://developer.apple.com/documentation/combine/cancellable  
  
`cancel`メソッドを呼ぶことで  
メモリから解放することができます。  
  
このように手動で`cancel`メソッドを呼ぶこともできますが  
忘れてしまってメモリが解放されないままでいるリスクもあります。  
  
そこで`AnyCancellable`というクラスが用意されており  
`deinit`が呼ばれた際に自動で`cancel`メソッドを呼びます。  
  
`AnyCancellable`  
https://developer.apple.com/documentation/combine/anycancellable  
  
上記で記述しました`sink`も紹介されていました。  
  
```swift

let canceller = trickNamePublisher.sink { trickName in
 // Do Something with trickName
}
```  
  
## Subjects  
  
次に`Subjects`について紹介されています。  
  
https://developer.apple.com/documentation/combine/subject  
  
`Subjects`は`Publisher`と`Subscriber`の間のような存在で  
複数の`Subscriber`に値を出力できます。  
  
![スクリーンショット 2019-06-08 17.08.33.png](/image/872690e1-befd-b1ac-c02c-bd24cab09452.png)  
  
### send(_:)  
  
`Subscriber`に値を出力します。  
  
`send(_:)`  
https://developer.apple.com/documentation/combine/subject/3213648-send  
  
### send(completion:)  
  
`Subscriber`に出力終了の通知を送ります。  
  
`send(completion:)`  
https://developer.apple.com/documentation/combine/subject/3213649-send  
  
2種類の`Subject`があります。  
  
### PassthroughSubject  
  
値を保持せずに来たものをそのまま出力します。  
  
`PassthroughSubject`  
https://developer.apple.com/documentation/combine/passthroughsubject  
  
### CurrentValueSubject  
  
最後に出力した値を一つ保持しつつ値を出力します。  
後から`subscribe`した場合の初期値に利用するためです。  
  
`CurrentValueSubject`  
https://developer.apple.com/documentation/combine/currentvaluesubject  
  
  
## Subjectの例  
  
下記のように`subscribe`したり  
`send`で値の出力もできます。  
  
  
```swift

let canceller = trickNamePublisher.assign(to: \.someProperty, on: someObject)

let magicWordsSubject = PassthroughSubject<String, Never>()

trickNamePublisher.subscribe(magicWordsSubject)

let canceller = magicWordsSubject.sink { value in
    // do something with the value
}

magicWordsSubject.send("Please")
```  
  
同じ`Publisher`に`subscribe`する`Subscriber`が多い場合は  
`share`使うことで参照型に変換でき  
値の共有をすることもできます。  
  
```swift

let sharedTrickNamePublisher = trickNamePublisher.share()
```  
  
`share`  
https://developer.apple.com/documentation/combine/publisher/3204754-share  
  
## SwiftUI  
  
SwiftUIは`Subscriber`を持っているため  
利用側は`Publisher`を設定するだけで  
値の更新などを行うことができるようになっています。  
  
### BindableObjectプロトコル(deprecated)  
  
`BindableObject`は`Publisher`プロトコルに適合した型の  
`didChange`というプロパティを持っています。  
  
![スクリーンショット 2019-06-08 18.43.10.png](/image/e629b672-db2f-e62a-6ca6-c09a9055ee74.png)  
  
`BindableObject`  
https://developer.apple.com/documentation/swiftui/bindableobject  
  
下記はスライドの例です。  
※コンパイルはできません。  
  
```swift

class WizardModel : BindableObject {
    var trick: WizardTrick { didSet { didChange.send() }
    var wand: Wand? { didSet { didChange.send() }

    let didChange = PassthroughSubject<Void, Never>()
}

struct TrickView: View {
    @ObjectBinding var model: WizardModel
    var body: some View {
        Text(model.trick.name)
    }
}
```  
  
`trick`や`wand`の変更されると  
`didChange`が`send`を送ると  
それに応じてSwiftUIが`@ObjectBinding var model: WizardModel`を更新し  
`Text(model.trick.name)`に更新結果が反映されます。  
  
※SwiftUIの詳細に関しては割愛させていただきます。  
  
2019/7/19 追記   
  
Xcode11 Beta4で`BindableObject`がdidからwillに変わったようです。  
  
```swift

class WizardModel : BindableObject {
    var trick: WizardTrick { willSet { willChange.send() }
    var wand: Wand? { willSet { willChange.send() }

    let willChange = PassthroughSubject<Void, Never>()
}

struct TrickView: View {
    @ObjectBinding var model: WizardModel
    var body: some View {
        Text(model.trick.name)
    }
}
```  
  
  
### 2019/7/30 追記 ObservableObjectプロトコル  
  
Xcode11 Beta5で`BindableObject`は非推奨になり  
`ObservableObject`を使うようになったようです。  
`BindableObject`は`ObservableObject`と`Identifiable`に適合するようになっています。  
これはSwiftUIではなくCombineフレームワークに含まれます。  
ちなみに`Identifiable`はStandard Libraryに移動しました。  
  
`ObservableObject`  
https://developer.apple.com/documentation/combine/observableobject  
  
`Identifiable`  
https://developer.apple.com/documentation/swift/identifiable  
  
また、`@ObjectBinding`も`@ObservedObject`に名前が変わりました。  
`@ObjectBinding`は`@ObservedObject`のtypealiasになっています。  
  
```swift

public typealias ObjectBinding = ObservedObject
```  
  
https://developer.apple.com/documentation/swiftui/observedobject  
  
```swift
import Combine

class WizardModel : ObservableObject {
    var trick: WizardTrick { willSet { willChange.send() }
    var wand: Wand? { willSet { willChange.send() }

    let willChange = PassthroughSubject<Void, Never>()
}

struct TrickView: View {
    @ObservedObject var model: WizardModel
    var body: some View {
        Text(model.trick.name)
    }
}
```  
  
# 既存のアプリに適応する  
  
以上のように  
Combineの特徴を例を挙げて  
見てきました。  
  
最後により詳細な例を通して  
もう少し深く見ていきます。  
  
  
![スクリーンショット 2019-06-09 3.49.17.png](/image/0f861c12-f9c1-3472-285a-cc2787be67b4.png)  
  
## 内容  
  
ユーザ名とパスワードとパスワード確認用のテキストフィールドがあり  
全ての項目が妥当な値になるとユーザ登録ボタンが有効になるようにします。  
  
```swift

var password: String = ""
@IBAction func passwordChanged(_ sender: UITextField) {
    password = sender.text ?? ""
}

var passwordAgain: String = ""
@IBAction func passwordAgainChanged(_ sender: UITextField) {
    passwordAgain = sender.text ?? ""
}
```  
  
これまでのUIKitですと  
上記のような形で変更を監視して  
値に変更を反映するためには  
さらにUIViewの更新処理を自分で書く必要がありました。  
  
これを`Publisher`を使用する形へ変更します。  
  
```swift

@Published var password: String = ""
@IBAction func passwordChanged(_ sender: UITextField) {
    password = sender.text ?? ""
}

@Published var passwordAgain: String = ""
@IBAction func passwordAgainChanged(_ sender: UITextField) {
    passwordAgain = sender.text ?? ""
}
```  
  
### @Published  
  
これはSwift5.1の**Property Wrappers**という機能を使って  
`@Published`というアノテーションを適用することで  
プロパティを`Publisher`にすることができます。  
  
Property Wrappersについては下記をご参照ください。  
https://github.com/DougGregor/swift-evolution/blob/property-wrappers/proposals/0258-property-wrappers.md  
  
https://forums.swift.org/t/pitch-3-property-wrappers-formerly-known-as-property-delegates/24961  
  
~~※現在下記のようなissueがあり~~  
~~Xcode上で色々調べることができない状態にあります。~~  
  
```
The Foundation integration for the Combine framework is unavailable. 
The following Foundation and Grand Central Dispatch integrations 
with Combine are 
unavailable: KeyValueObserving, NotificationCenter, RunLoop, OperationQueue, Timer, URLSession, DispatchQueue, JSONEncoder, JSONDecoder, 
PropertyListEncoder, PropertyListDecoder, 
and the @Published property wrapper. (51241500)
```  
  
### 2019/6/18追記  
Xcode11 Beta2より補完が効くようになりました。  
  
```
The Foundation integration for the Combine framework is now available. 
The following Foundation and Grand Central Dispatch integrations 
with Combine are available: 
KeyValueObserving, NotificationCenter, RunLoop, OperationQueue, Timer, 
URLSession, DispatchQueue, JSONEncoder, JSONDecoder, 
PropertyListEncoder, PropertyListDecoder, 
and the @Published property wrapper. (51241500)
```  
  
ドキュメントもあります。  
  
`Published `  
https://developer.apple.com/documentation/combine/published  
  
`Failure`が`Never`の`Publisher`です。  
https://developer.apple.com/documentation/combine/published/failure  
  
下記のように`Publisher`としての機能が活用できます。  
  
```swift

@Published var password: String = ""
self.password = "1234" // The published value is `1234`

let currentPassword: String = self.password
let printerSubscription = $password.sink {
    print("The published value is '\($0)'") // The published value is 'password'
}
self.password = "password"
```  
  
アプリの例では  
パスワードとパスワード確認用のテキストフィールドを  
組み合わせたバリデーションが必要になります。  
  
この時に`CombineLatest`を活用します。  
  
https://developer.apple.com/documentation/combine/publishers/combinelatest  
https://developer.apple.com/documentation/combine/publisher/3204695-combinelatest  
  
  
![スクリーンショット 2019-06-09 4.19.34.png](/image/616fc237-b4be-c170-cbff-56edef67847f.png)  
  
  
```swift

@Published var password: String = ""
@Published var passwordAgain: String = ""
var validatedPassword: CombineLatest<Published<String>, Published<String>, String?> {
    return CombineLatest($password, $passwordAgain) { password, passwordAgain in
        guard password == passwordAgain, password.count > 8 else { return nil }
        return password
    }
}
```  
  
`CombineLatest`は上記で記載の通り  
`password`か`passwordAgain`が更新されると  
クロージャの処理が実行されます。  
  
また処理には続きがあり  
正しいパスワードかどうかのチェックを行います。  
  
```swift

@Published var password: String = ""
@Published var passwordAgain: String = ""
var validatedPassword:  Map<CombineLatest<Published<String>, Published<String>, String?>> {
    return CombineLatest($password, $passwordAgain) { password, passwordAgain in
        guard password == passwordAgain, password.count > 8 else { return nil }
        return password
    }
    .map { $0 == "password1" ? nil : $0 }
}
```  
  
`map`を使用していることでさらに型が変化しました。  
  
`map`  
https://developer.apple.com/documentation/combine/publisher/3204718-map  
  
`Map`  
https://developer.apple.com/documentation/combine/publishers/map  
  
  
ここで問題が発生します。  
  
このままですと他の`Publisher`と型が合わず  
処理を組み合わせることが難しいです。  
  
そこで`eraseToAnyPublisher()`を使います。  
  
```swift

@Published var password: String = ""
@Published var passwordAgain: String = ""
var validatedPassword: AnyPublisher<String?, Never> {
    return CombineLatest($password, $passwordAgain) { password, passwordAgain in
        guard password == passwordAgain, password.count > 8 else { return nil }
        return password
    }
    .map { $0 == "password1" ? nil : $0 }
    .eraseToAnyPublisher()
}
```  
  
`eraseToAnyPublisher()`  
https://developer.apple.com/documentation/combine/publisher/3241548-erasetoanypublisher  
  
`AnyPublisher`  
https://developer.apple.com/documentation/combine/anypublisher  
  
こうすることで  
他の処理とのさらなるCompositionが可能になります。  
  
下記のような処理の流れになります。  
  
![スクリーンショット 2019-06-09 4.37.04.png](/image/fbceb9c7-cf47-3248-4c32-991cdcc6f7e6.png)  
  
  
### 2019/6/29追記 処理を繋げた際の型とtype eraserについて  
  
処理を繋げると  
コンパイラは型がネストされていくように解釈します。  
そのためかなり複雑な型になります。  
  
例えば下記のようになります。  
  
```swift

let x = PassthroughSubject<String, Never>()
  .flatMap { name in
      return Future<String, Error> { promise in
          promise(.success(""))
          }.catch { _ in
              Just("No user found")
          }.map { result in
              return "\(result) foo"
          }
}
```  
  
これの型を示すと  
  
```swift
Publishers.FlatMap<Publishers.Map<Publishers.Catch<Future<String, Error>,
Just<String>>, String>, PassthroughSubject<String, Never>>
```  
  
これをそのまま使おうとするのは大変難しいです。  
  
そのためCombineではこの型の複雑を解決するために  
Publisher、Subscriber、Subjectは  
型消去(type eraser)用のメソッドを提供しています。  
  
- eraseToAnyPublisher()  
https://developer.apple.com/documentation/combine/publisher/3241548-erasetoanypublisher  
  
- eraseToAnySubscriber()  
https://developer.apple.com/documentation/combine/subscriber/3241649-erasetoanysubscriber  
  
- eraseToAnySubject()  
https://developer.apple.com/documentation/combine/subject/3241648-erasetoanysubject  
  
```swift

let x = PassthroughSubject<String, Never>()
  .flatMap { name in
      return Future<String, Error> { promise in
          promise(.success(""))
          }.catch { _ in
              Just("No user found")
          }.map { result in
              return "\(result) foo"
          }
}.eraseToAnyPublisher()
```  
  
こうすることで  
  
```swift

AnyPublisher<String, Never>
```  
  
というシンプルな型になります。  
  
  
### 2019/07/31追記 `@Published`をつけると`willChange`などの記載が不要に  
  
`@Publish`をつけたプロパティは自動で値が変更した時の通知をしてくれるようになり  
`willChange`などの繰り返し書かなければいけなかったコードを省略できるようになりました。  
  
```swift

import Combine

class WizardModel : ObservableObject {
    @Published var trick: WizardTrick = WizardTrick()
    @Published var wand: Wand? = nil
}

struct TrickView: View {
    @ObservedObject var model: WizardModel
    var body: some View {
        Text(model.trick.name)
    }
}
```  
  
  
### 非同期処理  
  
次にユーザ名について見ていきます。  
  
```swift

@Published var username: String
```  
  
ユーザ名は利用可能かをサーバに確認する必要があります。  
そのため一文字一文字に変化するたびに  
サーバに問い合わせるのはちょっと処理が重くなってしまいます。  
  
そこでサーバに問い合わせる間隔を少し長くするために  
`Debounce`を活用します。  
  
#### Debounce  
  
`Debounce`は一定の間隔後にイベントが発行するように  
処理を遅らせることができます。  
  
`debounce`  
https://developer.apple.com/documentation/combine/publisher/3204702-debounce  
  
`Debounce`  
https://developer.apple.com/documentation/combine/publishers/debounce  
  
下記のように使います。  
  
```swift

@Published var username: String
var validatedUsername: Debounce<Published<String>, Never> {
    return $username
        .debounce(for: 0.5, scheduler: RunLoop.main)
}
```  
  
上記の例は0.5秒の間はイベントが発行されません。  
さらにイベントの発行をメインスレッドで実行されるように  
`RunLoop.main`を指定しています。  
  
さらに値が変わっていないのにサーバに処理を送るのも  
効率がよくありません。  
  
そこで`RemoveDuplicates`を活用します。  
  
`removeDuplicates`  
https://developer.apple.com/documentation/combine/publisher/3204745-removeduplicates  
  
`RemoveDuplicates`  
https://developer.apple.com/documentation/combine/publishers/removeduplicates  
  
#### RemoveDuplicates  
  
これを使用するとイベントを発行しようとする際に  
前回と同じ値であった場合はイベントを発行しなくなります。  
  
```swift

@Published var username: String
var validatedUsername: RemoveDuplicates<Published<String>, Never> {
    return $username
        .debounce(for: 0.5, scheduler: RunLoop.main)
        .removeDuplicates() 
}
```  
  
先ほどと同様に型を合わせるために  
`eraseToAnyPublisher()`を使います。  
  
```swift

@Published var username: String
var validatedUsername: AnyPublisher<String?, Never> {
    return $username
        .debounce(for: 0.5, scheduler: RunLoop.main)
        .removeDuplicates() 
        .eraseToAnyPublisher()
}
```  
  
  
次にサーバへの通信を考えます。  
  
非同期処理から新しい`Publisher`を返すため  
`flatMap`を使います。  
  
```swift

@Published var username: String
var validatedUsername: <String, Never> {
    return $username
        .debounce(for: 0.5, scheduler: RunLoop.main)
        .removeDuplicates() 
        .flatMap { username in
            // 下記のような非同期処理でコールバックが返ってくることを想定
            // func usernameAvailable(_ username: String, completion: (Bool) -> Void)            
        }
}
```  
  
ここで非同期処理から`Publisher`を返すために  
`Future`を使います。  
  
`Future`  
~~https://developer.apple.com/documentation/combine/publishers/future~~  
https://developer.apple.com/documentation/combine/future  
  
  
```swift

@Published var username: String
var validatedUsername: AnyPublisher<String?, Never> {
    return $username
        .debounce(for: 0.5, scheduler: RunLoop.main)
        .removeDuplicates()
        .flatMap { username in
            return Future { promise in
                self.usernameAvailable(username) { available in
                    promise(.success(available ? username : nil))
                }
            }
        }
}
```  
  
`nil`を返すために最終的な型は  
`AnyPublisher<String?, Never>`になっています。  
  
`promise`は下記のような形になっており  
処理結果を`Result`に包んだコールバックです。  
  
```swift

(Result<Output, Failure>) -> Void
```  
  
  
これまでの流れは下記のようになります。  
  
![スクリーンショット 2019-06-09 5.16.11.png](/image/8dee4fa5-0b49-4534-8d1d-0c6e48cc44a6.png)  
  
  
最後に  
`validatedPassword`と`validatedUsername`  
を`CombineLatest`を使って  
`Create Account`ボタンの有効無効を切り替えられるようにします。  
  
```swift

var validatedCredentials: AnyPublisher<(String, String)?, Never> {
    return CombineLatest(validatedUsername, validatedPassword) { username, password in
        guard let uname = username, let pwd = password else { return nil }
        return (uname, pwd)
            // ここでチェックを行う        
        }
        .eraseToAnyPublisher()
}
```  
  
最終的には下記のように使います。  
  
```swift

@IBOutlet var signupButton: UIButton!
var signupButtonStream: AnyCancellable?
override func viewDidLoad() {
    super.viewDidLoad()
    self.signupButtonStream = self.validatedCredentials
        .map { $0 != nil }
        .receive(on: RunLoop.main)
        .assign(to: \.isEnabled, on: signupButton)
}
```  
  
`validatedCredentials`でチェックを行ったあとに  
メインスレッドで値を受け取り  
`signupButton`の`isEnabled`プロパティにその値を適用しています。  
  
  
最終的な全体の処理の流れです。  
  
![スクリーンショット 2019-06-09 5.27.19.png](/image/4beebc71-29ee-17af-c066-f66a1d585431.png)  
  
  
今回の例を通して  
  
- 小さな要素をカスタムの`Publisher`に構成する  
- `Publisher`を少しづつ適応する  
- `@Published`をプロパティに適用して`Publisher`にする  
- `Future`を使ってコールバックと`Publisher`を組み合わせる  
  
ことを見てきました。  
  
  
# Combine in URLSession  
  
他の例として  
Advances in Networking, Part 1のセッション内ではURLSessionを使った例が紹介されていました。  
こちらもCombineの機能をたくさん使用しています。  
  
Advances in Networking, Part 1  
https://developer.apple.com/videos/play/wwdc2019/712/  
  
※　セッションでも言及していましたが  
現時点ではないメソッドなども使用しており  
Combineの正式版には  
導入予定のものとして紹介されています。  
  
  
### Combineを用いた通信の流れ  
  
`Subscriber`がリクエストを送ると  
実際に`Publisher`に伝わるまでに  
後ろから順番に`Operators`が適用されます。  
  
そして通信が完了した`Publisher`が値を送り出し  
リクエストとは逆の順番に`Operators`が適用されて  
`Subscriber`にまで値が届けられます。  
  
![スクリーンショット 2019-06-14 2.54.38.png](/image/4a39861b-f474-144c-9956-0576e0f1ecbf.png)  
  
  
セッションでは検索窓にキーワードを入力して  
検索APIを呼び出す例が紹介されています。  
  
  
![スクリーンショット 2019-06-14 2.47.38.png](/image/d22d0bdb-0688-17d7-1400-a73d054bc04c.png)  
  
### Sink  
  
Subscriberの一つで  
リクエストを何回も行うことができます。  
  
`sink`(下記はFutureのものです)  
~~https://developer.apple.com/documentation/combine/publishres/future/3236194-sink~~  
https://developer.apple.com/documentation/combine/future/3333406-sink  
  
`Sink`  
https://developer.apple.com/documentation/combine/subscribers/sink  
  
### DataTaskPublisher  
  
![スクリーンショット 2019-06-14 3.13.21.png](/image/37d3daa1-313e-594a-b0cd-47452df906df.png)  
  
セッションで紹介されていましたが  
~~現時点ではまだありません。~~  
  
おそらくこういう形だろうということで  
サンプルを紹介されているものもあります。  
  
https://gist.github.com/yamoridon/16c1cc70ac46e50def4ca6695ceff772  
https://dev.classmethod.jp/smartphone/iphone/use-combine-for-http-networking/  
  
※ 2019/6/18追記  
  
ドキュメントができていました。  
  
`dataTaskPublisher(for:)`  
https://developer.apple.com/documentation/foundation/urlsession/3329708-datataskpublisher  
  
`URLSession.DataTaskPublisher`  
https://developer.apple.com/documentation/foundation/urlsession/datataskpublisher  
  
`Failure`には`URLError`という`struct`が使われています。  
https://developer.apple.com/documentation/foundation/urlerror  
  
セッションでは  
セルの中のイメージを非同期で取得する例が使われています。  
  
※ 一部省略しています。かつ一部補完もしています。  
  
```swift

class MenuItemTableView: UITableViewCell {
    
    var subscriber: AnyCancellable?
    
    override func prepareForReuse() {
        subscriber?.cancel()
    }
}

enum PubsocketError: Error {
    case invalidServerResponse
}

// ↓　UITableViewControllerの中

override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    
    var request = URLRequest(url: menuItem.heighResImageURL)
    cell.subscriber = pubSession.dataTaskPublisher(for: request)
        .tryCatch { error -> URLSession.DataTaskPublisher in
            guard error.networkUnavailableReason == .constrained else {
                throw error
            }
            return pubSession.dataTaskPublisher(for: menuItem.lowResImageURL)
        }
        .tryMap { data, response -> UIImage in
            guard let httpResponse = response as? HTTPURLResponse,
                httpResponse.statusCode == 200 else {
                    throw PubsocketError.invalidServerResponse
            }
            return image
        }
        .retry(1)
        .replaceError(with: "sample")
        .receive(on: DispatchQueue.main)
        .assign(to: \.itemImageView.image, on: cell)
}
```  
  
### TryMap  
  
`tryMap`はエラーを投げる可能性のある`map`です。  
  
`tryMap`  
https://developer.apple.com/documentation/combine/publishers/output/3210207-trymap  
`TryMap`  
https://developer.apple.com/documentation/combine/publishers/trymap  
  
  
### Retry  
  
エラーが起きた時に再通信する回数を制限します。  
  
`retry`  
~~https://developer.apple.com/documentation/combine/publishers/future/3208063-retry~~  
https://developer.apple.com/documentation/combine/future/3333402-retry  
  
`Retry`  
https://developer.apple.com/documentation/combine/publishers/retry  
  
### ReplaceError  
  
`replaceError`はエラーを指定の要素に変換します。  
  
`replaceError(with:)`  
https://developer.apple.com/documentation/combine/publishers/output/3210183-replaceerror  
`ReplaceError`  
https://developer.apple.com/documentation/combine/publishers/replaceerror  
  
`tryCatch`  
  
https://developer.apple.com/documentation/combine/publishers/trycatch  
  
### AnyCancellableの活用   
  
Combineによって  
一連の流れで処理を書くことができることも  
十分なメリットですが  
  
`AnyCancellable`を活用することで  
画面から見えなくなった非同期処理を中断することができ  
よくある異なったセルに異なった画像が表示される  
といった不具合が起きなくなっています。  
  
これまでですと`OperationQueue`を作成して  
`cancel`メソッドを呼ぶなどが必要でしたが  
それが不要になりました。  
  
# デバッグについて  
  
他のReactiveプログラミングと同様に  
エラーが発生した時に大量にスタックトレースにログが出てしまうため  
エラーの原因を見つけるのが困難な状況が多々現れます。  
  
そういう事態にも対応できるように  
Combineフレームワークでは下記のデバッグメソッドが用意されています。  
  
## print  
  
下記のイベントを受け取った時にログを出力します。  
  
- `Subscriber`に登録されたとき(subscription)  
- 値を受け取った時(value)  
- 正常に出力を完了した時(completion)  
- エラーを受け取った時(failure)  
- キャンセルされた時(cancel)  
  
```swift

enum SomeError: Swift.Error {
    case somethingWentWrong
}

let subject = PassthroughSubject<String, SomeError>()

let printSubscription = subject.print("Print").sink { _ in }
subject.send("ヤッホー!")
printSubscription.cancel()

Print: receive subscription: (PassthroughSubject)
Print: request unlimited
Print: receive value: (ヤッホー!)
Print: receive cancel

```  
  
https://developer.apple.com/documentation/combine/publishers/print?changes=latest_minor  
  
## breakpoint  
  
クロージャ内がtrueを返す場合に処理が止まります。  
  
```swift

breakpoint(receiveOutput: { (items) -> Bool in
    // アイテムが空だったら処理が止まる
    return items.isEmpty
})
```  
  
引数には何も設定しないこともできますが  
この場合は何もせずに処理は継続されます。  
  
https://developer.apple.com/documentation/combine/publishers/print/3210433-breakpoint?changes=latest_minor  
https://developer.apple.com/documentation/combine/publishers/breakpoint  
  
## breakpointOnError  
  
エラーが発生した時(PublisherのFailureを受け取った時)にメソッドの位置で処理が止まります。  
  
https://developer.apple.com/documentation/combine/publishers/breakpoint/failure  
  
# Scheduler  
  
長くなってしまったので  
  
https://github.com/stzn/qiita/blob/master/【iOS】CombineフレームワークでのSchedulerの使い方.md  
  
に記載しましたのでそちらをご参照ください。  
(こっちに転記した方が良いなどのご要望がありましたら教えてください😃)  
  
# まとめ  
  
CombineフレームワークについてWWDC2019の動画を中心に見ていきました。  
  
スライドにも出てきていましたが  
Combineの全体の構成としては  
  
- enum Publishers名前空間にある多数のstruct  
- enum Subscribers名前空間にある多数のstruct  
- Subjects  
- 90以上の共通のOperators(インスタンスメソッド)  
  
となっており  
全ての処理が`struct`で構成されている点が  
小さい部品を組み合わせていくCompositionファーストを  
強く意識しているところなのかなと感じました。  
  
また  
Swiftの標準ライブラリや広く使われてきたライブラリと  
クラスやメソッド名などが似ているものがたくさんあることで  
導入への抵抗が少なくなっていることも良いなと感じています。  
(プログラミングの世界で共通的に使用されている単語だからということもあるとは思いますが)  
  
これから使う機会は多くあるのかなと思っていますが  
どういう所にどのように活用していくか  
考えていく必要はあると思っています。  
  
今回出てこなかったものはまだまだたくさんありますし  
ドキュメントに記載のないものなど  
今後変わってくるものもあります。  
  
まだまだ未知な可能性をたくさん含んでいる  
新しいフレームワークに今後も注目していきたいですね！  
  
間違いなどございましたらご指摘頂けますと幸いです:bow_tone1:  
