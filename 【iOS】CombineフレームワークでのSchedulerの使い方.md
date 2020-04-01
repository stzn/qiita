Combine.framework(以下Combineと略)は非同期の処理を扱うため  
どのスレッド(主にメインスレッドかそれ以外か)で処理を実行するのかが大切です。  
  
全てをメインスレッドで実行すれば  
画面が固まって  
ユーザの操作を阻害してしまうため  
アプリが使われなくなってしまう要因の一つにもなってしまいます。  
  
今回はCombineとスレッドの関係を管理するための  
Schedulerの基本や動作について学んだことを書きます。  
  
主に下記の記事を参考にしました。  
https://www.vadimbulavin.com/understanding-schedulers-in-swift-combine-framework/  
  
  
# CombineでののSchedulerの役割  
  
SchedulerはCombineが  
**「いつ」**  
**「どこで」**  
機能するかを決めます。  
  
## 「いつ」  
  
アプリが起動しているOSの現在時刻に依存せず  
Schedulerが持つ仮想時間の中で実行されるという意味です。  
  
例えば  
DispatchQueueはDispatchTimeを使用します。  
  
```swift

@available(OSX 10.15, iOS 13.0, tvOS 13.0, watchOS 6.0, *)
extension DispatchQueue : Scheduler {

    /// The scheduler time type used by the dispatch queue.
    public struct SchedulerTimeType : Strideable, Codable, Hashable {
        ...
        /// The dispatch time represented by this type.
        public var dispatchTime: DispatchTime
    }

    public struct Stride : SchedulerTimeIntervalConvertible, ... {
        ...        
    }

    /// Returns the minimum tolerance allowed by the scheduler.
    public var minimumTolerance: DispatchQueue.SchedulerTimeType.Stride { get }

    /// Returns this scheduler's definition of the current moment in time.
    public var now: DispatchQueue.SchedulerTimeType { get }

    ...
}
```  
  
## 「どこで」  
現在のRunLoopやDispatchQueue、OperationQueueといったどのスレッドで実行されるのか  
を決めます。  
  
  
https://developer.apple.com/documentation/combine/scheduler  
  
  
# CombineのSchedulerの種類  
  
4つの種類がありますが  
全て`Scheduler`プロトコルに適合しています。  
  
https://developer.apple.com/documentation/combine/scheduler  
  
  
## DispatchQueue  
  
特定のキューで実行します。  
自分でserial, concurrentとして定義したり  
Dispatch.mainやDispatch.globalなど  
事前に定義されたものを使用することができます。  
  
serialやglobalはバックグラウンドキューとして  
mainはUIに関連したメインスレッドで何かを行うために使用されることが多くあります。  
  
  
## OperationQueue  
  
DispatchQueueに似ていますが  
cancelなどが可能になります。  
mainはUIに関連したメインスレッドで何かを行うために使用され  
それ以外はバックグラウンドで動きます。  
  
https://developer.apple.com/documentation/foundation/operationqueue  
  
## RunLoop  
  
マウスやキーボードの入力イベントやTimerのイベントを処理します。  
  
https://developer.apple.com/documentation/foundation/runloop  
  
RunLoop  
https://developer.apple.com/documentation/foundation/runloop  
  
## ImmediateScheduler  
  
同期的に実行するアクションを即時に実行します。  
  
https://developer.apple.com/documentation/combine/immediatescheduler  
  
## 既存のクラスとScheduler  
  
CombineではImmediateScheduler以外に新しいSchedulerを導入せず  
上記でも紹介しているように  
既存のDispatchQueueなどを拡張しています。  
そのため上記のQueueなどはCombine以外とも一緒に利用できます。  
  
  
# Combineのデフォルトの動作  
  
もしSchedulerを特定しない場合  
Combineは要素が生成されたスレッド上で動きます。  
  
下記の例で考えてみます。  
  
```swift

let subject = PassthroughSubject<Int, Never>()
// 1
let token = subject.sink(receiveValue: { value in
    print(Thread.isMainThread)
})
// 2
subject.send(1)
// 3
DispatchQueue.global().async {
    subject.send(2)
}
```  
  
