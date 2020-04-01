- 2019/8/7 追記 iOS12以前でもUICollectionViewCompositionalLayoutが使用できるようにするライブラリについて追記しました  
- 2019/8/21 追記 Xcode11 Beta6でNSDiffableDataSourceSnapshotのメソッドがmutatingになったので変数をletからvarに変更しました。  
  
WWDC2019ではSwiftUIの登場に注目が浴びていますが  
`UICollectionView`(`UITableView`なども含まれる)にも  
すごく大きな変化がありました。  
  
iOS13以降ではこのスタイルが主流になるのではないかとも感じており  
この機会にまずは何が変わったのかについて  
**Layout**と**Data Source**という2つの観点から調べてみました。  
  
Advances in Collection View Layout  
https://developer.apple.com/videos/play/wwdc2019/215  
  
Advances in UI Data Sources  
https://developer.apple.com/videos/play/wwdc2019/220/  
  
サンプル  
https://developer.apple.com/documentation/uikit/views_and_controls/collection_views/using_collection_view_compositional_layouts_and_diffable_data_sources  
  
※  
今回はCollectionViewの話を中心にしていますが  
TableViewも同じように機能が追加されています。  
  
# 背景としての時代の変化  
  
これまでのアプリは  
今のアプリに比べて表示できる内容に限りがあり  
デザインなどももう少しシンプルなものが多くありました。  
  
しかし  
ハードウェアの機能やネットワーク通信速度の向上などに合わせて  
アプリの構造もどんどん複雑になってきています。  
  
そんな中でCollectionViewで表示する内容もより複雑になり  
これまでのAPIでは実装が難しい場面も増えてきました。  
  
# UICollectionViewFlowLayout  
  
iOS6で導入された`UICollectionViewFlowLayout`は  
非常にシンプルな`UICollectionViewLayout`のサブクラスです。  
  
https://developer.apple.com/documentation/uikit/uicollectionviewflowlayout  
  
画一的なレイアウトを構築することは得意ですが  
今日のアプリではより複雑なレイアウトを実現するのは  
かなり大変です。  
  
例としてApp Storeが出ていました。  
  
<img width="408" alt="スクリーンショット 2019-06-15 17.38.29.png" src="/image/26322252-5f64-b99f-6a29-f85ff078ab78.png">  
  
※ iOS6の頃は下記のような感じでした。  
![image.png](/image/304913ce-6df1-021c-22bc-815ebf5b8ecf.png)  
  
  
そういう場合には`UICollectionViewLayout`を継承して  
カスタムレイアウトを構築していきます。  
  
ここに関しては  
WWDC2018のセッションでも紹介されています。  
https://developer.apple.com/videos/play/wwdc2018/225/  
  
過去の記事で内容をまとめさせていただきました。  
https://github.com/stzn/qiita/blob/master/【Swift】CollectionViewを再理解する.md  
  
  
しかし  
  
- 大量のボイラープレートが生まれる  
- パフォーマンスに配慮する必要がある  
- フッターやヘッダーなどの付加的なViewの実装が難しい  
- サイズの計算が難しい  
  
など多くの問題に立ち向かう必要があります。  
  
<img width="1494" alt="スクリーンショット 2019-06-15 17.44.49.png" src="/image/51842825-b984-5b35-7cd2-ab2ef148e561.png">  
  
# Compositional Layoutsの登場  
  
そこで今回登場したのが  
**Compositional Layouts**です。  
  
特徴としては  
  
- 小さいLayoutをGroup化して構成していく  
- LayoutのそれぞれのGroupでひとまとまりの要素を構成する  
- サブクラスよりも小さいLayoutを組み合わせて使用する  
  
などがあります。  
  
# 4つのメインの構成要素  
  
下記の4つの要素から主に構成されます。  
  
- Item  
- Group  
- Section  
- Layout  
  
Item, Group, Section, Layoutは階層構造を構築します。  
  
<img width="1388" alt="スクリーンショット 2019-06-15 18.13.34.png" src="/image/3b7d7444-5172-10a4-24dc-9dc024bfb9f1.png">  
  
これらの大きさを決定するのがSizeです。  
  
## NSCollectionLayoutSize  
  
全てのレイアウトを構成する要素は明示的なサイズを持っています。  
それを表すのがNSCollectionLayoutSizeになります。  
  
NSCollectionLayoutSizeは  
widthとheight2つのDimensionから構成されます。  
  
https://developer.apple.com/documentation/uikit/nscollectionlayoutsize  
  
widthDimensionとheightDimensionは別々に設定することができ  
4つのメソッドが用意されています。  
  
```swift

class NSCollectionLayoutDimension {
    class func fractionalWidth(_ fractionalWidth: CGFloat) -> Self
    class func fractionalHeight(_ fractionalHeight: CGFloat) -> Self
    class func absolute(_ absoluteDimension: CGFloat) -> Self
    class func estimated(_ estimatedDimension: CGFloat) -> Self
}
```  
  
### fractionalWidth, fractionalHeight  
  
これはサイズを指定したいレイアウトの  
コンテナのサイズに対する割合になります。  
  
`fractionalWidth`  
https://developer.apple.com/documentation/uikit/nscollectionlayoutdimension/3199059-fractionalwidth  
  
`fractionalHeight`  
https://developer.apple.com/documentation/uikit/nscollectionlayoutdimension/3199058-fractionalheight  
  
<img width="1501" alt="スクリーンショット 2019-06-15 18.11.12.png" src="/image/defd6292-0d5d-d479-07d1-39812a1b85e3.png">  
  
上記の例ではコンテナのwidthの半分の大きさを表しています。  
高さも同様に設定が可能です。  
  
<img width="1474" alt="スクリーンショット 2019-06-15 18.16.42.png" src="/image/af1b5717-54c5-6c92-83c3-6f885cf2fa8a.png">  
  
上記の例ではwidthとheightの両方にコンテナのwidthの0.25をしています。  
そのためwidthとheightの大きさは同じになりaspectRatioは1です。  
  
### absolute, estimated  
  
上記では割合で指定をしていましたが  
この2つのメソッドはポイントで値を指定します。  
  
`absolute`  
https://developer.apple.com/documentation/uikit/nscollectionlayoutdimension/3199055-absolute  
  
`estimated`  
https://developer.apple.com/documentation/uikit/nscollectionlayoutdimension/3199057-estimated  
  
<img width="1272" alt="スクリーンショット 2019-06-15 18.19.42.png" src="/image/16b10239-cb67-b28e-c72a-efd366c90f26.png">  
  
absoluteは名前の通り指定した値にサイズを固定します。  
  
<img width="1369" alt="スクリーンショット 2019-06-15 18.21.03.png" src="/image/1cc0e2b4-54f2-3cae-2a7b-43bc3cfcd3bc.png">  
  
