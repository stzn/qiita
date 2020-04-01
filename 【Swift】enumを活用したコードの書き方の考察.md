Swiftでは言語機能や標準ライブラリを使って様々な方法でコードを書くことができます。  
  
中でもenumを活用できる場面はたくさんあると思っています。  
一方でenumを使うことでちょっと扱いづらくなる部分も出てきます。  
  
この記事では  
enumが活用できるところを考え  
ちょっと扱いづらいかなと感じるところを見ていき  
最後にそれを軽減する方法を検討してみたいと思います。  
  
過去の記事でもちょっとだけenumについて書かせて頂きました。  
https://github.com/stzn/qiita/blob/master/Swiftにおけるコードの書き方や表現方法の考察.md  
  
  
# enumのメリット  
  
## 文字列よりも安全にアクセスできる  
  
例えばURLRequestでHTTPメソッドを指定する場合は文字列を渡す必要があります。  
  
```swift

var request = URLRequest(url: url)
request.httpMethod = "PUT"
```  
  
しかし  
スペルミスをしてしまった場合  
  
```swift

var request = URLRequest(url: url)
request.httpMethod = "POT"
```  
  
"POT"は存在しないためエラーになります。  
そしてそれは**実行時エラー**になって気がつきます。  
  
さらに  
変更が必要になった場合は全ての箇所に修正が入ります。  
  
Xcodeで一括置換できますが  
もしかしたら抜け漏れがあるかもしれません。  
  
こういったリスクはできる限り減らしていきたいため  
enumで置き換えてみます。  
  
```swift

enum HTTPMethod: String {
    case get = "GET"
    case post = "POST"
    case put = "PUT"
    case delete = "DELETE"
}

var request = URLRequest(url: url)
request.httpMethod = HTTPMethod.put.rawValue

```  
  
こうすることで  
間違えるリスク可能性や修正が必要な箇所を  
１箇所に集約することができます。  
  
## 複数の型をまとめて扱うことができる  
  
時として複数の型を一律に扱いたい場合があります。  
  
例えば複数の動物がいるとします。  
  
```Swift

struct Dog {
    let name: String
    func bark() {
        print("ワン")
    }
}

struct Cat {
    let name: String
    func bark() {
        print("ニャー")
    }
}
```  
  
これを一律に配列で扱いたいとします。  
  
```Swift

// Heterogeneous collection literal could only be inferred to '[Any]'; add explicit type annotation if this is intentional
let animals = [Dog(name: "Pochi"), Cat(name: "Tama")]
```  
  
これはエラーになります。  
Swiftでは異なる型を同じ配列に格納することができません。  
エラーを解消するためにはメッセージに書いてあるように[Any]を明示します。  
  
```Swift

let animals: [Any] = [Dog(name: "Pochi"), Cat(name: "Tama")]
```  
  
しかし  
こうすると型の情報が失われてしまうため  
各型のメソッドは呼ぶことができなくなります。  
  
```swift

// これはできる
let dogs = [Dog(name: "taro"), Dog(name: "jiro")]
dogs.forEach { $0.bark() }

// Value of type 'Any' has no member 'bark'
let animals: [Any] = [Dog(name: "Pochi"), Cat(name: "Tama")]
animals.forEach { $0.bark() }
```  
  
こういった場合は型判定が必要になります。  
  
```swift

animals.forEach {
    switch $0 {
    case let dog as Dog:
        dog.bark()
    case let cat as Cat:
        cat.bark()
    default: fatalError("No way!!")
    }
}
```  
  
この場合  
型のチェックがコンパイル時に働きません。  
これは下記のような事態が起きる可能性があります。  
  
```swift

// 新しい動物を追加
struct Horse {
    let name: String
    func bark() {
        print("ヒヒーン")
    }
}


// Horseを追加
let animals: [Any] = [Dog(name: "Pochi"), Cat(name: "Tama"), Horse(name: "Pony")]

// Horseの条件分岐を追加するのを忘れていた
animals.forEach {
    switch $0 {
    case let dog as Dog:
        dog.bark()
    case let cat as Cat:
        cat.bark()
    default: fatalError("No way!!")
    }
}

// 実行すると、、、
// Fatal error: No way!!
```  
  
メソッドの呼び出しが限定的で近い場所にあれば良いですが  
開発の規模が大きくなるにつれて呼び出し箇所を探すのが困難になっていくことは  
あり得そうな話です。  
  
  
そこでenumを活用してみます。  
  
```swift

enum Animal {
    case dog(Dog)
    case cat(Cat)
}

let animals: [Animal] = [.dog(Dog(name: "Pochi")), .cat(Cat(name: "Tama"))]
animals.forEach {
    switch $0 {
    case .dog(let dog):
        dog.bark()
    case .cat(let cat):
        cat.bark()
    }
}
```  
  
