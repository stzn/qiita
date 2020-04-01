ビジネスには多くのルール(仕様)があり  
プログラムにもそれを反映させる必要があります。  
  
複雑な仕様があると  
明確に何をしているのかが不明瞭になったり  
思わぬミスをしてバグを生み出してしまう可能性があります。  
  
その際に  
型システムを活用することで  
仕様をコードにより明確に反映させたり  
バグを生むリスクを減らすことができます。  
  
今回はどうやって型システムを活用するかについて  
具体例から考えてみたいと思います。  
  
# 環境  
  
言語: Swift5.0  
IDE: Xcode10.2  
  
# 実装  
  
今回は型を活用するということに焦点を当てているため  
実装の詳細はかなり簡略にしております:bow_tone1:  
  
## 内容  
  
あるサービスにユーザがいて  
承認済みのメールアドレスと  
未承認のメールアドレスを持つユーザがいるとします。  
  
※今回は簡略化のためメールアドレスにのみ着目しています。  
  
運営側からユーザにメールを送りたいと考えおり  
  
- 未承認のメールアドレスにはキャンペーンのメールを送る  
- 承認済みのメールアドレスには秘密のメールを送る  
  
という仕様があるとします。  
  
  
## 最初の実装  
  
```swift

struct UserMail {
    let address: EmailAddress
    let isApproved: Bool
}

struct EmailAddress {
    let address: String
    init(_ address: String) {
        self.address = address
    }
}

struct MailCheckService {
    func isApproved (_ user: UserMail) -> Bool {
        return user.isApproved
    }
}

// 未承認のメールにのみキャンペーンメールを送りたい
func sendCampaignMail(user: UserMail, isApproved: (UserMail) -> Bool) {
    if !isApproved(user) {
        print("confirm mail send to \(user.address)!!")
    }
}

// 承認済みメールにのみ秘密のメールを送りたい
func sendSecretMail(user: UserMail, isApproved: (UserMail) -> Bool) {
    if isApproved(user) {
        print("secret mail send to \(user.address)!!")
    }
}


func send() {

    let unApprovedUserMail = UserMail(address: EmailAddress("unApproved@hoge.com"), isApproved: false)
    let approvedUserMail = UserMail(address: EmailAddress("approved@hoge.com"), isApproved: true)
    
    let service = MailCheckService()

    // print("これは送る🙆🏻‍♀️")
    sendCampaignMail(user: unApprovedUserMail, isApproved: service.isApproved)

    // print("これは送らない🙅🏻‍♀️")
    sendCampaignMail(user: approvedUserMail, isApproved: service.isApproved)
    
    // print("これは送る🙆🏻‍♀️")
    sendSecretMail(user: approvedUserMail, isApproved: service.isApproved)

    // print("これは送らない🙅🏻‍♀️")
    sendSecretMail(user: unApprovedUserMail, isApproved: service.isApproved)
}
```  
  
この場合では  
メールアドレスの承認・未承認を```isApproved```というプロパティで判定をしています。  
  
これでも正しく実装できていれば問題ありません。  
  
  
しかしいくつか不安な点があります。  
  
  
- `isApproved`では承認済みのメールアドレスと未承認のメールアドレスが存在することが明白に表現できてなかったり仕様が読み取りづらい(特に初見の開発者や開発者以外の人にとって)  
  
  
- 開発者が`isApproved`の設定ロジックを謝って  
間違ったメールを送ってしまうリスクがある  
(例えばメールアドレスが変更された場合に`isApproved`もオフにしなければいけないのに忘れていたなど)  
  
  
## 型システムを活用した実装  
  
これを型システムを活用して変更してみたいと思います。  
  
  
### UserMailをenumにする  
  
  
```swift

enum UserMail {
    case unApproved(EmailAddress)
    case approved(ApprovedEmailAddress)

    // approvedを初期化できないようにする
    init(_ mail: EmailAddress) {
        self = .unApproved(mail)
    }
}

struct EmailAddress {
    let address: String
    init(_ address: String) {
        self.address = address
    }
}

// approved用のクラス
struct ApprovedEmailAddress {
    let address: String
    init(_ address: String) {
        self.address = address
    }
}
```  
  
