https://blog.csdn.net/wenzhi20102321/article/details/135631544?spm=1001.2014.3001.5502
# 一、简介

Android.mk就是一个GNU make语法的脚本文件，用来编译出类库.jar,应用文件.apk，动态库.so，静态库.a作用。其中关键的就是include类型和不同的属性定义。

# 二、Android.mk 文件模版

## 1.编译.jar包

```
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
# 设置模块名为mylibrary
LOCAL_MODULE := mylibrary
# 添加需要编译的Java源文件
LOCAL_SRC_FILES := $(wildcard *.java)
# 设置编译目标为Java类文件
LOCAL_JAVA_FILES := $(LOCAL_SRC_FILES)
# 设置Java编译标志
LOCAL_JAVA_FLAGS := -source 1.8 -target 1.8
# 设置输出目录
LOCAL_MODULE_CLASS := classes
# 设置生成的jar包名称
LOCAL_MODULE := mylibrary.jar
include $(BUILD_JAVA_LIBRARY)
```

## 2.编译apk

### （1）以apk编译apk？

```
# 编译的目录，默认写法
include $(CLEAR_VARS)

# 设置模块名
LOCAL_MODULE := mymodule

# 设置预构建文件的源路径和目标路径
LOCAL_SRC_FILES := path/file.apk
LOCAL_MODULE_PATH := path/to/output/directory

# 设置系统权限和目录
LOCAL_CERTIFICATE := platform
LOCAL_PRIVATE_PLATFORM_APIS := true
# 设置是否编译在 priv-app,如果没有指定目录默认为 system/priv-app
LOCAL_PRIVILEGED_MODULE := true 

include $(BUILD_PREBUILT)

```

使用场景：
比如：普通应用签名成系统签名应用，普通应用放置到system/priv-app目录，获取系统权限等等。

### （2）以java源码编译apk

```
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)

# 设置模块名为myapp
LOCAL_MODULE := myapp
# 添加需要编译的Java源文件
LOCAL_SRC_FILES := $(wildcard *.java)
# 添加需要编译的资源文件
LOCAL_RESOURCE_DIR := $(LOCAL_PATH)/res
# 设置输出APK的路径和名称
LOCAL_PACKAGE_NAME := myapp
LOCAL_PACKAGE_OUTPUT_FILE := $(LOCAL_PATH)/$(LOCAL_PACKAGE_NAME).apk

# 设置系统权限和目录
LOCAL_CERTIFICATE := platform
LOCAL_PRIVATE_PLATFORM_APIS := true
# 设置是否编译在 priv-app,如果没有指定目录默认为 system/priv-app
LOCAL_PRIVILEGED_MODULE := true 

include $(BUILD_PACKAGE)

```

在源码环境中基本是没有用到 LOCAL_MODULE_PATH 属性的，这个属性是指定apk编译到哪个位置，

并且在系统源码中添加这个属性是没有作用的，这个位置指的是代码对应的位置，一般是本地编译使用有可能用到。

源码中的代码是在package/app/XXX下编译，编译后就会直接生成out下的，烧录后会运行到system/app 、system/priv-app目录。


## 3.编译动态库.so

```
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
# 设置模块名
LOCAL_MODULE := mylibrary
# 添加源文件
LOCAL_SRC_FILES := myfile.cpp
# 添加额外的依赖库
LOCAL_LDLIBS := -llog
include $(BUILD_SHARED_LIBRARY)

```

## 4.编译静态库.a

```
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
# 设置模块名
LOCAL_MODULE := mylibrary
# 添加源文件
LOCAL_SRC_FILES := myfile.cpp
include $(BUILD_STATIC_LIBRARY)

```

上面举例的都是最简单的示例。实际使用中肯定比上面的都复杂，但是都是一些属性的变化而已。

并且同一个mk文件是可以同时编译so和apk的。

## 5.Android.mk 编译文件小结

```
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)
# 设置模块名或者apk名称
LOCAL_MODULE := mylibrary

#根据情况设置相关编译属性
...
...

#编译相关类型
include $(TYEPXXX) 

#------------多个模块编译的情况，先清除属性
include $(CLEAR_VARS)
# 设置模块名或者apk名称
LOCAL_MODULE := mylibrary2

#根据情况设置相关编译属性
...
...

#编译相关类型
include $(TYEPXXX2)

```

