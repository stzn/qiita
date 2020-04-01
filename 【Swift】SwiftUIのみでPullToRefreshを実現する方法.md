次のページの読み込みに関してはこちらに記載しています。  
https://github.com/stzn/qiita/blob/master/【Swift】SwiftUIのListでスクロール末尾で次のデータを読み込み表示する方法.md  
  
SwiftUIを使っていく上でListを扱っているときに  
TableViewのように一覧を引っ張ってデータを更新する  
いわゆる**PullToRefresh**を  
SwiftUIのみで実現する実装方法がわからなかったため  
調べてみました。  
  
結論としては明確な方法がまだ見つかっていないのですが  
一応それらしく実現する方法ががありましたので  
記録として残しておきます。  
  
  
もし他に良い実装などございましたら  
教えて頂けましたらうれしいです🙇🏻‍♂️  
  
  
今回必要となるものは  
  
- `GeometryReader`  
  
https://developer.apple.com/documentation/swiftui/geometryreader  
  
です。  
  
※ Bata版のためSnapshotは載せられないのでコードだけ示させていただきます🙇🏻‍♂️  
※ 細かい処理などは省略しています🙇🏻‍♂️  
  
こちらを参考にしています。  
https://www.youtube.com/watch?v=m2o8iMU2LQA  
  
# 実装  
  
内容としてはQiitaのAPIから記事の一覧を取得します。  
  
  
## データの取得  
  
データを取得します。  
  
```swift

import Combine
import Foundation

final class ListViewModel: ObservableObject {
    
    @Published var items: [QiitaItem] = []
    @Published var isLoading = false
    
    private var cancellables: Set<AnyCancellable> = []
    
    private let perPage = 20
    
    func refresh() {
        if !isLoading {
            getQiitaList(page: 1, perPage: perPage) { [weak self] result in
                self?.handleResult(result)
            }
        }
    }

    private func getQiitaList(page: Int, perPage: Int,
                              completion: @escaping (Result<[QiitaItem], Error>) -> Void) {
        
        let parameters: [String: Any] = [
            "page": page,
            "per_page": perPage,
        ]
        
        guard let url = URL(string: "https://qiita.com/api/v2/items"),
            let request = makeGetRequest(url: url, parameters: parameters) else {
                return completion(.failure(APIError.requestError))
        }
        fetch(request: request) { result in
            completion(Result {
                let decoder = JSONDecoder()
                decoder.keyDecodingStrategy = .convertFromSnakeCase
                return try decoder.decode([QiitaItem].self, from: result.get())
            })
        }
    }

    private func fetch(request: URLRequest, completion: @escaping (Result<Data, Error>) -> Void) {
        isLoading = true
        URLSession.shared.dataTask(with: request) { data, response, error in
            if let error = error {
                return completion(.failure(APIError.responseError(error)))
            }
            guard let httpResponse = response as? HTTPURLResponse else {
                return completion(.failure(APIError.invalidResponse(response)))
            }
            guard (200 ..< 300) ~= httpResponse.statusCode else {
                return completion(.failure(APIError.invalidStatusCode(httpResponse.statusCode)))
            }
            guard let data = data else {
                return completion(.failure(APIError.noResponseData))
            }
            return completion(.success(data))
        }.resume()
    }
    
    private func makeGetRequest(url: URL, parameters: [String: Any]) -> URLRequest? {
        guard var components = URLComponents(url: url, resolvingAgainstBaseURL: false) else {
            return nil
        }
        components.queryItems = parameters.map { (arg) -> URLQueryItem in
            let (key, value) = arg
            return URLQueryItem(name: key, value: String(describing: value))
        }
        var request = URLRequest(url: components.url!)
        request.httpMethod = "GET"
        return request
    }
    
    private func handleResult(_ result: Result<[QiitaItem], Error>) {
        DispatchQueue.main.async { [weak self] in
            guard let self = self else {
                return
            }
            self.isLoading = false
            switch result {
            case .success(let items):
                self.items.append(contentsOf: items)
            case .failure(let error):
                print(error)
            }
        }
    }
    
}

enum APIError: Error {
    case requestError
    case responseError(Error)
    case invalidResponse(URLResponse?)
    case invalidStatusCode(Int)
    case noResponseData
}

```  
  
## 画面への表示  
  
次に画面に表示する実装です。  
  
```swift

import SwiftUI

struct ContentView: View {
    
    @ObservedObject var viewModel: ListViewModel
    
    var body: some View {
        NavigationView {
            List {
                // frameを取得してyの位置がある値を超えた時にrefreshメソッドを呼び出す
                GeometryReader { g -> Text in
                    let frame = g.frame(in: .global)
                    if frame.origin.y > 250 {
                        self.viewModel.refresh()
                        return Text("Loading...")
                    } else {
                        return Text("")
                    }
                }
                ForEach(self.viewModel.items) { item in
                    Text(item.title)
                }
            }.navigationBarTitle("検索結果")
        }
    }
}
```  
  
  
ポイントは  
  
```swift

GeometryReader { g -> Text in
    let frame = g.frame(in: .global)
    if frame.origin.y > 250 {
        self.viewModel.refresh()
        return Text("Loading...")
    } else {
        return Text("")
    }
}
```  
  
で`frame`を取得し  
`origin.y`の位置でListが引っ張られていることを検知します。  
今回は`Text`で「Loading...」という文字を出していますが、  
戻り値を別のViewに変更することも可能です。  
  
# まとめ  
  
`GeometryReader`を利用することで  
PullToRefreshのような動きをすることができました。  
  
しかし、どの位置でRefreshをしようとしているのかの判断を  
手動でしなければならなかったり  
ちょっと強引な気もしています。  
  
  
何か標準の機能として出てくれば嬉しいですね😃  
(あって気付いていないだけでしたらぜひ教えてください💦)  
