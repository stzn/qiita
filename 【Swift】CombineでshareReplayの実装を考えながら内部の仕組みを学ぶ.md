以前Publisherからの出力結果を共有するPublisherを見ていた時に  
CombineにはRxのshareReplayのような動きをするPublisherがないということがわかりました。  
  
https://github.com/stzn/qiita/blob/master/【Swift】Combineで一つのPublisherの出力結果を共有するメソッドやクラスの違い(share, multicast, Future).md  
  
そこで  
今回はいくつかの例や投稿などを参考にしながら  
shareReplayの実装の検証と  
実装をするために必要なCombineの内部の仕組みについて  
見てみたいと思います。  
  
# Combineの内部の仕組み  
  
下記の図のような流れになります。  
  
![Subscription.png](/image/6d5614c6-c13b-e361-09b4-2524abcd0369.png)  
  
ポイントとしてはPublisherからSubscriberへ  
直接値を渡している*わけではなく*  
PublisherがSubscriptionを生成し  
それを経由してSubscriberへ値を渡しているという点です。  
  
またDemandを通してSubscriberから必要な数を要求することができ  
これがBackPressure(Publisherから出力される値)を制御することができます。  
  
※  
`sink`や`assign`メソッドは自動で`.unlimited`を要求しており  
Publisherが値を出力する限り  
値を受け取り続けます。  
  
ではこれを踏まえてShareReplayの実装を検討していきます。  
  
# shareReplayに必要なもの  
  
- UpstreamのPublisherとSubscriptionを保持してSubscriberと連携するPublisher  
- 個々のSubscriberと連携するSubscription  
- PublisherとSubscritionは指定された数分流れてきた値を保持しておく  
- UpstreamのPublisherへのsubscribeは最初の一度きり。Publisherはそれを保持する。  
- それ以降はPublisherを通して同じUpstreamのPublisherの値をSubscriptionを経由してSubscriberへ出力する  
- SubscriptionがSubscriberたちとUpstreamのPublisherを持つ  
- Subscriberがsubscribe時にbufferに値があればすべて出力する  
- PublisherとSubscriptionはCompletion(完了時に出力する値)を保持しておきすでに終了している場合は完了イベントも出力する  
  
## 作成するクラス  
  
下記の2つの`class`を用意します。  
  
### ShareReplaySubscription  
  
役割  
Publisherから出力された値をSubscriberへ出力する  
  
必要なもの  
- ShareReplayPublisher(元はUpstreamのPublisher)から出力された値を受け取るSubscriber  
- Subscriberから渡されたSubscribers.Demand(どのくらい値を出力するのかの数)  
- ShareReplayPublisher(元はUpstreamのPublisher)から出力された値を保持する配列  
- ↑の配列の容量  
- ShareReplayPublisher(元はUpstreamのPublisher)から出力された完了イベント(`Subscribers.Completion`)  
  
Subscribers.Demand  
https://developer.apple.com/documentation/combine/subscribers/demand  
  
### ShareReplay(Publishersのextensionとして定義する)  
  
役割  
UpstreamのPublisherとSubscriptionを連携する  
  
必要なもの  
- UpstreamのPublisher  
- 値を流すSubscriptionの配列  
- UpstreamのPublisherから出力された値を保持する配列  
- ↑の配列の容量  
- UpstreamのPublisherから出力された完了イベント(`Subscribers.Completion`)  
  
Subscribers.Completion  
https://developer.apple.com/documentation/combine/subscribers/completion  
  
  
# Subscriptionを作成する  
  
まずはSubscriptionを作成します。  
  
  
## Subscriptionプロトコルに適合する  
   
Subscriptionプロトコルに適合するには  
requestとcancelメソッドが必要になります。  
  
```swift

private final class ShareReplaySubscription<Output, Failure: Error>: Subscription {
    func request(_ demand: Subscribers.Demand) {
    }

    func cancel() {
    }
}
```  
  
## 必要なものを定義  
  
次に上記で記載した必要なものを定義します。  
それをPublishers.ShareReplayがSubscriptionを生成する際に  
`init`で受け取れるようにします。  
  