estimateは値の指定はするものの  
優先される制約がある場合は値は変化します。  
  
  
## NSCollectionLayoutItem  
  
レイアウトを構成するための一番小さい部品です。  
初期化時にサイズを指定する必要があります。  
  
ItemはCellやヘッダー、フッターになります。  
  
<img width="1496" alt="スクリーンショット 2019-06-16 4.10.32.png" src="/image/3fd48a16-8132-f89d-bff8-2d7611b43146.png">  
  
`NSCollectionLayoutItem`  
https://developer.apple.com/documentation/uikit/nscollectionlayoutitem  
  
`NSCollectionLayoutSupplementaryItem`  
https://developer.apple.com/documentation/uikit/nscollectionlayoutsupplementaryitem  
  
## NSCollectionLayoutGroup  
  
Itemをまとめてレイアウトの基本単位を構成します。  
一つのカラムや一行のアイテム郡を表します。  
  
さらに  
クロージャを使ったイニシャライザでは  
アイテムごとにレイアウトを構築することもでき  
これまでよりもより細かい単位で  
レイアウトを構築することができるようになりました。  
  
<img width="1494" alt="スクリーンショット 2019-06-16 4.10.42.png" src="/image/0cb08b64-fff5-c6b2-ab83-5f4f48af3b66.png">  
  
`NSCollectionLayoutGroup`  
https://developer.apple.com/documentation/uikit/nscollectionlayoutgroup  
  
  
## NSCollectionLayoutSection  
  
いわゆるセクションを表すクラスで  
Groupを初期化時に設定しそこにさらにInsetsなどを追加できます。  
  
<img width="1414" alt="スクリーンショット 2019-06-16 4.12.41.png" src="/image/54ec3513-39e7-85e5-aefe-6c5155cf4cdb.png">  
  
`NSCollectionLayoutSection`  
https://developer.apple.com/documentation/uikit/nscollectionlayoutsection  
  
  
## UICollectionViewCompositionalLayout, NSCollectionViewCompositionalLayout  
  
最終的にレンダリングされる単位になります。  
セクションクラスを引数に渡して初期化することもできます。  
さらにグループと同様に  
クロージャ内でセクション単位でレイアウトの設定ができます。  
  
<img width="1473" alt="スクリーンショット 2019-06-16 4.22.11.png" src="/image/4ee68e6e-67fb-ed74-882c-974645d93863.png">  
  
`UICollectionViewCompositionalLayout`  
https://developer.apple.com/documentation/uikit/uicollectionviewcompositionallayout  
  
`NSCollectionViewCompositionalLayout`  
https://developer.apple.com/documentation/appkit/nscollectionviewcompositionallayout  
  
  
# サンプルコード  
  
ここからはサンプルのコードを見ながら  
Compositional Layoutsの詳細を見ていきたいと思います。  
  
  
## ListViewController  
  
シンプルなリストのレイアウトです。  
  
![Simulator Screen Shot - iPhone Xʀ - 2019-06-16 at 04.31.22.png](/image/1ab16f35-8258-78ce-b152-78a7a930b241.png)  
  
実際の設定方法は下記のようになります。  
  
```swift

private func createLayout() -> UICollectionViewLayout {
    let itemSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                          heightDimension: .fractionalHeight(1.0))
    let item = NSCollectionLayoutItem(layoutSize: itemSize)
    
    let groupSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                           heightDimension: .absolute(44))
    let group = NSCollectionLayoutGroup.horizontal(layoutSize: groupSize,
                                                   subitems: [item])
    
    let section = NSCollectionLayoutSection(group: group)
    
    let layout = UICollectionViewCompositionalLayout(section: section)
    return layout
}
```  
  
  
## GridViewController  
  
次にGridのレイアウトが登場します。  
  
![Simulator Screen Shot - iPhone Xʀ - 2019-06-16 at 04.30.44.png](/image/641afa00-c6e0-aa51-c366-69c4b2a9208a.png)  
  
```swift

extension GridViewController {
    private func createLayout() -> UICollectionViewLayout {
        let itemSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(0.2),
                                             heightDimension: .fractionalHeight(1.0))
        let item = NSCollectionLayoutItem(layoutSize: itemSize)
        
        // add for each item allocate same box
        item.contentInsets = NSDirectionalEdgeInsets(top: 5, leading: 5, bottom: 5, trailing: 5)

        let groupSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                              heightDimension: .fractionalWidth(0.2))
        let group = NSCollectionLayoutGroup.horizontal(layoutSize: groupSize,
                                                         subitems: [item])

        let section = NSCollectionLayoutSection(group: group)

        let layout = UICollectionViewCompositionalLayout(section: section)
        return layout
    }
}
```  
  
ポイントとしては  
  
```swift

let itemSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(0.2),
                                      heightDimension: .fractionalHeight(1.0))
```  
  
で個々のItemのWidthをGroupのWidthの1/5のサイズに設定し  
  
  
```swift

let groupSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                       heightDimension: .fractionalWidth(0.2))
```  
  
でGroupのheightをセクションのWidthの1/5のサイズに設定することで  
正方形のGridを構成しています。  
  
また  
  
```swift

item.contentInsets = NSDirectionalEdgeInsets(top: 5, leading: 5, bottom: 5, trailing: 5)
```  
  
を設定することで個々の正方形の外側にスペースを作っています。  
  
## TwoColumnViewController  
  
2つのセルで1行を構成します。  
  
![Simulator Screen Shot - iPhone Xʀ - 2019-06-16 at 04.40.44.png](/image/8ef79f35-6184-5334-ac5c-720d70180a16.png)  
  
```swift

extension TwoColumnViewController {
    func createLayout() -> UICollectionViewLayout {
        let itemSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                              heightDimension: .fractionalHeight(1.0))
        let item = NSCollectionLayoutItem(layoutSize: itemSize)

        let groupSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                               heightDimension: .absolute(44))
        let group = NSCollectionLayoutGroup.horizontal(layoutSize: groupSize, subitem: item, count: 2)
        let spacing = CGFloat(10)
        group.interItemSpacing = .fixed(spacing)

        let section = NSCollectionLayoutSection(group: group)
        section.interGroupSpacing = spacing
        section.contentInsets = NSDirectionalEdgeInsets(top: 0, leading: 10, bottom: 0, trailing: 10)

        let layout = UICollectionViewCompositionalLayout(section: section)
        return layout
    }
}
```  
  
ポイントとしては  
  
```swift

let itemSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                      heightDimension: .fractionalHeight(1.0))

let group = NSCollectionLayoutGroup.horizontal(layoutSize: groupSize, subitem: item, count: 2)
```  
  
個々のItemはGroupの100％に設定しているのに対して  
で`count`を2に設定することで  
1行に表示されるカラムの数を2に制限しています。  
  
また  
  
```swift

let spacing = CGFloat(10)
group.interItemSpacing = .fixed(spacing)
```  
で個々のカラムの外側にスペースを設定し  
  
