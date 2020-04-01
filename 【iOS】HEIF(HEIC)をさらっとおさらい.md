  
# HEIFとは？  
「High Efficiency Image File Format」の略で  
MPEGによって2013年に開発され  
H.265/MPEG-H HEVCで利用されるフォーマットとして  
標準化されています。  
  
HEIF  
https://ja.wikipedia.org/wiki/High_Efficiency_Image_File_Format  
  
MPEG  
https://ja.wikipedia.org/wiki/Moving_Picture_Experts_Group  
  
HEVCは動画圧縮規格の一つです。  
https://ja.wikipedia.org/wiki/H.265  
  
スマートフォンなどのカメラの性能が向上してきている中で  
画像や動画のサイズもどんどん大きくなり  
保存するストレージを確保することも  
それに合わせてどんどん難しくなってきました。  
  
そんな中でストレージを節約できるフォーマットとして  
HEIFは登場してきました。  
  
HEVCそのものは  
iOS9のときからFaceTimeでサポートされていましたが  
iOS11でAVFoundationやCore Imageといったフレームワークで  
HEIFがサポートされることにより  
「カメラ」や「写真」など画像系アプリで  
HEVCにより圧縮された静止画／動画を扱えるようになりました。  
  
iOSのバージョンによる制約もありますが  
HEVCのエンコード/デコードには高い処理能力が求められるため  
iPhone7/7s以降でないと全機能を利用できません。  
  
**.heif**や**.heic**という拡張子で  
HEIFを利用します。  
  
  
## HEIFのメリット  
  
### 容量の削減  
  
高画質のままで画像の容量が軽く  
JPEGの半分くらいの容量に抑えられます。  
  
### 画像以外の情報の保存  
  
JPEGのように一枚の画像を保存するだけではなく  
  
- Live Photoのようなイメージシーケンス  
- 深度やHDRデータなどの付加情報  
- 位置やカメラなどのメタデータ  
  
なども保存ができます。  
  
### 画像編集機能  
  
画像の向きやトリミングといった機能を基本として備えており  
編集指示を画像と同じファイルに保存しておくことで  
編集履歴から画像を元に戻すなどの操作も可能になります。  
  
イメージシーケンス  
https://jp.mathworks.com/help/images/what-is-an-image-sequence.html  
  
## HEIFのデメリット  
  
### エンコード/デコード処理に時間がかかる  
  
一方でHEIFの処理には  
JPEGより時間がかかります。  
  
  
# HEICとJPEGを比較する  
  
まだiPhone 6sはiOS13にバージョンアップできるものの  
HEIFの全機能を利用可能になるデバイスを使用するユーザが大半になると考えられ  
会社の方針としても  
アプリは最新バージョンの2つ前のiOS11以降をサポート対象とするため  
このタイミング(iOS13正式リリース時点)で  
HEIFについて改めて見てみました。  
  
今回は下記のような画面を使って  
HEICとJPEG画像生成時の  
処理時間やサイズを比較してみました。  
  
![Simulator Screen Shot - iPhone 11 Pro Max - 2019-10-05 at 12.02.31.png](/image/b14c54fc-7e57-c921-c034-f9d0a5697ed7.png)  
  
画像を選択するとHEICとJPEGの画像を作成し  
圧縮する時間と画像サイズを出力するだけの簡単なものです。  
  
# (補足)HEICデータの作成方法  
  
HEICの生成するには  
AVFoundationを利用します。  
  
```swift

enum HEICError: Error {
    case heicNotSupported
    case cgImageMissing
    case couldNotFinalize
}

extension UIImage {
    func heicData(compressionQuality: CGFloat) throws -> Data {
        let data = NSMutableData()
        guard let imageDestination =
            CGImageDestinationCreateWithData(
                data, AVFileType.heic as CFString, 1, nil
            ) else {
                throw HEICError.heicNotSupported
        }
        
        guard let cgImage = self.cgImage else {
            throw HEICError.cgImageMissing
        }
        
        let options: NSDictionary = [
            kCGImageDestinationLossyCompressionQuality: compressionQuality
        ]
        
        CGImageDestinationAddImage(imageDestination, cgImage, options)
        guard CGImageDestinationFinalize(imageDestination) else {
            throw HEICError.couldNotFinalize
        }
        return data as Data
    }
}
```  
  
`CGImageDestinationCreateWithData`の引数に  
`AVType.heic`を指定することでHEICのデータを生成することができます。  
  
  
# 比較結果  
  
まず圧縮しない状態の結果です。  
  
![Simulator Screen Shot - iPhone 11 Pro Max - 2019-10-05 at 12.26.30.png](/image/539e34cb-1015-5564-3d68-6068bf811bae.png)  
  
  
圧縮をしない状態だと  
容量もHEICの方が大きくなってしまいました。  
  
