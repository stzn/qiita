# Codableとは？  
  
```swift

typealias Codable = Decodable & Encodable
```  
  
Swift4で導入された`Encodable`と`Decodable`というProtocolのtype aliasで  
`Encodable`に適合することでSwiftの型をJSONやPropertyListなどに  
`Decodable`に適合することでJSONやPropertyListなどをSwiftの型に  
変換することができます。  
  
  
既存の型は`Codable`に適合しているものが多く  
独自の型を定義する際も中のプロパティが全て`Codable`に適合している場合  
宣言をするだけで自動で変換が可能になるため  
`Codable`のおかげで多くのボイラープレートを削除することができ  
日々の開発に使用されています。  
  
さらに  
独自の変換方法を定義して柔軟な変換を行うことも可能です。  
  
今回は特に**JSON**と**Decodable**に焦点を当て  
普段何気なく使用している`Codable`が  
改めてどのように活用できるのかを考えてみました。  
  
# 基本的な使い方  
  
まず下記のようなJSONデータがあるとします。  
  
```swift

let usersJson = """
[
    {
        "id": 1,
        "name": "Taro",
        "email": "taro@sample.com",
    },
    {
        "id": 2,
        "name": "Jiro",
        "email": "jiro@sample.com",
    },
    {
        "id": 3,
        "name": "Saburo",
        "email": "saburo@sample.com",
    }
]
"""

```  
  
これに対して  
`Codable`に適合したUser型を定義して変換してみたいと思います。  
  
```swift
struct User: Codable {
    let id: Int
    let name: String
    let email: String
}

let decoder = JSONDecoder()
let users = try! decoder.decode([User].self, from: Data(usersJson.utf8))

[User(id: 1, name: "Taro", email: "taro@sample.com"), 
User(id: 2, name: "Jiro", email: "jiro@sample.com"), 
User(id: 3, name: "Saburo", email: "saburo@sample.com")]
```  
  
プロパティがすべて`Codalbe`に適合しているため  
User型も`Codable`を宣言するだけで適合することができます。  
  
また  
JSONのキー名とプロパティの名前は一致しているため  
そのまま変換をすることが可能になっています。  
  
# JSONのキーと型のプロパティを違う名前にしたい場合  
  
名前が異なる場合はどうでしょうか？  
JSONのキーとプロパティ名の関連性によって  
いくつかの変換方法があります。  
  
## 自動で変換してくれるkeyEncodingStrategy & keyDecodingStrategy  
  
よくあるケースとして  
APIとのやりとりの中でJSONのキー名はsnakeCaseであることがあります。  
  
一方で  
SwiftのプロパティはlowerCamelCaseにすることが  
APIデザインガイドラインでも推奨されています。  
  
```
Follow case conventions. 
Names of types and protocols are UpperCamelCase. 
Everything else is lowerCamelCase.
```  
  
このような場合  
一定の規則に則っていると  
`keyEncodingStrategy`と`keyDecodingStrategy`を用いることで  
自動で変換してくれます。  
  
https://developer.apple.com/documentation/foundation/jsonencoder/keyencodingstrategy  
https://developer.apple.com/documentation/foundation/jsondecoder/keydecodingstrategy/convertfromsnakecase  
  
下記に例を示します。  
  
```swift
let usersJson = """
[
    {
        "id": 1,
        "user_name": "Taro",
        "email_address": "taro@sample.com",
    },
    {
        "id": 2,
        "user_name": "Jiro",
        "email_address": "jiro@sample.com",
    },
    {
        "id": 3,
        "user_name": "Saburo",
        "email_address": "saburo@sample.com",
    }
]
"""

struct User: Codable {
    let id: Int
    let userName: String
    let emailAddress: String
}

let decoder = JSONDecoder()

// ここでどういう風に変換するのかを決めている
decoder.keyDecodingStrategy = .convertFromSnakeCase

let users = try! decoder.decode([User].self, from: Data(usersJson.utf8))

```  
  
上記のように  
`keyDecodingStrategy`に`convertFromSnakeCase`を設定するだけで  
snakeCaseのキー名からlowerCamelCaseのプロパティ名の変換をしてくれます。  
  
