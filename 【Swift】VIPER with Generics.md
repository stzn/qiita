## 経緯  
  
昨年末にViperについて学んだ時に、  
すごい便利だなと思う反面、同じような記述の繰り返しが増えるという点は  
Generambaのようなコード生成ツールを使ったとしてもなかなか大変だなだと感じていました。  
https://qiita.com/sztk1209/items/605ad32ab799530d57aa  
  
しかし、今年のUIKonf2018の動画を見ていて、  
それの解決策になりそうな方法を知ったので学習したことをメモします。  
https://www.youtube.com/watch?v=vGCCMgJW81g&t=0s&list=PLdr22uU_wISohI7PIhzq0gotGfKZl1lGo&index=17  
  
  
あくまで自分の理解できた範囲で書きましたので  
何か間違いやご指摘ございましたら教えていただけましたら幸いです:bow_tone1:  
  
  
## そもそもVIPERとは  
  
・責務の分離(それぞれの役割を決めて処理を切り分けよう)  
・疎結合(それぞれが依存をなるべくしないようにしよう)  
  
などを主な目的とし、  
  
それによって  
  
・テストが簡単にできるようになる(依存関係が減るので独立してテストができる)  
・変更・修正がしやすくなる(役割が明確なので、そこで何をやっているのかを把握しやすいかつ変更点が集約できる)  
  
などのメリットが得られる設計手法の一つです。  
  
  
  
  
主な登場部品としては、  
  
#### View   
画面の表示とユーザーからのアクションを受け取る  
  
#### Interactor   
PresenterからのリクエストでAPIやDBから必要なデータを取得する  
  
#### Presenter   
司令塔的な役割。Viewからのアクションを受け取って、  
Interactorにデータ取得を依頼したり、Interactorから渡されたデータに合わせてViewに画面の更新の指示を出したり、Routerへ画面遷移を依頼したりなどを行う。  
  
#### Entity   
業務モデルを表すクラス(構造体)。アプリケーションのコアとなる部分  
  
#### Router   
画面遷移を行う  
  
#### Scene   
各部品の初期化と依存の注入を行う  
  
があります。  
  
これらの部品をつなげるためにProtocolを利用します。  
  
(※作り方によるのかもしれませんが、  
Routerはプロジェクトで一つにまとめる場合などが多く直接繋げることが多いようです。)  
  
  
各部品の関係を表すと下記のようになります。  
![Viper.001.png](/image/04d2ed5e-e280-a542-6e23-c288d4be1dae.png)  
  
  
  
これで各部品の関係がすっきりとするのですが、  
このままですと各機能ごとに全てのProtocolを定義しなければならなくなります。  
  
例えばListを表示するための各部品、Listの一つのアイテムの詳細を表示するための各部品、  
が必要になります。  
  
  
機能が増えていくごとにどんどんと同じようなコードが増えていくのは  
なるべく避けたいですね。  
  
  
そこで今回勉強した動画の中ではGenericsを活用しています。  
  
  
まず、各部品がやっていることをデータの流れから考えると、  
  
```
1. View はViewEventを発行する。Presenterはそれを受けとるEventListenerである。

2. PresenterはInteractorにRequestを発行する。Interactorはそれを受け取るRequestListenerである。

3. InteractorはServiceからデータを取得する。受け取ったResponseを発行する。Presenterはそれを受けとるResponseListenerである。

4. PresenterはCommandを発行する。Viewはそれを受け取るCommandListenerである。ViewはCommandによって画面を更新する。
```  
といった流れができます。  
  
  
  
図で表すと下記のような形になります。  
  
![Viper.002.png](/image/01dd3186-6d65-f52d-3079-51bd32a8c5ba.png)  
  
  
この流れに沿ってGenericsを用いてProtocolを定義していきます。  
  
  
発行されるイベントをまず定義します。  
  
```Protocols.swift
protocol ViewEvent{}
protocol PresenterCommand{}
protocol InteractorRequest{}
protocol InteractorResponse{}
```  
各イベントの実装はenumで定義をします。(後ほど例を示します。)  
  
  
次にそれを受け取るリスナーを定義します。  
  
```Protocols.swift
protocol ResponseListener {
    associatedtype Response: InteractorResponse
    func handle(response: Response)
}

protocol RequestListener {
    associatedtype Request: InteractorRequest
    func handle(request: Request)
}

protocol EventListener {
    associatedtype Event: ViewEvent
    func handle(event: Event)
}

protocol CommandListener {
    associatedtype Command: PresenterCommand
    func handle(command: Command)
}
```  
  
上記のProtocolが各部品に適合するように各部品のProtocolを定義します。  
  
