今年のWWDCでは  
マルチプラットフォームでの利用というものが注目され  
今後はより多様な形でアプリが使用されていくのかなと感じています。  
  
そんな中で  
インターネットへの接続はなくてはならない存在ですが  
環境によって接続状態は大きく変わります。  
  
地下鉄でデータをダウンロードしている最中に  
途中で接続が遮断されてダウンロードが止まってしまうこともあるかもしれません。  
  
そういった刻々と変化する状態をどうやって把握すれば良いのか  
今回はiOS11以降に導入された機能を中心に見ていきたいと思います。  
  
※   
今回iOS11以降に焦点を当てた理由として  
最新のOSの２世代前までサポートするアプリが多いと個人的には考えており  
今年の秋のiOS13バージョンアップ以降は  
iOS11以降の機能を使用していくことになると思っているからです。  
  
  
# SCNetworkReachability  
  
SCNetworkReachabilityはAppleが提供している  
SystemConfiguration.frameworkに含まれているインターフェイスの一つで  
現在のネットワークの設定状況や対象ホスト(Wifiなど)へ  
接続できるかどうかの確認をすることができます。  
  
https://developer.apple.com/documentation/systemconfiguration/scnetworkreachability-g7d  
  
iPhone SDKが公開した時には存在していませんでしたが  
開発者から多くの要望があり  
AppleはSCNetworkReachabilityを使用したサンプルコードを公開しました。  
  
https://developer.apple.com/library/archive/samplecode/Reachability/Introduction/Intro.html  
  
しかし  
これはかなり複雑なものとなっており  
この実装をラップしたライブラリが多く登場しました。  
  
# Reachability.swift  
  
Reachability.swiftがAppleのサンプルコードをSwiftに書き直したライブラリの一つで  
多くのアプリで活用されています。  
  
クロージャやローカル通知で接続状況の変化を受け取ることができます。  
  
https://github.com/ashleymills/Reachability.swift  
  
# Reachabilityへの誤解  
  
例えば  
画像を表示したい場合に  
取得先をネットワークでの接続状況で切り替えたいとします。  
  
その際に下記のようなコードを見かけることがあります。  
  
```swift

let reachability = Reachability()!
if reachability.connection != .none {
    // リモートから画像データをダウンロード
} else {
    // ローカルキャッシュから画像データを取得
}
```  
  
実はこれは間違いがあります。  
  
  
Appleのドキュメントにも下記のように書いてあります。  
  
```
“Always attempt to make a connection. 
Do not attempt to guess whether network service is available, 
and do not cache that determination.”
```  
https://developer.apple.com/library/archive/documentation/NetworkingInternetWeb/Conceptual/NetworkingOverview/WhyNetworkingIsHard/WhyNetworkingIsHard.html#//apple_ref/doc/uid/TP40010220-CH13-SW3  
  
つまり  
**接続を試して見る前に繋がっているかどうかをチェックしてはいけない**  
ということです。  
  
詳細な理由は書いていませんが  
おそらく刻々と変化する接続状態を正確に把握することは難しいからではないかと思います。  
  
これを行ってしまうと  
端末は接続が可能になっているのに  
アプリには最新の画像が表示されないという  
不具合に思われてしまうような現象が起きる可能性があります。  
  
  
# 正しい使用方法  
  
これに対してもドキュメントに記載があります。  
  
```
Important: The SCNetworkReachability API is not intended for use 
as a preflight mechanism for determining network connectivity. 
You determine network connectivity by attempting to connect. 
If the connection fails, consult the SCNetworkReachability API 
to help diagnose the cause of the failure.
```  
  
SCNetworkReachabilityは接続に失敗した原因を特定するために使用すると書いてあります。  
  
つまり  
接続を試みた後のReachabilityからのコールバックで  
リトライを試みたり  
ローカルからデータを取得する  
ように実装していくようにします。  
  
しかし  
失敗の理由はたくさんありますし  
接続状況が刻一刻と変化している可能性もあり  
自力で正確に対応するのはとても困難です。  
  
  
# waitsForConnectivityの登場  
  
そこでiOS11で登場したのが`waitsForConnectivity`プロパティです。  
  
https://developer.apple.com/documentation/foundation/urlsessionconfiguration/2908812-waitsforconnectivity  
  
このプロパティをtrueにすると  
リクエスト時に対象としているホストへの接続ができなかった場合  
Appleのフレームワーク側で  
接続が確立した際に自動でリクエストを行ってくれるようになります。  
  
こうすることで複雑な制御では任せることができるようになります。  
  
# waitsForConnectivityの設定方法  
  
ポイントがいくつかあります。  
  
## 設定タイミング  
  
URLSessionのインスタンスを**生成する前**に行う必要があります。  
  
```swift

let configuration = URLSessionConfiguration.default
configuration.waitsForConnectivity = true
let session = URLSession(configuration: configuration)
```  
  
## タイムアウトの指定  
  
いつまでの接続されない場合  
ずっと待ち続けることになります。  
  
そこでタイムアウトの時間を適切に設定する必要があります。  
デフォルトは7日です。  
  
```swift

configuration.timeoutIntervalForResource = 600 // 10 minutes (in seconds)
```  
  
## delegateで接続確立時のイベントを受け取る  
  
URLSessionのインスタンス生成時にdelegateとしてクラスを渡せます。  
  
```swift

let session = URLSession(configuration: configuration, delegate: delegate, delegateQueue: nil)
```  
※ delegateQueueにはデリゲートメソッドが呼ばれ  
コールバックが実行されるキューを指定できます。  
nilの場合は自動でシリアルキューを生成します。  
(シリアルなのは処理の順番を正しく制御するためです)  
  
デリゲートメソッドは`urlSession(_:taskIsWaitingForConnectivity:)`が呼ばれます。  
  
```swift

func urlSession(_ session: URLSession, taskIsWaitingForConnectivity task: URLSessionTask) {
    // For example, update the UI to notify the customer (with a cancel button)
    // or activate an interactive offline- or cellular-only-mode instead of blocking the UI
}
```  
  
https://developer.apple.com/documentation/foundation/urlsessiontaskdelegate/2908819-urlsession  
  
## URLSessionはdelegateに対して強参照を保持するので解放する  
  
ドキュメントにも記載がありますが  
  
```
“The session object keeps a strong reference to the delegate 
until your app exits or explicitly invalidates the session. 
If you don’t invalidate the session, your app leaks memory until it exits.”
```  
  
https://developer.apple.com/documentation/foundation/urlsession/1411597-init  
  
そこでsessionを破棄したり  
delegateを解放する場合は  
下記のメソッドを呼びます。  
  
```swift

session.invalidateAndCancel()
```  
  
## バックグラウンドでは常にtrueになる  
  
バックグラウンドで実行されるリクエストの場合は  
falseに設定をしてもtrueになり  
リクエストの再送をします。  
  
# waitsForConnectivityの落とし穴  
  
大変便利なwaitsForConnectivityですが  
気をつけたいことがあります。  
  
waitsForConnectivityが有効なのは  
リクエスト時に接続がなくその後接続が確立した場合であって  
**リクエスト時には成功したものの**  
**途中で接続が切れてしまった場合には再送してくれません**  
  
この場合には  
**NSURLErrorNetworkConnectionLost(1005)**  
が発生します。  
  
  
# NSURLErrorNetworkConnectionLost対策  
  
これが発生した時の対策として  
下記のQAへのリンクがドキュメントにも記載されています。  
https://developer.apple.com/library/archive/qa/qa1941/_index.html#//apple_ref/doc/uid/DTS40017602  
  
簡単に内容をまとめると  
  
## URLSessionの挙動  
  
2つのパターンに分かれます  
  
### リクエストの結果が変わらない場合  
  
URLSessionが自動で再送してくれることがあります。  
これは結果が変わらないので  
何度送っても問題ないと判定されているからだと思われます。  
  
GETなどはこれに該当することが多いと思います。  
  
### リクエストの結果が常に変わる場合  
  
URLSessionはリクエストを再送しません。  
もしリクエストを再送するとしたらどうなるでしょうか？  
  
例えば銀行の取引を行うAPIだった場合  
リクエストが途中で遮断された際に  
  
