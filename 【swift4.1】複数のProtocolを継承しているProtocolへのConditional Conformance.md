何気なく書いていて知らなかったことに気がついたので忘れないようにメモします。  
  
## 経緯  
  
下記のようなprotocolに適合したValueを持つstructを定義します。  
  
```Swift
protocol Identifiable {
    associatedtype RawIdentifier: Codable = String
    var id: Identifier<Self> { get }
}

struct Identifier<Value: Identifiable> {
    let rawValue: Value.RawIdentifier
    init(rawValue: Value.RawIdentifier) {
        self.rawValue = rawValue
    }
}
```  
  
これを文字列から初期化できるようにExpressibleByStringLiteralに適合しようとします。  
  
```Swift
extension Identifier: ExpressibleByStringLiteral
where Value.RawIdentifier == String {
    
    typealias StringLiteralType = String
    
    init(stringLiteral value: String) {
        rawValue = value
    }
}
```  
  
そうすると下記のようなエラーが出ます。  
  
```Swift
error: test3.playground:9:1: error: type 'Identifier<Value>' does not conform to protocol 'ExpressibleByExtendedGraphemeClusterLiteral'
extension Identifier: ExpressibleByStringLiteral
^
```  
  
※ExpressibleByStringLiteralはExpressibleByExtendedGraphemeClusterLiteralを継承し、ExpressibleByExtendedGraphemeClusterLiteralは  
ExpressibleByUnicodeScalarLiteralを継承しています。  
  
「あれConditional Conformanceは？」と思ったのですが、下記に記載がありました。  
[https://github.com/apple/swift-evolution/blob/master/proposals/0143-conditional-conformances.md#implied-conditional-conformances](https://github.com/apple/swift-evolution/blob/master/proposals/0143-conditional-conformances.md#implied-conditional-conformances)  
[https://forums.swift.org/t/conditional-conformance-with-protocol-inheritance/11262/11](https://forums.swift.org/t/conditional-conformance-with-protocol-inheritance/11262/11)  
[https://forums.swift.org/t/amending-se-0143-conditional-conformances-dont-imply-other-conformances/11017/2](https://forums.swift.org/t/amending-se-0143-conditional-conformances-dont-imply-other-conformances/11017/2)  
  
  
いまいち全体の要領がつかめなかったのですが、議論の中で出てきている  
  
```
the heart of the issue is 
that you may not have more than one extension 
that conforms a type to a particular protocol. 
Otherwise it’s ambiguous 
which versions of the protocol methods should be called 
when you invoke protocol methods on that type.
```  
  
から、  
「適用できるメソッドが複数ある場合、コンパイラーが何を選べばいいのわからない」ということのようです。(合っているのでしょうか？)  
  
  
## 結論  
  
下記のようにProtocolを明示して、typealiasを追加することでエラーは解消されました。  
  
```Swift
extension Identifier: ExpressibleByStringLiteral, ExpressibleByExtendedGraphemeClusterLiteral, ExpressibleByUnicodeScalarLiteral
where Value.RawIdentifier == String {
    
    typealias ExtendedGraphemeClusterLiteralType = String
    typealias UnicodeScalarLiteralType = String
    typealias StringLiteralType = String
    
    init(stringLiteral value: String) {
        rawValue = value
    }
}
```  
  
一つ勉強になりました。  
  
## 補足  
  
上記の実装は  
https://www.swiftbysundell.com/posts/type-safe-identifiers-in-swift  
の内容を色々触っていたときに気がつき、著者の[@johnsundell](https://twitter.com/johnsundell)さんにもTwitterで聞いてみたところ必要だとの回答をもらいました。  
加えて、元々ExtendedGraphemeClusterLiteralTypeとUnicodeScalarLiteralTypeのinitメソッドも追加していたのですが、typealiasだけで良いということも教えてもらいました:smiley:  
