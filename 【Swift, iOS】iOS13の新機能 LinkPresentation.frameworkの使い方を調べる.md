iOS13でLinkPresentationという新機能が追加されました。  
これによってURLリンク先の情報を  
より表現豊かに表示することができるようになるようです。  
  
今回はその新機能について  
AppleのWWDC2019の動画と検証結果などから見ていきたいと思います。  
  
Embedding and Sharing Visually Rich Links  
https://developer.apple.com/videos/play/wwdc2019/262  
  
※   
あまり情報がなく  
検証した結果から記載した部分もありますので  
間違っている部分やもっと良い方法ご存知の方いらっしゃいましたらぜひ教えてください🙇🏻‍♂️  
  
# LinkPresentation.frameworkとは？  
  
iOS13で新しく追加された  
リンクのプレビューを画像や埋め込み動画、音楽再生と合わせて  
リッチに一貫した方法で表示できるようにした  
フレームワークです。  
  
iOS10とmacOS Sierraから  
Appleのメッセージアプリなどでは先行して  
このフレームワークに含まれている機能を利用していたようです。  
  
<img width="1322" alt="スクリーンショット 2020-03-21 8.50.12.png" src="/image/ae649202-4997-a1e9-0869-aeeb6ff85b5c.png">  
  
  
https://developer.apple.com/documentation/linkpresentation  
  
# 主なクラス  
  
非常にシンプルで主に登場するクラスは3です。  
  
- LPMetadataProvider  
- LPLinkMetadata  
- LPLinkView  
  
## LPMetadataProvider  
URLのメタ情報を取得します。  
https://developer.apple.com/documentation/linkpresentation/lpmetadataprovider  
  
※ メタ情報とは？  
  
HTMLタグに含まれるタイトルやアイコン、画像、動画などの情報を読み取ります。  
  
特に  
OpenGraphというプロトコルを使用した  
`<meta og:XXX>`の情報を優先して読み取ります。  
  
例えば下記のようなものです。  
  
```html

<html prefix="og: http://ogp.me/ns#">
<head>
<title>The Rock (1996)</title>
<meta property="og:title" content="The Rock" />
<meta property="og:type" content="video.movie" />
<meta property="og:url" content="http://www.imdb.com/title/tt0117500/" />
<meta property="og:image" content="http://ia.media-imdb.com/images/rock.jpg" />
...
</head>
...
</html>
```  
  
詳細は下記をご参照ください。  
OpenGraph  
https://ogp.me/  
  
  
## LPLinkMetadata  
URLのメタ情報を保持するクラスです。  
https://developer.apple.com/documentation/linkpresentation/lplinkmetadata  
  
## LPLinkView  
URLのメタ情報をリッチに表示するUIViewのサブクラスです。  
https://developer.apple.com/documentation/linkpresentation/lplinkview  
  
  
# 使い方  
  
すごいシンプルです。  
  
1. `LPMetadataProvider`の`startFetchingMetadata`でURLのリンク先から`LPLinkMetadata`を取得する  
2. `LPLinkMetadata`を`LPLinkView`に設定する  
  
  
  
## SwiftUIでの実装  
  
### UIViewRepresentableに適合したクラスの生成  
  
`LPLinkView`に対応する  
`UIViewRepresentable`に適合したクラスを生成します。  
  
```swift

import SwiftUI
import LinkPresentation

struct LinkPresentationView: UIViewRepresentable {
    typealias UIViewType = LPLinkView

    func makeUIView(context: UIViewRepresentableContext<LinkPresentationView>) -> UIViewType {
    }

    func updateUIView(_ uiView: UIViewType, context: UIViewRepresentableContext<LinkPresentationView>) {
    }
}
```  
  
### メタ情報の取得  
  
`LPMetadataProvider`から`LPLinkMetadata`を取得します。  
取得するURLが必要なので初期化時に引数で受け取るように変数を宣言します。  
  
