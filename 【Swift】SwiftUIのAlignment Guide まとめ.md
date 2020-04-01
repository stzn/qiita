SwiftUIを使用しているときに  
複数の`View`の配置を設定する場合  
alignmentを使用することが多くあると思います。  
  
しかし  
コンテナや`View`や`frame`メソッドなど  
色々な場所に指定できるため  
どれをどう使っていいのかがはっきりとしていませんでした。  
  
そこで  
今回はalignmentついてどうすればどう動くのかについて  
見ていきたいと思います。  
  
# Alignment Guideとは？  
  
同じコンテナに含まれる`View`との相対的な位置を決めます。  
垂直(Vertical)の配置と水平(Horizontal)の配置があります。  
  
※ コンテナとはVStackやGroupなどのViewを含むことができるViewのこと  
  
# 図でイメージしてみる  
  
まずはイメージをつかむために  
図で見てみます。  
  
例えば水平の関係を図で示します。  
  
<img width="778" alt="スクリーンショット 2019-09-28 8.53.55.png" src="/image/dbd40653-c3ff-49ce-ca8d-7463286ebe44.png">  
  
ViewAを基点(0)として考えると  
  
ViewAは  
ViewBから20離れた位置にあり  
ViewCから10離れた位置にあります。  
  
  
垂直の場合も同じようになります。  
  
<img width="631" alt="スクリーンショット 2019-09-28 9.00.37.png" src="/image/6b12e8c9-d526-0617-8bc8-e9ff3df09555.png">  
  
この２つから言えることは  
垂直のコンテナ(`VStack`)には**水平の配置(horizontal alignment)**が必要  
水平のコンテナ(`HStack`)には**垂直の配置(vertical alignment)**が必要  
ということです。  
  
こう考えると`VStack(alignment:)`と`HStack(alignment:)`のalignmentが  
何を示しているのかがわかりやすくなると思います。  
  
`ZStack`の場合は両方が必要になります。  
  
# コードから理解する  
  
ここからはコードから詳細を見ていきたいと思います。  
  
まず下記のコードの各値が何を示し  
どういう意味を持つのかを考えます。  
  
```swift

struct ContentView: View {
    var body: some View {
        VStack(alignment: .leading) {
            Text("Hello World....")
                .alignmentGuide(.leading, computeValue: { d in (d[explicit: .leading] ?? 0) })
                .alignmentGuide(.leading, computeValue: { d in d[.leading] })
                .multilineTextAlignment(.leading)
        }.frame(alignment: .leading)
    }
}
```  
  
複数のalignmentが登場していますが  
それぞれ意味が異なります。  
  
<img width="957" alt="スクリーンショット 2019-09-28 9.35.47.png" src="/image/8cc5be33-5c7a-adf5-abce-a3af08f40845.png">  
  
### Container Alignment  
  
２つの目的があります。  
  
１.  
`View`の  
どの`alignmentGuide(alignment:)`が無視され  
どの`alignmentGuide(alignment:)`が有効になるか  
を決める  
  
2.  
明示的にalignmentが指定されていないコンテナ内のViewに  
暗黙的にalignmentを設定する。  
  
  
### Alignment Guide  
  
Container Alignmentの値と  
Alignment Guideが一致している場合(上記は同じ`.leading`なので一致している)  
値は有効になりますが**異なっている場合は無視されます。**  
  
  
### Implicit Alignment  
  
guide(今回は`.leading`)に対するデフォルトの数値(`CGFloat`)。  
この`subscript`はReadOnlyです。  
  
```swift

@available(iOS 13.0, OSX 10.15, tvOS 13.0, watchOS 6.0, *)
public struct ViewDimensions {

    /// Accesses the value of the given guide.
    public subscript(guide: HorizontalAlignment) -> CGFloat { get }

    /// Accesses the value of the given guide.
    public subscript(guide: VerticalAlignment) -> CGFloat { get }
}

```  
  
※   
`ViewDimensions`  
`HorizontalAlignment`  
`VerticalAlignment`  
については後ほど紹介します。  
  
  
### Explicit Alignment  
  
guide(今回は`.leading`)に対する値で同じですが  
コード上で明示的に指定された値です。  
設定していなければnilになります。  
この`subscript`もReadOnlyです。  
  
