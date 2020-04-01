少し前に「これは何だろう？」と思ったことについて調べてみました。  
  
# SwiftのResultとは？  
  
`Success`と`Failure`の2つのケースを持った`enum`です。  
  
多くの方が自前で今まで実装してきましたが  
Swif５.0で標準ライブラリに導入されました。  
  
https://github.com/apple/swift/blob/master/stdlib/public/core/Result.swift  
  
# 鉄道指向プログラミング(Railway Oriented Programming)とは？  
  
2014年にScott Wlaschinさんが提唱された  
関数型プログラミングを行っていくなかで  
**エラーハンドリングをどう扱っていくか**に主に焦点を当てたプログラミング手法です。  
  
https://fsharpforfunandprofit.com/rop/  
  
>Many examples in functional programming assume   
that you are always on the “happy path”.   
But to create a robust real world application   
you must deal with validation, logging, network   
and service errors, and other annoyances.   
  
>So, how do you handle all this in a clean functional way?   
  
>This talk will provide a brief introduction to this topic,   
using a fun and easy-to-understand railway analogy.  
  
-  
  
>多くの関数型プログラミングの例は  
いつも「ハッピーパス(いわゆる正常系のこと)」を通っていることを想定しているが  
現実にがバリデーションやログ、ネットワーク通信やサービスのエラー  
その他の腹立たしいことを扱わなければならない。  
  
>で、それをこれらをどうやってクリーンに関数型の手法で対処できるだろうか？  
  
>今回のトークでは、楽しく、わかりやすい鉄道との共通点を使ってこの問題について  
簡単な導入部分を紹介する。  
  
  
とあり  
  
抽象的な概念というよりは  
より具体的な問題に焦点を当てており  
すぐに実践に応用できるようになっています。  
  
  
# Resultと鉄道指向プログラミング  
  
鉄道指向プログラミングでは  
`Result`を用いたエラーハンドリングを行います。  
  
`Result`を用いることでそのままの型を用いる以上の  
多くのメリットを得ることができます。  
  
今回は  
鉄道指向プログラミングはどういうものなのかが気になったのと  
`Result`がどのように機能するのかを理解するために  
記録として残してみました。  
  
※  
記事の中で出てくる図はScottさんの講演スライドから拝借しています。  
ブログの中で「自由にして良い」という記載がございましたので  
活用させていただきました。  
  
> The powerpoint slides are also available from Github. Feel free to borrow from them!  
  
  
  
# 今回の例  
  
以下の処理を行います。  
  
1. リクエストを受け取る  
2. バリデーションチェックをする  
3. リクエストをDBに保存(更新)する  
4. 認証済みのメールに送信する  
5. ユーザに結果を返す  
  
# 命令型プログラミングで書いた例  
  
下記の例を見てみます  
  
<details><summary>命令型プログラミングの実装</summary><div>  
  
```swift

struct DB {
    func updateDb(from request: Request) throws -> Bool{
        return true
    }
}

struct SmtpServer {
    func sendEmail(to request: Request) throws -> Bool {
        return true
    }
}

func validateRequest(_ request: Request) -> Bool {
    return true
}

func executeUseCase(request: Request, db: DB, stmpServer: SmtpServer) -> String {
    if !validateRequest(request) {
        return "validation error"
    }
    do {
        let result = try db.updateDb(from: request)
        if !result {
            return "Record not found"
        }

        if !stmpServer.sendEmail(to: request) {
            return "Fail to send mail"
        }
    } catch {
        return "Fail to update"
    }
    return "OK"
}
```  
  
</div></details>  
  
これはいわゆる命令型と呼ばれるような形で書かれています。  
  
特に問題はないのですが  
こうすると  
`if`で判定をしたり  
`do catch`文が途中で入ってくるため  
本来のやりたいことが見えづらくなってしまいます。  
  
では  
このようなエラーハンドリングを  
どうやって関数型プログラミングを使って  
綺麗な形で処理できるでしょうか？  
  
# 結果のパターンを考えてみる  
  
上記の例の処理の流れを考えてみます。  
  
![スクリーンショット 2019-06-01 5.09.47.png](/image/d46ed82e-a977-5161-57f8-a99149c29e7f.png)  
  
`Request`を受け取り  
処理が成功した場合は次の処理へ  
エラーの場合はエラーになった時点で  
レスポンスを返しています。  
  
  
次に関数型プログラミングの形で考えてみます。  
  
![スクリーンショット 2019-06-01 5.09.20.png](/image/c6b4d714-9ea6-3db8-6b79-335f5a1e523b.png)  
  
関数型では処理を上から下へ向かう  
データの流れとして捉えます。  
  
ここで上記の図のように処理の結果は  
4つのパターンが考えられます。  
  
このようなレスポンスをどうやって表現することができるでしょうか？  
  