```swift

struct LinkPresentationView: UIViewRepresentable {
    let url: URL

    private func fetchMetadata(for url: URL, completion: @escaping (Result<LPLinkMetadata, Error>) -> Void) {
        let provider = LPMetadataProvider()

        provider.startFetchingMetadata(for: url) { metadata, error in
            if let error = error {
                completion(.failure(error))
            } else if let metadata = metadata {
                completion(.success(metadata))
            } else {
                completion(.failure(LPError(.unknown)))
            }
        }
    }
}
```  
  
エラーが発生した場合は`LPError`が返ってきます。  
https://developer.apple.com/documentation/linkpresentation/lperror  
  
ネットワークに繋がっていなかったり  
接続が遅すぎてタイムアウトになったり  
リクエストがキャンセルされた場合に生じます。  
  
### LPLinkViewの生成  
  
次に  
`makeUIView`の中で`LPLinkView`を生成します。  
  
  
```swift

struct LinkPresentationView: UIViewRepresentable {
    var url: URL

    func makeUIView(context: UIViewRepresentableContext<LinkPresentationView>) -> UIViewType {
        let view = LPLinkView(url: url) // ※
        self.fetchMetadata(for: url) { result in
            switch result {
            case .success(let metadata):
                DispatchQueue.main.async {
                    self.update(view: view, with: metadata)
                }
            case .failure:
                let metadata = LPLinkMetadata()
                metadata.title = "Error"
                let url = URL(fileURLWithPath: Bundle.main.path(forResource: "error", ofType: "png")!)
                metadata.iconProvider = NSItemProvider(contentsOf: url)
                self.update(view: view, with: metadata)
            }
        }
        return view
    }

    private func update(view: UIViewType, with metadata: LPLinkMetadata) {
        view.metadata = metadata
        view.sizeToFit()
    }
}
```  
  
※  
ここは疑問が残っているのですが  
ここでurlを引数に`LPLinkView`を初期化しないと  
画面に何も表示されませんでした。  
  
おそらく内部で取得した`LPLinkMetadata`のURLと  
`LPLinkView`のURLを比較しているんじゃないかと思っているのですが  
もし何かご存知の方いらっしゃいましたら  
教えていただけると嬉しいです🙇🏻‍♂️  
  
`update`メソッドの中で  
  
```swift

private func update(view: UIViewType, with metadata: LPLinkMetadata) {
    ....
    view.sizeToFit()
}
```  
  
としていますが  
これは  
`LPLinkView`自体もintrinsic sizeを持っているものの  
`sizeToFit`を使用することで  
現在のレイアウトに最適なサイズで表示されるようにするためです。  
  
エラーが起きた時は  
その場で`LPLinkMetadata`を生成して表示することもできます。  
  
```swift

case .failure:
    let metadata = LPLinkMetadata()
    metadata.title = "Error"
    let url = URL(fileURLWithPath: Bundle.main.path(forResource: "error", ofType: "png")!)
    metadata.iconProvider = NSItemProvider(contentsOf: url)
    self.update(view: view, with: metadata)
```  
  
  
実際に表示してみます。  
  
```swift

struct ContentView: View {
    var urls: [URL] = [
        URL(string: "https://www.apple.com/mac")!,
        URL(string: "https://www.apple.com/ipad")!,
        URL(string: "https://youtu.be/V85CQzsyvj4")!,
        URL(string: "https://twitter.com/yuukikikuchi/status/1240946299467259905")!,
    ]
    var body: some View {
        List(self.urls, id: \.self) { url in
            LinkPresentationView(url: url)
        }.onAppear {
            UITableView.appearance().separatorStyle = .none
        }
    }
}

```  
  
下記の様に表示されます。  
  
![Simulator Screen Shot - iPhone 11 Pro Max - 2020-03-21 at 11.22.26.png](/image/292aa559-8dde-6dd1-84b4-2bc4b99350ae.png)  
  
適切なサイズで表示がされていません。  
  
これは`LPLinkView`の初期化時のサイズのままになっており  
取得した画像のサイズなどが反映されていないためです。  
  
