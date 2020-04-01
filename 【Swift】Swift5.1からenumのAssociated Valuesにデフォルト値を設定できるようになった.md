# Swiftのenum  
  
**Associated Values**という付属の値をenumの要素に含めることができます。  
  
Associated Valuesは初期化時に値を設定する必要がありましたが  
Swift5.1よりデフォルト値を設定することができるようになりました。  
  
今回は公式のガイドに載っている例を使って見てみました。  
https://docs.swift.org/swift-book/LanguageGuide/Enumerations.html  
  
検証環境: Xcode11 Beta3  
  
## これまで  
  
下記ように全ての値を指定する必要があります。  
  
```swift

enum Barcode {
    case upc(Int, Int, Int, Int)
    case qrCode(String)
}

var productBarcode = Barcode.upc(8, 85909, 51226, 3)

print(productBarcode) // upc(8, 85909, 51226, 3)
```  
  
## Swift5.1  
  
```swift

enum Barcode {
    case upc(a: Int = 1, Int, Int, Int)
    case qrCode(String)
}

var productBarcode = Barcode.upc(85909, 51226, 3)

print(productBarcode) // upc(a: 1, 85909, 51226, 3)
```  
  
とデフォルトの値を設定することができるようになります。  
ただし  
**ラベル引数を設定しないとコンパイルエラーになりました。**  
  
  
例えば下記のように間を抜かしても大丈夫なようです。  
  
```swift

enum Barcode {
    case upc(a:Int = 1, Int, Int, d: Int = 4)
    case qrCode(String)
}

var productBarcode = Barcode.upc(85909, 51226)

print(productBarcode) // upc(a: 1, 85909, 51226, d: 4)
```  
  
少しの変化かもしれませんが  
これで冗長な記述が少し減ればうれしいですね:smiley:  
