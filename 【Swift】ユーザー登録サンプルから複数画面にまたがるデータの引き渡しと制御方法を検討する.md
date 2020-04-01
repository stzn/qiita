  
## 経緯  
  
先日[こちら](https://mag.n26.com/creating-flows-f1c2e1bc8108)の記事を見ていたときに  
過去に行っていた開発で同じようなことで悩んだ経験があり、  
紹介されていた記事を元にサンプルを作成してみました。  
  
作成したサンプル:https://github.com/stzn/ViperSampleGeneramba  
  
※[以前の記事](https://qiita.com/sztk1209/items/605ad32ab799530d57aa)で作成したサンプルにユーザー登録フローを追加しました。  
  
  
## 問題点  
  
  
例えば、ユーザー登録を行う際、  
入力項目が多いと１つの画面で全てを入力してもらうにはスクロールが必要になり、  
操作がわずらわしくなってしまうため、  
複数の画面にまたがることがあると思います。  
  
その際に問題の１つとなるのが、途中経過のデータの扱い。  
  
データの持ち方はいくつか方法が考えられます。  
  
### 1. 引数として渡していく  
  
```1.swift

/* 下記はイメージで実装とは異なります。 */

struct PersonalInformation {
    let name: String
    let mailAddress: String
    let address: String
}

struct AccountInformation {
    let nickname: String
    let loginId: String
    let password: String
}

final class PersonalWireframe {
    func perform() {
        ...
    }

    func next(personal: PersonalInformation) {
        let wireframe = AccountWireframe(personal: personal)
        wireframe.perform()
    }
}

final class AccountWireframe {

    let personal: PersonalInformation

    init(personal: PersonalInformation) {
        self.personal = personal
    }

    func perform() {
        ...
    }

    func next(account: AccountInformation) {
        let wireframe = ConfirmWireframe(personal: personal, account: account)
        wireframe.perform()
    }
}

final class ConfirmWireframe {

    let personal = personal
    let account = account

    init(personal: personal, account: AccountInformation) {
        self.personal = personal
        self.account = account
    }

    func perform() {
        ...
    }

    func register() {
        // personalとaccountを使って登録処理
        ...
    }
}

```  
  
・問題点  
  -処理が増減したり、順番が変更になった際の修正範囲が大きい  
  -処理とは関係のないデータを保持しておく必要がある  
  
  
### 2. 専用のモデルを作成する。  
  
```2.swift

/* 下記はイメージで実装とは異なります。 */

struct UserRegistration {
    var personal: PersonalInformation?
    var account: AccountInformation?
}

final class PersonalWireframe {

    var userRegistration: UserRegistration

    init(userRegistration: UserRegistration) {
        self.userRegistration = userRegistration
    }

    func perform() {
        ...
    }

    func next(personal: PersonalInformation) {
        
        self.userRegistration.personal = personal
        let wireframe = AccountWireframe(userRegistration: userRegistration)
        wireframe.perform()
    }
}

final class AccountWireframe {

    var userRegistration: UserRegistration

    init(userRegistration: UserRegistration) {
        self.userRegistration = userRegistration
    }

    func next(account: AccountInformation) {
        self.userRegistration.account = account
        let wireframe = ConfirmWireframe(userRegistration: userRegistration)
        wireframe.perform()
    }
}

final class ConfirmWireframe {

    var userRegistration: UserRegistration

    init(userRegistration: UserRegistration) {
        self.userRegistration = userRegistration
    }

    func register() {
        // userRegistrationを使って登録処理
        ...
    }
}
```  
  
・改善点  
　-変更に対する影響範囲が狭まった。  
　-順番が変更されても他に影響が与えられない。  
  
・問題点  
　-変数がOptionalのため、値がきちんと設定されているかどうかの手動チェックが必要になる  
　-使用する場合はアンラップするための処理が増える  
　-モデルのスコープが必要以上に大きくなる  
  
以前の開発ではこの方法を用いていて、  
コンパイラの型チェックが働かない分手動でチェックをしているところが  
どうにかならないか悩んでいました。  
  
  
今回の記事では以下の方法でそれらの問題を解消することができています。  
  
  
### 3. ステップを定義してフローで制御する  
  
#### ステップ　個々の画面の処理  
  
```Step.swift
protocol Step {
    
    // 引数で受け取る型 
    associatedtype Input

    // 結果として出力する型
    associatedtype Output

    // Inputを受け取り、非同期で処理を実行し、結果をOutputとしてコールバックに渡します。
    func perform(_ input: Input, completion: @escaping (_ output: Output) -> Void)
}
```  
  
Stepの実装としては以下のような形なりました。VIPERではWireframeを用います。  
  
``` PersonalWireframe.swift
final class PersonalWireframe: Step {

    let navigation: UINavigationController
    var completion: ((PersonalInformation?) -> Void)?

    init(navigation: UINavigationController) {
        self.navigation = navigation
    }

    func perform(_ input: Void, completion: @escaping (PersonalInformation?) -> Void) {

        ...        
        self.completion = completion
        navigation.pushViewController(viewController, animated: true)
    }

    // 次のステップへ進む
    func goToNextStep(_ output: PersonalInformation?) {
        self.completion?(output)
    }
    
    // 前の画面に戻る
    func back() {
        self.completion?(nil)
    }
}
```  
  
さらに、  
呼び出し側の可読性の向上とコンパイラーの型チェックを働せるために  
Stepプロトコルを実装したジェネリックな構造体を作成します。  
  
※記事ではこれを[thunk](https://en.wikipedia.org/wiki/Thunk)と呼んでいます。  
遅延評価を行うためのラッパーオブジェクトだそうです。  
詳しくはわからないのですが、やっていることはTypeEraserも同じなのでしょうか？  
  
  
```StepT.swift
struct StepT<Input, Output>: Step {

    private let _perform: (Input, @escaping (Output) -> Void) -> Void
 
    init<P: Step>(_ step: P) where P.Input == Input, P.Output == Output {
        _perform = step.perform
    }
    
    func perform(_ input: Input, completion: @escaping (Output) -> Void) {
        self._perform(input, completion)
    }
}

```  
  
以下のように使います。  
  
```
private static func enterPersonalStep(_ navigation: UINavigationController) -> StepT<Void, PersonalInformation?> {
    return StepT(PersonalWireframe(navigation: navigation))
}

```  
<br>  
  
#### フロー 各ステップの制御  
  
```UserRegistrationFlow.swift

// User Registration Flow
struct UserRegistrationFlow {

    static func startUserRegistration(on viewController: UIViewController) {
        
        let navigation  = UINavigationController()
        enterPersonalStep(navigation).perform(()) { personal in
            
            guard let personal = personal else {
                viewController.dismiss(animated: true)
                return
            }
            
            enterLoginStep(navigation).perform(()) { account in
                
                guard let account = account else {
                    navigation.popViewController(animated: true)
                    return
                }
                
                // ※1
                SetProfileImageStep.startSetProfileImage(navigation) { image in
                    
                    let user = UserRegistration(personal: personal, account: account, profileImage: image)

                    enterConfirmStep(navigation).perform(user) { result in
                        
                        if result == nil {
                            navigation.popViewController(animated: true)
                            return
                        }
                        viewController.dismiss(animated: true)
                    }
                }
            }
        }
        viewController.present(navigation, animated: true)
    }
    
    private static func enterPersonalStep(_ navigation: UINavigationController) -> StepT<Void, PersonalInformation?> {
        return StepT(PersonalWireframe(navigation: navigation))
    }
    
    private static func enterLoginStep(_ navigation: UINavigationController) -> StepT<Void, AccountInformation?> {
        return StepT(AccountWireframe(navigation: navigation))
    }
    
    private static func enterConfirmStep(_ navigation: UINavigationController) -> StepT<UserRegistration, Bool?>  {
        return StepT(ConfirmationWireframe(navigation: navigation))
    }
    
}

// Register Profile Image Step
enum ProfileImageType {
    case own(UIImage)
    case sample
    
}

// ※1 複雑な処理が必要な場合、ステップを独立されて実行する
// 例えばステップの中でさらに別のステップが必要な場合など
struct SetProfileImageStep {
    
    static func startSetProfileImage(_ navigation: UINavigationController, complete: @escaping (UIImage) -> Void) {
        
        profileImageSelectStep(navigation).perform(()) { result in
            
            guard let result = result else {
                navigation.popViewController(animated: true)
                return
            }
            
            switch result {
            
            case let .own(image):
                complete(image)
            
            case .sample:
                
                sampleImageSelectStep(navigation).perform(()) { image in
                    guard let image = image else {
                        navigation.popViewController(animated: true)
                        return
                    }
                    complete(image)
                }
            }
        }
    }
    
    private static func profileImageSelectStep(_ navigation: UINavigationController) -> StepT<Void, ProfileImageType?>  {
        return StepT(ProfileImageSelectWireframe(navigation: navigation))
    }
    
    private static func sampleImageSelectStep(_ navigation: UINavigationController) -> StepT<Void, UIImage?>  {
        return StepT(SampleImageSelectWireframe(navigation: navigation))
    }
}
```  
  
ログイン画面で「ユーザー登録はこちら」ボタンを押すと、  
RootViewControllerから以下のようにフローを呼び出します。  
  
```RootViewController.swift

func showRegiatrationScreen() {
    UserRegistrationFlow.startUserRegistration(on: self)
}

```  
  
  
・改善点  
　-不要なモデル(上記でいうUserRegistration)が必要なくなる  
　-前の画面の値が必要な時は簡単に使うことができる  
　-処理の流れが一箇所で確認できる  
  
・問題点  
　-ステップが多くなったときのネストの深さ  
  
  
### 前の画面の値によって画面が分岐するような場合  
  
今回のサンプルでは使用していないのですが、  
記事の中では以下のようにしています。  
  
  
まず、分岐を判定する処理をフロー制御と離すためにプロトコルを宣言します。  
  
```Decision.swift

protocol Decision {
    associatedType Output
    func make() -> Output    
}

```  
  
例えばユーザーの年齢によって次に出すアンケートの画面を変更したい場合  
  
```Decisionの使用例.swift

struct User {
   let name: String
   let age: Int
}

enum UserType {
    case adult
    case child
}

struct QuestionnaireFormDecision: Decision {

   typealias Output = UserType
   
   let user: User 
   init(_ user: User) {
       self.user = user
   }

   func make() -> Output {
       return (user.age >= 20) ? .adult : .child
   }
}

```  
  
```Flowでの使用例.swift

userFormStep(navigation).perform(()) { user in
    let decision = QuestionnaireFormDecision(user)
    
    switch decision {
        case .adult:
            enterAdultQuestionnaireForm(navigtion).perform()....
        case .child:
            enterChildQuestionnaireForm(navigtion).perform()....
    }
}

```  
  
フロー制御と分岐判定を完全に離すために  
Decisionからは画面遷移判定用のEnumを返すようにし  
フローの方ではswitchで切り替えるのみにする  
とのことでした。  
  
  
## まとめ  
  
悩みの種であった途中経過のデータの扱いを解消できたことに加え、  
個々のステップ(処理)は独立しているものの、  
フローとしては一箇所で管理ができるところはすごいわかりやすいと思いました。  
  
一方で、ステップが増えるたびにどんどんネストが深くなっていくところは  
どうにかしたいなと感じました。(横スクロールはしたくないですね。)  
  
ObservableやSwiftにも導入されるかもしれない(予定の?)async/awaitを使うと  
また違った方法で上手く制御できるようになるかもしれません。  
  
  
引き続き、色々なコードや記事に触れて学習してきたいと思います。  