```swift

enum Result {
    case Success
    case ValidationError
    case UpdateError
    case SendMailError
}
```  
  
パターンということで`enum`として捉えました。  
  
これですべてのケースを網羅できていますが  
他の処理でも同じように使えるようにしたいですね。  
  
それがSwiftの`Result`です。  
  
```swift

public enum Result<Success, Failure: Error> {
    case success(Success)
    case failure(Failure)
}
```  
  
※ F#にもResultはありますがSwiftのFailureはErrorプロトコルに適合する必要があります。  
  
`Result`を使うと処理の流れは下記のようになります。  
  
![スクリーンショット 2019-06-01 5.21.22.png](/image/680b4874-ac37-a010-2afc-087e773ae68c.png)  
  
こうすることで各関数が同じレスポンスを返すようになります。  
下記に`Result`を返す関数を提示してみます。  
  
<details><summary>Resultを返す関数</summary><div>  
  
```swift

struct ValidationError: LocalizedError {
    let field: String
    let value: Any
    let reason: String
    
    var localizedDescription: String {
        return "\(field) \(value) is not valid because \(reason)"
    }
}

enum UseCaseError: LocalizedError {
    case validation(ValidationError)
    case update(Error)
    case sendMail(Error)
    
    var localizedDescription: String {
        switch self {
        case .validation(let error):
            return error.localizedDescription
        case .update(let error):
            return "update error \(error)"
        case .sendMail(let error):
            return "sendMail error \(error)"
        }
    }
}

struct DB {
    func updateDb(from request: Request) -> Result<Request, UseCaseError> {
        return Result { try updateDb(request) }.mapError(UseCaseError.update)
    }
    
    private func updateDb(_ request: Request) throws -> Request {
        return request
    }
}

struct SmtpServer {
    func sendEmail(to request: Request) -> Result<Request, UseCaseError> {
        return Result {
            try sendEmail(request.email)
            return request
        }.mapError(UseCaseError.sendMail)
    }
    
    private func sendEmail(_ email: String) throws -> Void {
        return
    }
}

func validateRequest(_ request: Request) -> Result<Request, UseCaseError> {
    if request.name.isEmpty {
        let error = ValidationError(field: "name", value: request.name, reason: "name should not be empty")
        return .failure(.validation(error))
    }
    return .success(request)
}
```  
</div></details>  
  
同じレスポンスを返すということは  
各関数を一つのワークフローとして組み合わせて  
全体の処理を構成できそうですね。  
  
※   
値はなんでも良い`success`  
または`UseCaseError`を持った`failure`  
を返す  
  
しかし  
それぞれの型を見ると  
  
```swift

(Request) -> Result<Request, UseCaseError>
```  
  
ということで型が合いません。  
  
どうやって各処理を連結できるようになるでしょうか？  
  
# 鉄道の車線をイメージしてみる(鉄道指向プログラミング)  
  
下記の図を見てください。  
  
![スクリーンショット 2019-06-01 5.58.31.png](/image/b74e6124-6e07-ccfc-ff9b-ba6661783b06.png)  
  
  
一つのインプットを与えると一つのアウトプットを出力する関数を  
**鉄道の車線**に例えています。  
  
  
次の図を見てください。  
  
![スクリーンショット 2019-06-01 9.44.17.png](/image/62ce6c64-43c5-cc5f-1942-88da75d13662.png)  
![スクリーンショット 2019-06-01 9.46.21.png](/image/b3ee6134-e69d-b08e-0b8b-004dd9319dd5.png)  
![スクリーンショット 2019-06-01 9.46.28.png](/image/d6a2f541-48cf-7ca0-708f-e96f0b22c2f1.png)  
  
もう一つ車線が出てきました。  
左の関数のアウトプットと右の関数のインプットが一致しているため  
この二つの車線を繋げることができます。  
  
このような場合は  
シンプルですぐに理解できると思います。  
  
`Result`の場合はアウトプットが2つの可能性があります。  
これを表現するためには**車線の分岐**が必要になります。  
  
![スクリーンショット 2019-06-01 9.52.25.png](/image/644a7a37-cdc2-1a4a-a509-ad81bfa578b6.png)  
  
`Success`車線と`Failure`車線ができます。  
  
このような分岐を生じる関数を  
**スイッチ関数**  
と呼びます。  
  
ではスイッチ関数を連結した場合の動きはどうなるでしょうか？  
  
![スクリーンショット 2019-06-01 10.02.15.png](/image/031cfc63-146a-5c81-6f09-d3d67c965fdd.png)  
![スクリーンショット 2019-06-01 10.02.23.png](/image/7d76e158-8b17-374e-792f-758866f59332.png)  
  
上記の例で説明すると  
`Validate`関数が  
成功した場合 -> `Success`車線を通り`UpdateDb`を実行する  
失敗した場合 -> `Failure`車線を通り`UpdateDb`は**実行せず**に`Failure`車線を通り続ける  
  
となります。  
  
言い換えると  
  
**処理が成功している場合のみ処理を継続し**  
**エラーが発生した場合は以降の処理を行わずに最終的なアウトプットまで進む**  
  
ことになります。  
  
なんとなくイメージはできたでしょうか？  
  
では  
問題の`Result`の連結方法について車線で考えてみます。  
  
上記で示した1つの車線の関数の連結はシンプルでした。  
  
同様にインプットとアウトプットが２車線同士の関数の場合もシンプルです。  
  
![スクリーンショット 2019-06-01 10.21.28.png](/image/2a70e250-2a06-43a5-2839-0d503bf5056c.png)  
  
インプットとアウトプットが一致すればそのまま繋げることができます。  
  
![スクリーンショット 2019-06-01 10.27.22.png](/image/ffefb226-f7a0-6b6a-4d6d-0b356ef9f7b0.png)  
  
しかし`Result`を返すような関数は通常の値をインプットとして受け取るため  
インプットとアウトプットが一致するため連結できません。  
  
ではどうすれば良いのか？  
  
**車線の数を合わせれば良い**  
  
のです。  
  
１車線インプット、２車線アウトプットの関数から  
２車線インプット、２車線アウトプットの関数へ変換する  
ことで２車線関数同士の関数を繋ぎ合わせることと同じになります。  
  
![スクリーンショット 2019-06-01 10.27.33.png](/image/3d1cc26e-ff0c-459b-66e5-c2c5ff003dca.png)  
  
  
![スクリーンショット 2019-06-01 10.27.47.png](/image/53a0b505-0191-e535-2f86-52e811004fc5.png)  
  
それを実現するのが**flatMap**です。  
Scottさんの講演では**アダプターブロック**と読んでいました。  
  
<details><summary>flatMapの実装</summary><div>  
  
```swift

public enum Result<Success, Failure: Error> {
    case success(Success)    
    case failure(Failure)
    
    public func flatMap<NewSuccess>(
        _ transform: (Success) -> Result<NewSuccess, Failure>
        ) -> Result<NewSuccess, Failure> {
        switch self {
        case let .success(success):
            return transform(success)
        case let .failure(failure):
            return .failure(failure)
        }
    }
}
```  
</div></details>  
  
https://github.com/apple/swift/blob/master/stdlib/public/core/Result.swift#L96  
  
引数に  
`Result`の`Success`型を引数にして  
変換して`Result<NewSuccess, Error>`を返す関数を受け取り  
`Result<NewSuccess, Error>`を返します。  
  
この`transform`の形に注目すると  
  
```swift

(Success) -> Result<NewSuccess, Failure>
```  
これはまさに**スイッチ関数と同じ形**です。  
  
Scottさんの講演では下記のような**bind関数**を定義しています。  
  
```swift
func bind<A,B>(_ switchFuntion: @escaping (A) -> Result<B, Error>) -> (Result<A, Error>) -> Result<B, Error> {
    return { (a: Result<A, Error>) in
        switch a {
        case .success(let x):
            return switchFuntion(x)
            
        case .failure(let error):
            return .failure(error)
        }
    }
}
```  
  
これを活用することもできますが  
Swiftの標準で使われているメソッドで考えていきたいと思います。  
  
  
# Resultを用いた処理の例  
  
最初の方で命令型で書いた例を`Result`を使った形で考えてみます。  
  
<details><summary>Resultを使った実装例</summary><div>  
  
```swift

...

struct DB {
    func updateDb(from request: Request) -> Result<Request, UseCaseError> {
        return Result { try updateDb(request) }.mapError(UseCaseError.update)
    }
    
    func updateDb(_ request: Request) throws -> Request {
        return request
    }
}

struct SmtpServer {
    func sendEmail(to request: Request) -> Result<Request, UseCaseError> {
        return Result {
            try sendEmail(request.email)
            return request
        }.mapError(UseCaseError.sendMail)
    }
    
    func sendEmail(_ email: String) throws -> Void {
        return
    }
}

func validateRequest(_ request: Request) -> Result<Request, UseCaseError> {
    if request.name.isEmpty {
        let error = ValidationError(field: "name", value: request.name, reason: "name should not be empty")
        return .failure(.validation(error))
    }
    return .success(request)
}

func executeUserCase(request: Request, db: DB, stmpServer: SmtpServer) -> String {
    
    switch validateRequest(request)
        .flatMap(db.updateDb)
        .flatMap(stmpServer.sendEmail(to:)) {
    case .success:
        return "OK"
    case .failure(let error):
        return error.localizedDescription
    }
}
```  
  
</div></details>  
  
`flatMap`を使うことで連結ができるようになりました。  
  
