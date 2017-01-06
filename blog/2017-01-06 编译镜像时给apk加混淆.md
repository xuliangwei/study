# 编译镜像时给apk加混淆

---

## 打开混淆开关
Android.mk中经常会看到
```
LOCAL_PROGUARD_ENABLED := full
```
这一句，从字面上理解，是开启混淆。如果你以为这样编译出来的apk就已经被混淆了，结果会让你很困惑。将apk里的classes.dex反编译，会看到代码仍然是混淆之前的状态，甚至连文件大小都没有变化。这行语句加不加，效果是一样的。
事实上，系统编译时，如果LOCAL_PROGUARD_ENABLED没有设置，除非环境变量中显式地定义了DISABLE_PROGUARD为true，LOCAL_PROGUARD_ENABLED默认值就是full。
参见：build/core/package_internal.mk
```
LOCAL_PROGUARD_ENABLED:=$(strip $(LOCAL_PROGUARD_ENABLED))
ifndef LOCAL_PROGUARD_ENABLED
ifneq ($(DISABLE_PROGUARD),true)
    LOCAL_PROGUARD_ENABLED :=full
endif
endif
```
那为什么混淆不起作用呢？
>ProGuard是一个压缩、优化和混淆Java字节码文件的免费的工具，它可以删除无用的类、字段、方法和属性。可以删除没用的注释，最大限度地优化字节码文件。它还可以使用简短的无意义的名称来重命名已经存在的类、字段、方法和属性。常常用于Android开发用于混淆最终的项目，增加项目被反编译的难度。

默认情况下，系统关闭了ProGuard的混淆功能，将其仅用于压缩。
参考build/core/java.mk
```
ifeq ($(filter obfuscation,$(LOCAL_PROGUARD_ENABLED)),)
# By default no obfuscation
proguard_flags += -dontobfuscate
endif  # No obfuscation
ifeq ($(filter optimization,$(LOCAL_PROGUARD_ENABLED)),)
# By default no optimization
proguard_flags += -dontoptimize
endif  # No optimization
```
要想将混淆开关打开，需要在Android.mk中增加：
```
LOCAL_PROGUARD_ENABLED := full obfuscation
```
这时候再编译，就会看到类和方法大部分变成了a,b,c之类无意义的短变量，完成了混淆的功能。同时带来的好处是，apk的体积大约有20%的减小。

## 解决混淆带来的问题
### 问题：Warning: can't find referenced
如果应用中有引用的第三方jar库，在编译时会有可能碰到类似
“Warning: can't find referenced class”
之类的编译警告，导致编译失败，解决方案是要通过混淆配置文件来保留某些第三方的库不要被混淆。
首先在Android.mk中增加一行来指定混淆配置文件：
```
LOCAL_PROGUARD_FLAG_FILES := proguard.flags
```
混淆配置文件依惯例命名为proguard.flags，在Android.mk同目录下建立proguard.flags，将要保持不被混淆的类或成员加进来。示例：
```
-dontwarn com.google.zxing.**       # 消除针对com.google.zxing的所有警告
-keep class com.google.zxing.** {   # 保留com.google.zxing所有类的命名
    *;                              # 保留com.google.zxing的所有类的成员的命名
}
-verbose
```
-dontwarn是消除警告，-keep是保留指定类和成员的命名，具体语法请参考ProGuard的语法说明，这里就不介绍了。

### json序列化一个类时的要注意保留类的成员变量不被混淆
json的序列化函数会直接使用类的变量的命名，如果这个类被混淆了，会导致序列化出来的文件可读性比较差，需要在proguard.flags中将这个类的成员变量设置为保留。