そこで`LPLinkMetadata`の取得が完了した時点で  
サイズの再計算を行うように親のViewに伝達するようにします。  
  
```swift

struct LinkPresentationView: UIViewRepresentable {
    ...

    @Binding var redraw: Bool

    private func update(view: UIViewType, with metadata: LPLinkMetadata) {
        ...

        redraw.toggle()
    }
}

struct ContentView: View {
    ....

    @State var redraw = false

    var body: some View {
        List(self.urls, id: \.self) { url in
            LinkPresentationView(url: url, redraw: self.$redraw)
        }.onAppear {
            UITableView.appearance().separatorStyle = .none
        }
    }
}
```  
  
こうするとこんな形で表示されます。  
またYoutubeの動画はクリックすると再生することができたり  
Twitterの表示も自動で設定してくれます。  
  
(表示時のアニメーションをどうにかしたいですが。。。)  
  
<blockquote class="twitter-tweet"><p lang="und" dir="ltr"><a href="https://t.co/JUT6h3U4aE">pic.twitter.com/JUT6h3U4aE</a></p>&mdash; shiz(しず) (@stzn3) <a href="https://twitter.com/stzn3/status/1241197575358709760?ref_src=twsrc%5Etfw">March 21, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>  
  
  
### メタ情報をキャッシュする  
  
URLから毎回メタ情報を取得してくるのは  
ユーザにとっては通信料がかかってしまいますし  
アプリとして同じリンク先から毎回同じ情報を取得するためには  
パフォーマンスコストがかかるため  
キャッシュをするべきだと  
Appleの動画でも言っていました。  
  
そこでローカルにキャッシュするためのクラスを用意します。  
  
`LPLinkMetadata`はNSSecureCodingに適合しており  
信頼性の高い形でシリアライズ可能になっています。  
  
https://developer.apple.com/documentation/foundation/nssecurecoding  
  
※  
今回は実装を簡単にするために  
シングルトンのUserDefaults.standardやsharedを使用しています。  
  
  
```swift

final class MetaCache {
    static let shared = MetaCache()
    private init(){}

    private let storage = UserDefaults.standard

    private let key = "Metadata"

    func store(_ metadata: LPLinkMetadata) {
        do {
            let data = try NSKeyedArchiver.archivedData(withRootObject: metadata, requiringSecureCoding: true)
            var metadatas: [String: Data] = storage.dictionary(forKey: key) as? [String: Data] ?? [:]
            metadatas[metadata.originalURL!.absoluteString] = data
            storage.set(metadatas, forKey: key)
        }
        catch {
            print("Failed storing metadata with error \(error as NSError)")
        }
    }

    func metadata(for url: URL) -> LPLinkMetadata? {
        guard let metadatas = storage.dictionary(forKey: key) as? [String: Data] else {
            return nil
        }

        guard let data = metadatas[url.absoluteString] else {
            return nil
        }

        do {
            return try NSKeyedUnarchiver.unarchivedObject(ofClass: LPLinkMetadata.self, from: data)
        } catch {
            print("Failed to unarchive metadata with error \(error)")
            return nil
        }
    }
}
```  
  
最後にこれを使用します。  
  
```swift

    func makeUIView(context: UIViewRepresentableContext<LinkPresentationView>) -> UIViewType {
        let view = LPLinkView(url: url)
        if let cachedData = MetaCache.shared.metadata(for: url) {
            update(view: view, with: cachedData)
        } else {
            self.fetchMetadata(for: url) { result in
                switch result {
                case .success(let metadata):
                    MetaCache.shared.store(metadata)
                case .failure:
                    ...
                }
            }
        }
        return view
    }
```  
  
  
### 注意点  
  
ドキュメントにも記載がありますが  
`startFetchingMetadata(for:completionHandler:)`のcompletionHandlerは  
バックグラウンドで実行されるため  
UI関連の処理を行う場合はメインキューで実行するようにします。  
  
```
The completion handler executes on a background queue. 
Dispatch any necessary UI updates back to the main queue. 
```  
  
