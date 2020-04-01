使ったことがなかったのですが  
昨日たまたま活用する機会があったので  
忘れないように記録します。  
  
以下はスケジュールを日付別に時間順で並べています。  
  
```TableView.swift 
 
import UIKit
import PlaygroundSupport

struct Plan {
    let name: String
    let detail: String
    let date: Date
}

class MyViewController : UIViewController {
    
    var schedules = [Date: [Plan]]()
    var dateOrder = [Date]()
    
    private var formatter: DateFormatter  {
        let f = DateFormatter()
        f.dateStyle = .long
        f.timeStyle = .none
        f.locale = Locale(identifier: "ja_JP")
        return f
    }

    private var timeFormatter: DateFormatter  {
        let f = DateFormatter()
        f.dateStyle = .none
        f.timeStyle = .short
        f.locale = Locale(identifier: "ja_JP")
        return f
    }

    override func viewDidLoad() {
        super.viewDidLoad()
        
        prepare()
        
        let tv = UITableView(frame: self.view.frame)
        tv.dataSource = self
        tv.delegate = self
        view.addSubview(tv)
    }
    
    private func prepare() {
        let f = DateFormatter()
        f.locale = Locale(identifier: "en_US_POSIX")
        f.dateFormat = "yyyy/MM/dd HH:mm"
        
        let plans = [
            Plan(name: "休み", detail: "実家で過ごす", date: f.date(from: "2018/01/01 09:00")!),
            Plan(name: "外出", detail: "初詣", date: f.date(from: "2018/01/03 10:00")!),
            Plan(name: "仕事", detail: "社内会議", date: f.date(from: "2018/01/03 17:30")!),
            Plan(name: "外出", detail: "買い出し", date: f.date(from: "2018/01/15 19:00")!),
            Plan(name: "仕事", detail: "Team Meeting", date: f.date(from: "2018/01/15 15:30")!),
            Plan(name: "休み", detail: "家族で水族館", date: f.date(from: "2018/01/17 08:00")!),
            Plan(name: "休み", detail: "飲み会", date: f.date(from: "2018/01/17 20:00")!)
        ]
        
        schedules = Dictionary(grouping: plans) { plan -> Date in
            f.dateFormat = "yyyy/MM/dd"
            let s = f.string(from: plan.date)
            return f.date(from: s)!
        }
        .reduce(into: [Date: [Plan]]()) { dic, tuple in
            dic[tuple.key] = tuple.value.sorted { $0.date < $1.date }
        }

        // 日付順を保持するための配列
        dateOrder = Array(schedules.keys).sorted { $0 < $1 }
     }
    
}

extension MyViewController: UITableViewDataSource {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        let targetDate = dateOrder[section]
        return schedules[targetDate, default: []].count
    }

    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let reuseIdentifier = "cell"
        
        var cell = tableView.dequeueReusableCell(withIdentifier: reuseIdentifier)
        if cell == nil {
            cell = UITableViewCell(style: .subtitle, reuseIdentifier: reuseIdentifier)
        }
        
        let targetDate = dateOrder[indexPath.section]
        guard let plan = schedules[targetDate]?[indexPath.row] else {
            return cell!
        }

        cell!.textLabel?.text = plan.name
        
        let time = timeFormatter.string(from: plan.date)
        cell!.detailTextLabel?.text = "\(plan.detail) \(time)予定"
        return cell!
    }
}

extension MyViewController: UITableViewDelegate {
    
    func tableView(_ tableView: UITableView, viewForHeaderInSection section: Int) -> UIView? {

        let targetDate = dateOrder[section]
        
        let label = UILabel()
        label.backgroundColor = .lightGray
        label.textColor = .white
        
        label.text = formatter.string(from: targetDate)
        
        return label
    }
    
    func tableView(_ tableView: UITableView, heightForRowAt indexPath: IndexPath) -> CGFloat {
        return 60.0
    }
    
    func numberOfSections(in tableView: UITableView) -> Int {
        return schedules.keys.count
    }
}

let viewController = MyViewController()
let nv = UINavigationController(rootViewController: viewController)
nv.view.frame = CGRect(x: 0, y: 0, width: 320, height: 580)

// Present the view controller in the Live View window
PlaygroundPage.current.liveView = nv.view
```  
  
以下のようになりました。  
  
![20180206.png](/image/34ca2ce7-f68a-990c-cab0-08bed6ec07d1.png)  
  
  
## 疑問  
  
そういえば、Dictionaryにデフォルト値を設定できるようになったことも思い出し  
使ってみましたが、どういう時に使えばいいのかがよくわかっていません。。。  
  
  
```存在しない場合は0.swift
// schedules[targetDate]?.count ?? 0 の方がわかりやすい？
schedules[targetDate, default:[]].count
```  
  
```クラッシュする.swift

// 存在しないtargetDateを設定した場合

// guardの中に入る
guard let plan = schedules[targetDate]?[indexPath.row] else {
    return cell!
}

// Optionalではないのでguardチェックできず
// Fatal error: Index out of rangeになる
// こういう使い方はNG？
let plan = schedules[targetDate, default:[]][indexPath.row]

```  
  
何か良い使い方などございましたらぜひ教えてください:bow_tone1:  