```swift

@available(iOS 13.0, OSX 10.15, tvOS 13.0, watchOS 6.0, *)
public struct ViewDimensions {

    /// Returns the explicit value of the given alignment guide in this view, or
    /// `nil` if no such value exists.
    public subscript(explicit guide: HorizontalAlignment) -> CGFloat? { get }

    /// Returns the explicit value of the given alignment guide in this view, or
    /// `nil` if no such value exists.
    public subscript(explicit guide: VerticalAlignment) -> CGFloat? { get }
}
```  
  
  
### Frame Alignment  
  
`frame(alignment:)`を呼び出したコンテナに含まれる全てのViewが  
どのように配置されるのかをひとまとめに指定します。  
  
### Text Alignment  
  
複数行の`Text`に対して各行がどのように配置されるのかを指定します。  
全てのコンテナに含まれるViewが  
どのように配置されるのかをひとまとめに指定します。  
  
これらに関しては後ほどより詳細に紹介します。  
  
  
# ImplicitとExplicitの違い  
  
一番重要なポイントとして  
**コンテナ内の全てのViewはalignmentを持っています。**  
  
`.alignmentGuide()`を呼び出した場合は明示的(explicit)な値が  
特定していなければ暗黙的(implicit)な値が設定されます。  
  
implicitの値はコンテナの値から提供されます。  
`VStack(alignment: .leading)`など  
  
もしコンテナに何も指定しない場合は  
**デフォルト値の`.center`**が設定されます。  
  
  
## `ViewDimensions`  
  
alignmentは`computedValue`クロージャの中で`CGFloat`で指定します。  
この値は任意の値ですが  
単純に数値を指定するだけでは何をどう指定すれば良いのかが難しく感じます。  
  
そこで`ViewDimensions`が活用できます。  
  
`.alignmentGuide`は下記のような定義になっています。  
  
```swift

func alignmentGuide(_ g: HorizontalAlignment, 
                    computeValue: @escaping (ViewDimensions) -> CGFloat) -> some View
func alignmentGuide(_ g: VerticalAlignment, 
                    computeValue: @escaping (ViewDimensions) -> CGFloat) -> some View
```  
  
`computedValue`クロージャの引数で`ViewDimensions`という`struct`を受け取っています。  
この中にはViewに関する有用な情報が入っています。  
  
```swift

public struct ViewDimensions {

    public var width: CGFloat { get }

    public var height: CGFloat { get }

    public subscript(guide: HorizontalAlignment) -> CGFloat { get }

    public subscript(guide: VerticalAlignment) -> CGFloat { get }

    public subscript(explicit guide: HorizontalAlignment) -> CGFloat? { get }

    public subscript(explicit guide: VerticalAlignment) -> CGFloat? { get }
}
```  
  
高さと幅と`subscript`から各guideに対する値を取得できます。  
  
`subscript`から取得できる値がわかりづらいので  
下記のコードを使ってより詳しく見てみます。  
  
```swift

Text("Hello")
    .alignmentGuide(.leading, computeValue: { d in
        return d[.leading]
            + d.width / 3.0 - (d[explicit: .top] ?? 0)
    })
```  
  
## `HorizontalAlignment`  
  
`ViewDimension`が`HorizontalAlignment`を指定した場合  
下記の位置に対する値が取得できます。  
  
```swift

extension HorizontalAlignment {
    public static let leading: HorizontalAlignment
    public static let center: HorizontalAlignment
    public static let trailing: HorizontalAlignment
}
```  
  
  
```swift

d[.leading]
```  
これはimplicitな値を取得することができます。  
通常は  
**`.leading`が0  
`.center`が`2/width`  
`.trailing`が`width`**  
になります。  
  
  
```swift

d[explicit: .leading]
```  
  
これは明示的にコード上で指定した値を取得します。  
  
どういう時に必要かを考えると  
例えば`ZStack`を使っている時に  
`.leading`の位置を`.top`の位置から  
相対的に決めたい場合などに活用できます。  
  
設定していない場合はnilが返ってきます。  
  
## `VerticalAlignment`  
  
`VerticalAlignment`を指定した場合  
下記の位置に対する値が取得できます。  
  
```swift

extension VerticalAlignment {
    public static let top: VerticalAlignment
    public static let center: VerticalAlignment
    public static let bottom: VerticalAlignment
    public static let firstTextBaseline: VerticalAlignment
    public static let lastTextBaseline: VerticalAlignment
}
```  
  
テキストのベースラインを基にした値も取得できます。  
  