```swift

section.interGroupSpacing = spacing
section.contentInsets = NSDirectionalEdgeInsets(top: 0, leading: 10, bottom: 0, trailing: 10)

```  
  
で画面の両端にスペースを作っています。  
  
ここで使用されている`NSDirectionalEdgeInsets`は  
言語の方向に応じて(左から右か右から左か)自動にレイアウトを調整してくれます。  
  
`NSDirectionalEdgeInsets`  
https://developer.apple.com/documentation/uikit/nsdirectionaledgeinsets  
  
## DistinctSectionsViewController  
  
セクション単位でレイアウトを変えています。  
  
  
![Simulator Screen Shot - iPhone Xʀ - 2019-06-16 at 04.55.02.png](/image/affd7f48-1be6-b186-c061-8089445a1af9.png)  
  
  
```swift

extension DistinctSectionsViewController {
    func createLayout() -> UICollectionViewLayout {
        let layout = UICollectionViewCompositionalLayout { (sectionIndex: Int,
            layoutEnvironment: NSCollectionLayoutEnvironment) -> NSCollectionLayoutSection? in

            guard let sectionLayoutKind = SectionLayoutKind(rawValue: sectionIndex) else { return nil }
            let columns = sectionLayoutKind.columnCount

            // The `group` auto-calculates the actual item width to make
            // the requested number of `columns` fit, so this `widthDimension` will be ignored.
            let itemSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                                  heightDimension: .fractionalHeight(1.0))
            let item = NSCollectionLayoutItem(layoutSize: itemSize)
            item.contentInsets = NSDirectionalEdgeInsets(top: 2, leading: 2, bottom: 2, trailing: 2)

            let groupHeight = columns == 1 ?
                NSCollectionLayoutDimension.absolute(44) :
                NSCollectionLayoutDimension.fractionalWidth(0.2)
            let groupSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                                   heightDimension: groupHeight)
            let group = NSCollectionLayoutGroup.horizontal(layoutSize: groupSize, subitem: item, count: columns)

            let section = NSCollectionLayoutSection(group: group)
            section.contentInsets = NSDirectionalEdgeInsets(top: 20, leading: 20, bottom: 20, trailing: 20)
            return section
        }
        return layout
    }
}
```  
  
ポイントは  
クロージャの中でレイアウトの構成を設定しています。  
  
セクションのごとにカラム数を変化させています。  
  
```swift

class DistinctSectionsViewController: UIViewController {

    enum SectionLayoutKind: Int, CaseIterable {
        case list, grid5, grid3
        var columnCount: Int {
            switch self {
            case .grid3:
                return 3

            case .grid5:
                return 5

            case .list:
                return 1
            }
        }
    }
}
```  
  
Groupは`count`を設定すると  
Groupに指定したサイズは無視され  
`count`　に合わせて自動でサイズを調整します。  
  
```swift

let groupHeight = columns == 1 ?
    NSCollectionLayoutDimension.absolute(44) :
    NSCollectionLayoutDimension.fractionalWidth(0.2)

let groupSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                       heightDimension: groupHeight)
let group = NSCollectionLayoutGroup.horizontal(layoutSize: groupSize, subitem: item, count: columns)

```  
  
## AdaptiveSectionsViewController  
  
上記の例と似ていますが  
画面を回転した際のレイアウトを変化させています。  
  
![Simulator Screen Shot - iPhone Xʀ - 2019-06-16 at 05.03.14.png](/image/599fb86f-f0f4-54e7-fbc4-91f220b3eb66.png)  
  
```swift

extension AdaptiveSectionsViewController {
    func createLayout() -> UICollectionViewLayout {
        let layout = UICollectionViewCompositionalLayout {
            (sectionIndex: Int, layoutEnvironment: NSCollectionLayoutEnvironment) -> NSCollectionLayoutSection? in
            guard let layoutKind = SectionLayoutKind(rawValue: sectionIndex) else { return nil }

            let columns = layoutKind.columnCount(for: layoutEnvironment.container.effectiveContentSize.width)

            let itemSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(0.2),
                                                 heightDimension: .fractionalHeight(1.0))
            let item = NSCollectionLayoutItem(layoutSize: itemSize)
            item.contentInsets = NSDirectionalEdgeInsets(top: 2, leading: 2, bottom: 2, trailing: 2)

            let groupHeight = layoutKind == .list ?
                NSCollectionLayoutDimension.absolute(44) : NSCollectionLayoutDimension.fractionalWidth(0.2)
            let groupSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                                   heightDimension: groupHeight)
            let group = NSCollectionLayoutGroup.horizontal(layoutSize: groupSize, subitem: item, count: columns)

            let section = NSCollectionLayoutSection(group: group)
            section.contentInsets = NSDirectionalEdgeInsets(top: 20, leading: 20, bottom: 20, trailing: 20)
            return section
        }
        return layout
    }
}
```  
  
これを実現するために  
横幅に応じで1行に表示するカラム数を変えています。  
  
```swift

class AdaptiveSectionsViewController: UIViewController {

    enum SectionLayoutKind: Int, CaseIterable {
        case list, grid5, grid3
        func columnCount(for width: CGFloat) -> Int {
            let wideMode = width > 800
            switch self {
            case .grid3:
                return wideMode ? 6 : 3

            case .grid5:
                return wideMode ? 10 : 5

            case .list:
                return wideMode ? 2 : 1
            }
        }
    }
}
```  
  
  
# バッジ、ヘッダー、フッタ-などの付加的な要素の設定  
  
次にメインのカラム以外の付加的な要素の設定について見ていきます。  
  
  
## ItemBadgeSupplementaryViewController  
  
![Simulator Screen Shot - iPhone Xʀ - 2019-06-16 at 05.20.03.png](/image/c6c7e253-f412-b957-8615-76fc1204ed72.png)  
  
`NSCollectionLayoutSupplementaryItem`を使用して  
Anchorを変えることで位置をシンプルに設定することができます。  
  
`NSCollectionLayoutSupplementaryItem`  
https://developer.apple.com/documentation/uikit/nscollectionlayoutsupplementaryitem  
  
`NSCollectionLayoutAnchor`  
https://developer.apple.com/documentation/uikit/nscollectionlayoutanchor  
  
