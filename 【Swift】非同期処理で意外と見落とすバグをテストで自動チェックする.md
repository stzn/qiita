  
非同期処理を書く機会は多いとは思いますが、  
意外な落とし穴に出会うことがあります。  
(と、少なくとも私は思っています:sweat_smile:)  
  
今回はそんなことが起きそうな事例と  
テストを使って自動でチェックする方法について  
検討してみたいと思います。  
  
  
↓を参考にしています。  
https://github.com/essentialdevelopercom/essential-feed-case-study/blob/master/EssentialFeed/EssentialFeedTests/Feed%20API/RemoteFeedLoaderTests.swift  
  
  
# 実装の準備  
  
よくありそうな非同期でデータを取得する処理を考えてみます。  
  
  
```swift

// どこかからデータをloadして結果をコールバックで返却するprotocol

protocol DataLoader {
    associatedtype T
    associatedtype E: Error
    func load(completion: @escaping (Result<T, E>) -> Void)
}

enum Result<T, Error> {
    case success(T)
    case failure(Error)
}

```  
  
```swift

// 実際に通信をする機能を持つprotocol 

enum HTTPClientError: Error {
    case invalidResponse
    case error(Error)
}

typealias HTTPResponse = (Data, HTTPURLResponse)

protocol HTTPClient {
    func get(from url: URL,
             completion: @escaping (Result<(HTTPResponse), HTTPClientError>) -> Void)
}
```  
  
```swift

// 欲しいデータ(今回ほぼ出てきません)

struct Item: Decodable {
    let id: String
    let name: String
}
```  
  
```swift

// DataLoaderに適合したclass

final class RemoteDataLoader: DataLoader {
    
    let client: HTTPClient
    let url: URL
    
    enum Error: Swift.Error {
        case invalidData
        case invalidStatus(Int)
        case unknown(Swift.Error)
    }
    
    init(client: HTTPClient, url: URL) {
        self.client = client
        self.url = url
    }
    
    func load(completion: @escaping (Result<Item, Error>) -> Void) {
        
        client.get(from: url) { result in
            
            switch result {
            case .success(let data, let response):
                
                guard response.statusCode == 200 else {
                    completion(.failure(.invalidStatus(response.statusCode)))
                    return
                }
                
                guard let item = ItemTranslator.map(data) else {
                    completion(.failure(.invalidData))
                    return
                }
                completion(.success(item))

            case .failure(let error):
                completion(.failure(.unknown(error)))
            }
        }
    }
}
```  
  
```swift

// DataからItemに変換するstruct

struct ItemTranslator {
    static func map(_ data: Data) -> Item? {

        // 呼ばれたかどうかを確認するためにコンソールに出力しています
        print("!!!!!!!!!!!!!!!!!!!map called!!!!!!!!!!!!!!!!!!!!!!!!!!!")

        return try? JSONDecoder().decode(Item.self, from: data)
    }
}
```  
  
# テストの準備  
  
次に確認するためにテストを準備します。  
  
```swift

class RemoteDataLoaderTests: XCTestCase {

    // 通信後に呼ばれるコールバックを記録しておいて
    // 任意のタイミングで呼び出せるようにするクラス

    private class HTTPClientSpy: HTTPClient {
        
        func get(from url: URL, completion: @escaping (Result<(HTTPResponse), HTTPClientError>) -> Void) {
            messages.append((url: url, completion: completion))
        }
        
        var messages: [(url: URL, completion: (Result<(HTTPResponse), HTTPClientError>) -> Void)] = []
        
        var urls: [URL] {
            return messages.map { $0.url }
        }
        
        // エラーの結果を返すためのメソッド        
        func call(with error: HTTPClientError, at index: Int = 0) {
            messages[index].completion(.failure(error))
        }

        // 正常な結果を返すためのメソッド        
        func call(statusCode: Int = 200, data: Data, at index: Int = 0) {
            let response = HTTPURLResponse(
                url: urls[index], statusCode: statusCode,
                httpVersion: nil, headerFields: nil)
            messages[index].completion(.success((data, response!)))
        }
    }

    // セットアップ

    override func setUp() {
        super.setUp()
        let url = URL(string: "https://hogehoge.com")!
        client = HTTPClientSpy()
        sut = RemoteDataLoader(client: client, url: url)
    }

    // テスト用のインスタンスを用意するヘルパーメソッド

    private func prepareInstancesForTest(
        url: URL = URL(string: "https://hogehoge.com")!
        ) -> (HTTPClientSpy, RemoteDataLoader) {
        
        let client = HTTPClientSpy()
        let loader = RemoteDataLoader(client: client, url: url)
        
        return (client, loader)
    }
}

```  
  
