これはiOSDC2019の発表時に調べた  
Rob Napierさんのブログの翻訳(意訳)記事です。  
  
http://robnapier.net/start-with-a-protocol  
http://robnapier.net/a-mockery-of-protocols  
http://robnapier.net/thats-not-a-number  
  
  
ジェネリックなコードを構築していくときに考えるべきこと  
Protocolを使う上での具体的なテクニックが詰まったとても良い記事です。  
  
  
ここから翻訳(意訳)です。  
---  
  
# Start With a Protocolの真実  
  
## 最初にCrusty  
  
WWDC2015で行われたで「Protocol Oriented Programming」を提唱したセッションは  
今でも大きなインパクトを残したセッションとして記録に残っています。  
  
https://developer.apple.com/videos/play/wwdc2015/408/  
  
このセッションの中で  
  
> Start With a Protocol(Protocolから始めよう)  
  
という発言があり  
  
多くの開発者がよりProtocolを使用していこう考える  
きっかけになったのではないかと思います。  
  
私も  
新しい機能を追加する場合は  
まずProtocolを作成し  
それに適合する具体的な型を作成する  
という手順で開発することが基本になっています。  
  
しかし上記の発言は  
  
**Start with a protocol for every problem.(全ての問題に対してprotocolから始めよう)**  
  
という意味なのでしょうか？  
  
## 文脈のない引用にご注意  
  
実はセッション中での発言はもう少し長いフレーズとなっており  
  
> For example,   
if you want to write a generalized sort or binary search  
…Don’t start with a class.   
Start with a protocol.  
  
> 例えば、もし汎用的なソートやバイナリーサーチを実装したいとき  
クラスから始めるな  
Protocolから始めよう  
  
さらにセッションの登壇者であるDavid Abrahamさんはtwitterで下記のようにもおっしゃっています。  
  
  
> ...   
the context for that statement is easy to miss:   
...  
Point is, you should already have a need for generalization   
before you reach for a class or protocol. 1/  
  
> generalization over types (a.k.a. polymorphism), I mean.   
Classes mix polymorphism with the types themselves,   
and in a world where lots of people use classes for everything,   
“start with a protocol” probably should be preceded   
by “start with a struct or enum.” 2/  
  
> The relationship of properly-discovered protocols to algorithms is crucial,   
but that's not what I was focused on in the talk.  
It's “use value types, then if you need polymorphism, make them conform to protocols. Avoid classes.”  
  
> あの発言は誤解されやすい。  
大事なのはクラスの継承やprotocolに到達する前に  
汎用化(特定の型を超えた、いわゆる多態)  
する必要があるという状態にあるべきなんだ。  
  
> クラスは多態と自身の型をごっちゃにし、  
多くの人が全てに対してクラスを使用するような世界においては  
「strcutもしくはenumから始めよ」が  
「protocolから始めよ」よりも先に来るべきかもしれない。  
  
> ある問題への解決策として適切なProtocolを見つけることは非常に重要ですが、  
あのセッションの中で言いたかったことはそこではありません。  
「Value Typeを使おう。そしてもし多態が必要ならばprotocolに適合させましょう。クラスは避けること。」  
  
  
  
https://twitter.com/cocoaphony/status/1104114233288151043  
  
  
つまり  
  
**もしクラスの継承関係を構築しようと思った時は**  
**代わりにProtocolとValue Typeで実装することを試みよう**  
  
ということを言っていました。  
  
これは**全ての問題に対してprotocolから始めよう**とは全然異なります。  
  
Ben CohenさんはWWDC2018のセッションでより詳細にこのことについて言及しています。  
  
> So notice that we considered a varied number of concrete types first.   
And now, we’re thinking about a kind of   
protocol that could join them all together   
and then try and unify them with a protocol.   
And, it’s important to think of things as this way around.   
To start with some concrete types,   
and then try and unify them with a protocol.  
  
> 私たちはまず最初に数多くの具体的な型について考えていることに気がついた。  
そこからその具体的な型たちをまとめるためにprotocolなどについて考え  
protocolを使って一つにまとめることを試み、実行していった。  
このような手順で考えることは非常に重要です。  
まずは具体的な型で考え、  
そのあとにprotocolを使ってそれらをまとめることを試み、実装していくこと。  
  
https://developer.apple.com/videos/play/wwdc2018/406/  
  
今回の内容で一つだけ言いたいことを選べと言われたならば私はこう言いたいです。  
  
**「具体的なコードから書きはじめよ。そして汎用を見つけ出せ。」**  
  
まずは具体的な型から始め  
ユースケースを明確にして  
重複が起きている箇所を見つけ出す。  
  
そして共通する汎用的な箇所を見つけ出す。  
  
Protocol指向プログラミングのパワーは  
Protocolが実際に型として実装されるまで  
処理の詳細に関しては決めなくても良いところにあります。  
  
クラス継承を用いた場合  
早い段階で継承関係の設計をする必要が出てくるが  
Protocolだと後々まで引き延ばすことができます。  
  
私がProtocolで困ることがある時は  
大抵の場合「なるべく汎用化をする」ようにしていた時です。  
これは何も意味がありません。  
汎用化は選択肢であって  
ある方向に柔軟に対応できるように選択をすると  
一般的に別の方向に対しては逆に対応しづらくなってしまいます。  
  
明確なユースケースがないのに  
どういう汎用化が適しているのかを知ることはできません。  
  
## 事前準備  
  
ここからは具体的な実装例を見ていきます。  
  
Dataを非同期に取得して指定した型にデコードできる  
一般的なネットワーク通信のシステムを構築していきます。  
  
今回の目的は最終結果よりも構築していく過程で  
  
- いつ、どんな質問をするべきか？  
- どういう回答が良いとわかるのか？  
- 「Protocol指向」的なものがどうやって我々を導いてくれるのか？（最も重要）  
- 他のアプローチとどう違うのか？  
  
というところを見ていきたいと思います。  
  
まず最初に  
私にとっては決してうまくいかない共通の開始点を示します。  
  
私はこの過ちを繰り返しいつも追い詰められる。  
他の人が同じような過ちをしているのも見てきた。  
  
```swift

// RequestはURLRequestを知っていて、ある型のデータを取得してある型へ変換することができる
protocol Request {
    associatedtype Response
    func parse(data: Data) throws -> Response
    var urlRequest: URLRequest { get }
}
```  
  
なぜうまくいかないとわかるのか？  
  
Requestはassociatedtypeを持っている**protocol with associated type (PAT)**です。  
  
PATを作成する時にはいつも、自分自身に問いかけるべき質問があります。  
  
**「これを配列として使うことがあるか？」**  
  
もし回答がYesならばRequestがPATであることを望んでいないことになります。  
欲しいのはRequestを配列の中に入れたい何かであることが確かだからです。  
  
例えば  
  
- 実行待機しているRequestのリスト  
- リトライが必要なRequestのリスト  
- 優先順位の制御するためのキュートとしてのRequestのリスト  
  
他にもRequestを配列に入れたい理由はたくさんあります。  
  
何かワークアラウンドを見つけたくなるかもしれませんが、しない方がよいです。  
  
Type Eraser？  
Generalized Existential？？  
  
仮にいくつかの問題に対するワークアラウンドを見つけたとしても  
すぐに別の壁にぶつかるででしょう(私は何度も何度もぶつかっています)。  
  