Anyの時と同様にswitchで条件分岐が必要になりますが  
コンパイル時に型チェックをすることができ  
新規のcaseを追加した場合にも気がつくことができるようになります。  
  
```swift

enum Animal {
    case dog(Dog)
    case cat(Cat)
    case horse(Horse)
}

let animals: [Animal] = [.dog(Dog(name: "Pochi")), .cat(Cat(name: "Tama")), .horse(Horse(name: "Pony"))]
animals.forEach {
    // error: switch must be exhaustive
    switch $0 {
    case .dog(let dog):
        dog.bark()
    case .cat(let cat):
        cat.bark()
    }
}
```  
  
default文も不要です。  
  
  
### あれProtocolで良いのでは？  
  
その通りで上記の場合ですと  
同じ型と名前を持ったメソッドを呼んでいるため  
同じProtocolに適合させることができます。  
  
```swift

protocol Animal {
    func bark()
}

struct Dog: Animal {
    let name: String
    func bark() {
        print("ワン")
    }
}

struct Cat: Animal {
    let name: String
    func bark() {
        print("ニャー")
    }
}

struct Horse: Animal {
    let name: String
    func bark() {
        print("ヒヒーン")
    }
}

let animals: [Animal] = [Dog(name: "Pochi"), Cat(name: "Tama"), Horse(name: "Pony")]
animals.forEach { $0.bark() }
```  
  
enumの場合よりも簡潔で良さそうです。  
  
しかし  
それぞれが異なるメソッドを持っている場合は  
Protocolに適合させることはできません。  
  
さらに  
下記のようにするとProtocolでは実現できなくなります。  
  
  
```swift

protocol FeedType {
    var name: String { get }
}

struct AnimalFeed: FeedType {
    let name: String
}

protocol Animal {

    //  ...

    associatedtype Feed: FeedType
    func eat(feed: Feed)
}

struct Dog: Animal {

    //  ...

    typealias Feed = AnimalFeed
    func eat(feed: AnimalFeed) {
        print("\(name)の\(feed.name)")
    }
}

// error: protocol 'Animal' can only be used as a generic constraint because it has Self or associated type 
let animals: [Animal] = [Dog(name: "Pochi"), Cat(name: "Tama"), Horse(name: "Pony")]
let petFood = AnimalFeed(name: "エサ")
animals.forEach { $0.eat(feed: petFood) }
```  
  
これはProtocolの制約でassociatedtypeやSelfを使用すると  
直接型として使用することができなくなります。  
  
enumの場合ですとこういった問題はありません。  
  
```swift

enum AnimalEnum{
    case dog(Dog)
    case cat(Cat)
    case horse(Horse)
}

let animals: [AnimalEnum] = [.dog(Dog(name: "Pochi")), .cat(Cat(name: "Tama")), .horse(Horse(name: "Pony"))]
let petFood = AnimalFeed(name: "エサ")

animals.forEach {
    switch $0 {
    case .dog(let dog):
        dog.eat(feed: petFood)
    case .cat(let cat):
        cat.eat(feed: petFood)
    case .horse(let horse):
        horse.eat(feed: petFood)
    }
}

// Pochiのエサ
// Tamaのエサ
// Ponyのエサ
```  
  
ただし  
caseが増えていくような場合は  
switch文の処理が増えて可読性が下がったり修正の負担が増加しますので  
そういった際はtype eraserを使うなど他の方法の検討も考えるべきだと思います。  
  
  
## 不整合な状態を考慮しなくて良い  
  
### URLSessionとResult  
  
よく挙げられる例として  
URLSessionのdataTaskのcompletionHandlerの引数があります。  
  
今回はSwift5で導入されるResultの  
Proposalに書かれている例から考えてみたいと思います。  
https://github.com/apple/swift-evolution/blob/master/proposals/0235-add-result.md  
  
注:   
これは実際にこうなるという訳ではなく  
Resultを使うとこういう形にできるのではないかという話です。  
現状では戻ってきた結果を  
自分でResult型に変換するなどの共通の処理が必要になってきます。  
  
```swift

func dataTask(with request: URLRequest, 
completionHandler: @escaping (Data?, URLResponse?, Error?) -> Void) -> URLSessionDataTask
```  
  
となっていますが  
実際に起こりうる結果を見てみると下記のようになります。  
  
  
| Data?       |       URLResponse? |    Error?    |  戻り値としてあり得るか？　|  
|:-----------------:|:------------------:|:------------------:|:------------------:|  
| nil | nil | nil| No  
| nil | value | nil | No  
| value | nil | nil | No  
| value | nil | value | No  
| value | value | value | No   
| nil | value | value | Yes (※1)  
| nil | nil | value | Yes  
| value | value | nil | Yes  
  
  
この場合、実際にはあり得ない場合に対しても考慮が必要になってしまいます。  
  