それでは動作を確認してみます。  
  
```swift

let request = Request(userId: 1, name: "hoge", email: "hoge@hoge.com")
let result = executeUseCase(request: request, db: DB(), stmpServer: SmtpServer())
print(result) // success("OK")
```  
  
値がきちんと設定されている場合は`success`になります。  
  
  
では`name`を空文字にしてみたいと思います。  
  
```swift

let request = Request(userId: 1, name: "", email: "hoge@hoge.com")
let result = executeUseCase(request: request, db: DB(), stmpServer: SmtpServer())

print(result) 
// failure(UseCaseError.validation(ValidationError(field: "name", value: "", reason: "name should not be empty")))
```  
  
`UseCaseError.validation`が出力されました。  
  
想定だとそれ以降のメソッドは呼ばれていないはずですので確認をします。  
  
```swift

struct DB {
    func updateDb(from request: Request) -> Result<Request, UseCaseError> {
        print("updateDb")
        return Result { try updateDb(request) }.mapError(UseCaseError.update)
    }
}

struct SmtpServer {
    func sendEmail(to request: Request) -> Result<Request, UseCaseError> {
        print("sendEmail")
        return Result {
            try sendEmail(request.email)
            return request
        }.mapError(UseCaseError.sendMail)
    }
}
```  
この状態でもう一度`success`を出力すると  
  
```swift

// updateDb
// sendEmail
// success("OK")
```  
  
と出ますが  
`name`を空文字すると  
  
```swift

// failure(UseCaseError.validation(ValidationError(field: "name", value: "", reason: "name should not be empty")))
```  
とエラーのみ出力され  
その先のメソッドが呼ばれていないことが確認できました。  
  
# 他のメソッドと組み合わせるには？  
  
下記のメソッドを追加したいとします。  
  
```swift
func canonicalizeEmail(_ request: Request) -> Request {
    let canonicalized = request.email.trimmingCharacters(in: .whitespaces).lowercased()
    return Request(userId: request.userId, name: request.name, email: canonicalized)
}
```  
  
これは`Result`は登場しない１車線関数です。  
  
これも連結して処理できるようにしたいですが  
  
![スクリーンショット 2019-06-01 15.44.01.png](/image/c30f3ca5-ea1d-d246-a06e-4422449d872f.png)  
  
１車線関数のアプトプットと２車線関数のインプットをそのまま繋げることはできません。  
同様に2車線関数のインプットと1車線関数のアウトプットをそのまま繋げることはできません。  
  
ではどうするか？  
  
これも`flatMap`の時と同じように考えます。  
  
![スクリーンショット 2019-06-01 15.48.19.png](/image/5b4ff3d5-7aa9-f396-5614-c926a5a0a0c7.png)  
  
つまり１車線関数を２車線関数に変換するようにします。  
  
これを実現するのは**map**です。  
  
<details><summary>Mapの実装</summary><div>  
  
```swift

public enum Result<Success, Failure: Error> {
    
    public func map<NewSuccess>(
        _ transform: (Success) -> NewSuccess
        ) -> Result<NewSuccess, Failure> {
        switch self {
        case let .success(success):
            return .success(transform(success))
        case let .failure(failure):
            return .failure(failure)
        }
    }
}
```  
  
</div></details>  
  
https://github.com/apple/swift/blob/master/stdlib/public/core/Result.swift#L41  
  
`transform`を見てみると１車線関数を渡して`Result`型に変換して返します。  
  
これを使用すると  
  
```swift

func executeUserCase(request: Request, db: DB, stmpServer: SmtpServer) -> String {    
    
    switch validateRequest(request)
        .map(canonicalizeEmail) // ← ここ
        .flatMap(db.updateDb)
        .flatMap(stmpServer.sendEmail(to:)) {

    case .success:
        return "OK"
    case .failure(let error):
        return error.localizedDescription
}
```  
  
と上記のように連結することができました。  
  
  
次に下記のメソッドを考えてみたいと思います。  
  
```swift

struct DB {
    func updateDbVoid(_ request: Request) throws -> Void {
        return
    }
}
```  
  
これはまず１車線関数のため連結できません。  
さらにアウトプットがないため`map`を使っても連結ができません。  
  
※こういう関数は**デッドエンド関数**と呼ばれているそうです。  
  
![スクリーンショット 2019-06-01 16.33.01.png](/image/97c1742f-aadd-7324-2753-e341a3c66506.png)  
  
ではどうするのか？  
  
