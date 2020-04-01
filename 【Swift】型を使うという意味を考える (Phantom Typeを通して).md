# 経緯  
  
Swiftは型安全な言語であると言われています。  
  
その理由の一つとしてコンパイル時のチェックが行われるため  
実行時エラーの可能性を減らすことができるからです。  
  
もちろん毎日コードを書いている中で  
型安全の恩恵は受けているのですが  
  
型安全とはどういうことなのか  
もっと型を有効に使うことはできないのか  
  
そんなことを考える中で  
Phantom Typeについて少し検討してみました。  
  
# Phantom Typeとは？  
  
型としては出てくるものの  
実装としては登場してこない型を指します。  
  
見えないけど存在しているまさに幽霊のような存在です。  
  
# 何が良いのか？  
  
Phantom Typeが存在することでコンパイラチェックが働き  
実行時に起こりえない状態を除外することができます。  
  
# 使用例  
  
良く挙げられる例としては状態管理があります。  
Phantom Typeが現在の状態を表現すると同時に  
次に移行できる状態を特定の状態に制限することができます。  
  
# 具体例で考えてみる  
  
Amazonのようなショッピングサイトで  
商品の注文から商品が届くまでの流れを考えたいと思います。(かなり簡略版です)  
  
注文から料金が請求され、支払いをすると商品が配達されます。  
状態に応じて注文のキャンセル、返金、返品が可能になります。  
  
下記に簡単な図を示します。  
  
![PhantomType.png](/image/e9faabb0-91a5-aa45-8a8c-4890aac1d4b8.png)  
  
# 実装  
  
まず、それぞれの状態を定義します。  
  
```swift

// 注文の状態を表す何か
protocol OrderState {}

// 請求ができる
protocol Billable {}
// 支払いができる
protocol Payable {}
// キャンセルができる
protocol Cancellable {}
// 返金ができる
protocol Refundable {}
// 返品ができる
protocol Returnable {}

// 注文済状態
enum Ordered: OrderState, Cancellable, Billable {}
// 請求済状態
enum Billed: OrderState, Cancellable, Payable {}
// 支払い済状態
enum Paid: OrderState, Refundable {}
// 配達済み状態
enum Delivered: OrderState, Returnable {}
// キャンセル済状態
enum Cancelled: OrderState {}
// 返金済状態
enum Refunded: OrderState {}
// 返品済状態
enum Returned: OrderState {}

```  
  
protocolを使っているのは複数の状態から同じ処理をする場合に  
protocol extensionを活用できるからです。  
  
次に実際の注文の状態を表現するクラスを定義します。  
  
```swift

final class Order<T: OrderState> {
    private let id: Customer.ID
    private var item: OrderedItem
    init(id: Customer.ID,
         item: OrderedItem) {
        self.id = id
        self.item = item
    }
}
```  
  
次にある状態からある状態へ遷移する方法を定義していきます。  
各extensionの**where句**に注目してください。  
  
```swift

// 注文済状態へ
extension Order where T == Ordered {
    convenience init(id: Customer.ID, cart: Cart) {
        self.init(id: id, item: OrderedItem(items: cart.items))
    }
}

// 請求済状態へ
extension Order where T: Billable {
    var bill: Order<Billed> {
        return Order<Billed>(id: self.id, item: self.item)
    }
}

// 支払い済状態へ
extension Order where T: Payable {
    var pay: Order<Paid> {
        return Order<Paid>(id: self.id, item: self.item)
    }
}

// キャンセル済み状態へ
extension Order where T: Cancellable {
    var cancel: Order<Cancelled> {
        return Order<Cancelled>(id: self.id, item: self.item)
    }
}

// 返金済状態へ
extension Order where T: Refundable {
    var refund: Order<Refunded> {
        return Order<Refunded>(id: self.id, item: self.item)
    }
}

// 配達済状態へ
extension Order where T == Paid {
    var deliver: Order<Delivered> {
        return Order<Delivered>(id: self.id, item: self.item)
    }
}

// 返品済状態へ
extension Order where T: Returnable {
    var orderReturn: Order<Returned> {
        return Order<Returned>(id: self.id, item: self.item)
    }
}
```  
  
where句のおかげで  
**今〇〇状態にある場合のみ〇〇状態へ遷移できる**  
ということを明示できます。  
  
こうすることで  
〇〇状態へはある状態にある場合にしか遷移できない  
ということをコンパイラがチェックしてくれるようになります。  
  
### 補足: 上記で使われているCartやOrderedItemは下記のようになっています。  
  
※ 必要な箇所だけ示しています  
※ ここで使われているIdentifierもPhantom Typeです。詳細は後ほど記載します。  
  
