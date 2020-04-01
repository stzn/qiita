  
# 経緯  
  
普段使っているのに全体像があまり見えていないものとして  
Grand Central Dispatch (GCD)とOperationQueueが  
自分の中にあり色々と調べてみました。  
  
調べていけばいくほど新しいことがどんどん出てきて収拾がつかなくなってきたのですが、現時点までの部分でまとめてみたいと思います。  
  
  
# 概念の紹介  
  
最初にいくつかの概念について紹介します。  
  
  
## Concurrency  
  
複数のスレッドを用いて行われる同時並列処理のことです。  
iOSでは１つのプロセス(アプリケーション)の中で１つ以上のスレッドを用います。  
  
### **シングルコアの場合**  
  
時間で動かすスレッドを切り返す、  
**コンテキストスイッチ**という方法を使ってConcurrencyを実現しています。  
  
### **マルチコアの場合**  
  
parallelism(並列処理)を使って同時に複数のスレッドを動かします。  
  
  
## Task(タスク)  
  
実行される１まとまりの処理のことです。  
  
## Thread(スレッド)  
  
システムから提供される仕組みで  
Taskを同時に実行するために１つのアプリケーションの中で複数立ち上げることができます。  
スレッドの立ち上げや実行などのスケジュールはシステムによって決められます。  
  
  
![GCD.001.png](/image/6a21b83a-3c5e-9fe4-d7ff-d857fc5320b6.png)  
  
  
## Queue(キュー)  
  
複数のスレッドをより簡単に管理するためのクラスです。  
スレッドを管理するための高レベルなAPIを提供します。  
  
QueueはThreadの作成・管理をシステムに任せられるため、  
実際に実行したい処理に集中することができます。  
  
また、最終的な同時実行数はシステムによって決められ、手動で管理するよりもより効率的にタスクを実行することができます。  
  
  
  
# Grand Central Dispatch (GCD)  
  
Concurrent Operation(並行実行)を扱うためのローレベルなAPIです。  
Dispatch Queueを用いてスレッドでの処理を管理します。  
  
GCDはスレッドの概念の上位に構築され、共有スレッドプールを管理しています。  
Dispatch QueueにコードブロックやDispatchWorkItemを追加し、  
GCDがどのスレッドでそれを実行するか、どのくらい並列で処理を行うのかをシステムの使用状況や使用可能なリソースによって決めます。  
  
  
## DispatchQueue  
  
GCDはFIFO(First In First Out)なQueueを提供します。  
つまり、**処理の開始自体はQueueに追加された順に実行されます。**  
  
  
## DispatchQueueの種類  
  
### **Serial DispatchQueue(Private DispatchQueueとも言われる)**  
  
一度に一つのタスクを実行することを保証します。  
ただし、実行されるタスクはある特定のスレッドで実行されますが、  
全てのタスクが同じスレッドで実行されるとは限りません。  
ある特定のリソースに同期的にアクセスする場合などに使用されます。  
  
  
### **Concurrent DispatchQueue(Global DispatchQueueとも言われる)**  
  
１度に１つ以上のタスクを実行しますが、上記に書いたように処理の開始はQueueに登録された順に行われます。  
ただし、処理の終了はそれぞれのタスク次第になります。  
いくつタスクが実行されるかはシステムの状況によって変わります。  
  
## 具体的な３つのQueue種類  
  
### **Main Queue**  
  
アプリケーションのMain Threadで実行されるSerial Queueです。  
  
### **Global Queue**  
  
Concurrent Queueでシステム全体で共有されています。  
下記の４つの異なるタスクの実行優先度があります。  
  
- high  
- default  
- low  
- background  
  
ただし、これらを直接指定はせず```QoSClass```プロパティを指定します。  
これはタスクの重要度を表すenumで、GCDのタスクの実行順序を決める際に影響します。  
  