ただし  
変換方法には下記のようなルールがあり  
これに適合しない場合は手動で`CodingKeys`を定義する必要が出てきます。  
  
**1. アンダースコアの次に来る文字を大文字にする**  
**2. 最初と最後のアンダースコア以外のアンダースコアを除去する**  
**3. 全ての単語を一つの文字列にする**  
  
**URI**などは合致しないため  
手動での定義が必要になります。  
  
※ 補足として  
`keyDecodingStrategy`には`useDefaultKeys`もあり  
こちらがデフォルトでdecode時に  
JSONの名前をそのままプロパティの名前に使用します。  
  
  
## 名前が異なる場合は？  
  
`CodingKey`プロトコルに適合し  
`RawValue`が`String`のenumを生成することで  
JSONのキーとデータ構造のプロパティ名のマッピングをすることができます。  
  
```swift

let usersJson = """
[
    {
        "id": 1,
        "user_name": "Taro",
        "email_address": "taro@sample.com",
    },
    {
        "id": 2,
        "user_name": "Jiro",
        "email_address": "jiro@sample.com",
    },
    {
        "id": 3,
        "user_name": "Saburo",
        "email_address": "saburo@sample.com",
    }
]
"""

struct User: Codable {
    let id: Int
    let name: String
    let email: String

    // ここでJSONのキー名とUserのプロパティ名をマッピング
    // 必要のないものはそのまま
    enum CodingKeys: String, CodingKey {
        case id
        case name = "user_name"
        case email = "email_address"
    }
}
```  
  
## DateEncodingStrategy & DateDecodingStrategy  
  
JSONの型とプロパティの型が異なる場合があります。  
  
その中でも日付の変換に関しては  
`DateEncodingStrategy`と`DateDecodingStrategy`というenumを使うことで  
自動で変換をしてくれるようになります。  
  
https://developer.apple.com/documentation/foundation/jsonencoder/2895363-dateencodingstrategy  
https://developer.apple.com/documentation/foundation/jsondecoder/datedecodingstrategy  
  
```swift

let usersJson = """
[
    {
        "id": 1,
        "name": "Taro",
        "email": "taro@sample.com",
        "entry_date": 1558483628,
    },
    {
        "id": 2,
        "name": "Jiro",
        "email": "jiro@sample.com",
        "entry_date": 1559483628,
    },
    {
        "id": 3,
        "name": "Saburo",
        "email": "saburo@sample.com",
        "entry_date": 1560483628,
    }
]
"""

struct User: Codable {
    let id: Int
    let name: String
    let email: String
    let entryDate: Date
}

let decoder = JSONDecoder()
decoder.keyDecodingStrategy = .convertFromSnakeCase

// dateDecodingStrategyを設定することでDate型へ自動で設定してくれる
decoder.dateDecodingStrategy = .secondsSince1970

let users = try! decoder.decode([User].self, from: Data(usersJson.utf8))

print(users)

User(id: 1, name: "Taro", email: "taro@sample.com", entryDate: 2019-05-22 00:07:08 +0000), 
User(id: 2, name: "Jiro", email: "jiro@sample.com", entryDate: 2019-06-02 13:53:48 +0000),
User(id: 3, name: "Saburo", email: "saburo@sample.com", entryDate: 2019-06-14 03:40:28 +0000)

```  
  
※　もし型がうまく合わない場合は独自でフォーマットを定義することもできます。  
  
例えば下記のよう**ISO8601形式**のフォーマットは自動変換してくれません。  
https://ja.wikipedia.org/wiki/ISO_8601  
  
```Swift

// この形式はDate型への変換に失敗する🙀

let usersJson = """
[
    {
        "id": 1,
        "name": "Taro",
        "email": "taro@sample.com",
        "entry_date": "2019-05-22T05:50:00.000Z",
    },
    {
        "id": 2,
        "name": "Jiro",
        "email": "jiro@sample.com",
        "entry_date": "2019-05-23T05:50:00.000Z",
    },
    {
        "id": 3,
        "name": "Saburo",
        "email": "saburo@sample.com",
        "entry_date": "2019-05-24T05:50:00.000Z",
    }
]
"""

struct User: Codable {
    let id: Int
    let name: String
    let email: String
    let entryDate: Date
}
```  
  
