WWDC2019で発表されたApple製のCombineフレームワークは  
非同期なデータの流れを簡単に扱え  
多くの場面で活用できると期待されているフレームワークです。  
  
詳しくはこちらに記載していますので  
よろしければご参照ください。  
https://github.com/stzn/qiita/blob/master/【iOS】Combineフレームワークまとめ.md  
  
2019/7/5現在では  
`NotificationCenter`や`URLSession`などの`extension`で  
`Publisher`に適合する実装が用意されていますが  
UIKitを拡張したものは用意されていません。  
  
そこで  
今回はUIKitのイベントが発生した際にデータを送る  
Custom Publisherの作り方を見ていきたいと思います。  
  
また  
その中でCombineフレームワークが  
どういう仕組みで動いているのかについても  
改めて見ていきたいと思います。  
  
主に下記の記事を参考にしています。  
https://www.avanderlee.com/swift/custom-combine-publisher/  
  
# UIKitのイベントに応答するには？  
  
`UIControl`のイベントが発生した時に  
データを`Publisher`から送るようにします。  
  
# 必要なもの  
  
大きく分けて2つのものが必要になります。  
  
1. Custom Subscriptionを作成する  
2. Custom Publisherを作成する  
  
## Custom Subscriptionを作成する  
  
まずは下記のように`UIControl`を拡張します。  
  
```swift

extension UIControl {
        
    final class Subscription<SubscriberType: Subscriber, Control: UIControl>: Combine.Subscription where SubscriberType.Input == Control {
        private var subscriber: SubscriberType?
        private let control: Control
        
        init(subscriber: SubscriberType, control: Control, event: UIControl.Event) {
            self.subscriber = subscriber
            self.control = control
            control.addTarget(self, action: #selector(eventHandler), for: event)
        }
        
        func request(_ demand: Subscribers.Demand) {
        }
        
        func cancel() {
            subscriber = nil
        }
        
        @objc private func eventHandler() {
            _ = subscriber?.receive(control)
        }
        
        deinit {
            print("UIControlTarget deinit")
        }
    }
}
```  
  
上記のコードを見ていきます。  
  
### Protocolへの適合  
  
まず`Subscription` Protocolに適合させる必要があります。  
  
`Subscription`は`Publisher`と`Subscriber`を繋ぐ存在で  
`Publisher`と`Subscriber`の接続状態を表します。  
  
`Subscription` Protocolは`Cancellable` Protocolを継承しており  
`cancel`メソッドの実装が必須です。  
`Publisher`からデータを受け取ることをキャンセルした時の処理を実装する必要があります。  
  
さらに`request`メソッドも必須です。  
https://developer.apple.com/documentation/combine/subscription/3213720-request  
  
上記の例では何も行っていないため  
来たデータをそのまま流しているだけですが  
ここで流すデータに条件を追加したりすることもできます。  
  
さらに引数で指定する`Demand`の値で  
`Publisher`へ要求するデータ量をコントロールすることができます。  
  
`Demand`は２つのcaseを持つ`enum`で  
  
- max(Int) 指定した回数を超えたらデータを受け取ることを止める  
- unlimited 無制限にデータを受け取る `sink`メソッドはunlimitedになる  
  
https://developer.apple.com/documentation/combine/subscribers/demand  
  
### `Subscriber`のInputを制約する  
  
次に注目する点としてはクラスの制約で  
  
```swift

class Subscription<SubscriberType: Subscriber, Control: UIControl>
    : Combine.Subscription where SubscriberType.Input == Control
```  
  
とすることで  
`Subscriber`が受け取る値  
つまり`Publisher`が出力する値を`UIControl`を継承したものに限定しています。  
  
### イベントを受け取る  
  
`Subscription`は`Subscriber`とイベントを発生させる`UIControl`を保持しており  
  
- `init`時に`addTarget`でイベントハンドラーを登録  
- イベントハンドラーで保持する`Subscriber`へデータを流す  
  
ことを行っています。  
  
`Subscriber`は下記の3つのメソッドでデータを受け取ることができます。  
  
#### `receive(subscription: Subscription)`  
  
`Subscriber`が`Publisher`と接続に成功し  
データを要求することができるようになった通知を受け取ります。  
  
