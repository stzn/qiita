この前Responder Chainについて調べたついでに学んだことをメモします。  
  
Appleの[ヒューマンインターフェイスガイド](https://developer.apple.com/design/tips/jp/)に記載もあるようにコントロール要素は最低44X44ポイント以上の大きさで作成することが推奨されていますが、デザインの関係でUIButtonなどがかなり小さくなる場合があります。  
その際にUIButtonの代わりにUIGestureRecognizerを設定したUIViewを使うなどで反応を広げることも可能ですが、  
タッチ時の当たり判定でtrueになる範囲を広げることで  
擬似的にボタンを大きくすることで対処できます。  
  
```ExpandedButton.swift


class ExpandedButton: UIButton {

    private let minimumHitArea = CGSize(width: 44, height: 44)

    override open func point(inside point: CGPoint, with _: UIEvent?) -> Bool {
        if self.isHidden || !self.isUserInteractionEnabled || self.alpha < 0.01 { return  false }
        
        let buttonSize = self.bounds.size
        let widthToAdd = max(minimumHitArea.width - buttonSize.width, 0)
        let heightToAdd = max(minimumHitArea.height - buttonSize.height, 0)
        let area = self.bounds.insetBy(dx: -widthToAdd / 2, dy: -heightToAdd / 2)
        return area.contains(point)
    }
}
```  
  
タッチ判定をするメソッドでもう一つ[hitTest(_:with:)](https://developer.apple.com/documentation/uikit/uiview/1622469-hittest)があり、  
こちらでも同じことが可能でしたが、  
Viewを返してくれるのでタッチ判定を無効にして  
subviewに処理を任せたいときなどに活用している方が多いように感じました。  
※もしこっちの方が良いなどございましたら教えてください:bow_tone1:  
  
```NoTouchView.swift


class NoTouchView: UIView {
    
    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
    }
    
    override init(frame: CGRect) {
        super.init(frame:frame)
        
    }

    override func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {

        // 自分自身のタッチは無効にする
        let view = super.hitTest(point, with: event)        
        if view == self {
            return nil
        }
        return view
    }
}
```  
