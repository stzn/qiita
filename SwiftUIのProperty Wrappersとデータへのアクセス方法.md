- 2019/7/17 `@GestureState`について追記しました。  
- 2019/7/19 BindableObjectがdidChange->willChangeなど変更されていたので修正しました。  
https://developer.apple.com/documentation/swiftui/bindableobject  
- 2019/7/30 Xcode11 Beta5でBindableObjectはdeprecatedになり`ObservableObject`を使うようになったので追記しました。`@ObjectBinding`も`@ObservedObject`に名前が変わりました。  
- 2019/7/31 `ObservableObject`がCombineに含まれること、`Identifiable`はStandard Libraryに移動したことについて記載しました。また`@Published`をつけるとwillChangeなどの記載が不要になりました。  
- 2019/9/18 Property WrappersのwrappedValueとprojectedValueについて追記しました。  
  
  
  
SwiftUIの実装をしている中で  
`@State`などの**Property Wrappers**の違いを理解するために  
セッションの動画なども含めて調べてみました。  
※ Property Wrappersについては最後の方で紹介しています。  
  
Data Flow Through SwiftUI  
https://developer.apple.com/videos/play/wwdc2019/226/  
  
まだ学び始めたばかりの認識ですので  
もし勘違いしている点などありましたら  
教えていただけますと嬉しいです:bow_tone1:  
  
# `@State`  
  
Viewはstructのため  
通常の値を更新することができませんが  
`@State`を宣言することで  
メモリの管理がSwiftUIフレームワークに委譲され  
値を更新することができるようになります。  
  
その際にはViewの再構築を自動でしてくれるようになります。  
  
内部的には差分アルゴリズムを使って  
関連のあるViewのみ再構築をしてくれるようです。  
  
  
`State`  
https://developer.apple.com/documentation/swiftui/state  
  
```swift

struct ContentView : View {
    
    @State var value: Bool = false
    
    var body: some View {
        Toggle(isOn: $value) {
            Text("The value is " + "\($value)")
        }
    }
}
```  
  
`@State`は宣言したプロパティを保持するViewと  
その子Viewからしかアクセスができません。  
  
  
### 使いどころ  
  
`＠State`は宣言するたびに  
新しい状態を生み出し(SwiftUIのメモリ上に新しくアロケートされ)ます。  
そのため初期値の設定が必要になります。  
  
![スクリーンショット 2019-06-12 5.20.02.png](/image/7ba67ba5-ee58-b6b8-2d4c-b916ab566d68.png)  
  
そのため  
外部からのデータを参照するなどには適さず  
ローカルでしか使用しないデータのアクセスに適しているようです。  
  
※ ローカルでしか使用しないとは  
`Button`をクリックしたときの`Highlight`など  
その`View`内のみで使用されるようなものを想定しています。  
  
また、structなどの**Value Type**は  
値をコピーする性質から状態の共有をすることができないため  
Value Typeには`＠State`を使用するのが良いようです。  
  
`＠State`のプロパティは`private`修飾子が付けられるような形で  
使用することが好ましいようです。  
  
![スクリーンショット 2019-06-12 5.00.28.png](/image/375d1523-77fa-2490-2d63-728393c6f828.png)  
  
  
# BindableObjectと`＠ObjectBinding`(Xcode11 Beta5よりdeprecated)  
  
値の更新を通知するクラスが`BindableObject`プロトコルに適合することで  
`＠ObjectBinding`を宣言したプロパティへ値の更新通知を行うようになります。  
  
`BindableObject`は`AnyObject`に適合しているため  
参照型にしか使用できません。  
  
更新の通知方法などは開発者側で実装する必要があります。  
  
`＠ObjectBinding`を宣言したプロパティは  
SwiftUIに依存関係があることを明示的に伝え  
値の更新通知を自動で受けとることができるようになります。  
  
動き自体は`@State`に似ていますが、  
Viewの階層をまたがって  
値の更新通知を受け取ることができます。  
  
`＠ObjectBinding`  
https://developer.apple.com/documentation/swiftui/objectbinding  
  
