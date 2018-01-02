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

## Dalvik虚拟机与Java虚拟机的区别

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
	
	
	
	
           