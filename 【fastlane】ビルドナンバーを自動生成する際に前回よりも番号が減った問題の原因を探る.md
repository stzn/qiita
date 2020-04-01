# 経緯  
  
fastlaneでビルドナンバーを自動で設定する際に、  
なぜか前のバージョンよりも数が減ってしまう現象に遭遇しました。  
  
根本的にfastlaneやgitの知識がないことが原因ではありますが  
はじめは何が起きているのかがわからず  
原因を調べていく中で知らなかったことがたくさん知ることができたので  
記録として残しておきます。  
  
# ビルドナンバーの設定方法  
  
number_of_commitsというメソッドを使っており、  
↓の記載にあるような設定をしていました。  
https://docs.fastlane.tools/actions/number_of_commits/  
  
```ruby

build_number = number_of_commits(all: true)
increment_build_number(build_number: number_of_commits)
```  
  
ドキュメントにも書いてある通り  
**全コミット数をカウントする**  
ということなので  
開発を進めてコミット数が増えれば  
自動でビルドナンバーは増えていくものだと思っていました。  
  
しかし  
あるタイミングで自動生成をしたときに  
**なぜかビルドナンバーが減っていました。**  
  
# 原因を探す  
  
最初は何を調べればよいかもわからない状況ではありましたが  
fastlaneのドキュメントからソースコードを見てそこからヒントが得られました。  
  
```rb

def self.run(params)
  if is_git?
    if params[:all]
      command = 'git rev-list --all --count'
    else
      command = 'git rev-list HEAD --count'
    end
  else
    UI.user_error!("Not in a git repository.")
  end
  return Actions.sh(command).strip.to_i
end

```  
  
今回はパラメータにallにtrueを設定していたため  
  
```shell

git rev-list --all --count
```  
  
がというgitのコマンドを叩いていることがわかりました。  
  
# git rev-list  
  
これが何をしているのかを調べたところ  
  
```
List commits that are reachable 
by following the parent links from the given commit(s),
```  
  
つまり、ある特定のコミットの親のリンクを辿って発見できたコミットのリストを表示するというものでした。  
  
# --all  
  
--allを付けることで  
  
```
Pretend as if all the refs in refs/, along with HEAD, 
are listed on the command line as <commit>.
```  
  
refs/の下にあるreferenceを全て対象とすることになります。  
  
※そもそもrefsって何だということに関しては↓を参照  
https://git-scm.com/book/ja/v1/Git%E3%81%AE%E5%86%85%E5%81%B4-Git%E3%81%AE%E5%8F%82%E7%85%A7  
  
要はブランチとタグの数だと思っています。(間違っていたら教えてください:sweat_smile:)  
  
# --count  
  
最後にcountが付いているのでこのリストの合計数を数えています。  
  
# 原因がわかる  
  
ちょっと前にブランチの運用方法が変更され、  
過去に作業していたブランチの整理が行われていました。  
  
これによってブランチの数が減っていたためcount数が減っているようです。  
  
また  
開発はブランチを切ってmasterへマージをするような運用で  
定期的に不要なブランチの削除も行われるため  
数が必ずしも増え続けていくという訳ではないこともわかりました。  
  
# 対策案  
  
代替案を見つけなければと思っていた際に  
↓のドキュメントから辿れるExampleの中で時間をビルドナンバーに設定しているものがありました。  
https://github.com/fastlane/examples/blob/master/Artsy/eidolon/Fastfile  
  
上記の例のポイントとしては、使用したビルドナンバーをgitにコミットしてプッシュしている点です。  
  
これはbitriseなどのCIで実行する場合に必要な処理で  
下記のファイルのビルドナンバーが変更がされたかどうか(逆にこれら以外は変更されていないこと)を  
fastlaneの方でチェックしているようです。  
  
```
All .plist files
The .xcodeproj/project.pbxproj file
```  
  
https://github.com/fastlane/fastlane/blob/ba72a484f81c09eac616ac821b0186667835843f/docs/Actions.md#using-git  
  
  
まだやってみたという段階ですが、  
ビルドナンバーの更新やgitへのプッシュは無事にできているようです。  
  
  
# まとめ  
  
知識がないため  
最初は何を調べていいのかもわからない状況でしたが  
調べていく中で原因が見えてきてスッキリしました。  
  
こういった開発環境構築の知識は  
言語やフロントサイド、サーバサイドなどの領域を超えて活かせますので  
これからもコツコツと知識を増やしていけたらいいなと思っています。  
  
  
参考資料  
https://stackoverflow.com/questions/35801385/can-git-rev-list-all-count-ever-go-down  
