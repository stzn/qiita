Combineフレームワークを使っている時に  
いくつかのPublisherから出力される値を  
まとめて使いたい場面はあると思います。  
  
そのようなOperatorをCombining Operator(結合するOperator)と呼ばれ  
いくつかの種類があり  
それぞれ出力されるタイミングや値が異なります。  
  
今回はそんなOperatorの動きを  
見ていきたいと思います。  
  
※ 下記に記載しているコードは  
Playgroundで実行しています。  
  
# 前に値を出力する(Prepend)  
  
あるPublisherの前に値が出力されるようにPublisherを挿入します。  
  
<img width="631" alt="スクリーンショット 2019-10-19 11.42.53.png" src="/image/c96e5208-9cf0-e75b-2ed9-ed88c927959e.png">  
  
具体的には3つのメソッドがあります。  
  
## prepend(Output...)  
  
可変長引数を受け取ります。  
  
https://developer.apple.com/documentation/combine/publisher/3204739-prepend  
  
```swift

var subscriptions = Set<AnyCancellable>()
let publisher = ["c", "d"].publisher
publisher
    .prepend("a", "b")
    .sink(receiveValue: { print($0) })
    .store(in: &subscriptions)

// 出力結果
a
b
c
d
```  
  
## prepend(S) where S: Sequence  
  
Sequenceプロトコルに適合した型のインスタンスを受け取ります。  
  
https://developer.apple.com/documentation/combine/publisher/3204740-prepend  
  
```swift

var subscriptions = Set<AnyCancellable>()
let publisher = ["e", "f", "g"].publisher
publisher
   .prepend(["c", "d"])
   .prepend(Set(["a", "b"]))
   .sink(receiveValue: { print($0) })
   .store(in: &subscriptions)

// 出力結果
a
b
c
d
e
f
```  
※a, bはSetなので順不同です。  
  
この例の結果を見てみますと  
prependは  
**後に追加したものから先に出力される**  
ことがわかります。  
  
## prepend(P) where P: Publisher  
  
Publisherプロトコルに適合した型のインスタンスを受け取ります。  
  
https://developer.apple.com/documentation/combine/publisher/3204741-prepend  
  
```swift

var subscriptions = Set<AnyCancellable>()

let publisherA = ["c", "d"].publisher
let publisherB = ["a", "b"].publisher
publisherA
    .prepend(publisherB)
    .sink(receiveValue: { print($0) })
    .store(in: &subscriptions)

// 出力結果
a
b
c
d
```  
ここで注意が必要な点としては  
ドキュメントにも記載がありましたが  
前のPublisherが終了しないと  
次のPublisherの値が出力されないという点です。  
  
例えば下記の場合ですと  
  
```swift

var subscriptions = Set<AnyCancellable>()

let publisherA = ["c", "d"].publisher
let publisherB = PassthroughSubject<String, Never>()
publisherA
    .prepend(publisherB)
    .sink(receiveValue: { print($0) })
    .store(in: &subscriptions)
publisherB.send("a")
publisherB.send("b")

// 出力結果
a
b
```  
となり  
publisherAのc, dは出力されません。  
  
出力するためにはpublisherBを終了させます。  
  
  
```swift

var subscriptions = Set<AnyCancellable>()

let publisherA = ["c", "d"].publisher
let publisherB = PassthroughSubject<String, Never>()
publisherA
    .prepend(publisherB)
    .sink(receiveValue: { print($0) })
    .store(in: &subscriptions)
publisherB.send("a")
publisherB.send("b")
publisherB.send(completion: .finished)

// 出力結果
a
b
c
d
```  
  
  
# 後に値を出力する(Append)  
  
Prependとは反対に対象のPublisherの後に値を出力します。  
  
<img width="644" alt="スクリーンショット 2019-10-19 11.39.57.png" src="/image/176571fe-ba5e-d8e7-3f62-fc14ff6d04d7.png">  
  
メソッドも同様に3つあります。  
  
## append(Output...)  
  
可変長引数を受け取ります。  
  
https://developer.apple.com/documentation/combine/publisher/3204683-append  
  
```swift

var subscriptions = Set<AnyCancellable>()

let publisherA = ["c", "d"].publisher
let publisherB = ["a", "b"].publisher
publisherA
    .append(publisherB)
    .sink(receiveValue: { print($0) })
    .store(in: &subscriptions)

// 出力結果
c
d
a
b
```  
  
## append(S) where S: Sequence  
  
Sequenceプロトコルに適合した型のインスタンスを受け取ります。  
  
https://developer.apple.com/documentation/combine/publisher/3204684-append  
  
```swift

var subscriptions = Set<AnyCancellable>()

let publisher = ["e", "f", "g"].publisher
publisher
    .append(["c", "d"])
    .append(Set(["a", "b"]))
    .sink(receiveValue: { print($0) })
    .store(in: &subscriptions)

// 出力結果
e
f
g
c
d
b
a
```  
※a, bはSetなので順不同です。  
  