「ジェネリックの制約にしか使用できない」ということが何か大切なことをあなたに伝えようとしているのです。  
  
これはSwiftの問題ではなく  
PATの問題でもありません。  
  
この問題を解決する他の方法があるということを教えてくれているのです。  
  
なぜあなたがこれらのワークアラウンドを望んでいないのか  
については後ほど説明しますが  
根本的な問題は望んでいる解決策がいったい何なのかということを知る前に  
Protocolから実装を始めたことです。  
  
では「解決策を知る」というプロセスは  
実際にどうやって見えてくるのでしょうか？  
  
## まずは具体的に  
  
汎用的な解決策を見つける良い方法は複数の具体的な解決策から始めて  
その中の異なる部分をパラメータとして切り出していくことです。  
  
今回のケースでは  
複数のモデルのデータをAPIから取得しデコードしたいとします。  
  
具体的に始めるためにまずは具体的な型を作成します。  
  
```swift

struct User: Codable, Hashable {
    let id: Int
    let name: String
}

struct Document: Codable, Hashable {
    let id: Int
    let title: String
}
```  
  
最終的な形とは異なるかもしれませんが  
今のところこれで十分です。  
  
UserとDocumentは似ていますが完全に一緒ではありません。  
後々共通的な要素を抽出していきますが  
現状はこのままで十分です。  
  
  
サーバーへの接続を管理するClientも作成します。  
クラス継承をさせないように**final**をつけます。  
  
全てのクラス定義に必ず**final**をつけることを薦めているわけではありません。  
大抵の場合は必要ありません。  
  
Clientが共有状態を持つことになる可能性があるため  
唯一の参照型(継承した子クラスが存在しない型)にするために**final**をつけています。  
例えばログインが必要な場合にClientへの参照でログイン状態を共有したいのです。  
  
```swift

// APIからデータを取得するためのClient
final class APIClient {
    let baseURL = URL(string: "https://www.example.com")!
    let session = URLSession.shared

    // メソッドはここの下にくる
}
```  
  
APIClient上のメソッドとして  
Userデータを取得してデコードするコードが欲しいとします。  
  
```swift

func fetchUser(id: Int, completion: @escaping (Result<User, Error>) -> Void)
{
    // URLRequestの作成
    let url = baseURL
        .appendingPathComponent("user")
        .appendingPathComponent("\(id)")
    let urlRequest = URLRequest(url: url)

    // URLSessionにURLRequestを送る
    let task = session.dataTask(with: urlRequest) { (data, _, error) in
        if let error = error { 
            completion(.failure(error))
        } else if let data = data {
            let result = Result { try JSONDecoder().decode(Model.self, from: data) }
            completion(result)
        }
    }
    task.resume()
}
```  
  
おそらく多くの方がこのようなコードを書かれたことがあるでしょう。  
  
1. URLRequestを作成し  
2. データを取得し  
3. 変換して  
4. completionHandlerへ渡す  
  
Documentに対するfetchDocumentはどうなりますでしょうか？  
  
```swift

func fetchDocument(id: Int, completion: @escaping (Result<Document, Error>) -> Void)
{
    // URLRequestの作成
    let url = baseURL
        .appendingPathComponent("document")
        .appendingPathComponent("\(id)")
    let urlRequest = URLRequest(url: url)

    // URLSessionにURLRequestを送る
    let task = session.dataTask(with: urlRequest) { (data, _, error) in
        if let error = error { 
            completion(.failure(error)) 
        } else if let data = data {
            let result = Result { try JSONDecoder().decode(Document.self, from: data) }
            completion(result)
        }
    }
    task.resume()
}
```  
  
驚くことはないと思うがほぼ同じです。  
  
違うのが四箇所で  
  
- 関数名  
- クロージャに渡す型  
- URLPath  
- デコードする型  
  
非常に似ているのは当然で  
これはUserをコピペして作成しています。  
  
コピペを見つけたときは  
おそらく再利用可能なコードを発見するポイントです。  
  
なのでジェネリックな関数に抽出しました。  
  
```swift

func fetch<Model: Decodable>(_: Model.Type, id: Int, 
                             completion: @escaping (Result<Model, Error>) -> Void)
{
   ...
}
```  
  
## 型パラメータはどこに置くべきか?  
  
先に進む前に  
シグネチャについて探究してみるのは価値があります。  
  
Modelの型をパラメータに渡していることに注目してください。  
引数名も必要としていません。これは値として使用することがないのです。  
  
なぜ必要なのか？  
  
関数の引数で型付けするためです。  
  
completionHandlerのパラメータに型を指定することで  
型推論を働かせることができますが  
`fetch(2) { ... }`という形だと  
現状はIDがInt型でUserでもDocumentも同じなので  
読み手が何のデータを取得しているのかがわかりづらくなります。  
  
これは確かにそうだと思うこともあれば  
逆に違うと思う場合もあります。  
  
私が納得のいった良い例は  
JSONDecoderのdecodeメソッドです。  
  
```swift

let value = try JSONDecoder().decode(Int.self, from: data)
```  
  
しかしこれはこういう書き方もできたはずです。  
  
```swift

let value: Int = try JSONDecoder().decode(data)
```  
  
コード自体は短くなりますが  
呼び出し側で変数に型アノテーションをつける必要が出てきます。  
これは不自然ですしSwiftらしくありません。  
  
もし型パラメータが現れるのが戻り値だけの場合は  
私はだいたいのケースで引数として渡すことをおすすめします。  
  
しかし  
どんな場合であっても  
呼び出し側で明確になるようにすることに注力しなければなりません。  
  
## fetchメソッドを汎用化する  
  
`fetch`の実装はかなり愚直です。  
ある**小さい問題**を除いては。  
  
```swift

func fetch<Model>(_ model: Model.Type, id: Int,
                  completion: @escaping (Result<Model, Error>) -> Void)
    where Model: Fetchable
{
    // Construct the URLRequest
    let url =  baseURL
        .appendingPathComponent("??? user | document ???")
        .appendingPathComponent("\(id)")
    let urlRequest = URLRequest(url: url)

    // Send it to the URLSession
    let task = session.dataTask(with: urlRequest) { (data, _, error) in
        if let error = error { 
           completion(.failure(error)) 
        } else if let data = data {
            let result = Result { try JSONDecoder().decode(Model.self, from: data) }
            completion(result)
        }
    }
    task.resume()
}
```  
  
"user"または"document"という箇所があります。  
  
これはこの解決策に必要なものですが  
Decodableの一部ではありません。  
  
つまりDecodableではパワーが足りないのです。  
新しいProtocolが必要になります。  
  
```swift

// APIからデータを取得することができる何か
protocol Fetchable: Decodable {
    static var apiBase: String { get }
}
```  
  
Decodableである型と  
追加の文字列`apiBase`を提供することが必須である  
Protocolが必要です。  
  
これを使って`fetch`を完成させると  
  
```swift

// IDを使ってFetchable型のデータを取得して非同期で返す
func fetch<Model>(_ model: Model.Type, id: Int,
                  completion: @escaping (Result<Model, Error>) -> Void)
    where Model: Fetchable
{
    let url = baseURL
        .appendingPathComponent(Model.apiBase)
        .appendingPathComponent("\(id)")
    let urlRequest = URLRequest(url: url)

    let task = session.dataTask(with: urlRequest) { (data, _, error) in
        if let error = error { completion(.failure(error)) }
        else if let data = data {
            let result = Result { try JSONDecoder().decode(Model.self, from: data) }
            completion(result)
        }
    }
    task.resume()
}
```  
  