また、LPMetadataProviderは一回のリクエストにしか使用できないため  
例えば  
  
```swift

struct LinkPresentationView: UIViewRepresentable {
    let provider = LPMetadataProvider()

    func updateUIView(_ uiView: UIViewType, context: UIViewRepresentableContext<LinkPresentationView>) {
        provider.startFetchingMetadata(for: url) { metadata, error in
        }
    }
}

```  
  
などと実装してみると  
ListやForEachを使用した時に  
  
```
'Trying to start fetching on an LPMetadataProvider that has already started. 
LPMetadataProvider is a one-shot object.'
```  
  
といったエラーでクラッシュします。  
  
  
<details><summary>最終的なコード</summary><div>  
  
```swift

import SwiftUI
import LinkPresentation

struct LinkPresentationView: UIViewRepresentable {
    typealias UIViewType = LPLinkView

    let url: URL
    @Binding var redraw: Bool

    func makeUIView(context: UIViewRepresentableContext<LinkPresentationView>) -> UIViewType {
        let view = LPLinkView(url: url)

        // 画像を取得するまでの表示されないように設定しています
        view.isHidden = true
        if let cachedData = MetaCache.shared.metadata(for: url) {
            update(view: view, with: cachedData)
        } else {
            self.fetchMetadata(for: url) { result in
                switch result {
                case .success(let metadata):
                    MetaCache.shared.store(metadata)
                    DispatchQueue.main.async {
                        self.update(view: view, with: metadata)
                    }
                case .failure:
                    let metadata = LPLinkMetadata()
                    metadata.title = "Error"
                    let url = URL(fileURLWithPath: Bundle.main.path(forResource: "error", ofType: "png")!)
                    metadata.iconProvider = NSItemProvider(contentsOf: url)
                    self.update(view: view, with: metadata)
                }
            }
        }
        return view
    }

    func updateUIView(_ uiView: UIViewType, context: UIViewRepresentableContext<LinkPresentationView>) {
    }

    private func fetchMetadata(for url: URL, completion: @escaping (Result<LPLinkMetadata, Error>) -> Void) {
        let provider = LPMetadataProvider()

        provider.startFetchingMetadata(for: url) { metadata, error in
            if let error = error {
                completion(.failure(error))
            } else if let metadata = metadata {
                completion(.success(metadata))
            } else {
                completion(.failure(LPError(.unknown)))
            }
        }
    }

    private func update(view: UIViewType, with metadata: LPLinkMetadata) {
        view.metadata = metadata
        view.sizeToFit()
        redraw.toggle()
        view.isHidden = false
    }
}

struct ContentView: View {
    var urls: [URL] = [
        URL(string: "https://www.apple.com/mac")!,
        URL(string: "https://www.apple.com/ipad")!,
        URL(string: "https://youtu.be/V85CQzsyvj4")!,
        URL(string: "https://twitter.com/yuukikikuchi/status/1240946299467259905")!,
    ]

    @State var redraw = false

    var body: some View {
        List(self.urls, id: \.self) { url in
            LinkPresentationView(url: url, redraw: self.$redraw)
        }.onAppear {
            UITableView.appearance().separatorStyle = .none
        }
    }
}

struct LinkPresentationView_Previews: PreviewProvider {
    static var previews: some View {
        ContentView()
    }
}

```  
</div></details>  
  
## UIViewでの実装  
  
UITableViewでも実装をしてみました。  
  
やっていることはだいたい同じなので  
多くの部分は割愛しますが  
  
セルの実装でいくつか疑問に残っている部分があるので  
記載します。  
  
まず`LPLinkView`は毎回生成しないと  
画面に表示されないため  
毎回`addSubView`をする必要がありました。  
そのため`prepareForReuse`でsubviewsを一旦クリアする必要がありました。  
  
またセルサイズの再計算を行うために  
メタ情報取得後にクロージャを実行しています。  
  
