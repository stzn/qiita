# Ordered Collection Diffingとは？  
  
Swift5.1で導入される新しい機能です。  
  
Forumスレッド一覧  
https://forums.swift.org/search?q=SE-0240  
  
Original Pitch  
https://forums.swift.org/t/se-0240-ordered-collection-diffing/19514/34  
  
Proposal  
https://github.com/apple/swift-evolution/blob/master/proposals/0240-ordered-collection-diffing.md  
  
Review  
https://forums.swift.org/t/se-0240-ordered-collection-diffing/19514/42  
https://forums.swift.org/t/amendment-se-0240-ordered-collection-diffing/26084  
  
これは`difference(from:)`や`difference(from:by:)`を呼び出すことで  
2つのコレクションの差分を計算して  
何がどう変わっているのかを教えてくれます。  
  
`difference(from:)`  
https://github.com/apple/swift/blob/a0421124f0de37d696e503c17c05afba19f74cf2/stdlib/public/core/Diffing.swift#L188  
  
またOrdered Collection Diffingは全てSwiftで書かれています。  
  
今回はOrdered Collection Diffingがどういうものなのか  
とても参考になる記事がありましたので  
そちらを参考に実際の実装などを見ながら  
理解をしていきたいと思います。  
  
参考記事  
https://www.fivestars.blog/code/swift-5-1-collection-diffing.html  
  
  
最初に使用される要素を見ていき  
その後にメソッドを見ていきたいと思います。  
  
  
# CollectionDifference  
  
Genericな`Collection`で  
2つの整列されているコレクションの  
**差分(どの要素が削除され、どの要素が挿入されているのか)**を  
を保持しています。  
  
```
A collection of insertions and removals 
that describe the difference between two ordered collection states
```  
  
`CollectionDifference `  
https://github.com/apple/swift/blob/a0421124f0de37d696e503c17c05afba19f74cf2/stdlib/public/core/CollectionDifference.swift#L16  
  
  
## ChangeElement   
  
比較している要素の型でこれには何の制約も付いていません。  
`.insert`と`.remove`という2つのケースを持った  
`CollectionDifference.Change`という`enum`になっており  
個々の要素の変化を表します。(詳細は後ほど記載します)  
  
`CollectionDifference.Change`  
https://github.com/apple/swift/blob/a0421124f0de37d696e503c17c05afba19f74cf2/stdlib/public/core/CollectionDifference.swift#L19  
  
  
## CollectionDifferenceの特徴  
  
`CollectionDifference`には下記のような特徴があります。  
  
### CollectionDifferenceが返す値の順番が決まっている  
  
まず最初に削除された要素のコレクション  
その次に追加された要素のコレクションという順番になっています。  
  
これは内部の`init`で保証されており  
必ず適切な順番になって返ってきます。  
  
`init`  
https://github.com/apple/swift/blob/a0421124f0de37d696e503c17c05afba19f74cf2/stdlib/public/core/CollectionDifference.swift#L168  
  
  
### CollectionDifferenceの要素はユニークである  
  
`init`のコメントに下記のような記載があります。  
  
```
The collection of changes passed as `changes` must meet these
requirements: 
- All insertion offsets are unique
- All removal offsets are unique
- All associations between insertions and removals are symmetric
```  
  
つまり  
削除や挿入された要素のオフセットは全てユニークでなければなりません。  
また削除と挿入の関係は対称でなければなりません。  
  
また  
同じインデックスに対して削除の処理をすることはできません。  
挿入に関しても同じです。  
  
さらに  
移動した要素は削除と挿入に対になる要素が一つ存在している。  
どちらか片方だけというようなものは存在できません。  
  
これは内部でバリデーションをしており  
`init`で不適切なコレクションが渡ってきた場合はnilが返却されます。  
  
`_validateChanges`  
https://github.com/apple/swift/blob/a0421124f0de37d696e503c17c05afba19f74cf2/stdlib/public/core/CollectionDifference.swift#L89  
  
## CollectionDifference.Change  
  
2つのコレクションを比較した時に  
変化としては下記の3つのパターンがあります。  
  
- 新しい要素が追加された  
- 要素が削除された  
- 移動して位置が変わった(元の位置から削除され、新しい位置に挿入された)  
  
そのため  
`CollectionDifference.Change`は  
insertとremoveの2つのenumで表現できます。  
  
