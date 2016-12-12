title: Fix Bug for Interface Inspector on macOS Serria
date: 2016-11-08 16:32:57
categories: Development
tags: [Mac]

---

之前介绍过[如何破解 Interface Inspector] [1]，近来把系统升级到最新的 macOS Serria 之后，发现 Interface Inspector 不 work 了。启动倒是正常，但每次 attach app 时，总是提示无法 attach，查看 system log，发现有这么一个 error：

![log](/img/Fix_Bug_for_Interface_Inspector_on_macOS_Serria/log.png)

从 log 中看出，root cause 是 *mach_inject_bundle_stub* 去 load `___pthread_set_self` 时失败了，而这个函数本来应该是在 *libSystem.B.dylb* 这个系统库里面的。也就是说，macOS Serria 的 *libSystem.B.dylb* 已经不再有 `___pthread_set_self` 这个函数了。于是开始 google，最后在[一条 Twitter] [2] 上发现了咋回事，是的，`___pthread_set_self` 已经被替换成了 `_pthread_set_self`。

明白了咋回事，开始考虑怎么解决。

<!--more-->

## 方法 1：

修改 *mach_inject_bundle_stub* 的 load 指令，从旧版系统中 copy 一份老的 *libSystem.B.dylb*，然后让 *mach_inject_bundle_stub* 去 load 这个老的库。

## 方法 2：

还是修改 *mach_inject_bundle_stub*，把所有调用 `___pthread_set_self` 的地方改成调用 `_pthread_set_self`。

理论上这两种方法应该都可以的吧，但是考虑到自己逆向功力不够，所以就采用了如下的正向方法 XD。

# 方法 3：

我搜索了一下 *mach_inject_bundle_stub* 这个东西，发现原来是个 GitHub 上的[开源库] [3]，Interface Inspector 就是用的这个库。app 会在 `/Library/Frameworks/mach_inject_bundle.framework` 这里安装这个库，而 *mach_inject_bundle_stub* 就是这个库里面的一个子 bundle。有了 source code 就好办了，只需要修改 code，然后重新编译替换掉 *mach_inject_bundle_stub* 这个 bundle 就好了。

步骤：

### 1. fork it

### 2. 修改 code

![github](/img/Fix_Bug_for_Interface_Inspector_on_macOS_Serria/github.png)

### 3. 编译签名

注意，这里有个小细节，app 在 load *mach_inject_bundle_stub* 这个 bundle 时，是按照 bundle id `com.rentzsch.mach-inject-bundle-stub` 来找的，所以需要把工程文件里的 bundle id 改成跟原来的 bundle id 一样。

### 4. 替换到 *mach_inject_bundle.framework* 里面

### 5. 顺便提一个 pull request，老代码已经年久失修了 XD

完成之后，重新启动 app，可以正常 work 了！

-----

[1]: ../../../../2016/04/21/Hack_Interface_Inspector/
[2]: https://twitter.com/snielsen42/status/778405531383836674
[3]: https://github.com/wzqcongcong/mach_inject
