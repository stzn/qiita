WWDCを前に過去のiOSバージョンの見直しをしています。  
  
  
## なぜiOS10?  
  
iPhoneの工場出荷時の初期バージョンが最新の２つ前に設定されている(はず?)なので、方針として２つ前のバージョンからサポートをするというようにしています。  
(社内事情で端末のバージョンを上げることができない会社もあるようですので)  
  
そのため今回のiOS12へのバージョンアップによってiOS10で使用できる機能が使えるようになり、改めて調べたことを定期的に記録しておくことにしました。  
  
今回はUITableView、UICollectionViewの機能です。  
iOS9ではUITableViewやUICollectionViewで高速にスクロールを行うと画面がちらつくということが多々ありました。  
これはセルの表示直前に処理が集中することで、セルの再生成に時間がかかり、  
フレームが落ちてしまっているためです。  
  
iOS10ではこれを改善するために  
セルのライフサイクルイベントの改善と新しいAPIの提供が行われました。  
  
  
# セルのライフサイクルイベントの改善  
  
大きな変更は２つあります。  
・prepareForReuseとcellForItemAtIndexPathが呼ばれるタイミングが早くなった  
・didEndDisplayingCellが呼ばれるタイミングが遅くなった  
  
## セルのライフサイクルに関わるメソッド  
  
主に４つのメソッドが関わってきます。  
  
### prepareForReuse  
  
表示されなくなったセルを再利用する時に呼ばれる。  
主にセルの初期化を行う。  
  
### cellForItemAtIndexPath  
  
セルの生成や設定などセルを表示させるためのほとんどの処理を行う。  
  
### willDisplayCell  
  
セルが画面に表示される直前に表示される。  
  
### didEndDisplayingCell  
  
セルが表示がされなくなった際に呼ばれる。  
  
## iOS9までの問題  
  
上記のメソッドがセルが画面表示される直前に呼ばれていたことにより、  
セルの生成に時間がかかってしまうと処理が間に合わなくなり  
表示までに一時的に画面が固まったままのように見えてしまっていました。  
  
逆に画面から隠れた直後にセルが破棄されていたため、  
スクロールの向きを変えて高速にスクロールすると  
同様にセルの再生成が必要になるため画面が固まったように見えていました。  
  
  
## prepareForReuseとcellForItemAtIndexPathが呼ばれるタイミングが早くなった  
  
まずprepareForReuseとcellForItemAtIndexPathのタイミングが表示直前からもう少し前に行われるようになり、画面を表示するまでにセルの生成が完了されるようになりました。  
  
  
## didEndDisplayingCellが呼ばれるタイミングが遅くなった  
  
こうすることで、急にスクロールの向きが変更された場合でも  
セルが残っているのでそのまま画面に表示されるようになりました。  
  
## 使用するメモリ量は増えている  
  
パフォーマンスは改善しているものの、単純に保持するセルの数が増えているのでメモリの使用量は増えています。  
  
  
# Pre-Fetching API  
  
