## 経緯  
Swift4.1からJSONDecoderのKeyDecodingStrategyを設定することで  
スネークケースからキャメルケースへの変換を自動でしてくれるようになったので確認していたところ、  
ふと気になったことがあったのでメモします。  
  
## 実装  
  
```Swift
struct Profile: Decodable {
    
    let id: Int
    let messageTitle: String
    let profileImageUrl: String
    let aaaBbbCcc: String // ??
}

class ViewController: UIViewController {

    override func viewDidLoad() {
        super.viewDidLoad()

        do {
            let json = """
            {
                "id": 1,
                "message_title": "title",
                "profile_image_url": "hogehoge",
                "Aaa_bBb_ccC": "aaaa" //??
            }
            """.data(using: .utf8)!
            
            
            let decoder = JSONDecoder()
            decoder.keyDecodingStrategy = .convertFromSnakeCase
            let result = try decoder.decode(Profile.self, from: json)
            print(result)
            
        } catch {
            print(error)
        }
    }
}
```  
  
変換結果  
  
```Swift
Profile(id: 1, 
        messageTitle: "title", 
        profileImageUrl: "hogehoge", 
        aaaBbbCcc: "aaaa") // ??
```  
  
## 疑問  
  
なんで"Aaa_bBb_ccC"がaaaBbbCccになるんだろうと思い、  
わざとエラーにした場合のコンソールの出力を見ていると  
  
```Swift
"No value associated with 
key CodingKeys(stringValue: \"aaaBBbCcC\", intValue: nil) (\"aaaBBbCcC\"), converted to aaa_b_bb_cc_c."	
```  
  
と出ていたので、  
Decodableの中で宣言した変数の大文字部分を_(アンダーバー)で区切って全部を小文字に変換したものと  
JSON文字列の方も全て小文字に変換したものを比較して  
値を設定しているような動きをしています。  
どういう仕組みなのか時間あるときに詳しく調べたいです。  
  
スネークケースの中に大文字が混ざることはあまりないかもしれませんが、  
気になったので記録しました。  
  
