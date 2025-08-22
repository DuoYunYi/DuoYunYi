https://blog.csdn.net/sinat_20059415/article/details/88206937?ops_request_misc=&request_id=&biz_id=102&utm_term=android%20P%20%E8%AE%BE%E5%A4%87%E5%BC%80%E6%9C%BAwifi%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-1-88206937.142^v102^pc_search_result_base8&spm=1018.2226.3001.4187

**wifi相关的文件位置：**

WIFI Settings应用程序位于

       packages/apps/Settings/src/com/android/settings/wifi/

JAVA部分：

        frameworks/base/services/java/com/android/server/

        frameworks/base/wifi/java/android/net/wifi/

JNI部分：

       frameworks/base/core/jni/android_net_wifi_Wifi.cpp

wifi管理库。

        hardware/libhardware_legary/wifi/

 wifi用户空间的程序和库:

        external/wpa_supplicant/

       生成库libwpaclient.so和守护进程wpa_supplicant。


**调用流程：**

**wifi模块的初始化：**

（frameworks/base/services/java/com/android/server/SystemServer.Java）

在 SystemServer 启动的时候,会生成一个**ConnectivityService** 的实例,

```
            traceBeginAndSlog("StartConnectivityService");
            try {
                connectivity = new ConnectivityService(
                    context, networkManagement, networkStats, networkPolicy);
                ServiceManager.addService(Context.CONNECTIVITY_SERVICE, connectivity,
                            /* allowIsolated= */ false,
                    DUMP_FLAG_PRIORITY_HIGH | DUMP_FLAG_PRIORITY_NORMAL);
                networkStats.bindConnectivityManager(connectivity);
                networkPolicy.bindConnectivityManager(connectivity);
            } catch (Throwable e) {
                reportWtf("starting Connectivity Service", e);
            }
            traceEnd();
```

其中 ，**ConnectivityService.getInstance(context)**;  对应于（frameworks/base/services/java/com/android/server/ **ConnectivityService.Java**）**ConnectivityService.Java。**

aml_sdk/frameworks/base/services/core/java/com/android/server/ConnectivityService.java




# startwifi初始化


```
                if (context.getPackageManager().hasSystemFeature(
                            PackageManager.FEATURE_WIFI)) {
                    // Wifi Service must be started first for wifi-related services.
                    traceBeginAndSlog("StartWifi");
                    mSystemServiceManager.startService(WIFI_SERVICE_CLASS);
                    traceEnd();
                    traceBeginAndSlog("StartWifiScanning");
                    mSystemServiceManager.startService(
                        "com.android.server.wifi.scanner.WifiScanningService");
                    traceEnd();
                }
```




Setting菜单里点击打开Wifi时，调用的入口函数是WifiManager::setWifiEnabled(boolean enabled)：

# wifi模块的启动（Enable）



aml_sdk/frameworks/base/wifi/java/android/net/wifi/WifiManager.java
```
    public boolean setWifiEnabled(boolean enabled) {
        try {
            return mService.setWifiEnabled(mContext.getOpPackageName(), enabled);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```


通过AIDL方式，在Android6.0中，实际调用的是WifiServiceImpl::setWifiEnabled(boolean enable)：

aml_sdk/frameworks/base/wifi/java/android/net/wifi/IWifiManager.aidl

```
boolean setWifiEnabled(String packageName, boolean enable);
```


aml_sdk/frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiServiceImpl.java