[https://developer.apple.com/documentation/uikit/uicollectionviewdatasourceprefetching/prefetching_collection_view_data](https://developer.apple.com/documentation/uikit/uicollectionviewdatasourceprefetching/prefetching_collection_view_data)  
  
上記の改善に加えて、さらにスムーズなスクロールを実現するために  
Pre-Fetching APIが導入されました。  
  
これはセルの生成をしているcellForItemAtIndexPathよりもさらに前に呼ばれ、  
APIやDBからのデータの取得や画像の読み込みなど、  
重い処理を事前にやっておくことができます。  
  
こうすることでwillDisplayの負担を分割することができます。  
  
さらに、これらはメインスレッド外で実行されるためUIに影響を与えません。  
  
また、スクロールの方向が変化した際に呼ばれるAPIも存在し、  
読み込む必要のなくなったセルに対するデータの取得をキャンセルをすることも可能になっています。  
  
  
## UICollectionViewを実装して動きを見てみる  
  
下記のようにシンプルに画像を表示するアプリです。  
  
![Simulator Screen Shot - iPhone X - 2018-06-03 at 09.51.05.png](/image/6e95afea-9c17-daf5-6186-dda0ba323155.png)  
  
それぞれの場合で比較してみます。  
  
## iOS10 Pre-Fetching APIなしの場合  
  
![notprefetch.mov.gif](/image/7e13581c-c2a1-b9a0-97d1-ddc545a73a5f.gif)  
  
  
## iOS10 Pre-Fetching APIありの場合  
  
![prefetch.mov.gif](/image/5b784857-8b97-3a34-715e-3e65c40d32b7.gif)  
  
  
若干ではありますが、Prefetchありの場合の方が先に画像の表示が早くなっているように見えます。  
  
  
簡単に実装の中身を示します。  
  
モデル  
  
```swift
struct FavoritePhoto {
    let id: Int
    let name: String
}
```  
  
画像を非同期で読み込みは下記のOperation中で行います。  
読み込み時間を適当に送らせています。  
  
  
```swift
final class DataLoadOperation: Operation {
    
    var favoritePhoto: FavoritePhoto?
    var completionHandler: ((FavoritePhoto) -> ())?
    
    private let _favoritePhoto: FavoritePhoto
    
    init(_ favoritePhoto: FavoritePhoto) {
        _favoritePhoto = favoritePhoto
    }
    
    override func main() {
        if isCancelled  { return }
        
        let delay = arc4random_uniform(2000) + 500
        usleep(delay * 1000)
        
        if isCancelled { return }
        self.favoritePhoto = _favoritePhoto
        
        if let completionHandler = completionHandler {
            DispatchQueue.main.async {
                completionHandler(self._favoritePhoto)
            }
        }
    }
}
```  
  
  
ViewControllerでデリゲートメソッドを処理するように設定  
  
```swift
collectionView.prefetchDataSource = self
```  
  
デリゲートの実装  
  
```swift
extension ViewController: UICollectionViewDataSourcePrefetching {

    func collectionView(_ collectionView: UICollectionView, prefetchItemsAt indexPaths: [IndexPath]) {
        for indexPath in indexPaths {
            // すでに実行中の場合は何もしない
            if let _ = loadingOperations[indexPath] {
                return
            }

            // Operationを作成し、画像の読み込みを開始する
            if let loader = dataStore.loadFavoritePhoto(at: indexPath.item) {
                loadingQueue.addOperation(loader)
                loadingOperations[indexPath] = loader
            }
        }
    }
    
    // スクロールの方向が変わった場合に呼び出される
    func collectionView(_ collectionView: UICollectionView, cancelPrefetchingForItemsAt indexPaths: [IndexPath]) {
        for indexPath in indexPaths {

            // もうデータが必要ないためOpeationのキャンセルと削除をする
            if let loader = loadingOperations[indexPath] {
                loader.cancel()
                loadingOperations.removeValue(forKey: indexPath)
            }
        }
    }
}
```  
  
  
willDisplayとdidEndDisplayingの実装です。  
  
```swift

extension ViewController: UICollectionViewDelegate {
    
    // collectionViewにcellが表示される直前に呼ばれる
    func collectionView(_ collectionView: UICollectionView, willDisplay cell: UICollectionViewCell, forItemAt indexPath: IndexPath) {
        guard let cell = cell as? CollectionViewCell else {
            return
        }
        
        let update: (FavoritePhoto?) -> () = { [unowned self] photo in
            cell.show(photo)
            // セルの設定が完了したのでOperationを削除
            self.loadingOperations.removeValue(forKey: indexPath)
        }
        
        // すでにOpeationがあるか(データをロード中か)？
        if let loader = loadingOperations[indexPath] {
            
            // データのロードが完了しているか？
            if let photo = loader.favoritePhoto {
                cell.show(photo)
                // セルの設定が完了したのでOperationを削除
                self.loadingOperations.removeValue(forKey: indexPath)
            } else {
                
                // 画像のロードが未完了の場合、完了後のハンドラーを設定
                loader.completionHandler = update
            }
        } else {
            
            // Operationを作成する
            if let loader = dataStore.loadFavoritePhoto(at: indexPath.item) {
                loader.completionHandler = update
                loadingQueue.addOperation(loader)
                loadingOperations[indexPath] = loader
            }
            
        }
    }
    
    
    // collectionViewからcellが表示範囲外になった時に呼ばれる
    func collectionView(_ collectionView: UICollectionView, didEndDisplaying cell: UICollectionViewCell, forItemAt indexPath: IndexPath) {
        
        // Operationが存在する場合はキャンセルして削除する
        if let loader = loadingOperations[indexPath] {
            loader.cancel()
            loadingOperations.removeValue(forKey: indexPath)
        }
    }
}
```  
  
## 追記  
  
アップルのサンプルを真似て非表示領域を追加してみました。  
背景がグレーの部分は本当はCollctioViewは表示されない領域ですが、  
Clip to Boundsをfalseにしています。  
表示領域に入る直前に画像が設定されますが、すでに読込済なのですぐに画像を設定することができているようですね。  
  
![sample.mov.gif](/image/05e6dce3-01cf-f2a3-edb7-7a38c3b18a2d.gif)  
  
  
  
# まとめ  
  
基本的なスクロールの改善に加えてPre-Fetching APIを活用することで  
よりスムーズなUIを実現することが可能になりました。  
  
今回の例はシンプルなのであまり変化は感じられませんでしたが、  
より複雑なUIややり方を工夫することでもっと表現の幅が広がるのではないかと思います。  
  
  
明日(正確には明後日)はいよいよWWDC2018ですね。  
今年はどんなサプライズが待っているのでしょうか:grinning:  
  
  
## 関連記事:  
[【Swift】この時期だから見直すiOS10の新機能 UIGraphicsImageRendererとUIViewPropertyAnimator]  
(https://qiita.com/stzn/items/26d8c238e7055c523f80)  
  
[【Swift】この時期だから見直すiOS10の新機能 UserNotificationsとNotification Content ExtensionとNotification Service Extension]  
(https://qiita.com/stzn/items/ab39faae1ef3a758fc08)  
  
[【Swift】この時期だから見直すiOS10の新機能 AVCapturePhotoOutput AVCaptureSettings など]  
(https://qiita.com/stzn/items/d7738f998e4be2d37c0f)  
  
