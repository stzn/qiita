Sign in with Appleでは  
`ASAuthorizationAppleIDButton`を使用する必要がありますが  
このクラスはStoryboardでサポートされておらず  
コード上で追加する必要があります。  
  
そこで  
今回はCustom Classを作成することで  
StoryboardでもSign in with Appleのボタンを設定する方法について  
整理してみました。  
  
コードでの追加方法は以前↓に書かせていただきました。  
https://github.com/stzn/qiita/blob/master/【iOS】対応必須かも？Sign In with Appleまとめ.md  
  
# 実装方法  
  
## `UIButton`クラスのサブクラスを定義する  
  
`UIButton`クラスのサブクラスを定義します  
(名前は任意です)  
  
```swift

class SignInWithAppleIDButton: UIButton {
    override init(frame: CGRect) {
        super.init(frame: frame)
    }

    required init?(coder: NSCoder) {
        super.init(coder: coder)
    }
}
```  
  
## `ASAuthorizationAppleIDButton`をプロパティに保持する  
  
実際にSign in with Appleを実行するための  
`ASAuthorizationAppleIDButton`をプロパティに保持します。  
  
  
```swift

class SignInWithAppleIDButton: UIButton {

    private var appleIDButton: ASAuthorizationAppleIDButton!

    ...
}
```  
  
暗黙的にOptionalをUnwrapしていますが  
次に定義する`draw(_:)`メソッドで必ず初期化をするため  
問題はありません。  
  
## `draw(_:)`メソッドをoverrideする  
  
`draw(_:)`メソッドの中で  
ボタンを描画します。  
  
```swift

class SignInWithAppleIDButton: UIButton {
    super.draw(rect)

    appleIDButton = ASAuthorizationAppleIDButton(authorizationButtonType: .default, authorizationButtonStyle: .black)

    addSubview(appleIDButton)

    appleIDButton.translatesAutoresizingMaskIntoConstraints = false
    NSLayoutConstraint.activate([
        appleIDButton.topAnchor.constraint(equalTo: self.topAnchor),
        appleIDButton.leadingAnchor.constraint(equalTo: self.leadingAnchor),
        appleIDButton.bottomAnchor.constraint(equalTo: self.bottomAnchor),
        appleIDButton.trailingAnchor.constraint(equalTo: self.trailingAnchor),
    ])
}

```  
  
ここでは  
authorizationButtonTypeと  
authorizationButtonStyleに  
固定値を設定していますが  
後ほどカスタム可能にします。  
  
## Storyboardで設定可能にする  
  
次にStoryboardで表示できるように  
`@IBDesignable`をクラスに設定します。  
  
```swift

@IBDesignable
class SignInWithAppleIDButton: UIButton {
    super.draw(rect)

    appleIDButton = ASAuthorizationAppleIDButton(authorizationButtonType: .default, authorizationButtonStyle: .black)

    addSubview(appleIDButton)

    appleIDButton.translatesAutoresizingMaskIntoConstraints = false
    NSLayoutConstraint.activate([
        appleIDButton.topAnchor.constraint(equalTo: self.topAnchor),
        appleIDButton.leadingAnchor.constraint(equalTo: self.leadingAnchor),
        appleIDButton.bottomAnchor.constraint(equalTo: self.bottomAnchor),
        appleIDButton.trailingAnchor.constraint(equalTo: self.trailingAnchor),
    ])
}

```  
  
  
ここまででStoryboardに`UIButton`を設置してCustom Classに  
`SignInWithAppleIDButton`を指定すると  
下記のようにStoryboard上で表示させることができます。  
  
![スクリーンショット 2020-03-17 5.05.11.png](/image/64a72300-f772-9b2b-cff8-6fba094e1950.png)  
  
※ 制約などについてはAppleのHuman Interface Guidelinesに沿っています。  
こちらについては後ほど記載します。  
  
## ボタンをカスタマイズする  
  
ここからはさらに  
`@IBInspectable`を使って  
Storyboard上でボタンをカスタマイズ可能にしたいと思います。  
  
`ASAuthorizationAppleIDButton`の定義を見てみると  
  
`ASAuthorizationAppleIDButton.ButtonType`  
`ASAuthorizationAppleIDButton.Style`  
`cornerRadius`  
  
の設定が可能です。  
  
  
```swift

extension ASAuthorizationAppleIDButton {

    
    @available(iOS 13.0, *)
    public enum ButtonType : Int {

        
        case signIn = 0

        case `continue` = 1

        
        @available(iOS 13.2, *)
        case signUp = 2

        
        public static var `default`: ASAuthorizationAppleIDButton.ButtonType { get }
    }

    
    @available(iOS 13.0, *)
    public enum Style : Int {

        
        case white = 0

        case whiteOutline = 1

        case black = 2
    }
}

@available(iOS 13.0, *)
open class ASAuthorizationAppleIDButton : UIControl {

    
    public convenience init(type: ASAuthorizationAppleIDButton.ButtonType, style: ASAuthorizationAppleIDButton.Style)

    
    public init(authorizationButtonType type: ASAuthorizationAppleIDButton.ButtonType, authorizationButtonStyle style: ASAuthorizationAppleIDButton.Style)

    
    /** @abstract Set a custom corner radius to be used by this button.
     */
    open var cornerRadius: CGFloat
}
```  
  
