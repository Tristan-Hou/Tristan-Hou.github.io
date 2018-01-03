---
layout: post
title: Android逆向分析笔记(1)
data: 2018-1-1
tags:
    - 虚拟机
categories: Android

---
最近在读《Android软件安全与逆向分析》，虽然不从事逆向研究的工作，但作为一名Android开发者，觉得了解一下相关知识还是有必要的。因此，这里记录下该书所阐述的主要知识点，方便记忆和理解。
<!-- more -->

## Dalvik虚拟机与Java虚拟机
### 区别：

1. Java： 代码-编译-java字节码-class文件，虚拟机解码class文件
   Dalvik：java字节码-Dalvik字节码-Dex，虚拟机解释Dex文件
2. Dalvik文件体积更小
   - dx工具将java字节码-Dalvik字节码
   - dx工具对java文件重新排序，消除类文件中的冗余信息
   - 例如：
       - 多个类文件相互引用，被引用的类文件中的方法签名会复制到引用类文件中;
       - 常量字符串在多个类中也会被重复引用
   - dx工具分解常量池，所有文件共享一个常量池
 
 ![](https://raw.githubusercontent.com/Tristan-Hou/MarkdownImg/master/res/1514813806959.jpg)

3. Java: 基于栈架构，需要频繁读写数据
   Dalvik：寄存器架构，数据访问通过寄存器直接传递（lua的VM也是寄存器实现)
   - 无论是栈虚拟机，还是寄存器虚拟机，都要：
       - 将源码编译为VM指定的字节码
       - 包含操作数，指令(处理操作数运算)，操作数数据结构
       - 一个为所有函数操作的调用栈
       - 指向下一条将要执行的指令位置的指令指针（PC计数器)——类似ARM架构cpu的PC寄存器与x86架构cpu的IP寄存器
       - 操控指令的虚拟CPU
           -  根据PC计数器获取下一条指令
           -  解析指令的具体含义(+/-/*/'/等)
           -  执行指令
   - 注意：
       - 两种虚拟机都一个PC计数器和调用栈：
           - Java：记录方法的调用，调用方法压入一帧，方法完成弹出。每一帧包括局部变量区(方法参数、局部变量)和求职栈(求值中间结果、别的方法参数)
           - Dalvik: 调用栈只维护一个寄存器列表
           
   示例代码：
   
		public class Hello {
			public int foo(int a, int b) {
				return (a + b) * (a - b);
			}
			public static void main(String[] argc) {
				Hello hello = new Hello();
				System.out.println(hello.foo(5, 3));
			}
		}

  dx --dex --output=Hello.dex Hello.class可生成dex文件
  javap -c -classpath . Hello查看foo的Java字节码：
  ![](https://raw.githubusercontent.com/Tristan-Hou/MarkdownImg/master/res/1514822513334.jpg)
  每条指令占1个字节，共8字节。
  dexdump查看foodalvik字节码：
  ![](https://raw.githubusercontent.com/Tristan-Hou/MarkdownImg/master/res/1514822614888.jpg)
  
   - java虚拟机：更多的指令、更多的cpu消耗、更具可移植性
   - Dalvik：更多的指令空间、数据缓冲更易失效、更有利于进行AOT（ahead-of-time）优化（相对JIT）
  
### Dalvik寄存器
- ARM架构：Dalvik部分寄存器映射到ARM寄存器上，部分通过调用栈模拟
- 32位寄存器，64位通过连续的两个寄存器实现
- 根据Dalvik指令表，一共有2的16次方，0~65535个寄存器
- 每个虚拟机为一个进程维护一个调用栈，每个函数在函数头使用.registers指令指定用到的寄存器数目，虚拟机为其分配占空间，模拟寄存器。

## Dalvik虚拟机特性
### 内存管理
- 内存分类：Java Object Heap、Bitmap Memory、Native Heap
  - Java Object Heap：
     - Java `new`出来的对象都存在这里。
     - Xms和Xmx分别控制最大/小值
     - 为了避免Dalvik运行时不断调整此Heap大小而影响性能，一般让Xms=Xmx
     - ActivityManager.getMemoryClass获得该值。这个Java Object Heap就是平日所说的`Android应用程序进程能够使用的最大内存`
     
  代码：
  						
        public class ActivityManager { 
        /** 
        * Return the approximate per-application memory class of the current 
        * device.  This gives you an idea of how hard a memory limit you should 
        * impose on your application to let the overall system work best.  The 
        * returned value is in megabytes; the baseline Android memory class is 
        * 16 (which happens to be the Java heap limit of those devices); some 
        * device with more memory may return 24 or even higher numbers. */
        	  public int getMemoryClass() {  
        		  return staticGetMemoryClass();  
    		  } 
    	     /** @hide */  
    		  static public int staticGetMemoryClass() {  
              // Really brain dead right now -- just take this from the configured  
              // vm heap size, and assume it is in megabytes and thus ends with "m".  
        	      String vmHeapSize = SystemProperties.get("dalvik.vm.heapsize", "16m");  
                  return Integer.parseInt(vmHeapSize.substring(0, vmHeapSize.length()-1)); 
        }  

  - Bitmap Memory:
     - 3.1之前，在Native Heap分配，但是同样算入Java Object Heap中，因此Bitmap+Java Object<=Xmx
     - 3.1之后，直接放入Java Object Heap，接受Gc管理
  - Native Heap：
     - malloc()分配。不收Gc限制。
     - Android内部代码使用了大量`智能指针`避免内存泄漏

### GC
- 2.3之前：
   - Stop-the-world，垃圾收集线程在执行时，其它线程都停止
   - Full heap collection，一次收集完全部的垃圾
   - 程序中止通常大于100ms
- 2.3之后：
   - Cocurrent，垃圾收集线程与其它线程是并发执行的
   - Partial collection，一次可能只收集一部分垃圾
   - 程序中止通常都小于5ms

Gc日志：

    D/dalvikvm(9050): GC_CONCURRENT freed 2049K, 65% free 3571K/9991K, external 4703K/5261K, paused 2ms+2ms
- GC_CONCURRENT：GC原因
- 2049K：总共回收的内存
- 3571K/9991K：在9991K的Java Object Heap中，3571K正在使用的
- 4703K/5261K：在5261K的External Memory中，4703K正在使用的
- 2ms+2ms：垃圾收集造成的程序中止时间

### JIT
- 运行时编译，可以有效优化代码，但是占用运行时。
- 2-8原则：80%时间重复运行20%代码，因此只JIT这20%代码
- 假设某种情况，并作出代码优化--这种假设成立，保持现状--假设不成立，调整策略--只要假设不成立的情况很少发生或不发生，就会获得巨大收益--Gambing
- 例子：
   - Java的同步，Lock和Unlock操作是非常耗时的，多线程环境下需要这种操作
   - 但有些程序(同步块、同步函数)，很可能始终保持着单线程执行状态
   - JIT采取一种Lazy Unlocking机制：
      - 线程T1执行同步代码块C，先按照正常流程获取轻量级锁L1，线程T1的ID会记录在L1上
      - 当T1离开C时，并不释放L1
      - 当T1再次进入C，发现L1所有者就是自己，直接执行C
      - 此时另一个线程T2需要执行C，发现L1已经被T1占有
      - JIT检查T1调用堆栈，查看T1是否还在执行C
      - 是，将轻量级锁L1转换为重量级锁L2，将L2状态设置为锁定，再让T2在L2上睡眠
      - T1执行完C后，按正常流程释放L2，从而唤醒T2，执行C
      - 否，直接将L1所有者标记为T2，T2执行C
- 静态语言无法这么做。"从这个角度来看，我们就可以说，静态编译语言（如C++）并不一定比在虚拟机上执行的语言（如Java）快，这是因为后者可以有一种强大的武器叫做JIT"——罗升阳
- [JIT实现原理简要介绍](http://blog.reverberate.org/2012/12/hello-jit-world-joy-of-simple-jits.html)

### NDK
### 进程和线程管理
老罗看过Linux内核的几本书[老罗的Android学习之路](http://blog.csdn.net/luoshengyang/article/details/6557518)
[怎么实现一个虚拟的CPU](https://dl.packetstormsecurity.net/papers/general/Abstract-Processor.pdf)

## Android程序的安装流程

1. 系统程序：开机时安装，没有安装界面。开机时启动PackageManagerService服务，扫描/system/app重新安装所有程序
  - Zygote进程--SystemServer组件--PackageManagerService：
   ![](https://raw.githubusercontent.com/Tristan-Hou/MarkdownImg/master/res/0_1315661784a77A.gif)
    图片来源：[Android应用程序安装过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6766010)
2. Android市场安装：网络安装，没有安装界面。
3. Adb：没有安装界面
4. SD卡：apk文件安装，有安装界面。调用Android系统软件包packageinstall.apk安装。
  - 当点击apk，进入安装页面时，实际上就是启动了packageinstall.apk的PackageInstallerActivity，通过初始化PagcageManager和PackageParser.Packager对象，通过PackageUtil.getPackageInfo()解析程序包信息，主要解析Menifest.xml中的标签信息。失败，返回；成功，setContentView显示安装界面。
  - 点击安装按钮--startActivity--InstallAppProgress.class--PackageManager.installPackage()。这个方法最终通过PackageManagerService.java实现。总之，通过installPackage()，进行了程序安装权限验证，随后进行安装或替换。
  - 安装过程中会通过scanPackageLI()完成apk依赖库检测、签名验证、sharedUser的签名检查、更新Native库目录文件、组件名称检查等，这些都完成后，通过mInstaller.install()安装程序
  - install()构造字符串“install name uid gid"--transaction()--通过socket发送install指令--/system/bin/installd(常驻内存)--install指令函数installd.c do_install() install()--创建包路径/创建库路径/创建包目录/设置包目录权限/创建库目录/设置库目录权限/设置库目录所有者/设置包目录所有者--socket回传结果--成功/失败
  
## dex文件格式与class文件格式


	
	
	
	
           