プログラミングの手法で「カリー化」と呼ばれるものがあります。  
  
# カリー化とは？  
  
関数の引数の一部分にだけ実引数を適用し  
残りの引数を後から適用することを**部分適用**と呼びます。  
  
その中でも  
複数の引数を取る関数を  
引数１つの関数に変換することを  
**カリー化**と呼びます。  
  
今回は  
カリー化をすることでどういうことができるのか  
具体例から学んでみました。  
  
  
# 具体例  
  
## 1つの値を固定して様々な値を適用する  
  
まず簡単な例から考えていきたいと思います。  
  
```swift

func add(_ a: Int, _ b: Int) -> Int {
    return a + b
}
add(5, 2) // 7

```  
これは単純な足し算で2つの引数を取ります。  
  
では、ある範囲の値全てに２を足してみます。  
  
```swift

let xs = 1...5
xs.map { add($0, 5)} // [6, 7, 8, 9, 10]
```  
addの引数の一つを２に固定して  
範囲内の値を一つ一つもう一つの引数に適用していきます。  
  
では、  
これをカリー化してみます。  
  
```swift

func addFive(_ a: Int) -> Int {
    return add(a, 5)
}

let xs = 1...5
xs.map(addFive) // [6, 7, 8, 9, 10]
```  
同じ結果が得られました。  
  
変わった点としては  
  
- 引数を指定する必要がなくなった  
- {}が必要なくなった  
  
これは非常に単純な例ですが  
より複雑な関数になった場合などはちょっと見づらかったり  
冗長になってきます。  
  
また、カリー化することで名前をつけることができるので  
より意図を伝えやすくなります。  
  
では少し一般化してみます  
  
```swift

func add(_ a: Int) -> (Int) -> Int {
    return { b in a + b }
}

let addSix = add(6)
addSix(1) // 7
```  
  
これで色々なIntに対して適用できるようになりました。  
  
## 既存の関数をカリー化する  
  
既存の関数に関して新しい条件を適用したりしたい場合にも  
カリー化が役に立つことがあります。  
  
例えば、下記の関数をIntが持っていたとします。  
  
```swift

extension Int {
    static func add(_ a: Int, _ b: Int) -> Int {
        return a + b
    }
}
```  
  
これは  
(Int, Int) -> Int型  
の関数になります。  
  
これをカリー化するためにはこの型を  
Int -> ((Int) -> Int)型  
にする必要があります。  
  
この変換のための関数を定義します。  
  
  
```swift

func curry(_ f: @escaping (Int, Int) -> Int) -> ((Int) -> ((Int) -> Int)) {
    return { a in return { b in f(a, b) } }
}

let xs = 1...5
let curried = curry(Int.add)
let addSeven = curried(7)

xs.map(addSeven) // [8, 9, 10, 11, 12]
```  
  
だいぶ複雑になりました。  
  
何をしているのかというと  
  
**fがカリー化したい関数**で（今回の場合はadd）  
  
戻り値は(Int) -> ((Int) -> Int)で  
  
**Int(←fの引数の1つ)を引数に取った**  
**((Int（←fの引数の1つ）) -> Int（←fの戻り値)の関数**  
  
になります。  
  
具体例の中のaddSevenについて考えると  
7はすでに決まっているので  
  
```swift

func addSeven(_ b: Int) -> (Int) -> Int {
    return { b in
        Int.add(7, b)
    }
}
```  
になります。  
  
上記の足し算の例で示したaddFiveと同じ形になっていますね。  
  
  
上記はIntだけの例ですが  
ここからさらに一般化してみます。  
  
```swift

func curry<A, B, C>(_ f: @escaping (A, B) -> C) -> (A) -> (B) -> C {
    return { a in { b in return f(a, b) } }
}

let curried = curryInt(Int.add)
let addSeven = curried(7)

xs.map(addSeven) // [8, 9, 10, 11, 12]
```  
同じ結果になりました。  
  
※今回の例では引数が2つの関数を扱っていますが、  
それ以上の引数を持つ関数を使う場合は  
それ用の定義が毎回必要になります。  
  
このような感じで↓  
https://github.com/thoughtbot/Curry/blob/master/Source/Curry.swift  
  