`CollectionDifference.Change`  
https://github.com/apple/swift/blob/a0421124f0de37d696e503c17c05afba19f74cf2/stdlib/public/core/CollectionDifference.swift#L19  
  
それぞれのcaseは`offset` `element` `associatedWith`という   
**3つのassociated values**を持っています。  
  
  
### `offset`  
  
挿入または削除された位置です。  
  
#### 削除された場合  
  
```swift

let oldArray = ["a", "b", "c", "d"]
let newArray = ["a", "b", "d"]

// "c"という要素がindex 2から削除されました

let difference = newArray.difference(from: oldArray) 
// differenceは1つの要素を含んだ配列です。
// これは`.remove`で`offset`は2となります。
```  
  
#### 挿入された場合  
  
```swift

let oldArray = ["a", "b", "c", "d"]
let newArray = ["a", "b", "c", "d", "e"]

// "e"という要素がindex 4に挿入されました

let difference = newArray.difference(from: oldArray) 
// differenceは1つの要素を含んだ配列です。
// これは`.insert`で`offset`は4となります。
```  
  
### element  
  
実際の要素です。  
これがCollectionDifferenceがジェネリックになっている要因です。  
  
let oldArray = ["a", "b", "c", "d"]  
let newArray = ["a", "b", "d", "e"]  
  
// "c"という要素をindex 2から削除  
// "e"という要素をindex 3に追加  
  
let difference = newArray.difference(from: oldArray)  
// differenceは2つの要素の配列  
// 最初の要素は`.remove`でoffset 2、element "c"  
// 次の要素は`.insert`でoffset 3、element "e"  
  
### associatedWith  
  
要素を移動した時にこの値が入ります。  
`insert`の場合は移動後のオフセット  
`remove`の場合は移動前のオフセット  
が入ります。  
  
これは計算に時間がかかるため  
**デフォルトでは全てnil**となっており  
`inferringMoves()`を呼び出す必要があります。  
  
`inferringMoves`  
https://github.com/apple/swift/blob/a0421124f0de37d696e503c17c05afba19f74cf2/stdlib/public/core/CollectionDifference.swift#L323  
  
```swift

let oldArray = ["a", "b", "d", "e", "c"]
let newArray = ["a", "b", "c", "d", "e"]

// "c"をindex 4から2へ移動
// - index 4から削除
// - index 2へ追加

let difference = newArray.difference(from: oldArray)
// differenceは2つの要素の配列
// - 最初の要素は`.remove` offset 4 associatedWith nil
// - 次の要素は`.insert` offset 2 associatedWith nil

let differenceWithMoves = difference.inferringMoves()
// differenceWithMovesは2つの要素の配列
// - 最初の要素は`.remove` offset 4 associatedWith 2
// - 次の要素は`.insert` offset 2 associatedWith 4
```  
  
#### InferringMoves  
  
`CollectionDifference`インスタンスをスキャンして  
削除と挿入のオフセットがマッチした(移動した)要素をコレクションに格納していきます。  
もし存在しなければ元のコレクションと同じ要素が含まれたコレクションを返します。  
  
## insertionsとremovalsプロパティ  
  
これらの2つのプロパティを使用することで  
挿入したChangeと  
削除したChangeへ  
アクセスすることができます。  
  
### insertions  
挿入された要素が**オフセットが小さいものから順番に**並んでいます。  
  
`insertions`  
https://github.com/apple/swift/blob/a0421124f0de37d696e503c17c05afba19f74cf2/stdlib/public/core/CollectionDifference.swift#L70  
  
### removals  
削除された要素が**オフセットが小さいものから順番に**並んでいます。  
  
`removals`  
https://github.com/apple/swift/blob/a0421124f0de37d696e503c17c05afba19f74cf2/stdlib/public/core/CollectionDifference.swift#L73  
  
# CollectionDifferenceの要素の並び順  
  
`init`の中を見てみるとわかるのですが  
要素の順番は  
  
**削除が先 挿入があと**  
  
になります。  
  
挿入された要素は`insertions`配列に入り  
削除された要素は`removals`配列に入ります。  
  
しかし`subscript`を見てみると  
また違った順番で要素を参照しています。  
  
`removals`は大きいオフセットから小さいオフセットへ  
`insertions`は小さいオフセットから大きいオフセットに向かっています。  
  
