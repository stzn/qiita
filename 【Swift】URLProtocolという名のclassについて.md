# 経緯  
  
Swiftにはいわゆる一般的に言う抽象クラスという概念がありません。  
  
※  
コンパイラが検知してエラーにしないというだけでそれらしいものは作れます  
(メソッドを実装しないでfatalErrorを投げるなど)  
  
しかし  
そんな中でも抽象クラスと記載されているものがあります。  
  
その中の一つが**URLProtocol**です。  
https://developer.apple.com/documentation/foundation/urlprotocol  
  
  
**Protocol**と付いていますが**class**です  
  
冒頭の説明にも  
  
```html

An abstract class that handles the loading of protocol-specific URL data.
```  
  
とあります。  
  
WWDC2018で紹介されていたり  
よく記事などで見かけたりしていましたが  
具体的にどういう存在でどう活用できるのかについて  
よくわかっていなかったので調べたことをまとめます。  
  
WWDC 2018 Session 417 Testing Tips & Tricks  
https://developer.apple.com/videos/play/wwdc2018/417  
  
# URLProtocolとは  
  
iOSではインターネット通信を行う際に  
**URL Loading System**というものを使っています。  
  
簡単に言うと  
HTTPSなどのスタンダードな通信プロトコルを使って  
URLで特定されるリソースに非同期でアクセスするための仕組みです。  
  
※詳しくはドキュメントや過去にちょっとまとめたものがありますのでそちらをご参照頂けますと幸いです。  
https://developer.apple.com/documentation/foundation/url_loading_system  
https://github.com/stzn/qiita/blob/master/【Swift】URLSessionまとめ.md  
  
**URLProtocol**はURL　Loading Systemの通信に使われるインターフェイスで  
ネットワーク通信を開始し  
リクエストを送ってレスポンスを受け取ります。  
  
URLProtocolはサブクラス化することを前提としており  
Foundationフレームワークは  
HTTPSなどのスタンダードなProtocolに対するサブクラスを  
ビルドインで提供してくれています。  
  
URLProtocolのサブクラスは以下のメソッドを実装します。  
  
## class func canInit(with request: URLRequest) -> Bool  
  
このクラスが引数に渡されるrequestを扱う必要があるかを判断します。  
trueの場合は処理を継続します。  
  
## class func canInit(with task: URLSessionTask) -> Bool  
  
このクラスが引数に渡されるtaskを扱う必要があるかを判断します。  
trueの場合は処理を継続します。  
  
  
上記2つの内いずれかをoverrideしないと下記のエラーが出ます。  
  
```
*** Terminating app due to uncaught exception 
'NSInvalidArgumentException', 
reason: '*** -canInitWithRequest: only defined for abstract class.  
Define -[URLProtocolSampleTests.MockURLProtocol canInitWithRequest:]!'
```  
  
※  
ドキュメントでは  
  
```
you should override the task-based methods when subclassing,
```  
  
などと書いてあるようにtaskのメソッドをoverrideする方が良いように書いてあるものの  
WWDC2018ではrequestのメソッドを使用しているはなぜだろうというのはまた別の話  
  
## class func canonicalRequest(for request: URLRequest) -> URLRequest  
  
リクエストで送られたURLRequestにヘッダーを追加したりなどカスタマイズできます。  
特に何もする必要がなければ引数のrequestをそのまま返します。  
  
このメソッドもoverrideしないとエラーになります。  
  
## func startLoading()  
  
リクエストの読み込みを開始して  
URLProtocolClientを介して  
Systemにフィードバック(DataやHTTPURLResponseなど)を提供します。  
  
URLProtocolClient ※これはProtocol  
https://developer.apple.com/documentation/foundation/urlprotocolclient  
  
※  
ドキュメントにもありますが  
このprotocolは直接実装せず  
URLProtocolのサブクラスの中で呼び出します。  
  
# func stopLoading()  
  
リクエストがキャンセルされた場合などに呼ばれます。  
  
サードライブラリを見てみると  
ここでキャンセルフラグを立てて処理をストップさせたり  
URLSessionTaskのcancelを行なったりするようです。  
  
https://github.com/Alamofire/Alamofire/blob/master/Tests/URLProtocolTests.swift  
https://github.com/ishkawa/APIKit/blob/master/Tests/APIKitTests/TestComponents/HTTPStub.swift  
  
このメソッドもoverrideしないとエラーになります。  
  
  
# URLSessionとの関係  
  
URLSessionはURLProtocolのサブクラスの中で  
送られてきたリクエストの処理が可能なものを探して処理を行います。  
  
