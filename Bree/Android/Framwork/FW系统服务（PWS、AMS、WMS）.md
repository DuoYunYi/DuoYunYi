


[Framework学习（三）之PMS、AMS、WMS_ams pms-CSDN博客](https://blog.csdn.net/ljx1400052550/article/details/115518631)



[__Yvan-CSDN博客](https://blog.csdn.net/u010687761?type=blog)




# 一、PWM（PackageManagerService）

## 1.简介

PMS（PackageManagerService）是Android提供的包管理系统服务，它用来管理所有的包信息，包括应用安装、卸载、更新以及解析AndroidManifest.xml。

PMS对apk的解析最主要的就是去扫描/data/app和/system/app目录下的apk文件，找到apk包中的AndroidManifest.xml，然后解析AndroidManifest.xml的信息保存到系统内存中，这样AMS在需要应用数据时，就能找到PMS快速的从内存中拿到相关信息。

在开机启动的耗时中70%的都是在PMS的解析上，如果说优化开机启动速度，不妨从PMS入手。


## 2.PWM启动

在Android系统所有的核心服务都会经过SystemServer启动，PMS也不例外，SystemServer会在手机开机时启动运行。
我们知道PMS是在SystemServer进程中被启动，下面我们来看看具体是怎么启动的PMS:

/frameworks/base/services/java/com/android/server/SystemServer.java

```
348      public static void main(String[] args) {
349          new SystemServer().run();
350      }
......
370      private void run() {
......
507          // Start services.
508          try {
509              traceBeginAndSlog("StartServices");
510              startBootstrapServices();
511              startCoreServices();
512              startOtherServices();
513              SystemServerInitThreadPool.shutdown();
514          } catch (Throwable ex) {
515              Slog.e("System", "******************************************");
516              Slog.e("System", "************ Failure starting system services", ex);
517              throw ex;
518          } finally {
519              traceEnd();
520          }
......
543      }

......
623      private void startBootstrapServices() {
......
734          try {
735              Watchdog.getInstance().pauseWatchingCurrentThread("packagemanagermain");
736              mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
737                      mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
738          } finally {
739              Watchdog.getInstance().resumeWatchingCurrentThread("packagemanagermain");
740          }
741          mFirstBoot = mPackageManagerService.isFirstBoot();
742          mPackageManager = mSystemContext.getPackageManager();
......
818      }

```


/frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

```
2283      public static PackageManagerService main(Context context, Installer installer,
2284              boolean factoryTest, boolean onlyCore) {
2285          // Self-check for initial settings.
2286          PackageManagerServiceCompilerMapping.checkProperties();
2287  
2288          PackageManagerService m = new PackageManagerService(context, installer,
2289                  factoryTest, onlyCore);
2290          m.enableSystemUserPackages();
2291          ServiceManager.addService("package", m);
2292          final PackageManagerNative pmn = m.new PackageManagerNative();
2293          ServiceManager.addService("package_native", pmn);
2294          return m;
2295      }

```

Zygote进程调用SystemServer.java类中main()方法，在run()方法中的startBootstrapServices()方法对PMS等核心服务进行初始化，调用其main()方法进行创建对应的服务，并将PMS服务添加到ServiceManager中（AMS也是一样的操作）进行服务的管理。

ServiceManager只提供了addService()和getService()方法，当app进程需要获取到对应的系统服务，都会通过ServiceManager拿到相应服务的Binder代理，使用Binder通信获取数据。


/frameworks/base/core/java/android/app/ActivityThread.java

```
2131      @UnsupportedAppUsage
2132      public static IPackageManager getPackageManager() {
2133          if (sPackageManager != null) {
2134              //Slog.v("PackageManager", "returning cur default = " + sPackageManager);
2135              return sPackageManager;
2136          }
2137          IBinder b = ServiceManager.getService("package");
2138          //Slog.v("PackageManager", "default service binder = " + b);
2139          sPackageManager = IPackageManager.Stub.asInterface(b);
2140          //Slog.v("PackageManager", "default service = " + sPackageManager);
2141          return sPackageManager;
2142      }

```

## 3.PWM解析

PWM解析主要做了三件事：

```
`1. 遍历/data/app和/system/app文件夹，找到apk文件`  
`2. 解压apk文件`  
`3. dom解析AndroidManifest.xml文件，将xml信息存储起来提供给AMS使用`
```

### （1）遍历/data/app的文件夹

/frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

```
......
		 // /data/app目录
663      private static final File sAppInstallDir =
664              new File(Environment.getDataDirectory(), "app");
......
2380      public PackageManagerService(Context context, Installer installer,
2381              boolean factoryTest, boolean onlyCore) {
......
				  // /system/app 目录
2667              final File systemAppDir = new File(Environment.getRootDirectory(), "app");
				  // 扫描/system/app 目录下的apk文件
2668              scanDirTracedLI(systemAppDir,
2669                      mDefParseFlags
2670                      | PackageParser.PARSE_IS_SYSTEM_DIR,
2671                      scanFlags
2672                      | SCAN_AS_SYSTEM,
2673                      0);
......
				      // 扫描/data/app 目录下的apk文件
2914                  scanDirTracedLI(sAppInstallDir, 0, scanFlags | SCAN_REQUIRE_KNOWN, 0);
......
3372      }
......
9003      private void scanDirTracedLI(File scanDir, final int parseFlags, int scanFlags, long currentTime) {
9004          Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "scanDir [" + scanDir.getAbsolutePath() + "]");
9005          try {
9006              scanDirLI(scanDir, parseFlags, scanFlags, currentTime);
9007          } finally {
9008              Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
9009          }
9010      }
9011  
9012      private void scanDirLI(File scanDir, int parseFlags, int scanFlags, long currentTime) {
9013          final File[] files = scanDir.listFiles();
......
9023          try (ParallelPackageParser parallelPackageParser = new ParallelPackageParser(
9024                  mSeparateProcesses, mOnlyCore, mMetrics, mCacheDir,
9025                  mParallelPackageParserCallback)) {
9026              // Submit files for parsing in parallel
9027              int fileCount = 0;
9028              for (File file : files) {
9029                  final boolean isPackage = (isApkFile(file) || file.isDirectory())
9030                          && !PackageInstallerService.isStageName(file.getName());
9031                  if (!isPackage) {
9032                      // Ignore entries which are not packages
9033                      continue;
9034                  }
9035                  parallelPackageParser.submit(file, parseFlags);
9036                  fileCount++;
9037              }
......
9076          }
9077      }

```


我们看到这里遍历`/data/app`和`/system/app`文件夹，找到apk文件，然后通过`submit()`方法进行了apk的解析。我们继续往下看`submit()`方法

### （2）解压apk文件

/frameworks/base/services/core/java/com/android/server/pm/ParallelPackageParser.java

```
100      /**
101       * Submits the file for parsing
102       * @param scanFile file to scan
103       * @param parseFlags parse falgs
104       */
105      public void submit(File scanFile, int parseFlags) {
106          mService.submit(() -> {
107              ParseResult pr = new ParseResult();
108              Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "parallel parsePackage [" + scanFile + "]");
109              try {
110                  PackageParser pp = new PackageParser();
111                  pp.setSeparateProcesses(mSeparateProcesses);
112                  pp.setOnlyCoreApps(mOnlyCore);
113                  pp.setDisplayMetrics(mMetrics);
114                  pp.setCacheDir(mCacheDir);
115                  pp.setCallback(mPackageParserCallback);
					 // 需要解析的apk文件路径
116                  pr.scanFile = scanFile;
					 // 通过PackageParser对apk进行解析
117                  pr.pkg = parsePackage(pp, scanFile, parseFlags);
118              } catch (Throwable e) {
119                  pr.throwable = e;
120              } finally {
121                  Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
122              }
123              try {
124                  mQueue.put(pr);
125              } catch (InterruptedException e) {
126                  Thread.currentThread().interrupt();
127                  // Propagate result to callers of take().
128                  // This is helpful to prevent main thread from getting stuck waiting on
129                  // ParallelPackageParser to finish in case of interruption
130                  mInterruptedInThread = Thread.currentThread().getName();
131              }
132          });
133      }
134  
135      @VisibleForTesting
136      protected PackageParser.Package parsePackage(PackageParser packageParser, File scanFile,
137              int parseFlags) throws PackageParser.PackageParserException {
138          return packageParser.parsePackage(scanFile, parseFlags, true /* useCaches */);
139      }

```


将上面找到的apk文件路径传入PackageParser对象的parsePackage()进行apk的解析。这里要注意：在不同的系统源码版本解析的方式也不相同，在6.0、7.0、8.0版本启动解析的方式还是直接解析的，但在10.0版本开始使用线程池放到子线程去解析，加快了手机启动速度。

### （3）dom解析AndroidManifest.xml文件

/frameworks/base/core/java/android/content/pm/PackageParser.java

```
1011      @UnsupportedAppUsage
1012      public Package parsePackage(File packageFile, int flags, boolean useCaches)
1013              throws PackageParserException {
1014          Package parsed = useCaches ? getCachedResult(packageFile, flags) : null;
1015          if (parsed != null) {
				  // 直接返回缓存
1016              return parsed;
1017          }
1018  
1019          long parseTime = LOG_PARSE_TIMINGS ? SystemClock.uptimeMillis() : 0;
			  // apk文件非目录，执行parseMonolithicPackage()
1020          if (packageFile.isDirectory()) {
1021              parsed = parseClusterPackage(packageFile, flags);
1022          } else {
1023              parsed = parseMonolithicPackage(packageFile, flags);
1024          }
......
1036          return parsed;
1037      }
......
1289      @Deprecated
1290      @UnsupportedAppUsage
1291      public Package parseMonolithicPackage(File apkFile, int flags) throws PackageParserException {
......
1300          final SplitAssetLoader assetLoader = new DefaultSplitAssetLoader(lite, flags);
1301          try {
				  // apk解析方法parseBaseApk()
1302              final Package pkg = parseBaseApk(apkFile, assetLoader.getBaseAssetManager(), flags);
1303              pkg.setCodePath(apkFile.getCanonicalPath());
1304              pkg.setUse32bitAbi(lite.use32bitAbi);
1305              return pkg;
1306          } catch (IOException e) {
1307              throw new PackageParserException(INSTALL_PARSE_FAILED_UNEXPECTED_EXCEPTION,
1308                      "Failed to get path: " + apkFile, e);
1309          } finally {
1310              IoUtils.closeQuietly(assetLoader);
1311          }
1312      }
1313  
1314      private Package parseBaseApk(File apkFile, AssetManager assets, int flags)
1315              throws PackageParserException {
1316          final String apkPath = apkFile.getAbsolutePath();
......
1328  		  // 开始 dom 解析 AndroidManifest.xml
1329          XmlResourceParser parser = null;
1330          try {
1331              final int cookie = assets.findCookieForPath(apkPath);
1332              if (cookie == 0) {
1333                  throw new PackageParserException(INSTALL_PARSE_FAILED_BAD_MANIFEST,
1334                          "Failed adding asset path: " + apkPath);
1335              }
1336              parser = assets.openXmlResourceParser(cookie, ANDROID_MANIFEST_FILENAME);
......
1340              final Package pkg = parseBaseApk(apkPath, res, parser, flags, outError);
......
1351              return pkg;
1352  
1353          } catch (PackageParserException e) {
1354              throw e;
1355          } catch (Exception e) {
1356              throw new PackageParserException(INSTALL_PARSE_FAILED_UNEXPECTED_EXCEPTION,
1357                      "Failed to read manifest from " + apkPath, e);
1358          } finally {
1359              IoUtils.closeQuietly(parser);
1360          }
1361      }
......

1913      @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.P, trackingBug = 115609023)
1914      private Package parseBaseApk(String apkPath, Resources res, XmlResourceParser parser, int flags,
1915              String[] outError) throws XmlPullParserException, IOException {
1916          final String splitName;
1917          final String pkgName;
1918  
1919          try {
1920              Pair<String, String> packageSplit = parsePackageSplitNames(parser, parser);
1921              pkgName = packageSplit.first; // 包名
1922              splitName = packageSplit.second;
......
1932          }
......
			  // 将解析的信息（四大组件、权限等）存储到Package
1943          final Package pkg = new Package(pkgName);
.....
1975          return parseBaseApkCommon(pkg, null, res, parser, flags, outError);
1976      }
......
6403      /**
6404       * Representation of a full package parsed from APK files on disk. A package
6405       * consists of a single base APK, and zero or more split APKs.
6406       */
6407      public final static class Package implements Parcelable {
......
			  // 包名
6409          @UnsupportedAppUsage
6410          public String packageName; 
			  // 应用信息
6453          @UnsupportedAppUsage
6454          public ApplicationInfo applicationInfo = new ApplicationInfo();
6455  
			  // 权限相关信息
6456          @UnsupportedAppUsage
6457          public final ArrayList<Permission> permissions = new ArrayList<Permission>(0);
6458          @UnsupportedAppUsage
6459          public final ArrayList<PermissionGroup> permissionGroups = new ArrayList<PermissionGroup>(0);
			  // 四大组件相关信息
6460          @UnsupportedAppUsage
6461          public final ArrayList<Activity> activities = new ArrayList<Activity>(0);
6462          @UnsupportedAppUsage
6463          public final ArrayList<Activity> receivers = new ArrayList<Activity>(0);
6464          @UnsupportedAppUsage
6465          public final ArrayList<Provider> providers = new ArrayList<Provider>(0);
6466          @UnsupportedAppUsage
6467          public final ArrayList<Service> services = new ArrayList<Service>(0);
......
7499      }

```


到这里解析三步流程已经完成：

通过遍历/data/app或/system/app文件夹找到apk文件路径；
把找到的路径传入PackageParser对象的parsePackage()方法对apk的AndroidManifest.xml进行dom解析；
然后根据不同标签解析信息存储到Package类的对应字段并缓存到内存中，如：四大组件、权限等信息。方便后续AMS直接从PMS的Package缓存中获取使用。

![[PWM流程图.png]]

## 4.总结

PMS是包管理系统服务，用来管理所有的包信息，包括应用安装、卸载、更新以及解析AndroidManifest.xml。
手机开机后，它会遍历设备上/data/app/和/system/app/目录下的所有apk文件，通过解析所有安装应用的AndroidManifest.xml，将xml中的数据（应用信息、权限、四大组件等）信息都缓存到内存中，后续提供给AMS等服务使用。


PMS的整体流程：

手机开机，内核进程启动init进程，init进程启动SeriviceManager进程和启动Zygote进程，Zygote进程启动SystemServer，SystemServer进程启动AMS、PMS，并注册到ServiceManager。
PMS被SystemServer初始化后，开始扫描/data/app/和/system/app/目录下的所有apk文件，获取每个apk文件的AndroidManifest.xml文件，并进行dom解析。
解析AndroidManifest.xml将应用信息、权限、四大组件等数据信息转换为Java Bean缓存到内存中。
当AMS需要获取apk数据信息时，通过ServiceManager获取到PMS的Binder代理通过Binder通信获取。

![[Pasted image 20250612195837.png]]


# 二、AMS（ActivityManagerService）

[【Android Framework系列】第5章 AMS启动流程_android ams启动流程-CSDN博客](https://blog.csdn.net/u010687761/article/details/131438970?spm=1001.2014.3001.5502)

## 1.简介

AMS（Activity Manager Service）是Android中最核心的服务，管理着四大组件的启动、切换、调度及应用进程的管理和调度等工作。
AMS通过使用一些系统资源和数据结构（如进程、任务栈、记录四大组件生命周期的状态机等）来管理Activity、Service、Broadcast、ContentProvider四大组件的生命周期管理。其职责与操作系统中的进程管理和调度模块相类似，因此它在Android中非常重要。

## 2.AMS启动流程

PMS和AMS的启动都是在SystemServer进程，系统启动后Zygote进程第一个fork出SystemServer进程，进入到SystemServer:main()->run()->startBootstrapServices() 启动引导服务，进而完成PMS和AMS等核心服务的启动。
在Android系统所有的核心服务都会经过SystemServer启动，AMS和PMS都是一样。SystemServer会在手机开机时启动运行。

### （1）AMS被启动

/frameworks/base/services/java/com/android/server/SystemServer.java

```
348      public static void main(String[] args) {
349          new SystemServer().run();
350      }
......
370      private void run() {
......
507          // Start services.
508          try {
509              traceBeginAndSlog("StartServices");
				 // 引导服务
510              startBootstrapServices();
				 // 核心服务
511              startCoreServices();
				 // 其他服务
512              startOtherServices();
513              SystemServerInitThreadPool.shutdown();
514          } catch (Throwable ex) {
515              Slog.e("System", "******************************************");
516              Slog.e("System", "************ Failure starting system services", ex);
517              throw ex;
518          } finally {
519              traceEnd();
520          }
......
543      }

......
623      private void startBootstrapServices() {
......
658          ActivityTaskManagerService atm = mSystemServiceManager.startService(
659                  ActivityTaskManagerService.Lifecycle.class).getService();
660          mActivityManagerService = ActivityManagerService.Lifecycle.startService(
661                  mSystemServiceManager, atm);
662          mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
663          mActivityManagerService.setInstaller(installer);
664          mWindowManagerGlobalLock = atm.getGlobalLock();
......
779          mActivityManagerService.setSystemProcess();
......
818      }
......
874      /**
875       * Starts a miscellaneous grab bag of stuff that has yet to be refactored and organized.
876       */
877      private void startOtherServices() {
......
			  // 安装ContentProvider
982           mActivityManagerService.installSystemProviders();
......
1023          wm = WindowManagerService.main(context, inputManager, !mFirstBoot, mOnlyCore,
1024                      new PhoneWindowManager(), mActivityManagerService.mActivityTaskManager);
1025          ServiceManager.addService(Context.WINDOW_SERVICE, wm, /* allowIsolated= */ false,
1026                      DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PROTO);
1027          ServiceManager.addService(Context.INPUT_SERVICE, inputManager,
1028                      /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL);
			  // WMS与AMS/ATMS关联起来
1032          mActivityManagerService.setWindowManager(wm);
......
			  // 所有的服务已经准备就绪
2035          mActivityManagerService.systemReady(() -> {
......
				  // 启动阶段500
2038              mSystemServiceManager.startBootPhase(
2039                      SystemService.PHASE_ACTIVITY_MANAGER_READY);
......
2042              try {
					  // 监测Native Crash
2043                  mActivityManagerService.startObservingNativeCrashes();
2044              } catch (Throwable e) {
2045                  reportWtf("observing native crashes", e);
2046              }
......
2051              final String WEBVIEW_PREPARATION = "WebViewFactoryPreparation";
2052              Future<?> webviewPrep = null;
2053              if (!mOnlyCore && mWebViewUpdateService != null) {
2054                  webviewPrep = SystemServerInitThreadPool.get().submit(() -> {
2055                      Slog.i(TAG, WEBVIEW_PREPARATION);
2056                      TimingsTraceLog traceLog = new TimingsTraceLog(
2057                              SYSTEM_SERVER_TIMING_ASYNC_TAG, Trace.TRACE_TAG_SYSTEM_SERVER);
2058                      traceLog.traceBegin(WEBVIEW_PREPARATION);
2059                      ConcurrentUtils.waitForFutureNoInterrupt(mZygotePreload, "Zygote preload");
2060                      mZygotePreload = null;
						  // 启动WebView相关
2061                      mWebViewUpdateService.prepareWebViewInSystemServer();
2062                      traceLog.traceEnd();
2063                  }, WEBVIEW_PREPARATION);
2064              }
......
2073              try {
					  // 启动SystemUi
2074                  startSystemUi(context, windowManagerF);
2075              } catch (Throwable e) {
2076                  reportWtf("starting System UI", e);
2077              }
......
				  // 启动阶段600
2154              mSystemServiceManager.startBootPhase(
2155                      SystemService.PHASE_THIRD_PARTY_APPS_CAN_START);

......
2249          }, BOOT_TIMINGS_TRACE_LOG);
2250      }

```

SystemServer将AMS启动并初始化主要两大阶段：
#### **第一阶段：startBootstrapServices()方法中启动引导服务：**

创建ActivityTaskManagerService(ATMS)对象，用于管理Activity
创建AMS对象，并启动服务
将AMS所在的系统进程SystemServer，纳入到AMS的进程管理体系中
setSystemProcess()：将framewok-res.apk信息加入到SystemServer进程的LoadedApk中，构建SystemServe进程的ProcessRecord，保存到AMS中，以便AMS进程统一管理
#### **第二阶段：startOtherServices()方法中启动其他服务：**

为系统进程安装ContentProvider对象
installSystemProviders()：安装SystemServer进程中的SettingsProvider.apk，
初始化WMS，并关联AMS及ATMS两个服务。
setWindowManager()方法将AMS与WMS关联起来，通过ATMS来管理Activity
AMS启动完成，通知服务或应用完成后续工作，或直接启动新进程。
AMS.systemReady()：许多服务或应用进程必须等待AMS完成启动工作后，才能启动或进行一些后续工作，AMS就是在SystemReady()中，通知或启动这些等待的服务和应用进程，例如启动桌面等。


我们看到在第一阶段中，对AMS和ATMS进行创建时都是调用的startService()方法。其中AMS创建调用ActivityManagerService.Lifecycle.startService()，而ATMS创建是mSystemServiceManager.startService()实际上mSystemServiceManager内部也是调用的ActivityManagerService.Lifecycle.startService()对服务进行创建。下面我们通过AMS的Lifecycle来看看AMS的初始化。

### （2）AMS初始化

/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

```
......
2209      public static final class Lifecycle extends SystemService {
2210          private final ActivityManagerService mService;
2211          private static ActivityTaskManagerService sAtm;
2212  
2213          public Lifecycle(Context context) {
2214              super(context);
2215              mService = new ActivityManagerService(context, sAtm);
2216          }
2217  
2218          public static ActivityManagerService startService(
2219                  SystemServiceManager ssm, ActivityTaskManagerService atm) {
2220              sAtm = atm;
2221              return ssm.startService(ActivityManagerService.Lifecycle.class).getService();
2222          }
2223  
2224          @Override
2225          public void onStart() {
2226              mService.start();
2227          }
2228  
2229          @Override
2230          public void onBootPhase(int phase) {
2231              mService.mBootPhase = phase;
2232              if (phase == PHASE_SYSTEM_SERVICES_READY) {
2233                  mService.mBatteryStatsService.systemServicesReady();
2234                  mService.mServices.systemServicesReady();
2235              } else if (phase == PHASE_ACTIVITY_MANAGER_READY) {
2236                  mService.startBroadcastObservers();
2237              } else if (phase == PHASE_THIRD_PARTY_APPS_CAN_START) {
2238                  mService.mPackageWatchdog.onPackagesReady();
2239              }
2240          }
2241  
2242          @Override
2243          public void onCleanupUser(int userId) {
2244              mService.mBatteryStatsService.onCleanupUser(userId);
2245          }
2246  
2247          public ActivityManagerService getService() {
2248              return mService;
2249          }
2250      }
......

```

AMS 创建流程简述：
1.SystemServer：依次调用main()、run()、startBootstrapServices()，再调用Lifecyle的startService()方法；
2.Lifecyle：startService()方法中调用SystemServiceManager的startService()方法，并将Lifecyle.class传入；
3.SystemServiceManager：startService()方法通过反射调用Lifecyle的构造方法，生成Lifecyle对象；
4.Lifecyle：构造方法中调用AMS的构造方法创建AMS对象，并通过getService()方法返回AMS对象。


AMS的构造函数：
/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

AMS的构造方法主要是在做一些初始化的相关操作:

1)保存了自己的运行环境的Context和ActivityThread
2)AMS负责调度四大组件，初始化broadcast,service和contentProvider相关的变量，负责三大大组件的(service、broadcast、provider)管理和调度(activity移到了ActivityTaskManagerService中，但此处也绑定了ActivityTaskManagerService对象)，
3)接着初始化了电量统计服务，监控内存、电池、权限(可以了解下appops.xml)以及性能相关的对象或变量。mLowMemDetector、mBatteryStatsService、mProcessStats、mAppOpsService、mProcessCpuThread等。创建了系统的第一个用户，初始化了基本的配置信息
4)创建了Activity调度的核心类，因为Activity调度比较复杂，Activity相关的信息初始化会在ActivityStackSupervisor中


