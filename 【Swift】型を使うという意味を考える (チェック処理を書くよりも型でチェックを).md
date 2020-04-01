Swiftが静的型付け言語であるという特性を活かす方法として  
コンパイラチェックがあります。  
  
# 例  
  
例えば、下記のような例を考えます。  
  
```swift

enum UserType {
    case free
    case paying
}

struct User {
    let id: Int
    let name: String
    let type: UserType
}

func sayHello(user: User) {
    guard case .paying = user.type else { return }
    print("ようこそ！")
}

let free = User(id: 1, name: "1", type: .free)
let paying = User(id: 2, name: "2", type: .paying)

sayHello(user: free)
sayHello(user: paying)

```  
  
sayHelloメソッドでUserTypeによるチェックをしています。  
  
こういう風にしてしまうと  
  
- 毎回チェック処理が走る  
- コアのロジックに余計な処理な混じる  
- テストを書く必要がある  
  
などが起きます。  
  
これを型を活かす例を考えてみます。  
  
```swift

protocol WelcomeUser {}

struct FreeUser {
    let id: Int
    let name: String
}

struct PayingUser: WelcomeUser {
    let id: Int
    let name: String
}

func sayHello<U: WelcomeUser>(user: U) {
    print("ようこそ！")
}

let free2 = FreeUser(id: 1, name: "1") // Argument type 'FreeUser' does not conform to expected type 'WelcomeUser'
let paying2 = PayingUser(id: 2, name: "2")

sayHello(user: free2)
sayHello(user: paying2)
```  
  
こうすることでWelcomaUser以外は引数に設定することができなくなり  
  
- コンパイラが代わりにチェックをしてくれるのでテストを書く必要がなくなる  
- メソッドがコアのロジックに集中できる  
  
  
などがあります。  
  
  
今回はとてもシンプルな例ですが  
処理が複雑になればなるほど  
チェック処理が増え  
何をするものなのかがわかづらくなり  
理解するのに時間がかかってしまうこともあります。  
  
※もちろん処理を分割してシンプルにしていく必要はありますが  
  
またテストケースも増えていくため  
テストを書くコストとメンテナンスのコストも増えます。  
  
  
# まとめ  
  
型を活用するシンプルな例を紹介しました。  
  
言語が持つ特性を活用することでより良いコードが書けるようになるのは  
この言語を使用している意義が感じられて個人的には気持ちが良いなと思いました:smile:  
  
ぜひ言語の良さをどんどん活かしていきましょう。:muscle_tone1:  