```swift

private final class ShareReplaySubscription<Output, Failure: Error>: Subscription {
    typealias Sink = AnySubscriber<Output, Failure>

    let maxBufferSize: Int
    var buffer: [Output]
    var completion: Subscribers.Completion<Failure>?
    var demand: Subscribers.Demand = .none
    var sink: Sink?
    
    init<S>(subscriber: S,
            buffer: [Output],
            maxBufferSize: Int,
            completion: Subscribers.Completion<Failure>?)
        where S: Subscriber, Failure == S.Failure, Output == S.Input {
            self.sink = Sink(subscriber)
            self.buffer = buffer
            self.maxBufferSize = maxBufferSize
            self.completion = completion
    }

    func request(_ demand: Subscribers.Demand) {
    }

    func cancel() {
    }
}
```  
  
Subscriberプロトコルのメソッドが呼び出せれば良いため  
便利な`AnySubscriber`としてSubscriberを保持します。  
https://developer.apple.com/documentation/combine/subscriber  
  
## requestとcancelを実装する  
   
requestとcancelではどんな処理が必要になるか考えます。  
  
requestではDemandが  
引数で渡ってきたDemandを自身のDemandに追加します。  
  
この後にSubscriptionは値を出力し始めるため  
出力済みのbufferやCompletionイベントを出力します。  
  
参考  
https://github.com/tcldr/Entwine/blob/master/Sources/Common/Utilities/SinkQueue.swift#L63  
  
```swift

func request(_ demand: Subscribers.Demand) {
    self.demand += demand

    processDemand()
}

private func processDemand() {
    guard let sink = sink else { 
        return 
    }

    // SubscriberのDemandがnoneになるまでbufferに溜まっている値を出力する
    while demand > .none && !buffer.isEmpty {
        // 1つの出力分Demandを引く ※
        demand -= 1

        // bufferの値を一つsubscriberに渡す
        let nextDemand = sink.receive(buffer.removeFirst())
        // 戻り値の追加のDemandを追加する
        demand += nextDemand
    }

    // すでに完了していた場合はcompletionを出力する
    if let completion = completion {
        expediteCompletion(completion)
    }
}

private func expediteCompletion(_ completion: Subscribers.Completion<Failure>) {
    // subscriberがnilということはすでに完了していることを示している
    guard let sink = sink else { 
        return 
    }

    // completionを2回出力しないようにnilにする
    self.sink = nil
    self.completion = nil
    buffer.removeAll()

    sink.receive(completion: completion)
}
```  
  
※  
ここが一般的な足し算や引き算とは結果が異なるのでややこしいです。  
例えばDemandが`.unlimited`だった場合はいくら引いても`.unlimited`です。  
  
詳しくは  
下記のドキュメントの各オペレータをご参照ください、  
https://developer.apple.com/documentation/combine/subscribers/demand  
  
次にcancelメソッドを定義します。  
これはSubscribers.Demandの`.finished`を流します。  
  
```swift

func cancel() {
    expediteCompletion(.finished)
}
```  
  
次にPublisherから流れてくる値を受け取りSubscriberへ渡すメソッドを考えます。  
  
下記の実装を参考にしています。(名前はPublisherに合わせています。)  
https://github.com/tcldr/Entwine/blob/master/Sources/Common/Utilities/SinkQueue.swift#L51  
https://github.com/tcldr/Entwine/blob/master/Sources/Common/Utilities/SinkQueue.swift#L63  
  
```swift

func receive(_ input: Output) {
    guard completion == nil, sink != nil else { 
        return 
    }

    // 将来のsubscriberのために出力された値をbufferに入れる
    buffer.append(input)

    // maxBufferSizeを超えた場合は一番古い値を取り除く
    if buffer.count > maxBufferSize {
        buffer.removeFirst()
    }
    // processDemandの中ではbufferに値が一つ入っている状態なので
    // 受け取った値をそのままSubscriberへ出力する
    processDemand()
}

func receive(completion: Subscribers.Completion<Failure>) {
    guard self.completion == nil, let sink = sink else {
        return 
    }

    expediteCompletion(completion)
}
```  
  
これでSubscriptionの準備ができました。  
  
# Publisherを作成する  
  
ではShareReplyを定義していきます。  
  
まずはPublisherプロトコルに適合させるのに必要な要素を見てみます。  
  
