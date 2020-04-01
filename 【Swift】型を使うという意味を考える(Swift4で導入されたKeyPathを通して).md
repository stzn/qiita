先日[Phantom Typeについての記事](https://github.com/stzn/qiita/blob/master/【Swift】型を使うという意味を考える (Phantom Typeを通して).md  
)を書き、  
他にも型を活用するということで色々検討していた際に  
Swift4で導入されたKeyPathの存在をふと思い浮かべました。  
  
  
そんな中で  
try! Swift2019に参加した際に  
Benedikt TerhechteさんがKeyPathについて紹介していたこともあり  
今回改めて自分の中で勉強し直してみようと思い、記録として内容をまとめてみました。  
https://speakerdeck.com/terhechte/introduction-to-swift-keypaths  
  
※  
あと個人的な理由として  
今回のtry! Swiftでは役割上スピーカーの方の横にいることが多く  
Benediktさんの発表はランチ前で  
発表後にランチの順番を待ちながら色々話をすることができたので深く印象に残っていたという点もあります。  
(「お腹空いたねー。弁当何食べる？」とかほんとたわいのない内容でしたがw)  
  
# KeyPathとは  
  
ドキュメントによると  
  
```swift

Use key-path expressions to access properties dynamically
```  
  
「動的にプロパティにアクセスするために使用する」ための表現と書かれています。  
  
https://developer.apple.com/documentation/swift/swift_standard_library/key-path_expressions  
  
  
# なぜ必要だったの？  
  
プロポーサルには下記のようなことが書いてあります。  
  
### Stringよりもより扱いやすくすることができる  
  
既存の#keyPath()は安全にプロパティにアクセスできるけど、以下の制限がある  
  
- 型情報が失われる(Anyになる)  
- 解析が不必要に遅い  
- NSObjectsを継承しないと使えない  
- Darwinプラットフォームでしか使用できない  
  
https://github.com/apple/swift-evolution/blob/master/proposals/0161-key-paths.md#we-can-do-better-than-string  
  
### 特徴を使用する  
  
メソッドは```let x = foo.bar()```の代わりに```let x = foo.bar```を使うことで  
メソッドを実行しないでメソッドの参照を獲得できるが、プロパティやsubscriptではできない  
  
具体的なクラスのプロパティへの間接的な参照を作ってしまうと  
プロパティのメタデータや将来的に追加される動作を外部にさらしてしまう  
  
https://github.com/apple/swift-evolution/blob/master/proposals/0161-key-paths.md#usemention-distinctions  
  
### より多くの表現を可能に  
  
現状では不可能なコレクションやその他の添字アクセス可能なプロパティにKeyPathを使ってアクセスできるようにしたい  
  
https://github.com/apple/swift-evolution/blob/master/proposals/0161-key-paths.md#more-expressive-keypaths  
  
  
では、次に具体的なKeyPathの中身を見ていきたいと思います。  
  
  
# 5つのクラス  
  
実装を見てみると  
以下の5つのクラスが階層になって構成されています。  
  
```swift

AnyKeyPath
↓
PartialKeyPath<Root>
↓
KeyPath<Root, Value>
↓
WritableKeyPath<Root, Value>
↓
ReferenceWritableKeyPath<Root, Value>

```  
  
https://github.com/apple/swift/blob/master/stdlib/public/core/KeyPath.swift  
  
  
各クラスの適用範囲と何ができるのか(値の読み取りは全てできるので変更が可能かどうか)  
を簡単にまとめると下記になります。  
  
| クラス                     | 適用範囲                   | 値の変更(set)が可能 |  
|:-------------------------|:--------------------------|:---------|  
| AnyKeyPath               |              全て          |        ×  |  
| PartialKeyPath           |              全て          |   ×       |  
| KeyPath                  |              全て          |         × |  
| WritableKeyPath          | Value Semantics(値型)      |         ◯ |  
| ReferenceWritableKeyPath | Reference Semantics(参照型)|         ◯ |  
  
  
それでは  
個々のクラスについて見ていきます。  
  
今回、例として下記のクラスを使っています。  
※ varを使用しているのは意図的にmutableであることを示したいからです。  
  