- サーバへのリクエストが失敗しているので安全に再送できる  
- サーバへのリクエストは成功しているけどレスポンスが返ってこないので再送するとまずい  
  
という2つの状況が考えられます。  
100万円振り込んだはずが200万円振り込まれたことになっていたら大変です。  
  
## 対応方法  
  
URLSessionと同様に2つのパターンで考えます。  
  
### リクエストの結果が変わらない場合  
  
単純にリトライをすることが可能です。  
さらに結果が変わらないので共通ロジックで処理することができます。  
  
### リクエストの結果が常に変わる場合  
  
逆に個々のリクエストを特定し  
個別に処理をしていく必要があります。  
  
### 正しいHTTPメソッドを使用すること  
  
もしデータ登録などPOSTで扱うようなものをGETで行っていた場合  
URLSessionが誤ってリクエストを送ってしまうかもしれません。  
  
このような状況が生じている場合  
**サーバー側**で対処する必要があります。  
クライアントで対処したとしても  
もし途中のプロキシサーバーでURLSessionのような挙動を実装していた場合は  
結局同じことが起きてしまいます。  
  
  
  
# もう一つの誤解  
  
これまでは接続状況の確認について見てきましたが  
もう一つ誤解していることが多いパターンがあります。  
  
それが**モバイルデータ通信の使用可否**です。  
  
例えば  
モバイルデータ通信の場合は  
ローカルキャッシュの画像データを使用したいとした時  
  
```swift

let reachability = Reachability()!
if reachability.connection == .cellular {
    // ローカルキャッシュの画像データを使用する
} else {
    // リモートから画像データをダウンロード
}
```  
  
と書いてもうまく判断できないことがあります。  
  
Appleのドキュメントにも  
  
```
“Important: Checking the reachability flag does not guarantee 
that your traffic will never be sent over a cellular connection.”
```  
  
と記載があります。  
  
https://developer.apple.com/library/archive/documentation/NetworkingInternetWeb/Conceptual/NetworkingOverview/Platform-SpecificNetworkingTechnologies/Platform-SpecificNetworkingTechnologies.html#//apple_ref/doc/uid/TP40010220-CH212-SW9  
  
# 代わりにallowsCellularAccessを使用する  
  
この場合は`allowsCellularAccess`プロパティを使用します。  
  
https://developer.apple.com/documentation/foundation/urlsessionconfiguration/1409406-allowscellularaccess  
  
  
# allowsCellularAccessの設定方法  
  
URLRequestで個々のリクエストに設定することができます。  
  
```swift

var request = URLRequest(url: URL(string: "https://www.hogehoge.com")!)
request.allowsCellularAccess = true // or false
```  
  
URLSessionに設定することもできます。  
これもインタンスの生成前に設定する必要になります。  
  
```swift

let configuration = URLSessionConfiguration.default
configuration.allowsCellularAccess = true // or false

let session = URLSession(configuration: configuration)
```  
  
補足ですが  
`waitsForConnectivity`はこのプロパティの値も考慮してくれます。  
  
  
  
# Captive Portal  
  
もう一つ大きな問題としてCaptive Portalがあります。  
  
これは  
```
HTTPクライアントがインターネットを利用する前に
ネットワーク上の特定のWEBの参照（通常は認証目的で）を強制する技術
```  
  
です。  
  
無料のWifiスポットを使用する前に  
Webブラウザにログイン画面や登録画面が表示されるあれです。  
  
これは  
**Wifiには繋がっているけれども実際に通信はできない**  
という非常にあいまいな状態になります。  
  
そしてReachabilityはこの状態を**Wifiに接続している**と判断します。  
  
# iOSはどうやって判定しているのか？  
  
iOSは下記のプロトコルに適合しています。  
Wireless Internet Service Provider roaming (WISPr 2.0)  
https://wballiance.com/glossary/  
  
これは公共のIEEE 802.11(Wi-Fi)ネットワークへ  
ユーザーがアクセスする際にどう認証するかについて  
Captive Portalを使ってログインページを表示するという  
Universal Access Methodを使うように定義しています。  
  
https://en.wikipedia.org/wiki/Universal_access_method  
  
ログインをすることで安全に接続するための認証をもらうことができます。  
  
