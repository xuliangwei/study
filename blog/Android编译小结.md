正式开始一个新的平台，以前的项目拿到的代码都是供应商改好的，感觉有很多不规范的地方。

这次从一个全新的项目着手，严格按照android规范进行项目的添加、板级文件的支持。目标是争取不改动build目录下的文件，而是用好它的扩展机制。

### 在lunch中增加combo选项的标准方法
在执行完. build/envsetup.sh后，执行lunch，会出现lunch菜单：
```
You're building on Linux

Lunch menu... pick a combo:
     1. aosp_arm-eng
     2. aosp_arm64-eng
     3. aosp_mips-eng
     4. aosp_mips64-eng
     5. aosp_x86-eng
     6. aosp_x86_64-eng
     7. aosp_flounder-userdebug
     8. mini_emulator_mips-userdebug
     9. mini_emulator_x86-userdebug
     10. mini_emulator_arm64-userdebug
     11. mini_emulator_x86_64-userdebug
     12. m_e_arm-userdebug
     13. aosp_manta-userdebug
     ……
```
在build/envsetup.sh中只增加了默认的几项：
```sh
# add the default one here
add_lunch_combo aosp_arm-eng
add_lunch_combo aosp_arm64-eng
add_lunch_combo aosp_mips-eng
add_lunch_combo aosp_mips64-eng
add_lunch_combo aosp_x86-eng
add_lunch_combo aosp_x86_64-eng
```
lunch函数会到venor和device目录中寻找vendorsetup.sh，然后执行文件中类似的
```sh
add_lunch_combo aosp_flounder-userdebug
```
然后就会多出个
```
     7. aosp_flounder-userdebug
```

### Android 5.1可以自动识别JAVA_HOME
参见build/envsetup.sh，代码如下：
```sh
# Force JAVA_HOME to point to java 1.7 or java 1.6  if it isn't already set.
#
# Note that the MacOS path for java 1.7 includes a minor revision number (sigh).
# For some reason, installing the JDK doesn't make it show up in the
# JavaVM.framework/Versions/1.7/ folder.
function set_java_home() {
    # Clear the existing JAVA_HOME value if we set it ourselves, so that
    # we can reset it later, depending on the version of java the build
    # system needs.
    #
    # If we don't do this, the JAVA_HOME value set by the first call to
    # build/envsetup.sh will persist forever.
    if [ -n "$ANDROID_SET_JAVA_HOME" ]; then
      export JAVA_HOME=""
    fi

    if [ ! "$JAVA_HOME" ]; then
      if [ -n "$LEGACY_USE_JAVA6" ]; then
        case `uname -s` in
            Darwin)
                export JAVA_HOME=/System/Library/Frameworks/JavaVM.framework/Versions/1.6/Home
                ;;
            *)
                export JAVA_HOME=/usr/lib/jvm/java-6-sun
                ;;
        esac
      else
        case `uname -s` in
            Darwin)
                export JAVA_HOME=$(/usr/libexec/java_home -v 1.7)
                ;;
            *)
                export JAVA_HOME=/usr/lib/jvm/java-7-openjdk-amd64
                ;;
        esac
      fi

      # Keep track of the fact that we set JAVA_HOME ourselves, so that
      # we can change it on the next envsetup.sh, if required.
      export ANDROID_SET_JAVA_HOME=true
    fi
}
```
Android 5.1的编译使用了openjdk，在ubuntu上可以直接通过apt-get进行安装。

### 如何新建一个device
* 从原有的device目录复制一份，将目录命名为你要新建的设备名称。如把msm8916_64复制一份，重命名为msm8916_d500。
* 将msm8916_d500目录下的AndroidProducts.mk内容更新为：
```make
PRODUCT_MAKEFILES := \
    $(LOCAL_DIR)/msm8916_d500.mk
```
* 将msm8916_d500目录下原有的msm8916_64.mk重命名为msm8916_d500.mk。修改里面的PRODUCT_NAME和PRODUCT_DEVICE：
```make
PRODUCT_NAME := msm8916_d500
PRODUCT_DEVICE := msm8916_d500
```
* 在msm8916_d500目录下新建一个vendorsetup.sh，在里面增加combo选项，这样msm8916_d500就会出现在lunch菜单里。
```sh
add_lunch_combo msm8916_d500-userdebug
add_lunch_combo msm8916_d500-eng
add_lunch_combo msm8916_d500-user
```
* 修改msm8916_d500目录下的AndroidBoard.mk，修改KERNEL_DEFCONFIG，将其指向我们自己的defconfig文件：
```make
#----------------------------------------------------------------------
# Compile Linux Kernel
#----------------------------------------------------------------------
ifeq ($(KERNEL_DEFCONFIG),)
    ifeq ($(TARGET_BUILD_VARIANT),user)
      KERNEL_DEFCONFIG := msm8916_d500-perf_defconfig
    else
      KERNEL_DEFCONFIG := msm8916_d500_defconfig
    endif
endif
```
msm8916_d500_defconfig和msm8916_d500-perf_defconfig在kernel/arch/arm/configs目录下，不存在的话，要以msm8916_defconfig为模板新建一个，并在其中加入一行：
```make
CONFIG_BOARD_SEUIC_D500=y
```
这个变量会在导入dts中用到。
* kernel/arch/arm/boot/dts/qcom/Makefile中增加：
```make
dtb-$(CONFIG_BOARD_SEUIC_D500) += msm8916_d500.dtb
```