```Swift

struct User {
    let id: String
    let name: String
    let age: Int
}

var user = User(id: "1", name: "hoge", age: 20)

```  
  
### 検証環境  
  
Xcode10.2 beta3  
  
## AnyKeyPath  
  
https://developer.apple.com/documentation/swift/anykeypath  
  
これはType Eraserと呼ばれるものの一種で  
RootとValueの両方ともコンパイルの時点では型は特定されず  
実行時に具体的な型に解決されます。  
  
```swift

let kp: AnyKeyPath = \User.id
print(type(of: user[keyPath: anyKeyPath])) // Optional<Any>
```  
  
下記のプロパティを参照すると具体的な型はわかるようです。  
  
```swift

print(type(of: anyKeyPath).rootType) // User
print(type(of: anyKeyPath).valueType) // String
```  
  
また、ReadOnlyなため  
値の変更しようとするとエラーになります。  
  
```swift

// Cannot assign through subscript: 'user' is immutable
user[keyPath: anyKeyPath] = "3" 

```  
  
エラーメッセージはなんか不思議ですね:sweat_smile:  
  
  
## PartialKeyPath<Root>  
  
https://developer.apple.com/documentation/swift/partialkeypath  
  
これもType Eraserと呼ばれるものの一種で  
Rootはコンパイルの時点で型が特定していますが、  
Valueは実行時に具体的な型に解決されます。  
  
どういう時に活用できるかというと  
ある特定の型のKeyPathの配列を保持したい場合などに活用できます。  
  
```swift

let partialKeyPaths: [PartialKeyPath<User>] = [
    \User.id, \User.age
]

print(partialKeyPaths) // Swift.KeyPath<User, Swift.String>, Swift.KeyPath<User, Swift.Int>
```  
  
  
また、PartialKeyPathを使用した際に下記のような結果となります。  
  
```swift

let partialkeyPath: PartialKeyPath<User> = \User.id

print(type(of: user[keyPath: partialkeyPath])) // String
print(user[keyPath: partialkeyPath] is String) // true
```  
  
ドキュメント上だと  
  
```
any resulting value type.
```  
  
なのでAnyが返ってくると思ったのですが、型はStringになっています:thinking:  
もしご存知の方がいらっしゃいましたら教えてください:bow_tone1:  
  
また、ReadOnlyなため  
値の変更しようとするとエラーになります。  
  
```swift

// Cannot assign through subscript: 'user' is immutable
user[keyPath: partialkeyPath] = "3" 

```  
  
  
  
## KeyPath<Root, Value>  
  
https://developer.apple.com/documentation/swift/keypath  
  
RootもValueもコンパイルの時点で型が特定されます。  
  
  
```swift

let keyPath: KeyPath<User, String> = \User.id
print(type(of: user[keyPath: keyPath])) // String

```  
  
上記のkeyPath変数の型を確認すると  
  
```swift

print(type(of: keyPath)) // WritableKeyPath<User, String>
```  
  
と出力されます。  
  
しかし、ReadOnlyなため  
値の変更しようとすると下記のようにエラーになります。  
  
```swift

// Cannot assign through subscript: 'keyPath' is a read-only key path
user[keyPath: keyPath] = "3" 
```  
  
型がWritableKeyPathと出力される理由がいまいちわかっていません:sweat_smile:  
  
  
  
## WritableKeyPath<Root, Value>  
  
https://developer.apple.com/documentation/swift/writablekeypath  
  
RootもValueもコンパイルの時点で型が特定され  
値の変更も可能です。  
  
WritableKeyPathは**Value Semantics(値型)**に使用できます。  
  
※ Value Semanticsかどうかは下記の部分でReference Prefixがないことをチェックしています。  
  
```swift

_internalInvariant(!buffer.hasReferencePrefix,
    "WritableKeyPath should not have a reference prefix")
```  
  
https://github.com/apple/swift/blob/master/stdlib/public/core/KeyPath.swift#L300  
  
  
下記のように値の変更ができます。  
  
```swift

let writableKeyPath = \User.id

print(type(of: writableKeyPath)) // WritableKeyPath<User, String> 

user[keyPath: writableKeyPath] = "3" // OK
```  
  