```swift

URLSession.shared.dataTask(with: url) { (data, response, error) in
    guard error != nil else { self.handleError(error!) }
    
    guard let data = data, let response = response else { return // Impossible? }
    
    handleResponse(response, data: data)
}
```  
  
それをResultを活用することで不要な可能性を考慮する負担が減り  
下記のように簡潔に書くことができます。  
  
```swift

URLSession.shared.dataTask(with: url) { (result: Result<(response: URLResponse, data: Data), Error>) in // Type added for illustration purposes.(※1)
    switch result {
    case let .success(success):
        handleResponse(success.response, data: success.data)
    case let .error(error):
        handleError(error)
    }
}
```  
  
(※1)   
実際にはResponseとErrorに値が入っているケースもあるらしく  
下記のような形が正しいようです。  
  
```swift

Result<(Data, URLResponse), (Error, URLResponse?)>
```  
  
https://oleb.net/blog/2018/03/making-illegal-states-unrepresentable/  
ただしこれだとSwift5のResultの型には合いません。  
  
### 画面の状態を管理する  
  
他の例としてある画面の状態について考えてみたいと思います。  
  
すごいざっくりとした例ですが  
通信してデータを取得し表示する画面があるとします。  
  
下記は画面の状態を表現するデータ構造です。  
  
```swift

struct ViewState {
    let isLoading: Bool
    let isEmpty: Bool 
    let data: ViewData?
    let error: Error?
}

struct ViewData {
    let screenName: String
    let screenImage: String
}
```  
  
こちらもURLSessionと同様に  
あり得ない状態のチェックが必要になります。  
  
例えば  
  
**通信終了後、dataがあるのにisEmptyがtrueになっている**  
**dataがあるのにerrorもある**  
  
など  
  
これを確認するためにはunitテストを書いたり  
実際に動かして確認するなどが必要になり  
コストが増えてしまいます。  
  
今回は極めてシンプルな状況ですが  
これにリフレッシュやページングの処理が加わることで  
必要な変数が増えたりするとどんどんコストが増加していきます。  
  
また、SomeDataやErrorはOptionalになっているため  
所々でnilチェックをする必要も出てきます。  
  
  
では、これをenumで表現してみます。  
  
```swift

enum ViewState {
  case empty
  case loading
  case data(ViewData)
  case failed(Error)
}
```  
  
こうすることで  
  
**unitテストや動作確認が必要 -> コンパイラがチェックしてくれるので不要**  
  
**nilチェックが必要 -> 必要なcaseで必ず値があることが保証されるので不要**  
  
といったメリットが生まれます。  
  
  
# enumにしたことで扱いづらくなるものもある  
  
このようにenumにすることで恩恵を受けることができますが  
逆に複雑さを増してしまうこともあります。  
  
## Viewをコントロールする  
  
例えば、通信中の場合はローディングを表示したいとします。  
  
structの場合ですと  
  
```swift

// ViewControllerの中だと思ってください

var state: ViewState!
var indicator: UIActivityIndicator = UIActivityIndicatorView()

self.indicator.isHidden = state.isLoading == false
```  
  
enumの場合ですと  
  
```swift

self.indicator.isHidden = {
  guard case .loading = state else {
    return false
  }
  return true
}
```  
  
とちょっと複雑さが増してしまいます。  
  
さらに  
状況によってボタンのタイトルを変更するなどは  
caseごとの表示方法の定義が必要になるかもしれません。  
  
## Computed Propertyを活用してアクセスを楽にする  
  
これを解消するためにComputed Propertyを定義します。  
  
```swift

extension ViewState {
  var isLoading: Bool {
    guard case .loading = self else { return false }
    return true
  }
  
  var isEmpty: Bool {
    guard case .empty = self else { return false }
    return true
  }
  var data: ViewData? {
    guard let case .data(viewData) = self else { return nil }
    return viewData
  }
  var error: Error? {
    guard let case .error(error) = self else { return nil }
    return error
  }
}
```  
  
こうすることでstructの時と同じようにアクセスをすることができるようになります。  
  
一方でPropertyが増えるとコードも増えることになるので  
必要なものだけ定義をするか  
一部だけ宣言されていると「なぜここだけ？」となってしまうため  
一律宣言するべきかは後々の悩みどころかもしれません。  
  
※ Sourceryなどでコード生成をしている人もいるようです。  
  
  
## まとめ  
  
enumの性質を活用することで  
リスクの軽減やデータの扱いやすさの向上が期待できるのではないかと考え  
実際に活用している場面も多くあると感じています。  
  
一方で使いづらい状況というところもあり  
一律にこれが良いということも言えないということの再認識を行えました。  
  
どういった時にどういう状況で使えるのか  
日々考えながらちょうど良い落とし所を見つけられるようにしていきたいと思います。  
  
何か間違いなどありましたらご指摘頂けますと幸いです:bow_tone1:  
