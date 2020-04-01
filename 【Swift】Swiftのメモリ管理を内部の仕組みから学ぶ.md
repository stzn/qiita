普段コードを書いている時に  
Swiftが内部で  
どのようにオブジェクトを管理しているのかについて  
考えることはあまり多くないかもしれません。  
  
しかし  
非同期処理を扱う場合など  
  
```swift
DispatchQueue.main.async { [weak self] in
    ...
}

```  
  
などのように  
`weak`といった  
キーワードを使用することは多くあると思います。  
  
これは「弱参照」と呼ばれ  
直接の参照(強参照)を持たないように  
Swiftに内部に指示をして  
循環参照を起こさないための仕組みです。  
  
こういった適切なメモリ管理を行わないと  
メモリが解放されないことでランダムにクラッシュが起こるなど  
原因がわかりづらい不具合を発生させる可能性があります。  
  
そこで今回は  
`weak`などを使用することで  
Swiftが何をしているのかを  
内部の仕組みから見ていくことで  
  
`weak`や`unowned`の使い方や  
オブジェクトのライフサイクルについて  
学んでみたいと思います。  
  
  
# メモリの3つ仮想的な領域  
  
メモリ自体はただのバイトの配列ですが  
プログラミングという観点ですと  
3つの領域に分けることがよくあります。  
  
- スタック領域 全てのローカル変数を保管している場所です。  
- グローバルデータ 静的な変数や定数や型のメタ情報を保管している場所です。  
- ヒープ領域　オブジェクト※を保管している場所です。  
　　　　　　　  
※   
実行時にメモリを割り当てられ  
ある時点で解放される「寿命」を持つものを指します。  
Swiftでは主に参照型(reference type)を指します。  
  
# ARC(Automatic Reference Counting)  
  
メモリ管理には「オーナーシップ(所有権の帰属)」という概念が大切になります。  
これはあるオブジェクトを誰が解放する責務を持っているかということを意味します。  
  
詳細に関しては下記のOwnershipManifestに記載されています。  
https://github.com/apple/swift/blob/master/docs/OwnershipManifesto.md  
  
このオーナーシップを管理する仕組みとして  
Swiftは**ARC**を使用しています。  
  
ARCはオブジェクトの参照されている数を保持しておき  
カウントがゼロになるとメモリから自動で解放される仕組みです。  
  
# ２つのレベルの参照(強参照と弱参照)  
  
Swiftでは参照に2つのレベルがあります。  
それが冒頭でも登場した  
**強参照(strong reference)**  
と  
**弱参照(weak reference)**  
です。  
  
さらに  
弱参照の派生として**非参照(unowned reference)**があります。  
  
  
## 強参照(strong reference)  
  
Swiftでは  
強参照がある限り  
オブジェクトは生存し続けることができ  
逆になくなるとメモリから解放されます。  
  
SwiftではJavaのガベージコレクションのように  
自動でメモリを解放する仕組みは持っていないため  
循環参照を引き起こす可能性があります。  
  
これは例えば  
オブジェクトAとオブジェクトBが  
お互いにお互いのオブジェクトへの参照を持っている状態です。  
  
これを防ぐためにも`weak`などの記述が必要になってきます。  
  
  
## 弱参照(weak reference)  
  
弱参照を使用することで  
循環参照を断ち切ることができますが  
弱参照を持っていたとしても  
強参照がなくなればオブジェクトはメモリから解放されてしまいます。  
  
その場合  
参照しているオブジェクトを使おうとしても`nil`になっています。  
  
そのため`weak`を使用する時は`Optional`で扱われます。  
  
## 非参照(unowned reference)  
  
弱参照とほぼ同じですが  
参照しているオブジェクトを使おうとすると  
assertionエラーとなってプログラムはクラッシュします。  
  
`unowned`は参照がなくならないと想定しているのに  
予期せず参照が解放されてしまっている不具合を発見するのに役立ちます。  
  
`unowned`を使用する基準としては  
参照するオブジェクトと参照されるオブジェクトの寿命が同じような時です。  
  
例えばクラスの中で`lazy`をつけた変数を定義する場合に  
  