上記の例からもわかるように変数に代入する際に  
structの場合だと明示的に型を指定しない場合はWritableKeyPathになります。  
  
注意点としては  
値を書き換えようとしている変数がletで宣言されている場合は変更できません。  
  
```swift

let user2 = User(id: "2", name: "hoge2", age: 22)
let writableKeyPath = \User.id

// Cannot assign through subscript: 'user2' is a 'let' constant
user2[keyPath: writableKeyPath] = "3"
```  
  
  
## ReferenceWritableKeyPath<Root, Value>  
  
https://developer.apple.com/documentation/swift/referencewritablekeypath  
  
RootもValueもコンパイルの時点で型が特定され  
値の変更も可能です。  
  
ReferenceWritableKeyPathは**Reference Semantics(参照型)**に使用できます。  
  
```swift

class UserClass {
    var id: String
    var name: String
    let age: Int

    init(id: String, name: String, age: Int) {
        self.id = id
        self.name = name
        self.age = age
    }
}

var classUser = UserClass(id: "3", name: "piyod", age: 23)

let referenceWritableKeyPath: ReferenceWritableKeyPath<UserClass, String> = \UserClass.id
print(type(of: referenceWritableKeyPath)) // ReferenceWritableKeyPath<UserClass, String>

classUser[keyPath: referenceWritableKeyPath] = "5"

```  
  
ちなみにstructに無理やり適用しようとしてもキャストできません。  
  
```swift

let writableKeyPath: ReferenceWritableKeyPath<User, String> = \User.id as! ReferenceWritableKeyPath<User, String>

Could not cast value of type 'Swift.WritableKeyPath<User, Swift.String>' to 'Swift.ReferenceWritableKeyPath<User, Swift.String>'
```  
  
また、classをWritableKeyPathに適用しても  
ReferenceWritableKeyPathになります。  
  
```swift

let referenceWritableKeyPath: WritableKeyPath<UserClass, String> = \UserClass.id

print(type(of: referenceWritableKeyPath)) // ReferenceWritableKeyPath<User, String>

```  
  
以上の5つのクラスを使用して  
クラスのプロパティへアクセスします。  
  
  
# ネストされたクラスのプロパティへも簡単にアクセスが可能  
  
KeyPathは```_AppendKeyPath```というprotocolに適合しており、  
これの```appending```メソッドを使用することで  
ドット記法を用いて内部のプロパティのKeyPathへアクセスできます。  
  
https://developer.apple.com/documentation/swift/appendkeypath  
  
```swift

struct Owner {
    let name: String
    var pet: Pet
}

struct Pet {
    var name: String
    let age: Int
}

let petNameKeyPath = (\Owner.pet).appending(path: \Pet.name)
print(type(of: petNameKeyPath)) // WritableKeyPath<Owner, String>
print(type(of: petNameKeyPath).rootType) // Owner
print(type(of: petNameKeyPath).valueType) // String

var owner1 = Owner(name: "owner", pet: Pet(name: "ペット", age: 30))
print(owner1[keyPath: petNameKeyPath]) // ペット

owner1[keyPath: petNameKeyPath] = "Pet"

print(owner1[keyPath: petNameKeyPath]) // Pet
```  
  
  
# 具体的な活用例を見てみる  
  
KeyPathを活用することで  
  
- 重複の排除を行いコードを短くして可読性を向上させる  
- 特定のクラスに依存せずに値の取得・設定ができる  
  
といったメリットがあります。  
  
事例としてはたくさん存在しますが  
その中でも特徴的だと思われるものをいくつか紹介したいと思います。  
  
## ソート条件を指定する  
  
```swift

extension Sequence {
    func sorted<T: Comparable>(by keyPath: KeyPath<Element, T>) -> [Element] {
        return sorted { a, b in
            return a[keyPath: keyPath] < b[keyPath: keyPath]
        }
    }
}

let users = [
    User(id: "1", name: "a", age: 20),
    User(id: "2", name: "d", age: 30),
    User(id: "3", name: "c", age: 18),
    User(id: "4", name: "b", age: 22)
]

let nameSortedUsers = users.sorted(by: \.name)
print(nameSortedUsers)

/*
[
    User(id: "1", name: "a", age: 20), 
    User(id: "4", name: "b", age: 22), 
    User(id: "3", name: "c", age: 18), 
    User(id: "2", name: "d", age: 30)
]
*/


let ageSortedUsers = users.sorted(by: \.age)
print(ageSortedUsers)

/*
[
    User(id: "3", name: "c", age: 18), 
    User(id: "1", name: "a", age: 20), 
    User(id: "4", name: "b", age: 22), 
    User(id: "2", name: "d", age: 30)
]
*/

```  
  