```swift

final class Cell: UITableViewCell {
    static let identifier = "Cell"

    // ViewControllerにセルサイズを再計算をさせるためにLPLinkMetadata取得したことを伝える
    var onUpdate: (() -> Void)?

    ...

    // 毎回subViewをクリアする必要がある
    override func prepareForReuse() {
        super.prepareForReuse()
        contentView.subviews.forEach { $0.removeFromSuperview() }
    }

    func configure(with url: URL) {
        setLPLinkView(for: url)
    }


    ...

    private func update(_ view: LPLinkView, with metadata: LPLinkMetadata) {
        view.metadata = metadata
        // 毎回addSubViewする必要がある
        addSubView(linkView: view)
        view.sizeToFit()

        // ViewControllerにLPLinkMetadataを取得したことを伝える
        onUpdate?()
    }

    private func addSubView(linkView: LPLinkView) {
        contentView.addSubview(linkView)
        linkView.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            linkView.leadingAnchor.constraint(equalTo: contentView.leadingAnchor, constant: 12),
            linkView.topAnchor.constraint(equalTo: contentView.topAnchor, constant: 12),
            linkView.trailingAnchor.constraint(equalTo: contentView.trailingAnchor, constant: -12),
            linkView.bottomAnchor.constraint(equalTo: contentView.bottomAnchor, constant: -12),
        ])
    }
}
```  
  
下記のように動きます。  
  
(表示時のアニメーションをどうにかしたいですが。。。)  
  
  
<blockquote class="twitter-tweet"><p lang="und" dir="ltr"><a href="https://t.co/EEhtCIHt6u">pic.twitter.com/EEhtCIHt6u</a></p>&mdash; shiz(しず) (@stzn3) <a href="https://twitter.com/stzn3/status/1241212431319191553?ref_src=twsrc%5Etfw">March 21, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>  
  
  
<details><summary>最終的なコード</summary><div>  
  
```swift

import UIKit
import LinkPresentation

class ViewController: UIViewController {
    private var urls: [URL] = [
        URL(string: "https://www.apple.com/mac")!,
        URL(string: "https://www.apple.com/ipad")!,
        URL(string: "https://youtu.be/V85CQzsyvj4")!,
        URL(string: "https://twitter.com/yuukikikuchi/status/1240946299467259905")!,
    ]

    lazy var tableView: UITableView = UITableView()

    private var loadedIndexPaths: Set<IndexPath> = []

    override func viewDidLoad() {
        super.viewDidLoad()
        configureTableView()
    }

    private func configureTableView() {
        view.addSubview(tableView)
        tableView.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            tableView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            tableView.topAnchor.constraint(equalTo: view.topAnchor),
            tableView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            tableView.bottomAnchor.constraint(equalTo: view.bottomAnchor),
        ])

        tableView.register(Cell.self, forCellReuseIdentifier: Cell.identifier)
        tableView.dataSource = self
        tableView.separatorStyle = .none
    }
}

extension ViewController: UITableViewDataSource {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return urls.count
    }

    func numberOfSections(in tableView: UITableView) -> Int {
        return 1
    }

    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let url = urls[indexPath.row]
        let cell = tableView.dequeueReusableCell(withIdentifier: Cell.identifier, for: indexPath) as! Cell
        cell.configure(with: url)

        cell.onUpdate = { [weak self] in
            guard let self = self else {
                return
            }

            // LPLinkMetadataが取得されたらreloadする
            if !self.loadedIndexPaths.contains(indexPath) {
                self.loadedIndexPaths.insert(indexPath)
                self.tableView.reloadRows(at: [indexPath], with: .none)
            }
        }
        return cell
    }
}

final class Cell: UITableViewCell {
    static let identifier = "Cell"

    var onUpdate: (() -> Void)?

    override init(style: UITableViewCell.CellStyle, reuseIdentifier: String?) {
        super.init(style: style, reuseIdentifier: reuseIdentifier)
    }

    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }

    override func prepareForReuse() {
        super.prepareForReuse()
        contentView.subviews.forEach { $0.removeFromSuperview() }
    }

    func configure(with url: URL) {
        setLPLinkView(for: url)
    }

    private func setLPLinkView(for url: URL) {
        let linkView = LPLinkView(url: url)
        if let cachedData = MetaCache.shared.metadata(for: url) {
            update(linkView, with: cachedData)
            return
        }
        fetchMetadata(for: url) { result in
            switch result {
            case .success(let metadata):
                MetaCache.shared.store(metadata)
                DispatchQueue.main.async {
                    self.update(linkView, with: metadata)
                }
            case .failure:
                let metadata = LPLinkMetadata()
                metadata.title = "Error"
                let url = URL(fileURLWithPath: Bundle.main.path(forResource: "error", ofType: "png")!)
                metadata.iconProvider = NSItemProvider(contentsOf: url)
                self.update(linkView, with: metadata)
            }
        }
    }

    private func fetchMetadata(for url: URL, completion: @escaping (Result<LPLinkMetadata, Error>) -> Void) {
        let provider = LPMetadataProvider()

        provider.startFetchingMetadata(for: url) { metadata, error in
            if let error = error {
                completion(.failure(error))
            } else if let metadata = metadata {
                completion(.success(metadata))
            } else {
                completion(.failure(LPError(.unknown)))
            }
        }
    }

    private func update(_ view: LPLinkView, with metadata: LPLinkMetadata) {
        view.metadata = metadata
        addSubView(linkView: view)
        view.sizeToFit()

        onUpdate?()
    }

    private func addSubView(linkView: LPLinkView) {
        contentView.addSubview(linkView)
        linkView.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            linkView.leadingAnchor.constraint(equalTo: contentView.leadingAnchor, constant: 12),
            linkView.topAnchor.constraint(equalTo: contentView.topAnchor, constant: 12),
            linkView.trailingAnchor.constraint(equalTo: contentView.trailingAnchor, constant: -12),
            linkView.bottomAnchor.constraint(equalTo: contentView.bottomAnchor, constant: -12),
        ])
    }
}
```  
</div></details>  
  
  
### 2020/3/29 更新  
  