<img width="1393" alt="スクリーンショット 2019-06-16 5.14.01.png" src="/image/3a59d138-42c3-940b-4fe9-57b0876c68fe.png">  
  
  
```swift

extension ItemBadgeSupplementaryViewController {
    func createLayout() -> UICollectionViewLayout {
        let badgeAnchor = NSCollectionLayoutAnchor(edges: [.top, .trailing], fractionalOffset: CGPoint(x: 0.3, y: -0.3))
        let badgeSize = NSCollectionLayoutSize(widthDimension: .absolute(20),
                                              heightDimension: .absolute(20))
        let badge = NSCollectionLayoutSupplementaryItem(
            layoutSize: badgeSize,
            elementKind: ItemBadgeSupplementaryViewController.badgeElementKind,
            containerAnchor: badgeAnchor)

        let itemSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(0.25),
                                              heightDimension: .fractionalHeight(1.0))
        let item = NSCollectionLayoutItem(layoutSize: itemSize, supplementaryItems: [badge])
        item.contentInsets = NSDirectionalEdgeInsets(top: 5, leading: 5, bottom: 5, trailing: 5)

        let groupSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                               heightDimension: .fractionalWidth(0.2))
        let group = NSCollectionLayoutGroup.horizontal(layoutSize: groupSize, subitems: [item])

        let section = NSCollectionLayoutSection(group: group)
        section.contentInsets = NSDirectionalEdgeInsets(top: 20, leading: 20, bottom: 20, trailing: 20)

        let layout = UICollectionViewCompositionalLayout(section: section)
        return layout
    }
}
```  
  
下記のようにバッジを設定します。  
  
  
```swift

let badgeAnchor = NSCollectionLayoutAnchor(edges: [.top, .trailing], fractionalOffset: CGPoint(x: 0.3, y: -0.3))
let badgeSize = NSCollectionLayoutSize(widthDimension: .absolute(20),
                                       heightDimension: .absolute(20))
let badge = NSCollectionLayoutSupplementaryItem(
    layoutSize: badgeSize,
    elementKind: ItemBadgeSupplementaryViewController.badgeElementKind,
    containerAnchor: badgeAnchor)


let item = NSCollectionLayoutItem(layoutSize: itemSize, supplementaryItems: [badge])
```  
  
`NSCollectionLayoutSupplementaryItem`の`elementKind`には  
特定の文字列を指定します。  
  
```swift

static let badgeElementKind = "badge-element-kind"
```  
  
  
## SectionHeadersFootersViewController  
  
ヘッダーとフッターを設定するには  
`NSCollectionLayoutBoundarySupplementaryItem`を使用します。  
  
`NSCollectionLayoutBoundarySupplementaryItem`  
https://developer.apple.com/documentation/uikit/nscollectionlayoutboundarysupplementaryitem  
  
![Simulator Screen Shot - iPhone Xʀ - 2019-06-16 at 05.20.59.png](/image/7b19c9ac-1d85-7f96-ff1a-676cde7a3e5f.png)  
  
  
```swift

extension SectionHeadersFootersViewController {
    func createLayout() -> UICollectionViewLayout {
        let itemSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                             heightDimension: .fractionalHeight(1.0))
        let item = NSCollectionLayoutItem(layoutSize: itemSize)

        let groupSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                              heightDimension: .absolute(44))
        let group = NSCollectionLayoutGroup.horizontal(layoutSize: groupSize, subitems: [item])

        let section = NSCollectionLayoutSection(group: group)
        section.interGroupSpacing = 5
        section.contentInsets = NSDirectionalEdgeInsets(top: 0, leading: 10, bottom: 0, trailing: 10)

        let headerFooterSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                                     heightDimension: .estimated(44))
        let sectionHeader = NSCollectionLayoutBoundarySupplementaryItem(
            layoutSize: headerFooterSize,
            elementKind: SectionHeadersFootersViewController.sectionHeaderElementKind, alignment: .top)
        let sectionFooter = NSCollectionLayoutBoundarySupplementaryItem(
            layoutSize: headerFooterSize,
            elementKind: SectionHeadersFootersViewController.sectionFooterElementKind, alignment: .bottom)
        section.boundarySupplementaryItems = [sectionHeader, sectionFooter]

        let layout = UICollectionViewCompositionalLayout(section: section)
        return layout
    }
}

```  
  
SectionのboundarySupplementaryItemsに  
ヘッダーとフッターを設定します。  
  
```swift

let sectionHeader = NSCollectionLayoutBoundarySupplementaryItem(
    layoutSize: headerFooterSize,
    elementKind: SectionHeadersFootersViewController.sectionHeaderElementKind, alignment: .top)
let sectionFooter = NSCollectionLayoutBoundarySupplementaryItem(
    layoutSize: headerFooterSize,
    elementKind: SectionHeadersFootersViewController.sectionFooterElementKind, alignment: .bottom)
section.boundarySupplementaryItems = [sectionHeader, sectionFooter]
```  
  
こちらも特定の文字列を指定します。  
  
```swift

static let sectionHeaderElementKind = "section-header-element-kind"
static let sectionFooterElementKind = "section-footer-element-kind"
```  
  
## SectionDecorationViewController  
  
セクション単位に背景の設定などのカスタマイズをすることができます。  
  
`NSCollectionLayoutDecorationItem`を使用します。  
  
`NSCollectionLayoutDecorationItem`  
https://developer.apple.com/documentation/uikit/nscollectionlayoutdecorationitem  
  
![Simulator Screen Shot - iPhone Xʀ - 2019-06-16 at 05.30.45.png](/image/99bdad62-f7d9-9eaa-44e3-18b508140099.png)  
  
```swift

extension SectionDecorationViewController {
    func createLayout() -> UICollectionViewLayout {
        let itemSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                              heightDimension: .fractionalHeight(1.0))
        let item = NSCollectionLayoutItem(layoutSize: itemSize)

        let groupSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                               heightDimension: .absolute(44))
        let group = NSCollectionLayoutGroup.horizontal(layoutSize: groupSize, subitems: [item])

        let section = NSCollectionLayoutSection(group: group)
        section.interGroupSpacing = 5
        section.contentInsets = NSDirectionalEdgeInsets(top: 10, leading: 10, bottom: 10, trailing: 10)

        let sectionBackgroundDecoration = NSCollectionLayoutDecorationItem.background(
            elementKind: SectionDecorationViewController.sectionBackgroundDecorationElementKind)
        sectionBackgroundDecoration.contentInsets = NSDirectionalEdgeInsets(top: 5, leading: 5, bottom: 5, trailing: 5)
        section.decorationItems = [sectionBackgroundDecoration]

        let layout = UICollectionViewCompositionalLayout(section: section)
        layout.register(
            SectionBackgroundDecorationView.self,
            forDecorationViewOfKind: SectionDecorationViewController.sectionBackgroundDecorationElementKind)
        return layout
    }
}
```  
  
`NSCollectionLayoutDecorationItem`を  
Layoutに`register`することで設定が可能です。  
  
  
```swift

let sectionBackgroundDecoration = NSCollectionLayoutDecorationItem.background(
    elementKind: SectionDecorationViewController.sectionBackgroundDecorationElementKind)
sectionBackgroundDecoration.contentInsets = NSDirectionalEdgeInsets(top: 5, leading: 5, bottom: 5, trailing: 5)
section.decorationItems = [sectionBackgroundDecoration]

let layout = UICollectionViewCompositionalLayout(section: section)
layout.register(
    SectionBackgroundDecorationView.self,
    forDecorationViewOfKind: SectionDecorationViewController.sectionBackgroundDecorationElementKind)
```  
  
