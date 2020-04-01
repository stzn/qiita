# iOS13で変わったこと  
  
iOS13以降ではiOSアプリでもマルチウィンドウで使えるようになり  
アプリの起動経路が変わりました。  
  
※ マルチウィンドウ自体はオプトインの仕組みで  
ONにするにはinfo.plistの`UIApplicationSceneManifest`の設定が必要になります。  
  
https://developer.apple.com/documentation/uikit/app_and_environment/managing_your_app_s_life_cycle  
https://developer.apple.com/documentation/bundleresources/information_property_list/uiapplicationscenemanifest  
  
具体的には下記の`SceneDelegate`で画面の最初の表示を行います。  
  
```swift

class SceneDelegate: UIResponder, UIWindowSceneDelegate {

    var window: UIWindow?


    func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options connectionOptions: UIScene.ConnectionOptions) {
        // Use this method to optionally configure and attach the UIWindow `window` to the provided UIWindowScene `scene`.
        // If using a storyboard, the `window` property will automatically be initialized and attached to the scene.
        // This delegate does not imply the connecting scene or session are new (see `application:configurationForConnectingSceneSession` instead).

        // Use a UIHostingController as window root view controller
        if let windowScene = scene as? UIWindowScene {
            let window = UIWindow(windowScene: windowScene)
            window.rootViewController = UIHostingController(rootView: ContentView())
            self.window = window
            window.makeKeyAndVisible()
        }
    }
}
```  
  
  
`UIWindow`は`AppDelegate`から`SceneDelegate`に移動し  
下記のにUIWindowインスタンスを取得することができなくなりました。  
  
```swift

let delegate = UIApplication.shared.delegate as! AppDelegate
let window = delegate.window
```  
  
これまで見てきた限りですと  
`SceneDelegate`への適切なアクセス方法が見つからず  
さらに`UIWindow`が一つのインスタンスとは限らないため  
シングルトンでインスタンスを保持していても  
マルチウィンドウを使用している場合  
別の`UIWindow`を参照してしまうリスクもあります。  
  
# どういう時にUIWindowを使うか？  
  
例えば  
Sign in With Appleなどのような認証を使用する場合  
認証を行う画面を表示するためにUIWindowを提供する必要があります。  
  
  
```swift

extension LoginViewController: ASAuthorizationControllerPresentationContextProviding {
    func presentationAnchor(for controller: ASAuthorizationController) -> ASPresentationAnchor {
        return self.view.window!
    }
}
```  
https://developer.apple.com/documentation/authenticationservices/adding_the_sign_in_with_apple_flow_to_your_app  
  
https://developer.apple.com/documentation/authenticationservices/asauthorizationcontrollerpresentationcontextproviding/3237228-presentationanchor  
  
# SwiftUIのViewからアクセスするには？  
  
上記のような認証をSwiftUIから使用する場合  
`View`から`UIWindow`を提供することがあるかもしれません。  
  
その際に`View`から`UIWindow`のインスタンスへアクセスするにはどうしたら良いでしょうか？  
  
  
# `@Environment`を利用する  
  
一つの方法として  
`@Environment`を使って`UIWindow`インスタンスをDIする方法があります。  
  
これを実現するために  
`EnvironmentKey`と`EnvironmentValues`を活用します。  
  
https://developer.apple.com/documentation/swiftui/environmentkey  
https://developer.apple.com/documentation/swiftui/environmentvalues  
  
まず`EnvironmentKey`プロトコルに適合したstructを作成します。  
必須なのは`defaultValue`プロパティのみです。  
  
```swift

struct WindowKey: EnvironmentKey {
    static let defaultValue: UIWindow? = nil
}
```  
  
次に上記の`EnvironmentValues`を拡張して  
上記のキーの`subscript`を定義します。  
  
  
```swift

extension EnvironmentValues {
    var window: UIWindow? {
        get {
            self[WindowKey.self]
        }
        set {
            self[WindowKey.self] = newValue
        }
    }
}
```  
  
こうすることで`environment(_:_:)`を使ってwindowをDIすることができます。  
https://developer.apple.com/documentation/swiftui/view/3289000-environment  
  
  
```swift

class SceneDelegate: UIResponder, UIWindowSceneDelegate {

    var window: UIWindow?


    func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options connectionOptions: UIScene.ConnectionOptions) {
        if let windowScene = scene as? UIWindowScene {
            let window = UIWindow(windowScene: windowScene)
            
            // ↓↓↓↓UIWindowを設定している↓↓↓↓↓↓
            let contentView = ContentView().environment(\.window, window)
            window.rootViewController = UIHostingController(rootView: contentView)
            self.window = window
            window.makeKeyAndVisible()
        }
    }

}
```  
  
これで`View`からは  
  
```swift

struct ContentView: View {
    @Environment(\.window) var window
}
```  
  
とアクセスすることができます。  
  
参考  
https://github.com/objcio/swift-talk-app-swiftui/blob/episode-163/Account.swift  
  
# なぜ`@EnvironmentObject`ではなく`@Environment`のか？  
  
実は同じように`UIWindow`にアクセスできます。  
しかしその場合は  
`UIWindow`を`ObservableObject`に  
適合させないといけなくなります。  
`UIWindow`を拡張するのはあまり良くないのではないかということで  
`@Environment`を使用しています。  
  
# まとめ  
  
SwiftUIに触っていると  
「あれ？これどうやるんだ？」と悩むことって意外と多く感じています。  
  
まだまだ勉強中なので  
これからも知見、経験をためていって使いこなせるようになりたいです😃  
  
  
何か間違いなどございましたら教えていただけますと嬉しいです🙇🏻‍♂️  
  
  
