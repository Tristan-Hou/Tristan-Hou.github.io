---
layout: post
title: Xndroid学习篇(1)——初次编译
data: 2017-12-17
tags: 
    - Android
    - NDK
categories: NDK
---

## 简介

* 项目地址：[Xndroid](https://github.com/XndroidDev/Xndroid)
* 作者：[XndroidDev](https://github.com/XndroidDev)

### 编译
首次fork该项目，并进行编译时，编译时不能通过的我用的是最新的Andorid Studio 3.0）——当然有一个原因是我的NDK路径不对，汗~     

<!-- more -->

* 首先，直接sync会报错： 

![](https://raw.githubusercontent.com/Tristan-Hou/MarkdownImg/master/res/xndroid-shrinker-error.jpg)
>Error:Resource shrinker cannot be used for libraries.

原因是library中使用了混淆，并移除了无用的资源文件，解决办法就是把该library中的`shrinkResources`字段移除——按理说这个字段应该加上，但没想到其他更好的办法，暂时移除吧。 
    
接下来应该可以sync成功了。

* 随后开始编译，编译失败：

![](https://raw.githubusercontent.com/Tristan-Hou/MarkdownImg/master/res/xndroid-compileNdk.jpg)
>:app:ompileDebugNdk

根据提示，我们可以知道这是由于`gradle.properties`文件中使用了`useDeprecatedNdk`，而这个东西已经“no longer supported and will removed in the next version”。所以我们有两个解决办法：

1. 使用`CMake` 和 `ndk-build integration`;
2. 使用`android.deprecatedNdkCompileLease=xxx`这么一个东西，“for another 60 days”

我们先实验第二种方法，直接添加“android.deprecatedNdkCompileLease=1511832698813”到gradle.properties文件中，编译成功。

但是这种方法貌似只能延续60天？那60天后呢？所以为了长久之计，我们还是该使用第一种方法——CMake。

*********
我是参考这个人的博客[《NDK开发之------Error: Flag android.useDeprecatedNdk is no longer supported爬坑》](http://blog.csdn.net/qiantanlong/article/details/78622990)实现的。具体做法就是：

1. 先将app项目下的`build.gradle`文件内容按该博客所讲，补齐。
   
        android {
            compileSdkVersion 26
            buildToolsVersion '26.0.2'
            defaultConfig {
                applicationId "net.xndroid"
                minSdkVersion 14
                targetSdkVersion 26

               ndk {
                  moduleName "sockvpn"  //设置库(so)文件名称
                  //设置支持的so库架构
                  abiFilters "armeabi-v7a", "arm64-v8a", "armeabi"
                  ldLibs "log"
              }
              externalNativeBuild {
                  cmake {
                      cppFlags ""
                  }
              }

            versionCode 13
            versionName "1.1.3-3"
            testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
            }
            buildTypes {
                release {
                    minifyEnabled false
                    proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
                }
            }
            sourceSets.main {
                jniLibs.srcDirs = ['src/main/jni']
            }
            externalNativeBuild {
                cmake {
                    path "CMakeLists.txt"
                }
            }
        }
    
其中`moduleName "sockvpn"`对应了`src/main/jni`下的`sockvpn`文件名，也就是编译出来侯的.so文件名，当然你可以起其他名字。

`jniLibs.srcDirs = ['src/main/jni']`是你的jni路径。当然你也看到了，这里还有一个文件`CMakeLists.txt`，你需要在app项目的根目录创建它。

2. 创建`CMakeLists.txt`：
   
        # For more information about using CMake with Android Studio, read the
        # documentation: https://d.android.com/studio/projects/add-native-code.html

        # Sets the minimum version of CMake required to build the native library.

        cmake_minimum_required(VERSION 3.4.1)

        # Creates and names a library, sets it as either STATIC
        # or SHARED, and provides the relative paths to its source code.
        # You can define multiple libraries, and CMake builds them for you.
        # Gradle automatically packages shared libraries with your APK.

        add_library( # Sets the name of the library.
                     sockvpn

                     # Sets the library as a shared library.
                     SHARED
        
                     # Provides a relative path to your source file(s).
                     src/main/jni/sockvpn.c )

        # Searches for a specified prebuilt library and stores the path as a
        # variable. Because CMake includes system libraries in the search path by
        # default, you only need to specify the name of the public NDK library
        # you want to add. CMake verifies that the library exists before
        # completing its build.

        find_library( # Sets the name of the path variable.
                      log-lib

                      # Specifies the name of the NDK library that
                      # you want CMake to locate.
                      log )

        # Specifies libraries CMake should link to your target library. You
        # can link multiple libraries, such as libraries you define in this
        # build script, prebuilt third-party libraries, or system libraries.

        target_link_libraries( # Specifies the target library.
                               sockvpn

                               # Links the target library to the log library
                               # included in the NDK.
                               ${log-lib} )
你只需要将上面博客里所讲的一些模块名字改为`sockvpn`即可。

至于他说的`native-lib.cpp`，我们不需要。

3. 编译，成功。
![](https://raw.githubusercontent.com/Tristan-Hou/MarkdownImg/master/res/xndroid-compile-success.jpg)

##总结
刚开始接触NDK，很多东西都不懂，因此这么一个小小的编译，都浪费了许多时间。
这里所说的`CMakeLists.txt`内容，我并不能完全懂，还有项目里的`Android.mk`/`Application.mk`，我也不是很懂。
`所以说，还是要多学习一个啊！提高姿势水平~`






