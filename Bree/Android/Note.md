wifi和以太网如何共存
第一次开机对apk的加载，第二次开机对apk的加载
Launcher共存，有优先级
接口和抽象接口区别，什么时候用接口，什么时候用抽象
系统启动流程
点击apk是如何启动的
数据结构，节点，链表
透明Activity和不透明
View绘制过程




Activity与fragment的区别
# apk加载

## 二、首次开机特有流程

### 1. Dex优化阶段(OTA Dexopt)

- **执行动作**：

- /system/bin/dex2oat --dex-file=/data/app/xxx/base.apk --oat-file=/data/dalvik-cache/arm64/base.odex
    
- **生成文件**：
    
    - `/data/dalvik-cache/` 下的.odex/.vdex文件
        
    - `/data/app/` 下的优化后APK
        

### 2. 包扫描安装

- **系统服务**：`PackageManagerService`首次扫描
    
    java
    

- // frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java
    void scanDirLI(File scanDir, int parseFlags, int scanFlags, long currentTime)
    
- **扫描目录**：
    
    - `/system/app/` (系统应用)
        
    - `/system/priv-app/` (特权应用)
        
    - `/vendor/app/` (厂商应用)
        
    - `/data/app/` (用户应用)
        

### 3. 生成关键文件

- `/data/system/packages.list` (包列表)
    
- `/data/system/packages.xml` (包详细配置)
    
- `/data/system/users/0/package-restrictions.xml` (用户级限制)
    

## 三、后续开机流程

### 1. 快速加载机制

- **直接读取缓存**：
    
    java
    

- // 从已优化的odex加载
    PathClassLoader loader = new PathClassLoader(apkPath, libraryPath, parentLoader);
    
- **使用现有数据库**：
    
    sql
    

- SELECT * FROM packages WHERE name=?  // 直接查询现有数据库
    

### 2. 增量更新处理

- 只检查新增/更新的APK
    
- 执行部分Dex优化(仅对修改过的应用)



java 接口和抽象接口有什么区别，什么时候用接口，什么时候用抽象

# Java 接口与抽象类的区别及使用场景

## 主要区别

|特性|接口 (Interface)|抽象类 (Abstract Class)|
|---|---|---|
|定义关键字|`interface`|`abstract class`|
|方法实现|Java 8前不能有实现(默认方法除外)|可以有具体方法和抽象方法|
|变量|默认 `public static final`|无限制|
|构造方法|不能有|可以有|
|多继承|一个类可实现多个接口|一个类只能继承一个抽象类|
|访问修饰符|方法默认 `public`|方法可以有各种访问修饰符|
|Java 8+特性|支持默认方法和静态方法实现|同普通类|

## 何时使用接口

1. **需要多继承行为时**：一个类可以实现多个接口
    
2. **定义契约/规范时**：只定义方法签名而不关心实现
    
3. **定义回调机制时**：如事件监听器
    
4. **定义API时**：让不同实现类提供不同实现
    
5. **需要解耦时**：接口可以降低类之间的耦合度
    
6. **Java 8+的函数式编程**：函数式接口配合Lambda表达式
    

java

// 接口示例
public interface Drawable {
    void draw();  // 抽象方法
    default void resize() {  // 默认方法
        System.out.println("Resizing...");
    }
}

## 何时使用抽象类

1. **需要共享代码时**：多个相关类有共同代码可以放在抽象类中
    
2. **需要部分实现时**：有些方法可以实现，有些需要子类实现
    
3. **需要控制子类构造时**：抽象类可以有构造方法
    
4. **需要非public的成员时**：接口所有成员都必须是public的
    
5. **需要定义状态(实例变量)时**：接口只能有常量
    

java

// 抽象类示例
public abstract class Animal {
    private String name;  // 实例变量
    
    public Animal(String name) {  // 构造方法
        this.name = name;
    }
    
    public abstract void makeSound();  // 抽象方法
    
    public void eat() {  // 具体方法
        System.out.println(name + " is eating.");
    }
}

## 实际开发中的选择

- **优先考虑接口**：Java是单继承，接口更灵活
    
- **当需要以下特性时使用抽象类**：
    
    - 需要在多个相关类间共享代码
        
    - 需要定义非public的成员或方法
        
    - 需要定义状态(实例变量)
        
    - 需要控制子类的构造过程
        

Java 8以后，接口通过默认方法也能提供一些实现，进一步模糊了两者的界限，但核心区别仍然存在。