```swift

extension Publishers {
    final class ShareReplay<Upstream: Publisher>: Publisher {
        typealias Output = Upstream.Output
        typealias Failure = Upstream.Failure

        func receive<S>(subscriber: S) where S : Subscriber {
        }
    }
}
```  
OutputとFailureは  
Upstreamの型をそのまま使用します。  
  
次に必要な要素を定義していきます。  
  
```swift

extension Publishers {
    final class ShareReplay<Upstream: Publisher>: Publisher {
        typealias Output = Upstream.Output
        typealias Failure = Upstream.Failure

        private let maxBufferSize: Int
        private let upstream: Upstream
        private let lock = UnfairLock() // ※
        private var buffer: [Output] = []
        private var completion: Subscribers.Completion<Failure>?
        private var subscriptions: [ShareReplaySubscription<Output, Failure>] = []

        init(upstream: Upstream, maxBufferSize: Int) {
            self.upstream = upstream
            self.maxBufferSize = maxBufferSize
        }

        func receive<S>(subscriber: S) where S : Subscriber {
        }
    }
}
```  
  
※   
Lock用のカスタムクラスです。  
`lock.around`で囲んだ中をLockします。  
一番最後にコードを記載しています。  
ここはいくつかのソースを見ていく中で`os_unfair_lock`を使用したのですが  
Serialな`DispatchQueue`でも良いのかもしれません🤔  
  
ではreceiveメソッドについて考えてみます。  
冒頭の図でも示したように  
Subscriptionを作成し  
以降はこれを経由してSubscriberへ値を出力していきます。  
  
  
```swift

func receive<S: Subscriber>(subscriber: S)
    where Failure == S.Failure, Output == S.Input {
        lock.around {
            let subscription = ShareReplaySubscription(
                subscriber: subscriber,
                buffer: buffer,
                maxBufferSize: maxBufferSize,
                completion: completion)
            subscriptions.append(subscription)
            subscriber.receive(subscription: subscription)

            guard subscriptions.count == 1 else {
                return
            }

            let sub = AnySubscriber(
                receiveSubscription: { subscription in
                    // Upstramから出力される値をずっと流し続ける
                    subscription.request(.unlimited)
            },
                receiveValue: { [weak self] (value: Output) -> Subscribers.Demand in
                    self?.send(value)
                    // Demandの追加はしない
                    return .none
                },
                receiveCompletion: { [weak self] in
                    self?.send($0)
                }
            )
            upstream.subscribe(sub)
        }
}

// Upstreamから受け取った値をSubscriptionへ渡す
private func send(_ value: Output) {
    lock.around {
        // すでに出力が完了している場合は値を出力しない
        guard completion == nil else {
            return
        }

        buffer.append(value)
　　    // maxBufferSizeを超えた場合は一番古い値を取り除く
        if buffer.count > maxBufferSize {
            buffer.removeFirst()
        }

        subscriptions.forEach {
            _ = $0.receive(value)
        }
    }
}

// Upstreamから受け取ったSubscribers.CompletionをSubscriptionへ渡す
private func send(_ completion: Subscribers.Completion<Failure>) {
    lock.around {
        // 将来のsubscriberのために出力されたCompletionを保持
        // CompletionはSubscribe時に出力される
        self.completion = completion
        subscriptions.forEach {
            _ = $0.receive(completion: completion)
        }
    }
}
```  
  
これで完成です。  
  
# extensionのメソッドを定義する  
  
最後にアクセスしやすいようにメソッドを追加します。  
  
```swift

extension Publisher {
    func shareReplay(_ maxBufferSize: Int = .max) -> Publishers.ShareReplay<Self> {
        return Publishers.ShareReplay(upstream: self, maxBufferSize: maxBufferSize)
    }
}
```  
  
# 実験してみる  
  
では実験してみます。  
  