`subscript`  
https://github.com/apple/swift/blob/a0421124f0de37d696e503c17c05afba19f74cf2/stdlib/public/core/CollectionDifference.swift#L253  
  
これは順番に取得した要素を元のコレクション(変化前のコレクション)に適用していくと  
新しいコレクションになるようになっており  
**変化を再現**することができます。  
  
```swift

let oldArray = ["a", "b", "c", "d"]
let newArray = ["x", "a", "e", "c"]

var anotherArray = oldArray

let difference = newArray.difference(from: oldArray)
// differenceは4つの要素の配列
// 1. `.remove` at offset 3
// 2. `.remove` at offset 1 
// 3. `.insert` at offset 0
// 4. `.insert` at offset 2

for change in difference {
  switch change {
  case let .remove(offset, _, _):
    anotherArray.remove(at: offset)
  case let .insert(offset, newElement, _):
    anotherArray.insert(newElement, at: offset)
  }
}
// この時点で`anotherArray`は`newArray`と等しくなる 
```  
  
遷移をたどると下記のようになります。  
  
```swift

["a", "b", "c", "d"] // 初期状態
["a", "b", "c"] // "d"をindex 3から削除
["a", "c"] // "b"をindex 1から削除
["x", "a", "c"] // "x"をindex 0へ挿入
["x", "a", "e", "c"] // "e"をindex 2へ挿入
```  
  
# applying  
  
上記では`difference(from:)`で取得した要素をコレクションに順番に適用してきましたが  
`applying(_:)`メソッドを使用することで変化を別のコレクションに適用できます。  
  
`applying`  
https://github.com/apple/swift/blob/a0421124f0de37d696e503c17c05afba19f74cf2/stdlib/public/core/Diffing.swift#L68  
  
```swift

let oldArray = ["a", "b", "c", "d"]
let newArray = ["x", "a", "e", "c"]

var anotherArray = oldArray

let difference = newArray.difference(from: oldArray)

// ここでdifferenceをapplyingしている
anotherArray = anotherArray.applying(difference)!

// この時点で`anotherArray`は`newArray`と等しくなる 
```  
  
このメソッドは制約に適合している限りはどんなコレクションにも適用できます。  
戻り値として`difference`を適用した`Optional`なコレクションを返します。  
  
なぜ`Optional`なのかは  
`difference`の要素のオフセットが  
適用対称のコレクションのオフセットを超えた時に  
クラッシュを避けてnilを返すようにしています。  
  
  
# difference(from:)  
  
これまでも出てきましたOrdered Collection Diffingの中心となるメソッドです。  
  
```swift

extension BidirectionalCollection where Element : Equatable {
  public func difference<C: BidirectionalCollection>(
    from other: C
  ) -> CollectionDifference<Element> where C.Element == Self.Element {
    return difference(from: other, by: ==)
  }
}
```  
  
`difference(from:)`は`difference(from: by:)`を呼び出しているだけで  
要素が`Equatable`の時に使用できます。  
  
# difference(from: by:)  
  
byに等しくなる条件を指定することで`Equatable`に適合していない要素にも  
differenceを取得することができるようになります。  
  
  
```swift

public func difference<C: BidirectionalCollection>(
  from other: C,
  by areEquivalent: (Element, C.Element) -> Bool
) -> CollectionDifference<Element>
where C.Element == Self.Element 
```  
  
`difference(from:by:)`  
https://github.com/apple/swift/blob/a0421124f0de37d696e503c17c05afba19f74cf2/stdlib/public/core/Diffing.swift#L134  
  
シンタックスを見てみると2つのコレクションは  
`BidirectionalCollection`に適合している必要があります。  
  
つまり前から後からもコレクションの要素へ行ったり来たり参照できることが必要になります。  
また2つのコレクションはassociated typeの型は同じである必要があります。  
  
byの条件によってdifferenceで取得できる要素が変わってきます。  
  
例えば全てに対してfalseを返すようにすると  
  
```swift

let oldArray = ["a", "b", "c"]
let newArray = ["a", "b", "c"] // 同じ要素

let difference = newArray.difference(from: oldArray, 
                                     by: { _, _  in false })
// `difference`は６つの要素を持つ配列:
// - removalsに３要素
// - insertionsに３要素
```  
  
これは全ての要素の比較がfalseになるので  
要素を全部入れ替えたとみなされているからです。  
  
  
## difference(from: by:) の戻り値  
  
Optional **ではない** CollectionDifferenceを返します。  
  
