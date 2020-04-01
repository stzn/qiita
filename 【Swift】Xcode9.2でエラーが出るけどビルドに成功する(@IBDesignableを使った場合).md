何回か出会いましたので自身の備忘録の意味も含めて投稿させていただきます。  
  
下記の画像のようにエラーが出ているけれどもビルドには成功するという現象が起きます。  
  
![error.png](/image/99e012c6-ef0c-b9b3-c93d-6c0cdda6afc7.png)  
  
  
原因は@IBDesignableを付与しているクラスでエラーになる可能性がある場合に  
発生します。  
  
以下のファイルでエラーの原因箇所の特定ができます。  
  
```
Library/Logs/DiagnosticReports/IBDesignablesAgentCocoaTouch_XXXXXX.crash
```  
  
```IBDesignablesAgentCocoaTouch_XXXXXX.crash
Thread 0 Crashed:
0   sztk.sample.NSCocoaErrorSample	0x0000000219779d3b UITextViewPlus.changeFontSize() + 763 (UITextViewPlus.swift:26)
1   sztk.sample.NSCocoaErrorSample	0x000000021977980a UITextViewPlus.init(frame:textContainer:) + 234 (UITextViewPlus.swift:16)
2   sztk.sample.NSCocoaErrorSample	0x00000002197798ba @objc UITextViewPlus.init(frame:textContainer:) + 106
3   com.apple.UIKit               	0x0000000105a3e7d0 -[UITextView initWithFrame:] + 58
...
```  
  
<br>  
  
原因のファイルは以下です。  
  
  
```UITextViewPlus.swift
@IBDesignable
class UITextViewPlus: UITextView {
    override init(frame: CGRect, textContainer: NSTextContainer?) {
        super.init(frame: frame, textContainer: textContainer)
        changeFontSize()
    }
    
    required init(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)!
        changeFontSize()
    }
    
    private func changeFontSize() {        
        let ratio = UIScreen.main.bounds.size.width / 320
        
        // *****nilになる可能性がある**********
        self.font = UIFont(
            name: (self.font?.fontName)!,
            size: (self.font?.pointSize)! * ratio
        ) 
    }
}
```  
  
<br>  
  
以下のようにnilになる可能性を排除するとエラーは解消されます。  
  
```
private func changeFontSize() {

    guard let font = self.font else { return }

    let ratio = UIScreen.main.bounds.size.width / 320
    self.font = UIFont(
        name: font.fontName,
        size: font.pointSize * ratio) ?? font
}
```  
  
どうせならビルドエラーになって欲しいですね。。。  
