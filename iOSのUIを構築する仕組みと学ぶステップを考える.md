過去を振り返って  
iOSを学びはじめて一番最初に戸惑ったことは  
  
**どうやってUIを作成するのか？**  
  
ということでした。  
  
最初Xcodeでプロジェクトを作成すると  
Main.storyboardがあり  
Storyboardを使ってUIを作成していくものだと思いましたが  
  
色々なサイトで情報を調べてみると  
  
- コードレイアウト  
- Xib(Nib)  
- AutoLayout  
- autoResizingMask  
  
など色々なUIの構築方法が出てきて  
結局何が良いのかがわからなくなりました。  
  
今回は  
これからiOSを学ぶ人向けへの  
UIの構築方法のまとめ記事があったので  
それを参考にして  
  
**どのようなUIを構成する方法があるのか？**  
**どういう時にどの方法が選ばれているのか？**  
**何を学び、どう学ぶのか？**  
  
について見ていきたいと思います。  
  
# Xcodeの始まり  
  
AppleのSDKは  
1997年のスティーブジョブズの２番目のスタートアップの「NeXTSTEP」に由来し  
そこから約30年もの間継続的に改良を続けてきました。  
[こちらの動画](https://www.youtube.com/watch?v=dl0CbKYUFTY)は  
Interface Builder(以降IB)のデモ動画でXcodeのルーツとなるものです。  
  
Xcodeはその後改良が積み重ねられていますが  
根本的な要素は同じです。  
  
# ２種類のUIの構築方法  
  
いくつかUIを作成する方法はありますが  
大きく分けて2つのパターンに分かれます。  
- グラフィカルなデザインツールを使用する。XcodeのIBを使って紙にステッカーを貼るようにUIの構成要素を貼っていきます。  
- コードを中心にコードで個々の要素のインスタンスを生成し正しい位置に配置されるようにレイアウトします。  
  
分類をグラフにすると下記のようになります。  
  
![UIHistory.png](/image/5135e4a8-ecef-25af-b281-3cc4a1279882.png)  
  
ここからは個々について見ていきます。  
  
# XIB  
  
macOSのアプリを作成するための一番最初のGUIのツールです。  
XIBは「XML Interface Builder」の略で  
かつてのNIB(NeXT Interface Builder)を継承しています。  
  
![スクリーンショット 2019-11-30 7.57.31.png](/image/069df53e-8764-7f54-2adf-4bac6d176318.png)  
  
XIBはアプリがビルドされる際にNIBへと変換されます。  
NIBファイルはアプリのバンドルの中に配置され  
実行時にUIKitによって読み込まれ  
様々なUIのオブジェクトのインスタンスを生成と設定を行います。  
  
  
現在ではほとんどの役割をStoryboardに取って代わられていますが  
TableViewやCollectionViewのCellや他の似た様なUIの要素を構築するのには有用です。  
  
特にTableViewやCollectionViewはNIBに対しての特別なメソッドも存在します。  
https://developer.apple.com/documentation/uikit/uitableview/1614937-register  
  
  
XIBファイルは多くの場合にファイルの~Owner~としてトップレベルのオブジェクトを有しています。  
多くの場合  
これはUIViewControllerのサブクラスであったり  
UITableViewCellやUICollectionViewCellであったりしますが  
特に特別な制約はなく  
Objective-Cに互換性のあるどんなオブジェクトも~Owner~として使用できます。  
  
より具体的には  
`NSCoding`プロトコルに適合しているクラスでしたら可能なため  
特にXIBファイル用の特別なクラスを用意する必要はありません。  
  
https://developer.apple.com/documentation/foundation/nscoding  
  
  
# Storyboard  
  
エンジニアの中には  
  
**XIB + Flow = Storyboard**  
  
と考える人もいるように  
Storyboardは  
XIBで構成する複数のスクリーンと  
それらの画面遷移を`Segue`  
で構成します。  
  
![スクリーンショット 2019-11-30 7.57.14.png](/image/d17571b4-ea16-1491-71fa-1a9e4c47b965.png)  
  
## アプリの流れが把握しやすくなった  
  
例えばボタンをタップした時に他の画面へ遷移する場合は  
下記のようにStoryboard上でそれを構成することができます。  
  
![スクリーンショット 2019-11-30 8.00.49.png](/image/a4f269fc-9397-db84-c54e-f379e4f05673.png)  
  
Storyboardの登場によって  
画面もグラフィカルに見ることができるに加え  
アプリの全体的な動き(Flow)もひと目で把握できるようになりました。  
  
## LaunchScreen  
  
プロジェクトを作成した際にLaunchScreen.storyboardというファイルがあります。  
これはアプリが完全に起動する前にユーザーに表示されるデフォルトのイメージを設定できます。  
  
<img width="1072" alt="スクリーンショット 2019-11-30 8.07.22.png" src="/image/1bd67f25-76b8-6f46-b8ae-3e5c99f3924c.png">  
  
これはinfo.plistで他のStoryboardに変更が可能です。  
  
![スクリーンショット 2019-11-30 8.08.00.png](/image/57f55b09-13e3-f3ed-e14d-4b09d74bed0d.png)  
  
このLaunchScreenでは  
アプリのデフォルトの状態を表す  
「土台」的な画面がよく使用されています。  
(アプリのロゴを真ん中に表示するなど)  
  
元々は  
デバイスのサイズごとに複数のPNG画像を  
用意していましたが  
iOSやiPadOSで様々な解像度をサポートするようになったことから  
storyboardを使って自動で調整されるようにすることが  
推奨されています。  
  
一方で  
このLaunchStoryboardには多くの制限があります。  
  
例えば  
システムで用意されているクラスを使用しなければならず  
カスタムクラスを設定することができません。(UIViewControllerのサブクラスなど)  
これはアプリが完全に起動する前に表示される画面のため  
アプリ内で定義されているクラスの準備がまだできていないからです。  
  
また  
ローカライズが必要な要素など  
起動時に動的に決定されるようなものも指定できません。  
  
```
Always create a final version of the interface you want. 
Never include temporary content or content that requires localization. 
```  
  
https://developer.apple.com/documentation/uikit/app_and_environment/responding_to_the_launch_of_your_app  
  
  
# Spring and Strut  
  
これはiOSのViewをレイアウトするための独自の方法です。  
  
![スクリーンショット 2019-11-30 8.30.41.png](/image/9a7b5eef-2a63-1279-38b7-8eae11737a11.png)  
  
**Spring**または**Strut**という  
親のViewの変化に応じてどのようにViewを拡大縮小されるかを  
決める2つの仕組みがあります。  
  
### Spring  
  
親Viewがリサイズされるのに比例してViewをリサイズします。  
  
### Strut  
  
親Viewとの距離を保つようにします。  
  
拡大縮小できるのは下記の6つになります。  
  
- Subviewの高さ(Spring)  
- Subviewの幅(Spring)  
- SubViewの上下左右4つのmargin(Strut)  
  
下記のようにIB上でどのように動くのかを見ることができます。  
  
![画面収録-2019-11-30-8.37.24.gif](/image/dbe7b48c-98a8-937f-12c0-3fcad59ac401.gif)  
  
この場合は  
Springは高さ、幅両方に設定されているため親Viewの拡大縮小に合わせて変化します。  
Strutは上と左に設定されているため上と左の距離は保っています。  
  
また  
`autoresizingMask`という  
プロパティからコードで設定することも可能です。  
  
https://developer.apple.com/documentation/uikit/uiview/1622559-autoresizingmask  
  
このSpringとStrutはかなり歴史の古いもので  
新しいプロジェクトはほとんどAutoLayoutに置き換えられています。  
実際UIKitは`autoresizingMask`をAutoLayoutの制約に内部で変換しています。  
  
※  
コードレイアウトを行う際など  
この変換が不要な場合は  
`translatesAutoresizingMaskIntoConstraints`  
をfalseにします。  
  
https://developer.apple.com/documentation/uikit/uiview/1622572-translatesautoresizingmaskintoco  
  
# Auto Layout  
  
iOS6で登場しUIKitの要素を配置する最も好ましい仕組みとされています。  
https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/index.html  
  
AutoLayoutの基本設計は1998年の論文を元にされています。  
https://constraints.cs.washington.edu/cassowary/  
  
AutoLayoutは  
あるSubviewの属性(attribute)が  
他のSubviewや親Viewの属性と  
どのように関連しているのかを表す制約(Constraint)を  
宣言的に作成します。  
  
これらの制約は線形方程式でこの方程式が解かれるタイミングで  
Subviewの位置と大きさが決まります。  
  
つまり  
この線形方程式はUIWindowをルートにした  
それぞれのViewの階層において解決される必要があります。  
  
逆に言うと  
この方程式が解決できない場合にレイアウトの設定に失敗します。  
Consoleにエラーのログが出てくるのはこれが原因です。  
  
Viewのレイアウトを決定する方法は主に2つあります。  
  
### NSLayoutConstraint  
  
一つの制約を表現するオブジェクトで  
これを生成して親Viewに追加していきます。  
  
### Visual Format Language  
  
StringベースのDSLで一つの宣言で複数の制約を設定できます。  
  
https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/VisualFormatLanguage.html  
  
AutoLayoutは複雑になってしまう場合がありますが  
iOS9で登場した`UIStackView`を利用することで  
横並びまたは縦並びのSubview同士のレイアウトは  
自動で設定してくれるなど  
自分で制約を特定する必要も減ってきています。  
  
# UIKit Dynamics  
  
これまでに紹介してきたものは全て「静的な」レイアウトの設定をするものでしたが  
UIKit Dynamicsは名前の通り「動的な」レイアウトを設定します。  
  
https://developer.apple.com/documentation/uikit/animation_and_haptics/uikit_dynamics  
  
これは2次元の物理演算エンジンで  
スクリーン上であたかも本当の物質に触れているような動きを  
シミュレーションします。  
  
UIKit DynamicsはAutoLayoutに類似した制約を作成しますが  
位置やサイズを固定する代わりに対象のものを動かすために設定されます。  
そのためUIに一定レベルの動きを生み出します。  
  
例えば下記のような動きをUIKitが自動で引き起こしてくれます。  
  
![画面収録-2019-11-30-9.34.23_480.gif](/image/8bfc6565-c3d8-59dd-02fc-00be98b4572f.gif)  
  
  
<details><summary>コード例</summary>  
<div>  
  
```swift

import UIKit

class ViewController: UIViewController {

    @IBOutlet private weak var snapView: UIView!
    private var animator: UIDynamicAnimator!
    private var snapBehavior: UISnapBehavior!

    override func viewDidLoad() {
        super.viewDidLoad()

        animator = UIDynamicAnimator(referenceView: view)
        snapBehavior = UISnapBehavior(item: snapView, snapTo: view.center)
        animator.addBehavior(snapBehavior)

        let panGesture = UIPanGestureRecognizer(target: self, action: #selector(pannedView))
        snapView.addGestureRecognizer(panGesture)
        snapView.isUserInteractionEnabled = true
    }

    @objc
    private func pannedView(recognizer: UIPanGestureRecognizer) {
        switch recognizer.state {
        case .began:
            animator.removeBehavior(snapBehavior)
        case .changed:
            let translation = recognizer.translation(in: view)
            snapView.center = CGPoint(x: snapView.center.x + translation.x,
                                      y: snapView.center.y + translation.y)
            recognizer.setTranslation(.zero, in: view)

        case .ended, .cancelled, .failed:
            animator.addBehavior(snapBehavior)
        case .possible:
            break
        @unknown default:
            break
        }
    }
}
```  
</div></details>  
  
注意する点としては  
UIKit DynamicsとAuto Layoutの制約は  
お互いに**干渉しません**。  
  
一般的なAuto Layoutは全てのSubviewの大きさを取り扱うべきで  
UIKit DynamicsはそのSubview達が親Viewの中で  
どのように配置されるのかを取り扱います。  
  
※ UIKit DynamicsとAuto Layoutの両方が適用された場合です。  
  
# Manual Layout  
  
これまではUIKitにサイズや位置を決めてもらえるように  
設定をしてきましたが  
手動で計算をすることもできます。  
  
一番シンプルな方法は  
`layoutSubViews()`をoverrideする中で  
全てのSubviewsの`frame`に値を設定することでできます。  
https://developer.apple.com/documentation/uikit/uiview/1622482-layoutsubviews  
  
`intrinsicContentSize`や`systemLayoutSizeFitting(_:)`は  
UIKitが自動計算した結果ですが  
このような値も参照することで手動で計算する際に活用できます。  
  
https://developer.apple.com/documentation/uikit/uiview/1622600-intrinsiccontentsize  
https://developer.apple.com/documentation/uikit/uiview/1622624-systemlayoutsizefitting  
  
Manual Layoutはおそらく最も強力なレイアウト方法ではありますが  
最も難しい方法でもあります。  
  
しかし  
カスタムUIControlを作成する時など  
Manual Layoutが必要な場合もしばしば見られます。  
  
https://developer.apple.com/documentation/uikit/uicontrol  
  
# SwiftUI  
  
iOS13で登場したUIを構築する新しいフレームワークです。  
SwiftUIには下記のような特徴があります。  
  
## Declarative(宣言的)  
  
「宣言的」というのは各要素がどういったものかを「宣言」するだけで  
どうやってそれらを組み合わせていくのかはフレームワークに任せるような方法です。  
  
Manual LayoutよりもAuto Layoutの制約の設定に近い形で  
コードを記載していきます。  
  
## Binding  
  
これはデータとそれに応じて変化するUIが双方向に繋がっていて  
どちらかが変化するとそれと同期してもう一方も変更されることで  
不整合な状態を防ぐことができます。  
  
フレームワークがこれを内部で実行するため  
コードで記載する必要もありません。  
  
元々macOS用のIBでは可能でしたが  
SwiftUIの登場で他のプラットフォームでも可能になりました。  
https://developer.apple.com/documentation/appkit/nsobjectcontroller  
  
## まだまだこれから進化していく段階  
  
SwiftUIにはUIKitの機能を完全にカバーし切れておらず  
UIKitと並列で使用する機会多いように見られます。  
  
しかし  
SwiftUIの開発が進み  
将来的にはUIKitの全ての機能を網羅した際には  
UIKitに取って代わるかもしれません。  
  
# いつ何を使うのか？  
  
これまで見てきた仕組みにはそれぞれ長所と短所があります。  
それに応じて使い所や使い方も変わってきます。  
  
## UIの構築  
  
### StoryboardとXIBで構築する  
  
Storyboardは一連のUIとFlowを一緒に構築できます。  
これによって他の開発者もアプリ流れや構成を理解しやすくなります。  
  
XIBはほとんどStoryboardに取って代わられましたが  
TableViewやCollectionViewのCellの作成や  
複数のStoryboardで繰り返し使用するようなカスタムViewを生成にも適しています。  
  
Cellが一種類の場合はStoryboard上で直接レイアウトをする方が簡単ですが  
複数種類のCellを表示する必要がある場合などは  
XIBで個々のCellを作成した方が管理がしやすくなります。  
  
### コードで構築する  
  
大規模なiOSアプリの開発ではコードでレイアウトを書くケースも多くあります。  
例えば複数人で同じ画面の機能を実装する可能性があります。  
  
この場合に  
同じ箇所に変更を加えると  
GitのようなSource Controlシステムで  
コンフリクトが発生します。  
  
コードの場合はある程度自動でコンフリクトを解消してくれますが  
StoryboardやXIBはXMLベースのファイルで  
システムがどの解決方法が正しいのかを判断できず  
コンフリクトを解決するのは非常に困難です。  
  
結果として手動で解決しなければならなくなりますが  
ファイル自体が人が理解できるような形で記載されていないため  
解読に時間も労力もかかってしまいます。  
  
## レイアウト  
  
Auto Layoutをデフォルトで使用するべきです。  
SpringとStrutは  
iOS5以前をサポートしている場合など特別な理由がない限り  
使用する理由は見つかりません。  
  
一方でAuto Layoutだけでは表現しきれない時は  
Manual Layoutは現実的な選択肢になります。  
  
例えば  
Auto Layoutの制約が有効な範囲は  
親Viewとその直接のSubview(子View)との間だけで  
SubviewのSubview(孫View)には効きません。  
こういう場合にManual Layoutが必要になります。  
  
## アニメーション  
  
ユーザの動きに合わせたアニメーションが必要な際は  
UIKit Dynamicsを使用すると  
Appleの提供するアプリのような動きを実現することができます。  
  
例えばApple Storeのアプリで  
アプリを選択したときのズームしてバウンドする動作は  
Core Animationで手動で実装しなくても  
UIKit Dynamicsを使って実現できます。  
  
## SwiftUI  
  
もしiOS13以降をターゲットをしているならば  
入力フォームなどはSwiftUIのBindingがとても役に立ちます。  
  
一方で  
まだまだ不足している機能や不具合と思われる動きもあり  
全てをSwiftUIにするのは難しい段階であります。  
  
rootViewControllerやトップレベルのViewControllerには  
UIKitを使用しその先の部分的な機能にSwiftUIを活用していくのが  
良いのかと思われます。  
  
  
# 何から学び、どう学ぶ？  
  
最後にこれらの仕組みの中で  
何を学びまたどうやって学ぶのかについて考えたいと思います。  
  
もしiOSエンジニアとして働きたいと思うならば  
上記で紹介したもの全てに関して学ぶ必要があります。  
  
参加した開発プロジェクトで  
SpringとStrutを使用している可能性もありますし  
手動で位置やサイズを計算しているかもしれません。  
  
## 学習ステップ  
  
SwiftUIに関しては少し控えておいて良いでしょう。  
多くのアプリでは  
２世代前のOSバージョンまでサポートしていることが多く  
SwiftUIを本格的に使用するのは  
2年以上先になる場合が多いのではないかと考えられます。  
  
### StoryboardでAuto Layoutを利用する  
  
学習を開始してそれ用のアプリを作ろうとした時は  
まずStoryboard上でAuto Layoutを使ってUIを構築してみましょう。  
その際にUIStackViewも効果的に使えるように学んでいきましょう。  
  
### コードでAuto Layoutを利用する  
  
次にVisual Format Languageや  
NSConstraintでのAuto Layoutの設定方法についても学びましょう。  
  
Storyboardに加えてコードで設定する方法を学ぶことで  
Auto Layoutがどのように働くのかが把握できるようになります。  
  
### コードでAuto Layoutを動的に変更する  
  
次にアプリの実行中に動的に制約を変更する方法を学びます。  
これはNSConstraintで個々の制約を作成し  
`constant`プロパティを変更することで実現できます。  
  
https://developer.apple.com/documentation/uikit/nslayoutconstraint/1526928-constant  
  
制約を追加したり削除することで  
レイアウトがどのように変化するかも学ぶことができます。  
  
animationブロックで囲むことでレイアウトの変化を  
視覚的に見ることもできるようになります。  
  
### UIKit Dynamicsを利用する  
  
コードでのAuto Layoutの設定に慣れてきたら  
UIKit Dynamicsについて学んでみましょう。  
制約をどのように設定することで  
上手くアニメーションするのかを学びましょう。  
  
Apple Storeアプリの動きなどは  
良いお手本になります。  
  
デバイスのスクリーンの録画などもできますので  
録画して比較してみるのも良いかもしれません。  
https://developer.apple.com/documentation/uikit/nslayoutconstraint/1526928-constant  
  
### SwiftUIを学び始める  
  
これらの学習の上にSwiftUIを触ってみましょう。  
全ての要素が含まれているUIViewControllerを実装しているのに近い感覚で  
使ってみましょう。  
使う際はより小さく明確な機能を扱うコンポーネントとして使用しましょう。  
ユーザがデータを入力するようなFormなどのUIで力を発揮します。  
  
`UIHostingController`を利用することで  
UIViewControllerの階層に追加することもできます。  
https://developer.apple.com/documentation/swiftui/uihostingcontroller  
  
`UIViewControllerRepresentable`や  
`UIViewRepresentable`に適合することで  
UIViewとして利用することも可能になります。  
https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable  
https://developer.apple.com/documentation/swiftui/uiviewrepresentable  
  
# まとめ  
  
iOSでUIを構築する仕組みについて見ていきました。  
  
歴史を重ねるに連れて  
使われている仕組みは進歩していますが  
どんなものにでも使える方法というものは確立されておらず  
それぞれの長所や短所を考慮して  
色々な仕組みは利用されているプロジェクトに出会うこともあると思います。  
  
そこで  
全体的な知識や経験を持っていることで  
開発をスムーズに進めることができるのではなるでしょう。  
  
大事なのは  
学習して使ってみて感覚として知ることですので  
少しづつでも触ってみるのがよいのかなと思います😃  
  
もし何か間違いなどございましたらご指摘頂けましたら幸いです🙇🏻‍♂️  
  
# 参考記事  
https://cutecoder.org/programming/newbie-learn-ios-user-interface-programming/  
https://medium.com/@raulriera/uikit-dynamics-in-the-real-world-ef0dfd924260  