## Retroactive Modeling  
  
上記のfetchメソッドを使用するために  
UserとDocumentをFetchableに適合させます。  
  
```swift

extension User: Fetchable {
    static var apiBase: String { return "user" }
}

extension Document: Fetchable {
    static var apiBase: String { return "document" }
}
```  
  
この小さなextensionが  
Protocol指向プログラミングの  
最も強力で簡単に見落としがちな側面の一つ  
**Retroactive Modeling**  
を表しています。  
  
Userのように元々はFetchableとして設計していなかった型に対しても  
extensionでFetchableとして拡張できることはあまり自明ではありません。  
  
しかも  
このextensionは同じモジュールにいなくてさえも良いのです。  
  
これはクラス継承ではできないことです。  
クラス継承の場合は型を定義する時に親クラスを決める必要があります。  
  
必要としているどんな型でも  
自分のProtocolを適合させることができ  
元々の型の作成者が  
思いもよらない新しい強力な方法で使うことができるようになります。  
  
Userを一つのユースケースやAPIに縛る必要はありません。  
だからこのProtocolは**Fetchable**であって  
**Model**みたいな名前ではないのです。  
  
これでは型を表すモデルではなく  
データを取得できる何かであって  
それを可能にするためのメソッドやプロパティを提供しているだけです。  
  
全てのユースケースに適合するような  
Protocolを作ることを薦めているわけではありません。  
むしろ逆です。  
  
本当に良いProtocolが多くの汎用的な解決方法として利用できます。  
しかしあなたがProtocolを利用するたいていの場合には  
そのProtocolが必須としているものを使って欲しいでしょう。  
  
もしProtocolがある型の全部のAPIをコピーしたものだとしたら  
そのProtocolは十分な役割を果たしていません。  
これについては後ほど紹介します。  
  
# プロトコルの偽物  
  
これまで実装内容は下記のようになります。  
  
```swift

protocol Fetchable: Decodable {
    static var apiBase: String { get }
}

final class APIClient {
    let baseURL = URL(string: "https://www.example.com")!
    let session: URLSession = URLSession.shared

    func fetch<Model: Fetchable>(_ model: Model.Type, id: Int,
                                 completion: @escaping (Result<Model, Error>) -> Void)
    {
        let url = baseURL
            .appendingPathComponent(Model.apiBase)
            .appendingPathComponent("\(id)")
        let urlRequest = URLRequest(url: url)

        let task = session.dataTask(with: urlRequest) { (data, _, error) in
            if let error = error {
                completion(.failure(error))
            } else if let data = data {
                let result = Result { try JSONDecoder().decode(Model.self, from: data) } 
                completion(result)
            }
        }
        task.resume()
    }
}
```  
  
まだまだ改善できそうな余地があります。  
  
自然に湧き上がる最初の疑問としては  
「どうやってテストをするのか？」です。  
  
現状  
URLSessionに依存しているので  
テストをするのが非常に難しくなります。  
  
自然な流れとしては  
URLSessionのモック用のProtocolを作成することかもしれません。  
  
しかし  
これを読んでいる方が「モックのためにProtocolを作成する」と聞くと  
思わずあとずさりしてしまうようになることを私は期待しています。  
  
モックの基本的な前提は  
テストで入れ替えたいあるオブジェクトを模倣した  
テスト用のオブジェクトを構築することです。  
  
こうするとProtocolは  
既存のインターフェイスに似たように設計されます。  
  
それに応じてモックオブジェクトも  
実際のオブジェクトに近いものになります。  
  
これは「本物のオブジェクト」とProtocolとモックが  
密接に結びつくことになり  
Protocolが特定の一つの実装に縛られない  
より強力なProtocolを作成する機会をなくしてしまいます。  
  
もしProtocolを使う理由がテストだけの場合  
それはProtocolを使いこなしていない証です。  
  
Protocolにはもっとできることがたくさんあります。  
  
私の目的はURLSessionのモックを作成することではありません。  
必要な機能を汎用的に利用できるようにすることです。  
今やりたいことはURLRequestをDataに非同期的に変換することです。  
  
```swift

// 非同期的にURLRequestをDataへ変換する
protocol Transport {
    func send(request: URLRequest, completion: @escaping (Result<Data, Error>) -> Void)
}
```  
  
これは「ネットワーク上のHTTPServer」とは  
言っていないことに注目してください。  
  
URLRequestを  
Dataに非同期的に変換できるものであれば何でも適合できます。  
  
データベースかもしれませんし  
ユニットテスト用の固定データかもしれません。  
ただのファイルデータである場合もあります。  
  
スキーマによって参照場所が異なることもあります。  
  
ここでRetroactive Modelingを活用して  
URLSessionをTransportに拡張します。  
  
```swift

extension URLSession: Transport {
    func send(request: URLRequest, completion: @escaping (Result<Data, Error>) -> Void)
    {
        let task = self.dataTask(with: request) { (data, _, error) in
            if let error = error { completion(.failure(error)) }
            else if let data = data { completion(.success(data)) }
        }
        task.resume()
    }
}
```  
  
こうすることで  
Transportが必要な場所には  
直接URLSessionを使用することができるようになりました。  
  
URLSessionはFoundationライブラリに含まれますが  
AppleがTransportのことを何も知らなくても機能させることができます。  
  
URLSessionの機能を何も失うこともなしに  
たった数行で機能させることができます。  
  
ここでAPIClientがTransportを使用するように変えます。  
  
```swift

final class APIClient {
    let baseURL = URL(string: "https://www.example.com")!
    let transport: Transport   

    init(transport: Transport = URLSession.shared) { self.transport = transport }

    func fetch<Model: Fetchable>(_ model: Model.Type, id: Int,
                                 completion: @escaping (Result<Model, Error>) -> Void)
    {
        let url = baseURL
            .appendingPathComponent(Model.apiBase)
            .appendingPathComponent("\(id)")
        let urlRequest = URLRequest(url: url)

        // TransportにURLRequestを送る
        transport.send(request: urlRequest) { data in
            let result = Result { try JSONDecoder().decode(Model.self, from: data.get()) }
            completion(result)
        }
    }
}
```  
  
`init`にデフォルト引数を使うことで  
呼び出し側では前と同じように`APIClient()`のシンタックスで  
以前と同じように使うことができます。  
  
Transportはただの「URLSessionのモック」よりもかなり強力になりました。  
  
これはURLRequestからDataへ変換する一つの関数です。  
これは合成することができるということを意味してます。  
Transportに他のTransportをラップして構築することができます。  
  
例えば全リクエストにヘッダーを追加するTransportを構築できます。  
  
  
```swift

// 既存のTransportにヘッダーを追加する
final class AddHeaders: Transport
{
    let base: Transport
    var headers: [String: String]

    init(base: Transport, headers: [String: String]) {
        self.base = base
        self.headers = headers
    }

    func send(request: URLRequest, completion: @escaping (Result<Data, Error>) -> Void)
    {
        var newRequest = request
        for (key, value) in headers { newRequest.addValue(value, forHTTPHeaderField: key) }
        base.send(request: newRequest, completion: completion)
    }
}

let transport = AddHeaders(base: URLSession.shared,
                           headers: ["Authorization": "..."])
```  
  