そこでこの3つのプロパティをカスタマイズできるようにします。  
  
```swift

@IBDesignable
class SignInWithAppleIDButton: UIButton {

    ...

    @IBInspectable
    var cornerRadius: CGFloat = 6.0

    @IBInspectable
    var type: Int = ASAuthorizationAppleIDButton.ButtonType.default.rawValue

    @IBInspectable
    var style: Int = ASAuthorizationAppleIDButton.Style.black.rawValue

    ...

    override func draw(_ rect: CGRect) {
        super.draw(rect)

        let type = ASAuthorizationAppleIDButton.ButtonType.init(rawValue: self.type) ?? .default
        let style = ASAuthorizationAppleIDButton.Style.init(rawValue: self.style) ?? .black
        appleIDButton = ASAuthorizationAppleIDButton(authorizationButtonType: type, authorizationButtonStyle: style)
        appleIDButton.cornerRadius = cornerRadius

        addSubview(appleIDButton)

        appleIDButton.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            appleIDButton.topAnchor.constraint(equalTo: self.topAnchor),
            appleIDButton.leadingAnchor.constraint(equalTo: self.leadingAnchor),
            appleIDButton.bottomAnchor.constraint(equalTo: self.bottomAnchor),
            appleIDButton.trailingAnchor.constraint(equalTo: self.trailingAnchor),
        ])
    }
}
```  
  
こうするとStoryboard上で設定が変更可能になります。  
  
![スクリーンショット 2020-03-17 5.20.44.png](/image/ffc94f6c-13da-a50c-dc7c-14886a05b017.png)  
  
`IBInspectable`では`enum`が指定できないため  
`RawValue`の`Int`を設定し  
`enum`のcaseに存在しない値に関してはデフォルトの値を設定しています。  
  
`cornerRadius`の値はAppleがデフォルトで提供している値のようです。  
  
## ボタンをタップ可能にする  
  
ここまででデザインはできましたが  
ボタンをタップしても動作しません。  
  
そこでボタンがタップされたことを  
プロパティで保持している`ASAuthorizationAppleIDButton`へ  
伝えるようにします。  
  
```swift

@IBDesignable
class SignInWithAppleIDButton: UIButton {

    private var appleIDButton: ASAuthorizationAppleIDButton!

    ...
    
    override func draw(_ rect: CGRect) {
        super.draw(rect)

        ...

        appleIDButton.addTarget(self, action: #selector(appleIDButtonTapped), for: .touchUpInside)

        ...
    }

    @objc
    func appleIDButtonTapped(_ sender: Any) {
        sendActions(for: .touchUpInside)
    }
}
```  
  
`sendActions(for:)`はUIControlのメソッドで  
関連しているUIControl(今回は`UIButton`のsubViewの`ASAuthorizationAppleIDButton`)へ  
イベントを伝播させることができます。  
  
https://developer.apple.com/documentation/uikit/uicontrol/1618211-sendactions  
  
# Human Interface Guidelines  
  
上記のようにStoryboardで設定はできるようになるものの  
`ASAuthorizationAppleIDButton`はデザインに関して  
多くのガイドラインが存在します。  
  
そこでそれらが記載されている  
Human Interface Guidelines  
を見ていきたいと思います。  
https://developer.apple.com/design/human-interface-guidelines/sign-in-with-apple/overview/buttons/  
  
## 全てに共通のルール  
  
- **目立つように表示すること**  
- 他のサインインボタン(Facebook, Googleなど)よりも小さくしないこと  
- スクロールしなくてもボタンは見えるようにすること  
  
スクロールしなくてもボタンは見えるようにすることに関しては  
iPhoneSEなどの小さい端末での表示に注意する必要がありそうですね。  
  
## システムが提供するボタンを使用する場合  
  
## 3つのスタイルの注意点  
  
### White   
  
全プラットフォーム※で利用可能  
※iOS, macOS, iPadOS, watchOS  
  
- 暗い背景や十分にコントラストが与えられる色のついた背景で使用すること  
  
![image.png](/image/94b21890-e9e2-9e0c-9b90-b114d6b18bd9.png)  
参照元:https://developer.apple.com/design/human-interface-guidelines/sign-in-with-apple/overview/buttons/  
  
### White with Outline   
  
iOS, macOS, webで利用可能  
  
- 白い背景やWhiteで使うと十分にコントラストが与えられない背景色で使用すること  
- 黒のアウトラインが見辛いので暗かったり飽和した背景とは一緒に使わないこと  
  
![image.png](/image/0186df22-4c63-a00c-d338-84a9e4bc76c9.png)  
参照元:https://developer.apple.com/design/human-interface-guidelines/sign-in-with-apple/overview/buttons/  
￼￼  
### Black   
  
全プラットフォームで利用可能  
  
- 白や明るい背景と一緒に使用すること  
  
