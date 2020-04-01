今年もクリスマスが近いてきました。  
  
昨年はCore Animationを使ってクリスマスツリーを作成してみました。  
https://github.com/stzn/qiita/blob/master/【iOS】 Core Animationまとめ.md  
  
そこで  
今年はSwiftUIを使ってクリスマスツリーを作成したいと思います。  
  
SwiftUIの要素はたくさん使用していますが  
`drawingGroup`に注目したいと思います。  
  
# Shapeに適合させツリーのパーツを作成する  
  
まずツリーに必要なパーツを作っていきます。  
図形をShapeプロトコルに適合させたstructとして定義して  
それを組み合わせてViewを構築します。  
  
## 星  
  
![スクリーンショット 2019-12-01 10.46.06.png](/image/293c35c2-2ae5-5952-0595-bc16972a463d.png)  
  
こちらは下記のサイトを参照させて頂きました。  
https://www.hackingwithswift.com/quick-start/swiftui/how-to-draw-polygons-and-stars  
  
`path`メソッドの中でPathクラスを生成します。  
Pathの指定はUIBezierPathと似たような形で設定できます。  
  
<details>  
<summary>コード</summary><div>  
  
```swift

struct Star: Shape {
    let corners: Int
    let smoothness: CGFloat

    func path(in rect: CGRect) -> Path {
        guard corners >= 2 else { return Path() }

        let center = CGPoint(x: rect.width / 2, y: rect.height / 2)

        var currentAngle = -CGFloat.pi / 2

        let angleAdjustment = .pi * 2 / CGFloat(corners * 2)

        let innerX = center.x * smoothness
        let innerY = center.y * smoothness

        var path = Path()

        path.move(to: CGPoint(x: center.x * cos(currentAngle), y: center.y * sin(currentAngle)))

        var bottomEdge: CGFloat = 0

        for corner in 0..<corners * 2  {
            let sinAngle = sin(currentAngle)
            let cosAngle = cos(currentAngle)
            let bottom: CGFloat

            if corner.isMultiple(of: 2) {
                bottom = center.y * sinAngle
                path.addLine(to: CGPoint(x: center.x * cosAngle, y: bottom))
            } else {
                bottom = innerY * sinAngle
                path.addLine(to: CGPoint(x: innerX * cosAngle, y: bottom))
            }
            if bottom > bottomEdge {
                bottomEdge = bottom
            }
            currentAngle += angleAdjustment
        }
        let unusedSpace = (rect.height / 2 - bottomEdge) / 2
        let transform = CGAffineTransform(translationX: center.x, y: center.y + unusedSpace)
        return path.applying(transform)
    }
}
```  
</div>  
</details>  
  
  
## 木  
  
次に木の部分です。  
  
![スクリーンショット 2019-12-01 10.46.46.png](/image/ce0f8f30-85d3-ad22-e8c2-1b28a15e801c.png)  
  
### 木の形の部分  
  
まずは緑の部分の三角形Triangleを定義します。  
  
<details>  
<summary>コード</summary><div>  
  
```swift

struct Triangle: Shape {
    func path(in rect: CGRect) -> Path {
        Path { path in
            let middle = rect.midX
            let width: CGFloat = rect.size.width
            let height = rect.height
            path.move(to: CGPoint(x: middle, y: 0))
            path.addLine(to: CGPoint(x: middle + (width / 2), y: height))
            path.addLine(to: CGPoint(x: middle - (width / 2), y: height))
            path.addLine(to: CGPoint(x: middle, y: 0))
        }
    }
}

```  
</div>  
</details>  
  
### 白い線状の飾り  
  
木の上にある飾りを作成します。  
  
今回は`addQuadCurve`を使用して  
ちょっと曲線にしています。  
  
https://developer.apple.com/documentation/swiftui/path/3271274-addquadcurve  
  
<details>  
<summary>コード</summary><div>  
  
```swift

struct Slope: Shape {
    func path(in rect: CGRect) -> Path {
        Path { path in
            path.move(to: CGPoint(x: rect.minX, y: rect.midY))
            path.addQuadCurve(to: CGPoint(x: rect.maxX, y: rect.maxY),
                              control: CGPoint(x: rect.midX * 0.8, y: rect.midY * 0.8))
        }
    }
}
```  
</div>  
</details>  
  
