普段使っているものの  
実は注意しないと予期せぬ動作を起こしてしまう可能性があるものは  
意外とたくさんあると感じています。  
  
そういう時  
Swiftのドキュメントを見てみると  
注意書(NOTE)があったりして役に立ちます。  
  
今回は一つの例として  
`lazy`のついたプロパティについて見てみたいと思います。  
  
# Lazy Stored Property(遅延格納プロパティ)とは？  
  
Swiftでは  
`lazy`という修飾子を使うことで  
最初に利用されるまで  
初期化処理を走らなくさせることができます。  
  
これを  
Lazy Stored Property(遅延格納プロパティ)  
と呼びます。  
  
# どういうときに使う？  
  
下記のような場合に役に立ちます。  
  
- 初期値がインスタンス生成後の状態(他のプロパティなど)に依存している  
- セットアップが複雑で重く、使われるまでは生成する必要がない  
  
例えば  
下記のようなCalendarクラスを考えてみます。  
  
```swift

class Calendar {
    private lazy var formatter: DateFormatter = {
        print("call")
        let formatter = DateFormatter()
        formatter.dateStyle = .short
        formatter.locale = Locale.init(identifier: "ja_JP")
        return formatter
    }()

    var today: String {
        formatter.string(from: Date())
    }
}

let c = Calendar()
```  
  
formatterの初期化処理は  
Calendarインスタンスを生成した時点では  
呼ばれませんが  
  
```swift

c.today 

// call
```  
  
とtodayプロパティ経由で  
formatterに始めてアクセスした時点で  
初期化処理が呼ばれるようになります。  
  
そして一度初期化処理が走ると  
値がメモリに保存されるため  
再度処理は走りません。  
  
```swift

let c = Calendar()
c.today
c.today

// call ※1回しか出力されない
```  
  
# 実は複数回呼ばれる場合がある  
  
しかし  
これには例外があります。  
  
Swiftのドキュメントによると  
  
```
NOTE
If a property marked with the lazy modifier is accessed by multiple threads simultaneously 
and the property has not yet been initialized, 
there is no guarantee that the property will be initialized only once.
```  
  
↓のLazy Stored Propertiesの項  
https://docs.swift.org/swift-book/LanguageGuide/Properties.html  
  
  
マルチスレッドで同時にアクセスされた場合に  
まだ初期化処理が終わっていないと  
1回しか呼ばれないことは保証されない。  
  
とあります。  
  
先ほどのソースを少し変更してみます。  
  
```swift

class Calendar {
    private lazy var formatter: DateFormatter = {
        sleep(1)
        print("call")
        let formatter = DateFormatter()
        formatter.dateStyle = .short
        formatter.locale = Locale.init(identifier: "ja_JP")
        return formatter
    }()

    var today: String {
        formatter.string(from: Date())
    }
}
```  
  
初期化処理で少しsleepします。  
  
そして2つのキューからアクセスするようにしてみると  
  
```swift

let q1 = DispatchQueue(label: "1")
let q2 = DispatchQueue(label: "2")

let c = Calendar()
q1.async { print(c.today) }
q2.async { print(c.today) }

// call
// call
...

```  
  
初期化処理が2回走ることがわかりました。  
  
この中で  
もし値の変更など  
副作用を起こすような処理が入っていたら  
2回値を変更してしまった結果  
予期せぬ不具合を引き起こす可能性があるので  
注意が必要です。  
  
# Stored Type Property(格納型プロパティ)は1回しか呼ばれない  
  
ここで似たようなものとして  
Stored Type Property(格納型プロパティ)  
があります。  
  
Stored Type Propertyはインスタンスに属しておらず  
`class`や`static`という修飾子がついた  
型自体に属しているプロパティです。  
  
そしてSwiftのドキュメントに  
下記の記載があります。  
  
```
Stored type properties are lazily initialized on their first access. 
They are guaranteed to be initialized only once, 
even when accessed by multiple threads simultaneously, 
and they do not need to be marked with the lazy modifier.
```  
  
↓のType Propertiesの項  
https://docs.swift.org/swift-book/LanguageGuide/Properties.html  
  
つまり  
Stored Type Propertyの場合には  
マルチスレッドで同時にアクセスされたとしても  
必ず1回だけ処理が走るということを保証しています。  
  
先ほどのformatterプロパティを  
Type Propertyに変えて  
同じことをやってみます。  
  
```swift

class Calendar {
    private static var formatter: DateFormatter = {
        sleep(1)
        print("call")
        let formatter = DateFormatter()
        formatter.dateStyle = .short
        formatter.locale = Locale.init(identifier: "ja_JP")
        return formatter
    }()

    var today: String {
        Calendar.formatter.string(from: Date())
    }
}

let q1 = DispatchQueue(label: "1")
let q2 = DispatchQueue(label: "2")

let c = Calendar()
q1.async { print(c.today) }
q2.async { print(c.today) }

// call ※ 1回だけしか出力されない

```  
  
1回しか初期化処理が走らないことがわかりました。  
  
  
# まとめ  
  
`lazy`なプロパティは  
不要なメモリの消費やパフォーマンスの低下を防ぐことができますが  
使い方を誤ると  
わかりづらい不具合を混ぜてしまう可能性があることがわかりました。  
  
この`lazy`のように  
Swiftのドキュメントには  
実は気をつけなければいけないことなどの  
情報が書いてありますので  
一度全てに目を通しておくことや  
定期的にチェックすることは大事ですね😃  
  
  
もし何か間違いなどございましたら  
ご指摘いただけますと嬉しいです🙇🏻‍♂️  
