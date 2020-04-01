  
potatotips #55にiOSブログ枠として参加させていただきましたので、  
iOSの発表内容をまとめました。  
https://github.com/potatotips/potatotips/wiki/potatotips-55  
  
私の知識が乏しく、  
不足している箇所や間違いがある可能性がございますので、  
何かご指摘があれば教えてください:bow_tone1:  
  
# iOS 11以降でUIViewControllerTransitionCoordinatorによるアニメーションが初回だけ効かない問題  
  
## 発表者  
[kazuhiro4949](https://twitter.com/kazuhiro494949?lang=ja)さん  
  
## スライド  
https://speakerdeck.com/kazuhiro4949/ios-11yi-jiang-deuiviewcontrollertransitioncoordinator-niyoruanimesiyongachu-hui-dakexiao-kanaiwen-ti  
  
## 参考資料  
https://qiita.com/kazuhiro4949/items/e8df2a904cb994ce6ecc  
https://github.com/kazuhiro4949/UIViewControllerTransitionCoordinatorBug  
  
  
## 概要  
  
### UIViewControllerTransitionCoordinator  
  
カスタムトランジションを書かなくても部分的なアニメーションがいい感じに実装できて便利。  
  
### だがiOS11以降では一部おかしな挙動を発見  
画面遷移のアニメーションで起動後の最初の遷移でアニメーション後の状態が最初に表示されてしまう。  
  
### 実はiOSのバグであったことが判明。  
https://openradar.appspot.com/38135706  
  
### 解決策  
カスタムトランジションに処理をそのまま移して実行すると問題なく動く。  
  
  
# Embedded Unity  
  
## 発表者  
[tanakasan2525](https://twitter.com/tanakasan2525)さん  
  
## スライド  
https://speakerdeck.com/tattn/embedded-unity  
  
## 参考資料  
https://gist.github.com/tattn/d779acc86b7cd32996860f5fe97104c4  
  
## 概要  
  
### UnityのUI  
  
ネイティブのUIではなく独自のUIで実装する。  
UnityのUIをネイティブUIに組み込むのは大変  
そこでUnityをネイティブアプリに埋め込む  
  
### UnityのiOSビルドの流れ  
1.　C#のコードをC#のコンパイラでマネージドアセンブリ化  
2.　最適化  
3.　[IL2CPP](https://docs.unity3d.com/ja/current/Manual/IL2CPP-HowItWorks.html)でC++のコードに変換  
4.　ClangでiOS用のバイナリへビルド　   
  
こうするとUnityのViewを引っ張ってくれば好きなところにUnityの画面を置ける  
  
### Embedded frameworkを作成する  
  
- Unityから生成されたXcode用プロジェクトの.mmや.hファイルをほぼコピー(Unityの仕組みがよくわかっていないです...)  
- framework外から使用するHeaderをPublicにする  
- .xcconfigファイルを作ってプロジェクトの設定を書き換える(https://gist.github.com/tattn/d779acc86b7cd32996860f5fe97104c4)  
- パスを変更したり、extern"C"などを追加する  
  
※組み込む時は自動生成されたmain.mmを参考にすると良い  
  
### iOSとUnityのやりとり  
*UnitySendMessage*でUnityにデータを送る  
http://kan-kikuchi.hatenablog.com/entry/Safety_UnitySendMessage  
  
  
# iOS の HTTP キャッシュについて  
  
## 発表者  
[jpmartha_jp](https://twitter.com/jpmartha_jp)さん  
  
## スライド  
https://speakerdeck.com/jp_pancake/ios-false-http-kiyatusiyunituite  
  
  
## 概要  
  
APIのレスポンスで更新内容がすぐには反映されない事象がありiOS側が疑われた。  
そこでキャッシュについて色々調べてみた。  
  
前提: キャッシュはパフォーマンス向上に必要  
  
  
### iOSでキャッシュを読み込まない方法  
  
CachePolicyを変更  
  
```swift

let configuration = URLSessionConfiguration.default
configuration.requestCachePolicy = .reloadIgnoringLocalAndRemoteCacheData
```  
※実は.reloadIgnoringLocalAndRemoteCacheDataは実装されていない  
https://developer.apple.com/documentation/foundation/nsurlrequest/cachepolicy/reloadignoringlocalandremotecachedata  
  
### CachePolicyの設定だけではキャッシュを読み込まないが💾保存はする💾  
  
保存場所をシミュレータで確認してみると、  
/Library/Developer/CoreSimulator/Devices/xxx/data/Containers/Data/Application/xxx/Library/Caches/(BundleID)  
のcache.dbにあった  
  
中身はSQLiteなのでツールで見ることができる  
  
  
### でも、読み込まれないなら良いのでは？  
->もれたらまずい個人情報などは保存するべきではない  
  
対策としてcapacityを0にする方法もあるがキャッシュを利用できるなくなりそう。  
  
### 読み込みも保存もさせない方法  
  
urlCacheをnilにする  
  
```swift
let configuration = URLSessionConfiguration.default
configuration.urlCache = nil
```  
  
.ephemeralを使用する  
  
```swift
let configuration = URLSessionConfiguration.ephemeral
let session = URlSession(configuration: configuration)
```  
  
  
しかし、既存のキャッシュは残っているので削除する方法を調べた。  
  
### キャッシュをすべて削除する方法(特定のキャッシュを消す方法はうまくいかなかった)  
  
```swift
URLCache.shared.removeAllCachedResponses()
```  
  
  
# Pocochaにおけるアセットの管理  
  
## 発表者  
[noppe](https://twitter.com/noppefoxwolf?lang=ja)さん  
  
## スライド  
https://speakerdeck.com/noppefoxwolf/potatotips55  
  
## 参考資料  
https://github.com/noppefoxwolf/inaba  
https://github.com/noppefoxwolf/AppIconGen  
  
## 概要  
  
今回の発表のアセットにはコードは含まれない。  
画像や色やStoryboardなど。  
  
### Pocochaのアセット管理のルール  
  
#### １. Asset Literalは使わない  
  
Xcode上だと似たような画像や色が区別しづらい。  
-> 名前を付けて呼び出すことがベスト。  
-> 見た目と特徴で特定できる名前をつける + 共通で使用するものはCommonと頭に付ける  
-> xcassetsのネームスペースを活用するとシンプルなファイル名でアクセスできる(ProfileEdit/Triangle/Largeなど)  
  
#### ２. 文字列でリソースは使わない  
  
Typoするとランタイムでクラッシュする。  
-> R.swiftやSwiftGenなどを使ってコンパイル時にTypoに気がつくようにする  
  
#### ３. Interface Builderで画像は設定しない  
存在しない画像を設定してもビルドできてしまい、ランタイムでクラッシュする問題。  
-> IBでは設定しないルールを設定  
-> [Inaba](https://github.com/noppefoxwolf/inaba)で確認する(上記参考資料を参照)  
  
IBで画像を消すとAutoLayoutが壊れることがある  
-> Intrinsic content sizeを設定する  
  
#### ４. アプリアイコンは単一ソースから生成  
アプリアイコンはなぜかベクターpdfからラスタ画像をビルド時に自動生成してくれない  
-> [AppIconGen](https://github.com/noppefoxwolf/AppIconGen)を活用する  
  
#### ５. ダミーアセットは明確にプロジェクトに分けて管理する  
デバッグ用のアセットは隠しにくい。アプリにバンドルすると色々問題が出てくる可能性もある。  
-> 機能ごとにターゲットを作成してターゲットごとのxcassetsを作成する  
  
# How to use Dictionary.compactMapValues  
  
## 発表者  
[d_date](https://twitter.com/d_date?lang=ja)さん  
  
## スライド  
https://speakerdeck.com/d_date/how-to-use-dictionary-dot-compactmapvalues  
  
## 参考資料  
https://github.com/apple/swift-evolution/blob/master/proposals/0218-introduce-compact-map-values.md  
https://speakerdeck.com/d_date/make-our-swift-better  
  
## 概要  
  
Dictionary.compactMapValuesがSwift５で導入される(作者は発表者のd_dateさん)  
  
### どうやって使うのか？  
  
```swift
let d1 = ["a","1","b","2","c",nil]
d1.filter({ $0.value != nil}).mapValues({ $0! })
// ["b":"2","a":"1"](順番は任意)
```  
これがこうなる↓  
  
```swift
let d1 = ["a":"1","b":"2","c":nil]
d1.compactMapValues({ $0 })
// ["b":"2","a":"1"](順番は任意)
```  
  
JavascriptのlodashのomitByを参考にしたとのこと  
https://lodash.com/  
http://tacamy.hatenablog.com/entry/2018/03/04/005938  
  
  
### どうやって使うのか？２  
  
```swift
let d = ["a":"1","b":"2","c":"three"]
d.mapValues(Int.init).filter({ $0.value != nil}).mapValues({ $0! })
// ["b":2,"a":1](順番は任意)
```  
  
これがこうなる↓  
  
```swift
let d = ["a":"1","b":"2","c":"three"]
d.compactMapValues(Int.init)
// ["b":2,"a":1](順番は任意)
```  
  
### Swiftへのコントリビュート方法は下記を参照  
https://speakerdeck.com/d_date/make-our-swift-better  
  
  
# Continuous Delivery With Unity Command line arguments  
  
## 発表者  
[DayBySay](https://twitter.com/DayBySay)さん  
  
## スライド  
https://speakerdeck.com/daybysay/continuous-delivery-with-unity-command-line-arguments  
  
  
## 概要  
  
### Unityで開発中の話  
  
「アプリの最新版を共有してほしい」との要望があり、  
HockyApp経由でアップロードを行う。  
  
### Unityのビルド  
  
すごい時間がかかる。  
さらに、**アップロード中にUnity Editorは触れない!!!**  
  
### Xcodeのビルド  
  
さらに1~2分かかる  
  
### 悲劇  
  
設定を間違えてしまった。。。  
  
-> もう一度Unitiyのビルドからやり直し  
  
  
  
### そこで、Unityのビルド・配布の自動化  
  
今回は社内サーバーのビルドマシン+Jenkinsで構成  
  
### Unity Player -> Xcode.projの吐き出し  
  
**Unity Command Line Arguments**を使用する  
  
#### -executeMethod   
ビルド起動前(Unity起動時)に処理を挟める  
コマンドライン引数を受け取れる  
  
### Xcode.projの設定変更  
  
fastlaneは事情により使えない  
  
#### [PostProcessBuild]   
Playerのビルド完了後に処理を挟める  
  
  
# AVPlayer周りの設計tips  
  
## 発表者  
[toshi0383](https://twitter.com/toshi0383)さん  
  
## スライド  
https://speakerdeck.com/toshi0383/avplayerzhou-rifalseshe-ji-tips  
  
## 参考資料  
https://github.com/toshi0383/VideoPlayerManager  
  
## 概要  
  
AVPlayer周りのリファクタリング  
  
### 方針   
AVPlayerはUIとして捉えない  
  
### ViewControllerがAVPlayerを保持していると  
  
コードが散らばったり、他のViewから参照する場合にViewControllerを経由しなければならない  
  
  
### そもそもAVPlayerってUIなのか？  
  
*AVPlayerの初期化や操作はメインスレッド必須は迷信*  
※ただしKVOの登録はメインスレッドの方が良いという記述あり  
  
```
You can use Key-value observing (KVO) 
to observe state changes 
to many of the player’s dynamic properties, 
such as its currentItem or its playback rate. 
You should register and unregister 
for KVO change notifications on the main thread.
```  
  
https://developer.apple.com/documentation/avfoundation/avplayer  
  
  
加えて  
サブスレッドの処理がメインスレッドの処理の後回しにされて動画視聴のUXを低下させる場合があるので要検証。  
  
でも、UIに縛られる必要はないはず。  
  
### VideoPlayerManagerの導入  
- UI層から分離したらすっきり  
(RxSwiftを活用することでさらにすっきり)  
- 複数のクラスから共通のplayerの操作が可能になった  
- Mockを使ってテストがやりやすくなった  
  
  
# Sourceryを使ってイベントログ実装を改善した話  
  
## 発表者  
[_sgr_ksmt](https://twitter.com/_sgr_ksmt?lang=ja)さん  
  
## スライド  
https://speakerdeck.com/sgrksmt/improve-event-log-using-sourcery-in-ios  
  
## 参考資料  
https://github.com/sgr-ksmt/SourceryDemo  
  
  
## 概要  
  
イベントログをiOSに仕込む  
  
### 元々の実装  
- イベント名やパラメータ名をハードコーディング  
- 型がAnyなのでなんでも入れられる  
- 定義が散らばっていて迷子になる  
  
これを改善しよう。  
  
### 改善1  
イベント名とパラメータにProtocolとenumを活用  
  
### 結果  
- 定義の集約と文字列ハードコーディングの削減  
- 型推論を効かせて安全になった  
  
Swiftらしい実装になったが、  
イベント名とパラメータを毎回手で書くのは非常に大変  
  
### 改善2  
Sourceryによる自動生成  
https://github.com/krzysztofzablocki/Sourcery  
  
### 結果  
200個くらいのボイラープレートの生成がSourceryを使って0.1-2秒でできるようになった  
  
