iOSのアプリでは  
重いリソース(大きいデータや画像など)を外部から毎回取得してくると  
パフォーマンスや通信量に負担がかかってしまうということもあり  
端末(ローカル)にデータを保存して  
同じデータの場合は端末上のデータを利用することがあります。  
  
そしてその中でも  
データの種類や使用用途によって  
保存方法や保存場所も変える必要があります。  
  
これは  
扱いやすさという点だけではなく  
アプリ審査のリジェクトを防ぐという点でも  
必要になってきます。  
  
今回は  
端末にデータを保存する方法にはどんなものがあるのか？  
どうやってデータは保存されているのか？  
どういうデータをどういう方法で保存する必要があるのか？  
  
などについて見ていきたいと思います。  
  
今回取り上げるのは下記の4つです。  
  
- UserDefaults  
- ディスク上のファイル  
- Keychain  
- Database  
  
# UserDefaults  
  
アプリ内の  
**Library/Preferences**ディレクトリに  
**plist**ファイルとしてデータを保存しています。  
  
https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/FileSystemProgrammingGuide/FileSystemOverview/FileSystemOverview.html#//apple_ref/doc/uid/TP40010672-CH2-SW1  
  
### データの読み書きは速いか？  
  
ディスクへの書き込みが発生するため  
それなりのコストはかかりますが  
アプリ起動時にUserDefaultsはメモリ上に展開されるので  
データの読み込みは速いです。  
  
### どういうデータを保存するか？  
  
boolなどのプリミティブ型を使用して  
アプリのユーザーの設定やユーザー体験を向上させるような  
データを保存するのに向いています。  
  
メモリに展開されるので  
あまり大きなデータを保存してしまうと  
端末メモリを圧迫してしまいます。  
  
### 保存したデータはいつ削除されるか？  
  
アプリが削除されると消えます。  
  
### 注意点  
  
UserDefaultsは値をそのまま保存しており  
plistの中身を書き変えされてしまうリスクもあります。  
  
そのため個人を特定できるようなセキュアな値を保存してはいけません。  
(emailアドレスやパスワードなど)  
  
### 使い方  
  
UserDefaultsにはデフォルトのstandardという  
staticなプロパティを利用することができます。  
  
```swift

UserDefaults.standard.set(true, forKey: "isLoggedIn")
let isLoggedIn = UserDefaults.standard.bool(forKey: "isLoggedIn")

```  
  
また  
独自のUserDefaultsのインスタンスを生成することもできます。  
  
```swift

let myUserDefaults = UserDefaults("suiteName: com.myapp.myUserDefaults")
myUserDefaults.set(true, forKey: "isLoggedIn")
let isLoggedIn = myUserDefaults.bool(forKey: "isLoggedIn")

```  
  
詳細はドキュメントや多くの実装がありますので  
そちらを参照してください。  
https://developer.apple.com/documentation/foundation/userdefaults  
  
  
WWDC2019では  
Swift5.1から導入された`Property Wrappers`を利用して  
簡単にアクセスできる例も紹介されていました。  
  
https://github.com/apple/swift-evolution/blob/master/proposals/0258-property-wrappers.md#user-defaults  
https://developer.apple.com/videos/play/wwdc2019/402  
  
  
# ディスク上のファイル  
  
アプリ内の特定のディレクトリへ  
ファイルとしてデータを保存することができます。  
  
### データの読み書きは速いか？  
  
UserDefaultsに比べると  
扱っているデータも大きく  
読み込みは遅くなる傾向にあります。  
  
### どういうデータを保存するか？  
  
ユーザーが作成したテキストやPDF  
ダウンロードした画像や動画  
ネットワークから受け取ったレスポンス(JSONをエンコードしたもの)  
などUserDefaulsよりも大きな容量のデータを保存します。  
  
### 保存したデータはいつ削除されるか？  
  
アプリが削除されるのと同時に削除されます。  
  
