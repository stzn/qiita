  
# はじめに  
  
Swift Advent Calendar 2018 の 16 日目です。  
  
AdventCalendar初参加させていただきます。  
  
@tattnさんの[Better Swift](https://qiita.com/tattn/items/3b90bb01e9771ec27b00)と少し趣旨が被っているなと感じていますが  
そこはご了承いただけますと幸いです。  
(月初には９割方書いており、テーマを変更する余裕がありませんでした:bow_tone1:)  
  
# 今回の内容の動機  
  
11月に転職をしてiOS専任のエンジニアになりました。(今のところ)  
  
これまでは人数が少ない受注開発の会社に勤めており、  
1人1人がプロジェクト単位で  
複数のプロジェクトを受け持つというスタイルで開発していました。  
  
そのため  
コードレビューという経験がほとんどなく  
  
インターネットや書籍で調べ  
サンプルを作成して検証して  
これは確からしいことを確認して実装する  
  
というスタイルで開発を進めることがほとんどでした。  
  
今回  
ご縁をいただいて転職し  
ほぼ始めてのチーム開発を経験している中で  
  
**「どうやって他の人にとっても読みやすいコードを書くか」**  
**「どうしたら自分の意図することを他の人に伝えることができるのか」**  
  
といったことに対する意識を持つようになりました。  
  
今回は  
そんな中でコードの書き方やデータの表現方法について  
学んでいることや考えていることについて書きたいと思います。  
  
  
# 名前の付け方  
  
名前の付け方って本当に重要で難しいなと感じることが増えました。  
  
わかりやすい名前をつければ、読む側の負担も減らせますし  
後々改修作業を行う際に何をしたいのかが明確であれば  
思い出すという作業時間が減り、すぐに改修作業が始められます。  
  
(名前とやっていることが一致しているという前提ですが、、、:sweat_smile:)  
  
具体的にこれはわかりやすいなと感じた例をご紹介します。  
  
## 〇〇IfNeeded  
  
初期化処理(initなど)を書き方について出てきた話です。  
  
今までは  
処理の最後にguard文を書き  
処理がなければ処理を終わらせる  
という形で書いていることが多くありました。  
  
  
ところが  
こう書いてしまうと  
  
後からメンテナンスするときに  
どこに処理を追加する必要があるか見通しが悪くなってしまう  
  
という指摘を受け  
確かに意図として十分に伝えられていないなと思いました。  
  
さらに  
if文で値があった時のみ処理を行うという書き方をすると  
もし複数のif文があった場合  
initの中がifだらけで複雑になってしまい  
見づらくなる可能性もあるということも考えられます。  
  
そこで出てきたのが**〇〇IfNeeded**でした。  
  
こうすることで  
必要なときだけ処理を行なっている  
ということが明示できます。  
  
AppleでもlayoutIfNeededなどのメソッドで使っていたり  
他の有名なオープソース化されている多くのアプリでも  
使われているのが確認できました。  
  
  
## How視点とWhat視点  
  
ちょっと広い話になりますが、  
あるデータ構造に名前をつける場合は、  
  
**どう使われるか(How)**  
よりも  
**何をするものなのか(what)**  
を考えつけた方が良いなと感じることが多いです。  
  
これはHowで考えると焦点が具体的過ぎて  
役割が限定的になり過ぎてしまうからであると考えています。  
  
例えば  
ある投票結果を集計するアプリがあるとします。  
  
アプリの要件としては  
投票結果のトップ3を表示する  
というものだとします。  
  
ここで、この集計結果を表現する場合に考えられる型名としては、  
  
```swift

enum VoteType {
    case coffee
    case sport
}

struct VotedItem {
    let type: VoteType
    let name: String
    let number: Int
}

struct TopThreeItemList {
    let items: [VotedItem]
    
    func topThree(of type: VoteType) -> [VotedItem] {
        let items = self.items.filter { $0.type == .sport }
            .sorted(by: { $0.number < $1.number })
            .reversed().prefix(3)
        return Array(items)
    }
}

let items = [
    VotedItem(type: .sport, name: "Baseball", number: 100),
    VotedItem(type: .sport, name: "Football", number: 200),
    VotedItem(type: .sport, name: "Golf", number: 300),
    VotedItem(type: .sport, name: "Basketball", number: 400),
]

let itemList = TopThreeItemList(items: items)
let topThree = itemList.topThree(of: .sport)
```  
みたいなものが考えられます。  
  
ここで仕様追加が入り、ワースト3も出したいとなった場合、どうなりますでしょうか？  
  
まず考えられることとしては、新しい型を定義します。  
  
```swift

struct WorstThreeItemList {
    let items: [VotedItem]
    
    func worstThree(of type: VoteType) -> [VotedItem] {
        let items = self.items.filter { $0.type == .sport }
            .sorted(by: { $0.number < $1.number })
            .prefix(3)
        return Array(items)
    }
}

let worstItemList = WorstThreeItemList(items: items)
let worstThree = worstItemList.worstThree(of: .sport)
```  
  
これはこれで問題なく動きますが、  
なんだか似たようなコードが増えてしまっている気がします。  
仕様追加があった場合はどんどん増えていくことになりそうです。  
  
では、Whatで考えてみるとTopThreeItemListは何をしていますでしょうか？  
  
「投票結果を集めるもの」と考えるとどうでしょうか？  
  
```swift

struct VoteResultAggregator {
    let items: [VotedItem]
    func extract(type:  VoteType, sortedBy: (VotedItem, VotedItem) -> Bool) -> [VotedItem] {
        return self.items.filter { $0.type == type }.sorted(by: sortedBy)
    }
}

let topSportItemList = VoteResultAggregator(items: items)
    .extract(type: .sport, sortedBy: { $0.number > $1.number })
    .prefix(3)

let worstSportItemList = VoteResultAggregator(items: items)
    .extract(type: .sport, sortedBy: { $0.number < $1.number })
    .prefix(3)
```  
  
こうすると、型が増えることもなく色々なパターンに対応できます。  
  
もっと抽象化して「何かを集計するもの」と捉えることもできます。  
  
```swift

struct Aggregator<V> {
    let items: [V]
    func extract(filter: (V) -> Bool, sortedBy: (V, V) -> Bool) -> [V] {
        return self.items.filter(filter).sorted(by: sortedBy)
    }
}
```  
  
このようにWhatで考えた方が、より抽象的に広い範囲で考えることができ、  
多くのケースを網羅できるようになるのではないかと思っています。  
  
もちろんこれは型の使用範囲によると思います。  
例えばある型の中のインナークラスなどローカルなものとして使用する場合は  
逆にHowに焦点を当てた方が意図がわかりやすい場合もあります。  
  
ただ、今回のケースように  
結構広い範囲で使われることが想定される場合や  
変更がありそうな場合は  
Whatで考えていった方が後々楽になることが多かったと思っています。  
  
  
## より良い名前を付けるには？  
  
どうしたら良い名前がつけられるかということを学ぶために  
個人的に参考にしている方法を紹介します。  
  
  
### APIデザインガイドラインを見る  
  
https://swift.org/documentation/api-design-guidelines/  
  
Swiftプログラマとしては  
まず読むべきであると個人的には考えています。  
  
理由として  
  
Appleが提唱しているから正しい  
  
というよりも  
  
エンジニア間で共通の理解を持てる  
  
からです。  
  
ある意味デザインパターンと  
同じような役割をしており  
ある名前を聞けば  
何をするのかがわかるので  
認識合わせの手間や誤解を招くリスクが減らせます。  
  
  
### リーダブルコードを読む  
  
これは非常に有名な本で  
ページ数が少なく  
内容も専門的なことは書かれていないため  
とても読みやすいです。  
  
どうしたら伝わるコードが書けるのか  
といったことが網羅されています。  
  
https://www.oreilly.co.jp/books/9784873115658/  
  
※実はAmazonでは売っていない  
電子書籍版があることを最近知って  
買い直してしまいました:open_mouth:  
  
  
### 有名なオープンソースアプリを眺めてみる  
  
github上で  
- スターが多くついている  
- 現在でも開発が行われている  
ようなアプリのリポジトリを見てみると  
  
使う前置詞が統一されていたり  
直感的にわかりやすい名前をたくさん発見しました。  
  
初めて見ても  
これはこういうことをするんだな  
とわかるような名前を参考にしてみると良いかもしれません。  
  
具体的には下記のようなリポジトリを参照しています。  
  
https://github.com/kickstarter/ios-oss/tree/master/Kickstarter-iOS  
https://github.com/artsy/eidolon  
https://github.com/wordpress-mobile/WordPress-iOS  
https://github.com/wireapp  
https://github.com/mozilla-mobile/firefox-ios  
  
  
# 条件分岐の書き方  
  
これも1人で開発していた時はあまり意識していなかったところで  
  
- 各条件でどういう処理が起きるのかをどうやって伝えるか  
- 将来的なミスを起きないようにできるか  
  
などを考えるようになりました。  
  
そんなきっかけになったいくつかの例を紹介します。  
  
## 早期returnとif else  
  
基本的に早めにreturnできる時は  
returnしようと考えており  
下記のような書き方をしていました。  
  
```swift

// ある条件のtrue, falseで処理が変わる場合

if 条件がfalseの場合 {
    return
}

// もう一つの処理
...

```  
  
  
しかし  
  
早期returnがあると  
選択肢と思わず何かエラーがあったり  
存在するはずの値が存在しないという印象を持つ  
  
という意見がありました。  
  
読む側にとっては  
まずその可能性を考え  
  
次の処理で初めて  
これは条件分岐なんだとわかるので  
  
２段階考える必要が出てくる  
  
とのことでした。  
  
場合によって  
コードの書き方を変える必要があるんだな  
という意識を持つようになりました。  
  
### 追記  
  
コメントでもご指摘いただいたのですが、  
ここに書いてあることの前提条件として  
**副作用が生じない場合**  
という前提が抜けておりました。  
  
副作用が生じる場合は  
いただいたコメントの通り  
条件分岐の中で生じる副作用について  
条件分岐を抜けるまで考慮しておかなければいけない状態になってしまうので、  
そういう場合は早期returnをするようにしています。  
  
私の書き方の配慮不足でした:sweat:申し訳ございません。  
  
## switch文のdefault  
  
いくつかのcaseの場合だけ処理をする場合、  
下記のように書いていました。  
  
```swift

enum Animal {
    case dog, cat, fish, bird
}

func run(animal: Animal) {
    switch animal {
    case .dog:
        print("dog run")
    case .cat:
        print("cat run")
    default:
        print("I can not run!!!!!")
    }
}

func fly(animal: Animal) {
    switch animal {
    case .bird:
        print("bird fly")
    default:
        print("I can not fly!!!!!")
    }
}
```  
  
この書き方ですと  
caseが追加された場合でもdefaultの動作をします。  
  
Xcode上で検索をすれば  
使われている場所はわかりますし  
何もしないcaseを列挙するのは  
正直面倒だなと思っていました。  
  
  
ただ  
これも自分1人ならわかる話ですが  
他の人がcaseを追加した場合などは  
気がつかない可能性があります。  
  
そのため  
下記のように書き換えます。  
  
```swift

func run(animal: Animal) {
    switch animal {
    case .dog:
        print("dog run")
    case .cat:
        print("cat run")
    case .fish, .bird:
        print("I can not run!!!!!")
    }
}

func fly(animal: Animal) {
    switch animal {
    case .bird:
        print("bird fly")
    case .dog, .cat, .fish:
        print("I can not fly!!!!!")
    }
}
```  
  
こうすることで  
新しいケースを追加すると  
コンパイルエラーが起き  
漏れを防ぐことができます。  
  
当初はcaseの列挙が増えるのは  
大変だなと思いましたが  
  
そもそもそういう場合が多いということは  
  
**このenum自体が妥当かどうか**  
  
を検討した方が良いと考えるようになりました。  
  
  
## Swift5の@unknown属性  
  
Swift5からは  
@unknownをdefaultに付けることで  
警告を出してくれるようになるようです。  
  
https://github.com/apple/swift-evolution/blob/master/proposals/0192-non-exhaustive-enums.md  
  
```swift

func fly(animal: Animal) {
    switch animal {
    case .bird:
        print("bird fly")
    @unknown default:
        print("I can not fly!!!!!")
    }
}
```  
  
こうすることでcaseが追加される可能性を示すことができるようになります。  
  
ただ  
  
エラーにはならないですし  
@unknownをつけておらず  
後で新しいcaseが追加されたような場合は  
やはり気づかないかもしれません。  
  
  
## Optionalなenumをswitch文で分岐する  
  
Optionalなenumを扱う場合  
まずはnilチェックをするようにしていました。  
  
```swift

enum Weapon {
    case sword, arrow, bow
}

func attack(weapon: Weapon?) {
    guard let weapon = weapon else {
        print("hand attack")
        return
    }
    switch weapon {
    case .sword:
        print("sword attack")
    case .arrow:
        print("arrow attack")
    case .bow:
        print("bow attack")
    }
}
```  
  
しかし  
こうすると  
選択肢として武器がない場合が  
特別なcaseのように見えてしまいます。  
  
ここでコンパイラの力を借りようと思います。  
  
```swift

func attack(weapon: Weapon?) {

    switch weapon {
    case .sword?:
        print("sword attack")
    case .arrow?:
        print("arrow attack")
    case .bow?:
        print("bow attack")
    case nil:
        print("hand attack")
    }
}
```  
  
Optionalなenumの場合  
caseの後ろに?をつけます。  
またcase nil(もしくは.none)も必要です。  
  
見づらいようにも思えますが  
ない場合はコンパイルエラーになるため  
明示的に必要なことに気がつけます。  
  
### 追記  
  
コメントでご指摘いただいたのですが  
そもそ組み合わせを網羅できるはずのenumが  
Optionalになっていること自体が変なのかもしれません。  
  
上記の例でも  
武器が存在しない場合(素手という武器)という  
選択肢を追加することで  
Optionalである必要がなくなります。  
  
私の中で存在しないという状態を示すのがnilである  
という前提ができてしまっていることに  
気がつくことができました:cop_tone1:  
  
## Optionalを含んだtupleをswitch文で分岐する  
  
例えば  
Memberという型が  
Optionalなfirstとsecondというプロパティ  
を持っているとします。  
  
  
```swift

struct Member {
    let id: Int
    var first: String?
    var second: String?    
}
```  
  
ここにdisplayNameという  
Computedプロパティを追加します。  
  
firstとsecondが存在する場合に  
⭐️を間に差し込む仕様だとします。  
  
```swift

struct Member {
    ...
    var displayName: String {
        var name: String = ""
        if let first = first {
            name += first
        }
        
        if let second = second {
            
            if first != nil {
                name += "⭐️"
            }
            name += "\(second)"
        }
        
        return name
    }
}

let member = MemberName(id: 1, first: "スター", second: "です")
print(member.displayName) // スター⭐️です

let member2 = MemberName(id: 1, first: "スター", second: nil)
print(member2.displayName) // スター

let member3 = MemberName(id: 1, first: nil, second: "です")
print(member3.displayName) // です

let member4 = MemberName(id: 1, first: nil, second: nil)
print(member4.displayName) //

```  
  
どういう条件で何が起きるのか  
ちょっと見づらいですね。  
  
こうしたらどうでしょうか？  
  
```swift

struct Member {
    ...
    var displayName: String {
        var stringArray: [String] = []
        if let first = first {
            stringArray.append(first)
        }
        
        if let second = second {
            stringArray.append(second)
        }
        
        return stringArray.joined(separator: "⭐️")
    }
}
let member = MemberName(id: 1, first: "スター", second: "です")
print(member.displayName) // スター⭐️です

let member2 = MemberName(id: 1, first: "スター", second: nil)
print(member2.displayName) // スター

let member3 = MemberName(id: 1, first: nil, second: "です")
print(member3.displayName) // です

let member4 = MemberName(id: 1, first: nil, second: nil)
print(member4.displayName) //
```  
  
少し見やすくなりましたが  
これでもちょっとわかりづらいような気がします。  
  
そこで  
switch文を活用します。  
  
```swift

struct Member {
    ...
    var displayName: String {
        switch (first, second) {
        case (let first?, let second?):
            return "\(first)⭐️\(second)"
        case (let first?, nil):
            return "\(first)"
        case (nil, let second?):
            return "\(second)"
        case (nil, nil):
            return ""
        }
    }
}
let member = MemberName(id: 1, first: "スター", second: "です")
print(member.displayName) // スター⭐️です

let member2 = MemberName(id: 1, first: "スター", second: nil)
print(member2.displayName) // スター

let member3 = MemberName(id: 1, first: nil, second: "です")
print(member3.displayName) // です

let member4 = MemberName(id: 1, first: nil, second: nil)
print(member4.displayName) //
```  
  
こうすると  
caseで条件と出力内容が列挙できるので  
わかりやすくなった気がします。  
  
  
## 空文字を返すか、Optionalを返すか  
  
上記の例ですと  
firstとsecondいずれもnilの場合  
空文字を返しています。  
  
Optionalを返すと  
毎回のnilチェックや？？で  
値の設定をしなければいけない点が面倒なため  
空文字を返していることって  
意外とあるのではないでしょうか？  
  
しかし  
状況によっては  
思わぬ不具合に繋がる可能性もあります。  
  
例えば  
ユーザに何かの招待状を送るとします。  
  
  
```swift

func sendInvitation(to name: String) {
    print("\(name)様、せひお越しください!!!")
}
```  
  
これにfirst、secondがnilのユーザに招待状を送った場合  
  
```swift

func sendInvitation(to name: String) {
    print("\(name)様、せひお越しください!!!")
}

let member4 = MemberName(id: 1, first: nil, second: nil)
sendInvitation(to: member4.displayName) 
```  
  
```
// 様、せひお越しください!!!
```  
  
となります。  
  
これが意図した動作だとしたら問題ないのですが  
気がつかないで誰かが埋め込んでしまったとしたら  
実行されるまで気がつかない可能性があります。  
  
これをdisplayNameをOptionalにしていた場合  
コンパイルがOptionalチェックを強制してくるため  
少なくともnilになる可能性がある  
ということに気がつけます。  
  
Optionalにする必要があるのが限定的であるならば  
別の役割を持つ別のプロパティとするのもありなのかもしれません。  
  
  
# データの表現方法  
  
あるデータを型で表現する場合  
特に意識しなければ  
まずはstructで考えるようにしていました。  
  
しかし  
他の表現方法の可能性を検討することが  
増えていると感じています。  
  
特に感じるのは  
enumを活用する場合が増えています。  
  
いくつか例をご紹介します。  
  
  
## structをenumに変換する  
  
会員制サイトのUserを示す型があるとします。  
そしてUserには4種類の会員が存在します。  
  
```swift

enum UserType {
    case normal
    case gold
    case silver
    case bronze
}
```  
  
Userをstructで表す場合、下記のようになります。  
  
```swift

struct User {
    let registeredDate: Date
    let needDiscount: Bool
    let userType: UserType
}
```  
  
ここで作成できるUserの組み合わせは非常にたくさんあります。  
  
四則演算で表すと  
  
**Bool(2) x Date(たくさん) x UserType(4)**  
  
になります。  
  
  
次にenumで表現してみます。  
  
```swift

enum User {
    case normal(registeredDate: Date, needDiscount: Bool)
    case gold(registeredDate: Date, needDiscount: Bool)
    case silver(registeredDate: Date, needDiscount: Bool)
    case bronze(registeredDate: Date, needDiscount: Bool)
}
```  
  
四則演算で表すと  
  
**Bool(2) x Date(たくさん)   
+ Bool(2) x Date(たくさん)  
+ Bool(2) x Date(たくさん)   
+ Bool(2) x Date(たくさん)**  
  
になります。  
  
つまり  
  
**(Bool(2) x Date(たくさん)) x UserType(4)**  
  
となり  
  
実はここで作成できるUserの組み合わせは  
structの場合と同じです。  
  
しかし  
enumだと全体として考える必要のあるパターンは  
4つに限定されました。  
  
enumの方が同時に扱えるパターンが一つに決められ  
考える必要のある数も絞れるため  
個人的にはenumを使った方がより良いのではないかと思っています。  
  
会員の種類別で処理が分岐するような場合  
  
structの場合だと  
**user.registeredDate**  
など**user.**を付ける必要がありますが、  
  
enumの場合  
Userの型自体で分岐ができ  
より扱いやすくなります。  
  
  
## 間違いが起きそうな文字列の扱いをenumで吸収する  
  
enumを活用することで  
文字列をそのまま扱うことによる間違い  
のリスクを軽減させることができます。  
  
例えば  
ファイルをアップロードする画面があり  
ユーザは様々な拡張子のファイルを  
アップロードすることができるとします。  
  
その際にアップロードされた拡張子によって  
表示するメッセージを変えるような処理があるとします。  
  
※本来はメタデータのチェックなど必要ですが  
今回の趣旨から外れるため割愛させていただきます。  
  
```swift

func showMessage(for fileExtension: String) {
    switch fileExtension {
    case "jpg":
        print("This is jpg")
    case "png":
        print("This is png")
    case "gif":
        print("This is gif")
    case "bmp":
        print("This is bmp")
    default:
        print("Invalid!!!")
    }
}

showMessage(for: "jpg") // This is jpg

```  
  
これは正常に動きます。  
  
しかし  
いくつかのリスクを含んでいます。  
  
jpgは**jpeg**や**JPEG**という場合もありえます。  
  
このような場合  
  
```
This is jpg
```  
  
と出力されて欲しいのに  
  
```
Invalid！！！
```  
  
と出力されます。  
  
また  
同じ文字列をメソッドの引数として  
繰り返し使用するような場合  
  
**全てのメソッドで文字列の妥当性をチェックをする**  
  
または、  
  
**ずっと間違えた状態で処理が継続する**  
  
といったことが起きます。  
  
また  
新しい拡張子が追加されたけれども  
あるメソッド処理に処理を追加し忘れた場合  
  
コンパイルは問題なく通ってしまい  
実行時の動作は意図したものになりません。  
  
  
これを解消するために  
enumを活用します。  
  
```swift

enum ImageExtension: String {
    case jpg
    case png
    case gif
    case bmp
    
    init?(rawValue: String) {
        switch rawValue.lowercased() {
        case "jpg", "jpeg":
            self = .jpg
        case "png":
            self = .png
        case "gif":
            self = .gif
        case "bmp":
            self = .bmp
        default:
            return nil
        }
    }
}

func showMessage(for imageExtension: ImageExtension) {
    switch imageExtension {
    case .jpg:
        print("This is jpg")
    case .png:
        print("This is png")
    case .gif:
        print("This is gif")
    case .bmp:
        print("This is bmp")
    }
}
```  
  
ポイントとしては  
**failable initializer**  
を活用している点です。  
  
まずlowercasedを使うことで  
大文字小文字の区別をなくします。  
  
その後  
複数の文字列がマッチする可能性のある拡張子は  
複数の文字列のcaseを受け取れるようにしています。  
  
さらに  
どのケースにも当てはまらない場合は  
nilを返します。  
  
こうすることで  
まず拡張子の文字列が妥当かどうかのチェックをしたあとに  
処理を続けることができます。  
  
```swift

guard let imageExtension = ImageExtension else {
    // エラー処理
}

showMessage(for: imageExtension) // This is jpg
```  
  
こうすると  
文字列で新しいを追加したい場合も  
まずenumにcaseを追加することで  
自動でコンパイルエラーになってくれます。  
  
もちろんケース自体を追加し忘れた場合はどうにもなりませんが:innocent:  
  
  
## OptionalなBoolをenumとして扱う  
  
Boolといえば  
**true**  
**false**  
の２択を表す型ですが、  
  
Swiftの場合、  
  
```swift

Bool?
```  
とすると、  
**true**  
**false**  
**nil**  
の３パターンの可能性があります。  
  
例えば  
APIの戻り値で下記のような値が返ってくるとします。  
  
```swift

let returned: [String: Any] = [
    "autoLogin": false, "UserId": 1, "canUseSpecial": true]
```  
  
この中のをcanUseSpecial取り出すとBool？になります。  
  
```swift

let canUseSpecial = returned["canUseSpecial"] as? Bool
print(canUseSpecial) // Optional(true)
```  
  
Optionalなまま扱うのはちょっと気持ち悪いですね。  
  
ではどう対処するか？  
  
例えば  
  
**nilはfalse**  
  
として扱うとみなして  
default値を設定してみるとどうでしょうか？  
  
```swift

let canUseSpecial = returned["canUseSpecial"] ?? false
print(canUseSpecial) // false
```  
  
これでBoolとして扱えるようになりました。  
  
しかし  
これは必ず意図した動作になりますでしょうか？  
  
例えば、上記の例で  
  
**trueもしくは未設定の場合、設定ページを開く**  
  
という動作をさせたいとしたら  
どうなりますでしょうか？  
  
``` swift

let canUseSpecial = returned["canUseSpecial"] ?? false

...

if canUserSpecial {
   goToSettingPage()
}
```  
  
この場合、  
意図した動作とは逆になってしまいます。  
  
**一概にnilの場合はfalseとできない可能性がある**  
  
ということです。  
  
  
ではどうするか？  
  
**3つの状態を持つenum**にしてみるのはどうでしょうか？  
  
```Swift

enum UseSpecial: RawRepresentable {
    case enabled
    case disabled
    case notSet
    
    init(rawValue: Bool?) {
        switch rawValue {
        case true?:
            self = .enabled
        case false?:
            self = .disabled
        default:
            self = .notSet
        }
    }
    
    var rawValue: Bool? {
        switch self {
        case .enabled:
            return true
        case .disabled:
            return false
        case .notSet:
            return nil
        }
    }    
}


let returned: [String: Any] = ["autoLogin": false, "UserId": 1]
let canUseSpecial = returned["canUseSpecial"] as? Bool
let state = UseSpecial(rawValue: canUseSpecial)
print(state) // notSet

```  
  
RawRepresentableに準拠することで  
Bool？からの変換が可能になっています。  
  
こうしておくと  
  
**ユーザがどういう設定をしているのか(またはしていないのか)**  
  
がわかり  
より明確にユーザの状態を  
把握することができるようになります。  
  
  
## Protocolである必要性を考える  
  
WWDC2015で  
AppleがProtocol Oriented Programmingを提唱して以来  
Protocolを中心にコードを組み立てる人が  
多くなったのではないでしょうか？  
  
Protocolのメリットとして  
- 様々な型を同じように扱うことができる  
- デフォルト実装で同じ処理書く必要がなくなる  
など多くの恩恵を受けることができます。  
  
しかし  
Protocolが適さない場合もあるような気がしています。  
  
  
例えば  
以下の２つはいかがでしょうか？  
  
### *associatedtypeやSelfを使ったProtocolを型として扱う*  
  
公園で遊べるものを表すProtocolと  
それに準拠した具体的な遊び方を表すstructがあるとします。  
  
```swift

protocol ParkPlayable: Hashable { func play() }

struct Baseball: ParkPlayable {
    func play() { print("Enjoy Baseball!") }
}

struct Football: ParkPlayable {
    func play() { print("Enjoy Football!") }
}
```  
  
各遊び方で何人まで遊べるのかを知りたいので  
ParkPlayableをキーにDictionaryで保持　しようとします。  
  
```swift

// error: using 'Playable' as a concrete type conforming to protocol 'Hashable' is not supported
var numbers: [Playable: Int] = [:]

```  
  
これはエラーです。  
  
理由はHashableがEquatableを継承しており、  
EqatableでSelfが使用されているため  
具体的な型として使えないからです。  
  
ではどうすれば良いか？  
  
一つの手段として**TypeEraser**としてAnyParkPlayable型を作成します。  
  
※TypeEraserことはこちらに大変詳しくまとめられておりますので  
リンク先の紹介のみとして割愛させて頂きます。  
https://qiita.com/omochimetaru/items/5d26b95eb21e022106f0  
  
  
```swift

struct AnyParkPlayable: ParkPlayable {
    private let _play: () -> Void
    private let _hashable: AnyHashable

    init<S: ParkPlayable>(_ sport: S) {
        self._play = sport.play
        self._hashable = AnyHashable(sport)
    }

    func play() {
        self._play()
    }
}

extension AnyParkPlayable: Hashable {
    func hash(into hasher: inout Hasher) {
        _hashable.hash(into: &hasher)
    }
    static func ==(lhs: AnyParkPlayable, rhs: AnyParkPlayable) -> Bool {
        return lhs._hashable == rhs._hashable
    }
}

// OK
var numbers: [AnyParkPlayable: Int] = [
    AnyParkPlayable(Baseball()): 100,
    AnyParkPlayable(Football()): 200
]
```  
  
エラーはなくなりました。  
  
しかし  
これを表現するために  
  
- AnyParkPlayableクラスの作成  
- AnyParkPlayableをHashableに準拠させるための実装  
   
が必要になりました。  
  
さらに  
処理が複雑で何をしているのかが  
パッと見てわかりづらくなっている  
ようにも思えます。  
  
  
本当にProtocolを用いる必要はあるのでしょうか？  
  
例えば  
enumを使ってみます。  
  
```swift

struct Baseball: Hashable {
    func play() { print("Enjoy Baseball!") }
}

struct Football: Hashable {
    func play() { print("Enjoy Football!") }
}

enum ParkPlay: Hashable {
    case baseball(Baseball)
    case football(Football)
    
    func play() {
        switch self {
        case .baseball(let baseball):
            baseball.play()
        case .football(let football):
            football.play()
        }
    }
}

var numbers: [ParkPlay: Int] = [
    .baseball(Baseball()): 100,
    .football(Football()): 200,
]

print(numbers.values)
```  
  
たったこれだけで済みます。  
  
caseがたくさんある場合や  
デフォルト実装をもっと活用したいといった場合は  
Protocolを活用した方が良いことが多くなってくると思います。  
  
しかし  
caseが限られていて  
型としてまとめて使用したい場合などでは  
enumの方が簡単に使える場合もあるのではないかと  
今回のような場合を考えると感じられます。  
  
  
### *ある一つの処理を複数のタイプで使用する*  
  
バリデーションチェックをすることを表すProtocolがあるとします。  
  
```swift

protocol Validatable {
    associatedtype Value
    func validate(_ value: Value) -> Bool
}

struct MinLength: Validatable {
    let minLength: Int
    func validate(_ value: String) -> Bool {
        return value.count >= minLength
    }
}

let min3Length = MinLength(minLength: 3)
min3Length.validate("aaa") // true
min3Length.validate("aa") // false
```  
  
これでも十分に動きますが、  
一つ一つのチェックに対して毎回型を宣言する必要が出てきます。  
ちょっと面倒な気がしますね。  
  
例えば  
genericな型を持ったstructにしてみるとどうでしょうか？  
  
```swift

struct Validator<Value> {
    let validate: (Value) -> Bool
}

let min3Length = Validator<String> { string in
    return string.count >= 3
}

min3Length.validate("aaa") // true
min3Length.validate("aa") // false

```  
  
genericなValueを使用しているため  
どんな型に対しても使用できます。  
  
また  
型を宣言する必要がないため  
こちらの方が簡単に生成できるように感じられます。  
  
さらに以下のメソッドを追加してみます。  
  
```swift

extension Validator {
    func combine(_ other: Validator) -> Validator<Value> {
        return Validator { value in
            return self.validate(value) && other.validate(value)
        }
    }
}
```  
  
こうすることでチェックを組み合わせることができ、  
より高度なチェックも簡単することができます。  
  
```swift

let min5Length = Validator<String> { string in
    return string.count >= 5
}
let notEmpty = Validator<String> { string in
    return !string.isEmpty
}

let nonEmptyAndMin5 = notEmpty.combine(min5Length)

nonEmptyAndMin5.validate("") // true
nonEmptyAndMin5.validate("aaaaaa") // false
```  
  
これをProtocolで実現しようとすると  
新しい型が必要になり  
いわゆるボイラープレートが増えていきます。  
  
  
Protocolは大変便利で使いどころは非常に多くあるとは思いますが  
一概に  
Protocolを使うのがベスト  
と考えるのではなく  
他の選択肢の可能性にも目を向ける必要があるな  
と思うことが増えました。  
  
  
# エラー処理  
  
## エラーの種類  
  
エラーは大きく3つに分かれています。  
  
- プログラミング上のエラー(arrayのout　of　boundsや０で割り算をするなど)  
- ユーザが起こすエラー (間違った入力や設定ミスなど)  
- システムが起こす実行時のエラー　(容量上限でファイルが作成できない、ネットワークに繋がらないなど)  
  
この中で最初の2つは  
下記のような方法で防げる可能性が高まります。  
  
- プログラミング上のエラー　  
-> ユニットテストやassertを書いて開発中にミスに気がつけるようにする  
  
- ユーザが起こすエラー (間違った入力や設定ミスなど)　  
-> より意図が伝わりやすくするようにUIを変える。説明を加える  
  
しかし  
システムが起こす実行時のエラーは  
その時の状況によって発生するかしないかもわからないため  
適切にエラーに対処する必要があります。  
  
## Swiftのエラー処理方法  
  
Swiftではエラーの処理方法が4つに分かれていると書かれています。  
https://docs.swift.org/swift-book/LanguageGuide/ErrorHandling.html  
  
- エラーを呼び出し側に伝播させる  
- do-catch文  
- Optionalとして扱う  
- エラーが起きないことをassertで宣言する  
  
また  
こちらの記事などに詳しくまとめられています。  
https://qiita.com/koher/items/a7a12e7e18d2bb7d8c77  
  
  
プロジェクトによって  
エラーの処理方法は異なると思いますが  
対処方法として検討できるものをいくつかご紹介します。  
  
  
## Quick Help用のドキュメントを作成する  
  
Swiftのthrowsは  
その関数やメソッドが  
どんなErrorを投げるのかを明示できません(Swift4.2時点)  
  
そのため  
関数やメソッドを追っていく必要があります。  
  
そういった時に  
　**alt + クリック**  
で表示される**Quick Help**に  
throwする可能性のあるエラーの内容が出てくると  
便利です。  
  
関数やメソッドにカーソルを合わせて   
  
**cmd + alt + /　**  
  
でテンプレートが生成されるので  
そこにエラーの種類を記述するだけです。  
  
記載すると下記のようにQuick Helpにエラーが出てきます。  
  
![スクリーンショット 2018-12-02 9.38.17 2.png](/image/dae1b507-b9ee-f34c-310f-d10f15fa8e33.png)  
  
  
## throwsを活用する  
  
Optionalを返す方法は楽ですが、  
個人的にはthrowsを使った方が良いと考えています。  
  
なぜならば  
Optionalはエラーに関する情報を提供しないため  
デバッグ時など原因を探すのに苦労をするケースがあるからです。  
  
さらに  
Optionalで良い、となった場合でも  
**try?**を使うことで  
呼び出し側でOptionalとthrowsの両方の処理の仕方に対応できます。  
  
  
色々な例を考えてみたのですが  
上記で記載したValidatorと同じような例で  
throwsを活用した非常にわかりやすい記事があり  
これは参考にしたいと思ったので紹介させて頂きます。  
  
[Using errors as control flow in Swift](https://www.swiftbysundell.com/posts/using-errors-as-control-flow-in-swift?utm_campaign=Swift%20Weekly&utm_medium=Swift%20Weekly%20Newsletter%20Issue%20139&utm_source=Swift%20Weekly)  
  
上記で記載したValidatorの場合、結果がtrueかfalseしかわからず  
どの項目がエラーになっているのかなどの詳細情報がわかりません。  
  
そこで  
throwsを使った形に変換してみます。  
  
```swift

struct Validator<Value> {
    let closure: (Value) throws -> Void
}
```  
※関数名と衝突している関係上、変数名がclosureになっています。  
  
さらに  
このままですと無数のErrorに準拠した型を作成することになるため  
共通のエラー用の型を定義します。  
  
```swift

struct ValidationError: LocalizedError {
    let message: String
    var errorDescription: String? { return message }
}
```  
※LocalizedErrorに関しては後ほど紹介しておりますが  
ユーザに表示するためのエラーメッセージを定義します。  
  
次に  
これを利用した関数を定義します。  
  
```swift

func validate(
    _ condition: @autoclosure () -> Bool,
    errorMessage messageExpression: @autoclosure () -> String
    ) throws {
    guard condition() else {
        let message = messageExpression()
        throw ValidationError(message: message)
    }
}
```  
  
以下のように利用します。  
  
```swift

let userNameValidator = Validator<String> { value in
    if value.count < 5 {
        throw ValidationError(message: "５文字以上で入力してください")
    }
}

do {
    try userNameValidator.closure("me")
} catch {
    print(error.localizedDescription) // ５文字以上で入力してください
```  
  
また  
もっと簡単に利用するために下記のような方法も紹介されていました。  
  
```swift

func validate<T>(_ value: T,
                 using validator: Validator<T>) throws {
    try validator.closure(value)
}
```  
  
こちらは以下のように利用します。  
  
```swift

extension Validator where Value == String {
    static var userNumber: Validator {
        return Validator { string in
            try validate(
                !string.isEmpty,
                errorMessage: "文字を入力してください"
            )
            try validate(
                Int(string) != nil,
                errorMessage: "数字のみ入力してください"
            )
        }
    }
}

func showMessageIfValid(with input: String) throws {
    try validate(input, using: .userNumber)
    print("validation ok")
}

do {
    try showMessageIfValid(with: "") // 文字を入力してください
} catch {
    print(error.localizedDescription)
}

do {
    try showMessageIfValid(with: "aaaaaaa") // 数字のみ入力してください
} catch {
    print(error.localizedDescription)
}
```  
  
ちょっと横道に逸れてしまいましたが  
このようにすることで  
catchしてエラー情報を取得することができます。  
  
また  
Validationのチェックも汎用的にできるので良いなと感じました。  
  
  
## エラーになった際に変更を元に戻す方法を考える  
  
トランザクションのコールバックのように  
関数やメソッドで何かの状態を変更していた場合  
エラー発生時にはその状態を元に戻す必要が出てきます。  
  
これに対処する方法としてはいくつか考えられます。  
  
  
### *throwsする関数、メソッドでそもそも状態を変更しない*  
  
矛盾しているようですが  
そもそも状態を変更しなければ  
戻す必要もなくなります。  
  
可能かどうか検討してみる価値はあると思います。  
  
### *一時変数に変更を加えていく*  
  
こちらもそもそも状態を変更しないに近いですが  
  
変更したい値をコピーした値に対して処理を加え  
正常時は一時変数を返し  
エラー時は元の値を返す。  
  
そうすれば  
エラー時に何か特別な処理をする必要もなくなります。  
  
### *deferの中に後始末の処理を書く*  
  
deferを使うことで  
関数やメソッドのどの時点でエラー投げられたとしても  
最終的な後片付けをすることができます。  
  
```swift

func throwError() throws {
    defer { print("後始末します")}
    throw UnbelievableError.unbelievable
}
throwError() // 後始末します
```  
  
ただし  
下記の場合はdeferが出力されないので  
defer文は関数やメソッドの上の方に書くのがよいかと思います。  
  
```swift

func throwError() throws {
    throw UnbelievableError.unbelievable
    defer { print("後始末します")}
}
throwError() 何も出ない
```  
  
さらに  
気をつけたい点として  
defer文で後始末をする際に  
元の状態とは違った状態にならないようにする点です。  
  
どこかに元の状態を保持しておき  
きちんと元に戻せる状態にしておけるように方法も  
検討してみる必要がありそうです。  
  
このように考えていくと  
そもそも元の値は変更しないようにする  
という方を探す方がより安全な気がします。  
  
  
## LocalizedErrorに準拠させる  
  
上記でも一部出てきましたが  
プログラマが確認するエラーのメッセージと  
ユーザに表示するエラーメッセージは異なることがよくあります。  
  
そんな時に **LocalizedError** プロトコルに準拠させることで  
ユーザに表示するエラーメッセージを指定することができます。  
https://developer.apple.com/documentation/foundation/localizederror  
  
4つのプロパティを持っています。  
  
- failureReason   
- recoverySuggestion   
- errorDescription   
- localized String  
  
これらは全てOptionalでデフォルト値を持っているため  
必要なプロパティだけ定義するだけで済みます。  
  
この中でもerrorDescriptionを設定することで  
ErrorのlocalizedDescriptionプロパティから使用可能になります。  
  
https://developer.apple.com/documentation/swift/error/2292912-localizeddescription  
  
```swift

enum SurprisingError: Error, LocalizedError {
    case fired
    case inTheWater
    
    var errorDescription: String? {
        switch self {
        case .fired:
            return "バッテリーから火が出た"
        case .inTheWater:
            return "水没！！！"
        }
    }
}

func throwSurprisingError() throws {
    throw SurprisingError.fired
}

do {
    try throwSurprisingError()
} catch {
    print(error.localizedDescription) // バッテリーから火が出た
}
```  
  
今回は割愛しましたが  
NSLocalizedStringを使用することで  
ユーザのlocaleに合わせたメッセージを出力することもできます。  
https://developer.apple.com/documentation/foundation/nslocalizedstring  
  
  
## Errorを処理する場所(do-catchする位置)を統一する  
  
当たり前のことなのかもしれませんが  
エラーをキャッチする位置を決めておかないと  
エラー処理のコードが色々な箇所に散ってしまいます。  
  
色々なソースを見てみると  
エラー処理は呼び出し側でコントロールしたいことが多いため  
呼び出し側に戻るまではthrowし  
do-catch文で共通のエラーハンドラーに処理をさせる  
といったパターンが多く見られます。  
  
  
  
## 余談: 共通の理解があればFunctional Programmingを取り入れてみる  
  
個人的には非常に興味があるのですが  
なかなか導入するのは難しいとも感じているため  
最後に余談として書かせていただきました。  
  
  
structやenumを使用してドメインを表現しようとすると  
ネストが深くなることが多くなります。  
  
そうすると  
中の値を取得したり  
ある値だけ更新するといったことが面倒です。  
  
Functional Programmingを活用すると  
そういった問題の複雑さを軽減できる場合があります。  
  
具体的には  
あるデータ構造から特定の値を取り出したり  
値を更新したりするデータ構造を作成することで  
「どういう役割を果たすのか」  
「何に対して何を行なっているのか」  
を型として表現できます。  
  
下記の発表の内容がとてもわかりやすく  
Functional Programmingのメリットが感じられました。  
https://www.youtube.com/watch?v=ki2WSw2WXV4  
  
この中で3つのデータ構造が紹介されていますが  
一部簡単にご紹介させて頂きます。  
  
### Lens  
  
下記のような構造になっています。  
  
```swift

struct Lens<Root, Value> {
    let view: (Root) -> Value
    let update: (Value, Root) -> Root
}
```  
  
2つの関数を持っています。  
  
Rootがデータ全体を表し  
Valueはその中のある値です。  
  
viewはデータ全体からある特定の値を取り出す関数で  
updateはある特定の新しい値と  
前の状態のデータ全体を引数として  
新しいデータ全体を戻り値として返却します。  
  
SwiftでLensを作成する場合  
Swift4から使用できるKeyPathを使うことによって  
簡単に表現することができます。  
https://developer.apple.com/documentation/swift/keypath  
  
  
```swift

func makeLens<Root, Value>(_ wkp: WritableKeyPath<Root, Value>) -> Lens<Root, Value> { |
    return Lens<Root, Value>(
        view: { root in root[keyPath: wkp] },
        update: { newValue, root in
            var m_root = root
            m_root[keyPath: wkp] = newValue
            return m_root
    })
}
```  
  
また2つのLensを組み合わせるメソッドも宣言します。  
  
```swift

func zip<Root, Value1, Value2>(_ lens1: Lens<Root, Value1>, _ lens2: Lens<Root, Value2>) -> Lens<Root, (Value1, Value2)> {
    return Lens<Root, (Value1, Value2)>(
        view: { root in
            (lens1.view(root), lens2.view(root))
    },
        update: { tuple, root in
            lens2.update(tuple.1, lens1.update(tuple.0, root))
    })
}
```  
  
次に  
ある特定の値を修正したLensを作成するメソッドを宣言します。  
  
```swift

extension Lens {
    func modify (_ transformValue: @escaping (Value) -> Value) -> (Root) -> Root { |
        return { root in
            self.update(
                transformValue(self.view(root)),
                root)
        }
    }
}
```  
  
簡単な具体例を示すと  
例えばユーザが入力した名前を表現する  
FullNameを持ったUserInputがあります。  
スペースの入力は可能ですが  
最終的には前後のスペースはなくして扱いたい場合  
下記のようの処理できます。  
  
※WritableKeyPathを使うので  
structのプロパティはvarで宣言します。  
  
しかし  
structはValue Semanticsなので  
参照による思わぬ値の変更といった影響を気にする必要はありません。  
  
```swift

struct UserInput {
    var name: FullName

    static func lens<Value>(_ wkp: WritableKeyPath<UserInput, Value>) -> Lens<UserInput, Value> {
        return makeLens(wkp)
    }
}

struct FullName {
    var first: String
    var family: String
    
    static func lens<Value>(_ wkp: WritableKeyPath<FullName, Value>) -> Lens<FullName, Value> {
        return makeLens(wkp)
    }    
}

let nameLens = zip(
    UserInput.lens(\.name.first),
    UserInput.lens(\.name.family)
)

let trimmedName =
    nameLens.modify {
        (
            $0.0.trimmingCharacters(in: CharacterSet(charactersIn: " ")),
            $0.1.trimmingCharacters(in: CharacterSet(charactersIn: " "))
        )
    }


let input = UserInput(name: FullName(first: "   first   ", family: "    family"))
let trimmedInput = trimmedName(input)

print(trimmedInput.name) // FullName(first: "first", family: "family")

```  
  
今回の例は  
簡単なものなので恩恵をあまり感じられないかもしれませんが  
もっとデータ構造が複雑になった場合でも  
同じような形で処理することができるため  
可読性は向上するのではないかと考えられます。  
  
上記で紹介した発表では  
PrismやAffineといった他の構造も紹介されていますので  
ご興味のある方はぜひ見てみてください。  
  
  
ただし  
これには前提として　  
Functional Programmingに対する  
チーム内での理解が必要です。  
  
自分がわかっていても  
周りがわからなければ可読性は下がりますし  
コード量が増えて  
余計に面倒になるということは大いに考えられます。  
  
  
## 最後に  
  
コードの書き方や表現方法について書かせて頂きました。  
  
もちろん今回のことのみならず  
もっと考える必要がある項目は限りなくあると思います。  
  
また「これが正しい」というものはなく  
正しいと思う判断をしても  
  
「あっ、しまった。こうすればよかった。」  
  
と後で思い直すことも多いと感じています。  
  
今回自分なりに考えていることを色々と書いてきましたが  
  
最終的には  
#### チーム内での共通認識とコードの統一性  
が大事なんだなと思います。  
  
目的は  
**いかに開発メンバーや将来の自分に意図をわかりやすく伝えられるか**  
であり  
これを考えないと  
かえってプロジェクトを複雑にしてしまう可能性もあります。  
  
ここは気をつけなければいけないところだと強く感じています。  
(特に私の場合は考えすぎて失敗することがよくあるので、、、:sweat:)  
  
チーム開発という新しい経験を通じで  
今まで意識してこなかったことに目を向けるようになり  
日々学ぶ機会を得られたことが嬉しくも楽しくもあり、  
今の環境にいられることに大変感謝しています:star2:  
  
正解のないところではありますが  
今後も日々学び、考え続けていきたいと思います！  
  
**「こっちの方が良い」**  
**「こんな書き方もある」**  
  
などのご意見ございましたらぜひ教えてください:smiley:  