`BindableObject`  
https://developer.apple.com/documentation/swiftui/bindableobject  
  
```swift

class ViewModel: BindableObject {
    
    let willChange = PassthroughSubject<ViewModel, Never>()
    
    var isEnabled = false {
        willSet {
            willChange.send(self)
        }
    }
}

struct ContentView : View {
    
    @ObjectBinding(initialValue: ViewModel()) var viewModel: ViewModel
    
    var body: some View {
        Toggle(isOn: $viewModel.isEnabled) {
            Text("The value is " + "\(viewModel.isEnabled)")
        }
    }
}
```  
  
子Viewにデータを渡す際は`init`を使って  
Viewの階層を辿って依存関係を宣言していく必要があります。  
  
  
![スクリーンショット 2019-06-12 4.31.55.png](/image/7ad02a68-fb07-7f9e-7b9b-d62cb27e5f68.png)  
  
  
### 使いどころ  
  
複数のViewにまたがって参照されるクラスなどに使用するのに適しています。  
  
すでに存在しているViewを表示するためのモデルや  
データベースへの保存に必要なモデルなど  
SwiftUIの外部のデータを参照する場合に適しているようです。  
  
![スクリーンショット 2019-06-12 4.30.44.png](/image/be6f551e-5379-4630-299b-7f51e0e13cfb.png)  
  
  
# ObservableObjectと`＠ObservedObject`(2019/7/30 追記)   
  
Xcode11 Beta5で`BindableObject`は非推奨になり  
`ObservableObject`を使うようになったようです。  
`BindableObject`は`ObservableObject`と`Identifiable`に適合するようになっています。  
これはSwiftUIではなくCombineフレームワークに含まれます。  
ちなみに`Identifiable`はStandard Libraryに移動しました。  
  
  
また、`@ObjectBinding`も`@ObservedObject`に名前が変わりました。  
`@ObjectBinding`は`@ObservedObject`の`typealias`になっています。  
  
```swift

public typealias ObjectBinding = ObservedObject
```  
  
  
`＠ObservedObject`  
https://developer.apple.com/documentation/swiftui/observedobject  
  
`ObservableObject`  
https://developer.apple.com/documentation/combine/observableobject  
  
`Identifiable`  
https://developer.apple.com/documentation/swift/identifiable  
  
```swift

class ViewModel: ObservableObject {
    
    let willChange = PassthroughSubject<ViewModel, Never>()
    
    var isEnabled = false {
        willSet {
            willChange.send(self)
        }
    }
}

struct ContentView : View {
    
    @ObservedObject(initialValue: ViewModel()) var viewModel: ViewModel
    
    var body: some View {
        Toggle(isOn: $viewModel.isEnabled) {
            Text("The value is " + "\(viewModel.isEnabled)")
        }
    }
}
```  
  
# (2019/7/31 追記) `@Published`で自動通知をするようになりました。  
  
Xcode11 Beta5より  
`@Published`を使用すると上記のような`willChange`の記載する必要がなくなりました。  
  
```swift

import Combine
import SwiftUI
class ViewModel: ObservableObject {
    @Published var isEnabled: Bool = false
}

struct ContentView : View {
    
    @ObservedObject(initialValue: ViewModel()) var viewModel: ViewModel
    
    var body: some View {
        Toggle(isOn: $viewModel.isEnabled) {
            Text("The value is " + "\(viewModel.isEnabled)")
        }
    }
}
```  
  
https://developer.apple.com/documentation/combine/published  
  
# environmentObjectと`＠EnvironmentObject`  
  
### Environmentとは  
  
「SwiftUI Essentials」のセッションで出てきましたが  
SwiftUIの特徴の１つで  
Viewの階層の中でコンテキストを共有するコンテナのような存在です。  
  
![スクリーンショット 2019-06-13 8.45.15.png](/image/303f51b9-8b12-9a2e-1eb2-022663e0f4f3.png)  
  