```swift

class ViewController: UIViewController {
   lazy var label: UILabel = { [unowned self] in
      self.someSetup()
      ...
   }() 
}
```  
  
これはViewControllerクラスの変数labelが  
self(ここではViewController)を参照しています。  
  
これはクラスオブジェクトが解放されるタイミングで  
label変数も解放されるので  
selfの参照がなくなることはありません。  
  
  
参照については下記のドキュメントに詳細が記載されています。  
https://github.com/apple/swift/blob/master/docs/weak.rst  
  
  
# Swift Runtime  
  
ARCはSwift Runtimeというライブラリで実装されています。  
  
他にもSwift Runtimeでは  
実行時にジェネリクスやプロトコルを具体的な型に解決する  
などの重要な機能を有しています。  
  
https://github.com/apple/swift/blob/master/docs/Runtime.md  
  
全てのオブジェクトは  
**`HeapObject`**という`struct`  
で表現されます。  
  
https://github.com/apple/swift/blob/4fd0671e542299d7805e41cf9426640ab3b399af/stdlib/public/SwiftShims/HeapObject.h  
  
HeapObjectはオブジェクトの型のメタ情報と参照カウント(RefCount)を持っています。  
RefCountには`strong`と`weak`と`unowned`用の３種類があります。  
https://github.com/apple/swift/blob/master/stdlib/public/SwiftShims/RefCount.h  
  
SwiftのコンパイラはSIL生成段階(SIL Generation)※で  
`swift_retain()`  
`swift_release()`  
というメソッドを適切な場所に差込みます。  
  
これによって`HeapObject`の作成や解放がされます。  
  
※  
SIL生成段階(SIL Generation)はSwiftのコンパイル時の一つのフェーズです。  
下記のドキュメントに詳細が記載されています。  
https://swift.org/compiler-stdlib/#compiler-architecture  
  
## Side Table  
  
全てのオブジェクトは  
弱参照(weak reference)用のRefCountを持っているものの  
多くのオブジェクトで弱参照を持っていません。  
そこで弱参照用のRefCountにメモリを割り当てても無駄になることが多いため  
弱参照の情報はSide Table※という別の場所に保管され  
本当に必要になった時にメモリが割り当てられるようになっています。  
  
※   
正式には`HeapObjectSideTableEntry`です。  
内部ではオブジェクトのポインタとRefCountを持っています。  
https://github.com/apple/swift/blob/master/stdlib/public/SwiftShims/RefCount.h#L1199  
  
弱参照が示すメモリアドレスは参照したいオブジェクトではなく  
このSide Tableを示しています。  
  
これによって  
無駄なメモリが必要なくなることに加え  
直接オブジェクトを参照していないため  
オブジェクトの解放と弱参照が参照するタイミングが競合することなく  
弱参照を取り除くことができます。  
  
# オブジェクトのライフサイクル  
  
下記のコメントを参考にすると  
https://github.com/apple/swift/blob/master/stdlib/public/SwiftShims/RefCount.h#L112  
  
Swiftのオブジェクトは3つの参照の保持の仕方によって  
状態を5つに分けることができます。  
  
- LIVE  
- DEINITING  
- DEINITED  
- FREED  
- DEAD  
  
  
簡単に図にすると下記のように状態が変化していきます。  
  
![名称未設定ファイル (2).png](/image/731c7f9e-da7b-e1b1-d918-595e68bbc20d.png)  
  
  
次にそれぞれの状態について  
見ていきます。  
  
各状態で弱参照(Side Table)があるかないかで  
挙動や次に遷移する状態が変化していきます。  
  
## LIVE without side table  
  
#### オブジェクト   
生存している。  
  
#### 参照カウント   
強参照1 非参照1 弱参照1で初期化される。  
  
#### Side Tableと弱参照カウントのメモリ割り当て  
なし 。  
  
#### 強参照変数の操作  
正常に機能する  
  
#### 非参照変数の操作  
正常に機能する。  
  
#### 弱参照変数を使用した時の挙動  
起こり得ない。  
  
#### 弱参照変数へのオブジェクトの代入  
Side Tableを追加する。  
LIVE with Side Table状態になる。  
  