`isApproved`の判定をenumで表現するようにしました。  
こうすることで未承認と承認済みのメールアドレスがあることが明示できています。  
  
さらに承認済みメールアドレス(`EmailAddress`)と  
未承認メールアドレス(`ApprovedEmailAddress`)をクラスで分け  
UserMailのイニシャライザでは`EmailAddress`しか受け取らず  
未承認のケース(`unApproved`)しか作成できないようにしています。  
  
こうすることで  
承認していないのにも関わらず  
承認済みのケース(`approved`)を作成してしまうことを防ぐことができます。  
  
  
### 承認サービスを通してのみapproveできるようにする  
  
  
`approved`にするためには承認サービスを通すようにします。  
  
  
```swift

struct MailApproveService {

    func approveMail(mail: EmailAddress) -> UserMail? {

        // 何かの承認チェックをする...失敗した場合はnilになる

 
        return .approved(ApprovedEmailAddress(mail.address))
    }
}

```  
  
下記のような形で使用します。  
  
```swift

let unApprovedUserMail = UserMail.unApproved(EmailAddress("user@hoge.com"))

// これはできない🙅🏻‍♀️
//let approvedUserMail = UserMail.approved(VerifiedEmailAddress("approved@hoge.com"))


let service = MailApproveService()

guard case .unApproved(let mail) = unApprovedUserMail, 
    let approvedUserMail = service.approveMail(mail: mail) else {
        return
}

// UserMail.approved(ApprovedEmailAddress("user@hoge.com"))
print(approvedUserMail) 
```  
  
これが生み出すメリットとして  
  
- 承認を通さなければ承認済みのUserMailが作成できないことを保証できる  
- 承認メソッドの引数が`EmailAddress`なので承認済みメールアドレスを二重で承認することがなくなる  
  
などがあります。  
  
### 送信メソッドの引数で制限をかける  
  
さらにメールを送信する方法も下記のように変更します。  
  
```swift

func sendCampaignMail(mail: EmailAddress) {
    print("campaign mail send to \(mail.address)!!")
}

func sendSecretMail(mail: ApprovedEmailAddress) {
    print("secret mail send to \(mail.address)!!!!")
}

```  
  
こうすることで  
  
- 送るべきメールアドレスが引数で決まっているため謝って違う種類のメールを送るリスクが減る  
- 何を設定すれば良いのかを明示的に示すことができる  
  
使用すると下記のような結果になります。  
  
```swift

if case .unApproved(let mail) = unApprovedUserMail {
    print("ここは通る🙆🏻‍♀️")
    sendCampaignMail(mail: mail)
}

if case .approved(let mail) = unApprovedUserMail {
    print("ここは通らない🙅🏻‍♀️")
    sendSecretMail(mail: mail)
}

if case .approved(let mail) = approvedUserMail {
   print("ここは通る🙆🏻‍♀️")
   sendSecretMail(mail: mail)
}

if case .unApproved(let mail) = approvedUserMail {
    print("ここは通らない🙅🏻‍♀️")
    service.approveMail(mail: mail)
}

if case .unApproved(let mail) = approvedUserMail {
    print("ここは通らない🙅🏻‍♀️")
    sendCampaignMail(mail: mail)
}
```  
  
# 別の例  
  
もう一つ具体例を考えてみます。  
  
## 内容  
  
ユーザの入力データからユーザを登録する処理を考えます  
  
その際に  
  
- 名前は必ず入力する(空文字は🙅🏻‍♀️)  
- 連絡先として電話番号かメールアドレスいずれかが必須  
  
という仕様があるとします。  
  
  
## 最初の実装  
  