まずは通常の動作を確認してみます。  
  
```swift

    func test_通常処理() {

        let (client, loader) = prepareInstancesForTest()

        var results: [Result<Item, RemoteDataLoader.Error>] = []
        
        // loadの中でHTTPClientのgetを呼んでいるので
        // loadのcompletionはSpyのmessagesに追加される
        loader.load { results.append($0) }
        
        // callを呼ぶとresultsに値は追加される
        client.call(data: Data(), at: 0)
        
        // loadのcompletionは呼ばれているはずなのでresultsのcountは１になる
        XCTAssertEqual(results.count, 1)
    }
```  
  
これで準備が整いました。  
  
  
# メモリリークを確認する  
  
メモリリークは隠れたところに潜んでいることがあります。  
  
Debug Memory Graphを使用すれば  
最終的には見つけられる可能性は高いですが  
  
繰り返しチェックするのは  
時間的にも精神的にも面倒に感じることもある一方で  
  
しばらくチェックをしないと色々な場所でメモリリークが発生して  
原因が特定しづらくなってしまうということもあるのではないかと思います。  
  
そんな時にテストでチェックができる仕組みがあると  
良いのではないかと感じています。  
  
まずメモリリークを発生させるために  
RemoteDataLoaderのloadメソッドを下記のようにします。  
  
  
```swift

    func load(completion: @escaping (Result<Item, Error>) -> Void) {
        
        client.get(from: url) { result in
            
            self.hoge()
            
            switch result {
            case .success(let data, let response):
                
                guard response.statusCode == 200 else {
                    completion(.failure(.invalidStatus(response.statusCode)))
                    return
                }
                
                guard let item = ItemTranslator.map(data) else {
                    completion(.failure(.invalidData))
                    return
                }
                completion(.success(item))

            case .failure(let error):
                completion(.failure(.unknown(error)))
            }
        }
    }
    
    private func hoge() {
        print("hoge")
    }
```  
  
こうすることで  
クロージャの中でselfを強参照しているため  
selfがdeinitされなくなります。  
  
ではテストを再度実行するとどうなるでしょうか？  
  
<h3>成功します</h3>  
  
処理的には間違っていることがないからです。  
  
つまり  
**メモリリークが見逃されてしまう可能性がある**  
のです。  
  
  
そこで  
メモリリークをテストを通して自動でチェックできるようにしてみます。  
  
RemoteDataLoaderTestsのprepareInstancesForTestを下記のようにします。  
  
```swift

    private func prepareInstancesForTest(
        url: URL = URL(string: "https://hogehoge.com")!,
        file: StaticString = #file, line: UInt = #line
        ) -> (HTTPClientSpy, RemoteDataLoader) {
        
        let client = HTTPClientSpy()
        let loader = RemoteDataLoader(client: client, url: url)
        
        checkMemoryLeaks(loader, file: file, line: line)
        checkMemoryLeaks(client, file: file, line: line)
        
        return (client, loader)
    }
    
    private func checkMemoryLeaks(_ instance: AnyObject,
                                  file: StaticString = #file, line: UInt = #line) {
        addTeardownBlock { [weak instance] in
            XCTAssertNil(instance, "メモリリーク!!!!", file: file, line: line)
        }
    }
```  
  