メインスレッドかどうかをprintしています。  
出力結果は  
  
```swift

true // 2の結果
false // 3の結果
```  
  
となります。  
つまりメインスレッドで生成された要素はメインスレッドに流れ  
バックグラウンドで生成された要素はバックグラウンドに流れてきます。  
  
# Schedulerの動きを確認する  
  
※ 例はすべてPlaygroundで実行しています。  
  
多くのCombineを使用するケースとして  
特定のリソースをバックグラウンドで取得し  
メインスレッドでUIに反映する  
があります。  
  
Combineフレームワークでは  
`receive(on:)`と`subscribe(on:)`を利用して  
これをコントロールします。  
  
## receive  
  
このメソッドが定義された**後**の処理を  
定義したスレッドで実行するようにします。  
  
下記の例を考えてみます。  
  
```swift

Just(1)
    .map { _ in print(Thread.isMainThread) } // 1
    .receive(on: DispatchQueue.global()) // 2
    .map { print(Thread.isMainThread) } // 3
    .sink { print(Thread.isMainThread) } // 4
```  
  
出力結果は  
  
```swift

true
false
false
```  
となります。  
  
順番に考えていくと  
  
1. メインスレッドで呼ばれているためtrue  
2. バックグランドキューへ切り替え  
3. バックグラウンドキューで実行されるためfalse  
4. バックグラウンドキューで実行されるためfalse  
  
という動きをしていることが確認できました。  
  
https://developer.apple.com/documentation/combine/publishers/receiveon/3210587-receive  
  
## subscribe  
  
receiveの反対で定義された**前**の処理を指定します。  
具体的にはsubscribeとcancelとrequestが実行されるスレッドを指定します。  
  
`receive`でSchedulerが指定されるまで全ての処理は  
subscribeで指定したSchedulerのスレッド上で実行されます。  
  
下記の例を考えてみます。  
  
```swift

Just(1)
   .subscribe(on: DispatchQueue.global())
   .map { _ in print(Thread.isMainThread) }
   .sink { print(Thread.isMainThread) }
```  
  
出力結果は  
  
```swift

false
false
```  
  
になります。  
  
Justがバックグラウンドキューから要素を流していることが確認できました。  
  
これの順番を変更すると  
  
```swift

Just(1)
    .map { _ in print(Thread.isMainThread) } // 1
    .subscribe(on: DispatchQueue.global())
    .sink { print(Thread.isMainThread) } // 2

```  
  
出力結果は  
  
```swift

true
false
```  
  
になります。  
  
これは1の時点ではメインスレッドで要素を流していたJustが  
subscribeでスレッドが切り替えられ  
2ではバックグランドから要素を流すようになりました。  
  
https://developer.apple.com/documentation/combine/publishers/handleevents/3208173-subscribe  
  
# 非同期処理の例  
  
上記でも少し言及しましたが  
Combineの使用例としてデータを非同期で取得してUIに反映するという  
処理が考えられます。  
  
これを下記の例から考えてみます。  
  
```swift

struct SomePublisher: Publisher {
    typealias Output = Int
    typealias Failure = Never
    
    func receive<S>(subscriber: S) where S : Subscriber, Failure == S.Failure, Output == S.Input {
        sleep(10)
        subscriber.receive(subscription: Subscriptions.empty)
        _ = subscriber.receive(1)
        subscriber.receive(completion: .finished)
    }
}
```  
  
このように10秒間Sleepした後に値を流すようなPublisherを作成します。  
  
下記のように実行してみます。  
  
```swift

SomePublisher()
   .sink { _ in print("Received value") }

print("Hello")
```  
  
  
この場合はメインスレッドで実行されているため  
  
10秒間フリーズした後に  
  
```swift

Received value
Hello
```  
という順番で出力されます。  
  
sink内のprintが完了するまで  
Helloは出力されません。  
  
ではSchedulerを利用して  
  
