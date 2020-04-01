Combine.frameworkを使用していると  
複数の`Subscriber`で同じ処理結果を共有したい場合があります。  
  
例えば  
  
- ネットワーク通信を介したデータの取得  
- 画像のダウンロード  
  
といった重い処理を繰り返し実行すると  
メモリをたくさん使用したり  
時間がかかるためユーザ体験を損なってしまう可能性もあります。  
  
Combine.frameworkでは  
そのような時に利用できるメソッドやクラスが用意されています。  
  
今回はそういったメソッドやクラスの違いについて  
見てみたいと思います。  
  
下記のサンプルコードは全てPlayground上で実行しています。  
  
# share  
  
`share`メソッドでは  
`Publishers.Share`という  
**`class`**のインスタンスが返ってきます。  
  
`Publishers.Share`  
https://developer.apple.com/documentation/combine/publisher/3204754-share  
  
この`class`は  
**前(上流)の`Publisher`**  
と  
**後(下流)の`Subscription`**  
を保持することで  
同じ`Publisher`からの値を下流の`Subscriber`へ値を流すことができます。  
  
※ `Subscription`は`Subscriber`がsubscribeした時に返ってくるものです。  
https://developer.apple.com/documentation/combine/subscription  
  
最初の`Publishers.Share`へのsubscribeで  
`Publishers.Share`が内部の`Publisher`へsubscribeし  
`Publisher`は処理を開始して値を流し始めます。  
  
そしてそれ以降のsubscribeからは  
**同じ`Publisher`**からの値の出力を受け取ることができます。  
  
それでは具体的な例として  
ネットワークリクエストの結果を  
複数の`Subscriber`で共有する例を見ていきたいと思います。  
  
```swift

var cancellables: Set<AnyCancellable> = []
let shared = URLSession.shared
    .dataTaskPublisher(for: URL(string: "https://www.google.com")!)
    .map(\.data)
    .print("shared")
    .share() // **ここでshareを呼んでいる**

print("subscribe 1回目")
shared
    .sink( receiveCompletion: { _ in },
           receiveValue: { print("subscription1 receiveValue: '\($0)'") })
    .store(in: &cancellables)
print("subscribe 2回目")
shared
    .sink( receiveCompletion: { _ in },
           receiveValue: { print("subscription2 receiveValue: '\($0)'") })
    .store(in: &cancellables)

// 出力結果
subscribe 1回目
shared: receive subscription: (DataTaskPublisher)
shared: request unlimited
subscribe 2回目
shared: receive value: (13761 bytes)
subscription1 receiveValue: '13761 bytes'
subscription2 receiveValue: '13761 bytes'
shared: receive finished

```  
  
出力結果を見てみると  
1回目のsubscribeでは  
  
```
subscribe 1回目
shared: receive subscription: (DataTaskPublisher)
shared: request unlimited
```  
と(上流の)`Publisher`へsubscribeしていますが  
  
2回目の場合  
  
```
subscribe 2回目
shared: receive value: (13761 bytes)
subscription1 receiveValue: '13761 bytes'
subscription2 receiveValue: '13761 bytes'
shared: receive finished
```  
と  
1回目のsubscribeで出力されていた  
```
shared: receive subscription: (DataTaskPublisher)
shared: request unlimited
```  
がなく  
**subscribeされていない**  
ことがわかりました。  
  
それでも2回目のsubscribeも値を受け取っています。  
  
このような結果から  
`Publisher`は一つしか存在していないことがわかります。  
  
では  
`share`がなかった場合を見てみます。  
  
```swift

var cancellables: Set<AnyCancellable> = []
let shared = URLSession.shared
    .dataTaskPublisher(for: URL(string: "https://www.google.com")!)
    .map(\.data)
    .print("shared")
    //.share() // **コメントアウト**

print("subscribe 1回目")
shared
    .sink( receiveCompletion: { _ in },
           receiveValue: { print("subscription1 receiveValue: '\($0)'") })
    .store(in: &cancellables)
print("subscribe 2回目")
shared
    .sink( receiveCompletion: { _ in },
           receiveValue: { print("subscription2 receiveValue: '\($0)'") })
    .store(in: &cancellables)

// 出力結果
subscribe 1回目
shared: receive subscription: (DataTaskPublisher)
shared: request unlimited
subscribe 2回目
shared: receive subscription: (DataTaskPublisher)
shared: request unlimited
shared: receive value: (13761 bytes)
subscription2 receiveValue: '13761 bytes'
shared: receive finished
shared: receive value: (13763 bytes)
subscription1 receiveValue: '13763 bytes'
shared: receive finished
```  
  
