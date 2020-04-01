  
URLSession を使って multipart/form-dataでPOSTを行う際、  
かなりしょうもないことではまってしまい反省のため記録を残したいと思います。  
  
また、もし同じような状態で悩んでいる方がいらっしゃれば  
何かお役に立てれば幸いです。  
  
  
結論としては  
[RFC6266](https://tools.ietf.org/html/rfc6266#section-4.1)では  
末尾のセミコロンは許可されていないということです。  
  
正しいものはこちらを参考にしてください。  
https://github.com/newfivefour/BlogPosts/blob/master/swift-form-data-multipart-upload-URLRequest.md  
  
  
以下は間違っている例です。  
  
## 現象  
  
URLSession を使ってマルチパート送信を行う際、  
下記のように設定を行なっていました。  
  
```間違い例.swift

import UIKit

class APIClient {

    func multipartPost(urlString: String, parameters: [String: Any]) {
        
        let url = URL(string: urlString)
        var request = URLRequest(url: url!)
        request.httpMethod = "POST"
        
        let (headers, body) = APIClient.createMultiPartPost(parameters: parameters)
        
        // ヘッダーの設定
        for header in headers {
            request.addValue(header.value, forHTTPHeaderField: header.key)
        }
        
        // Bodyの設定
        request.httpBody = body
        
        let task = URLSession.shared.dataTask(with: request) { data, response, error in
            if let data = data, let response = response {
                print(response)
                do {
                    let json = try JSONSerialization.jsonObject(with: data, options: JSONSerialization.ReadingOptions.allowFragments)
                    print(json)
                } catch {
                    print("parse error")
                }
            } else {
                print(error ?? "unknown error")
            }
        }
        
        task.resume()

        
    }
    
    static func createMultiPartPost(parameters: [String: Any]) -> (headers: [String:String], body: Data) {
        
        let uniqueId = UUID().uuidString
        let boundary = "---------------------------\(uniqueId)"
        
        let header = [
            "Content-Type" : "multipart/form-data; boundary=\(boundary)"
        ]
        
        var body = Data()
        
        let boundaryText = "--\(boundary)\r\n"
        
        
        for param in parameters {
            
            switch param.value {
            case let image as UIImage:
                
                let imageData = UIImageJPEGRepresentation(image, 1.0)
                
                body.append(boundaryText.data(using: .utf8)!)
                body.append("Content-Disposition: form-data; name=\"\(param.key)\"; filename=\"\(uniqueId).jpg\"\r\n".data(using: .utf8)!)
                body.append("Content-Type: image/jpeg\r\n\r\n".data(using: .utf8)!)
                
                body.append(imageData!)
                body.append("\r\n".data(using: .utf8)!)
                
            case let string as String:
                
                body.append(boundaryText.data(using: .utf8)!)
                body.append("Content-Disposition: form-data; name=\"\(param.key)\";\r\n\r\n".data(using: .utf8)!)
                body.append(string.data(using: .utf8)!)
                body.append("\r\n".data(using: .utf8)!)
                
            case let value as Any:
                
                body.append(boundaryText.data(using: .utf8)!)
                
                // ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ここが問題↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
                body.append("Content-Disposition: form-data; name=\"\(param.key)\";\r\n\r\n".data(using: .utf8)!)
                body.append(String(describing: value).data(using: .utf8)!)
                body.append("\r\n".data(using: .utf8)!)
                
            default:
                break
            }
        }
        
        body.append("--\(boundary)--\r\n".data(using: .utf8)!)
        
        return (header, body)
    }
    
}
```  
  
```使用例.swift
let client = APIClient()

let parameters:[String: Any]  = ["image": UIImage(named: "noimage"), "id": 1234]

client.multipartPost(urlString: "https://XXXXXXXXXXX", parameters: parameters)

```  
  
これで送信してみるとサーバーでパースエラーが起きます。  
  
パースに使用しているのは.netの[ReadAsMultipartAsync](https://msdn.microsoft.com/ja-jp/library/hh944544(v=vs.118).aspx  
)です。  
  
当初は改行の数が異なっていたりしていたため、そこを直せばすぐに解決するかと  
思ったのですが、改行を直しても解決せず、、、  
  
パラメータの順番を変えたり、  
画像のフォーマットを変えてみたりなど試行錯誤してみるものの一向に解決せず、、、  
  
  
しばらくしてから、ふとpostmanから送ってみたらどうなるかと思い、  
試したところ成功。  
  
ログを取得し、何が違うのかを比較すると、  
  
```
正 Content-Disposition: form-data; name=“xxxxxx”
誤 Content-Disposition: form-data; name=“xxxxxx”; 
```  
  
犯人は  
  
```
Content-Disposition: form-data; name=“xxxxxx”; ←これ
```  
でした。  
  
  
```
content-disposition = "Content-Disposition" ":"
                            disposition-type *( ";" disposition-parm )
```  
  
失敗テスト例:  
http://test.greenbytes.de/tech/tc2231/#attwithasciifilenamenqs  
  
  
## 今回の反省  
  
・ごちゃごちゃと試行錯誤する前に他の方法でうまくいくかどうかを試してみるべきであった。  
・ネットの情報を検索してそのまま引用し、RFCの確認などに行き着くまでに時間がかかった。  
・そもそもRFCに対する知識不足があった。  
  
  
<br>  
  
色々な情報を参照するにしても自分自身での確認や検証の大切さを改めて感じました。  
  
また、文字列の間違いは気がつきにくいため、  
できる限り文字列による設定は避けていけるような工夫は常に心がけていきたいと思います。  