ActivityManagerService的start()方法：
/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
```
2594      private void start() {
			  // 移除所有的进程组
2595          removeAllProcessGroups();
			  // 启动CPU进程
2596          mProcessCpuThread.start();
2597  
			  // 启动电池状态服务
2598          mBatteryStatsService.publish();
2599          mAppOpsService.publish(mContext);
2600          Slog.d("AppOps", "AppOpsService published");
2601          LocalServices.addService(ActivityManagerInternal.class, new LocalService());
			  // 创建本地服务并注册，将创建的本地服务放入本地服务集合完成注册
2602          mActivityTaskManager.onActivityManagerInternalAdded();
2603          mUgmInternal.onActivityManagerInternalAdded();
2604          mPendingIntentController.onActivityManagerInternalAdded();
2605          // Wait for the synchronized block started in mProcessCpuThread,
2606          // so that any other access to mProcessCpuTracker from main thread
2607          // will be blocked during mProcessCpuTracker initialization.
2608          try {
				  // 等待mProcessCpuThread完成初始化后，释放锁
2609              mProcessCpuInitLatch.await();
2610          } catch (InterruptedException e) {
2611              Slog.wtf(TAG, "Interrupted wait during start", e);
2612              Thread.currentThread().interrupt();
2613              throw new IllegalStateException("Interrupted wait during start");
2614          }
2615      }

```

