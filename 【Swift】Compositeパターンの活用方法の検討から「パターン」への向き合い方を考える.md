  
# Compositeパターンとは？  
  
```text

Compose objects into tree structures to represent part-whole hierarchies. 
Composite lets clients treat individual objects and compositions of objects uniformly.
```  
  
「構造に関するパターン」に属するデザインパターンの1つで  
ざっくりというとある型とその型を要素に構成される集合体も  
同じものとみなして扱えるようにするパターンです。  
  
Wikipediaの定義はこちら  
https://ja.wikipedia.org/wiki/Composite_%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3  
  
  
# なぜ使うのか？  
  
### 処理が簡潔になり、読みやすくなる  
  
個々の要素も要素の集合も同じ処理で扱うことができるため  
例えば要素の合計を計算したい場合やまとめて同じ動作をさせたい場合などに  
要素の検索や処理の分岐がなくなり  
コードがすっきりとわかりやすくなります。  
そうすることで不具合が入る余地や  
将来的に変更が入った場合の影響を減らすことができます。  
  
### 責務がより明確に分かれる  
  
例えば  
条件分岐でクラスを使い分ける必要がある処理があり  
呼び出し側でその条件分岐をおこなっている場合  
仮に呼び出すクラスの条件に変更が生じた際は  
呼び出し側の変更が必要になってしまいます。  
  
モジュールを分割している場合だと  
呼び出し側のモジュールの変更が発生し再ビルドが必要になってしまいます。  
  
また  
呼び出し側のテストを行う場合にも  
呼び出される側の事情を考慮する必要が出てくるため  
余計なセットアップやテストケースが増える可能性もあります。  
  
Compositeパターンを使うことで責務を分離し  
こういった変更の影響を最小限に抑えることにも繋がります。  
  
  
# SwiftでCompositeパターンを検討してみる  
  
protocolを定義して、これに適合させることで  
Compositeパターンを実現することができます。  
  
言葉だけだと漠然としてしまうので  
いくつかの具体例から考えてみたいと思います。  
  
## 合計を計算する  
  
例えば全国の小学生1年生を対象に  
算数のテストを行い  
そのテスト結果の合計や平均などを調査したいとします。  
  
まずは個々の要素と集合体で共通のProtocolを定義します。  
  
```swift

protocol Examinee {
    var name: String { get }
    var score: Int { get }
    func report() -> String
}
```  
  
次に個々の学生を定義します。  
  
```swift

struct Student: Examinee {
    let name: String
    let score: Int
    
    func report() -> String {
        return "\(name)の点数は\(score)です。"
    }
}
```  
  
次に受験生の集合体を定義します。  
  
```swift

struct Unit: Examinee {
    let name: String
    let subunits: [Examinee]
    var score: Int {
        return subunits.map { $0.score }.reduce(0, +)
    }
    func report() -> String {
        return "\(name)の合計は\(score)です。"
    }
}
```  
  
下記が使用例です。  
  
```swift

let student1 = Student(name: "Stu1", score: 51)
let student2 = Student(name: "Stu2", score: 52)
let student3 = Student(name: "Stu3", score: 83)
let student4 = Student(name: "Stu4", score: 84)

student1.report() // Stu1の点数は51です。
student2.report() // Stu2の点数は52です。
student3.report() // Stu3の点数は83です。
student4.report() // Stu4の点数は84です。

let school1 = Unit(name: "School1", subunits: [student1, student2])
let school2 = Unit(name: "School2", subunits: [student3, student4])

school1.report() // School1の合計は103です。
school2.report() // School2の合計は167です。

let area1 = Unit(name: "Area1", subunits: [school1, school2])
area1.report() // Area1の合計は270です。

let area2 = Unit(name: "Area2", subunits: [school1, student3])
area2.report() // Area2の合計は186です。

let prefecture1 = Unit(name: "Pre1", subunits: [area1, school2, student4])
prefecture1.report() // Pre1の合計は521です。
```  
  
このように生徒も学校も地域も県も同じように扱うことができ  
reportメソッドを呼ぶだけで色々な合計値を得ることができます。  
  
## 複数の箇所にデータを送る  
  
あるイベントに対して  
複数のアナリティクスへデータ送ったり  
ログに書き込むといったことはよくありそうな状況です。  
  
そうした際に1つ1つ処理するのは面倒です。  
  
そこでCompositeパターンがよく使われます。  
  
まずは共通で使用するProtocolを定義します。  
  
```swift

// データを送る何か
protocol AnalyticEngine {    
    func record(event: AnalyticEvent)    
}

// 送信するデータ
protocol AnalyticEvent {
    var name: String { get }
}
```  
  
適当に具体的なクラスを定義します。  
  
  
```swift

struct DefaultEvent: AnalyticEvent {
    let name: String
}

struct OSLogAnalyticEngine: AnalyticEngine {
    func record(event: AnalyticEvent) {
        print("\(event.name) to OSLog")
    }    
}

struct FirebaseAnalyticEngine: AnalyticEngine {
    func record(event: AnalyticEvent) {
        print("\(event.name) to Firebase")
    }
}

struct CrashlyticsAnalyticEngine: AnalyticEngine {
    func record(event: AnalyticEvent) {
        print("\(event.name) to Crashlytics")
    }
}
```  
  
  
次にAnalyticsEngineの集合を扱えるようにします。  
  
```swift

final class CompositeEngine: AnalyticEngine {
    
    private let engines: [AnalyticEngine]
    
    public init(engines: AnalyticEngine...) {
        self.engines = engines
    }
    
    public func record(event: AnalyticEvent) {
        engines.forEach { $0.record(event: event) }
    }
}
```  
  
```Swift

let firebase = FirebaseAnalyticEngine()
let crashlytics = CrashlyticsAnalyticEngine()
let osLog = OSLogAnalyticEngine()

let event1 = DefaultEvent(name: "サンプルイベント1")
firebase.record(event: event1)
```  
  
結果  
  
```txt

サンプルイベント1 to Firebase
```  
  
```swift

let composite1 = CompositeEngine(engines: firebase, crashlytics, osLog)
let event2 = DefaultEvent(name: "サンプルイベント2")
composite1.record(event: event2)
```  
  
結果  
  
```txt

サンプルイベント2 to Firebase
サンプルイベント2 to Crashlytics
サンプルイベント2 to OSLog
```  
  
  
```swift

let composite2 = CompositeEngine(engines: firebase, osLog)
let event3 = DefaultEvent(name: "サンプルイベント3")
composite2.record(event: event3)
```  
  
結果  
  
```txt

サンプルイベント3 to Firebase
サンプルイベント3 to OSLog
```  
  
このように1つのイベントに対して  
複数のAnalyticsEngineにデータを送信することができます。  
  
  
## ネットワークの状況に応じてデータの取得先を変える  
  
データをインターネット上から取得したいけど  
オフラインであるため取得できない場合はローカルのデータを使う  
という状況があったとします。  
  
このような場合に  
条件分岐をどこで行うか  
という問題が出てきます。  
  
呼び出し側でネットワークの状況を確認すると  
例えば取得方法が変わったりした際に  
呼び出し側の変更が発生してしまいます。  
  
それをCompositeパターンを使って  
呼びだされる側で処理を簡潔させる方法を検討してみます。  
  
  
まずはデータを取得するProtocolを定義します。  
  
※ 下記の内容はWWDC2018の中に出てきたクラスを参考にしています。  
細かい詳細は割愛させていただきます。  
https://developer.apple.com/videos/play/wwdc2018/417/  
  
  
```swift

protocol RequestLoader {
    associatedtype R: Request
    func loadRequest(requestData: R.RequestDataType,
                     completionHandler: @escaping (R.ResponseDataType?, Error?) -> Void)
}
```  
  
次にローカルからデータを取得するクラスを定義します。  
  