そこで独自フォーマットを指定することで  
この問題を解決します。  
  
```swift

extension DateFormatter {
    static let fullISO8601: DateFormatter = {
        let formatter = DateFormatter()
        formatter.dateFormat = "yyyy-MM-dd'T'HH:mm:ss.SSSZZZZZ"
        formatter.calendar = Calendar(identifier: .iso8601)
        formatter.timeZone = TimeZone(secondsFromGMT: 0)
        formatter.locale = Locale(identifier: "en_US_POSIX")
        return formatter
    }()
}

let decoder = JSONDecoder()
decoder.keyDecodingStrategy = .convertFromSnakeCase

// ここで独自フォーマットを指定する
decoder.dateDecodingStrategy = .formatted(DateFormatter.fullISO8601)

let users = try! decoder.decode([User].self, from: Data(usersJson.utf8))

User(id: 1, name: "Taro", email: "taro@sample.com", entryDate: 2019-05-22 05:50:00 +0000), 
User(id: 2, name: "Jiro", email: "jiro@sample.com", entryDate: 2019-05-23 05:50:00 +0000), 
User(id: 3, name: "Saburo", email: "saburo@sample.com", entryDate: 2019-05-24 05:50:00 +0000)

```  
  
## SingleValueEncodingContainer & SingleValueDecodingContainer  
  
例えばUserを下記のように変更してみます。  
  
```swift

struct User: Codable {
    let id: Int
    let name: String
    let email: Email
}

struct Email: Codable {
    let value: String
}
```  
  
Userのemailを  
一つのプロパティを持ったEmail型にします。  
  
このような場合  
今までのようなJSON形式では変換できません。  
  
自動変換するにはJSONが下記のような形である必要があります。  
  
  
```swift

{
    "id": 1,
    "name": "Taro",
    "email": { "value": "taro@sample.com" },
}
```  
  
解決するために  
`SingleValueEncodingContainer`や`SingleValueDecodingContainer`が活用できます。  
  
https://developer.apple.com/documentation/swift/singlevaluedecodingcontainer  
https://developer.apple.com/documentation/swift/singlevalueencodingcontainer  
  
これを使うことで型を単純な値として扱えるようになります。  
  
```swift

struct Email: Codable {
    let value: String

    // SingleValueDecodingContainerを用いることでJSONの値から直接型に変換できる
    init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        self.value = try container.decode(String.self)
    }
    
    func encode(to encoder: Encoder) throws {
        var container = encoder.singleValueContainer()
        try container.encode(value)
    }
}
```  
  
## RawRepresentableでもう少しスッキリさせることも  
  
上記の例は`RawRepresentable`を使うと  
もう少しコードを短くすることができます。  
  
```swift

struct Email: Codable, RawRepresentable {
    let rawValue: String
}
```  
  
これだけでOKです。  
これは`RawRepresentable`がextensionで  
`Encodable`, `Decodable`に適合しているからです。  
  
```swift

extension RawRepresentable where Self : Encodable, Self.RawValue == String {

    /// Encodes this value into the given encoder, when the type's `RawValue`
    /// is `String`.
    ///
    /// This function throws an error if any values are invalid for the given
    /// encoder's format.
    ///
    /// - Parameter encoder: The encoder to write data to.
    public func encode(to encoder: Encoder) throws
}

extension RawRepresentable where Self : Decodable, Self.RawValue == String {

    /// Creates a new instance by decoding from the given decoder, when the
    /// type's `RawValue` is `String`.
    ///
    /// This initializer throws an error if reading from the decoder fails, or
    /// if the data read is corrupted or otherwise invalid.
    ///
    /// - Parameter decoder: The decoder to read data from.
    public init(from decoder: Decoder) throws
}
```  
さらにプロパティの名前を`rawValue`にすることで  
イニシャライザも自動で定義されるようになりました。  
  
  
# ネストされた型を変換する  
  
APIなどから返ってくるJSONは  
JSONの中にさらにJSONが含まれているような形式のものが多く見られ  
作りたいデータ構造とは異なる形式である場合があります。  
  
