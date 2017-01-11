# Android热补丁方案 #

## 开发背景 ##

#### 一、正常开发流程 ####
<img src="http://img.blog.csdn.net/20170106092631597?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMTk3MTE4MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast" />
从流程来看，传统的开发流程存在很多弊端：

- 重新发布版本代价太大
- 用户下载安装成本太高
- BUG修复不及时，用户体验太差

#### 二、热修复开发流程 ####
<img src="http://img.blog.csdn.net/20170106092600468?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMTk3MTE4MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast" />

而热修复的开发流程显得更加灵活，优势很多：

- 无需重新发版，实时高效热修复
- 用户无感知修复，无需下载新的应用，代价小
- 修复成功率高，把损失降到最低


所以，热补丁技术成为了当前非常热门的 Android 开发技术，绝大部分的APP项目其实都需要一个动态化方案，来应对线上紧急 bug 修复发新版本的高成本问题。

#### 三、业界热门的热修复技术 ####

继插件化后，热补丁技术在2015年开始爆发，目前已经是非常热门的Android开发技术。其中比较著名的有淘宝的Dexposed、支付宝的AndFix以及Qzone的超级热补丁方案。热修复作为当下热门的技术，在业界内比较著名，并且在 Github 上 star 比较多的几个开源热更方案：

- 阿里巴巴的 AndFix、Dexposed
- 腾讯QQ空间 Qzone
- 微信的 Tinker

它们在原理各有不同，适用场景各异，到底采用哪种方案，是开发者比较头疼的问题。

## 热补丁方案分析 ##

首先，我们针对以上热门的热补丁方案总得来看下他们的功能及性能的优缺点（Dexposed一直没有兼容ART,这里就先不详细分析）：
<img src="http://img.blog.csdn.net/20170105160322336?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMTk3MTE4MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast" width="300px" />  

总的来说：   

AndFix 作为 native 解决方案，亮点就是即时生效，但是也有一些问题，首先面临的是稳定性与兼容性问题，更重要的是它无法实现类替换，它是需要大量额外的开发成本的；  

Qzone 方案可以做到发布产品功能，但是它主要问题是插桩带来Dalvik的性能问题，以及为了解决Art下内存地址问题而导致补丁包急速增大的；
 
Tinker 热补丁方案不仅支持类、So以及资源的替换，它还对 Android2.X－7.X 的全平台支持；利用 Tinker 我们不仅可以用做 bugfix，甚至可以替代功能的发布。

下面我们就来具体分析这几种热修复方案（目前主要研究Tinker，另外两个方案暂放）：

#### 一、Tinker ####

Tinker 是微信在2016年9月下旬开源出来的 Android 热补丁方案。Tinker开源之后的热度,维护程度，文档等状态都是比较良心的，目前已经release 八个版本出来了，并且支持代码、so库和资源更新，在热修复这种坑比较多的技术方案中，开源作者能活跃在第一线会给开发者带来很大的帮助。

Tinker 的方案来源 gradle 编译的 instant run 与 buck 编译的 exopackage。它们的思想都是全量替换新的 Dex。即我们完全使用了新的 Dex，那样既不出现 Art 地址错乱的问题，在 Dalvik 也无须插桩。

但是 instant run 针对的是编译期，它可以直接将最后生成的所有变化都直接拷到手机端。对于线上方案，这肯定是不可行的。所以当前核心问题是找到合适的，使补丁结果更小的差分算法。

经过微信程序猿不断尝试，最后决定基于 dex 的格式，自研出一种 Dexdiff 算法，它需要达到以下目标： 

- diff 结果小
- 合成过程占用内存小
- 支持删除、新增、修改 dex 中的 class

这里面主要的原理是深度利用原来 dex 中的信息，对于 dex 的每一个 section 做处理，这块在今天不再深入，后续再去研究。  

Tinker 的组成，它主要包括以下几部分： 

- gradle编译插件: `tinker-patch-gradle-plugin` 
- 核心sdk库: `tinker-android-lib`
- 非gradle编译用户的命令行版本**: `tinker-patch-cli.jar`