### 注意点  
  
UserDefaultsと同様に保存されるデータは自動で暗号化されません。  
  
端末内から見られるリスクに加え  
iCloudを経由して見られてしまう可能性があります。  
  
そのため個人を特定できるようなセキュアな値を保存してはいけません。  
(emailアドレスやパスワードなど)  
  
  
また保存するディレクトリによって保存できる  
データの種類が変わってきます。  
これに違反するとアプリの審査でリジェクトされる可能性があります。  
  
まずは下記のような種類があります。  
大まかにまとめると下記のようになります。  
  
| ディレクトリ       | 保存内容            | データの読み込み     | データの書き込み     | iTunes、iCloudへのバックアップ  
|:-----------------|:------------------|:-------------------|:-------------------|:-------------------|  
| Documents        | ユーザ作成のコンテンツ(文書や画像、動画など) | ○ | ○ | ○ |  
| Documents/Inbox  | 他のアプリから受け取ったファイル | ○ | ○(削除のみ) | ○ |  
| Library          | ユーザ作成以外に保存したいファイル | ○ | ○ | ○(Cache以外) |  
| Library/Cache    | キャッシュでなど自動削除されてもよいファイル | ○ | ○ | × |  
| tmp              | アプリ起動中など一時的に使用するファイル | ○ | ○ | × |  
  
https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/FileSystemProgrammingGuide/FileSystemOverview/FileSystemOverview.html  
  
さらにガイドラインには下記のように書かれています。  
  
---  
1.  
ユーザが作成した文書やその他のデータ  
アプリで再生成できないようなデータは**Documents**ディレクトリに保存する。  
  
---  
  
2.  
再ダウンロードや再生成可能なデータは**Library/Caches**ディレクトリに保存する。  
  
例:  
漫画や雑誌、マップアプリなどで使われるデータベースのキャッシュファイルなど  
  
---  
  
3.  
一時的に保存が必要なものは**tmp**ディレクトリに保存する。  
不要になった際には削除をして端末の空きスペースを圧迫させないこと。  
  
---  
4.  
もし特定のファイルで端末の空きスペースが少ない場合でも  
削除されないようにしたい場合は  
"do not back up"属性を設定すること。  
これはどのディレクトリにのファイルでも有効になる。  
  
https://developer.apple.com/documentation/foundation/urlresourcekey/1408756-isexcludedfrombackupkey  
  
ただし空きスペースを使用し続けているため  
監視を続けて定期的に削除すること。  
  
例:  
再生成できるけどアプリを正しく動作させるのに必要なものや  
オフライン時でもユーザが使用できるようにしたいものなど。  
  
---  
  
https://developer.apple.com/icloud/documentation/data-storage/index.html  
  
### 使い方  
  
  
`FileManager`を使用します。  
  
```swift

do {
    let fileManager = FileManager.default
    let docs = try fileManager.url(for: .documentDirectory,
                                   in: .userDomainMask,
                                   appropriateFor: nil, create: false)
    let path = docs.appendingPathComponent("myFile.txt")
    let data = "Hello, world!".data(using: .utf8)!

    fileManager.createFile(atPath: path.absoluteString,
                           contents: data, attributes: nil)
} catch {
    print(error)
}
```  
  
  
# Keychain  
  
### データの読み書きは速いか？  
  
パフォーマンスが良くないといった情報は見つかりませんでしたが  
暗号化や復号することを考えるとUserDefaultsと比べて多少はコストが増えると考えています。  
(もしそういう情報がありましたら教えて頂けましたらうれしいです🙇🏻‍♂️)  
  
### どういうデータを保存するか？  
  
データを暗号化できるため  
emailやOAuthのトークンなどセキュアな小さい情報を  
保存するのみ主に使用されます。  
  
### 保存したデータはいつ削除されるか？  
  
