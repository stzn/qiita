## 経緯  
  
なんとなく知っているデザインパターンについて学ぶ機会があり、  
調べていくと  
  
あの時もっとこうしておけば良かった  
この困ったところはこういう風に書けるんだ  
  
と思うようなことが多々あったので、まとめていこうと考えました。  
  
## 気をつけたいこと  
  
「デザインパターン厨」という言葉もあるぐらい  
デザインパターンにこだわり過ぎることに気をつけ、  
デザインパターンはあくまでもわかりやすいコードを書くための良い参考例として学んでいきたいと思います。  
  
  
## Open-Closed Principle  
  
クラスは拡張に対して開いており、修正に対して閉じていなければならない  
  
既存のコードに修正を加えずに、拡張のためにコードを追加することで変更によるバグを起きにくくするという方針のことのようです。  
  
## 例  
  
検索条件を指定するという場面を考えたいと思います。  
  
```
enum Color {
    case green, red, blue
}

enum Size {
    case small, middle, large
}

enum Area {
    case domestic
    case foreign
}

struct Fruit {
    let name: String
    let color: Color
    let size: Size
    let price: Int
}

struct FruitFilter {
    
    func filterByColor(_ fruits: [Fruit], _ color: Color) -> [Fruit] {
        var result = [Fruit]()
        for p in fruits {
            if p.color == color {
                result.append(p)
            }
        }
        return result
    }
    
    func filterBySize(_ fruits: [Fruit], _ size: Size) -> [Fruit] {
        var result = [Fruit]()
        for p in fruits {
            if p.size == size {
                result.append(p)
            }
        }
        return result
    }
    
    func filterBySizeAndColor(_ fruits: [Fruit],
                              _ size: Size, _ color: Color) -> [Fruit] {
        var result = [Fruit]()
        for p in fruits {
            if (p.size == size) && (p.color == color) {
                result.append(p)
            }
        }
        return result
    }
}
```  
  
```
let apple = Fruit(name: "Apple", color: .red, size: .small, area: .domestic)
let melon = Fruit(name: "Melon", color: .green, size: .large, area: .foreign)
let grape = Fruit(name: "Grape", color: .blue, size: .small, area: .domestic)
    
let fruits = [apple, melon, grape]
    
let filter = FruitFilter()

for f in filter.filterBySize(fruits, .small) {
    print(" - \(f.name) is small ")
    // Apple is small
    // Grape is small
}
    
for f in filter.filterBySizeAndColor(fruits, .small, .red) {
    print(" - \(f.name) is small and red ")
    // Apple is small and red
}
```  
  
さらに、生産地による検索条件を追加しようとする場合、FruitFilterクラスに新たにメソッドを加えます。  
  
```
func filterByArea(_ fruits:[Fruit], _ area: Area) -> [Fruit] {
    var result = [Fruit]()
    for p in fruits {
        if (p.area == area) {
           result.append(p)
        }
    }
    return result
}
```  
  
では、生産地と色が検索条件の場合は？  
となるとまたメソッドが増え、クラスはどんどん大きくなっていってしまいます。  
  
小さいうちはいいですが、大きくなってくると見辛くなり、  
もしかしたら予期せぬ箇所への影響を与えることもあるかもしれません。  
  
  
そこでOpen-Closed Principleを適用してみたいと思います。  
  
## Specification(仕様)パターン  
  