インプットで受け取った値を内部で関数を実行した後に  
そのままインプットの値を返すようにします。  
  
  
![スクリーンショット 2019-06-01 16.35.56.png](/image/5d13e9e5-d947-1e66-d0a3-8ebe1005f229.png)  
  
  
今回はメソッドを一つ追加します。  
  
  
```swift

extension Result {
    static func tee(_ f: @escaping (Success) -> ()) -> (Success) -> Result<Success, Failure> {
        return { a in
            f(a)
            return .success(a)
        }
    }

    static func tee(_ f: @escaping (Success) throws -> ()) -> (Success) -> Result<Success, Error> {
        return { a in
            do {
                try f(a)
                return .success(a)
            } catch {
                return .failure(error)
            }
        }
    }
}
```  
  
これを先ほどの`updateDbVoid`に適用します。  
  
```swift

struct DB {
    
    func updateDb(_ request: Request) -> Result<Request, UseCaseError> {
        return Result<Request, UseCaseError>
            .tee(self.updateDbVoid)(request).mapError(UseCaseError.update)
    }

    func updateDbVoid(_ request: Request) throws -> Void {
        return
    }
}
```  
  
こうすると今までと同じように扱うことができます。  
  
```swift

func executeUserCase(request: Request, db: DB, stmpServer: SmtpServer) -> String {
    
    switch validateRequest(request)
        .map(canonicalizeEmail)
        .flatMap(db.updateDb)
        .flatMap(stmpServer.sendEmail(to:)) {
    case .success:
        return "OK"
    case .failure(let error):
        return error.localizedDescription
    }
}
```  
  
# Resultを使うことで得られること  
  
これまで見てきたように  
`Result`を使うことで  
全体で統一的にエラーハンドリングを行えるようになりました。  
  
さらに各メソッドの型を見てみると  
  
```swift

executeUserCase: (Request, DB, SmtpServer) -> String

validateRequest: (Request) -> Result<Request, UseCaseError>
canonicalizeEmail: (Request) -> Request
updateDb: (Request) -> Result<Request, UseCaseError>
sendEmail(Request) -> Result<Request, UseCaseError>
```  
  
となっています。  
  
`Result`を使ったことで  
メソッドが失敗するかもしれないということを  
目で見てわかるようになりました。  
  
具体的なエラーの内容は`enum`で確認できます。  
  
```swift

enum UseCaseError {
    case validation(ValidationError)
    case update(Error)
    case sendMail(Error)
}

... 一部省略
```  
  
こうすることで  
他の人が見てもどういう処理をしているのかを型で伝えやすくなり  
いわゆる**自己文書化（Self-documenting）**につながります。  
  
※ 自己文書化については下記のサイトなどに詳しく書かれています  
https://www.webprofessional.jp/self-documenting-javascript/  
  
  
# その他の鉄道指向プログラミング  
  
この他にも色々な場合に関しての  
鉄道指向プログラミングのアプローチが紹介されています。  
  
いくつか挙げたいと思います。  
  
## 複数のエラーが欲しい場合は？  
  
これまで見てきたケースですと  
エラーは一つしか扱うことができません。  
  
![スクリーンショット 2019-06-02 5.12.58.png](/image/dc312f61-85d6-fd37-f878-60c6ace6f3ed.png)  
  
  
例えばバリデーションチェックのエラーは  
発生した全てのエラーが欲しい場合があるかもしれません。  
  
そのような場合  
講演の中では詳細には触れていませんが  
それぞれの処理をPairとして組み合わせていけば  
いくつでも組み合わせることができる  
といったことをおっしゃっています。  
  
![スクリーンショット 2019-06-02 5.15.09.png](/image/626fe28e-37e2-f760-9e9c-02824e49997d.png)  
  
講演で紹介されていたブログの内容はこちら↓  
https://fsharpforfunandprofit.com/posts/monoids-without-tears/  
  
  
SwiftではiOS13よりCombineというフレームワークが増え  
その中で`Zip`が定義されています。  
https://developer.apple.com/documentation/combine/publishers/zip  
※2019/6/22現在のベータ版の情報です。  
  
また  
`zip`を定義しているライブラリがあります。  
  
例えば下記のライブラリでは`zip`というメソッドで複数の処理結果をペアの組み合わせにしています。  
https://github.com/pointfreeco/swift-validated  
  
※ちなみに`zip`はHaskellなどの関数型プログラミングでも定義されており  
同様の使われ方をされています。  
  
RxSwiftでも使用されています。  
https://github.com/ReactiveX/RxSwift/blob/master/RxSwift/Observables/Zip.swift  
https://github.com/ReactiveX/RxSwift/blob/master/RxSwift/Observables/Zip+arity.swift#L23  
  
  
## ドメインイベントなどのメッセージを渡したい場合  
  
処理結果以外に他の機能にメッセージを送りたい場合もあるかもしれません。  
(講演ではメールの送信に成功したことをCRMに伝えるなど)  
  
  
![スクリーンショット 2019-06-02 5.35.12.png](/image/85703d54-5070-50b3-e4c1-d38e9383ae13.png)  
  
  
このような場合は  
`success`の場合にメッセージをリストとして持ち  
処理結果とメッセージのリストの組み合わせを伝達するようにします。  
  