この例の結果を見てみますと  
appendは  
**前に追加したものから先に出力される**  
ことがわかります。  
  
## append(P) where P: Publisher  
  
Publisherプロトコルに適合した型のインスタンスを受け取ります。  
  
https://developer.apple.com/documentation/combine/publisher/3204685-append  
  
```swift

var subscriptions = Set<AnyCancellable>()

let publisherA = ["c", "d"].publisher
let publisherB = ["a", "b"].publisher
publisherA
    .append(publisherB)
    .sink(receiveValue: { print($0) })
    .store(in: &subscriptions)

// 出力結果
c
d
a
b
```  
prependと同様に  
前のPublisherが終了しないと  
次のPublisherの値が出力されないという点です。  
  
例えば下記の場合ですと  
  
```swift

var subscriptions = Set<AnyCancellable>()

let publisherA = ["c", "d"].publisher
let publisherB = PassthroughSubject<String, Never>()
let publisherC = PassthroughSubject<String, Never>()
publisherA
    .append(publisherB)
    .append(publisherC)
    .sink(receiveValue: { print($0) })
    .store(in: &subscriptions)
publisherB.send("a")
publisherB.send("b")
publisherC.send("e")
publisherC.send("f")

// 出力結果
c
d
a
b
```  
となり  
publisherCのe, fは出力されません。  
  
出力するためにはpublisherBを終了させます。  
  
  
```swift

var subscriptions = Set<AnyCancellable>()

let publisherA = ["c", "d"].publisher
let publisherB = PassthroughSubject<String, Never>()
let publisherC = PassthroughSubject<String, Never>()
publisherA
    .append(publisherB)
    .append(publisherC)
    .sink(receiveValue: { print($0) })
    .store(in: &subscriptions)
publisherB.send("a")
publisherB.send("b")
publisherB.send(completion: .finished)
publisherC.send("e")
publisherC.send("f")

// 出力結果
c
d
a
b
e
f
```  
  
# merge(with:)  
  
Publisherの後に出力されるPublisherを繋げます。  
最大8つ繋げることができます。  
  
https://developer.apple.com/documentation/combine/publisher/3204723-merge  
  
<img width="822" alt="スクリーンショット 2019-10-19 11.50.39.png" src="/image/40020c3b-9c82-310d-2a9f-d68c935af60c.png">  
  
  
```swift

var subscriptions = Set<AnyCancellable>()

let publisherA = PassthroughSubject<String, Never>()
let publisherB = PassthroughSubject<String, Never>()

publisherA
    .merge(with: publisherB)
    .sink(receiveCompletion: { _ in print("finish") },
          receiveValue: { print($0) })
    .store(in: &subscriptions)

publisherA.send("a")
publisherA.send("b")
publisherB.send("c")
publisherA.send("d")
publisherB.send("e")

publisherA.send(completion: .finished)
publisherB.send(completion: .finished)

// 出力結果
a
b
c
d
e
finish
```  
switchToLatestと同様に  
結合した全てのpublisherが終了しないと  
対象のpublisherは終了しません。  
  
  
```swift

...

publisherA.send(completion: .finished) 
// publisherB.send(completion: .finished)

// 出力結果
a
b
c
d
e
❌finishは出力されない
```  
  
# switchToLatest  
  
複数のPublisherを  
一つのPublisherとして出力するように結合します。  
  
この際に新しいPublisherを出力すると  
前のPublisherのSubscriptionはキャンセルされます。  
  
https://developer.apple.com/documentation/combine/publisher/3204759-switchtolatest  
  
<img width="1420" alt="スクリーンショット 2019-10-19 12.12.43.png" src="/image/2c6c9cab-9918-94c4-8fdd-227363a79c00.png">  
  
```swift

var subscriptions = Set<AnyCancellable>()

let publisherA = PassthroughSubject<String, Never>()
let publisherB = PassthroughSubject<String, Never>()
let publisherC = PassthroughSubject<String, Never>()

let publishers = PassthroughSubject<PassthroughSubject<String,
    Never>, Never>()

publishers.switchToLatest()
    .sink(receiveCompletion: { _ in print("finish") },
          receiveValue: { print($0)})
    .store(in: &subscriptions)

publishers.send(publisherA)
publisherA.send("a")
publisherA.send("b")

// publisherBへ切り替え
publishers.send(publisherB)
publisherA.send("c") // 出力されない
publisherB.send("d")
publisherB.send("e")

// publisherCへ切り替え
publishers.send(publisherC)
publisherB.send("f") // 出力されない
publisherC.send("g")
publisherC.send("h")
publisherC.send("i")

publisherC.send(completion: .finished)
publishers.send(completion: .finished)

// 出力結果
a
b
d
e
g
h
i
finish
```  
  
