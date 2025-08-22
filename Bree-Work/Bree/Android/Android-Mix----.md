# 一、开机动画 

![[Pasted image 20250715203606.png]]

# 二、编译固件出现版本apk错误

```
# Bree patch start

rm -drf ../aml_sdk/out/target/product/a2/obj/APPS/Fj*

echo "bree rm success=================="

# Bree patch end
```

# 三、APK签名

setprop persist.sys.allow.install 1 

![[img_v3_02nk_50fd343c-6f8b-4793-90a7-9a03c1fbb59g.jpg]]

![[img_v3_02nk_865f61fb-3b52-4d1c-8239-18707a593b5g.jpg]]



给apk签名

```
java -jar apksigner.jar sign --ks E:\Android\ap9\mf\ap9.jks --ks-key-alias android --ks-pass pass:android --key-pass pass:android --out new_wfd.apk E:\Android\ap9\mf\WFDSinkTest.apk
（常用）

java -jar signapk.jar platform.x509.pem platform.pk8 E:\Android\ap9\mf\screensharereceiver_unsigned.apk screensharereceiver.apk
```

```
java -jar apksigner.jar sign --ks 签名路径.jks --ks-key-alias alias名称 --ks-pass pass:密码 --key-pass pass:密码 --out 签名后的新包路径.apk 待签名的包路径.apk
```

```
java -jar apksigner.jar sign --ks platform.jks --ks-key-alias fj --ks-pass pass:fj171216 --key-pass pass:fj171216 --out new111.apk Hasim.apk
```

```
查看签名：

java -jar apksigner.jar verify -v Hasim.apk
```

```
如上图所示，apksigner.jar在Android SDK的安装目录下(\Android\Sdk\build-tools\33.0.0\lib)。
我的安装目录是：
C:\Users\Administrator\AppData\Local\Android\Sdk\build-tools\33.0.0\lib
1）使用cmd打开以上文件路径
cd C:\Users\Administrator\AppData\Local\Android\Sdk\build-tools\33.0.0\lib
2）查看apk签名情况
java -jar apksigner.jar verify -v apk路径.apk
这是未签名的APK返回的的结果：
DOES NOT VERIFY
ERROR: Missing META-INF/MANIFEST.MF
这是v1签名的APK返回的的结果：
Verifies
Verified using v1 scheme (JAR signing): true
Verified using v2 scheme (APK Signature Scheme v2): false
这是v1v2都签名的APK返回的的结果：
Verifies
Verified using v1 scheme (JAR signing): true
Verified using v2 scheme (APK Signature Scheme v2): true
3）对加固后的apk进行重新签名
java -jar apksigner.jar sign --ks 签名路径.jks --ks-key-alias alias名称 --ks-pass pass:密码 --key-pass pass:密码 --out 签名后的新包路径.apk 待签名的包路径.apk
```

# 四、apk install流程

```
processPendingInstall
```

```
    private void processPendingInstall(final InstallArgs args, final int currentStatus) {
        // Queue up an async operation since the package installation may take a little while.
        mHandler.post(new Runnable() {
            public void run() {
                mHandler.removeCallbacks(this);
                 // Result object to be returned
                PackageInstalledInfo res = new PackageInstalledInfo();
                res.setReturnCode(currentStatus);
                res.uid = -1;
                res.pkg = null;
                res.removedInfo = null;
                if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
                    args.doPreInstall(res.returnCode);
                    synchronized (mInstallLock) {
                        installPackageTracedLI(args, res);
                    }
```

# 五、将app设置成系统应用