そのような際に独自の変換が活用できます。  
これもいくつかの方法があります。  
  
## データ構造を合わせる  
  
上記の例と同様にデータ構造を合わせることで変換が可能になります。  
  
例えば下記のようなJSONに対して  
  
```swift

let usersJson = """
[
    {
        "id": 1,
        "name": "Taro",
        "email": "taro@sample.com",
        "profile": {
            "nick_name": "t",
            "profile_image_url": "profile/taro.jpg",
            "follow_number": 10,
        }
    },
    {
        "id": 2,
        "name": "Jiro",
        "email": "jiro@sample.com",
        "profile": {
            "nick_name": "j",
            "profile_image_url": "profile/jiro.jpg",
            "follow_number": 20,
        }
    },
    {
        "id": 3,
        "name": "Saburo",
        "email": "saburo@sample.com",
        "profile": {
            "nick_name": "s",
            "profile_image_url": "profile/saburo.jpg",
            "follow_number": 30,
        }
    },
]
"""
```  
  
下記のようなデータ構造を作成することで  
自動変換が可能になります。  
  
```swift

struct User: Codable {
    let id: Int
    let name: String
    let email: String
    let profile: Profile

    struct Profile: Codable {
        let nickName: String
        let profileImageUrl: URL
        let followNumber: Int
    }
}

let decoder = JSONDecoder()
decoder.keyDecodingStrategy = .convertFromSnakeCase
let users = try! decoder.decode([User].self, from: Data(usersJson.utf8))

[User(id: 1, name: "Taro", email: "taro@sample.com", profile: User.Profile(nickName: "t", profileImageUrl: profile/taro.jpg, followNumber: 10)), 
User(id: 2, name: "Jiro", email: "jiro@sample.com", profile: User.Profile(nickName: "j", profileImageUrl: profile/jiro.jpg, followNumber: 20)), 
User(id: 3, name: "Saburo", email: "saburo@sample.com", profile: User.Profile(nickName: "s", profileImageUrl: profile/saburo.jpg, followNumber: 30))]
```  
  
# KeyedEncodingContainer & KeyedDecodingContainer  
  
例えば上記のJSONの形式と  
欲しいデータ構造の形が異なる場合を考えてみたいと思います。  
  
先ほどと似たような例ですが  
一部変更を加えています。  
  
```swift

let usersJson = """
[
    {
        "id": 1,
        "name": "Taro",
        "email": "taro@sample.com",
        "profile": {
            "nick_name": "t",
            "profile_image_url": "profile/taro.jpg",
            "follow_number": 10,
        }
    },
    {
        "id": 2,
        "name": "Jiro",
        "email": "jiro@sample.com",
        "profile": {
            "nick_name": "j",
            "profile_image_url": "profile/jiro.jpg",
            "follow_number": 20,
        }
    },
    {
        "id": 3,
        "name": "Saburo",
        "email": "saburo@sample.com",
        "profile": {
            "nick_name": "s",
　　　　　　　// profile_imageがない
            "follow_number": 30,
        }
    },
]
"""
```  
  
これを下記のデータ構造に変換したいと思います。  
  
```swift

struct User: Codable {
    let id: Int
    let name: String
    let email: String

    // この2つはProfileから取ってきたい
    let nickName: String
    let profileImageUrl: String
}
```  
  
このような場合は  
`CodingKey`に適合したenumと  
`KeyedEncodingContainer`と`KeyedDecodingContainer`  
を用いて変換方法を定義することができます。  
  
https://developer.apple.com/documentation/swift/keyedencodingcontainer  
https://developer.apple.com/documentation/swift/keyeddecodingcontainer  
  