アニメーションですが  
UITableViewの更新方法を下記に変更したら少し改善?しました  
  
```swift

self.tableView.reloadRows(at: [indexPath], with: .none)

↓

self.tableView.beginUpdates()
self.tableView.endUpdates()
```  
  
<blockquote class="twitter-tweet"><p lang="und" dir="ltr"><a href="https://t.co/uyo3WuCOW4">pic.twitter.com/uyo3WuCOW4</a></p>&mdash; shiz(しず) (@stzn3) <a href="https://twitter.com/stzn3/status/1244038812516282369?ref_src=twsrc%5Etfw">March 28, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>  
  
  
# ShareSheetにLPLinkMetadataを利用する  
  
`LPLinkMetadata`はShareSheetでも  
`UIActivityItemSource`の  
`activityViewControllerLinkMetadata(_:)`から  
利用することができ  
プレビュー情報をリンク先から取得して表示できるようになりました。  
  
https://developer.apple.com/documentation/uikit/uiactivityitemsource/3144571-activityviewcontrollerlinkmetada  
  
## UIActivityItemSourceに適合したクラスを作成  
  
ShareSheetに表示するアイテムを表すクラスを定義します  
  
```swift

import UIKit
import LinkPresentation

final class ShareActivityItemSource: NSObject, UIActivityItemSource {
    private let linkMetadata: LPLinkMetadata

    init(_ url: URL) {
        linkMetadata = LPLinkMetadata()
        super.init()
        setPlaceholder(for: url)

        if let cachedData = MetaCache.shared.metadata(for: url) {
            setMetadata(cachedData)
            return
        }

        let metadataProvider = LPMetadataProvider()
        metadataProvider.startFetchingMetadata(for: url) { [weak self] metadata, error in
            guard let self = self, let metadata = metadata else {
                return
            }
            self.setMetadata(metadata)
        }
    }

    private func setPlaceholder(for url: URL) {
        linkMetadata.title = "loading..."
        linkMetadata.originalURL = url
        let loadingImageURL = URL(fileURLWithPath: Bundle.main.path(forResource: "loading", ofType: "png")!)
        linkMetadata.iconProvider = NSItemProvider(contentsOf: loadingImageURL)
    }

    private func setMetadata(_ metadata: LPLinkMetadata) {
        linkMetadata.title = metadata.title
        linkMetadata.url = metadata.url
        linkMetadata.originalURL = metadata.originalURL
        linkMetadata.iconProvider = metadata.iconProvider
        linkMetadata.imageProvider = metadata.imageProvider
    }

    func activityViewControllerLinkMetadata(_ activityViewController: UIActivityViewController) -> LPLinkMetadata? {
        linkMetadata
    }

    func activityViewControllerPlaceholderItem(_ activityViewController: UIActivityViewController) -> Any {
        linkMetadata as Any
    }

    func activityViewController(_ activityViewController: UIActivityViewController, itemForActivityType activityType: UIActivity.ActivityType?) -> Any? {
        linkMetadata
    }
}
```  
  