![スクリーンショット 2019-06-02 5.33.47.png](/image/837aee9c-f548-6d28-ed40-d73a690789f7.png)  
  
  
<details><summary>Githubを参考にSwiftでも実装してみました(結構無理あり:sweat_smile:)</summary><div>  
  
```swift

struct Request {
    let userId: Int
    let name: String
    let email: String
}

struct ValidationError: LocalizedError {
    let field: String
    let value: Any
    let reason: String

    var localizedDescription: String {
        return "\(field): \(value) is not valid because \(reason)"
    }
}

enum UseCaseMessage: LocalizedError {
    case validation(ValidationError)
    case update(Error)
    case sendMail(Error)


    case UpdateSuccess
    case SendMailSuccess

    var localizedDescription: String {
        switch self {
        case .validation(let error):
            return error.localizedDescription
        case .update(let error):
            return "update error \(error)"
        case .sendMail(let error):
            return "sendMail error \(error)"
        case .UpdateSuccess:
            return "Update Success"
        case .SendMailSuccess:
            return "Send Mail Success"
        }
    }
}

extension Array: Error where Element: Error {}

enum ROPResult<Success, Message> {
    case success(Success, [Message])
    case failure([Message])

    func map<NewSuccess>(_ f: (Success) -> NewSuccess) -> ROPResult<NewSuccess, Message> {
        switch self {
        case .success(let x, let msgs):
            return .success(f(x), msgs)
        case .failure(let errors):
            return .failure(errors)
        }
    }

    func flatMap<NewSuccess>(
        _ transform: (Success) -> ROPResult<NewSuccess, Message>
        ) -> ROPResult<NewSuccess, Message> {
        switch self {
        case .success(let x, let msgs):
            do {
                let result = try transform(x).get()
                return .success(result.0, result.1 + msgs)
            } catch let errors as [Error] {
                return .failure(errors as! [Message])
            } catch {
                return .failure([error] as! [Message])
            }
        case .failure(let errors):
            return .failure(errors)
        }
    }

    func get() throws -> (Success, [Message]) {
        switch self {
        case .success(let x, let msgs):
            return (x, msgs)
        case .failure(let errors):
            throw errors as! [Error]
        }
    }

    func mapError(
        _ transform: (Message) -> Message
        ) -> ROPResult<Success, Message> {
        switch self {
        case .success(let x, let msgs):
            return .success(x, msgs)
        case .failure(let errors):
            let newErrors = errors.map(transform)
            return .failure(newErrors)
        }
    }

    static func tee(_ f: @escaping (Success) throws -> (), msgs: [Message] = []) -> (Success) -> ROPResult {
        return { a in
            do {
                try f(a)
                return .success(a, msgs)
            } catch {
                return .failure([error] as! [Message])
            }
        }
    }
}

extension ROPResult where Message == Swift.Error {
    init(catching body: () throws -> (Success, Message)) {
        do {
            let result = try body()
            self = .success(result.0, [result.1])
        } catch {
            self = .failure([error])
        }
    }
}

struct DB {
    func updateDb(_ request: Request) -> ROPResult<Request, UseCaseMessage> {
        return ROPResult<Request, UseCaseMessage>
            .tee(self.updateDbVoid, msgs: [UseCaseMessage.UpdateSuccess])(request)
            .mapError(UseCaseMessage.update)
    }

    func updateDbVoid(_ request: Request) throws -> Void {
        return
    }
}

struct SmtpServer {
    func sendEmail(to request: Request) -> ROPResult<Request, UseCaseMessage> {
        do {
            try sendEmail(request.email)
            return .success(request, [UseCaseMessage.SendMailSuccess])
        } catch {
            return .failure([UseCaseMessage.sendMail(error)])
        }
    }

    func sendEmail(_ email: String) throws -> Void {
        return
    }
}

func validateRequest(_ request: Request) -> ROPResult<Request, UseCaseMessage> {
    if request.name.isEmpty {
        let error = ValidationError(field: "name", value: request.name, reason: "name should not be empty")
        return .failure([.validation(error)])
    }
    return .success(request, [])
}

func canonicalizeEmail(_ request: Request) -> Request {
    let canonicalized = request.email.trimmingCharacters(in: .whitespaces).lowercased()
    return Request(userId: request.userId, name: request.name, email: canonicalized)
}

func executeUserCase(request: Request, db: DB, stmpServer: SmtpServer) -> String {

    switch validateRequest(request)
        .map(canonicalizeEmail)
        .flatMap(db.updateDb)
        .flatMap(stmpServer.sendEmail(to:)) {
    case .success(let x, let messages):
        // Request(userId: 1, name: "hoge", email: "hoge@hoge.com")
        print("\(x)")

        // Messages are [UseCaseMessage.SendMailSuccess, UseCaseMessage.UpdateSuccess]
        print("Messages are \(messages)")

        return "OK"
    case .failure(let errors):
        return errors.reduce("", { total, error in
            return total + error.localizedDescription
        })
    }
}

let request = Request(userId: 1, name: "hoge", email: "hoge@hoge.com")
let result = executeUserCase(request: request, db: DB(), stmpServer: SmtpServer())
print("result is \(result)") // result is OK

let errorRequest = Request(userId: 1, name: "", email: "hoge@hoge.com")
let errorResult = executeUserCase(request: errorRequest, db: DB(), stmpServer: SmtpServer())
print("errorResult is \(errorResult)") // errorResult is name:  is not valid because name should not be empty


```  
  
