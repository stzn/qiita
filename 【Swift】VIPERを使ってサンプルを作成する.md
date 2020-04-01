初投稿、失礼します。  
  
日頃一人であれやこれやと検索しつつ頭を抱えて開発をしていますが、  
他の方がどうやって開発されているのかを実際に見聞きしようと  
最近外に出るようになりました。  
  
今までブログなど全然書いたことないのですが、  
結果を人に見せるという意識を自分自身で持った方がより効果的に勉強ができると思い、  
備忘録がでら投稿してみました。  
  
今回は、VIPERの設計を中心にサンプルを書いて学んだことを書きます。  
  
# VIPERとは？  
  
既に多くの良い記事がありますので、参照リンクだけ貼らせて頂きます。  
[iOS Project Architecture : Using VIPER [和訳]](https://qiita.com/YKEI_mrn/items/67735d8ebc9a83fffd29)  
[ぼくのやっているVIPER(のようなもの)](https://qiita.com/Yaruki00/items/d350709678ff0aec93cc#_reference-b1177ce196907a3e8aea)  
  
# なぜVIPER？  
ちょっと前まで開発していたアプリで  
  
[MVVM+C(Coordinator)パターン](https://github.com/macdevnet/mvvmc-demo) ※主に参考したサンプル  
  
を使用していたところ、それなりに責務の切り分けができ変更が入っても影響範囲を限定することができていると感じたものの、  
  
・ViewModelが大きくなる  
・Delegateが色々な箇所にあり、ソースを追うのにたくさんのファイルを参照する  
  
などのようなことが起き、VIPERが前から気になっていたことに加え、ちょうどVIPERを勧めていただいたこともあり、サンプルを作成しつつ勉強しました。  
  
作成したコードは[Github](https://github.com/stzn/ViperSampleGeneramba)に上げています。  
  
# 構成  
https://github.com/infinum/iOS-VIPER-Xcode-Templates  
を参考に、  
[Generamba](https://github.com/rambler-digital-solutions/Generamba#overview)  
で独自のテンプレートを作成してModules以下を自動生成するようにしました。  
  
```
Application
    - AppDelegate.swift
    - RootViewController.swift
Common
    - VIPER ※各Modulesの各部品の共通Interfaces デフォルト実装などを行う
    - ...
Moules
    - Login
       - Interactors
       - Interfaces
       - Presenter
       - Views
       - Wireframe
    - QiitaItems
        - ...
    - Splash
        - ...

```  
  
※自動生成でできる構成は以下になります。  
  
```
Modules
    - Interactors
    - Interfaces
    - Presenter
    - Views
    - Wireframe
    - DataManager
        - API(※ファイル名はAPIDataManager)
        - Local(※ファイル名はLocalDataManager)
```  
今回はあまり大きなサンプルではないため、  
DataManagerは使わずInteractorから  
APIを呼び出しクロージャで結果を受け取るようにしています。  
  
## Modules内の部品の役割  
  
<dl>  
<dt>Interactors</dt>  
<dd>データの取得、保存を行うDataManager呼び出す(今回は直接APIServiceの呼び出し)</dd>  
<dt>Interfaces</dt>  
<dd>モジュール内の各部品のprotocol宣言</dd>  
<dt>Presenters</dt>  
<dd>画面表示用のデータ加工。状態の管理。View、Interactor、Wireframeを繋ぐ。</dd>  
<dt>Views</dt>  
<dd>画面表示 + ユーザーアクションをpresenterへ受け渡す</dd>  
<dt>Wireframes</dt>  
<dd>画面遷移 ＋ 各部品のインスタンス化と依存関係の注入</dd>  
</dl>  
  
以下がログイン画面で行なっている処理です。  
※記載していないファイルや適当に省略している処理があります。  
  
**LoginWireframe.swift**  
```swift:LoginWireframe.swift
final class LoginWireframe: LoginWireframeInterface {

    // MARK: - Private properties -
    
    // MARK: - Module setup -    
    func configureModule() -> UIViewController {
        
        let viewController = LoginViewController.fromStoryboard()
        let interactor = LoginInteractor()
        let presenter = LoginPresenter(wireframe: self, view: viewController, interactor: interactor)
        viewController.presenter = presenter
        interactor.output = presenter
        
        let navigationController = UINavigationController(rootViewController: viewController)
        
        return navigationController
    }

    // MARK: - Transitions -
    
    func showMainScreen() {        
        let root = AppDelegate.shared.rootViewCotnroller
        root.showMainScreen()
    }
}
```  
  
**LoginInteractor.swift**  
```swift:LoginInteractor.swift
final class LoginInteractor {
    weak var output: LoginInteractorOutputInterface?
}

// MARK: - Extensions -

extension LoginInteractor: LoginInteractorInterface {
func saveUserInfo(_ userInfo: UserInfo) {
        
        /// ログイン状態を作るための仮処理
        UserDefaults.standard.set(true, forKey: UserDefaultsKeys.loggedIn)
        
        output?.userInfoSaveSucceeded()
    }
    
    func login(id: String, password: String) {
        output?.loginSuceeded()
    }
}
```  
  
**LoginWireframeInterface.swift**  
```swift:LoginWireframeInterface.swift
protocol LoginWireframeInterface: WireframeInterface {
    func configureModule() -> UIViewController
    func showMainScreen()
}

protocol LoginViewInterface: ViewInterface {
}

protocol LoginPresenterInterface: PresenterInterface {
    func loginButtonTapped(id: String, password: String)
}

protocol LoginInteractorInterface: InteractorInterface {
    func login(id: String , password: String)
    func saveUserInfo(_ userInfo: UserInfo)
}

protocol LoginInteractorOutputInterface: InteractorOutputInterface {
    func loginSuceeded()
    func loginFailed()
    func userInfoSaveSucceeded()
    func userInfoSaveFailed()
}
```  
  
**LoginPresenter.swift**  
```swift:LoginPresenter.swift
final class LoginPresenter {
    
    // MARK: - Private properties -
    
    private weak var _view: LoginViewInterface?
    private var _interactor: LoginInteractorInterface
    private var _wireframe: LoginWireframeInterface
    
    // MARK: - Lifecycle -
    
    init(wireframe: LoginWireframeInterface, view: LoginViewInterface, interactor: LoginInteractorInterface) {
        _wireframe = wireframe
        _view = view
        _interactor = interactor
    }
}

// MARK: - Extensions -

extension LoginPresenter: LoginPresenterInterface {
    
    func loginButtonTapped(id: String, password: String) {
        
        guard !id.isEmpty && !password.isEmpty else {
            showLoginError()
            return
        }
        
        _view?.showLoading()
        _interactor.login(id: id, password: password)
    }
    
    private func showLoginError() {
        _view?.showAlert(title: "エラー", message: "ログインIDとパスワードは必ず入力してください。")
    }
    
    private func showIdValidationError() {
        _view?.showAlert(title: "エラー", message: "IDは英数字8文字以上で入力してください。")
    }
    
    private func showPasswordValidationError() {
        _view?.showAlert(title: "エラー", message: "パスワードは英数字8文字以上で入力してください。")
    }
}

extension LoginPresenter: LoginInteractorOutputInterface {
    func loginSuceeded() {
        let userInfo = UserInfo(name: "Dummy")
        _interactor.saveUserInfo(userInfo)
    }
    
    func loginFailed() {
        _view?.hideLoading()
        _view?.showErrorAlert(message: "ログインIDまたはパスワードが間違っています。")
    }
    
    func userInfoSaveSucceeded() {
        _view?.hideLoading()
        _wireframe.showMainScreen()
    }
    
    func userInfoSaveFailed() {
        _view?.hideLoading()
        _view?.showErrorAlert(message: "ログインIDまたはパスワードが間違っています。")
    }
}
```  
  
**LoginViewController.swift(一部省略)**  
```swift:LoginViewController.swift(一部省略)
final class LoginViewController: UIViewController {
    
    @IBOutlet weak var messageLabel: UILabel!
    @IBOutlet weak var idTextField: UITextField!
    
    @IBOutlet weak var passwordTextField: UITextField!
    
    @IBOutlet weak var loginButton: UIButton!
    
    // MARK: - Public properties -
    
    var presenter: LoginPresenterInterface!
    
    // MARK: - Lifecycle -
    
    override func viewDidLoad() {
        super.viewDidLoad()
        _setupUI()
    }
    
    @IBAction func loginButtonTapped(_ sender: Any) {
        presenter.loginButtonTapped(id: idTextField.text ?? "", password: passwordTextField.text ?? "")
    }
}

// MARK: - Extensions -

extension LoginViewController {
    private func _setupUI() {
        idTextField.delegate = self
        passwordTextField.delegate = self
    }
}
extension LoginViewController: LoginViewInterface {
}

extension LoginViewController: StoryboardLoadable {
    static var storyboardName: String {
        return Storyboard.LoginViewController.name
    }
}
```  
  
## 良かった点  
  
・InterfacesでModule全体のプロトコルをまとめて見ることできるので見通しがよくなった。  
・役割が明確なのでロジックの置き場所を決めやすくなった。  
・処理の追加や修正が発生した場合は修正箇所が見つけやすくなった。  
・テストが書きやすくなった?(Spyを使って部品間の疎通確認と個々のロジックのテストに分割)  
  
## 課題  
  
・そもそもどういう場面でVIPERのような設計が効果があるのか判断基準を見つける。  
・部品間の連携や役割をきちんと理解してシンプルでわかりやすいコードを書く。  
・テストの書き方がいまいちわかっていないので他の実装を参考にテストの書き方を学ぶ。  
  
## その他  
  
VIPERの学習に合わせて以下の内容も勉強しました。  
・Decodableプロトコル  
・tableviewのprefetch処理  
・UINavigationItemのsearchControllerプロパティ  
  
<br>  
まだまだ不明な点や間違っていると思われる点が多々ありますので、  
引き続き頭を抱えながら色々なことを学んでいきます。  
  
RxSwiftを使ったサンプルも作ってみたので、またの機会に上げてみます。  
<br>  
長文失礼しました。  
