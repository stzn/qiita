  
[XibでViewを作成した場合](https://qiita.com/sztk1209/items/854afdfe8c9e0eed5999)  
  
チャットアプリなどでよく見かける  
画面の下にずっと出ている入力欄を実装するとき  
```inputAccessaryView```を使うと楽に実装できますが、
iPhoneXだとSafeAreaに被ってしまう現象があります。

<img width="30%" width="375" alt="Simulator Screen Shot - iPhone X - 2018-02-07 at 08.48.17.png" src="/image/52351653-e422-0f65-d9e1-1194832cf7ac.png">


以下のような実装で解決できました。

```Swift  
  
import UIKit  
  
class CustomView: UIView {  
    
    // inputAccessoryViewのサイズが  
    // AutoLayout制約から適切に設定されるように必要  
    override var intrinsicContentSize: CGSize {  
        return CGSize.zero  
    }  
}  
class ViewController: UIViewController {  
    
    override func viewDidLoad() {  
        super.viewDidLoad()  
    }  
    
    override func viewDidDisappear(_ animated: Bool) {  
        super.viewDidDisappear(animated)  
    }  
    
    override var inputAccessoryView: UIView? {  
        return inputContainerView  
    }  
    
    override var canBecomeFirstResponder : Bool {  
        return true  
    }  
    
    lazy var textField: UITextField = {  
        let tf = UITextField()  
        tf.placeholder = "Enter message"  
        tf.backgroundColor = .white  
        tf.borderStyle = .roundedRect  
        tf.translatesAutoresizingMaskIntoConstraints = false  
        tf.delegate = self  
        return tf  
    }()  
    
    lazy var inputContainerView: UIView = {  
        
        let containerView = CustomView()  
        containerView.backgroundColor = .lightGray  
        containerView.translatesAutoresizingMaskIntoConstraints = false  
        
        containerView.addSubview(textField)  
        
        self.textField.leadingAnchor.constraint(equalTo: containerView.leadingAnchor, constant: 8).isActive = true  
        self.textField.trailingAnchor.constraint(equalTo: containerView.trailingAnchor, constant: -8).isActive = true  
  
        // layoutMarginsGuide.bottomAnchorに対して制約を付ける  
        self.textField.bottomAnchor.constraint(equalTo: containerView.layoutMarginsGuide.bottomAnchor, constant: -8).isActive = true  
        
        let separatorLineView = UIView()  
        separatorLineView.backgroundColor = .black  
        separatorLineView.translatesAutoresizingMaskIntoConstraints = false  
        containerView.addSubview(separatorLineView)  
        
        separatorLineView.leadingAnchor.constraint(equalTo: containerView.leadingAnchor).isActive = true  
        separatorLineView.topAnchor.constraint(equalTo: containerView.topAnchor).isActive = true  
        separatorLineView.widthAnchor.constraint(equalTo: containerView.widthAnchor).isActive = true  
        separatorLineView.heightAnchor.constraint(equalToConstant: 1).isActive = true  
  
        self.textField.topAnchor.constraint(equalTo: separatorLineView.bottomAnchor, constant: 8).isActive = true  
  
        return containerView  
    }()  
}  
```

結果

<img width="30%" alt="Simulator Screen Shot - iPhone X - 2018-02-07 at 08.49.08.png" src="/image/5b7cf939-1ad3-0798-7340-f3f51c590538.png">

<br>
参考サイト: https://stackoverflow.com/questions/46282987/iphone-x-how-to-handle-view-controller-inputaccessoryview

## 補足

下記のような実装をCustomViewに加える方法もありましたが、

```Swift  
  
class CustomView: UIView {  
    override func didMoveToWindow() {  
        super.didMoveToWindow()  
        if #available(iOS 11.0, *) {  
            if let window = self.window {  
                self.bottomAnchor.constraintLessThanOrEqualToSystemSpacingBelow(  
                window.safeAreaLayoutGuide.bottomAnchor, multiplier: 1.0).isActive = true  
            }  
        }  
    }  
}  
```

この場合、SafeAreaへ背景色が反映されませんでした。

<img width="30%" alt="9.png" src="/image/20065e2f-456f-101f-28ce-f21f1787918c.png">
