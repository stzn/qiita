  
## 経緯  
  
[トップダウンエラーハンドリングという記事](https://medium.com/@londeix/top-down-error-architecture-d8715a28d1ad)を読んでいて、  
確かに非同期処理中にthrowするようなエラーが発生すると  
コールバックの中で同じような処理を何度も書いて面倒だなと思うことが  
多々あるなと感じました。  
  
  
こんな感じでしょうか？  
![1.png](/image/b8eacbc9-6a1d-c92f-f8f3-f42cf9420132.png)  
※Coordinatorは画面遷移を制御するクラスのようなものです。  
  
特にthrowするようなエラーは特定の機能に関係するものよりは  
APIの403や500などのHTTPステータスエラー、継続不能なアプリ内のエラー)など  
一般的なエラーが多く上の階層でまとめて処理したいという気持ちが強くあります。  
  
そこでエラー処理をうまく一箇所にまとめられないか  
エラーハンドリングについていくつか検討してみました。  
  
  
## トップダウンエラーハンドリングとは？  
  
通常エラーが発生した箇所から上の階層へ順々にエラー処理の判定をしていくが、  
これを２段階に分けてトップダウンでエラー処理をしていくという方式です。  
  
1段階目　エラーをルートまで伝達をする。  
2段階目　ルートから順にエラーハンドリングできる階層まで下っていく  
  
```Errors.swift
enum ApiError: LocalizedError {
    case firstError
    case firstNextError
    case secondError
    
    var errorDescription: String? {
        switch self {
        case .firstError:
            return "First Error"
        case .firstNextError:
            return "First Next Error"
        case .secondError:
            return "Second Error"
        }
    }
}
enum UnexpectedError: LocalizedError {
    case only
    var errorDescription: String? {
        return "Unexpected Error"
    }
    
}
```  
  
```ErrorHandler.swift

public typealias HandleAction<T> = (T) throws -> ()
public typealias ErrorMatcher = (Error) -> Bool

public protocol ErrorHandleable: class {
    func `throw`(_: Error, finally: @escaping (Bool) -> Void)
    func `catch`(action: @escaping HandleAction<Error>) -> ErrorHandleable
}

public class ErrorHandler: ErrorHandleable {
    private var parent: ErrorHandler?
    private let action: HandleAction<Error>
    
    public convenience init(action: @escaping HandleAction<Error> = { throw $0 }) {
        self.init(action: action, parent: nil)
    }
    
    private init(action: @escaping HandleAction<Error>, parent: ErrorHandler? = nil) {
        self.action = action
        self.parent = parent
    }

    public func `throw`(_ error: Error, finally: @escaping (Bool) -> Void) {
        `throw`(error, previous: [], finally: finally)
    }
    
    private func `throw`(_ error: Error, previous: [ErrorHandler], finally: ((Bool) -> Void)? = nil) {
        
        // 1段階目
        if let parent = parent {
            parent.`throw`(error, previous: previous + [self], finally: finally)
            return
        }
        // 2段階目
        serve(error, next: AnyCollection(previous.reversed()), finally: finally)
    }
    
    private func serve(_ error: Error, next: AnyCollection<ErrorHandler>, finally: ((Bool) -> Void)? = nil) {
        do {
            try action(error)
            finally?(true)
        } catch {
            if let nextHandler = next.first {
                nextHandler.serve(error, next: next.dropFirst(), finally: finally)
            } else {
                finally?(false)
            }
        }
    }
    
    public func `catch`(action: @escaping HandleAction<Error>) -> ErrorHandleable {
        return ErrorHandler(action: action, parent: self)
    }
}

public extension ErrorHandleable {
    func `do`<A>(_ section: () throws -> A) {
        do {
            _ = try section()
        } catch {
            `throw`(error)
        }
    }
}

public extension ErrorHandleable {
    func `throw`(_ error: Error) {
        `throw`(error, finally: { _ in })
    }
}

public extension ErrorHandleable {
    func `catch`<K: Error>(_ type: K.Type, action: @escaping HandleAction<K>) -> ErrorHandleable {
        return `catch`(action: { (e) in
            if let k = e as? K {
                try action(k)
            } else {
                throw e
            }
        })
    }
    func `catch`<K: Error>(_ value: K, action: @escaping HandleAction<K>) -> ErrorHandleable where K: Equatable {
        return `catch`(action: { (e) in
            if let k = e as? K, k == value {
                try action(k)
            } else {
                throw e
            }
        })
    }
}
...

(以下略)

```  
  
```RootViewController.swift

final class RootViewController: UIViewController, AlertShowable {
    
    // 
    private let excludeErrors: [ErrorMatcher] = [
        { error in (error is ApiError) }
    ]
    
    private lazy var errorHandler = {
        
        ErrorHandler()
        .catch(UnexpectedError.self) { [unowned self] error in
            self.showAlert(title: "Root Error", message: error.localizedDescription)
        
        }
        // 最終的に特定のエラーに当てはまらなかった場合
        .catch { [unowned self] error in

            // 下の階層に渡す必要があるエラーの場合はハンドリングしない
            if self.excludeErrors.contains(where: { $0(error) }) {
                throw error
            }

            self.showAlert(title: "Root Error", message: "Somethig wrong!!!")
        }
    }()
    
    func showFirst() {

        // ErrorHandlerは下の階層に渡していく
        let first = FirstFlowController(errorHandler)
        
        ...
    }
}
```  
  
```FirstFlowController.swift

final class FirstFlowController: UIViewController, AlertShowable {
    
    private let parentErrorHandler: ErrorHandleable
    
    // 上の階層のエラーにさらにハンドリング処理を追加する
    private var errorHandler: ErrorHandleable {
        return parentErrorHandler.catch(ApiError.firstError) { [weak self] error in
            self?.showAlert(title: "Error", message: error.localizedDescription)
        }.catch(ApiError.firstNextError) { [weak self] error in
            self?.showAlert(title: "Error", message: error.localizedDescription)
        }
    }

    init(_ parentErrorHandler: ErrorHandleable) {
        self.parentErrorHandler = parentErrorHandler

        ...
    }
}

extension FirstFlowController: FirstViewDelegate {
    func goToNext(user: User) {
        let viewController = FirstNextViewController.fromStoryboard()
        let viewModel = FirstNextViewModel(errorHandler: errorHandler, user: user)
        viewController.viewModel = viewModel
        navigation.pushViewController(viewController, animated: true)
    }
}
```  
  
```FirstNextViewModel.swift

struct FirstNextViewModel {
    
    private let errorHandler: ErrorHandleable
    let user: User
    
    init(errorHandler: ErrorHandleable, user: User) {
        self.errorHandler = errorHandler
        self.user = user
    }
    
    func throwError() throws {
        
        asyncAction {
            throw ApiError.firstError
        }
    }
 
    func throwFirstNextError() throws {
        asyncAction {
            throw ApiError.firstNextError
        }
    }

    func throwUnexpectedError() throws {
        asyncAction {
            throw UnexpectedError.only
        }
    }
    
    func throwNSError() throws {
        asyncAction {
            throw NSError(domain: "example.stzn.nseeror", code: 999, userInfo: ["message": "This is NSError!!"])
        }
    }

    func asyncAction(callback: @escaping () throws -> Void) {
        ApiClient.doAsync {
            // 何かしらのエラーを発生させる            
            do {
                try callback()
            } catch let error {
                self.errorHandler.throw(error)
            }
        }
    }
}
```  
  
### 良いと思うところ  
  
・逐一処理を書かないで済むようになる  
  
・必要な場合は下の階層にエラーを伝播して処理させることができる  
  
### 悩みどころ  
  
・最終的にハンドリングできなかった場合のエラー処理をするために下の階層に渡す必要があるエラーを除外しなければいけない  
  
・ErrorHandlerを引数で渡していかなければならない  
  
・記事にもあるように、networkエラーなどエラーのスタックが欲しい場合には対応できない  
※以下を参考にしてログは残しておくなどの対処が良いのかもしれません。  
https://tech.just-eat.com/2018/01/26/effective-ios-error-management/  
  
  
## 他の方法も探してみる  
  
他にも何かないかと探してみたところ、  
以下の動画も参考にしてみました。  
https://academy.realm.io/posts/try-swift-nyc-2017-eleni-papanikolopoulou-kostas-kremizas-error-handling/  
  
  
```ErrorHandler(一部省略).swift

extension Notification.Name {
    public static let errorOccured = Notification.Name("errorOccured")
}

public enum HandleStrategy {
    case next
    case stop
}

public typealias ErrorAction = (Error) -> HandleStrategy
public typealias Tag = String
    
public final class ErrorHandler {

    // ErrorMatcherは(Error)->Boolをクラスにしたようなもの    
    private var errorActions: [(ErrorMatcher, ErrorAction)] = []
    private var onNoMatch:[ErrorAction] = []
    private var alwaysActions: [ErrorAction] = []
    private var errorStack: [Error] = []
    
    public init() {}
    
    public func on(matches: @escaping ((Error)-> Bool), do action: @escaping ErrorAction) -> ErrorHandler {
        let matcher = ClosureErrorMatcher(matches: matches)
        return on(matcher, do: action)
    }
    
    public func on(_ matcher: ErrorMatcher, do action: @escaping ErrorAction) -> ErrorHandler {
        errorActions.append((matcher, action))
        return self
    }
    
    public func onNoMatch(do action: @escaping ErrorAction) -> ErrorHandler {
        onNoMatch.append(action)
        return self
    }
    
    public func always(do action: @escaping ErrorAction) -> ErrorHandler {
        alwaysActions.append(action)
        return self
    }
    
    
    public func handle(_ error: Error) {
        
        // 最後に必ず実行するアクション
        defer {
            for alwaysAction in alwaysActions.reversed() {
                let alwaysHandleStrategy = alwaysAction(error)
                if case .stop = alwaysHandleStrategy {
                    break
                }
            }
        }

        // 条件にあったエラーに対してアクションをする
        var found = false
        for (matcher, action) in errorActions.reversed() {
            if matcher.matches(error) {
                found = true
                let strategy = action(error)
                // stopがあっても引き続きアクションは実行していく
                if case .stop = strategy {
                    return
                }
            }
        }
        
        // アクション済みの場合は終了
        if found { return }
        
        // 条件に合うエラーではなかったエラーに対してアクションをする
        for otherwise in onNoMatch.reversed() {
            let strategy = otherwise(error)
            if case .stop = strategy {
                break
            }
        }
    }
    
    // ErrorのTypeにまとめて同じアクションを設定する
    public func onError<T: Error>(ofType type: T.Type, do action: @escaping ErrorAction) -> ErrorHandler {
        return on(ErrorTypeMatcher<T>(), do: action)
    }

    public func on<E:Error & Equatable>(_ error: E, do action: @escaping ErrorAction) -> ErrorHandler {
        return self.on(matches: { (handleError) -> Bool in
            guard let handleError = handleError as? E else { return false }
            return handleError == error
        }, do: action)

    }

    public func onNSError(domain: String, code: Int? = nil, do action: @escaping ErrorAction) -> ErrorHandler {
        return self.on(NSErrorMatcher(domain: domain, code: code), do: action)
    }
    
    public func postError(_ error: Error) {
        NotificationCenter.default.post(name: .errorOccured, object: nil, userInfo: ["error": error])
    }
}
```  
  
```DefaultErrorHandler.swift

extension ErrorHandler {
    
    static var `default`: ErrorHandler {
        
        return ErrorHandler()
            .onError(ofType: ApiError.self,
                do: { error -> HandleStrategy in
                    print(error.localizedDescription)
                    AlertManager.shared.showErrorAlert(message: error.localizedDescription)
                    return .next
            })
            .onError(ofType: UnexpectedError.self,
                     do: { error -> HandleStrategy in
                        print(error.localizedDescription)
                        AlertManager.shared.confirm(title: "Error", message: "Exit?")
                        return .next
            })
           .onNSError(domain: "stzn.sample.nserror",
                do: { error -> HandleStrategy in
                    let nserror = error as NSError
                    let message = "NSError \n domain:\(nserror.domain) \n code:\(nserror.code) \n userInfo:\(nserror.userInfo)"
                    print(message)
                    AlertManager.shared.showErrorAlert(message: message)
                    return .next
            })
            .onNoMatch(do: {_ in
                AlertManager.shared.showErrorAlert(message: "Something wrong!!")
                print("Something wrong!!")
                return .stop
            })
            .always(do: { _ in
                print("Finish error handle!!")
                return .stop
            })
    }
}
```  
  
```RootViewController.swift

final class RootViewController: UIViewController {
    
    private var errorHandler = ErrorHandler.default
    private var token: NSObjectProtocol?
    
    override init(nibName nibNameOrNil: String?, bundle nibBundleOrNil: Bundle?) {
        super.init(nibName: nil, bundle: nil)
        
        token = NotificationCenter.default.addObserver(forName: .errorOccured, object: nil, queue: nil) { [weak self] note in
            guard let strongSelf = self,
                let info = note.userInfo,
                let error = info["error"] as? Error else {
                    return
            }
            strongSelf.errorHandler.handle(error)
        }
    }
    
    deinit {
        token = nil
    }    
}
```  
  
```FirstFlowController.swift(一部省略)

final class FirstFlowController: UIViewController {
    
    // こっちはErrorHandlerを渡していっているだけ
    private let errorHandler: ErrorHandler
    
    init(_ errorHandler: ErrorHandler) {
        self.errorHandler = errorHandler
        ...
    }    
}
```  
  
``` FirstNextViewModel.swift

struct FirstNextViewModel {
    
    private let errorHandler: ErrorHandler

    init(errorHandler: ErrorHandler) {
        self.errorHandler = errorHandler
    }
    
    func throwError() throws {
        
        asyncAction {
            self.errorHandler.postError(ApiError.firstError)
        }
    }
 
    func asyncAction(callback: @escaping () -> Void) {
        // 何かしらの非同期処理
        ApiClient.doAsync {
            callback()
        }
    }
}
```  
  
### 良かったと思う点  
・エラーハンドリングを一箇所で管理できるようになった  
  
・エラーをthrowするときにNotificationを使用しているが、RootViewControllerで管理しているため通知解除の処理がわずらわしくない  
  
・必要に応じでdefaultのエラーハンドリングを変更することができる  
  
### 悩みどころ  
  
・アラートを表示する以外などのエラー処理が必要な場合に、ErrorHandlerから再度処理の移譲をしなければならない(例えばRootViewControllerのメソッドを呼び出すなど)  
  
・さらに特定の画面に対して処理が必要な場合は、ルートから辿っていく必要がある。またはDelegateや通知で知らせる方法もあるが、それはそれで管理する対象が増えてしまう。  
  
  
## まとめ  
  
エラー処理をまとめて管理できることは  
今後の修正や拡張という点で楽になりそうな気がしました。  
  
一方で、エラーによって特定の画面の一部を変更するなど  
仕様に左右される部分が出てきた際に苦労することもありそうだとも感じました。  
  
一概に使えるというものは存在しないと思いますので、  
都度都度、どうしたらもっとシンプルにコードが書くことができるようになるのか  
ということは追い求めていきたいです。  
  
  
作成したサンプル: https://github.com/stzn/ErrorHandlerSample  