上記でCollectionDifferenceのinitは  
nilを返す可能性があることを紹介しましたが  
なぜ違うのか？  
  
これは内部でnilにならないinitを保持しており  
そちらを使用しているからです。  
  
これは妥当だと証明されたコレクションを使用しているので  
Swift的にもnon-optionalで問題ありません。  
  
`init `  
https://github.com/apple/swift/blob/a0421124f0de37d696e503c17c05afba19f74cf2/stdlib/public/core/CollectionDifference.swift#L168  
  
  
## difference(from: by:)の内部  
  
まず`_CountingIndexCollection`のコレクションを２つ作成し  
これを`_CollectionChanges`への変換しなおしています。  
  
```swift

let source = _CountingIndexCollection(other)
let target = _CountingIndexCollection(self)

let result = _CollectionChanges(from: source, to: target, by: areEquivalent)
```  
  
`_CountingIndexCollection`は実際のコレクションを包んで  
開始のindexからのoffsetへアクセスしやすくします。  
  
```swift

let originalCollection = ["a", "b", "c", "d"]
let offsetCollection = _CountingIndexCollection(originalCollection)
if let index = offsetCollection.index(of: "d") {
  print(index.offset) // 3
  print(offsetCollection[index]) // d
}
```  
  
`_CollectionChanges`はCollectionDifference.Changeインスタンスの配列です。  
  
その後取得したChangeインスタンスの配列から  
`CollectionDifference`のインスタンスを生成して返却しています。  
  
このメソッドないでは実際の差分計算はしておらず  
`_CollectionChanges`の初期化の中で行っています。  
  
# _CollectionChanges  
  
２つのコレクションの要素を辿っていき、下記のようなことを行っています。  
  
- ２つのコレクションの**最長共通部分列**を取得  
https://ja.wikipedia.org/wiki/%E6%9C%80%E9%95%B7%E5%85%B1%E9%80%9A%E9%83%A8%E5%88%86%E5%88%97%E5%95%8F%E9%A1%8C  
  
- 削除と挿入の**最短経路**を探して最も少ない手順で`source`コレクションから`target`コレクションへ遷移する方法を見つけます。  
https://ja.wikipedia.org/wiki/%E6%9C%80%E7%9F%AD%E7%B5%8C%E8%B7%AF%E5%95%8F%E9%A1%8C  
  
この２つは表裏一体のような関係となっており  
片方が見つかるともう片方も見つかるようになっています。  
  
挿入や削除は非常にコストの高い操作であるため  
このプロセスで最も効率差分更新するための方法を導き出すことが  
Ordered Collection Diffingの根幹になります。  
  
`_CollectionChanges `  
https://github.com/apple/swift/blob/a0421124f0de37d696e503c17c05afba19f74cf2/stdlib/public/core/Diffing.swift#L247  
  
ある状態からある状態への遷移方法は複数見つかる中で  
もっとも効率の良いものを_CollectionChangesは導き出すことを保証します。  
  
## Endpoint  
  
_CollectionChangesの定義の冒頭にあるtypealiasです。  
  
```swift

typealias Endpoint = (x: SourceIndex, y: TargetIndex)
```  
  
これは`source`コレクションと`target`コレクションの関係を表します。  
例えば`source`コレクションの最初の要素と`target`コレクションの２番目の要素を関連づけたい場合  
`Endpoint`は(1,2)と定義します。  
  
  
## pathStorage  
  
ここからは_CollectionChangesが  
具体的にどういう計算をしているのかを見ていきます。  
  
下記のような２次元のチャートがあります。  
  
x軸がsourceのコレクションを  
y軸はtargetのコレクションを表します。  
  
```
  | • | X | A | B | C | D |
 • |   |   |   |   |   |   |
 X |   |   |   |   |   |   |
 Y |   |   |   |   |   |   |
 C |   |   |   |   |   |   |
 D |   |   |   |   |   |   |
```  
  
左上を(0,0)として一番右下の(lastIndexOfTarget, lastIndexOfSource)を目指します。  
  
下記のようなイメージです。  
```
  | • | X | A | B | C | D |
 • | S |   |   |   |   |   | 
 X |   |   |   |   |   |   | // S: 開始 
 Y |   |   |   |   |   |   | // T: 目的地
 C |   |   |   |   |   |   |
 D |   |   |   |   |   | T |

// 開始位置 S: (0,0)
// 目的位置 T: (4,5)
```  
  