```Protocols.swift
protocol View: class, CommandListener {
    associatedtype Event: ViewEvent

    // !!!!!!!!!!!!!!!
    var eventListener: EventListener<Event>? { get set }  
}
```  
  
と、これはassociatedtypeによる制約のためできません。  
  
  
これを解決するためにTypeEraserを作成します。  
  
```TypeEraser.swift
class AnyRequestListener<T: InteractorRequest>: RequestListener {
    typealias Request = T
    private let handler: ((T) -> Void)
    init(handler: @escaping (T) -> Void) {
        self.handler = handler
    }
    
    func handle(request: T) {
        self.handler(request)
    }
}

class AnyResponseListener<T: InteractorResponse>: ResponseListener {
    typealias Response = T
    private let handler: ((T) -> Void)
    
    init(handler: @escaping ((T) -> Void)) {
        self.handler = handler
    }
    
    func handle(response: T) {
        self.handler(response)
    }
}

class AnyCommandListener<T: PresenterCommand>: CommandListener {
    typealias Command = T
    private let handler: ((T) -> Void)
    
    init(handler: @escaping ((T) -> Void)) {
        self.handler = handler
    }
    
    func handle(command: T) {
        self.handler(command)
    }
}


class AnyEventListener<T: ViewEvent>: EventListener {
    typealias Event = T
    private let handler: ((T) -> Void)
    
    init(handler: @escaping (T) -> Void) {
        self.handler = handler
    }
    
    func handle(event: T) {
        self.handler(event)
    }
}
```  
  
改めて各部品のProtocolを定義します。  
  
```Protocols.swift
protocol Interactor: class, RequestListener {
    associatedtype Response: InteractorResponse
    var responseListener: AnyResponseListener<Response>? { get set }
}

protocol Presenter: class, EventListener, ResponseListener {
    associatedtype Command: PresenterCommand
    associatedtype Request: InteractorRequest
    var requestListener: AnyRequestListener<Request>? { get set }
    var commandListener: AnyCommandListener<Command>? { get set }
    var router: Router? { get set }
    var scenePresenter: ScenePresenter? { get set }
}

protocol View: class, CommandListener {
    associatedtype Event: ViewEvent
    var eventListener: AnyEventListener<Event>? { get set }
}
```  
  
Genericsを使わなかった場合の各部品の関係と  
全く変わらずにGenericsに対応できました。  
  
  
最後に各部品をつなげます。  
  
最後に各部品を構築する時にそれぞれの関係がつながるようにします。  
部品を構築するのは前と同じでSceneです。  
  
  
```Scene.swift
protocol Scene {
    var viewController: UIViewController { get }
}

extension Scene {
    
    func build<V: View, P: Presenter, I: Interactor>(
        view: V,
        presenter: P,
        interactor: I,
        scenePresenter: ScenePresenter?)
        where
        V.Event == P.Event,
        I.Request == P.Request,
        I.Response == P.Response,
        V.Command == P.Command {
            
            
            presenter.commandListener = AnyCommandListener<P.Command>(handler: view.handle)
            presenter.requestListener = AnyRequestListener<P.Request>(handler: interactor.handle)
            view.eventListener = AnyEventListener<P.Event>(handler: presenter.handle)
            interactor.responseListener = AnyResponseListener<P.Response>(handler: presenter.handle)
            if let scenePresenter = scenePresenter {
                presenter.router = Router()
                presenter.scenePresenter = scenePresenter
            }
    }
}
```  
  
  
サンプルを作成してみました。単純にQiitaの記事を取得してくるだけのものです。  
一部を下記に載せます。  
  
  
```AppScene.swift
enum AppScene: Scene {

    case qiitaItemList
    
    var viewController: UIViewController {
        switch self {
        case .qiitaItemList:
            return configureQiitaItemsList()
        }
    }
    
    private func configureQiitaItemsList() -> UIViewController {
        
        let viewController = QiitaItemsListViewController.storyboardInstance
        let presenter = QiitaItemsListPresenter()
        let interactor = QiitaItemsListInteractor(baseApiClient: BaseAPIClient.shared)
        
        self.build(view: viewController,
                   presenter: presenter,
                   interactor: interactor,
                   scenePresenter: viewController)
        let navigationController = UINavigationController(rootViewController: viewController)
        return navigationController
    }    
}
```  
  
  
```QiitaItemListProtocols.swift

enum QiitaItemListPresenterCommand: PresenterCommand {
    case reload(list: [QiitaItem])
    case scrollTop
    case showError(title: String, message: String)
    case showNoContent
}

enum QiitaItemListViewEvent: ViewEvent {
    case viewDidLoad
    case didSelect(item: QiitaItem)
    case searchBarTextDidChange(text: String)
    case loadMore(text: String)
    case refresh(text: String)
}

enum QiitaItemListInteractorRequest: InteractorRequest {
    case fetchList(query: String, page: Int)
}

enum QiitaItemListInteractorResponse: InteractorResponse {
    case listReceived(result: ServiceResult<[QiitaItem]>)
}
```  
  
  
  
