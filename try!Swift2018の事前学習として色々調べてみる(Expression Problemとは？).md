  
## 経緯  
  
[try!Swift2018](https://www.tryswift.co/events/2018/tokyo/jp/#schedule)に参加するにあたり、  
知識、経験共に乏しい自分が講演内容についていくためにも  
事前準備として関連することを色々調べてみようと思い、  
調べたことを忘れないために記録します。  
  
  
[Expression Problemを解決する](https://www.tryswift.co/events/2018/tokyo/jp/#proofs)  
  
というタイトルを見て、まず「Expression Problem」って何だろう？と思ったところから調べてみました。  
  
## Expression Problemとは？  
  
静的な型の安全性を保った状態(キャストしない)で、  
元のソースに変更を加えず(再コンパイルしない)、  
新しいデータ型の追加や新しい処理の追加をするのは難しい  
  
参考元: https://ja.wikipedia.org/wiki/Expression_problem  
  
ということだそうです。  
  
ざっくり言うと  
フレームワークや外部ライブラリの拡張のしやすさのことなのかなと思いました。  
  
## Swiftだと  
  
以降は講演者である[Brandon Kaseさん](https://twitter.com/bkase_)がお話されていた  
[dotswift2018](https://www.dotconferences.com/2018/01/brandon-kase-finally-solving-the-expression-problem)  
[objc.io](https://talk.objc.io/episodes/S01E88-extensible-libraries-1-enums-vs-classes)  
の内容に加えて実際に動かしてみた結果を加えています。  
  
  
Enumベースとclassベースの図を描くためのライブラリがあるとします。  
  
  
### Enumベース  
  
```EnumBasedDiagrams.swift


public enum Diagram {
    case rectangle(CGRect, NSColor)
    case ellipse(in: CGRect, NSColor)
    indirect case combined(Diagram, Diagram)
}

extension Diagram {
    
    public func draw(_ context: CGContext) {
        context.saveGState()
        switch self {
        case let .rectangle(rect, color):
            context.setFillColor(color.cgColor)
            context.fill(rect)
        case let .ellipse(rect, color):
            context.setFillColor(color.cgColor)
            context.fillEllipse(in: rect)
        case let .combined(d1, d2):
            d1.draw(context)
            d2.draw(context)
        }
        context.restoreGState()
    }
}
```  
  
```ViewController.swift

import Cocoa

final class LayerView: NSView {
    convenience init(_ rect: CGRect, _ layer: CALayer) {
        self.init(frame: rect)
        self.layer = layer
        self.layerUsesCoreImageFilters = true
    }
    
    override var isFlipped: Bool { return true }
}

final class CGContextView: NSView {
    let render: (CGContext) -> ()
    init(frame: CGRect, render: @escaping (CGContext) -> ()) {
        self.render = render
        super.init(frame: frame)
    }
    
    required init?(coder decoder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    override func draw(_ dirtyRect: NSRect) {
        render(NSGraphicsContext.current!.cgContext)
    }
    
    override var isFlipped: Bool { return true }
}

final class CGContextView: NSView {
    let render: (CGContext) -> ()
    init(frame: CGRect, render: @escaping (CGContext) -> ()) {
        self.render = render
        super.init(frame: frame)
    }
    
    required init?(coder decoder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    override func draw(_ dirtyRect: NSRect) {
        render(NSGraphicsContext.current!.cgContext)
    }
    
    override var isFlipped: Bool { return true }
}


import EnumBasedDiagrams

let diagram = Diagram.combined(
    .rectangle(CGRect(x: 20, y: 20, width: 100, height: 100), .red),
    .ellipse(in: CGRect(x: 60, y: 60, width: 80, height: 80), .green),
)

class ViewController: NSViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        let frame = CGRect(x: 0, y: 0, width: 200, height: 200)
        let diagramView = CGContextView(frame: frame, render: diagram.draw)
        view.addSubview(diagramView)
    }
}
```  
  
<img width="482" alt="スクリーンショット 2018-02-23 16.42.30.png" src="/image/027ac186-60ea-4a12-0e53-8000853c3daf.png">  
  
  
上のrenderではCGContextを使用していますが、  
代わりにCALayerを使用したい場合はどうしますでしょうか？  
  
  
```EnumBasedDiagram+Extension.swift

extension Diagram {
    func render() -> CALayer {
        switch self {
        case let .rectangle(rect, color):
            let result = CALayer()
            result.frame = rect
            result.backgroundColor = color.cgColor
            return result
        case let .ellipse(rect, color):
            let result = CAShapeLayer()
            result.path = CGPath(ellipseIn: rect, transform: nil)
            result.fillColor = color.cgColor
            return result
        case let .combined(d1, d2):
            let result = CALayer()
            result.addSublayer(d1.render())
            result.addSublayer(d2.render())
            return result
        }
    }
}
```  
  
  
```ViewController.swift

final class LayerView: NSView {
    convenience init(_ rect: CGRect, _ layer: CALayer) {
        self.init(frame: rect)
        self.layer = layer
        self.layerUsesCoreImageFilters = true
    }
    
    override var isFlipped: Bool { return true }
}

class ViewController: NSViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        let frame = CGRect(x: 0, y: 0, width: 200, height: 200)
        let diagramView = LayerView(frame, diagram.render())
        view.addSubview(diagramView)
    }
}
```  
このようにEnumでは各ケースに応じて処理を実装することで、  
簡単に拡張することができます。  
  
  
では、新しいcaseが必要になった場合はどうでしょうか？  
  
```EnumBasedDiagrams.swift


public enum Diagram {
    ...
    indirect case alpha(CGFloat, Diagram)
}

extension Diagram {
    public func draw(_ context: CGContext) {
        context.saveGState()
        switch self {
        
        ...
        case let .alpha(alpha, d):
            context.setAlpha(alpha)
            d.draw(context)
        }
        context.restoreGState()
    }
}
```  
  
```EnumBasedDiagram+Extension.swift


extension Diagram {
    func render() -> CALayer {
        switch self {
        // ...
        case let .alpha(alpha, d):
            let result = CALayer()
            result.opacity = Float(alpha)
            result.addSublayer(d.render())
            return result
        }
    }
}
```  
  
  
<img width="439" alt="スクリーンショット 2018-02-23 16.37.58.png" src="/image/d6bbb1bf-bea7-f346-cada-6dee3dac81c0.png">  
  
ライブラリの側に修正が必要になり、  
caseが増えたことによってrenderメソッドにも処理の追加が必要になります。  
  
※[dotswift](https://www.dotconferences.com/2018/01/brandon-kase-finally-solving-the-expression-problem)でも話されていましたが、  
Non-Exhausitive Enumが導入された場合はまた話が変わってきます。  
  
  
### Classベース  
  
```ClassBasedDiagrams.swift


open class Diagram {
    public init()
    open func draw(_ context: CGContext)
}

public class Rectangle: Diagram {
    let rect: CGRect
    let color: NSColor
    public init(_ rect: CGRect, _ color: NSColor) {
        self.rect = rect
        self.color = color
    }

    override public func draw(_ context: CGContext) {
        context.saveGState()
        context.setFillColor(color.cgColor)
        context.fill(rect)
        context.restoreGState()
    }
}

public class Ellipse: Diagram {
    let rect: CGRect
    let color: NSColor
    public init(in rect: CGRect, _ color: NSColor) {
        self.rect = rect
        self.color = color
    }

    override public func draw(_ context: CGContext) {
        context.saveGState()
        context.setFillColor(color.cgColor)
        context.fillEllipse(in: rect)
        context.restoreGState()
    }
}

public class Combined: Diagram {
    let d1: Diagram
    let d2: Diagram
    public init(_ d1: Diagram, _ d2: Diagram) {
        self.d1 = d1
        self.d2 = d2
    }
    
    override public func draw(_ context: CGContext) {
        d1.draw(context)
        d2.draw(context)
    }
}
```  
  
使い方はEnumベースのライブラリと同様です。  
  
```ViewController.swift


let diagram = Combined(
    Rectangle(CGRect(x: 20, y: 20, width: 100, height: 100), .red),
    Ellipse(in: CGRect(x: 60, y: 60, width: 80, height: 80), .green)
)

class ViewController: NSViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        let frame = CGRect(x: 0, y: 0, width: 200, height: 200)
        let diagramView = CGContextView(frame: frame, render: diagram.draw)
        view.addSubview(diagramView)
    }
}
```  
  
Enumベースで最後に追加をした.alphaを  
Classベースのライブラリで追加する場合は  
シンプルにSubclassを作成することで可能になります。  
  
```ClassBasedDiagrams.swift


class Alpha: Diagram {
    
    let alpha: CGFloat
    let diagram: Diagram
    init(alpha: CGFloat, diagram: Diagram) {
        self.alpha = alpha
        self.diagram = diagram
    }

    override func draw(_ context: CGContext) {
        context.saveGState()
        context.setAlpha(alpha)
        diagram.draw(context)
        context.restoreGState()
    }
}

...
```  
  
```ViewController.swift


let diagram = Combined(
    Rectangle(CGRect(x: 20, y: 20, width: 100, height: 100), .red),
    Alpha(alpha: 0.5, diagram: Ellipse(in: CGRect(x: 60, y: 60, width: 80, height: 80), .green))
)

....
```  
  
逆に、Enumベースで実装したように  
CALayerを使用したい場合はどうしますでしょうか？  
  
この場合、drawメソッドの引数にCALayerを設定する必要があり、  
BaseのDiagramクラスの修正が必要になります。  
  
それに伴い、全てのSubclassにも実装が必要になります。  
  
  
このようにExpression Problemが発生します。  
  
  
## Expression Problemを解決するためには？  
  
Protocolを使うことでこれらが解消できるようです。  
  
ポイントは  
  
ProtocolのメソッドがSelfを返すこと  
Protocolを継承したクラス(struct)を利用すること  
  
です。  
  
```ProtocolBasedDiagrams.swift

public protocol Diagram {
    static func rectangle(_ rect: CGRect, _ fill: NSColor) -> Self
    static func ellipse(_ rect: CGRect, _ fill: NSColor) -> Self
    static func combined(_ d1: Self, _ d2: Self) -> Self
}
```  
  
描画メソッドの代わりに、  
Protocolベースではstructを活用します。  
  
```Renderer.swift

public struct ContextRenderer {
    public let render: (CGContext) -> ()
    public init(_ draw: @escaping (CGContext) -> ()) {
        self.render = { context in
            context.saveGState()
            draw(context)
            context.restoreGState()
        }
    }
}

extension ContextRenderer: Diagram {
    public static func rectangle(_ rect: CGRect, _ fill: NSColor) -> ContextRenderer {
        return ContextRenderer { context in
            context.setFillColor(fill.cgColor)
            context.fill(rect)
        }
    }
    
    public static func ellipse(_ rect: CGRect, _ fill: NSColor) -> ContextRenderer {
        return ContextRenderer { context in
            context.setFillColor(fill.cgColor)
            context.fillEllipse(in: rect)
        }
    }
    
    public static func combined(_ d1: ContextRenderer, _ d2: ContextRenderer) -> ContextRenderer {
        return ContextRenderer { context in
            d1.render(context)
            d2.render(context)
        }
    }
}
```  
  
使用する場合は、他と同じように使用可能です。  
  
  
```ViewController.swift


import ProtocolBasedDiagrams

...

let diagram = ContextRenderer.combined(
    .rectangle(CGRect(x: 20, y: 20, width: 100, height: 100), .red),
    .ellipse(CGRect(x: 60, y: 60, width: 80, height: 80), .green)
)

class ViewController: NSViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        let frame = CGRect(x: 0, y: 0, width: 200, height: 200)
        let diagramView = CGContextView(frame: frame, render: diagram.render)
        view.addSubview(diagramView)
    }
}
```  
  
では、EnumベースのようにCALayerで処理したい場合はどうでしょうか？  
  
```Renderer.swift


public struct LayerRenderer {
    public let render: () -> (CALayer)
    public init(_ draw: @escaping () -> (CALayer)) {
        self.render = {
            draw()
        }
    }
}

extension LayerRenderer: Diagram {
    
    public static func rectangle(_ rect: CGRect, _ fill: NSColor) -> LayerRenderer {
        return LayerRenderer {
            let layer = CALayer()
            layer.backgroundColor = fill.cgColor
            layer.frame = rect
            return layer
        }
    }
    
    public static func ellipse(_ rect: CGRect, _ fill: NSColor) -> LayerRenderer {
        return LayerRenderer {
            let layer = CALayer()
            layer.frame = rect
            layer.backgroundColor = fill.cgColor
            layer.cornerRadius = rect.width / 2
            return layer
        }
    }
    
    public static func combined(_ d1: LayerRenderer, _ d2: LayerRenderer) -> LayerRenderer {
        return LayerRenderer {
            let layer = CALayer()
            layer.addSublayer(d1.render())
            layer.addSublayer(d2.render())
            return layer
        }
    }
}
```  
  
```ViewController.swift


let diagram = LayerRenderer.combined(
    .rectangle(CGRect(x: 20, y: 20, width: 100, height: 100), .red),
    .ellipse(CGRect(x: 20, y: 20, width: 80, height: 80), .green)
)

class ViewController: NSViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        let frame = CGRect(x: 0, y: 0, width: 200, height: 200)
        let diagramView = LayerView(frame, diagram.render())
        view.addSubview(diagramView)
    }
}
```  
  
では、新しいデータ型を追加する場合はどうでしょうか？  
この場合はライブラリのプロトコルを継承した新しいプロトコルを定義します。  
  
```Renderer.swift

public protocol AlphaDiagram {
    static func alpha(_ alpha: CGFloat, _ diagram: Self) -> Self
}


...

extension ContextRenderer: AlphaDiagram {
    public static func alpha(_ alpha: CGFloat, _ diagram: ContextRenderer) -> ContextRenderer {
        return ContextRenderer { context in
            context.setAlpha(alpha)
            diagram.render(context)
        }
    }
}

...

extension LayerRenderer: AlphaDiagram {
    public static func alpha(_ alpha: CGFloat, _ diagram: LayerRenderer) -> LayerRenderer {
        return LayerRenderer {
            let layer = diagram.render()
            layer.opacity = Float(alpha)
            return layer
        }
    }
}
```  
  
```ViewController.swift

let diagram = LayerRenderer.combined(
    .rectangle(CGRect(x: 20, y: 20, width: 100, height: 100), .red),
    .alpha(0.5, .ellipse(CGRect(x: 20, y: 20, width: 100, height: 100), .green))
)

...

```  
  
## まとめ  
  
SwiftのProtocolの性質を活用することで、  
新しいデータ型の追加、新しい処理の追加が可能になり、  
Expression Problemを解決することができたという結論のようです。  
  
実はこの方法はTagless Finalというようで、  
数学の世界?では有名なものなようです。  
(ここらへん全然わかりませんでした。すいません。。。)  
  
参照元:  
http://okmij.org/ftp/tagless-final/index.html#course-oxford  
http://okmij.org/ftp/tagless-final/course/lecture.pdf  
http://wkwkes.hatenablog.com/entry/2017/12/21/173029  
  
  
今回色々と調べてみましたが、  
理解できていない部分、不明な部分がまだまだありますので  
当日の講演を聞いてもっと理解を深めたいと思います。  