アプリを削除してもデータは残ります。  
削除をするためには自身でAPIを呼び出して削除する必要があります。  
  
  
```swift

let status = SecItemDelete(query as CFDictionary)
guard status == errSecSuccess || status == errSecItemNotFound else { 
    throw KeychainError.unhandledError(status: status) 
}
```  
  
https://developer.apple.com/documentation/security/keychain_services/keychain_items/updating_and_deleting_keychain_items  
  
### 注意点  
  
Keychainはプロビジョニングファイルと紐づけて暗号化されており  
セキュリティは強固だとは思いますが  
それでもと端末上にデータを保存するため  
情報が見られてしまう可能性は考慮した方が良いかもしれません。  
  
  
### 使い方  
  
多くのKeychainのAPIはC言語ベースで書かれており  
扱いづらい部分があります。  
  
https://developer.apple.com/documentation/security/keychain_services  
  
Appleはサンプルコードで利用方法を紹介していますが  
かなり古いコードです。(Swift3)  
https://developer.apple.com/library/archive/samplecode/GenericKeychain/Introduction/Intro.html#//apple_ref/doc/uid/DTS40007797-Intro-DontLinkElementID_2  
  
自身で扱いやすいラッパークラスを生成したり  
すでに便利なライブラリもたくさんありますので  
そちらを活用するのが良いかもしれません。  
  
ライブラリ例:  
https://github.com/kishikawakatsumi/KeychainAccess  
  
  
# Databases  
  
Core DataやRealm、SQLiteなど  
特定のフォーマットやアクセス方法で  
端末上のデータを扱います。  
  
※  
Core Dataに関しては正確にはDatabaseというご意見もあるようですが  
同じように扱われるという点から例として挙げさせて頂いています。  
https://davedelong.com/blog/2018/05/09/the-laws-of-core-data/  
  
### データの読み書きは速いか？  
  
データは**Documents**ディレクトリに保存されることが多く  
ディスク上のファイルへの読み書きと同様に  
UserDefaultsに比べると時間がかかります。  
  
### どういうデータを保存するか？  
  
大きいデータや  
将来的にデータの数がどんどん増えそうなデータを  
配列として保存することに適しています。  
  
特に  
クエリを利用して  
特定条件のデータを取得できたり  
データ同士の関連性を持つことができるため  
構造化されたデータが扱いやすくなっています。  
  
例:  
APIから取得したJSONデータを  
独自のモデルにデコードしたものなど  
  
### 保存したデータはいつ削除されるか？  
  
アプリが削除されると消えます。  
  
### 注意点  
  
データは**Documents**ディレクトリに  
保存されることが多く  
データが見られてしまう可能性があるため  
セキュアな情報を保存することには向いていません。  
  
またスレッドセーフでない場合もあるため  
複数スレッドから同時にアクセスする場合は  
自分でコントロールする必要があります。  
  
### 使い方  
  
基本的にはSQLでアクセスするところを  
利用しやすいようにラップしたAPIが提供されています。  
具体的な利用方法はDatabaseによって異なりますので  
APIのドキュメントなどをご参照ください。  
  
Core Data  
https://developer.apple.com/documentation/coredata  
  
Realm  
https://realm.io/docs/swift/latest/api/  
  
SQLite  
https://github.com/stephencelis/SQLite.swift/blob/master/Documentation/Index.md#sqliteswift-documentation  
  
# まとめ  
  
端末(ローカル)にデータを保存する方法を見ていきました。  
それぞれ特徴によって  
保存するべきデータの種類が変わってきたり  
保存先も変わってきます。  
  
特にユーザを特定できるような個人情報に関しては  
情報漏洩など犯罪に関わってしまうリスクもあるため  
細心の注意が必要になります。  
  
近年ではJWTなどを利用することで  
個人情報を直接保持するリスクを  
減らしていけるので  
そういった仕組みは活用していきたいですね😃  
  
何か間違いなどございましたら  
ご指摘して頂けますとうれしいです🙇🏻‍♂️  