上記のように  
  
publisherBへ切り替えた後のpublisherA  
と  
publisherCへ切り替えた後のpublisherB  
の値の出力はされません。  
  
  
また注意点として  
現在のpublisherが終了しないと  
publishersの終了は出力されません。  
  
  
```swift

...

// publisherC.send(completion: .finished)
publishers.send(completion: .finished)

// 出力結果
a
b
d
e
g
h
i
❌finishは出力されない
```  
  
## どういう時に使うか？  
  
イメージしづらいかもしれませんが  
例えば  
ボタンを押すと外部からデータを取得するなどを行う時に  
何度もボタンを押されても  
通信を重複して行わないようにする場合などが  
使用例として考えられます。  
  
  
# combineLatest  
  
これも複数のPublisherを一つにまとめて値を出力しますが  
下記の3つの特徴があります。  
  
- 出力される値はそれぞれのPublisherから出力される値の**Tuple**  
- 結合している**全ての**Publisherから新しい値が一つ出力されないと値は出力されない  
- 結合している**いずれかの**Publisherから新しい値が出力されると値が出力される  
  
https://developer.apple.com/documentation/combine/publisher/3333677-combinelatest  
  
<img width="1017" alt="スクリーンショット 2019-10-19 12.17.39.png" src="/image/326918ea-e987-315d-e94c-d1c1a8ed4500.png">  
  
```swift

var subscriptions = Set<AnyCancellable>()

let publisherA = PassthroughSubject<Int, Never>()
let publisherB = PassthroughSubject<String, Never>()

publisherA
    .combineLatest(publisherB)
    .sink(receiveCompletion: { _ in print("finish") },
          receiveValue: { print("A: \($0), B: \($1)") }) 
    .store(in: &subscriptions)

publisherA.send(1)
publisherA.send(2)
publisherB.send("a")
publisherB.send("b")
publisherA.send(3)
publisherB.send("c")

publisherA.send(completion: .finished)
publisherB.send(completion: .finished)

// 出力結果
A: 2, B: a
A: 2, B: b
A: 3, B: b
A: 3, B: c
finish

```  
  
## どういう時に使うか？  
  
考えられる例としては  
冒頭で紹介したような  
複数の入力フォームがあって  
全てに入力しないと登録ボタンが押せないように  
ボタンの状態を制御するときなど  
各フォームの値の変更を自動で検知してくれるので  
便利です。  
  
  
# zip  
  
combineLatestに似ていますが  
下記部分が異なります。  
  
- 結合している**全ての**Publisherから新しい値が出力されると値が出力される  
  
https://developer.apple.com/documentation/combine/publisher/3333677-combinelatest  
  
<img width="1023" alt="スクリーンショット 2019-10-19 12.19.30.png" src="/image/3e54a2e6-07bb-6d15-9e12-57121bdc3289.png">  
  
  
```swift

var subscriptions = Set<AnyCancellable>()

let publisherA = PassthroughSubject<Int, Never>()
let publisherB = PassthroughSubject<String, Never>()

publisherA
    .zip(publisherB)
    .sink(receiveCompletion: { _ in print("finish") },
          receiveValue: { print("A: \($0), B: \($1)") }) 
    .store(in: &subscriptions)

publisherA.send(1) // publisherA①
publisherA.send(2) // publisherA②
publisherB.send("a") // publisherB①
publisherB.send("b") // publisherB②
publisherA.send(3) // publisherA③
publisherB.send("c") // publisherB③

publisherA.send(completion: .finished)
publisherB.send(completion: .finished)

// 出力結果
A: 1, B: a
A: 2, B: b
A: 3, B: c
finish

```  
  
ちょっとややこしいのですが  
publisherA①の後に  
publisherA②を出力していますが  
まだ何も出力されません。  
  
その次のpublisherB①が出力された後に初めて  
A: 1, B: a  
という値が出力されます。  
  
そして  
publisherB②が出力されると  
A: 2, B: b  
が出力されます。  
  
publisherの出力値に番号がついていて  
同じ番号のものが全て揃った時に  
値が出力されるようなイメージだとわかりやすいかもしれません。  
  
## どういう時に使うか？  
  
複数の非同期で行うAPIリクエストの  
全ての結果を受け取った後に  
処理を行いたい場合などに活用できます。  
  
# まとめ  
  
Combineの中でも  
Publisherを結合するOperatorについて見てきました。  
  
使うと便利なものがたくさんありますが  
使い方を間違えると  
微妙に思った通りの動きをしてくれないなど  
混乱してしまう可能性があります。  
  
他にも便利ですが  
似た様なOperatorはありますので  
また頭の整理のためにも  
見ていきたいと思います😃  
  
もし間違いなどございましたら  
ご指摘頂けますとうれしいです🙇🏻‍♂️  