```swift

d[.top]
```  
これはimplicitな値を取得することができます。  
  
通常は  
**`.top`が0  
`.center`が`2/height`  
`.bottom`が`height`**  
になります。  
  
# `Alignment`型  
  
`ZStack`を使用している場合  
HorizontalとVerticalの２つを指定する必要があります。  
  
`Alignment`型は両方指定できる`struct`です。  
  
```swift

public struct Alignment : Equatable {

    public var horizontal: HorizontalAlignment

    public var vertical: VerticalAlignment

    @inlinable public init(horizontal: HorizontalAlignment, vertical: VerticalAlignment)

    public static let center: Alignment

    public static let leading: Alignment

    public static let trailing: Alignment

    public static let top: Alignment

    public static let bottom: Alignment

    public static let topLeading: Alignment

    public static let topTrailing: Alignment

    public static let bottomLeading: Alignment

    public static let bottomTrailing: Alignment
}
```  
  
下記のように使用します。  
  
```swift

ZStack(alignment: Alignment(horizontal: .leading, vertical: .top)) { ... }
```  
  
また簡単に設定できるように  
定数も定義されています。  
  
```swift

ZStack(alignment: .topLeading) { ... }
```  
  
## Container Alignment  
  
上記でContainer Alignmentの２つの目的を書きました。  
  
１.  
`View`の  
どの`alignmentGuide(alignment:)`が無視され  
どの`alignmentGuide(alignment:)`が有効になるか  
を決める  
  
2.  
明示的にalignmentが指定されていないコンテナ内の`View`に  
暗黙的にalignmentを設定する。  
  
これを下記のコードを参照して見ていきます。  
  
```swift

struct Implicit: View {
    @State private var alignment: HorizontalAlignment = .leading
    
    var body: some View {
        VStack {
            Spacer()
            
             VStack(alignment: alignment) {
               LabelView(title: "A", color: .green)
                .alignmentGuide(.leading, computeValue: { _ in 30 } )
                   .alignmentGuide(HorizontalAlignment.center, computeValue: { _ in 30 } )
                   .alignmentGuide(.trailing, computeValue: { _ in 90 } )
                 
               LabelView(title: "B", color: .red)
                   .alignmentGuide(.leading, computeValue: { _ in 90 } )
                   .alignmentGuide(HorizontalAlignment.center, computeValue: { _ in 30 } )
                   .alignmentGuide(.trailing, computeValue: { _ in 30 } )
                 
               LabelView(title: "C", color: .blue)
                
             }

            Spacer()
            HStack {
                Button("leading") { withAnimation(.easeInOut(duration: 2)) { self.alignment = .leading }}
                Button("center") { withAnimation(.easeInOut(duration: 2)) { self.alignment = .center }}
                Button("trailing") { withAnimation(.easeInOut(duration: 2)) { self.alignment = .trailing }}
            }
        }
    }
}

struct Implicit_Previews: PreviewProvider {
    static var previews: some View {
        Implicit()
    }
}

struct LabelView: View {
    let title: String
    let color: Color
    
    var body: some View {
        Text(title)
            .font(.title)
            .padding(10)
            .frame(width: 200, height: 40)
            .background(RoundedRectangle(cornerRadius: 8)
                .fill(LinearGradient(gradient: Gradient(colors: [color, .black]), startPoint: UnitPoint(x: 0, y: 0), endPoint: UnitPoint(x: 2, y: 1))))
    }
}
```  
  
Cは`alignmentGuide(alignment:)`を使用していないため  
implicitな値が設定され  
  
**`.leading`が0  
`.center`が`2/width`  
`.trailing`が`width`**  
になります。  
  
下記のような動きになります。  
  
#### `.leading`の場合  
  
![68747470733a2f2f71696974612d696d6167652d73746f72652e73332e61702d6e6f727468656173742d312e616d617a6f6e6177732e636f6d2f302f3231393936352f32303135626434352d333631332d613031382d616432652d3438323166623333346562612e706e67.png](/image/cefa5545-752f-371c-95a8-6873804d04b1.png)  
  
  
Cの左端が0になり  
AはCより30左に  
BはCより90左に  
配置されています。  
  
#### `.center`の場合  
  
![68747470733a2f2f71696974612d696d6167652d73746f72652e73332e61702d6e6f727468656173742d312e616d617a6f6e6177732e636f6d2f302f3231393936352f31386636636533362d306535652d643135382d626237302d3632616262316434376336332e706e67.png]  
(/image/683c48a5-3531-dddb-fc79-b0e67615bda6.png)  
  
