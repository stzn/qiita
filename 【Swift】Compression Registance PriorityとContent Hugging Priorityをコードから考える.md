  
## 経緯  
  
レイアウトを設定していると  
Compression Registance PriorityとContent Hugging Priorityはよく使いますが、  
どっちがどっちだろうと混同することが多く、  
優先順に関しても理解があいまいであったため  
メモとして記録します。  
  
### Compression Registance Priority  
・コンテンツのつぶれにくさ  
・文字列がコントローラの幅よりも長いとき末尾を「...」とならないように値を増加する  
・デフォルトの優先度の値は750  
  
### Content Hugging Priority  
・余白のできにくさ  
・文字列がコントローラの幅よりも短いとき末尾の空白を減らすために値を増加する  
・デフォルトの優先度の値は250  
  
  
  
わかりずらいのがContent Hugging Priorityの方で、  
末尾の空白を**減らしたい**場合に値を**増加する**と逆になっています。  
  
今回、この値を変更することでどのように値が変わっていくのかを色々と試してみました。  
  
検索してみると意外と出てこなかったのでコードで優先度を変更しています。  
  
  
### 初期表示  
  
```Swift
import UIKit

class ViewController4: UIViewController {
    
   let label1: UILabel = {
        let l = UILabel()
        l.font = UIFont.systemFont(ofSize: 16)
        l.numberOfLines = 1
        l.backgroundColor = .gray
        l.text = "aaaaaaaaaaaaaaaaaaaaaaaaaaa"
        l.translatesAutoresizingMaskIntoConstraints = false
        return l
    }()
    
    let label2: UILabel = {
        let l = UILabel()
        l.font = UIFont.systemFont(ofSize: 16)
        l.numberOfLines = 1
        l.backgroundColor = .gray
        l.text = "bbbbbbbbbbbbbbbbbbbbbbbbbbb"
        l.translatesAutoresizingMaskIntoConstraints = false
        return l
    }()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        self.view.addSubview(label1)
        self.view.addSubview(label2)
        
        label1.centerYAnchor.constraint(equalTo: view.centerYAnchor).isActive = true
        label1.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 16).isActive = true
        label2.leadingAnchor.constraint(equalTo: label1.trailingAnchor, constant: 16).isActive = true
        label2.centerYAnchor.constraint(equalTo: view.centerYAnchor).isActive = true
        view.trailingAnchor.constraint(greaterThanOrEqualTo: label2.trailingAnchor, constant: 16).isActive = true
    }
}

```  
  
優先度を意識せずに制約を設定しており、Anchorにはデフォルトの優先度1000が設定されます。  
  
<img width="280" alt="Simulator Screen Shot - iPhone SE - 2018-03-29 at 09.36.23.png" src="/image/89bf2357-a30b-2f70-e4f0-a1657ed75ab3.png">  
  
この場合、ContentCompressionResistanceにはデフォルトの750が設定され、  
label1とlabel2には同じ値が入っていますが、左側が優先的に表示されるようになりました。(左側が優先される理由まで調べきれていないので、時間があれば調べたいです。)  
  
  
### label2にContentCompressionResistanceを設定  
  
```Swift
let label2: UILabel = {
    let l = UILabel()
    l.font = UIFont.systemFont(ofSize: 16)
    l.numberOfLines = 1

    // ***********ContentCompressionResistanceを751に設定****************
    l.setContentCompressionResistancePriority(UILayoutPriority.init(751), for: .horizontal)
    l.backgroundColor = .gray
    l.text = "bbbbbbbbbbbbbbbbbbbbbbbbbb"
    l.translatesAutoresizingMaskIntoConstraints = false
    return l
}()
```  
  
意図的にlabel2のContentCompressionResistanceをlabel1よりも高くしました。  
  
<img width="280" alt="Simulator Screen Shot - iPhone SE - 2018-03-29 at 09.37.49.png" src="/image/3c1afbf7-1681-59fe-1ac3-01058b30392b.png">  
  
  
結果としてlabel2が優先的に表示されるようになりました。  
  
  
### label1にContentCompressionResistanceを設定  
  
```Swift
let label1: UILabel = {
    let l = UILabel()
    l.font = UIFont.systemFont(ofSize: 16)
    l.numberOfLines = 1
    l.backgroundColor = .gray
    
    // ***********ContentCompressionResistanceを751に設定****************
    l.setContentCompressionResistancePriority(UILayoutPriority.init(751), for: .horizontal)
    l.text = "aaaaaaaaaaaaaaaaaaaaaaaaaaa"
    l.translatesAutoresizingMaskIntoConstraints = false
    return l
}()

```  
  
先ほどと同じ値でlabel1にもContentCompressionResistanceを設定しました。  
  
