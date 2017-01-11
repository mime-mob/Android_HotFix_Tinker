#微信热补丁Tinker -- 补丁流程

作为开发人员，会用别人的框架是远远不够的，我们可以学习别人的设计思想、实验原理，积累知识，才能不断提升自己。今天这一章主要给大家介绍Tinker补丁流程，深入到代码中去探索Tinker。  

## 补丁流程 ##
在文章[Android热补丁方案][7]中介绍了Tinker的原理框架，那里只是简单介绍一下Tinker框架的补丁流程，这里我重新整理了一份Tinker补丁流程：

<img src="http://img.blog.csdn.net/20170111110637740?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMTk3MTE4MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast">

上面的流程图中，已经把Tinker的补丁流程和集成Tinker需要注意的逻辑都已经表示的很清楚了。需要注意的地方就是需要开发者自己编写下载过程，然后调用Tinker的Api传入补丁，以及补丁成功后重启应用的逻辑，这个开发者需要结合自己项目的需求来制定，我自己项目里面是通过熄屏监听来杀掉应用进程，进行重启。

在合成补丁流程中，总结了以下几点最重要的部分：

- 安全校验: 无论在补丁合成还是加载，我们都需要有必要的安全校验。 
- 版本管理: Tinker 支持补丁升级，甚至是多个补丁不停的切换。这里我们需要保证所有进程版本的一致性；
- 补丁加载: 如果通过反射系统加载我们合成好的 dex，so 与资源；
- 补丁合成: 这些都在单独的 patch 进程工作，这里包括 dex，so 还有资源，主要完成补丁包的合成以及升级；
- 监控回调: 在合成与加载过程中，出现问题及时回调；  

经过源码分析，总结出了以下Tinker补丁流程图（上图虚线部分）： 
<img src="http://img.blog.csdn.net/20170111110202142?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMTk3MTE4MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast">

以上图片中，标出了加载流程以及它实现的类和方法，本章也是主要着重这个流程分析。

## 源码跟踪 ##
###一、收到补丁
当补丁下载完成后，即调用下面的函数开始补丁： 

    /**
     * new patch file to install, try install them with :patch process
     *
     * @param context
     * @param patchLocation
     */
    public static void onReceiveUpgradePatch(Context context, String patchLocation) {
        Tinker.with(context).getPatchListener().onPatchReceived(patchLocation, true);
    }
此时的Tinker已经被初始化，在ApplicationLike中调用：  

    TinkerManager.installTinker(this);  

初始化的函数：  

    /**
     * install tinker with custom config, you must install tinker before you use their api
     * or you can just use {@link TinkerApplicationHelper}'s api
     *
     * @param applicationLike
     * @param loadReporter
     * @param patchReporter
     * @param listener
     * @param resultServiceClass
     * @param upgradePatchProcessor
     * @param repairPatchProcessor
     */
    public static void install(ApplicationLike applicationLike, LoadReporter loadReporter, PatchReporter patchReporter,
                               PatchListener listener, Class<? extends AbstractResultService> resultServiceClass,
                               AbstractPatch upgradePatchProcessor, AbstractPatch repairPatchProcessor) {

        Tinker tinker = new Tinker.Builder(applicationLike.getApplication())
            .tinkerFlags(applicationLike.getTinkerFlags())
            .loadReport(loadReporter)
            .listener(listener)
            .patchReporter(patchReporter)
            .tinkerLoadVerifyFlag(applicationLike.getTinkerLoadVerifyFlag()).build();

        Tinker.create(tinker);
        tinker.install(applicationLike.getTinkerResultIntent(), resultServiceClass, upgradePatchProcessor, repairPatchProcessor);
    }
传入的参数有： 

- ApplicationLike：应用的代理application
- LoadReporter：加载合成的包的报告类
- PatchReporter：打修复包过程中的报告类
- PatchListener：对修复包最开始的检查
- ResultService：从合成进程取合成结果监听
- UpgradePatchProcessor：生成一个新的patch合成包
- ReparePatchProcessor：修复上一次合成失败的修复包

