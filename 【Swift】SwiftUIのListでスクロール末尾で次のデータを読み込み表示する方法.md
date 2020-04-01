SwiftUIでListを使って一覧を表示するサンプルはよく見かけますが  
画面表示時に全てのデータを取得して設定するものが多く  
APIなどでページを分けて読み込む方法がなかなか見つけられませんでした。  
  
そこで色々調べていて  
一つ方法を見つけましたので書いてみました。  
  
もし間違いやうまくいかないケースなどございましたら  
ご指摘いただけますとうれしいです:bow_tone1:  
  
今回必要となるものは  
  
- `RandomAccessCollection`の拡張  
- `onAppear`メソッド  
  
https://developer.apple.com/documentation/swift/randomaccesscollection  
https://developer.apple.com/documentation/swiftui/text/3276931-onappear  
  
です。  
  
※ Bata版のためSnapshotは載せられないのでコードだけ示させていただきます🙇🏻‍♂️  
※ 細かい処理などは省略しています🙇🏻‍♂️  
  
# 実装  
  
内容としてはQiitaのAPIから記事の一覧を取得します。  
  
## RandomAccessCollectionの拡張  
  
こちらを参考にしています。  
  
https://github.com/crelies/ListPagination/blob/master/Sources/ListPagination/public/Extensions/RandomAccessCollection%2BisLastItem.swift  
  
```swift

extension RandomAccessCollection where Self.Element: Identifiable {
    public func isLastItem<Item: Identifiable>(_ item: Item) -> Bool {
        guard !isEmpty else {
            return false
        }
        
        guard let itemIndex = lastIndex(where: { AnyHashable($0.id) == AnyHashable(item.id) }) else {
            return false
        }
        
        let distance = self.distance(from: itemIndex, to: endIndex)
        return distance == 1
    }    
}
```  
  
やっていることは  
引数に渡されたitemの次のitemが  
配列の一番最後のitemの場合に  
trueを返すようにします。  
  
  
  
## データの取得  
  
次にデータを取得します。  
  
  
```swift

import Foundation

// QiitaのAPIから取得するデータモデル

struct QiitaItem: Decodable, Equatable, Identifiable {
    let id: String
    let likesCount: Int
    let reactionsCount: Int
    let commentsCount: Int
    let title: String
    let createdAt: String
    let updatedAt: String
    let url: URL
    let tags: [Tag]
    let user: User?
    
    var profileImageURL: URL? {
        guard let url = user?.profileImageUrl else {
            return nil
        }
        return URL(string: url)
    }
    
    struct Tag: Decodable, Equatable {
        let name: String
    }
    
    struct User: Decodable, Equatable {
        let githubLoginName: String?
        let profileImageUrl: String?
    }
}
```  
  
```swift

import Combine
import Foundation

final class ListViewModel: ObservableObject {
    
    @Published var items: [QiitaItem] = []
    @Published var isLoading = false
    
    private var cancellables: Set<AnyCancellable> = []
    
    private let perPage = 20
    private var currentPage = 1
    
    func loadNext(item: QiitaItem) {
        if items.isLastItem(item) {
            self.currentPage += 1
            getQiitaList(page: currentPage, perPage: perPage) { [weak self] result in
                self?.handleResult(result)
            }
        }
    }
    
    func onAppear() {
        getQiitaList(page: currentPage, perPage: perPage) { [weak self] result in
             self?.handleResult(result)
        }
    }
    
    private func getQiitaList(page: Int, perPage: Int,
                              completion: @escaping (Result<[QiitaItem], Error>) -> Void) {
        
        let parameters: [String: Any] = [
            "page": currentPage,
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
                self.currentPage += 1
                self.items.append(contentsOf: items)
            case .failure(let error):
                self.currentPage = 1
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
    case resultError
}
```  
  
## 画面への表示  
  
```swift

import SwiftUI

struct ContentView: View {
    
    @ObservedObject var viewModel: ListViewModel
    
    var body: some View {
        NavigationView {
            List(viewModel.items) { item in
                Text(item.title)
                    .onAppear {
                        self.viewModel.loadNext(item: item)
                }
            }.onAppear {
                self.viewModel.onAppear()
            }.navigationBarTitle("検索結果")
        }
    }
}
```  
  
ここでのポイントは  
  
```swift

Text(item.title)
    .onAppear {
        self.viewModel.loadNext(item: item)
}
```  
  
でListの中のデータが画面に表示された時に  
ListViewModelのloadNextを呼び出して  
現在取得しているデータ配列の最後の一つ手前のデータだった場合に  
次のデータを取得しています。  
  
# まとめ  
  
スクロール末尾の検知(みたいなこと)  
それに合わせた次のページの読み込み  
ができるということは確認できました。  
  
まだやってみたという段階なので  
まだ見えていない問題などが出てくるかもしれませんので  
今後も見つけたら更新します。  
  
参考にしたRandomAccessCollectionの拡張には  
offset使って読み込むタイミングを調整する方法もありますので  
ぜひそちらも参考にしてみてください😀  