```swift

final class LocalRequestLoader<R: Request>: RequestLoader {
    let request: R
    
    enum Error: Swift.Error {
        case invalidURL
        case responseError(Swift.Error)
    }
    
    init(request: R) {
        self.request = request
    }
    
    func loadRequest(requestData: R.RequestDataType,
                     completionHandler: @escaping (R.ResponseDataType?, Swift.Error?) -> Void) {
        do {
            let urlRequest = try request.makeRequest(from: requestData)
            
            guard let url = urlRequest.url else { throw Error.invalidURL }
            let data = try Data(contentsOf: url)
            let parsedResponse = try self.request.parseResponse(data: data)
            completionHandler(parsedResponse, nil)
        } catch { return completionHandler(nil, Error.responseError(error)) }
    }
}
```  
  
次にリモートからデータを取得するクラスを定義します。  
  
```swift

final class RemoteRequestLoader<R: Request>: RequestLoader {
    let request: R
    let urlSession: URLSession
    
    init(request: R, urlSession: URLSession = .shared) {
        self.request = request
        self.urlSession = urlSession
    }
    
    func loadRequest(requestData: R.RequestDataType,
                     completionHandler: @escaping (R.ResponseDataType?, Error?) -> Void) {
        do {
            let urlRequest = try request.makeRequest(from: requestData)
            let task = urlSession.dataTask(with: urlRequest) { data, response, error in
                guard let data = data else { return completionHandler(nil, error) }
                do {
                    let parsedResponse = try self.request.parseResponse(data: data)
                    completionHandler(parsedResponse, nil)
                } catch {
                    completionHandler(nil, error)
                }
            }
            task.resume()
            
        } catch { return completionHandler(nil, error) }
    }
}
```  
  
それでは  
ネットワークの状態によって  
取得するデータの場所を変えるようにします。  
  
```swift

final class CompositeRequestLoader<R: Request>: RequestLoader {
    
    let remoteLoader: RemoteRequestLoader<R>
    let localLoader: LocalRequestLoader<R>
    init(request: R) {
        self.remoteLoader = RemoteRequestLoader(request: request)
        self.localLoader = LocalRequestLoader(request: request)
    }
    
    func loadRequest(requestData: R.RequestDataType,
                        completionHandler: @escaping (R.ResponseDataType?, Swift.Error?) -> Void) {
        // ネットワークの接続状況を確認する
        if Reachability.isConnectedToNetwork() {
            remoteLoader.loadRequest(requestData: requestData, completionHandler: completionHandler)
        } else {
            localLoader.loadRequest(requestData: requestData, completionHandler: completionHandler)
        }
    }
}
```  
  
こうすることで  
呼び出し側は呼び出される側の状況を気にせずに  
ある場所からデータを取得してくることができるようになります。  
  
また仮に取得方法が変更になった場合でも  
呼び出し側に影響を与えることはなくなります。  
  
# 他にも  
  
SwiftでのCompositeパターンの活用例は多く  
例えばWWDC2016のセッションではUIKitへの活用方法も紹介されています。  
https://developer.apple.com/videos/play/wwdc2016/419/  
  
  
# まとめ  
  
Compositeパターンの使い方を検討してみました。  
  
このようなデザインパターンは  
**なんとなくみんなが良いと言うからそうしている**  
ということが結構あり  
  
**「あれ？これってこのパターンと同じ状況だ」**  
とあとで気がつき  
  
**同じような状況に遭遇しても応用できていない**  
と痛感することがあります。  
  
  
- なぜこのパターンが使われているのか？  
- このパターンで解決できる問題は何か？  
- どういう経緯でこのパターンは生まれてきたのか？  
  
こういったことを考えることで  
より理解深めることができると同時に  
似たような問題に遭遇した際に  
柔軟に対応できようになると考えています。  
  
デザインパターンに限らず  
何かのいわゆる「パターン」を見つけた際には  
上記のことを意識して  
もっと状況に即して活用できるようにしていきたいと思います。  
  
※  
このようなことは繰り返し感じたり考えることではありますが  
なんとなく使ってしまうことが多い自分に対しての自戒の念も込めて  
定期的にアウトプットしてもっと意識できるようにします:innocent:  
  
間違いやご指摘などございましたら教えていただけるとうれしいです:bow_tone1:  