<img width="1380" alt="スクリーンショット 2019-02-02 10.08.25.png" src="/image/1f5928d8-75b3-6ef5-f5a9-fe5497b22b55.png">  
WWDC2018 Session417 p47  
  
# どう活用するか？  
  
WWDCの動画にもあるように  
テスト時のリモート通信のモックに使用できます。  
  
他にもログの挿入やデバッグ時の監視することにも活用されているようです。  
  
この方法を使うといくつかのメリットが考えられます。  
  
## プロダクションコードに影響を与えない  
  
色々な実装を見ていると  
URLSessionのメソッドに合わせたProtocolを作成し  
それに適合させたMockを作成して  
利用するクラスでProtocolをDIする  
  
というパターンをよく見ますが  
途中から通信をモックしたテストを追加しようとする場合  
プロダクションコードに変更を加える必要があります。  
  
一方でURLProtocolを使用すると  
実際の通信をインターセプトして戻り値を自由に返すことができるようになるため  
プロダクションコードは特に変更する必要がなくなります。  
  
また  
リクエストを行うメソッドがstaticメソッドかインスタンスメソッドかの影響もありません。  
  
## 実装の変更にテストが影響を受けない  
  
例えば  
URLSessionを使っていたところをAlamofireに変更したとしても  
モックの部分がやることは変わらないためMockやテストケースに修正は必要ありません。  
初期のセットアップを変えるだけで良くなります。  
  
# 実装例  
  
ではWWDCの動画を参考に実装を見てみたいと思います。  
  
コードはWWDC2018の動画のコードをほぼ参照しています。  
  
まずは仮のプロダクションコードとして使用するコードです。  
  
```swift

// 通信をするクラス

protocol APIRequest {
    associatedtype RequestDataType
    associatedtype ResponseDataType
    
    func makeRequest(from data: RequestDataType) throws -> URLRequest
    func parseResponse(data: Data) throws -> ResponseDataType
}

final class APIRequestLoader<T: APIRequest> {
    let apiRequest: T
    let urlSession: URLSession
    
    init(apiRequest: T, urlSession: URLSession = .shared) {
        self.apiRequest = apiRequest
        self.urlSession = urlSession
    }
    
    func loadAPIRequest(requestData: T.RequestDataType,
                        completionHandler: @escaping (T.ResponseDataType?, Error?) -> Void) {
        do {
            let urlRequest = try apiRequest.makeRequest(from: requestData)
            
            urlSession.dataTask(with: urlRequest) { data, response, error in
                guard let data = data else { return completionHandler(nil, error) }
                do {
                    let parsedResponse = try self.apiRequest.parseResponse(data: data)
                    completionHandler(parsedResponse, nil)
                } catch {
                    completionHandler(nil, error)
                }
                }.resume()
        } catch { return completionHandler(nil, error) }
    }
}
```  
  
```swift

// レスポンスで返ってくる想定のデータ

struct PointOfInterest: Decodable, Equatable {
    let name: String
}


// URLRequestとレスポンスのDataをパースするクラス

struct PointsOfInterestRequest: APIRequest {

    enum RequestError: Error {
        case invalidCoordinate
    }

    func makeRequest(from coordinate: CLLocationCoordinate2D) throws -> URLRequest {
        guard CLLocationCoordinate2DIsValid(coordinate) else {
            throw RequestError.invalidCoordinate
        }
        var components = URLComponents(string: "https://example.com/locations")!
        components.queryItems = [
            URLQueryItem(name: "lat", value: "\(coordinate.latitude)"),
            URLQueryItem(name: "long", value: "\(coordinate.longitude)")
        ]
        return URLRequest(url: components.url!)
    }
    
    func parseResponse(data: Data) throws -> [PointOfInterest] {
        return try JSONDecoder().decode([PointOfInterest].self, from: data)
    }
}

```  
  
それではテストモジュールの方でMockとテストを見ていきます。  
  
  
```swift

final class MockURLProtocol: URLProtocol {
    
    // 期待するResponseとDataを保持する
    static var requestHandler: ((URLRequest) throws -> (HTTPURLResponse, Data))?
 
    // 来たリクエストを処理するかどうか   
    override class func canInit(with request: URLRequest) -> Bool {
        return true
    }
    
    // 来たリクエストに必要な処理をすることができる  
    override class func canonicalRequest(for request: URLRequest) -> URLRequest {
        return request
    }
    
    // リクエストを読み込んでURLProtocolClientを介してSystemに結果を返す  
    override func startLoading() {

        guard let handler = MockURLProtocol.requestHandler else {
            XCTFail("Received unexpected request with no handler set")
            return
        }
        
        do {
            let (response, data) = try handler(request)
            client?.urlProtocol(self, didReceive: response, cacheStoragePolicy: .notAllowed)
            client?.urlProtocol(self, didLoad: data)
            client?.urlProtocolDidFinishLoading(self)
        } catch {
            client?.urlProtocol(self, didFailWithError: error)
        }
    }
    
    // リクエストがキャンセルされた場合に呼ばれる
    override func stopLoading() {}
}
```  
  
