# 経緯  
  
仕事の関係で  
トリミングする枠を変形させて画像をトリミングできる画面や  
グラデーションで色が変化していくプログレスバーなどを作成する機会をいただき  
今まで馴染みのなかったCore Animationを使用することになりました。  
  
正直、CA〇〇とかCG〇〇という名前を見ると  
「うっ」となってしまい苦手意識を持っていました。  
  
あまり知識がないことから何が起きているのかがわからず  
不思議な挙動に出会うと解決するのに  
時間がかかるのが非常に辛く感じていました。  
(今もそうですが。。。)  
  
ただ  
今回調べたことでちょっと仲良くなれたのかなと感じており  
せっかくの機会ですので調べたことをまとめてみたいと思います。  
  
# Core Animationとは？  
  
Appleのドキュメントによると  
  
**Viewやその他の画面に表示されるような要素を  
アニメーションさせるために使われる  
グラフィックの描画やアニメーションのインフラストラクチャ**  
  
とあります。  
  
[Core Animation Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CoreAnimation_guide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40004514-CH1-SW1)  
※ドキュメントにも記載がありますが、このドキュメントは古いものとなっております  
  
つまり、名前にもあるようにViewをアニメーションさせるためのフレームワークです。  
  
UIKitの下にあり、QuartzCore.frameworkに含まれています。  
  
Core Animationでアニメーション可能なプロパティを操作することで  
デフォルトでアニメーションを含めて変化したり  
独自にアニメーションを定義して細かくアニメーションできるようになります。  
  
## UIKitとの違い  
  
UIViewは画面に表示するオブジェクトを直接管理し、  
画面のレイアウトやユーザーからのタッチイベントなどを色々なことを行っています。  
  
Core Animationは  
ハードウェア上でアプリのコンテンツを操作したり構成するために使用されます。  
  
中心的な存在として**layerオブジェクト**あります。  
アプリのデータを管理するいわゆる**モデルオブジェクト**です。  
  
layerは情報を**bitmap**形式で管理をしており、  
同様にbitmap形式のデータを必要するGPUから簡単に操作できるようにします。  
  
UIViewの描画はメインスレッドかつCPUを用いて描画を行います。  
  
一方でGPUはメインスレッドをブロックすることなく  
描画に特化しているため軽量で高速です。  
  
  
よく使われるものとして**CALayer**があり、これはViewに関する情報を管理します。  
  
## CALayer  
  
iOSに関していえばUIViewは必ずlayerというプロパティにCALayerを持っています。  
UIViewは画面の描画やアニメーションの実行も担っているように見えますが、  
実は内部でこのCALayerへ処理を委譲しています。  
  
例えば、boundsの設定を行うと、UIViewはlayerプロパティに設定されているCALayerへ処理を渡します。  
  
![ROPPONGI2.001.png](/image/7e24680b-a96e-d12b-e0a5-6b22de27fffa.png)  
  
このケースの場合  
ここで知らないと「あれ？」と思うことがあります。  
上記でCore Animationは「Viewをアニメーションさせる」と言いましたが  
このケースではアニメーションはしません。  
  
なぜか？と言いますと、  
UIViewはlayerの[CALayerDelegate](https://developer.apple.com/documentation/quartzcore/calayerdelegate  
)を実装しており、  
[action(for:forKey:)](https://developer.apple.com/documentation/quartzcore/calayerdelegate/2097264-action)というメソッドで  
アニメーション可能な値を探索します。  
  
  
その際にNSNullを返すことで  
探索するのを止めることが可能になりますが  
UIViewはここで全てNSNullを返すようにしているため  
アニメーションが起きません。  
  
CALayerはUIViewと同様に階層構造を持ち、  
画面の初期表示時はUIViewと同じ階層構造を持ちます。  
一方で独立した階層を作成することもできます。  
  
![sublayer_hierarchy_2x.png](/image/6b7b6cca-cc3a-b549-0aaa-d0e47ec532c6.png)  
  
  
### CALayerのサブクラスを返すことも可能  
  
デフォルトではCALayerを返しますが、layerClassをoverrideすることで、サブクラス返すことも可能です。  
  
```swift

override class var layerClass : AnyClass {
    return CATiledLayer.self
}
```  
  
https://developer.apple.com/documentation/uikit/uiview/1622626-layerclass  
  
## CALayerの3つのレイヤー  
  
CALayerは3つのレイヤに分かれています。  
  
### **モデルレイヤ**  
  
描画やアニメーションに必要な情報を管理しており、私たちがアクセスするプロパティのほとんどはここで管理されています。  
アニメーション後の最終的な値もここに設定することでアニメーション後の状態を保つことができます。  
  
### **プレゼンテーションレイヤ**  
  
上記のモデルデータと反対に、アニメーション中の現在値を保持します。  
アニメーション中のその時点の値を参照したい場合はここから値を取得する必要があります。  
アニメーションの一時停止、再開をしたいときなどに活用できます。  
  
### **描画レイヤ**  
  
プレゼンテーションレイヤとは別スレッドで実際の描画を行います。  
これはCore AnimationのPrivateクラスなので直接操作することはできません。  
  
  
![sublayer_hierarchies_2x.png](/image/33f8a520-9f7a-1469-2aeb-f76aeaca0e01.png)  
  
  
  
## UIViewとCore Animationの処理の流れ  
  
今までのことを図にしてみると、このような感じになります。  
  
![ROPPONGI2.001.png](/image/399dd5be-0db5-1082-f610-ba4d25c53b3a.png)  
  
  
## Core Animationが必要なとき  
  
**基本的にはUIViewで良い**と思っています。  
UIViewの方が簡単に実装ができます。  
  
しかし、独自の形の図形を描画したかったり、  
よい細かいアニメーションを実現したい場合は、Core Animationが力を発揮します。  
  
また、パフォーマンスに問題がある場合に、より軽量で高速なCore Animationを使用することで  
問題が解決することもあるかもしれません、  
  
また、コンテンツをキャッシュするという特徴を生かして、  
複数の箇所で使用するような静的な画像などは、  
複数のLayerで同じ画像データを参照するためメモリの使用量を抑えることができます。  
  
## Contentsの提供方法  
  
3つの方法があります。  
  
### contentsプロパティにCGImageRef型のデータを設定する  
  
通常の画像などを提供する場合に使用します。  
一点注意点としてcontentsScaleプロパティに適切な値を設定する必要があります。  
これはRetinaディスプレイへの対応で正しい値が設定されていないと表示される画像の大きさが変わります。  
直接値を設定するというよりも端末のscaleに合わせる記述方法が多く見られます。  
  
```swift

sublayer.contentsScale = UIScreen.main.scale
```  
  
### デリゲートメソッドを使用する  
  
[draw(_:)](https://developer.apple.com/documentation/quartzcore/calayerdelegate/2097262-draw)か[display(_:)](https://developer.apple.com/documentation/quartzcore/calayerdelegate/2097261-display)を使用します。  
  
下記はdisplayメソッドの例です。  
contentsに画像データなどを設定する場合に使用します。  
  
```swift

class DemoView: UIView {
    
    /*
     !!!!ハマりポイント!!!!
     */
    override func draw(_ rect: CGRect) {
        super.draw(rect)
    }
    
    // 画像などのデータを提供する
    override func display(_ layer: CALayer) {
        layer.contentsScale = UIScreen.main.scale
        layer.contents = UIImage(named: "test2.png")?.cgImage
    }
}
```  
  
上記で「ハマりポイント」と示しましたが  
layerに独自のコンテンツを表示したい場合は、  
drawメソッドをオーバーライドする必要があります。  
そうすると自動的に、対応するレイヤのデリゲートを生成し、  
必要なメソッドを実装されます。  
逆にここをコメントアウトすると何も描画されなくなります。  
  
  
下記はdrawメソッドの例です。  
動的に図形などを描画したい場合に使用します。  
  
```swift

class DemoView: UIView {
    
    /*
     !!!!はまりポイント!!!!
     自動レイヤつきビューに独自のコンテンツを表示したい場合は、
     当該ビューの描画メソッドをオーバーライドする。
     ビューは自動的に、対応するレイヤのデリゲートを生成し、
     必要なメソッドを実装する。
     したがって、コンテンツを描画するためには、ビューのdrawRect:メソッドを実装する。
     */
    override func draw(_ rect: CGRect) {
        super.draw(rect)
    }

    // CGContextに書き込む    
    override func draw(_ layer: CALayer, in ctx: CGContext) {
        super.draw(layer, in: ctx)

        ctx.move(to: CGPoint(x: 0.0, y: 0.0))
        ctx.addLine(to: CGPoint(x: 0.0, y: self.frame.size.height))
        ctx.addLine(to: CGPoint(x: self.frame.size.width, y: self.frame.size.height))
        ctx.addLine(to: CGPoint(x: self.frame.size.width, y: 0.0))
        ctx.closePath()

        ctx.setFillColor(UIColor.red.cgColor)
        ctx.fillPath()
    }
}
```  
  
ちなみに、両方のメソッドを実装した場合は、displayメソッドが優先されます。  
さらに、初期化時contentsに画像を設定した場合は  
これが一番優先されます。  
  
  
### サブクラスを作成する  
  
CALayerのサブクラスを作成し、layerClassをoverrideすることもできます。  
こちらも同様に[display(_:)](https://developer.apple.com/documentation/quartzcore/calayerdelegate/2097262-draw)か[display(_:)](https://developer.apple.com/documentation/quartzcore/calayerdelegate/2097261-display)を使用します。  
  
  
displayの例  
  
```swift

// サブクラス
class DemoLayer: CALayer {
    
    // contentsにプロパティを設定
    override func display() {
        super.display()
        contents = UIImage(named: "test3.png")?.cgImage
    }
}

class DemoView: UIView {
    
    // サブクラス化
    override class var layerClass : AnyClass {
        return DemoLayer.self
    }
    
    override func draw(_ rect: CGRect) {
        super.draw(rect)
    }
}
```  
  
drawの例  
  
```swift

// サブクラス
class DemoLayer: CALayer {
    // drawメソッドで動的に描画する
    // contextに描きこんでいく
    override func draw(in ctx: CGContext) {
        super.draw(in: ctx)
        ctx.move(to: CGPoint(x: 0.0, y: 0.0))
        ctx.addLine(to: CGPoint(x: 0.0, y: self.frame.size.height))
        ctx.addLine(to: CGPoint(x: self.frame.size.width, y: self.frame.size.height))
        ctx.addLine(to: CGPoint(x: self.frame.size.width, y: 0.0))
        ctx.closePath()

        ctx.setFillColor(UIColor.red.cgColor)
        ctx.fillPath()
    }
}

class DemoView: UIView {
    
    // サブクラス化
    override class var layerClass : AnyClass {
        return DemoLayer.self
    }
    
    override func draw(_ rect: CGRect) {
        super.draw(rect)
    }
    
}
```  
  
# CAShapeLayer  
  
CALayerのサブクラスはたくさん用意されており  
それぞれが特徴的な機能を持っています。  
  
その中でもCAShapeLayerは使われることは多いです。  
  
## CAShapeLayerの特徴  
  
### Vector形式の図形を描画する  
  
Vector形式の画像は解像度に依らないという特徴があり  
Core Animationが自動で最適なbitmapデータを作成してくれます。  
  
### アニメーション可能なプロパティが豊富  
  
Core Animationではデフォルトでアニメーションが定義されていたり  
独自にアニメーションを定義することができるため  
アプリの表現力を高めることができます。  
  
アニメーションに関しては後ほどご紹介させていただきます。  
  
## pathに図形を設定する  
  
pathプロパティにCGPathを設定することで図形の描画などが可能です。  
CGPathは名前の通り、CoreGraphicsに含まれるクラスです。  
Core GraphicはCore Animationよりも  
より低レイヤーにあるフレームワークです。  
  
古いフレームワークで  
iPhoneがまだシングルコアで動いてたころからのものでもあり  
メインスレッドでCPUベースで動きます。  
  
そのため  
古い端末などでは描画に時間がかかって動きがカクカクすることもあります。  
  
これをCore AnimationでGPUで動くCAShapeLayerで使うことで  
よりスムーズな描画を実現することも可能になります。  
  
## layerとして使用する  
  
以下に例を示します。  
  
```swift

class DemoView: UIView {
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        
        backgroundColor = .darkGray
        
        makeShapeLayer()
    }
    
    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
    }
    
    // 使う必要はない！！
//    override func draw(_ rect: CGRect) {
//        super.draw(rect)
//    }
    
    func makeShapeLayer() {
        
        let path = UIBezierPath()
        path.move(to: CGPoint(x: self.frame.width/2, y: 0.0))
        path.addLine(to: CGPoint(x: 0.0, y: self.frame.size.height))
        path.addLine(to: CGPoint(x: self.frame.size.width, y: self.frame.size.height))
        path.close()
        
        // pathの設定は無視される
        UIColor.red.setFill()
        path.fill()
        
        UIColor.blue.setStroke()
        path.stroke()
        
        let shapeLayer = CAShapeLayer()
        shapeLayer.path = path.cgPath
        
        // 設定しないと黒
        shapeLayer.fillColor = UIColor.orange.cgColor
        shapeLayer.strokeColor = UIColor.brown.cgColor
        shapeLayer.lineWidth = 3.0
        
        layer.addSublayer(shapeLayer)
    }
}
```  
  
1. CAShapeLayerオブジェクトを生成  
2. UIBezierPathオブジェクトを生成  
3. UIBezierPathで描きたい図形を設定する  
4. CAShapeLayerのcgPathにUIBezierPathを設定する  
  
上記ではlayerにサブレイヤとして追加していますが、  
UIViewのlayerに設定する方法も可能です。  
  
コメントでも記載していますが  
drawメソッドをoverrideする必要はありません。  
  
またUIBezierPathで設定した色などは無視され、  
CAShapeLayerで設定しないとデフォルトの黒が設定されます。  
  
  
下記に描画結果を示します。  
  
![スクリーンショット 2018-12-21 5.24.28.png](/image/1d2b1a81-9c16-1556-7656-1f9380a3610d.png)  
  
  
## maskとして使用する  
  
単純に図形を描画するだけではなく  
マスクとしても活用できます。  
  
マスクをするというのは  
Layerで描画された図形以外の部分を透明なピクセルのブロックで覆うことで  
このLayerの下のLayerもその図形内のみが表示されるようにすることです。  
※ 調査した結果を記載しましたが、間違っていたらご指摘ください🙇🏻‍♂️  
  
```swift

class DemoView: UIView {
    
    var path: UIBezierPath!
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        
        // 色はこちらで
        backgroundColor = .darkGray
        
        makeShapeLayer()
    }
    
    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
    }
    
    func makeShapeLayer() {
        self.makeTriangle()
        
        let shapeLayer = CAShapeLayer()
        shapeLayer.path = path.cgPath
        
        // 色は反映されない
        shapeLayer.fillColor = UIColor.yellow.cgColor
        shapeLayer.strokeColor = UIColor.brown.cgColor
        shapeLayer.lineWidth = 3.0
        
        //layer.addSublayer(shapeLayer)
        // Layerの形に切り抜かれる
        layer.mask = shapeLayer
    }
    
    func makeTriangle() {
        path = UIBezierPath()
        path.move(to: CGPoint(x: self.frame.width/2, y: 0.0))
        path.addLine(to: CGPoint(x: 0.0, y: self.frame.size.height))
        path.addLine(to: CGPoint(x: self.frame.size.width, y: self.frame.size.height))
        path.close()
    }
}
```  
  
下記が表示例です。  
  
![スクリーンショット 2018-12-21 5.25.25.png](/image/a79f6c7b-a4ae-a329-858d-56bb745be101.png)  
  
  
まず図形以外の箇所は透明なブロックで覆われたため背景色が見えなくなりました。  
また、shapeLayerで色を設定していますが、色は反映されません。  
色を設定したい場合はbackgroundColorで設定します。  
  
## 複数設定が可能  
  
CAShapeLayerは複数追加することが可能になります。  
  
```swift

class DemoView: UIView {
    
    var path: UIBezierPath!
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        
        backgroundColor = .darkGray
        twoShapes()
    }
    
    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
    }
    
    func twoShapes() {
        let width: CGFloat = self.frame.size.width/2
        let height: CGFloat = self.frame.size.height/2
        
        let path1 = UIBezierPath()
        path1.move(to: CGPoint(x: width/2, y: 0.0))
        path1.addLine(to: CGPoint(x: 0.0, y: height))
        path1.addLine(to: CGPoint(x: width, y: height))
        path1.close()
        
        let path2 = UIBezierPath()
        path2.move(to: CGPoint(x: width/2, y: height))
        path2.addLine(to: CGPoint(x: 0.0, y: 0.0))
        path2.addLine(to: CGPoint(x: width, y: 0.0))
        path2.close()
        
        let shapeLayer1 = CAShapeLayer()
        shapeLayer1.path = path1.cgPath
        shapeLayer1.fillColor = UIColor.yellow.cgColor
        shapeLayer1.backgroundColor = UIColor.magenta.cgColor

        let shapeLayer2 = CAShapeLayer()
        shapeLayer2.path = path2.cgPath
        shapeLayer2.fillColor = UIColor.green.cgColor
        
        self.layer.addSublayer(shapeLayer1)
        self.layer.addSublayer(shapeLayer2)
    }    
}
```  
  
出力結果は下記になります。  
  
![スクリーンショット 2018-12-21 5.31.57.png](/image/f6b3043f-2e02-7f9a-bc86-259abe7f827c.png)  
  
図形が被ってしまいました。  
どちらも原点からの描画を行っているからです。  
描画の位置を変えて描画するということも可能ですが  
より簡単な方法としてpositionプロパティを使用します。  
  
```swift

class DemoView: UIView {
    
    var path: UIBezierPath!
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        
        backgroundColor = .darkGray
        twoShapes()
    }
    
    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
    }
    
    func twoShapes() {
        let width: CGFloat = self.frame.size.width/2
        let height: CGFloat = self.frame.size.height/2
        
        let path1 = UIBezierPath()
        path1.move(to: CGPoint(x: width/2, y: 0.0))
        path1.addLine(to: CGPoint(x: 0.0, y: height))
        path1.addLine(to: CGPoint(x: width, y: height))
        path1.close()
        
        let path2 = UIBezierPath()
        path2.move(to: CGPoint(x: width/2, y: height))
        path2.addLine(to: CGPoint(x: 0.0, y: 0.0))
        path2.addLine(to: CGPoint(x: width, y: 0.0))
        path2.close()
        
        let shapeLayer1 = CAShapeLayer()
        shapeLayer1.path = path1.cgPath
        shapeLayer1.fillColor = UIColor.yellow.cgColor
        shapeLayer1.backgroundColor = UIColor.magenta.cgColor

        let shapeLayer2 = CAShapeLayer()
        shapeLayer2.path = path2.cgPath
        shapeLayer2.fillColor = UIColor.green.cgColor
        
        self.layer.addSublayer(shapeLayer1)
        self.layer.addSublayer(shapeLayer2)
        
        // 2つのpositionの調整
        shapeLayer2.position = CGPoint(x: width/2, y: height)
        shapeLayer1.position = CGPoint(x: width/2, y: 0.0)
    }    
}
```  
  
![スクリーンショット 2018-12-21 5.35.03.png](/image/613402dd-b131-62ba-c17f-b6ecc95c6bfa.png)  
  
  
  
ここで一つハマりポイントがあります。  
  
このpositionで設定する値とboundsのoriginで設定する値の方向が逆になります。  
  
どういうことかと言いますと下記の例をご覧ください。  
  
```swift

class DemoView: UIView {
    
    var path: UIBezierPath!
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        
        backgroundColor = .darkGray
        twoShapes()
    }
    
    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
    }
    
    func twoShapes() {
        let width: CGFloat = self.frame.size.width/2
        let height: CGFloat = self.frame.size.height/2
        
        let path1 = UIBezierPath()
        path1.move(to: CGPoint(x: width/2, y: 0.0))
        path1.addLine(to: CGPoint(x: 0.0, y: height))
        path1.addLine(to: CGPoint(x: width, y: height))
        path1.close()
        
        let path2 = UIBezierPath()
        path2.move(to: CGPoint(x: width/2, y: height))
        path2.addLine(to: CGPoint(x: 0.0, y: 0.0))
        path2.addLine(to: CGPoint(x: width, y: 0.0))
        path2.close()
        
        let shapeLayer1 = CAShapeLayer()
        shapeLayer1.path = path1.cgPath
        shapeLayer1.fillColor = UIColor.yellow.cgColor
        shapeLayer1.backgroundColor = UIColor.magenta.cgColor

        let shapeLayer2 = CAShapeLayer()
        shapeLayer2.path = path2.cgPath
        shapeLayer2.fillColor = UIColor.green.cgColor
        
        self.layer.addSublayer(shapeLayer1)
        self.layer.addSublayer(shapeLayer2)
        
        // 2つのpositionの調整が必要になる
        shapeLayer2.position = CGPoint(x: width/2, y: height)
        shapeLayer1.position = CGPoint(x: width/2, y: 0.0)
        
        // 右下に移動する(通常とは逆になる)
        shapeLayer1.bounds.origin = CGPoint(x: -20.0, y: -40.0)        
    }    
}
```  
  
![スクリーンショット 2018-12-21 5.44.36.png](/image/723c1203-0349-d601-32d6-a9690124dabe.png)  
  
  
見てわかるようにマイナス値を設定すると右下に移動します。  
これはiOSの世界では逆の動きにあっており、結構違和感を感じるポイントではないかと思います。  
※ 理由までは調べ切れませんでした。もしご存知の方いらっしゃいましたら教えてください🙇🏻‍♂️  
  
  
### もっと複雑な図形も可能  
  
今までは三角や四角といった単純な図形を描画してきましたが、  
UIBezierPathを使えばどんな形の図形でも作成することができます。  
  
```swift

class DemoView: UIView {
    
    var path: UIBezierPath!
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        
        backgroundColor = .darkGray
        complexShape()
    }
    
    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
    }
    
    func complexShape() {
        path = UIBezierPath()
        path.move(to: CGPoint(x: 0.0, y: 0.0))
        path.addLine(to: CGPoint(x: self.frame.size.width/2 - 50.0, y: 0.0))
        path.addArc(withCenter: CGPoint(x: self.frame.size.width/2 - 25.0, y: 25.0),
                    radius: 25.0,
                    startAngle: CGFloat(180.0).toRadians(),
                    endAngle: CGFloat(0.0).toRadians(),
                    clockwise: false)
        path.addLine(to: CGPoint(x: self.frame.size.width/2, y: 0.0))
        path.addLine(to: CGPoint(x: self.frame.size.width - 50.0, y: 0.0))
        path.addCurve(to: CGPoint(x: self.frame.size.width, y: 200.0),
                      controlPoint1: CGPoint(x: self.frame.size.width + 100.0, y: 25.0),
                      controlPoint2: CGPoint(x: self.frame.size.width - 150.0, y: 50.0))
        path.addLine(to: CGPoint(x: self.frame.size.width, y: self.frame.size.height))
        path.addCurve(to: CGPoint(x: self.frame.size.width - 100.0, y: self.frame.size.height),
                      controlPoint1: CGPoint(
                        x: self.frame.size.width/2 - 50.0,
                        y: self.frame.size.height - 150.0),
                      controlPoint2: CGPoint(
                        x: self.frame.size.width/2 - 100.0,
                        y: self.frame.size.height - 100))
        path.addLine(to: CGPoint(x: 0.0, y: self.frame.size.height))
        path.close()
        
        let shapeLayer = CAShapeLayer()
        shapeLayer.path = path.cgPath
        
        self.backgroundColor = UIColor.orange
        self.layer.mask = shapeLayer
    }   
}
```  
  
![スクリーンショット 2018-12-21 5.47.38.png](/image/b0c99e18-7d06-5fce-4fca-bbe710d0ef40.png)  
  
  
UIBezierPathについては今回割愛させていただきますが、  
[Paint Code](https://www.paintcodeapp.com/)というツールを活用すると  
自動でpathを作成してくれるのでそれを参考に図形の作成をすることもできます。  
  
# アニメーション  
  
冒頭でも紹介しましたが、  
Core AnimationはViewをアニメーションさせるために使われます。  
  
多くのプロパティは設定するとデフォルトのアニメーションに合わせて値が反映されます。  
  
CAAnimationという抽象クラスのサブクラスを使用することで  
独自のアニメーションを行うことが可能です。  
  
https://developer.apple.com/documentation/quartzcore/caanimation  
  
  
アニメーション可能なプロパティは下記から確認できます。  
https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW1  
  
## デフォルトのアニメーション  
  
下記の例ではopacityを変更しています。  
  
```swift

class DemoView: UIView {
    
    var path: UIBezierPath!
    var shapeLayer: CAShapeLayer!
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        
        backgroundColor = .darkGray
        
        makeShapeLayer()
    }
    
    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
    }
    
    func makeShapeLayer() {
        self.makeTriangle()
        
        shapeLayer = CAShapeLayer()
        shapeLayer.path = path.cgPath
        
        layer.addSublayer(shapeLayer)
    }
    
    func makeTriangle() {
        path = UIBezierPath()
        path.move(to: CGPoint(x: self.frame.width/2, y: 0.0))
        path.addLine(to: CGPoint(x: 0.0, y: self.frame.size.height))
        path.addLine(to: CGPoint(x: self.frame.size.width, y: self.frame.size.height))
        path.close()
    }
    
    func animateLayer() {
        // 何も設定しなくてもアニメーションする
        shapeLayer.opacity = 0.1
    }
}
```  
  
![1.gif](/image/c20c142d-2f65-78db-e208-10267554fa85.gif)  
  
  
## UIViewのlayerはアニメーションしない  
  
最初の方でも記載しましたが、UIViewのlayerはアニメーションがオフになっています。  
そのため値の変更がすぐに反映されます。  
  
  
```swift

class DemoView: UIView {
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        backgroundColor = .red
    }
    
    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
    }
}

class MyViewController : UIViewController {
    
    var demoView: DemoView!
    
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        
        view.backgroundColor = .white
        
        let width: CGFloat = 300
        let height: CGFloat = 300
        let size = view.frame.size
        
        demoView = DemoView(frame: CGRect(
            x: size.width/2 - width/2,
            y: size.height/2 - height/2,
            width: width, height: height))
        
        view.addSubview(demoView)
        
        let button = UIButton(frame: CGRect(
            x: size.width/2 - 75, y: size.height - 100,
            width: 150, height: 50))
        button.setTitle("アニメーション", for: .normal)
        button.backgroundColor = .blue
        button.addTarget(self, action: #selector(animate), for: .touchUpInside)
        view.addSubview(button)
    }
    
    @objc func animate() {
        // layerではアニメーションしない
        demoView.layer.opacity = 0.1
    }
}
```  
  
![2.gif](/image/90e58647-5b83-3784-e7d2-639a06ffe3fa.gif)  
  
  
## CABasicAnimation  
  
名前にあるように基本的なアニメーションを実行することができます。  
  
https://developer.apple.com/documentation/quartzcore/cabasicanimation  
  
```swift

class DemoView: UIView {
    
    var path: UIBezierPath!
    var shapeLayer: CAShapeLayer!
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        
        backgroundColor = .black
        
        makeShapeLayer()
    }
    
    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
    }
    
    func makeShapeLayer() {
        self.makeTriangle()
        
        shapeLayer = CAShapeLayer()
        shapeLayer.path = path.cgPath        
        shapeLayer.lineWidth = 3.0
        
        layer.mask = shapeLayer
    }
    
    func makeTriangle() {
        path = UIBezierPath()
        path.move(to: CGPoint(x: self.frame.width/2, y: 0.0))
        path.addLine(to: CGPoint(x: 0.0, y: self.frame.size.height))
        path.addLine(to: CGPoint(x: self.frame.size.width, y: self.frame.size.height))
        path.close()
    }
    
    func animateLayer() {
        
        // keyPath!!
        let animation = CABasicAnimation(keyPath: #keyPath(CALayer.opacity))
        animation.fromValue = 1
        animation.toValue = 0.1
        animation.duration = 3
        animation.timingFunction = CAMediaTimingFunction(name: .easeInEaseOut)
        
        // アニメーション終了後の状態を維持する
        animation.isRemovedOnCompletion = false
        animation.fillMode = .forwards
        shapeLayer.add(animation, forKey: nil)        
    }
}
```  
![ダウンロード (1).gif](/image/759b0f82-4d3f-af81-b2eb-8bfd1f4b1b12.gif)  
  
fromValueとtoValueでアニメーションの開始と終了の値を設定し、durationでアニメーションの時間を決めます。  
CALayerの[add(_:forKey:)](https://developer.apple.com/documentation/quartzcore/calayer/1410848-add)に追加されることでアニメーションを開始します。  
  
isRemovedOnCompletionをfalseに、かつ[fillMode](https://developer.apple.com/documentation/quartzcore/camediatiming/fill_modes  
)をforwardsに指定すると、  
アニメーションの状態を終了時に留まらせることができます。  
  
だたし、この場合モデルレイヤ自体の値は更新されないので、  
値の更新を目的とした場合には注意が必要です。  
  
  
```swift

class DemoView: UIView {
    
    var path: UIBezierPath!
    var shapeLayer: CAShapeLayer!
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        
        backgroundColor = .black
        
        makeShapeLayer()
    }
    
    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
    }
    
    func makeShapeLayer() {
        self.makeTriangle()
        
        shapeLayer = CAShapeLayer()
        shapeLayer.path = path.cgPath        
        shapeLayer.lineWidth = 3.0
        
        layer.mask = shapeLayer
    }
    
    func makeTriangle() {
        path = UIBezierPath()
        path.move(to: CGPoint(x: self.frame.width/2, y: 0.0))
        path.addLine(to: CGPoint(x: 0.0, y: self.frame.size.height))
        path.addLine(to: CGPoint(x: self.frame.size.width, y: self.frame.size.height))
        path.close()
    }
    
    func animateLayer() {
        
        let animation = CABasicAnimation(keyPath: #keyPath(CALayer.opacity))
        animation.fromValue = 1
        animation.toValue = 0.1
        animation.duration = 3
        animation.timingFunction = CAMediaTimingFunction(name: .easeInEaseOut)
        
        // アニメーション終了後の状態を維持する
        animation.isRemovedOnCompletion = false
        animation.fillMode = .forwards

        animation.delegate = self
        shapeLayer.add(animation, forKey: nil)
    }
}

extension DemoView: CAAnimationDelegate {
    func animationDidStop(_ anim: CAAnimation, finished flag: Bool) {
        // １．０が出力される
        print(shapeLayer.opacity)
    }
}
```  
  
アニメーション後の状態を保持し、かつモデルレイヤの値も更新したい場合は  
プロパティの値を変更します。  
  
```swift

class DemoView: UIView {
    
    var path: UIBezierPath!
    var shapeLayer: CAShapeLayer!
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        
        backgroundColor = .black
        
        makeShapeLayer()
    }
    
    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
    }
    
    func makeShapeLayer() {
        self.makeTriangle()
        
        shapeLayer = CAShapeLayer()
        shapeLayer.path = path.cgPath        
        shapeLayer.lineWidth = 3.0
        
        layer.mask = shapeLayer
    }
    
    func makeTriangle() {
        path = UIBezierPath()
        path.move(to: CGPoint(x: self.frame.width/2, y: 0.0))
        path.addLine(to: CGPoint(x: 0.0, y: self.frame.size.height))
        path.addLine(to: CGPoint(x: self.frame.size.width, y: self.frame.size.height))
        path.close()
    }
    
    func animateLayer() {

        let animation = CABasicAnimation(keyPath: #keyPath(CALayer.opacity))
        animation.fromValue = 1
        animation.toValue = 0.1
        animation.duration = 3
        animation.timingFunction = CAMediaTimingFunction(name: .easeInEaseOut)
        
        // アニメーション終了後の値を維持する
        // ↓↓↓↓↓↓↓↓↓
        shapeLayer.opacity = 0.1

        // animation.isRemovedOnCompletion = false
        // animation.fillMode = .forwards
        
        animation.delegate = self
        shapeLayer.add(animation, forKey: nil)
        
    }
}

extension DemoView: CAAnimationDelegate {
    func animationDidStop(_ anim: CAAnimation, finished flag: Bool) {
        // 0.1が出力される
        print(shapeLayer.opacity)
    }
}
```  
  
また  
Core Animationとは直接関係ありませんが、  
keyPathがパワーアップしたことで文字列で指定していたところを  
コンパイル時にチェックが走る形で記述ができるようになりました。  
  
```swift

let animation = CABasicAnimation(keyPath: "opacity")

↓

let animation = CABasicAnimation(keyPath: #keyPath(CALayer.opacity))
```  
  
先に出してしまいましたが  
CAAnimationはdelegateという  
CAAnimationDelegateプロトコルのプロパティを持っており  
これに適合することでアニメーションの開始と終わりに処理を追加することができます。  
  
https://developer.apple.com/documentation/quartzcore/caanimationdelegate  
  
### fromValue, toValue, byValueの値の設定による違い  
  
fromValue, toValue, byValueの設定の仕方で  
フレームワークのアニメーションの仕方が変わります。  
設定したdurationの間に与えられた値の変化をどう実現するかです。  
  
下記のようなバリエーションがあります。  
ここはあまり飲み込めませんでした。もしご存知の方いらっしゃいましたら教えていただけると嬉しいです🙇🏻‍♂️  
  
前提として全てOptionalですが、最大2つの値を指定する必要があります。  
  
- fromValueとtoValueが設定されている場合はこの間で計算される  
- fromValueとbyValueが設定されている場合はfromValueとfromValue + byValueの間で計算される  
- toValueとbyValueが設定されている場合はtoValue - byValueとtoValueとの間で計算される  
- fromValueのみ設定されている場合はfromValueと現在表示されている値の間で計算される  
- toValueのみ設定されている場合は**プレゼンテーションレイヤー**の現在値とfromValueとの間で計算される  
- byValueのみ設定されている場合は**プレゼンテーションレイヤー**の現在値とその値 + byValueとの間で計算される  
- 全てがnilの場合は前回アニメーション時のプレゼンテーションレイヤーの値と現在のプレゼンテーションレイヤーの値との間で計算される  
  
# CAKeyframeAnimation  
  
CABasicAnimationよりもより細かい設定ができます。  
CGPathを設定するで図形を描くような動きのアニメーションなどが実現できます。  
  
```swift

class DemoView: UIView {
    
    var path: UIBezierPath!
    var shapeLayer: CAShapeLayer!
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        
        backgroundColor = .black
        
        makeShapeLayer()
    }
    
    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
    }
    
    func makeShapeLayer() {
        self.makeTriangle()
        
        shapeLayer = CAShapeLayer()
        shapeLayer.path = path.cgPath        
        shapeLayer.lineWidth = 3.0
        
        layer.mask = shapeLayer
    }
    
    func makeTriangle() {
        path = UIBezierPath()
        path.move(to: CGPoint(x: self.frame.width/10, y: 0.0))
        path.addLine(to: CGPoint(x: 0.0, y: self.frame.size.height / 10))
        path.addLine(to: CGPoint(x: self.frame.size.width / 10, y: self.frame.size.height / 10))
        path.close()
    }
    
    func animateLayer() {
        
        let keyAnimation = CAKeyframeAnimation(keyPath: #keyPath(CALayer.position))
        let path = CGMutablePath()
        path.move(to: CGPoint(x: 0, y: 0))
        path.addCurve(to: CGPoint(x: 74, y: 200),
                      control1: CGPoint(x: 120, y: 200),
                      control2: CGPoint(x: 120, y: 74))
        path.addCurve(to: CGPoint(x: 120, y: 200),
                      control1: CGPoint(x: 366, y: 200),
                      control2: CGPoint(x: 366, y: 74))
        keyAnimation.path = path
        keyAnimation.duration = 3
        
        shapeLayer.add(keyAnimation, forKey: nil)
    }
}
```  
  
![ダウンロード.gif](/image/a2d8e67c-8bf3-55f5-49aa-3bec632e9049.gif)  
  
https://developer.apple.com/documentation/quartzcore/cakeyframeanimation  
  
calculationModeやrotationModeを設定すると色々動きが変わりますので  
ご興味ございましたらぜひ試してみてください！  
  
# CAAnimationGroup  
  
複数のアニメーションをまとめて同じ設定を反映させることができます。  
  
また  
デリゲートメソッドを実装し、delegateプロパティを設定することで  
グループ内のアニメーションが全て完了したあとの処理を  
追加することもできます。  
※ グループ内の個々のアニメーションのdelegateプロパティに値を設定してもデリゲートメソッドは呼ばれません。  
  
https://developer.apple.com/documentation/quartzcore/caanimationgroup  
  
  
```swift

class DemoView: UIView {
    
    var path: UIBezierPath!
    var shapeLayer: CAShapeLayer!
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        
        // 色はこっちで設定する
        backgroundColor = .black
        
        makeShapeLayer()
    }
    
    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
    }
    
    func makeShapeLayer() {
        self.makeTriangle()
        
        shapeLayer = CAShapeLayer()
        shapeLayer.path = path.cgPath
        shapeLayer.lineWidth = 3.0
        
        layer.mask = shapeLayer
    }
    
    func makeTriangle() {
        path = UIBezierPath()
        path.move(to: CGPoint(x: self.frame.width/10, y: 0.0))
        path.addLine(to: CGPoint(x: 0.0, y: self.frame.size.height / 10))
        path.addLine(to: CGPoint(x: self.frame.size.width / 10, y: self.frame.size.height / 10))
        path.close()
    }
    
    func animateLayer() {
        
        let fadeOut = CABasicAnimation(keyPath: #keyPath(CALayer.opacity))
        fadeOut.fromValue = 1
        fadeOut.toValue = 0
        fadeOut.duration = 5
        
        let expandScale = CABasicAnimation(keyPath: #keyPath(CALayer.transform))
        expandScale.valueFunction = CAValueFunction(name: .scale)
        expandScale.fromValue = [1, 1, 1]
        expandScale.toValue = [3, 3, 3]
        
        let fadeAndScale = CAAnimationGroup()
        fadeAndScale.animations = [fadeOut, expandScale]
        fadeAndScale.duration = 1
        
        fadeAndScale.delegate = self
        
        shapeLayer.add(fadeAndScale, forKey: nil)
    }
}

extension DemoView: CAAnimationDelegate {
    func animationDidStop(_ anim: CAAnimation, finished flag: Bool) {
        
        let transition = CABasicAnimation(keyPath: #keyPath(CALayer.position))
        transition.fromValue = shapeLayer.position
        transition.toValue = CGPoint(x: shapeLayer.position.x + 100.0, y: shapeLayer.position.y + 100)
        transition.duration = 2
        shapeLayer.add(transition, forKey: nil)
    }
}
```  
  
![ダウンロード (4).gif](/image/b5edeb39-69ce-9f99-2b5b-cdbe02d60f83.gif)  
  
注意点として  
個々のアニメーションにdurationを設定した場合は  
そちらも適応されるため  
意図して設定していない際は思わぬ動きをしてしまうかもしれません。  
  
  
```swift

class DemoView: UIView {
    
...

    func animateLayer() {
        
...        
        expandScale.duration = 1

        fadeAndScale.duration = 3
...        
    }
}
```  
  
![ダウンロード (1).gif](/image/ed410060-b03e-3acb-f7e0-d6aa8dfa39fd.gif)  
  
  
# CATransaction  
  
UINavigationControllerの遷移時のアニメーションの変更などをすることができます。  
https://developer.apple.com/documentation/quartzcore/catransition  
  
  
```swift

class ViewController : UIViewController {
    
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        
        view.backgroundColor = .red

        let size = view.frame.size

        let button = UIButton(frame: CGRect(
            x: size.width/2 - 75, y: size.height - 100,
            width: 150, height: 50))
        button.setTitle("遷移", for: .normal)
        button.setTitleColor(.black, for: .normal)
        button.backgroundColor = .yellow
        button.addTarget(self, action: #selector(animate), for: .touchUpInside)
        view.addSubview(button)
    }
    

    @objc func animate() {

        let vc2 = ViewController2()

        let transition = CATransition()
        transition.duration = 0.6
        transition.timingFunction = CAMediaTimingFunction(name: .easeIn)
        transition.type = .fade
        transition.subtype = .fromTop
        
        self.navigationController?.view.layer.add(transition, forKey: nil)
        self.navigationController?.pushViewController(vc2, animated: false)
    }
}

class ViewController2 : UIViewController {
    
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        
        view.backgroundColor = .blue
    }
}

let vc = ViewController()
let nc = UINavigationController(rootViewController: vc)

// Present the view controller in the Live View window
PlaygroundPage.current.liveView = nc
```  
  
![ダウンロード.gif](/image/e78bc73c-a54e-b291-7ea7-b01afcb12d57.gif)  
  
UINavigationControllerのViewのlayerプロパティにアニメーションを追加します。  
  
注意点としてpushViewControllerのanimatedをtrueにすると  
デフォルトの左から右への動作も動いてしまいます。  
  
![ダウンロード (1).gif](/image/25b7a1e3-1875-e462-55a7-f52025046123.gif)  
  
  
# CATransaction  
  
名前間違えやすいですが  
上が「Transition(トランジション)」で  
こちらは「Transaction(トランザクション)」です。  
  
CAAnimationGroupと同様にアニメーションをまとめることができますが、  
多数のレイヤー階層に設定されたアニメーションを一つのバッチ更新として処理してくれます。  
  
そもそもCore Animationはレイヤーが修正された場合  
明示的にトランザクションが作成されていない際は自動でトランザクションを作成し  
次のrunloopのイテレーションの中で実行されるようになります。  
  
CAAnimationGroupと同様に  
同じ設定をトランザクション内のアニメーションに適用することができ、  
[setCompletionBlock(_:)](https://developer.apple.com/documentation/quartzcore/catransaction/1448281-setcompletionblock)でアニメーション完了後の処理を追加することができます。  
  
さらに［setDisableActions(_:)］(https://developer.apple.com/documentation/quartzcore/catransaction/1448261-setdisableactions)をfalseに設定することで  
暗黙のトランザクションを無効化することもできます。  
  
```swift

class DemoView: UIView {
    
    var path: UIBezierPath!
    var shapeLayer: CAShapeLayer!
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        
        backgroundColor = .black
        
        makeShapeLayer()
    }
    
    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
    }
    
    func makeShapeLayer() {
        self.makeTriangle()
        
        shapeLayer = CAShapeLayer()
        shapeLayer.path = path.cgPath
        
        shapeLayer.fillColor = UIColor.yellow.cgColor
        shapeLayer.strokeColor = UIColor.brown.cgColor
        shapeLayer.lineWidth = 3.0
        
        layer.mask = shapeLayer
    }
    
    func makeTriangle() {
        path = UIBezierPath()
        path.move(to: CGPoint(x: self.frame.width/2, y: 0.0))
        path.addLine(to: CGPoint(x: 0.0, y: self.frame.size.height))
        path.addLine(to: CGPoint(x: self.frame.size.width, y: self.frame.size.height))
        path.close()
    }
    
    func animateLayer() {

        CATransaction.begin()
        
        // 終わったあとの処理を定義できる
        CATransaction.setCompletionBlock { [weak self] in
            self?.shapeLayer.fillColor = UIColor.blue.cgColor
            self?.shapeLayer.lineWidth = 3.0
            self?.shapeLayer.opacity = 1
        }
        CATransaction.setAnimationDuration(2)
        CATransaction.setDisableActions(false)
        CATransaction.setAnimationTimingFunction(CAMediaTimingFunction(name: .easeIn))

        shapeLayer.lineWidth = 50
        shapeLayer.opacity = 0.1
        CATransaction.commit()
    }
}

```  
  
![ダウンロード (4).gif](/image/62a7b7b3-9e70-a31b-2579-0385387469f8.gif)  
  
  
さらにトランザクションを入れ子にすることもでき  
特定のアニメーションだけ別の設定を適用することもできます。  
  
```swift
class DemoView: UIView {
    
    var path: UIBezierPath!
    var shapeLayer: CAShapeLayer!
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        
        backgroundColor = .black
        
        makeShapeLayer()
    }
    
    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
    }
    
    func makeShapeLayer() {
        self.makeTriangle()
        
        shapeLayer = CAShapeLayer()
        shapeLayer.path = path.cgPath
        
        shapeLayer.fillColor = UIColor.yellow.cgColor
        shapeLayer.strokeColor = UIColor.brown.cgColor
        shapeLayer.lineWidth = 3.0
        
        layer.mask = shapeLayer
    }
    
    func makeTriangle() {
        path = UIBezierPath()
        path.move(to: CGPoint(x: self.frame.width/2, y: 0.0))
        path.addLine(to: CGPoint(x: 0.0, y: self.frame.size.height))
        path.addLine(to: CGPoint(x: self.frame.size.width, y: self.frame.size.height))
        path.close()
    }
    
    func animateLayer() {

        CATransaction.begin()
        
        // 終わったあとの処理を定義できる
        CATransaction.setCompletionBlock { [weak self] in
            self?.shapeLayer.fillColor = UIColor.blue.cgColor
            self?.shapeLayer.lineWidth = 3.0
            self?.shapeLayer.opacity = 1
        }
        CATransaction.setAnimationDuration(4)
        CATransaction.setDisableActions(false)
        CATransaction.setAnimationTimingFunction(CAMediaTimingFunction(name: .easeIn))

        shapeLayer.lineWidth = 50

        // Transactionの中のTransactionで設定を変えることもできる
        CATransaction.begin()
        CATransaction.setAnimationDuration(2)
        shapeLayer.opacity = 0.1
        CATransaction.commit()


        CATransaction.commit()
    }
}
```  
  
![ダウンロード (1).gif](/image/007b86ca-f11b-372d-84fe-a32f37ff72c2.gif)  
  
  
  
# サブクラス  
  
他にもフレームワークで用意されている様々なサブクラスがあります。  
この場では割愛させていただきますが  
下記のオープンソースでそれぞれ紹介されており  
どの値を設定するとどうアニメーションが変わるかなどが見てわかるようにできていますので  
ご興味がございましたらぜひ見てみてください  
  
https://github.com/raywenderlich/LayerPlayer  
  
# 余興  
  
今回の内容は2018/12/21に開催されましたROPPONGI.swiftでもさせて頂いたのですが  
テーマが「望年会」ということもあり  
最後にCore Animationを使ってこんなものを作ってみました。  
  
![ezgif.com-video-to-gif.gif](/image/e2cad825-4190-22c7-f07f-d5dbf8fed567.gif)  
  
# まとめ  
  
Core Animationについて調べてみた結果  
機能が非常に豊富でアプリの表現力を高めることができるフレームワークであることがわかりました。  
  
ただし、**基本的にUIViewでできることはUIViewで**と考えております。  
UIViewの方が簡単に記述ができますし、コードも複雑になりません。  
  
それでも必要になってくる場面は出てくると思いますし  
そういういざという時のためにも知っておいても損はないのかなと思っています。  
  
今回の内容が少しでもお役に立てましたら幸いです。  
  
  
ご指摘などございましたらぜひ教えてください:bow_tone1:  
  