となり  
2回目のsubscribe時でも  
  
```
subscribe 2回目
shared: receive subscription: (DataTaskPublisher)
shared: request unlimited
```  
  
と出力されています。  
  
つまり毎回subscribeしていることがわかります。  
  
このように`share`を使うことで  
`Share`が1度subscribeをするだけで済み  
不要なsubscribeがなくなりました。  
  
ただし  
`share`には注意点もあります。  
  
`share`では  
**subscribeする前に出力された値を再度出力しません。**  
  
つまり  
参照を保持するタイミングによっては  
期待したい値が得られないかもしれません。  
  
※  
参照を保持する時点で  
`Publisher`がcompletionしていた場合は  
finished(completionイベント)のみが返ってきます。  
※ sink(receiveCompletion:)の方に値が流れてきます。  
  
下記の例を見てみます。  
  
```swift

var cancellables: Set<AnyCancellable> = []
let shared = URLSession.shared
    .dataTaskPublisher(for: URL(string: "https://www.google.com")!)
    .map(\.data)
    .print("shared")
    .share()

print("subscribe 1回目")
shared
    .sink( receiveCompletion: { _ in },
           receiveValue: { print("subscription1 receiveValue: '\($0)'") })
    .store(in: &cancellables)

DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
    print("subscribe 2回目")
    shared
        .sink(receiveCompletion: { print("subscription2 receiveCompletion \($0)")},
              receiveValue: { print("subscription2 receiveValue: '\($0)'") })
        .store(in: &cancellables)
}

// 出力結果
subscribe 1回目
shared: receive subscription: (DataTaskPublisher)
shared: request unlimited
shared: receive value: (13750 bytes)
subscription1 receiveValue: '13750 bytes'
shared: receive finished
subscribe 2回目
subscription2 receiveCompletion finished
```  
  
出力結果からもわかるように  
2回目の`subscribe`のタイミングを遅らせると  
`Publisher`はすでにcompletionしているため  
finished(completionイベント)のみを受け取っていることがわかります。  
  
  
# multicast  
  
上記の`share`ではタイミングによっては  
値を受け取れない可能性がありました。  
  
そこで  
Combineでは値の出力を能動的にコントロールできる  
`multicast`というメソッド(戻り値は`Publishers.Multicast`の**`class`**)があります。  
  
https://developer.apple.com/documentation/combine/publisher/3204734-multicast  
https://developer.apple.com/documentation/combine/publisher/3204733-multicast  
https://developer.apple.com/documentation/combine/publishers/multicast  
  
これは`ConnectablePublisher`プロトコルに適合しています。  
  
```swift

/// A publisher that uses a subject to deliver elements to multiple subscribers.
final public class Multicast<Upstream, SubjectType>
    : ConnectablePublisher
    where Upstream : Publisher, SubjectType : Subject,
Upstream.Failure == SubjectType.Failure, Upstream.Output == SubjectType.Output {
```  
  
https://developer.apple.com/documentation/combine/connectablepublisher  
  
このプロトコルは`connect`というメソッドを有しており  
`connect`を呼び出して初めて`Publisher`は`Subscriber`を受け取りを処理を開始します。  
https://developer.apple.com/documentation/combine/connectablepublisher/3204394-connect  
  
`share`の例を  
`multicast`に変えて違いを見てみます。  
  
```swift

var cancellables: Set<AnyCancellable> = []
let multicasted = URLSession.shared
    .dataTaskPublisher(for: URL(string: "https://www.google.com")!)
    .map(\.data)
    .print("shared")
    .multicast { PassthroughSubject<Data, URLError>() }

print("subscribe 1回目")
multicasted
    .sink( receiveCompletion: { _ in },
           receiveValue: { print("subscription1 receiveValue: '\($0)'") })
    .store(in: &cancellables)
print("subscribe 2回目")
multicasted
    .sink( receiveCompletion: { _ in },
           receiveValue: { print("subscription2 receiveValue: '\($0)'") })
    .store(in: &cancellables)

multicasted
    .connect()
    .store(in: &cancellables)

// 出力結果
subscribe 1回目
subscribe 2回目
shared: receive subscription: (DataTaskPublisher)
shared: request unlimited
shared: receive value: (13741 bytes)
subscription1 receiveValue: '13741 bytes'
subscription2 receiveValue: '13741 bytes'
shared: receive finished
```  
  