<img width="320" alt="Simulator Screen Shot - iPhone SE - 2018-03-29 at 09.38.47.png" src="/image/bdcbc1dc-75cc-5fc2-faf0-6da1ef160672.png">  
  
一番最初の状態と同様にlabel1がlabel2よりも優先的に表示されるようになりました。  
  
  
### label1にWidth制約を設定  
  
```Swift
override func viewDidLoad() {
    super.viewDidLoad()
    
    self.view.addSubview(label1)
    self.view.addSubview(label2)
    
    // ***********幅制約を設定****************
    label1.widthAnchor.constraint(equalToConstant: 100).isActive = true
    label1.centerYAnchor.constraint(equalTo: view.centerYAnchor).isActive = true
    label1.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 16).isActive = true
    label2.leadingAnchor.constraint(equalTo: label1.trailingAnchor, constant: 16).isActive = true
    label2.centerYAnchor.constraint(equalTo: view.centerYAnchor).isActive = true
    view.trailingAnchor.constraint(greaterThanOrEqualTo: label2.trailingAnchor, constant: 16).isActive = true
}
```  
label1に固定の幅を設定しました。  
  
  
<img width="280" alt="Simulator Screen Shot - iPhone SE - 2018-03-29 at 09.41.20.png" src="/image/4d961ca1-5c10-f1b3-e7e4-c207e9c09226.png">  
  
Anchorに対するデフォルトの優先度は1000のため、優先的にlabel1を幅100まで表示し、その後label2を表示できる限り表示するようになりました。  
  
  
### label1にWidth制約の優先度をlabel2のContentCompressionResistanceよりも下げる  
  
```Swift
override func viewDidLoad() {
    super.viewDidLoad()
    
    self.view.addSubview(label1)
    self.view.addSubview(label2)
    
    // ************ width制約を750に ***************
    let widthConstraint = label1.widthAnchor.constraint(equalToConstant: 100)
    widthConstraint.priority = UILayoutPriority.init(750)
    widthConstraint.isActive = true
    label1.centerYAnchor.constraint(equalTo: view.centerYAnchor).isActive = true
    label1.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 16).isActive = true
    label2.leadingAnchor.constraint(equalTo: label1.trailingAnchor, constant: 16).isActive = true
    label2.centerYAnchor.constraint(equalTo: view.centerYAnchor).isActive = true
    view.trailingAnchor.constraint(greaterThanOrEqualTo: label2.trailingAnchor, constant: 16).isActive = true
}

```  
  
あえてlabel1のwidthAnchorの優先度をlabel2のContentCompressionResistanceよりも下げました。  
  
<img width="280" alt="Simulator Screen Shot - iPhone SE - 2018-03-29 at 09.47.21.png" src="/image/6b7f3622-01fc-26fa-19f9-46be1709df63.png">  
  
label2の表示が優先されるようになりました。  
  
  
### label1にWidth制約の優先度をlabel2のContentCompressionResistanceよりも上げる  
  
```Swift
override func viewDidLoad() {
    super.viewDidLoad()
    
    self.view.addSubview(label1)
    self.view.addSubview(label2)
    
    let widthConstraint = label1.widthAnchor.constraint(equalToConstant: 100)

    // ************ width制約を752に ***************
    widthConstraint.priority = UILayoutPriority.init(752)
    widthConstraint.isActive = true
    label1.centerYAnchor.constraint(equalTo: view.centerYAnchor).isActive = true
    label1.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 16).isActive = true
    label2.leadingAnchor.constraint(equalTo: label1.trailingAnchor, constant: 16).isActive = true
    label2.centerYAnchor.constraint(equalTo: view.centerYAnchor).isActive = true
    view.trailingAnchor.constraint(greaterThanOrEqualTo: label2.trailingAnchor, constant: 16).isActive = true
}

```  
  
上記とは逆にします。  
  
<img width="280" alt="Simulator Screen Shot - iPhone SE - 2018-03-29 at 09.49.24.png" src="/image/e05ee5f5-95be-99aa-14dd-526e1a49a1a3.png">  
  
label1の表示が優先されるようになりました。  
  
  
次に文字列が少ない場合を試してみます。  
  
  
### 初期表示変更  
  
```Swift
class ViewController4: UIViewController {
    
   let label1: UILabel = {
        let l = UILabel()
        l.font = UIFont.systemFont(ofSize: 16)
        l.numberOfLines = 1
        l.backgroundColor = .gray
        l.text = "a"
        l.translatesAutoresizingMaskIntoConstraints = false
        return l
    }()
    
    let label2: UILabel = {
        let l = UILabel()
        l.font = UIFont.systemFont(ofSize: 16)
        l.numberOfLines = 1
        l.backgroundColor = .gray
        l.text = "b"
        l.translatesAutoresizingMaskIntoConstraints = false
        return l
    }()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        self.view.addSubview(label1)
        self.view.addSubview(label2)
        
        label1.centerYAnchor.constraint(equalTo: view.centerYAnchor).isActive = true
        label1.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 16).isActive = true
        label2.leadingAnchor.constraint(equalTo: label1.trailingAnchor, constant: 16).isActive = true
        label2.centerYAnchor.constraint(equalTo: view.centerYAnchor).isActive = true
        view.trailingAnchor.constraint(greaterThanOrEqualTo: label2.trailingAnchor, constant: 16).isActive = true
    }
}

```  
  