所有的Android.mk都是基于上面的模版进行适配的。
除了头文件的固定写法：“LOCAL_PATH := $(call my-dir)”, 后续都是先清除属性，再编译某个类型模块…。
属性的定义就是 “XXX := YYY”, 注释使用 # 在前面表示注释标识的。
在实际的文件中可能有几百行，这一般是多个模块编译的情况，
不用头晕，只看自己相关模块那段代码，
每个模块都有一个模块名称 LOCAL_MODULE ，找到开始和结束的部分适配修改即可。

 **最后编译的：include $(TYEPXXX)类型总结：**
```
编码类型和关键字
1、apk文件,BUILD_PREBUILT
2、app代码,BUILD_PACKAGE
3、动态库,BUILD_SHARED_LIBRARY
4、静态库,BUILD_STATIC_LIBRARY
5、代码编译成Jar包,BUILD_JAVA_LIBRARY
6、jar包编译到系统,BUILD_MULTI_PREBUILT
```

常用的编译类型基本就上面这些。

**Android系统源码编译Android.mk文件方式：**

（1）在源码release目录，输入 make -j64 “LOCAL_MODULE名称”

（2） cd 到Android.mk模块目录，输入"mm"

# 三、Android.mk 示例

## 1.编译某个apk文件到系统中

```
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)

#模块名称LOCAL_MODULE ，这里是 apk 名称
LOCAL_MODULE := AMSDemo

#指定模块标签 LOCAL_MODULE_TAGS，基本都是用 optional 即可
LOCAL_MODULE_TAGS := optional

#源文件 LOCAL_SRC_FILES， apk文件和目录（当前目录）
LOCAL_SRC_FILES := $(LOCAL_MODULE).apk

#模块类别 LOCAL_MODULE_CLASS ，常见的模块类别包括"EXECUTABLES"、"SHARED_LIBRARIES"、"STATIC_LIBRARIES"、"JAVA_LIBRARIES"、"APPS"等。
LOCAL_MODULE_CLASS := APPS

# LOCAL_MODULE_SUFFIX 用于指定模块的后缀,这里是 Android 的.apk
LOCAL_MODULE_SUFFIX := $(COMMON_ANDROID_PACKAGE_SUFFIX)

#系统签名 LOCAL_CERTIFICATE ，
# presigned 表示预签名，已经签名的apk，编译不会重新签名
# platform 表示需要系统名，编译后会重新签名
LOCAL_CERTIFICATE := platform

#是否特权应用 LOCAL_PRIVILEGED_MODULE，是否生成到system/priv-app ，不写就是生成在app目录
LOCAL_PRIVILEGED_MODULE := true

#重新编译apk文件
include $(BUILD_PREBUILT)

```

上面就是一个普通apk添加系统签名和编译到系统目录的脚本代码

## 2、编译某个app源码到系统中

```
LOCAL_PATH:= $(call my-dir)

include $(CLEAR_VARS)

LOCAL_PACKAGE_NAME := MyMenuUi
LOCAL_MODULE_TAGS := optional

# AndroidManifest 文件目录位置定义，如果位置未移动，如果一般可以不用写
#LOCAL_MANIFEST_FILE := $(LOCAL_PATH)/app/src/main/AndroidManifest.xml

# res 资源文件目录位置定义
LOCAL_RESOURCE_DIR := \
    $(LOCAL_PATH)/res

# Java资源文件目录位置定义，如果有aidl也是要定义
LOCAL_SRC_FILES := \
    $(call all-java-files-under, java) \
   $(call all-Iaidl-files-under, aidl)

#资源编译相关的选项 LOCAL_AAPT_FLAGS，想要打包哪些资源到模块里面
LOCAL_AAPT_FLAGS := \
  --auto-add-overlay \
  --extra-packages androidx.constraintlayout_constraintlayout \
  --extra-packages androidx.recyclerview_recyclerview

#启用AAPT2作为资源打包工具，不设置默认为 AAPT，源码安装了AAPT2的情况，使用AAPT2 会高效一点。
LOCAL_USE_AAPT2 := true

# 加载系统存在的Android静态库
LOCAL_STATIC_ANDROID_LIBRARIES := \
    androidx.recyclerview_recyclerview \
    com.google.android.material_material

# 加载系统存在的Java静态库，gson_menuui 是预编译进去的
LOCAL_STATIC_JAVA_LIBRARIES := \
	androidx.appcompat_appcompat \
	gson_menuui

#系统签名
LOCAL_PRIVATE_PLATFORM_APIS := true
LOCAL_CERTIFICATE := platform
#生成到system/priv-app目录
LOCAL_PRIVILEGED_MODULE := true

include $(BUILD_PACKAGE)

#把已有的jar预编译Java包使用
include $(CLEAR_VARS)

LOCAL_PREBUILT_STATIC_JAVA_LIBRARIES := \
		gson_menuui:../../libs/gson-2.8.0.jar

include $(BUILD_MULTI_PREBUILT)

```