[DDD本](https://www.amazon.co.jp/dp/B00GRKD6XU/ref=dp-kindle-redirect?_encoding=UTF8&btkr=1)で紹介されているパターンで  
  
・入力チェックなどのバリデーション(検証)  
・集合(配列、リストなど)から特定の要素を抽出する(選択)  
・条件を満たした新しいオブジェクトを作る(生成)  
  
などに用いられます。  
  
  
  
```
protocol Specification {
    
    associatedtype T
    func isSatisfied(_ item: T) -> Bool
}

protocol Filter {
    associatedtype T
    func filter<Spec: Specification>(_ items: [T], _ spec: Spec)
        -> [T] where Spec.T == T    
}

struct ColorSpecification: Specification {
    
    typealias T = Fruit

    let color: Color
    
    init(color: Color) {
        self.color = color
    }
    
    func isSatisfied(_ item: T) -> Bool {
        return item.color == color
    }
}

struct SizeSpecification: Specification {
    
    typealias T = Fruit
    
    let size: Size
    
    init(size: Size) {
        self.size = size
    }
    
    func isSatisfied(_ item: T) -> Bool {
        return item.size == size
    }
}

struct FruitFilter: Filter {
    typealias T = Fruit
    
    func filter<Spec>(_ items: [Fruit], _ spec: Spec) -> [Fruit]
        where Spec : Specification, FruitFilter.T == Spec.T {

        var result = [Fruit]()
        for i in items {
            if spec.isSatisfied(i) {
                result.append(i)
            }
        }
        return result
    }
}
```  
  
```
let apple = Fruit(name: "Apple", color: .red, size: .small, area: .domestic)

let melon = Fruit(name: "Melon", color: .green, size: .large, area: .foreign)

let grape = Fruit(name: "Grape", color: .blue, size: .small, area: .domestic)
    
let fruits = [apple, melon, grape]
let filter = FruitFilter()
    
let color = ColorSpecification(color: .blue)
    
for f in filter.filter(fruits, color) {
    print(" - \(f.name) is blue ")
    // Grape is blue
}
    
let size = SizeSpecification(size: .small)
for f in filter.filter(fruits, size) {
   print(" - \(f.name) is small ")
   // Apple is small
   // Grape is small
}
    
let colorAndSize = ColorAndSizeSpecfication(color, size)    
for f in filter.filter(fruits, colorAndSize) {
    print(" - \(f.name) is blue and small ")
    // Grape is blue and small
}
```  
  
複合条件の場合は以下のようにします。  
  
```
protocol AndSpecification: Specification {
    associatedtype SpecA: Specification
    associatedtype SpecB: Specification
    
    var specA: SpecA { get }
    var specB: SpecB { get }
    init(_ specA: SpecA, _ specB: SpecB)
}

extension AndSpecification where SpecA.T == T, SpecB.T == T {
    func isSatisfied(_ item: T) -> Bool {
        return specA.isSatisfied(item) && specB.isSatisfied(item)
    }
}

struct ColorAndSizeSpecfication: AndSpecification {

    typealias T = Fruit
    typealias SpecA = ColorSpecification
    typealias SpecB = SizeSpecification
    
    private let _specA: SpecA
    private let _specB: SpecB
    
    var specA: ColorSpecification { return _specA }
    var specB: SizeSpecification { return _specB }
        
    init(_ specA: SpecA, _ specB: SpecB) {
        _specA = specA
        _specB = specB
    }
}
```  
  
```
let colorAndSize = ColorAndSizeSpecfication(color, size)
    
for f in filter.filter(fruits, colorAndSize) {
    print(" - \(f.name) is blue and small ")
    // Grape is blue and small
}
```  
  
コード量はあまり変わらないかもしれませんが、  
今後条件の追加が起こったとしても  
既存のコードに変更を加えずに拡張が可能になり、  
オープンクローズド原則に沿った形になりました。  
  
  
  
## 悩んでいるところ  
  
AND条件を可変にしたい場合、以下のような実装が思い浮かびます。  
  
```
// Associated Typeを使って定義されたプロトコル（抽象型）は
// そのまま型として使用できないためType Eraser(型消去)をする必要がある
struct AnySpecification<T>: Specification {
    
    private let _isSatisfied: (T) -> Bool
    init<Spec: Specification>(_ spec: Spec) where Spec.T == T {
        self._isSatisfied = spec.isSatisfied
    }
    
    func isSatisfied(_ item: T) -> Bool {
        return _isSatisfied(item)
    }    
}

protocol VariadicAndSpecification: Specification {
    associatedtype Spec: Specification
    
    var specs: [Spec] { get }
    init(_ specs: Spec...)
}

extension VariadicAndSpecification where Spec.T == T {
    
    func isSatisfied(_ item: T) -> Bool {
        var isSatisfied = true
        for spec in specs {
            if !spec.isSatisfied(item) {
                isSatisfied = false
                break
            }
        }
        return isSatisfied
    }
}

struct ColorAndSizeAndAreaSpecification: VariadicAndSpecification {

    typealias T = Fruit
    typealias Spec = AnySpecification<T>

    private let _specs: [Spec]
    var specs: [Spec] { return _specs }

    init(_ specs: Spec...) {
        _specs = specs
    }
}

struct AreaSpecification: Specification {
    
    typealias T = Fruit

    let area: area
    init(area: Area) {
        self.area = area
    }
    
    func isSatisfied(_ item: T) -> Bool {
        return item.producingArea == area
    }
}
```  
  
```
let area = AreaSpecification(area: .domestic)
    
let colorAndSizeAndArea = ColorAndSizeAndAreaSpecification(
    AnySpecification(color),
    AnySpecification(size),
    AnySpecification(area)
)

for f in filter.filter(fruits, colorAndSizeAndArea) {
    print(" - \(f.name) is blue and small and domestic")
    // Grape is blue and small and domestic
}
```  
  
何かもっと簡単に書ける方法があるような気がします。。。  
アドバイスなどございましたら大変うれしいです。:bow:  
  
  
## まとめ  
  
デザインパターンという型を通して  
具体的な例の作成や過去に行っていた実装を書き換えてみるなどを行うことで  
だんだんと理解を深めることができました。  
  
使いどころを見出し、  
コーディング時の選択肢の１つとして使えるようにしていきたいと思います。  
