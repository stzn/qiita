  
新しいクラスを作成するとき  
大抵は特定の目的やユースケースを思い浮かべると思います。  
  
あるデータを表現するための新しいモデルであったり  
構築している機能を果たすための複数のロジックを一つにまとめたりすることもあります。  
  
しかし  
時間が立つと  
作成した型やロジックに  
ある部分は全く同じである一方で一部だけ異なっているような  
すごい似ているバージョンを使用したくなることは  
結構多いのではないでしょうか？  
  
そんな時は  
既存の型やロジックを使い回したくありますが  
微妙に手順が違っていたり  
結果が異なる型になっているなど  
もどかしい経験をしたことがある人はたくさんいると思います。  
  
今回はJohn Sundellさんのブログ記事を参考に  
ある特定の型を一部変更可能型に変化させていく過程を通して  
より汎用的で再利用可能な型を構築する方法を見ていきたいと思います。  
  
  
**Configurable types in Swift**  
https://www.swiftbysundell.com/posts/configurable-types-in-swift  
  
  
# 最初の具体的なユースケース  
  
まずは特定のユースケースを解決する例を見ていきます。  
  
今回はノートを取るアプリを想定しており  
フォルダー内のファイルからノートのデータをインポートする機能を考えていきます。  
  
最初の機能として  
- プレーンなテキストファイル  
- Markdownを使ってフォーマットされたファイル  
のみをサポートできるようにします。  
  
```swift

struct NoteImporter {
    func importNotes(from folder: Folder) throws -> [Note] {
        return try folder.files.compactMap { file in
            // フォルダー内のファイルの拡張子がtxt, mdの場合のみ処理をする
            switch file.extension {
            case "txt":
                return try importPlainTextNote(from: file)
            case "md", "markdown":
                return try importMarkdownNote(from: file)
            default:
                return nil
            }
        }
    }
}
```  
  
  
# 機能を拡張する  
  
特定のユースケースに対応する分には上記のコードで十分です。  
  
しかし、さらにサポートする拡張子を増やしていくとなると  
コアのロジックに毎回手を加えていかなければならなくなります。  
  
下記の様な場合は想定してみます。  
  
### 別のテキストフォーマットもサポートできるようにする場合  
  
理想としては  
NoteImporter自体は実際のインポートを取りまとめる役に徹して  
特定のフォーマットに関しては知らない方が良いでしょう。  
そうしないとswitch文はどんどん大きくなっていって  
いずれコントロールしきれなくなってくることが目に見えてわかります。  
  
### テキストファイル以外もインポートできるようにする場合  
  
音声ファイルや画像ファイルもインポートできるようにしたいとなった場合はどうでしょうか？  
  
現状ですと  
種類が増えていくたびに新しいクラスを作成していく必要が出てきます。  
  
```swift

struct AudioImporter {
    func importAudio(from folder: Folder) throws -> [Audio] {
        ...
    }
}

struct PhotoImporter {
    func importPhotos(from folder: Folder) throws -> [Photo] {
        ...
    }
}
```  
  
これらは新しいクラスですが  
やろうとしていることはNoteImporterと似ており  
同じようなコードを複数回実装することになります。  
  
個々の処理に分けておくことで  
関心を分離して  
個々のケースに最適化して対応できることは良いことですが  
  
共有できる抽象層や共通のAPIがないと  
膨大な量のコードを再利用できる可能性を失ってしまいます。  
そして利用する側でも共有できるAPIを書くことが難しくなります。  
  
そこで共通の処理を定義してコードの重複を試みていきます。  
  
# Protocolを利用する  
  
まずProtocolを使うことを考えてみます。  
全てのFileImporterは下記のProtocolに適合させます。  
  
```swift

protocol FileImporting {
    associatedtype Result
    func importFiles(from folder: Folder) throws -> [Result]
}
```  
  
```swift

extension AudioImporter: FileImporting { ... }
extension PhotoImporter: FileImporting { ... }
extension PlainTextImporter: FileImporting { ... }
extension MarkdownImporter: FileImporting { ... }
```  
  
こうすることで  
共有可能で一貫したAPIを提供することができる一方で  
個々のファイルフォーマットに合わせた処理も可能になります。  
  
しかし  
これでもコードの重複が発生します。  
  
フォルダーの中をイテレートして処理をする  
  
というコードは全てのFileImporterで全く同じですが  
全てのクラスで書かなければなりません。  
  
# 変更可能型(configurable types)を利用する  
  
複数の実装を持つ抽象層を作成する代わりに  
既存のユースケースとこれから導入されるユースケースのどちらにも対応できる  
変更可能な一つのFileImporter型を生成してみるのはどうでしょうか？  
  
今回のケースでは個々のFileImporterが違うのは  
ファイルの処理の仕方部分のみです。  
そこだけ変更可能にしてみます。  
  
```swift

struct FileImporter<Result> {
    typealias FileType = String

    var handlers: [FileType : (File) throws -> Result]
}
```  
※  
File型はFileの情報を持った型だと思ってください。   
typealiasは可読性向上のために使われています。  
  
個々のファイルタイプに対してFile型からジェネリックなResultを返す  
handlerの配列をプロパティとして保持します。  
  
そしてimportFilesという一つメソッドを定義します。  
  
```swift

extension FileImporter {
    func importFiles(from folder: Folder) throws -> [Result] {
        return try folder.files.compactMap { file in
            guard let handler = handlers[file.extension] else {
                return nil
            }
            return try handler(file)
        }
    }
}
```  
  
やっていることは  
- フォルダー内をイテレート  
- ファイルの拡張子に対応したhandlerをhandlersから取得  
- handlerが存在した場合に処理を実行  
だけです。  
  