```QiitaItemsListPresenter.swift
final class QiitaItemsListPresenter: Presenter {
    
    typealias Command = QiitaItemListPresenterCommand
    typealias Request = QiitaItemListInteractorRequest
    typealias Event = QiitaItemListViewEvent
    typealias Response = QiitaItemListInteractorResponse
    
    var requestListener: AnyRequestListener<QiitaItemListInteractorRequest>?
    var commandListener: AnyCommandListener<QiitaItemListPresenterCommand>?
    var router: Router?
    var scenePresenter: ScenePresenter?
    
   // ViewEventを受け取る
    func handle(event: QiitaItemListViewEvent) {
        switch event {
        case .viewDidLoad:
            self.requestListener?.handle(request: .fetchList(query: "", page: 1))
        case .didSelect(let item):
            self.didSelect(item)
        case .searchBarTextDidChange(let text):
            self.searchBarTextDidChange(text: text)
        case .loadMore(let text):
            self.loadMore(searchText: text)
        case .refresh(let text):
            self.refresh(text: text)
        }
    }
    
    func handle(response: QiitaItemListInteractorResponse) {
        switch response {
        case .listReceived(let result):
            self.listReceived(result: result)
        }
    }    
}

// MARK: - Extensions -
extension QiitaItemsListPresenter {
    
    func viewDidLoad() {
        // リクエストを発行
        requestListener?.handle(request: .fetchList(query: "", page: 1))
  
    }
        
    func didSelect(_ item: QiitaItem) {
        
        guard let presenter = scenePresenter else {
            return
        }
        // Routerに画面遷移を依頼
        router?.present(scene: AppScene.qiitaItemDetail(item: item), scenePresenter: presenter)
    }
}

extension QiitaItemsListPresenter {
    
    private func listReceived(result: ServiceResult<[QiitaItem]>) {
        
        switch result {
        case .success(let value):
            // コマンドを発行
            commandListener?.handle(command: .reload(list: state.contents))          
        case .failure(let error):
            // コマンドを発行
            commandListener?.handle(command: .showError(title: "", message: error.debugDescription))
        }
    }
}
```  
  
  
```QiitaItemsListInteractor.swift
final class QiitaItemsListInteractor: Interactor {
    
    typealias Response = QiitaItemListInteractorResponse
    typealias Request = QiitaItemListInteractorRequest
    
    var responseListener: AnyResponseListener<QiitaItemListInteractorResponse>? 
    
    private let baseApiClient: BaseAPIClient
    
    init (baseApiClient: BaseAPIClient) {
        self.baseApiClient = baseApiClient
    }
    
    // リクエストを受け取る
    func handle(request: QiitaItemListInteractorRequest) {
        switch request {
        case .fetchList(let query, let page):
            fetchList(query: query, page: page)
        }
    }
    
    private func fetchList(query: String, page: Int) {
        let resource = Resource<[QiitaItem]>(requestRouter: RequestRouter.fetchList(query: query, page: page))
        
        do {
            
            try baseApiClient.request(resource) { [weak responseListener] result in
                
                switch result {
                case .success(let value):
                        // レスポンスを渡す
                    responseListener?.handle {
                        response: .listReceived(result: ServiceResult.success(value)))
                case .failure(let error):
                        // レスポンスを渡す
                    responseListener?.handle(
                        response: .listReceived(result: ServiceResult.failure(error)))
                }
            }
            
        } catch {
            // レスポンスを渡す
            responseListener?.handle(
                response: .listReceived(result: ServiceResult.failure(error as! ApplicationError)))
        }
    }
}
```  
  
こんな感じに動きます。  
  
![viper.mov.gif](/image/1240a9b8-f411-a065-7b35-2cf6b7a2e7fa.gif)  
  
  
GitHubに今回のサンプルを載せました。  
https://github.com/stzn/ViperWithGenerics  
  
## まとめ  
VIPERは各部品の役割を明確に決めることで、多くのメリットが得られる一方で、  
テンプレート的な記述が多くなるという印象を受けていました。  
  
今回Genericsを導入することでそういった部分は少し軽減できたのではないかと感じています。  
※ファイル数はあまり変わらないのかもしれませんが、、、  
  
  
VIPERは実務ではまだ使ったことはないのですが、  
機会があればぜひ使ってみたいと思います。  
