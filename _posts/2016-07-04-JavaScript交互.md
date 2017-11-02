---
layout: post
title: JavaScript交互
categories: iOS
tags: iOS
author: SindriLin
---

* content
{:toc}

当前混合开发模式迎来了前所未有的发展，跨平台开发、热更新等优点决定了这种模式的重要地位。虽然前端界面在交互、动效等多方面距离原生应用还有差距，但毫无疑问混合开发只会被越来越多的公司接受。在iOS中，混合开发模式被分为两个时代，分别是iOS7之前的坑爹时代与之后的黄金时代，其分割代表为`JavaScriptCore`框架
<span><img src="/images/JavaScript交互/1.jpeg" width="800"></span>

坑爹时代
----
作为完美避开iOS7之前版本的幸运儿，我只能从某位前辈的口中得知那悲惨的岁月。作为那个年代唯一能与前端界面交互的手段就是`UIWebView`，先不说它自身的内存泄露缺陷，下面是一段前辈写过的代码：

    - (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType
	{
	    NSString * address = request.URL.absoluteString;
	    for (NSString * black in _blackList) {
	        if ([address containsString: black]) {
	            return NO;
	        }
	    }
	    for (NSString * event in _eventList) {
	        if ([address containsString: event]) {
	            SEL callback = NSSelectorFromString(_callbacks[event]);
	            [self performSelector: callback];
	            return [event containsString: @"shouldOpen=1"];
	        }
	    }
	    return YES;
	}
	
在那个年代，前辈的小伙伴们把前端事件的触发条件设置为链接跳转，然后通过链接中的关键字符来判断处理操作。为此，需要定义好些个数据集合来存储这些关键字符的处理操作。如果遇到应用和前端交换交互数据的时候，那一长串的参数字符全部拼接在请求地址里，想想也是醉了。另外的交互方法就是通过`stringByEvaluatingJavaScriptFromString`方法来执行`js`代码。

  JavaScriptCore
  `JavaScriptCore`是一套用来对`JS`代码进行解析和提供执行环境的[开源框架](https://github.com/phoboslab/JavaScriptCore-iOS)，极大的简化了我们的交互过程。下面从项目和`JS`代码相互调用的两个不同操作介绍其中相对应的方法

  > 项目调用JS代码

- **JSContext**
  一个`JSContext`对象是`JavaScript`运行的全局环境对象，它提供了代码运行和注册方法接口的服务。下面的代码就创建了一个`JSContext`对象，并且定义了一部分的`JS`代码加入到执行环境中
  
		let context = JSContext()
		context.evaluateScript(" var age = 22 ")
		context.evaluateScript(" var name = 'SindriLin' ")
		context.evaluateScript(" var birth = 1993-01-01 ")
		context.evaluateScript(" var createPerson = 
		function(age, name, birth) 
		{ 
			return {'age': age, 'name': name, 'birth': birth}
		} ")
		context.evaluateScript(" var codeDescription = 'The code create three value and a function to create a dictionary stored person information' ")
		
  此外，在`JS`代码执行过程中，可能会出现语法错误等多种错误，通过下面的代码可以对这些错误进行处理
  
		context?.exceptionHandler = { context, exception in 
		  print("Java Script Run Error: \(exception)")
		}

- **JSValue**
    `JSValue`是所有`JSContext`操作后返回的值，包装了几乎所有的数据类型，包括错误和`IMP`指针等。在`JSValue`类结构中存在多个`toXXXX`命名的方法转换成`iOS`数据类型以及`call`方法来调用方法。下面的代码从`JSContext`环境中获取已存在的部分变量，并且执行创建一个存储`person`信息的字典
    
        let age = context?.objectForKeyedSubscript("age")
        let name = context?.objectForKeyedSubscript("name")
        let birth = context?.objectForKeyedSubscript("birth")
        let createFunction = context?.objectForKeyedSubscript("createPerson")
        let codeDescription = context?.objectForKeyedSubscript("codeDescription")
        let person = createFunction.call(withArguments: [age.toInt32(), name.toString(), birth.toString()])
        
        let personInfo = "name: \(person["name"]) age: \(person["age"] and birth: \(person["birth"])"
        print("The javaScript code description: \(codeDescription.toString())")
        print("The created person \(personInfo) ")

  通过上面的例子，我们可以看到，只要了解到`JS`代码中我们需要调用的方法信息，通过`JSContext + JSValue`的方式我们就能轻松的在项目中调用前端界面的方法，而不再需要拼接长串参数字符通过链接地址传递给前端界面

  > JS调用项目代码

  `JavaScript`访问我们代码中的对象以及方法有两种方式：`Blocks`和`JSExport`。
  - **Blocks**
    自定义的`block`代码可以通过`JSContext`转换成`JS`代码中的函数指针调用，这里存在一个坑就是`Swift`中的闭包无法完成这样的类型转换，因此这种方式的操作流程在`Swift`中是这样的：`Closure` -> `block` -> `function pointer`。在闭包转成`block`的这一过程中，需要使用一个重要的关键符`@convention`
    
			let stringConvert: @convention(block) (String)->String = {
				let pinyin = NSMutableString(string: $0) as CFMutableString
				CFStringTransform(pinyin, nil, kCFStringTransformToLatin, false)
				CFStringTransform(pinyin, nil, kCFStringTransformStripCombiningMarks, false)
				return pinyin as String
			}      
			   
			let convertObjc = unsafeBitCast(stringConvert, to: AnyObject.self)
			context?.setObject(convertObjc, forKeyedSubscript: "convertFunc")
			let convertFunc = context?.objectForKeyedSubscript("convertFunc")
			print("林欣达的拼音是\(convertFunc.call(withArguments: ["林欣达"]).toString())")
		
这时候，只要前端在`JS`的按钮点击代码中调用`convertFunc()`这句代码就会执行这个`closure`中的代码。使用这种方式要注意由于闭包的捕获特性，有可能会导致你的`JSContext`对象被引用而无法被释放，使用`JSContext.current()`获取当前上下文来解决引用问题

  - **JSExport**
    在`JS`中调用`iOS`方法的时候，通过调用`JSExport`的派生协议方法来实现。所有派生协议的方法会自动提供给`JavaScript`代码使用，这个在下面的demo中可以看到 

  实战
  在本文demo中我写了一段`JS`代码，下面放出这段代码以及运行效果。其中要注意的是按钮的`onclik`表示按钮点击的响应事件：

      <!DOCTYPE html>
      <html>
          <head>
              <meta charset="UTF-8">
          </head>
          <body>
              <div style="margin-top: 20px">
                  <h2 align="center" style="color:ff0000">JS与iOS交互</h2>
                  <input type="button" value="点击后切换控制器的背景颜色" onclick="sindrilin.call()">
              </div>
              <div style="color:7BBDE5">
                  <br />
                  <br />
                  账户：
                  <input id="account" type="text">
                  <br />
                  密码：
                  <input id="password" type="password">
              </div>
              <div>
                  <input type="button" value="登录" onclick="login()">
              </div>
          
              <script>
              
                  var login = function()
                  {
                      account = document.getElementById("account")
                      password = document.getElementById("password")
                      var accountInfo = JSON.stringify({"account": account.value, "password": password.value});
                      sindrilin.login(accountInfo);
                  }
          
                  var alertFromIOS = function(message)
                  {
                          alert(message)
                  }
          
              </script>
          </body>
      </html>
      
  <span><img src="/images/JavaScript交互/2.jpeg" width="800"></span>
  首先我们需要加载这个`HTML`文件，然后获取代码运行的全局环境对象。基本上在所有的`HTML`格式文件中，获取环境对象的`keyPath`都是一样的：

		let jsPath = Bundle.main().pathForResource("interaction", ofType: "html")
		webView.loadRequest(URLRequest(url: URL(fileURLWithPath: jsPath!)))
		interactionContext = webView.value(forKeyPath: "documentView.webView.mainFrame.javaScriptContext") as? JSContext
		interactionContext?.exceptionHandler = {
			print("Interaction Error: \($1?.toString())")
		}
		
  对照`HTML`代码，最上面的按钮点击之后会调用一个`sindrilin.call()`的方法，这个方法最终要由我们的控制器来进行处理。我们可以把这个字符串分成类似`Target-Action`机制的两部分，前者`sindrilin`表示响应者，后面`call()`表示响应事件。其中`Target`的设置方式如下

	interactionContext?.setObject(self, forKeyedSubscript: "sindrilin")
	
  响应者已经有了，那么响应事件也要我们实现代码，这里就需要用到`JSExport`协议了。所有这种类似`Target-Action`的事件触发都会通过这个协议获取方法实现，因此我们需要自定义响应协议以及响应事件。对于有参数的方法我们需要用`@objc(name)`的方式给方法起`OC`式的方法名，才能保证能被正确调用响应：

		@objc protocol LXDInteractionExport: JSExport {
			func call()                                    ///响应sindrilin.call()
			@objc(login:) func login(accountInfo: String)  ///响应sindrilin.login(accountInfo)
		}
		 
		extension ViewController: LXDInteractionExport {
			func call() {
				print("call from html button clicked")
				view.backgroundColor = UIColor(red: CGFloat(arc4random() % 256) / 255, green: CGFloat(arc4random() % 256) / 255, blue: CGFloat(arc4random() % 256) / 255, alpha: 1)
			}
			 
			func login(accountInfo: String) {
				do {
					if let JSON: [String: String] = try JSONSerialization.jsonObject(with: accountInfo.data(using: String.Encoding.utf8)!, options: JSONSerialization.ReadingOptions()) as? [String: String] {
						print("JSON: \(JSON)")
						let alert = interactionContext?.objectForKeyedSubscript("alertFromIOS")
						let message = "The alert from javascript call\naccount: \(JSON["account"]) and password: \(JSON["password"])"
						_ = alert?.call(withArguments: [message])
					}
				} catch {
					print("Error: \(error)")
				}      
			}
		}
		
  用户在前端界面输入账户和密码信息之后点击登录就会调用`login(accountInfo: String)`方法，将用户名和密码拼凑成`JSON`字符串传递过来。在响应方法中我解析获取对应字段的用户信息，并且组转成新的字符串调用`JS`的弹窗函数弹出响应[demo下载](https://github.com/JustKeepRunning/LXDJavaScriptCoreDemo)
  <span><img src="/images/JavaScript交互/1.gif" width="800"></span>

  ​
  关注[iOS开发](http://www.jianshu.com/notebooks/2348850/latest)文集收看更多文章

  转载请注明原文作者和地址

  ​