### 球状の飾り  
  
次に球状の飾りを作ります。  
これは`Circle`に  
`Gradient`と`LinierGradient`を使って  
色をグラデーションしています。  
  
https://developer.apple.com/documentation/swiftui/gradient  
https://developer.apple.com/documentation/swiftui/lineargradient  
  
`Gradient`の初期化時にグラデーションさせたい色を指定します。  
変化させる位置を直接指定することもできますが  
指定しない場合は  
フレームワークで自動で調整してくれるようです。  
  
`LinearGradient`は  
`Gradient`と開始と終了位置を指定します。  
  
  
<details>  
<summary>コード</summary><div>  
  
```swift

struct BallView: View {
    let gradientColors = Gradient(colors: [Color.pink, Color.purple])
    var body: some View {
        let linearGradient = LinearGradient(
            gradient: gradientColors,
            startPoint: .top, endPoint: .bottom)
        return Circle()
            .fill(linearGradient)
    }
}
```  
</div>  
</details>  
  
そしてこれを複数組み合わせてViewを作ります。  
  
`GeometryReader`を使って  
Viewの中に規則的に`BallView`を配置しています。  
  
https://developer.apple.com/documentation/swiftui/geometryreader  
  
三角形をはみ出さないように  
`mask`を使用してBallViewを描画する範囲を限定しています。  
https://developer.apple.com/documentation/swiftui/view/3278595-mask  
  
<details>  
<summary>コード</summary><div>  
  
```swift


struct BallsSlopeView<Mask: View>: View {
    let drawArea: Mask
    var body: some View {
        GeometryReader { gr in
            ForEach(1...10, id: \.self) { index in
                BallView()
                    .position(
                        self.getPosition(at: index,
                                         midX: gr.frame(in: .local).maxX,
                                         midY: gr.frame(in: .local).maxY))
                    .frame(height: 20)
            }
        }
        .mask(drawArea)
    }

    private func getPosition(at index: Int, midX: CGFloat, midY: CGFloat) -> CGPoint{
        let x = midX * CGFloat(1 - CGFloat(index) * 0.1)
        let y = midY * CGFloat(1 - CGFloat(index) * 0.05)
        return CGPoint(x: x, y: y)
    }
}
```  
</div>  
</details>  
  
### 組み合わせる  
  
最後に上記で作った部品を組み合わせます。  
  
すべてを`ZStack`でグループにして重ねます。  
白い飾りの部分では  
`rotationEffect`を活用することで  
少し回転させて木にかかっているようにしています。  
  
https://developer.apple.com/documentation/swiftui/scaledshape/3273943-rotationeffect  
  
  
<details>  
<summary>コード</summary><div>  
  
  
```swift

struct TreeView: View {
    var body: some View {
        GeometryReader { gr in
            ZStack(alignment: .center) {
                Triangle()
                    .foregroundColor(Color.green)
                Slope()
                    .stroke(lineWidth: 20)
                    .mask(Triangle())
                    .foregroundColor(Color.white)
                Slope()
                    .stroke(lineWidth: 20)
                    .rotationEffect(Angle.degrees(300))
                    .mask(Triangle())
                    .foregroundColor(Color.white)
                BallsSlopeView(drawArea: Triangle())
            }
        }
    }
}
```  
</div>  
</details>  
  
  
## 土台  
  
![スクリーンショット 2019-12-01 10.47.10.png](/image/b6d517af-a6c6-26f3-3023-2fb81835e571.png)  
  
次に土台の部分を作ります。  
  
`Rectangle`の中に白い線の`Shape`を載せます。  
  
白い線は`Shape`で作成します。  
  
<details>  
<summary>コード</summary><div>  
  
```swift

struct FoundationLine: Shape {
    func path(in rect: CGRect) -> Path {
        Path { path in
            path.move(to: CGPoint(x: 0, y: rect.maxY / 3))
            path.addLine(to: CGPoint(x: rect.maxX, y: rect.maxY / 3))
            path.move(to: CGPoint(x: 0, y: rect.maxY * 2 / 3))
            path.addLine(to: CGPoint(x: rect.maxX, y: rect.maxY * 2 / 3))
            path.move(to: CGPoint(x: rect.width / 3, y: 0))
            path.addLine(to: CGPoint(x: rect.width / 3, y: rect.maxY))
            path.move(to: CGPoint(x: rect.width * 2 / 3, y: 0))
            path.addLine(to: CGPoint(x: rect.width * 2 / 3, y: rect.maxY))
        }
    }
}
```  
</div>  
</details>  
  
  
上記で作った白い線と`Rectangle`を`ZStack`でグループにします。  
  
