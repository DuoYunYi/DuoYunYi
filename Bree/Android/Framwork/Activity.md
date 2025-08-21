# 一 、Activity的启动流程

## Activity跨进程启动

[https://juejin.im/post/6844903959581163528#heading-1](https://juejin.im/post/6844903959581163528#heading-1)

[http://gityuan.com/2016/03/12/start-activity/](http://gityuan.com/2016/03/12/start-activity/)

https://zhuanlan.zhihu.com/p/430292809

启动流程：

- 点击桌面App图标，Launcher进程采用Binder IPC向system_server进程发起startActivity请求；

- system_server进程接收到请求后，向zygote进程发送创建进程的请求；

- Zygote进程fork出新的子进程，即App进程；

- App进程，通过Binder IPC向sytem_server进程发起attachApplication请求；

- system_server进程在收到请求后，进行一系列准备工作后，再通过binder IPC向App进 程发送scheduleLaunchActivity请求；

- App进程的binder线程（ApplicationThread）在收到请求后，通过handler向主线程发送LAUNCH_ACTIVITY消息；

- 主线程在收到Message后，通过发射机制创建目标Activity，并回调Activity.onCreate()等方法。

## Activity进程内启动

TODO

# 二、onSaveInstanceState()

主要用于在 **系统主动销毁** 组件时（如内存不足、配置变更等）临时保存数据，以便后续恢复。它的核心作用是 **状态持久化**，但不同于 `SharedPreferences` 或数据库，它仅用于 **临时保存** 轻量级数据。

## 1.核心作用

- **保存临时状态**：当 Activity/Fragment 被系统销毁并可能重建时（如屏幕旋转、内存回收），保存当前的 UI 状态（如输入框内容、滚动位置等）。
- **数据恢复**：在 `onCreate(Bundle)` 或 `onRestoreInstanceState` 中恢复数据。
- **系统自动调用**：开发者无需手动触发，系统会在可能销毁组件前自动调用。

> ⚠️ **注意**：`onSaveInstanceState` **不会** 在用户主动退出（如按返回键）或调用 `finish()` 时触发。

## 2.调用时机

**onSaveInstanceState(Bundle outState)会在以下情况被调用：**

1、从最近应用中选择运行其他的程序时。

2、当用户按下HOME键时。

3、屏幕方向切换时(无论竖屏切横屏还是横屏切竖屏都会调用)。

4、按下电源按键（关闭屏幕显示）时。

5、从当前activity启动一个新的activity时。

```
onPause -> onSaveInstanceState -> onStop。
```

onSaveInstanceState android 28之前是在onStop之前， 28之后在onStop之后

# 三、onRestoreInstanceState()

`onRestoreInstanceState` 是 Android **Activity** 的一个生命周期回调方法，主要用于 **恢复 `onSaveInstanceState` 保存的临时数据**。它在 Activity **被系统销毁后重建时** 被调用，确保 UI 状态能够正确恢复。

## 1.核心作用

- **恢复临时数据**：从 `Bundle` 中读取 `onSaveInstanceState` 保存的数据，重建 Activity 的 UI 状态。
- **仅在重建时调用**：只有 Activity **被系统销毁后重建**（如屏幕旋转、内存回收）才会触发，用户主动退出（如返回键）不会调用。
- **补充 `onCreate` 的数据恢复**：适用于需要在 `onCreate` 之后恢复数据的场景（如 View 已初始化完毕）。

## 2.调用时机

**onRestoreInstanceState(Bundle outState)会在以下情况被调用：**

onRestoreInstanceState(Bundle savedInstanceState)只有在activity确实是**被系统回收，重新创建activity**的情况下才会被调用。

| **场景**                        | **是否调用 `onRestoreInstanceState`** | **原因说明**               |     |
| ----------------------------- | --------------------------------- | ---------------------- | --- |
| **屏幕旋转/配置变更**                 | ✅ 调用                              | 系统销毁并重建 Activity       |     |
| **内存不足被系统回收后恢复**              | ✅ 调用                              | 系统尝试恢复之前的状态            |     |
| **从最近任务列表重新进入**               | ⚠️ 可能调用                           | 取决于系统是否保留了 Activity 实例 |     |
| **用户按返回键退出**                  | ❌ 不调用                             | 属于主动销毁，不会触发状态恢复        |     |
| **调用 `finish()` 结束 Activity** | ❌ 不调用                             | 属于主动销毁                 |     |
| **跳转到其他 Activity 后返回**        | ❌ 通常不调用                           | Activity 仍在栈中，未被销毁     |     |

```
onCreate() → onStart() → onRestoreInstanceState() → onResume()
```

# 四、Activity启动模式

## （1）Standard(标准启动模式)

默认模式，每次启动Activity都会创建一个新的Activity实例。

**特点**
- 每次启动都会创建新实例
- 允许同一个 Activity 在栈中出现多次
- 新实例会进入启动它的 Activity 所在的任务栈

**适用场景**
- 普通页面（如新闻详情页）
- 不需要特殊管理实例的情况
## （2）SingleTop(栈顶复用模式)

如果目标 Activity **已经在栈顶**，则复用该实例，否则创建新实例。

**特点**
- **栈顶检查**：仅当 Activity 在栈顶时复用
- **调用 `onNewIntent()`**：复用时会触发此方法
- **不清理栈内其他 Activity**
如果要启动的Activity已经在栈顶，则不会重新创建Activity，只会调用该该Activity的onNewIntent()方法。
如果要启动的Activity不在栈顶，则会重新创建该Activity的实例。

**适用场景**
- 防止重复打开同一个页面（如通知点击）
- 避免快速点击导致多个相同页面
## （3）SingleTask(栈内复用模式)

在整个任务栈中**只保留一个实例**，并会**清除其上方所有 Activity**

**特点**
- **单例模式**：一个任务栈中只允许一个实例
- **任务栈管理**：可指定 `taskAffinity` 决定所属栈
- **清除上方 Activity**：复用时会销毁其上的所有页面
- **触发 `onNewIntent()`**
如果要启动的Activity已经存在于它想要归属的栈中，那么不会创建该Activity实例，将栈中位于该Activity上的所有的Activity出栈，同时该Activity的onNewIntent()方法会被调用。

**适用场景**
- 主界面（如 MainActivity）
- 登录页（防止重复登录）
- 核心入口页面
## （4）SingleInstance(单例独占模式)

Activity 会运行在**独立的任务栈**中，且该栈只允许存在这一个 Activity。

**特点**
- **独立任务栈**：不与其他 Activity 共享栈
- **全局单例**：整个系统只保留一个实例
- **任何应用共享**：其他应用也可访问该实例
要创建在一个新栈，然后创建该Activity实例并压入新栈中，新栈中只会存在这一个Activity实例。

**适用场景**
- 系统级页面（如来电界面）
- 需要全局唯一性的特殊页面（如支付收银台）

## Intent Flags（动态控制启动方式）

除了静态 XML 配置，还可以通过 Intent Flags 动态控制：

|**Flag**|**作用**|
|---|---|
|`FLAG_ACTIVITY_NEW_TASK`|在新任务栈中启动|
|`FLAG_ACTIVITY_SINGLE_TOP`|等同于 singleTop|
|`FLAG_ACTIVITY_CLEAR_TOP`|清除目标 Activity 上方的所有页面|
|`FLAG_ACTIVITY_CLEAR_TASK`|清除整个任务栈后重新创建|

```
Intent intent = new Intent(this, TargetActivity.class);
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_CLEAR_TOP);
startActivity(intent);
```

# 五、onStart 和 onResume、onPause 和 onStop 的区别

## 1.onStart 和 onResume 的区别

|方法|调用时机|是否可见|是否可交互|
|---|---|---|---|
|`onStart`|Activity **即将可见**|❌ 还未绘制完成|❌ 不可操作|
|`onResume`|Activity **完全可见并可交互**|✅ 已显示|✅ 可触摸/点击|

## 2. onPause 和 onStop 的区别

| 方法        | 调用时机                | 是否可见             | 是否可交互   |
| --------- | ------------------- | ---------------- | ------- |
| `onPause` | Activity **开始失去焦点** | ⚠️ 部分可见（如被对话框覆盖） | ❌ 已不可交互 |
| `onStop`  | Activity **完全不可见**  | ❌ 完全不可见          | ❌ 不可交互  |

# 六、Activity之间Intent传递数据

startActivity->startActivityForResult->Instrumentation.execStartActivity->ActivityManger.getService().startActivity

intent中携带的数据要从APP进程传输到AMS进程，再由AMS进程传输到目标Activity所在进程
# 七、onNewIntent()

`onNewIntent()` 是 Activity 的一个重要生命周期方法，它在特定情况下被调用，用于处理 **已有 Activity 实例被重新激活** 的情况，而不是创建新实例。

## 1.调用时机

`onNewIntent()` 在以下 **两种启动模式** 下会被调用：

1. **`singleTop`（栈顶复用模式）**
    - 当 Activity 已经位于 **任务栈的顶部**，并且再次被启动时。
2. **`singleTask`（栈内复用模式）**
    - 当 Activity 已经存在于 **任务栈的任意位置**，并且再次被启动时。

> ⚠️ **注意**：`standard`（默认）和 `singleInstance` 模式 **不会触发** `onNewIntent()`，因为它们总是创建新实例或独占任务栈。

## 2.核心作用

避免重复创建实例，更新已有 Activity 的数据。
# 八、显示启动和隐式启动

## 1.显示启动（Explicit）

直接指定目标 Activity 的类名，适用于同一应用内的页面跳转。

**特点**
- 必须明确指定目标 Activity 的 `ComponentName`（类名）
- 只能启动当前应用内的 Activity（或已知的其他应用公开 Activity）
- 不依赖 `Intent-Filter`，直接匹配类名

```
// 方式1：直接指定目标类
Intent intent = new Intent(this, TargetActivity.class);
startActivity(intent);

// 方式2：通过 ComponentName 指定
ComponentName component = new ComponentName(this, "com.example.TargetActivity");
Intent intent = new Intent();
intent.setComponent(component);
startActivity(intent);
```
## 2.隐式启动（Implicit）

[https://www.jianshu.com/p/12c6253f1851](https://www.jianshu.com/p/12c6253f1851)

隐式Intent是通过在AndroidManifest文件中设置action、data、category，让系统来筛选出合适的Activity，适用于跨应用调用

**特点**
- 不指定具体类名，而是声明操作类型（如查看、编辑、分享）
- 依赖目标 Activity 的 `<intent-filter>` 配置
- 可能弹出选择器让用户选择匹配的应用（如果多个应用响应同一 Intent）
# 九、ANR 的四种场景

ANR 的四种场景：

**Service TimeOut**: service 未在规定时间执行完成：前台服务 `20s`，后台 `200s`

**BroadCastQueue TimeOut**: 未在规定时间内未处理完广播：前台广播 `10s` 内, 后台 `60s` 内

**ContentProvider TimeOut**: publish 在 `10s`内没有完成

**Input Dispatching timeout**: `5s` 内未响应键盘输入、触摸屏幕等事件
# 十、Activty间传递数据的方式

1.通过 **Intent** 传递（Intent.putExtra 的内部也是维护的一个 Bundle，因此，通过 putExtra 放入的 数据，取出时也可以通过 Bundle 去取）
2.通过**全局变量**传递
3.通过 **SharedPreferences** 传递
4.通过**数据库**传递
5.通过**文件**传递

# 十一、跨App启动Activity注意事项

[https://juejin.im/post/6844904056461197326#heading-0](https://juejin.im/post/6844904056461197326#heading-0)


# 十二、Activity任务栈

1.android任务栈又称为Task，它是一个**栈结构**，具有**后进先出**的特性，用于存放我们的Activity组件。

2.我们每次打开一个新的Activity或者退出当前Activity都会在一个称为任务栈的结构中添加或者减少一个Activity组件， 一个任务栈包含了一个activity的集合, **只有在任务栈栈顶的activity才可以跟用户进行交互**。

3.在我们退出应用程序时，必须把所有的任务栈中**所有**的activity清除出栈时,任务栈才会被销毁。当然任务栈也可以移动到后台, 并且保留了每一个activity的状态. 可以有序的给用户列出它们的任务, 同时也不会丢失Activity的状态信息。

4.对应AMS中的ActivityRecord、TaskRecord、ActivityStack(AMS中的总结)

5.android通过ActivityRecord、TaskRecord、ActivityStack，ActivityStackSupervisor，ProcessRecord有序地管理每个activity。



# 十三、Activity的透明性和不透明性

![[Pasted image 20250724111953.png]]


## 1.视觉表现

|**特性**|**不透明 Activity (默认)**|**透明 Activity**|
|---|---|---|
|**背景显示**|完全遮挡下层 Activity|允许下层 Activity 部分或全部可见|
|**窗口背景**|使用主题定义的 `windowBackground`|通常设置为透明或半透明背景|
|**状态栏/导航栏**|默认不透明|可设置为透明（需配合 `FLAG_TRANSLUCENT_STATUS`）|

## 2.对下层 Activity 生命周期的影响

|**行为**|**不透明 Activity**|**透明 Activity**|
|---|---|---|
|**覆盖下层 Activity**|下层触发 `onStop()`|下层仅触发 `onPause()`，**不触发 `onStop()`**|
|**返回键退出**|下层从 `onStop()` 恢复|下层直接从 `onPause()` 恢复|
|**内存管理**|系统可能回收下层资源|下层资源保留（因未进入 `onStop()`）|

**关键区别**：  
透明 Activity 不会让下层 Activity 进入 `onStop()`，因此：
- **优点**：下层 Activity 的视图和数据保持活跃（如音乐播放器界面）。
- **缺点**：可能增加内存占用（下层未被销毁）。
## 3.常见使用场景

|**场景**|**不透明 Activity**|**透明 Activity**|
|---|---|---|
|**常规页面**|主界面、列表页、设置页|弹窗、引导页、悬浮窗口|
|**系统交互**|全屏输入、普通对话框|半透明登录框、权限提示|
|**视觉特效**|无特殊效果|磨砂玻璃效果、背景模糊|

## 4.性能与注意事项

| **维度**     | **不透明 Activity** | **透明 Activity** |
| ---------- | ---------------- | --------------- |
| **渲染性能**   | 更高（无图层混合开销）      | 较低（需合成透明图层）     |
| **内存占用**   | 下层可能被回收          | 下层常驻内存          |
| **开发注意事项** | 无需特殊处理           | 需处理焦点冲突、透明状态栏适配 |

# 十四、Activity生命周期

1. **onCreate()**
    - Activity 首次创建时调用，用于初始化布局和数据绑定。
    - 必须实现，因为它是 Activity 的入口点。

2. **onStart()**
    - Activity 进入“可见但不可交互”状态（例如被其他半透明 Activity 覆盖）。
    - 用户可能还未直接与 Activity 交互。

3. **onResume()**
    - Activity 进入前台并变为**可交互**状态（获得焦点）。
    - 适合在此处恢复动画、传感器监听等。

4. **onPause()**
    - Activity **失去焦点**（例如被另一个 Activity 部分覆盖）。
    - 应快速释放资源或保存临时数据（不能耗时操作）。

5. **onStop()**
    - Activity 完全不可见（例如被其他 Activity 完全覆盖或应用退到后台）。
    - 可在此释放非关键资源。

6. **onDestroy()**
    - Activity 被销毁前调用（用户主动退出或系统回收资源）。
    - 用于最终清理（如解绑服务、释放内存）。

7. **onRestart()**
    - Activity 从 `onStop()` 后重新回到前台时调用（例如用户按返回键回到之前 Activity）。
    - 通常后接 `onStart()`。

![[Pasted image 20250724144814.png]]