#### 次の状態への遷移   
参照がゼロになった時に  
deinitが呼ばれDEINITINGの状態になる。  
  
## LIVE with side table  
  
#### 弱参照変数の操作  
正常に機能する。  
  
それ以外は  
LIVE without side tableと同じ。  
  
## DEINITING without side table  
  
#### オブジェクト   
`deinit()`を実行中。  
  
#### 強参照変数の操作  
何も起こらない。  
  
#### 非参照変数を使用した時の挙動  
`swift_abortRetainUnowned()`で処理が中断されます。  
https://github.com/apple/swift/blob/ebcbaca9681816b9ebaa7ba31ef97729e707db93/include/swift/Runtime/Debug.h#L122  
  
https://github.com/apple/swift/blob/master/stdlib/public/runtime/Errors.cpp#L451  
  
#### 非参照変数へオブジェクトを代入した時の挙動  
正常に機能する。  
  
#### 弱参照変数を使用した時の挙動  
起こり得ない。  
  
#### 弱参照変数へオブジェクトを代入した時の挙動  
nilが代入される  
  
#### 次の状態への遷移  
  
参照がゼロになった時の挙動  
`deinit()`が完了し  
`swift_deallocObject()`というメソッドが呼ばれる。  
  
`canBeFreedNow()`で弱参照または非参照があるかどうかをチェックする。  
`canBeFreedNow`がtrueの場合  
オブジェクトは解放されてDEINITEDの状態になる。  
  
## DEINITING with side table  
  
#### 弱参照変数を使用した時の挙動  
nilを返却する。  
  
#### 弱参照変数へオブジェクトを代入した時の挙動  
nilが代入される。  
  
`canBeFreedNow()`は常にfalseになり  
そのままDEAD状態にはならない。  
  
その他はDEINITING without side tableと同じ。  
  
## DEINITED without side table  
  
#### オブジェクト   
`deinit()`は完了しているが  
非参照は存在している。  
  
#### 強参照変数の操作  
起こり得ない。  
  
#### 非参照変数を使用した時の挙動  
`swift_abortRetainUnowned()`の中でロードを停止している。  
  
#### 非参照変数へオブジェクトを代入した時の挙動  
起こり得ない。  
  
#### 弱参照変数の操作  
起こり得ない。  
  
#### 次の状態への遷移  
  
非参照カウントがゼロになった時  
オブジェクトは解放されてDEAD状態になる。  
  
## DEINITED with side table  
  
#### 弱参照変数を使用した時の挙動  
nilが返却される。  
  
#### 弱参照変数へオブジェクトを代入した時の挙動  
起こり得ない。  
  
#### 次の状態への遷移  
  
非参照カウントがゼロになった時  
オブジェクトは解放されて弱参照カウントが減り  
オブジェクトはFREED状態になる。  
  
他はDEINITED without side tableと同じ  
  
  
## FREED without side table  
  
起こり得ない状態。  
  
## FREED with side table  
  
#### オブジェクト  
オブジェクトは解放されているが  
弱参照がSide Tableに残っている状態。  
  
#### 強参照の操作  
起こり得ない。  
  
#### 非参照の操作  
起こり得ない。  
  
#### 弱参照変数を使用した時の挙動  
nilが返却される。  
  
#### 弱参照変数へオブジェクトを代入した時の挙動  
起こり得ない。  
  
#### 次の状態への遷移  
  
弱参照カウントがゼロになった時  
Side Tableオブジェクトは解放され  
オブジェクトはDEAD状態になる。  
  
## DEAD  
オブジェクトもSide Tableもなくなっている。  
  
# まとめ  
  
Swiftの内部の仕組みから  
メモリ管理について見ていきました。  
  
普段はあまり意識していませんでしたが  
こういう知識を知っていることで  
原因がわからない不具合などの解決にも  
役に立つことがあるかもしれません。  
  
また  
XcodeにはMemoryDebuggerもあり  
そのグラフを理解するのに役に立つかもしれません💡  
  
もし何か間違いなどございましたら  
ご指摘頂けましたらうれしいです🙇🏻‍♂️  
  
# 参考記事  
  
https://www.vadimbulavin.com/swift-memory-management-arc-strong-weak-and-unowned/  