上記の結果では  
`share`と同じ結果を取得できました。  
  
では  
ここで`connect`をコメントアウトしてみると  
  
```swift

...

//multicasted
//    .connect()
//    .store(in: &cancellables)

// 出力結果
subscribe 1回目
subscribe 2回目
```  
  
となり  
`subscribe`してはいるものの  
`Publisher`はまだ処理を実行していません。  
  
このことから`multicast`では  
`connect`を呼ばないと  
処理を実行しないことがわかります。  
  
では  
２回目のsubscribeを少し遅らせ  
`connect`を呼び出してみた場合を今度は見てみます。  
  
```swift

var cancellables: Set<AnyCancellable> = []
let multicasted = URLSession.shared
    .dataTaskPublisher(for: URL(string: "https://www.google.com")!)
    .map(\.data)
    .print("shared")
    .multicast { PassthroughSubject<Data, URLError>() }

print("subscribe 1回目")
multicasted
    .sink( receiveCompletion: { _ in },
           receiveValue: { print("subscription1 receiveValue: '\($0)'") })
    .store(in: &cancellables)

DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
    print("subscribe 2回目")
    multicasted
        .sink( receiveCompletion: { _ in },
               receiveValue: { print("subscription2 receiveValue: '\($0)'") })
        .store(in: &cancellables)
}

DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
    print("connect")
    multicasted
        .connect()
        .store(in: &cancellables)
}

// 出力結果
subscribe 1回目
subscribe 2回目
connect
shared: receive subscription: (DataTaskPublisher)
shared: request unlimited
shared: receive value: (13745 bytes)
subscription1 receiveValue: '13745 bytes'
subscription2 receiveValue: '13745 bytes'
shared: receive finished
```  
`connect`が呼ばれる前にsubscribeしているため  
２回目のsubscribeでもデータがきちんと取得できています。  
  
これで`share`で起きていた問題は緩和できそうですが  
ちょっと書き方が複雑であったり  
`connect`を呼ぶタイミングによっては  
subscribeし損ねてしまう可能性もまだ残っています。  
  
他のライブラリでは出力された値を再度流してくれるメソッドがあります。  
  
例えば  
RxSwiftでは`shareReplay`というメソッドと利用することで  
指定したサイズ分の流れてきたデータをキャッシュすることができ  
subscribe時にその値が流れてくるようにできますが  
Combineでは現状存在しません。  
  
もし必要な場合は↓のように独自の実装が必要になります。  
https://github.com/tcldr/Entwine/blob/master/Sources/Entwine/Operators/ShareReplay.swift  
  
  
※  
補足になりますが  
`ConnectablePublisher`プロトコルには  
`autoconnect`というメソッドもありこれを呼び出した場合は  
subscribeすると自動で処理を開始して値の出力を始めます。  
  
https://developer.apple.com/documentation/combine/connectablepublisher/3235788-autoconnect  
  
# Future  
  
少し形が違いますが  
`share`や`multicast`と同様に  
処理の結果を複数の`Subscriber`で共有できます。  
  
https://developer.apple.com/documentation/combine/future  
  
他と違う特徴としては  
処理が実行されるタイミングが  
  
初めてSubscribeされたタイミング  
  
ではなく  
  
**`Future`インスタンスが生成されたタイミング**  
  
になります。  
  
下記の例を見ていきます。  
  
```swift

var cancellables: Set<AnyCancellable> = []
let future = Future<Data, URLError> { promise in
    URLSession.shared
        .dataTaskPublisher(for: URL(string: "https://www.google.com")!)
        .map(\.data)
        .print("shared")
        .sink(receiveCompletion: {
            if case .failure(let error) = $0 {
                promise(.failure(error))
            }
        }, receiveValue: {
            promise(.success($0))
        }).store(in: &cancellables)
}

// 出力結果
shared: receive subscription: (DataTaskPublisher)
shared: request unlimited
shared: receive value: (14676 bytes)
shared: receive finished
```  
  