`AMS`的`start()`方法很简单，只是启动了几个服务，并把`AMS`服务自己保存到`localService`中供程序内部调用;  
`AMS`的构造方法和`start()`方法中做了`AMS`服务一些变量的初始化和相关服务的初始化。

ActivityManagerService的setSystemProcess()方法
/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
```
2040      public void setSystemProcess() {
2041          try {
				  // 将AMS注册到ServiceManager中
2042              ServiceManager.addService(Context.ACTIVITY_SERVICE, this, /* allowIsolated= */ true,
2043                      DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_NORMAL | DUMP_FLAG_PROTO);
				  // 注册进程状态服务
2044              ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
				  // 注册内存Binder
2045              ServiceManager.addService("meminfo", new MemBinder(this), /* allowIsolated= */ false,
2046                      DUMP_FLAG_PRIORITY_HIGH);
				  // 注册图像信息Binder
2047              ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
				  // 注册数据库Binder
2048              ServiceManager.addService("dbinfo", new DbBinder(this));
2049              if (MONITOR_CPU_USAGE) {
					  // 注册监控CPU使用状态Binder
2050                  ServiceManager.addService("cpuinfo", new CpuBinder(this),
2051                          /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL);
2052              }
				  // 注册权限控制Binder
2053              ServiceManager.addService("permission", new PermissionController(this));
				  // 注册进程服务Binder
2054              ServiceManager.addService("processinfo", new ProcessInfoService(this));
2055  
				  // 查询并处理ApplicationInfo
2056              ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
2057                      "android", STOCK_PM_FLAGS | MATCH_SYSTEM_ONLY);
				  // 将Application信息配置到ActivityThread中
2058              mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());
2059  
2060              synchronized (this) {
					  // 创建并处理ProcessRecord
2061                  ProcessRecord app = mProcessList.newProcessRecordLocked(info, info.processName,
2062                          false,
2063                          0,
2064                          new HostingRecord("system"));
2065                  app.setPersistent(true);
2066                  app.pid = MY_PID;
2067                  app.getWindowProcessController().setPid(MY_PID);
2068                  app.maxAdj = ProcessList.SYSTEM_ADJ;
2069                  app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
2070                  mPidsSelfLocked.put(app);
2071                  mProcessList.updateLruProcessLocked(app, false, null);
2072                  updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_NONE);
2073              }
2074          } catch (PackageManager.NameNotFoundException e) {
2075              throw new RuntimeException(
2076                      "Unable to find android system package", e);
2077          }
2078  
2079          // Start watching app ops after we and the package manager are up and running.
2080          mAppOpsService.startWatchingMode(AppOpsManager.OP_RUN_IN_BACKGROUND, null,
2081                  new IAppOpsCallback.Stub() {
2082                      @Override public void opChanged(int op, int uid, String packageName) {
2083                          if (op == AppOpsManager.OP_RUN_IN_BACKGROUND && packageName != null) {
2084                              if (mAppOpsService.checkOperation(op, uid, packageName)
2085                                      != AppOpsManager.MODE_ALLOWED) {
2086                                  runInBackgroundDisabled(uid);
2087                              }
2088                          }
2089                      }
2090                  });
2091      }

```

1)setSystemProcess()方法中，首先将自己AMS服务注册到了ServiceManager中，然后又注册了权限服务等其他的系统服务;
2)通过先前创建的Context，得到PMS服务，检索framework-res的Application信息，然后将它配置到系统的ActivityThread中;
3)为了能让AMS同样可以管理调度系统进程，也创建了一个关于系统进程的ProcessRecord对象，ProcessRecord对象保存一个进程的相关信息;
4)然后将它保存到mPidsSelfLocked集合中方便管理;
5)AMS具体是如何将检索到的framework-res的application信息，配置到ActivityThread中的，需要继续分析ActivityThread的installSystemApplicationInfo(ApplicationInfo info, ClassLoader classLoader))方法;

ActivityThread类中的`installSystemApplicationInfo()`方法
/frameworks/base/core/java/android/app/ActivityThread.java
```
2417      public void installSystemApplicationInfo(ApplicationInfo info, ClassLoader classLoader) {
2418          synchronized (this) {
2419              getSystemContext().installSystemApplicationInfo(info, classLoader);
2420              getSystemUiContext().installSystemApplicationInfo(info, classLoader);
2421  
2422              // give ourselves a default profiler
2423              mProfiler = new Profiler();
2424          }
2425      }
```

这个方法中最终调用上面创建的SystemContext和SystemUiContext的installSystemApplicationInfo()方法。
那就接着看ConxtextImpl的installSystemApplicationInfo()方法：

/frameworks/base/core/java/android/app/ContextImpl.java
```
......
2427      /**
2428       * System Context to be used for UI. This Context has resources that can be themed.
2429       * Make sure that the created system UI context shares the same LoadedApk as the system context.
2430       * @param systemContext The system context which created by
2431       *                      {@link #createSystemContext(ActivityThread)}.
2432       * @param displayId The ID of the display where the UI is shown.
2433       */
2434      static ContextImpl createSystemUiContext(ContextImpl systemContext, int displayId) {
2435          final LoadedApk packageInfo = systemContext.mPackageInfo;
2436          ContextImpl context = new ContextImpl(null, systemContext.mMainThread, packageInfo, null,
2437                  null, null, 0, null, null);
2438          context.setResources(createResources(null, packageInfo, null, displayId, null,
2439                  packageInfo.getCompatibilityInfo()));
2440          context.updateDisplay(displayId);
2441          return context;
2442      }
2443  
2444      /**
2445       * The overloaded method of {@link #createSystemUiContext(ContextImpl, int)}.
2446       * Uses {@Code Display.DEFAULT_DISPLAY} as the target display.
2447       */
2448      static ContextImpl createSystemUiContext(ContextImpl systemContext) {
2449          return createSystemUiContext(systemContext, Display.DEFAULT_DISPLAY);
2450      }
......
2581      void installSystemApplicationInfo(ApplicationInfo info, ClassLoader classLoader) {
2582          mPackageInfo.installSystemApplicationInfo(info, classLoader);
2583      }
......
```