これで全リクエスト一つ一つにヘッダーを加えるのではなく  
一つのTransportに追加処理を集約することができました。  
  
認証トークン変わった場合も一箇所の修正で完了し  
将来の全てのリクエストに正しいヘッダーを追加することができます。  
  
ユニットテストもしやすいままです。  
AddHeadersよりも下の層のTransportを入れ替えることが可能だからです。  
  
これは既存のシステムを柔軟に拡張できることを意味しています。  
暗号化、ログ、キャッシュ、自動リトライなどを  
実際のネットワーク通信の層と混ぜることなく追加することができます。  
  
ユニットテストの能力を失うことなしに  
VPNプロトコルの上にネットワーク層を置くこともできます。  
  
モックも作成でき  
ユニットテストもでき  
さらなる機能の追加も簡単にできるようになりました。  
  
補足としてTransportの「モック」を提示しますが  
このProtocolでできることとして最も興味のないものでしょう。  
  
```swift

// テスト用に固定値を返すTransport
enum TestTransportError: Swift.Error { case tooManyRequests }

final class TestTransport: Transport {
    var history: [URLRequest] = []
    var responseData: [Data]

    init(responseData: [Data]) { self.responseData = responseData }

    func send(request: URLRequest, completion: @escaping (Result<Data, Error>) -> Void) {
        history.append(request)
        if !responseData.isEmpty {
            completion(.success(responseData.removeFirst()))
        } else {
            completion(.failure(TestTransportError.tooManyRequests))
        }
    }
}
```  
  
## Common currency(共通通念)  
  
ジェネリックな`APIClient.fetch`と  
ジェネリックでない`Transport.send`を分割することは  
私が探していた共通構造です。  
  
`Transport.send`は  
具体的な型で構成された  
小さいセット(入力がURLRequestで出力がData)を扱います。  
  
このような具体的な型で構成された小さいセットを扱っている時は  
そのセット同士を合成することは簡単にできます。  
  
URLRequestを作成できるもの  
もしくは  
Dataを受け取って処理ができるもの  
ならばどんなものでも合成することができます。  
  
`APIClient.fetch`は  
DataをジェネリックなFetchable型に変換します。  
  
ジェネリックやassociated typeが出てくると  
より表現が豊かなコードを書くことができますが  
必要な型を全て準備しなければいけなくなるため  
合成して使用することはむずかしくなります。  
  
インターネットのすごいところは  
ほとんどのケースでたった一つの型  
パケットのみで動いていることです。  
  
パケットの中身が何であるか  
何を意味しているのかは全く関係ありません。  
  
パケットをある場所から別の場所に移動させているだけです。  
入力がパケットで出力もパケットです。  
  
だからこそインターネットは非常に柔軟であり  
膨大な数のベンダーが多様な方法で実装をしていたとしても  
一緒に動作することができているのです。  
  
ネットワークレイヤーより上の各レイヤーで  
追加の文脈や意味合いを情報に適用させます。  
  
それは  
  
- ユーザの情報  
- 実行コマンド  
- 画面に表示する動画  
  
など多様に解釈されます。  
  
これがコンポジション(合成)です。  
それぞれがそれぞれの関心を持つ独立したレイヤー同士をつなぎ合わせます。  
  
私はProtocolを設計する時に  
これと同じ手法を採用するように努めています。  
  
特に一番下のレイヤーでは共通で利用可能な具体的な型を探します。  
  
- URLとURLRequest  
- DataとInt  
- シンプルな関数型`() -> Void`など。  
  
さらにスタックの上の層に行くに応じて  
モデルやそれに近い形で  
より大きな意味合いをデータに適用していきます。  
  
つまり  
Transportを書くのは簡単で  
他のものはTransportを簡単に使うことができます。  
これが目標です。  
  
現在の実装は  
まだ期待しているほど十分に柔軟で強力ではありませんが  
合成可能でテスト可能な方法で  
特定のAPIから広い種類のモデルを取得することができます。  
  
他のシンプルなAPIにならばすぐに適応可能で  
現在の目的を果たすためにはこれ以上柔軟にする必要はありません。  
  
  
# それは数字ではない  
  
現状のUserとDocumentを見てみます。  
  
```swift

struct User: Codable, Hashable {
    let id: Int
    let name: String
}

struct Document: Codable, Hashable {
    let id: Int
    let title: String
}
```  
  
ここでAPIに変更が入り  
DocumentのidがIntからStringに変わったとします。  
(これは現実にあった話です。)  
  
実際はIDはIntではありません。  
ましてや数字でもありません。  
  
２つを足すのか？もしくは割るのか？  
ほとんどの数字のような操作が意味のないものならば  
どうやってIDを数字のような振る舞わせるのか？  
  
現在の設計だとUserのIDにDocumentのIDに設定することもできてしまいます。  
ランダムな数字であっても可能です。  
  
これは正しくありません。  
IDはそれぞれのクラス自身のものです。  
それぞれにIDの型が必要です。  
  
まずは具体的なUserから考えて  
そこから汎用化できるかどうかを見ていきます。  
  
```swift

struct User: Codable, Hashable {
    struct ID: Codable, Hashable { 
        let value: Int 
    }
    let id: ID
    let name: String
}
```  
  
これでUserインスタンスを下記のように作成できます。  
  
```swift

let user = User(id: User.ID(value: 1), name: "Alice")
```  
  
これでOKですが  
`value:`という引数ラベルがあまり好きではありません。  
これはAPIデザインガイドラインにも違反しています。  
  
> In initializers that perform value preserving type conversions, omit the first argument label,   
e.g. Int64(someUInt32).  
  
これを補完するために  
引数なしのイニシャライザを追加します。  
  
```swift

struct User: Codable, Hashable {
    struct ID: Codable, Hashable { 
        let value: Int 
        init(_ value: Int) { self.value = value }
    }
    let id: ID
    let name: String
}

let user = User(id: User.ID(1), name: "Alice")
```  
  
より良くなりました。  
  
Documentにも同じようにします。  
  
```swift

struct Document: Codable, Hashable {
    struct ID: Codable, Hashable {
        let value: String
        init(_ value: String) { self.value = value }
    }
    let id: ID
    let title: String
}
```  
  
大して長いコードではありませんが  
毎回コピペしたくなるようなコードです。  
こういう時は汎用的なコードが隠れているのではないかと考えるタイミングです。  
結局このシステムのほとんどのモデルはIDを持つことになるでしょう。  
  
## たぶんProtocol？  
  
コードの重複を見かけた時  
私は汎用的な解決方法を抽出するために  
最初にProtocolに辿りつくことが多いです。  
  
Protocolはそういうことをするのが得意です。  
  
今回のIDの場合は２つの重複した概念があります。  
  
- CodableとHashableに適合したIdentifier  
- 引数なしイニシャライザ  
  
