---
title: Swift 中的渐变色按钮
date: 2022-11-30 00:44:43
excerpt: 今天在做项目的时候偶然需要一个“渐变色”按钮，查阅了许多资料，走了一些弯路，不过好在最终是达成效果了...
categories:
  - 大前端
tags: 
  - Swift
---

今天在做项目的时候偶然需要一个“渐变色”按钮，查阅了许多资料，走了一些弯路，不过好在最终是达成效果了。

下面主要讲解两种常用解决方案，针对iOS和macOS，可以分开对应使用。

### draw(_ dirtyRect:)

该方案macOS和iOS通用，且macOS推荐使用此方法。

下面以macOS (Big Sur 11.4) 为例：

```swift
// subclass一个NSButton类

class GradientButton: NSButton {}
```

加入如下属性

```swift
class GradientButton: NSButton {
    var needsGradientBackground: Bool = false
    var startColor: NSColor?
    var endColor: NSColor?
    var gradientAngle: CGFloat = 0
}
```

对NSButton的draw(_ dirtyRect:)进行override

```swift
override func draw(_ dirtyRect: NSRect) {
    // Drawing code here.
    if needsGradientBackground,
       let startColor = startColor,
       let endColor = endColor
    {
        let gradient = NSGradient(starting: startColor, ending: endColor)
        /* OR */
        // 适用于混合两种及以上颜色
        let gradient2 = NSGradient(colors: [startColor, endColor, NSColor.systemPink])

        gradient?.draw(in: bounds, angle: gradientAngle)
    }
    
    // 配置好之后再调用
    super.draw(dirtyRect)
}
```

最后实例化一个button

```swift
class ViewController: NSViewController {
    var gradientBtn: GradientButton!

    override func viewDidLoad() {
        super.viewDidLoad()
        
        gradientBtn = .init(frame: NSMakeRect(50, 50, 200, 50))
        gradientBtn.isBordered = false
        // 记得设置开始与结束的颜色
        gradientBtn.startColor = NSColor.orange
        gradientBtn.endColor = NSColor.systemPink
        // 记得设置为true
        gradientBtn.needsGradientBackground = true
        // 可以自定义layer
        gradientBtn.wantsLayer = true
        gradientBtn.layer?.cornerRadius = 24
        // title颜色默认systemColor，可以用attributedTitle代替
        gradientBtn.title = "Gradient"
        /* OR */
        gradientBtn.attributedTitle = NSAttributedString(string: "Gradient", attributes: [.foregroundColor: NSColor.systemBlue])
        
        view.addSubview(gradientBtn)
    }
}
```

大功告成！

### CAGradientLayer

该方法不推荐macOS使用（至少从我的测试结果来看，为NSButton添加一层layer，会直接覆盖掉title，导致无法显示按钮文字。如果有解决办法，欢迎在评论区指出）。

下面以iOS (14.6) 为例，在整合了所查阅到的部分资料后，得到以下UIView的extension，可以直接拷贝使用：

```swift
// 创建extension文件

extension UIView {
    
    @discardableResult
    func applyGradient(_ colours: [UIColor]) -> CAGradientLayer {
        // 起点终点坐标参数可选，默认从左往右渐变
        // startPoint和endPoint参数参考：minX: 0, minY: 0, maxX: 1. maxY: 1
        return applyGradient(colours: colours)
    }
    
    @discardableResult
    func applyGradient(colours: [UIColor], startPoint: CGPoint = CGPoint(x: 0, y: 0.5), endPoint: CGPoint = CGPoint(x: 1, y: 0.5)) -> CAGradientLayer {
        let gradient: CAGradientLayer = CAGradientLayer()
        gradient.frame = self.bounds
        gradient.colors = colours.map { $0.cgColor }
        gradient.startPoint = startPoint
        gradient.endPoint = endPoint
        gradient.cornerRadius = layer.cornerRadius
        // 一定要插入到index = 0处，保证layer在最上层
        layer.insertSublayer(gradient, at: 0)
        
        return gradient
    }
}
```

实例化一个UIButton

```swift
class ViewController: UIViewController {
    @IBOutlet var gradientBtn: UIButton!

    override func viewDidLoad() {
        super.viewDidLoad()
        
        gradientBtn.setTitle("Gradient", for: .normal)
        gradientBtn.layer.cornerRadius = 12
        gradientBtn.applyGradient([.systemPink, .systemYellow])
    }
}
```

Done!

<br/>

> 不足之处，欢迎指正。
