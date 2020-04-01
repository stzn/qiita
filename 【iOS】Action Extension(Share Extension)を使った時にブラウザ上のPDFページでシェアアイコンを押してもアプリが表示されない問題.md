# 経緯  
  
個人的な開発をしている際に  
Action Extension(Share Extension)を初めて使う機会があり  
それとなく調べながらやってみたのですが  
色々すんなりといかなかったことがあったので  
記録してみました。  
  
もし同じ現象で悩んでいる人のお役に立てましたら幸いです。  
  
また、まだわからない部分が多いので  
間違いなどございましたら教えていただけますと嬉しいです🙇🏻‍♂️  
  
  
※今回の例ではAction Extensionを使用していますが、  
Share Extensionでも同様の現象が起きました。  
  
また、Extensionの名前は「TestActionExtension」で作成しています。  
  
# 検証環境  
  
iOS: 12.1  
Xcode: 10.1  
ブラウザ: Safari, Chrome  
使用PDF: http://www.takaramono-pj.jp/wp-content/uploads/2018/09/samplePDF.pdf  
  
※MacOS  
  
# 現象  
  
Web上のPDFのページを開いた際に  
自分のアプリでそのPDFの情報を共有したいと思ったため  
Action Extensionを使用しました。  
  
基本的な使い方に関しては↓を参考にさせていただきました。  
https://qiita.com/KosukeQiita/items/994693da551a7101cc9c  
  
## 検証1  
  
作成したばかりの状態でSafariで実行してみたところ  
  
![IMG_0192.PNG](/image/6824de52-2335-be75-ffef-f56ab7dbb82e.png)  
  
きちんと表示されました。  
  
info.plistの中身をみてみると、  
  
![スクリーンショット 2019-01-05 8.16.50.png](/image/05477a77-c580-b795-214a-cadc8b698c34.png)  
  
赤枠内のNSExtensionActivationRuleの値でアプリの表示・非表示を制限しています。  
TRUEPREDICATEは「全てOK」という意味になります。  
  
ただ、ドキュメントにも下記のように書かれているようにこのままですと  
**アプリの申請でリジェクトされます。**  
  
```swift

IMPORTANT

Before you submit your containing app to the App Store, 
be sure to replace all uses of TRUEPREDICATE stub predicates 
with functional predicate statements or with NSExtensionActivationRule keys. 
If any app extensions in your containing app include the string TRUEPREDICATE, 
the app will be rejected.
```  
  
https://developer.apple.com/library/archive/documentation/General/Conceptual/ExtensibilityPG/ExtensionScenarios.html#//apple_ref/doc/uid/TP40014214-CH21-SW1  
  
## 検証2  
  
そこでNSExtensionActivationRuleを設定します。  
  
![スクリーンショット 2019-01-05 8.32.55.png](/image/18da8e13-7111-2574-8dbc-81b7be47eb3a.png)  
  
この状態でSafariから実行すると  
  
  
![IMG_0196.PNG](/image/27e17535-af04-7eb4-fcbe-9e2250300495.png)  
  
  
「あれ？アプリが出てこない」  
  
  
何か設定方法を間違えているのかと思い、  
設定値を消したり、変えてみたりしましたが一向に出てきませんでした。  
  
そこで、Chromeで試してみたところ  
**なんとChromeではきちんと表示されました。**  
  
ここまでの経過でわかったこととしては、  
- TRUEPREDICATEの場合だと表示されるのでSafariだから表示されないということはない  
- Chromeでは表示されるため設定の方法自体が間違っていることはないようだ  
  
## 検証3  
  
そこで色々とまた調べてみたところ下記の記事に当たりました。  
https://qiita.com/dolfalf/items/121c02b3726874014af1  
  
問題となのは設定方法のようで  
SUBQUERYというものが必要になります。  
  
これはPredicate方式と呼ばれるもので書く必要があります。  
  
SQLを書いている方などは似ているので理解はしやすいかと思います。  
(文字列記載で扱いやすいとは個人的には思っていません。)  
  
詳細は下記をご参照ください。  
https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Predicates/Articles/pSyntax.html#//apple_ref/doc/uid/TP40001795  
  
  
まず下記のような方法で記載をしてみました。  
※plistをProperty Listで表示すると見づらいのでSource Codeで開いています  
  
![スクリーンショット 2019-01-05 8.51.54.png](/image/f6e39dab-4701-a3b8-c6ac-755c60067a49.png)  
  