看一下Tinker中核心的install方法：  

    /**
     * you must install tinker first!!
     *
     * @param intentResult
     * @param serviceClass
     * @param upgradePatch
     * @param repairPatch
     */
    public void install(Intent intentResult, Class<? extends AbstractResultService> serviceClass,
                        AbstractPatch upgradePatch, AbstractPatch repairPatch) {
        sInstalled = true;
        AbstractResultService.setResultServiceClass(serviceClass);
        TinkerPatchService.setPatchProcessor(upgradePatch, repairPatch);

        if (!isTinkerEnabled()) {
            TinkerLog.e(TAG, "tinker is disabled");
            return;
        }
        if (intentResult == null) {
            throw new TinkerRuntimeException("intentResult must not be null.");
        }
        tinkerLoadResult = new TinkerLoadResult();
        tinkerLoadResult.parseTinkerResult(getContext(), intentResult);
        //after load code set
        loadReporter.onLoadResult(patchDirectory, tinkerLoadResult.loadCode, tinkerLoadResult.costTime);

        if (!loaded) {
            TinkerLog.w(TAG, "tinker load fail!");
        }
    }
主要做了以下的工作：  

- 设置自定义的ResultService
- 设置自定义的UpgradePatch和ReparePatch
- 创建TinkerLoadResult调用parseTinkerResult(Context context, Intent intentResult)解析上次合成之后的信息：花费时间，返回值等。
- 调用LoaderReporter的onLoadResult方法，通知，加载结果。

### 二、补丁校验  

准备工作都做完了，下面就是校验补丁包操作了。  

调用第一步中的收到补丁函数，则PatchListener接收到补丁包：  

    /**
     * when we receive a patch, what would we do?
     * you can overwrite it
     *
     * @param path
     * @param isUpgrade
     * @return
     */
    @Override
    public int onPatchReceived(String path, boolean isUpgrade) {

        int returnCode = patchCheck(path, isUpgrade);

        if (returnCode == ShareConstants.ERROR_PATCH_OK) {
            TinkerPatchService.runPatchService(context, path, isUpgrade);
        } else {
            Tinker.with(context).getLoadReporter().onLoadPatchListenerReceiveFail(new 
                                           File(path), returnCode, isUpgrade);
        }
        return returnCode;
    }
DefaultPatchListener的onPatchReceived方法：

- 对补丁包进行检查；
- 如果修复包校验通过，则开启一个单独的进程合成全量包；
- 如果修复包不完整或者有其它问题，则调用LoadReporter的onLoadPatchListenerReceiveFail(File patchFile, int errorCode, boolean isUpgrade)方法通知，并将原有赋予errorCode传达，这里跟我上面总结的流程分析图中一样。

下面我们来看下上述的对补丁的具体校验：  

    protected int patchCheck(String path, boolean isUpgrade) {
        Tinker manager = Tinker.with(context);
        //check SharePreferences also
        if (!manager.isTinkerEnabled() || 
            !ShareTinkerInternals.isTinkerEnableWithSharedPreferences(context)) {
            return ShareConstants.ERROR_PATCH_DISABLE;
        }
        File file = new File(path);

        if (!file.isFile() || !file.exists() || file.length() == 0) {
            return ShareConstants.ERROR_PATCH_NOTEXIST;
        }

        //patch service can not send request
        if (manager.isPatchProcess()) {
            return ShareConstants.ERROR_PATCH_INSERVICE;
        }

        //if the patch service is running, pending
        if (TinkerServiceInternals.isTinkerPatchServiceRunning(context)) {
            return ShareConstants.ERROR_PATCH_RUNNING;
        }
        return ShareConstants.ERROR_PATCH_OK;
    }
这个是DefaultPatchListener的patchCheck方法，这里主要做了四件事：  

- 检查Tinker开关是否开启，需要打开
- 检查Patch文件是否存在，需要存在
- 检查是否是合成进程的操作，需要否
- 检查合成进程是否正在执行，需要否  