これと同じ要領でフィルター条件などにも使えます。  
  
KeyPathKitの作者のVincent Pradeillesさんがここら辺を講演の中で詳しく解説されています。  
https://www.youtube.com/watch?v=20k3000Pn4s  
  
KeyPathKit  
https://github.com/vincent-pradeilles/KeyPathKit  
  
  
## AutoLayoutの同じAnchorへの制約を設定する  
  
コードでAutoLayoutを設定する場合に  
  
```swift

let view1 = UIView()
let view2 = UIView()

view1.topAnchor.constraint(equalTo: view2.topAnchor)

```  
  
と書きますが、  
topAnchorを２回書くのがちょっと冗長な気がします。  
  
これをKeyPathを使うと下記のように書くことができるようになります。  
  
  
```swift

import UIKit

typealias Constraint = (UIView, UIView) -> NSLayoutConstraint

func equal<L, Axis>(_ to: KeyPath<UIView, L>) -> Constraint where L: NSLayoutAnchor<Axis> {
    return { view1, view2 in
        view1[keyPath: to].constraint(equalTo: view2[keyPath: to])
    }
}

let v1 = UIView()
let v2 = UIView()

let constraint = equal(\.topAnchor)(view1, view2)
```  
  
https://www.objc.io/blog/2018/10/30/auto-layout-with-key-paths/  
  
  
## 共通の項目に個々のクラスのプロパティ値を設定する  
  
Benedikt Terhechteさんの発表では  
アプリ内の設定画面を構築する際に異なるモデルに対して  
共通的に処理できるようにしていました。  
  
同じような使い方としてUITableViewCellに対して  
値を設定するという活用法もあります。  
  
  
```swift

struct CellConfigurator<Model> {
    let titleKeyPath: KeyPath<Model, String>
    let subtitleKeyPath: KeyPath<Model, String>
    let imageKeyPath: KeyPath<Model, UIImage?>

    func configure(_ cell: UITableViewCell, for model: Model) {
        cell.textLabel?.text = model[keyPath: titleKeyPath]
        cell.detailTextLabel?.text = model[keyPath: subtitleKeyPath]
        cell.imageView?.image = model[keyPath: imageKeyPath]
    }
}
```  
  
https://www.swiftbysundell.com/posts/the-power-of-key-paths-in-swift  
  
個々のモデルに依存せず、  
どれがセルのどのViewに当てはめられるのかをKeyPathを通して伝えることができるようになります。  
  
## 2019/3/29 追記 Key Paths Expressions as Functions  
  
ちょうど  
SE-0249: Key Path Expressions as Functions  
がAcceptedになったので追記します。  
https://forums.swift.org/t/accepted-se-0249-key-paths-expressions-as-functions/22287  
  
これによって  
mapやfilterなどにもKeyPathが活用できるので  
下記のような感じでより簡潔な記法が可能になるようです。  
  
```swift

struct User {
    var id: String
    var name: String
    var isActive: Bool
}

users.map(\.name)
users.filter(\.isActive)
```  
  
## まとめ  
  
KeyPathの基礎的な部分の見直しからどうやって使われているのかを少し見てみました。  
  
改めてKeyPathは  
型安全な面と動的な面の良さを兼ね備えた存在だなと感じました。  
  
他にも活用例はまだまだたくさんありますし  
さらに気づいていない活用法はもっとあると思うので  
まず今回見直した基礎を踏まえた上で  
  
**どうKeyPathを活用できるのか**  
**なぜこの状況でKeyPathを使うのか**  
  
こういったことを考えながら検証していきたいと思います。  
  
  
ご指摘などございましたら教えていただけますと幸いです:bow_tone1:  