ルール  
- 左から右にしか移動できない  
- 上から下にしか移動できない  
- 左右と上下は同じ距離だけ同時に動かして良い(左に1、下に1など)  
- 左右と上下を同時に動かして良いのはxとyが同じ時だけ((A,A)の時など)  
  
この中でもっとも少ない数で対角線の位置に移動する方法を探します。  
  
これが最長共通部分列問題です。  
  
`_CollectionChanges`に当てはめると  
  
- 垂直への移動が挿入  
- 平行への移動が削除  
  
になります。  
  
結果は下記になります。  
  
```
  | • | X | A | B | C | D |
 • | S |   |   |   |   |   | 
 X |   | x | x | x |   |   | // S: 開始 
 Y |   |   |   | x |   |   | // T: 目的地
 C |   |   |   |   | x |   |
 D |   |   |   |   |   | T |

// 経路:
(0,0) → (1,1) → (1,2) → (1,3) → (2,3) → (3,4) → (4,5)
```  
  
これを_CollectionChangesで考えると  
  
- (1,1) → (1,2)、(1,2) → (1,3)は２つの削除  
- (1,3) → (2,3)は１つの挿入  
  
になります。  
  
この一つ一つの地点が`Endpoint`であり  
pathは`Endpoint`の配列になります。  
  
これが`pathStorage`プロパティになります。  
  
`pathStorage`  
https://github.com/apple/swift/blob/a0421124f0de37d696e503c17c05afba19f74cf2/stdlib/public/core/Diffing.swift#L276  
  
## pathStartIndex  
  
`_CollectionChanges`が持つもう一つのプロパティで  
差分計算のパスを探す開始位置を表します。  
  
これは内部最適化のために使われ  
例えば**AE**OSと**AE**FTという文字列があった場合  
**AE**は共通なのでこの分の計算は無駄になります。  
そこで`pathStartIndex`を見てこの無駄をスキップします。  
  
`pathStartIndex`  
https://github.com/apple/swift/blob/a0421124f0de37d696e503c17c05afba19f74cf2/stdlib/public/core/Diffing.swift#L279  
  
## enum Element  
  
internalな要素で上記の計算で見つけたパスを  
`CollectionDifference.Change`に変換するときに使用します。  
  
```swift

enum Element {
    case removed(Range<SourceIndex>)
    case inserted(Range<TargetIndex>)
    case matched(Range<SourceIndex>, Range<TargetIndex>)
}
```  
  
`_CollectionChanges.Element`  
https://github.com/apple/swift/blob/a0421124f0de37d696e503c17c05afba19f74cf2/stdlib/public/core/Diffing.swift#L300  
  
※ なぜ`Range`なのか？  
  
上記の問題の例では一つ一つ`Endpoint`を分けて考えましたが  
(1,1) → (1,2) → (1,3)は連続した平行移動で  
(1,1) → (1,3)と一緒にできます。  
これを表現するために`Range`となっています。  
  
`difference`メソッドでは  
この`Element`を一つ一つ取り出して`CollectionDifference.Change`へと変換していきます。  
  
# _CollectionChangesの内部  
  
まず中心となる`init`の形です。  
  
```swift

fileprivate
init<Source: BidirectionalCollection, Target: BidirectionalCollection>(
    from source: Source,
    to target: Target,
    by areEquivalent: (Source.Element, Target.Element) -> Bool
) where
    Source.Element == Target.Element,
    Source.Index == SourceIndex,
    Target.Index == TargetIndex
{
    self.init()
    formChanges(from: source, to: target, by: areEquivalent)
}

```  
  
空の`init`は下記になります。  
  
```swift

private init() {
    self.pathStorage = []
    self.pathStartIndex = 0
}
```  
  
`init()`  
https://github.com/apple/swift/blob/a0421124f0de37d696e503c17c05afba19f74cf2/stdlib/public/core/Diffing.swift#L291  
  
  
`formChanges`メソッドで`pathStorage`に`Endpoint`を追加していきます。  
  
`formChanges`  
https://github.com/apple/swift/blob/a0421124f0de37d696e503c17c05afba19f74cf2/stdlib/public/core/Diffing.swift#L372  
  
下記ような処理を行っています。  
  
### 最初のチェック  
  
最初のチェックとして下記の２つを行います。  
  
1. 両方のコレクションが空の場合  
  