この中で実際に通信を行う部分のstartLoadingを見ていきます。  
  
```swift

guard let handler = MockURLProtocol.requestHandler else {
    XCTFail("Received unexpected request with no handler set")
    return
}
```  
まず戻り値を定義しないとテストは失敗するようになっています。  
  
  
```swift

do {
    let (response, data) = try handler(request)
    client?.urlProtocol(self, didReceive: response, cacheStoragePolicy: .notAllowed)
    client?.urlProtocol(self, didLoad: data)
    client?.urlProtocolDidFinishLoading(self)
} catch {
    client?.urlProtocol(self, didFailWithError: error)
}
```  
  
handlerから取得したHTTPURLResponseとDataを  
URLProtocolClientのメソッドに渡すことで  
Systemに結果を返しています。  
  
  
これを下記のように使います。  
  
```swift

class APILoaderTests: XCTestCase {

    var loader: APIRequestLoader<PointsOfInterestRequest>!
    
    override func setUp() {
        let request = PointsOfInterestRequest()
        
        let configuration = URLSessionConfiguration.ephemeral
        configuration.protocolClasses = [MockURLProtocol.self]
        let urlSession = URLSession(configuration: configuration)
        
        loader = APIRequestLoader(apiRequest: request, urlSession: urlSession)
    }
    
    func testLoaderSuccess() {
        let inputCoordinate = CLLocationCoordinate2D(latitude: 37.3293, longitude: -121.8893)
        
        let mockJSONData = "[{\"name\":\"MyPointOfInterest\"}]".data(using: .utf8)!
        
        MockURLProtocol.requestHandler = { request in
            XCTAssertEqual(request.url?.query?.contains("lat=37.3293"), true)
            return (HTTPURLResponse(), mockJSONData)
        }
        
        let expectation = XCTestExpectation(description: "response")
        loader.loadAPIRequest(requestData: inputCoordinate) {  pointsOfInterest, error in
            XCTAssertEqual(pointsOfInterest, [PointOfInterest(name: "MyPointOfInterest")])
            expectation.fulfill()
        }
        wait(for: [expectation], timeout: 1)
    }
}

```  
  
これも分解して見ていきます。  
  
```swift

override func setUp() {
    let request = PointsOfInterestRequest()
   
    // 🌟
    let configuration = URLSessionConfiguration.ephemeral
    configuration.protocolClasses = [MockURLProtocol.self]
    let urlSession = URLSession(configuration: configuration)
        
    loader = APIRequestLoader(apiRequest: request, urlSession: urlSession)
}
```  
  
初期処理ですが🌟の部分が大事です。  
  
```swift

configuration.protocolClasses = [MockURLProtocol.self]
```  
  
ここで作成したMockを登録することで  
URLSessionがMockを見つけてくれるようになります。  
  
https://developer.apple.com/documentation/foundation/urlsessionconfiguration/1411050-protocolclasses  
  
ただし  
この場合URLSession.sharedを使用した場合はMockは検索対象になりません。  
  
```swift

loader = APIRequestLoader(apiRequest: request, urlSession: URLSession.shared)
```  
  
  
URLSession.sharedを使用する場合は下記のようにします。  
  
```swift

override func setUp() {
    let request = PointsOfInterestRequest()
    URLProtocol.registerClass(MockURLProtocol.self)
    loader = APIRequestLoader(apiRequest: request, urlSession: URLSession.shared)
}
    
override func tearDown() {
    URLProtocol.unregisterClass(MockURLProtocol.self)
}
```  
https://developer.apple.com/documentation/foundation/urlprotocol/1407208-registerclass  
  
しかし逆にshared以外は使えなくなります。  
  
さらにAlamofireなどは内部でsharedではないURLSessionを使用しているため  
上記の方法だとテストがうまくできません。  
  
https://github.com/Alamofire/Alamofire/issues/1160  
  
使う場合には注意が必要です。  
  
  
どういう動きをするか見てみた結果が下記です。  
  
### configuration.protocolClassesに設定した場合   
  
```swift

let configuration = URLSessionConfiguration.ephemeral

//　設定しない
// configuration.protocolClasses = [MockURLProtocol.self]

let urlSession = URLSession(configuration: configuration)
        
print(configuration.protocolClasses)

// Optional([_NSURLHTTPProtocol, _NSURLDataProtocol, 
//           _NSURLFTPProtocol, _NSURLFileProtocol, NSAboutURLProtocol])
```  
  
