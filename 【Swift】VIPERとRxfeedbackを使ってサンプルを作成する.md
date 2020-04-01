RxSwiftについて調べている中で[Rxfeedback](https://academy.realm.io/posts/try-swift-nyc-2017-krunoslav-zaher-modern-rxswift-architectures/)というものを見つけ、どのように使うのか気になってサンプル作成に挑戦してみました。  
  
## Rxfeedbackとは？  
  
※認識が間違っていたらすいません。  
  
```
typealias FeedbackLoop<State, Event> = (Observable<State>) -> Observable<Event>
typealias FeedbackLoop<State, Event> = (Driver<State>) -> Driver<Event>
```  
  
アプリケーションの状態(mutableなもの)が特定の状態へと変化することをトリガーに  
UIの更新やAPIの呼び出しなどを引き起こし、  
その結果をイベントとして発行することで、  
さらにアプリーケーションの状態を別の状態に変化させる  
  
という循環的な状態変化(=FeedbackLoop)をRxSwiftを用いて簡潔に宣言的に記載できるようにしたもの  
  
  
Rxfeedbackの中で宣言されている下記のメソッドを見るとFeedbackLoopの流れがわかりやすいのかなと思います。  
  
```
public func react<State, Query, Event>(
    query: @escaping (State) -> Query?,
    areEqual: @escaping (Query, Query) -> Bool,
    effects: @escaping (Query) -> Observable<Event>
    ) -> (ObservableSchedulerContext<State>) -> Observable<Event> {
    return { state in
        return state.map(query)
            .distinctUntilChanged { lhs, rhs in
                switch (lhs, rhs) {
                case (.none, .none): return true
                case (.none, .some): return false
                case (.some, .none): return false
                case (.some(let lhs), .some(let rhs)): return areEqual(lhs, rhs)
                }
            }
            .flatMapLatest { (control: Query?) -> Observable<Event> in
                guard let control = control else {
                    return Observable<Event>.empty()
                }

                return effects(control)
                    .enqueue(state.scheduler)
            }
    }
}
```  
  
<br>  
  
下記のsystem関数を用いて状態変化を起こします。  
  
  
```
func system<State, Event>(
	initialState: State, // 初期状態
	reduce: (State, Event) -> State, // 現在の状態とイベントから新しい状態を生成する
	feedback: FeedbackLoop<State, Event>... // イベントをトリガーに引き起こされる処理、最後にまたイベントを発行する
) -> Observable<State>
```  
  
## なぜ使うのか？  
  
・アプリケーションの状態(副作用が起きる箇所)を一箇所で管理できる  
・書き方が統一できる(作者によると少なくとも90%は上記のパターンで処理できるとおっしゃっています)  
・  
## いつ使うのか？  
  
典型的な2つの使用方法として  
・API呼び出し(状態変化からURLを受け取り、API呼び出し結果を受け取りObservableを取得して処理を行い、再びイベントを発行する)  
・UI更新(状態変化の結果をUIにbindする。また、UIイベントを監視し、再び状態変化を起こす)  
が挙げられていました。  
  
[RxfeedbackのGithubリポジトリのサンプル](https://github.com/NoTests/RxFeedback.swift/blob/master/Examples/GithubPaginatedSearch.swift)の中で紹介されている一覧画面でのページング処理が個人的には一番イメージしやすかったです。  
  
  
## どう使うのか？(実装)  
  
上記のサンプルを参考に[以前作成したVIPERを使ったサンプル](https://qiita.com/sztk1209/items/a256ba9e0ce6e5249e22)を元にRxfeedbackを使ってサンプルを作成しました。  
  
  
今回の作成物:  
https://github.com/stzn/ViperSampleGeneramba/tree/rxfeedback  
  
  
  
**QiitaItemsListViewController.swift(一部抜粋)**  
```swift:QiitaItemsListViewController.swift(一部抜粋)

import UIKit
import RxSwift
import RxCocoa
import RxFeedback

final class QiitaItemsListViewController: UIViewController {
    
    @IBOutlet weak var noResultLabel: UILabel!
    @IBOutlet weak var tableView: UITableView!
    
    // MARK: - Public properties -
    
    var presenter: QiitaItemsListPresenterInterface!
    
    // MARK: - Private properties -
    
    private var searchController = UISearchController(searchResultsController: nil)
    
    private let refreshControl = UIRefreshControl()
    private let disposeBag = DisposeBag()

    
    private typealias State = QiitaItemsListPresenter.State
    private typealias Event = QiitaItemsListPresenter.Event
    
    private func _setupUI() {
        
        navigationItem.title = "List"
        
        definesPresentationContext = true
        searchController.obscuresBackgroundDuringPresentation = false
        
        
        let triggerLoadNextPage: (Driver<State>)  -> Signal<Event> = { state in
            return state.flatMapLatest { [weak self] state -> Signal<Event> in
                
                guard let `self` = self else { return Signal.empty() }
                
                if state.shouldLoadNext || state.error != nil {
                    return Signal.empty()
                }
                return self.tableView.rx.reachBottom.map { _ in Event.startLoadingNextPage }
            }
        }
        
        let bindUI: (Driver<State>) -> Signal<Event> = bind(self) { me, state in
            
            let subscriptions = [
                state.map { $0.searchText }.drive(me.searchController.searchBar.rx.text),
                state.filter{ $0.error != nil }.map{ $0.error }
                    .drive(onNext: { error in
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.1) {
                            me.showErrorAlert(with: error!.localizedDescription)
                        }
                    }),
                state.map { $0.contents }.drive(me.tableView.rx.items(cellIdentifier: QiitaItemTableViewCell.reuseIdentifier, cellType: QiitaItemTableViewCell.self))(me.configureCell)
            ]
            
            let events: [Signal<Event>] = [
                me.searchController.searchBar.rx.text.orEmpty.changed.asSignal().map(Event.searchChanged),
                me.searchController.searchBar.rx.cancelButtonClicked.asSignal().map(Event.searchReset),
                me.refreshControl.rx.controlEvent(.valueChanged).asSignal().map(Event.refreshing),
                triggerLoadNextPage(state)
            ]
            return Bindings(subscriptions: subscriptions, events: events)
        }
        
        Driver.system(
            initialState: State.empty,
            reduce: State.reduce,
            feedback:
            bindUI,
            react(query: { $0.loadNextPage }, effects: { [weak self] (text, nextPage) in
                
                guard let `self` = self else { return Signal.empty() }
                
                if !self.refreshControl.isRefreshing {
                    self.showLoading()
                }
                
                return self.presenter
                    .fetchList(text: text, nextPage: nextPage)
                    .asSignal(onErrorJustReturn: .error(NetworkError.offline))
                    .do(onNext: { [weak self] _ in
                        
                        guard let `self` = self else { return }
                        
                        self.hideLoading()
                        if self.refreshControl.isRefreshing {
                            self.refreshControl.endRefreshing()
                        }
                    })
                    .map(Event.response)
                
            })
            )
            .drive()
            .disposed(by: disposeBag)
        
    }        
}
```  
  
  
**QiitaItemsListPresenter.swift(一部抜粋)**  
```swift:QiitaItemsListPresenter.swift(一部抜粋)

final class QiitaItemsListPresenter {
    
    struct State {
        
        var searchText: String {
            didSet {
                if searchText.isEmpty {
                    nextPage = 1
                    shouldLoadNext = false
                    contents = []
                    error = nil
                    return
                }
                shouldLoadNext = true
                error = nil
            }
        }
        
        var nextPage: Int = 1
        var shouldLoadNext: Bool = true
        var contents: [QiitaItem]
        var error: Error? = nil
        
        static var empty: State {
            return State(searchText: "", nextPage: 1, shouldLoadNext: true, contents: [], error: nil)
        }
        
        var loadNextPage: (searchText: String, nextPage: Int)? {
            return shouldLoadNext ? (searchText, nextPage) : nil
        }
        
        static func reduce(state: State, event: Event) -> State {
            switch event {
            case .searchChanged(let searchText):
                var result = state
                result.searchText = searchText
                result.contents = []
                result.nextPage = 1
                return result
            case .response(.success(let response)):
                var result = state
                result.contents += response
                if !response.isEmpty {
                    result.nextPage += 1
                    result.error = nil
                } else {
                    result.error = NetworkError.noData
                }
                result.shouldLoadNext = false
                result.error = nil
                return result
            case .response(.error(let error)):
                var result = state
                result.shouldLoadNext = false
                result.error = error
                return result
            case .startLoadingNextPage:
                var result = state
                result.shouldLoadNext = true
                return result
            case .refreshing:
                var result = state
                result.shouldLoadNext = false
                result.contents = []
                result.nextPage = 1
                result.error = nil
                return result
            case .searchReset:
                var result = state
                result.searchText = ""
                return result
            }
        }
    }
    
    enum Event {
        case searchChanged(String)
        case response(QiitaItemsResponse)
        case startLoadingNextPage
        case refreshing(Any)
        case searchReset(Any)
    }
    
    typealias QiitaItemsResponse = Result<[QiitaItem]>    
}

```  
  
## 良かった点  
  
・変数の宣言とそれに対応した処理を書く必要がなくなり書く量が減った。  
・上記に伴って処理を追う労力が減り、他に力を注げるようになった。  
・状態のモックを作成し、feedbackに注入することで簡単にテストができる  
  
## 難しかった点  
  
・処理の流れを追えるようになるのに時間がかかった(仕組みを理解しないと「次はどこで何をするんだっけ？」とわからなくなりました。それに上記に伴ってうまくいかない原因を探すのにも始め時間がかかりました。)  
  
・状態の設定を間違えて無限ループが発生した(APIリクエストが延々と起こりリクエスト上限に達してしまったことがありました。)  
  
  
## わからない点  
  
エラーが発生した場合にアラートを表示するようにしたのですが、  
データが存在しない状態で、リフレッシュを行なったときにエラーが発生した場合、refreshControlのendRefreshing()を呼んでもrefreshControlが隠れませんでした。  
  
  
色々試していたところ  
  
```
me.showErrorAlert(with: error!.localizedDescription)
```  
  
を  
  
```
DispatchQueue.main.asyncAfter(deadline: .now() + 0.1) {
    me.showErrorAlert(with: error!.localizedDescription)
}
```  
とすることで解決しました(?)が、これで良いのかがわからないです。。。(スレッドの問題?)  
  
## 課題  
  
・今のところページング処理以外の使用用途が思いつかないので利用できそうな場面を考える。  
・[ActivityIndicator](https://github.com/sergdort/CleanArchitectureRxSwift/blob/master/CleanArchitectureRxSwift/Utility/ActivityIndicator.swift)や[ErrorTracker](https://github.com/sergdort/CleanArchitectureRxSwift/blob/master/CleanArchitectureRxSwift/Utility/ErrorTracker.swift)を使うともっと処理が簡潔に書けそうなので使い方を調べる。  
  
  
  
<br>  
簡潔に書けるのですごい便利な気がする一方、仕組みをもっと理解しないと落とし穴がありそうなので、さらに試していって実際の開発でも活用できればいいなと感じました。  
  