Swift5の時点ではGenericsでは可変長引数を使うことができないためです。  
将来的には導入を検討しているようです。  
https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md#variadic-generics  
  
## UserDefaultsでの例  
  
次に上記で一般化した関数を使ってみたいと思います。  
  
UserDefaultlsは値の保存を行う際にsetという関数が提供されています。  
  
例えばuserNameというキーで値を保存する場合は  
  
```swift

UserDefaults.standard.set("Tom", forKey: "userName")
```  
のように書きます。  
  
これでも十分わかりやすいのかもしれませんが  
カリー化を使うことでより可読性を上げることができます。  
  
```swift

extension UserDefaults {
    func setT<T>(key: String, value: T?) {
        return self.set(value, forKey: key)
    }
    
    func curried<T>(_ key: String) -> (T?) -> Void {
        return curry(self.setT)(key)
    }
}
```  
  
このような拡張をすることで  
  
```swift

extension UserDefaults {
    var getUserName: String? {
        return self.string(forKey: "userName")
    }
    
    var setUserName: (String?) -> Void {
        return self.curried("userName")
    }
}

let standard = UserDefaults.standard
standard.setUserName(nil)
standard.getUserName // nil

standard.setUserName("hoge") // hoge
standard.getUserName
```  
  
といったようなプロパティを宣言することができます。  
※getUserNameは確認用に使用しているだけです。  
  
こちらの方が何をしているのかが相手に伝わりやすいのではないかと思います。  
  
## DIとして使用する  
  
例えば  
ネットワーク通信を行って  
ユーザ情報を取得したいとします。  
  
  
```swift

struct User {
    let id: String
    let name: String
}

typealias Completion<T> = (Result<T, Error>) -> Void

protocol UserFetchable {
    func fetch(id: String, completion: @escaping Completion<User>)
}
```  
  
この通信Protocolを呼び出すサービスを定義します。  
  
```swift

enum UserService {
    
    static func getUserBy(fetcher: UserFetchable, id: String, completion: @escaping Completion<User>) {
        fetcher.fetch(id: id, completion: completion)
    }
}
```  
  
ここではfetcherを引数に設定していますが  
状況によってはfetcherの決定を遅らせたいこともあります。  
  
そんな時にカリー化を活用します。  
  
```swift

enum UserService {
    static func getUserBy(id: String, completion: @escaping Completion<User>) -> (UserFetchable) -> Void {
        return { fetcher in
            curry(fetcher.fetch)(id)(completion)
        }
    }
}
```  
  
下記のように使用します。  
  
```swift

extension URLSession: UserFetchable {
    func fetch(id: String, completion: @escaping Completion<User>) {
        completion(.success(User(id: "1", name: "Tom")))
    }
}

let getUser = UserService.getUserBy(id: "1") { result in
    print(result)
}
getUser(URLSession.shared) // success(User(id: "1", name: "Tom"))

```  
  
こうすることでgetが呼び出されるまでUserFetchableの  
実装の決定を遅らせることができます。  
  
※ これはカリー化を活用した例で他にもReader Monadのように型を活用する方法もあります。  
  
詳細は↓などをご参照ください。  
https://qiita.com/suin/items/0255f0637921dcdfe83b  
https://github.com/orakaro/Swift-monad-Maybe-Reader-and-Try  
  
  
# まとめ  
  
カリー化を使った例を見てみました。  
  
カリー化は一般的な関数であり  
もっと色々な箇所で活用できると思います。  
  
特に関数型プログラミングの世界では  
関数を繋げて使うことが基本であり  
引数を１つにすることで  
関数を繋げて使いやすくなります。（関数の合成と言うそうです。）  
  
正直今まであまり使用することもなくこの機会に学んでみました。  
  
まだまだ知らないことが多々あると思いますので  
もしもっと良い例がございましたらぜひ教えてください:smiley:  
  
間違いなどございましたらご指摘いただけましたら幸いです:bow_tone1:  
  
# 補足  
  
SwiftではSwift３の時に一部記法が削除されています。  
過去の記事で現在はコンパイルできないものがあるかもしれないのでご注意ください。  
https://github.com/apple/swift-evolution/blob/master/proposals/0002-remove-currying.md  
