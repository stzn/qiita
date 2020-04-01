Firebase Meetup#10にブログ枠で参加させていただき、  
その時の内容を簡単に報告をさせていただきます。  
  
※何か間違いなどございましたらご指摘いただけますと幸いです:bow_tone1:  
  
# Firebase Meetup#10   
https://firebase-community.connpass.com/event/115709/  
  
# FJUC(Firebase Japan User Group)コミュニティの紹介  
  
### どういうところ？  
日本のFirebaseユーザーのコミュニティ  
  
### 主な活動  
勉強会などイベントの開催  
  
### Slackチャンネルはこちらから参加  
https://firebase.asia/join  
  
### Twitter ハッシュタグ #FJUG  
https://twitter.com/search?src=typd&q=%23FJUG&lang=ja  
  
# 発表  
  
以下に発表内容をまとめます  
  
  
## タイトル  
What's new in Cloud Firestore and MLKit  
  
## 登壇者  
[@d_date](https://twitter.com/d_date)さん  
  
- GDE https://developers.google.com/experts/people/daiki-matsudate  
- 株式会社FOLIO iOSエンジニア  
- FirebaseのPodcastをやっている [FJUG CAST](https://cast.firebase.asia)  
- try! Swift Tokyoオーガナイザー https://www.tryswift.co/events/2019/tokyo/jp/  
  
## 資料  
https://speakerdeck.com/d_date/whats-new-in-cloud-firestore-and-mlkit  
  
## 内容  
FirestoreとMLKitの最新情報  
  
### FirestoreがGAになりました:raised_hands_tone1:  
  
Google Cloud Blog  
https://cloud.google.com/blog/products/databases/announcing-cloud-firestore-general-availability-and-updates  
  
Firebase Blog  
https://firebase.googleblog.com/2019/01/cloud-firestore-in-general-availability.html  
  
### 10個の新しいローケーションに対応  
  
Multi-region  
- Europe (eur3)  
  
North America (Regional)  
- Los Angeles (us-west2)  
- Montréal (northamerica-northeast1)  
- Northern Virginia (us-east4)  
- South America (Regional)  
- São Paulo (southamerica-east1)  
- Europe (Regional)  
- London (europe-west2)  
  
Asia (Regional)  
- Mumbai (asia-south1)  
- Hong Kong (asia-east2)  
- **<span style="color:red">Tokyo (asia-northeast1)</span>**:clap_tone1:  
- Australia (Regional)  
- Sydney (australia-southeast1)  
  
### 東京リージョンになると何が嬉しいのか？  
- レイテンシーが少なくなる(特にCloud Functionを使っても海を超えることがなくなる *Podcastより)  
  
### プロジェクトのロケーションを指定する時の注意点  
  
ドキュメントにも記載があるように  
  
```
警告: プロジェクトのロケーションを選択した後は、ロケーションを変更できません。 
プロジェクトのロケーションの設定は複数のプロダクトに適用されます。
```  
  
なので注意が必要。  
国内で使うなら東京リージョンを選択すること。  
  
  
### SLAが利用可能に  
https://cloud.google.com/firestore/sla  
  
SLAとは、Service Level Agreementの略で簡単に言うと、サービスを落とさずに稼働しつづけることができる確率。  
これに違反した場合は請求金額から割引をしてくれる。  
  
- マルチリージョンインスタンス: 99.999%以上  
- 各リージョンのインスタンス: 99.99%以上  
  
### マルチリージョンの設定が可能に  
  
マルチリージョンとは複数のリージョンで同じDBのデータを保持することで災害時のデータ復旧に備えることができる  
  
  
※  
ゴジラが来ても大丈夫！！！と書いてある。  
  
### 料金が最大50%オフ  
3月3日に~~発表~~実施されるらしい(現在2月9日)  
  
※料金に関してはこちら  
[Firestore](https://firebase.google.com/docs/firestore/pricing?hl=ja)  
[インターネット下り料金](https://cloud.google.com/firestore/pricing?hl=ja#internet-egress)  
  
### Stack Driverでモニタリングが可能に(まだBeta)  
  
ReadやWriteがどのくらいされているのか、料金がどのくらいかかっているのかをモニタリングして  
アラートを飛ばすこともできるように。  
  
これから色々な機能が追加されていくようです。  
  
下記はFirebaseのブログ記事にあった画像です  
<img src="https://1.bp.blogspot.com/-jyaCn-ztHX8/XFH4Mus9a3I/AAAAAAAADWE/-HUL6kEXvTUe4-DcQmBI71PWdtpSJjVkwCLcBGAs/s1600/image2.png"></img>  
  
### Cloud Databaseユーザーは2019年の後半に自動アップグレードがあるらしい  
  
ブログに記載がありました。  
  
```
Existing Cloud Datastore users will be live-upgraded 
to Cloud Firestore automatically later in 2019. 
```  
  
[アップグレード方法について](https://cloud.google.com/datastore/docs/upgrade-to-firestore)  
  
  
### MLKit for Firebase(Beta)  
  
https://firebase.google.com/docs/ml-kit/?hl=ja  
  
VisionAPIや[TensorFlow Lite](https://www.tensorflow.org/lite/?hl=ja)のカスタムモデルを使用して  
簡単に機械学習をさせることができます。  
  
  
### 認識できる5つのもの  
- テキストの認識  
- 顔の検出  
- バーコードのスキャン  
- 画像のラベル付け  
- ランドマークの認識  
  
### 料金  
  
#### オンデバイスの学習モデルは無料  
- テキスト認識はLatin文字のみなので日本語を使用する場合はオンラインでやる必要がある  
- バーコードスキャンができる  
- カスタムモデルの推論ができる  
  
#### API通信が挟まるモデル  
- $1.50 /K（Blazeのみ）  
- 認識できる種類が増える  
  
### 実は自然言語処理ができるように??  
  
日本語ドキュメント更新されていませんが、  
実はLanguage Identificationが追加になっていました。  
https://firebase.google.com/docs/ml-kit/  
https://firebase.google.com/docs/ml-kit/identify-languages  
  
### iOSでの使い方  
  
Podfileを作成し、  
  
```swift

pod 'Firebase/Core'
pod 'Firebase/MLNaturalLanguage'
pod 'Firebase/MLNLLanguageID'
```  
  
importして  
  
```swift

import Firebase
```  
  
以下のような感じで実装(動きません)  
  
```swift

let languageId = NaturalLanguage.naturalLanguage().languageIdentification()

languageId.identifyLanguage(for: inputTextView.text) { (languageCode, error) in
  if let error = error {
    print("Failed with error: \(error)")
    return
  }
  if let languageCode = languageCode, languageCode != "und" {
    print("Identified Language: \(languageCode)")
  } else {
    print("No language was identified")
  }
}

```  
  
デフォルトだとund(undefined)の結果は返さず、  
信頼係数が0.5を超えるものしか変えられませんが、  
Optionで調整もできるようです。  
  
```swift

let options = LanguageIdentificationOptions(confidenceThreshold: 0.4)
let languageId = NaturalLanguage.naturalLanguage().languageIdentification(options: options)
```  
  
また、LanguageIdentificationを使うことでどの言語に近いかどうかのリストを返してくれます。  
  
  
```swift

let languageId = NaturalLanguage.naturalLanguage().languageIdentification()

languageId.identifyPossibleLanguages(for: text) { (identifiedLanguages, error) in
  if let error = error {
    print("Failed with error: \(error)")
    return
  }
  guard let identifiedLanguages = identifiedLanguages,
    !identifiedLanguages.isEmpty,
    identifiedLanguages[0].languageCode != "und"
  else {
    print("No language was identified")
    return
  }

  print("Identified Languages:\n" +
    identifiedLanguages.map {
      String(format: "(%@, %.2f)", $0.languageCode, $0.confidence)
      }.joined(separator: "\n"))
}
```  
  
ちなみに１つの文字列で１つの言語にしか対応しておらず、  
複数言語が混ざっていても認識はできないようです。  
  
また、こちらもOptionで信頼係数の変更ができます。  
  
```swift

let options = LanguageIdentificationOptions(confidenceThreshold: 0.4)
let languageId = NaturalLanguage.naturalLanguage().languageIdentification(options: options)
```  
  
### TensorFlow LiteがモバイルのGPUを使うことでより高速に  
  
[詳しくはこちら](https://medium.com/tensorflow/tensorflow-lite-now-faster-with-mobile-gpus-developer-preview-e15797e6dee7)  
  
## タイトル  
Firebaseでつくるグループチェックリスト管理サービス  
  
## 登壇者  
[@mogaming](https://twitter.com/_mogaming)さん  
  
- iOS、Androidエンジニア  
  
## 資料  
https://speakerdeck.com/mogaming/check-list-apps-on-firebase?slide=35  
https://gist.github.com/mogaming217/5a672f2ed05ed84ef3ceeb2572fb5642  
  
## 内容  
Firebaseを使ってchecka!というアプリを作った際の知見(特にFirestoreとCloud Functions)  
https://itunes.apple.com/jp/app/checka-チェッカ/id1451433619  
  
### Firebaseで使用しているサービス  
- Cloud Functions  
- Firestore  
- Authentication  
- Analytics  
- Crashlytics  
- Firebase Cloud Messaging  
  
### Firestore  
  
#### rulesで権限の管理をする  
- 読めるデータと書き込めるデータを制限  
  
#### データ構造の工夫が必要  
- 必要なデータは同じデータを複数のドキュメントにコピーする  
- その中で非同期でデータの整合性を取るようにする  
  
### Cloud Functions  
firestore.rulesで複雑で表現しきれない部分を補う  
特にデータのの作成が複雑な場合や権限でユーザーが作成できない範囲をカバー  
※  
Cloud FunctionsはAdmin SDKを使用する  
  
### 具体例  
  
#### データ構造  
  
![113d83f2e5a4a5bd437620859688c268-png.png](/image/8f77bc89-386f-ce03-2cc8-dcc2dbb93f57.png)  
  
#### 設計する時に考えた大事なこと  
  
##### create権限は重要な箇所には与えない  
  
Board, Task, Commentグループに属していれば作成できようしたい  
↓  
グループに属しているかどうかを権限で管理したい  
↓  
exists(path)を使用する  
https://firebase.google.com/docs/firestore/security/rules-conditions?hl=ja  
  
```js

exists(/../gruops/$(groupID)/users/$(requestUserID))
```  
  
#### 重要なデータはCloud Functionsで作成  
  
グループに属するユーザーはグループ参加用のCloud Functionsをアプリから呼び出して作成する  
↓  
同時にUserの属するグループも作成する  
  
### 工夫したこと  
  
#### updateの反映は非同期にする  
  
##### グループの名前の変更  
  
あるユーザーがグループの名前を変更したい  
↓  
でもupdate権限は渡したくない  
↓  
User側のグループの名前を変更してCloud Functionsを呼び出し変更を反映する  
  
### その他の工夫していること  
  
#### SnapshotListnerの活用  
リアルタイムにUIを更新できて気持ちが良い  
  
#### Userのデータはアプリ内で別途キャッシュ機構を作っている  
User名を表示する箇所は多数あるのでパフォーマンスや料金を意識  
  
## タイトル  
無限スクロールの解説  
  
## 登壇者  
[@kahirokunn](https://twitter.com/kahirokunn)さん  
- 株式会社scouty  
  
## 資料  
https://slides.com/kahirokunn/deck-9#/  
https://github.com/kahirokunn/book-management  
  
## 内容  
Firestoreやライブラリを使った無限スクロールの解説  
  
### 使用ライブラリ  
[pring](https://github.com/1amageek/Pring)  
[rxfire](https://www.npmjs.com/package/rxfire)  
[rxjs](https://github.com/ReactiveX/rxjs)  
vue + vuex  
  
##### Firestoreのデータ取得は順序が決まっていない  
  
##### 更新日時の降順で取得することで更新されたことも一番上に来るようにする  
  
##### {id:data}形式で流れてきたデータを全部保持しておき、getterで降順に並べ替え  
  
## タイトル  
Firestore導入前に検討したかったベスト5  
  
## 登壇者  
[@pitown](https://twitter.com/pitown)さん  
- 株式会社Hotspring  
- [ズボラ旅](https://www.cocolocala.jp/lp/zubora/)というサービスを作成  
  
## 資料  
https://speakerdeck.com/pochisato/firestoredao-ru-qian-nijian-tao-sitakatutabesuto5-at-firebase-meetup-number-10  
  
## 内容  
FirestoreをメインDBとしてFirestore中心に本番のプロダクトを開発した話  
  
### 第５位 料金  
  
結構な使用量でも１万円もかかっていないので安い(これからさらに安くなる)  
  
### 第4位 プロダクトへの適正  
  
DB構造さえも変わってしまうような仕様変更のあるスピード感が必要なサービスには良い。  
スキーマレスなので変更に非常に強い。  
  
一方でリレーションがすごい重要なデータを扱い場合には合わないかも(そんなにあるかは疑問)  
  
※  
TipsとしてReadとWriteの型があっていると処理が楽になる  
  
### 第3位 アーキテクチャ  
  
![f91d0c4407d48b8cf2f7497a6a767ab2-png.png](/image/10fe26ed-95d3-d6a1-6cba-fecef2695ef9.png)  
  
![83dc61008471e6382c460df28a81634c-png.png](/image/df06d681-fc26-f4e1-e3ba-e23184819edb.png)  
  
#### Firestoreの変更をトリガーにCloud Functionsを呼びだしてさらにFirestoreに書き込み  
  
まずはクライアントジョインでいいや(２箇所でデータを置くと更新が面倒)  
↓  
パフォーマンスがまずくなる(１万件 x １万件くらいのクライアントサイドジョインで辛くなる)  
↓  
CQRS(読み込みと書き込み用のテーブルを用意)の考えを導入  
書き込みはやや正規化し、Cloud Functionsで読み込み用にデータ整形する  
  
#### Mobxを使用して極力シンプルに  
  
State管理を行うライブラリ  
https://github.com/mobxjs/mobx  
  
### 第2位 バグ  
  
#### UTF-8 UTF-16の実装を間違えると何も返ってこれない、Console画面も開けなくなった  
  
#### １分くらい書き込めていないということが何度かあった  
  
#### サーバサイドでコネクションを張りっぱなしにしているとたまに切れる  
  
  
色々あったが特性を理解することで解決できるものばかりであった。  
  
  
制限事項  
https://firebase.google.com/docs/firestore/quotas  
  
### 第1位 実際の知見  
  
「ぶっちゃけ、どうなの？」という話が聞きたかったがなかった。  
↓  
実際運用してみると本番運用に十分に耐えられるものだった！  
  
  
とりあえず、Firestoreおすすめだからやってみるのが良い  
  
  
## タイトル  
最近のFirebase  
  
## 登壇者  
[@コキチーズ](https://twitter.com/k2wanko)さん  
- 株式会社Hotspring  
- FJUGオーガナイザー  
- GCPUG Tokyo  
  
## 資料  
https://speakerdeck.com/k2wanko/recent-firebase  
https://github.com/firebase  
https://github.com/firebase/functions-samples  
https://github.com/k2wanko/fire-timeline  
  
## 内容  
最近のFirebase事情  
  
### Firebaseとは？  
  
- モバイルアプリ作成のプラットフォーム  
- クライアントアプリだけでの開発が可能  
- サーバのメンテが不要  
  
  
### 対応プラットフォーム  
- Android  
- iOS  
- Web  
- C++  
- Unity  
  
### とは言ってもサーバーが必要な時もある  
Admin SDKはサーバーでFirebaseを操作する  
  
**対応言語**  
- C#  
- Node.js  
- Python  
- Go  
  
### Github  
いくつかのクライアントコードやSDKはOSSで提供  
https://github.com/firebase  
  
### Firebaseを始めるだけならまずここから  
- Google Analytics for Firebase  
- Cloud Storage for Firebase  
- Firebase Hosting  
- Firebase Authentication  
- Cloud Firestore  
- Firebase Cloud Messaging  
- Cloud Functions for Firebase  
- Google Analytics for Firebase  
  
### Google Analytics for Firebase  
- 無料で使えるモバイル向けのアナリティクスサービス  
- イベントベースに分析が可能  
- BigQueryにデータエクスポート可能  
- 独自のイベントの定義も可能  
- 特定の行動をしたユーザーをグループ化して追うことができる  
  
##### 新機能  
https://firebase.googleblog.com/2019/01/a-crash-course-in-using-new-audiences.html  
  
##### 除外するユーザーを指定できるように  
  
それに伴いオーディエンスをより柔軟に制御できる機能も追加された  
  
ドキュメント  
https://firebase.google.com/docs/analytics/  
  
##### 有効期間の設定  
  
「2週間以内にこの操作を行ったユーザー」というセグメンテーションが可能になどの設定ができる  
  
#### Android, iOS向けのGoogle　AnalyticsSDKが終了  
- 2019年10月  
- 移行先はFirebase  
- 有料版のGoogleAnalytics360は影響なし  
  
https://support.google.com/analytics/answer/9167112?hl=ja  
  
### Cloud Functions for Firebase  
  
- イベント駆動で独自の関数を実行できるサービス  
- PubSubやHTTPやFirebaseのほとんどのサービスのイベントを取得できる  
- 標準のサポートはNode.jsのみ  
- GCP経由のCloud FunctionsでもFirestoreやRealtime DB、Analyticsのいべんとも受け取れる  
  
```js

gcloud functions event-types list
```  
  
でイベントの確認ができる  
  
##### Firebase Management API  
  
FirebaseプロジェクトのFirebaseリソースやFirebaseアプリなどの設定や管理を  
プログラムから行うことができるように。  
  
https://firebase.google.com/docs/projects/api/reference/rest/?hl=ja  
  
##### Goサポート  
Firebaseでという訳ではない  
https://github.com/firebase/firebase-admin-go  
  
### Authentication  
  
- パスワード認証、Google、Twitterなどのプロバイダー認証、SMS認証、独自認証に対応を使ってFirebaseユーザーにできる  
  
#### Cloud Identity for Customers and Partners   
https://cloud.google.com/identity-cp/docs/?hl=ja  
  
#### SAML認証やOIDCなどもサポートされた  
  
### Firebase Hosting  
  
- 静的ファイルを配信するホスティングサービス  
- 標準でHTTPS/HTTP2に対応  
- 独自証明書には対応していない  
- Cloud Functionsとの接続で動的コンテンツ配信もできる  
- サーバーサイドレンダリングやOpen Graph Protocolでに使える  
  
#### 複数のサイトを1つのプロジェクトで管理できる  
そんなに新しくはない  
  
### Firebase Cloud Messaging  
- Push通知を遅れるサービス  
- Webにも対応  
- トピック機能でサブスクライブしているユーザーにだけ送ることも可能  
  
#### スケジューリングの設定が可能  
  
毎日決まったメッセージを送ることなどが可能に。  
https://firebase.google.com/docs/notifications/?hl=ja  
  
#### GCMの終了  
  
- 2019年4月11日  
- 古いSDKやエンドポイントを使っている場合は移行が必要  
  
https://developers.googleblog.com/2018/04/time-to-upgrade-from-gcm-to-fcm.html  
  
### Cloud Firestore  
- リアルタイムにデータを同期できる分散NoSQLデータベース  
- クエリーの全ては強い整合性を持つ  
- クライアントから直接書き込める。オフラインでも書き込み可能  
- 読み書きの制限はセキュリティルール  
  
#### 基本構造  
- Collection  
- Document  
- Data  
  
#### GAになりSLA99.999%で利用可能に  
  
#### 値下げが行われる  
  
東京リージョンが安い。(変動している？)  
https://cloud.google.com/firestore/pricing  
  
#### ローカルエミュレーターでセキュリティルールのテストができる  
  
[解説記事はこちら](https://medium.com/google-cloud-jp/firestore-%E3%83%AD%E3%83%BC%E3%82%AB%E3%83%AB%E3%82%A8%E3%83%9F%E3%83%A5%E3%83%AC%E3%83%BC%E3%82%BF%E3%83%BC%E3%82%92%E8%A9%A6%E3%81%97%E3%81%A6%E3%81%BF%E3%81%9F-e435668c18d1)  
  
#### セキュリティルール  
  
- クライアントから直接書き込めるFirestoreでデータを保護する  
- Javascriptのような独自構文  
- テストモードで誰でも読み書き可能にできるが、最初からルールは作成するべき（後々の漏れを防げる）  
  
##### 書き方  
  
認証しているユーザーのみに読み書きさせる。  
  
```javascript

service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if request.auth.uid != null;
    }
  }
}
```  
  
  
##### 基本ルール  
  
- read  
    - get  
    - list  
- write  
    - create  
    - update  
    - delete  
  
  
##### 検証はAuthorization → Scheme Validation → Business Logicの順に行う  
  
参考サンプル  
https://github.com/k2wanko/fire-timeline  
  
#### Realtime DBとの使い分け  
  
- ほとんどの場合はFirestoreでOK  
- 少ないデータを頻繁にアップデートする用途の場合はRealtime DBの方が安いこともある  
- ユーザーがオフラインになったことを検知したい場合はRealtime DBでしかできない  
  
### Firebaseより深く知るためには  
  
#### ドキュメントを読む  
https://firebase.google.com/docs/?hl=ja  
  
#### サポートに聞いてみる  
https://firebase.google.com/support/?hl=ja  
  
#### コミュニティに参加してみる  
https://firebase.asia  
  
#### FJUG CASTを聞いてみる  
https://cast.firebase.asia/  
  
#### Youtube Channel  
https://www.youtube.com/channel/UCP4bf6IHJJQehibu6ai__cg  
  
  
# 最後に  
  
現在、社内で1つのサービスにFirebaseの導入を検討しています。  
  
そんなときにFirebase Meetupを見つけ初参加をしてみたところ、  
非常に多くのことを知ることができました。  
  
Web上にも情報はたくさん乗っていますが、  
実際の運用経験や使用状況などの情報はあまり見つからず、  
今回の参加を通して実際に使っている方のお話を聞くことができ  
導入していくにあたっての不安が少し和らいだと感じています。  
  
今後も機会があった際はMeetupに参加して知見を深めると共に、  
実際に使用していく中で何か情報共有できることができてきた際は  
発表などを通してFirebaseの輪を広げていけたら良いなと思っております。  
  
運営のみなさま、  
会場や飲食物などをご提供くださった株式会社メルカリ様、  
登壇されたみなさま、  
会場やTwitterなどで共に参加されたみなさま、  
今回は有意義な時間を誠にありがとうございました。  
