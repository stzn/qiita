[コードでViewを作成した場合](https://qiita.com/sztk1209/items/0dd53237fe85473b925f)  
  
チャットアプリなどでよく見かける  
画面の下にずっと出ている入力欄を実装するとき  
```inputAccessaryView```を使うと楽に実装できますが、
iPhoneXだとこの入力欄がホームボタンのエリアにかぶります。

<img width="30%" alt="4.png" src="/image/0388015c-4d31-f4d8-64b1-8885b46408ad.png">

これを解決する方法を探していました。
コードでViewを作成した場合はうまくいったのですが、
XibでCustomViewを作成した場合に色々模索した結果、
１つ方法を見つけたので記録として残しておきます。

まず下記のサイトを参考にしました。
https://medium.com/code-with-rohit/inputaccessoryview-and-iphonex-7b5547fe98da

##### 1. UseSafeAreaにチェックを付ける
![1.png](/image/24f6dda1-be40-87c6-a62c-5ebcf03bd442.png)

##### 2. ViewのSafeAreaLayoutGuideのチェックを外す
![2.png](/image/a1b60ae3-26cf-c2b0-018f-744103e33f6c.png)

##### 3. ContainerViewのSafeAreaLayoutGuideにチェックを付ける
![5.png](/image/bc176b00-d267-c596-d9e1-78afda54577f.png)

#### 4. TextViewのbottom制約をSafeLayoutに変更する
![4.png](/image/3f7aa220-dd65-53b8-43f2-0e1d9d0272d5.png)

結果がこうなりました。
<img width="30%" alt="1.1.png" src="/image/792024b1-b34e-9245-732a-efb62de78c58.png">


色々試行錯誤してみましたがうまく行っていません。

最終的にCustomViewのコードを以下のようにすると、

```InputAccessaryView.swift  
  
import UIKit  
  
class InputAccessaryView: UIView {  
  
    @IBOutlet weak var textView: UITextView!  
  
    //!!!!!!!!!!BottomAnchorを調整!!!!!!!!!!!!!  
    override func didMoveToWindow() {  
        super.didMoveToWindow()  
        if #available(iOS 11.0, *) {  
            if let window = self.window {  
                self.bottomAnchor.constraintLessThanOrEqualToSystemSpacingBelow(  
                    window.safeAreaLayoutGuide.bottomAnchor, multiplier: 1.0).isActive = true  
                
                self.backgroundColor = .lightGray  
            }  
        }  
    }  
    
    override init(frame: CGRect) {  
        super.init(frame: frame)  
        setup()  
    }  
    
    required init?(coder aDecoder: NSCoder) {  
        super.init(coder: aDecoder)  
        setup()  
    }  
    
    func setup() {  
        let view = Bundle.main.loadNibNamed("InputAccessaryView", owner: self, options: nil)?.first as! UIView  
        view.frame = self.bounds  
        self.addSubview(view)  
    }  
}  
```

結果こうなりました。

<img width="30%" alt="1.3.png" src="/image/bb7b7dd0-4894-0875-11b5-f140466a4bce.png">

参考サイト
https://stackoverflow.com/questions/46282987/iphone-x-how-to-handle-view-controller-inputaccessoryview
http://ahbou.org/post/165762292157/iphone-x-inputaccessoryview-fix

かぶる現象は解決しましたが、

・[LayoutConstraints] Unable to simultaneously satisfy constraints.がコンソールに出力される
・ホームボタンエリアの背景色が変わらない。(ViewControllerの背景色を変えることで変更は可能)


という問題がまだ残っています。

何か解決方法をご存知の方いらっしゃいましたら
教えていただけましたらうれしいです:bow_tone1:
