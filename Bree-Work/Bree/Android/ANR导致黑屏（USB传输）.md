
### **2. 为什么必须同时修改两个锁？**

#### **场景复现（假设只改 `getExternalFilesDirs`）**

1. **线程A（主线程）**：
    
    - 调用 `getTheme()` → 持有 `mSync` 锁。
        
    - 若 `initializeTheme()` 内部间接调用 `getExternalFilesDirs()`（即使逻辑上不直接调用，但依赖路径可能隐藏）。
        
    - 尝试获取 `mStorageSync` 锁（已被线程B持有）。
        
2. **线程B（WetCheck线程）**：
    
    - 调用 `getExternalFilesDirs()` → 持有 `mStorageSync` 锁。
        
    - 执行 `sm.mkdirs()`（Binder阻塞）。
        
    - 若后续需要 `mTheme`（如日志记录），尝试获取 `mSync` 锁（已被线程A持有）。
        

**结果**：仍然死锁！


aml_sdk/frameworks/base/core/java/android/app/ContextImpl.java
```
private final Object mThemeSync = new Object();
private final Object mStorageSync = new Object();

@Override
public Resources.Theme getTheme() {
    synchronized (mThemeSync) {
        // ... 原有逻辑
    }
}

@Override
public File[] getExternalFilesDirs(String type) {
    synchronized (mStorageSync) {
        // ... 原有逻辑 + 异步化 Binder 调用
    }
}
```





方哥，关于之前那个USB传输黑屏的问题，目前在AT2 MAX上又复现到了，报错原因跟上次是一样的：



fjDynamic的WetCheck线程调用了getExternalFilesDirs->ensureExternalDirsExistOrFilter->sm.mkdirs
并持有mSync锁，同时mkdirs是耗时操作

系统主线程这边想调getTheme，等待mSync死锁了，导致anr黑屏





# 8.4分析


fjdynamic调用getExternalFilesDirs

aml_sdk/frameworks/base/core/java/android/content/ContextWrapper.java
```
@Override
    public File[] getExternalFilesDirs(String type) {
        return mBase.getExternalFilesDirs(type);
    }
```

aml_sdk/frameworks/base/core/java/android/app/ContextImpl.java
```
@Override
    public File[] getExternalFilesDirs(String type) {
        synchronized (mSync) {
            File[] dirs = Environment.buildExternalStorageAppFilesDirs(getPackageName());
            if (type != null) {
                dirs = Environment.buildPaths(dirs, type);
            }
            return ensureExternalDirsExistOrFilter(dirs);
        }
    }
```


main调用 ContextImpl.java中的 getTheme

```
@Override
    public Resources.Theme getTheme() {
        synchronized (mSync) {
            if (mTheme != null) {
                return mTheme;
            }

            mThemeResource = Resources.selectDefaultTheme(mThemeResource,
                    getOuterContext().getApplicationInfo().targetSdkVersion);
            initializeTheme();

            return mTheme;
        }
    }
```


getTheme也需要synchronized同步锁，但是ensureExternalDirsExistOrFilter调用sm.mkdirs(dir)，即

```
private File[] ensureExternalDirsExistOrFilter(File[] dirs) {
        final StorageManager sm = getSystemService(StorageManager.class);
        final File[] result = new File[dirs.length];
        for (int i = 0; i < dirs.length; i++) {
            File dir = dirs[i];
            if (!dir.exists()) {
                if (!dir.mkdirs()) {
                    // recheck existence in case of cross-process race
                    if (!dir.exists()) {
                        // Failing to mkdir() may be okay, since we might not have
                        // enough permissions; ask vold to create on our behalf.
                        try {
                            sm.mkdirs(dir);
                        } catch (Exception e) {
                            Log.w(TAG, "Failed to ensure " + dir + ": " + e);
                            dir = null;
                        }
                    }
                }
            }
            result[i] = dir;
        }
        return result;
    }
```

aml_sdk/frameworks/base/core/java/android/os/storage/StorageManager.java
```
/** {@hide} */
    public void mkdirs(File file) {
        try {
            mStorageManager.mkdirs(mContext.getOpPackageName(), file.getAbsolutePath());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```

SVEA V7 V8
Fjdynamic


```
@Override
    public Resources.Theme getTheme() {
        synchronized (mSync) {
            if (mTheme != null) {
                return mTheme;
            }

            mThemeResource = Resources.selectDefaultTheme(mThemeResource,
                    getOuterContext().getApplicationInfo().targetSdkVersion);
            initializeTheme();

            return mTheme;
        }
    }
```



![[Pasted image 20250707191704.png]]


```
@Override
    public File[] getExternalFilesDirs(String type) {
        synchronized (mSync) {
            File[] dirs = Environment.buildExternalStorageAppFilesDirs(getPackageName());
            if (type != null) {
                dirs = Environment.buildPaths(dirs, type);
            }
            return ensureExternalDirsExistOrFilter(dirs);
        }
    }
```

![[Pasted image 20250707191541.png]]