这里的合成进程指的是TinkerPatchService，这个需要检查校验完后才能去启动一个单独的进程去合成补丁。
除了DefaultPatchListener，我们还可以在SimplePatchListener中自定义一些补丁校验，官方Demo中还检查了以下方面：  

- 检查Rom空间
- 检查Crash次数
- 检查新旧补丁PatchVersion
- 检查补丁Patch条件是否符合  

经过所有的检查，当返回ERROR_PATCH_OK时才会去开启合成进程；若不是则通过LoadReporter的onLoadPatchListenerReceiveFail(File patchFile, int errorCode, boolean isUpgrade)方法通知，并将原有赋予errorCode传达。

###三、补丁加载  

当检查OK后，即开启  

    public static void runPatchService(Context context, String path, boolean isUpgradePatch) {
        Intent intent = new Intent(context, TinkerPatchService.class);
        intent.putExtra(PATCH_PATH_EXTRA, path);
        intent.putExtra(PATCH_NEW_EXTRA, isUpgradePatch);

        context.startService(intent);
    }
在TinkerPatchService中开启合成进程的服务。它是一个IntentSerivce，是在单独的线程中进行的操作。下面来看一下onHandleIntent中的操作。  

    protected void onHandleIntent(Intent intent) {
        final Context context = getApplicationContext();
        Tinker tinker = Tinker.with(context);
        tinker.getPatchReporter().onPatchServiceStart(intent);

        if (intent == null) {
            TinkerLog.e(TAG, "TinkerPatchService received a null intent, ignoring.");
            return;
        }
        String path = getPatchPathExtra(intent);
        if (path == null) {
            TinkerLog.e(TAG, "TinkerPatchService can't get the path extra, ignoring.");
            return;
        }
        File patchFile = new File(path);

        boolean isUpgradePatch = getPatchUpgradeExtra(intent);

        long begin = SystemClock.elapsedRealtime();
        boolean result;
        long cost;
        Throwable e = null;

        increasingPriority();
        PatchResult patchResult = new PatchResult();
        try {
            if (isUpgradePatch) {
                if (upgradePatchProcessor == null) {
                    throw new TinkerRuntimeException("upgradePatchProcessor is null.");
                }
                result = upgradePatchProcessor.tryPatch(context, path, patchResult);

            } else {
                //just recover from exist patch
                if (repairPatchProcessor == null) {
                    throw new TinkerRuntimeException("upgradePatchProcessor is null.");
                }
                result = repairPatchProcessor.tryPatch(context, path, patchResult);
            }
        } catch (Throwable throwable) {
            e = throwable;
            result = false;
            tinker.getPatchReporter().onPatchException(patchFile, e, isUpgradePatch);
        }

        cost = SystemClock.elapsedRealtime() - begin;
        tinker.getPatchReporter().
            onPatchResult(patchFile, result, cost, isUpgradePatch);

        patchResult.isSuccess = result;
        patchResult.isUpgradePatch = isUpgradePatch;
        patchResult.rawPatchFilePath = path;
        patchResult.costTime = cost;
        patchResult.e = e;

        AbstractResultService.runResultService(context, patchResult);
    }  

这里主要有几个操作：  

- 调用PatchRepoter的onPatchServiceStart(intent)，表示合成Patch的Service开启，用户可以自定义一些标记，用于跟踪合成进程；
- 调用increasingPriority()方法，让服务置于前台服务；
 - 第一，startForeground(notificationId, notification); 这个方法，可以让后台服务置于前台，就像音乐播放器的，播放服务一样，不会被系统杀死。
 - 第二，开启一个InnerService降低被杀死的概率。
- 调用tryPatch(context, path, patchResult)方法来执行合并操作，并将结果返回，具体的下面再分析这个合成方法，如果两个AbstractPatch为空，则会捕获异常。并且调用PatchReporter的onPatchException来通知。
- 修复成功后调用PatchReporter的onPatchResult()来通知，有花费时间等信息。
- 调用AbstactResultService的runResultService(Context context, PatchResult result)方法。也是在TinkerInstaller的Install的时候赋值。也是一个IntentService。这个Service会将补丁合成进程返回的结果返回给主进程，在单独的线程中执行。onHandleIntent中只是回调了runResultService(Context context, PatchResult result)方法。通过IntentService完成了进程间的通信。

最后我们先来预览一下合成方法：  

    public boolean tryPatch(Context context, String tempPatchPath, PatchResult patchResult) {
        Tinker manager = Tinker.with(context);

        final File patchFile = new File(tempPatchPath);

        if (!manager.isTinkerEnabled() || !ShareTinkerInternals.isTinkerEnableWithSharedPreferences(context)) {
            TinkerLog.e(TAG, "UpgradePatch tryPatch:patch is disabled, just return");
            return false;
        }

        if (!patchFile.isFile() || !patchFile.exists()) {
            TinkerLog.e(TAG, "UpgradePatch tryPatch:patch file is not found, just return");
            return false;
        }
        //check the signature, we should create a new checker
        ShareSecurityCheck signatureCheck = new ShareSecurityCheck(context);

        int returnCode = ShareTinkerInternals.checkTinkerPackage(context, manager.getTinkerFlags(), patchFile, signatureCheck);
        if (returnCode != ShareConstants.ERROR_PACKAGE_CHECK_OK) {
            TinkerLog.e(TAG, "UpgradePatch tryPatch:onPatchPackageCheckFail");
            manager.getPatchReporter().onPatchPackageCheckFail(patchFile, true, returnCode);
            return false;
        }

        patchResult.patchTinkerID = signatureCheck.getNewTinkerID();
        patchResult.baseTinkerID = signatureCheck.getTinkerID();

        //it is a new patch, so we should not find a exist
        SharePatchInfo oldInfo = manager.getTinkerLoadResultIfPresent().patchInfo;
        String patchMd5 = SharePatchFileUtil.getMD5(patchFile);

        if (patchMd5 == null) {
            TinkerLog.e(TAG, "UpgradePatch tryPatch:patch md5 is null, just return");
            return false;
        }

        //use md5 as version
        patchResult.patchVersion = patchMd5;

        SharePatchInfo newInfo;

        //already have patch
        if (oldInfo != null) {
            if (oldInfo.oldVersion == null || oldInfo.newVersion == null) {
                TinkerLog.e(TAG, "UpgradePatch tryPatch:onPatchInfoCorrupted");
                manager.getPatchReporter().onPatchInfoCorrupted(patchFile, oldInfo.oldVersion, oldInfo.newVersion, true);
                return false;
            }

            if (oldInfo.oldVersion.equals(patchMd5) || oldInfo.newVersion.equals(patchMd5)) {
                TinkerLog.e(TAG, "UpgradePatch tryPatch:onPatchVersionCheckFail");
                manager.getPatchReporter().onPatchVersionCheckFail(patchFile, oldInfo, patchMd5, true);
                return false;
            }
            newInfo = new SharePatchInfo(oldInfo.oldVersion, patchMd5);
        } else {
            newInfo = new SharePatchInfo("", patchMd5);
        }

        //check ok, we can real recover a new patch
        final String patchDirectory = manager.getPatchDirectory().getAbsolutePath();

        TinkerLog.i(TAG, "UpgradePatch tryPatch:dexDiffMd5:%s", patchMd5);

        final String patchName = SharePatchFileUtil.getPatchVersionDirectory(patchMd5);

        final String patchVersionDirectory = patchDirectory + "/" + patchName;

        TinkerLog.i(TAG, "UpgradePatch tryPatch:patchVersionDirectory:%s", patchVersionDirectory);

        //it is a new patch, we first delete if there is any files
        //don't delete dir for faster retry
        //SharePatchFileUtil.deleteDir(patchVersionDirectory);

        //copy file
        File destPatchFile = new File(patchVersionDirectory + "/" + SharePatchFileUtil.getPatchVersionFile(patchMd5));
        try {
            SharePatchFileUtil.copyFileUsingStream(patchFile, destPatchFile);
            TinkerLog.w(TAG, "UpgradePatch after %s size:%d, %s size:%d", patchFile.getAbsolutePath(), patchFile.length(),
                destPatchFile.getAbsolutePath(), destPatchFile.length());
        } catch (IOException e) {
            //e.printStackTrace();
            TinkerLog.e(TAG, "UpgradePatch tryPatch:copy patch file fail from %s to %s", patchFile.getPath(), destPatchFile.getPath());
            manager.getPatchReporter().onPatchTypeExtractFail(patchFile, destPatchFile, patchFile.getName(), ShareConstants.TYPE_PATCH_FILE, true);
            return false;
        }

        //we use destPatchFile instead of patchFile, because patchFile may be deleted during the patch process
        if (!DexDiffPatchInternal.tryRecoverDexFiles(manager, signatureCheck, context, patchVersionDirectory, destPatchFile, true)) {
            TinkerLog.e(TAG, "UpgradePatch tryPatch:new patch recover, try patch dex failed");
            return false;
        }

        if (!BsDiffPatchInternal.tryRecoverLibraryFiles(manager, signatureCheck, context, patchVersionDirectory, destPatchFile, true)) {
            TinkerLog.e(TAG, "UpgradePatch tryPatch:new patch recover, try patch library failed");
            return false;
        }

        if (!ResDiffPatchInternal.tryRecoverResourceFiles(manager, signatureCheck, context, patchVersionDirectory, destPatchFile, true)) {
            TinkerLog.e(TAG, "UpgradePatch tryPatch:new patch recover, try patch resource failed");
            return false;
        }

        final File patchInfoFile = manager.getPatchInfoFile();

        if (!SharePatchInfo.rewritePatchInfoFileWithLock(patchInfoFile, newInfo, SharePatchFileUtil.getPatchInfoLockFile(patchDirectory))) {
            TinkerLog.e(TAG, "UpgradePatch tryPatch:new patch recover, rewrite patch info failed");
            manager.getPatchReporter().onPatchInfoCorrupted(patchFile, newInfo.oldVersion, newInfo.newVersion, true);
            return false;
        }


        TinkerLog.w(TAG, "UpgradePatch tryPatch: done, it is ok");
        return true;
    }