特にAnchor以外は制約を設定していません。  
  
<img width="280" alt="Simulator Screen Shot - iPhone SE - 2018-03-29 at 09.54.00.png" src="/image/f9ee4794-4601-2450-322e-e04b58083558.png">  
  
各Labelのintrinsic content sizeからLabelの幅が決定され、それに従って表示されています。  
  
  
### label1にwidth制約を設定  
  
```Swift
override func viewDidLoad() {
    super.viewDidLoad()
    
    self.view.addSubview(label1)
    self.view.addSubview(label2)
    
    // ************ width制約を設定 ***************
    label1.widthAnchor.constraint(equalToConstant: 100).isActive = true // デフォルトは1000
    label1.centerYAnchor.constraint(equalTo: view.centerYAnchor).isActive = true
    label1.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 16).isActive = true
    label2.leadingAnchor.constraint(equalTo: label1.trailingAnchor, constant: 16).isActive = true
    label2.centerYAnchor.constraint(equalTo: view.centerYAnchor).isActive = true
    view.trailingAnchor.constraint(greaterThanOrEqualTo: label2.trailingAnchor, constant: 16).isActive = true
}

```  
  
label1に固定の幅を設定します。  
  
<img width="280" alt="Simulator Screen Shot - iPhone SE - 2018-03-29 at 10.03.20.png" src="/image/64f64a7e-3cc9-83cf-6158-cbac9d0d99af.png">  
  
ContentHuggingPriorityのデフォルトの優先度は250なのでwidthAnchorのデフォルト優先度の1000が優先され、UILabelが固定の幅まで表示されるようになっています。  
  
  
### label1にContentHuggingPriorityを設定、width制約の優先度をContentHuggingPriorityより下げる  
  
```Swift
let label1: UILabel = {
    let l = UILabel()
    l.font = UIFont.systemFont(ofSize: 16)
    l.numberOfLines = 1
    l.backgroundColor = .gray
    l.setContentHuggingPriority(UILayoutPriority.init(250), for: .horizontal)
    l.text = "a"
    l.translatesAutoresizingMaskIntoConstraints = false
    return l
}()

override func viewDidLoad() {
    super.viewDidLoad()
    
    self.view.addSubview(label1)
    self.view.addSubview(label2)
    
    // *********優先度を249に設定************
    let widthConstraint = label1.widthAnchor.constraint(equalToConstant: 100)
    widthConstraint.priority = UILayoutPriority.init(249)
    widthConstraint.isActive = true
    label1.centerYAnchor.constraint(equalTo: view.centerYAnchor).isActive = true
    label1.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 16).isActive = true
    label2.leadingAnchor.constraint(equalTo: label1.trailingAnchor, constant: 16).isActive = true
    label2.centerYAnchor.constraint(equalTo: view.centerYAnchor).isActive = true
    view.trailingAnchor.constraint(greaterThanOrEqualTo: label2.trailingAnchor, constant: 16).isActive = true
}
```  
  
label1のwidthAnchorをlabel1のContentHuggingPriorityよりも小さくしました。  
  
<img width="280" alt="Simulator Screen Shot - iPhone SE - 2018-03-29 at 10.06.18.png" src="/image/48648d46-9d0d-fa95-bcd3-d32af7685e89.png">  
  
  
こうすると、label1の末尾の空白が取り除かれ、文字列の幅と同じように表示されるようになりました。  
  
  
### label2のContentCompressionResistancePriorityを一番高い優先度にする  
  