![image.png](/image/59459162-bb7a-fd11-4ae5-f9f48e57d70f.png)  
参照元:https://developer.apple.com/design/human-interface-guidelines/sign-in-with-apple/overview/buttons/  
  
※ WatchOSの場合、純粋な黒の背景に対応するために  
グレーのボタンが用意されている  
  
![image.png](/image/4a6b03e9-fd9b-ee18-6db4-bdb57f42cd7a.png)  
参照元:https://developer.apple.com/design/human-interface-guidelines/sign-in-with-apple/overview/buttons/  
  
## Button Size and Corner Radius  
  
アプリの他のボタンに合わせて調整が可能ですが  
その中でも下記のようなガイドラインがあります。  
  
![image.png](/image/d7f8e42b-b4be-4e2c-aff4-2e0bf7aad35a.png)  
https://developer.apple.com/design/human-interface-guidelines/sign-in-with-apple/overview/buttons/  
  
- iOS, macOS, webではボタンの最小サイズと最小マージンを維持すること  
  
ロケールでボタンタイトルの長さが変わるので注意が必要  
  
![スクリーンショット 2020-03-18 5.04.33.png](/image/5522ef7a-7fd9-93f3-24e0-935e1197fef8.png)  
参照元:https://developer.apple.com/design/human-interface-guidelines/sign-in-with-apple/overview/buttons/  
  
## カスタムボタンを作成する場合   
  
iOS, macOS, Webで利用可能  
  
Apple Design Resourcesからロゴのダウンロードが可能  
https://developer.apple.com/design/resources/  
  
### ロゴを使用する場合のガイドライン  
  
- ロゴ自体をボタンとして使用しないこと  
- ロゴファイルの高さをボタンの高さに合わせること  
- 切り取らないこと  
- 垂直方向(vertical)のpaddingを追加しないこと  
- 独自のカラーを使用しないこと  
  
  
## 左寄せロゴボタンを作成する場合  
  
### ボタンの高さを基にロゴファイルのフォーマットを選択する  
  
SVGとPDFはベクター形式のなのでどの高さでも利用できるが、PNGは44ptの高さのみで使用すること  
44ptはiOSでのデフォルトの高さで推奨される高さ  
  
### タイトル※にはシステムのフォントを利用すること  
  
※ Sign in with Apple, Sign up with Apple, or Continue with Apple  
  
### タイトルのフォントサイズはボタンの高さの43%  
  
逆を言えばボタンの高さはフォントサイズの233%  
  
サイズの例  
![image.png](/image/47c508d0-59d1-ac88-2a2e-43ef86b05bfd.png)  
参照元:https://developer.apple.com/design/human-interface-guidelines/sign-in-with-apple/overview/buttons/  
  
### タイトルの大文字、小文字を変えない  
  
- 最初の一文字を大文字にすること　  
  
具体的にはSign, Continue, Appleは最初が大文字  
それ以外は全て小文字にする  
  
### タイトルとボタンの右端の間のマージンをボタンの幅を最低8%以上にする  
  
### ボタンのサイズと周りのマージンを最小以上にする  
  
システムで提供されているものと同様のガイドラインです。  
￼  
![スクリーンショット 2020-03-18 5.04.33.png](/image/5522ef7a-7fd9-93f3-24e0-935e1197fef8.png)  
参照元:https://developer.apple.com/design/human-interface-guidelines/sign-in-with-apple/overview/buttons/  
  
## ロゴのみのボタンを作成する場合  
  
### ボタンのサイズを基にロゴファイルのフォーマットを選択する  
  
- ボタンの高さを基にロゴファイルのフォーマットを選択する  
- SVGとPDFはベクター形式のなのでどの高さでも利用できるが、PNGは44ptx44ptでのサイズでのみ使用すること  
  
### 右側に平行方向のpaddingを入れない  
  
- ロゴのみの画像はすでに適切なpaddingが全てのサイドに入っているので変更はしないこと  
- アスペクト比は1:1を維持すること  
  
### デフォルトの四角い形を変更したい場合はMaskを利用すること  
  
- 元の画像を切り取ったりPaddingを追加してはいけない  
  
### ボタンの周りのマージンは最低ボタンの高さの1/10以上にする  
  
# まとめ  
  
Sign in with Appleで使用するボタンについて  
Storyboardでの設定方法について見てみました。  
  
画面に関してStoryboardを使用している場合には  
他のパーツと一緒にデザインできるので  
役に立つのかと思っています。  
  
一方でしっかりと使用する際のガイドラインも存在し  
そこを守らないとアプリの審査でリジェクトされる可能性もあるため  
しっかりとチェックする必要がありますね。  
  
もし何か間違いなどございましたらご指摘ください🙇🏻‍♂️  
  
# 参照記事  
  
https://developer.apple.com/documentation/authenticationservices/implementing_user_authentication_with_sign_in_with_apple  
https://developer.apple.com/design/human-interface-guidelines/sign-in-with-apple/overview/buttons/  
https://developer.apple.com/library/archive/referencelibrary/GettingStarted/DevelopiOSAppsSwift/ImplementingACustomControl.html  
https://medium.com/swlh/how-to-use-asauthorizationappleidbutton-in-storyboard-653f9cd94817  
