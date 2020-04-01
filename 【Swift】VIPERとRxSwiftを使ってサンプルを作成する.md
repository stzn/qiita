  
[前回](https://qiita.com/sztk1209/items/605ad32ab799530d57aa)の続きです。  
  
## 経緯  
  
普段はDelegateパターンをよく使っているのですが、  
  
・メソッドの宣言と呼び出す箇所が分かれているため探すのに苦労する(Cmd+クリックを多用する)  
・処理が離れているため一連の処理の流れが把握するのに時間がかかる。  
・クラス間のやりとりが雑多になる(書き方の問題なのでしょうが、双方を行ったり来たりする処理が増えます。)  
  
  
このようなことからReactiveプログラミングを学ぼうと思い、  
VIPERを使用して作成したサンプルに今回はRxSwiftを加えるとどうなるのかを試してみました。  
  
RxSwiftのバージョンは4.1.0です。  
  
  
## 実装  
  
作成物は[Github](https://github.com/stzn/ViperSampleGeneramba/tree/rxswift)に上げています。  
  
以下にQiitaから取得したデータをリスト表示する処理の一部を記載します。  
  
基本的な流れとしては、  
・Viewでユーザーの入力を受け取り、Signal, DriverとしてPresenterに渡す  
・PresenterからInteractor呼び出す  
・InteractorからAPI経由でデータを取得し、PresenterへデータをSingleとして渡す  
・Presenterでデータを加工/計算などを行い、Signal, DriverとしてViewに渡す  
  
  
```QiitaItemDTO.swift
struct QiitaItem: Decodable, Equatable {
    let id: String
    let likes_count: Int
    let reactions_count: Int
    let comments_count: Int
    let title: String
    let created_at: String
    let updated_at: String
    let url: URL
    let tags: [Tag]
    let user: User?
    
    static func ==(lhs: QiitaItem, rhs: QiitaItem) -> Bool {
        return lhs.id == rhs.id
        && lhs.likes_count == rhs.likes_count
        && lhs.reactions_count == rhs.reactions_count
        && lhs.comments_count == rhs.comments_count
        && lhs.title == rhs.title
        && lhs.created_at == rhs.created_at
        && lhs.updated_at == rhs.updated_at
        && lhs.tags == rhs.tags
        && lhs.url == rhs.url
        && lhs.user == rhs.user
    }
    

    struct Tag: Decodable, Equatable {
        let name: String
        
        static func ==(lhs: QiitaItem.Tag, rhs: QiitaItem.Tag) -> Bool {
            return lhs.name == rhs.name
        }
    }
    
    struct User: Decodable, Equatable {
        let github_login_name: String?
        let profile_image_url: String?
        
        static func ==(lhs: QiitaItem.User, rhs: QiitaItem.User) -> Bool {
            return lhs.github_login_name == rhs.github_login_name
            && lhs.profile_image_url == rhs.profile_image_url
        }
    }
}
```  
  
  
```QiitaItemsListInteractor.swift
final class QiitaItemsListInteractor {
    
    weak var output: QiitaItemsListInteractorOutputInterface?
    
    // 参照を保持しておく必要がある
    private var api: API.QiitaItemsApi!
}

// MARK: - Extensions -

extension QiitaItemsListInteractor: QiitaItemsListInteractorInterface {
    func fetchList(query: String, page: Int) -> Single<Result<[QiitaItem]>> {
    
        api = API.QiitaItemsApi.init(query: query, page: page)
    
        return Single.create(subscribe: { [weak self] observer in

            guard let `self` = self else {
                observer(.error(OtherError.unknownError))
                return Disposables.create()
            }
            
            self.api.fetchQiitaItemsData().response { result in
                
                DispatchQueue.main.async {
                    switch result {
                    case .response(let data):
                        observer(.success(Result.success(data)))
                    case .error(let error):
                        observer(.error(error))
                    }
                }
            }
            return Disposables.create()
        })
    }   
}
```  
  
```QiitaItemsListPresenter.swift
final class QiitaItemsListPresenter {
    
    struct State: Equatable {
        let trigger: TriggerType
        let state: NetworkState
        let nextPage: Int
        fileprivate(set) var shouldLoadNext: Bool
        let contents: [QiitaItem]
        let error: Error?

        var canLoad: Bool {
            let loadingState: [NetworkState] = [.requesting, .reachedBottom]
            return !loadingState.contains(state)
        }
        
        static func ==(lhs: QiitaItemsListPresenter.State, rhs: QiitaItemsListPresenter.State) -> Bool {
            return lhs.trigger == rhs.trigger
                && lhs.state == rhs.state
                && lhs.nextPage == rhs.nextPage
                && lhs.shouldLoadNext == rhs.shouldLoadNext
                && lhs.contents == rhs.contents
        }
        
        static var initial = State(
            trigger: .refresh,
            state: .nothing,
            nextPage: 1,
            shouldLoadNext: true,
            contents: [],
            error: nil
        )
    }
    
    // MARK: - Public properties -
    
    let didItemsChange: Driver<[QiitaItem]>
    let didStateChange: Signal<State>
    let didErrorChange: Signal<Error>
    
    // MARK: - Private properties -
    
    private let didItemsChangeRelay: BehaviorRelay<[QiitaItem]>
    private let didStateChangeRelay: PublishRelay<State>
    private let didErrorChangeRelay: PublishRelay<Error>
    
    private weak var _view: QiitaItemsListViewInterface?
    private var _interactor: QiitaItemsListInteractorInterface
    private var _wireframe: QiitaItemsListWireframeInterface
    private let disposeBag = DisposeBag()
    
    private var state: State = State.initial {
        didSet {
            if state == oldValue {
                return
            }
            state.shouldLoadNext = (state.nextPage != oldValue.nextPage)
            
            if state.contents != oldValue.contents {
                didItemsChangeRelay.accept(state.contents)
            }
            
            if state.error != nil {
                didErrorChangeRelay.accept(state.error!)
            }
            didStateChangeRelay.accept(state)
        }
    }
    
    // MARK: - Lifecycle -
    
    init(wireframe: QiitaItemsListWireframeInterface, view: QiitaItemsListViewInterface, interactor: QiitaItemsListInteractorInterface) {
        _wireframe = wireframe
        _view = view
        _interactor = interactor
        
        let didStateChangeRelay = PublishRelay<State>()
        self.didStateChangeRelay = didStateChangeRelay
        self.didStateChange = didStateChangeRelay.asSignal()

        let didItemsChangeRelay = BehaviorRelay<[QiitaItem]>(value: [])
        self.didItemsChangeRelay = didItemsChangeRelay
        self.didItemsChange = didItemsChangeRelay.asDriver()
        
        let didErrorChangeRelay = PublishRelay<Error>()
        self.didErrorChangeRelay = didErrorChangeRelay
        self.didErrorChange = didErrorChangeRelay.asSignal()
    }
}

// MARK: - Extensions -

extension QiitaItemsListPresenter: QiitaItemsListPresenterInterface {
    
    func bind(input: Input) {
        
        input.searchBarText.drive(onNext: { [weak self] text in
            
            guard let `self` = self else { return }
            
            if !self.state.canLoad { return }
            
            self._view?.showLoading()
            
            self.fetchFirstPage(trigger: .searchTextChange, text: text)
        })
        .disposed(by: disposeBag)
        
        input.didReachBottom
            .withLatestFrom(input.searchBarText)
            .emit(onNext: { [weak self] text in
                
                guard let `self` = self else { return }
                
                if !self.state.canLoad || !self.state.shouldLoadNext { return }
                
                self.state = State(trigger: .loadMore,
                              state: .requesting,
                              nextPage: self.state.nextPage,
                              shouldLoadNext: false,
                              contents: self.state.contents,
                              error: nil
                )

                self._view?.showLoading()

                self.fetchList(text: text)
            })
            .disposed(by: disposeBag)
        
        input.didItemSelect.drive(onNext: { [weak self] item in
            
            guard let `self` = self else { return }
            
            self._wireframe.showDetail(item: item)
        })
        .disposed(by: disposeBag)
        
        input.searchBarCancelTap.emit(onNext: { [weak self] _ in
            guard let `self` = self else { return }
            
            self._view?.showLoading()
            
            self.fetchFirstPage(trigger: .searchTextChange)
        })
        .disposed(by: disposeBag)
        
        input.refreshDidChange
            .withLatestFrom(input.searchBarText)
            .emit(onNext: { [weak self] text in
            
            guard let `self` = self else { return }
            
            self.fetchFirstPage(trigger: .refresh, text: text)
        })
        .disposed(by: disposeBag)     
    }
    
    private func fetchFirstPage(trigger: TriggerType, text: String? = nil) {
        
        state = State(trigger: trigger,
                           state: .requesting,
                           nextPage: 1,
                           shouldLoadNext: true,
                           contents: [],
                           error: nil
        )
        
        fetchList(text: text ?? "")
    }
    
    private func fetchList(text: String) {
        
        _view?.showLoading()
        
        _interactor
            .fetchList(query: text, page: state.nextPage)
            .subscribeOn(MainScheduler.instance)
            .subscribe(
                onSuccess: { [weak self] result in
                    
                    guard let `self` = self else { return }
                    
                    switch result {
                    case let .success(items):
                        self.fetchedList(items: items)
                    case let .error(error):
                        self.fetchedListError(error: error)
                    }
                },
                onError: { error in
                    self.fetchedListError(error: error)
            })
            .disposed(by: disposeBag)
    }
    
    private func fetchedList(items: [QiitaItem]) {
        
        var newtWorkState: NetworkState
        if state.contents.count != 0 && items.count == 0 {
            newtWorkState = .reachedBottom
        } else {
            newtWorkState = .requestFinished
        }
        
        state = State(trigger: state.trigger,
                      state: newtWorkState,
                      nextPage: state.nextPage + 1,
                      shouldLoadNext: state.shouldLoadNext,
                      contents: state.contents + items,
                      error: nil
        )
    }
    
    private func fetchedListError(error: Error) {
        
        state = State(trigger: state.trigger,
                      state: .nothing,
                      nextPage: 1,
                      shouldLoadNext: true,
                      contents: [],
                      error: error
        )
    }
}
```  
  
  
```QiitaItemsListViewController.swift
final class QiitaItemsListViewController: UIViewController {
    
    @IBOutlet weak var noResultLabel: UILabel!
    @IBOutlet weak var tableView: UITableView!
    
    // MARK: - Public properties -
    
    var presenter: QiitaItemsListPresenterInterface!
    var didReachedBottom: Signal<Void>
    
    // MARK: - Private properties -
    
    private var didReachedBottomRelay: PublishRelay<Void>
    
    
    private var searchController = UISearchController(searchResultsController: nil)
    
    private let refreshControl = UIRefreshControl()
    private let disposeBag = DisposeBag()
    
    
    // MARK: - Lifecycle -
    
    required init?(coder aDecoder: NSCoder) {
        
        let didReachedBottomRelay = PublishRelay<Void>()
        self.didReachedBottomRelay = didReachedBottomRelay
        self.didReachedBottom = didReachedBottomRelay.asSignal()
        
        super.init(coder: aDecoder)
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
        _setupUI()
        _setTableView()
    }
    
    private func _setupUI() {
        navigationItem.title = "List"
        
        definesPresentationContext = true
        searchController.obscuresBackgroundDuringPresentation = false
        
        presenter.didStateChange.emit(onNext: { [weak self] state in
            
            guard let `self` = self else { return }
            
            if state.state == .requestFinished {
                self.hideLoading()
                self.refreshControl.endRefreshing()
            }
            
            if state.state == .requestFinished
                && state.trigger == .searchTextChange {
                
                DispatchQueue.main.async {
                    self.scrolltoTop()
                }
            }
        }).disposed(by: disposeBag)
        
        presenter.bind(input: (
            
            searchBarText: searchController.searchBar.rx.text
                .orEmpty
                .debounce(0.2, scheduler: MainScheduler.instance)
                .distinctUntilChanged()
                .asDriver(onErrorJustReturn: ""),
            
            didItemSelect: tableView.rx
                .modelSelected(QiitaItem.self)
                .asDriver(),
            
            didReachBottom: didReachedBottom,
            
            searchBarCancelTap: searchController.searchBar.rx
                .cancelButtonClicked.asSignal(),
            
            refreshDidChange: refreshControl.rx
                .controlEvent(.valueChanged)
                .asSignal()
        ))
    }
    
    private func _setTableView() {
        
        tableView.register(QiitaItemTableViewCell.self)
        tableView.refreshControl = self.refreshControl
        
        tableView.tableFooterView = UIView()
        
        tableView.rx.setDelegate(self)
            .disposed(by: disposeBag)
        
        if #available(iOS 11.0, *) {
            self.navigationItem.searchController = self.searchController
            self.navigationItem.hidesSearchBarWhenScrolling = true
        } else {
            tableView.tableHeaderView = self.searchController.searchBar
        }
        
        if #available(iOS 10.0, *) {
            tableView.prefetchDataSource = self
        }
        
        tableView.rx.reachBottom.asSignal(onErrorSignalWith: .empty())
            .emit(onNext: { [weak self] _ in
                
                guard let `self` = self else { return }

                self.didReachedBottomRelay.accept(())
            })
            .disposed(by: disposeBag)
        
        tableView.rx.willDisplayCell.asDriver()
            .drive(onNext: { [weak self] (cell, indexPath) in
                
                guard let `self` = self else { return }
                
                if !self.cellHeightList.keys.contains(indexPath) {
                    self.cellHeightList[indexPath] = cell.frame.size.height
                }
            }).disposed(by: disposeBag)
        
        tableView.rx.didEndDisplayingCell.asDriver()
            .drive(onNext: { [weak self] cell, indexPath in
                
                guard let `self` = self else { return }
                
                guard let imageLoadOperation = self.imageLoadOperations[indexPath] else { return }
                
                imageLoadOperation.cancel()
                
                self.imageLoadOperations.removeValue(forKey: indexPath)
            })
            .disposed(by: disposeBag)
        
        presenter.didItemsChange.drive(tableView.rx.items) { [weak self] (tableView, row, element) in
            
            guard let `self` = self else { return UITableViewCell() }
            
            let indexPath = IndexPath(row: row, section: 0)
            
            return self.configureCell(indexPath: indexPath, item: element)
            }
            .disposed(by: disposeBag)
    }    
}
```  
  
## 良かった点  
  
・「これが変わると何かが起き、その結果が返ってきたらこれをする」という一連の流れを一箇所で見ることができる。  
  
・同じ処理に対して複数の処理が必要なとき(処理結果をログに書き込むなど)に同じような処理を書かなくてよくなる。  
  
## 課題  
  
・RxSwift(Reactiveプログラミング)の書き方に慣れていないためもっとサンプルを作成して書き方を身につける。  
  
・Operatorに対する理解が乏しいためどういう時に何を使うのかをもっと学ぶ。  
  
<br>  
[Try! Swift2017 NYC](https://academy.realm.io/posts/try-swift-nyc-2017-krunoslav-zaher-modern-rxswift-architectures/)で紹介されていた[Rxfeedback](https://github.com/NoTests/RxFeedback.swift)も試してみたので、またの機会に投稿してみたいと思います。  
  
また[ReactiveSwift](https://github.com/ReactiveCocoa/ReactiveSwift)でも試してみたいと思います。  
  
長文失礼致しました。  