ここでは`LPLinkMetadata`に事前のデータを設定しておき  
データが取得できたらメタ情報を入れ替えるようにします。  
  
  
## SwiftUIでの実装方法  
  
UIActivityViewControllerの機能を有するViewを用意します。  
  
```swift

struct ShareSheet: UIViewControllerRepresentable {
    typealias Callback = (_ activityType: UIActivity.ActivityType?, _ completed: Bool, _ returnedItems: [Any]?, _ error: Error?) -> Void

    let activityItems: [Any]
    let applicationActivities: [UIActivity]? = nil
    let excludedActivityTypes: [UIActivity.ActivityType]? = nil
    let callback: Callback? = nil

    func makeUIViewController(context: Context) -> UIActivityViewController {
        let controller = UIActivityViewController(
            activityItems: activityItems,
            applicationActivities: applicationActivities)
        controller.excludedActivityTypes = excludedActivityTypes
        controller.completionWithItemsHandler = callback
        return controller
    }

    func updateUIViewController(_ uiViewController: UIActivityViewController, context: Context) {
    }
}
```  
  
ナビゲーションバーのボタンを押すと  
`ShareSheet`を表示するようにします。  
  
```swift

struct ContentView: View {
    var urls: [URL] = [
        URL(string: "https://www.apple.com/mac")!,
        URL(string: "https://www.apple.com/ipad")!,
        URL(string: "https://youtu.be/V85CQzsyvj4")!,
        URL(string: "https://twitter.com/yuukikikuchi/status/1240946299467259905")!,
    ]

    @State var redraw = false
    @State var showShareSheet = false

    var body: some View {
        NavigationView {
            List(self.urls, id: \.self) { url in
                LinkPresentationView(url: url, redraw: self.$redraw)
            }.onAppear {
                UITableView.appearance().separatorStyle = .none
            }
        }.navigationBarItems(trailing:
            Button(action: {
                self.showShareSheet = true
            }) {
                Text("Share").bold()
            }
        ).sheet(isPresented: $showShareSheet) {
            ShareSheet(activityItems: [ShareActivityItemSource(self.urls[Int.random(in: 0..<4)])])
        }
    }
}
```  
  
`ShareSheet`の`activityItems`に  
`ShareActivityItemSource`を渡します。  
  
すると下記のような動きをします。  
  
<blockquote class="twitter-tweet"><p lang="und" dir="ltr"><a href="https://t.co/pGIKsys1da">pic.twitter.com/pGIKsys1da</a></p>&mdash; shiz(しず) (@stzn3) <a href="https://twitter.com/stzn3/status/1241276948996747264?ref_src=twsrc%5Etfw">March 21, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>  
  
1回目のShareSheetの表示では  
事前に準備した情報がまず表示され  
取得後に入れ替わっています。  
  