すぐに処理を中断し、`diffrence`の結果も空になります。  
  
2. 共通の接頭辞を見つけ、かつ1つのコレクションがもう1つのコレクションの全要素を網羅している場合  
  
上記の最短経路問題で考えると  
右か下にぶつかるまで斜めに移動することができ  
その後平行か垂直移動して対角線にいきます。  
  
いくつかの例を見てイメージしてみます。  
  
```

// source = ASDFO 
// target = ASD
// targetはsourceの接頭辞になっています:

   | • | A | S | D | F | O |
 • | S |   |   |   |   |   | 
 A |   | x |   |   |   |   | // S: 開始 
 S |   |   | x |   |   |   | // T: 目的地
 D |   |   |   | x | x | T |
                 ↑
        ここで端にぶつかって平行移動している

// 最終経路:
(0,0) → (3,3) → (3,5)

// - 3つが一致 (0,0) → (3,3)
// - ２つ削除 (FとO), (3,3) → (3,5)

// source = ASD
// target = ASDFO
// sourceがtargetの接頭辞になっています:

   | • | A | S | D |
 • | S |   |   |   | // S: 開始
 A |   | x |   |   | // T: 目的地
 S |   |   | x |   | 
 D |   |   |   | x | ← ここで端にぶつかって垂直移動している
 F |   |   |   | x |
 O |   |   |   | T |

// 最終経路:
(0,0) → (3,3) → (5,3)

// - ３つが一致 (0,0) → (3,3)
// - ２つ挿入 (FとO), (3,3) → (5,3)

// source = ASD
// target = ASD
// 両方同じ要素

   | • | A | S | D |
 • | S |   |   |   | // S: 開始 
 A |   | x |   |   | // T: 目的地
 S |   |   | x |   | 
 D |   |   |   | T | ← 端にぶつかる

// 最終経路:
(0,0) → (3,3)

// - ３つが一致 (0,0) → (3,3)
```  
  
上記では２か３つの要素を持ったpathStorageができます。  
  
## formChangesCore  
  
上記の条件に合わない場合は  
下記のような状態になっています。  
  
- ２つのコレクションの要素が等しくない(byで示した条件ですべてtrueにならない)  
- ２つのコレクションが空ではない  
- 共通の接頭辞がある可能性がある  
  
この際には`formChangesCore`メソッドを呼び出します。  
  
`formChangesCore`  
https://github.com/apple/swift/blob/a0421124f0de37d696e503c17c05afba19f74cf2/stdlib/public/core/Diffing.swift#L411  
  
引数には２つのコレクションと等しいとみなすための条件に加えて  
最初のロジックで見つけた共通の接頭辞のindexです。  
これを渡すことで不要な計算をスキップします。  
  
このメソッド内では  
Greedy LCS/SES Algorithmを実装しています。  
http://www.xmailserver.org/diff2.pdf  
  
詳細は割愛しますが簡単に何をしているのかを示すと  
  
1. まずはもっとも時間がかかるように斜め移動なしでのDまでの移動手順を見つけます。  
2. この時点で目的地に到着したら計算を終了します。  
3. 到着していない場合、Dの値を一つずつ増やして再び最初から計算を開始します。  
  
この際により最適化するために前のパスをロジックに従った形で保存しておくことで  
できる限り計算の無駄をなくすようにされています。  
  
ご興味のある方は冒頭の参照記事をご覧ください。  
  
  
# まとめ  
  
Ordered Collection Diffingについて見てきました。  
使う側としては非常にシンプルに使えますが  
  
内部の複雑なアルゴリズムに支えられ  
安全に効率良く差分計算を実現されていることもわかりました。  
  
参照記事の筆者も結論で述べていますが  
これは`CollectionView`や`TableView`では使用しない方が良く  
代わりに**Diffable Data Sources**というより最適化されたAPIが  
iOS13で用意されています。  
  
詳細は下記にまとめましたのでもしよろしければご参照ください  
https://github.com/stzn/qiita/blob/master/時代の変化に応じて進化するCollectionView ~Compositional LayoutsとDiffable Data Sources~.md  
  
Swiftはまだまだ発展途上で  
たくさんのプロポーサルが出てきています。  
  
これからもどんな新しい機能が増えていくのか楽しみですね:smiley:  
  
もし間違いなどございましたら教えていただけると嬉しいです:bow_tone1:  
