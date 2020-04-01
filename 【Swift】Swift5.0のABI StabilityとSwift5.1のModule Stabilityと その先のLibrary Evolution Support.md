2019年3月18日に  
Swift5.1のfinal branchingがアナウンスされてから  
１ヶ月ほど経過しました。  
  
リリース日はまだ出ていませんが  
Appleのブログを見ている限り短いスパンでのリリースを考えており  
次のバージョンについても学び始める時期なのかと思っています。  
(WWDCに合わせるのかもしれませんが)  
  
https://swift.org/blog/5-1-release-process/  
  
Swift5.0ではABI Stabilityが達成されましたが、  
Swift5.1はModule Stabilityを主な目的としています。  
  
今回はSwift5.0で実現したABI Stability  
そして5.1で達成されるModule Stability  
さらに今後実現される予定のLibrary Evolution Support  
について簡単にまとめてみたいと思います。  
  
もし何か間違いなどごさいましたら教えていただけますとうれしいです:bow_tone1:  
  
# ABI Stability  
  
ABI StabilityはSwiftのバイナリの互換性を確保することによって  
Swiftのバージョンが異なっても  
他のバージョンのstandard libraryやruntime libraryでアプリの実行が可能になります。  
  
メリットとして下記のようなものがあります。  
  
## アプリサイズの削減  
  
今まではアプリを使用する時に同じバージョンのstandard libraryやruntime libraryが必要になるため  
これらをアプリのバンドルに含めなければならず容量が増えてしまっていました。(約5MB)  
  
今後はObjective-Cのruntimeと同様にOSと一緒に同梱されるため  
アプリのバンドルサイズを削減することができます。  
  
## パフォーマンスの向上  
  
ホストのOSとより統合することができるため  
  
- 起動時間の短縮  
- runtime時の処理の最適化  
- 使用メモリの削減  
  
が期待できます。  
  
## Appleが将来のOSバージョンでSwiftを使ったPlatformのFrameworkを提供可能になる  
  
ブログに書かれていたのですが、具体的にどういうメリットがあるのかがわかっていません。  
  
FrameworkがSwiftで書かれることで  
Swiftで書かれたアプリとの連携が向上してパフォーマンスが良くなるということでしょうか:thinking:  
  
※   
ABI Stabilityは  
2019年4月現在 Appleプラットフォームで実現していますが  
LinuxやWindowsはまだのようです。  
  
https://swift.org/blog/abi-stability-and-more/  
https://swift.org/blog/abi-stability-and-apple/  
  
  
# Module Stability  
  
ABI Stabilityは実行時のSwiftのバージョン差異を吸収してくれる一方、  
Module Stabilityはコンパイル時のSwiftのバージョンの違いを吸収してくれます。  
  
Swiftでは「swiftmodule」というアーカイブフォーマットでライブラリのinterfaceを定義していますが、  
このファイルの内容が現在のコンパイラのバージョン(実質Swiftのバージョン)と密接に結びついているために  
別のバージョンでビルドされたライブラリをimportとしようとしてもコンパイラがエラーにしてしまいます。  
  
これによってバイナリフレームワークを提供しているライブラリで、  
バージョンに合わせてビルドされたライブラリを用意する必要が毎回あり  
クライアント側でも  
ライブラリが対応していないからSwiftのバージョンを上げることができない  
ということが起きていました。  
  
これをModule Stabilityが実現されると  
どのコンパイラのバージョンで実装されたかを気にする必要がなくなります。  
  
※ 実はObjective-CでWrapすればできるということをInstabugの方がブログで書かれていました。  
https://instabug.com/blog/swift-5-module-stability-workaround-for-binary-frameworks/  
  
詳細は下記のSwift Forumの中で話されています。  
https://forums.swift.org/t/plan-for-module-stability/14551  
  
  
# Library Evolution Support  
  
これはまだ最終決定がされていないものですが  
AppleのブログではすでにStandard Libraryなどで実装されているようです。  
  
Swift4.2で導入された@inlinableもこれの一つです。  
https://github.com/apple/swift-evolution/blob/master/proposals/0193-cross-module-inlining-and-specialization.md  
  
今まではSwift(コンパイラ)のバージョンが上がった時の話をしていましたが、  
今度はアプリで使用しているライブラリのバージョンが上がった時の話です。  
  
現在ですと  
ライブラリのバージョンが変わるたびに  
アプリの再コンパイルが必要になります。  
  
ブログによると  
この動きにはメリットがあり  
コンパイラがどのバージョンを使用しているのかがわかるので  
最適化ができると書いてあります。  
しかし  
ライブラリのバージョンが違うと  
この最適化が正しい最適化ではなくなる可能性があります。  
そのため再コンパイルが必要になってくるようです。  
  
Library Evolution Supportを達成すると  
この再コンパイルが不要になります。  
  
例えば  
あるライブラリの更新が  
他のライブラリの更新を引き起こすことがなくなります。  
  
AとBのライブラリのbinary frameworkを使用している。  
↓  
AはBのライブラリに依存している。  
↓  
Bのライブラリを上げる  
↓  
~~Aも再コンパイルしなければならなくなる~~  
  
# まとめ  
  
Swiftは表だけではなく  
裏側の仕組みもどんどん進化していっています。  
  
普段はあまり目に見えないことに関しても  
気がつかないところで恩恵を受けていたり  
逆に何か影響を受けているかもしれないので  
こういう裏側に仕組みを学ぶことも大事だなと思っています。  
  
これからもSwiftがどう進化していくのかを追っていくのが楽しみです:smiley:  