今回の例では背景色を付けています。  
  
```swift

extension SectionBackgroundDecorationView {
    func configure() {
        backgroundColor = UIColor.lightGray.withAlphaComponent(0.5)
        layer.borderColor = UIColor.black.cgColor
        layer.borderWidth = 1
        layer.cornerRadius = 12
    }
}
```  
  
ここでも特定の文字列を指定します。  
  
```swift

static let sectionBackgroundDecorationElementKind = "section-background-element-kind"
```  
  
  
## PinnedSectionHeaderFooterViewController  
  
上記の例に似ていますが  
ヘッダーが上部に固定されるようになっています。  
  
![Simulator Screen Shot - iPhone Xʀ - 2019-06-16 at 05.25.01.png](/image/2ef451d4-be85-f7be-3d5d-ee8a5bb55a05.png)  
  
  
```swift

extension PinnedSectionHeaderFooterViewController {
    func createLayout() -> UICollectionViewLayout {
        let itemSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                              heightDimension: .fractionalHeight(1.0))
        let item = NSCollectionLayoutItem(layoutSize: itemSize)

        let groupSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                               heightDimension: .absolute(44))
        let group = NSCollectionLayoutGroup.horizontal(layoutSize: groupSize, subitems: [item])

        let section = NSCollectionLayoutSection(group: group)
        section.interGroupSpacing = 5
        section.contentInsets = NSDirectionalEdgeInsets(top: 0, leading: 10, bottom: 0, trailing: 10)

        let sectionHeader = NSCollectionLayoutBoundarySupplementaryItem(
            layoutSize: NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                               heightDimension: .estimated(44)),
            elementKind: PinnedSectionHeaderFooterViewController.sectionHeaderElementKind,
            alignment: .top)
        let sectionFooter = NSCollectionLayoutBoundarySupplementaryItem(
            layoutSize: NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                               heightDimension: .estimated(44)),
            elementKind: PinnedSectionHeaderFooterViewController.sectionFooterElementKind,
            alignment: .bottom)
        sectionHeader.pinToVisibleBounds = true
        sectionHeader.zIndex = 2
        section.boundarySupplementaryItems = [sectionHeader, sectionFooter]

        let layout = UICollectionViewCompositionalLayout(section: section)
        return layout
    }
}
```  
  
ポイントは  
  
```swift

sectionHeader.pinToVisibleBounds = true
```  
  
を設定するだけでヘッダーを固定させることができます。  
  
  
# ネストしたGroup  
  
ここからはさらに複雑なレイアウトの設定方法について見ていきます。  
  
## NestedGroupsViewController  
  
1行の中に2つの異なるレイアウトを設定しています。  
  
![Simulator Screen Shot - iPhone Xʀ - 2019-06-16 at 05.40.46.png](/image/50fd0fed-0a07-cf54-ed5e-0bea9886e69e.png)  
  
```swift

extension NestedGroupsViewController {
   func createLayout() -> UICollectionViewLayout {
        let layout = UICollectionViewCompositionalLayout {
            (sectionIndex: Int, layoutEnvironment: NSCollectionLayoutEnvironment) -> NSCollectionLayoutSection? in

            let leadingItem = NSCollectionLayoutItem(
                layoutSize: NSCollectionLayoutSize(widthDimension: .fractionalWidth(0.7),
                                                   heightDimension: .fractionalHeight(1.0)))
            leadingItem.contentInsets = NSDirectionalEdgeInsets(top: 10, leading: 10, bottom: 10, trailing: 10)

            let trailingItem = NSCollectionLayoutItem(
                layoutSize: NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                                   heightDimension: .fractionalHeight(0.3)))
            trailingItem.contentInsets = NSDirectionalEdgeInsets(top: 10, leading: 10, bottom: 10, trailing: 10)
            let trailingGroup = NSCollectionLayoutGroup.vertical(
                layoutSize: NSCollectionLayoutSize(widthDimension: .fractionalWidth(0.3),
                                                   heightDimension: .fractionalHeight(1.0)),
                subitem: trailingItem, count: 2)

            let nestedGroup = NSCollectionLayoutGroup.horizontal(
                layoutSize: NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                                   heightDimension: .fractionalHeight(0.4)),
                subitems: [leadingItem, trailingGroup])
            let section = NSCollectionLayoutSection(group: nestedGroup)
            return section

        }
        return layout
    }
}
```  
  
Groupの中にGroupを設定することで  
実現することができます。  
  
```swift

let nestedGroup = NSCollectionLayoutGroup.horizontal(
    layoutSize: NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                       heightDimension: .fractionalHeight(0.4)),
    subitems: [leadingItem, trailingGroup])
```  
  
## OrthogonalScrollingViewController  
  
上記の例と似ていますが  
個々のネストしているGroupの中で  
横へのスクロールをできるようにしています。  
  
`UICollectionLayoutSectionOrthogonalScrollingBehavior` を使用します。  
  
`UICollectionLayoutSectionOrthogonalScrollingBehavior`  
https://developer.apple.com/documentation/uikit/uicollectionlayoutsectionorthogonalscrollingbehavior  
  
※ 実際の動きはサンプルをご参照ください。  
  
  
![Simulator Screen Shot - iPhone Xʀ - 2019-06-16 at 05.46.02.png](/image/aee6ef70-6152-0304-94d0-a3ded420d56b.png)  
  
  
```swift

extension OrthogonalScrollingViewController {
    func createLayout() -> UICollectionViewLayout {
        ...

        section.orthogonalScrollingBehavior = .continuous
        
        ...
    }
}
```  
  
これまではこの動きを実現するのに  
結構な量のコードを書く必要がありましたが  
たった1行で可能になります。  
  
サンプルでは他の設定の際の動きも見れますので  
ぜひ一度試してみてください。  
  
  
# Compositional Layoutsまとめ  
  
- シンプルで統一的で宣言的なAPI  
- フレームワークに任せる部分が増え、実装がより簡単に  
- 小さな機能を組み合わせてレイアウトのカスタマイズが可能に  
- 小さな単位で開発することでUI開発のイテレーションをより短時間で可能に  
  
# Data Source  
  
今使われている`UICollectionViewDataSource`は  
非常にシンプルな設計となっています。  
  
ジェネリックであるため  
どんな型からもCollectionViewを構築することが可能です。  
  
`UICollectionViewDataSource`  
https://developer.apple.com/documentation/uikit/uicollectionviewdatasource  
  
これまでは１つの情報源から取得できる配列を元にして  
CollectionViewを構築することが多く  
現在の`UICollectionViewDataSource`の形が大変有効でした。  
  
