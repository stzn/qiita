SwiftUIで`List`を使用していて  
`UITableViewCell`のように各行をダイナミックな高さで表示をしようとする際に  
`layoutPriority`を使うと簡単に実現することができます。  
https://developer.apple.com/documentation/swiftui/view/3278584-layoutpriority  
  
実際に行っているのはQiitaAPIから取得したデータをリストに表示しています。  
左側にユーザのアイコン  
真ん中に記事のタイトルとユーザ名  
右側に静的ないいねマークの画像といいねの数  
を表示します。  
  
まずは`layoutPriority`がない状態です。  
  
```swift

struct QiitaView: View {
    let item: QiitaItemViewModel
    var body: some View {
        HStack {
            ImageLoadingView(loader: ImagePath(path: item.profileImageURL))
                .frame(width: 44, height: 44)
                .clipShape(Circle())
                .overlay(Circle().stroke(Color.white, lineWidth: 4))
                .shadow(radius: 10)

            VStack(alignment: .leading) {
                Text(item.title)
                    .lineLimit(nil)
                    .padding([.bottom], 12)

                Text(item.userName)
                    .lineLimit(nil)
            }
            .padding([.leading, .trailing], 12)

            Spacer()

            VStack(alignment: .center) {
                Image(systemName: "faceid")
                    .resizable()
                    .frame(width: 44, height: 44)

                Text("\(item.likesCount)")
            }
            .frame(width: 60)
        }
    }
}
```  
  
これを表示すると下記のようになります。  
<img width="456" alt="スクリーンショット 2019-09-23 14.16.15.png" src="/image/33431c83-cd3a-ab3b-6a95-1cced9631559.png">  
  
真ん中のラベルの横幅は想定よりも狭くなってしまっています。  
  
これを`layoutPriority`を用いて改善していきます。  
  
まずは真ん中の`VStack`に`layoutPriority`を設定します。  
`layoutPriority`はデフォルトが0で  
1を設定することで優先度を他よりも高くすることができます。  
  
```swift

struct QiitaView: View {
    let item: QiitaItemViewModel
    var body: some View {
        HStack {
            ImageLoadingView(loader: ImagePath(path: item.profileImageURL))
                .frame(width: 44, height: 44)
                .clipShape(Circle())
                .overlay(Circle().stroke(Color.white, lineWidth: 4))
                .shadow(radius: 10)

            VStack(alignment: .leading) {
                Text(item.title)
                    .lineLimit(nil)
                    .padding([.bottom], 12)

                Text(item.userName)
                    .lineLimit(nil)
            }
            .padding([.leading, .trailing], 12)
            .layoutPriority(1) // ← ここに追加

            Spacer()

            VStack(alignment: .center) {
                Image(systemName: "faceid")
                    .resizable()
                    .frame(width: 44, height: 44)

                Text("\(item.likesCount)")
            }
            .frame(width: 60)
        }
    }
}
```  
  
すると下記のイメージのような状態になります。  
  
<img width="457" alt="スクリーンショット 2019-09-23 14.17.07.png" src="/image/817dad83-29d2-0cc1-638b-14f3eb31771d.png">  
  
横幅は改善されました。  
  
しかし、まだタイトルが切れてしまっている状態です。  
  
そこで`Text`にも`layoutPriority`を設定します。  
  
```swift

struct QiitaView: View {
    let item: QiitaItemViewModel
    var body: some View {
        HStack {
            ImageLoadingView(loader: ImagePath(path: item.profileImageURL))
                .frame(width: 44, height: 44)
                .clipShape(Circle())
                .overlay(Circle().stroke(Color.white, lineWidth: 4))
                .shadow(radius: 10)

            VStack(alignment: .leading) {
                Text(item.title)
                    .lineLimit(nil)
                    .padding([.bottom], 12)
                    .layoutPriority(1) // ← ここに追加

                Text(item.userName)
                    .lineLimit(nil)
            }
            .padding([.leading, .trailing], 12)
            .layoutPriority(1)

            Spacer()

            VStack(alignment: .center) {
                Image(systemName: "faceid")
                    .resizable()
                    .frame(width: 44, height: 44)

                Text("\(item.likesCount)")
            }
            .frame(width: 60)
        }
    }
}
```  
  
すると下記のような形になります。  
  
<img width="458" alt="スクリーンショット 2019-09-23 14.18.30.png" src="/image/62ec1054-db80-fe0a-4fc7-92c7c8a10e93.png">  
  
もし何か参考になりましたら幸いです。  
  
逆に間違いや他に良い方法があれば  
教えていただけると嬉しいです🙇🏻‍♂️  
