## 経緯  
  
structを使っている時、  
できる限り不変なものとして扱いたいが特定の値は変更できるようにしたい  
というような状況によく出会います。  
  
  
例えば、  
  
```
enum Position {
    case director, manager, engineer, officeWorker
}

struct Staff {
    let name: String
    let position: Position
    let monthlySalary: Int
    let address: Address
}

struct Address {
    let postCode: String
    let prefecture: Prefecture    
}

struct Prefecture {
    let name: String
}

let staff = Staff(
    name: "スタッフ",
    position: .engineer,
    monthlySalary: 350000,
    address: Address(
        postCode: "111-1111",
        prefecture: Prefecture(
            name: "XX県" ← ****ここだけ変更したい****
        ))
)
```  
  
といった場合、不変を保とうとすると、  
  
```
let newStaff = Staff(
    name: staff.name,
    position: staff.position,
    monthlySalary: staff.monthlySalary,
    address: Address(
        postCode: staff.address.postCode,
        prefecture: Prefecture(
            name: "〇〇県" ← ****変更****
        ))
)

```  
  
というようにかなり冗長な書き方になってモヤモヤしていました。  
  
  
つい先日、また同じような経験をし、何か良い方法はないかと探したところ、  
Lensというものを見つけました。  
  
## Lensとは？  
  
複雑な(ネストが深いなど)構造のデータを簡単なgetter/setterのような形で書けるようにするといったもののようです。  
※現在理解している内容です。もっと勉強します。  
  