SwiftUI Essentials  
https://developer.apple.com/videos/play/wwdc2019/216/  
  
environmentObjectと`＠EnvironmentObject`を使うと  
上記のEnvironmentの中にカスタムな値を設定できます。  
  
`environmentObject`を使用したViewのEnvironment内で  
引数に渡した`ObservableObject`の値を  
`＠EnvironmentObject`から参照できます。  
  
`＠EnvironmentObject`  
https://developer.apple.com/documentation/swiftui/environmentobject  
  
`environmentObject`  
https://developer.apple.com/documentation/swiftui/view/3308681-environmentobject  
  
```swift

class ViewModel: ObservableObject {
    
    let willChange = PassthroughSubject<ViewModel, Never>()
    
    var isEnabled = false {
        willSet {
            willChange.send(self)
        }
    }
}

struct ContentView : View {
    
    @EnvironmentObject var viewModel: ViewModel
    
    var body: some View {
        Toggle(isOn: $viewModel.isEnabled) {
            Text("The value is " + "\(viewModel.isEnabled)")
        }
    }
}

let view = ContentView().environmentObject(ViewModel())
```  
  
`＠ObservedObject `は子Viewへデータを渡すのに  
`init`を使用してViewの階層を辿っていく必要がありましたが  
  
`＠EnvironmentObject`を使うとその必要なしに値を参照することができます。  
  
内部的に`＠EnvironmentObject`は**dynamic member lookup**を使用しており  
そのEnvironmentの`ObservableObject`を探してくれます。  
  
![スクリーンショット 2019-06-12 4.32.38.png](/image/fe2e88b7-3ae1-bc4d-305a-12e146e84cc5.png)  
  
  
### 使いどころ  
  
Environment全体の設定に関わる値など  
`＠ObservedObject`よりも広く  
アプリ全体で使用する必要があるようなデータに適しているようです。  
  
  
# `@Environment`  
  
`EnvironmentValues`という事前に定義されているプロパティの値を使って  
Environment全体にまたがるような設定の取得、更新ができます。  
  
`Environment`  
https://developer.apple.com/documentation/swiftui/environment  
  
`EnvironmentValues`  
https://developer.apple.com/documentation/swiftui/environmentvalues  
  
  
```swift

struct ContentView : View {
    
    @Environment(\.isEnabled) var enabled: Bool
    
    var body: some View {
        Text("The value is " + "\(enabled)")
    }
}
```  
  
値の変更も可能です。  
  
![スクリーンショット 2019-06-13 8.50.21.png](/image/3260d545-8d68-7539-6c95-9a6fd72003f0.png)  
  
  
### 使いどころ  
  
上記でも記載した通り  
`EnvironmentValues`で定義されている値を参照するために使用します。  
  
  
  
# `@Binding`  
  
値を参照する側のプロパティに宣言をすることで  
親Viewの`＠State`や  
下記で紹介する`@ObservedObject ` `@Binding`の  
値の更新通知を受け取ることができます。  
  
  
`＠Binding`  
https://developer.apple.com/documentation/swiftui/binding  
  
`@Binding`では**値を参照**します。  
  
ViewはValue Typeで  
親Viewから子Viewへ値を直接渡してもコピーを渡してしまいますが、  
`@Binding`を使用することで  
Value Typeへの参照を渡して親子間でデータの同期を保つことができます。  
  
![スクリーンショット 2019-06-12 4.53.48.png](/image/705d5e8a-c109-1617-12aa-3cf4232d7d94.png)  
  
  
### 使いどころ  
  
ViewはImmutableであるのが好ましいものの  
値の更新をする必要がある場合などは  
基本的には`@Binding`を使うのが良いとセッションでは言っています。  
  
  
# `@GestureState`　2019/7/17追記  
  
特定のViewへのジェスチャーイベントを受け取ることができます。  
注意点としては  
**イベント終了後に変化した値は保持されない**という点です。  
  
使い方は受け取りたいイベントの変数を作成し  
`gesture`を使用します。  
  