キーストロークではなく概念に注目することは大切なことです。  
DRY(Don't repeat yourself)は  
「同じ文字を２回打つな」  
という意味ではありません。  
  
ポイントは  
**一緒に変化するもの**  
を抽出することです。  
  
「`：Codable, Hashable` 'init(_...)'という文字列」  
  
を抽出したいのではなく  
  
「識別子として振舞うもの」  
  
を抽出したいのです。  
そこでIdentifierとして上記の概念を抽出します。  
  
```swift

protocol Identifier: Codable, Hashable {
    associatedtype Value: Codable, Hashable
    var value: Value { get }
    init(value: Value)
}

extension Identifier {
    init(_ value: Value) { self.init(value: value) }
}
```  
  
これを使うことでUser.IDはよりシンプルになります。  
  
```swift

struct User: Codable, Hashable {
    struct ID: Identifier { let value: Int }
    let id: ID
    let name: String
}
```  
  
これを使うには`APIClient.fetch`ではID型を受け取る必要があります。  
  
```swift

func fetch<Model: Fetchable>(_ model: Model.Type, id: Model.ID,
                             completion: @escaping (Result<Model, Error>) -> Void)
```  
  
当然FetchableにもID型が必要になります。  
  
```swift

protocol Fetchable: Decodable {
    associatedtype ID: Identifier
    static var apiBase: String { get }
}
```  
  
※※待ってください。**  
  
最後の変更は決して「当然」では済ませられません。  
  
FetchableはシンプルなProtocolとして使われていましたが  
今は**PAT(protocol with associated type)**になっています。  
  
これは大きな変化です。  
`associatedtype`と書いた時はいつでも  
一度立ち止まって考えてみてください。  
  
**「これを配列に入れたいか？」**  
  
現在のSwiftではProtocolに`associatedtype`または`Self`を使用すると  
Protocolはもはや「具体的なもの」ではなくなります。  
extensionやジェネリック関数の制約でしかなくなります。  
  
変数や関数の引数、その他値として扱われる全てのことには使用できなくなります。  
  
## 次のコードは何？  
  
Identifierに戻って先ほどの質問をしてみます。  
  
**「Identifierは配列として使用するか？」**  
  
これまでIdentiferをプロダクションでも使用してきましたが  
配列で使用することはありませんでした。  
なんとか例を提示しようとしてみましたが  
うまくいくようなケースが作成できませんでした。  
しかし  
その過程をみていくのは価値があると思います。  
  
もしIdentifierの配列を作成しようとすると  
下記のようなエラーが発生します。  
  
```swift

let ids: [Identifier] = [User.ID(1), Document.ID("password")]
// Protocol 'Identifier' can only be used as a generic constraint because it has Self or associated type requirements
```  
  
では仮に  
将来的にgeneralized existentialが実装されたり  
AnyIdentifierなどのType Eraserを作成して  
配列に格納することが可能になったとします。  
  
```swift

for id in ids {
    // .valueというAny型のプロパティにアクセスができるだけで何をする？
}
```  
  
idに対してできることはvalueプロパティを取得することだけです。  
  
しかし  
それぞれのIDは異なる型になるのでAnyとして扱わなければならなくなると思います。  
これだと`fetch`メソッドを呼ぶこともできません。  
  
IDがどのModelのものかさえもわかりません。  
  
「IDがどのModelのものかさえもわからない。」  
  
今回配列に入れてみることを試みた時に  
Identifierの不自然な点を見つけてしまいました。  
  
Identifierは  
それが何の識別子を表しているものなのかを知らないです。  
  
これは間違いなのでしょうか？  
  
私はそうは思いません。  
  
これまで私が抱えていた問題を解決していますし  
逆に問題になっていることもありません。  
  
それが今の問題を解決できているのであれば  
それはそれで良いのです。  
  
ただし改善はできます。  
  
  
## 全ては関数  
  
Identifierを改善する前に  
Identifierを現状の状態のままで  
今の問題を解決できるような方法を考えてみます。  
  
というのも  
もしかしたらあなたが扱っている型は  
容易に変更できないようなものの場合もあるかもしれません。  
  
そんな時に役に立つ道具を持っておくことは良いことだと思います。  
  
それでは  
Identifierを配列として使用したいけれども  
できないケースを考えていきます。  
  
例えば  
１時間に１回モデルを再取得してリフレッシュしたいとします。  
そこでリフレッシュしたいIdentifierを  
配列で保持しようとしますができません。  
  
もう一度本当に必要なことを考えてみます。  
  
必要なのは  
  
「Identifierの配列」  
  
ではなく  
  
「Identifierに紐づいたモデルを取得するためのリフレッシュリクエスト」  
  
を必要としているのです。  
  
リフレッシュリクエストは未来のアクションです。  
未来のアクションはクロージャで表現できます。  
  
私はいつもクロージャを型にラップするようにしています。  
  
  
```swift

struct RefreshRequest {
    // 遅延実行する
    let perform: () -> Void

    init<Model: Fetchable>(id: Model.ID,
                           with client: APIClient = APIClient.shared,
                           updateHandler: @escaping (Model) -> Void,    // On success
                           errorHandler: @escaping (Model.ID, Error) -> Void = logError) // On failure, with a default
    {
        //updateHandlerとerrorHandlerを() -> Voidの中にまとめて入れてしまう
        perform = {
            client.fetch(Model.self, id: id) {
                switch $0 {
                case .success(let model): updateHandler(model)
                case .failure(let error): errorHandler(id, error)
                }
            }
        }
    }

    // errorHandlerのデフォルト値としてのヘルパー関数
    static func logError<ID: Identifier>(id: ID, error: Error) {
        print("Failure fetching \(id): \(error)")
    }
}

let requests = [
    RefreshRequest(id: userID, updateHandler: { users[$0.id] = $0 }),
    RefreshRequest(id: documentID, updateHandler: { documents[$0.id] = $0 }),
]
```  
  
ここでのポイントは特定のデータ構造ではありません。  
  
`() -> Void`に全てを格納していることです。  
`() -> Void`はとてつもない強力で柔軟性のある型です。  
この型は他のあらゆる関数から構築することができます。  
  
これは「共通通念」の別のケースです。  
  
もし遅延実行したいアクションが必要な時は  
それはただの関数です。  
  
多くの複雑なジェネリックなコードは  
あとで実行しようとしているジェネリックな関数に渡す  
全てのパラメータの値を追跡しようとしていることから引き起こされます。  
  
もし必要なものが関数そのものだけだったとしたら  
上記のようなものは一切必要がありません。  
  
これは**type-eraser**に注目しているのではなく  
**type-erasure**のコアです。  
  
型を隠しているためModelのような型を気にする必要がありません。  
  
上記の例ですとModelに依存している`updateHandler`と`errorHandler`は  
`() -> Void`という  
  
- 唯一の  
- 何の型にも依存しない  
- ジェネリックでない  
  
クロージャの中に組み込まれます。  
  
これは広く使われているテクニックです。  
  
他にも改善できる箇所もありますが  
Identifier自体の改善に話を戻したいと思います。  
  
## 本当のIdentifier  
  
本当に欲しいIdentifierは  
  
- ModelがIDの型を知っていて  
- IDもModelの型を知っている  
  
という状態です。  
  
APIClient.fetch`は型とIdentifierの両方を受け取っています。  
  
```swift

func fetch<Model: Fetchable>(_ model: Model.Type, id: Model.ID,
                             completion: @escaping (Result<Model, Error>) -> Void)
```  
  
これはひどい重複が発生しています。  
  
```swift

client.fetch(User.self, id: User.ID(1), completion: { print($0)} )
```  
  
Identifierにassociated typeとしてModelを追加することもできますが  
ちょっと汚くなってしまいます。  
`where`などのSwift独自のシンタックスが増えてくると  
とても良いProtocolとは言えなくなってきます。  
  
実装を見てみましょう  
  
```swift

struct User.ID: Identifier { let value: Int }
struct Document.ID: Identifier { let value: String }
```  
  
他の実装について考えてみても  
valueというプロパティを一つ持ったstructとして  
ほとんど同じになるでしょう。  
  
他の実装方法も思いつきません。  
  
もし全てのProtocolのインスタンスが全く同じ方法で適合する場合は  
おそらくジェネリックなstructであるべきだと思います。  
  
```swift

// ある特定のModel型に適用するある特定のValue型の識別子
struct Identifier<Model, Value> where Value: Codable & Hashable {
    let value: Value
    init(_ value: Value) { self.value = value }
}

extension Identifier: Codable, Hashable {
    init(from decoder: Decoder) throws {
        self.init(try decoder.singleValueContainer().decode(Value.self))
    }

    func encode(to encoder: Encoder) throws {
        var container = encoder.singleValueContainer()
        try container.encode(value)
    }
}
```  
  
Identifierは２つの型パラメータを持ちます。  
ModelはIdentifierに適用する型で  
Valueは必須のIdentifierの型になります。  
  
Modelはどこでも使われていませんが  
`Identifier<User, Int>`と  
`Identifier<Document, Int>`は異なる型になります。  
  
Userは  
  
```swift

struct User: Codable, Hashable {
    let id: Identifier<User, Int>
    let name: String
}
```  
  
これでOKですが  
typealiasを使って  
User.IDとしてアクセスできた方がより良いでしょう。  
  
```swift

struct User: Codable, Hashable {
    typealias ID = Identifier<User, Int>
    let id: ID
    let name: String
}
```  
  
さらにこれをProtocolに抽出して  
Fetchableに適合させるともう少し良くなります。  
  
```swift

// Identifierで識別される何か
protocol Identified: Codable {
    associatedtype IDType: Codable & Hashable
    typealias ID = Identifier<Self, IDType>
    var id: ID { get }
}

// IDを使ってAPIでデータを取得できる何か
protocol Fetchable: Identified {
    static var apiBase: String { get }
}

struct User: Identified {
    typealias IDType = Int
    let id: ID
    let name: String
}

extension User: Fetchable {
    static var apiBase: String { return "user" }
}

```  
  
最終的にfetchは型引数がなくなりました。  
User.IDから取得できるのはUserのみです。  
  
```swift

func fetch<Model: Fetchable>(_ id: Model.ID,
                             completion: @escaping (Result<Model, Error>) -> Void)

client.fetch(User.ID(1), completion: { print($0)} )
```  
  
# 型さえも必要ない  
  
fetchメソッドに戻って考えてみます。  
  
IDでモデルを取得できることは素晴らしいことですが  
もう一つやりたいことがあります。  
/keepaliveのようなPOSTリクエストを実装して  
エラーが生じた場合はエラーを返したいです。  
  
fetchとkeepAliveはとても似ていますが  
少し異なります。  
  
```swift

func fetch<Model: Fetchable>(
    id: Model.ID,
    completion: @escaping (Result<Model, Error>) -> Void) {
    
    let request = URLRequest(url: URL(string: Model.apiBase)!
        .appendingPathComponent("\(id)"))
    
    transport.send(request: request) { result in
        switch result {
        case .success(let data):
            completion(Result {
                try JSONDecoder().decode(Model.self, from: data)
            })
        case .failure(let error):
            completion(.failure(error))
        }
    }
}

func keepAlive(
    completion: @escaping (Error?) -> Void) {
    
    var urlRequest = URLRequest(url: baseURL
        .appendingPathComponent("keepalive")
    )
    urlRequest.httpMethod = "POST"
    
    transport.send(request: urlRequest) {
        switch $0 {
        case .success: completion(nil)
        case .failure(let error):
            completion(error)
        }
    }
}
```  
  
どちらも  
  
1. URLRequestを構築する  
2. transportに渡す  
3. Resultを処理する  
  
というのが基本パターンです。  
  
実際に重複しているのはたったの1行ですが  
構造的にはとても似ています。  
  
これは切り出したくなります。  
fetchは多くのことをやりすぎています。  
  
おそらく変化する部分を切り出してRequestと呼ぶことになるでしょう。  
  
では  
Requestはどういう形になるでしょうか？  
  
結構な頻度で  
下記のような実装にいきなり飛躍する人を見かけます。  
  
```swift

protocol Request {
    var id: Int { get }
    associatedtype Response
    var completion: (Response) -> Void { get }
}
```  
  
つまりPATです。  
  
PATが出てきた時はまず質問してみましょう。  
  
このProtocolを配列として必要だろうか？  
  
今回のケースの場合  
明らかに配列に格納したいと考えるでしょう。  
  
- 送信待ちのリクエストのリスト  
- 順番に送信するリクエストのリスト  
- リトライが必要なリクエストのリスト  
  
だれかがこう言うかもしれません。  
「将来的にGeneralized Existentialsが実装されたら使うことができる」  
  
しかし  
これは問題の解決になりません。  
  
transportのfetchメソッドはDataを生成しますが  
Responseは生成しません。  
  
この違いを埋めるための言語仕様はありません。  
これは設計の問題です。  
  
ではどうすればよいでしょうか？  
  
何度も言いますが  
  
**「具体的なコードから書きはじめよ。そして汎用を見つけ出せ。」**  
  
では具体的なRequestとはどういうものでしょうか？  
  
```swift

struct Request {
    let urlRequest: URLRequest
    let completion: (Result<Data, Error>) -> Void
}
```  
  
ただのstructです。  
  
しかし  
JSONパーサーはどこでしょうか？  
どうにかしなければいけません。  
  
そこでfetchメソッドを呼び出す側のコードから  
何が必要なのかを考えてみたいと思います。  
  
```swift

let baseURL = URL(string: "https://www.example.org")!

extension Request {

    // GET /<model>/<id> -> Model
    static func fetching<Model: Fetchable>(
        id: Model.ID,
        completion: @escaping (Result<Model, Error>) -> Void)
        -> Request {
        ???
    }
}

```  
  
欲しいものはmodelを取得するRequestです。  
Requestのメソッドにしています。  
  
なぜイニシャライザではなくstaticなメソッドなのでしょうか？  
  
このような問題対してはこういう形の方がスケールしやすいのです。  
  
同じ引数を取るものの  
ちょっと異なっているようなRequestがある場合は  
扱いづらいです。  
  
そこをstaticメソッドがその違いを吸収してくれます。  
  
```swift

struct Request {
    let completion: (Result<Data, Error>) -> Void
}

extension Request {
    static func fetching<Model: Fetchable>(
        completion: @escaping (Result<Model, Error>) -> Void)
    }
}
```  
  
２つの異なる型を持ったクロージャがあります。  
一つをModelを受け取ってVoidを返します。  
もう一つはDataを受け取ってVoidを返します。  
  
つまり欲しいものは  
その２つの間のDataからModelへ変換するメソッドが必要になります。  
  
```