它有最终调用了`mPackageInfo`的`installSystemApplication()`方法：  
/frameworks/base/core/java/android/app/LoadedApk.java
```
240      /**
241       * Sets application info about the system package.
242       */
243      void installSystemApplicationInfo(ApplicationInfo info, ClassLoader classLoader) {
244          assert info.packageName.equals("android");
245          mApplicationInfo = info;
246          mDefaultClassLoader = classLoader;
247          mAppComponentFactory = createAppFactory(info, mDefaultClassLoader);
248          mClassLoader = mAppComponentFactory.instantiateClassLoader(mDefaultClassLoader,
249                  new ApplicationInfo(mApplicationInfo));
250      }
```

mPackageInfo就是在创建Context对象的时候传进来的LoadedApk，里面保存了一个应用程序的基本信息;
setSystemProcess()主要就是设置系统集成的一些信息，在这里设置了系统进程的Application信息，创建了系统进程的ProcessRecord对象将其保存在进程集合中，方便AMS管理调度;

到这里，我们第一阶段startBootstrapServices()方法中启动引导服务里对AMS做了初始化已经结束，下面我继续看第二阶段startOtherServices()方法中启动其他服务中对AMS的初始化。

省略。。。。。。

## 3.总结

（1）系统启动后Zygote进程第一个fork出SystemServer进程

