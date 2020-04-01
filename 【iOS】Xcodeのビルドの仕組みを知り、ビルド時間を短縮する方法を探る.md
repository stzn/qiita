「Xcodeのビルドに時間がかかる！」  
「なんか急に遅くなった？」  
私はこう感じることがよくあります。  
  
かつては  
早く終わらないかなと思いつつ  
他の作業をしながらただただ待っていました。  
  
その後  
色々と調べて改善する方法を少しは知ることができましたが  
そもそもXcodeのビルドの仕組みを知らず  
根本的に何が問題になっているのかがわかっていませんでした。  
  
そこで  
今回は  
  
Xcodeがビルドの仕組みや  
ビルド時間を短縮するための方法などを  
いくつか見ていきたいと思います。  
  
  
# Xcodeのビルドの流れ  
  
Xcodeは下記の図のようにビルドを行います。  
  
![Untitled Diagram.png](/image/7d5f1714-a670-7512-5b80-8c8e4a572875.png)  
  
それぞれ何をしているのかを簡単に見ていきます。  
  
## PreProcessor  
  
- Compilerへプログラムを送り込めるように変換  
- マクロをそれぞれの定義へ置換  
- 依存関係の発見  
- PreProcessorのディレクティブ(#if DEBUGなど)を解決  
  
## Compiler  
  
- ソースコードをマシンコード(機械語)へ変換  
  
コンパイラはビルドの中でも大きな役割をしており  
下記の流れでSwiftのコードを変換していきます。  
  
![Untitled Diagram2.png](/image/91413d91-e483-1404-22fc-5f528c158b56.png)  
  
詳しくは下記をご参照ください。  
https://swift.org/compiler-stdlib/#compiler-architecture  
  
## Assembler  
  
人が読めるアセンブリコードを  
メモリ内のどの領域に配置しても実行できるマシンコード(機械語)に変換して  
コードとデータを集めたMach-Oファイルを作成  
  
※ Mach-Oは意味をなす単位で固まったバイトのストリームで  
iOSデバイスのARMプロセッサやMacのIntelプロセッサ上で動作します。  
  
## Linker  
  
iOSやmacOS上で動作するように  
様々なオブジェクトファイルとライブラリをまとめて  
ひとつのMach-O実行ファイルを作成します。  
  
下記のファイルを集めているようです。  
  
```
オブジェクトファイル + dylib, .a , .tbd => Single Executable file
```  
  
## Loader  
  
OSの機能の一部でプログラムをメモリに載せて実行します。  
  
- プログラム実行に必要なメモリ領域の確保  
- レジスターを初期状態に初期化  
- dylibsや他のDynamicライブラリーのロード  
  
このロード時間がアプリの起動時間に比例します。  
  
# ビルドの種類  
  
上記でビルドの流れを見てきましたが  
ビルドといっても2つの種類があります。  
  
- クリーンビルド  
- インクリメンタルビルド  
  
## クリーンビルド  
  
derived dataのようなキャッシュを全部クリーンしてからビルドを行います。  
  
## インクリメンタルビルド  
  
クリーンビルドした後の変更した部分だけのビルドを行います。  
  
# コンパイラでビルド時間を短縮するための2つの方法  
  
ここからは具体的にビルド時間を短縮するためにできることを  
見ていきます。  
  
特にコンパイラの影響が大きく  
Appleのドキュメントにも  
下記のコンパイラの要素が重要であるとしています。  
  
– **Whole Module**か**Primary-file**であるかどうか  
- Optimizing(-O, -Osize)かOnoneかどうか  
- インクリメンタルビルド時にどれだけ作業を削減できるかどうか  
- ファイル外の定義を参照する量をどれだけ抑えることができるか  
  
https://github.com/apple/swift/blob/master/docs/CompilerPerformance.md#summing-up-high-level-picture-of-compilation-performance  
  
そこでここからは  
コンパイラの仕組みを利用して  
ビルド時間を短縮していく方法を見ていきます。  
  
# 1. Build Settingsの設定  
  
Swiftのプログラムをコンパイルしたり実行する際  
`swift`または`swiftc`というプログラムを利用します。  
※ `swiftc`は`swift`のシンボリックリンクです。  
  
このプログラムでは多くの引数を指定することができ  
それによって振る舞いがかなり違います。  
  
Xcodeを使用している場合は  
Build Settingsの値を引数として利用します。  
  
その中でもビルド時間に影響するものとして  
  
- Compilation Mode  
- Optimization Mode  
  
の2つがあります。  
  
## Compilation Mode  
  
Xcodeでは  
Swift Compiler > Code Generation > Compilation Modeで設定できます。  
  
<img width="524" alt="スクリーンショット 2019-12-12 8.26.02.png" src="/image/122cc42e-1d62-9287-2ef9-a1d36a789781.png">  
  
Compilation Modeは  
**Driver**と**Frontend jobs**の動きを定義します。  
  
### Driver  
  
複数同時に実行される`swiftc`プロセスの中で  
一番最初に来るものを指します。  
  
どのファイルでコンパイルが必要になるのかを判断します。  
  
そして子プロセス(いわゆるジョブ)を起動して  
コンパイルやリンクを行います。  
  
https://github.com/apple/swift-driver  
  
### Frontend jobs  
  
Driverが起動させる`swift -frontend ...`で始まるプロセスです。  
swift-frontendを起動して下記のようなことを実行します。  
  
- コンパイル  
- PCHファイルの生成  
- モジュール間のマージ  
  
このプロセスたちが  
コンパイル時に大きな負荷になります。  
  
https://github.com/apple/swift/blob/master/include/swift/Frontend/Frontend.h  
  
  
## Compilation Modeの種類  
  
Compilation Modeには2種類あります。  
  
- Primary-file  
- Batch  
  
### Primary-file  
  
Driverは複数のFrontend jobsにタスクを分担して  
それぞれが出力した結果をマージします。  
Frontend jobsはモジュール内の全てのファイルを読み込み  
その中で自分がメインで担当するファイルを扱います。  
  
さらに  
Primary-fileには2つのサブモードがあります。  
- single-file  
- Batch(swift4.2で追加)  
  
#### single-file  
  
1ファイルにつき1Frontend jobが実行され  
それぞれのジョブで1つのメインファイルを扱います。  
  
#### Batch  
  
1CPUにつき1Frontend jobが実行され  
均等なサイズに分けられたモジュールの複数ファイルを  
メインファイルとして扱います。  
  
#### Primary-fileのメリット  
  
更新が必要なファイルのFrontend jobだけコンパイルすれば良いので  
インクリメンタルなコンパイルをすることに加えて  
CPUを並列に利用できるのでビルド時間が短縮できます。  
  
#### Primary-fileのデメリット  
  
一方で全てのFrontend jobsが全てのファイルを走査するため  
Frontend jobsの数がコンパイルの時間にかなり影響していきます。  
  
実際にはサイズも速度も問題がないようですが  
可能性として数が爆発的に増加してしまうリスクがあります。  
  
### whole-module  
  
**1つの**Frontend jobが  
全体のモジュールをコンパイルします。  
  
全てのモジュールを1つにまとめ  
一度に全てをコンパイルします。  
  
#### whole-moduleのメリット  
  
モジュール全体で確実に最適化をすることが可能です。  
例えばデッドコードを削除することで  
後々に不要なコンパイル作業を減らすことができます。  
  
またのPrimary-fileように  
部分的な結果をまとめる二次的な作業も行う必要がなく  
ビルド時間が少なくなる場合もあります。  
  
#### whole-moduleのデメリット  
  
毎回全体のリビルドが必要になります。  
  
さらにビルド時間は短くなる可能性はあるものの  
そこまで大きなメリットはなく  
Inrementalビルドのメリットも得られなくなるため  
あまり推奨される設定ではないようです。  
  
https://github.com/apple/swift/blob/master/docs/OptimizationTips.rst#whole-module-optimizations  
  
## Optimization Mode  
  
コンパイラがどのくらい最適化(バイトサイズ削減や実行スピードの短縮)を行うのかを指定します。  
  
Swift Compiler > Code Generation > Optimization Levelで設定できます。  
  
<img width="553" alt="スクリーンショット 2019-12-12 8.27.04.png" src="/image/1f1681fe-6c8e-7218-bf25-4b30125172be.png">  
  
### 3つのOptimization Mode  
  
Optimization Modeには  
下記の3つがあります。  
  
#### -Onone  
  
最適化はしません(※は行われます)  
  
#### -Osize  
  
パフォーマンスよりも  
バイトサイズに対する最適化を優先して行います。  
  
#### -O  
  
かなりアグレッシブにコンパイラが最適化をかけ  
デバッグ情報は圧縮されて出力されます。  
  
https://github.com/apple/swift/blob/master/docs/OptimizationTips.rst#enabling-optimizations  
  
### コンパイラが扱うSwiftコードの3つの形  
  
Swiftコンパイラは  
主に3つの形に変換してSwiftコードを扱います。  
  
- ASTs  
- SIL  
- LLVM IR  
  
詳細はこちらをご参照ください  
https://github.com/apple/swift/blob/master/docs/CompilerPerformance.md#amount-of-optimization  
  
#### SILとLLVMと最適化  
  
3つの中でもSILとLLVMを生成する段階では  
Optimization ModeがOn(-Osize, -O)になっていると  
コンパイルが最適化を行うため  
ビルド時間やメモリの使用量も増えます。  
  
※  
Off(-Onone)になっていたとしても  
SILとLLVM IRでは最適化が行われますが  
Onの時よりは最適化の程度は低くなります。  
  
さらにLLVMでは  
Front jobの中で並列処理を行うことができます。  
  
## どんなCompilation ModeとOptimization Modeの設定が良いのか？  
  
Appleのドキュメントによると  
  
- Debug => `Primary-file` + `-Onone`  
- Release => `whole-module` + `-O`  
  
という設定が記載されています。  
  
## 型推論にかかる時間を計測するフラグの設定  
  
型推論はSwiftの大きな特徴ですが  
型推論が働けば働くほどビルド時間は長くなります。  
  
そこでどのくらい時間がかかっているのかを知るために  
  
**Swift Compiler > Custom Flags > Other Swift Flags**  
  
に下記のようなフラグを設定します。  
  
```
-Xfrontend -warn-long-expression-type-checking=100
```  
  
これはビルド時に型推論に100ms以上かかったものへwarningを出してくれます。  
  
<img width="698" alt="スクリーンショット 2019-12-12 8.52.45.png" src="/image/32bf40d5-a14e-0187-2073-5969416a454e.png">  
  
こういう時間のかかる型は  
分割して型をシンプルにすることで  
ビルド時間を短縮することができます。  
  
※   
ただし  
このフラグによる計測自体が  
ビルド時間を長くする要因にもなるため  
不要な場合は外しておいた方が良いかもしれません。  
  
  
## dSYMファイルをDebug時に生成しないようにする  
  
dSYM(debug symbols file)はコンパイルごとに生成される  
デバッグ情報が含まれたファイルです。  
これはCrashファイルを解析するためのファイルのため  
Debug時に必要になることはほとんどないかと思われます。  
  
そこで  
Debug時には生成しないように設定を変更できます。  
  
Build Options > Debug Information Format  
  
<img width="564" alt="スクリーンショット 2019-12-12 9.04.09.png" src="/image/47f182b2-76b6-645b-b211-6888e95b7a2f.png">  
  
**DWARF**にすると生成しません。  
  
## Run Script PhaseのInput/Output Fileを使用する  
  
Incrementalビルドの場合  
Input(Output) Filesを利用すると  
毎回同じScriptを実行しなくなります。  
  
ただし  
Input Filesに変更を加えると  
Scriptは実行されます。  
  
**Xcode Help > Run a shell script**  
に下記の記載がございます。  
  
```
For incremental builds, 
the inputs and outputs determine if the shell script is run for each build 
after the initial clean build. 
If you don’t specify inputs and outputs, or you change an input file, 
the shell script is run. 
If you don’t want a script to run during each build, 
don’t change the inputs between builds.
```  
  
<img width="971" alt="スクリーンショット 2019-12-12 9.07.17.png" src="/image/55313f50-f571-84ab-d054-8ebe7ab71a97.png">  
  
※  
余談ですが  
Use discovered dependency fileはXcode11の新しい機能で  
MakeFile形式の.dファイルの依存も参照できるようになったようです。  
  
https://developer.apple.com/documentation/xcode_release_notes/xcode_11_release_notes?preferredLanguage=occ#3318304  
  
# 2. コンパイラの作業量を減らす  
  
Building Settingsは  
一度の設定でビルド時間の短縮を目指しました。  
  
しかし  
日々の開発の中で  
ちょっとコードを追加しただけでも  
急にビルド時間が長くなることがあります。  
  
ここからは  
日々の開発の中でもビルド時間をなるべく短縮するための  
方法を見ていきます。  
  
## コンパイラは頑張っている  
  
コンパイラはできる限り作業量を抑えるためにも  
コンパイルするべきファイルを一定のルールに従って判断します。  
(詳細は後ほど紹介します。)  
  
一方でコードの記述方法によっては  
コンパイラが正しい判断ができず  
作業量を増やしてしまう場合もあります。  
  
そういうことが起こり得そうな状況として  
下記の2つがあります。  
  
## 1. インクリメンタルなコンパイル(Incremental compilation)を行っているとき  
  
上記でも記載しましたが  
Compilation Modeが**Primary-file**の時に  
DriverはIncremental Modeとして  
全てのFrontend jobsでコンパイルをしないようにします。(インクリメンタルビルド)  
  
そして  
どのファイルにコンパイルが必要かどうかの判断は  
ファイル同士の依存関係を要約した  
Dependency Graph(依存関係グラフ)を  
コンパイラが作成する必要があるかに関係します。  
  
コンパイラは  
変更があったファイルと依存しているファイルのみを  
コンパイルすることで  
必要最低限の依存関係グラフの再作成を行います。  
  
## 2. 外部参照の遅延解決(Lazy resolution)  
  
Swiftのソースコードファイルには  
同じモジュール内の別ファイルや  
外部のモジュールなどを参照した名前を含んでいますが  
この参照は2つのかなり異なる場所のモジュールから解決されます。  
  
- Clang Importerが提供するC/ObjCモジュール   
- シリアライズされたSwiftモジュール  
  
これらは全然異なる種類ではありますが  
どちらの種類のモジュールも  
インデックス化されたバイナリー形式のフォーマットになっているため  
コンパイラ内部では同じ方法で参照を解決します。  
  
このことによって  
クラスや関数などの名前単位で参照ができることができ  
モジュール全体を読む必要がありません。  
  
さらにバイナリ形式であることで  
読み込みスピードもかなり速いです。  
  
そのため  
コンパイラはこの外部参照の解決を  
最小限にするために  
モジュールから定義を読み込もうとします。  
  
## コンパイラの3つのルール  
  
上記の2つの過程で  
コンパイラが効率よく働いてもらうために  
何ができるのかを見ていきます。  
  
そこでまず  
コンパイラが  
コンパイルする必要があると判断するための  
3つのルールを見ていきます。  
  
### ルール1  
  
```
関数の内部を変更してもコンパイルは発生しない
```  
  
これはSwiftは型安全な言語なので  
型に変更がなければ依存関係に変更もないからです。  
  
### ルール2  
  
```
あるファイルに新しい関数やstructやextensionを追加した場合
このファイルに依存関係があるファイル全てにコンパイルが発生する
```  
  
これはSwiftはファイル名に関係なく  
structやclassなども自由にどこにでも書けます。  
そのため改めて全ての依存関係を知る必要があります。  
  
### ルール3  
  
```
frameworkを利用したアプリの場合
依存しているframeworkに変更があると
アプリ内の全てのファイルにコンパイルが発生する
```  
  
これは  
アプリ内のどこで使用されているのかがわからないため  
全てのファイルを走査して依存関係グラフを再作成する必要が出てきます。  
  
  
## コンパイルの発生を抑える方法  
  
では  
このルールを利用して  
コンパイルの発生を抑えるためにできることを  
考えていきたいと思います。  
  
例えば  
下記のように複数のExtensionを一つにまとめたファイルがあります  
  
```AllExtensions.swift

extension String {
    func useStringExtension() {
        print(self)
    }
}

extension Int {
    func useIntExtension() {
        print(self)
    }
}

extension Double {
    func useDoubleExtension() {
        print(self)
    }
}
```  
  
このextensionをそれぞれ別のファイルで参照します。  
  
```UseStringExtension.swift

func useStringExtension() {
    "hoge".useStringExtension()
}
```  
  
```UseIntExtension.swift

func useIntExtension() {
    1.useIntExtension()
}
```  
  
```UseDoubleExtension.swift

func useDoubleExtension() {
    0.1.useDoubleExtension()
}
```  
  
このときAllExtensions.swiftに  
新しいextensionを追加してみると  
参照している全てのファイルがコンパイルされます。  
  
<img width="270" alt="スクリーンショット 2019-12-13 9.02.28.png" src="/image/427ebfa4-414e-00f6-90f8-ad2970c67518.png">  
  
では、Extensionを別のファイルに分けてみます。  
  
```StringExtension.swift

extension String {
    func useStringExtension() {
        print(self)
    }
}
```  
  
すると参照しているファイルにしか  
コンパイルは発生しません。  
  
<img width="267" alt="スクリーンショット 2019-12-13 9.03.20.png" src="/image/407f4233-5095-17be-a7df-ca8e57d2b37f.png">  
  
つまり  
Extensionを集めたファイルや  
共通で使用する関数を集めたUtilityファイルを作成すると  
コンパイルが発生した時のビルド時間が長くなる可能性があります。  
  
新しいコードを追加したときのビルド時間が気になる場合は  
こういう点に注目してみるのもよいかもしれません。  
  
# まとめ  
  
Xcodeのビルドの流れと  
ビルド時間をどう短縮できるかについて見てきました。  
  
こういった内部の仕組みはとても奥が深く  
なかなか理解することが難しくもありますが  
  
ちょっと理解をしておくと  
今後ビルド時間が遅くなった時も  
原因を見つけるスピードが上がることで  
開発効率上げていくことができることがあるかもしれませんね😄  
  
もし何か間違いなどございましたら  
教えていただけると嬉しいです🙇🏻‍♂️  
  
  
# 参考記事  
  
https://swift.org/compiler-stdlib/#compiler-architecture  
https://github.com/apple/swift/blob/master/docs/CompilerPerformance.md  
https://github.com/apple/swift/blob/master/docs/OptimizationTips.rst  
https://developer.apple.com/videos/play/wwdc2018/408/  
https://bytes.swiggy.com/advanced-techniques-to-speed-up-the-compile-time-in-xcode-27819cb3be59  
http://ikesyo.hatenablog.com/entry/2018/09/27/142826  