上面就是一个app源码编译到编译到系统目录的具体的脚本代码，上面的代码都是系统源码中进行编译的。

# 四、Android.mk 主要属性

Android.mk可以根据项目的需求进行配置和调整，用于定义模块的构建规则、依赖关系、标签、签名配置等。通过合理地设置这些属性，可以控制模块的编译、链接、打包和签名等行为，以满足项目的需求。

以下是一些常见的主要属性的详细解释：

```
1. LOCAL_PATH：指定当前Makefile文件所在的路径。
2. LOCAL_MODULE：指定模块的名称。
3. LOCAL_SRC_FILES：指定模块的源文件列表。
4. LOCAL_C_INCLUDES：指定C/C++源文件的包含路径。
5. LOCAL_SHARED_LIBRARIES：指定模块依赖的共享库。
6. LOCAL_STATIC_LIBRARIES：指定模块依赖的静态库。
7. LOCAL_MODULE_TAGS：指定模块的标签，用于在构建系统中对模块进行分类和归类。
8. LOCAL_MODULE_CLASS：指定模块的类别，表示模块在构建系统中的类型或角色。
9. LOCAL_MODULE_SUFFIX：指定模块的文件名后缀。
10. LOCAL_CERTIFICATE：指定模块的签名配置。
11. LOCAL_USE_AAPT2：指示是否使用AAPT2作为资源打包工具。
12. LOCAL_PACKAGE_NAME：指定生成APK文件的包名。
13. LOCAL_PACKAGE_SIGNED_KEY：指定APK文件的签名配置，如"release"或"debug"。
```

详细的可以看看：https://blog.csdn.net/pashanhu6402/article/details/96584229

Android 官方对于Android.mk 的介绍：https://developer.android.google.cn/ndk/guides/android_mk?hl=zh-cn#local_src_files

不同类型编译脚本的内容天差地别，相同类别的编译一般就几个属性的差异。

其实模版是固定的，主要就是要用对属性，用的时候进行具体查询即可。

并且不同的编译环境可能有的属性不生效或者编译异常，这些都是有可能遇到的问题。

# 五、总结

## 1、Android.mk 的简单使用总结

除了头文件的固定写法：“LOCAL_PATH := $(call my-dir)”, 后续都是先清除属性，再编译某个类型模块…。

属性的定义就是 “XXX := YYY”。

大致模版：
```
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)
# 设置模块名或者apk名称
LOCAL_MODULE := mylibrary

#根据情况设置相关编译属性
...
...

#编译相关类型
include $(TYEPXXX) 

```

最后编译的：include $(TYEPXXX)类型总结：

```
编码类型和关键字
1、apk文件,BUILD_PREBUILT
2、app代码,BUILD_PACKAGE
3、动态库,BUILD_SHARED_LIBRARY
4、静态库,BUILD_STATIC_LIBRARY
5、代码编译成Jar包,BUILD_JAVA_LIBRARY
6、jar包编译到系统,BUILD_MULTI_PREBUILT
```

## 2、Android.mk 详解

前言那里对Android.mk只是简单提了一句介绍，下面的介绍是更加官方一点和详细的介绍。