Cの中心が0になり  
AもBもCより30中心が右にずれています。  
  
  
#### `.trailing`の場合  
  
![68747470733a2f2f71696974612d696d6167652d73746f72652e73332e61702d6e6f727468656173742d312e616d617a6f6e6177732e636f6d2f302f3231393936352f62663161616630662d373465342d346665642d383734392d3131343339323439656134312e706e67.png](/image/fdd68cb0-f9c2-a8e8-4820-268a11b4900d.png)  
  
Cの右端が0になり  
AはCより90左に  
BはCより30左に  
配置されています。  
  
## Frame Alignment  
  
これまで見てきたalignmentは  
`View`の相対関係と基にした位置を扱ってきましたが  
レイアウトシステムはこの後にコンテナ全体の配置を決めます。  
`frame(alignment:)`がこの役割を担っています。  
指定していない場合はデフォルト値の`.center`になります。  
  
しかし  
設定したとしても効果がない場合があります。  
  
それは  
**コンテナ内の`View`でコンテナのスペースが埋まっている場合です。**  
  
この時コンテナは動けません。  
  
## Multiline Text Alignment  
  
これはとてもシンプルです。  
  
下記のコードを例にします。  
  
```swift

struct TextAlignment: View {
    var body: some View {
        Text("Hello!\nNice to meet you!\nHow are you?")
            .multilineTextAlignment(.leading)
    }
}

struct TextAlignment_Previews: PreviewProvider {
    static var previews: some View {
        TextAlignment()
    }
}
```  
  
#### `.leading`の場合  
  
![スクリーンショット 2019-09-28 11.31.54.png](/image/332b0d08-6605-9689-0ea3-65b1f118ab5e.png)  
  
#### `.center`の場合  
  
![スクリーンショット 2019-09-28 11.32.12.png](/image/a75cc215-a04b-35b5-b293-73aeac0ccff6.png)  
  
#### `.trailing`の場合  
  
![スクリーンショット 2019-09-28 11.32.32.png](/image/937e6a63-2500-46c3-771b-7852902186a8.png)  
  
  
  
  
## Custom Alignment  
  
これまでは標準のalignmentについて見てきましたが  
独自のalignmentを作成することもできます。  
  
下記のコードから考えてみます。  
  
```swift

extension HorizontalAlignment {
    private enum WeirdAlignment: AlignmentID {
        static func defaultValue(in d: ViewDimensions) -> CGFloat {
            return d.height
        }
    }
    
    static let weirdAlignment = HorizontalAlignment(WeirdAlignment.self)
}
```  
  
独自のalignmentを実装する場合  
2つのものが必要になります。  
  
1. 水平(hotrizontal)か垂直(vertical)かを決める  
2. implicitなalignment用のデフォルト値を提供する  
  
下記のコードを使います。  
  
```swift

struct CustomAlignment: View {
    var body: some View {
        VStack(alignment: .weirdAlignment, spacing: 10) {
            
            Rectangle()
                .fill(Color.primary)
                .frame(width: 1)
                .alignmentGuide(.weirdAlignment, computeValue: { d in d[.trailing] })
            
            ColorLabel(label: "Monday", color: .red, height: 50)
            ColorLabel(label: "Tuesday", color: .orange, height: 70)
            ColorLabel(label: "Wednesday", color: .yellow, height: 90)
            ColorLabel(label: "Thursday", color: .green, height: 40)
            ColorLabel(label: "Friday", color: .blue, height: 70)
            ColorLabel(label: "Saturday", color: .purple, height: 40)
            ColorLabel(label: "Sunday", color: .pink, height: 40)
            
            Rectangle()
                .fill(Color.primary)
                .frame(width: 1)
                .alignmentGuide(.weirdAlignment, computeValue: { d in d[.leading] })
        }
    }
}

struct CustomAlignment_Previews: PreviewProvider {
    static var previews: some View {
        CustomAlignment()
    }
}

struct ColorLabel: View {
    let label: String
    let color: Color
    let height: CGFloat
    
    var body: some View {
        Text(label).font(.title).foregroundColor(.primary).frame(height: height).padding(.horizontal, 20)
            .background(RoundedRectangle(cornerRadius: 8).fill(color))
    }
}
```  
  