2回目はメタ情報をすでに取得しているので  
最初からメタ情報が表示されています。  
  
  
## UIKitでの実装方法  
  
同じ様にナビゲーションバーのボタンを押すと  
ShareSheetが表示されるようにします。  
  
  
```swift

class ViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()

        ...
        navigationItem.rightBarButtonItem = UIBarButtonItem(title: "Share", style: .plain, target: self, action: #selector(shareTapped))
    }

    @objc func shareTapped() {
        showShareSheet(url: urls[Int.random(in: 0..<4)])
    }
}

extension ViewController {
    func showShareSheet(url: URL) {
        let item = ShareActivityItemSource(url)
        let activity = UIActivityViewController(activityItems: [item], applicationActivities: nil)
        present(activity, animated: true)
    }
}
```  
下記のように動きます。  
  
<blockquote class="twitter-tweet"><p lang="und" dir="ltr"><a href="https://t.co/9bQisNJ043">pic.twitter.com/9bQisNJ043</a></p>&mdash; shiz(しず) (@stzn3) <a href="https://twitter.com/stzn3/status/1241276720210046981?ref_src=twsrc%5Etfw">March 21, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>  
  
  
# メタ情報のベストプラクティス  
  
以上のように使い方を見てきましたが  
Appleがメタ情報にどういう情報を載せるべきかを  
紹介している動画があるので  
次に見ていきます。  
  
https://developer.apple.com/videos/play/tech-talks/205  
  
メタ情報にはあらゆる情報を含めることができますが  
その中でもAppleの公式動画の中でベストプラクティスを紹介しています。  
  
## タイトルについてのベストプラクティス  
  
- タイトルからリンク先の内容がわかるようにする  
- `<head>`の`<title>`からタイトルを読み取ることもできるがサイト名がURLのドメインなどと重複して表示されるのを避けるために`<meta og:title="">`を設定する  
- JavaScriptは動かないので動的なタグの生成をしないようにする  
  
## アイコンについてのベストプラクティス  
  
`<link rel="icon">`の情報を読み取るが  
`<meta og:image="">`を指定するとアイコンが表示されなくなるので  
アイコンを表示したい場合は`<meta og:image="">`を指定しないようにする  
  
## 画像についてのベストプラクティス  
  
- 興味を引かせるような特定のページの内容を表す画像のみ`og:image`を設定する  
- 画像が取得できなかった場合の対処としてアイコンを設定する  
- テキストは含めない方が良い(全サイズのデバイスで表示する際にサイズなどがスケールしない)  
  
## 動画についてのベストプラクティス  
  
- アイコン、画像、動画合わせて10MBまでなのでサイズに気を付ける  
- 自動再生をするためには直接参照したビデオファイルを使用する(不可能な場合、Youtubeの埋め込み動画のURLを指定すればユーザがタップして再生ができる。Youtube以外のサービスでは不可能)  
- HTMLやプラグインが必要な埋め込み動画のサポートはしていない  
  
# まとめ  
  
LinkPresentation.frameworkについて見てみました。  
  
使い方は非常にシンプルで便利ですが  
表示時のアニメーションがいまいちであったり  
まだ使い方が完全に把握できていないため  
今後も試してみて理解を深めていく必要があります。  
  
ここに記載したことはあくまで検証結果に基づいていますので  
もし間違いなどございましたらぜひご指摘ください🙇🏻‍♂️  
  
# 参照先  
  
https://developer.apple.com/videos/play/wwdc2019/262/  
https://developer.apple.com/videos/play/tech-talks/205  
https://medium.com/better-programming/ios-13-rich-link-previews-with-swiftui-e61668fa2c69  
https://augmentedcode.io/2019/09/15/loading-url-previews-using-linkpresentation-framework-in-swift/  
https://nshipster.com/ios-13/  
https://www.swiftjectivec.com/linkpresentation-introduction/  
https://qiita.com/ezura/items/6036c6e100599b601482  
https://forums.developer.apple.com/thread/123951  
https://github.com/SDWebImage/SDWebImageLinkPlugin  