```
    @Override
    public synchronized boolean setWifiEnabled(String packageName, boolean enable)
            throws RemoteException {
        if (enforceChangePermission(packageName) != MODE_ALLOWED) {
            return false;
        }

        Slog.d(TAG, "setWifiEnabled: " + enable + " pid=" + Binder.getCallingPid()
                    + ", uid=" + Binder.getCallingUid() + ", package=" + packageName);
        mLog.info("setWifiEnabled package=% uid=% enable=%").c(packageName)
                .c(Binder.getCallingUid()).c(enable).flush();

        boolean isFromSettings = checkNetworkSettingsPermission(
                Binder.getCallingPid(), Binder.getCallingUid());

        // If Airplane mode is enabled, only Settings is allowed to toggle Wifi
        if (mSettingsStore.isAirplaneModeOn() && !isFromSettings) {
            mLog.info("setWifiEnabled in Airplane mode: only Settings can enable wifi").flush();
            return false;
        }

        // If SoftAp is enabled, only Settings is allowed to toggle wifi
        boolean apEnabled = mWifiApState == WifiManager.WIFI_AP_STATE_ENABLED;

        if (apEnabled && !isFromSettings) {
            mLog.info("setWifiEnabled SoftAp not disabled: only Settings can enable wifi").flush();
            return false;
        }

        /*
        * Caller might not have WRITE_SECURE_SETTINGS,
        * only CHANGE_WIFI_STATE is enforced
        */
        long ident = Binder.clearCallingIdentity();
        try {
            if (! mSettingsStore.handleWifiToggled(enable)) {
                // Nothing to do if wifi cannot be toggled
                return true;
            }
        } finally {
            Binder.restoreCallingIdentity(ident);
        }


        if (mPermissionReviewRequired) {
            final int wiFiEnabledState = getWifiEnabledState();
            if (enable) {
                if (wiFiEnabledState == WifiManager.WIFI_STATE_DISABLING
                        || wiFiEnabledState == WifiManager.WIFI_STATE_DISABLED) {
                    if (startConsentUi(packageName, Binder.getCallingUid(),
                            WifiManager.ACTION_REQUEST_ENABLE)) {
                        return true;
                    }
                }
            } else if (wiFiEnabledState == WifiManager.WIFI_STATE_ENABLING
                    || wiFiEnabledState == WifiManager.WIFI_STATE_ENABLED) {
                if (startConsentUi(packageName, Binder.getCallingUid(),
                        WifiManager.ACTION_REQUEST_DISABLE)) {
                    return true;
                }
            }
        }

        mWifiController.sendMessage(CMD_WIFI_TOGGLED);
        return true;
    }
```


mSettingsStore.handleWifiToggled(enable)

mWifiController.sendMessage(CMD_WIFI_TOGGLED);

从代码可以看出，这里主要的操作是将wifi是否enable的状态存入数据库、向WiFiController发送了CMD_WIFI_TOGGLED消息。

WifiController实际上是一个状态机，相比WifiStateMachine，它的状态较少，结构也比较简单。WifiController的定义及构造函数：

注：状态机：退出一个状态的时候调用exit方法，进入一个状态的时候调用enter方法，不走多余状态

aml_sdk/frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiController.java

```
// CHECKSTYLE:OFF IndentationCheck
        addState(mDefaultState);
            addState(mStaDisabledState, mDefaultState);
            addState(mStaEnabledState, mDefaultState);
                addState(mDeviceActiveState, mStaEnabledState);
            addState(mStaDisabledWithScanState, mDefaultState);
            addState(mEcmState, mDefaultState);
        // CHECKSTYLE:ON IndentationCheck

```

简略了很多，ap相关（softap）剥离了，StaEnabledState和DeviceActiveState其实是一个了，可以合成一个。

初始状态

```
        if (checkScanOnlyModeAvailable()) {
            setInitialState(mStaDisabledWithScanState);
        } else {
            setInitialState(mStaDisabledState);
        }
```

先暂时略过ScanOnly，先看StaDisabledState

aml_sdk/frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiController.java






打开WiFi后WifiController状态机切换（这边先梳理主动打开WiFi逻辑，always scan后续梳理）