```swift

struct ShapeTapView: View {
    var body: some View {
        let tap = TapGesture()
            .onEnded { _ in
                print("View tapped!")
            }
        
        return Circle()
            .fill(Color.blue)
            .frame(width: 100, height: 100, alignment: .center)
            .gesture(tap)
    }
}
```  
  
コールバックを受け取るには3つの方法があります。  
  
### `updating(_:body:)`  
  
ジェスチャーの変化ごとにViewを更新していく場合などに使用します。(例えばズームやドラッグなど)  
ジェスチャーが完了したりキャンセルされるとコールバックを受け取られなくなります。  
  
この値はジェスチャー中だけ保持され  
ジャスチャーが止まると値はリセットされます。  
  
下記の例はロングプレス中は円の色が変わりますが  
指を離すと色は元の色に戻ります。  
  
```swift

struct CounterView: View {
    @GestureState var isDetectingLongPress = false
    
    var body: some View {
        let press = LongPressGesture(minimumDuration: 1)
            .updating($isDetectingLongPress) { currentState, gestureState, transaction in
                gestureState = currentState
            }
        
        return Circle()
            .fill(isDetectingLongPress ? Color.yellow : Color.green)
            .frame(width: 100, height: 100, alignment: .center)
            .gesture(press)
    }
}
```  
https://developer.apple.com/documentation/swiftui/gesture/3284686-updating  
  
### `onChanged(_:)`  
  
ジャスチャーの値が変化した時にコールバックを受け取ることができます。  
ジェスチャーが完了した後も変化した値を保持したい場合に`＠State`と組み合わせて使用します。  
  
下記の例はロングプレスをしている間はカウントが上がっていきます。  
`@State`の`totalNumberOfTaps`で  
ロングプレスが終わった後も値を保持するようにしています。  
  
```swift

struct CounterView: View {
    @GestureState var isDetectingLongPress = false
    @State var totalNumberOfTaps = 0
    
    var body: some View {
        let press = LongPressGesture(minimumDuration: 1)
            .updating($isDetectingLongPress) { currentState, gestureState, transaction in
                gestureState = currentState
            }.onChanged { _ in
                self.totalNumberOfTaps += 1
            }
        
        return VStack {
            Text("\(totalNumberOfTaps)")
                .font(.largeTitle)
            
            Circle()
                .fill(isDetectingLongPress ? Color.yellow : Color.green)
                .frame(width: 100, height: 100, alignment: .center)
                .gesture(press)
        }
    }
}
```  
https://developer.apple.com/documentation/swiftui/gesture/3268692-onchanged  
  
### `onEnded(_:)`  
  
ジェスチャーが完了後に最終的な値を受け取ることができます。  
成功した場合のみコールバックを受け取ることができます。  
  
例えばロングプレスのようなジェスチャーをしたものの  
設定している`minimumDuration`に到達していない場合や  
`maximumDistance`を超えて動かした場合は  
コールバックを受け取ることはできません。  
  
`minimumDuration`  
https://developer.apple.com/documentation/swiftui/longpressgesture/3270364-minimumduration  
  
`maximumDistance`  
https://developer.apple.com/documentation/swiftui/longpressgesture/3077970-maximumdistance  
  
下記の例は  
ロングプレスをしている間はカウントを上げていきますが  
指を離すとカウントは上がらなくなります。  
  
```swift

struct CounterView: View {
    @GestureState var isDetectingLongPress = false
    @State var totalNumberOfTaps = 0
    @State var doneCounting = false
    
    var body: some View {
        let press = LongPressGesture(minimumDuration: 1)
            .updating($isDetectingLongPress) { currentState, gestureState, transaction in
                gestureState = currentState
            }.onChanged { _ in
                self.totalNumberOfTaps += 1
            }
            .onEnded { _ in
                self.doneCounting = true
            }
        
        return VStack {
            Text("\(totalNumberOfTaps)")
                .font(.largeTitle)
            
            Circle()
                .fill(doneCounting ? Color.red : isDetectingLongPress ? Color.yellow : Color.green)
                .frame(width: 100, height: 100, alignment: .center)
                .gesture(doneCounting ? nil : press)
        }
    }
}
```  
https://developer.apple.com/documentation/swiftui/gesture/3268693-onended  
  