```swift

var cancellables: Set<AnyCancellable> = []

let subject = PassthroughSubject<String,Never>()

let publisher = subject
    .print("ShareReplay")
    .shareReplay(2)

subject.send("a")

print("Subscribe1回目")
publisher.sink(
    receiveCompletion: {
        print("Subscribe1回目 receiveCompletion: \($0)")
},
    receiveValue: {
        print("Subscribe1回目 receiveValue \($0)")
}
).store(in: &cancellables)

subject.send("b")
subject.send("c")
subject.send("d")

print("Subscribe2回目")
publisher.sink(
    receiveCompletion: {
        print("Subscribe2回目 receiveCompletion: \($0)")
},
    receiveValue: {
        print("Subscribe2回目 receiveValue \($0)")
}
).store(in: &cancellables)

subject.send("e")
subject.send("f")
subject.send(completion: .finished)

DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
    print("Upstreamが完了した後にSubscribe")
    print("Subscribe3回目")
    publisher.sink(
        receiveCompletion: {
            print("Subscribe3回目 receiveCompletion: \($0)")
    },
        receiveValue: {
            print("Subscribe3回目 receiveValue \($0)")
    }
    ).store(in: &cancellables)
}
```  
  
shareReplayに2を指定しているので  
2つの値を保持し  
後でsubscribeした際は最初に2つの値が流れてきます。  
  
## 出力結果  
  
全体の結果です。  
  
```
Subscribe1回目
ShareReplay: receive subscription: (PassthroughSubject)
ShareReplay: request unlimited
ShareReplay: receive value: (b)
Subscribe1回目 receiveValue b
ShareReplay: receive value: (c)
Subscribe1回目 receiveValue c
ShareReplay: receive value: (d)
Subscribe1回目 receiveValue d
Subscribe2回目
Subscribe2回目 receiveValue c
Subscribe2回目 receiveValue d
ShareReplay: receive value: (e)
Subscribe1回目 receiveValue e
Subscribe2回目 receiveValue e
ShareReplay: receive value: (f)
Subscribe1回目 receiveValue f
Subscribe2回目 receiveValue f
ShareReplay: receive finished
Subscribe1回目 receiveCompletion: finished
Subscribe2回目 receiveCompletion: finished
Upstreamが完了した後にSubscribe
Subscribe3回目
Subscribe3回目 receiveValue e
Subscribe3回目 receiveValue f
Subscribe3回目 receiveCompletion: finished
```  
  
## 1回目  
  
まず1回目だけに注目してみます。  
  
```
Subscribe1回目
ShareReplay: receive subscription: (PassthroughSubject)
ShareReplay: request unlimited
ShareReplay: receive value: (b)
Subscribe1回目 receiveValue b
ShareReplay: receive value: (c)
Subscribe1回目 receiveValue c
ShareReplay: receive value: (d)
Subscribe1回目 receiveValue d
ShareReplay: receive value: (e)
Subscribe1回目 receiveValue e
ShareReplay: receive value: (f)
Subscribe1回目 receiveValue f
ShareReplay: receive finished
Subscribe1回目 receiveCompletion: finished
```  
  
最初に出力しているaは  
ShareReplayへまだ誰もsubscribeしていないため  
値を受け取っていません。  
  
Subscribe1回目の後に  
ShareReplayはSubscriptionを  
Upstream(今回はPassthroughSubject<String,Never>)から受け取ります。  
  
そしてその後にSubscritionを受け取る出力は起きていないため  
1回しかUpstreamへsubscribeしていないことがわかります。  
  
あとはShareReplayから出力された値を全て受け取っています。  
  
## 2回目  
  
```
Subscribe2回目
Subscribe2回目 receiveValue c
Subscribe2回目 receiveValue d
ShareReplay: receive value: (e)
Subscribe2回目 receiveValue e
ShareReplay: receive value: (f)
Subscribe2回目 receiveValue f
ShareReplay: receive finished
Subscribe2回目 receiveCompletion: finished
```  
  
Subscribe2回目の前にabcdという4つの値が流れており  
その内の最後の2つの値(今回のmaxBufferSize)をsubscribe後に受け取っています。  
  
## 3回目  
  
```
Upstreamが完了した後にSubscribe
Subscribe3回目
Subscribe3回目 receiveValue e
Subscribe3回目 receiveValue f
Subscribe3回目 receiveCompletion: finished
```  
  
Subscribe3回目の前にUpstreamが出力を完了しています。  
  
そのため最後の2つの値(今回のmaxBufferSize)に加えて完了イベントを  
subscribe後に受け取っています。  
  
# まとめ  
  
SharePlayの実装を考えながら  
Combineの仕組みの理解しようと試みました。  
  
