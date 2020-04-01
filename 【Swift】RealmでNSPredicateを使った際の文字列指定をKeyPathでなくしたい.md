  
https://github.com/kishikawakatsumi/RealmTypeSafeQuery  
https://qiita.com/nakau1/items/40865299dacc50d71604  
を参考にさせて頂きました。  
  
## 経緯  
Realmでデータ取得時にNSPredicateを使ってfilterの条件を指定することがあるかと  
思いますが、通常の方法ですと以下のように文字列指定になります。  
  
```Swift
NSPredicate(format: "name = %@", argumentArray: ["佐藤"])
```  
  
この場合、下記のような間違いをすると実行時エラーが起きます。  
  
```Swift
NSPredicate(format: "nema = %@", argumentArray: ["佐藤"])
                      ↑↑↑
```  
  
これを避けるための実装を試してみました。  
  
作成したサンプル  
https://github.com/stzn/RealmSample  
  
## 実装  
  
※下記で登場する```RealmProperty```などの定義は  
[RealmTypeSafeQuery](https://github.com/kishikawakatsumi/RealmTypeSafeQuery/blob/master/RealmTypeSafeQuery/RealmPropertyType.swift)を参考にしておりますので割愛させて頂きます。  
  
  
元々下記のような実装があり、  
  
```QueryType.swift
protocol Queryable {
    associatedtype Query: QueryType = DefaultQuery
}

protocol QueryType {
    var predicate: NSPredicate? { get }
    var sortDescriptors: [SortDescriptor] { get }
}

enum DefaultQuery: QueryType {
    
    var predicate: NSPredicate? {
        return nil
    }
    
    var sortDescriptors: [SortDescriptor] {
        return []
    }
}
```  
  
```Person.swift
public final class Person: Object {
    
    @objc public dynamic var id: Int = 0
    @objc public dynamic var name: String = ""
    @objc public dynamic var age: Int = 0
    @objc public dynamic var isAdmin: Bool = false
    
    public override static func primaryKey() -> String? {
        return "id"
    }
}

extension Person: Queryable {
    
    public enum Query: QueryType {
        case nameEqual(String)
        case nameStartWith(String)
        case nameEndWith(String)
        case overAge(Int)
        case underAge(Int)
        case ageBetween(Int, Int)
        case admin(Bool)
        case nameAndAge(String, Int)
        case nameOrAge(String, Int)
        case notNameEndWith(String)
        case all

        public var predicate: NSPredicate? {
            switch self {
            case .nameEqual(let name):
                return NSPredicate(format: "name = %@", argumentArray: [name])
            case .nameStartWith(let name):
                return NSPredicate(format: "name BEGINSWITH '%@'", argumentArray: [name])
            case .nameEndWith(let name):
                return NSPredicate(format: "name ENDSWITH '%@'", argumentArray: [name])
            case .overAge(let age):
                return NSPredicate(format: "age >= %@", argumentArray: [age])
            case .underAge(let age):
                return NSPredicate(format: "age < %@", argumentArray: [age])
            case .ageBetween(let min, let max):
                return NSPredicate(format: "age BETWEEN {%@, %@}", argumentArray: [min, max])
            case .admin(let isAdmin):
                return NSPredicate(format: "isAdmin = %@", argumentArray: [isAdmin])
            case .nameAndAge(let name, let age):
                return NSPredicate(format: "name = %@ AND age = %@", argumentArray: [name, age])
            case .nameOrAge(let name, let age):
                return NSPredicate(format: "name = %@ OR age = %@", argumentArray: [name, age])
            case .notNameEndWith(let name):
                return NSPredicate(format: "NOT (name ENDSWITH '%@')", argumentArray: [name])
            case .all:
                return nil
            }
        }
        
        // 利便性のためにデフォルト値を設定(必要に応じて上書き)
        public var sortDescriptors: [SortDescriptor] {
            return [SortDescriptor(keyPath: "name")]
        }
    }
}
```  
  
上記の文字列指定の部分をSwift4のKeyPathに置き換えていきます。  
  
  
まず、KeyPathを使ったNSPredicateのイニシャライザを定義します。  
  
```NSPredicate+Initializers.swift
public extension NSPredicate {
    
    private convenience init<T: Object, P:RealmProperty>(_ property: KeyPath<T, P>, _ operation: String, _ value: P) {
        self.init(format: "\(property._kvcKeyPathString!) \(operation) %@", argumentArray: [value._object])
    }
    
    public convenience init<T: Object, P:EquatableProperty>(_ property: KeyPath<T, P>, equal value: P) {
        self.init(property, "=", value)
    }
    
    public convenience init<T: Object, P:EquatableProperty>(_ property: KeyPath<T, P>, notEqual value: P) {
        self.init(property, "!=", value)
    }
    
    public convenience init<T: Object, P:ComparableProperty>(_ property: KeyPath<T, P>, equalOrGreaterThan value: P) {
        self.init(property, ">=", value)
    }
    
    public convenience init<T: Object, P:ComparableProperty>(_ property: KeyPath<T, P>, equalOrLessThan value: P) {
        self.init(property, "<=", value)
    }
    
    public convenience init<T: Object, P:ComparableProperty>(_ property: KeyPath<T, P>, greaterThan value: P) {
        self.init(property, ">", value)
    }
    
    public convenience init<T: Object, P:ComparableProperty>(_ property: KeyPath<T, P>, lessThan value: P) {
        self.init(property, "<", value)
    }
    
}

// 文字列検索(NSStringの場合はエスケープをする)
public extension NSPredicate {
    
    public convenience init<T: Object, P:RegexMatchableProperty>(_ property: KeyPath<T, P>, contains value: P) {
        if let v = value._object as? NSString {
            self.init(format: "\(property._kvcKeyPathString!) CONTAINS '\(v.realmEscaped)'" )
            return
        }
        self.init(format: "\(property._kvcKeyPathString!) CONTAINS '\(value._object)'" )
    }
    
    public convenience init<T: Object, P:RegexMatchableProperty>(_ property: KeyPath<T, P>, beginsWith value: P) {
        if let v = value._object as? NSString {
            self.init(format: "\(property._kvcKeyPathString!) BEGINSWITH '\(v.realmEscaped)'" )
            return
        }
        self.init(format: "\(property._kvcKeyPathString!) BEGINSWITH '\(value._object)'" )
    }
    
    public convenience init<T: Object, P:RegexMatchableProperty>(_ property: KeyPath<T, P>, endsWith value: P) {
        if let v = value._object as? NSString {
            self.init(format: "\(property._kvcKeyPathString!) ENDSWITH '\(v.realmEscaped)'" )
            return
        }
        self.init(format: "\(property._kvcKeyPathString!) ENDSWITH '\(value._object)'" )
    }
}

// 範囲検索
public extension NSPredicate {
    
    public convenience init<T: Object, P:ComparableProperty>(_ property: KeyPath<T, P>, in values: [P]) {
        self.init(format: "\(property._kvcKeyPathString!) IN %@", argumentArray: values.map { $0._object } )
    }
    
    public convenience init<T: Object, P:ComparableProperty>(_ property: KeyPath<T, P>, between min:P, to max: P ) {
        self.init(format: "\(property._kvcKeyPathString!) BETWEEN {%@, %@}", argumentArray: [min, max] )
    }
}

// 日付範囲
public extension NSPredicate {
    
    public convenience init<T: Object, P:ComparableProperty>(_ property: KeyPath<T, P>, fromDate: Date?, to: Date?) {
        
        var format = ""
        var args = [Any]()
        if let from = fromDate {
            format += "\(property._kvcKeyPathString!) >= %@"
            args.append(from)
        }
        
        if  let to = to {
            if !format.isEmpty {
                format += " AND "
            }
            format += "\(property._kvcKeyPathString!) <= %@"
            args.append(to)
        }
        if !args.isEmpty {
            self.init(format: format, argumentArray: args)
        } else {
            self.init(value: true)
        }
    }
}

// エスケープ
public extension NSString {
    
    public var realmEscaped: NSString {
        let reps = [
            "\\":"\\\\",
            "'":"\\"
        ]
        var ret = self
        for rep in reps {
            ret = self.replacingOccurrences(of: rep.0, with: rep.1) as NSString
        }
        return ret
    }
}
```  
  
複数の検索条件の場合は下記のメソッドを使用します。  
  
```NSPredicate+Compound.swift
public extension NSPredicate {
    
    public func compound(_ predicates: [NSPredicate], type: NSCompoundPredicate.LogicalType = .and) -> NSPredicate {

        var p = predicates
        p.insert(self, at: 0)
        
        switch type {
        case .and:
            return NSCompoundPredicate(andPredicateWithSubpredicates: p)
        case .not:
            return NSCompoundPredicate(notPredicateWithSubpredicate: self.compound(p))
        case .or:
            return NSCompoundPredicate(orPredicateWithSubpredicates: p)
        }
    }
    
    public func and(_ predicate: NSPredicate) -> NSPredicate {
        return self.compound([predicate], type: .and)
    }
    
    public func or(_ predicate: NSPredicate) -> NSPredicate {
        return self.compound([predicate], type: .or)
    }
    
    public func not(_ predicate: NSPredicate) -> NSPredicate {
        return self.compound([predicate], type: .not)
    }

    // Notだけ使いたいときなどに便利
    public static var empty: NSPredicate {
        return NSPredicate(value: true)
    }
}
```  
  
Personクラスを以下のように変更します。  
  
```Person.swift

extension Person: Queryable {
    
    public enum Query: QueryType {
        case nameEqual(String)
        case nameStartWith(String)
        case nameEndWith(String)
        case overAge(Int)
        case underAge(Int)
        case ageBetween(Int, Int)
        case admin(Bool)
        case nameAndAge(String, Int)
        case nameOrAge(String, Int)
        case notNameEndWith(String)
        case all

        public var predicate: NSPredicate? {
            switch self {
            case .nameEqual(let name):
                return NSPredicate(\Person.name, equal: name)
            case .nameStartWith(let name):
                return NSPredicate(\Person.name, beginsWith: name)
            case .nameEndWith(let name):
                return NSPredicate(\Person.name, endsWith: name)
            case .overAge(let age):
                return NSPredicate(\Person.age, equalOrGreaterThan: age)
            case .underAge(let age):
                return NSPredicate(\Person.age, lessThan: age)
            case .ageBetween(let min, let max):
                return NSPredicate(\Person.age, between: min, to: max)
            case .admin(let isAdmin):
                return NSPredicate(\Person.isAdmin, equal: isAdmin)
            case .nameAndAge(let name, let age):
                return NSPredicate(\Person.name, equal: name)
                        .and(NSPredicate(\Person.age, equal: age))
            case .nameOrAge(let name, let age):
                return NSPredicate(\Person.name, equal: name)
                        .or(NSPredicate(\Person.age, equal: age))
            case .notNameEndWith(let name):
                return NSPredicate.empty
                        .not(NSPredicate(\Person.name, endsWith: name))
            case .all:
                return nil
            }
        }
        
        // 利便性のためにデフォルト値を設定(必要に応じて上書き)
        public var sortDescriptors: [SortDescriptor] {
            return [SortDescriptor(keyPath: \Person.name)]
        }
    }
}
```  
  
今回用にクエリを実行するメソッドを追加します。  
  
```Swift
extension Queryable where Self: Object {
    
    static func query(q: QueryType) -> [Self] {
        
        let realm = try! Realm()
        var objects = realm.objects(Self.self)

        if let predicate = q.predicate {
            objects = objects.filter(predicate)
        }
        objects = objects.sorted(by: q.sortDescriptors)

        var items = [Self]()
        for obj in objects {
            items.append(obj)
        }
        return items
    }
}
```  
  
最終的に下記のようにアクセスします。  
  
```Swift
func getAllItems() -> [Person] {
    return Person.query(q: Person.Query.all)
}
    
func getNameEquals(name: String) -> [Person] {
    return Person.query(q: Person.Query.nameEqual(name))
}
```  
  
  
## RealmOptionalだと  
  
_kvcKeyPathStringがnilになり実行時エラーが発生します。(`@objc`が使えないため)  
  
```Swift
public init<T: Object, P: ComparableProperty>(keyPath: KeyPath<T, RealmOptional<P>>, ascending: Bool = true) {
    // 実行時エラー
    self.init(keyPath: keyPath._kvcKeyPathString!, ascending: ascending)
}

```  
ですので、RealmOptionalを使用するところは文字列を使わざるを得ないのが現状です。  
  
## ※※※  
_kvcKeyPathStringは非公開なので突然の変更があるかもしれません。  
現在、公開してほしいとのリクエストがされています。  
https://bugs.swift.org/browse/SR-5220  