```Swift
class ViewController4: UIViewController {
    
   let label1: UILabel = {
        let l = UILabel()
        l.font = UIFont.systemFont(ofSize: 16)
        l.numberOfLines = 1
        l.backgroundColor = .gray

        // 優先度250
        l.setContentHuggingPriority(UILayoutPriority.init(250), for: .horizontal)
        l.text = "a"
        l.translatesAutoresizingMaskIntoConstraints = false
        return l
    }()
    
    let label2: UILabel = {
        let l = UILabel()
        l.font = UIFont.systemFont(ofSize: 16)
        l.numberOfLines = 1
        
        // 優先度751        
l.setContentCompressionResistancePriority(UILayoutPriority.init(751), for: .horizontal)
        l.backgroundColor = .gray
        l.text = "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb"
        l.translatesAutoresizingMaskIntoConstraints = false
        return l
    }()
   
    override func viewDidLoad() {
        super.viewDidLoad()
        
        self.view.addSubview(label1)
        self.view.addSubview(label2)
        
        // 優先度249
        let widthConstraint = label1.widthAnchor.constraint(equalToConstant: 100)
        widthConstraint.priority = UILayoutPriority.init(249)
        widthConstraint.isActive = true
        label1.centerYAnchor.constraint(equalTo: view.centerYAnchor).isActive = true
        label1.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 16).isActive = true


        // 優先度750
        let widthConstraint2 = label2.widthAnchor.constraint(equalToConstant: 100)
        widthConstraint2.priority = UILayoutPriority.init(750)
        widthConstraint2.isActive = true

        label2.leadingAnchor.constraint(equalTo: label1.trailingAnchor, constant: 16).isActive = true
        label2.centerYAnchor.constraint(equalTo: view.centerYAnchor).isActive = true
        view.trailingAnchor.constraint(greaterThanOrEqualTo: label2.trailingAnchor, constant: 16).isActive = true
    }
}
```  
  
上記のように優先度を設定すると、  
  
<img width="280" alt="Simulator Screen Shot - iPhone SE - 2018-03-29 at 10.29.57.png" src="/image/84bb24e6-5a27-e48a-298e-30d0675e4b2e.png">  
  
優先度順で考えると、  
1.Anchorを使った制約がデフォルトの1000になっているためにまずAnchorの制約が設定される。  
2.label2へのContentCompressionResistancePriorityからlabel2を極力大きく表示しようとする。  
  
となるため、2の時点でlabel2の表示が入りきらないために、label1の表示は消えてしまいました。(ここちょっと自信ないです。。。)  
  
  
### label1のContentHuggingPriority→label1のWidthAnchor->label2のContentCompressionResistancePriority→label2のWidthAnchorの順に優先度を設定する  
  
```Swift
class ViewController4: UIViewController {
    
   let label1: UILabel = {
        let l = UILabel()
        l.font = UIFont.systemFont(ofSize: 16)
        l.numberOfLines = 1
        l.backgroundColor = .gray

        // 優先度250
        l.setContentHuggingPriority(UILayoutPriority.init(250), for: .horizontal)
        l.text = "a"
        l.translatesAutoresizingMaskIntoConstraints = false
        return l
    }()
    
    let label2: UILabel = {
        let l = UILabel()
        l.font = UIFont.systemFont(ofSize: 16)
        l.numberOfLines = 1
        
        // 優先度248
l.setContentCompressionResistancePriority(UILayoutPriority.init(248), for: .horizontal)
        l.backgroundColor = .gray
        l.text = "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb"
        l.translatesAutoresizingMaskIntoConstraints = false
        return l
    }()

    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        self.view.addSubview(label1)
        self.view.addSubview(label2)
　　　　　
　　　　　// 優先度249        
        let widthConstraint = label1.widthAnchor.constraint(equalToConstant: 100)
        widthConstraint.priority = UILayoutPriority.init(249)
        widthConstraint.isActive = true
        label1.centerYAnchor.constraint(equalTo: view.centerYAnchor).isActive = true
        label1.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 16).isActive = true

　　　　　// 優先度247
        let widthConstraint2 = label2.widthAnchor.constraint(equalToConstant: 100)
        widthConstraint2.priority = UILayoutPriority.init(247)
        widthConstraint2.isActive = true
        label2.leadingAnchor.constraint(equalTo: label1.trailingAnchor, constant: 16).isActive = true
        label2.centerYAnchor.constraint(equalTo: view.centerYAnchor).isActive = true
        view.trailingAnchor.constraint(greaterThanOrEqualTo: label2.trailingAnchor, constant: 16).isActive = true
    }
}
```  
  
優先度順で考えると、  
1.Anchorを使った制約がデフォルトの1000になっているためにまずAnchorの制約が設定される。  
2.label1へのContentHuggingPriorityからlabel1を文字列の幅で表示しようとする。  
3.label2へのContentCompressionResistancePriorityでlabel2の文字列を最大限表示しようとする  
  
となるため、  
  
<img width="280" alt="Simulator Screen Shot - iPhone SE - 2018-03-29 at 10.48.02.png" src="/image/f6172d3e-8aab-4125-5eba-0fbea3a5fa3e.png">  
  
のようにlabel1は文字列の幅、label2は表示できる限りまで幅を広げて表示されるようになりました。  
  
  
## まとめ  
  
IBを使っているとかちゃかちゃと値の増減をしてなんとなくうまくいったのかなくらいの意識で行なっていましたが、コードで書いてみることで優先度のレイアウトに対する影響への理解をやや高めることができました。  
  
  
実際に試してやってみた結果ですので、  
何か間違っている箇所などのご指摘がございましたらぜひ教えてください:bow_tone1:  