```
    /**
     * Parent: StaEnabledState
     *
     * TODO (b/79209870): merge DeviceActiveState and StaEnabledState into a single state
     */
    class DeviceActiveState extends State {
        @Override
        public void enter() {
            mWifiStateMachinePrime.enterClientMode();
            mWifiStateMachine.setHighPerfModeEnabled(false);
        }

        @Override
        public boolean processMessage(Message msg) {
            if (msg.what == CMD_USER_PRESENT) {
                // TLS networks can't connect until user unlocks keystore. KeyStore
                // unlocks when the user punches PIN after the reboot. So use this
                // trigger to get those networks connected.
                if (mFirstUserSignOnSeen == false) {
                    mWifiStateMachine.reloadTlsNetworksAndReconnect();
                }
                mFirstUserSignOnSeen = true;
                return HANDLED;
            } else if (msg.what == CMD_RECOVERY_RESTART_WIFI) {
                final String bugTitle;
                final String bugDetail;
                if (msg.arg1 < SelfRecovery.REASON_STRINGS.length && msg.arg1 >= 0) {
                    bugDetail = SelfRecovery.REASON_STRINGS[msg.arg1];
                    bugTitle = "Wi-Fi BugReport: " + bugDetail;
                } else {
                    bugDetail = "";
                    bugTitle = "Wi-Fi BugReport";
                }
                if (msg.arg1 != SelfRecovery.REASON_LAST_RESORT_WATCHDOG) {
                    (new Handler(mWifiStateMachineLooper)).post(() -> {
                        mWifiStateMachine.takeBugReport(bugTitle, bugDetail);
                    });
                }
                return NOT_HANDLED;
            }
            return NOT_HANDLED;
        }
    }
```

这边不一样了，出现了个WifiStateMachinePrime

            mWifiStateMachinePrime.enterClientMode();  
            mWifiStateMachine.setHighPerfModeEnabled(false);


2.4 WifiStateMachinePrime

aml_sdk/frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiStateMachinePrime.java

```
WifiInjector

        mWifiStateMachinePrime = new WifiStateMachinePrime(this, mContext, wifiStateMachineLooper,
                mWifiNative, new DefaultModeManager(mContext, wifiStateMachineLooper),
                mBatteryStats);
...
        mWifiController = new WifiController(mContext, mWifiStateMachine, wifiStateMachineLooper,
                mSettingsStore, mWifiServiceHandlerThread.getLooper(), mFrameworkFacade,
                mWifiStateMachinePrime);
```

看下enterClientMode

    /**  
     * Method to switch wifi into client mode where connections to configured networks will be  
     * attempted.  
     */  
    public void enterClientMode() {  
        changeMode(ModeStateMachine.CMD_START_CLIENT_MODE);  
    }  
看下ModeStateMachine的模式

        // Commands for the state machine  - these will be removed,  
        // along with the StateMachine itself  
        public static final int CMD_START_CLIENT_MODE    = 0;  
        public static final int CMD_START_SCAN_ONLY_MODE = 1;  
        public static final int CMD_DISABLE_WIFI         = 3;  
有3个模式，ClientMode模式，应该就是WiFi普通打开模式，Scan_ONLY应该就是不打开WiFi只打开WiFi扫描的模式，Disable_WIFI应该是上面两个都没打开的情况。（具体情况待更新）

    private void changeMode(int newMode) {  
        mModeStateMachine.sendMessage(newMode);  
    }
看下ModeStateMachine状态机

aml_sdk/frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiStateMachinePrime.java

```
private class ModeStateMachine extends StateMachine {
        // Commands for the state machine  - these will be removed,
        // along with the StateMachine itself
        public static final int CMD_START_CLIENT_MODE    = 0;
        public static final int CMD_START_SCAN_ONLY_MODE = 1;
        public static final int CMD_DISABLE_WIFI         = 3;

        private final State mWifiDisabledState = new WifiDisabledState();
        private final State mClientModeActiveState = new ClientModeActiveState();
        private final State mScanOnlyModeActiveState = new ScanOnlyModeActiveState();

        ModeStateMachine() {
            super(TAG, mLooper);

            addState(mClientModeActiveState);
            addState(mScanOnlyModeActiveState);
            addState(mWifiDisabledState);

            Log.d(TAG, "Starting Wifi in WifiDisabledState");
            setInitialState(mWifiDisabledState);
            start();
        }
```


初始WifiDiabledState处理刚发来的切换状态消息

```
        class WifiDisabledState extends ModeActiveState {
            @Override
            public void enter() {
                Log.d(TAG, "Entering WifiDisabledState");
                mDefaultModeManager.sendScanAvailableBroadcast(mContext, false);
                mScanRequestProxy.enableScanningForHiddenNetworks(false);
                mScanRequestProxy.clearScanResults();
            }

            @Override
            public boolean processMessage(Message message) {
                Log.d(TAG, "received a message in WifiDisabledState: " + message);
                if (checkForAndHandleModeChange(message)) {
                    return HANDLED;
                }
                return NOT_HANDLED;
            }

            @Override
            public void exit() {
                // do not have an active mode manager...  nothing to clean up
            }

        }
```