```swift

let configuration = URLSessionConfiguration.ephemeral
configuration.protocolClasses = [MockURLProtocol.self]
let urlSession = URLSession(configuration: configuration)
        
print(configuration.protocolClasses)

// Optional([URLProtocolSampleTests.MockURLProtocol]) 
```  
  
protocolClassesを上書きしているのがわかります。  
  
### URLProtocol.registerClassに設定した場合   
  
```swift

// 設定しない
// URLProtocol.registerClass(MockURLProtocol.self)

let configuration = URLSession.shared.configuration

print(configuration.protocolClasses)

Optional([_NSURLHTTPProtocol, _NSURLDataProtocol, _NSURLFTPProtocol, _NSURLFileProtocol, NSAboutURLProtocol])
```  
  
  
```swift

URLProtocol.registerClass(MockURLProtocol.self)
let configuration = URLSession.shared.configuration

print(configuration.protocolClasses)

Optional([_NSURLHTTPProtocol, _NSURLDataProtocol, _NSURLFTPProtocol, _NSURLFileProtocol, NSAboutURLProtocol])
```  
  
この場合はconfiguration.protocolClassesの値は変わらないようです。  
  
  
URLProtocolの実装を見てみると  
  
```swift

_registeredProtocolClasses
```  
に登録しているようですが  
中身が見られないためおそらくとまでしかわかりませんでした:bow_tone1:  
  
https://github.com/apple/swift-corelibs-foundation/blob/master/Foundation/URLProtocol.swift#L357  
  
そしてURLSession.sharedはこの値を見ています。  
  
```swift

configuration.protocolClasses = URLProtocol.getProtocols()
```  
  
https://github.com/apple/swift-corelibs-foundation/blob/f51bd9f81230fbc3fd5bc47e3a5f2af04de2a4e8/Foundation/URLSession/URLSession.swift#L213  
  
  
ひとまず設定方法の違いで見ている場所が違うということはわかりました。  
  
  
# Alamofireの場合  
  
ではAlamofireを使った場合を見てみます。  
バージョンは5.0.0-betaです。  
  
※久々に使ったら記法が変わっていてびっくりしました。  
  
```swift

class AlamofireAPILoaderTests: XCTestCase {

    var loader: AlamofireAPIRequestLoader<PointsOfInterestRequest>!
    
    override func setUp() {
        let request = PointsOfInterestRequest()
        
        let configuration = URLSessionConfiguration.ephemeral
        configuration.protocolClasses = [MockURLProtocol.self]
        let session = Session(configuration: configuration)
        loader = AlamofireAPIRequestLoader(apiRequest: request, session: session)
    }
    
    func testLoaderSuccess() {
        let inputCoordinate = CLLocationCoordinate2D(latitude: 37.3293, longitude: -121.8893)
        
        let mockJSONData = "[{\"name\":\"MyPointOfInterest\"}]".data(using: .utf8)!
        
        MockURLProtocol.requestHandler = { request in
            XCTAssertEqual(request.url?.query?.contains("lat=37.3293"), true)
            return (HTTPURLResponse(), mockJSONData)
        }
        
        let expectation = XCTestExpectation(description: "response")
        loader.loadAPIRequest(requestData: inputCoordinate) {  pointsOfInterest, error in
            XCTAssertEqual(pointsOfInterest, [PointOfInterest(name: "MyPointOfInterest")])
            expectation.fulfill()
        }
        wait(for: [expectation], timeout: 1)
    }
}
```  
  
Mockは何も変わりません。  
テストケースも何も変わりません。  
  
初期設定の時にloaderに渡すクラスを変えるだけでした。  
  
  
# まとめ  
  
URLProtocolについて見てみました。  
  
簡単にMockが作成できることや  
変更が起こった際に影響範囲を狭くできるのは良いですね。  
  
特に既存のコードに対してテストを追加する際に  
テストをするためにプロダクションコードを変更するという経験があり  
当時URLProtocolを活用できていたら  
そういったリスクも減らすことができたのではないかなと感じています。  
  
  
また本題とは別の話ですが  
WWDCで使われていたサンプルを見ていく中で  
Appleの人のAPI通信クラスの定義の仕方やコードの書き方など  
改めて発見したこともありました。  
  
※ こんな風に書くんだと思ったところ:eyes:  
  
```swift

guard let data = data else { return completionHandler(nil, error) }
```  
  
WWDCの動画は結構見たつもりでしたが  
何度見ても学べるものはあるなと感じ  
機会を見て動画の見直しをしていこうと思います。  
  
  
何かご指摘などございましたら教えていただけるとうれしいです:bow_tone1:  
