WWDC中ですが過去のiOSバージョンの見直しを継続しています。  
  
  
## なぜiOS10?  
  
iPhoneの工場出荷時の初期バージョンが最新の２つ前に設定されている(はず?)なので、方針として２つ前のバージョンからサポートをするというようにしています。  
(社内事情で端末のバージョンを上げることができない会社もあるようですので)  
  
そのため今回のiOS12へのバージョンアップによってiOS10で使用できる機能が使えるようになり、改めて調べたことを定期的に記録しておくことにしました。  
  
今回はDateIntervalです。  
  
  
すごい新機能というわけではありませんが、  
地味に日常の開発の中で役に立つことが結構あるのかなと思いました。  
  
  
## DateInterval(NSDateInterval)   
  
※SwiftなのでDateIntervalとして話を進めます。  
  
Foundationフレームワークに追加されたクラスで  
ある日時とある日時の間の期間を簡単に表現できます。  
  
  
下記の３つのプロパティを持ち、  
  
start(開始日時)  
end(終了日時)  
duration(日時間隔)  
  
下記のような形で使用します。  
  
```swift
let today = Date()
let tomorrow = Date(timeIntervalSinceNow: 60*60*24)

let next24Hours = DateInterval(start: today, end: tomorrow)

print(next24Hours) // 2018-06-05 21:48:48 +0000 to 2018-06-06 21:48:48 +0000

```  
  
## DateIntervalFormatter  
  
他の日時クラスと同様にフォーマッタークラスで表示方法を変更できます。  
  
```swift
let formatter = DateIntervalFormatter()
formatter.locale = Locale(identifier: "ja_JP")
formatter.dateStyle = .full
formatter.timeStyle = .full

print(formatter.string(from: next24Hours)) // 2018/06/06 6:51:52～2018/06/07 6:51:52
// ↑と同じ意味になる
formatter.string(from: today, to: tomorrow)

```  
  
## 範囲チェック  
  
containsメソッドを使うことである期間の中にある日時が含まれているかを確認できます。  
  
```swift

// これはtrue
let holiday = DateInterval(start: Date(), duration: 60*60*24*4)
holiday.contains(Date(timeIntervalSinceNow: 60*60*24*3))

// これはfalse
let holiday = DateInterval(start: Date(), duration: 60*60*24*1)
holiday.contains(Date(timeIntervalSinceNow: 60*60*24*3)) // false

// 同じだとfalseになる
let holiday = DateInterval(start: Date(), duration: 60*60*24*3)
holiday.contains(Date(timeIntervalSinceNow: 60*60*24*3))

```  
  
## 比較をする  
  
compareメソッドがありますが、オペレータを使っても比較をすることができます。  
比較方法は、最初に開始日時を比較し、同じ場合は日時間隔を比較します。  
  
  
```swift

// 開始日時がc1の方が前
let c1 = DateInterval(start: Date(), duration: 60*60*24*2)
let c2 = DateInterval(start: Date(timeIntervalSinceNow: 60*60*24*1), duration: 60*60*24*2)

c1 < c2 // true

// 開始日時は同じ、日時間隔はc3の方が短い
let now = Date()
let c3 = DateInterval(start: now, duration: 60*60*24*1)
let c4 = DateInterval(start: now, duration: 60*60*24*2)

c3 < c4 // true


// compareメソッドを使用した場合
// orderedAscendingが出力される
switch c1.compare(c2) {
case .orderedAscending:
    print("orderedAscending")
case .orderedDescending:
    print("orderedDescending")
case .orderedSame:
    print("orderedSame")
}
```  
  
## 重複チェックをする  
  
あるDateIntervalがもう一つのDateIntervalと重なる部分があるかを知るために  
intersectsメソッドを使用することができます。  
  
```swift

var components = Calendar.current.dateComponents([.year, .weekOfYear], from: Date())

components.weekday = 2
components.hour = 10
let firstDayOfWeek = Calendar.current.date(from: components)!

components.weekday = 6
components.hour = 19
let lastDayOfWeek = Calendar.current.date(from: components)!

let workWeek = DateInterval(start: firstDayOfWeek, end: lastDayOfWeek)

formatter.locale = Locale(identifier: "ja_JP")
formatter.string(from: workWeek) // 2018/06/04 10:00:00～2018/06/08 19:00:00

// これは重複しているのでtrue
components.weekday = 4
components.hour = 0
let holidayStart = Calendar.current.date(from: components)!
let paidHolidays = DateInterval(start: holidayStart, duration: 60*60*24*3)
paidHolidays.intersects(workWeek)

// 重複していないのでfalse
components.weekday = 6
components.hour = 20
let otherHolidayStart = Calendar.current.date(from: components)!
let otherHolidays = DateInterval(start: otherHolidayStart, duration: 60*60*24*2)
otherHolidays.intersects(workWeek)  // false
```  
  
実際の重複している部分を取り出す場合は、  
intersectionメソッドを使用します。  
  
```swift

if let holidays = paidHolidays.intersection(with: workWeek) {
    formatter.string(from: holidays) // 2018/06/06 0:00:00～2018/06/08 19:00:00
}
```  
  
## まとめ  
  
DateIntervalは地味な追加かもしれませんが、  
日常の中で役に立ちそうな点は多いのかもしれません。  
  
他にも実は追加されていたクラスがあるかもしれませんので、  
探してみます。  
  
  
何か間違いなどございましたらご指摘頂けますと幸いです:bow_tone1:  
  
  
## 関連記事:  
  
[【Swift】この時期だから見直すiOS10の新機能 UIGraphicsImageRendererとUIViewPropertyAnimator]  
(https://qiita.com/stzn/items/26d8c238e7055c523f80)  
  
[【Swift】この時期だから見直すiOS10の新機能 UserNotificationsとNotification Content ExtensionとNotification Service Extension]  
(https://qiita.com/stzn/items/ab39faae1ef3a758fc08)  
  
[【Swift】この時期だから見直すiOS10の新機能 AVCapturePhotoOutput AVCaptureSettings など]  
(https://qiita.com/stzn/items/d7738f998e4be2d37c0f)  
  
[【Swift】この時期だから見直すiOS10の新機能 UITableView UICollectionViewの改善とPre-Fetching API]  
(https://qiita.com/stzn/items/66ae7e772d315dadf7d4)  
  
[【Swift】この時期だから見直すiOS10の新機能 UIPreviewInteraction(3D Touch)]  
(https://github.com/stzn/qiita/blob/master/【Swift】この時期だから見直すiOS10の新機能 UIPreviewInteraction(3D Touch).md)  
  