上記のようにsubscribeしていない状態でも  
クロージャ内の処理が実行されていることがわかります。  
  
ではsubscribeしてみます。  
  
```swift

var cancellables: Set<AnyCancellable> = []
let future = Future<Data, URLError> { promise in
    URLSession.shared
        .dataTaskPublisher(for: URL(string: "https://www.google.com")!)
        .map(\.data)
        .print("shared")
        .sink(receiveCompletion: {
            if case .failure(let error) = $0 {
                promise(.failure(error))
            }
        }, receiveValue: {
            promise(.success($0))
        }).store(in: &cancellables)
}

print("subscribe 1回目")
future
    .sink(receiveCompletion: { print("subscription1 receiveCompletion: '\($0)'") },
            receiveValue: { print("subscription1 receiveValue: '\($0)'") })
    .store(in: &cancellables)

print("subscribe 2回目")
future
    .sink(receiveCompletion: { print("subscription2 receiveCompletion: '\($0)'") },
        receiveValue: { print("subscription2 receiveValue: '\($0)'") })
    .store(in: &cancellables)

// 出力結果
shared: receive subscription: (DataTaskPublisher)
shared: request unlimited
subscribe 1回目
subscribe 2回目
shared: receive value: (14676 bytes)
subscription1 receiveValue: '14676 bytes'
subscription1 receiveCompletion: 'finished'
subscription2 receiveValue: '14676 bytes'
subscription2 receiveCompletion: 'finished'
shared: receive finished
```  
  
このように`Future`内の処理は1回しか行われていませんが  
2つの`Subscriber`はどちらも値を受け取ることができています。  
  
では処理が完了した後に  
subscribeした場合はどうでしょうか？  
  
```swift

var cancellables: Set<AnyCancellable> = []
let future = Future<Data, URLError> { fulfill in
    URLSession.shared
        .dataTaskPublisher(for: URL(string: "https://www.google.com")!)
        .map(\.data)
        .print("shared")
        .sink(receiveCompletion: {
            if case .failure(let error) = $0 {
                fulfill(.failure(error))
            }
        }, receiveValue: {
            fulfill(.success($0))
        }).store(in: &cancellables)
}

print("subscribe 1回目")
future
    .sink(receiveCompletion: { print("subscription1 receiveCompletion: '\($0)'") },
            receiveValue: { print("subscription1 receiveValue: '\($0)'") })
    .store(in: &cancellables)

DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
    print("subscribe 2回目")
    future
        .sink(receiveCompletion: { print("subscription2 receiveCompletion: '\($0)'") },
            receiveValue: { print("subscription2 receiveValue: '\($0)'") })
        .store(in: &cancellables)
}

// 出力結果
shared: receive subscription: (DataTaskPublisher)
shared: request unlimited
subscribe 1回目
shared: receive value: (14676 bytes)
subscription1 receiveValue: '14676 bytes'
subscription1 receiveCompletion: 'finished'
shared: receive finished
subscribe 2回目
subscription2 receiveValue: '14676 bytes'
subscription2 receiveCompletion: 'finished'
```  
  
上記のように2回目のsubscribeでも  
値を受け取れていることがわかりました。  
  
`Future`の注意点としては  
処理が実行されるタイミングが  
`Future`インタンスが生成された時点になりますので  
subscribeした後に処理が実行されることを期待していると  
予期せぬ挙動に遭遇する可能性があります。  
  
# まとめ  
  
出力結果を共有できるメソッドやクラスについて見ていきました。  
  
メモリの使用量や通信量を抑えることができるという点で  
非常に有用なものですが  
  
いつ`Publisher`は処理を開始して値を出力し始めるのか？  
いつsubscribeすると値を受け取ることができるのか？  
  
といったことを知らないと  
「何が起きているんだ。。。？」  
と悩んでしまうような事象に出くわすかもしれません。  
  
そのような自体にならないためにも  
違いを比較して見ていくことは大切だなと感じています。  
  
Combine.frameworkはメソッドやクラスがたくさんあり  
全てをは把握することは大変ですが  
ある分類に分けて少しずつみていくと  
効率的に把握できるのかなと思います。😃  
  
もし間違いなどございましたらご指摘いただけますと助かります🙇🏻‍♂️  
