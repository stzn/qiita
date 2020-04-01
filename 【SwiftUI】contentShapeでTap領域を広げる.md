  
SwiftUIで  
あるViewに  
`onTapGesture`を設定した際  
中にある個々のViewをタップすることはできますが  
その他の領域はタップしても反応しません。  
  
例えば下記ような`VStack`に対して`onTapGesture`を設定します。  
  
```swift

import SwiftUI

struct ContentView: View {

    @State private var checked = false

    var body: some View {

        VStack(spacing: 20) {
            Image(systemName: checked ? "checkmark.seal.fill" : "checkmark.seal")
                .foregroundColor(checked ? .green : .gray)
            Text("画像とテキストのみTapできる")
        }
        .frame(width: 300, height: 200)
        .padding()
        .background(RoundedRectangle(cornerRadius: 20).stroke(Color.green, lineWidth: 1))
        // 👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻
        .onTapGesture {
            self.checked.toggle()
        }
        .font(.title)
    }
}

struct ContentView_Previews: PreviewProvider {
    static var previews: some View {
        ContentView()
    }
}
```  
  
上記の場合ですと  
VStackの中の画像とテキストにしかタップできません。  
  
![cannottap720.gif](/image/97b25255-96a5-f4c2-c5fb-9405afb52a0a.gif)  
  
  
そんな時に活用できるのが`contentShape`です。  
https://developer.apple.com/documentation/swiftui/vstack/3278318-contentshape  
  
これを実行すると自身のhitテストを実行してくれるViewを返します。  
  
今回は枠内をタップできるように設定します。  
  
```swift


struct ContentView: View {

    @State private var checked = false

    var body: some View {

        VStack(spacing: 20) {
            Image(systemName: checked ? "checkmark.seal.fill" : "checkmark.seal")
                .foregroundColor(checked ? .green : .gray)
            Text("枠の中は全部Tapできる")
        }
        .frame(width: 300, height: 200)
        .padding()
        .background(RoundedRectangle(cornerRadius: 20).stroke(Color.green, lineWidth: 1))
        // 👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻👇🏻
        .contentShape(RoundedRectangle(cornerRadius: 20)) 
        .onTapGesture {
            self.checked.toggle()
        }
        .font(.title)
    }
}

struct ContentView_Previews: PreviewProvider {
    static var previews: some View {
        ContentView()
    }
}
```  
  
すると下記のようにタップエリアを拡大することができます。  
![cantap720.gif](/image/c945cb17-88c1-1ce1-700e-b4abf251de3a.gif)  
  
  
  
画像をタップできるようにしたいけれど  
画像だけだとタップできる領域が狭い場合などに  
コードを１行加えて領域を広げることができるのは  
大変便利ですね😄  