しかし  
時代の変化に応じて  
複数のウェブサービスや端末内のデータベースなどから  
情報を取得する必要が出てきたり  
非同期通信を行う関係でタイミングも  
バラバラに取得されるようになることが増えてきました。  
  
セッションの中でも出てきていましたが  
  
UIからデータソースへ必要な情報を要求し  
セルをレンダリングする時にデータが設定されたセルを要求します。  
  
<img width="1328" alt="スクリーンショット 2019-06-15 12.19.08.png" src="/image/a15ed8b7-f687-cb08-c6af-204f37f2e019.png">  
  
  
しかし  
外部からのデータの取得を元にUIの更新を  
Controller側から行おうとすると  
ControllerからUIへデータの更新を通知する必要が出てくるため  
Controllerの保持しているデータとUIの表示状態に解離が発生するリスクが生まれます。  
  
そしてController内のデータとViewの表示状態の同期を取る必要が出てきます。  
  
<img width="1409" alt="スクリーンショット 2019-06-15 12.21.49.png" src="/image/ed674cf0-2e19-dce5-8f37-11d974e7f75b.png">  
  
WWDC2018のCollectionViewのセッションで扱っていたのが  
まさにこの問題が生じる`performBatchUpdates`メソッドで  
2つの更新の同期が取れていないと`NSInternalInconsistencyException`という  
一見すると何が起きているのかがわかりづらい問題が起きます。  
  
`performBatchUpdates`  
https://developer.apple.com/documentation/uikit/uicollectionview/1618045-performbatchupdates  
  
<img width="1522" alt="スクリーンショット 2019-06-15 12.23.43.png" src="/image/0e971665-6db4-e7fc-1517-efe58ebedffd.png">  
  
# Diffable Data Sourceの登場  
  
そこで登場してきたのが**Diffable Data Source**です。  
  
UIの更新時に`performBatchUpdates`ではなく  
`apply`というメソッドを使うことで  
フレームワーク内で不整合が生じないようにUIの更新をしてくれます。  
  
<img width="1240" alt="スクリーンショット 2019-06-15 12.30.06.png" src="/image/2f78acb5-ed79-f238-4321-c908db023b2e.png">  
  
# 3つのDiffable Data Source  
  
`UICollectionViewDiffableDataSource`  
https://developer.apple.com/documentation/uikit/uicollectionviewdiffabledatasourcereference  
  
`UITableViewDiffableDataSource`  
https://developer.apple.com/documentation/uikit/uitableviewdiffabledatasource  
  
`NSCollectionViewDiffableDataSourceReference`  
https://developer.apple.com/documentation/appkit/nscollectionviewdiffabledatasourcereference  
  
  
# NSDiffableDataSourceSnapshot  
  
もう一つ  
Diffable Data Sourceの中心的存在が**Snapshot**で  
  
**UIの状態を表す唯一の存在(Single Source of Truth)**で  
これまで使用していた`IndexPath`を使わずに  
関連したCollectionView内のセクションやアイテム間で一意となる  
Identifierを使用します。  
  
  
`NSDiffableDataSourceSnapshot`  
https://developer.apple.com/documentation/uikit/nsdiffabledatasourcesnapshot  
  
  
# Diffable Data Sourcesの3ステップ  
  
1. 新しいSnapshot(UIの状態)を生成  
2. 更新したい情報の追加  
3. 現在のSnapshot(UIの状態)の適用(applyメソッドの呼び出し)  
  
と考えるとイメージしやすいかもしれません。  
  
<img width="1383" alt="スクリーンショット 2019-06-15 12.47.16.png" src="/image/1c9e40d7-4c95-7952-70c5-19bf64d0fcf8.png">  
  
# コードから動きを追っていく  
  
ではここからは公式に提供されているサンプルから  
実装の詳細を見てみたいと思います。  
  
  
## MountainsViewController(基本的な使い方)  
  
最初の例は山の名前のリストで  
検索窓にキーワードを入力してフィルターをかけます。  
  
![スクリーンショット 2019-06-15 14.32.13.png](/image/2b58cf85-e610-08b0-c179-78f23d2b8c02.png)  
  
  
```swift

extension MountainsViewController: UISearchBarDelegate {
    func searchBar(_ searchBar: UISearchBar, textDidChange searchText: String) {
        performQuery(with: searchText)
    }
}
```  
  
ここで呼ばれている`performQuery`でUIの更新を行っています。  
  
```swift

extension MountainsViewController {

    func performQuery(with filter: String?) {
        let mountains = mountainsController.filteredMountains(with: filter).sorted { $0.name < $1.name }

        // 1. 新しいSnapshotの生成
        var snapshot = NSDiffableDataSourceSnapshot<Section, MountainsController.Mountain>()

        // 2. 更新したい情報の追加
        snapshot.appendSections([.main])
        snapshot.appendItems(mountains)

        // 3. applyの呼び出し
        dataSource.apply(snapshot, animatingDifferences: true)
    }
}
```  
  
たったこれだけでUIの更新が可能になります。  
  
ではより詳細に見ていきます。  
  
```swift

let snapshot = NSDiffableDataSourceSnapshot<Section, MountainsController.Mountain>()
```  
  
Snapshotはジェネリクスとなっています。  
  
```swift

class NSDiffableDataSourceSnapshot<SectionIdentifierType, ItemIdentifierType> 
      where SectionIdentifierType : Hashable, ItemIdentifierType : Hashable
```  
  
指定するのは`SectionIdentifierType`と`ItemIdentifierType`で  
いずれも`Hashable`に適合する必要があります。  
  
今回のケースですと  
  
`SectionType`にはSection  
  
```swift

class MountainsViewController: UIViewController {
    enum Section: CaseIterable {
        case main
    }
}
```  
Swiftの`enum`は`Hashable`に適合しています。  
  
  
`ItemIdentifierType`にはMountainという  
`Hashable`に適合した`struct`を使用しています。  
  
```swift

class MountainsController {

    struct Mountain: Hashable {
        let name: String
        let height: Int
        let identifier = UUID()
        func hash(into hasher: inout Hasher) {
            hasher.combine(identifier)
        }
        static func == (lhs: Mountain, rhs: Mountain) -> Bool {
            return lhs.identifier == rhs.identifier
        }
        func contains(_ filter: String?) -> Bool {
            guard let filterText = filter else { return true }
            if filterText.isEmpty { return true }
            let lowercasedFilter = filterText.lowercased()
            return name.lowercased().contains(lowercasedFilter)
        }
    }
}
```  
  
WWDCのセッションではSnapshotはハッシュ値を使って個々を識別していると言っています。  
セクションやアイテムは一意になるようにハッシュ値を生成する必要があります。  
  
デモでは下記のようなidentifierを使用しています。  
  
```swift

let identifier = UUID()
```  
  
