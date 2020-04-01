  
## 経緯  
  
[AKIBA.swift](https://classmethod.connpass.com/event/78947/?utm_campaign=event_waitlist_promotion_to_participant&utm_source=notifications&utm_medium=email&utm_content=detail_btn)に参加させて頂いた際、  
GridViewのようなレイアウトをどうやって組むのかという話が出て、  
  
・UITableView  
・UICollectionView  
・UIScrollView+UIView  
  
という選択肢がありました。  
  
全体の意見としてはUIScrollView+UIViewを使うが一番多かったのですが、やってみるとどうやるのかなと思って試してみました。  
  
  
## 結果  
  
![scrollview.mov.gif](/image/9a6c4da8-c0ce-6112-b181-bc12f0d7c52f.gif)  
  
  
コードは一切書いていません。  
UIStackViewを組み合わせてUIScrollViewの中に入れました。  
※iPad用にSize ClassでUIStackViewの大きさを変更しています。  
  
![スクリーンショット 2018-02-22 8.11.25（2）.png](/image/e81cc81a-edb8-1545-8a75-56a5bddd5579.png)  
  
  
  
UIScrollViewがScrollしないというありがちな現象に悩まされたのですが、下記の回答を参考にしたら解消されました。  
https://stackoverflow.com/questions/31668970/is-it-possible-for-uistackview-to-scroll  
  
  
実際のデザインはもっと複雑になるとは思いますが  
こういうデザインが来た際の対応法の一つとして  
勉強になりました。  
  
  
何か間違っているなどご指摘があれば教えて頂けますと幸いです:bow_tone1:  