<details>  
<summary>コード</summary><div>  
  
```swift

struct FoundationView: View {
    var body: some View {
        GeometryReader { gr in
            ZStack {
                Rectangle()
                    .foregroundColor(Color.red)
                FoundationLine()
                    .stroke(lineWidth: 3)
                    .foregroundColor(Color.white)
                    .mask(Rectangle())
            }
            .frame(width: gr.size.width / 3, height: gr.size.width / 3)
        }
    }
}
```  
</div>  
</details>  
  
  
# クリスマスツリーを組み立てる  
  
ではこれまで作ったものを組み合わせます。  
星と個々の木が少しづつ重なるように位置の調整をしています。  
  
また  
そのままですと  
木の三角の重なり方が  
逆になってしまう(上の頂点の部分が上に重なって見える)ため  
`zIndex`で重なり方を変更しています。  
https://developer.apple.com/documentation/swiftui/view/3278679-zindex  
  
<details>  
<summary>コード</summary><div>  
  
```swift

struct ChristmasTree: View {
    var body: some View {
        GeometryReader { gr in
            VStack(spacing: -12) {
                VStack(spacing: -(gr.size.width * 0.1)) {
                    Star(corners: 5, smoothness: 0.5)
                        .foregroundColor(Color.yellow)
                        .frame(width: gr.size.width * 0.3,
                               height: gr.size.width * 0.3)
                        .zIndex(2)
                    ZStack {
                        VStack(spacing: -(gr.size.width / 5)) {
                            TreeView()
                                .frame(width: gr.size.width * 0.6)
                                .zIndex(3)
                            TreeView()
                                .frame(width: gr.size.width * 0.7)
                                .zIndex(2)
                            TreeView()
                                .frame(width: gr.size.width * 0.8)
                                .zIndex(1)
                        }
                        .frame(height: gr.size.height * 0.5)
                        .foregroundColor(Color.green)
                    }
                    .zIndex(1)
                }
                FoundationView()
                    .frame(height: gr.size.height * 0.2)
            }
        }
    }
}
```  
</div>  
</details>  
  
  
# 背景  
  
![スクリーンショット 2019-12-01 10.50.50.png](/image/17e9c627-d362-453a-5662-c59db1a1c017.png)  
  
次に背景を作成していきます。  
  
`Circle`をランダムな大きさとopacityと位置に配置します。  
  
<details>  
<summary>コード</summary><div>  
  
```swift

struct Particles: View {
    var body: some View {
        GeometryReader { gr in
            ZStack {
                ForEach(0...200, id: \.self) { _ in
                    Circle()
                        .foregroundColor(.red)
                        .opacity(.random(in: 0.1...0.4))
                        .frame(width: .random(in: 10...100),
                               height: .random(in: 10...100))
                        .position(x: .random(in: 0...gr.size.width),
                                  y: .random(in: 0...gr.size.height))
                }
            }
        }
    }
}
```  
</div>  
</details>  
  
  
  
# 背景にアニメーションを設定する  
  
せっかくなので背景にアニメーションをつけて  
もう少し豪華(?)にしてみます。  
  
今回は`interactiveSpring`という`Animation`を利用しました。  
  
※   
値は色々と触ってみて  
こんな感じなのかなと思った値を設定しているので  
適当です。  
  
https://developer.apple.com/documentation/swiftui/animation/3344959-interactivespring  
  
  
## 画面表示時にアニメーションを起こすための処理  
  
SwiftUIのアニメーションを設定する上で注意したい点として  
単純にアニメーションを設定しただけでは  
**アニメーションが起動しません。**  
  
これを画面表示時に発生させるためには  
例えば`@State`を付けたの変数を  
`onAppear`の中で変更することで  
Viewの中の値を動的に変更させて再レンダリングさせるなどの  
処理が必要になります。  
  
<details>  
<summary>コード</summary><div>  
  
