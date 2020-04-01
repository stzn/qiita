  
## 経緯  
  
参考にした記事: https://github.com/onmyway133/blog/issues/106  
  
開発の中でCoordinatorパターンを使っていたことと  
Coordinatorパターンを提唱していた[@khanlou](https://twitter.com/khanlou)さんも評価をしていたこともあり、  
FlowControllerに興味を持ったので調べてみました。  
※ほぼ記事の内容を訳したような内容になってしまいましたが。。。  
  
作成したサンプル: https://github.com/stzn/ViperSampleGeneramba/tree/flowcontroller  
  
  
## Coordinatorとは？  
  
ざっくりと言えば、画面遷移の制御を行うためのクラスです。  
  
ViewControllerやViewModelから画面遷移の責務を分離することで、  
コードの整理やテストしやすさを向上させることができます。  
  
たくさん良い解説がありますので、詳しい内容は割愛させていただきます。  
  
参照記事  
https://www.youtube.com/watch?v=a1g3k3NObkE  
http://khanlou.com/2015/01/the-coordinator/  
https://qiita.com/endorno@github/items/d0b25c47e3c5a7b48865  
  
## FlowControllerとは？  
  
基本的に役割はCoordinatorと同じですが、  
UIViewControllerを継承しています。  
  
参照記事  
http://albertodebortoli.com/blog/2014/09/03/flow-controllers-on-ios-for-a-better-navigation-control/  
https://github.com/Flinesoft/Imperio  
  
  
## 何が違うのか？  
  
参考にした記事の中でFlowControllerの使い方や  
Coordinatorの違いについて述べられています。  
  
  
### 1. AppDelegate  
  
AppFlowControllerを  
UIWindowのrootViewControllerに設定します。  
  
  
```AppDelegate.swift
struct DependencyContainer {

    let authService: AuthService
    let networkingService: NetworkingService
   
    static func make() -> DependencyContainer {
        return DependencyConteiner(
            networkingService: NetworkingService()
            authService: AuthService()
        )
    }
}

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {

    var window: UIWindow?
    var appFlowController: AppFlowController!
    
    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
        
        appFlowController = AppFlowController(dependencyContainer: DependencyContainer.make())
        
        window = UIWindow(frame: UIScreen.main.bounds)
        window?.rootViewController = appFlowController
        window?.makeKeyAndVisible()
        
        appFlowController.start()
        
        return true
    }
}
```  
  
AppFlowControllerでは  
子FlowControllerを加えることで次のフローを開始します。  
  
また、必要な依存関係をAppFlowControllerから渡します。  
  
```AppFlowController.swift
private func startLogin() {
        
    let dependencyContainer = ListDependencyContainer(
        networkingService: dependencyContainer.networkService 
    )

    let loginFlowController = LoginFlowController(
        dependencyContainer: dependencyContainer
    )

    // controllerとcontroller.viewを拡張メソッドで追加しています
    loginFlowController.delegate = self
    add(childController: loginFlowController)

    // 次のフローを開始
    loginFlowController.start()
}
```  
  
### 2. FlowControllerはContainerView  
  
FlowControllerは画面遷移管理に徹し、  
自身をContainerViewとして実際の画面表示は子ViewControllerを使います。  
  
```swift
override func viewDidLayoutSubviews() {
    super.viewDidLayoutSubviews()

    // 通常FlowControllerは１つの子FlowControllerを表示するため
    // 以下のようにviewのframeを更新する
    childViewControllers.first?.view.frame = view.bounds
}
```  
  
子ViewControllerで画面遷移イベント(セルをタップなど)が起きた場合は  
FlowControllerへdelegateやcallbackで伝達し、  
必要な場合はFlowControllerから別の画面へ遷移する  
  
  
  
### 3.FlowControllerは依存関係を管理  
  
1でも出てきましたが、  
FlowControllerで親のFlowControllerから必要な依存関係を注入し、  
さらにその中から個々のViewControllerに必要なinitメソッドを渡します。  
  
  
### 4. 子FlowControllerの管理が簡単  
  
Coordinatorの場合、参照を保持するために  
childCoordinatorの配列を持つ必要があります。  
  
```swift
class Coordinator {
    
    private var children: [Coordinator] = []

    func add(child: Coordinator) {
        ...
    }

    func remove(child: Coordinator) {
        ...
    }
}
```  
  
一方で、FlowControllerはUIVIewControllerを継承しているため  
子FlowControllerを保持するための```viewControllers``` プロパティを持っています。  
  
なので、下記のようなUIViewControllerの拡張メソッドを作成することで  
簡単に子FlowControllerの管理ができます。  
  
```swift
extension UIViewController {
    func add(childController: UIViewController) {
        addChildViewController(childController)
        view.addSubview(childController.view)
        childController.didMove(toParentViewController: self)
    }
 
    func remove(childController: UIViewController) {
        childController.willMove(toParentViewController: nil)
        childController.view.removeFromSuperview()
        childController.removeFromParentViewController()
    }
}
```  
  
  
### 5.UIWinowを保持する必要がない  
  
CoordinatorはUIViewControllerの階層とは別になっているため  
UIWindow管理する必要があります。  
  
```AppCoordinator.swift
final class AppCoordinator: Coordinator {
    
    private let window: UIWindow
    init(window: UIWindow) {
        self.window = window
    }
}
```  
  
一方で、FlowControllerはUIViewControllerなので  
rootViewControllerにそのまま設定できます。  
  
### 6.個々のフローを疎結合にできる   
  
例えば、ログイン画面を表示するために、  
ログインのフローを新しく開始したいとします。  
  
Coordinatorの場合、  
rootViewControllerにUINavigationControllerを設定するためには  
AppCoordinatorで設定する必要があります。  
(もしくはUIWindowをLoginのCoordinatorに渡します。)  
  
``` AppCoordinator.swift
final class AppCoordinator: Coordinator {
    private let window: UIWindow

    private func startLogin() {
        let navigationController = UINavigationController()
 
        let loginCoordinator = LoginCoordinator(navigationController: navigationController)
 
        window.rootViewController = navigationController
        loginCoordinator.start()
    }
}

```  
```LoginCoordinator.swift
final class LoginCoordinator: Coordinator {
    private let navigationController: UINavigationController
 
    init(navigationController: UINavigationController) {
        self.navigationController = navigationController
    }

    func start() {

        let loginController = LoginController(dependencyContainer: dependencyContainer)    
        navigationController.viewControllers = [loginController]
    } 
}
```  
  
  
一方、FlowControllerでは通常のUIKitの使い方と同様に扱うことができます。  
  
1.にあるような方法でLoginFlowControllerをaddし、  
LoginFlowController内でUINavigationControllerを作成します。  
  
```LoginFlowController.swift
final class LoginFlowController: UIViewController {
    private let dependencyContainer: DependencyContainer
    private var embeddedNavigationController: UINavigationController!
    weak var delegate: LoginFlowControllerDelegate?

    init(dependencyContainer: DependencyContainer) {
        self.dependencyContainer = dependencyContainer
        super.init(nibName: nil, bundle: nil)

        embeddedNavigationController = UINavigationController()
        add(childController: embeddedNavigationController)
    }

    func start() {
        let loginController = LoginController(dependencyContainer: dependencyContainer)
        embeddedNavigationController.viewControllers = [loginController]
    }
}
```  
  
### 7. UIResponderを継承している  
  
ViewControllerのボタンのタップイベントをCoordinatorに伝達する場合、  
delegateやclosureで処理を定義する必要が出てきます。  
  
※以下のようにUIReponderに適応させる方法もあるようです。  
参照記事: http://aplus.rs/2017/highly-maintainable-app-architecture/  
  
```Coordinator.swift
extension UIViewController {
	private struct AssociatedKeys {
		static var ParentCoordinator = "ParentCoordinator"
	}

	public var parentCoordinator: Any? {
		get {
			return objc_getAssociatedObject(self, &AssociatedKeys.ParentCoordinator)
		}
		set {
			objc_setAssociatedObject(self, &AssociatedKeys.ParentCoordinator, newValue, .OBJC_ASSOCIATION_ASSIGN)
		}
	}
}

open class Coordinator<T: UIViewController>: UIResponder, Coordinating {
	open var parent: Coordinating?	
	
	override open var coordinatingResponder: UIResponder? {
		return parent as? UIResponder
	}
}
```  
FlowControllerの場合、UIViewControllerなので  
UIResponderを活用することができます。  
  
  
### 8. FlowControllerから画面の表示などを管理できる  
  
FlowControllerが画面表示を行っているViewControllerの親になるため、  
[setOverrideTraitCollection](https://developer.apple.com/documentation/uikit/uiviewcontroller/1621406-setoverridetraitcollection)などを用いることで  
一箇所で画面表示の管理をすることができます。  
  
  
### 9. NavigationControllerの戻るボタン  
  
CoordinatorはNavigationControllerのスタックとは関係ないため、  
NavigationControllerの戻るボタンを押された際は、  
Coordinatorと同期を取る必要があります。  
  
```Coordinator.swift
extension Coordinator: UINavigationControllerDelegate {
    func navigationController(navigationController: UINavigationController, 
            didShowViewController viewController: UIViewController, animated: Bool) {
		
		// viewControllerがスタックにあるかを確認する
		guard
		  let fromViewController = navigationController.transitionCoordinator?.viewController(forKey: .from),
		  !navigationController.viewControllers.contains(fromViewController) else {
			return
    	}
		
		// 対象のCoordinatorの場合
		if fromViewController is TargetViewControllerInCoordinator) {
			//coordinatorをremoveする
		}
	}
}
```  
  
または　NavigationControllerとCoordinatorを管理するクラスが必要になります。  
参照記事: http://irace.me/navigation-coordinators  
  
```NavigationController.swift
final class NavigationController: UIViewController {

    private let rootViewController: UIViewController

    private var viewControllersToChildCoordinators: [UIViewController: Coordinator] = [:]

    private lazy var childNavigationController: UINavigationController =
      UINavigationController(rootViewController: self.rootViewController)

    init(rootViewController: UIViewController) {
        self.rootViewController = rootViewController
        super.init(nibName: nil, bundle: nil)
    }

    override func viewDidLoad() {
        super.viewDidLoad()

        ...
        childNavigationController.delegate = self         
        childNavigationController.interactivePopGestureRecognizer?.delegate = self
        ...
    }
}
extension NavigationController: UIGestureRecognizerDelegate {

    // childNavigationControllerのpop gestureに反応させるために必要
    func gestureRecognizer(gestureRecognizer: UIGestureRecognizer,
        shouldRecognizeSimultaneouslyWithGestureRecognizer otherGestureRecognizer: UIGestureRecognizer) -> Bool {
    
        return true
    }
}

extension NavigationController: UINavigationControllerDelegate {    
    func navigationController(navigationController: UINavigationController,
        didShowViewController viewController: UIViewController, animated: Bool) {

        cleanUpChildCoordinators()
    }

    private func cleanUpChildCoordinators() {
        for viewController in viewControllersToChildCoordinators.keys {
            if !childNavigationController.viewControllers.contains(viewController) {
                viewControllersToChildCoordinators.removeValueForKey(viewController)
            }
        }
    }
}
```  
  
FlowControllerの場合はUIViewControllerのため  
NavigationControllerの戻るボタンが押されると  
同時にFlowControllerもNavigationControllerのスタックから外れます。  
  
### 10. コールバックで画面遷移を依頼する  
  
あるViewControllerから次のViewControllerへ遷移する場合、  
delegateまたはclosureを用いてFlowControllerに伝達して画面遷移を行います。  
  
```swift
extension FlowController: ListControllerDelegate {
    func didSelect(_ controller: ListController, item: Item) {
        let detailController = DetailController(
        networkingService: dependencyContainer.NetworkingService
    )
        detailController.delegate = self
        embeddedNavigationController.pushViewController(detailController, animated: true)
    }
}
```  
  
```swift
final class FlowController {
    func start() {
        
        let listController = ListController(
            networkingService: dependencyContainer.NetworkingService
        )
    
        listController.didSelect = { [weak self] product in
            self?.showDetail(for: product)
        }
        embeddedNavigationController.viewControllers = [listController]
    }
}
```  
  
## まとめ  
  
全体の印象としてはCoordinatorと役割はほぼ同じため、  
処理の流れなどはほとんど変わらない印象でした。  
  
※作成したサンプルではVIPERを使用しており、  
クラスの関係が、  
ViewController → Presenter → Wireframe  
から  
Presenter ← ViewController → FlowController  
になったという変化がありました。  
  
一方で、記事にも多く書かれていたように  
CoorndinatorでViewControllerを別に管理しなければいけなかった部分がなくなり  
ややコードの量としてはすっきりしたのかなというように感じました。  
  
