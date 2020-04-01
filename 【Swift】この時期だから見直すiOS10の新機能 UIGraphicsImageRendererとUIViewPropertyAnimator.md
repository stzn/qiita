  
WWDCを前に過去のiOSバージョンの見直しをしようと思います。  
  
## なぜiOS10?  
  
iPhoneの工場出荷時の初期バージョンが最新の２つ前に設定されている(はず?)なので、方針として２つ前のバージョンからサポートをするというようにしています。  
(社内事情で端末のバージョンを上げることができない会社もあるようですので)  
  
そのため今回のiOS12へのバージョンアップによってiOS10で使用できる機能が使えるようになり、  
改めて調べたことを定期的に記録しておくことにしました。  
  
今回はUIGraphicsImageRendererとUIViewPropertyAnimatorです。  
  
## UIGraphicsImageRenderer  
  
画像を描画するためのクラスです。  
  
iOS10以前では以下のように書きます。  
  
```a.swift
// サイズに合わせて四角形を書く
private func drawRectangleLegacy(size: CGSize) -> UIImage? {
    
    // コンテキスト(画像を描画する下地のようなもの)を生成    
    UIGraphicsBeginImageContext(size)
    
    // 後始末(しないとメモリーリークを起こす)    
    defer { UIGraphicsEndImageContext() }
    
    // コンテキストを取得    
    guard let context = UIGraphicsGetCurrentContext() else { return nil }
    
    // 画像を描画
    context.setFillColor(UIColor.red.cgColor)
    context.fill(CGRect(origin: CGPoint.zero, size: size))
    
    // 描いた画像を取得
    guard let image = UIGraphicsGetImageFromCurrentImageContext() else { return nil }
    
    return image
}
```  
  
この場合、  
・contextを管理する必要がある  
・描いた画像を取り出さなければいけない  
・optionalチェックをする必要がある  
・後始末を忘れた場合に事故を起こす可能性がある  
・処理でまとまっておらずわかりづらい(個人的には)  
  
  
これをUIGraphicsImageRendererを使うと  
  
```a.swift
private func drawRectangle(size: CGSize) -> UIImage {

    // コンテキストを生成+取得(後始末は自動)  
    // imageメソッドでUIImageを取得  
    return UIGraphicsImageRenderer(size: size).image { context in

        // 画像を描画  
        context.cgContext.setFillColor(UIColor.red.cgColor)
        context.fill(CGRect(origin: .zero, size: size))
    }
}
```  
  
と書けるため上記の懸念点を解決することができています。  
  
  
## UIViewPropertyAnimator  
  
名前の通りですが、アニメーションを指定するクラスです。  
これが導入されたことによってアニメーションを動的に変化させることができるようになりました。  
  
例えば、下記のような場合ですと、  
  
```a.swift
UIView.animate(withDuration: 10) { [weak self] in
    self?.imageView.center = point
}
```  
  
この場合、一回アニメーションを実行するとアニメーションが終了するまで  
次のアニメーションは起きません。  
  
![UIVIewAnimate.mov.gif](/image/b4325f32-1580-db18-2f6d-509a6196d011.gif)  
  
  
下記のようにアニメーションを一度キャンセルすると挙動がおかしくなります。  
  
```a.swift
imageView.layer.removeAllAnimations()
UIView.animate(withDuration: 5) { [weak self] in
    self?.imageView.center = point
}
```  
  
![UIViewCancel.mov.gif](/image/ba3bad39-a183-faa3-d34f-f0c26ceb146b.gif)  
  
  
  
一方で、  
  
```a.swift
let animator = UIViewPropertyAnimator(duration: 5, curve: .linear)
animator.addAnimations { [weak self] in
    self?.imageView.center = point
}
animator.startAnimation()
```  
  
とするとクリックした時点で次のアニメーションに動的に変更することができます。  
  
![UIViewProperty.mov.gif](/image/b14668a4-32f7-6225-23cd-6e8e3d159710.gif)  
  
  
  
確認のために使用したコード全体です。  
  
```ViewController.swift
import UIKit

class ViewController: UIViewController {
    
    let imageView: UIImageView = {
        let i = UIImageView()
        i.translatesAutoresizingMaskIntoConstraints = false
        return i
    }()
    
    
    private var targetPoint: CGPoint!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let screen = UIScreen.main.bounds
        
        imageView.image = drawRectangle(
            size: CGSize(width: screen.width * 0.5, height: screen.width * 0.5))
        
        view.addSubview(imageView)
        imageView.centerXAnchor.constraint(equalTo: view.centerXAnchor).isActive = true
        imageView.centerYAnchor.constraint(equalTo: view.centerYAnchor).isActive = true
    }
    
    private func drawRectangleLegacy(size: CGSize) -> UIImage? {
        
        UIGraphicsBeginImageContext(size)
        
        defer { UIGraphicsEndImageContext() }
        
        guard let context = UIGraphicsGetCurrentContext() else { return nil }
        
        context.setFillColor(UIColor.red.cgColor)
        context.fill(CGRect(origin: CGPoint.zero, size: size))
        
        guard let image = UIGraphicsGetImageFromCurrentImageContext() else { return nil }
        
        return image
    }
    
    private func drawRectangle(size: CGSize) -> UIImage {
        
        return UIGraphicsImageRenderer(size: size).image { context in
            context.cgContext.setFillColor(UIColor.red.cgColor)
            context.fill(CGRect(origin: .zero, size: size))
        }
    }
    
    private func handleTouches(_ touches: Set<UITouch>) {
        guard let touch = touches.first else {
            return
        }
        let p = touch.location(in: view)
        if p == targetPoint {
            return
        }
        targetPoint = p
        moveto(point: p)
    }
    
    private func moveto(point: CGPoint) {
        
//        imageView.layer.removeAllAnimations()
//        UIView.animate(withDuration: 5) { [weak self] in
//            self?.imageView.center = point
//        }

        let animator = UIViewPropertyAnimator(duration: 5, curve: .linear)
        animator.addAnimations { [weak self] in
            self?.imageView.center = point
        }
        animator.startAnimation()
    }
    
    override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
        handleTouches(touches)
    }
    
    override func touchesMoved(_ touches: Set<UITouch>, with event: UIEvent?) {
        handleTouches(touches)
    }
}

```  
  
一度WWDCの後に確認しているはずですが、やはり使わないと忘れていることはたくさんあるなと痛感しました。  
  
  
何か間違った認識などございましたらご指摘いただけましたら幸いです:bow_tone1:  