参考ドキュメント  
https://developer.apple.com/documentation/swiftui/gestures/adding_interactivity_with_gestures  
  
  
# セッションの内容を見ていく  
  
ここからはセッションの内容についてもう少し見ていきたいと思います。  
  
# データフローの原理  
  
### データを読み込む全てのViewは依存関係を構築する  
  
データの更新が発生するとViewが更新され  
データとViewの間には依存関係が生まれます。  
  
### データを読み込むViewは情報源を持っている  
  
データには発生元が必ず存在し  
Viewはそれに対しての参照を持っています。  
  
### Single Source Of Truthを目指す  
  
情報源はいくつも生成することができますが  
特定の情報に対する情報源は１つであるべきです。  
  
そうしないとデータの不整合などのバグを生み出す要因になります。  
  
変更が加えられたときはその唯一の情報源から変化を伝達させるようにします。  
  
  
![スクリーンショット 2019-06-12 5.21.24.png](/image/5a4492ff-c27e-3acd-416d-d4b7fe023f6a.png)  
![スクリーンショット 2019-06-12 5.21.37.png](/image/5bdedba8-e189-de8b-a3df-1f78536e3558.png)  
![スクリーンショット 2019-06-12 5.23.39.png](/image/81ea5b2e-1f71-828c-5cf3-097c70265140.png)  
  
  
# SwiftUIの更新の仕組み  
  
SwiftUIは上記のデータフローの原理を簡単に実現できるようになっています。  
  
`View`は`struct`なので自身の値を更新できません。  
その代わりに  
`Combine`フレームワークの`Publisher`や  
`Property Wrapper`を使ってデータの更新を`View`に伝達します。  
  
これらを利用することで  
データの流れが一方向になり予測可能で理解しやすくします。  
  
![スクリーンショット 2019-06-12 5.15.50.png](/image/24f09dcf-95bd-8cf8-f60c-6a07174001c0.png)  
  
これまではデータの同期を取るために  
`UIViewController`が必要になりましたが  
SwiftUIではその必要もなくなりよりシンプルな仕組みになっています。  
  
![スクリーンショット 2019-06-12 5.18.25.png](/image/491a7b75-7fe4-e12b-ebc9-4ddd218f8000.png)  
  
それではそれぞれの伝達方法について見ていきたいと思います。  
  
# CombineフレームワークのPublisher  
  
イベント駆動型の非同期処理を簡単に扱えるようにしたフレームワーク  
https://developer.apple.com/documentation/combine  
  
`Publisher`がEmitしたイベントを`onReceive`で受け取ることができます。  
https://developer.apple.com/search/?q=onReceive  
  
![スクリーンショット 2019-06-12 5.23.39.png](/image/ff4be96b-942d-9722-361f-30e69f91c46f.png)  
  
  
Combineフレームワークについてはこちらにも書かせていただきました。  
https://github.com/stzn/qiita/blob/master/【iOS】Combineフレームワークまとめ.md  
  
# Property Wrappers  
  
Swift5.1で導入された新しい機能で  
冒頭で紹介している`＠State` `@ObservedObject` `@Binding`などもProperty Wrappersです。  
アノテーション(@)を使って宣言したプロパティへの独自のアクセス方法を定義することができます。  
  
https://github.com/apple/swift-evolution/blob/master/proposals/0258-property-delegates.md  
  
そしてまだ今後に向けて議論は続いているようです。  
https://forums.swift.org/t/pitch-3-property-wrappers-formerly-known-as-property-delegates/24961/201  
  
※以前は `Property Delegates`と呼ばれていました。  
  
これを使用することで  
共通のデータアクセス方法を一箇所で定義でき  
重複コードの削減をすることができます。  
  
![スクリーンショット 2019-06-12 8.57.22.png](/image/9539b8e5-cc1d-6a1e-71e0-fb7ff9936d4d.png)  
  
「Modern Swift API Design」セッションで  
仕組みが詳しく解説されています。  
  
内部でStored Propertyを持ち  
それに対応したComputed Propertyで  
Stored Propertyにどうアクセスするのかを定義します。  
  
```swift

@propertyWrapper
public struct LateInitialized<Value> {
    private var storage: Value?
    
    public init() {
        storage = nil
    }
    
    public var value: Value {
        get {
            guard let value = storage else {
                fatalError("value has not yet been set!")
            }
            return value
        }
        set {
            storage = newValue
        }
    }
}
```  
  
```swift

public struct MyType {


    @LateInitialized public var text: String


    // Compiler-synthesized code...

    var $text: LateInitialized<String> = LateInitialized<String>()

    public var text: String {
 get { $text.value }

        set { $text.value = newValue }
 }

    
}
```  
  
初期値を渡したり  
`extension`で新しく`init`を定義することなども可能です。  
  
```swift

@propertyWrapper
public struct DefensiveCopying<Value: NSCopying> {
    private var storage: Value
    
    public init(initialValue value: Value) {
        storage = value.copy() as! Value
    }
    
    public var value: Value {
        get { storage }
        set {
            storage = newValue.copy() as! Value
        }
    }
}

@DefensiveCopying public var path: UIBezierPath = UIBezierPath()


extension DefensiveCopying {

    public init(withoutCopying value: Value) {

        storage = value

    }

}

public struct MyType {

    @DefensiveCopying(withoutCopying: UIBezierPath())

    public var path: UIBezierPath

}
```  
  
Modern Swift API Design  
https://developer.apple.com/videos/play/wwdc2019/415  
  
  
「What's New in Swift」セッションの中でも  
`UserDefaults`へのアクセスパターンが紹介されていました。  
  
```swift

@propertyWrapper
struct UserDefault<T> {
    let key: String
    let defaultValue: T

    init(_ key: String, defaultValue: T) {
        self.key = key
        self.defaultValue = defaultValue
    }

    var value: T {
        get {
            return UserDefaults.standard.object(forKey: key) as? T ?? defaultValue
        }
        set {
            UserDefaults.standard.set(newValue, forKey: key)
        }
    }
}
```  
  
```swift

@UserDefault("USES_TOUCH_ID", defaultValue: false)
static var usesTouchID: Bool
@UserDefault("LOGGED_IN", defaultValue: false)
static var isLoggedIn: Bool
if !isLoggedIn && usesTouchID {
 !authenticateWithTouchID()
}

```  
  
What's New in Swift  
https://developer.apple.com/videos/play/wwdc2019/402/  
  
  
## 追記  
  
この後にvalueはwrappedValueに変更されています。  
  
  
```swift

@propertyWrapper
struct UserDefault<T> {
    let key: String
    let defaultValue: T

    init(_ key: String, defaultValue: T) {
        self.key = key
        self.defaultValue = defaultValue
    }

    var wrappedValue: T {
        get {
            return UserDefaults.standard.object(forKey: key) as? T ?? defaultValue
        }
        set {
            UserDefaults.standard.set(newValue, forKey: key)
        }
    }
}
```  
  
さらに`projectedValue`を定義することで  
`＄`を通して変数へアクセスをすることができるようになりました。  
  
例えば`@State`の場合  
Bindingでラップされた値へアクセスすることができるようになります。  
  
```swift

@propertyWrapper public struct State<Value> : DynamicProperty {
    ...
    /// Produces the binding referencing this state value
    public var projectedValue: Binding<Value> { get }
}
```  
  
  
# まとめ  
  
SwiftUIのProperty Wrappersと  
データへのアクセス方法について見てきました。  
  
まずはセッションの動画から  
どういう使われ方が期待されているのか  
注意するべきところはどういうところなのか  
などを少し理解することはできたのかなと思っています。  
  
まだまだチュートリアル程度しか触れていないので  
これから実践していく中で色々な発見をするのが楽しみです！  
