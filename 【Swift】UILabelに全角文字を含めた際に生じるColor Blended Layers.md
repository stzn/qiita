最近、Instrumentsやパフォーマンスに関して色々と調べている中で、発見したことがあったのでメモします。  
  
# Color Blended Layersとは？  
  
レイヤーが重なっている部分で色や透明度が異なる場合、  
最終的に画面に表示するピクセルカラーを決めるためにシステムのレンダリングサーバが計算を行います。  
これを***Alpha Blending***と呼びます。  
  
これが発生しているレイヤを示したのがColor Blended Layersです。  
これ自体は必要なときに使用していれば問題ありませんが、  
意図してシステムが計算をしてしまい、フレームレートが下がってしまう場合に問題になります。  
  
Alpha Blendingが発生するのは  
alpha、backgroundColor、opaqueといったプロパティや  
画像のアルファチャネルなど様々な要素が影響してきます。  
  
  
# Color Blended Layersを調べる方法  
  
XcodeやSimulatorのDebugメニューから見ることができます。  
赤色の部分がColor Blended Layersになっている部分です。  
  
<img width="1283" alt="スクリーンショット 2018-08-03 8.13.31.png" src="/image/ccad291a-f6d7-1114-ca02-0bbfbb8c7c15.png">  
  
  
<img width="1506" alt="スクリーンショット 2018-08-03 7.57.59.png" src="/image/00444e76-6d7b-3dbe-15d8-0c39f00ef9a0.png">  
  
赤色になっている部分がColor Blended Layersになっている部分です。  
今回の場合は背景色などを特に指定している訳ではないので、  
意図せずに生じています。  
  
  
# 現象の確認  
  
まずは下記のような設定の場合を見て見ます。  
対象は画面にあるラベルの中の真ん中のラベルです。  
  
```
statusLabel.text = "aaaaaaaaaaaaaaa"
```  
  
<img width="434" alt="スクリーンショット 2018-08-03 8.03.49.png" src="/image/dd5fa389-aa1d-1f0c-5a83-dbb35f51a9b9.png">  
  
上記のようにColor Blended Layersになっています。  
  
これを解消するために以下のようにコードを追加します。  
  
```
statusLabel.backgroundColor = .white
```  
  
すると下記のように解消されます。  
  
<img width="421" alt="スクリーンショット 2018-08-03 8.04.40.png" src="/image/39bf2061-c447-13be-9c58-a64a13fa4662.png">  
  
これは背景色がwhiteに対してラベルはデフォルトで背景色が設定されていないため、  
色が異なっていたのを修正しました。  
  
※　opacityが異なる場合も発生します。今回は何も設定しないので全て1.0とみなされています。  
  
  
半角英数字記号などはこれで解決しますが、  
***全角文字***が含まれるとまたややこしくなります。  
  
```
statusLabel.text = "スタートを押してね"
statusLabel.backgroundColor = .white
```  
  
<img width="412" alt="スクリーンショット 2018-08-03 8.05.50.png" src="/image/5640c461-4731-ca08-20a0-8b4ea4e463c5.png">  
  
  
なんと、またColor Blended Layersになってしまいました。  
色々調べてみると、どうやら全角文字があると、UILabelが自動でsublayerを作成しているらしく、  
これがUILabel領域外に表示をしていることが原因のようです。  
  
<img width="1407" alt="スクリーンショット 2018-08-03 8.39.21.png" src="/image/d035eeeb-6200-2363-4103-727ff9f7e5d5.png">  
  
ちなみに半角の場合は作られません。  
  
<img width="1282" alt="スクリーンショット 2018-08-03 9.40.08.png" src="/image/afcf9020-7c1b-675e-96ac-18bf4e5cf23c.png">  
  
  
対処法としてはclipsToBoundsをtrueに設定することで解決しました。  
  
```
statusLabel.clipsToBounds = true
```  
<img width="425" alt="スクリーンショット 2018-08-03 8.40.45.png" src="/image/cc6a1144-f5b6-4847-35cf-974dfbb3f8ee.png">  
  
  
# 補足  
  
今回はUILabelを対象にしていますが、UIButtonのtitleも同じ現象が起き、同じ対処法で解決しました。  
  
# まとめ  
  
今回はColor Blended Layersの中でもラベルということに焦点を当てましたが、UIImageViewの透過度とUIImageのアルファチャネルの関係などでも同じ現象が起きます。  
また、Offscreen renderingといった別の問題もレンダリングのパフォーマンスに影響を与えます。  
  
こういった問題はTableViewやCollectionViewを使用した際に  
スクロールがカクツクような現象を引き起こしやすくなりますので気をつけたいですね。  
  
レンダリングの問題はUIViewの色々なプロパティが絡んできて複雑なので、またまとめてみたいと思います。  
  
# 余談  
  
早取りカルタというタイトルは  
そもそもカルタが早く取ることを目的としたゲームなので  
あまり意味がないですね:innocent:  