（2）SystemServer->run()->createSystemContext()：
创建了系统的ActivityThread对象，运行环境mSystemContext、systemUiContext。

（3）SystemServer->run()->startBootstrapServices()->ActivityManagerService.Lifecycle.startService()：
AMS在引导服务启动方法中，通过构造函数new ActivityManagerService()进行了一些对象创建和初始化(除activity外3大组件的管理和调度对象创建；内存、电池、权限、性能、cpu等的监控等相关对象创建)，start()启动服务(移除进程组、启动cpu线程、注册权限、电池等服务)。

（4）SystemServer->run()->startBootstrapServices()->setSystemServiceManager()、setInstaller()、initPowerManagement()、setSystemProcess()：
AMS创建后进行了一系列相关的初始化和设置。
setSystemProcess()：将framework-res.apk的信息加入到SystemServer进程的LoadedApk中，并创建了SystemServer进程的ProcessRecord，加入到mPidsSelfLocked，由AMS统一管理。

（5）SystemServer->run()->startOtherServices()：
AMS启动后的后续工作，主要调用systemReady()和运行调用时传入的goingCallback。
systemReady()/goingCallback：各种服务或进程等AMS启动完成后需进一步完成的工作及系统相关初始化。 桌面应用在systemReady()方法中启动，systemui在goingCallback中完成。当桌面应用启动完成后，发送开机广播ACTION_BOOT_COMPLETED