</div></details>  
  
  
  
※ 講演の中では一覧ができるから見やすいとのことで  
エラーとドメインのメッセージを一緒に扱っていますが  
エラーとメッセージは別に違う型でも  
良いのではないかなと個人的には思っています。  
そもそもResultで分岐しているので分かるから良いのかもしれませんが。  
  
<details><summary>Failureを分けてみた例(これも結構無理あり:sweat_smile:)</summary><div>  
  
```swift

struct Request {
    let userId: Int
    let name: String
    let email: String
}

struct ValidationError: LocalizedError {
    let field: String
    let value: Any
    let reason: String
    
    var localizedDescription: String {
        return "\(field): \(value) is not valid because \(reason)"
    }
}

enum UseCaseError: LocalizedError {
    case validation(ValidationError)
    case update(Error)
    case sendMail(Error)
    
    var localizedDescription: String {
        switch self {
        case .validation(let error):
            return error.localizedDescription
        case .update(let error):
            return "update error \(error)"
        case .sendMail(let error):
            return "sendMail error \(error)"
        }
    }
}

enum UseCaseMessage {
    case UpdateSuccess
    case SendMailSuccess
}

extension Array: Error where Element: Error {}

enum ROPResult<Success, Failure: Error, Message> {
    case success(Success, [Message])
    case failure([Failure])
    
    func map<NewSuccess>(_ f: (Success) -> NewSuccess) -> ROPResult<NewSuccess, Failure, Message> {
        switch self {
        case .success(let x, let msgs):
            return .success(f(x), msgs)
        case .failure(let errors):
            return .failure(errors)
        }
    }

    func flatMap<NewSuccess>(
        _ transform: (Success) -> ROPResult<NewSuccess, Failure, Message>
        ) -> ROPResult<NewSuccess, Failure, Message> {
        switch self {
        case .success(let x, let msgs):
            
            do {
                let result = try transform(x).get()
                return .success(result.0, result.1 + msgs)
            } catch let errors as [Error] {
                return .failure(errors as! [Failure])
            } catch {
                return .failure([error] as! [Failure])
            }
        case .failure(let errors):
            return .failure(errors)
        }
    }
    
    func get() throws -> (Success, [Message]) {
        switch self {
        case .success(let x, let msgs):
            return (x, msgs)
        case .failure(let errors):
            throw errors
        }
    }

    func mapError<NewFailure>(
        _ transform: (Failure) -> NewFailure
        ) -> ROPResult<Success, NewFailure, Message> {
        switch self {
        case .success(let x, let msgs):
            return .success(x, msgs)
        case .failure(let errors):
            let newErrors = errors.map(transform)
            return .failure(newErrors)
        }
    }
    
    static func tee(_ f: @escaping (Success) throws -> (), msgs: [Message] = []) -> (Success) -> ROPResult {
        return { a in
            do {
                try f(a)
                return .success(a, msgs)
            } catch {
                return .failure([error] as! [Failure])
            }
        }
    }
}

extension ROPResult where Failure == Swift.Error {
    init(catching body: () throws -> (Success, Message)) {
        do {
            let result = try body()
            self = .success(result.0, [result.1])
        } catch {
            self = .failure([error])
        }
    }
}

struct DB {
    func updateDb(_ request: Request) -> ROPResult<Request, UseCaseError, UseCaseMessage> {
        return ROPResult<Request, UseCaseError, UseCaseMessage>
            .tee(self.updateDbVoid, msgs: [UseCaseMessage.UpdateSuccess])(request)
            .mapError(UseCaseError.update)
    }

    func updateDbVoid(_ request: Request) throws -> Void {
        return
    }
}

struct SmtpServer {
    func sendEmail(to request: Request) -> ROPResult<Request, UseCaseError, UseCaseMessage> {
        do {
            try sendEmail(request.email)
            return .success(request, [UseCaseMessage.SendMailSuccess])
        } catch {
            return .failure([UseCaseError.sendMail(error)])
        }
    }

    func sendEmail(_ email: String) throws -> Void {
        return
    }
}

func validateRequest(_ request: Request) -> ROPResult<Request, UseCaseError, UseCaseMessage> {
    if request.name.isEmpty {
        let error = ValidationError(field: "name", value: request.name, reason: "name should not be empty")
        return .failure([.validation(error)])
    }
    return .success(request, [])
}

func canonicalizeEmail(_ request: Request) -> Request {
    let canonicalized = request.email.trimmingCharacters(in: .whitespaces).lowercased()
    return Request(userId: request.userId, name: request.name, email: canonicalized)
}

func executeUserCase(request: Request, db: DB, stmpServer: SmtpServer) -> String {

    switch validateRequest(request)
        .map(canonicalizeEmail)
        .flatMap(db.updateDb)
        .flatMap(stmpServer.sendEmail(to:)) {
    case .success(let x, let messages):        
        print("\(x)")
        print("Messages are \(messages)")
        return "OK"
    case .failure(let errors):
        return errors.reduce("", { total, error in
            return total + error.localizedDescription
        })
    }
}
```  
  