そもそもはHaskellに[lensパッケージ](https://github.com/ekmett/lens)が存在し、  
それを参考に各言語にも実装されているようです。  
  
Swiftでも[Focus](https://github.com/typelift/Focus)などの実装があるようです。  
  
  
参考記事:  
https://medium.com/@EnnioMa/functional-lenses-an-exploration-in-swift-25b4d3a6a536  
https://qiita.com/to4iki/items/f0cc28e1102cf53be85d  
  
## 実装  
  
Lensの内容は以下のような形です。(※実際はもっとメソッドあります。)  
  
```

public struct Lens<A, B> {
    public let get: (A) -> B
    public let set: (A, B) -> A
}

precedencegroup ComposePrecedence {
    associativity: left
}

infix operator >>> : ComposePrecedence

func >>> <A, B, C>(lhs: Lens<A, B>, rhs: Lens<B, C>) -> Lens<A, C> {
    return Lens(

        // get(A, B) => get(B, C) = get(A, C)
        // (1) get(A, B)でBが取得でき、
        // (2) get(B, C)でCが取得できるのでget(A, C)が成り立つ

        //       (2)     (1)
        get: { rhs.get(lhs.get($0)) },

        // set(A, B) => set(B, C) = set(A, C)
        // (1) get(A, B)でBが取得でき、
        // (2) set(B, C)でBが取得でき、
        // (3) set(A, B)でAが取得できるのでset(A, C)が成り立つ

        //              (3)         (2)      (1)
        set: { return lhs.set($0, rhs.set(lhs.get($0), $1)) }
    )
}
```  
  
これを使用すると以下のように書けます。  
  
```StructVarAndLens.playground

enum Position {
    case director
    case manager
    case engineer
    case officeWorker
}

struct Address {
    let postCode: String
    let prefecture: Prefecture
    
    static let postCode_ = Lens<Address, String>(
        get: { $0.postCode },
        set: { Address(postCode: $1, prefecture: $0.prefecture)}
    )

    static let prefecture_ = Lens<Address, Prefecture>(
        get: { $0.prefecture },
        set: { Address(postCode: $0.postCode, prefecture: $1)}
    )
}

struct Prefecture {
    let name: String

        static let name_ = Lens<Prefecture, String>(
            get: { $0.name },
            set: { Prefecture(name: $1)}
        )
}

struct Staff {
    let name: String
    let position: Position
    let monthlySalary: Int
    let address: Address
    
        static let address_ = Lens<Staff, Address>(
            get: { $0.address },
            set: { Staff(name: $0.name, position: $0.position, monthlySalary: $0.monthlySalary, address: $1 )}
        )

        static let name_ = Lens<Staff, String>(
            get: { $0.name },
            set: { Staff(name: $1, position: $0.position, monthlySalary: $0.monthlySalary, address: $0.address)}
        )
}

let prefecture = Prefecture(
    name: "XX県"
)

let address = Address(
    postCode: "111-1111",
    prefecture: prefecture
)

let staff = Staff(
    name: "スタッフ",
    position: .engineer,
    monthlySalary: 300000,
    address: address
)

print(staff)
// Staff(
//      name: "スタッフ", 
//      position: __lldb_expr_23.Position.engineer, 
//      monthlySalary: 300000, 
//      address: __lldb_expr_23.Address(postCode: "111-1111", 
//      prefecture: __lldb_expr_23.Prefecture(name: "XX県")))

let address2 = Address.postCode_.set(address, "222-2222")
let newStaff = Staff.address_.set(staff, address2)

print(newStaff)

//Staff(
//      name: "スタッフ", 
//      position: __lldb_expr_23.Position.engineer, 
//      monthlySalary: 300000, 
//      address: __lldb_expr_23.Address(postCode: "222-2222", 
//      prefecture: __lldb_expr_23.Prefecture(name: "XX県")))


let lens = Staff.address_ >>> Address.prefecture_ >>> Prefecture.name_

let newPrefectureName = lens.set(staff, "aaaaaa")

print(newPrefectureName)

//Staff(
//      name: "スタッフ", 
//      position: __lldb_expr_23.Position.engineer, 
//      monthlySalary: 300000, 
//      address: __lldb_expr_23.Address(postCode: "222-2222", 
//      prefecture: __lldb_expr_23.Prefecture(name: "aaaaaa")))
```  
  
と簡潔に書けるようになりました。  
  
  
## けれども・・・  
  
この場合  
  
```StructVarAndLens2.playground

enum Position {
    case director
    case manager
    case engineer
    case officeWorker
}

struct Address {
    var postCode: String
    var prefecture: Prefecture
}

struct Prefecture {
    var name: String
}

struct Staff {
    var name: String
    var position: Position
    var monthlySalary: Int
    var address: Address
}

let prefecture = Prefecture(
    name: "XX県"
)

let address = Address(
    postCode: "111-1111",
    prefecture: prefecture
)

let staff = Staff(
    name: "スタッフ",
    position: .engineer,
    monthlySalary: 300000,
    address: address
)

print(staff)
// Staff(
//      name: "スタッフ", 
//      position: __lldb_expr_23.Position.engineer, 
//      monthlySalary: 300000, 
//      address: __lldb_expr_23.Address(postCode: "111-1111", 
//      prefecture: __lldb_expr_23.Prefecture(name: "XX県")))

// varで宣言
var newStaff = staff
newStaff.address.postCode = "222-2222"
newStaff.address.prefecture.name = "aaaaaa"

print(newStaff)
//Staff(
//      name: "スタッフ", 
//      position: __lldb_expr_23.Position.engineer, 
//      monthlySalary: 300000, 
//      address: __lldb_expr_23.Address(postCode: "222-2222", 
//      prefecture: __lldb_expr_23.Prefecture(name: "aaaaaa")))


print(staff)
// Staff(
//      name: "スタッフ", 
//      position: __lldb_expr_23.Position.engineer, 
//      monthlySalary: 300000, 
//      address: __lldb_expr_23.Address(postCode: "111-1111", 
//      prefecture: __lldb_expr_23.Prefecture(name: "XX県")))

```  
  
とした方が自分にとってはわかりやすくていいなと感じました。  
  
ただ、この場合は変数の宣言もvarを使用しなければならないため、  
値が変更できてしまうので不変ではなくなります。  
  
とはいうものの、  
  
上記の最後のstaffのように  
参照型とは異なり値の変更が他のstructに影響を与えることはないので、  
varで宣言した変数の扱いに気をつけていれば良いのではないかとも思いました。  
  
ただ、どこで何が起きるのかを把握し続けなければならないリスクは避けたいですね。  
  
  
## RxExampleのLensを見てみる  
  
続けて色々見ていく中で、RxExampleにもLensの実装がありました。  
  
```Lenses.swift

// These are kind of "Swift" lenses. 
// We don't need to generate a lot of code this way 
// and can just use Swift `var`.

protocol Mutable {
}

extension Mutable {
    func mutateOne<T>(transform: (inout Self) -> T) -> Self {
        var newSelf = self
        _ = transform(&newSelf)
        return newSelf
    }
    
    func mutate(transform: (inout Self) -> ()) -> Self {
        var newSelf = self
        transform(&newSelf)
        return newSelf
    }
    
    func mutate(transform: (inout Self) throws -> ()) rethrows -> Self {
        var newSelf = self
        try transform(&newSelf)
        return newSelf
    }
}

```  
  
これを使用することで、  
  
```StructVarAndLens3.playground

enum Position {
    case director
    case manager
    case engineer
    case officeWorker
}

struct Staff {
    var name: String
    var position: Position
    var monthlySalary: Int
    var address: Address
}

struct Address {
    var postCode: String
    var prefecture: Prefecture
}

struct Prefecture {
    var name: String
}

extension Staff: Mutable {}


let prefecture = Prefecture(
    name: "XX県"
)

let address = Address(
    postCode: "111-1111",
    prefecture: prefecture
)

let staff = Staff(
    name: "スタッフ",
    position: .engineer,
    monthlySalary: 300000,
    address: address
)

print(staff)
// Staff(
//      name: "スタッフ", 
//      position: __lldb_expr_23.Position.engineer, 
//      monthlySalary: 300000, 
//      address: __lldb_expr_23.Address(postCode: "111-1111", 
//      prefecture: __lldb_expr_23.Prefecture(name: "XX県")))

// letで宣言できる
let newStaff = staff.mutateOne { $0.address.postCode = "222-2222" }

print(newStaff)
// Staff(
//      name: "スタッフ", 
//      position: __lldb_expr_23.Position.engineer, 
//      monthlySalary: 300000, 
//      address: __lldb_expr_23.Address(postCode: "222-2222", 
//      prefecture: __lldb_expr_23.Prefecture(name: "XX県")))


// letで宣言できる
let newStaff2 = staff.mutate {
    $0.address.postCode = "333-3333"
    $0.address.prefecture.name = "aaaaaaa"
}

print(newStaff2)

// Staff(
//      name: "スタッフ", 
//      position: __lldb_expr_23.Position.engineer, 
//      monthlySalary: 300000, 
//      address: __lldb_expr_23.Address(postCode: "222-2222", 
//      prefecture: __lldb_expr_23.Prefecture(name: "aaaaaa")))

```  
  
と書けるようになります。  
  
これをすることで、  
  
・structの中身の変更したい値はvarで宣言する必要があるものの、  
　struct自体はletで宣言できるため、通常の値の代入は不可能になる  
  
・値の変更をしたい場合はstructがprotocol(ここではMutable)を継承し、  
　意図的にメソッドを呼ぶ必要があるため、値の変更箇所が明確になる  
  
といったメリットがあるのではないかと思っています。  
  
  
## まとめ  
  
結論としては、最後の方法を使ってメリット・デメリットを検討していきたいなと思います。  
ただ、Lensの実装方法は非常に勉強になり、他の機能もたくさんありましたのでもっと色々見て学びたいと思います。  
  
コード: https://github.com/stzn/StructVarAndLens  
<br>  