```
        private boolean checkForAndHandleModeChange(Message message) {
            switch(message.what) {
                case ModeStateMachine.CMD_START_CLIENT_MODE:
                    Log.d(TAG, "Switching from " + getCurrentMode() + " to ClientMode");
                    mModeStateMachine.transitionTo(mClientModeActiveState);
                    break;
                case ModeStateMachine.CMD_START_SCAN_ONLY_MODE:
                    Log.d(TAG, "Switching from " + getCurrentMode() + " to ScanOnlyMode");
                    mModeStateMachine.transitionTo(mScanOnlyModeActiveState);
                    break;
                case ModeStateMachine.CMD_DISABLE_WIFI:
                    Log.d(TAG, "Switching from " + getCurrentMode() + " to WifiDisabled");
                    mModeStateMachine.transitionTo(mWifiDisabledState);
                    break;
                default:
                    return NOT_HANDLED;
            }
            return HANDLED;
        }
```


状态切换到ClientModeActiveState


```
        class ClientModeActiveState extends ModeActiveState {
            ClientListener mListener;
            private class ClientListener implements ClientModeManager.Listener {
                @Override
                public void onStateChanged(int state) {
                    // make sure this listener is still active
                    if (this != mListener) {
                        Log.d(TAG, "Client mode state change from previous manager");
                        return;
                    }

                    Log.d(TAG, "State changed from client mode. state = " + state);

                    if (state == WifiManager.WIFI_STATE_UNKNOWN) {
                        // error while setting up client mode or an unexpected failure.
                        mModeStateMachine.sendMessage(CMD_CLIENT_MODE_FAILED, this);
                    } else if (state == WifiManager.WIFI_STATE_DISABLED) {
                        // client mode stopped
                        mModeStateMachine.sendMessage(CMD_CLIENT_MODE_STOPPED, this);
                    } else if (state == WifiManager.WIFI_STATE_ENABLED) {
                        // client mode is ready to go
                        Log.d(TAG, "client mode active");
                    } else {
                        // only care if client mode stopped or started, dropping
                    }
                }
            }

            @Override
            public void enter() {
                Log.d(TAG, "Entering ClientModeActiveState");

                mListener = new ClientListener();
                mManager = mWifiInjector.makeClientModeManager(mListener);
                mManager.start();
                mActiveModeManagers.add(mManager);

                updateBatteryStatsWifiState(true);
            }

            @Override
            public void exit() {
                super.exit();
                mListener = null;
            }

            @Override
            public boolean processMessage(Message message) {
                if (checkForAndHandleModeChange(message)) {
                    return HANDLED;
                }

                switch(message.what) {
                    case CMD_START_CLIENT_MODE:
                        Log.d(TAG, "Received CMD_START_CLIENT_MODE when active - drop");
                        break;
                    case CMD_CLIENT_MODE_FAILED:
                        if (mListener != message.obj) {
                            Log.d(TAG, "Client mode state change from previous manager");
                            return HANDLED;
                        }
                        Log.d(TAG, "ClientMode failed, return to WifiDisabledState.");
                        // notify WifiController that ClientMode failed
                        mClientModeCallback.onStateChanged(WifiManager.WIFI_STATE_UNKNOWN);
                        mModeStateMachine.transitionTo(mWifiDisabledState);
                        break;
                    case CMD_CLIENT_MODE_STOPPED:
                        if (mListener != message.obj) {
                            Log.d(TAG, "Client mode state change from previous manager");
                            return HANDLED;
                        }

                        Log.d(TAG, "ClientMode stopped, return to WifiDisabledState.");
                        // notify WifiController that ClientMode stopped
                        mClientModeCallback.onStateChanged(WifiManager.WIFI_STATE_DISABLED);
                        mModeStateMachine.transitionTo(mWifiDisabledState);
                        break;
                    default:
                        return NOT_HANDLED;
                }
                return NOT_HANDLED;
            }
        }
```