これでうまくいくかなと思ったのですが  
**Safariではやはり表示されませんでした。**  
  
そして**Chromeでは表示されます。**  
  
## 検証4  
  
さらに色々試してみた結果  
  
```swift

SUBQUERY (
    extensionItems,
    $extensionItem,
    SUBQUERY (
        $extensionItem.attachments,
        $attachment,
        ANY $attachment.registeredTypeIdentifiers UTI-CONFORMS-TO "com.adobe.pdf"
        || ANY $attachment.registeredTypeIdentifiers UTI-CONFORMS-TO "public.url"
    ).@count == $extensionItem.attachments.@count
).@count == 2
```  
  
この場合でもSafariで表示されるようになりました。  
  
**しかし、Chromeでは表示されなくなりました。**  
  
  
中身を見ていくと、  
  
SafariではNSExtensionItemが2つ取れており、  
２種類のURLが取れていることがわかりました  
  
- file:///private/var/mobile/Containers/Data/Application/XXXXXXXXXXXXXXX/tmp/OpenIn/samplePDF.pdf   
- http://www.takaramono-pj.jp/wp-content/uploads/2018/09/samplePDF.pdf  
  
```swift

2 elements
  - 0 : <NSExtensionItem: 0x280c180a0> - userInfo: {
    NSExtensionItemAttachmentsKey =     (
        "<NSItemProvider: 0x2825004d0> {types = (\n    \"com.adobe.pdf\"\n)}"
    );
    NSExtensionItemAttributedContentTextKey = <7b5c7274 66315c61 6e73695c 616e7369 63706731 3235320a 7b5c666f 6e747462 6c7d0a7b 5c636f6c 6f727462 6c3b5c72 65643235 355c6772 65656e32 35355c62 6c756532 35353b7d 0a7b5c2a 5c657870 616e6465 64636f6c 6f727462 6c3b3b7d 0a7d>;
    "com.apple.UIKit.NSExtensionItemUserInfoIsContentManagedKey" = 0;
}
  - 1 : <NSExtensionItem: 0x280c18100> - userInfo: {
    NSExtensionItemAttachmentsKey =     (
        "<NSItemProvider: 0x282500540> {types = (\n    \"public.url\"\n)}"
    );
    "com.apple.UIKit.NSExtensionItemUserInfoIsContentManagedKey" = 0;
    supportsJavaScript = 1;
}

```  
  
  
一方でChromeで表示できている状態で見てみたところ  
ChromeではNSExtensionItemは1つとして見なされていました。  
  
- file:///private/var/mobile/Containers/Data/Application/XXXXXXXXXXXXXXX/tmp/OpenIn/samplePDF.pdf   
  
```swift

 1 element
  - 0 : <NSExtensionItem: 0x282c8c9a0> - userInfo: {
    NSExtensionItemAttachmentsKey =     (
        "<NSItemProvider: 0x280588460> {types = (\n    \"com.adobe.pdf\",\n    \"public.file-url\"\n)}"
    );
    "com.apple.UIKit.NSExtensionItemUserInfoIsContentManagedKey" = 0;
}
```  
  
## 検証5  
  
なんだろうなと思い、  
Appleのドキュメントを見返してみたところ  
  
```swift

SUBQUERY (
    extensionItems,
    $extensionItem,
    SUBQUERY (
        $extensionItem.attachments,
        $attachment,
        ANY $attachment.registeredTypeIdentifiers UTI-CONFORMS-TO "com.adobe.pdf"
    ).@count == $extensionItem.attachments.@count
).@count == 1
```  
  
と記載があり、  
  
**これが上手く行きました**  
  
  
## 考察  
  
ここから動きをみた考察になってしまいますが  
  
SafariとChromeで取得できる要素数が異なるため  
  
### 検証3がダメな理由  
２種類のtypeを指定しているが、countが1なので２種類とも取得できるSafariの数に合わない  
  
### 検証4がダメな理由  
２種類のtypeを指定しているが、countが2なので1種類しか取得できないChromeの数に合わない  
  
### 検証5がOKな理由  
1種類のtypeを指定しているかつcountが1で  
SafariもChromeも取得可能  
  
ということなのだと思います。  
  
## まとめ  
  
今回色々試しながらなんとか上手くいく方法を見つけましたが、  
App Extensionはまだまだ謎な部分が多くあるので、  
時間を見つけて調査してみたいと思います。  
  
※Chromeでpublic.urlを取得する方法、JavaScriptでWebページの情報を取得する方法など  