![[AMS流程-1.png]]


![[AMS流程-2.png]]


# 三、WMS（WindowManagerService）

[【Android Framework系列】第7章 WMS原理_android wms原理-CSDN博客](https://blog.csdn.net/u010687761/article/details/131716999?spm=1001.2014.3001.5502)

https://github.com/huangruiLearn/hrl_android_notes/blob/master/1.android/18.WMS/Wms.md

## 1.简介
`WindowManagerService`简称`WMS`，是系统的核心服务，主要分为四大部分，分别是`窗口管理`，`窗口动画`，`输入系统中转站`和`Surface管理`。

1.**窗口管理**：WMS是窗口的管理者，负责窗口的启动，添加和删除，另外窗口的大小也是由WMS管理的，管理窗口的核心成员有DisplayContent，WindowToken和WindowState。窗口的显示顺序、尺寸、位置， 最终都会反馈SurfaceFlinger。  

2.**窗口动画**：窗口间进行切换时，使用窗口动画可以更好看一些，窗口动画由WMS动画子系统来负责，动画的管理系统为WindowAnimator。

3.**输入系统的中转站**：通过对窗口触摸而产生的触摸事件，InputManagerServer(IMS)会对触摸事件进行处理，他会寻找一个最合适的窗口来处理触摸反馈信息，WMS是窗口的管理者，因此理所当然的就成为了输入系统的中转站。 

4.**Surface管理**：窗口并不具备绘制的功能，因此每个窗口都需要有一个块Surface来供自己绘制，为每个窗口分配Surface是由WMS来完成。
## 2.WMS是如何管理Window的

（1）setContentView把layoutResID 添加到 decorView中. 在Activity启动过程中会调用ActivityThread->performLaunchActivity->Activity的attach方法,attach创建了PhoneWindow 之后PhoneWindow对象里面添加了一个DecorView

（2）DecorView绑定一个ViewRootImpl，获取 IWindowSession 和 WMS 实例。

（3）ViewRootImpl将Session将请求委托给WMS， 并向 WMS 添加一个WindowToken和注册窗口。

（4）向 WMS 申请对窗口进行 ReLayout。也就是根据窗口新的属性去调整其 Surface 相关的属性， 由SurfaceFlinger分配Surface。向 WMS 添加窗口之后，仅仅是将其在 WMS 中进行了注册， 只有经过重新布局，窗口才拥有 WMS 为其分配的画布

（5）App客户端调用View.onDraw方法进行绘制

（6）绘制完之后，SurfaceFlinger就会按照WMS里面提供的层级等信息进行合成，最终显示



## 3.总结

`WMS`窗口管理服务，每一个`Window`都对应着一个`View`和一个`ViewRootImpl`。`Window`表示一个窗口的概念，也是一个抽象的概念，它并不是实际存在的，它是以`View`的方式存在的。  
`WindowManager`是我们访问`Window`的入口，`Window`的具体实现位于`WindowManagerService`中。`WindowManager`和`WindowManagerService`交互是一个`IPC`的过程，最终的IPC是在`RootViewImpl`中完成的。

我们知道窗口的展示主要有`Activity`和`Dialog`、`Toast`，结合我们前面AMS的章节，以Activity启动为例对WMS流程进行总结：

1. 由`Launcher`发起调用
2. `AMS`管理的是配置信息数据
3. 具体的对象创建过程在`ActivityThread`中完成
4. 有`AMS`的`startActivity`触发`resume`生命周期，在`resume`生命周期中对于`View`数据进行推送至`WindowManager`进行管理，同时生成一个·ViewRootImpl·对象对于所有·View·数据进行绘制管理
5. `ViewRootImpl`中依赖于编舞者工具对于绘制进行控制，一般情况为`60FPS`
6. 具体绘制有`ViewRootImpl`中的`performTraversals`进行具体绘制操作
7. 绘制完成之后，因为每一个`View`都是一个`Surface`，通过`SurfaceFilnger`提供的`Surface`每个`View`绘制完成之后
8. 由`WMS`统一协调各个View的`层级`、`尺寸`、`布局`等
9. 最终交由`SurfaceFilnger`进行帧合成，完成整体界面输出
10. 底层由`opengl`完成




