  
ちょっとした実装をしているときに「あれ、これ何でうまくいかないんだろう？」と思ったことがあり、メモします。  
  
UICollectionViewを話題にしていますがUITableViewも同じです。  
  
  
# CellをSwapしたい  
  
やりたいことはシンプルで、あるセルとあるセルの位置を交換したく、  
下記のようにDataSourceのデータをSwapしてreloadDataを行います。  
  
ちなみに、UICollectionViewの```moveItem(at indexPath: IndexPath, to newIndexPath: IndexPath)```を使用するとこの現象は起きませんが、今回は２つのセルを入れ替えたいという目的があり、  
上下のセルを入れ替えると他のセルも動いてしまうため、使いませんでした。  
  
![IMG_0031.TRIM.MOV.gif](/image/9267d207-a3e3-7b98-ab71-3baf797463d1.gif)  
  
  
  
```swift

@objc func handlePanGesture(_ gesture: UIPanGestureRecognizer) {


    // movingCellは移動中のセルの状態を保持しているクラス
    switch gesture.state {
        
    case .began:
        movingCell = MovingCell(cell: cell.0, originalLocation: cell.0?.center, indexPath: cell.1)
        
    case .changed:
        movingCell?.cell.layer.zPosition = 100
        movingCell?.cell.center = gesture.location(in: gesture.view!)
    case .ended:
        swapMovingCellWith(cell: cell.0, at: cell.1)
    default:
        movingCell?.reset()
        movingCell = nil
    }
}

private func swapMovingCellWith(cell: UICollectionViewCell?, at indexPath: IndexPath?) {
    guard let cell = cell, let indexPath = indexPath, let moving = movingCell else {
        movingCell?.reset()
        return
    }
    
    viewModel.swap(from: moving.indexPath.item, to: indexPath.item)
    animate(moving: moving, to: cell)
}

private func animate(moving movingCell: MovingCell, to cell: UICollectionViewCell) {
    panGesture.isEnabled = false
    
    UIView.animate(withDuration: 0.4, delay: 0, usingSpringWithDamping: 0.1, initialSpringVelocity: 0.7, options: .allowUserInteraction, animations: {
        
        movingCell.cell.center = cell.center
        cell.center = movingCell.cell.center
        
    }, completion: { _ in

        // ここが問題
        self.collectionView.reloadData()
        self.panGesture.isEnabled = true
        self.movingCell = nil
    })
}
```  
  
一見問題なく動きますが、  
  
![IMG_0025.TRIM.MOV.gif](/image/1e2902e4-95a4-7ed2-41a7-470251a5897f.gif)  
  
  
一番左上のセルを移動させようとすると移動位置がおかしくなります。  
  
![IMG_0026.TRIM.MOV.gif](/image/368747e8-a83f-12b3-babd-a97a3499d14d.gif)  
  
画面上では消えたように見えますが、  
実際には同じ位置の下にいます。  
  
この現象が起きるのは、一番左上のセルから他のセルに移動された場合のみで、  
逆に他のセルから左上のセルに移動させる場合には問題はありませんでした。  
  
色々原因を探ってみましたが、  
  
Storyboardでcellを作成しているから? →コードから生成しても変わらず。  
  
MainThreadで処理していない？→あえてDispatchQueue.main.async内でやってみたが変わらず。  
  
  
# 回避策  
  
結局この条件の時だけ、最後に移動させるようにしました。  
reloadDataのあとに確実に処理を行うために、animationのcompletionで処理を行なっています。  
  
```
UIView.animate(withDuration: 0, animations: {
    self.collectionView.reloadData()
}, completion: { _ in
    if movingCell.indexPath.item == 0 {
        cell.frame.origin.x = 0
    }
    self.panGesture.isEnabled = true
    self.movingCell = nil
    completion()
})
```  
  
![IMG_0027.TRIM.MOV.gif](/image/a40d0ce8-9c44-e7f9-14db-4dd9d113ba1e.gif)  
  
  
# まとめ  
  
なぜこの現象が起きるのかがわかっていないのですごい気持ち悪いです:cold_sweat:  
IndexPath(item: 0, section: 0)であることが何か関係しているのか。:thinking:  
  
もうちょっと調べてみます。  
  
何か他の解決法をご存知の方がいらっしゃいましたら教えてください:bow_tone1:  