```swift

SomePublisher()
   .subscribe(on: DispatchQueue.global())
   .receive(on: DispatchQueue.main)
   .sink { _ in print("Received value") }

print("Hello")
```  
  
とすると  
まず即座に  
  
```swift

Hello
```  
  
が出力され  
  
その後10秒経過すると  
  
```swift

Hello
Received value
```  
  
と出力されます。  
  
これはPublisherはsubscribeによって  
バックグラウンドで実行されるようになっているため  
メインスレッドの処理は止まらずにHelloを出力しています。  
  
  
# DispatchQueue.mainとRunLoop.main  
  
↓のスレッドによると  
https://forums.swift.org/t/runloop-main-or-dispatchqueue-main-when-using-combine-scheduler/26635/4  
  
```
RunLoop.main as a Scheduler ends up calling RunLoop.main.perform 
whereas DispatchQueue.main calls DispatchQueue.main.async to do work, 
for practical purposes they are nearly isomorphic. 
The only real differential is that the RunLoop call ends up being executed 
in a different spot in the RunLoop callouts 
whereas the DispatchQueue variant will perhaps execute immediately 
if optimizations in libdispatch kick in. 
In reality you should never really see a difference tween the two.
```  
  
と書かれており  
本当にそうなのか疑問を思っていたところ  
  
[twitter](https://twitter.com/stzn3/status/1173818163554766848)で  
  
```
手元ですと、RunLoop.mainにするとスクロール中は発火しませんでした。
そのため、即時で受けたいところはDispatchQueue.mainを使っています。
```  
  
というお話をお伺いし  
試してみたところ違いがありました。  
  
例えば非同期にデータを取得して  
リストで表示したいとします。  
  
この時にスクロール中に  
次のページのデータを読み込んでリストに追加する処理をします。  
  
※ 下記は必要なところのみ記載しています。全ソースは最後に記載します。  
  
```swift

final class CollectionViewController: UIViewController {

　　...

    private func setupBindings() {
        viewModel.namePublisher
            .subscribe(on: DispatchQueue.global())
            .receive(on: DispatchQueue.main)
            .sink { [weak self] index, name in
                guard let self = self else { return }
                print("index\(index) end\(Date())")
                self.names.append(name)
                self.isLoading = false
        }.store(in: &self.cancellables)
    }
    
    private var index = 1
}

extension CollectionViewController: UICollectionViewDelegate {
    func collectionView(_ collectionView: UICollectionView, willDisplay cell: UICollectionViewCell, forItemAt indexPath: IndexPath) {
        if indexPath.item == names.count - 5 {
            isLoading = true
            print("index\(index) start\(Date())")
            index += 1
            viewModel.fetchNext()
        }
    }
}
```  
  
ViewModelではあえて通信が遅くなるように  
sleepで10秒待機します。  
  
  
```swift

final class ViewModel {
    private let nameSubject = PassthroughSubject<(Int,String), Never>()
    var namePublisher: AnyPublisher<(Int,String), Never> {
        return nameSubject.eraseToAnyPublisher()
    }
    private var index = 1
    func fetchNext() {
        DispatchQueue.global().async { [weak self] in
            sleep(10)
            self?.nameSubject.send((self!.index, "追加\(self!.index)"))
            self?.index += 1
        }
    }
}
```  
  
今回比較したのは  
  
```swift

    private func setupBindings() {
        ...
            .receive(on: DispatchQueue.main)
        ...
```  
  
と  
  
```swift

    private func setupBindings() {
        ...
            .receive(on: RunLoop.main)
        ...
```  
  
の場合です。  
  
やり方は画面をスクロールして止めるを繰り返します。  
  
結果として  
  
## receive(on: DispatchQueue.main)  
  
全てのindexのstartとendの間隔は10秒になっています。  
  
```
index1 start2019-09-21 03:36:47 +0000
index1 end2019-09-21 03:36:57 +0000

index2 start2019-09-21 03:36:58 +0000
index2 end2019-09-21 03:37:08 +0000

index3 start2019-09-21 03:37:10 +0000
index3 end2019-09-21 03:37:20 +0000

index4 start2019-09-21 03:37:20 +0000
index4 end2019-09-21 03:37:30 +0000
```  
  
## receive(on: RunLoop.main)  
  
動かしてみるとわかるのですが  
スクロール中は要素は流れて来ず  
スクロールを終了した瞬間に流れてきます。  
  
そしてスクロールを10秒以上続けると  
間隔は10秒よりも長くなります。  
  
```
index1 start2019-09-21 03:33:31 +0000
index1 end2019-09-21 03:33:43 +0000

index2 start2019-09-21 03:33:45 +0000
index2 end2019-09-21 03:33:55 +0000

index3 start2019-09-21 03:33:56 +0000
index3 end2019-09-21 03:34:06 +0000

index4 start2019-09-21 03:34:16 +0000
index4 end2019-09-21 03:34:33 +0000

index5 start2019-09-21 03:34:42 +0000
index5 end2019-09-21 03:34:58 +0000
```  
  
## なぜ？（考察）  
  
このことから  
RunLoop.mainを使うとスクロール中に発火しないことがわかりました。  
  
これはTimerをメインスレッドで動かすと  
スクロース中にTimerが止まってしまうことと同じように  
RunLoop内のスクロールや他のイベントの次の処理として登録されるため  
スクロールが終わるまでは発火していないのかなと思われます。  
  
  
一方でDispatchQueueは  
ConcurrencyProgrammingGuideに  
  
```
This queue works with the application’s run loop (if one is present) 
to interleave the execution of queued tasks with the execution of other event sources 
attached to the run loop. 
```  
  
https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html  
  
と書いているように  
RunLoopの途中に割り込んで処理を実行できるようなので  
きちんと間隔通りに処理を実行できているのではないかと思います。  
  
ここら辺は調べてみてそうなのではないかと思っているだけなので  
もしご存知の方いらっしゃればぜひ教えていただきたいです🙇🏻‍♂️  
  
# まとめ  
  
CombineとSchedulerについて見てみました。  
  
注意したい点としては  
  
*デフォルトだと要素が生成されたスレッド上で動く*  
  
ためメインスレッドで生成された場合は  
ユーザの操作を阻害してしまうリスクがある  
ことかなと思いました。  
  
基本的なことは見てきましたが  
まだまだ使い方は色々あると思いますので  
各Schedulerの使い方をさらに理解して  
非同期処理をまさにスケジュール通りに動かせるようになりたいですね😃  
  
間違いなどございましたらご指摘いただけると嬉しいです🙇🏻‍♂️  
  
  
# iOS13.3から挙動が変わるようです。  
  
forumの投稿によると  
今まで非同期でSubscriptionを渡していたのを  
同期的に渡すようになるとのことです。  
https://forums.swift.org/t/combine-receive-on-runloop-main-loses-sent-value-how-can-i-make-it-work/28631/39  
  
これで上記ページの冒頭にあったように  
タイミングによっては値の出力が抜けてしまう現象が起きなくなるようです。  
  
# 実験に使用したコード  
  
最後に使用したコードを全て載せておきます。  
  
```swift

import UIKit
import Combine

final class ViewModel {
    private let nameSubject = PassthroughSubject<(Int,String), Never>()
    var namePublisher: AnyPublisher<(Int,String), Never> {
        return nameSubject.eraseToAnyPublisher()
    }
    private var index = 1
    func fetchNext() {
        DispatchQueue.global().async { [weak self] in
            sleep(10)
            self?.nameSubject.send((self!.index, "追加\(self!.index)"))
            self?.index += 1
        }
    }
}

final class Cell: UICollectionViewCell {
    
    let label = UILabel()
    let seperatorView = UIView()
    

    override init(frame: CGRect) {
        super.init(frame: frame)
        configure()
    }
    
    required init?(coder: NSCoder) {
        fatalError("not implemented")
    }

    func configure() {
        contentView.backgroundColor = .systemBackground
        
        label.translatesAutoresizingMaskIntoConstraints = false
        contentView.addSubview(label)

        seperatorView.translatesAutoresizingMaskIntoConstraints = false
        seperatorView.backgroundColor = .gray
        contentView.addSubview(seperatorView)

        let inset = CGFloat(10)
        NSLayoutConstraint.activate([
            label.leadingAnchor.constraint(equalTo: contentView.leadingAnchor, constant: inset),
            label.topAnchor.constraint(equalTo: contentView.topAnchor, constant: inset),
            label.bottomAnchor.constraint(equalTo: contentView.bottomAnchor, constant: -inset),
            label.trailingAnchor.constraint(equalTo: contentView.trailingAnchor, constant: -inset),
            seperatorView.leadingAnchor.constraint(equalTo: contentView.leadingAnchor, constant: inset),
            seperatorView.bottomAnchor.constraint(equalTo: contentView.bottomAnchor),
            seperatorView.trailingAnchor.constraint(equalTo: contentView.trailingAnchor, constant: -inset),
            seperatorView.heightAnchor.constraint(equalToConstant: 0.5),
        ])
    }
}

class CollectionViewController: UIViewController {

    private var names = ["太郎","次郎","三郎","四郎","五郎","六郎","七郎","八郎"] {
        didSet {
            setData()
        }
    }
    
    enum Section {
        case main
    }
    
    private var isLoading = false
    private var dataSource: UICollectionViewDiffableDataSource<Section, String>!
    private var collectionView: UICollectionView! = nil
    private let viewModel = ViewModel()
    private var cancellables: Set<AnyCancellable> = []
    
    override func viewDidLoad() {
        isLoading = true
        super.viewDidLoad()
        configureHierarchy()
        configureDataSource()
        setupBindings()
        isLoading = false
    }
    
    private func configureHierarchy() {
        collectionView = UICollectionView(frame: view.bounds, collectionViewLayout: createLayout())
        collectionView.autoresizingMask = [.flexibleWidth, .flexibleHeight]
        collectionView.backgroundColor = .systemBackground
        collectionView.register(Cell.self, forCellWithReuseIdentifier: "cell")
        view.addSubview(collectionView)
        collectionView.delegate = self
    }

    private func createLayout() -> UICollectionViewCompositionalLayout {
        let itemSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                              heightDimension: .fractionalHeight(1.0))
        let item = NSCollectionLayoutItem(layoutSize: itemSize)
        
        let groupSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                              heightDimension: .absolute(200))
        let group = NSCollectionLayoutGroup.horizontal(layoutSize: groupSize, subitems: [item])
        
        let section = NSCollectionLayoutSection(group: group)
        
        let layout = UICollectionViewCompositionalLayout(section: section)
        return layout
    }

    private func configureDataSource() {
        dataSource = UICollectionViewDiffableDataSource<Section, String>(collectionView: collectionView) { collectionView, indexPath, name in
            let cell = collectionView.dequeueReusableCell(withReuseIdentifier: "cell", for: indexPath) as! Cell
            cell.label.text = name
            return cell
        }
        setData()
    }
    
    private func setData() {
        var snapshot = NSDiffableDataSourceSnapshot<Section, String>()
        snapshot.appendSections([.main])
        snapshot.appendItems(names)
        dataSource.apply(snapshot, animatingDifferences: true)
    }
    
    private func setupBindings() {
        viewModel.namePublisher
            .subscribe(on: DispatchQueue.global())
            .receive(on: RunLoop.main)
            .sink { [weak self] index, name in
                guard let self = self else { return }
                print("index\(index) end\(Date())")
                self.names.append(name)
                self.isLoading = false
        }.store(in: &self.cancellables)
    }
    
    private var index = 1
}

extension CollectionViewController: UICollectionViewDelegate {
    func collectionView(_ collectionView: UICollectionView, willDisplay cell: UICollectionViewCell, forItemAt indexPath: IndexPath) {
        if indexPath.item == names.count - 5, !isLoading {
            isLoading = true
            print("index\(index) start\(Date())")
            index += 1
            viewModel.fetchNext()
        }
    }
}
```  
