title: Hack Interface Inspector
date: 2016-04-21 20:38:11
categories: Development
tags: [Mac]

---

之前在 iOSRE 上看到[一篇文章] [1]，里面有讲到一个逆向 Mac app UI 的工具 [Interface Inspector] [2]，扫了一眼，尼玛神器啊，简直就是 Reveal 的 Mac 版。上个图感受一下它的淫威吧：

![Interface Inspector](/img/Hack_Interface_Inspector/ii.png)

这么好的工具，这么贵的价格，心痒了。好吧，凭借着[之前的破解经验] [3]，再来搞一次吧。

<!--more-->

### 0. 开始

首先从官网下载试用版，打开 app，直接弹出一个提示框：

![Alert](/img/Hack_Interface_Inspector/alert.png)

啊哈哈，最喜欢这么直接的打招呼了。

用 Hopper 加载完 *Interface Inspector* 后，可以直接奔着两个函数去了：

![Launching](/img/Hack_Interface_Inspector/launching.png)

### 1. 绕过签名校验

首先看一下 `[SMAppDelegate applicationWillFinishLaunching:]`，里面有这么一段：

![applicationWillFinishLaunching](/img/Hack_Interface_Inspector/will.png)

也就是说，app 启动完成之前，会调用 `[[NSBundle mainBundle] codeSignState]` 来检查 bundle 的签名是否合法（具体如何检查后面详谈）。如果不合法，会弹出一个提示框说 "Signature of the Interface Inspector is broken"，然后 app 就直接退出了。所以我们首先要绕过这个检查。

显然，`codeSignState` 这个函数并非官方 api，应该是作者自己添加的 *NSBundle category*。搜索了一下该函数名，没有找到。于是就去 app 的 `Frameworks` 目录下看看是不是由第三方库引入的。在这里有 3 个 framework：*DFeedback.framework*、*SMFoundation.framework*、*Sparkle.framework*，很明显，第一个肯定是收集用户 feedback 用的，第三个大家都知道是 update 模块，那就剩下第二个了。用 Hopper 加载 *SMFoundation*，果然搜索到了 `codeSignState` 这个函数。

在这个函数里，作者是用 `SecStaticCodeCheckValidityWithErrors` 这个官方 api 来检查签名的合法性的。具体用法可以查阅文档，这里只提一下比较重要的 2 个参数`staticCode` 和 `requirement`：`staticCode` 是待校验的 *code object*，`requirement` 则表示 `staticCode` 需要满足的校验条件。作者使用的校验条件是：`certificate leaf = H"0E1D40082148472951C6FB2DDCD8800D82629792"`。看到这里一开始我也蒙了，这是什么鬼。后来查阅了一下文档才明白这个用法。其实就是校验一下签名证书的叶子节点是不是 `H"0E1D40082148472951C6FB2DDCD8800D82629792"` 这个值，而这一串字符是签名证书的 *SHA1 FingerPrints*，由 40 个 HEX 字符组成，可以在自己的开发者证书里查到：

![Cert](/img/Hack_Interface_Inspector/cert.png)

也就是说，如果用别人的证书重新签名该 app，而又没有同时修改这个校验条件的话，那么最终的签名就是不合法的，app 就会闪退了。所以我们需要先把这个校验条件改掉。简单，直接把这个字符串的值改成自己开发者证书里面的 *SHA1 FingerPrints* 就好了。

OK，签名校验已经绕过去了，下面就可以随意地修改 app 然后重新签名了。

### 2. 破解 License 机制

`[SMAppDelegate applicationWillFinishLaunching:]` 已经没什么好看的了，接着看 `[SMAppDelegate applicationDidFinishLaunching:]`，里面有这么一段：

![applicationDidFinishLaunching](/img/Hack_Interface_Inspector/did.png)

太明显了，没有 license 的话就提示用户输入序列号进行激活。顺藤摸瓜，最终找到了这么个函数 `[SMLicenseManager verifyLicenseWithName:code:]`，它就是用来验证 license 是否合法的！只要我们对它进行破解，这样随便输入任意序列号就可以激活了，啊哈哈。

简单得不能再简单了，直接用 Hopper 的 *Modify* 功能修改成 `mov eax, 0x1` `ret` 即可，也就是直接返回 `YES` 通过验证。

修改完成，重新签名，运行 app，啊哦，闪退了！

### 3. 作者的鬼点子

看来还是有什么地方没改好。重新回去检查整个 `SMLicenseManager` 类，一个函数一个函数地检查，最终发现了这么个函数 `[SMLicenseManager load]`。作者在这里耍了一个小花招：

![Tricky](/img/Hack_Interface_Inspector/tricky.png)

是的，我们把 `[SMLicenseManager verifyLicenseWithName:code:]` 改成了总是返回 `YES`，但是代码走到这里撞墙了，*Test User* 验证之后也返回 `YES` 了，然后 app 直接 `terminate:` 了。也就是说，作者在这里放了一个本来就是非法的 license，正常验证的话肯定是返回 `NO` 的，就不会导致直接退出了。所以我们还需要把这里的判断条件改成 `XXX == NO`，这样就没有问题了。真是淘气的作者，藏得这么深。

好了，改好，重新签名，运行 app，随便输入 license 信息，啊哈哈，注册成功！

![Done](/img/Hack_Interface_Inspector/done.png)

-----

[1]: http://bbs.iosre.com/t/mac/3373
[2]: http://www.interface-inspector.com/
[3]: ../../../../2016/01/12/Hack_XtraFinder/