## より扱いやすくするためのFactoryメソッド  
  
上記で共通の処理を実現することができましたが  
これだと呼び出し側が自由に実装を入れ替えることができ  
処理に一貫性がなくなってしまう可能性があります。  
  
これは作成側の我々にとっては  
意図しない結果を生み出してしまうかもしれません。  
  
そこで実装をコントロールするために  
FileImporterインスタンスを作成する  
staticなFactoryメソッドを作成します。  
  
```swift

extension FileImporter where Result == Note {
    static func notes() -> FileImporter {
        return FileImporter(handlers: [
            "txt": importPlainTextNote,
            "text": importPlainTextNote,
            "md": importMarkdownNote,
            "markdown": importMarkdownNote
        ])
    }

    private static func importPlainTextNote(from file: File) throws -> Note {
        ...
    }

    private static func importMarkdownNote(from file: File) throws -> Note {
        ...
    }
}
```  
  
こうすることでコアのロジックに影響を与えずに  
新しいファイルフォーマットを追加することもできますし  
Noteを生成するための新しいprivateなユーティリティメソッドを追加することもできます。  
  
これまでのところ  
より責務を分離させつつコードの構造化をきちんと保つことができています。  
  
インスタンスの生成も下記のようにFactoryメソッドを呼ぶだけです。  
  
```swift
let notesImporter = FileImporter.notes()
let audioImporter = FileImporter.audio()
let photoImporter = FileImporter.photos()
```  
  
さらに  
staticメソッドを活用することで  
パラメータを使うことで  
同じ型でも設定の変更をすることができるようになります。  
  
例えば  
AudioでOGGフォーマットも扱いたい場合が出てきたとします。  
その場合、audioメソッドにパラメータを追加します。  
  
```swift

extension FileImporter where Result == Audio {
    static func audio(
        includeOggFiles: Bool = FeatureFlags.Audio.enableOggImports
    ) -> FileImporter {
        var handlers = [
            "mp3": importMp3Audio,
            "aac": importAacAudio
        ]

        if includeOggFiles {
            handlers["ogg"] = importOggAudio
        }

        return FileImporter(handlers: handlers)
    }
}
```  
  
これはResultがAudioの場合のみに影響を与え  
他のケースへの影響を心配する必要がありません。  
  
  
# 異なるファイルの種類もまとめて扱えるようにする  
  
この時点でも様々なファイルフォーマットへ対応可能になり  
膨大な数のコードの重複を減らすこともできました。  
  
しかし  
ファイルの拡張子によって扱える処理が分かれてしまっています。  
  
```swift

let notesImporter = FileImporter.notes() // テキストファイルのみ
let audioImporter = FileImporter.audio() // 音声ファイルのみ
let photoImporter = FileImporter.photos() // 画像ファイルのみ
```  
  
これは両方を扱いたいケースもあるはずなので  
理想的ではありません。  
  
そこで全てのフォーマットを一括で扱うための  
包括する型を作ってみるのはどうでしょうか？  
  
```swift

enum FileImportResult {
    case note(Note)
    case audio(Audio)
    case photo(Photo)
}
```  
  
しかし  
今の形ですとこの型を扱うことはできません。  
Dictionary中でどの拡張子にどのhandlerを使うかを指定できるだけです。  
  
そこで下記のように変更します。  
  
```swift

struct FileImporter<Result> {
    typealias Handler = (File) throws -> Result?
    var handler: Handler
}
```  
  
DictionaryではなくFileを受け取ってResultを返す  
Handler自体を初期化時に受け取るようにします。  
  
  
また  
既存のDictionaryを受け取る場合にも  
機能し続けるように拡張します。  
  
```swift

extension FileImporter {
    typealias FileType = String

    init(handlers: [FileType : Handler]) {
        handler = { file in
            try handlers[file.extension]?(file) ?? nil
        }
    }
}
```  
  
  
こうすることでimportFilesメソッドは  
よりシンプルになります。  
  
```swift

extension FileImporter {
    func importFiles(from folder: Folder) throws -> [Result] {
        return try folder.files.compactMap(handler)
    }
}
```  
  
こうすることで  
具体的なオプションの代わりに  
「無限」に変更可能にすることができました。  
  
個々のファイルに対してどのような処理を行うかを  
完全にコントロールすることができます。  
  
この新しい実装を使って  
これまでに定義してきたファイルフォーマット全てに対応した  
FileImporterインスタンスを生成することが可能になります。  
  
```swift

extension FileImporter where Result == FileImportResult {
    static func combined() -> FileImporter {
        let importers = (
            notes: FileImporter<Note>.notes(),
            audio: FileImporter<Audio>.audio(),
            photos: FileImporter<Photo>.photos()
        )

        return FileImporter { file in
            try importers.notes.handler(file).map(Result.note) ??
                importers.audio.handler(file).map(Result.audio) ??
                importers.photos.handler(file).map(Result.photo)
        }
    }
}
```  
  
この最後のステップで  
自由にファイルを取り扱う方法を提供することができるようになったことに加え  
これまでに提供していたデフォルトの実装もそのまま提供可能にすることができます。  
  
# まとめ  
  
型を様々な方法で変更可能にすることで  
新しいユースケースなどへも既存のコードで対応することができます。  
新たな実装を加えないので管理の複雑さや難しさが増すこともありません。  
  
あるユースケースに対する具体的な型を使ったハードコーディングな実装は  
いつも一番シンプルな解決法だと思います。  
  
一方で  
ある共通の土台の上に複数の機能を構築したい時などは  
変更可能型を提供して柔軟性を追加することは努力に値するものでしょう。  
  
  