```swift

// お客様が注文する前の商品を入れるカート
final class Cart {
    var items: [Item: Int] = [:]    
}

// 商品
struct Item: Hashable {
    let id: ID
    let price: Int
    
    typealias ID = Identifier<Item, String>
}

// 注文した商品
struct OrderedItem {
    var items: [Item: Int] = [:]
}
```  
  
実際に試してみます。  
  
```swift

let orderedItem = OrderedItem(items: [
    Item(id: "ball", price: 100): 10,
    Item(id: "bat", price: 1000): 1
])
        
let r = Order<Ordered>(id: "hoge", item: orderedItem).bill.pay.deliver
print(r) // PhantomType.Order<PhantomType.Delivered>
```  
  
配達済状態にするには  
  
**注文済->請求済->支払済->配達済**  
  
というルートを辿らなければなりません。  
  
例えば、  
  
```swift

let r = Order<Ordered>(id: "hoge", item: orderedItem).deliver
// 'Order<Ordered>' is not a subtype of 'Order<Paid>'
```  
  
となって注文済->配達済にジャンプすることはできないですし  
  
```swift

let r = Order<Ordered>(id: "hoge", item: orderedItem).bill.pay.refund.deliver
// 'Order<Refunded>' is not a subtype of 'Order<Paid>'
```  
  
返金したあとに配達してしまうという間違いも起きません。  
  
また  
今回はただ状態の遷移だけを扱っていますが  
遷移の途中に処理を加えたり  
メソッドにして引数を渡すなど色々なことができます。  
  
もうちょっと他のことも試したソースのURLを下記に貼っておきます。  
https://github.com/stzn/PhantomTypeSample  
  
  
# もう一つのPhantom Typeの活用法  
  
途中で少し出てきましたが  
Phantom Typeの活用方法として  
識別子に用いられることも多くあります。  
  
例えば  
モデルを定義するときにidを使うことはよくあると思いますが  
多くのケースでStringが使われます。  
  
すると下記のようなバグを引き起こす可能性があります。  
  
```swift

struct User {
    let id: String
}

struct Dog {
    let id: String
}

func getUser(id: String) -> User {
}

let user = User(id: "1")
let dog = Dog(id: "1")

// 取れてしまう
let user1 = getUser(id: dog.id)

```  
  
UserのIDもDogのIDもStringなので  
エラーにならずに取得できてしまいます。  
  
さらにモデルが多くなると  
色々な場所で出てくるIDがどのIDなのかが  
すぐに判断できなくなるかもしれません。  
  
そこで下記のようなID用のクラスを定義します。  
  
```swift

struct Identifier<T, RawValue> {
    let value: RawValue
    init(_ value: RawValue) {
        self.value = value
    }
}
```  
  
TがPhantom TypeでRawValueが実際のIDの型です。  
  
※ 割愛しますが、Hashable, Equatablや文字列から初期化できるようにProtocolに適合させています。   
  
  
こうすると先ほどの例でコンパイラチェックが働くようになります。  
  
```swift

struct User {
    let id: ID
    typealias ID = Identifier<User, String>
}

struct Dog {
    let id: ID
    typealias ID = Identifier<Dog, String>
}

func getUser(id: User.ID) -> User {
}

let user = User(id: "1")
let dog = Dog(id: "1")

// Cannot convert value of type 'Dog.ID' (aka 'Identifier<Dog, String>') to expected argument type 'User.ID' (aka 'Identifier<User, String>')

let user1 = getUser(id: dog.id)
```  
  
# まとめ  
  
Phantom Typeを通して型を使うということに考えてみました。  
  
Phantom Typeという  
あえてなんの実装も持たない型を使うということを通して  
Swiftが型を持っていることの強さや恩恵をより深く感じることができた気がします。  
  
また  
型を通して実行時エラーの可能性を減らせることに加えて  
できることを制限することで  
複数人で開発しているときの共通理解を深めることができたり  
スタイルの統一などにも繋がるのかなと思いました。  
  
一方で  
コード量が増えるケースは多いと思います。  
  
また  
知らない人が最初に見た場合  
extensionが多くて読みづらかったり  
理解に少し時間がかかってしまうかもしれないとも感じ  
わかりやすい名前付けなどを意識することが大切だとも思いました。  
  
今回はPhantom Typeを用いましたが  
まだまだ型を活用した例はたくさんありますので  
型という特性を持つSwiftをもっと理解して活用できるように  
また型について考えてみたいと思います:smiley:  
  
  
間違いなどございましたらご指摘いただけますとうれしいです:bow_tone1:  
  
  
# 参照記事  
  
https://lukematthewsutton.com/post/type-safe-state-machines/  
http://kean.github.io/post/phantom-types  
https://github.com/pointfreeco/swift-tagged  