下面可以看一下Tinker的实现原理：
<img src="http://img.blog.csdn.net/20170106135116598?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMTk3MTE4MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast">

简单来说，在编译时通过新旧两个Dex生成差异path.dex。在运行时，将差异patch.dex重新跟原始安装包的旧Dex还原为新的Dex。这个过程可能比较耗费时间与内存，所以它是单独放在一个后台进程:patch中。为了补丁包尽量的小，微信自研了DexDiff算法，它深度利用Dex的格式来减少差异的大小。它的粒度是Dex格式的每一项，可以充分利用原本Dex的信息，而BsDiff的粒度是文件，AndFix/QZone的粒度为class。

然后我们来看看 Tinker 的补丁合成框架设计，它主要包括以下几部分：

- 补丁合成：在单独的 patch 进程工作，这里包括 dex，so 还有资源，主要完成补丁包的合成以及升级；
- 补丁加载：通过反射系统加载我们合成好的 dex，so 与资源；
- 监控回调：在合成与加载过程中，出现问题及时回调；
- 版本管理：支持补丁升级，甚至是多个补丁不停的切换。这里我们需要保证所有进程版本的一致性；
- 安全校验：在补丁合成还是加载，都需要有必要的安全校验。  

这个框架可以简单总结成下面的流程图：
<img src="http://img.blog.csdn.net/20170106092707655?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMTk3MTE4MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast" />
这里具体的加载流程不再分析，后续文章会详细介绍。 

最后简单对Tinker做下总结。 

优点：  

- 支持类、资源、so修复
- 兼容性处理的很好，全平台支持
- 由于不用插庄，所以性能损耗很小
- 完善的开发文档和官方技术支持
- gradle支持，再自己定义下可以一键打补丁包
- dexDiff算法使得补丁文件较小
- 扩展性良好，代码中处处为开发者留出开放接口，业界良心
- 支持多次补丁 

缺点： 

- 不支持及时生效，下发补丁需要重启生效，MultiDex方案决定的
- 占用ROM空间较大，这点空间在如今的手机大ROM下也不算个事
- 对加固支持不太好 

总结下来Tinker是一种基于单ClassLoader加载多dex方案的热补丁框架，兼容性做的比较好，功能强大，地精修补匠，带你无限刷新！
#### 二、AndFix ####
Andfix是阿里推出的开源框架，它在github的地址是：[AndFix][1]。

#### 三、Qzone ####

超级补丁Qzone技术基于DEX分包方案，使用了多DEX加载的原理，大致的过程就是：把BUG方法修复以后，放到一个单独的DEX里，插入到dexElements数组的最前面，让虚拟机去加载修复完后的方法。
## 学习文章 ##
大家可以点击下方的阅读原文，即可直接点击跳转。 

- [微信Tinker的一切都在这里，包括源码(一)][2]
- [Tinker Dexdiff算法解析][3]
- [从Instant run谈Android替换Application和动态加载机制][4]
- [Android动态加载基础 ClassLoader工作机制][5]
- [AndFix github][6]
- [Android N混合编译与对热补丁影响解析][7]
- [Qzone实现原理解析][8]

---------
[1]: https://github.com/alibaba/AndFix
[2]: http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=2649286384&idx=1&sn=f1aff31d6a567674759be476bcd12549&scene=4#wechat_redirect
[3]: https://www.zybuluo.com/dodola/note/554061
[4]: http://w4lle.github.io/2016/05/02/%E4%BB%8EInstant%20run%E8%B0%88Android%E6%9B%BF%E6%8D%A2Application%E5%92%8C%E5%8A%A8%E6%80%81%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6/
[5]: https://segmentfault.com/a/1190000004062880
[6]: https://github.com/alibaba/AndFix
[7]: http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=2649286341&idx=1&sn=054d595af6e824cbe4edd79427fc2706&scene=1&srcid=0811lK7OQal4gyEfWwUngZ9L#rd
[8]: https://mp.weixin.qq.com/s?__biz=MzI1MTA1MzM2Nw==&mid=400118620&idx=1&sn=b4fdd5055731290eef12ad0d17f39d4a