こうすると高さに合わせて  
全ての`View`の間の幅が変わります。  
  
#### `VStack`の`spacing`が10の場合  
  
![スクリーンショット 2019-09-28 11.52.43.png](/image/29e04222-15ff-beb1-5792-4a3061c53abf.png)  
  
#### `VStack`の`spacing`が40の場合  
  
![スクリーンショット 2019-09-28 11.53.52.png](/image/3bf67810-d149-e728-3067-1357aae40fca.png)  
  
  
## Custom Alignmentはいつ使う？  
  
異なる階層構造にある`View`同士を揃えたい場合に  
有効活用できます。  
  
  
下記のコードを見ていきます。  
  
```swift

extension VerticalAlignment {
    private enum MyAlignment : AlignmentID {
        static func defaultValue(in d: ViewDimensions) -> CGFloat {
            return d[.bottom]
        }
    }
    static let myAlignment = VerticalAlignment(MyAlignment.self)
}

struct DifferentViewHierarchy: View {
    @State private var selectedIdx = 3
    
    let days = ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday", "Sunday"]
    
    var body: some View {
            HStack(alignment: .myAlignment) {
                Image(systemName: "arrow.right.circle.fill")
                    .alignmentGuide(.myAlignment, computeValue: { d in d[VerticalAlignment.center] })
                    .foregroundColor(.green)

                VStack(alignment: .leading) {
                    ForEach(days.indices, id: \.self) { idx in
                        Group {
                            if idx == self.selectedIdx {
                                Text(self.days[idx])
                                    .transition(AnyTransition.identity)
                                    .alignmentGuide(.myAlignment, computeValue: { d in d[VerticalAlignment.center] })
                            } else {
                                Text(self.days[idx])
                                    .transition(AnyTransition.identity)
                                    .onTapGesture {
                                        withAnimation {
                                            self.selectedIdx = idx
                                        }
                                }
                            }
                        }
                    }
                }
            }
            .padding(20)
            .font(.largeTitle)
    }
}

struct DifferentViewHierarchy_Previews: PreviewProvider {
    static var previews: some View {
        DifferentViewHierarchy()
    }
}
```  
  
これは下記のような構造になっています。  
  
![スクリーンショット 2019-09-28 12.00.13.png](/image/c4849965-2385-d5b5-731f-3648deaf9e2d.png)  
  
`Image`と`Text`が同じ`HStack`にいるため  
それぞれに`.center`を指定することで高さを合わせています。  
同様にHStackに`.center`を指定することも必要です。  
  
ここでちょっと複雑なことが起きています。  
  
選択された`Text`以外はalignmentを明示的に指定していないため  
全ての`Text`が一番上のテキストに上乗せされるように思えます。  
  
しかし  
`VStack`を使用しているため  
全ての`View`は垂直に並ぶようになります。  
  
そこに明示的にalignmentを指定すると  
レイアウトシステムはそれを優先的に使用するため  
選択された`Text`は`Image`と揃うようになり  
他はそれに合わせて相対的に配置されるようになります。  
  
# 理解するためのサンプル  
  
https://gist.github.com/swiftui-lab/793ca53ad1f2f0d7eb07aa23b54d9cbf  
  
alignmentを理解するためのサンプルとして  
とても参考になります。  
  
"Show in Two Phases"をONにすると  
`alignmentGuide`がどう移動して  
その後`View`が実施にどう移動するのかを分けてみることができます。  
  
また  
作者は下記について確認してみて欲しいと言っています。  
  
- コンテナのスペースを`View`で埋めている場合と余裕がある場合の`frame(alignment:)`の動きの違い  
- コンテナのguideとViewのguideが異なる場合に何も変化しないこと  
- `.leading`などを指定した場合と数値を指定した場合の違い  
- マイナスや`View`の幅以上の値を設定した場合の動き   
  
  
# まとめ  
  
alignmentについて見てきました。  
  
一見複雑ですが  
どういうルールでalignmentが決まるのかがわかれば  
比較的わかりやすいのではないかと思いました。  
  
あとはやはり実際に動かしてみて  
どういう動きをするのかを見ていくのが一番良いですね😃  
  
何か間違いなどございましたら  
教えて頂けますとうれしいです🙇🏻‍♂️  
  
# 参考記事  
  
こちらの記事を主に参考にさせていただきました。  
https://swiftui-lab.com/alignment-guides/  