enter方法里主要干了两件事

```
                Log.d(TAG, "Entering ClientModeActiveState");
 
                mListener = new ClientListener();
                mManager = mWifiInjector.makeClientModeManager(mListener);
                mManager.start();
                mActiveModeManagers.add(mManager);
 
                updateBatteryStatsWifiState(true);
 
 
    /**
     * Create a ClientModeManager
     *
     * @param listener listener for ClientModeManager state changes
     * @return a new instance of ClientModeManager
     */
    public ClientModeManager makeClientModeManager(ClientModeManager.Listener listener) {
        return new ClientModeManager(mContext, mWifiStateMachineHandlerThread.getLooper(),
                mWifiNative, listener, mWifiMetrics, mScanRequestProxy, mWifiStateMachine);
    }
 
 
    /**
     *  Helper method to report wifi state as on/off (doesn't matter which mode).
     *
     *  @param enabled boolean indicating that some mode has been turned on or off
     */
    private void updateBatteryStatsWifiState(boolean enabled) {
        try {
            if (enabled) {
                if (mActiveModeManagers.size() == 1) {
                    // only report wifi on if we haven't already
                    mBatteryStats.noteWifiOn();
                }
            } else {
                if (mActiveModeManagers.size() == 0) {
                    // only report if we don't have any active modes
                    mBatteryStats.noteWifiOff();
                }
            }
        } catch (RemoteException e) {
            Log.e(TAG, "Failed to note battery stats in wifi");
        }
    }
```

1. ClientModeManager.start
2. updateBatteryStatsWifiState
```
/**
 * Manager WiFi in Client Mode where we connect to configured networks.
 */
public class ClientModeManager implements ActiveModeManager {
 
/**
 * Base class for available WiFi operating modes.
 *
 * Currently supported modes include Client, ScanOnly and SoftAp.
 */
public interface ActiveModeManager {
    String TAG = "ActiveModeManager";
 
    /**
     * Method used to start the Manager for a given Wifi operational mode.
     */
    void start();
 
    /**
     * Method used to stop the Manager for a give Wifi operational mode.
     */
    void stop();
 
    /**
     * Method to dump for logging state.
     */
    void dump(FileDescriptor fd, PrintWriter pw, String[] args);
 
    /**
     * Method that allows Mode Managers to update WifiScanner about the current state.
     *
     * @param context Context to use for the notification
     * @param available boolean indicating if scanning is available
     */
    default void sendScanAvailableBroadcast(Context context, boolean available) {
        Log.d(TAG, "sending scan available broadcast: " + available);
        final Intent intent = new Intent(WifiManager.WIFI_SCAN_AVAILABLE);
        intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT);
        if (available) {
            intent.putExtra(WifiManager.EXTRA_SCAN_AVAILABLE, WifiManager.WIFI_STATE_ENABLED);
        } else {
            intent.putExtra(WifiManager.EXTRA_SCAN_AVAILABLE, WifiManager.WIFI_STATE_DISABLED);
        }
        context.sendStickyBroadcastAsUser(intent, UserHandle.ALL);
    }
}
```

这里从api注释来看解耦出来了一个接口，用于Client, ScanOnly and SoftAp的实现。

看下start()方法

```
    ClientModeManager(Context context, @NonNull Looper looper, WifiNative wifiNative,
            Listener listener, WifiMetrics wifiMetrics, ScanRequestProxy scanRequestProxy,
            WifiStateMachine wifiStateMachine) {
        mContext = context;
        mWifiNative = wifiNative;
        mListener = listener;
        mWifiMetrics = wifiMetrics;
        mScanRequestProxy = scanRequestProxy;
        mWifiStateMachine = wifiStateMachine;
        mStateMachine = new ClientModeStateMachine(looper);
    }
 
    /**
     * Start client mode.
     */
    public void start() {
        mStateMachine.sendMessage(ClientModeStateMachine.CMD_START);
    }
```

初始化了一个状态机，start是发出了一个消息给状态机来处理

```
        ClientModeStateMachine(Looper looper) {
            super(TAG, looper);
 
            addState(mIdleState);
            addState(mStartedState);
 
            setInitialState(mIdleState);
            start();
        }
```

