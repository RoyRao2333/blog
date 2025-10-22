---
title: Swift 中获取App上次打开时间
date: 2022-11-30 00:37:03
excerpt: 在日常开发中，我们经常会遇到一些情况，需要获取用户上次打开App的时间...
categories:
  - 大前端
tags: 
  - Swift
---

在日常开发中，我们经常会遇到一些情况，需要获取用户上次打开App的时间。

这里介绍一个最简单的方法，以macOS为例：

首先我们需要在App退出之前保存当前时间

```swift
// In AppDelegate

func applicationWillResignActive(_ notification: Notification) {
	UserDefaults.standard.set(Date(), forKey: "LastOpened") 
}

// OR

func applicationWillTerminate(_ aNotification: Notification) {
    // Insert code here to tear down your application
    UserDefaults.standard.set(Date(), forKey: "LastOpened") 
}
```

然后在App下次启动的时候去获取

```swift
// In AppDelegate

func applicationDidBecomeActive(_ notification: Notification) { 
	guard let lastOpened = UserDefaults. standard. object(forKey: "LastOpened") as? Date else { return }
	
	let elapsed = Calendar.current.dateComponents([.day], from: tastOpened, to: Date()) 
	if elapsed.day! >= 1 { // Do your logics
		// show alert 
	}
}

// OR

func applicationDidFinishLaunching(_ aNotification: Notification) {
    // Create the popover and set the content view.

	// 以上相同
}
```

That’s it! iOS同理。

<br/>

> 不足之处，欢迎指正。