個人的な意見ですが  
sinkは`request(Subscribers.Demand)`を内部で自動で行っているので  
`request(Subscribers.Demand)`を理解するのが難しいなと感じました。  
  
あまり意識することはないかもしれませんが  
内部仕組みを知ることで  
デバッグ時の問題の早期発見などに役立つかもしれません😃  
  
もし何か間違いなどございましたら  
ご指摘頂けますと幸いです🙇🏻‍♂️  
  
# 参考  
  
https://github.com/ReactiveX/RxSwift/blob/master/RxSwift/Observables/ShareReplayScope.swift  
https://github.com/ReactiveX/RxSwift/blob/master/RxSwift/Subjects/ReplaySubject.swift  
  
https://github.com/tcldr/Entwine/blob/master/Sources/Common/Utilities/SinkQueue.swift  
https://github.com/tcldr/Entwine/blob/master/Sources/Entwine/Operators/ReplaySubject.swift  
  
https://github.com/broadwaylamb/OpenCombine/blob/master/Sources/OpenCombine/Helpers/Locking.swift  
https://github.com/Alamofire/Alamofire/blob/master/Source/Protector.swift#L30  
https://github.com/broadwaylamb/OpenCombine/blob/master/Sources/OpenCombine/PassthroughSubject.swift  
https://forums.swift.org/t/thread-safety-for-combine-publishers/29491/16  
  
# 今回使用したコード  
  
```swift

import Foundation
import Combine

private final class ShareReplaySubscription<Output, Failure: Error>: Subscription {
    typealias Sink = AnySubscriber<Output, Failure>

    let maxBufferSize: Int
    var buffer: [Output]
    var completion: Subscribers.Completion<Failure>?
    var demand: Subscribers.Demand = .none
    var sink: Sink?

    init<S>(subscriber: S,
            buffer: [Output],
            maxBufferSize: Int,
            completion: Subscribers.Completion<Failure>?)
        where S: Subscriber,
        Failure == S.Failure,
        Output == S.Input {
            self.sink = Sink(subscriber)
            self.buffer = buffer
            self.maxBufferSize = maxBufferSize
            self.completion = completion
    }


    func request(_ demand: Subscribers.Demand) {
        self.demand += demand
        processDemand()
    }

    func receive(_ input: Output) {
        guard completion == nil, sink != nil else {
            return
        }

        buffer.append(input)
        if buffer.count > maxBufferSize {
            buffer.removeFirst()
        }

        processDemand()
    }

    func receive(completion: Subscribers.Completion<Failure>) {
        guard self.completion == nil, let sink = sink else {
            return
        }

        expediteCompletion(completion)
    }

    func cancel() {
        expediteCompletion(.finished)
    }

    private func expediteCompletion(_ completion: Subscribers.Completion<Failure>) {
        guard let sink = sink else { 
            return
        }

        self.sink = nil
        self.completion = nil
        buffer.removeAll()

        sink.receive(completion: completion)
    }

    private func processDemand() {
        guard let sink = sink else {
            return
        }

        while demand > .none && !buffer.isEmpty {
            demand -= 1

            let nextDemand = sink.receive(buffer.removeFirst())
            demand += nextDemand
        }

        if let completion = completion {
            expediteCompletion(completion)
        }
    }
}

// Lock用のクラス
// https://github.com/Alamofire/Alamofire/blob/master/Source/Protector.swift#L30
final class UnfairLock {
    private let unfairLock: os_unfair_lock_t

    init() {
        unfairLock = .allocate(capacity: 1)
        unfairLock.initialize(to: os_unfair_lock())
    }

    deinit {
        unfairLock.deinitialize(count: 1)
        unfairLock.deallocate()
    }

    private func lock() {
        os_unfair_lock_lock(unfairLock)
    }

    private func unlock() {
        os_unfair_lock_unlock(unfairLock)
    }

    /// Executes a closure returning a value while acquiring the lock.
    ///
    /// - Parameter closure: The closure to run.
    ///
    /// - Returns:           The value the closure generated.
    func around<T>(_ closure: () -> T) -> T {
        lock(); defer { unlock() }
        return closure()
    }

    /// Execute a closure while acquiring the lock.
    ///
    /// - Parameter closure: The closure to run.
    func around(_ closure: () -> Void) {
        lock(); defer { unlock() }
        return closure()
    }
}
```  
