---
title: JVM 概览
date: 2017-12-07 21:38:50
tags: java 
cover: https://blog-1256162814.cos.ap-nanjing.myqcloud.com/jvm/1.webp
top: 6
---
## Java
说到 Java，大家第一时间想到的类似于下面的程序语句：

```java
package com.zqlite.jvm;

public class HelloJVM {

    private static final int k = 100;
    public static void main(String ...args){
        int a = 0,b=3;
        System.out.println(a+b+k);
    }
}
```
<!-- more -->
但这仅仅只属于 Java 技术体系中的 Java 程序设计语言。Java 的技术体系从传统意义上来看有以下几个：

1. Java 程序设计语言
2. 各硬件平台上的 Java 虚拟机
3. Class 文件格式
4. Java API 类库
5. 其他商业机构或开源社区的第三方 Java 类库

其中 1、4、5 大家应该比较熟悉，在编程中都能直接接触。2、3 对于大家来说可能陌生了些，但不夸张的讲，Java 能有如今的活力，其功臣正是 2 和 3。所以接下来的内容将围绕这两点展开，让你从更深的层次了解你所熟悉的 Java。

## JVM

JVM（Java Virtual Machine）就是上面所说的 Java 虚拟机，其作用是加载与运行 Class 文件。有句话大家一定不陌生，“一次编写，到处运行”。简单解释一下，因为在 Class 文件和硬件平台中间隔着一个 JVM，由 JVM 负责加载和执行 Class 文件，这样平台的差异性就留给了 JVM 去考虑，而不是 Class 文件。也正因如此，Class 文件的格式可以真正做到平台无关性。

### JVM 中的 Java 内存区域划分

对于从事 C、C++ 的开发人员来说，在内存管理领域他们是拥有最高权力的“皇帝”，但又是从事最底层基础工作的“劳动人民”。他们即拥有与内存直接打交道的权利，又要负责维护每一个对象的生命周期。

对于 Java 开发人员来说就轻松多了，在 JVM 自动内存管理机制的帮助下，不再需要维护每一个对象的生命周期，内存控制的权利由开发人员转移到了 JVM 。

因为 JVM 需要管理 Java 运行期间的各种对象的生命周期，所以它在执行 Java 程序的时候会把它所管辖的内存分成若干个不同的数据区域，具体如下图：

![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/jvm/1.webp)

对以上几个区域做下简单的介绍：

- 方法区：线程共享区域，用于存储被虚拟机加载的类信息、常量、静态变量等数据。
- 堆：线程共享区域，存放类实例以及数组。我们平时听到的 GC（垃圾回收）就是在堆上进行的。 
- 虚拟机栈：线程私有区域，用于描述 Java 方法的执行模型。每个方法在执行同时会创建一个栈帧用于存储局部变量表、操作数、动态链接、方法出口等信息。
- 本地方法栈：线程私有区域，与虚拟机栈作用类似，区别在于本地方法栈执行的是 Native 方法。
- 程序计数器：线程私有区域，用于指向当前线程下一条需要执行的字节码指令。

除了以上几个主要区域外，另外还要介绍一个区域：运行时常量池。它是方法区的一部分，用于存放编译期生成的字面量和符号引用。字面量好理解，比如：

```java
String str = "a";
```

上面的 a 就是字面量。而符号引用则是一些字符串，用于给虚拟机定位类或类方法等。

> 面试的时候有时会遇到这样的问题，Java 中每次声明并初始化一个 String 对象都会在堆上面分配内存吗？这里的答案当然是否，因为有可能新声明的对象已经在常量池中存在了，这时候 JVM 会将新对象的引用指向常量池中对应的值而不是在堆重新分配。

### 对象在 JVM 中的创建过程

在 Java 中创建对象很简单，一个简单的 new 关键字就可以。但在虚拟机中，对象的创建过程是怎样的呢？我们来一起了解一下。