状态机很简单，就两个状态，IdleState是初始状态。

```
        private class IdleState extends State {
 
            @Override
            public void enter() {
                Log.d(TAG, "entering IdleState");
                mClientInterfaceName = null;
                mIfaceIsUp = false;
            }
 
            @Override
            public boolean processMessage(Message message) {
                switch (message.what) {
                    case CMD_START:
                        updateWifiState(WifiManager.WIFI_STATE_ENABLING,
                                        WifiManager.WIFI_STATE_DISABLED);
 
                        mClientInterfaceName = mWifiNative.setupInterfaceForClientMode(
                                false /* not low priority */, mWifiNativeInterfaceCallback);
                        if (TextUtils.isEmpty(mClientInterfaceName)) {
                            Log.e(TAG, "Failed to create ClientInterface. Sit in Idle");
                            updateWifiState(WifiManager.WIFI_STATE_UNKNOWN,
                                            WifiManager.WIFI_STATE_ENABLING);
                            updateWifiState(WifiManager.WIFI_STATE_DISABLED,
                                            WifiManager.WIFI_STATE_UNKNOWN);
                            break;
                        }
                        sendScanAvailableBroadcast(false);
                        mScanRequestProxy.enableScanningForHiddenNetworks(false);
                        mScanRequestProxy.clearScanResults();
                        transitionTo(mStartedState);
                        break;
                    default:
                        Log.d(TAG, "received an invalid message: " + message);
                        return NOT_HANDLED;
                }
                return HANDLED;
            }
        }
```

这边逻辑和Android O的WiFi启动逻辑很像了，应该是初始化驱动和启动supplicant，具体看下。

```
        private class StartedState extends State {
 
            private void onUpChanged(boolean isUp) {
                if (isUp == mIfaceIsUp) {
                    return;  // no change
                }
                mIfaceIsUp = isUp;
                if (isUp) {
                    Log.d(TAG, "Wifi is ready to use for client mode");
                    sendScanAvailableBroadcast(true);
                    mWifiStateMachine.setOperationalMode(WifiStateMachine.CONNECT_MODE,
                                                         mClientInterfaceName);
                    updateWifiState(WifiManager.WIFI_STATE_ENABLED,
                                    WifiManager.WIFI_STATE_ENABLING);
                } else {
                    if (mWifiStateMachine.isConnectedMacRandomizationEnabled()) {
                        // Handle the error case where our underlying interface went down if we
                        // do not have mac randomization enabled (b/72459123).
                        return;
                    }
                    // if the interface goes down we should exit and go back to idle state.
                    Log.d(TAG, "interface down!");
                    updateWifiState(WifiManager.WIFI_STATE_UNKNOWN,
                                    WifiManager.WIFI_STATE_ENABLED);
                    mStateMachine.sendMessage(CMD_INTERFACE_DOWN);
                }
            }
 
            @Override
            public void enter() {
                Log.d(TAG, "entering StartedState");
                mIfaceIsUp = false;
                onUpChanged(mWifiNative.isInterfaceUp(mClientInterfaceName));
                mScanRequestProxy.enableScanningForHiddenNetworks(true);
            }
```

进入到StartedState可以看到wifi状态就更新为enabled了。

### 2.5 WifiNative

xxxx


这边逻辑不大一样，framework基本梳理完了到hal了，先梳理下面3步

- startHal
- startSupplicant
- createStaIface
```
    /** Helper method invoked to start supplicant if there were no ifaces */
    private boolean startHal() {
        synchronized (mLock) {
            if (!mIfaceMgr.hasAnyIface()) {
                if (mWifiVendorHal.isVendorHalSupported()) {
                    if (!mWifiVendorHal.startVendorHal()) {
                        Log.e(TAG, "Failed to start vendor HAL");
                        return false;
                    }
                } else {
                    Log.i(TAG, "Vendor Hal not supported, ignoring start.");
                }
            }
            return true;
        }
    }
```


流程图：
https://i-blog.csdnimg.cn/blog_migrate/fcd5787c7407af2e40d160e5a35b0769.jpeg


![[Pasted image 20250722201317.png]]