いくつかの画像で試しましたが  
全て同じ結果でした。  
これは圧縮しないため  
内部で処理が行われていないのかなと思っています🤔  
  
もしご存知の方いらっしゃいましたら  
教えていただけるとうれしいです🙇🏻‍♂️  
  
少しだけ圧縮すると逆転します。  
半分とまではいきませんが容量は減りました。  
  
![Simulator Screen Shot - iPhone 11 Pro Max - 2019-10-05 at 12.27.42.png](/image/a5192fb5-8ddc-cfc0-d2da-a37712b92477.png)  
  
  
半分くらいに圧縮してみると  
さらに容量に差が出てきました。  
  
![Simulator Screen Shot - iPhone 11 Pro Max - 2019-10-05 at 12.26.44.png](/image/1c209ed4-9631-7918-54ad-2cb2058346cf.png)  
  
20%くらいに圧縮してみるとさらに容量に差が出てきました。  
  
![Simulator Screen Shot - iPhone 11 Pro Max - 2019-10-05 at 12.27.26.png](/image/a927f6ad-e249-f940-d95f-a14cd098a51a.png)  
  
  
# まとめ  
  
今回見てきたことから  
  
- 圧縮しないと容量は小さくならない  
- 圧縮すればするほどJPEGとの容量の差は大きくなる  
  
という結果になりました。  
  
最大で半分くらいに削減されるという話でしたが  
今回のケースではそこまでの削減は見られませんでした。  
もしかしたらもっと容量が大きいものでしたら  
より差が大きく出るのかもしれません。  
  
HEIFのメリットは容量の削減もそうですが  
画像編集や画像以外の情報も一緒に保存できるなど  
より幅広い用途で使用しやすくなっています。  
  
今後は間違いなく使用する頻度が高くなるであろう  
HEIFの特徴や使い方を学び  
これからの開発に備えていきたいですね😃  
  
何か間違いなどございましたら  
教えていただけるとうれしいです🙇🏻‍♂️  
  
# 使用したコード  
  
最後に今回使用したコードを下記に記載します。  
  
※ エラー処理などは省略しています  
  
  
### ImagePicker  
  
```swift

final class ImagePickerData: ObservableObject {
    var image: UIImage?
    var extensionName: String
    init(image: UIImage?, extensionName: String) {
        self.image = image
        self.extensionName = extensionName
    }
}

struct ImagePicker: UIViewControllerRepresentable {
    @Binding var isShown: Bool
    @Binding var data: ImagePickerData
    
    class Coordinator: NSObject, UINavigationControllerDelegate, UIImagePickerControllerDelegate {
        @Binding var isShown: Bool
        @Binding var data: ImagePickerData
        init(isShown: Binding<Bool>,
             data: Binding<ImagePickerData>) {
            _isShown = isShown
            _data = data
        }
        
        func imagePickerController(_ picker: UIImagePickerController, didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey : Any]) {
            let uiImage = info[.originalImage] as! UIImage
            let imageURL = info[.imageURL] as! URL
            data.image = uiImage
            data.extensionName = imageURL.pathExtension
            isShown = false
        }
        
        func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
            isShown = false
        }
    }
    
    func makeCoordinator() -> ImagePicker.Coordinator {
        return Coordinator(isShown: $isShown, data: $data)
    }
    
    func makeUIViewController(context: UIViewControllerRepresentableContext<ImagePicker>) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.delegate = context.coordinator
        return picker
    }
    
    func updateUIViewController(_ uiViewController: UIImagePickerController, context: UIViewControllerRepresentableContext<ImagePicker>) {
    }
}
```  
  
### ImageCompressor  
  