#### `receive(Self.Input) -> Subscribers.Demand`  
  
`Publisher`が出力した値を受け取ります。  
  
#### `receive(completion: Subscribers.Completion<Self.Failure>)`  
  
`Publisher`がデータを出力するのを完了したことを受け取ります。  
正常に終了した場合もエラーが起きて終了した場合もあります。  
  
### cancelされた時  
  
保持する`Subscriber`への参照をnilにすることで  
メモリから解放されるようにしています。  
  
  
## Custom Publisherを作成する  
  
次に`Publisher`を作成します。  
  
```swift

extension UIControl {
    
    struct Publisher<Control: UIControl>: Combine.Publisher {
        
        typealias Output = Control
        typealias Failure = Never
        
        let control: Control
        let controlEvents: UIControl.Event
        
        init(control: Control, events: UIControl.Event) {
            self.control = control
            self.controlEvents = events
        }
        
        func receive<S>(subscriber: S) where S : Subscriber, S.Failure == Failure, S.Input == Output {
            let subscription = UIControl.Subscription(subscriber: subscriber, control: control, event: controlEvents)
            subscriber.receive(subscription: subscription)
        }
    }    
}
```  
  
先ほども少し触れましたが  
`Publisher`の`Output`は`UIControl`を継承したクラスになります。  
`init`時に`Control`と出力する`UIControl.Event`を保持します。  
  
また  
`Publisher` Protocolに適合させる必要があります。  
そのためには`receive<S>(subscriber: S)`メソッドを実装します。  
  
この中で  
先ほど作成したCustom Subscriptionのインスタンスを生成し  
引数の`Subscriber`の`receive(subscription: Subscription)`メソッドに渡します。  
  
やることはこれだけです。  
  
## もっと簡単にPublisherを作成できるようにする  
  
`NotificationCenter`などと似た形で`UIControl.Publisher`を作成できるようにします。  
  
```swift

protocol CombineCompatible { }
extension UIControl: CombineCompatible { }
extension CombineCompatible where Self: UIControl {
    func publisher(for events: UIControl.Event) -> UIControl.Publisher<Self> {
        return UIControl.Publisher(control: self, events: events)
    }
}
```  
  
`CombineCompatible`という空のProtocolを使用することで  
`UIControl`以外にも拡張しやすくしたり  
テスト時のモックを作りやすくしています。  
  
これを使って実装例を見てみます。  
  
## 実装例  
  
参考にした記事に出てきた例で単純にボタンをタップしたときの挙動を見てみます。  
  
```swift

let button = UIButton()
button.publisher(for: .touchUpInside).sink { button in
    print("Button is pressed!")
}
button.sendActions(for: .touchUpInside) // Button is pressed!
```  
  
`sendActions`を呼び出すとprintの内容が出力されます。  
  
## UISwitchのイベントに応答する  
  
`UISwitch`の`isOn`プロパティは  
KVOに対応していないため  
`Publisher`で変化をキャッチできるようにしてみます。  
  
  
```swift

extension CombineCompatible where Self: UISwitch {
    var isOnPublisher: AnyPublisher<Bool, Never> {
        return publisher(for: [.allEditingEvents, .valueChanged]).map { $0.isOn }.eraseToAnyPublisher()
    }
}

```  
  
※   
これはUIのイベントには応答しますが  
プログラム上で`isOn`プロパティを変更しても反応しません。  
  
  
# まとめ  
  
Custom Publisherの作成を通して  
Combineフレームワークの動作を確認してみました。  
  
Combineフレームワークは  
応用できる幅がたくさんあると思いますので  
今後も色々な使い方を見ていきたいと思います。  
  
  
Custom Publisherを活用しているライブラリには  
下記のようなものもありますので  
ぜひ参考にしてみてください😄  
  
  
### @noppefoxwolf さんのCombinative  
https://github.com/noppefoxwolf/Combinative  
  
### iOS13より下のバージョンやLinux, WindowsでもCombineを使用可能にしようと試みているOpenCombine  
https://github.com/broadwaylamb/OpenCombine  
  
間違いなどございましたらご指摘いただけると嬉しいです🙇🏻‍♂️  