</div></details>  
  
## 非同期処理  
  
講演内では具体的に扱っていませんでしたが  
  
鉄道指向プログラミングは  
全ての処理をシーケンシャルに扱うという訳ではなく  
インプットとアウトプットをどうやって繋げていくのかということを示しています。  
  
なのでレールの途中は並列に処理を行うこともあります。  
  
F#では`async`のような  
非同期を同期的に扱う仕組みがあるので  
それを活用することで変わらず形でコードを書くことができます。  
  
より複雑な処理の場合は  
メッセージを送ることで他のワークフローに任せてしまうなどを挙げていました。  
  
  
SwiftでもCombineフレームワークの中で  
`Future`が定義されています。  
https://developer.apple.com/documentation/combine/publishers/future  
  
また`Future`の中で  
非同期処理のコールバックで`Promise`という型を受け取っていますが  
これは`(Result<Output, Failure>) -> Void`の`typealias`です。  
https://developer.apple.com/documentation/combine/publishers/future/promise  
  
※2019/6/22現在のベータ版の情報です。  
  
  
他にも非同期を同期的に扱うライブラリが活用できます。  
  
ライブラリの参考例  
https://github.com/malcommac/Hydra  
  
また将来的には標準として採用される予定の`async/await`などが活用できる可能性があります。  
  
これらの他にもログや処理失敗時DBのロールバックなどについても少し言及されていたので  
ご興味のある方はスライドや動画をご参照ください。  
  
  
# 補足: EitherやMonadとの違い  
  
Scottさんもサイトで言及されていましたが  
鉄道指向プログラミングは  
下記のような理由でHaskellの用語を用いていないと言っています。  
  
### より具体的な形で多くの人に理解して欲しい  
  
これは特定のエラーハンドリングの問題を解決するためのものであり  
モナドを知らない人にも  
まずはより目に見える具体的な形で見てもらいたかった。  
  
具体的なものから抽象概念を理解する方が理解の進むが早いと強く思っている。  
  
### 正確にモナド則に従っているわけではない  
  
`flatMap`はmonadに必ずしも従っているわけではなく  
モナドの方がもっと複雑。  
  
### Eitherは抽象的すぎる  
  
道具ではなくレシピを提示したかった。  
  
パンを作るためのレシピが欲しいのに  
「小麦粉とオーブンを使え」とだけ言うのが役に立たないのと同様に  
エラーハンドリングのためのレシピが欲しいのに  
「`Either`と`bind(flatMap)`を使え」  
とだけ言うのは役に立たない。  
  
なので具体的なカスタムオペレーターや  
`map`、`tee`などの数多くの状況に使えるけれども  
書き方は一つに限定されるようなテンプレートを提供したかった。  
  
こうすることで後々誰がメンテナンスしても全体像が理解しやすくなって楽になる。  
  
# 最後に  
  
鉄道指向プログラミングの概要と  
`Result`の動きを見てみました。  
  
鉄道指向プログラミングでは  
型を合わせていくことに焦点を当てており  
型を通して処理を考えることの大切さや効果といったことを学べました。  
  
またF#というあまり触れる機会がない言語に触れ  
普段とは違う考え方やコードの書き方を知り  
すごい勉強になりました。  
  
今回は出てきませんが  
鉄道指向プログラミングは  
ドメイン駆動設計などの話にも繋がっており  
Scottさんもドメインモデルについての本や講演もされています。  
  
https://fsharpforfunandprofit.com/books/  
https://www.youtube.com/watch?v=Up7LcbGZFuo  
  
まだまだ私が理解できていない部分も多々あると思いますので  
引き続き学んでみたいと思います。  
  
  
間違いなどございましたらご指摘して頂けますとうれしいです:bow_tone1:  