```swift

struct User: Codable {
    let id: Int
    let name: String
    let email: String
    let nickName: String
    let profileImageUrl: String?

    // JSONのキーを指定
    enum CodingKeys: String, CodingKey {
        case id
        case name
        case email
        case profile

        enum ProfileKeys: String, CodingKey {
            case nickName
            case profileImageUrl
            case followNumber
        }
    }

    init(from decoder: Decoder) throws {

        // decoderからcontainerを取得する
        let container = try decoder.container(keyedBy: CodingKeys.self)

        // containerを使って各値をdecodeする
        id = try container.decode(Int.self, forKey: .id)
        name = try container.decode(String.self, forKey: .name)
        email = try container.decode(String.self, forKey: .email)

        // containerからさらにnestedContainerを取得する
        let profileContainer = try container.nestedContainer(keyedBy: CodingKeys.ProfileKeys.self, forKey: .profile)

        // nestedContainerを使って各値をdecodeする
        nickName = try profileContainer.decode(String.self, forKey: .nickName)
        profileImageUrl = try profileContainer.decodeIfPresent(String.self, forKey: .profileImageUrl)
    }
    
    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(id, forKey: .id)
        try container.encode(name, forKey: .name)
        try container.encode(email, forKey: .email)        
        var profileContainer = container.nestedContainer(keyedBy: CodingKeys.ProfileKeys.self, forKey: .profile)
        try profileContainer.encode(nickName, forKey: .nickName)
        try profileContainer.encodeIfPresent(profileImageUrl, forKey: .profileImageUrl)
    }
}
```  
  
こうすることでネストされた型から値を取得したり  
JSONから取得した値をさらに加工して設定したりなど可能になります。  
  
また場合によっては値がないものに関しては`decodeIfPresent`のように  
IfPresentをつけることでnilが入るようになります。  
  
※ ちなみにIfPresent系のメソッドにthrowsが付いていますが  
これは値があった際に変換する型が合わなかった場合に**typeMismatch**が起きるからです。  
https://github.com/apple/swift/blob/master/stdlib/public/core/Codable.swift.gyb#L474  
  
  
## UnkeyedEncodingContainer & UnkeyedDecodingContainer  
  
もっと複雑な例を考えています。  
ネストされたJSONの中に配列があるとします。  
  
```swift

let usersJson = """
[
    {
        "id": 1,
        "name": "Taro",
        "email": "taro@sample.com",
        "profile": {
            "nick_name": "t",
            "profile_image_url": "profile/taro.jpg",
            "follow_number": 10,
            "follows": [
                { "id": 1 },
            ],
        }
    },
    {
        "id": 2,
        "name": "Jiro",
        "email": "jiro@sample.com",
        "profile": {
            "nick_name": "j",
            "profile_image_url": "profile/jiro.jpg",
            "follow_number": 20,
            "follows": [
                { "id": 1 },
                { "id": 2 },
            ],
        }
    },
    {
        "id": 3,
        "name": "Saburo",
        "email": "saburo@sample.com",
        "profile": {
            "nick_name": "s",
            "profile_image_url": "profile/saburo.jpg",
            "follow_number": 30,
            "follows": [
                { "id": 1 },
                { "id": 2 },
                { "id": 3 },
            ],
        }
    },
]
"""
```  
  
これに`UnkeyedEncodingContainer`と`UnkeyedDecodingContainer`  
を活用してみます。  
  
これは`CodingKey`を使用せずに  
Continerの中身に順番にアクセスすることができます。  
  
https://developer.apple.com/documentation/swift/unkeyedencodingcontainer  
https://developer.apple.com/documentation/swift/unkeyeddecodingcontainer  
  
```swift

struct User: Codable {
    let id: Int
    let name: String
    let email: String
    let nickName: String
    let profileImageUrl: String
    let followIds: [Int]

    enum CodingKeys: String, CodingKey {
        case id
        case name
        case email
        case profile

        enum ProfileKeys: String, CodingKey {
            case nickName
            case profileImageUrl
            case followNumber
            case follows

            enum FollowsKeys: String, CodingKey {
                case id
            }
        }
    }

    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        id = try container.decode(Int.self, forKey: .id)
        name = try container.decode(String.self, forKey: .name)
        email = try container.decode(String.self, forKey: .email)

        let profileContainer = try container.nestedContainer(keyedBy: CodingKeys.ProfileKeys.self, forKey: .profile)
        nickName = try profileContainer.decode(String.self, forKey: .nickName)
        profileImageUrl = try profileContainer.decode(String.self, forKey: .profileImageUrl)

        // ここでUnkeyedContainerを取得する(ここではfollowersの配列)
        // varであることに注意
        var followsContainer = try profileContainer.nestedUnkeyedContainer(forKey: .follows)
        var ids: [Int] = []

        // UnkeyedContainerの中の配列を順番に辿って中の値を取得する
        // followsContainerの値を抜き出しているためfollowsContainerはvarである必要がある
        while !followsContainer.isAtEnd {
            let idContaier = try followsContainer.nestedContainer(keyedBy: CodingKeys.ProfileKeys.FollowsKeys.self)
            let id = try idContaier.decode(Int.self, forKey: .id)
            ids.append(id)
        }
        followIds = ids
    }
    
    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(id, forKey: .id)
        try container.encode(name, forKey: .name)
        try container.encode(email, forKey: .email)
        
        var profileContainer = container.nestedContainer(keyedBy: CodingKeys.ProfileKeys.self, forKey: .profile)
        try profileContainer.encode(nickName, forKey: .nickName)
        try profileContainer.encode(profileImageUrl, forKey: .profileImageUrl)
        
        var followsContainer = profileContainer.nestedUnkeyedContainer(forKey: .follows)
        var idContaier = followsContainer.nestedContainer(keyedBy: CodingKeys.ProfileKeys.FollowsKeys.self)
        for id in followIds {
            try idContaier.encode(id, forKey: .id)
        }
    }
}


let decoder = JSONDecoder()
decoder.keyDecodingStrategy = .convertFromSnakeCase
let users = try! decoder.decode([User].self, from: Data(usersJson.utf8))

[User(id: 1, name: "Taro", email: "taro@sample.com", nickName: "t", profileImageUrl: "profile/taro.jpg", followIds: [1]), 
User(id: 2, name: "Jiro", email: "jiro@sample.com", nickName: "j", profileImageUrl: "profile/jiro.jpg", followIds: [1, 2]), 
User(id: 3, name: "Saburo", email: "saburo@sample.com", nickName: "s", profileImageUrl: "profile/saburo.jpg", followIds: [1, 2, 3])]
```  
  
## ちょっと複雑なので  
  
上記では  
`UnkeyedEncodingContainer`と`UnkeyedDecodingContainer`を使いましたが  
ちょっとわかりづらいかなと個人的に感じました。  
  
このような複雑な場合は  
下記のように変換用のデータ構造を挟むことで  
よりわかりやすく表現できるのではないかと私は思います。  
  
```swift

struct CodableUser: Codable {
    let id: Int
    let name: String
    let email: String
    let profile: Profile

    struct Profile: Codable {
        let nickName: String
        let profileImageUrl: URL
        let followNumber: Int
        let follows: [Follow]

        struct Follow: Codable {
            let id: Int
        }
    }
}

struct User {
    let id: Int
    let name: String
    let email: String
    let nickName: String
    let profileImageUrl: String
    let followIds: [Int]

    init(codable: CodableUser) {
        self.id = codable.id
        self.name = codable.name
        self.email = codable.email
        self.nickName = codable.profile.nickName
        self.profileImageUrl = codable.profile.profileImageUrl.absoluteString
        self.followIds = codable.profile.follows.map { $0.id }
    }
}


let decoder = JSONDecoder()
decoder.keyDecodingStrategy = .convertFromSnakeCase
let codableUsers = try! decoder.decode([CodableUser].self, from: Data(usersJson.utf8))
let user = codableUsers.map(User.init)
```  
  
  
# まとめ  
  
`Codable`は自動で変換をしてくれることで  
余分なコードを削除できることに加え  
独自の変換方法でSwiftの型とJSONを変換できる  
柔軟性も兼ね備えていることがわかりました。  
  
ただし  
あまりにも複雑になっていると感じた場合は  
データ構造や設計を見直すタイミングであるのかもしれません。  
  
今回考えてきたケース以外にも  
まだまだ使い方はたくさんあります。  
  
Appleのドキュメントにもサンプルコードがありますので  
こちらもぜひご参照ください。  
  
https://developer.apple.com/documentation/foundation/archives_and_serialization/using_json_with_custom_types  
  
また  
もしこういう使い方があるといったご意見や  
ここはもっとこうした方が良いなどのご指摘がございましたら  
教えていただけましたら幸いです🙇🏻‍♂️  