Android.mk文件是Android开发中的一个编译脚本文件，用于描述和管理项目的编译过程。

下面是Android.mk文件的一些详解：

```
概述：Android.mk文件是使用GNU make语法编写的，它定义了项目中的模块和编译规则。每个模块都有一个对应的Android.mk文件，用于指定编译参数、依赖关系和生成的目标文件等。

模块定义：Android.mk文件中可以定义多个模块，每个模块都有一个唯一的模块名。模块可以是可执行文件、静态库、共享库等。通过LOCAL_MODULE变量来指定模块名。

源文件定义：Android.mk文件中使用LOCAL_SRC_FILES变量来指定模块的源文件。可以使用通配符来匹配多个文件，如*.cpp表示所有的cpp文件。

编译选项：Android.mk文件中可以指定编译选项，如指定编译器、编译标志等。通过LOCAL_CFLAGS、LOCAL_CPPFLAGS、LOCAL_LDFLAGS 等变量来设置。

依赖关系：Android.mk文件中可以指定模块的依赖关系，即一个模块依赖于其他模块。通过LOCAL_STATIC_LIBRARIES、LOCAL_SHARED_LIBRARIES等变量来指定依赖的静态库或共享库。

目标文件生成：Android.mk文件中可以指定生成的目标文件的名称和路径。通过LOCAL_MODULE_FILENAME、LOCAL_MODULE_PATH等变量来设置。

其他功能：Android.mk文件还支持其他一些功能，如指定需要编译的源文件、排除某些源文件、指定编译器、链接器等。可以通过查阅Android.mk文件的官方文档或相关教程来了解更多功能和用法。
```

总之，Android.mk文件是Android开发中的一个重要文件，用于管理项目的编译过程。通过编写Android.mk文件，可以指定模块的编译选项、依赖关系和生成的目标文件等。这样可以方便地管理和组织项目的代码和资源。

## 3、Android.mk的其他知识

（1）Android.mk编译的模块类别和属性定义字符串都是大写的
（2）编译的属性定义对应的值的字符串，大小写是敏感的，根据实际情况写即可

如果写错，会编译不通过，有的只是没有作用而已，所以发现没有作用的情况可以检测是否有字符串写错了。

比如：

```
LOCAL_PATH:= $(call my-dir) //固定写法的不用管
include $(CLEAR_VARS)
LOCAL_PACKAGE_NAME := MyMenuUi //属性定义大写，对应的值根据情况写
LOCAL_MODULE_TAGS := optional //固定的写法之一
```

（3）Android 源码编译 及 mk文件解读

https://blog.csdn.net/sunlei663/article/details/114104154

（4）Android.mk和Android.bp的关系

Android.mk和Android.bp都是Android项目中的构建脚本文件，用于管理项目的编译过程。它们之间有以下关系和区别：
关系：
```
Android.mk是较早版本的Android构建系统使用的构建脚本文件，它使用GNU make语法编写。
Android.bp是Android 7.0及更高版本引入的新一代Android构建系统使用的构建脚本文件，它使用Starlark语法编写。
区别：

语法：Android.mk使用GNU make语法，而Android.bp使用Starlark语法。Starlark是一种基于Python的轻量级脚本语言。

灵活性：相对于Android.mk，Android.bp在模块定义、依赖关系和编译选项等方面更加灵活和易于维护。Android.bp具有更强大的表达能力，可以更好地处理复杂的项目结构和构建需求。

版本兼容性：Android.mk适用于Android的早期版本，而Android.bp适用于Android 7.0及更高版本。在新版Android中，建议使用Android.bp来管理和构建项目。

向后兼容性：Android.bp仅在Android 7.0及更高版本引入，而Android.mk仍然可以继续使用。为了保持向后兼容性，Android构建系统可以同时支持Android.mk和Android.bp文件，可以在同一个项目中同时使用两种格式的构建文件。

综上所述，Android.mk和Android.bp是两种不同版本的Android构建脚本文件，它们之间有一定的关系和区别。Android.bp是更加灵活和推荐的构建脚本文件，在较新的Android版本中使用。但为了向后兼容性，Android.mk仍然可以继续使用，并且可以与Android.bp文件同时存在于同一个项目中。
```