```swift

struct PhoneNumber {
    let number: Int
    init(_ number: Int) {
        self.number = number
    }
}

struct MailAddress {
    let address: String
    init(_ address: String) {
        self.address = address
    }
}

struct Input {
    let name: String
    let address: MailAddress?
    let phoneNumber: PhoneNumber?
}

struct RegisterService {
    func register(_ input: Input) -> Bool {
        if name.isEmpty {
            return false
        }
        if input.address == nil && input.phoneNumber == nil {
            return false
        } else {

            // 何かの登録処理...失敗したらfalse

            return true
        }
    }
}

func register() {

    let registerService = RegisterService()
    let valid = registerService.register(
        Input(name: "hoge", address: MailAddress("hoge@hoge.com"), phoneNumber: PhoneNumber (123)))
    if valid {
        print("登録完了🙆🏻‍♀️")
    }
    
    let inValid = registerService.register(
        Input(name: "hoge", address: nil, phoneNumber: nil))
    if !inValid {
        print("登録失敗🙅🏻‍♀️")
    }
}
```  
  
この場合  
  
- 入力値のチェックをしなければならない  
- ありえないケースも考慮しなければならない(`PhoneNumber`と`MailAddress`の両方がnil)  
- どういうケースがあるのかが不明瞭  
- 開発者がケースを見落とす可能性がある(`PhoneNumber`と`MailAddress`の両方が入力されている場合もOK)  
  
といった懸念点が挙げられます。  
  
  
## 型システムを活用した実装  
  
これを型システムを活用して変更してみたいと思います。  
  
### Contactをenumにする  
  
```swift

enum Contact {
    case both(MailAddress, PhoneNumber)
    case address(MailAddress)
    case phoneNumber(PhoneNumber)

    var phoneNumber: PhoneNumber? {
        switch self {
        case .both(_, let p):
            return p
        case .address:
            return nil
        case .phoneNumber(let p):
            return p
        }
    }
    
    var mailAddress: MailAddress? {
        switch self {
        case .both(let a, _):
            return a
        case .address(let a):
            return a
        case .phoneNumber:
            return nil
        }
    }
}
```  
  
こうすることで  
  
- すべてのケースが明確になる  
- 開発者がケースを見落とすリスクが減る  
- ありえないケースを排除できる  
  
というメリットがあります。  
  
### 空文字が入らないStringの型を作る  
  
  
Inputも変更します。  
  
```swift

struct Input {
    let name: NonEmptyString
    let contact: Contact
}

struct NonEmptyString {
    let value: String
    init?(_ value: String) {
        if value.isEmpty {
            return nil
        }
        self.value = value
    }
}
```  
  
ここでのポイントは  
NonEmptyStringという型を作ることで  
`name`に空文字が入ることがなくなり  
入力チェックが不要になります。  
  
  
### registerから入力チェックをなくす  
  
最後に登録メソッドを変更します。  
  
```swift

struct RegisterService {
    func register(_ input: Input) -> Bool {
        
        let name: NonEmptyString = input.name
        let phone: PhoneNumber? = input.contact.phoneNumber
        let address: MailAddress? = input.contact.mailAddress

        // 何かの登録処理...失敗したらfalse

        return true
    }
}
```  
  
ここでのメリットとして  
  
- 不正な状態のInputを受け取らないので入力チェックが不要になる  
- 不正な状態のInputを受け取らないので不正なデータが登録されるリスクが減る  
  
  
# まとめ  
  
型システムを活用する方法を考えてみました。  
  
この方法は定義する型の増加や  
enumによる条件分岐が増加などにより  
結果としてコード量が増えている場合が多いかもしれません。  
  
その代わりに  
  
- 型でチェックができるのでより安全により安心した開発ができる  
- 型を作成することで仕様がより明確に表現できるようになり初見の開発者や開発者以外の人でも仕様が理解しやすくなる  
- ランタイムチェックでは必要であったテストケースが必要がなくなる  
  
といったメリットを得ることができます。  
  
特に仕様が明確になるという点に関しては  
うまく型を作成することで  
コードがドキュメントの役割も担ってくれるようになるので  
大きなメリットになるなと感じています。  
  
すべての場合に型を作成していくことは  
時間がかかることですし  
不必要な部分もあると思います。  
  
どこまでやるかの塩梅はプロジェクトに依りますが  
できる部分はできる限り活用できたらいいなと私は思っています。  
  
  
何か間違いなどございましたらご指摘いただけましたら幸いです:bow_tone1:  