```swift

struct Particles: View {
    // レンダリングを起こすために必要
    @State private var scaling = false

    var animation: Animation {
        Animation
            .interactiveSpring(response: 5, dampingFraction: 0.5)
            .repeatForever()
            .speed(.random(in: 0.05...0.9))
            .delay(.random(in: 0...2))
    }

    var body: some View {
        GeometryReader { gr in
            ZStack {
                ForEach(0...200, id: \.self) { _ in
                    Circle()
                        .foregroundColor(.red)
                        .opacity(.random(in: 0.1...0.4))
                        .animation(self.animation)
                        // この値を変化させることで再レンダリングを起こしている
                        .scaleEffect(self.scaling ? .random(in: 0.1...2) : 1)
                        .frame(width: .random(in: 10...100),
                               height: .random(in: 10...100))
                        .position(x: .random(in: 0...gr.size.width),
                                  y: .random(in: 0...gr.size.height))
                }
            }
            .onAppear {
                // レンダリングを起こすために必要
                self.scaling = true
            }
        }
    }
}
```  
</div>  
</details>  
  
## パフォーマンスを向上させる  
  
画面としては上記で完成ですが  
一つ問題があります。  
  
上記のアニメーションの処理を行ったことによって  
メモリの使用量がどんどん増えていきます。  
  
![drawNo_480.gif](/image/78e7cb97-da50-5e3a-352e-19f3bc98e4a2.gif)  
  
※ この後もずっと増えていきます。  
  
これは  
ZStackの中の各Viewが描画をする際に  
それぞれでレイヤーを構築します。  
  
そうするとその分のメモリを使用する結果  
CPUへの負荷大きくなります。  
  
これはアプリのパフォーマンスの低下を招くことがあります。  
  
そこで`drawingGroup`を使って負荷を減らすことができます。  
  
### drawingGroup  
  
https://developer.apple.com/documentation/swiftui/group/3284805-drawinggroup  
  
このメソッドは  
Viewの中の全てのViewを  
画面上には見えない**オフスクリーン**上で  
**Metal API**を使用して  
一つのイメージにまとめて描画し  
最終的な内容を画面に出力するようにしてくれます。  
  
こうすることでメモリへの負荷を軽減させて  
パフォーマンスを向上させることができます。  
  
<details>  
<summary>コード</summary><div>  
  
```swift

struct Particles: View {
    // レンダリングを起こすために必要
    @State private var scaling = false

    var animation: Animation {
        Animation
            .interactiveSpring(response: 5, dampingFraction: 0.5)
            .repeatForever()
            .speed(.random(in: 0.05...0.9))
            .delay(.random(in: 0...2))
    }

    var body: some View {
        GeometryReader { gr in
            ZStack {
                ForEach(0...200, id: \.self) { _ in
                    Circle()
                        .foregroundColor(.red)
                        .opacity(.random(in: 0.1...0.4))
                        .animation(self.animation)
                        // この値を変化させることで再レンダリングを起こしている
                        .scaleEffect(self.scaling ? .random(in: 0.1...2) : 1)
                        .frame(width: .random(in: 10...100),
                               height: .random(in: 10...100))
                        .position(x: .random(in: 0...gr.size.width),
                                  y: .random(in: 0...gr.size.height))
                }
            }
            // ここに設定をする
            .drawingGroup()
            .onAppear {
                // レンダリングを起こすために必要
                self.scaling = true
            }
        }
    }
}
```  
</div>  
</details>  
  
![draw_480.gif](/image/cc656a8a-c283-bb97-7eb1-1be89a29bac3.gif)  
  
※   
一つ注意点として  
drawingGroupのドキュメントに下記のような記載あります。  
  
```
Views backed by native platform views don’t render into the image.
```  
  
**native platform views**には  
`drawingGroup`は効果がないようです。  
  
この**native platform views**とは  
何を指すのかわからなかったのですが  
twitter上でAppleの方が回答されていた内容によると  
**NSView**や**UIView**のことのようです。  
  
https://twitter.com/jsh8080/status/1137045666939768833  
  
  
  
# まとめ  
  
アドベントカレンダーのネタとして  
クリスマスツリーを作っていく中で  
SwiftUIの機能についていくつか見ていきました。  
  
SwiftUIは宣言的に小さい部品を作り  
それを組み合わせていくことができるので  
再利用性が高く読みやすいなと改めて感じました。  
  
