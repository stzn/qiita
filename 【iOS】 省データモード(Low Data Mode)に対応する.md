iOS13が正式に登場するに当たって  
新しく追加される機能などについて学んだことを記録しています。  
  
# 省データモード(Low Data Mode)とは？  
  
iOS13に追加されたユーザがデータ通信を抑えるために設定できる機能です。  
  
# 何が起きるの？  
  
これを設定することで  
バックグラウンドで実行されていたデータ通信を行わなくなります。  
  
例えば  
Photosアプリは  
バックグラウンドでデータの同期などを行いますが  
省データモードだと行いません。  
  
またニュースアプリなどで使用されている  
Background App Refreshも行われなくなります。  
  
さらに緊急を要さないバックグラウンドタスクも延期されます。  
  
<img width="1608" alt="スクリーンショット 2019-09-14 12.10.47.png" src="/image/f62d811a-4ed6-52e6-d40b-f69e547af155.png">  
  
# 設定方法  
  
モバイル通信とWifiで設定方法が異なります。  
  
## モバイル通信の場合  
  
モバイル通信 > 通信オプションから設定できます。  
  
![IMG_59E64D6BEF5C-1.jpeg](/image/a415a183-6bcb-61c0-2d1a-a993b83c98d7.jpeg)  
  
  
## Wifiの場合  
  
個々のWifiに対して設定します。  
表示されているWifiのインフォメーションアイコンをタップします。  
  
![IMG_A2B99A9E9012-1.jpeg](/image/b5fc3174-4a4d-8452-be9b-7aa7374eae79.jpeg)  
  
すると下記の画面が表示されます。  
  
![IMG_001B195EB0DD-1.jpeg](/image/cc9bdbb9-a757-fbae-991d-fc4fb50c628e.jpeg)  
  
  
# 対応方針  
  
では  
省データモードに必要な対応は何か？  
どう対応していけば良いのか？  
についてWWDC2019のセッションを中心にまとめてみたいと思います。  
  
セッションでは下記のような項目が挙げられています。  
  
<img width="1603" alt="スクリーンショット 2019-09-14 12.34.50.png" src="/image/e326cd84-ebae-da8a-e47e-8330c77938a3.png">  
  
## 画像の解像度を減らす  
  
画像がメインコンテンツのアプリでないならば  
多くのデータ量を小さくできる上に  
ユーザも継続してアプリを使用続けてもらえます。  
  
## prefetchを減らす  
  
prefetchはパフォーマンスの観点で非常に優れたテクニックですが  
データ通信という観点では使われない可能性のあるデータも取得してしまうため  
避けた方が良いでしょう。  
  
スクロールして次のページのデータをロードしている時間が少し長くなる可能性はあります。  
  
## 同期の頻度を減らす  
  
データは少しの間最新ではなくなるかもしれませんが  
データ自体は持っているのでユーザは継続して使うことができます。  
  
## バックグラウンドタスクを任意の時間の実行にする  
  
プロパティの設定で  
バックグラウンドタスクの実行タイミングをシステムに任せるようにします。  
  
もしかしたら即時に実行するべきバックグラウンドタスクの少なさに気がつくかもしれません。  
  
## 自動再生しないようにする  
  
ユーザが関心のあることに対して自動で再生されるのは良い体験を与えますが  
関心のないことにそれをやる必要はありません。  
  
## ユーザの操作は邪魔しない  
  
データ通信量を抑えられるからといっても  
ユーザが何かを行おうしていることを邪魔しては行けません。  
そういうケースにはデータ通信を抑える度合いを少なくすることも  
選択肢に入れるのが良いでしょう。  
  
# 対応方法は？  
  
では上記の方針を踏まえた上で  
具体的にどういう対応ができるのかを見ていきます。  
  
## allowsConstrainedNetworkAccess  
  
URLSessionに新しく`allowsConstrainedNetworkAccess`というプロパティができました。  
これはデフォルトがtrueになっており  
省データモードの利用が許可されています。  
  
https://developer.apple.com/documentation/foundation/urlsessionconfiguration/3235751-allowsconstrainednetworkaccess  
  
URLSessionの個々のリクエストまたはconfigurationで設定できます。  
  
