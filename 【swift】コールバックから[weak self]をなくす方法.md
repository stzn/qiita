下記の記事を読んでいて、こんな方法もあるんだなと勉強になったのでメモします。  
https://medium.com/anysuggestion/preventing-memory-leaks-with-swift-compile-time-safety-49b845df4dc6  
  
  
## 非同期処理の戻りをどう処理するか？  
  
主に２つのパターンがあると考えています。  
※処理内容は適当です。  
  
### ①Delegateパターン  
  
```Swift

struct Item: Decodable {
    let name: String
}

protocol LoadDataDelegate: class {
    func loaded(data: [Item])
}

class LoadController {
    
    weak var delegate: LoadDataDelegate?
    
    func load(url: URL) {
        let task = URLSession.shared.dataTask(with: url) { data, response, error in
            let decoder: JSONDecoder = JSONDecoder()
            do {
                let items = try decoder.decode([Item].self, from: data!)
                self.delegate?.loaded(data: items)
            } catch {
                // エラー
            }
        }
        task.resume()
    }
}

class Controller {
    let loader = LoadController()
    
    var items: [Item]?
    
    init() {
        loader.delegate = self
    }

    func dataLoad() {
        loader.load(url: URL(string: "XXXXX")!)
    }

}

extension Controller: LoadDataDelegate {
    func loaded(data: [Item]) {
        self.items = data
    }
}

```  
  
### ②クロージャパターン  
  
```Swift

class LoadController {
    
    var loaded: (([Item]) -> Void)?
    
    func load(url: URL) {
        let task = URLSession.shared.dataTask(with: url) { data, response, error in
            let decoder: JSONDecoder = JSONDecoder()
            do {
                let items = try decoder.decode([Item].self, from: data!)
                self.loaded?(items)
            } catch {
                // エラー
            }
        }
        task.resume()
    }
}

class Controller {
    let loader = LoadController()
    
    var items: [Item]?
    
    init() {
        // メモリーリークを防ぐためにselfの強参照を避ける
        loader.loaded = { [weak self] data in
            self?.items = data
        }
    }
    
    func dataLoad() {
        loader.load(url: URL(string: "XXXXX")!)
    }
}

```  
  
個人的にはクロージャパターンをよく使うのですが、  
[weak self]を何度も書くのがなんとも面倒だなと常日頃思っていました。  
また、省略していますが大体が  
```Swift
guard let stringSelf = self else {
    // 何かしらログやエラー処理をする
    return
}
```  
といった形でguard文を書きます。  
  
冒頭の記事でも言及されていましたが、  
APIガイドラインにも「メソッドやプロパティの宣言は一度だけ、行いそれを繰り返し使う」  
と書かれており、メモリーリークを防ぐという責務は呼ばれる側が持つべきだというようなこともおっしゃっています。(解釈間違っていたらすいません)  
  
その問題を冒頭の記事の中では以下のようなクラスを作成することで解決しています。  
  
  
```Swift

struct DelegatedCall<Input> {
    
    private(set) var callback: ((Input) -> Void)?
    
    mutating func delegate<Object : AnyObject>(to object: Object, with callback: @escaping (Object, Input) -> Void) {

        // ここで[weak object]としている
        self.callback = { [weak object] input in
            guard let object = object else {
                return
            }
            callback(object, input)
        }
    }
}
```  
  
```Swift
class LoadController {
    
    var loaded = DelegatedCall<[Item]>()

    func load(url: URL) {
        let task = URLSession.shared.dataTask(with: url) { data, response, error in
            let decoder: JSONDecoder = JSONDecoder()
            do {
                let items = try decoder.decode([Item].self, from: data!)
                self.loaded.callback?(items)
            } catch {
                // エラー
            }
        }
        task.resume()
    }
}

class Controller {
    let loader = LoadController()
    
    var items: [Item]?
    
    init() {
        loader.loaded.delegate(to: self) { (self, data) in

　　　　　　　// [weak self]がいらない
            self.items = data
        }
    }
    
    func dataLoad() {
        loader.load(url: URL(string: "XXXXX")!)
    }
}
```  
こうして呼ばれる側で[weak self]を毎回書かずに済むようになるというでした。  
  
上記のような方法はFoundationのUndoManagerなどでも用いられているようです。  
https://developer.apple.com/documentation/foundation/undomanager/2427208-registerundo  
  
記事中で紹介されていたのはあくまで簡単な例で筆者はこれをライブラリ形式にまとめています。  
https://github.com/dreymonde/Delegated  
  
  
## まとめ  
  
[weak self]はいつも鬱々として書いていたので、これは有用なのかなと何となく感じています。ただ、使い始めると見えてくる部分がたくさん出てくると思いますので、色々と試してみます。  
  
また、今回の一番の気づき(反省)として  
冒頭の記事の筆者がFoundationから発想を得ているように、  
FoundationやSwiftの公式リポジトリなどを見ることで  
新しい考え方や実装方法を学べる機会は思っている以上にたくさんありそうだと改めて感じました。  
当たり前かもしれませんが、こういった基本的なものをもっとよく見ていこうと思います。  
  
  