[DispatchQoS.QoSClass](https://developer.apple.com/documentation/dispatch/dispatchqos/qosclass)  
  
  
#### **userInteractive**  
  
UIの更新など即座にタスクが実行されて欲しい場合に使用します。  
なるべく小さくメインスレッドで実行されるべきタスクを指定します。  
  
#### **userInitiated**  
  
ボタンのタップなどUIを起点に非同期で処理するべきタスクなどを実行する場合に使用します。  
ユーザが即座の反応を待っている場合やユーザと対話的にやりとりをする必要がある場合などです。  
これは優先度highに位置付けられます。  
  
#### **default**  
  
デフォルトのQoSクラスです。QoSクラスを指定しないとこの値になります。  
userInitiatedとutilityの間に位置します。  
  
#### **utility**  
  
これはプログレスの表示などが必要な長い時間かかるタスクなどを実行する場合に使用します。  
例えば計算やネットワーク通信などがこの分類に入ります。  
これは優先度lowに位置付けられます。  
  
#### **background**  
  
ユーザが直接反応しなくても良いようなタスクの実行に使用します。  
例えばデータのプリフェッチなどです。  
これは優先度backgroundに位置付けられます。  
  
#### **unspecified**  
  
QoS情報がないということを示します。  
こうすることでシステムが優先度を推測してくれるようになります。  
  
  
Appleのドキュメントには下記のような目安が書かれています。  
  
|QoS Class  |使用例  |処理時間の目安  |  
|---|---|---|  
|User-interactive  |メインスレッドで実行されるユーザとやりとりをするような処理。<br>アニメーションやUIの更新など。  |事実上一瞬 |  
|User-initiated  |ユーザのアクションへのレスポンス。<br>保存ファイルを開くなど。  |ほぼ一瞬から数秒 |  
|Utility  |ある程度の計算が必要で、即座の結果が必要ではない<br>プログレスバーの表示が必要になるような処理。<br>ダウンロードやデータのインポートなど。 |数秒から数分 |  
|Background  |バックグラウンドで実行され、ユーザの目に見えない処理。<br>バックアップなど  |数分から数時間 |  
  
[Energy Efficiency Guide for iOS Apps](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/EnergyGuide-iOS/PrioritizeWorkWithQoS.html#//apple_ref/doc/uid/TP40015243-CH39-SW1)  
  
  
### **Custom Queue**  
  
独自に作成するQueueです。  
設定でSerialまたはConcurrentどちらも作成できますが、  
主にSerial Queueとして使用することが多いかと思います。  
作成後はGlobal Queueのように管理されます。  
  
  
  
## いつどのQueueを使うか  
  
目安として下記のような風に考えられています。  
  
  
|Main Queue  |Global Queue  |Custom Serial Queue  |  
|---|---|---|  
|Concurrent Queueでタスク完了後のUI更新など  |バックグランドで行うUIに関わらないタスク  |バックグラウンドで順番にタスクを実行したい場合や<br>実行結果を使ってさらに処理を実行したい場合。 |  
  
  
## syncとasyncの違い  
  
簡単に言うと下記のようになります。  
  
- sync {}内の処理の終了待ってから呼び出し元に制御を返す。  
- async {}内の処理の終了待たずに呼び出し元に制御を返す。  
  
### **(補足情報)asyncAfterで実行を遅らせる**  
  
  
asyncAfterメソッドを使うとタスクの実行を指定時間後に開始させることができます。  
  
  
```Swift
DispatchQueue.main.asyncAfter(deadline: .now() + 1.0) {
    print("1秒後に実行")
}
```  
  
[asyncAfter](https://developer.apple.com/documentation/dispatch/dispatchqueue/2300020-asyncafter)  
  
  
## タスク管理の仕組み  
  
タスクは主にクロージャとして指定します。  
このクロージャはDispatchWorkItemにカプセル化され、Queueで管理されます。  
  
DispatchWorkItemオブジェクトを直接作成して追加することも可能です。  
  
  
## 考慮しなければならないこと  
  
- もし２つのThreadが同時にロックしてしまった場合、Threadの同時実行数はゼロになるか相当少なくなります。  
  
- このThreadモデルは２つのThreadを必要とし、カーネルとユーザ領域の両方のメモリスペースを使用します。  
つまり同じメモリ領域を使用することの問題が起きたり、ブロックされることはないということになります。  
  
- 各Dispatch Queueは同時並行で実行されます。タスクが順番に実行されるのは１つのDispatch Queueの中だけです。  
  
- タスクの同時実行数を決めるのはシステムであり、  
100個のDispatch Queueを作成したからと言って100個同時に実行されるということではありません。  
  
- システムは次にどのタスクを開始しようか決める時にタスクの優先度(QoS)を考慮に入れます。  
  
- Private Dispatch Queueは参照型です。  
コードの中で参照をするのに加え、Dispatch Source(下記参照)も参照を保持し、参照カウントを増やしています。  
そのため、Dispatch Sourceが全てキャンセルされ、参照がきちんと解放されているかを確認する必要があります。  
  
  
## 動作の確認  
  
各Queueの動作の違いを見ていきます。  
  
  
```Swift
// 時間測定用
func duration(_ block: () -> ()) -> TimeInterval {
  let startTime = Date()
  block()
  return Date().timeIntervalSince(startTime)
}

// 検証用タスク
func task1() {
  print("Task 1 started")
  // 1秒待つことでrun2よりも終わりを遅らせる
  sleep(1)
  print("Task 1 finished")
}

func task2() {
  print("Task 2 started")
  print("Task 2 finished")
}
```  
  
  
### **Concurrent Queue**  
  
```Swift
let userQueue = DispatchQueue.global(qos: .userInitiated)
print("=== Starting userInitated global queue ===")
duration {
    userQueue.async {
        task1()
    }
    userQueue.async {
        task2()
    }
}

/* 結果
=== Starting userInitated global queue ===
Task 1 started
Task 2 started
Task 2 finished
Task 1 finished

時間 0.000238895416259....秒
*/
```  
  
Concurrentなので処理は同時に行われtask2の方が処理が先に終わっています。  
  
  
### **Custom Serial Queue**  
  
```Swift
let mySerialQueue = DispatchQueue(label: "shiz.eample.serial")

print("\n=== Starting mySerialQueue ===")
duration {
    mySerialQueue.async {
        task1()
    }
    mySerialQueue.async {
        task2()
    }
  
}

/* 結果
=== Starting mySerialQueue ===
Task 1 started
Task 1 finished
Task 2 started
Task 2 finished

時間 0.000411033630371093秒
*/

```  
  
### **Custom Concurrent Queue**  
  
```Swift
let workerQueue = DispatchQueue(label: "shiz.example.worker", attributes: .concurrent)

print("\n=== Starting workerQueue ===")
duration {
    workerQueue.async {
        task1()
    }
    workerQueue.async {
        task2()
    }
}

/* 結果
=== Starting workerQueue ===
Task 1 started
Task 2 started
Task 2 finished
Task 1 finished

時間 0.0004069805145263672...秒
*/

```  
  
Concurrentなので処理は同時に行われ、  
task2の方が処理が先に終わっています。  
  
ちなみにGlobal QueueよりもCustom Queueの方が遅いのは  
インスタンス生成の時間に時間がかかっているからだと考えられます。  
  
  
### **syncメソッドの場合**  
  
上記３つはasyncメソッドのため、処理の終了を待たずにdurationを抜けますが、  
syncの場合は処理の終了を待ってからdurationを抜けます。  
  
```Swift
let userQueue = DispatchQueue.global(qos: .userInitiated)
print("=== Starting userInitated global queue ===")
duration {
    userQueue.sync {
        task1()
    }
    userQueue.sync {
        task2()
    }
}


/* 結果
=== Starting userInitated global queue ===
Task 1 started
Task 1 finished
Task 2 started
Task 2 finished

時間 1.002609014511108秒
*/
```  
  
Serial Queueなので処理は順番に行われ、  
task1、task2の順番に終わっています。  
  
また、処理の終了を待っているため、durationで計測した時間も  
asyncと比べて長くなっています。  
  
  
  
## 値の更新から見るasyncとsyncの違い  
  
もう少しAsyncとSyncの違いを見ていきます。  
  
```Swift
// 検証用変数
var value = 99

// 検証用メソッド
func changeValue() {
  sleep(1)
  value = 0
}
```  
  
### **asyncメソッドの場合**  
  
```Swift
mySerialQueue.async {
    changeValue()
}

// 99
print(value) 

```  
  
asyncなので{}をすぐ抜けるためvalueは変わりません。  
  
### **syncメソッドの場合**  
  
```Swift
mySerialQueue.sync {
    changeValue()
}

// 0
print(value) 

```  
  
syncなので処理が終了してからvalueを出力しています。  
  
  
### **同期メソッドをチェインさせる**  
  
Queueの中で実行するタスクの典型的なパターンとして、  
同期メソッドで結果を取得し、  
その結果を使って次の処理を実行するというパターンがあります。  
  
```Swift
// 検証用のメソッド
func say1() -> String {
  return "One, "
}

func say2(inString: String) -> String {
  return inString + "Two, "
}

func say3(inString: String) -> String {
  return inString + "Three, "
}

let userQueue = DispatchQueue.global(qos: .userInitiated)
print("=== Starting userInitiated global queue ===")
userQueue.async {
    let out0 = say1()
    let out1 = say2(inString: out0)
    let out2 = say3(inString: out1)
    print(out2)
}

// out2の中身
One, Two, Three

```  
  
結果からわかるように、上から順番に処理の結果を受け取って次の処理を実行しています。  
  
実際にはもっと複雑な状況で単純にデータの受け渡しができないことの方が多いです。  
後ほど紹介する例では、プロトコルを使用することでデータの受け渡しの橋渡しをしています。  
  
  
## Concurrent Queueを有効活用できる場面  
  
例えば同じような処理を繰り返し行う、かつそれぞれが独立(同じ変数を参照していないなど)の場合  
Concurrentは同時に複数の処理ができるため非常に有効です。  
  
例えば、画像のダウンロードや画像の加工処理など、  
それぞれ別々の画像に対して行うことが多いかと思います。  
このような場合一つ一つの処理を待って行うとすごい時間がかかってしまいます。  
(CollectionViewなどですと処理が終わった頃にはもう非表示になっているなんてことも)  
  
DispatchQueueを使用することで、簡潔にConcurrent Queueを作成できます。  
  
また、後ほど紹介するOperationQueueを活用すれば、  
途中でキャンセルしたい、複数の処理を順番に行いたいなど、  
複雑な処理の制御を行うことも可能です。これは後ほど紹介します。  
  
  
## Dispatch Queueに関連した技術  
  
### **Dispatch Group**  
  
一連のDispatch Queueをまとめて管理することができます。  
SyncでもAsyncいずれにおいてもグループ内のタスクの完了を待つことができます。  
  
下記のように使います。  
  
```Swift
func add(_ input: (Int, Int)) -> Int {
  sleep(1)
  return input.0 + input.1
}

let workerQueue = DispatchQueue(label: "shiz.sample.group", attributes: .concurrent)
let numberArray = [(0,1), (2,3), (4,5), (6,7), (8,9)]

let addGroup = DispatchGroup()

for pair in numberArray {
    workerQueue.async(group: addGroup) {
        let result = add(pair)
        print("Result = \(result)")
    }
}

// waitでグループ内のタスクが全て終わるまで待機する
addGroup.wait(timeout: DispatchTime.distantFuture)

let defaultQueue = DispatchQueue.global()

// 全てのタスクが完了したのちにクロージャ内の処理を実行します。
addGroup.notify(queue: defaultQueue) {
  print("Completed!")
}

// 結果
Result = 17
Result = 1
Result = 5
Result = 13
Result = 9
Completed!
```  
  
  
#### **waitメソッドのtimeoutについて**  
  
DispatchTimeとDispatchWallTime指定することができます。  
  
- DispatchTime CPU時間。  
例えばシステムがスリープ状態の場合は時間のカウントはストップします。  
  
- DispatchWallTime wallclock time(実際の計測時間)  
例えばシステムがスリープ状態の場合でも時間はカウントします。  
  
  
```Swift
// 指定した秒数
init(uptimeNanoseconds: UInt64)

// システムが状況に応じて判断する
static let distantFuture: DispatchTime

// 現在の時刻
static func now() -> DispatchTime

```  
[DispatchTime](https://developer.apple.com/documentation/dispatch/dispatchtime)  
  
  
  
### **Dispatch Semaphore**  
  
いわゆるセマフォをより効率的に動作するようにしたものです。  
セマフォが呼び出しているスレッドがブロックされていて使用できない場合にのみカーネルを呼び出します。  
つまり処理がそこで待機します。  
セマフォが利用可能な場合はカーネルが呼ばれることはありません。  
  
  
活用できる場面としては同時に実行するタスクを制限するというような場合や  
非同期処理の結果を関数の戻り値として使用したい場合などに使用できます。  
  
下記の例では戻り値に非同期処理の結果を使っています。  
  
DispatchSemaphoreインスタンスを作成します。  
この際初期値に同時にタスクを実行できる上限数を設定しておきます。  
  
``wait```メソッドを呼びます。  
waitメソッドを呼び出すとDispatchSemaphore内の値をデクリメントします。  
waitメソッドはセマフォの値が正の値になるのを待ち、正になった場合にデクリメントして次に進みます。  
  
処理が完了後に```signal```メソッドを呼びます。  
siganlメソッドを呼び出すとDispatchSemaphore内の値をインクリメントします。  
  
  
```Swift
func getAsyncResult() -> Int {
    
    var returnValue = 0;

    // 0を初期値として設定する
    let semaphore = DispatchSemaphore(value: 0)
    
    workerQueue.async {
        asyncAdd((1, 2), runQueue: workerQueue, completionQueue: defaultQueue) { result, error in
            print("Semaphore Result = \(result)")
            returnValue =  result

            // X ここではできない error: unexpected non-void return value in void function
            // return result

            // セマフォの値をインクリメントする
            semaphore.signal()
        }
    }

    // セマフォの値が正になるまで待つ
    semaphore.wait()
    
    // 非同期で処理した結果をリターンする
    return returnValue
}
```  
  
  
### **Dispatch Source**  
  
ある種のシステムイベント(Unixシグナル, ファイルディスクリプタ, Machポート, VFSノードなど)  
のイベント監視を行います。  
  
監視方法としては下記のような種類があります。  
  
- Timer sources 定期的な通知を生成します。  
  
- Signal sources UNIXシグナルが送られてきた際に通知をします。  
  
- Descriptor sources ファイルやソケットベースの動作に関する通知を送ります。  
例えば、データの読み書きができるようになった時、ファイルシステムがファイルを削除した時、など  
  
- Process sources　システムのプロセスに関連する通知を送ります。  
例えば、プロセスが終了した時、プロセスにシグナルが送られた時、など  
  
- Mach port sources　Machに関連したイベントの通知を送ります。  
- Custom sources　独自に生成した条件の通知を送ります。  
  
  
下記のような形で定義します。  
今回はSIGSTOPシグナルが送られたイベントをハンドリングしています。  
  
  
```Swift
// グローバルな部分に定義する

// Debug時のみデータを取得する
# if DEBUG
  var signal: DispatchSourceSignal?
  private let setupSignalHandlerFor = { (_ object: AnyObject) -> Void in
    let queue = DispatchQueue.main

    // Main QueueでSIGSTOPシグナルをハンドリングする
    signal = DispatchSource.makeSignalSource(signal: Int32(SIGSTOP), queue: queue)
    
    signal?.setEventHandler {
      print("from: \(object.description!)")
    }
    signal?.resume()
  }
# endif
```  
  
ViewControllerのviewDidLoadで下記を呼び出します。  
  
```Swift
# if DEBUG
    _ = setupSignalHandlerFor(self)
# endif
```  
  
  
他にも色々な使われ方があります  
[https://github.com/dagostini/DAFileMonitor/blob/blog_dispatch_sources/DAFileMonitor/DAFileMonitor.swift](https://github.com/dagostini/DAFileMonitor/blob/blog_dispatch_sources/DAFileMonitor/DAFileMonitor.swift)  
  
  
[Dispatch Source](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/GCDWorkQueues/GCDWorkQueues.html#//apple_ref/doc/uid/TP40008091-CH103-SW1)  
  
  
### **DispatchWorkItem**  
  
タスクをカプセル化したクラスです。  
Queueを指定せずにタスクの実行や待機、キャンセルを行うことができます。  
※キャンセル可能なのは**実行される前**のDispatchWorkItemのみです。  
  
下記はSearchBarの検索処理の例です。  
  
```Swift
private var pendingRequestWorkItem: DispatchWorkItem?

func searchBar(_ searchBar: UISearchBar, textDidChange searchText: String) {
    
    // 実行前のDispatchWorkItemをキャンセルする
    pendingRequestWorkItem?.cancel()
    
    // DispatchWorkItemでAPI処理をラップ
    let requestWorkItem = DispatchWorkItem { [weak self] in
        
        // API呼び出しなどの処理
        self?.search(forQuery: searchText)
    }
    
    // ワークアイテムをクラスに保存し、25秒後に処理が開始されるように設定
    // 25秒以内に処理をキャンセルすれば処理は実行されない
    pendingRequestWorkItem = requestWorkItem
    DispatchQueue.main.asyncAfter(deadline: .now() + .milliseconds(250),
                                  execute: requestWorkItem)
}

```  
  
# Operation Queue  
  
OperationQueueはGCDの高レベルに抽象化したものです。  
コードブロックでタスクを定義するのではなく、  
Operationクラスを使用してタスクを定義します。  
  
  
## Operation  
  
実行したい処理やデータをカプセル化したクラス。  
Operationクラス自体は抽象化されたベースクラスで実際はサブクラスを作成して使用します。  
Operationクラスですでに必要なものはほぼ実装されているため、必要最小限の実装でサブクラス化は可能です。  
  
また、FoundationフレームワークはOpeartionクラスのサブクラスを用意しています。  
  
### **BlockOperation**  
  
一つ以上の処理の同時実行を管理します。  
それぞれの処理ごとにOpeartionオブジェクトを生成する必要がなくなります。  
また、処理は全てのブロック処理が完了して時点で終了したと見なされます。  
  
追加されたブロック処理はデフォルトの優先度で適切なQueueに送られます。  
そのため、特定の環境に依存したような処理を入れるべきではありません。  
  
  
## Operationの仕組み  
  
Operationの中はどのようになっているのかを見ていきたいと思います。  
後ほど実際にどういう挙動をするのかはコードで提示します。  
  
  
### **実行は一度きり**  
  
一度処理を実行すると、再び処理が実行されることはありません。  
※single-shot objectと言うそうです。  
  
### **Operation Queueにオブジェクトを加えると実行される**  
  
Operation Queueの```addOperation```メソッドなどで追加をすることで  
処理が開始されます。  
  
この場合追加されるのが**SyncなOperationa**であったとしても  
Operation QueueはOperationを実行するために**新しくスレッドを作成します。**  
つまり、結果的に処理は**Async**になります。  
  
[addOperation](https://developer.apple.com/documentation/foundation/operationqueue/1410704-addoperation)  
  
  
### **Operation Queueを使わなくても実行できる**  
  
Operationクラスの```start```メソッドを使用すれば実行できます。  
デフォルトでは```start```メソッドが呼ばれたスレッド上でSyncに実行されます。  
  
この方法はOperationクラスのステートを手動で管理しなければいけないなど負担は大きくなります。  
例えば、isReadyがfalseの場合に実行しようとするとエラーになります。  
  
[start](https://developer.apple.com/documentation/foundation/operation/1416837-start)  
  
  
### **Operationの処理順序を制御できる(依存関係を構築できる)**  
  
Operationクラスの```addDependency```メソッドや  
```removeDependency```メソッドを使用することでOperationの実行順序を決められます。

```Swift  
let op1 = Operation()  
let op2 = Operation()  
  
// ...処理の設定  
  
// op2はop1が実行されたあとに実行される  
op2.addDependency(op1)  
  
```

デフォルトの挙動として、依存しているOperaionがあるOperationは
依存しているOperationが全て完了しないと実行する準備ができていないと見なされ、
最後の依存しているOperationの実行が完了すると、準備完了になり実行ができるようになります。

#### **注意点①**

Operationクラスが管理している依存関係は
依存しているOperationが成功したかどうかがわかりません。
例えば、Opeartionをキャンセルした場合でもステータスとしては実行終了とされてしまいます。

つまり、実行が成功していなかった場合にどう対処するかはあなた次第になります。
独自のエラー処理の仕組みが必要になってくるかもしれません。

#### **注意点②**

Dead Lockにも注意が必要です。
例えば、別々のQueueで実行しているOperation同士で依存関係を構築してしまうと依存関係が循環してしまいます。


![GCD.003.png](/image/afccc171-a4dc-fcf4-caf3-30ce3af211fc.png)



### **OptionalなCompletionBlockを持っている**

これらはメインの処理が完了したのちに実行されます。


### **KVO(key-value observing)通知に応じて実行状態の監視・変更を行う**

下記のプロパティはKeyPathを利用することができ、値に応じた状態管理などを行います。

- isCancelled 読取専用
- isAsynchronous 読取専用
- isExecuting 読取専用
- isFinished 読取専用
- isReady 読取専用
- dependencies 読取専用
- queuePriority 読み書きOK
- completionBlockreadonly 読み書きOK

#### **注意点**
Cocoaバインディングを用いてUIと繋げることは避けるべきです。
なぜならば、UIはメインスレッドで更新しなければいけないため、
別スレッドで実行されるOperationのプロパティでは処理が難しくなります。

Operationクラスに新しくプロパティを追加する場合は、
KVCとKVOに対応することが推奨されています。

下記に簡単な状態遷移図を示します。
Finished以外は全てCancelledに遷移することができます。

![GCD.007.png](/image/d6911a77-231c-9737-0560-b58f1a2c896f.png)



### **優先度を設定できる**

優先順位を設定することで実行順序の制御ができます。

### **キャンセルができる**

処理の実行停止が可能です。(ただしすでに実行中のものはisCancelledプロパティをチェックする必要があります。)


### **マルチコアに対応している**

複数のスレッドから呼ばれたとしてもロック制御を追加する必要がなく、
安全にメソッドを呼び出すことができます。

Operationは生成されたスレッドとはだいたいは異なるスレッドで実行されるため、
この挙動は必要なのです。

そのため、Operationのサブクラスを作成する際、
overrideする場合でも新しいメソッドを生成する場合でも
スレッドセーフであることを保証しなければなりません。


### **OperationをAsyncにする場合**

Operationを直接実行する(startメソッドを呼ぶ)場合、
同期的に(Syncで)実行するか非同期的に(Asyncで)実行するかを設計できます。
デフォルトはSyncです。
Syncの場合は別スレッドを作成せず現在のスレッド上で即座に処理を実行するため、
```start```メソッドが呼び出し元にアプリの制御を返す時に処理が完了します。  
  
Asyncで実行する場合、処理は別スレッドで実行します。  
また、処理が継続中でもアプリの制御は呼び出し元に戻ります。  
  
Asyncにする場合は、処理が継続中の状態を監視しなければならず、  
KVO通知で状態の変化を通知する必要があります。  
しかし、Async Operationは呼び出し元のスレッドをブロックさせたくない場合などに有用です。  
  
  
### **Opeartion Queueを使用する場合**  
  
Asyncのことを考慮に入れる必要はありません。  
この場合、OperationがAsyncであるかどうかを示す```isAsynchronous```というプロパティは無視され、  
```start```メソッドは別スレッドから必ず呼ばれるようになります。

[isAsynchronous](https://developer.apple.com/documentation/foundation/operation/1408275-isasynchronous)

[Operation](https://developer.apple.com/documentation/foundation/operation)


## 基本的な動きの確認

BlockOperationを使って動きの確認をします。

```Swift  
var result: Int?  
  
let op = BlockOperation {  
  result = 2 + 3  
  sleep(3)  
}  
duration {  
  op.start()  
}  
  
// 5  
print(result)  
  
// 計測時間  
3.002378940582275  
```

処理
時間を見てみると3秒以上かかっており、
ここからOperationがSync(処理が完了するまで待つ)であることがわかります。


```Swift  
let op = BlockOperation()  
op.completionBlock = {  
  print("終了!")  
}  
  
op.addExecutionBlock {  print("1"); sleep(2) }  
op.addExecutionBlock {  print("2"); sleep(2) }  
op.addExecutionBlock {  print("3"); sleep(2) }  
op.addExecutionBlock {  print("4"); sleep(2) }  
op.addExecutionBlock {  print("5"); sleep(2) }  
  
duration {  
  op.start()  
}  
  
/* 結果  
  
4  
1  
3  
2  
5  
終了！  
  
*/  
  
// 計測時間  
4.002833008766174  
```

出力結果は毎回変わります。

出力結果や計測時間からOperationがConcurrent(同時実行する)であり、
追加された各ブロックは別スレッドで実行されていることがわかります。


次にカスタムOperationを作成してみます。
下記は一番単純な方法でmainメソッドのみoverrideしています。

```Swift  
class PrintOperation: Operation {  
    var input: String?  
    var output: String?  
    
    override func main() {  
        output = "> \(input!) 出力しました。"  
    }  
}  
  
let op = PrintOperation()  
op.input = "入力しました。"  
duration {  
    sleep(3)  
    op.start()  
}  
print(op.output!)  
  
  
/* 結果  
> 入力しました。 出力しました。  
*/  
  
// 処理時間  
3.001950979232788  
  
```


### **Asyncメソッドを実行したい場合は？**

例えばURLSessionを使用したネットワーク通信などはAsyncで実行されます。

デフォルトのままですと、OperaiotnはSyncで処理を実行しますが、
中でAsyncメソッドを実行すると処理がすぐに戻ってきて終了したとみなされてしまいます。

これを解決するためにはOperationをAsyncにする場合で記載したのと同様に
Opearionの状態を自己管理する必要があります。

下記にAsyncメソッドを処理するOperationのサブクラスの例を示します。

※これも抽象クラスでmainメソッドのoverrideはしていません。

```Swift  
class AsyncOperation: Operation {  
  
  // Operationの状態管理をするenum  
  // caseが大文字なのはkeyPathで使う値をrawValueから取得したいため  
  enum State: String {  
    case Ready, Executing, Finished  
    
    fileprivate var keyPath: String {  
      return "is" + rawValue  
    }  
  }  
  
  // KVO通知で状態が変化したことをSuperクラスの伝える  
  var state = State.Ready {  
    willSet {  
      willChangeValue(forKey: newValue.keyPath)  
      willChangeValue(forKey: state.keyPath)  
    }  
    didSet {  
      didChangeValue(forKey: oldValue.keyPath)  
      didChangeValue(forKey: state.keyPath)  
    }  
  }  
}  
  
// 各プロパティが返す値やメソッドもAsyncメソッドの実行に合わせるようにoverrideする  
  
extension AsyncOperation {  
  
  override var isReady: Bool {  
    return super.isReady && state == .Ready  
  }  
  
  override var isExecuting: Bool {  
    return state == .Executing  
  }  
  
  override var isFinished: Bool {  
    return state == .Finished  
  }  
  
  override var isAsynchronous: Bool {  
    return true  
  }  
  
  override func start() {  
    if isCancelled {  
      // ステータスを終了に変更  
      state = .Finished  
      return  
    }  
    main()  
    // ステータスを実行中に変更  
    state = .Executing  
  }  
  
  override func cancel() {  
    // ステータスを終了に変更  
    state = .Finished  
  }   
}  
```

使用する場合はAsyncOperationのサブクラスを使用します。
```main```メソッドの最後でstateを終了にしなければなりません。  
  
```Swift
final class ConcatOperation: AsyncOperation {
    let lhs: String
    let rhs: String
    var result: String?
    
    init(lhs: String, rhs: String) {
        self.lhs = lhs
        self.rhs = rhs
        super.init()
    }
    
    override func main() {
        asyncConcat(lhs: lhs, rhs: rhs) { result in
            self.result = result

            // .Finishedにしないと完了したことが伝えられない
            self.state = .Finished
        }
    }
}

let q = OperationQueue()
let c = ConcatOperation(lhs: "Hello", rhs: "World")
c.completionBlock = {
    guard let result = result else {
        return
    }
    print(result)
}
q.addOperation(c)

/* 結果
Concat: HelloWorld
*/

```  
  
このAsyncOperationは後ほどアプリを使った実装例でも使用します。  
  
  
## キャンセル時の動きの確認  
  
単純にメッセージを追加していくだけの処理です。  
途中経過を出力するようにしています。  
  
```Swift
// 検証用メソッド
func messageAddArray(_ words: [String], progress: ((Double) -> (Bool))? = nil) -> [String] {
    var results = [String]()
    for word in words {

        // 1秒後待機
        sleep(1)

        results.append(word)
        if let progress = progress {
            if !progress(Double(results.count) / Double(words.count)) { return results }
        }
    }
    return results
}

final class MessageMakeOperation: Operation {
    let inputArray: [String]
    var outputArray = [String]()
    
    init(input: [String]) {
        inputArray = input
        super.init()
    }
    
    override func main() {
        outputArray = messageAddArray(inputArray) { progress in
            print("\(progress*100)% 完了")
            return !self.isCancelled
        }
    }
}
```  
  
キャンセルを行わない場合、下記のように通常通り終了します。  
  
```Swift
let words = ["みな", "さん", "こんばんわ", "Hello", "World"]

let op = MessageMakeOperation(input: words)
let queue = OperationQueue()

queue.addOperation(op)

//sleep(4)
//op.cancel()

op.completionBlock = {
    print(op.outputArray)
    PlaygroundPage.current.finishExecution()
}

/*結果

20.0% 完了
40.0% 完了
60.0% 完了
80.0% 完了
100.0% 完了
["みな", "さん", "こんばんわ", "Hello", "World"]

*/
```  
  
4秒後のキャンセルを行った場合、下記のようになります。  
  
```Swift
let words = ["みな", "さん", "こんばんわ", "Hello", "World"]

let op = MessageMakeOperation(input: words)
let queue = OperationQueue()

queue.addOperation(op)

// 4秒後にキャンセル
sleep(4)
op.cancel()

op.completionBlock = {
    print(op.outputArray)
    PlaygroundPage.current.finishExecution()
}

/*結果

20.0% 完了
40.0% 完了
60.0% 完了
80.0% 完了
["みな", "さん", "こんばんわ", "Hello"]

*/
```  
  
  
## Operation Queue  
  
現在のスレッド、またはもう一つのスレッド、またはlibdispatchを使って間接的に、  
operationを実行する  
  
### **OpearionQueueのプロパティ**  
  
```Swift
// メインスレッドで処理を実行するOperationQueueを返します
class var main: OperationQueue

// 現在の処理を実行しているOperationQueueを返します
class var current: OperationQueue?

// 最大同時実行数のデフォルト値
// デフォルトは-1となっており、システムに任せるという意味です。
class let defaultMaxConcurrentOperationCount: Int

// 最大同時実行数のデフォルト値
// Queueごとに設定でき、１の場合はSerial Queueと同じ意味になります。
var maxConcurrentOperationCount: Int

```  
  
```Swift
// Operationを追加・実行します。
func addOperation(_ op: Operation)

// blockに書かれた処理をOperationとして追加・実行します。
func addOperation(_ block: @escaping () -> Swift.Void)

// 複数のOperationを同時に追加します。これは同じスレッドで実行され、
// waitUntilFinishedがtrueの場合は全てのOperationが終了するまで次の処理に移るのを待ちます。(=Sync)
func addOperations(_ ops: [Operation], waitUntilFinished wait: Bool)
```  
  
#### **Operation追加時の注意点**  
  
ある特定のOperationオブジェクトは１つのOpeartionQueueに１つしか入れられません。  
もし同じオブジェクトを入れようとした場合invalidArgumentExceptionが投げられます。  
また、オブジェクトが実行中であったり、すでに終了している場合もinvalidArgumentExceptionが投げられます。  
  
```Swift
// 現在追加されているOperationの配列
var operations: [Operation] { get }

// 現在追加されているOperationの数
var operationCount: Int { get }

// 全てのOperationのcancelメソッドを呼びます。
func cancelAllOperations()

// 全てのオペレーションが終了するまで待機します。
func waitUntilAllOperationsAreFinished()

// OperationQueueの優先順位
var qualityOfService: QualityOfService

// Operationの実行が中断されているかどうか
var isSuspended: Bool

// Operationの実行にしようするDispatchQueue
unowned(unsafe) open var underlyingQueue: DispatchQueue?

// Queueの名前
var name: String?
```  
#### **cancelAllOperationsの注意点**  
  
これを呼んだからといってOperationがQueueから削除されたり、実行中のOperationが停止するわけではありません。  
  
実行前のOperationはキャンセルされたと見なされる前に終了状態になるように処理の実行をしなければなりません。  
  
またすでに実行中のものはキャンセル状態をチェックして処理をキャンセルさせなければなりません。  
(キャンセル状態になるとその後終了状態に遷移する)  
  
そうすることでどちらもQueueから削除される前にcompletion blockの実行ができます。  
  
#### **キャンセルが呼ばれた時の依存関係について**  
  
Appleのドキュメントによりますと、このcancelが呼ばれた場合にOperation同士の依存関係は無視され、```start```メソッドが呼ばれるようです。  
つまり、Operationを終了状態に持っていくことができ、Queueから削除できるようになります。  
  
このことから、キャンセルされた場合のチェックをOperation内で考慮していない場合、予期せず動作をする可能性があります。  
(nilチェックをしていないため落ちるなど)  
  
  
#### **waitUntilAllOperationsAreFinishedの注意点**  
  
メインスレッドで呼んではいけません。  
  
#### **isSuspendedの注意点**  
  
trueの場合でも実行中の処理は継続され、  
現在実行待ちのOpearionが実行されなくなります。  
falseになった際に実行を再開します。  
これはKVOで監視をすることができます。  
  
#### **underlyingQueueについて**  
  
デフォルトはnilです。これを設定すると指定してDispatchQueuの中に  
DispatchQueueに追加されるblockと同じようにOperationを追加することができます。  
  
この値はOperation QueueにOperationがない状態でないと設定できません。  
そうでないとinvalidArgumentExceptionが投げられます。  
  
  
### **Operationの実行順序**  
  
Operationの部分でも少し記載しましたが、  
Operationの準備状態、QoS、Dependenciesにしたがって決まります。  
QoSが同じ場合でかつ全てが準備完了の場合、追加された順番で実行します。  
基本的には優先度の高いものから順に実行されます。  
  
#### **注意点**  
  
この追加された順番で実行されるということに依存するのは良くありません。  
仮に何かの原因であるOperationの準備が遅れた場合に順番は変わってしまいます。  
必要なものは優先度をあげたり、  
処理に特定の順番が必要な場合はDependencyを設定するようにします。  
  
### **KVO**  
  
下記のプロパティが対応しています。  
  
- operations 読取専用  
- operationCount 読取専用  
- maxConcurrentOperationCount 読み書きOK  
- isSuspended 読み書きOK  
- name 読み書きOK  
  
これもOperationと同様にCocoaバインディングはしないほうが良いです。  
  
### **スレッドセーフ**  
  
Operationと同様に複数のスレッドから呼び出しても問題ありません。  
  
繰り返しの記載になりますが、  
Operaiotn QueueはOperation初期化時にDispatchフレームワークを使用しているため、個々のOperationは常に別スレッドで実行されます。  
  
## 基本的な動きの確認  
  
```Swift
let queue = OperationQueue()

// 同時接続数 2
queue.maxConcurrentOperationCount = 2

// 計測時間 0.00065398216247
duration {
  queue.addOperation { print("1"); sleep(3) }
  queue.addOperation { print("2"); sleep(3) }
  queue.addOperation { print("3"); sleep(3) }
  queue.addOperation { print("4"); sleep(3) }
  queue.addOperation { print("5"); sleep(3) }
}

// 計測時間 9.011060953140259
duration {
  queue.waitUntilAllOperationsAreFinished()
}

/* 結果
1
2
3
4
5
*/
```  
  
この結果から考えますと、addOperationはAsyncのため、  
最初のdurationの計測時間は短くなっています。  
  
２つ目のdrurationの計測時間を見ますと、9秒ちょっととなっています。  
これは同時接続数を2にしているため、5つの処理を３回に分けて行なっていることがわかります。  
  
同時接続数を変更した場合、  
  
```Swift
let queue = OperationQueue()

// ☆☆☆☆☆同時接続数 3
queue.maxConcurrentOperationCount = 3

// 計測時間 0.00072300434112
duration {
  queue.addOperation { print("1"); sleep(3) }
  queue.addOperation { print("2"); sleep(3) }
  queue.addOperation { print("3"); sleep(3) }
  queue.addOperation { print("4"); sleep(3) }
  queue.addOperation { print("5"); sleep(3) }
}

// 計測時間 6.005084991455078
duration {
  queue.waitUntilAllOperationsAreFinished()
}

/* 結果
1
2
3
4
5
*/
```  
といったように同時接続数が増え、５つの処理を２回に分けて行なっているため、  
6秒ちょっとになっています。  
  
#### **注意点**  
出力結果は変わらず、追加した順番通りに出力されていますが、  
これはたまたま優先順位が同じなだけであり、  
同時実行されていることからもわかるように  
追加した順番で結果が得られるというわけではありません。  
  
  
# アプリの実装例  
  
Operation、OperationQueueを使用したサンプルを作成しました。  
このサンプルではOperation同士が依存関係を構築できることを活用し、  
  
TableViewの各セルに対して  
  
・Network(Dummy)から圧縮された画像データを取得する。  
・圧縮された画像データを解凍する  
・画像データをセピア色にフィルターする  
・フィルターしたデータを取り出してUIImageViewに設定する  
  
という処理を順番に行います。  
  
主な依存関係を構築している箇所は下記のようになります。  
  
  
```Swift
// (一部を抜粋しています)

final class SepiaImageProvider {
    
    private let operationQueue = OperationQueue()
    let photoInfo: PhotoInfo
    
    init(photoInfo: PhotoInfo, completion: @escaping (UIImage?) -> ()) {
        self.photoInfo = photoInfo
        
        let url = Bundle.main.url(forResource: photoInfo.name, withExtension: "compressed")!
        
        let dataLoad = DataLoadOperation(url: url)
        let decompress = ImageDecompressionOperation(data: nil)
        let sepiaFilter = SepiaFilterOperation(image: nil)
        let output = ImageFilterOutputOperation(completion: completion)
        
        let operations = [dataLoad, decompress, sepiaFilter, output]
        
        // 依存関係を設定
        decompress.addDependency(dataLoad)
        sepiaFilter.addDependency(decompress)
        output.addDependency(sepiaFilter)
        
        operationQueue.addOperations(operations, waitUntilFinished: false)
    }    
}
```  
  
もちろん各セルの画像はConcurrentに処理されます。  
  
**[2018/7/11追加]**  
Time Profilerで確認するとMain Thread以外で動いているのがわかります。  
  
![スクリーンショット 2018-07-11 6.15.25（2）.png](/image/0ee904c3-81a2-d3ec-a5bb-2a5e5551466d.png)  
  
  
  
[https://github.com/stzn/OperationQueueSample](https://github.com/stzn/OperationQueueSample)  
  
  
# Concurrencyの問題  
  
処理を複数同時に行うことによって生じる問題として大きく３つのことがあります。  
  
  
## Race Condition  
  
Concurrent処理内で同じ共有している変数の値を読み書きする場合に  
データの不整合が起きることを指しています。  
  
例えば下記の図の場合、  
  
Threa2で書き換えようとしている値はbbbbのはずですが、  
書き換えが完了する前に他のThreadが値の読み込みをおこなっているため、  
中途半端に書き換えられた値を読み込んでしまいます。  
  
さらに途中でThread1も値の書き換えを始めているため、  
さらによくわからない状態になります。  
  
![GCD.004.png](/image/84da1558-3002-48b0-e56d-0b537bb67fba.png)  
  
  
  
この状態を解消するためには、  
データの書き込みの際は一つのスレッドしか値にアクセスできないようにすることです。  
  
方法として下記のようなものがあります。  
  
### **Serialなキューを利用する**  
  
そもそもタスクが一度に一つしか処理されないため、Race Conditionは発生しません。  
  
### **Dispatch Barrierを使う**  
  
DispatchWorkItemをQueueに追加する際に、```.barrier```というflagを設定することできます。  
これを設定することで、このItemが実行されている間Queueの中で実行されるのは  
唯一このItemだけという状態を作ることができます。  
  
このItemが追加される前にタスクに関してはItemが実行される前に全て完了されます。  
  
Itemの実行終了後は元の通りに処理は継続されます。  
  
  
```Swift
private let concurrentQueue = DispatchQueue(label: "concurrentQueue", attributes: .concurrent)
private var dictionary: [String: Any] = [:]
    
func set(_ value: Any?, forKey key: String) {
    // .barrierのItemが追加される前の値を読み込むタスクは書き込みを使用する前に完了され、
    // 書き込みが完了するまで値の読み込みはされません。
    concurrentQueue.async(flags: .barrier) {
        self.dictionary[key] = value
    }
}

func object(forKey key: String) -> Any? {
    var result: Any?

    // concurrentQueueを利用することで書き込み中の場合はアクセスを待機する
    // 逆に読み出し中に値が変更されることもなくなる
    // syncメソッドを使用している
    concurrentQueue.sync {
        result = dictionary[key]
    }

    // concurrentに値の取得が完了したデータを返す
    return result
}
```  
  
  
#### **注意点**  
  
Global Queueで.barrierを使用する場合は注意が必要です、  
この場合Queue自体が共有されているため、処理をブロックしてしまう可能性があります。  
  
また、Custom Serial Queueを使用してもすでにSerialであるため、意味はありません。  
  
一番良いのはCustom Concurrent Queueを使用することです。  
  
さらに注意が必要なのは、現在実行中のQueueの中でsyncメソッドを呼ばないでください。  
  
現在実行中のQueueで行なった場合、syncメソッドはクロージャ内の処理の終了を待ちますが、  
クロージャは現在実行中のタスク(クロージャ)が終了しないと処理を開始しません。  
逆に現在実行中のタスクはsyncメソッドが終了しないと完了できません。  
  
  
**[2018/7/14 追記]**  
  
下記のような方法もあります。  
  
```Swift

// MARK: - DispatchQueue
class DispatchQueueAtomicProperty {
    private let queue = DispatchQueue(label: "shiz.sample.DispatchQueueAtomicProperty")
    private var _value = 0

    var value: Int {
        get {
            return queue.sync { _value }
        }
        set {
            queue.sync { [weak self] in
                self?._value = newValue
            }
        }
    }
}

// MARK: - OperationsQueue
class OperationsQueueAtomicProperty {
    private let queue: OperationQueue = {
        var q = OperationQueue()
        q.maxConcurrentOperationCount = 1
        return q
    }()
    
    private var _value = 0

    var value: Int {
        get {
            var value: Int!
            execute(on: queue) { [_value] in
                value = _value
            }
            return value
        }
        set {
            execute(on: queue) { [weak self] in
                self?._value = newValue
            }
        }
    }

    private func execute(on q: OperationQueue, block: @escaping () -> Void) {
        let op = BlockOperation(block: block)
        q.addOperation(op)
        op.waitUntilFinished()
    }
}
```  
  
**[2018/8/7 追記]**  
  
下記のサイトにAppleのlockingAPIの違いによるベンチマークが掲載されていました。  
結論としては、まずDispatchQueueを使うことを考え、  
それが難しい場合はNSLockを使うことが推奨されています。  
[http://www.vadimbulavin.com/benchmarking-locking-apis/?utm_campaign=Revue%20newsletter&utm_medium=Swift%20Weekly%20Newsletter%20Issue%20125&utm_source=Swift%20Weekly](http://www.vadimbulavin.com/benchmarking-locking-apis/?utm_campaign=Revue%20newsletter&utm_medium=Swift%20Weekly%20Newsletter%20Issue%20125&utm_source=Swift%20Weekly)  
  
## Priority Inversion  
  
優先度　低  タスク1  
優先度　中  タスク2  
優先度　高  タスク3  
  
があるとします。  
  
まず、タスク1が先に開始されて共有データをロックします。  
その後タスク2が開始されるとタスク1はストップします。  
この時タスク2が共有データにアクセスする必要がある場合、  
タスク1がロックしているため、タスク2は待たなければならなくなります。  
  
さらにタスク3が次に開始されるとタスク2はストップします。  
この時タスク3が共有データにアクセスする必要がある場合、  
タスク1がロックしているため、タスク3は待たなければならなくなります。  
  
  
![GCD.005.png](/image/880f28e5-fda1-96b9-8d66-04ccf374807a.png)  
  
  
  
### **Serial Queueで実行している場合**  
  
この場合、Inversionが生じている間だけ  
システムが自動で優先度が低いタスクの優先度を上げるように動きます。  
  
これはSerial Queueでsyncやwaitメソッドが呼ばれている時に起きます。  
  
  
### **Concurrent Queueで実行している場合**  
  
上記の例の場合、システムはInversionの状態のまま問題の解決を試みます。  
  
まずタスク2を開始させます。  
タスク2のリソースの使用が完了した後、タスク1のリソースの使用が開始されます。  
タスク１のリソースの使用が完了した後、タスク3のリソースの使用が開始されます。  
  
![GCD.006.png](/image/23001d04-d235-2821-59dd-9175f0024230.png)  
  
  
  
## Dead Lock  
  
Race Conditionの箇所でも出てきましたが、  
複数のスレッドがお互いのスレッドの終了を待っているため、  
処理を先に進めることができなくなってしまう状態です。  
  
  
注意する点としては下記のような場合があります。  
  
- Main Queueでsyncメソッドを呼ぶ  
- Serial Queueでsyncメソッドを呼ぶ  
- Serial Queueでwaitメソッドを呼ぶ  
- 現在実行中のQueueでsyncメソッドを呼ぶ  
- 異なるQueueのOperationでお互いに依存関係を構築する  
  
  
## Livelock  
  
これはDeadlockの特殊な例で、  
複数のスレッドが1つのリソースを取り合い、  
さらにそのリソースの状態が常に変化するため、  
競合ー＞待機ー＞再試行を繰り返して作業が完了できない状態です。  
  
イメージ的には  
２人の人がお互いに道を譲ろうとして  
同じ方向にどいてしまうのを繰り返すようなイメージらしいです。  
  
  
##  Heavily Contended Locks  
  
Lockをかける時間が長かったり、ロック中の処理が遅く  
処理を待機しているスレッドがどんどん溜まっていってしまう状態です。  
  
## Thread Starvation  
  
これは優先度の違うタスクを設定している場合に生じます。  
例えばいくつかの優先度の低いタスクとたくさんの優先度の高いタスクがある場合、  
優先度の低いタスクはいつまで立っても処理が開始されなかったり、  
開始時間がすごい遅くなってしまう状態です。  
  
## Thread Explosion  
  
globalなDispatchQueueを大量に使用する場合などに起こることがあります。  
  
CPUは実行する前に次に何をするのかということをある程度予測(事前ロード)しておく動きをします。  
ただし、これは同スレッド内での話で、コンテキストスイッチが生じると、この事前ロードがクリアされてしまい、  
結局CPUは、目の前にあるタスクのみを都度都度ロードして実行するようになり、効率が非常に悪くなっている状態を指します。  
  
  
## Concurrency問題に立ち向かう...その前に  
  
そもそもこういう問題を起こらないように考慮することが大事です。  
一般的に下記のような点に気をつける必要があります。  
  
- 共有データを使用する場合は１つのQoSクラスのタスクのみアクセスするようにする  
- 共有データにアクセスするのはSerial Queue上で行う  
- Operationの依存関係の循環を避ける  
- syncメソッドを呼ぶ時は特に注意を払う  
- 現在実行中のQueueでsyncメソッドは呼ばない  
- Main Queueでsyncメソッドは**絶対**呼ばない  
  
  
これらのようなことは下記を用いてチェックすることができます。  
  
### **Thread Sanitizer(TSan)を活用する**  
  
Thread Sanitizerを有効にすることで、  
Race Conditionになっているかどうかを自動でチェックしてくれます。  
  
使い方に関してはこちらに詳しく書かれていますのでご参照ください  
[https://qiita.com/mono0926/items/901d39ef06f2ac330c68](https://qiita.com/mono0926/items/901d39ef06f2ac330c68)  
  
  
[Thread Sanitizer](https://developer.apple.com/documentation/code_diagnostics/thread_sanitizer)   
  
  
### **dispatchPreconditionを使う**  
  
iOS10より使用可能なメソッドで、現在のQueueがどのQueueを使用しているのかを事前チェックできます。  
確認をすることで誤って現在実行中のQueueでsyncメソッドを呼ぶリスクを回避することなどが可能です。  
  
以下のように使います。  
  
```Swift
let queue = DispatchQueue.global(attributes: .userInitiated)
let mainQueue = DispatchQueue.main

mainQueue.async {

    // MainQueue上なのでfalseになってエラー
    dispatchPrecondition(condition: .notOnQueue(mainQueue))
}

queue.async {

    // 同じQueue上なのでtrueで処理は継続
    dispatchPrecondition(condition: .onQueue(queue))
}
```  
  
[dispatchPrecondition](https://developer.apple.com/documentation/dispatch/1780605-dispatchprecondition)  
  
  
  
# GCDとOperationQueueをどう使い分けるのか  
  
これまで見てきたことからざっくりと以下のように考えることができるのではないかと思います。  
  
  
## GCD  
  
- 繰り返し使うことがなく、タスク同士に依存関係(ある処理結果を用いて次の処理を行うなど)や  
処理のキャンセル、再開などが必要がない場合。  
  
- 現在存在している同期メソッドを非同期に動作させるためにWrapする場合など。  
  
- Queue間で共通のデータの読み書きが必要な際のデータの同期にも用いる。  
  
例えば、GCDには便利なメソッドとして```concurrentPerform(iterations:execute:)```があります。  
[concurrentPerform](https://developer.apple.com/documentation/dispatch/dispatchqueue/2016088-concurrentperform)  
ほとんどドキュメントに記載はありません。  
  
これを使用するとforループの処理をconcurrentに実行することができます。  
ただし、注意が必要でiterationsに指定する数が多すぎるとoverheadが生じていまいます。  
  
また、Serial Queueに使ってもConcurrentに処理ができないため意味がありません。  
  
有効な場面としてはConcurrent Queueでforループ処理が必要な場合などがあります。  
  
```Swift
DispatchQueue.concurrentPerform(iterations: 5) { (i) in
    print(i)
}
```  
  
  
## OpearionQueue  
  
- Operation間で依存関係が必要である場合  
  
- 処理のキャンセルや再開が必要な場合  
  
- タスク(Operation)が繰り返し利用される場合  
  
  
# まとめ  
  
まとまらないですね:frowning2:  
理解を深めていくためにも随時更新して整理をしていきたいと思います。  
  
  
間違いなどご指摘ございましたらよろしくお願い致します:bow_tone1:  
  
  
  
## 主な参考資料  
  
[Concurrent Programming With GCD in Swift 3](https://developer.apple.com/videos/play/wwdc2016/720/#)  
[Parallel programming with Swift: Basics](https://medium.com/flawless-app-stories/basics-of-parallel-programming-with-swift-93fee8425287)  
[iOS Concurrency — Underlying Truth](https://medium.com/@chetan15aga/ios-concurrency-underlying-truth-1021a0bb2a98)  
[Dispatch Barriers in Swift 3](https://medium.com/@oyalhi/dispatch-barriers-in-swift-3-6c4a295215d6)  
[Grand Central Dispatch Tutorial for Swift 3: Part 1/2](https://www.raywenderlich.com/148513/grand-central-dispatch-tutorial-swift-3-part-1)  
[Grand Central Dispatch Tutorial for Swift 3: Part 2/2](https://www.raywenderlich.com/148513/grand-central-dispatch-tutorial-swift-3-part-2)  
[Operation and OperationQueue Tutorial in Swift](https://www.raywenderlich.com/190008/operation-and-operationqueue-tutorial-in-swift)  
[Atomic Properties in Swift](http://www.vadimbulavin.com/atomic-properties/?utm_campaign=iOS%2BDev%2BWeekly&utm_medium=email&utm_source=iOS%2BDev%2BWeekly%2BIssue%2B360)  
  
他Appleドキュメント  