**大きいリソースの取得やprefetchの場合は  
この設定をfalseにして取得できるかどうかをまず確認することを推奨しています。**  
  
それで失敗してかつその理由が  
  
```swift

error.networkUnavailableReason == .constrained
```  
  
であった場合は省データモードであることになるので  
省データモード用の処理を実行するようにします。  
  
こうすることで  
すでにキャッシュにあるはずのデータを利用することができることもあり  
より大きなメリットを享受できるかもしれません。  
  
例としては  
リソースを取得しようとして  
省データモードならばリソースの小さいものを取得するケースが紹介されています。  
  
```swift

func adaptiveLoader(regularURL: URL, lowDataURL: URL) -> AnyPublisher<Data, Error> {
    var request = URLRequest(url: regularURL)
    // 省データモードを禁止
    request.allowsConstrainedNetworkAccess = false
    return URLSession.shared.dataTaskPublisher(for: request) 
        .tryCatch { error -> URLSession.DataTaskPublisher in
            // 省データモードかどうか
            guard error.networkUnavailableReason == .constrained else { 
                throw error 
            }
            // 省データモード用のリソースを取得
            return URLSession.shared.dataTaskPublisher(for: lowDataURL)
        }
    ...
}
```  
  
## prohibitConstrainedPaths  
  
Network.frameworkのプロパティで  
trueにすると省データモードの使用できなくすることができます。  
  
https://developer.apple.com/documentation/network/nwparameters/3200461-prohibitconstrainedpaths  
  
しかし  
Network.frameworkには他のオプションがあります。  
同じホストに接続しようとすると接続を確立することができます。  
そして一度確立すると`NWPath`の`isConstrained`からLaw Data Modeかどうかの判定ができ  
`prohibitConstrainedPaths`の設定を変更することができます。  
  
https://developer.apple.com/documentation/network/nwpath/3200463-isconstrained  
  
# モバイルデータ通信  
  
省データモードではありませんが  
モバイルデータ通信やインターネット共有を介したWifi通信の場合も  
データ通信を抑えたいケースがあり  
その場合の対応方法も紹介されています。  
  
## allowsExpensiveNetworkAccess  
  
iOS12でNetwork.frameworkで登場したものが  
URLSessionにもできました。  
  
https://developer.apple.com/documentation/foundation/urlrequest/3358305-allowsexpensivenetworkaccess  
  
ただ  
これはシステムが判定することがほとんどであり  
モバイルデータ通信やインターネット共有を介したWifiに接続している時に設定されるので  
ユーザがコントロールできるものではありません。  
できる限り省データモードでの判定を推奨しています。  
  
もし必要な場合は  
モバイルデータ通信かどうか(Cellular)  
またはExpensiveかどうかで判定できますが  
Expensiveを使った方が良いようです。  
  
今現在だとCellularはExpensiveと判定されていますが  
将来ずっとExpensiveであるとは限らないからです。  
  
## prohibitExpensivePaths  
  
Network.frameworkも`prohibitConstrainedPaths`と同様に  
`prohibitExpensivePaths`が存在します。  
  
https://developer.apple.com/documentation/network/nwparameters/2998702-prohibitexpensivepaths  
  
trueにすると省データモードの使用できなくすることができます。  
  
https://developer.apple.com/documentation/network/nwparameters/3200461-prohibitconstrainedpaths  
  
`NWPath`の`isExpensive`からLaw Data Modeかどうかの判定ができ  
`prohibitExpensivePaths`の設定を変更することができます。  
  
https://developer.apple.com/documentation/network/nwpath/2998722-isexpensive  
  
# まとめ  
  
省データモードについて見てみました。  
  
公共のWifiスポットが増えたり  
MVNO(格安SIMを提供している事業者)の利用者が今後増えていく中で  
省データモードを利用する人は多いのではないかと思います。  
  
またもしかしたらまた新しい接続手段が増えていくことも考えられます。  
  
その中でいかにユーザ体験を損わないかを知るためのツールを  
持っておくことはとても大切だなと感じました。  
  
これから使っていく中で  
色々と気がつくことは多いと思いますので  
その時はまた記録に残したいと思います。  
  
もし間違いなどございましたらご指摘頂けるとうれしいです🙇🏻‍♂️  
