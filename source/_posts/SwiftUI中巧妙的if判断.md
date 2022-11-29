---
title: SwiftUI中巧妙的if判断
date: 2022-11-30 00:22:13
excerpt: 我们一般对于选择性的显示View，或者在满足一定条件的情况下在View之间切换的做法，常见的有两种...
categories:
  - Swift
tags: 
  - Swift
  - SwiftUI
---

我们一般对于选择性的显示View，或者在满足一定条件的情况下在View之间切换的做法，常见的有两种：

```swift
// 使用if
var body: some View {
	if boolean {
		MyView_1()
	} else {
    	MyView_2()
	}
}
```
```swift
// 使用switch
var body: some View {
    switch identifier {
    case "FirstView": 
        MyView_1()
        
    case "secondView": 
        MyView_2()
        
    case "ThirdView": 
        MyView_3()
        
    default:
        DefaultView()
    }
}
```

但当我们遇到更为复杂的情况，例如对View有更多自定义的要求的时候，以上方法显然有些捉襟见肘了。

很久之前查阅相关资料的时候，偶然看到了一段实用的代码，经过改善后如下：

```swift
extension View {
    @ViewBuilder
    func `if`<Content: View>(_ conditional: Bool, ifTure: ((Self) -> Content)?, ifFalse: ((Self) -> Content)?) -> some View {
        if conditional {
            if ifTure != nil {
                ifTure!(self)
            } else {
                self
            }
        } else {
            if ifFalse != nil {
                ifFalse!(self)
            } else {
                self
            }
        }
    }
}
```

建议单独写为一份扩展文件，可以随时添加到新项目中去。

<br/>

> 不足之处，欢迎指正。