Data -> Void
    ↑ Data -> Model
Model -> Void
```  
  
これはどうやって機能するでしょうか？  
  
これらを組み合わせると  
Data->Model->Voidと型を取得していきます。  
  
これは最終的には  
DataからVoidを取得することと同じになります。  
  
```
Data -> Void
Data -> Model -> Void
     ↓
Data -> Void
```  
  
今何をしたのでしょうか？  
  
**type erasure**です。  
  
関数を合成することで中間の型を消去しました。  
  
余分な型をなくしたいと思ったときは  
**type eraser**ではなく関数について考えて欲しいです。  
  
たいてい必要になるのは**type erasure**になるのです。  
  
では処理はどうなるでしょうか？  
  
APIClientのfetchで行っていることを  
Requestのfetchingに移動します。  
  
```swift

extension Request {
    static func fetching<Model: Fetchable>(
        id: Model.ID,
        completion: @escaping (Result<Model, Error>) -> Void) -> Request {

        let urlRequest = URLRequest(url: baseURL
            .appendingPathComponent(Model.apiBase)
            .appendingPathComponent("\(id)")
        )

        return self.init(urlRequest: urlRequest) {
            data in
            completion(Result {
                let decoder = JSONDecoder()
                return try decoder.decode(
                    Model.self,
                    from: data.get())
            })
        }
    }
}