看到这么长的方法是不是很慌，不要慌，Tinker开发者写代码非常有条理性，从前面的代码分析中可以发现了，好了，下面来分析一下这段代码：  

- 首先，还是常规的Tinker开关和Patch文件检查，随后ShareTinkerInternals的静态方法checkTinkerPackage(Context context, int tinkerFlag，File patchFile, ShareSecurityCheck securityCheck)传入了ShareSecurityCheck，主要做了一下检查操作，如果检查失败通过PatchReporter抛出onPatchPackageCheckFail：
 - 签名检查，ShareSecurityCheck是检查签名的类，里面封装了签名检查的方法；
 - TinkerId检查
 - 检查Tinker开关类型，dex、resource、libriry 的支持性。  
 - SharePatchInfo，存储了修复包的版本信息，有oldVersion和newVersion，newVersion使用的是修复包的md5值。
 - oldInfo何时会存在呢？加载成功过一次，也修复成功了，再次执行合成的时候，如果下载的包还是之前的包，则会报告onPatchVersionCheckFail。如果是新的修复包，则会把 oldVersion赋值给  SharePatchInfo(String oldVer, String newVew)中的ondVersion。到此为止，所有的检查已经完成。
- 拷贝修复包到data/data目录，从下载目录文件通过流读出写入data/data目录。
- DexDiff合成dex、BsDiff合成library、ResDiff合成res。
- 拷贝SharePatchInfo到PatchInfoFile中，PatchInfoFile在TinkerInstaller的install方法中初始化。

到这里，整个Tinker修复的流程已经走完，虽然感觉很复杂，但是条理很清晰，安全措施很齐全，也提现了Tinker的稳定性，最后合成补丁到下一篇研究。





---------

[1]: http://blog.csdn.net/qq_19711823/article/details/54138070