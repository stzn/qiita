以前投稿した内容で  
Combine.frameworkの`Future`はインスタンスを生成した時点で  
中のクロージャを実行させるということについて書きました。  
  
https://github.com/stzn/qiita/blob/master/【Swift】Combineで一つのPublisherの出力結果を共有するメソッドやクラスの違い(share, multicast, Future).md  
https://developer.apple.com/documentation/combine/future  
  
この場合  
Subscribeをしていないのに処理が実行されてしまい  
無駄にリソースを消費してしまう可能性や  
副作用を起こして思わぬ動作をしてしまう可能性もあります。  
  
  
そういう場合`Deferred`を活用する方法があります。  
  
https://developer.apple.com/documentation/combine/deferred  
  
`Deferred`はinitで`createPublisher`というクロージャを受け取り  
中でPublisherを生成します。  
このクロージャは**Subscribeした時に初めて**実行されます。  
  
# 検証  
  
下記のコードをPlaygroundで実行して確認します。  
  
まずFutureの場合  
  
```swift

let futurePublisher = Future<String, Never> { promise in
    print("Future 実行")
    promise(.success("hello"))
}
```  
  
この時点で  
  
```
Future 実行
```  
  
が出力されます。  
  
一方`Deferred`の場合  
  
```swift

let deferredPublisher = Deferred { () -> Just<String> in
    print("Deferred 実行")
    return Just("hello")
}
```  
  
これだけ書いても何も実行されません。  
  
Subscribeしてみると  
  
```swift

let cancellable = deferredPublisher
    .sink(receiveCompletion: {
        print("receiveCompletion \($0)")
    }, receiveValue: {
        print("receiveValue \($0)")
    })
```  
  
```
Deferred 実行
receiveValue hello
receiveCompletion finished
```  
  
とSubscribeしてから  
クロージャの中が実行されていることがわかります。  
  
これを活用して`Future`を遅延実行させる方法が  
下記の動画でも紹介されています。  
  
※   
下記の例は`Future`の特徴をよく表しているため引用させていただきました。  
  
注意点として下記の例にある２番目の1729という値を受け取ることはありません。  
  
Futureは値を一度受け取るとすぐにCompletionするため  
受け取れるのは**最初の42だけ**です。  
  
動画では値を継続的に受け取りたいため  
この後Custom Publisherの生成を行っています。  
  
https://www.pointfree.co/episodes/ep80-the-combine-framework-and-effects-part-1  
  
```swift

let aFutureInt = Deferred {
  Future<Int, Never> { callback in
    DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
      print("Hello from inside the future!")
      callback(.success(42))
      callback(.success(1729))
    }
  }
}
```  
※ この時点ではまだSubscribeしていないため  
実行はされません。  
  
  
# まとめ  
  
`Deferred`について検証してみました。  
`Deferred`を活用することで不必要な処理やセットアップなどが  
実行されるリスクを減らすことができます。  
  
また`Future`を一緒に活用することで  
非同期処理の遅延実行も実現することができることもわかりました。  
  
しかし  
`Future`は値を一度して受け取らないという特徴を持っているため  
使い方を知らないと  
  
「あれっ？なんか思っていたのと違う」  
  
となってしまう可能性もあるので注意が必要ですね😅  
  
何か間違いなどございましたら教えていただけますとうれしいです🙇🏻‍♂️  
