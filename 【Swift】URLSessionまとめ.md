## 経緯  
  
日頃から使っているURLSessionですが、  
実際に使っている機能は結構限られており、  
いまいち使いこなせていない気がしていました。  
  
そう思ってWWDCの動画を改めて見たり、  
あれやこれやとURLSessionについて調べてみたのでまとめたいと思います。  
  
  
基本的にはAppleのドキュメントなどを参考にしています。  
[URLSession Programming Guide](https://developer.apple.com/documentation/foundation/url_loading_system)  
[URLセッションプログラミングガイド](https://developer.apple.com/jp/documentation/Cocoa/Conceptual/URLLoadingSystem/Articles/UsingNSURLSession.html)  
[Advances in Networking Part 2](https://developer.apple.com/videos/play/wwdc2017/709/)  
[Optimizing Your App for Today’s Internet](https://developer.apple.com/videos/play/wwdc2018/714/)  
  
  
  
## URLローディングシステムとは？  
  
HTTPSや自作のプロトコルが使用されているURLから特定されるリソースを非同期で提供するシステム。  
これを実現するためにURLSessionとそれに関連するクラスを使用します。  
  
  
  
## URLSessionとは?  
  
関連するネットワーク上のデータ転送処理群(=URLSessionTask)をまとめるクラス  
１つのSessionで繰り返しURLSessionTaskの作成が可能。  
  
  
  
## URLSessionTaskとは？  
  
URLで特定されるリソースを実際に取得し、アプリにデータを返却したり、  
リモートサーバからファイルのダウンロードやアップロードを行います。  
  
  
  
## URLSessionConfigurationとは？  
  
URLSessionの設定を提供するクラス。  
タイムアウト値、キャッシュやクッキーの扱い方、  
端末回線での接続の許可などが変更できます。  
  
  
  
### 補足情報 NSCopyingの挙動  
  
- URLSession, URLSessionTask  
このオブジェクトはコピーすると同じオブジェクトを取得します。  
  
- URLSessionConfiguration  
このオブジェクトはコピーする新しいコピーを取得します。  
  
<br />  
  
## URLSessionの種類  
  
### <u>shared</u>  
  
シングルトンのURLSessionインスタンス。URLSessionConfigurationは設定できません。  
基本的なリクエストはこれを使えばデータの取得が可能です。  
  
### <u>shared以外</u>  
  
下記の３つのURLConfigurationと共にURLSessionの初期化を行います。  
  
  
  
## ３つのURLSessionConfiguration  
  
### <u>.default(デフォルトセッション)</u>  
sharedと同じような設定ですが、設定のカスタマイズが可能です。  
ディスクの保存されるキャッシュ、認証情報、クッキーを使用します。  
  
### <u>.ephemeral(一時セッション)</u>  
こちらもsharedと同じような設定ですが、  
キャッシュ、クッキー、認証情報を端末内のディスクに書き込まず、メモリーに保持します。  
セッションが破棄されるとキャッシュやクッキーも破棄されます。  
(いわゆるプライベートモードです。)  
  
### <u>.background(バックグラウンドセッション)</u>  
アップロード、ダウンロードをバックグラウンド(アプリが起動していない状態)で行います。  
  
[Appleドキュメント](https://developer.apple.com/documentation/foundation/urlsessionconfiguration#1660412?changes=_3)  
  
  
  
  
  
## URLSessionConfigurationで変更できる設定  
  
一部だけ紹介します。  
  
### <u>networkServiceType</u>  
  
OSにどのようにデータを扱うかのヒントを与えます。  
これを与えることでOSが決める処理の優先度を変えることができます。  
  
例えばprefetchなど後ほど必要な情報の場合、処理の優先度が低くなり、  
取得前に処理のキャンセルを行うなどバッテリーやリソースの消費を抑える  
といったことも可能になります。  
  
  
### <u>allowsCellularAccess</u>  
  
端末回線時に通信を行うかどうかを決めます。  
  
例えば、動画のダウンロードを行っている場合、端末回線を使用すると  
かなりの通信料を消費してしまいます。  
そのような場合はfalseに設定することでwifi接続時のみ実行するなどの  
調整が行えます。  
  
デフォルトはtrueです。  
  
### <u>waitsForConnectivity(iOS11以降)</u>  
  
URLSessionの設定を満たした接続状態になっていない場合に、  
接続可能状態になるまで待つか、即座にエラーにするかを決めます。  
  
例えば、```allowsCellularAccess```がfalseで端末回線の接続しかできない場合、  
URLSessionは処理を行うことができません。  
この場合、  
URLSessionTaskDelegateの```urlSession(_:taskIsWaitingForConnectivity:)```が**一度**だけ呼ばれ、  
接続可能になるまで待ちます。  
  
```Swift
func urlSession(_ session: URLSession, taskIsWaitingForConnectivity task: URLSessionTask) {
    // 例えばアラートを出すなど
    DispatchQueue.main.async {
        let alert = UIAlertController(title: "接続エラー", message: "wifiの接続状況を確認してください", preferredStyle: .alert)
        let action = UIAlertAction(title: "OK", style: .default, handler: nil)
        alert.addAction(action)
        self.present(alert, animated: true, completion: nil)
    }
}
```  
  
今まではReachabilityなどで接続確認、リトライなど必要でしたが、このプロパティを用いることで、マニュアルでの処理が不要になります。  
  
Apple的にも常にtrueにすることを推奨しています。  
※例外として、即座に完了されなければいけないまたは中止しなければいけない株の取引などには不適切だそうです。  
  
バックグラウンドセッションの場合は自動でtrueになります。  
  
ちなみに機内モードにすると動作が簡単に確認できます。  
  
#### 注意点  
  
一度接続が確立されたあとの接続エラーに関しては、```NSURLErrorConnectionLost```などのエラーが起きます。  
  
この場合の対応として、下記のAppleの回答がありますので参考までにURLを記載します。  
(内容が難しいのですが、GETなどリソースの修正がない場合は自動でリトライ、  
POSTなどの場合は状況に応じて個々に対応していくべきというようなことかと思います。)  
  
[Apple Q&A](https://developer.apple.com/library/archive/qa/qa1941/_index.html#//apple_ref/doc/uid/DTS40017602)  
  
  
### <u>isDiscretionary</u>  
  
バックグラウンド時に有効になるプロパティで、  
trueにすることで実行タイミングをシステムに任せ、  
最適なタイミングで(リソースの使用状況やバッテリーの状況を測って)タスクを実行してくれるようになります。  
  
  
  
  
他のプロパティに関しては[Appleドキュメント](https://developer.apple.com/documentation/foundation/nsurlsessionconfiguration)をご参照ください。  
  
  
  
  
## URLSessionTaskの種類  
  
### <u>URLSessionDataTask</u>  
Dataオブジェクトを経由してデータの送受信を行います。  
主に小さいデータやサーバと対話的にデータのやり取りを実行する場合に使います。  
  
### <u>URLSessionUploadTask</u>  
DataTaskに似ていてデータのアップロードを行いますが、  
ファイルやストリーム、バックグラウンドでのアップロードも行うことができます。  
  
### <u>URLSessionDownloadTask</u>  
ファイルからデータを取得します。また、バックグラウンドでのダウンロードとアップロードができます。  
  
### <u>URLSessionStreamTask</u>  
TCP/IP接続をしてデータを取得します。URLSessionDataTaskを継承しています。  
  
  
  
## URLかURLRequestか、completionHandlerのがあるかないか  
  
各Taskのメソッドを見てみると主に引数がURLかURLRequestか、  
completionHandlerがあるかないかでメソッドが分かれています。  
  
```Swift
func dataTask(with: URL) -> URLSessionDataTask
func dataTask(with: URL, completionHandler: (Data?, URLResponse?, Error?) -> Void) -> URLSessionDataTask
func dataTask(with: URLRequest) -> URLSessionDataTask
func dataTask(with: URLRequest, completionHandler: (Data?, URLResponse?, Error?) -> Void) -> URLSessionDataTask

func downloadTask(with: URL) -> URLSessionDownloadTask
func downloadTask(with: URL, completionHandler: (URL?, URLResponse?, Error?) -> Void) -> URLSessionDownloadTask
func downloadTask(with: URLRequest) -> URLSessionDownloadTask
func downloadTask(with: URLRequest, completionHandler: (URL?, URLResponse?, Error?) -> Void) -> URLSessionDownloadTask
func downloadTask(withResumeData: Data) -> URLSessionDownloadTask
func downloadTask(withResumeData: Data, completionHandler: (URL?, URLResponse?, Error?) -> Void) -> URLSessionDownloadTask

func uploadTask(with: URLRequest, from: Data) -> URLSessionUploadTask
func uploadTask(with: URLRequest, from: Data?, completionHandler: (Data?, URLResponse?, Error?) -> Void) -> URLSessionUploadTask
func uploadTask(with: URLRequest, fromFile: URL) -> URLSessionUploadTask
func uploadTask(with: URLRequest, fromFile: URL, completionHandler: (Data?, URLResponse?, Error?) -> Void) -> URLSessionUploadTask
func uploadTask(withStreamedRequest: URLRequest) -> URLSessionUploadTask
```  
  
  
### <u>URLかURLRequestか</u>  
  
これはURLConfigurationの設定を変更するかしないかの違いによります。  
URLRequestはURLConfigurationと同じようなプロパティを持っており、  
その値を変更することでURLConfigurationの設定を上書きできます。  
  
  
### <u>completionHandlerがあるかないかの違い</u>  
  
completionHandlerを使用するとサーバと対話的に処理を行います。  
サーバに接続し、返って来たレスポンスをその場で処理する場合に使用します。  
  
一方で、completionHandlerがないパターンはデリゲートメソッドを使用します。  
セッションを作成する際にデリゲートを設定することで、  
様々な処理をデリゲートメソッドで行うことが可能です。  
  
バックグラウンドの場合はデリゲートで処理を行います。  
  
### completionHandlerのとり得る値について(2018/10/23 追記)  
  
Appleのドキュメントによると  
  
```
If the request completes successfully, 
the data parameter of the completion handler block contains the resource data, 
and the error parameter is nil. 
If the request fails, the data parameter is nil 
and the error parameter contains information about the failure. 
If a response from the server is received, 
regardless of whether the request completes successfully or fails, 
the response parameter contains that information.
```  
  
と書かれており、  
completionHandlerは、  
  
- URLSessionResponseがあるかないか  
- DataまたはErrorのどちらか一方  
  
の組み合わせで決まるようですので、  
パターンとしては、  
  
1. URLResponse と Data  
2. URLResponse と Error  
3. Dataのみ  
4. Errorのみ  
  
の４通りが考えられます。  
  
  
  
## 基本的な使い方  
  
iTunesのSearchAPIを使った例を示します。  
  
デフォルトの設定で良い場合は、  
URLSessionのシングルトンインスタンスであるsharedを使ってURLを指定してデータを取得できます。  
  
```Swift

let url = URL(string: "https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles")!

let task = URLSession.shared.dataTask(with: url) { data, response, error in
    
    // ここのエラーはクライアントサイドのエラー(ホストに接続できないなど)
    if let error = error {
        print("クライアントエラー: \(error.localizedDescription) \n")
        return
    }
    
    guard let data = data, let response = response as? HTTPURLResponse else {
        print("no data or no response")
        return
    }
    
    if response.statusCode == 200 {
        print(data)
    } else {
        // レスポンスのステータスコードが200でない場合などはサーバサイドエラー
        print("サーバエラー ステータスコード: \(response.statusCode)\n")
    }

}
task.resume()
```  
  
  
  
  
## 設定を変更したい場合  
  
URLConfigurationの設定を変更し、URLSessionインスタンスを作成します。  
  
  
```Swift

// URLSessionConfigurationの設定を変更
let config = URLSessionConfiguration.default
config.waitsForConnectivity = true

// 新しくURLSessionインスタンスを作成
let session = URLSession(configuration: config)

let url = URL(string: "https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles")!

let task = session.dataTask(with: url) { data, response, error in
    
    // ここのエラーはクライアントサイドのエラー(ホストに接続できないなど)
    if let error = error {
        print("クライアントサイドエラー: \(error.localizedDescription) \n")
        return
    }
    
    guard let data = data, let response = response as? HTTPURLResponse else {
        print("no data or no response")
        return
    }
    
    if response.statusCode == 200 {
        print(data)
        
        // ...これ以降decode処理などを行い、UIのUpdateをメインスレッドで行う

    } else {
        // レスポンスのステータスコードが200でない場合などはサーバサイドエラー
        print("サーバサイドエラー ステータスコード: \(response.statusCode)\n")
    }

}
task.resume()
```  
  
  
  
## リクエストごとに設定を変更したい場合  
  
すでに記載済みですがURLRequestのプロパティを設定することで、  
URLConfigurationの設定を上書きすることができます。  
  
  
まずURLを使った場合のURLRequestは  
  
<details>  
<summary>URLをそのまま使った結果</summary>  
  
```Swift

let url = URL(string: "https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles")!

let task = URLSession.shared.dataTask(with: url)

/*
Optional(https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles)
  ▿ some: https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles
    ▿ url: Optional(https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles)
      ▿ some: https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles
        - _url: https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles #0
          - super: NSObject
    - cachePolicy: 0
    - timeoutInterval: 60.0
    - mainDocumentURL: nil
    - networkServiceType: __ObjC.NSURLRequest.NetworkServiceType
    - allowsCellularAccess: true
    ▿ httpMethod: Optional("GET")
      - some: "GET"
    - allHTTPHeaderFields: nil
    - httpBody: nil
    - httpBodyStream: nil
    - httpShouldHandleCookies: true
    - httpShouldUsePipelining: false
▿ https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles
  ▿ url: Optional(https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles)
    ▿ some: https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles
      - _url: https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles #0
        - super: NSObject
  - cachePolicy: 0
  - timeoutInterval: 60.0
  - mainDocumentURL: nil
  - networkServiceType: __ObjC.NSURLRequest.NetworkServiceType
  - allowsCellularAccess: true
  ▿ httpMethod: Optional("GET")
    - some: "GET"
  - allHTTPHeaderFields: nil
  - httpBody: nil
  - httpBodyStream: nil
  - httpShouldHandleCookies: true
  - httpShouldUsePipelining: false
*/
dump(task.currentRequest)

```  
</details>  
  
  
  
同様にURLRequestを使用した場合、  
  
<details>  
<summary>URLRequestを使った結果</summary>  
  
```Swift

let url = URL(string: "https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles")!

var request = URLRequest(url: url)  // inspect with Show Result button

let taskWithRequest = URLSession.shared.dataTask(with: request)

/*
Optional(https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles)
  ▿ some: https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles
    ▿ url: Optional(https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles)
      ▿ some: https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles
        - _url: https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles #0
          - super: NSObject
    - cachePolicy: 0
    - timeoutInterval: 60.0
    - mainDocumentURL: nil
    - networkServiceType: __ObjC.NSURLRequest.NetworkServiceType
    - allowsCellularAccess: true
    ▿ httpMethod: Optional("GET")
      - some: "GET"
    - allHTTPHeaderFields: nil
    - httpBody: nil
    - httpBodyStream: nil
    - httpShouldHandleCookies: true
    - httpShouldUsePipelining: false
▿ https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles
  ▿ url: Optional(https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles)
    ▿ some: https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles
      - _url: https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles #0
        - super: NSObject
  - cachePolicy: 0
  - timeoutInterval: 60.0
  - mainDocumentURL: nil
  - networkServiceType: __ObjC.NSURLRequest.NetworkServiceType
  - allowsCellularAccess: true
  ▿ httpMethod: Optional("GET")
    - some: "GET"
  - allHTTPHeaderFields: nil
  - httpBody: nil
  - httpBodyStream: nil
  - httpShouldHandleCookies: true
  - httpShouldUsePipelining: false
*/
dump(taskWithRequest.currentRequest)
```  
</details>  
  
  
設定値は同じになります。  
  
  
  
URLRequestの設定値を変えると  
  
<details>  
<summary>URLRequestの設定値を変えた結果</summary>  
  
```Swift

let url = URL(string: "https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles")!

var request = URLRequest(url: url)  // inspect with Show Result button

request.cachePolicy = .reloadIgnoringLocalAndRemoteCacheData
request.networkServiceType = .background
request.allowsCellularAccess = false

let taskWithRequest = URLSession.shared.dataTask(with: request)

/*
Optional(https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles)
  ▿ some: https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles
    ▿ url: Optional(https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles)
      ▿ some: https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles
        - _url: https://itunes.apple.com/search?media=music&entity=song&lang=ja_jp&term=beatles #0
          - super: NSObject
    - cachePolicy: 1
    - timeoutInterval: 60.0
    - mainDocumentURL: nil
    - networkServiceType: __ObjC.NSURLRequest.NetworkServiceType
    - allowsCellularAccess: false
    ▿ httpMethod: Optional("GET")
      - some: "GET"
    - allHTTPHeaderFields: nil
    - httpBody: nil
    - httpBodyStream: nil
    - httpShouldHandleCookies: true
    - httpShouldUsePipelining: false
*/
dump(taskWithRequest.currentRequest)
```  
</details>  
  
リクエストの設定が変わりました。  
  
  
  
## DataTaskを使用したPOSTやPUTの場合  
  
`httpBody`プロパティへの値の変更や`httpMethod`プロパティの変更などが必要になるため  
URLRequestが必須となります。  
  
```Swift

let url = URL(string: "https://example.com/messages")!

var request = URLRequest(url: url)  // inspect with Show Result button

// POSTにメソッド設定
request.httpMethod = "POST"

// ヘッダーにcontent-typeを設定(JSONを送るのでapplication/jsonにしている)
request.addValue("application/json", forHTTPHeaderField: "content-type")

// Message構造体を作成
struct Message: Codable {
    let user: String
    let content: String
}
let encoder = JSONEncoder()
let message = Message(user: "me", content: "Hello!!!")
do {
    // EncodeしてhttpBodyに設定
    let data = try encoder.encode(message)
    request.httpBody = data
} catch let encodeError as NSError {
    print("Encoder error: \(encodeError.localizedDescription)\n")
}

let task = URLSession.shared.dataTask(with: request) { data, response, error in
    
    if let error = error {
        print("クライアントエラー: \(error.localizedDescription) \n")
        return
    }

    // EncodeしてhttpBodyに設定
    guard let data = data, let response = response as? HTTPURLResponse,
        response.statusCode == 201 else {
            print("No data or statusCode not CREATED")
            return
    }
    print("success")
}
task.resume()


print(task.currentRequest.httpBody) // nil
```  
  
最後にtask.currentRequest.httpBodyを出力してみると、  
```httpBody```プロパティはnilになります。これはデータがサーバに送られたためです。




## アップロード

[Uploading Data to a Website](https://developer.apple.com/documentation/foundation/url_loading_system/uploading_data_to_a_website)
を参考にしています。


リクエストのボディコンテンツを３種類の方法でアップロードすることが可能です。

- Dataオブジェクトを使用する uploadTask(with:from:), uploadTask(with:from:completionHandler:), 
- ファイルを使用する uploadTask(with:fromFile:),uploadTask(with:fromFile:completionHandler:)
- ストリームを使用する uploadTask(withStreamedRequest:)

この場合URLSessionUploadTaskを使用します。
URLSessionUploadTaskは
対話的に処理する方法とバックグラウンドの両方に対応しています。

下記ではDataオブジェクトを使用します。

### アップロードするDataを用意する

アップロード可能なデータとしては、
画像データをJPEGやPNGデータ形式に変換したり、
文字列データをUTF-8でエンコードしたり、
Encodableに適合した構造体をJSONEncoderでJSON形式に変換するなど
多様なデータをアップロードすることができます。(サーバが対応していれば)

### URLRequestを用意する

URLSessionUploadTaskはURLRequestを引数に取ります。
URLRequestでは```httpMethod```プロパティをサーバの要求に合わせてPOSTまたはPUTにし、
ヘッダーにContent-typeなどを設定します。

※URLSessionは設定されたデータからサイズを自動で計算するため、
Content-Lengthの設定は不要です。


```Swift  
  
let url = URL(string: "https://example.com/post")!  
var request = URLRequest(url: url)  
request.httpMethod = "POST"  
  
// JSONの場合はapplication/json  
request.setValue("application/json", forHTTPHeaderField: "Content-Type")  
```

### URLSessionUploadTaskの作成と開始

設定したURLRequestからURLSessionUploadTaskを作成し、処理を開始します。
下記の例では、completionHandlerを用いてレスポンスを処理しています。

```Swift  
  
let task = URLSession.shared.uploadTask(with: request, from: uploadData) { data, response, error in  
    if let error = error {  
        print ("クライアントエラー: \(error)")  
        return  
    }  
    guard let response = response as? HTTPURLResponse,  
        (200...299).contains(response.statusCode) else {  
        print ("サーバエラー")  
        return  
    }  
    if let mimeType = response.mimeType,  
        mimeType == "application/json",  
        let data = data,  
        let dataString = String(data: data, encoding: .utf8) {  
        print ("got data: \(dataString)")  
    }  
}  
task.resume()  
  
```

### デリゲートを使用することもできる

completionHandlerがないメソッドを呼んだ場合、
デリゲートを実装して処理をすることができます。

例えばアップロードの進行状況を示したい場合、URLSessionDataDelegateの
```urlSession(_ session: URLSession, dataTask: URLSessionDataTask, didReceive data: Data)```  
を用いることで進捗状況が把握できます。  
  
※iOS11よりURLSessionTaskでprogressプロパティが使えるようになりました。  
  
```Swift
func urlSession(_ session: URLSession, dataTask: URLSessionDataTask, didReceive data: Data) {

    // fractionCompletedは0.0~1.0の間で完了状態を示す
    let progress = Float(dataTask.progress.fractionCompleted)
    
    DispatchQueue.main.async {
        progressView.setProgress(progress, animated: true)
    }
}
```  
  
[progress](https://developer.apple.com/documentation/foundation/progress)  
  
  
### URLSessionStreamTaskを使用する場合の注意点  
  
- ストリームを使用する場合、  
サーバがリクエストする可能姓の追加のヘッダ情報を追加する必要があります。  
（コンテンツタイプやコンテンツの長さなど）  
  
- セッションはデータを巻き戻して読み込み直すことができないため、  
リトライする場合などのために  
URLSessionTaskDelegate```urlSession(_:task:needNewBodyStream:)```  
を実装する必要があります。  
  
```Swift
func urlSession(_ session: URLSession, task: URLSessionTask, needNewBodyStream completionHandler: @escaping (InputStream?) -> Void) {
    self.closeStream()

    var inStream: InputStream? = nil
    var outStream: OutputStream? = nil
    Stream.getBoundStreams(withBufferSize: 4096, inputStream: &inStream, outputStream: &outStream)
    self.outputStream = outStream

    completionHandler(inStream)
}
```  
  
  
## ダウンロード(フォアグラウンド)  
  
流れとしては他の処理と同様ですが、  
  
URLSessionDownloadTaskには  
** @escaping (Data?) -> Void)```**  
```cancel(byProducingResumeData completionHandler: @escaping (Data?) -> Void)```
があるため、処理の一時停止からダウンロードを再開するための処理も考えたいと思います。


### ダウンロードの状況を管理するクラスを用意する

例えば複数同時にダウンロードをする場合など、
それぞれのダウンロード状況を管理する必要があります。
(どのくらいダウンロードできているのか、そもそも今ダウンロード中なのかなど)

例えば下記のようなクラスを用意します。

```Swift  
  
final class Download: NSObject {  
  
  var url: URL  
  
  // 現在ダウンロード中か  
  var isDownloading = false   
  
  // 進行状況  
  var progress: Float = 0  
  
  var task: URLSessionDownloadTask?  
  
  // ダウンロードが中断された時のダウンロード済のデータ  
  var resumeData: Data?  
  
  init(url: URL) {  
    self.url = url  
  }  
}  
```

### URLSessionDownloadTaskを用意して各処理を実装する。

他の処理と同様にTaskを作成し、処理を開始します。

ダウンロードは自動でダウンロードが開始されるというよりは、
ボタンを押してダウンロードを開始するということが多いと思いますので、
ボタンが押された時に処理を開始する例を記載しました。

また、ダウンロードのキャンセルや一時停止、再開の処理も合わせて記載しています。

※サンプルで作成したアプリの処理の一部を抜粋しています。


```Swift  
  
let url = URL(string: "https://audio-ssl.itunes.apple.com/apple-assets-us-std-000001/Music4/v4/a0/05/df/a005df47-d4d5-1fd1-eefc-77553fa59689/mzaf_5617395189778548804.plus.aac.p.m4a")!  
  
// ダウンロード処理中のDownloadクラスを保持  
var activeDownloads: [URL: Download] = [:]  
  
// ダウンロード開始処理  
func startDownload() {  
  
  let download = Download(url: url)  
  download.task = URLSession.shared.downloadTask(with: url) { location, response, error in  
    
    // ダウンロードデータの一時保存URL  
    print(location)  
  
    // ...ローカルファイルへ保存を行う  
  }  
  download.task!.resume()  
  download.isDownloading = true  
  activeDownloads[download.url] = download  
}  
  
// 一時停止処理  
func pauseDownload() {  
    guard let download = activeDownloads[url] else { return }  
    if download.isDownloading {  
        download.task?.cancel(byProducingResumeData: { data in  
        // ここで途中までダウンロードしていたデータを保持しておく  
        download.resumeData = data  
        })  
        download.isDownloading = false  
    }  
}  
  
// キャンセル処理  
func cancelDownload() {  
    if let download = activeDownloads[url] {  
        download.task?.cancel()  
        activeDownloads[url] = nil  
    }  
}  
  
// ダウンロード再開処理  
func resumeDownload() {  
    guard let download = activeDownloads[url] else { return }  
  
    // pause時に保存していたresumeDataがある場合はそこから続きのダウンロードを行う  
    if let resumeData = download.resumeData {  
        download.task = URLSession.shared.downloadTask(withResumeData: resumeData)  
        download.task!.resume()  
        download.isDownloading = true  
    } else {  
        download.task = URLSession.shared.downloadTask(with: download.url)  
        download.task!.resume()  
        download.isDownloading = true  
    }  
}  
```




## バックグラウンド

バックグラウンドでデータの送受信を行う場合、デリゲートメソッドを実装します。
あるデリゲートメソッドは呼ばれる順番が決まっており、
それぞれのタイミングで必要な処理というものもあります。


[Downloading Files in the Background](https://developer.apple.com/documentation/foundation/url_loading_system/downloading_files_in_the_background)
を参考にしています。

### <u>バックグラウンド用のURLSessionを作成する</u>

1. .backgroundのURLSessionConfigurationインスタンスを作成する。この際にidentifierを設定します。
※システムはこのidentifierを用いてTaskとURLSessionを紐づけているため、
１つのURLConfigurationに対して１つのURLSessionしか作成できないということになります。


2. バックグランドタスク完了後にシステムがアプリを起動するためには、
URLSessionConfigurationの```sessionSendsLaunchEvents```プロパティがtrueにする必要があります(デフォルトはtrue)


3. 急を要さないタスクに関しては、URLSessionConfigurationの```isDiscretionary```プロパティをtrueにします。
こうすることでシステムが都合の良いタイミングでタスクを実行するようになります。(都合が良いというのはネットワークの接続状況などを考慮してくれるということです)

4. URLSessionインスタンス作成時に上記で設定したURLConfigurationを渡します。
また、バックグラウンド時のイベントを受け取るためにデリゲートを設定しなければなりません。


```Swift  
private lazy var urlSession: URLSession = {  
    let config = URLSessionConfiguration.background(withIdentifier: "BackgroundSession")  
    config.isDiscretionary = true  
    config.sessionSendsLaunchEvents = true  
    return URLSession(configuration: config, delegate: self, delegateQueue: nil)  
}()  
```

#### delegateQueueに関して

delegateQueueにnilを設定すると、アプリはシリアルなOperation Queueを作成します。
このQueueの中で全てのデリゲートメソッドとcompletionHandlerは実行されます。
独自で設定する場合にもシリアルなOperation Queueを作成して渡します。
シリアルでないと、処理の実行順序が変わってしまい結果が異なってきてしまう可能性があります。

ちょっと複雑なのですが、
このQueueが制御するのは**デリゲートメソッドとcompletionHandler**で
URLSessionのネットワーク通信の同時接続を制御するものではありません。

URLSessionの同時接続は```httpMaximumConnectionsPerHost```プロパティの値を変更します。iOSのデフォルトは4です。
※注意点としてこの値は１セッションごとの値であり、仮に複数セッションを使っていた場合は、この上限を超えます。

また、アプリが提供するOperation Queueは並行処理の上限を設定することができません。
例えば、10Mの画像を50個ダウンロード使用とする場合、同時並行処理の上限がないと、
タスクの完了に時間がかかりタイムアウトになる可能性があります。
最適な数としては4か５と下記のURLでは記載がありますが、
それぞれの使用状況に合わせて検討する必要があるのかと思います。


さらに、ある処理を行った後にある処理を行いたい場合などは
Operationのdependencyを活用することで処理順序の制御ができるようになります。

参考にした情報:
[https://stackoverflow.com/questions/21918722/how-do-i-use-nsoperationqueue-with-nsurlsession](https://stackoverflow.com/questions/21918722/how-do-i-use-nsoperationqueue-with-nsurlsession)



###  <u>DonwloadTaskを作成する</u>

1. completionHandlerのない```downloadTask(with:)```でURLDownloadTaskを作成します。

2. (iOS11以降)```earliestBeginDate```プロパティを設定するとダウンロードをスケジューリングできます。
設定された日時以降に処理が開始されるようになります。
ただし、本当の開始日時はシステムが判断するので正確に設定した時間に開始される訳ではありません。
さらにURLSessiontaskDelegateの```urlSession(_:task:willBeginDelayedRequest:completionHandler:)```で
本当に処理が開始される時点で挙動の変更を行うことも可能です。
これはその時点ではすでに情報が古くなっている可能性がある場合など
最新の情報に入れ換えが可能になります。

```Swift  
func urlSession(_ session: URLSession, task: URLSessionTask, willBeginDelayedRequest request: URLRequest, completionHandler: @escaping (URLSession.DelayedRequestDisposition, URLRequest?) -> Void) {  
    
    var updatedRequest = request   
  
    // 何かヘッダーの値を変更する  
    updatedRequest.addValue("...", forHTTPHeaderField: "...")  
    completionHandler(.useNewRequest, updatedRequest)  
  
    // .continueLoadingはそのまま継続  
    // .cancelならば中止  
}  
```

ちなみに、バックグラウンドセッション以外に設定しても影響は何もありません。


[earliestBeginDate](https://developer.apple.com/documentation/foundation/urlsessiontask/2873413-earliestbegindate)


3. (iOS11以降)システムに効率良くスケジューリングしてもらうために、
```countOfBytesClientExpectsToSend```  
と  
```countOfBytesClientExpectsToReceive```
を設定します。
送受信の上限のバイト数の推定値を設定します。
設定がない場合は```NSURLSessionTransferSizeUnknown```が設定されます。
(要はシステムが勝手に決めるよ、と)
※推定値にはボディデータに加えてヘッダーのバイト数も計算に入れる必要があります。

4. resume()でタスクを開始します。

```Swift  
let backgroundTask = urlSession.downloadTask(with: url)  
// 1時間後にダウンロードを開始する予定  
backgroundTask.earliestBeginDate = Date().addingTimeInterval(60 * 60)  
// 最大200バイトのデータを送る予定  
backgroundTask.countOfBytesClientExpectsToSend = 200  
// 最大500キロバイトのデータを受け取る予定  
backgroundTask.countOfBytesClientExpectsToReceive = 500 * 1024  
// タスク開始  
backgroundTask.resume()  
```



### <u>アプリが再起動した際の処理を定義する</u>

- アプリがバックグランド状態の場合、システムはアプリを停止状態にします。
(バックグラウンドタスクは別プロセスで実行しています)

- ダウンロードが完了した際に、システムはアプリを再起動させ、
UIApplicationの```application(_:handleEventsForBackgroundURLSession:completionHandler:)```を呼び出します。

この際にやるべきことが２つあります。

1. 引数の最後のcompletionHandlerはappDelegateなどの変数として保存する必要があります。
これはcompletionHandlerを呼ぶことでシステムにバックグラウンド処理が完了したことを知らせ、
システムはアプリから最新のスナップショットを取得し、アプリスウィッチャーの画像を更新します。
※注意点

この時点でcompletionHandlerを呼んでしまうとその後のデリゲートメソッドは呼ばれなくなります。
また、新しくURLSessionを作成する前に保存する必要があります(とドキュメントに書いてあります)。
これはURLSessionDelegateの```urlSessionDidFinishEvents(forBackgroundURLSession:)```
の中で使用します。

2. 引数に設定されているidentifierからバックグラウンドセッションを再作成する必要があります。
ただし、他の場所、例えば起動後最初に表示するViewControllerなどでバックグラウンドセッションの再作成が完了している場合は、
すでにタスクとセッションの紐付けは完了しているため不要です。


#### 注意点

ユーザが意図的にアプリを終了(アプリスウィッチャーでタスクキル)させた場合、バックグランド処理は全てキャンセルされ、アプリの再起動も行われません。


```Swift  
  
func application(_ application: UIApplication, handleEventsForBackgroundURLSession identifier: String, completionHandler: @escaping () -> Void) {  
    backgroundSessionCompletionHandler = completionHandler  
  
    // ここは不要な場合が多い  
    let configuration = URLSessionConfiguration.background(withIdentifier: identifier)  
    let downloadssession = URLSession(configuration: configuration, delegate: SearchViewController(), delegateQueue: nil)  
    let downloadService = DownloadService()  
    downloadService.downloadsSession = downloadssession  
}  
```



### <u>バックグランド処理完了後の処理を定義する</u>

- 全てのイベントが完了したあと、
URLSessionDelegateの```urlSessionDidFinishEvents(forBackgroundURLSession:)```が呼ばれます。
この際上記で保存したcompletionHandlerを最後の呼ぶことでシステムにバックグランドタスクが完了したことを伝えます。
Appleのドキュメントによると、これを呼ぶことでシステムがアプリを再度一時停止しても良いということを知らせる役割があるようです。

#### 注意点

保存しているcompletionHandlerはUIKitに含まれているため、メインスレッドで呼び出す必要があります。

```Swift  
func urlSessionDidFinishEvents(forBackgroundURLSession session: URLSession) {  
    DispatchQueue.main.async {  
        guard let appDelegate = UIApplication.shared.delegate as? AppDelegate,  
            let backgroundCompletionHandler =  
            appDelegate.backgroundCompletionHandler else {  
                return  
        }  
        backgroundCompletionHandler()  
    }  
}  
```



### <u>ダウンロード完了後の完了処理、エラー処理を定義する</u>

アプリ再開後(またはフォアグラウンドに移行後)、
URLSessionDownloadDelegateの```urlSession(_:downloadTask:didFinishDownloadingTo:)```が呼ばれます。

引数のdownloadTaskの```response```を見ることで、
サーバからエラーが返ってきていないかなどの判定をすることができます。
(※クライアントサイドのエラーはここでは取得できません)

成功していた場合、引数```location```にデータをダウンロードした一時保存ファイルのURLが格納されているため、そのデータをコピーして別ファイルに保存するようにします。

#### 注意点
locationのURLが有効なのはデリゲートメソッド内のみです。

```Swift  
func urlSession(_ session: URLSession, downloadTask: URLSessionDownloadTask,  
                didFinishDownloadingTo location: URL) {  
    guard let httpResponse = downloadTask.response as? HTTPURLResponse,  
        (200...299).contains(httpResponse.statusCode) else {  
            print ("server error")  
            return  
    }  
    do {  
        let documentsURL = try  
            FileManager.default.url(for: .documentDirectory,  
                                    in: .userDomainMask,  
                                    appropriateFor: nil,  
                                    create: false)  
        let savedURL = documentsURL.appendingPathComponent(  
            location.lastPathComponent)  
        try FileManager.default.moveItem(at: location, to: savedURL)  
    } catch {  
        print ("file error: \(error)")  
    }  
}  
```

#### クライアントサイドのエラーを取得するには？

URLSessionTaskDelegateの```urlSession(_:task:didCompleteWithError:)```で処理をします。

このメソッドは種類に関わらずタスクが完了した際に呼ばれ、
もし最後の引数のerrorがnon-nilであった場合にエラーを処理します。




### <u>バックグランド処理で気をつけなければならないこと</u>

バックグランドでの処理はアプリのプロセスとは別で行われます。
そのため、システムのリソースの状態などにより以下のような制限を考える必要があります。


- URLSessionは全てのデータ転送に対して必ずデリゲートを設定する必要があります。

- HTTPまたはHTTPSのみをサポートしています。

- サーバからリダイレクト要求が来た場合はそのままリダイレクトされます。
```urlSession(_:task:willPerformHTTPRedirection:newRequest:completionHandler:)```を実装しても呼ばれません。  
  
- アップロードを行う際はファイルからのデータ転送しかサポートしていません。  
(Dataやstreamはアプリが終了した時点で失敗します。)  
  
  
  
## サンプル  
  
下記のURLを参考にダウンロードやの動作確認用にサンプルを作成してみました。  
[https://github.com/stzn/URLSessionSample](https://github.com/stzn/URLSessionSample)  
  
  
参考URL:[https://www.raywenderlich.com/158106/urlsession-tutorial-getting-started](https://www.raywenderlich.com/158106/urlsession-tutorial-getting-started)  
  
  
  
  
## URLSessionTaskMetrics  
  
iOS 10からURLSessionTaskMetricsクラスが追加され、  
通信に関する様々な情報を取得することができるようになりました。  
  
[URLSessionTaskMetrics](https://developer.apple.com/documentation/foundation/urlsessiontaskmetrics)  
  
### 3つのプロパティを持っています  
  
```Swift
// URLSessionTaskTransactionMetricsクラスの配列
var transactionMetrics: [URLSessionTaskTransactionMetrics]

// タスクがインスタンス化されてから完了するまでの時間
var taskInterval: NSDateInterval

// タスク実行中にリダイレクトされた回数
var redirectCount: Int
```  
  
### URLSessionTaskTransactionMetrics  
  
３つに分かれています。  
  
1. リクエストとレスポンス  
  
```Swift
var request: URLRequest
var response: URLResponse
```  
  
  
2.それぞれの処理の開始終了日時  
  
URLSessionの通信は下記の図のような流れで行われ、  
それぞれのタイミングの開始終了日時を取得できます。  
  
  
![23287374-c86a-4b5c-80d4-c1d596f20f85.png](/image/b980bfbe-16dd-7550-417a-80c240460dd0.png)  
  
  
```Swift
// タスクがリソースの取得を開始した時間
var fetchStartDate: Date?

// タスクがリソースの名前解決をする直前の時間
var domainLookupStartDate: Date?

// タスクがリソースの名前解決を完了した後の時間
var domainLookupEndDate: Date?

// タスクがサーバとのTCPコネクションを確立する直前の時間
var connectStartDate: Date?

// タスクがサーバとのTLSハンドシェイクを開始する直前の時間
var secureConnectionStartDate: Date?

// タスクがサーバとのTLSハンドシェイクを完了した後の時間
var secureConnectionEndDate: Date?

// タスクがサーバとの接続を完了した後の時間
var connectEndDate: Date?

// タスクがサーバへのリソースのリクエストを開始する直前の時間
var requestStartDate: Date?

// タスクがサーバへのリソースのリクエストを完了した後の時間
var requestEndDate: Date?

// タスクがサーバから最初のレスポンスのバイトを取得した後の時間
var responseStartDate: Date?

// タスクがサーバから最後のレスポンスのバイトを取得した後の時間
var responseEndDate: Date?
```  
  
  
3.プロトコルとコネクション  
  
```Swift
// 使用されたプロトコル名
var networkProtocolName: String?

// プロキシが使用されたか
var isProxyConnection: Bool

// 永続コネクションが使用されたか
var isReusedConnection: Bool

// リソースの種類
var resourceFetchType: URLSessionTaskMetrics.ResourceFetchType

enum ResourceFetchType {
    case unknown // 不明
    case networkLoad // ネットワークから取得
    case serverPush // サーバプッシュから取得
    case localCache // ローカルキャッシュから取得
}
```  
  
### 実際に取得する  
  
データの取得方法は簡単で  
URLSesssionTaskDelegateの```urlSession(URLSession:task:didFinishCollecting)```を実装するだけです。  
  
<details>  
<summary>取得例</summary>  
  
  
```Swift
func urlSession(_ session: URLSession, task: URLSessionTask,
                didFinishCollecting metrics: URLSessionTaskMetrics) {
    print(metrics)
}
```  
  
  
  
```
(Task Interval) <_NSConcreteDateInterval: 0x6000006208e0> (Start Date) 2018-07-03 23:15:40 +0000 + (Duration) 0.385012 seconds = (End Date) 2018-07-03 23:15:40 +0000
(Redirect Count) 0
(Transaction Metrics) (Request) <NSURLRequest: 0x600000200300> { URL: https://audio-ssl.itunes.apple.com/apple-assets-us-std-000001/Music/v4/15/84/c1/1584c1fc-687f-88d1-001f-468b26cc6a6e/mzaf_949527388120129365.plus.aac.p.m4a }
(Response) (null)
(Fetch Start) 2018-07-03 23:15:40 +0000
(Domain Lookup Start) (null)
(Domain Lookup End) (null)
(Connect Start) (null)
(Secure Connection Start) (null)
(Secure Connection End) (null)
(Connect End) (null)
(Request Start) 2018-07-03 23:15:40 +0000
(Request End) 2018-07-03 23:15:40 +0000
(Response Start) 2018-07-03 23:15:40 +0000
(Response End) (null)
(Protocol Name) (null)
(Proxy Connection) NO
(Reused Connection) YES
(Fetch Type) Unknown

(Request) <NSURLRequest: 0x6040000156e0> { URL: https://audio-ssl.itunes.apple.com/apple-assets-us-std-000001/Music/v4/15/84/c1/1584c1fc-687f-88d1-001f-468b26cc6a6e/mzaf_949527388120129365.plus.aac.p.m4a }
(Response) <NSHTTPURLResponse: 0x604000434660> { URL: https://audio-ssl.itunes.apple.com/apple-assets-us-std-000001/Music/v4/15/84/c1/1584c1fc-687f-88d1-001f-468b26cc6a6e/mzaf_949527388120129365.plus.aac.p.m4a } { Status Code: 200, Headers {
    "Accept-Ranges" =     (
        bytes
    );
    "Access-Control-Allow-Origin" =     (
        "*"
    );
    Connection =     (
        "keep-alive"
    );
    "Content-Length" =     (
        1000714
    );
    "Content-Type" =     (
        "audio/x-m4a"
    );
    Date =     (
        "Tue, 03 Jul 2018 23:15:40 GMT"
    );
    Etag =     (
        "\"0c09d3cd4e5c8cca49bb08ba1e056d30\""
    );
    "Last-Modified" =     (
        "Fri, 28 Mar 2014 00:28:41 GMT"
    );
    "X-Cache" =     (
        "TCP_HIT from a72-246-188-22.deploy.akamaitechnologies.com (AkamaiGHost/9.3.4.1.2-22867550) (-)"
    );
} }
(Fetch Start) 2018-07-03 23:15:40 +0000
(Domain Lookup Start) 2018-07-03 23:15:40 +0000
(Domain Lookup End) 2018-07-03 23:15:40 +0000
(Connect Start) 2018-07-03 23:15:40 +0000
(Secure Connection Start) 2018-07-03 23:15:40 +0000
(Secure Connection End) 2018-07-03 23:15:40 +0000
(Connect End) 2018-07-03 23:15:40 +0000
(Request Start) 2018-07-03 23:15:40 +0000
(Request End) 2018-07-03 23:15:40 +0000
(Response Start) 2018-07-03 23:15:40 +0000
(Response End) 2018-07-03 23:15:40 +0000
(Protocol Name) http/1.1
(Proxy Connection) NO
(Reused Connection) NO
(Fetch Type) Network Load
```  
</details>  
  
  
  
## キャッシュについて  
  
[Accessing Cached Data](https://developer.apple.com/documentation/foundation/url_loading_system/accessing_cached_data)  
を参考にしています。  
  
URL Loading Systemでは、メモリとディスクの両方にレスポンスをキャッシュします。  
ネットワーク通信からのキャッシュはURLCacheクラスも用い、シングルトンインスタンスのsharedに直接アクセスできます。  
また、独自にインスタンスを作成することもでき、URLSessionConfigurationに設定をすることで利用可能です。  
  
キャッシュが可能なのはHTTP、HTTPSプロトコルのみです。  
  
### Cache Policyの設定  
  
Cache Policyはリクエストごとに設定することができます。  
どのように動作するかは、URLRequest.CachePolicy次第で変わってきます。  
各キャッシュの設定による動作の違いは下記のようになります。  
  
  
|キャッシュポリシー名  |ローカルキャッシュの使用  |ネットワークからの取得 |  
|---|---|---|  
|reloadIgnoringLocalCacheData  |使用しない  |必ず行う  |  
|returnCacheDataDontLoad  |必ず使用する  |行わない  |  
|returnCacheDataElseLoad  |最初に使用可能か試す  |ローカルキャッシュが使用できない場合に行う  |  
|useProtocolCachePolicy   |プロトコル次第  |プロトコル次第  |  
  
#### useProtocolCachePolicyについて  
  
下記のような流れで動作します。  
  
1. キャッシュされたレスポンスが存在しない場合、ソース元からデータを取得します。  
2. 上記以外の場合、キャッシュの再検証が不要であったり、キャッシュが最新である(有効期限が切れていない)場合は  
キャッシュされたレスポンスを返します。  
3. キャッシュされたレスポンスが有効期限切れ、または再検証の必要がある場合、ソース元にレスポンスに変更がないか  
HEADリクエストを送ります。変更があった場合はデータを取得、ない場合はキャッシュを返します。  
  
  
  
![46788d50-fb95-48f2-8360-b8c2a4bf1648.png](/image/56868765-47db-4536-7461-eb5b95338b85.png)  
  
  
  
  
#### HTTPのByte Rangeリクエストの場合  
  
いわゆるバイトを分割してダウンロードをするような場合は  
reloadIgnoringLocalCacheDataを使うようにすることとドキュメントには記載があります。  
  
#### usePrococolCachePolicyのHTTPSのキャッシュの注意点  
  
この場合、レスポンスはディスクに保存されますが、  
セキュリティ的に保存しておくのがよろしくないデータも保存されてしまう可能性があります。  
これを避けるために```storeCachedResponse(_:for:)```を使って独自にキャッシュを行う必要があります。  
  
URLSessionDataDelegateの```urlSession(_:dataTask:willCacheResponse:completionHandler:)```で  
レスポンスごとのキャッシュを処理することが可能です。  
注意点として、このメソッドが呼ばれるのはdefaultセッションのdataTaskとuploadTaskのみです。  
  
引数にはCachedURLResponseオブジェクトとcompletionHandlerが渡ってきます。  
このcompletionHandlerは必ず呼ばなければならず、引数には下記の３つパターンがあります。  
  
- 引数で渡ってきたCachedURLResponseオブジェクトをそのまま渡す  
- nilを渡す(キャッシュを防ぐ)  
- ```storagePolicy```や```userInfo```を修正した新しいCachedURLResponseオブジェクトを渡す  
  
下記の例ですと、HTTPS通信の場合はメモリーにのみキャッシュをするようにしています。  
  
```Swift
func urlSession(_ session: URLSession, dataTask: URLSessionDataTask,
                willCacheResponse proposedResponse: CachedURLResponse,
                completionHandler: @escaping (CachedURLResponse?) -> Void) {
    if proposedResponse.response.url?.scheme == "https" {
        let updatedResponse = CachedURLResponse(response: proposedResponse.response,
                                                data: proposedResponse.data,
                                                userInfo: proposedResponse.userInfo,
                                                storagePolicy: .allowedInMemoryOnly)
        completionHandler(updatedResponse)
    } else {
        completionHandler(proposedResponse)
    }
}
```  
  
  
  
  
## Cookieについて  
  
ステートレスなREST APIを使用する場合など、  
URLリクエストの状態を確認するためにcookieを使って認証の確認などを行います。  
※iOSのアプリケーション間でCookieを共有することはありません。  
  
HTTPCookieクラスとHTTPCookieStorageクラスを使います。  
  
[HTTPCookie](https://developer.apple.com/documentation/foundation/httpcookie)  
  
[HTTPCookieStorage](https://developer.apple.com/documentation/foundation/httpcookiestorage)  
  
  
HTTPCookieStorageはシングルトンのsharedインスタンスを持っており、  
Cookieの取得、保存、削除などが可能です。  
  
  
例えば、レスポンスで返ってきたCookieを保存する場合  
  
```Swift
if let fields = response.allHeaderFields as? [String: String], let url = response.url {
    for cookie in HTTPCookie.cookies(withResponseHeaderFields: fields, for: url) {
        HTTPCookieStorage.shared.setCookie(cookie)
    }
}
```  
  
取得をする場合  
  
```Swift
if let cookies = HTTPCookieStorage.shared.cookies(for: url) {
    for cookie in cookies {
        // 特定の名前のCookieを取得
        if (cookie.name == name) {
            return cookie.value
        }
    }
}
```  
  
削除する場合  
  
```Swift
if let cookies = HTTPCookieStorage.shared.cookies(for: url) {
    for cookie in cookies {
        // 特定の名前のCookieを削除
        if (cookie.name == name) {
           HTTPCookieStorage.shared.deleteCookie(cookie) 
        }
    }
}
```  
  
```Swift
// 指定した日付以降に保存されたクッキーを削除
HTTPCookieStorage.shared.removeCookies(since: Date()) 

```  
  
  
  
  
## 認証について  
  
[Handling an Authentication Challenge](https://developer.apple.com/documentation/foundation/url_loading_system/handling_an_authentication_challenge)  
を参考にしています。  
  
URLSessionTaskを使ってサーバにリクエストを送った際に、サーバから認証要求が返ってくることがあります。  
URLSessionTaskは自動でこの要求に対応しようとしますが、対応できなかった時はデリゲートメソッドが呼ばれ、  
独自に対応する必要が出てきます。  
デリゲートを実装していない場合はサーバから401(Forbidden)が返ってきます。  
  
### <u>対応する2つのデリゲートメソッド</u>  
  
要求された認証の種類によって対応するデリゲートメソッドが下記の２つに分かれます。  
  
1. Task単位の認証の処理を行う。  
URLSessionTaskDelegateのurlSession(_:task:didReceive:completionHandler:)  
例: Basic認証といったユーザ名とパスワードが必要となるものなど  
  
2. Task単位の認証処理が行われなかった場合に、Session単位の認証の処理を行う。一度認証が満たされれば全てのTaskに影響する。  
URLSessionDelegateのurlSession(_:didReceive:completionHandler:)  
例: TLSの妥当性チェックなど  
  
  
### どっちのメソッドで対処すべきか  
  
デリゲートメソッドが呼ばれた際に、引数にURLAuthenticationChallengeが渡ってきます。  
その中に  
  
```
URLProtectionSpaceのprotectionSpace
```  
というプロパティが存在し、  
  
さらに、そのプロパティが持つ  
  
```
authenticationMethod
```  
というプロパティの値から判定ができます。  
  
※詳しくは下記のAppleドキュメントにどの定数がどちらのメソッドで対処するべきかの記載があります。  
[NSURLProtectionSpace Authentication Method Constants](https://developer.apple.com/documentation/foundation/urlprotectionspace/nsurlprotectionspace_authentication_method_constants)  
[URLProtectionSpace](https://developer.apple.com/documentation/foundation/urlprotectionspace)  
  
  
  
以下に処理の流れを紹介します。  
  
### 対処する認証の種類を決める  
  
各メソッドのcompletionHandlerに渡すURLSession.AuthChallengeDispositionの値を変えることで、  
その後の動作が変わります。  
  
[URLSession.AuthChallengeDisposition](https://developer.apple.com/documentation/foundation/urlsession/authchallengedisposition)  
  
例えば、下記の場合ですと、Basic認証以外はデフォルトの方法で対処する。  
つまり、Basic認証以外はこのメソッド内では対応しないということを示しています。  
  
```Swift
let authMethod = challenge.protectionSpace.authenticationMethod
guard authMethod == NSURLAuthenticationMethodHTTPBasic else {
    completionHandler(.performDefaultHandling, nil)
    return
}
```  
  
### 信用情報(クレデンシャル)を作成する  
  
対処する認証要求を受け取った場合、信用情報(クレデンシャル)を作成します。  
例えばBasic認証の場合、入力値からusernameとpasswordを取得します。  
  
```Swift
func credentialsFromUI() -> URLCredential? {
    guard let username = usernameField.text, !username.isEmpty,
        let password = passwordField.text, !password.isEmpty else {
            return nil
    }

    // この場合はURLSessionインスタンスに信用情報は保存される
    // 他のURLSessionインスタンスの場合は新たに信用情報が必要
    return URLCredential(user: username, password: password,
                         persistence: .forSession)
}
```  
  
### completionHandlerを呼ぶ  
  
最終的に認証要求に応じるためにcompletionHandlerは必ず呼ばなければなりません。  
仮に信用情報の作成に失敗した場合やユーザが認証のキャンセルを行った場合、  
completionHandlerにキャンセルすることを伝えます。  
  
```Swift
guard let credential = credentialOrNil else {
    completionHandler(.cancelAuthenticationChallenge, nil)
    return
}
completionHandler(.useCredential, credential)
```  
  
### 認証に失敗した場合どうなる？  
  
例えば、信用情報が拒否された場合はデリゲートメソッドが再度呼ばれます。  
その際  
URLAuthenticationChallengeの```proposedCredential```プロパティに  
拒否された信用情報が格納されています。  
また、```previousFailureCount```プロパティには失敗した回数が格納されています。  
こういった値からどのように対処するのかを決めることができます。  
  
[proposedCredential](https://developer.apple.com/documentation/foundation/urlauthenticationchallenge/1417749-proposedcredential)  
  
[previousFailureCount](https://developer.apple.com/documentation/foundation/urlauthenticationchallenge/1416522-previousfailurecount)  
  
  
## HTTP対応について  
  
  
  
## 効率よくURLSessionを使うためのTips  
  
WWDC2017で紹介されていたURLSessionを使用する際のTipsを  
記載します。  
  
### URLSessionのタイムアウトを活用する  
  
- timeoutIntervalForResource(リソース全体の取得までのタイムアウト)  
リクエストを初期化してからこのURLConfigurationに関わる全セッションの全タスクが完了するまでの時間。デフォルトは7日。設定は秒数で行う。  
  
- timeoutIntervalForRequest(リクエストごとのタイムアウト)  
あるタスクがデータを受け取ってから次のデータを受け取るまでの時間。  
デフォルトは60秒。  
バックグラウンドのアップロード、ダウンロードタスクはタイムアウトした際に自動でリトライを行う。この制限は上記のtimeoutIntervalForResourceの設定値。  
  
  
### １アプリ１URLSession  
  
- 同時平行処理される複数のURLSessionTaskは１つのURLSessionで共有可能。  
- 独自でURLSessionを作成している場合は、finishTasksAndInvalidateやinvalidateAndCancelでデリゲートやコールバックオブジェクトへの参照をクリアしないとメモリーリークを起こす。  
  
[finishTasksAndInvalidate](https://developer.apple.com/documentation/foundation/urlsession/1407428-finishtasksandinvalidate)  
  
[invalidateAndCancel](https://developer.apple.com/documentation/foundation/urlsession/1411538-invalidateandcancel)  
  
  
### コールバックとデリゲートは１つのURLSessionで両方使用しない  
  
- コールバックを使用している場合、デリゲートメソッドが呼ばれることはない  
※例外的にtaskIsWaitingForConnectivityとdidReceiveAuthenticationChallengeは呼ばれる。  
  
### URLConfigurationの設定とタスクの優先度を考える  
  
  
|Default and Ephemeral Configuration + waitsForConnectivity  |Background Configuration  |Background Configuration + discretionary |  
|---|---|---|  
|アプリプロセス内  |アプリプロセス外  |アプリプロセス外  |  
|リトライなし  |timeoutIntervalForResourceまで自動リトライ  |timeoutIntervalForResourceまで自動リトライ  |  
|デリゲート、コールバック両方可能  |デリゲートのみ  |デリゲートのみ  |  
|タスク即時実行。実行開始に失敗した場合は、taskIsWaitingForConnectivityが呼ばれ、ネットワーク再接続時に自動でリトライをする  |タスクがネットワークの状況やバッテリーの状態を配慮して実行する。  |システムが最適な状況でタスクが実行されるようにスケジュールされる  |  
|緊急度高|緊急度中|緊急度低|  
  
  
### 単純なbyte streamを扱う場合はURLSessionStreamTaskを使おう  
  
HTTP以外の通信を行う場合、例えばメールの送信などは  
URLSessionStreamTaskを使うと効率が良いようです。  
  
### HTTP2を活用してレイテンシーを減らそう  
  
URLSessionはHTTP2に自動で対応しています。  
コネクションの使い回しによるネットワーク遅延の減少(毎回の3Wayハンドシェイクが不要になる)  
といった恩恵が得られます。  
  
#### HTTP/2 Connection Coalescing  
  
さらに、Connection Coalescingという機能がiOS12より追加されます。  
これは  
- 同じIPアドレス  
- 同じTLS認証がされたホスト名  
である場合、コネクションの再利用ができるようになりました。  
  
例えば、  
*.example.comというドメインに対する証明書に対して  
menu.example.com  
deliver.example.com  
という２つのサブドメインがあるとします。  
  
今までですと、両方に対してTLS認証を行う必要がありましたが、  
iOS12よりこれが一度認証されていれば認証されていると見なされ、  
接続時間を短くすることができます。  
  
#### URLSessionインスタンスの生成数を減らす  
  
上記でも紹介したようにURLSessionインスタンスは極力少ない方が良い。  
HTTP2を活用することでよりインタンス生成の必要性を減らすことができる。  
  
  
### リクエストヘッダーのサイズを小さくしてスループットを上げよう  
  
#### HTTPCookie  
  
- Cookieを保存するドメインとパスを特定する  
- 必要なものしかCookieに使用しない  
- 不要なものはCookieから削除する  
- 状態(state)はサーバで管理する  
  
#### HTTPヘッダーの圧縮  
  
- Gzip  
HTTPとHTTPSに対応している  
  
- Brotil(iOS11以降)  
HTTPSのみに対応し、textとHTMLに関してはこちらの方が最適化されている  
  
  
### サーバからのレスポンスを考えよう  
  
#### QoSの活用  
細かくは言ってなかったがprefetchなどバックグラウンドで良いものはバックグラウンドで行う。  
task.resume()が呼ばれたタイミングでDispatchQueueのQoSは決定される。  
Delegateメソッドの呼び出しの優先順位もQoSによって決められる。  
  
#### Network Service Type  
  
だいたいは.defaultか.backgroundで良い。  
  
iOS12で```.responsiveData```が追加された。  
これは.defaultよりも優先が高く、  
買い物アプリを使用している際の支払いリクエストなどに活用すると、  
通常よりもレスポンスの反応が良くなる。  
  
#### URLSession Adaptable Connectivity  
  
- URLConfigurationのところでも紹介した```waitsForConnectivity```は常にtrueにする  
- 不要になったタスクに関してはtask.cancel()をする  
  
#### システムリソースを効率良く使用する  
  
- バックグラウンドセッションの使用  
```isDiscretionary```をtrueにしてシステムにタスクの実行タイミングを任せる

- キャッシュを賢く使う
不要なものはキャッシュをしないようにデリゲートメソッドで設定する



## 様々なデリゲート

最後にURLSessionに関連するデリゲートメソッドの一覧を載せておきます。

<details>
<summary>デリゲート一覧</summary>

URLSession(それに関連するクラス)には様々な状況に対応するためのデリゲートがたくさんあります。主に分けると下記の５つになります。

### <u>URLSessionDelegate</u>

```Swift  
/* ライフサイクルに関連したメソッド */  
  
// URLSessionがInvalidateになった時に呼ばれる  
optional func urlSession(URLSession, didBecomeInvalidWithError: Error?)  
  
// URLSessionにキューされた全てのメッセージが伝達された時に呼ばれる  
optional func urlSessionDidFinishEvents(forBackgroundURLSession session: URLSession)  
```

```Swift  
/* 認証に関連したメソッド */  
  
// URLSessionレベルの認証要求をサーバからされた際の処理をする  
// URLSessionTaskが認証要求の処理に失敗した際に呼ばれる(基本的にはURLSessionTask内で完結)  
// ここで認証要求を満たすとURLSessionに関わる全てのURLSessionTaskが認証を満たしているとされる  
optional func urlSession(_ session: URLSession,   
              didReceive challenge: URLAuthenticationChallenge,   
       completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void)  
```

### <u>URLSessionTaskDelegate</u>

```Swift  
/* ライフサイクルに関連したメソッド */  
  
// Taskが完了した際に呼ばれる  
// ここのerrorにはサーバのerrorは含まれない  
// hostnameの解決ができないなどclient側の理由によるエラーが返ってくる  
optional func urlSession(_ session: URLSession,   
                    task: URLSessionTask,   
    didCompleteWithError error: Error?)  
```


```Swift  
/* リダイレクトに関連したメソッド */  
  
// サーバからリダイレクトを要求された際に呼ばれる  
// configurationがdefaultかephemeralの時のみ呼ばれ、  
// backgroundの場合は自動でリダイレクトする  
optional func urlSession(_ session: URLSession,   
                    task: URLSessionTask,   
willPerformHTTPRedirection response: HTTPURLResponse,   
              newRequest request: URLRequest,   
       completionHandler: @escaping (URLRequest?) -> Void)  
```

```Swift  
/* UploadTaskに関連したメソッド */  
  
// 定期的に呼ばれ、サーバにどのくらいデータの送信が完了したかを取得する  
optional func urlSession(_ session: URLSession,   
                    task: URLSessionTask,   
         didSendBodyData bytesSent: Int64,   
          totalBytesSent: Int64,   
totalBytesExpectedToSend: Int64)  
  
  
// サーバにデータを送信するために  
// taskから新しいhttpBodyStreamを要求された際に呼ばれる  
// ファイルURLやデータオブジェクトからhttpBodyを設定している場合は実装不要  
optional func urlSession(_ session: URLSession,   
                    task: URLSessionTask,   
       needNewBodyStream completionHandler: @escaping (InputStream?) -> Void)  
  
下記の条件の際にTaskは上記のメソッドを呼び出す  
  
・uploadTask(withStreamedRequest:)から作成されたTaskの  
最初のhttpBodyStreamが提供されたとき  
  
・認証要求やサーバの回復可能なエラーが原因で  
httpBodyStreamを持ったリクエストを再送信する必要がある場合で  
代わりになるhttpBodyStreamが提供されたとき  
  
```

[uploadTask(withStreamedRequest:)](https://developer.apple.com/documentation/foundation/urlsession/1410934-uploadtask?changes=_3)

```Swift  
/* 認証に関連したメソッド */  
  
// URLSessionレベル以外の認証要求をサーバからされた際の処理をする  
// URLSessionDelegateを実装している場合に実装しなければならない  
optional func urlSession(_ session: URLSession,   
                    task: URLSessionTask,   
              didReceive challenge: URLAuthenticationChallenge,   
       completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void)  
```

```Swift  
/* Delayed、Waiting Taskに関連したメソッド */  
  
// delayedURLSessionTaskが始まる際に呼ばれる  
// そのまま処理を継続するか、新しいリクエストとして処理するか、  
// キャンセルするかを決めることができる  
optional func urlSession(_ session: URLSession,   
                    task: URLSessionTask,   
 willBeginDelayedRequest request: URLRequest,   
       completionHandler: @escaping (URLSession.DelayedRequestDisposition, URLRequest?) -> Void)  
```

```Swift  
/* Metricsに関連したメソッド */  
  
// Sessionが特定のTaskに関連したMetrics情報を収集し終えた際に呼ばれる  
optional func urlSession(_ session: URLSession,   
                    task: URLSessionTask,   
     didFinishCollecting metrics: URLSessionTaskMetrics)  
```

[Appleドキュメント](https://developer.apple.com/documentation/foundation/urlsessiontaskdelegate?changes=_3)


### <u>URLSessionDataDelegate</u>

```Swift  
/* ライフサイクルに関連したメソッド */  
  
// サーバから最初にデータ(ヘッダー)を受け取った際に呼ばれる  
// multipart/x-mixed-replaceが必要な場合は必須になる  
// この時点で処理のキャンセルをすることもできる  
optional func urlSession(_ session: URLSession,   
                dataTask: URLSessionDataTask,   
              didReceive response: URLResponse,   
       completionHandler: @escaping (URLSession.ResponseDisposition) -> Void)  
  
  
// 上記のdidReceiveメソッドで  
// URLSession.ResponseDisposition.becomeDownloadが指定され  
// dataTaskからdownloadTaskに変わった際に呼ばれる  
// これが呼ばれると元のdataTaskのデリゲートは呼ばれなくなる  
optional func urlSession(_ session: URLSession,   
                dataTask: URLSessionDataTask,   
               didBecome downloadTask: URLSessionDownloadTask)  
  
// 上記のdidReceiveメソッドで  
// URLSession.ResponseDisposition.becomeStreamが指定され  
// dataTaskからstreamTaskに変わった際に呼ばれる  
// これが呼ばれると元のdataTaskのデリゲートは呼ばれなくなる  
// 元々繋がっていたURLリクエストに対してはstreamTaskは読み取り専用になり  
// URLSessionStreamDelegateのwriteClosedForメソッドが即座に呼ばれる  
// configurationのhttpShouldUsePipeliningをfalseにすることで  
// リクエストのパイプライン化(分割)を不可にすることができる  
// 個々のリクエストの設定はURLRequestのhttpShouldUsePipelining  
optional func urlSession(_ session: URLSession,   
                dataTask: URLSessionDataTask,   
               didBecome streamTask: URLSessionStreamTask)  
  
```

```Swift  
/* データ受信に関連したメソッド */  
  
// データを受信した際に呼ばれる  
// 受信データは多くの異なるオブジェクトのまとめてあるものが多いため  
// 可能な時はenumeratedBytesを使うようにする  
// ※bytesを使うと受信データは１つのメモリーブロックに入っている  
// 受信データは前回以降に受信したデータのみを受け取る  
optional func urlSession(_ session: URLSession,   
                dataTask: URLSessionDataTask,   
              didReceive data: Data)  
```

[enumerateBytes](https://developer.apple.com/documentation/foundation/nsdata/1408400-enumeratebytes?changes=_3)

[bytes](https://developer.apple.com/documentation/foundation/nsdata/1410616-bytes?changes=_3)


```Swift  
/* キャッシュに関連したメソッド */  
  
// 全てのデータ受信が完了した後に呼ばれ、  
// レスポンスをキャッシュに保存するかどうかを決める  
// このメソッドが実装されていない場合、configurationのポリシーに従う  
// ある特定のURLのキャッシュを防いだり、userInfoのデータを変更する場合に使う  
optional func urlSession(_ session: URLSession,   
                dataTask: URLSessionDataTask,   
       willCacheResponse proposedResponse: CachedURLResponse,   
       completionHandler: @escaping (CachedURLResponse?) -> Void)  
  
レスポンスがキャッシュされる条件  
-HTTPまたはHTTPSのURLであること  
-リクエストが成功している(ステータスコードが200~299)  
-サーバから来たレスポンスである  
-configurationでキャッシュの保存を許可している  
-URLRequestでキャッシュの保存を許可している  
-サーバレスポンスのヘッダーのでキャッシュの保存が許可されている(ヘッダーがある場合)  
-レスポンスサイズがキャッシュできる範囲の大きさであること(例えばディスクキャッシュサイズの5%以下であること)  
```

[Appleドキュメント](https://developer.apple.com/documentation/foundation/urlsession?changes=_3)


### <u>URLSessionDownloadDelegate</u>

```Swift  
/* ライフサイクルに関連したメソッド */  
  
// ダウンロードが完了した際に呼ばれる  
// locationのURLはダウンロードしたtemporaryファイルのURL  
func urlSession(_ session: URLSession,   
   downloadTask: URLSessionDownloadTask,   
didFinishDownloadingTo location: URL)  
  
```

```Swift  
/* Resumeデータに関連したメソッド */  
  
// ダウンロードが再開された際に呼ばれる  
// ダウンロード済のデータが使用できなくなっていた場合、fileOffsetは0  
optional func urlSession(_ session: URLSession,   
            downloadTask: URLSessionDownloadTask,   
       didResumeAtOffset fileOffset: Int64,   
      expectedTotalBytes: Int64)  
  
  
再開可能なTaskがキャンセルされた場合や失敗した場合  
再開するために必要なデータを取得することができる  
これを使用してdownloadTask(withResumeData:)や  
downloadTask(withResumeData:completionHandler:) を呼ぶことができる  
これらのメソッドが呼ばれTaskが開始されると、  
sessionはデリゲートメソッドを即時に呼び、ダウンロードが再開されたことを  
教えてくれる  
  
```

```Swift  
/* プログレスに関連したメソッド */  
  
// 定期的に呼ばれ、ダウンロードの進行状況の処理をする  
optional func urlSession(_ session: URLSession,   
            downloadTask: URLSessionDownloadTask,   
            didWriteData bytesWritten: Int64,   
       totalBytesWritten: Int64,   
totalBytesExpectedToWrite: Int64)  
```

[Appleドキュメント](https://developer.apple.com/documentation/foundation/urlsessiondownloaddelegate?changes=_3)


### <u>URLSessionStreamDelegate</u>

```Swift  
/* 再ルーティングに関連したメソッド */  
  
// より良いルートを利用可能になった際に呼ばれる  
// 例えば端末回線を使用していてWifiが利用可能になった際など  
// 待機中のタスクを完了させたり、タスクの新規作成などを検討する  
optional func urlSession(_ session: URLSession,   
betterRouteDiscoveredFor streamTask: URLSessionStreamTask)  
```

```Swift  
/* ストリームの完了に関連したメソッド */  
  
// captureStreamsメソッドが呼ばれ、  
// Task内の読み書き処理が全て完了した際に呼ばれる  
optional func urlSession(_ session: URLSession,   
              streamTask: URLSessionStreamTask,   
               didBecome inputStream: InputStream,   
            outputStream: OutputStream)  
```
[captureStream](https://developer.apple.com/documentation/foundation/urlsessionstreamtask/1410132-capturestreams?changes=_3)

```Swift  
/* streamのクローズに関連したメソッド */  
  
// 読み取り側のソケットが閉じた際に呼ばれる  
// 読むデータがなくなった場合にも呼ばれる  
// 呼ばれたからといってEOFに到達したという訳ではない  
optional func urlSession(_ session: URLSession,   
           readClosedFor streamTask: URLSessionStreamTask)           
  
// 書き込み側のソケットが閉じた際に呼ばれる  
// 書くデータがなくなった場合にも呼ばれる  
optional func urlSession(_ session: URLSession,   
          writeClosedFor streamTask: URLSessionStreamTask)  
```

[Appleドキュメント](https://developer.apple.com/documentation/foundation/urlsessionstreamdelegate?changes=_3)



</details>



## まとめ

全然まとまってないですね:cry:

URLSessionはこれからも機能の増加や変更などが入ってくると思いますので、
随時更新して頭の整理をしていきたいと思います。

間違いなどございましたらご指摘いただけるとうれしいです:bow_tone1:

## 悩み

ソースコードの部分はアコーディオンになるように設定しているのですが、
VSCodeだときちんと反映され、Qiitaだとなぜかうまくいかないですね:cold_sweat:
