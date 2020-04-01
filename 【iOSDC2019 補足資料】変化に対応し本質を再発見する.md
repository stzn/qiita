コードを書き始めるとき  
誰しもコードをきれいに保とうと思うはずです。  
  
最初は機能が少ないため  
そのクラスの意図は明確で  
バグが発生する可能性も少ないでしょう。  
  
しかし  
次第に機能が増えていったり  
様々な条件分岐が起きるなど  
複雑さが増してくるとクラスの意図が不明確になり  
バグを起きそうなポイントもたくさん増えていきます。  
  
今回は  
そういった変化に対応していくためにはどうしていけば良いのか  
Souroush Khanlouさんの発表を追いながら考えていきたいと思います。  
  
**From Problem to Solution**  
https://youtu.be/N90_q8Uzc4A?list=PLLcE3DL3f5Bx0IAHAw6hsdZ3z_samz2iX  
  
  
※  
コードの中でPromiseというライブラリを使用しています。  
簡単に言うと非同期処理を扱いやすくするためのものです。  
詳しくは下記をご参照ください。  
https://github.com/khanlou/Promise  
  
# 良いコード例  
  
下記の実装を見てみてください。  
  
```swift

final class NetworkClient {
    let session = URLSession.shared

    func send<Output: Codable>(request: Request<Output>) -> Promise<Output> {
        let urlRequest = URLRequesBuilder(request: request).urlRequest

        return session.data(with: urlRequest)
            .then({ data, response in
                guard (200..<300).contains(response.statusCode) else {
                    throw StatusCodeError(statusCode: response.statusCode)
                }
                return try JSONDecoder().decode(Output.self, from: data) as! Promise<Output>
            })
    }
}
```  
  
ネットワーク通信を行うクラスですが、  
URLSessionを使ってデータを取得し  
欲しいクラスの型へとデコードしています。  
  
これは良い実装だと思います。  
  
それはテストがしやすいからです。  
  
# テストがしやすい理由  
  
下記の２つの観点があります。  
  
## シングルトンの数  
  
シングルトンが一つしか存在せず  
このシングルトンはProtocolに置き換えて依存関係を注入することで  
モックと差し替えることが簡単にできます。  
  
```swift

protocol URLSessionProtocol {
    func dataTask(with request: URLRequest) -> URLSessionDataTask
    func dataTask(with url: URL) -> URLSessionDataTask
    func dataTask(with request: URLRequest, completionHandler: @escaping (Data?, URLResponse?, Error?) -> Void) -> URLSessionDataTask
    func dataTask(with url: URL, completionHandler: @escaping (Data?, URLResponse?, Error?) -> Void) -> URLSessionDataTask
}

extension URLSession: URLSessionProtocol{}


final class NetworkClient {
    let session: URLSessionProtocol
    init(session: URLSessionProtocol = URLSession.shared) {
        self.session = session
    }

    func send<Output: Codable>(request: Request<Output>) -> Promise<Output> {
        let urlRequest = URLRequesBuilder(request: request).urlRequest

        return session.data(with: urlRequest)
            .then({ data, response in
                guard (200..<300).contains(response.statusCode) else {
                    throw StatusCodeError(statusCode: response.statusCode)
                }
                return try JSONDecoder().decode(Output.self, from: data) as! Promise<Output>
            })
    }
}

```  
  
## 循環的複雑度(Cyclomatic Complexity)  
  
もう一つの観点として循環的複雑度があります。  
https://ja.wikipedia.org/wiki/%E5%BE%AA%E7%92%B0%E7%9A%84%E8%A4%87%E9%9B%91%E5%BA%A6  
  
これは  
条件分岐の数を集計し  
多ければ多いほど複雑であることを示しています。  
  
この条件分岐の数がそのままテストケースの数にもなるため  
複雑度が低いとテストをする数も少なくて済みます。  
  
上記の実装での数を数えてみると  
  
1. 正常  
2. ネットワークエラー(URLSessionにリクエストを送ったが返ってこないなど)  
3. HTTPステータスコードエラー  
4. JSONデコードエラー  
  
とたったの４つの条件分岐しかなく  
複雑度が低いということがわかりました。  
  
  
# 新しい機能を追加した時にいつまでも良いままでいるのは難しい  
  
ここに新しい機能を追加していくとどうなるでしょうか？  
  
以下の機能を追加していきます。  
  
## 認証ヘッダーの追加  
  
```swift

final class NetworkClient {
    let session = URLSession.shared

    func send<Output: Codable>(request: Request<Output>) -> Promise<Output> {
        var urlRequest = URLRequesBuilder(request: request).urlRequest

        // ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
        if let authToken = UserDefaults.standard.string(forKey: "authToken") {
            urlRequest.addValue(authToken, forHTTPHeaderField: "X-Auth-Token")
        }

        return session.data(with: urlRequest)
            .then({ data, response in
                guard (200..<300).contains(response.statusCode) else {
                    throw StatusCodeError(statusCode: response.statusCode)
                }
                return try JSONDecoder().decode(Output.self, from: data) as! Promise<Output>
            })
    }
}

```  
  
## バックグラウンドタスクの追加  
  
```swift

final class NetworkClient {
    let session = URLSession.shared

    func send<Output: Codable>(request: Request<Output>) -> Promise<Output> {
        var urlRequest = URLRequesBuilder(request: request).urlRequest

        if let authToken = UserDefaults.standard.string(forKey: "authToken") {
            urlRequest.addValue(authToken, forHTTPHeaderField: "X-Auth-Token")
        }

        // ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
        var identifier: UIBackgroundTaskIdentifier?
        identifier = UIApplication.shared.beginBackgroundTask(expirationHandler: {
            if let identifier = identifier {
                UIApplication.shared.endBackgroundTask(identifier)
            }
        })

        return session.data(with: urlRequest)
            .then({ data, response in
                guard (200..<300).contains(response.statusCode) else {
                    throw StatusCodeError(statusCode: response.statusCode)
                }
                return try JSONDecoder().decode(Output.self, from: data) as! Promise<Output>
            // ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
            }).always {
                if let identifier = identifier {
                    UIApplication.shared.endBackgroundTask(identifier)
                }
        }
    }
}

```  
  
## RequestCounterの追加  
  
```swift

class RequestCounter {
    static let shared = RequestCounter()
    var counter = 0 {
        didSet {
            UIApplication.shared.isNetworkActivityIndicatorVisible = counter == 0
        }
    }
}

final class NetworkClient {
    let session = URLSession.shared

    func send<Output: Codable>(request: Request<Output>) -> Promise<Output> {
        var urlRequest = URLRequesBuilder(request: request).urlRequest

        if let authToken = UserDefaults.standard.string(forKey: "authToken") {
            urlRequest.addValue(authToken, forHTTPHeaderField: "X-Auth-Token")
        }

        var identifier: UIBackgroundTaskIdentifier?
        identifier = UIApplication.shared.beginBackgroundTask(expirationHandler: {
            if let identifier = identifier {
                UIApplication.shared.endBackgroundTask(identifier)
            }
        })

        // ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
        RequestCounter.shared.counter += 1
        
        return session.data(with: urlRequest)
            .then({ data, response in
                guard (200..<300).contains(response.statusCode) else {
                    throw StatusCodeError(statusCode: response.statusCode)
                }
                return try JSONDecoder().decode(Output.self, from: data) as! Promise<Output>
            }).always {
                if let identifier = identifier {
                    UIApplication.shared.endBackgroundTask(identifier)
                }
                // ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
                RequestCounter.shared.counter -= 1
        }
    }
}

```  
  
どんどん大きく  
どんどん複雑になってきました。  
  
# なぜ複雑になってしまったのか？  
  
## SRP(Single Responsibility Principle)ではなくなる  
  
当初はネットワークリクエストを行って必要な型へでデコードするという  
1つの機能を持つクラスでした。  
  
そこに上記の３つの機能が追加になることで  
責務が増えてしまいました。  
  
## コードの長さ  
  
今までは20行にも満たない程度でしたが  
機能の追加によって２倍以上になりました。  
  
## 循環的複雑度(Cyclomatic Complexity)  
  
どれだけ分岐があるかを見てみると  
  
```swift

class RequestCounter {
    static let shared = RequestCounter()

    // ２ケース
    var counter = 0 {
        didSet {
            UIApplication.shared.isNetworkActivityIndicatorVisible = counter == 0
        }
    }
}

final class NetworkClient {
    let session = URLSession.shared

    func send<Output: Codable>(request: Request<Output>) -> Promise<Output> {
        var urlRequest = URLRequesBuilder(request: request).urlRequest

        // ２ケース
        if let authToken = UserDefaults.standard.string(forKey: "authToken") {
            urlRequest.addValue(authToken, forHTTPHeaderField: "X-Auth-Token")
        }

        // 3ケース
        var identifier: UIBackgroundTaskIdentifier?
        identifier = UIApplication.shared.beginBackgroundTask(expirationHandler: {
            if let identifier = identifier {
                UIApplication.shared.endBackgroundTask(identifier)
            }
        })

        RequestCounter.shared.counter += 1
        
        // 4ケース
        return session.data(with: urlRequest)
            .then({ data, response in
                guard (200..<300).contains(response.statusCode) else {
                    throw StatusCodeError(statusCode: response.statusCode)
                }
                return try JSONDecoder().decode(Output.self, from: data) as! Promise<Output>
            }).always {
                // 2ケース
                if let identifier = identifier {
                    UIApplication.shared.endBackgroundTask(identifier)
                }
                RequestCounter.shared.counter -= 1
        }
    }
}
```  
  
合計としては  
  
2 x 2 x 4 x 2 = **96ケース**  
  
となります。  
  
これはかなり複雑度が高いです。  
  
## シングルトンの数  
  
シングルトンを抽出してみると  
4つあることがわかります。  
  
```swift

final class NetworkClient {

    // singletons
    let session = URLSession.shared
    let application = UIApplication.shared
    let defaults = UserDefaults.standard
    let counter = RequestCounter.shared

    func send<Output: Codable>(request: Request<Output>) -> Promise<Output> {
        var urlRequest = URLRequesBuilder(request: request).urlRequest

        // responsiblity 1
        if let authToken = defaults.string(forKey: "authToken") {
            urlRequest.addValue(authToken, forHTTPHeaderField: "X-Auth-Token")
        }

        // responsiblity 2
        var identifier: UIBackgroundTaskIdentifier?
        identifier = application.beginBackgroundTask(expirationHandler: { [weak self] in
            if let identifier = identifier {
                self?.application.endBackgroundTask(identifier)
            }
        })

        // responsiblity 3
        counter.counter += 1

        return session.data(with: urlRequest)
            .then({ data, response in
                guard (200..<300).contains(response.statusCode) else {
                    throw StatusCodeError(statusCode: response.statusCode)
                }
                return try JSONDecoder().decode(Output.self, from: data) as! Promise<Output>
            }).always { [weak self] in
                // responsiblity 2
                if let identifier = identifier {
                    self?.application.endBackgroundTask(identifier)
                }

                // responsiblity 3
                self?.counter.counter -= 1
        }
    }
}

```  
  
これをテストするためにProtocolを定義して  
依存関係を注入してみるとどうなるでしょうか？  
  
```swift

protocol URLSessionProtocol {
    func dataTask(with request: URLRequest) -> URLSessionDataTask
    func dataTask(with url: URL) -> URLSessionDataTask
    func dataTask(with request: URLRequest, completionHandler: @escaping (Data?, URLResponse?, Error?) -> Void) -> URLSessionDataTask
    func dataTask(with url: URL, completionHandler: @escaping (Data?, URLResponse?, Error?) -> Void) -> URLSessionDataTask
}
extension URLSession: URLSessionProtocol{}


protocol RequestCounterProtocol {
    var counter: Int { set get }
}
extension RequestCounter: RequestCounterProtocol {}

protocol UIApplicationProtocol {
    func beginBackgroundTask(expirationHandler handler: (() -> Void)?) -> UIBackgroundTaskIdentifier
    func beginBackgroundTask(withName taskName: String?, expirationHandler handler: (() -> Void)?) -> UIBackgroundTaskIdentifier
    func endBackgroundTask(_ identifier: UIBackgroundTaskIdentifier)
}
extension UIApplication: UIApplicationProtocol {}

protocol UserDefaultsProtocol {
    func string(forKey defaultName: String) -> String?
}
extension UserDefaults: UserDefaultsProtocol {}

final class NetworkClient {
    let session:URLSessionProtocol
    let application: UIApplicationProtocol
    let defaults: UserDefaultsProtocol
    var counter: RequestCounterProtocol

    init(session:URLSessionProtocol = URLSession.shared,
         application: UIApplicationProtocol = UIApplication.shared,
         defaults: UserDefaultsProtocol = UserDefaults.standard,
         counter: RequestCounterProtocol = RequestCounter.shared) {
        self.session = session
        self.application = application
        self.defaults = defaults
        self.counter = counter
    }

    func send<Output: Codable>(request: Request<Output>) -> Promise<Output> {
        var urlRequest = URLRequesBuilder(request: request).urlRequest

        // responsiblity 1
        if let authToken = defaults.string(forKey: "authToken") {
            urlRequest.addValue(authToken, forHTTPHeaderField: "X-Auth-Token")
        }

        // responsiblity 2
        var identifier: UIBackgroundTaskIdentifier?
        identifier = application.beginBackgroundTask(expirationHandler: { [weak self] in
            if let identifier = identifier {
                self?.application.endBackgroundTask(identifier)
            }
        })

        // responsiblity 3
        counter.counter += 1

        return session.data(with: urlRequest)
            .then({ data, response in
                guard (200..<300).contains(response.statusCode) else {
                    throw StatusCodeError(statusCode: response.statusCode)
                }
                return try JSONDecoder().decode(Output.self, from: data) as! Promise<Output>
            }).always { [weak self] in
                // responsiblity 2
                if let identifier = identifier {
                    self?.application.endBackgroundTask(identifier)
                }

                // responsiblity 3
                self?.counter.counter -= 1
        }
    }
}
```  
  
コードはさらに大きくなってしまいました。  
これは望ましい状態でしょうか？  
  
  
# 本質を取り出していく方法を身につける  
  
本当にやりたい機能以外の機能が増えすぎてクラスの意図がぼやけてしまいました。  
そこでここからクラスの輪郭をはっきりとさせるために  
コアの機能以外のものを整理していきます。  
  
## なぜこのタイミングなのか？  
  
- 管理やテストができなくなり始め苦痛に感じるようになった  
- 問題が非常に理解しがたいように見えた  
  
  
## Step1 問題の中から類似点を見つけ出す   
  
ある観点で処理の中の類似点で分類します。  
  
### 時間軸による分離  
  
時間軸で考えると下記の３つに分かれます。  
  
- ヘッダーへ値を追加(headers)  
- リクエストを送る前(before)  
- リクエスト完了後(成功、失敗関わらず)(after)  
  
それを踏まえた実装は下記のようになります。  
  
```swift

final class NetworkClient {
    let session:URLSessionProtocol

    let headers = Headers()
    let before = Before()
    let after = After()

    init(session:URLSessionProtocol = URLSession.shared) {
        self.session = session
    }

    func send<Output: Codable>(request: Request<Output>) -> Promise<Output> {
        var urlRequest = URLRequesBuilder(request: request).urlRequest

        headers.add(urlRequest)

        before.execute()

        return session.data(with: urlRequest)
            .then({ data, response in
                guard (200..<300).contains(response.statusCode) else {
                    throw StatusCodeError(statusCode: response.statusCode)
                }
                return try JSONDecoder().decode(Output.self, from: data) as! Promise<Output>
            }).always { [weak self] in
                // リクエスト完了後
                after.execute()
        }
    }
}
```  
  
一見良さそうに見えますが、  
問題はバックグラウンドタスクのidentifireの状態を  
beforeとafterで共有しなければいけない点です。  
  
こうすると状態の管理をする必要が出てきてしまい  
処理をうまく切り分けられていないように見えます。  
  
### 責務による分離  
  
次に責務によって分けてみたいと思います。  
  
- 認証ヘッダーの追加  
- バックグラウンドタスクの追加  
- RequestCounterの追加  
  
```swift

final class NetworkClient {
    let session:URLSessionProtocol

    let authToken = AuthToken()
    let counter = Counter()
    let background = Backgrounding()

    func send<Output: Codable>(request: Request<Output>) -> Promise<Output> {
        var urlRequest = URLRequesBuilder(request: request).urlRequest

        authToken.add(to: urlRequest)

        bakground.prepare()

        counter.increament()

        return session.data(with: urlRequest)
            .then({ data, response in
                guard (200..<300).contains(response.statusCode) else {
                    throw StatusCodeError(statusCode: response.statusCode)
                }
                return try JSONDecoder().decode(Output.self, from: data) as! Promise<Output>
            }).always { [weak self] in

                bakgrounding.release()

                counter.decrement()
        }
    }
}
```  
  
ここには出てきていませんが  
こうすることでBackgroundTaskの中で  
identifierの状態管理を完結させることができ  
よりはっきりと分類ができています。  
  
## Step2 より本質を抽出していく  
  
ここでもう一度時間軸での分類を考えてみます。  
  
```swift

final class NetworkClient {
    let session:URLSessionProtocol

    let authToken = AuthToken()
    let counter = Counter()
    let background = Backgrounding()

    func send<Output: Codable>(request: Request<Output>) -> Promise<Output> {
        var urlRequest = URLRequesBuilder(request: request).urlRequest

        // headers
        authToken.add(to: urlRequest)

        // before
        bakground.prepare()
        counter.increament()

        return session.data(with: urlRequest)
            .then({ data, response in
                guard (200..<300).contains(response.statusCode) else {
                    throw StatusCodeError(statusCode: response.statusCode)
                }
                return try JSONDecoder().decode(Output.self, from: data) as! Promise<Output>
            }).always { [weak self] in

                // after
                bakground.release()
                counter.decrement()
        }
    }
}
```  
  
もっとはっきりさせるために名前を変更します。  
  
```swift

final class NetworkClient {
    let session:URLSessionProtocol

    let authToken = AuthToken()
    let counter = Counter()
    let background = Backgrounding()

    func send<Output: Codable>(request: Request<Output>) -> Promise<Output> {
        var urlRequest = URLRequesBuilder(request: request).urlRequest

        // headers
        authToken.addHeaders(to: urlRequest)

        // before
        bakground.before()
        counter.before()

        return session.data(with: urlRequest)
            .then({ data, response in
                guard (200..<300).contains(response.statusCode) else {
                    throw StatusCodeError(statusCode: response.statusCode)
                }
                return try JSONDecoder().decode(Output.self, from: data) as! Promise<Output>
            }).always { [weak self] in

                // after
                bakground.after()
                counter.after()
        }
    }
}
```  
  
はじめはちょっと違和感のあった時間軸による分離ですが  
責務による分離を行ったことで  
時間軸による分離も活かせそうに見えてきました。  
  
そこで下記のProtocolを定義してみます。  
  
```swift

protocol RequestBehavior {
    func addHeaders(to request: URLRequest)
    func before()
    func after()
}
```  
  
さらに実装を変更していきます。  
  
```swift

final class NetworkClient {
    let session:URLSessionProtocol

    let authToken: RequestBehavior = AuthToken()
    let counter: RequestBehavior = Counter()
    let background: RequestBehavior = Backgrounding()
    // ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
    let behaviors: [RequestBehavior] = [AuthToken(), Counter(), Backgrounding()]

    init(behaviors: [RequestBehavior] = [AuthToken(), Counter(), Backgrounding()]) {
        self.behaviors = behaviors
    }

    func send<Output: Codable>(request: Request<Output>) -> Promise<Output> {
        var urlRequest = URLRequesBuilder(request: request).urlRequest

        // headers
        behaviors.forEach({ $0.addHeaders(to: urlRequest) })

        // before
        behaviors.forEach({ $0.before() })

        return session.data(with: urlRequest)
            .then({ data, response in
                guard (200..<300).contains(response.statusCode) else {
                    throw StatusCodeError(statusCode: response.statusCode)
                }
                return try JSONDecoder().decode(Output.self, from: data) as! Promise<Output>
            }).always { [weak self] in

                // after
                behaviors.forEach({ $0.after() })
        }
    }
}
```  
  
行っていることは全然違う機能を  
一つのProtocolでまとめることができ  
sendメソッドの中がよりすっきりしてきました。  
  
もっとわかりやすくしていきます。  
  
```swift

protocol RequestBehavior {

    // URLRequestのことを知る必要はなく、Headerに設定する情報を知れれば良い
    var additionalHeaders: [String: String] { get }

    // beforeだけだと何の前かがわからない
    func beforeSend()

    // 成功時と失敗時で処理を分離
    func afterSuccess(result: AnyResponse)
    func afterFailure(error: Error)
}
```  
  
※   
AnyResponseに関しては特に触れていませんでしたが  
おそらくどんな型でも受け入れることができる型だと思います。  
  
  
さらに使いやすくするために  
下記を追加します。  
  
```swift

extension RequestBehavior {
    var additionalHeaders: [String: String] { return [:] }

    func beforeSend() {}

    func afterSuccess(result: AnyResponse)  {}
    func afterFailure(error: Error)  {}
}
```  
  
こうすることで  
必要のない処理を実装する必要がなくなります。  
  
  
### DRYの先の構造的に似ているものを抽出する  
  
上記のポイントとして  
形としては似ていないが  
概念としては似ているものでまとめている点です。  
  
下記はコードを形と概念の類似性の関係をチャートで示したものです。  
  
![スクリーンショット 2019-08-11 8.17.28.png](/image/e49258de-094f-ac75-8dcd-0e2be0a18418.png)  
  
形も概念も似ているコードは見つけやすいです。  
DRY原則は開発者が初期の段階で学びまず考慮するべきものですが  
経験を積んできたらもう１段階考慮するべきポイントがあります。  
  
それが形は異なるものの概念が似ているコードです。  
今回のケースではbackgroudとcounterは全く異なる機能ですが  
処理を実行するタイミングとしては同じです。  
  
こういったケースを抽出することで  
よりコアの機能の輪郭をはっきりさせることができます。  
  
一方で形は似ているものの概念が異なるものをまとめてしまうと  
それぞれ異なった理由で変更を加える必要が出てくる可能性があり  
負担を増やしてしまうリスクもあります。  
  
**DRY原則(２つの重複しているコードを１つにまとめる)**  
https://ja.wikipedia.org/wiki/Don%27t_repeat_yourself  
  
**false positiveとfalse negative**  
https://qiita.com/steel_code/items/101c9d037d5e8c2b7876  
  
## Step3 さらに抽出しやすくする  
  
上記で見つけた共通点を  
さらに使いやすいものにしていきます。  
  
### 不要な処理の記述を減らす  
  
デフォルト実装をすることで  
不要な処理を書く必要をなくします。  
  
```swift

extension RequestBehavior {
    var additionalHeaders: [String: String] { return [:] }
    
    // before what?
    func beforeSend() {}
    
    // separate success failure
    func afterSuccess(result: AnyResponse)  {}
    func afterFailure(error: Error)  {}
}
```  
  
  
### Optionalをなくす  
  
今までのところRequestBehaviorは  
初期化時に決まった処理を設定していますが  
今後sendメソッドごとに  
さらに必要なBehaviorが必要になる可能性があります。  
この際にBehaviorが不要な場合も考えられるため  
Optionalにします。  
  
```swift

func send(
    request: Request,
    with requestBehavior: RequestBehavior? = nil
)
```  
  
しかし  
Optionalにすると都度チェックが必要になったり  
条件分岐によってコアの輪郭がぼやけてしまいます。  
  
そこで空のクラスを作成することによって  
引数にOptionalを渡すことをなくし  
Optionalチェックを不要にします。  
  
```swift

struct EmptyRequestBehavior: RequestBehavior {}

func send(
    request: Request,
    with requestBehavior: RequestBehavior = EmptyRequestBehavior()
)
```  
  
### 複数の処理を一つのものとして扱えるようにする  
  
さらに複数のBehaviorが必要な場合は  
配列で渡す必要があります。  
そうするとシンタックスを変更する必要があります。  
  
```swift

func send(
    request: Request,
    with requestBehaviors: [RequestBehavior] = [EmptyRequestBehavior()]
)
```  
  
しかし  
こうすると単体の場合でも配列にしなければならず  
ちょっと煩わしく感じます。  
  
そこで複数のBehaviorも  
１つのBehaviorとして扱えるようにします。  
  
```swift

struct CombinedRequestBehavior: RequestBehavior {
    let behaviors: [RequestBehavior]
    
    var additionalHeaders: [String : String] {
        return behaviors.reduce([String: String](), { sum, behavior in
            return sum.merging(behavior.additionalHeaders, uniquingKeysWith: { a, b in a })
        })
    }
    
    func beforeSend() {
        behaviors.forEach { $0.beforeSend() }
    }
    
    // separate success failure
    func afterSuccess(result: AnyResponse)  {
        behaviors.forEach { $0.afterSuccess(result: result) }
    }
    
    func afterFailure(error: Error)  {
        behaviors.forEach { $0.afterFailure(error: error) }
    }
}
```  
このCombinedRequestBehaviorを使うことで  
当初の形のまま複数のBehaviorも扱うことができます。  
  
```swift
func send(
    request: Request,
    with requestBehavior: RequestBehavior = EmptyRequestBehavior()
)

let additionalBehavior: [RequestBehavior]
    = CombinedRequestBehavior(behaviors: [AdditionalBehavior1(), AdditionalBehavior2()])

send(request: request, with: additionalBehavior)
```  
  
コードが短くなることは良いことです。  
  
## Step4 具体的に個々の処理を抽出していく  
  
それでは具体的に個々の機能を抽出していきます。  
例として認証ヘッダーを追加するBehaviorを定義してみます。  
  
```swift

final class AuthTokenHeaderBehavior: RequestBehavior {
    
    let defaults = UserDefaults.standard
    
    var additionalHeaders: [String : String] {
        if let token = defaults.string(forKey: "authToken") {
            return ["X-Auth-Token": token]
        } else {
            return [:]
        }
    }
}
```  
  
UserDefaultsにトークンがあれば  
ヘッダーの値を返却し  
なければ空を返すだけのシンプルで明確なクラスです。  
  
テストの数も２ケースで完了します。  
  
## Step5 メリットを振り返る  
  
上記のようなプロセスを通しての  
メリットを考えてみたいと思います。  
  
### 他に何を構築することができる？  
  
個々の機能が独立しているため  
コードを汚くしたり  
お互いが影響し合うような難しいテストをする必要もなくなります。  
バグの修正も楽にできます。  
  
## 何を得られたか？  
  
今回行ったプロセスは  
広い意味でいうと  
  
別々の要素を  
別々に「コントロールする」ための  
明確な「境界」を構築する「非干渉化(decoupling)」  
  
を得ることができました。  
  
これによって  
他への影響を与えずに  
機能の追加を簡単に行うことができるようになります。  
  
「コントロールする」というのは  
下記のことができることを指します。  
  
- テストが簡単にできること  
- 再利用できること  
- ひと目で全体を見ることができること(読みやすい。理解しやすい)  
  
# まとめ  
  
コードは機能の追加や仕様の変更により  
どんどん変化していくものです。  
  
その中で単純に処理を追加していくと  
本来の意図が不明瞭になったり  
条件分岐が増えてバグが発生しやすくなったり  
バグの修正も困難になってくることが多々あります。  
  
そんな時に今回のようなプロセスを通して  
機能の明確な境界を構築し  
コードをきれいに保ち  
本来の意図を明確にし続けていくことが大切になります。  
  
日々の変化に対応し  
より安全でスムーズな開発を  
進めていきましょう😄  