iOSはCaptive Portalを使ってWifi接続しているかどうかを判断するために  
いくつかのテスト用のURLにアクセスを試みます。  
  
ここにはHTMLのページが用意されて  
**Success**の文字列が表示されていれば  
Wifi接続に成功していると判定しています。  
  
接続先例  
https://www.apple.com/library/test/success.html  
  
  
逆に**Success**が表示されていない場合は  
Captive Portalに乗っ取られていると判定して  
Webブラウザを開くようにします。  
  
  
しかし  
このフレームワークは開発者に提供されていません。  
  
そこでこれを実現するライブラリが存在しています。  
https://github.com/rwbutler/Connectivity  
  
  
# NWPathMonitorの登場(iOS12)  
  
Network.frameworkの中にNWPathMonitorというクラスがあります。  
https://developer.apple.com/documentation/network/nwpathmonitor  
  
このクラスを使うとネットワークの状態の変化を監視することができます。  
  
# 使い方  
  
## インスタンスの作成  
  
```swift

let monitor = NWPathMonitor()
```  
  
特定の接続ホストに限定したい場合は引数に指定します。  
  
```swift

let monitor = NWPathMonitor(requiredInterfaceType: .wifi)
```  
  
※このインスタンスへの参照はどこかに保持しておかないと  
参照が解放されてコールバックを受け取ることができなくなってしまいます  
  
`pathUpdateHandler`にNWPathを引数に受け取るクロージャを設定することで  
接続状態が変更したときに通知を受け取ることができます。  
  
https://developer.apple.com/documentation/network/nwpath  
  
```swift

monitor.pathUpdateHandler = { path in
    if path.status == .satisfied {
        print("Connected")
    }
}
```  
  
NWPath.Statusというenumが.satisfiedだと接続が利用可能な状態です。  
https://developer.apple.com/documentation/network/nwpath/status  
  
現在の接続状態を確認するためには  
NWPathMonitorの`currentPath`プロパティから取得できます。  
currentPathはOptionalです。  
https://developer.apple.com/documentation/network/nwpathmonitor/2998732-currentpath  
  
NWPath自体は  
iOS9からNetworkExtension.frameworkのクラスとして存在していましたが  
iOS12のNetwork.frameworkのNWPathはさらに色々な状態を取得できるようになりました。  
  
- `isExpensive`は現在の接続状態が高いかどうかを見てくれる。（モバイルデータ通信は負荷も通信料も高い）  
- DNS, IPv4 or IPv6に対応しているかどうかがわかる  
- `usesInterfaceType`でどの接続状態のコールバックを受け取れるかどうかなどがわかる  
  
## 通知を受け取る  
  
startメソッドを呼びます。  
  
```swift

let queue = DispatchQueue.global(qos: .background)
monitor.start(queue: queue)
```  
  
https://developer.apple.com/documentation/network/nwpathmonitor/2998737-start  
  
currentPathはstartが呼ばれるまでnilです。  
  
止めるときはcancelメソッドを呼びます。  
  
```swift

monitor.cancel()
```  
  
一度cancelを呼ぶと再開することはできずに  
再びインスタンスを生成する必要があります。  
  
# Captive Portal問題  
  
NWPathMonitorでは.satisfiedという状態が返ってくるのは  
**Captive Portalでログインをした後のみ**になりました。  
  
そのためReachabilityで生じていた問題は解消され  
ライブラリを導入する必要もなくなりました。  
  
  
# まとめ  
  
Reachabilityによくある間違いから  
iOS11で登場したwaitsForConnectivity  
Captive Portal問題  
iOS12で登場したNWPathMonitor  
  
と見ていきました。  
  
今回紹介したような機能を使って  
変化する接続状態に正しく対処することで  
ユーザーに滞りなくアプリを使ってもらえるようにしたいですね😄  
  
  
何か間違いなどございましたらご指摘していただけましたら幸いです。  
  
  
### 参考資料  
https://medium.com/@rwbutler/solving-the-captive-portal-problem-on-ios-9a53ba2b381e  
https://medium.com/@rwbutler/nwpathmonitor-the-new-reachability-de101a5a8835  
  