在 JVM 中，创建对象的字节码指令是 new，当 JVM 执行引擎遇到 new 指令后会进行如下五步操作：

一、检查这个指令的参数能否在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已经加载、解析和初始化。

下面我们看一下：

```java
package com.zqlite.jvm;

public class HelloJVM {

    public static void main(String ...args){
        Object obj = new Object();
    }
}
```
然后用 javap 对这段代码生成的 class 文件进行反汇编处理，结果如下：

```class
Classfile /Users/scott/workspace/jvm/out/production/jvm/com/zqlite/jvm/HelloJVM.class
  Last modified 2017-11-29; size 446 bytes
  MD5 checksum bcc0b2d9129748ef0caa00e95cf95a47
  Compiled from "HelloJVM.java"
public class com.zqlite.jvm.HelloJVM
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #2.#19         // java/lang/Object."<init>":()V
   #2 = Class              #20            // java/lang/Object
   #3 = Class              #21            // com/zqlite/jvm/HelloJVM
   #4 = Utf8               <init>
   #5 = Utf8               ()V
   #6 = Utf8               Code
   #7 = Utf8               LineNumberTable
   #8 = Utf8               LocalVariableTable
   #9 = Utf8               this
  #10 = Utf8               Lcom/zqlite/jvm/HelloJVM;
  #11 = Utf8               main
  #12 = Utf8               ([Ljava/lang/String;)V
  #13 = Utf8               args
  #14 = Utf8               [Ljava/lang/String;
  #15 = Utf8               obj
  #16 = Utf8               Ljava/lang/Object;
  #17 = Utf8               SourceFile
  #18 = Utf8               HelloJVM.java
  #19 = NameAndType        #4:#5          // "<init>":()V
  #20 = Utf8               java/lang/Object
  #21 = Utf8               com/zqlite/jvm/HelloJVM
{
  public com.zqlite.jvm.HelloJVM();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 6: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/zqlite/jvm/HelloJVM;

  public static void main(java.lang.String...);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC, ACC_VARARGS
    Code:
      stack=2, locals=2, args_size=1
         0: new           #2                  // class java/lang/Object
         3: dup
         4: invokespecial #1                  // Method java/lang/Object."<init>":()V
         7: astore_1
         8: return
      LineNumberTable:
        line 9: 0
        line 10: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  args   [Ljava/lang/String;
            8       1     1   obj   Ljava/lang/Object;
}
SourceFile: "HelloJVM.java"

```
暂时无需理解上面的所有内容，我们只关注 new 字节码部分。

找到

```class
0: new           #2
```

可以看到 new 指令码的参数是 #2，接着通过参数 #2 在常量池（Constant pool）中找到 #2 对应的内容：

```class
#2 = Class              #20            // java/lang/Object
```

这里说明了 #2 对应的是一个类，类符号引用需要到 #20 中找，在看 #20 ：

```class
#20 = Utf8               java/lang/Object
```

这里的 #20 是一个字符串类型，其内容是 ava/lang/Object，表示 Object 类的符号引用即全限定名。接着 JVM 就会去寻找并加载、解析和初始化这个类。

二、当在第一步确定了目标类以后，JVM 就会为这个类的新生对象在堆上面分配内存。对象所需要的内存大小在第一步的类加载后已经确定了，所以为对象分配内存也就等同于把一块确定大小的内存从 Java 堆中划分出来。
 
三、分配完内存后，JVM 需要将分配到的内存空间都初始化为零值（不包括对象头）。
 
 > 这里需要注意，由于 JVM 会对对象的内存空间做初始化操作，所以在类中类似的定义一个 int i ，其默认值就是整型的零值 0 。Java 类变量也正因如此允许不赋初始值。但在类方法中，如果这样定义一个 i，在接下来的代码中如果使用 i 的话，会有报错提示 i 没有初始化值，
 
四、当初始化完以后，JVM 就要对对象做必要的处理，比如这个对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码、对象的 GC 分代年龄表的信息。这些信息都会放在对象头中。
 
五、上面的工作都完成之后，从 JVM 的角度来看，一个新的对象已经产生了，但从 Java 程序的角度来看，对象创建才刚开始。<init> 方法还没有执行，所有的字段还都为零。所以一般在 new 指令码后都会跟 invokespecial 指令来调用 <init> 方法。此处的 <init> 即我们通常说的构造方法。
 
### 对象的内存布局
 
在上小节介绍了对象在 JVM 中的创建过程，这节简单介绍下对象的内存布局。
 
对象的内存布局可以分三个区域：对象头、实例数据和对齐填充。
 
其中对象头包括两部分信息，第一部分用于存储对象自身的运行时数据，如哈希码、GC 分代年龄、锁状态标志、线程持有的锁等。另外一部分则是类型指针，用于指向它的类元数据。JVM 可以通过这个指针来确定对象是属于哪个类的。另外需要注意的是如果对象是数组，那么对象头中还需要记录数组的长度。
 
实例数据部分是对象真正存储的有效信息，也是在程序代码中所定义的各种类型字段的内容，包括从父类继承下来的和自己定义的。
 
最后一部分并不是必须的，它仅仅起到了占位符的作用。有些虚拟机要求对象的起始地址必须是 8 字节的整数倍，换句话说就是对象大小必须是 8 字节的整数倍。所以当不足整数倍的时候，就需要进行对齐填充处理。
 
下面这张图简单表示了对象在内存中的布局：
 
 ![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/jvm/2.webp)
 

## Class 文件

计算机只认识 0 和 1 ，这是常识。所以我们编写的程序都需要经过编译器翻译成有 0 和 1 构成的二进制格式才能被计算机执行。但在近十几年内，虚拟机以及大量建立在虚拟机上的程序语言如雨后春笋般出现并蓬勃发展，由此二进制本地机器码已不再是唯一的选择，越来越多的程序语言选择了与操作系统和机器指令集无关的、平台中立的格式作为程序编译后的存储格式。我们这里的 Class 文件就是其中一种。

我们通常都把 Java 和 JVM 有意识无意识的联系在一起，但 JVM 其实并不关心 Java。什么意思呢？JVM 面向的是 Class 格式的文件，它并不关系 Class 的来源是何种语言。如 Clojure、Groovy、Jruby、Jython、Scala 等都可以被编译成 Class 文件，从而在 JVM 上运行。

### Class 内部结构

Class 文件是一组以 8 位字节为基础单位的二进制流，各个数据项目严格按照顺序紧凑地排列在 Class 文件中，其中没有任何分隔符，这使得整个 Class 文件中存储的内容几乎全部都是程序运行必须的数据，没有空隙存在。

Class 文件的完整格式见下表：

![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/jvm/3.webp)

其中类型中的 u4，u2以及没有这上图出现的 u1，u8都表示无符号数。其中 u1 表示 1 个字节，u2 表示 2 个字节，以此类推。名称则说明该位置数据的作用，如 constant\_pool 表示常量池。数量则说明 Class 文件中，不同名称的数据块的个数。如 constant\_pool 的个数取决于 constant\_pool\_count，而 constant\_pool\_count 则是占用两个字节的无符号数。

不管是多复杂的 Class 文件，其内部数据结构必然是按照上表来排列的，由于其内部各类型的数据比较复杂，这里也不展开讲。这一节大家只要知道 Class 文件内部有这些数据就可以了，如果需要查看 Class 文件，可以使用反编译工具 javap 。具体命令如下：

```sh
javap -v ClassFile
```

### 字节码指令介绍

JVM 中的指令由一个字节长度的、代表着某种特定操作含义的数字（操作码，字节码）以及跟随其后的零至多个代表此操作所需参数（操作数）构成。由于 JVM 采用面向操作数栈而不是寄存器的结构，所以大多数的指令都不包含操作数，只有一个操作码。由于指令长度是 8 位的无符号数，所以 JVM 指令最多 256 条。


大部分指令从其助记符就能知道它所操作的数据类型，比如 iload 指令用于从局部变量表加载 int 型的数据到操作数栈，而 fload 指令加载的则是 float 类型的数据。

大部分与数据相关的字节码指令，其助记符都有特殊的字符来表示其服务的数据类型：i 代表对 int 型数据的操作，l 代表 long，s 代表 short，b 代表 byte，c 代表 char，f 代表 float，d 代表 double，a 代表 reference。

如需查看所有 JVM 指令，点击此链接：[http://17b84ff5.wiz03.com/share/s/0nK4_R1FrArb2bpd-P3HNDCf0YLy5c0dYAqj2-_XjE3Jsh_A](http://17b84ff5.wiz03.com/share/s/0nK4_R1FrArb2bpd-P3HNDCf0YLy5c0dYAqj2-_XjE3Jsh_A)


## JVM 与 Class 文件

前面分开讲了 JVM 与 Class 文件，相信大家对它们有了一定的了解。Java 程序之所以可以运行起来，离不开这两位的紧密配合，下面我要给大家介绍的就是关于它俩的一些相关知识。

### 类加载

Class 文件从被 JVM 加载到内存，到卸载出内存为止，它的整个生命周期包括：加载、验证、准备、解析、初始化、使用和卸载。其中验证、准备、解析三个部分统称为连接，这七个阶段发生顺序如下图：

![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/jvm/4.webp)


接下来我们简单了解一下这几个过程

- 加载：在类的加载阶段 JVM 主要做了下面三件事：

1. 通过类的全限定名来获取定义此类的二进制字节流。
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
3. 在内存中生成一个代表这个类的 java.lang.Class 对象，作为方法区这个类的各种数据的访问入口。

这里需要注意第一点，它只说明了通过全限定名获取定义此类的二进制字节流，但是没有具体说从哪获取，怎么获取。正是有了这么一个开放的入口，才有了现在 Java 的各种有趣的玩法。

- 验证：这一阶段的目的是为了确保 Class 文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。

- 准备：为类变量（static 变量）分配内存并设置变量的初始值。

- 解析：JVM 将常量池内的符号引用替换为直接引用。

- 初始化：前面几个阶段都是 JVM 主导和控制的，到了这一步开始，才真正开始执行 Java 程序代码。在准备阶段 JVM 为类变量设置了初始值，而在初始化阶段，JVM 会调用类构造器 <clinit>() 方法，即我们平时写的 static{ ... } 这部分代码和静态变量赋值操作的集合。

- 使用：这部分是我们开发者最熟悉的，即我们所写的程序运行的过程。

- 卸载：当 JVM 确定某个类永久不需要的时候，就会执行类卸载，将其在内存中占用的空间全部释放。

### 类加载器

前面介绍 JVM 加载类的时候说到，JVM 对于从哪加载二进制字节流是对外开放的，即这部分是在 JVM 外部实现的。而用于实现类加载的代码模块称为“类加载器”。类加载器可以说是 Java 语言的一项创新，也是 Java 语言流行的重要原因之一，它起初是为 Java Applet 而开发出开的。虽然目前 Java Applet 技术基本已经“死了”，但类加载器却在类层次划分、OSGi、热部署、代码加密等领域大放异彩，成为了 Java 体系中一块重要的基石。

下面我用代码展示如何自定义一个类加载器，从网络上加载一个类，然后调用其 toString 方法。其实现很简单，如下：

```java
package com.zqlite.jvm;

import java.io.IOException;
import java.io.InputStream;
import java.net.HttpURLConnection;
import java.net.URL;

public class HelloJVM {

    public static void main(String ...args){
        
        MyNetClassLoader classLoader = new MyNetClassLoader();
        try {
            Class<?> clazz = classLoader.findClass("http://7xprgn.com1.z0.glb.clouddn.com/RemoteClass.class");
            Object o = clazz.newInstance();
            System.out.print(o.toString());
        } catch (ClassNotFoundException |IllegalAccessException | InstantiationException e) {
            e.printStackTrace();
        }
    }

    private static class MyNetClassLoader extends ClassLoader{
        @Override
        protected Class<?> findClass(String name) throws ClassNotFoundException {
            try {
                URL url = new URL(name);
                HttpURLConnection connection = (HttpURLConnection) url.openConnection();
                InputStream inputStream =  connection.getInputStream();
                byte[] b = new byte[inputStream.available()];
                inputStream.read(b);
                return defineClass("com.zqlite.jvm.RemoteClass",b,0,b.length);

            } catch (IOException e) {
                e.printStackTrace();
            }
            return super.findClass(name);
        }
    }
}
```

首先我们自定义一个 MyNetClassLoader 类来继承 ClassLoader ，然后重写它的 findClass 方法，此方法会在系统找不到指定类的时候调用。在 findClass 中我们做的事情很简单，从网络上获取类的二进制字节流，然后读入数组，最后通过 defineClass 方法，将其转为 class 对象并返回。

然后通过 class 对象的 newInstance 方法获得一个对应的实例对象，即 RemoteClass 的对象，然后输入其 toString 返回的内容。具体输出什么大家可以在本地跑了试试。


### 基于栈的解释器执行过程

在 JVM 的内存区域划分的时候讲到过虚拟机栈这个区域，它是线程私有区域，用于描述 Java 方法的执行模型。每个方法在执行同时会创建一个栈帧用于存储局部变量表、操作数、动态链接、方法出口等信息。接下来我就用一个实际的例子来介绍在 JVM 中，一个方法是如何被解释执行的。借此机会也可以更进一步了解虚拟机栈相关知识。

#### 运行时栈帧结构

栈帧是用于支持 JVM 进行方法调用和方法执行的数据结构，它是虚拟机栈的栈元素。每一个方法从调用到完成，都对应着一个栈帧在虚拟机栈中的入栈和出栈操作。

每个栈帧都包括局部变量表、操作数、动态连接、方法返回地址和一些额外的附加信信息。并且在程序进行编译的时候，栈帧需要多大的局部变量表，多深的操作数栈都是已经确定的，因此一个栈帧需要分配多少内存也是确定不变的。JVM 中线程、虚拟机栈、栈帧的典型结构图如下：

![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/jvm/5.webp)

#### 解释器执行过程实例

本节中，我准备了如下一段代码：

```java
package com.zqlite.jvm;

public class HelloJVM {

    public static void main(String ...args){
        calc();
    }

    public static int calc(){
        int a = 100;
        int b = 200;
        int c = 300;
        return (a + b) * c;
    }
}
```

从 Java 的语言角度来讲，这段代码没有任何解释的必要，我们接下来用 javap 来看看他的字节码指令，如下：

```class
Classfile /Users/scott/workspace/jvm/out/production/jvm/com/zqlite/jvm/HelloJVM.class
  Last modified 2017-11-30; size 549 bytes
  MD5 checksum 1bf313c415146d6070a30e8addc83a8c
  Compiled from "HelloJVM.java"
public class com.zqlite.jvm.HelloJVM
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #4.#24         // java/lang/Object."<init>":()V
   #2 = Methodref          #3.#25         // com/zqlite/jvm/HelloJVM.calc:()I
   #3 = Class              #26            // com/zqlite/jvm/HelloJVM
   #4 = Class              #27            // java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = Utf8               Code
   #8 = Utf8               LineNumberTable
   #9 = Utf8               LocalVariableTable
  #10 = Utf8               this
  #11 = Utf8               Lcom/zqlite/jvm/HelloJVM;
  #12 = Utf8               main
  #13 = Utf8               ([Ljava/lang/String;)V
  #14 = Utf8               args
  #15 = Utf8               [Ljava/lang/String;
  #16 = Utf8               calc
  #17 = Utf8               ()I
  #18 = Utf8               a
  #19 = Utf8               I
  #20 = Utf8               b
  #21 = Utf8               c
  #22 = Utf8               SourceFile
  #23 = Utf8               HelloJVM.java
  #24 = NameAndType        #5:#6          // "<init>":()V
  #25 = NameAndType        #16:#17        // calc:()I
  #26 = Utf8               com/zqlite/jvm/HelloJVM
  #27 = Utf8               java/lang/Object
{
  public com.zqlite.jvm.HelloJVM();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/zqlite/jvm/HelloJVM;

  public static void main(java.lang.String...);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC, ACC_VARARGS
    Code:
      stack=1, locals=1, args_size=1
         0: invokestatic  #2                  // Method calc:()I
         3: pop
         4: return
      LineNumberTable:
        line 6: 0
        line 7: 4
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  args   [Ljava/lang/String;

  public static int calc();
    descriptor: ()I
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=0
         0: bipush        100
         2: istore_0
         3: sipush        200
         6: istore_1
         7: sipush        300
        10: istore_2
        11: iload_0
        12: iload_1
        13: iadd
        14: iload_2
        15: imul
        16: ireturn
      LineNumberTable:
        line 10: 0
        line 11: 3
        line 12: 7
        line 13: 11
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            3      14     0     a   I
            7      10     1     b   I
           11       6     2     c   I
}
SourceFile: "HelloJVM.java"

```

我们把注意点放在 calc 方法上，其中

```class
descriptor: ()I
```
表示这个方法的方法描述符是 ()I。了解 JNI 的小伙伴对方法描述符应该不会陌生，这里我不多介绍，想了解更多可以参考[此处](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/types.html)。

接着的

```class
flags: ACC_PUBLIC, ACC_STATIC
```

表示这个是一个 public static 修饰的方法。

然后 Code 部分是重点，其中 stack = 2 说明操作数栈的深度为 2 ，locals = 3 说明本地变量表有三个变量，args_size = 0 说明此方法没有参数。

接下来先跳过下面的字节码，来看到 LocalVariableTable，这就是所谓的本地变量表了，表中有三行数据，分别是 a，b，c 和代码中定义的变量名一致。且他们的签名都是 I ，表示是 Int 型整数。

接下来，我通过下面 7 幅图来讲解下解释执行这个方法的过程和此过程中局部变量表和操作数栈的变化情况。

![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/jvm/6.webp)

上图是执行偏移地址为 0 的指令的情况。


![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/jvm/7.webp)

上图是执行偏移地址为 2 的指令的情况。

![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/jvm/8.webp)

上图是执行偏移地址为 11 的指令的情况。

![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/jvm/9.webp)

上图是执行偏移地址为 12 的指令的情况。

![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/jvm/10.webp)

上图是执行偏移地址为 13 的指令的情况。

![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/jvm/11.webp)

上图是执行偏移地址为 14 的指令的情况。

![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/jvm/12.webp)

上图是执行偏移地址为 16 的指令的情况。

方法的执行模型到这就结束了，请大家记住这仅仅是一种概念模型，虚拟机最终会对执行过程做一些优化来提升性能，实际的执行过程与概念模型差距可能会很大。之所以会有这种差距是因为虚拟机的执行引擎和即时编译器都会对输入的字节码进行优化。

## 总结

这篇文章我主要带着大家简单了解了一下 JVM 的内层划分、对象的创建、字节码结构、字节码指令和类加载与解释执行这几部分的相关内容。由于篇幅有限，如 GC，编译优化和 JVM内部实现并发等内容没有介绍，如果大家感兴趣可以阅读《深入理解 Java 虚拟机-JVM 高级特性与最佳实践》。本文也是参考此书总结而来。