そして`identifier`を使ってハッシュ値の生成や等価判定を行っています。  
  
  
```swift

func hash(into hasher: inout Hasher) {
    hasher.combine(identifier)
}
static func == (lhs: Mountain, rhs: Mountain) -> Bool {
    return lhs.identifier == rhs.identifier
}
```  
  
  
次にData Sourceの設定を行う部分を見ていきます。  
  
```swift

extension MountainsViewController {
    func configureDataSource() {
        dataSource = UICollectionViewDiffableDataSource
            <Section, MountainsController.Mountain>(collectionView: mountainsCollectionView) {
                (collectionView: UICollectionView, indexPath: IndexPath,
                mountain: MountainsController.Mountain) -> UICollectionViewCell? in
            guard let mountainCell = collectionView.dequeueReusableCell(
                withReuseIdentifier: LabelCell.reuseIdentifier, for: indexPath) as? LabelCell else {
                    fatalError("Cannot create new cell") }
            mountainCell.label.text = mountain.name
            return mountainCell
        }
    }
}
```  
  
これまでcellForItem(at:)で行っていたセルへの設定方法の指定を  
Data Sourceの初期化の引数(クロージャ)として渡しています。  
  
ポイントとしては  
内部で関連したアイテム(ここではMountain)を特定してくれているため  
IndexPathを使う必要がなくなり  
不整合を生じるリスクがありません。  
  
  
## WiFiSettingsViewController(2つの異なるセクション、複数の動的なUIの切り替え手段)  
  
次にもう少し複雑な例です。  
  
![スクリーンショット 2019-06-15 14.31.41.png](/image/fa1a9a1c-3b78-2351-22f8-721b51085df2.png)  
  
2つの異なるセッションがあります。  
そして下のセッションはWi-Fiトグルボタンの設定で  
表示非表示の切り替えができます。  
  
さらに下のセッションのリストは  
定期的にリストの内容が更新されます。  
  
表示の更新は下記のメソッドで行っています。  
  
```swift

extension WiFiSettingsViewController {

    func updateUI(animated: Bool = true) {
        guard let controller = self.wifiController else { return }

        let configItems = configurationItems.filter { !($0.type == .currentNetwork && !controller.wifiEnabled) }

        currentSnapshot = NSDiffableDataSourceSnapshot<Section, Item>()

        currentSnapshot.appendSections([.config])
        currentSnapshot.appendItems(configItems, toSection: .config)

        if controller.wifiEnabled {
            let sortedNetworks = controller.availableNetworks.sorted { $0.name < $1.name }
            let networkItems = sortedNetworks.map { Item(network: $0) }
            currentSnapshot.appendSections([.networks])
            currentSnapshot.appendItems(networkItems, toSection: .networks)
        }

        self.dataSource.apply(currentSnapshot, animatingDifferences: animated)
    }
}

```  
  
基本的な設定方法は最初の例と変わりません。  
  
注目する点としては  
  
```swift

currentSnapshot.appendSections([.config])
currentSnapshot.appendItems(configItems, toSection: .config)
```  
  
と  
  
```swift

currentSnapshot.appendSections([.networks])
currentSnapshot.appendItems(networkItems, toSection: .networks)
```  
  
でそれぞれのセクションに値を設定しています。  
  
そして`wifiEnabled`の値でnetworkセクションは  
表示するかしないかの判定を行っています。  
  
ItemIdentifierTypeで使用しているItemについて見てみます。  
  
```swift

class WiFiSettingsViewController: UIViewController {

    enum Section: CaseIterable {
        case config, networks
    }

    enum ItemType {
        case wifiEnabled, currentNetwork, availableNetwork
    }

    struct Item: Hashable {
        let title: String
        let type: ItemType
        let network: WIFIController.Network?

        init(title: String, type: ItemType) {
            self.title = title
            self.type = type
            self.network = nil
            self.identifier = UUID()
        }
        init(network: WIFIController.Network) {
            self.title = network.name
            self.type = .availableNetwork
            self.network = network
            self.identifier = network.identifier
        }
        var isConfig: Bool {
            let configItems: [ItemType] = [.currentNetwork, .wifiEnabled]
            return configItems.contains(type)
        }
        var isNetwork: Bool {
            return type == .availableNetwork
        }

        private let identifier: UUID
        func hash(into hasher: inout Hasher) {
            hasher.combine(self.identifier)
        }
    }
}

```  
今回ですとWi-Fiセルや使用可能なセルなど異なるセルを  
まとめて扱う必要があるため少し複雑になっていますが  
`Hashable`への適合方法などは変わりません。  
  
  
次にデータソースの設定を見ていきます。  
  
```swift

func configureDataSource() {
    self.dataSource = UITableViewDiffableDataSource
        <Section, Item>(tableView: tableView) { [weak self]
            (tableView: UITableView, indexPath: IndexPath, item: Item) -> UITableViewCell? in
         ...
    }
    self.dataSource.defaultRowAnimation = .fade
}
```  
  
今回は`UITableViewDiffableDataSource`を使用していましたが  
CollectionViewの時と変わらないため割愛させていただきました。  
  
一つ異なる点として`defaultRowAnimation`を設定して  
UI更新時のアニメーションの設定を変更しています。  
  
`defaultRowAnimation`  
https://developer.apple.com/documentation/uikit/uitableviewdiffabledatasource/3255148-defaultrowanimation  
  
  
## InsertionSortViewController(部分更新の例)  
  
最後にこれまでとは異なる形の更新例を見ていきます。  
  
更新ある部分のみが更新されるようにしています。  
  
![スクリーンショット 2019-06-15 15.00.37.png](/image/1dbbc6f7-f872-bc93-2c1e-6152bc3c9031.png)  
  
※  
実際の動きはセッションを見てください。  
  
```swift

extension InsertionSortViewController {

    func performSortStep() {
        if !isSorting {
            return
        }

        var sectionCountNeedingSort = 0

        // 空のSnapshotを新しく生成するのではなく、現在のSnapshotを取得
        var updatedSnapshot = dataSource.snapshot()

        // 各セクションごとに必要な場合はソートをかける
        updatedSnapshot.sectionIdentifiers.forEach {
            let section = $0
            if !section.isSorted {

                // ソート
                section.sortNext()
                let items = section.values

                // 現在のセクションのアイテムを削除してソート後のアイテムと入れ替える
                updatedSnapshot.deleteItems(items)
                updatedSnapshot.appendItems(items, toSection: section)

                sectionCountNeedingSort += 1
            }
        }

        var shouldReset = false
        var delay = 125
        if sectionCountNeedingSort > 0 {
            dataSource.apply(updatedSnapshot)
        } else {
            delay = 1000
            shouldReset = true
        }
        let bounds = insertionCollectionView.bounds
        DispatchQueue.main.asyncAfter(deadline: .now() + .milliseconds(delay)) {
            if shouldReset {
                let snapshot = self.randomizedSnapshot(for: bounds)
                self.dataSource.apply(snapshot, animatingDifferences: false)
            }
            self.performSortStep()
        }
    }
}
```  
  
