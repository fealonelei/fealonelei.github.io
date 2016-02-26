---
layout: post
title: Swift 通过 Protocols 做模拟测试
category: 开发
tags: Swift Protocols
keywords: Swift, Protocols, Unit test
description:
---

## 前言
***
原文地址：<http://blog.eliperkins.me/mocks-in-swift-via-protocols>
***


在过去，如果想为一个写的不是那么好的 iOS app 做测试是一件很困难的事情。开发者们已经开发出了很多测试的工具和方法，包括像 [OCMock](http://ocmock.org/) 和 [OCMockito](https://github.com/jonreid/OCMockito) 这样的模拟测试框架(mocking frameworks). 这些框架高度依赖 Objective-C runtime，当 Swift 时代来临的时候，这些框架的实现方式就显得毫无用处(原文：these frameworks have seen their implementations rendered useless). <br />
但是，在 Swift 实践中，在不依赖 Objective-C runtime 的情况下，我们可能有更好的方法来对我们的代码进行 hacking 和 swizzling.         
<br />

***

## Mocking `UIApplication`

举个🌰， `UIApplication` 是一个比较难做模拟测试的 class，这次让我们试试。<br /> <br />
在这个🌰中，让我们试试一个操作(handle) push notifications 的结构体(原文：a type). <br />

		struct PushNotificationController {
		}

我们将为这个 struct 添加一系列的函数，像请求用户允许我们推送通知的函数。通过 `UIApplication` 的 `registerUserNotificationSettings(_:)`, 我们可以处理基于 device token 等的推送和代理的函数调用。     

比方说，我们在 app 的某些状态下不要弹出“允许推送通知”的 UIAlertView，举个具体实例：我们希望在用户登录之后才询问“是否允许推送通知”，而不是用户刚打开 app 时就开始用各种 Alert view 轰炸用户。在需要的函数里调用 `UIApplication.sharedApplication().registerUserNotificationSettings(_:)`是简单暴力的做法。<br />
像下面这样：

	struct PushNotificationController {
	    var user: User {
	        didSet {
	            let application = UIApplication.sharedApplication()
	            application.registerUserNotificationSettings(_:)
	        }
	    }
	}

很简单吧？好吧，我们测试一下这个函数：我们为 `PushNotificationController` 写单元测试，这样我们就知道是否是在用户登录的时候才注册推送通知。

	import XCTest
	
	class PushNotificationControllerTests: XCTestCase {
	    func testControllerRegistersForPushesAfterSettingAUser() {
	        let controller = PushNotificationController()
	        controller.user = User()
	        
	        // uhhh... now what?
	    }
	}

<br />
***

## 测试出了什么问题？
让我们回退一步，看看是什么原因使我们的代码无法测试。看样子我们做错了几个地方。我们不 own `UIApplication` 或者它的 `sharedApplication()`，所以将我们的函数集成进去有点困难。再就是，在单元测试中，我们没法知道调用 `UIApplication.sharedApplication().registerUserNotificationSettings(_:)` 是否有用。我们不能知道屏幕是否真有一个 alert view.  <br />

我们到底要测试什么？测试 UIKit？那是苹果工程师的活。我们真需要测试的只是 struct(即：PushNotificationController)询问用户允许注册推送通知，在这种情况下，相关的就是 `UIApplication` 了。
<br /> <br />
***

## Protocol-Oriented Programming

我们可以灵活一点不？怎么验证 struct(即：PushNotificationController)的功能呢？<br />
**我觉得 protocols 是 Swift 中最佳的模拟测试的方式**
<br />

创建一个 Protocol 代表 UIApplication. 

	protocol PushNotificationRegistrar {
	    func registerUserNotificationSettings(_:)
	} 

很简单，一个 `PushNotificationRegistrar` 是实现了 `registerUserNotificationSettings(_:)` 的任意类型。
接下来，将 `PushNotificationRegistrar` 注入到 `PushNotificationController` 

	struct PushNotificationController {
	    let registrar: PushNotificationRegistrar
	    init(registrar: PushNotificationRegistrar) {
	        self.registrar = registrar
	    }
	}

完美！调用 `registrar` 而不是调用 `registerUserNotificationSettings(_:)`.
	
	struct PushNotificationController {
	    let registrar: PushNotificationRegistrar
	    init(registrar: PushNotificationRegistrar) {
	        self.registrar = registrar
	    }
	
	    var user: User {
	        didSet {
	            registrar.registerUserNotificationSettings(_:)
	        }
	    }
	}

漂亮！ `UIApplication.sharedApplication()` 已经完全被移除了，更少的全局变量给我们单元测试更多的回旋余地。
<br />
***

## 用 `PushNotificationRegistrar` hooking up `UIApplication`
既然不能控制 `UIApplication.sharedApplication()` , 那么怎么解决单元测试的问题呢？这就要用到 Swift 里非常优雅的部分了。
我们可以让 `UIApplication` conform to `PushNotificationRegistrar`

	extension UIApplication: PushNotificationRegistrar {}
	
因为 `UIApplication` 已经有 `registerUserNotificationSettings(_:)`, 所以 `UIApplication` 已经实现了 `PushNotificationRegistrar` protocol. 我们可以在 application delegate 里创建一个 `PushNotificationController`，放在 `application(_:didFinishLaunchingWithOptions:)`,像这样：

	extension AppDelegate: UIApplicationDelegate {
	    func application(application: UIApplication, didFinishLaunchingWithOptions launchOptions: [NSObject : AnyObject]?) -> Bool {
	        let controller = PushNotificationController(application: application)
	        controller.user = User()
	    }
	}	
<br />
***

## 通过 Protocols 模拟测试
好吧，写个测试。不用 `UIApplication` 了，我们来伪造一个注册过程。

	import XCTest
	
	class PushNotificationControllerTests: XCTestCase {
	    func testControllerRegistersForPushesAfterSettingAUser() {
	        class FauxRegistrar: PushNotificationRegistrar {
	            var registered = false
	            func registerUserNotificationSettings(notificationSettings: UIUserNotificationSettings) {
	                registered = true
	            }
	        }
	
	        let registrar = FauxRegistrar()
	        var controller = PushNotificationController(registrar: registrar)
	        controller.user = User()
	        XCTAssertTrue(registrar.registered)
	    }
	}

就这样，我们就创建了一个测试。这个测试在 application 设置了一个用户后，application 将注册推送通知。
<br /> <br />
***

## Crusty 会怎么做？
通过 Protocols 来做 Swift 的模拟测试不仅仅是因为 `UIApplication` 里的方法测试比较困难， Protocols 做的更好。Protocols 能够在整个工程各个组成部分之前创建高清晰的界限(原文：Protocols contribute greatly in creating boundaries around pieces of your architecture)，并且能让软件不会变得 too crusty. <br />
这对 Swift 不新鲜，但是现在用这种模式来编程的还较少。 Protocols extensions with default implementions 将极大地释放 Swift 2 的生产力。 <br />

***
***

原文作者写了一个 Demo 放在 Github 上，地址在[这里](https://gist.github.com/eliperkins/8f4115151497dc1953ea).

***
***
