extension Client {
    func fetch(request: Request) {
        transport.fetch(request: request.urlRequest,
                        completion: request.completion)
    }
}
```  
  
単なるstructを活用することで  
Protocolのような柔軟性を持ちつつ  
PATのような頭痛の種もありません。  
  
同じようにPOSTメソッドの/keepaliveにも同じパターンを当ててみます。  
  
Error？を受け取りVoidに変換する  
クロージャを引数として受け取ります。  
  
  
```swift

extension Request {

    // POST /keepalive -> Error?
    static func keepAlive(
        completion: @escaping (Error?) -> Void) -> Request {
        var urlRequest = URLRequest(url: baseURL
            .appendingPathComponent("keepalive")
        )
        urlRequest.httpMethod = "POST"

        return self.init(urlRequest: urlRequest) {
            switch $0 {
            case .success: completion(nil)
            case .failure(let error):
                completion(error)
            }
        }
    }
}
```  
  
汎用的なコードは  
たくさんのジェネリクスやProtocolやassociated typeがあることではありません。  
  
汎用的なコードは具体的なコードを書くことから始まり  
共通の部分と変化する部分を切り分けます。  
  
それはProtocolであったり  
他の関数を変換するただの関数  
  
であることもあります。  
  
  
## 最後に  
  
汎用的なコードは難しいです。トレードオフがたくさんあります。  
いくつかはSwiftがまだ進化中だからということもありますし  
汎用的なコードがただ単純に難しいということもあります。  
  
問題を見つけた時  
解決策をシンプルに具体的に構築し抽出していくこと  
  
自分自身で「問題の発明」をしてはいけません。  
「まだ十分に汎用的ではない」ことは問題ではありません。  
あなたが本当に抱えている問題を汎用的なコードが問題を解決しているのかを確認し  
できる限りそれを解決済みの状態にし続けるようにしましょう。  
どこかのタイミングで再設計することになるでしょう。  
  
# Type Erasers  
  
Protocolを使用していると  
どこかでType Eraserにぶつかることになるでしょう。  
  
私は過去の記事にType Eraserについての記事を書いたりもしましたが  
たいていの場合はType Eraserはおすすめしないとここで言います。  
  
Type Eraserを使うことで設計を再考したり  
変換関数や具体的なstructにするだけで避けられるように頭痛の種を増やすことになる。  
  
これまでのところType Eraserを避けることについて書いてきました。  
これは最終手段であって膨大な複雑さを増やしてしまいます。  
  
もし頻繁にType Eraserを使用していると思ったら  
それはProtocolを過剰に使用しているのでしょう。  
  
しかし  
ときどきType Eraserが必要なときもあります。  
なのでType Eraserについて見ていきましょう。  
  
```swift

public struct AnySequence<Element> : Sequence { ... }
let s = AnySequence([1,2,3])
```  
  
これは「Any」な型です。  
Sequenceのような振る舞いをすれば型については何も気にしません。  
これは文字通り「Sequenceプロトコルに適合した型なんでも」ということです。  
  
「eraser」は水面下の型を隠すための「小さい箱」を作ります。  
「小さい箱」と思って貰えばわかりやすいと思います。  
  
つまりtype eraseは  
ただ明示的に手動でexistentialを作っているに過ぎないのです。  
だからassociated typeと一緒に使われるのを見かけるのが多いのです。  
  
PATではない単純なProtocolは  
Swiftが自動でtype eraserとexistentialを作ります。  
  
将来的にgeneralized existentialが導入されると  
PATに対しても自動でtype eraserを作るようになります。  
  
しかし何度も言っているように  
generalized existentialやtype eraserは  
問題の解決にはなりません。  
  
こういう警告をした上で  
type eraserがまさに適切なツールであることがあります。  
  
Swiftでtype eraserを実装する２つの主要な方法があります。  
  
### Functional Type-Erasers  
  
１つ目を**Functional Type-Erasers**と私は呼びます。  
これは値を関数のコレクションへと変換することから由来しています。  
特にクロージャのコレクションを保持するstructを作成することが多くあります。  
  
```swift

protocol Frobulating {
 associatedtype Input
 associatedtype Output
 func frobulate(from input: Input) -> Output
}
```  
  
まずこのProtocolから始めます。  
これはただの１つのメソッドを持ちます。  
つまりRequestの時と同じように  
ジェネリックなstructに単純に置き換えることを真剣に考えるべきです。  
またはただ関数を使うことだけで良いかもしれません。  
Protocolはもちろんstructでさえも必要ないかもしれません。  
  
```swift

struct Frobulator<Input, Output> {
 let frobulate: (Input) -> Output
}

(Input) -> Output
```  
  
今回の例では  
他にもたくさんのメソッドがあったり  
追加のコンテキストか何かが存在することを想定します。  
  
しかし  
Protocolをジェネリックなstructやただの関数に置き換えることが  
Functional Type-Erasersのポイントです。  
  
下記に使い方を示します。  
  
```swift

protocol Frobulating {
    associatedtype Input
    associatedtype Output
    func frobulate(from input: Input) -> Output
}

struct AnyFrobulator<Input, Output>: Frobulating {
    private let _frobulate: (Input) -> Output
    func frobulate(from input: Input) -> Output {
        _frobulate(input)
    }
    init<F: Frobulating>(_ base: F)
        where F.Input == Input, F.Output == Output {
            self._frobulate = base.frobulate
    }
}
```  
assoiciated typeはジェネリックなパラメータになり  
メソッドはクロージャになります。  
  
残念なことにSwiftのクロージャプロパティは  
Protocolに適合させることができないため  
_frobulateのような形になっています。  
  
initにFrobulatingに適合するものなら何でも受け取ることができ  
クロージャプロパティにメソッドの参照を保持します。  
_frobulateとbase.frobulateは同じ参照を持つようになり  
クロージャの中にbaseへの参照を保持します。  
  
以上です。  
これがtype eraserの全体像です。  
面倒ですが難しくはありません。  
  
しかし  
お気づきかもしれませんが  
type eraserからbaseへと戻ることはできません。  
baseへの参照はクロージャで保持しているだけです。  
  
ここで第２のtype eraserが登場します。  
  
### Boxing Type Erasers  
  
私は**Boxing Type Erasers**と呼びます。  
これは内部でprivateな箱を含んでいることから由来します。  
これは元の値へ戻るためにプロパティとして保存しておくためです。  
  
```swift