## **解决方案**

### 1. 拆分锁粒度

为 **UI 操作** 和 **文件操作** 使用 **不同的锁对象**：

```
private final Object mThemeLock = new Object();  // 专用于 Theme 的锁
private final Object mFileLock = new Object();  // 专用于文件操作的锁

public Resources.Theme getTheme() {
    synchronized (mThemeLock) {  // 使用独立锁
        if (mTheme != null) return mTheme;
        mThemeResource = Resources.selectDefaultTheme(...);
        initializeTheme();
        return mTheme;
    }
}

public File[] getExternalFilesDirs(String type) {
    synchronized (mFileLock) {  // 使用独立锁
        File[] dirs = Environment.buildExternalStorageAppFilesDirs(...);
        if (type != null) dirs = Environment.buildPaths(dirs, type);
        return ensureExternalDirsExistOrFilter(dirs);
    }
}
```



StorageManagerService

![[Pasted image 20250707195017.png]]

这个错误日志表明 Android 系统在尝试挂载外部存储设备（如 U 盘或 SD 卡）时遇到了严重问题。以下是详细的原因分析和解决方案：

![[Pasted image 20250707202154.png]]

![[Pasted image 20250707202211.png]]





张工，关于USB数据传输黑屏的问题，我这边看了一下StorageManagerService文件，这一块的逻辑是没有被修改过的。StorageManagerService的报错是因为尝试挂载的时候出现问题导致的，看起来感觉也是因为死锁导致的挂载失败。

分析原因主要是阻塞在mkdirs中，后续尝试将getExternalFilesDirs中缩小同步范围进行优化，将mkdir放在synchronized外面。

![[Pasted image 20250708111337.png]]

|   |   |   |   |
|---|---|---|---|
|**NetCheck** (tid=136)|`<0x09ab13f3>` (mSync)|-|阻塞在 `IStorageManager.mkdirs()` Binder 调用|


# 解决死锁


分析原因：

1.锁竞争机制

竞争资源：`<0x09ab13f3>`（即`ContextImpl.mSync`对象锁）

线程关系：

**主线程**：尝试获取该锁执行`getTheme()`（用于Activity界面绘制）
        
**线程136**：已持有该锁执行`getExternalFilesDirs()`中的存储操作



### **2. 关键问题点**

1. **锁竞争**
    
    - `ContextImpl.mSync` 锁被 `NetCheck` 线程长期持有（由于Binder调用阻塞）
        
    - 主线程需要同一把锁来执行 `getTheme()`
        
2. **Binder阻塞**
    
    - `StorageManager.mkdirs()` 是同步Binder调用
        
    - 如果存储服务响应慢，会阻塞调用线程
        
3. **调用链**
    
    text
    

getExternalFilesDirs()
→ ensureExternalDirsExistOrFilter()
  → StorageManager.mkdirs()
    → Binder.transact()  // 同步阻塞点


关键阻塞点

```
// ContextImpl.java
public Resources.Theme getTheme() {
    synchronized (mSync) {  // 主线程在此阻塞
        // ...
    }
}

public File[] getExternalFilesDirs() {
    synchronized (mSync) {  // 线程136长期持有
        // 包含Binder IPC调用（如StorageManager.mkdirs()）
    }
}
```

![[Pasted image 20250709114338.png]]





方案一：
```
// 修改ContextWrapper实现
override fun getExternalFilesDirs(type: String): Array<File> {
    val dirs = synchronized(mSync) {
        Environment.buildExternalStorageAppFilesDirs(packageName).apply {
            if (type != null) Environment.buildPaths(this, type)
        }
    }
    
    // 异步执行可能阻塞的操作
    lifecycleScope.launch(Dispatchers.IO) {
        ensureDirsExist(dirs) // 包含Binder调用
    }
    
    return dirs
}
```

- 锁持有时间从**500ms+**降至**<5ms**
    
- 完全解耦Binder阻塞与锁竞争



优化方案：
### **方案1：异步执行存储操作（推荐）**

**核心思想**：将可能阻塞的Binder调用移到锁外执行

```
@Override
public File[] getExternalFilesDirs(String type) {
    // 1. 快速计算路径（持锁时间<1ms）
    File[] dirs;
    synchronized (mSync) {
        dirs = Environment.buildExternalStorageAppFilesDirs(getPackageName());
        if (type != null) {
            dirs = Environment.buildPaths(dirs, type);
        }
    }

    // 2. 异步创建目录（通过Handler或线程池）
    new Thread(() -> {
        ensureExternalDirsExistOrFilter(dirs); // 包含Binder调用
    }).start();

    // 3. 直接返回未经校验的路径（根据业务需求调整）
    return dirs;
}
```

**优点**：

- 锁持有时间从 **100ms+** 降至 **<1ms**
    
- 完全避免因Binder阻塞导致的死锁
    

**注意事项**：

- 需处理目录未及时创建的情况（可通过回调通知）
    
- 适合对目录实时性要求不高的场景