addTeardownBlockは  
現在のテストの終了後のteardown処理をブロックで追加できるメソッドです。  
https://developer.apple.com/documentation/xctest/xctestcase/2887226-addteardownblock  
  
※ fileとlineはテストが失敗した箇所をわかりやすくするために追加しています。  
  
これを追加することでテストが失敗し  
メモリリークを確認することができるようになりました。  
  
![スクリーンショット 2019-01-27 11.05.05.png](/image/9a7fcf2d-3fc3-4031-3edf-0da36df76c3f.png)  
  
# メモリリークを解消  
  
これは単純な話で[weak self]をつければ解消します。  
  
```swift

    func load(completion: @escaping (Result<Item, Error>) -> Void) {
        
        client.get(from: url) { [weak self] result in
            
            self?.hoge()            
            ...
        }
    }
```  
  
  
# 非同期処理時の思わぬ挙動を確認する  
  
非同期処理を実装していると  
思わぬときに  
「あれ、なんでこのメソッド呼ばれているんだ？」  
みたいな事象に遭遇することがあります。  
  
そんな事象を確認するために  
下記のテストを追加します。  
  
```swift

    func test_非同期の挙動チェック() {

        let url = URL(string: "https://hogehoge.com")!
        let client = HTTPClientSpy()

        // nilにしたいのでOptionalにする
        var loader: RemoteDataLoader? = RemoteDataLoader(client: client, url: url)
        
        var results: [Result<Item, RemoteDataLoader.Error>] = []

        // loadの中でHTTPClientのgetを呼んでいるので
        // loadのcompletionはSpyのmessagesに追加される
        loader?.load { results.append($0) }

        // ここでテスト対象をnilにするので
        // callを呼んでもresultsに値は追加されないはず
        loader = nil
        client.call(data: Data(), at: 0)

        // loadのcompletionは呼ばれないはずなのでresultsは空のはず
        XCTAssertTrue(results.isEmpty)
    }
```  
  
テストを実行するとどうなるでしょうか？  
  
<h3>失敗します</h3>  
  
コンソールの出力を見てみるとtranslatorのmapメソッドが呼ばれています。  
  
```swift

Test Case '-[FeedDataMemoryLeakDetectionTests.RemoteDataLoaderTests test_非同期の挙動チェック]' started.
!!!!!!!!!!!!!!!!!!!map called!!!!!!!!!!!!!!!!!!!!!!!!!!!
```  
  
これはclientのHTTPClientSpyがcompletionを保持しているためです。  
  
loaderのdeinitと同時にclientもdeinitするようにすれば処理は発生しませんが  
clientがシングルトンであった場合などは困った状況になります。  
  
想定される場面としては  
ある画面でデータをロード中に前の画面に戻ったときに  
ViewControllerはdeinitされているのに  
裏でcompletionの処理が動いてしまう。  
  
などが考えられます。  
  
  
# 思わぬ挙動を解決する  
  
これも非常にシンプルですが  
RemoteDataLoaderのloadメソッドの中でインスタンスの存在チェックをします。  
  
```swift

    func load(completion: @escaping (Result<Item, Error>) -> Void) {
        
        client.get(from: url) { [weak self] result in
            
            // selfがdeinitしていた場合は処理をしない
            guard self != nil else {
                return
            }
            ...            
        }
    }
```  
  
こうすることで処理が発生しなくなります。  
  
  
# まとめ  
  
非同期処理はほぼ当たり前のように使用しており  
気をつけなればいけない箇所は把握しているかもしれませんが  
上記のような実は見落としているのかもしれないという可能性も捨て切れません。  
  
そんな時に手動での確認となると  
手間と時間がかかるのに加え  
確認漏れが発生する可能性があるなど  
意外と大掛かりな作業になってしまうかもしれません。  
  
そこで  
テストで自動確認できる仕組みを使って  
そういった不安と負担を軽減できたら嬉しいですね:smiley:  
  
何か間違いなどございましたらご指摘いただけますと幸いです:bow_tone1:  