struct AnyFrobulator<Input, Output>: Frobulating {
    private let _frobulate: (Input) -> Output
    func frobulate(from input: Input) -> Output {
        _frobulate(input)
    }
    init<F: Frobulating>(_ base: F)
        where F.Input == Input, F.Output == Output {
            self._frobulate = base.frobulate
            self.base = base
    }
    let base: Any
}
let f = AnyFrobulator(SomeFrobulator())
let s = f.base as? SomeFrobulator
```  
  
上記のように元の値へ戻ることができます。  
  
しかしちょっと変に見えます。  
というのもbaseを保持しているのにメソッドもプロパティに設定して  
baseへの参照を保持するようにしています。  
  
baseプロパティから呼び出したいと思うかもしれませんができません。  
baseはAnyだからです。  
  
Anyは最も強力なtype eraserです。全てを消去します。  
そのためbaseのメソッドを呼ぶためには  
baseへの参照をクロージャに保持しておく必要があります。  
  
変数はキャストすることができません。  
ちょっとイライラするかもしれませんが  
そんなに大きな問題にはなりません。  
  
しかしもう一つ問題があります。  
  
このテクニックでは  
複数のクロージャで同じ値への参照を保持することが前提となっています。  
  
これが使えるのはProtocolのメソッドが不変である場合のみです。  
  
もしProtocolのメソッドにmutatingがついていて  
Value typeの値を渡したとしたら  
それぞれのクロージャはコピーを保持するようになり  
メソッドを使って自身の変更をしても機能しません。  
  
つまり値を１つだけ保持する方法が必要になります。  
  
```swift

private class _Box<F: Frobulating>
where F.Input == Input, F.Output == Output {
    let _base: F
    init(_ base: F) { self._base = base }
    var base: Any { _base }
    func frobulate(from input: Input) -> Output {
        _base.frobulate(from: input)
    }
}
struct AnyFrobulator<Input, Output>: Frobulating {
    ...
    private var _box: _Box<????>
    ...
}
```  
  
こうすることで可能になります。  
元々の型へのジェネリックなboxを作成して  
元のオブジェクトへをプロパティとして保持しておきます。  
メソッドが呼ばれると元の値のメソッドを呼び出すようにします。  
  
しかし新しい問題がここで発生します。  
  
上記のboxを保持するためのプロパティが必要ですが  
そのためには元の型を追い続ける必要があります。  
  
元々この型を隠すように色々と試みています。  
  
そこでboxが保持している型を知ることなしに  
boxとやり取りをする方法が必要になります。  
  
そのためにとても美しくないトリックを使う必要が出てきます。  
  
```swift

private class _BoxBase<Input, Output>: Frobulating {
    var base: Any { fatalError() }
    func frobulate(from: Input) -> Output {
        fatalError()
    }
}
```  
  
非常にややこしいのでこれについて２回見ていきます。  
これは後々取得する型を隠すためによく使われているテクニックです。  
  
具体的な型ではなく  
publicな情報のみに対してジェネリックな抽象クラスを作成します。  
これは入力値(input)と出力値(output)です。  
  
Swiftは抽象クラスがありません。  
そのため美しくありませんが  
全ての箇所でfataErrorを呼ばなければなりません。  
  
次に上記の抽象クラスを継承した  
消そうとしている型に対するジェネリックなサブクラスを作成します。  
  
```swift

private class _Box<F: Frobulating>: _BoxBase<Input, Output> {
    ...
}
```  
  
次に抽象クラス型のプロパティを作成します。  
  
  
```swift

private var _box: _BoxBase<Input, Output>
```  
  
そして抽象クラスではなく  
そのサブクラスの型のインスタンスを割り当てます。  
  
```swift

self._box = _Box(base)
```  
  
ここで一度立ち止まって  
実際の実装の中でどうなるのか見てみます。  
  
```swift

struct AnyFrobulator<Input, Output>: Frobulating {
    // A base class that erases the concrete Frobulating type
    private class _BoxBase<Input, Output>: Frobulating {
        var base: Any { fatalError() }
        func frobulate(from: Input) -> Output { fatalError() }
    }

    ...
}
```  
  
publicなInputとOutputという情報に対してのみジェネリックにした  
_BoxBaseと呼ばれる抽象クラスがあります。  
Protocolに適合しています。  
  
baseへのアクセスをAnyとして提供します。  
つまり特定の型の情報は持っていないことを覚えておいてください。  
  
```swift

struct AnyFrobulator<Input, Output>: Frobulating {

    // 具体的なFrobulating型を消すためのbaseクラス
    private class _BoxBase<Input, Output>: Frobulating {
        var base: Any { fatalError() }
        func frobulate(from: Input) -> Output { fatalError() }
    }

    // 具体的なFrobulating型を知っているサブクラス
    private class _Box<F: Frobulating>: _BoxBase<Input, Output>
    where F.Input == Input, F.Output == Output {
        init(_ base: F) { self._base = base }
        override var base: Any { _base }
        override func frobulate(from input: Input) -> Output {
            _base.frobulate(from: input)
        }
        let _base: F
    }
    ...
}
```  
  
次にBoxと呼ばれる具体的なサブクラスがあります。  
これは特定の具体的な型に対するジェネリックです。  
  
プロパティとして元のオブジェクトを保持し  
すべてを元のオブジェクトを呼び出するようにします。  
  
```swift

struct AnyFrobulator<Input, Output>: Frobulating {
    ...
    // 本物のBox
    private var _box: _BoxBase<Input, Output>
    var base: Any { _box.base }
    init<F: Frobulating>(_ base: F)
        where F.Input == Input, F.Output == Output {
            self._box = _Box(base)
    }
    func frobulate(from input: Input) -> Output {
        _box.frobulate(from: input)
    }
}
```  
  
抽象クラス型のプロパティがあり  
initで具体的なサブクラスを設定します。  
  
以上です。  
  
これは本当に美しくありません。  
特にSwiftが持っていない抽象クラスに依存していることです。  
  
そしてとてつもない数のボイラープレートを必要とします。  
  
しかし  
この方法はAnySequenceのように  
Standard Libraryで使われている  
type eraserの方法にとても似ています。  
  
一つ一つの詳細は微妙に異なり  
AnyHashableはそれ自身はジェネリックではないため  
２つ以上のclassとProtocolとstructを使っていますが  
基本的にはこの二重のボックス化手法で構築されています。  
  
  
  
ここまでが翻訳(意訳)です。  
---   
  
# 最後に  
  
具体的な実装から始まり  
どうやって汎用的なコードにしていくのかを  
ステップを踏んで見ていきました。  
  
筆者がこの内容の発表している時に言っていましたが  
  
この一連の内容で一番重要なことは  
  
**Write concrete code first. Then work out the generics.**  
**具体的なコードから書きはじめよ。そして汎用を見つけ出せ。**  
  
ジェネリックなコードはきれいで便利です。  
  
しかし  
今抱えている問題から離れて汎用化を進めることで  
むしろ扱いが難しくなってしまうことがしばしばあります。  
  
まずは目の前の問題から具体的に考えること。  
当たり前かもしれませんが  
忘れてはいけない大切なことを教えていただけたと私は思っています😃  