まず注目したい点として  
  
```swift

var updatedSnapshot = dataSource.snapshot()
```  
  
では  
新しいSnapshotではなく現在のSnapshotを取得しています。  
  
そうすることで現在の各セクションの状態から  
必要なセクションにだけ処理を実行するようにできます。  
  
さらに  
処理の中では`deleteItems`メソッドを使用しています。  
他にも使えるメソッドはたくさんあります。  
  
今回の例でも`IndexPath`は一切しておらず  
アイテムやセルの特定はIdentifierを通してフレームワークが全て行ってくれます。  
  
データソースの設定に関してはこれまでの例と同様なので  
割愛させていただきます。  
  
# より深掘りして考えてみる  
  
これまで見てきた例などから改めてこの新しいAPIの特徴を考えてみます。  
  
#### すべてがapplyメソッドで更新するように統一されている  
  
以前のように複数の手段がなくわかりやすくなっています。  
  
<img width="1251" alt="スクリーンショット 2019-06-15 15.40.01.png" src="/image/5106bef9-6518-a447-5e5e-2e8466cdc691.png">  
  
#### 空のSnapshotと現在のUIのSnapshotの２パターンから更新ができる  
  
必要な部分にだけ処理を実行するといったこともできます。  
  
<img width="1108" alt="スクリーンショット 2019-06-15 15.40.08.png" src="/image/9d2bc85f-14dc-2421-8e5b-4e57aa2bebf2.png">  
  
#### Snapshotの状態を取得したり設定をするための便利なメソッドがある  
  
今回出てきていないメソッドもたくさんあるため  
ドキュメントを一通り見てみるのも良いかもしれません。  
https://developer.apple.com/documentation/uikit/nsdiffabledatasourcesnapshot  
  
#### Identifiers  
  
セクションやアイテム間でユニークである必要があり  
UUIDなどで一意性を確保しておくことを推めています。  
  
さらに`Hashable`に適合する必要がありますが  
Swiftではenumなど自動でHashableに適合している型も多くあるため  
簡単に色々な型をIdentifierにすることができます。  
  
またセルに多くの情報を表示させる場合  
何かしらのDataModelを使用することもあるかと思いますが  
これもHashableに適合させることでIdentifierとして活用できます。  
  
例の中にも出てきたように  
下記のような形でIdentifierを作成するのが良いと言っていました。  
  
```swift

struct MyModel: Hashable {
    let identifier = UUID()
    func hash(into hasher: inout Hasher) {
        hasher.combine(identifier)
    }
    static func == (lhs: MyModel, rhs: MyModel) -> Bool {
        return lhs.identifier == rhs.identifier
    }
}
```  
  
#### これまでのIndexPathを使用したメソッドとも一緒に使用できる  
  
indexPathとIdentifierは内部で管理されており  
IndexPathからItemIdentifierTypeを取得することができます。  
  
https://developer.apple.com/documentation/uikit/uicollectionviewdiffabledatasourcereference/3182924-itemidentifier  
  
後の処理はIdentifierを使用する形で実行できます。  
  
```swift

func collectionView(_ collectionView: UICollectionView,
                    didSelectItemAt indexPath: IndexPath) {
    if let identifier = dataSource.itemIdentifier(for: indexPath) {
        // Do something
    }
}
```  
  
逆の変換も可能です。  
https://developer.apple.com/documentation/uikit/uicollectionviewdiffabledatasourcereference/3182922-indexpath  
  
※  
変換にかかる時間は一定なので  
パフォーマンスは悪くはないとセッションでは言っていました。  
  
# 全体としてのパフォーマンス  
  
セッションでは  
ローレベルでの処理を実装しているため良いと言っています。  
  
また`apply`メソッドは  
メインスレッド以外のスレッドやバックグラウンドで計算を実行し  
結果はメインスレッドで返すことをフレームワーク側で保証しているようです。  
  
何か不整合が生じそうな場合はフレームワーク側でログを出力したり  
アサーションが走るとのことでした。  
  
とは言うものの  
開発時の継続的なパフォーマンスの計測の必要性も挙げています。  
  
# Diffable Data Sourcesまとめ  
  
- シンプルで統一的で宣言的なAPI  
- フレームワークに任せる部分が増え、実装がより簡単に  
- Identifierを使うことによるData SourceとUIの不整合を防止  
- 差分更新によるパフォーマンスの向上  
  
  
# 全体のまとめ  
  
今回は  
新しく登場した  
Compositional Layoutsと  
Diffable Data Sourcesについて見てきました。  
  
全体を通して  
シンプルに統一的な方法で書けるのがすごい良いと思いました。  
コードの見通しも良く読みやすさも格段に上がっていると思います。  
  
またサンプルの動きがすごいスムーズで  
パフォーマンスも非常に良いと感じています。  
  
まだ触れたことがない方は  
まずはサンプルを見てみることで  
基本的な動きが把握できるのでオススメです:smiley:  
  
  
間違いなどございましたらご指摘頂けますと幸いです:bow_tone1:  
  
# (雑記)SwiftUIとの関係を考えてみる🤔  
  
※ Swiftで新規開発をするならばという前提で話を進めます。  
  
完全に個人的な意見ですが  
SwiftUIはシンプルなデザインのViewの構築を  
素早く簡単に型安全に実現できるものだと思っています。  
  
一方で  
今回登場したCompositional LayoutsやDiffable Data Sourcesは  
複雑化するアプリという時代の流れに対応するためにできたという経緯もあるように  
画一的なレイアウトでは対応できない画面や  
パフォーマンスへの配慮が必要になる画面などの  
適所で使っていくようになるのかと感じています。  
  
CollectionViewの  
直接代替となるようなものをSwiftUIは提供していません。  
  
なので全体のレイアウトとしてはSwiftUIで構築し  
よりカスタマイズが必要な画面に関しては  
CollectionViewなどを活用していくのかなというのが現在の印象です。  
  
※ 使われる所が限定的になると言っているのではなく  
むしろアプリの複雑化は今後も進んでいき  
使用される機会は現在よりも増えていくのではないかと思っています。  
  
まだプロダクションでの使用経験もなく印象レベルでの話です。  
ぜひここら辺に関してご意見があればお聞きしたいです:bow_tone1:  
  
# IBPCollectionViewCompositionalLayout  
  
@kishikawakatsumi さんが作成している  
iOS12以前でもUICollectionViewCompositionalLayoutが使用できるようにするライブラリがあります。  
こちらもぜひ参考にしてみてください。  
  
https://github.com/kishikawakatsumi/IBPCollectionViewCompositionalLayout  
  
# DiffableDataSources  
  
Diffable Data Sourcesに関しても同様のライブラリがあります。  
  
https://github.com/ra1028/DiffableDataSources  