```swift

import AVFoundation
import UIKit

enum HEICError: Error {
    case heicNotSupported
    case cgImageMissing
    case couldNotFinalize
}

extension UIImage {
    func heicData(compressionQuality: CGFloat) throws -> Data {
        let data = NSMutableData()
        guard let imageDestination =
            CGImageDestinationCreateWithData(
                data, AVFileType.heic as CFString, 1, nil
            )
            else {
                throw HEICError.heicNotSupported
        }
        
        guard let cgImage = self.cgImage else {
            throw HEICError.cgImageMissing
        }
        
        let options: NSDictionary = [
            kCGImageDestinationLossyCompressionQuality: compressionQuality
        ]
        
        CGImageDestinationAddImage(imageDestination, cgImage, options)
        guard CGImageDestinationFinalize(imageDestination) else {
            throw HEICError.couldNotFinalize
        }
        return data as Data
    }
}

extension Data {
    var prettySize: String {
        let formatter = ByteCountFormatter()
        formatter.countStyle = .binary
        return formatter.string(fromByteCount: Int64(count))
    }
}


struct CompressedResult {
    let data: Data
    let size: String
    let elapsedTime: String?
}

final class ImageCompressor {
    private let numberFormatter: NumberFormatter = {
        let formatter = NumberFormatter()
        formatter.maximumSignificantDigits = 1
        formatter.maximumFractionDigits = 3
        return formatter
    }()
    private let compressionQueue = OperationQueue()
    
    func compressJPGImage(_ image: UIImage, quality: CGFloat, completion: @escaping (CompressedResult?) -> Void) {
        let startDate = Date()
        compressionQueue.addOperation {
            guard let data = image.jpegData(compressionQuality: quality) else {
                completion(nil)
                return
            }
            let time = self.elapsedTime(from: startDate)
            DispatchQueue.main.async {
                completion(CompressedResult(data: data, size: data.prettySize, elapsedTime: time))
            }
        }
    }
    
    func compressHEICImage(_ image: UIImage, quality: CGFloat, completion: @escaping (CompressedResult?) -> Void) {
        let startDate = Date()
        compressionQueue.addOperation {
            do {
                let data = try image.heicData(compressionQuality: quality)
                let time = self.elapsedTime(from: startDate)
                DispatchQueue.main.async {
                    completion(CompressedResult(data: data, size: data.prettySize, elapsedTime: time))
                }
            } catch {
                print("Error creating HEIC data: \(error.localizedDescription)")
                completion(nil)
            }
        }
    }
    
    private func elapsedTime(from startDate: Date) -> String? {
        let endDate = Date()
        let interval = endDate.timeIntervalSince(startDate)
        let intervalNumber = NSNumber(value: interval)
        return numberFormatter.string(from: intervalNumber)
    }
    
    func cancel() {
        compressionQueue.cancelAllOperations()
    }
}
```  
  
### ContentView  
  
```swift

import SwiftUI

final class ImageViewData: ObservableObject {
    var image: UIImage?
    var extensionName: String
    var elapsedTime: String
    var dataSize: String
    init(image: UIImage? = nil,
         extensionName: String = "-",
         elapsedTime: String = "-",
         dataSize: String = "-") {
        self.image = image
        self.extensionName = extensionName
        self.elapsedTime = elapsedTime
        self.dataSize = dataSize
    }
}

struct ContentView: View {
    @State var isShown = false
    @State var quality: CGFloat = 1
    @State var selectedData: ImagePickerData = ImagePickerData(image: nil, extensionName: "")
    @State var originalImageData = ImageViewData()
    @State var heicImageData = ImageViewData(extensionName: "HEIC")

    private var compressor = ImageCompressor()
    private var formattedQuality: String {
        let rounded = round(quality*10)/10
        return String(rounded.description.prefix(3))
    }
    
    var body: some View {
        VStack(alignment: HorizontalAlignment.center, spacing: 24) {
            Text("写真を選択してください")
            Button(action: {
                self.isShown = true
            }) { Text("画像を選択") }
            HStack(alignment: VerticalAlignment.center, spacing: 24) {
                VStack(alignment: HorizontalAlignment.center, spacing: 12) {
                    Text("Original")
                    ImageView(data: $originalImageData, quality: $quality)
                }
                VStack(alignment: HorizontalAlignment.center, spacing: 12) {
                    Text("HEIC")
                    ImageView(data: $heicImageData, quality: $quality)
                }
            }
            .padding()
            Slider(value: $quality)
                .padding()
            Text("Compression Quality: \(formattedQuality)")
            Button(action: { self.compress() }) { Text("圧縮する") }
        }
        .sheet(isPresented: $isShown, onDismiss: {
            self.isShown = false
            self.compress()
        }, content: {
            ImagePicker(isShown: self.$isShown,
                        data: self.$selectedData)
        })
    }
    
    private func compress() {
        guard let image = self.selectedData.image else {
            return
        }
        self.compressor.cancel()
        self.compressor.compressJPGImage(image, quality: self.quality) {
            if let result = $0 {
                self.originalImageData = ImageViewData(
                    image: UIImage(data: result.data),
                    extensionName: self.selectedData.extensionName,
                    elapsedTime: result.elapsedTime ?? "",
                    dataSize: result.size)
            } else {
                self.originalImageData = ImageViewData()
            }
        }
        self.compressor.compressHEICImage(image, quality: self.quality) {
                if let result = $0 {
                    self.heicImageData = ImageViewData(
                        image: UIImage(data: result.data),
                        extensionName: "HEIC",
                        elapsedTime: result.elapsedTime ?? "",
                        dataSize: result.size)
                } else {
                    self.heicImageData = ImageViewData()
                }
        }
    }
}
```  
  
# 参考資料  
  
https://www.raywenderlich.com/4726843-heic-image-compression-for-ios  
