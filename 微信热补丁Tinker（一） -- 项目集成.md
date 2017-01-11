# 微信热补丁Tinker -- 项目集成 #

在上篇文章[Android热补丁方案][7]中介绍了Tinker的原理框架，那么我们现在就从初级入门开始，学习一下它的项目集成，首先，我们来看看它官方Demo的使用，体验一下热修复。  

## 体验官方Demo ##

### 步骤： ###

- 下载 Sample  
 打开Tinker开源代码地址[Tinker][1]，把 Tinker 项目下载到本地后，使用 AS 导入项目 `tinker-sample-android`；
 
- 设置 tinkerId  
 打开 app 的 build.gradle文件，找到`getTinkerIdValue()`方法中：TINKER_ID : `gitSha()`，将`gitSha()`替换成自己想要的 tinkerId 命名规则；  
 
- 编译 Base APK  
 编译打包，此时 Tinker 会在工程的 `app/build/bakApk/` 目录下保存打包好的apk文件，先在手机上安装该 apk ； 
 
- 设置Base APK路径   
 找到刚才生成的 apk 文件，复制其完整文件名,在 app 的 build.gradle 文件，设置：
    `tinkerOldApkPath = "${bakPath}/<刚才生成的apk文件名>"`
  
- 修复 Bug  
 在 Base Apk的代码基础上修改代码修复 Bug；  
 
- 生成补丁  
 找到 Gradle 脚本中的tinker目录下 `tinkerPatchDebug`双击运行它将生成 debug 版的 patch (补丁) apk 文件，在 output/tinkerPatch/debug 下，文件为 `patch_signed_7zip.apk`；  
 
- 打入补丁  
 将 `patch_signed_7zip.apk` 这个文件拷贝到 Android 设备的 `ExternalStorageDirectory()` 路径下.文件的路径可以随意设定,只需在 `MainActivity` 中指明补丁 Apk 路径即可；随后点击 Demo 中 Load Patch 按钮，提示成功后，点击 Kill Self 结束当前进程，重启应用，即可看到所改的代码修复的 Bug 现象。

##项目集成  

###步骤：
（1） 在项目的 build.gradle 中，添加 `tinker-patch-gradle-plugin` 的依赖；

    buildscript {
        dependencies {
            classpath ('com.tencent.tinker:tinker-patch-gradle-plugin:${TINKER_VERSION}')
        }
    }
TINKER_VERSION 可以在项目 properties 中配置。  

（2） 在 app 的 gradle 文件 `app/build.gradle` ，我们需要添加 Tinker 的库依赖以及 apply tinke r的 gradle 插件；   

    dependencies {
        // tinker 热修复导入
        compile('com.tencent.tinker:tinker-android-lib:${TINKER_VERSION}') { changing = true }
        compile('com.tencent.tinker:tinker-android-anno:${TINKER_VERSION}') { changing = true }
        // 多dex支持
        compile 'com.android.support:multidex:1.0.1'
    }

    // apply tinker插件
    apply plugin: 'com.tencent.tinker.patch' 


（3） 参照官方 Sample 工程，把 gradle 文件中剩下的拷贝进来（需要的考，已有的不需要考）；   
在这里，我们可以定制一些自己的配置，比如 Tinkerid、PatchVersion 等，并且记得修改 `buildWithTinker()` 中 dex 的 loader 修改成自己的 Application 名。

    def createTinkerId(){
        return YourTinkerID;
    }

    def createPatchVersion(){
        return YourPatchVersion;
    }

有些具体的gradle配置的参数，大家可以参考[Tinker介入指南][3]中的表格，要是你英语够好的话，可以去看sample中的app/build.gradle的英文介绍。

（4） 拷贝官方 Sample 项目中的文件并配置；  

 - 拷贝 keyStore 文件夹；
 - 拷贝 keep_in_main_dex.txt 混淆文件且自定义其中的application，并在 proguard-rules 混淆文件加入保护：  
     `-keepattributes SourceFile,LineNumberTable`

 - 拷贝 java 文件，并作适当修改，如修改文件名、在 service 的 onPatchResult 函数中加入自己的逻辑

（5） 配置ApplicationLike代理
  
 `XXApplicationLike.java` 中的注解包名，用于自动生成 Applicaion，并在 Menifest 中给 Application 节点设置 name ，指向自动生成的 Application:  

     -public class YourApplication extends Application {
     +public class YourApplicationLike extends DefaultApplicationLike {
同时我们需要将 gradle 的 dex loader 中的 Application 改为新的 YourApplication: 

    dex {
        loader = ["com.tencent.tinker.loader.*",
            //warning, you must change it with your application
            "tinker.sample.android.YourApplication"
        ]       
    }
然后配置一下 ApplicationLike 中 Application 以及 Tinker 配置：  

    @DefaultLifeCycle(
        application = ".SampleApplication",                       //application类名
        flags = ShareConstants.TINKER_ENABLE_ALL,                 //tinkerFlags
        loaderClass = "com.tencent.tinker.loader.TinkerLoader",   //loaderClassName, 这里使用默认即可!
        loadVerifyFlag = false)                                   //tinkerLoadVerifyFlag
    public class SampleApplicationLike extends DefaultApplicationLike {
采用 Annotation 生成 Application ,需要将原来的 Application 类删掉。
将原本 Application 中的内容全部拷贝到 ApplicationLike.java 中。
 
（6） 编译和补丁  

- 每次编译或发包将安装包与mapping文件备份；
- 若有补丁包的需要，按自身需要修改你的代码、库文件等；
- 将备份的基准安装包与mapping文件输入到tinkerPatch的配置中；
- 运行tinkerPatchRelease，即可自动编译最新的安装包，并与输入基准包作差异，得到最终的补丁包。  

在打补丁时注意gralde中关于路径的修改：  

<img src="http://img.blog.csdn.net/20161205151920789" />
<img src="http://img.blog.csdn.net/20161205151933781" />

##Tinker 接入文档  

- 如何快速接入请参考[Tinker 接入指南][3]；
- 如何自定义类请参考[Tinker 自定义扩展][4]；
- Tinker的API预览请参考[Tinker API预览][5]；
- 其他常见问题，请参考[常见问题][6]；


---------

[1]: https://github.com/Tencent/tinker
[2]: http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=2649286306&idx=1&sn=d6b2865e033a99de60b2d4314c6e0a25&scene=1&srcid=0811AOttpqUnh1Wu5PYcXbnZ#rd
[3]:https://github.com/Tencent/tinker/wiki/Tinker-%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97
[4]:https://github.com/Tencent/tinker/wiki/Tinker-%E8%87%AA%E5%AE%9A%E4%B9%89%E6%89%A9%E5%B1%95
[5]:https://github.com/Tencent/tinker/wiki/Tinker-API%E6%A6%82%E8%A7%88
[6]:https://github.com/Tencent/tinker/wiki/Tinker-%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98
[7]:http://blog.csdn.net/qq_19711823/article/details/54138070