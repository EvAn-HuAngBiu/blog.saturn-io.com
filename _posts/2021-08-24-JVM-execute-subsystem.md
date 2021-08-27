---
layout: post
title: JVM虚拟机执行子系统
categories: JVM
description: Java类文件格式、类加载机制、运行时栈帧结构以及方法分派的分析
keywords: JVM
---

## 虚拟机执行子系统

### 类文件结构

Java实现平台无关性的基石就是Class文件，JVM虚拟机并不与包括Java在内的任何语言绑定，而是只与Class文件这种特殊的二进制文件绑定。如果某种语言能够编译为符合JVM规范的Class文件，那就可以被JVM虚拟机加载、解释和执行。

**Class文件是一组以8个字节为基础单位的二进制流**，当遇到需要占用8个字节以上空间的数据项时，会采用高位在前（BE）的方式分割为若干个8字节来存储。Class文件采用了一种伪结构体的方式存储，这种伪结构体中**只有两种数据类型：无符号数和表**。其中无符号数属于基本数据类型，以u1、u2、u4、u8来分别代表1个字节、2个字节、4个字节、8个字节的无符号数，**无符号数可以用来描述数字、索引引用、数量值或者按照UTF-8编码构成的字符串值**。表是由多个无符号数或者其他表作为数据结构项构成的复合数据类型。

#### Class文件格式

Class文件格式顺序如下表：

|      类型      |        名称         |          数量           |
| :------------: | :-----------------: | :---------------------: |
|       u4       |        magic        |            1            |
|       u2       |    minor_version    |            1            |
|       u2       |    major_version    |            1            |
|       u2       | constant_pool_count |            1            |
|    cp_info     |    constant_pool    | constant_pool_count - 1 |
|       u2       |     access_flag     |            1            |
|       u2       |     this_class      |            1            |
|       u2       |     super_class     |            1            |
|       u2       |  interfaces_count   |            1            |
|       u2       |     interfaces      |    interfaces_count     |
|       u2       |    fields_count     |            1            |
|   field_info   |       fields        |      fields_count       |
|       u2       |    methods_count    |            1            |
|  method_Info   |       methods       |      methods_count      |
|       u2       |  attributes_count   |            1            |
| attribute_info |     attributes      |    attributes_count     |

##### 魔数与Class文件的版本

每个Class文件的前4个字节为固定的魔数0xCAFEBABE。紧接着魔数的4个字节存储的是Class文件的版本号：第5和第6个字节是次版本号Minor Version，第7和第8个字节是主版本Major Version。

我们来看下面这个类：

```java
public class Main {}
```

在JDK 11.0.10版本编译后的Class文件前16字节如下：

```
00000000: cafe babe 0000 0037 000d 0a00 0300 0a07  .......7........
```

可以看到第5和第6个字节均为0，表示次版本号为0，而第7和第8个字节为十六进制数0037（对应十进制数55）代表了JDK11（后续版本不断加1，JDK1.4以后的版本不断减1）。

目前在JDK1.2之后，次版本号并未被使用，固定为0；JDK12之后，如果当前Class文件使用了尚未正式支持的预览功能，那么必须把次版本号标识为FF（65535），以便虚拟机在加载类文件时能区分出来。

##### 常量池

紧接着次版本号、主版本号之后的是常量池，常量池的大小是不固定的，它由一个紧接在版本号之后的u2类型的数据指定，**常量池的最大大小为65535，从1开始计数（0表示不引用任何常量）。**常量池中主要存放两大变量：字面量和符号引用，字面量包括文本字符串、被声明为final的常量等；符号引用包括被模块导出或开放的包、类和接口的全限定名、字段的名称和描述符、方法的名称和描述符、方法句柄和类型、动态调用点和动态常量。**Class文件中并不会保存上述符号引用真正的内存布局信息，这些内存布局信息是在虚拟机加载类时从常量池读取，并在类创建或运行时解析、翻译到具体的内存地址中去的。**

我们首先来看下面这段代码已经反编译后的常量池：

```java
import java.util.ArrayList;

public class Main {
    private int i = 0;

    public static void main(String[] args) {
        ArrayList<Integer> array = new ArrayList<>();
        System.out.println("Hello");
    }
}

```

上面是很简单的一段示例代码，我们利用`javap -c -l -constants -v -p Main`来反编译，最后看输出的反编译代码：

```
public class Main
  minor version: 0
  major version: 55
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #8                          // Main
  super_class: #9                         // java/lang/Object
  interfaces: 0, fields: 1, methods: 2, attributes: 1
Constant pool:
   #1 = Methodref          #9.#20         // java/lang/Object."<init>":()V
   #2 = Fieldref           #8.#21         // Main.i:I
   #3 = Class              #22            // java/util/ArrayList
   #4 = Methodref          #3.#20         // java/util/ArrayList."<init>":()V
   #5 = Fieldref           #23.#24        // java/lang/System.out:Ljava/io/PrintStream;
   #6 = String             #25            // Hello
   #7 = Methodref          #26.#27        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #8 = Class              #28            // Main
   #9 = Class              #29            // java/lang/Object
  #10 = Utf8               i
  #11 = Utf8               I
  #12 = Utf8               <init>
  #13 = Utf8               ()V
  #14 = Utf8               Code
  #15 = Utf8               LineNumberTable
  #16 = Utf8               main
  #17 = Utf8               ([Ljava/lang/String;)V
  #18 = Utf8               SourceFile
  #19 = Utf8               Main.java
  #20 = NameAndType        #12:#13        // "<init>":()V
  #21 = NameAndType        #10:#11        // i:I
  #22 = Utf8               java/util/ArrayList
  #23 = Class              #30            // java/lang/System
  #24 = NameAndType        #31:#32        // out:Ljava/io/PrintStream;
  #25 = Utf8               Hello
  #26 = Class              #33            // java/io/PrintStream
  #27 = NameAndType        #34:#35        // println:(Ljava/lang/String;)V
  #28 = Utf8               Main
  #29 = Utf8               java/lang/Object
  #30 = Utf8               java/lang/System
  #31 = Utf8               out
  #32 = Utf8               Ljava/io/PrintStream;
  #33 = Utf8               java/io/PrintStream
  #34 = Utf8               println
  #35 = Utf8               (Ljava/lang/String;)V
```

可以看到常量池中有35个变量，其中前面#1、#2代表了常量的序号，通过序号可以找到对应的常量，后面就是常量对应的类型，类型接下来会分析，最后右侧的是常量所指向的字面量或符号引用。

下面我们来看常量池的项目类型：

|               类型               | 标志 |             描述             |
| :------------------------------: | :--: | :--------------------------: |
|        CONSTANT_Utf8_info        |  1   |      UTF-8编码的字符串       |
|      CONSTANT_Integer_info       |  3   |          整形字面量          |
|       CONSTANT_Float_info        |  4   |         浮点型字面量         |
|        CONSTANT_Long_info        |  5   |         长整型字面量         |
|       CONSTANT_Double_info       |  6   |      双精度浮点型字面量      |
|       CONSTANT_Class_info        |  7   |      类或接口的符号引用      |
|       CONSTANT_String_info       |  8   |       字符串类型字面量       |
|      CONSTANT_Fieldref_info      |  9   |        字段的符号引用        |
|     CONSTANT_Methodref_info      |  10  |      类中方法的符号引用      |
| CONSTANT_InterfaceMethodref_info |  11  |     接口中方法的符号引用     |
|    CONSTANT_NameAndType_info     |  12  |   字段或方法的部分符号引用   |
|    CONSTANT_MethodHandle_info    |  15  |         表示方法句柄         |
|     CONSTANT_MethodType_info     |  16  |         表示方法类型         |
|      CONSTANT_Dynamic_info       |  17  |     表示一个动态计算常量     |
|   CONSTANT_InvokeDynamic_info    |  18  |    表示一个动态方法调用点    |
|       CONSTANT_Module_info       |  19  |         表示一个模块         |
|      CONSTANT_Package_info       |  20  | 表示一个模块中开发或导出的包 |

每个常量都有自己的数据结构，这里我们列出上述所有常量的数据结构：

<img src="/images/JVM/JVM-Class-ConstantPool-01.png" alt="JVM-Class-ConstantPool-01" style="zoom: 67%;" />

<img src="/images/JVM/JVM-Class-ConstantPool-02.png" alt="JVM-Class-ConstantPool-02" style="zoom:67%;" />

<img src="/images/JVM/JVM-Class-ConstantPool-03.png" alt="JVM-Class-ConstantPool-03" style="zoom:67%;" />

这里解释一下UTF8类型的常量的定义，UTF8类型常量定义了长度length单位为字节，最大值为0xFF字节（65535B = 64KB），也就是说如果类、接口的全限定名以及方法、变量的名称如果在压缩UTF-8编码下大于64KB，那么即使满足命名规则也无法编译（压缩UTF-8是指，如果一个字节能存储的下那就使用一个字节存储，如果两个字节能存储的下，那就使用两个字节存储以此类推，普通UTF-8在一个字节能存储的情况下也会使用两个字节来存储）

##### 访问标志

在常量池之后的是访问标志access_flag，其含义如下：

| 标志名称       | 标志值 | 含义                                                    |
| :------------- | :----: | :------------------------------------------------------ |
| ACC_PUBLIC     | 0x0001 | 是否为public类型                                        |
| ACC_FINAL      | 0x0010 | 是否被声明为final，只有类可设置                         |
| ACC_SUPER      | 0x0020 | 是否运行使用invokespeical的新语义，JDK1.0.2之后必须为真 |
| ACC_INTERFACE  | 0x0200 | 标识这是一个接口                                        |
| ACC_ABSTRACT   | 0x0400 | 是否为abstract类型，对于接口或者抽象类来说此标志为真    |
| ACC_SYNTHETIC  | 0x1000 | 标识这个类是由非用户代码生成的                          |
| ACC_ANNOTATION | 0x2000 | 标识这是一个注解                                        |
| ACC_ENUM       | 0x4000 | 标识这是一个枚举                                        |
| ACC_MODULE     | 0x8000 | 表示这是一个模块                                        |

访问标志非常好理解，就是用来表示类或者接口的描述符特征的，一共有16个标识符可用，目前用到了9个，要注意目前ACC_SUPER标识符始终为真。

##### 类索引、父类索引和接口索引集合

这里类索引指的是当前类也就是this_class，由于Java是单继承结构所以只会有一个父类，所以**super_class指向一个CONSTANT_Class_info类型的常量（通过常量编号），通过CONSTANT_Class_info常量就可以找到定义在CONSTANT_Utf8_info常量中的全限定字符串了**。而对于Object类，由于它没有父类，所以这里的super_class指向0号常量（即不存在任何引用关系）。在super_class之后就是接口计数器了，interfaces_count用来指示当前类实现了多少个接口，最大值为65535，紧接着有interfaces_count个u2类型的常量指向常量池中对应的接口的CONSTANT_Class_info常量。

##### 字段表集合

字段表用于描述接口或类中声明的变量，这些字段包括类级字段和实例级变量，但是不包括定义在方法内部的局部变量，字段表的结构如下：

<img src="/images/JVM/JVM-Class-FieldTable-Arch.png" alt="JVM-Class-FieldTable-Arch" style="zoom: 67%;" />

字段的修饰被放在了access_flags项目中，它与类中的access_flag非常一致，都是一个u2的数据类型：

<img src="/images/JVM/JVM-Class-Field-AccessFlag.png" alt="JVM-Class-FieldTable-Arch" style="zoom:67%;" />

在access_flags之后就是指向字段（方法）简单名称和字段（方法）的描述符。例如ArrayList中的`void set(int index, E element)`方法，它的简单名称就是set，描述符就是(IL)V，全限定名称就是java/util/ArrayList; 

在这里全限定名称就是将包名中的 . 换成了 /，并且在末尾加上分号作为结束。字段描述符的含义如下表：

<img src="/images/JVM/JVM-Class-Field-Descriptor.png" alt="JVM-Class-Field-Descriptor" style="zoom:67%;" />

对于数组类型会在前面加上"["，例如String[]则是 "L[java/lang/String;"，对于多维数组则在前面加上多个"["，例如String\[\]\[\]则是"L[[java/lang/String;"。

我们来看下面这个类定义：

```java
public class Main {
    private Integer i;
    private static String[] strs;
}
```

反编译后代码如下：

```
public class Main
  minor version: 0
  major version: 55
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #2                          // Main
  super_class: #3                         // java/lang/Object
  interfaces: 0, fields: 2, methods: 1, attributes: 1
Constant pool:
   #1 = Methodref          #3.#14         // java/lang/Object."<init>":()V
   #2 = Class              #15            // Main
   #3 = Class              #16            // java/lang/Object
   #4 = Utf8               i
   #5 = Utf8               Ljava/lang/Integer;
   #6 = Utf8               strs
   #7 = Utf8               [Ljava/lang/String;
   #8 = Utf8               <init>
   #9 = Utf8               ()V
  #10 = Utf8               Code
  #11 = Utf8               LineNumberTable
  #12 = Utf8               SourceFile
  #13 = Utf8               Main.java
  #14 = NameAndType        #8:#9          // "<init>":()V
  #15 = Utf8               Main
  #16 = Utf8               java/lang/Object
{
  private java.lang.Integer i;
    descriptor: Ljava/lang/Integer;
    flags: (0x0002) ACC_PRIVATE

  private static java.lang.String[] strs;
    descriptor: [Ljava/lang/String;
    flags: (0x000a) ACC_PRIVATE, ACC_STATIC

  public Main();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0
}
SourceFile: "Main.java"
```

可以看到Integer类型的变量i，描述符为Ljava/lang/Integer，访问标志为ACC_PRIVATE；String数组strs描述符为[Ljava/lang/String;，访问标志为ACC_PRIVATE和ACC_STATIC。

我们再来看对应的16进制字节码：

![JVM-Class-Field-XXDAnly1](/images/JVM/JVM-Class-Field-XXDAnly1.png)

我们看蓝色方框内的就是第一个字段的定义，这个字段前的0002表示有2个字段，蓝色方框内的0002表示access_flag，这里表示ACC_PRIVATE（0x0002），后续的0004表示简单名称为常量池中第4个常量，我们找上面反编译结果查看4号常量：

`#4 = Utf8   i`即当前字段的简易名称为i，同理0005表示描述符为：`#5 = Utf8   Ljava/lang/Integer;`

绿色方框内的常量也同理，access_flag为0x000a表示ACC_PREVATE | ACC_STATIC（0x0002|0x0008=0x000a），其它的分析和上面一样，这里略过。

这里先不解释属性表的内容，等后面章节再进行解释。这里要注意，字段表中不会列出从父类或父接口中继承而来的字段，但有可能出现Java代码中原本不存在的字段（例如内部类中会添加访问外部类实例的字段）。

##### 方法表集合

方法表的结构和字段表完全一样：

<img src="/images/JVM/JVM-Class-FieldTable-Arch.png" alt="JVM-Class-FieldTable-Arch" style="zoom:67%;" />

唯一稍有区别的就是access_flags属性，由于volatile和transient不能修饰方法，而synchronized、native、strictfp和abstract关键字可以修饰方法，所以相应增减了一些标志：

<img src="/images/JVM/JVM-Class-Method-AccFlag.png" alt="JVM-Class-Method-AccFlag" style="zoom:67%;" />

而在方法表中只保存了对方法名称、标识符等信息，实际的代码保存在属性表（后面会说到）的Code属性中了。比如下面这个类：

```java
public class Main {
    public void sayHello() {
        System.out.println("Hello world");
    }
}
```

反编译后的常量池如下：

```
   #1 = Methodref          #6.#14         // java/lang/Object."<init>":()V
   #2 = Fieldref           #15.#16        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #17            // Hello world
   #4 = Methodref          #18.#19        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #20            // Main
   #6 = Class              #21            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               sayHello
  #12 = Utf8               SourceFile
  #13 = Utf8               Main.java
  #14 = NameAndType        #7:#8          // "<init>":()V
  #15 = Class              #22            // java/lang/System
  #16 = NameAndType        #23:#24        // out:Ljava/io/PrintStream;
  #17 = Utf8               Hello world
  #18 = Class              #25            // java/io/PrintStream
  #19 = NameAndType        #26:#27        // println:(Ljava/lang/String;)V
  #20 = Utf8               Main
  #21 = Utf8               java/lang/Object
  #22 = Utf8               java/lang/System
  #23 = Utf8               out
  #24 = Utf8               Ljava/io/PrintStream;
  #25 = Utf8               java/io/PrintStream
  #26 = Utf8               println
  #27 = Utf8               (Ljava/lang/String;)V
```

Class文件的十六进制码如下：

<img src="/images/JVM/JVM-Class-Method-XXDAnly1.png" alt="JVM-Class-Method-XXDAnly1" style="zoom:67%;" />

这里还是比较复杂的，因为后面有属性表的缘故。上面的类中定义了1个方法和隐式生成的构造方法共2个方法。第一个方法标识符为0x0001即PUBLIC，方法名定义在0x0007号常量中，查看常量池可知是`#7 = Utf8  <init>`即\<init\>方法，描述符为0x0008号常量即" ()V "，属性表共有1个属性，定义在后面，属性表之后再进行分析；第二个方法同理，标识符为0x0001即PUBLIC，名称为0x000b即11号常量sayHello，描述符为0x0008即" ()V "。

以上就是方法表的内容了，但是要注意对于继承自父类的方法，如果没有被重写，那么它不会出现在方法表中，但是可能会出现编译器添加的方法，比如构造函数\<init\>()或者静态构造方法\<clinit\>()。同时类文件中允许在重载方法时只有返回值不同，因为返回值和参数值都是由描述符提供的，只要描述符不同就认为方法没有重复（但是在Java层面不行，Java要求重载方法必须拥有不同的参数列表）。

##### 属性表集合

属性表的内容很多也很复杂，这里首先列出预定义的属性：

<img src="/images/JVM/JVM-Class-AttributeTable-Arch-1.png" alt="JVM-Class-AttributeTable-Arch-1" style="zoom: 67%;" />

<img src="/images/JVM/JVM-Class-AttributeTable-Arch-2.png" alt="JVM-Class-AttributeTable-Arch-2" style="zoom:67%;" />

属性表的结构如下：

<img src="/images/JVM/JVM-Class-AttributeTable-Arch.png" alt="JVM-Class-AttributeTable-Arch" style="zoom:67%;" />

这里就只简略分析比较重要的一些属性：

1. Code属性

Code属性是使用最频繁的一个属性，它的作用是存储每个方法中的实际代码（接口或抽象类不一定存在）。Code属性的结构如下：

<img src="/images/JVM/JVM-Class-AttributeTable-Code-Arch.png" alt="JVM-Class-AttributeTable-Code-Arch" style="zoom:67%;" />

**attribute_name_index**是指向CONSTANT_Utf8_info类型常量的索引，为固定值”Code“，attribute_length为属性值的长度，由于attribute_name_index和attribute_length的长度固定为6个字节，所以整个属性表的长度为attribute_length + 6。**max_stack**是最大操作数栈深度，在方法执行的任何时候操作数栈都不会超过这个深度，虚拟机在执行时根据这个值来分配栈帧中的操作数栈深度，也就是操作数栈的深度在编译完成后就是确定的。**max_local**是局部变量所需的存储空间，单位是槽，方法参数、异常参数、局部变量都依赖局部变量表来存储，在这里当某个变量超出其作用域后会对其所占用的变量槽进行复用，也就是这个max_local并不只是简单统计变量个数，而是会根据变量的作用域来计算最小需要的槽的个数。**code_length和code**用来存储Java编译后生成的字节码指令。Java目前规定了约200条指令，而使用一个u1类型的变量最大可以存储0xFF条指令（255条）。将指令顺序排列就构成了方法中的实际代码执行顺序了。这里需要注意code_length虽然存储空间为u4，即理论上最大可以存储0xFFFF条指令（2的32次幂），但是Java虚拟机规范规定了，如果代码超过0xFF条指令（65535）就会拒绝编译。所以code_length的高2字节始终为0。我们接下来回到上面的例子：

<img src="/images/JVM/JVM-Class-Method-XXDAnly1.png" alt="JVM-Class-Method-XXDAnly1" style="zoom:67%;" />

第一个方法属性表中的0x0009表示当前属性的名称在0x0009号常量中存储，查看可知是`#9 = Utf8  Code`，接着u4类型的0x0000001d表明当前属性值的长度为29个字节，接着两个0x0001表示最大栈深度和局部变量表大小均为1，这也可以在反编译后的信息中看到：

```
{
  public Main();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0

  public void sayHello();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String Hello world
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 3: 0
        line 4: 8
}
```

可以看到，构造方法中stack=1，locals=1，args_size=1，**对于每个方法而言，参数列表都有一个隐式参数`this`用于只是当前类实例，这就导致了locals和args_size均为1的情况，如果是静态方法，那么就不会有这个额外的this参数**。然后一个u4变量0x00000005表明code长度为5，所以后续的5条指令2a b7 00 01 b1就代表了实际的指令，他们分别是`2a: aload_0`，`b7: invokespecial`，00 01是invokespecial的参数，表明调用了常量池中0x0001这个方法，对应的就是构造方法"java/lang/Object."\<init\>":()V"，接着`b1: return`这样实际就保存了这个方法需要执行的全部字节码指令。

对于sayHello方法也同理，属性值长度为0x00000025，stack=2，locals=1，args_size=1，代码长度为0x00000009，分别是b2 00 02 12 03 b6 00 04 b1，具体分析这里就不展开了，可以查看对应的字节码指令了解具体的指令。

接着来看Code属性，在代码之后列出的就是异常表了（和Exception属性不一样），这里的异常表指的是如果当前字节码的start_pc行到end_pc行（不含）之间存在类型为catch_type的异常或子异常（指向一个CONSTANT_Class_info型常量），那么就转到hander_pc行继续执行：

<img src="/images/JVM/JVM-Class-AttributeTable-Exception-Arch.png" alt="JVM-Class-AttributeTable-Exception-Arch" style="zoom:67%;" />

这个异常表不是必须的，内容也比较通俗易懂，这就不再赘述了。好了以上就是Code属性的内容，Code属性的最后还有attribute值可供属性的嵌套使用，这里就不分析了。

2. Exception属性

Exception属性是指当前方法声明的所有受查异常，也就是定义方法时throws列出的异常，他的结构如下：

<img src="/images/JVM/JVM-Class-AttributeTable-ExceptionAttr-Arch.png" alt="JVM-Class-AttributeTable-ExceptionAttr-Arch" style="zoom:67%;" />

这里的exception_index_table就是指向常量池中具体异常类型CONSTATNT_Class_info的序号。

3. ConstantValue属性

ConstantValue属性的作用是通知虚拟机自动为静态变量赋值，只有被static关键字修饰的变量（类变量）才能使用这个属性。对于非static类型的变量的初始化是在实例构造器\<init\>()中进行的；而对于类变量则有两种方式可以选择：1. 在类构造器\<clinit\>()方法中初始化；2. 使用ConstantValue属性。

Javac编译器的选择是，对于同时使用final和static修饰的变量并且这个变量是String类型的，那么就使用ConstantValue初始化；如果这个变量没有被final修饰，或者非基本类型及字符串，则将会选择在\<clinit\>()方法中进行初始化。我们来看ConstantValue的属性结构：

<img src="/images/JVM/JVM-Class-AttributeTable-ConstantValue.png" alt="JVM-Class-AttributeTable-ConstantValue" style="zoom:67%;" />

这里constantvalue_index代表了常量池中一个字面量常量的引用，根据字段类型的不同，字面量可以是CONSTANT_Long_info、CONSTANT_Float_info、CONSTANT_Double_info、CONSTANT_Integer_info和CONSTANT_String_info常量中的一种。

好了以上就是分析的三种属性表，其它属性表这里不再分析了。

### 虚拟机类加载机制

虚拟机的类加载机制是指虚拟机把描述类的数据从Class文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型。**Java类型的加载、连接和初始化都是在运行阶段完成的**。

一个类型从被加载到虚拟机内存中开始到卸出内存为止，它的整个生命周期将会经历**加载、验证、准备、解析、初始化、使用和卸载**七个阶段，其中**验证、准备、解析**三个部分统称为连接。

<img src="/images/JVM/JVM-Class-Load-Process.png" alt="JVM-Class-Load-Process" style="zoom:67%;" />

其中加载、验证、准备、初始化和卸载这五个阶段的执行顺序是确定的，但是解析阶段则不一定，它有可能在初始化之后进行（为了支持动态绑定）。由于类加载各个阶段可能会激活其它阶段运行，所以上述这些阶段的顺序并不是说是按照这个顺序完成的，而是按照这个顺序开始。

#### 类加载的时机

Java虚拟机规范并没有规定类的加载时机，但是严格规定了6种初始化时机，即有且只有遇到这六种情况且类还未初始化时需要立即对类进行初始化（在初始化之前加载、验证、准备必须开始）：

1. 遇到new、putstatic、getstatic和invokestatic四条字节码指令时，即使用new初始化类实例、获取或设置静态变量（除同时被final修饰、已经在编译期放入常量池中的静态常量）以及调用一个类的静态方法时；
2. 使用反射调用时，如果类还未初始化那么需要先初始化
3. 当初始化类时如果发现其父类还没有被初始化，那么需要先初始化父类
4. 当虚拟机启动时，用户需要指定一个执行的主类，虚拟机需要先初始化这个主类
5. JDK7加入的动态语言支持，如果一个MethodHandle实例最后的解析结果是REF_getStatic、REF_putStatic、REF_invokeStatic、REF_newInvokeStatic四种类型方法的句柄，并且这个方法句柄对应的类没有被初始化，则需要先触发其初始化。
6. 当某个定义了默认方法的接口的实现类被初始化时该接口要先被初始化。

比较典型的几种不会加载类的情况（可以添加虚拟机参数 -Xlog:class+load来查看类加载情况）：

1. 通过子类引用父类静态对象不会引起子类的初始化：

   ```java
   class SuperClass {
       static {
           System.out.println("Load SuperClass");
       }
       
       public static int value = 123;
   }
   
   class SubClass extend SuperClass {
       static {
           System.out.println("Load SubClass");
       }
   }
   
   public class Test {
       public static void main(String[] args) {
           System.out.println(SubClass.value);
       }
   }
   ```

   上述代码这种间接访问并不会导致SubClass类被初始化（也即static构造块内的代码不会被执行，即\<clinit\>()不会被执行）

   > 回顾一下父类和子类的加载顺序：
   >
   > 1. 父类静态变量（取决于代码顺序，如果在静态块后那会先执行静态块代码）
   > 2. 父类静态代码块（若有多个按代码先后顺序执行）
   > 3. 子类静态变量
   > 4. 子类静态代码块（若有多个按代码先后顺序执行）
   > 5. 父类非静态变量
   > 6. 父类非静态代码块（若有多个按代码先后顺序执行）
   > 7. 父类构造函数
   > 8. 子类非静态变量
   > 9. 子类非静态代码块（若有多个按代码先后顺序执行）
   > 10. 子类构造函数

2. 通过数组类引用类不会造成类初始化：

   ```java
   public class Test {
       public static void main(String[] args) {
           SubClass[] classes = new SubClass[10];
       }
   }
   ```

   虽然定义了SubClass数组但是不会引起其初始化。(但是创建数组的newarray指令会导致虚拟机初始化一个L[com.test.SubClass类的初始化。

3. 如果使用的是类中的编译阶段存入常量池的常量，那么不会导致类的初始化。

   ```java
   class SuperClass {
       static {
           System.out.println("Load SuperClass");
       }
       
       public static final String value = "Hello";
   }
   
   public class Test {
       public static void main(String[] args) {
           System.out.println(SuperClass.value);
       }
   }
   ```

   这里调用了SuperClass类中的字符串常量，这里也不会触发SuperClass类的初始化，因为常量在编译期放入常量池，所以实际使用常量时与类没有联系，也就不会导致类的初始化。

以上就是类加载的时机问题，但是这里要注意一点，接口也是会加载的。接口的加载是由于其中定义的成员变量以及default方法导致的，接口中是不能写static方法的，但是编译器任然会为其生成\<clinit\>函数。接口加载和类加载的一个明显不同就是，接口加载不需要所有父接口都被加载，但是类需要，类加载需要所有使用的父类和接口都被加载才能使用，接口的父接口会在真正使用时被加载。

#### 类加载的过程

##### 加载

类加载阶段要完成以下三件事：

1. 通过一个类的全限定名来获取定义此类的二进制字节流
2. 将这个字节流所代表的静态存储结构转化为方法区（元空间）的运行时数据结构
3. 在内存中生成一个代表此类的Class对象，作为方法区这个类的各种数据的访问入口

对于**数组类以外的类**既可以通过虚拟机内置的引导类加载器来加载，也可以通过用户自定义的类加载器来加载。

对于**数组类**，它本身不通过类加载器创建，而是由虚拟机直接在内存中动态构造，数组类的创建遵循以下规则：

​	a. 如果数组的组件类型（去掉一个维度的类型）是引用类型，那就递归加载这个引用类型，如果还是数组，那么继续这个步骤；如果不是数组，那么调用对应类的加载过程；数组将被标记在和实际引用类型的类加载器的类名称空间上。（即数组和具体元素加载器一致）

​	b. 如果数组的组件类型是基本类型，那么将数组标记为与引导类加载器相关联

​    c. 数组类的可访问性和它的组件类型的可访问性一致，基本类型默认为public

加载完成之后，代表类的二进制字节流就转换为一个运行时数据结构存储在方法区中了，同时也会生成一个Class对象。

##### 验证

验证阶段的目的是为了确保Class文件中的字节流中包含的信息复合虚拟机规范的全部约束要求，保证这些信息被当做代码运行后不会危害虚拟机自身的安全。**验证阶段大概会完成以下四个阶段的检验动作：文件格式验证、元数据验证、字节码验证和符号引用验证**。

文件格式验证：保证输入的字节流能正确地解析并存储于方法区，格式上符合描述一个Java类型信息的要求。

元数据验证：对类的元数据信息进行语义校验，保证不存在与语言规范定义相悖的元数据信息。

字节码验证：通过数据流分析和控制分析，确定程序语义是合法的、符合逻辑的。

符号引用验证：可以看作是对类自身以外（常量池中的各种符号引用）的各类信息进行匹配性校验，它发生在虚拟机将符号引用转化为直接引用的时候，这个转化动作在连接的第三阶段——解析阶段中发生。

##### 准备

准备阶段是正式为类中定义的变量（即静态变量）分配内存并设置类变量初始值的阶段。**从概念上讲这些变量作为类变量即类元数据的一部分，应该随类元数据一起放在方法区中，JDK8之后即元空间中，但是实际实现的时候这部分内容实际放在了堆中。**

准备阶段并不是为了给静态变量赋值，我们知道即使是直接给静态变量赋值，例如下面这个语句：

```java
public static int i = 10;
```

实际上这个赋值的过程是在静态构造块即\<clinit\>函数中实现的，而这个赋值操作实际上发生在初始化阶段。准备阶段要做的是为这个变量分配内存空间，并且设置一个零值，这样即使没有初始化这个静态变量，我们也可以访问到一个”合理“的值。各个变量的零值如下：

<img src="/images/JVM/JVM-Class-Prepare-ZeroValue.png" alt="JVM-Class-Prepare-ZeroValue" style="zoom:67%;" />

但是，如果一个变量存在ConstantValue属性（这在类文件格式属性表中说过），那么实际上会用ConstantValue对应的值来进行初始化，能使用ConstantValue的变量一定是static final类型的变量：

```java
public static final int i = 10;
```

**这里我们知道，对于static final String类型的变量，在编译后是直接放入常量池中的，在准备阶段实际上不进行内存分配，而只是创建一个指向常量池的指针；但是对于其它static final类型，则在编译阶段会生成ConstantValue，并在准备阶段将这个值进行初始化。**

##### 解析

解析阶段是虚拟机将常量池中的符号引用替换为直接引用的过程，解析的对象就是在Class文件中定义的CONSTANT_Class_info、CONSTANT_Fieldref_info、CONSTANT_Methodref_info等类型的常量。

**符号引用**就是用任何形式的字面量，只要能无歧义地定位到目标的常量。它和虚拟机内部实际的内存布局无关，引用的对象也不一定已经被加载到虚拟机内存中。

**直接引用**就是指直接指向目标的指针、相对偏移量或句柄。它和虚拟机内部实际的内存布局联系紧密。

Java虚拟机保证多次解析同一个符号引用得到的结果是一致的，如果失败那就一直失败，即使以某种方法加载到内存中了，解析结果依然是失败；如果成功那就一直成功。（除invokedynamic以外，动态调用点方法不一定成立的原因是解析操作只有实际执行到这条指令时才能触发，所以解析的结果是不确定的）

解析方法主要针对类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用点限定符这7类符号引用进行。这里只讨论前4种：

1. 类或接口的解析

   如果一个类型处于某个类中，那么就要分数组类型和非数组类型两种进行解析。如果是非数组类型，那么直接交给所在类的类加载器加载即可（这个过程可能会触发其它类的加载工作）；如果是数组类型，那么则加载组件类型，并动态构造数组。加载完毕后对符号引用进行验证以确保类具有对类型的访问权限。

2. 字段解析

   字段解析的流程是：先加载类（字段表内class_index会指向一个CONSTANT_Class_info常量记录了所属类），再按当前类或接口、所有父接口、所有父类的顺序查找字段，查找失败会抛出找不到字段异常，查找成功则会验证访问权限。

3. 方法解析

   方法解析的第一步同样要查找class_index对应的类或接口的符号引用，然后判断当前类是否是一个接口，如果是一个接口那么会直接抛出异常（接口类方法是无法解析的，default方法并不由这里解析）；如果是一个类那么就查找是否有简单名称和描述符都匹配的方法，如果找不到则继续搜索所有父类，如果所有父类都找不到说明不存在这个方法，此时还会搜索所有父接口，如果在父接口中找到了说明当前类是一个抽象类，抛出异常，否则都找不到则抛出一个找不到方法异常。同样查找结束后也会校验访问权限。

4. 接口方法解析

   接口方法解析也需要找到对应的类或接口的符号引用，但是与方法解析相反，如果找到的是一个类，那么抛出异常，因为接口无法继承自一个类，所以接口方法不可能对应一个类。如果对应的是一个接口，那么搜索当前接口、搜索所有父接口直到Object类为止。如果存在多个匹配的方法，那么只会选择其中的一个返回，如果没有匹配，那么会抛出异常。

##### 初始化

这里的初始化指的是对类进行初始化而不是对实例进行初始化，类的初始化的本质就是执行类构造器\<clinit\>的过程，也是首个开始执行用户代码的过程。（实例初始化就是指实例化对象，即调用构造函数\<init\>的过程）。**初始化过程的目的是为了根据开发人员的目的通过编码去初始化类变量和其他资源（准备阶段已经由虚拟机进行一次初始化，但只是单纯的置0和对final值的设置，这一步会真正给类变量赋上我们所设定的值）**。

\<clinit\>()方法是由编译期生成的，其通过自动收集对所有类变量的赋值以及static块代码组合而成（**也就是说即使直接给类变量赋值，它的初始化也是依赖\<clinit\>方法执行的，实例化同理，即使直接给字段赋值，具体的初始化也发生在\<init\>方法中，如果使用UNSAFE跳过\<init\>那么这种赋值就不会成功，但是类变量不一样，\<clinit\>方法是无法跳过的**）。编译器对类变量赋值和static块收集操作严格按照代码顺序，就是说static块内只能访问到定义在它之前的变量，定义在它之后的变量可以完成赋值，但是无法访问。例如：

```java
public class Test {
    static {
        i = 10;
        System.out.println(i);
    }
    
    static int i = 5;
}
```

上述代码中给i赋值为10是可以通过的，但是访问i的值会出现非法前向引用错误。**同时由于收集按照代码顺序的缘故，static块中的赋值i = 10会被后续i = 5的赋值覆盖**。

虚拟机保证在子类的\<clinit\>方法执行前，父类的\<clinit\>方法一定已经执行完毕，所以这也就解释了父类中静态变量会先于子类赋值的原因（实例化构造同理）。但是\<clinit\>方法不是必须的，也就是说如果方法或接口中没有类变量或static块（接口不允许声明static块），那么就不会生成\<clinit\>方法。**而如果接口中有\<clinit\>方法，就说明其存在类变量的赋值，此时只会执行当前接口的\<clinit\>方法而不会执行父接口的\<clinit\>方法，父接口的\<clinit\>会等到真正使用父接口中的类变量时才会被调用**。同样，虚拟机保证\<clinit\>方法只会被执行一次，是天生线程安全的方法，只有一个线程会获取锁，如果\<clinit\>执行耗时较长，那么其它使用这个类的方法也会阻塞（因为这是类加载步骤，未完成这一步说明类还没有被成功加载，其它用到这个类的地方会触发加载过程，但是\<clinit\>只允许一个线程进入，其它线程必须要阻塞，而当\<clinit\>运行结束后，其它线程发现类已经成功初始化就不会再次执行\<clinit\>方法了）。

#### 类加载器

类加载器的目的很简单就是为了将二进制字节流加载进来并转换为运行时数据结构。但是对于任意的一个类都必须由加载它的类加载器本身和这个类本身一起共同确立其在Java虚拟机中的唯一性，每一个类加载器都有一个独立的类名称空间。

所以在比较两个类是否相等时，必须建立在两个是被同一个类加载器加载的前提下才是有意义的，否则即使这两个类来源于同一个Class文件，但是被不同的类加载器加载，它们也会被认为是不同的。

##### 双亲委派模型

首先先说类加载器的分类：

对于虚拟机来说，它认为只存在两种类加载器：1. **启动类加载器**，它由C++实现，是虚拟器的一部分；2. 其它类加载器，由Java实现，独立于虚拟机外部，继承自ClassLoader类。

对于开发人员来说，则是一个**三层类加载器，双亲委派的类加载架构**：1. **启动类加载器**（Bootstrap Classloader），负责加载<JAVA_HOME>\lib目录下以及-Xbootclasspath参数指定的路径中存放的，而且是虚拟机能够识别的类库到虚拟机内存中。无法直接被用户调用，是最上层的类加载器，如果要使用这个类加载器，直接让ClassLoader为null即可；2. **扩展类加载器**（Extension Classloader），负责加载<JAVA_HOME>\lib\ext目录下以及被java.ext.dirs系统变量指定路径中的所有类库；3. **应用程序类加载器**，负责加载用户类路径（ClassPath）上所有类库，如果没有自己定义过类加载器，那么这个类加载器就是默认的类加载器。

类加载器采用的是双亲委派模型，即除了顶层的启动类加载器，其它类加载器都应该拥有自己的父类加载器，在进行类加载的时候，当前类加载器首先不会自己尝试去加载这个类，而是交给父类加载器去完成，最终所有的请求都应该被传送到最顶层的启动类加载器完成，只有父类加载器反馈无法加载时，子类加载器才会尝试自己加载。结构图如下：

<img src="/images/JVM/JVM-Class-ClassLoader-Model.png" alt="JVM-Class-ClassLoader-Model" style="zoom:25%;" />

双亲委派模型的好处就是让类加载有了层次关系，保证了大多数类都由公共的类加载器加载，例如Object类始终由启动类加载器加载，这样就防止由于类加载器不同而产生了多个不同的Object类。

##### 破坏双亲委派模型

**线程上下文类加载器**是破坏双亲委派模型的一个典型例子，它允许线程设置一个子类加载器，这个子类加载器是允许父类使用的，如果线程未设置子类加载器，那么默认是使用父线程的类加载，如果父线程也没有设置，那么默认使用应用程序类加载器。这实际上就是一种父类加载器去逆向使用子类加载器的情况。



### 虚拟机字节码执行引擎

#### 运行时栈帧结构

虚拟机以方法作为最基本的执行单元，栈帧则是用于支持虚拟机进行方法调用和方法执行背后的数据结构，它也是虚拟机栈中的栈元素。**栈帧存储了局部变量表、操作数栈、动态链接和方法返回等信息**。一个方法的执行也就对应着一个栈帧入栈和出栈的过程。栈帧中的局部变量表、操作数栈的大小已经在编译期确定并且写入到方法Code属性中了，所以**栈帧的大小在编译期就能确定，并不会受到程序运行期变量数据的影响**。对于执行引擎而言只有处于栈顶部的栈帧才是处于运行状态的，称为当前栈帧，这个栈帧关联的方法称为当前方法。（正是因为方法是以栈帧的方法存在的，所以调试时“回退到上一步”实现就是通过丢弃栈顶活动栈帧来实现的）

<img src="/images/JVM/JVM-StackFrame-Logic-Arch.png" alt="JVM-StackFrame-Logic-Arch" style="zoom:30%;" />

##### 局部变量表

局部变量表的大小由方法对应Code属性中的max_local属性决定，在编译期就可知。**局部变量表用于存储方法参数和方法内部定义的局部变量，局部变量表中的变量以槽作为最小单位。**

1. ***注意***，reference也是一种局部变量表中的变量，但是虚拟机没有规定它的结构，一般来说，reference至少应该实现：1. 直接或者间接的获取对象在堆中的数据存放的起始地址或索引；2. 直接或间接的获取对象的类型信息。（这两点在讨论直接引用和句柄时说过对象的存储方式的问题）

2. ***注意***，关于long、double占用两个槽的并发问题：由于局部变量表是线程私有的，所以不会产生数据竞争和线程安全问题。

局部变量表的分配顺序：首先**0号槽固定是this指针**，其次是所有方法参数，最后是方法中用到的变量，这些变量按照定义的顺序排列。

3. ***注意***，变量槽并不是无限制的进行分配的，当某个变量超过了其作用域之后，那这个变量对应的槽就可以被其它变量重用。这可能会影响GC的行为，因为如果一个局部变量超出作用域之后没有其它变量去重用其对应的槽，那么由于存在局部变量表对该变量的强引用，那么这类变量是无法被GC的，例如下面的例子：

```java
public class Test {
	public static void test1() {
        {
            byte[] placeholder = new byte[64 * 1024 * 1024];
        }
        // 无法被GC
        System.gc();
    }
    
    public static void test2() {
        {
            byte[] placeholder = new byte[64 * 1024 * 1024];
        }
        int a = 0;
        // 槽被复用，变量和局部变量表的强引用关系被消除，可以被GC
        System.gc();
    }
}
```

4. ***注意***，局部变量和类变量不同，局部变量不存在准备和初始化阶段，如果没有赋初值是不能被使用的。

##### 操作数栈

操作数栈的大小由方法对应Code属性中的max_stacks属性决定，在编译期就可知。32位变量所占用的栈容量为1，64位变量所占用的栈容量为2。

##### 动态连接

每个栈帧中都包含一个指向运行时常量池中该栈帧所属方法的引用，持有这个引用是为了支持方法调用过程中的动态连接。字节码中方法调用指令就以常量池中指向方法的符号引用作为参数。这些符号引用一部分会在类加载或者第一次使用时就被转化为直接引用，这种转化称为**静态解析**。另外一部分将在每一次运行期间都转换为直接引用，这部分就称为**动态连接**。

##### 方法返回地址

方法返回时（无论是正常返回还是异常返回）可能都需要在栈帧中保存一些信息，这些信息用来帮助恢复它的上层主调方法的执行状态。一般来说方法正常退出时，主调方法的PC计数器的值就可以作为返回地址，栈帧中很可能保存了这个值。而方法异常返回时，返回地址是要通过异常处理器来确定的，栈帧中一般不保存这部分信息。

方法退出相当于栈帧的出栈操作，需要完成：恢复上层方法的局部变量表和操作数栈，把返回值（如果有）压入调用者栈帧的操作数栈中，调整PC计数器等。

#### 方法调用

方法调用阶段要做的事是确定调用的究竟是哪一个方法，而不是方法内部的运行情况。一切方法调用在Class文件里面存储的都只是符号引用而不是实际运行时内存布局中的入口地址（即直接引用）。就是这种运行时才确定实际调用哪个方法的机制使得Java具有更强的动态扩展能力。

##### 解析

**方法在编译完成后会将一部分运行期不可改变的方法转化为直接引用，这类方法主要有静态方法和私有方法两类**。前者直接与类型挂钩，后者则在外部不可被访问，这决定了它们在外部不可能通过继承等别的方式重写出其它版本，所以适合在类加载阶段进行解析。（注意静态方法不可被重写，静态方法的行为不具备多态性）。

调用不同类型的方法，字节码指令集中设计了不同的指令，Java虚拟机支持以下5条方法调用字节码指令：

- invokestatic：调用静态方法
- invokespecial：调用实例构造器方法、私有方法和父类中的方法
- invokevirtual：调用所有的虚方法
- invokeinterface：调用接口方法，会在运行时确定一个实现该接口的对象
- invokedynamic：现在运行时解析出调用点限定符所引用的方法，然后再执行该方法。前面4条指令，分派逻辑都固定在虚拟机内部，而这条分发的分派逻辑是由用户设定的引导方法来决定的。

**虚方法和非虚方法**：只要能被invokespecial、invokespecial指令调用的方法都可以在解析阶段确定唯一的调用版本，Java中符合这个条件的只有静态方法、私有方法、实例构造器、父类方法以及final方法（尽管它使用invokevirtual调用），所以这些方法在编译期能确定调用版本的方法就是非虚方法，其它方法则是虚方法。

final方法虽然使用invokevirtual指令调用，但是由于其不可覆盖的特点，所以在编译期也可以唯一确定一个调用版本，所以它也是非虚方法。

##### 分派

分派的逻辑要比解析复杂，解析在编译期就可以玩完成，是一种唯一确定一种的最优情况，但是分派则不是。分派可以是静态的也可以是动态的，可以是单分派也可以是多分派。上述两类结合起来分派就具有静态单分派、静态多分派、动态单分派和动态多分派四种情况。

分派的概念是伴随着多态性出现的，试想一下，如果不存在继承，那么调用的方法是唯一确定的，那么所有的方法调用在编译期都可以确定唯一的调用版本，分派就没有存在的意义。所以分派的存在必然是和多态性想挂钩的。

1. ##### 静态分派

先看示例代码：

```java
public class StaticDispatcherTest {
    static abstract class Human {}

    static class Man extends Human {}

    static class Woman extends Human {}

    public void sayHello(Human human) {
        System.out.println("Hey guys");
    }

    public void sayHello(Man man) {
        System.out.println("Hey, gentleman");
    }

    public void sayHello(Woman woman) {
        System.out.println("Hey, lady");
    }

    public static void main(String[] args) {
        StaticDispatcherTest sd = new StaticDispatcherTest();
        Human man1 = new Man();
        Human woman = new Woman();
        Man man2 = new Man();
        sd.sayHello(man1);
        sd.sayHello(man2);
        sd.sayHello(woman);
    }
}
```

这段代码的执行结果为: 

Hey guys	

Hey, gentleman	

Hey guys

很明显，如果将子类类型赋值给父类类型，那么最终选择重载时会使用父类类型参数的重载函数。要想知道为什么是这样，首先要明确，对于父类Human而言，它是变量的**静态类型**或**外观类型**，而子类Man则是变量的**实际类型**或**运行时类型**。静态类型和实际类型在运行中都会发生变化，区别就是，静态类型的变化仅仅在使用时发生，变量本身的静态类型不会被改变，而且最终的静态类型在编译期是可知的；而实际类型变化的结果要在运行期才可确定，编译期在编译程序的时候并不知道一个对象的实际类型是什么。

从上面的例子可以看出，**虚拟机（编译器）在重载时是通过参数的静态类型而不是实际类型来判断应该调用哪个重载方法的**。

**所有依赖静态类型来决定方法执行版本的分派动作，都称为静态分派**。静态分派最典型的应用就是方法重载。**静态分派的动作实际上不是由虚拟机来执行，而是由编译器在编译阶段执行的**。

Java编译期虽然能确定出方法的重载版本，但是这种确定只是一种最佳匹配，而不是唯一匹配（这就是解析和分派的区别），比如举个例子，如果有下面这些方法:

```java
public class Overload {
    public static void sayHello(char c) {}
    public static void sayHello(int c) {}
    public static void sayHello(long c) {}
    public static void sayHello(Character c) {}
    public static void sayHello(Serializable c) {}
    public static void sayHello(Object c) {}
    public static void sayHello(char... c) {}
}
```

当调用`char c = 'a'; sayHello(c);`时，上述7个方法都能匹配。匹配的顺序就是上面代码的顺序。如果有char参数的函数，则优先调用char参数；否则会发生一次自动类型转换，char类型可以安全的转型为int、long、float、double，并且顺序也严格按照这里的顺序，先匹配整数，再匹配浮点数，但是不会与short和byte匹配，因为这种类型转换是不安全的；如果没有基本类型匹配，那么会自动装箱为Character类型，如果没有Character类型会匹配Character类实现的接口Serializable，如果没有Serializable接口，那么会匹配Character继承的Object类，最后才会匹配变长队列类型。（但是变长队列类型必须要严格匹配，也就是不能匹配`int...c`类型）如果出现了多个匹配的可能，那么会抛出异常拒绝编译（例如Character类实现了Seriablzable和Comparable接口，如果同时提供参数为这两个接口的方法，那么会报告类型模糊。

2. ##### 动态分配

上面说的静态分配和重载密切相关，那么现在说的动态分配就和重写密切相关了。我们先来看代码：

```java
public class DynamicDispatcherTest {
    static abstract class Human {
        public abstract void sayHello();
    }

    static class Man extends Human {
        @Override
        public void sayHello() {
            System.out.println("Hey, gentleman");
        }
    }

    static class Woman extends Human {
        @Override
        public void sayHello() {
            System.out.println("Hey, lady");
        }
    }

    public static void main(String[] args) {
        Human man = new Man();
        Human woman = new Woman();
        man.sayHello();
        woman.sayHello();
    }
}
```

这里sayHello方法不再接受参数，而是作为类内部方法，那么此时调用的结果很显然就是：

Hey，gentleman

Hey，lady

这个结果并不难理解，由于多态性的存在，导致了它调用了实际类型的方法。**也就是说动态分派是以实际类型为准的**。我们接着将上面的main函数反编译一下，来看字节码：

```
 0: new           #2                  // class com/learn/jvm/chcp8/DynamicDispatcherTest$Man
 3: dup
 4: invokespecial #3                  // Method com/learn/jvm/chcp8/DynamicDispatcherTest$Man."<init>":()V
 7: astore_1
 8: new           #4                  // class com/learn/jvm/chcp8/DynamicDispatcherTest$Woman
11: dup
12: invokespecial #5                  // Method com/learn/jvm/chcp8/DynamicDispatcherTest$Woman."<init>":()V
15: astore_2
16: aload_1
17: invokevirtual #6                  // Method com/learn/jvm/chcp8/DynamicDispatcherTest$Human.sayHello:()V
20: aload_2
21: invokevirtual #6                  // Method com/learn/jvm/chcp8/DynamicDispatcherTest$Human.sayHello:()V
24: return
```

其中0到15行都在准备两个对象，调用invokespecial来实例化对象。第16行可以看到，先将当前类加载到了栈顶（aload_1），然后使用invokevirtual调用了Human类的sayHello方法，同样20行将当前类加载到栈顶，然后invokevirtual调用Human类的sayHello方法。所以从这里可以看出，动态分派的核心就在于invokevirtual方法。invokevirtual方法的解析过程大概分为以下几步：

	1. 找到操作数栈顶的第一个元素所指向的对象的实际类型实际类型，记为C
	2. 如果类型C中找到与常量中的描述符和简单名称都相符的方法，那么进行访问校验，如果校验通过，那么返回这个方法的直接引用，查找过程结束；否则抛出异常；
	3. 如果C找没有找到相符的方法，那么按照继承关系从下往上依次对C的各个父类进行第二步的搜索和校验过程；
	4. 如果始终没有找到匹配的方法，那么说明当前类的方法是抽象方法，不能被调用，抛出异常。

正是由于invokevirtual的这种特性，它的实际类型即为当前栈顶常量所指向的类型，这就是重写的本质，重写方法运行时会将其实际类型对应的场景放到栈顶，invokevirtual根据栈顶所指向的实际类型完成方法调用。这同时也意味着，如果我们把第20行的aload_2改为aload_1，那么实际类型就会发生改变，最终调用的方法也会不同。

同时也可是说明，对于Java而言，其多态性是由invokevirtual的运行机制来保证实现的，所以**字段永远不具备多态性，字段的存取并不依赖invokevirtual指令完成，也就不具备多态性了**。比如我们看下面这个方法：

```java
public class FieldDispatcherTest {
    static class Father {
        private int money = 1;

        public Father() {
            money = 2;
            showMoney();
        }

        public void showMoney() {
            System.out.printf("I am father, my money is %d\n", this.money);
        }
    }

    static class Son extends Father {
        private int money = 3;

        public Son() {
            money = 4;
            showMoney();
        }

        @Override
        public void showMoney() {
            System.out.printf("I am son, my money is %d\n", this.money);
        }
    }

    /*
    * 这里先进入Son的构造函数，由于Son继承自Father类所以<init>构造函数会先用invokespecial
    * 调用父类Father的构造函数。此时包括Son类成员变量money在内的所有变量都没有被初始化（但是在准备阶段赋了初值0）
    * 父类Father利用invokevirtual调用的showMoney是子类的方法（重写），由于字段不存在多态，所以子类Son直接访问了
    * 自己的money字段（此时还只有准备阶段赋的初值0）。接着回到Son的构造方法中会先对字段赋值3(putfield)，
    * 再赋值4(putfield)，最终完成构造函数
    * */
    public static void main(String[] args) {
        Father father = new Son();
        System.out.printf("This guy has %d money\n", father.money);
    }
}
```

注释里写的很清楚了，这里父类的money只是被子类同名变量money给隐藏了，在父类构造函数中调用的方法实际上是一个虚方法，它实际调用的是子类的方法，所以永远也不可能输出父类money的值。这一点从字节码上可以很清楚的得知。

3. ##### 单分派和多分派

先给出结论，**Java中的静态分派属于多分派，动态分派属于单分派**。

我们定义方法的接受者与方法的参数统称为宗量，单分派是指依赖一个宗量对目标进行选择，多分派是指依赖多个宗量对目标进行选择。接着看下面的代码：

```java
public class MultiDispatcherTest {
    static class Tom {}
    static class Jerry {}

    public static class Father {
        public void choose(Tom tom) {
            System.out.println("Father choose Tom");
        }

        public void choose(Jerry tom) {
            System.out.println("Father choose Jerry");
        }
    }

    public static class Son extends Father {
        @Override
        public void choose(Tom tom) {
            System.out.println("Son choose Tom");
        }

        @Override
        public void choose(Jerry tom) {
            System.out.println("Son choose Jerry");
        }
    }

    public static void main(String[] args) {
        Father father = new Father();
        father.choose(new Tom());

        Father son = new Son();
        son.choose(new Jerry());
    }
}
```

输出为：

Father choose Tom
Son choose Jerry

第一个输出丝毫没有问题，类内是静态分派，类间动态分派，对于类间而言它只有唯一的选择就是Father，但是对于类内它有多个选择即Father::choose(Tom)和Father::choose(Jerry)，由于类内是静态分派，所以静态分派是多分派。

第二个输出也很简单，由于我们知道由于多态性的存在，这里实际调用的是Son类的方法，而在这里选择Son类就是动态分派，动态分派显然只会根据实际类型来选择应该分派到哪个方法，而不会管分派之后内部到底是选择了哪一个，所以动态分派是单分派。

这个东西很好理解，也就是说对于动态分派而言，能影响分派的只有实际类型；而对于静态分派而言，能影响分派的有若干个方法，所以这也就决定了静态分派是多分派而动态分派是单分派。

#### 动态类型语言支持

动态类型语言是指它的类型检查的主体过程是在运行期完成的而不是编译期。Java作为一个静态语言，实际上在编译期就生成了各种方法、字段等信息的符号引用，并且保存到了常量池中，所以Java不是一个动态类型语言。

##### 方法句柄

方法句柄要解决的一个问题就是，Java语言中是无法将函数作为参数进行传递的，所以说方法句柄就赋予了函数成为变量以及参数的能力。就像下面这段代码：

```java
public class MethodHandleTest {
    static class ClassA {
        public void println(String s) {
            System.out.println("ClassA print: " + s);
        }
    }

    public static void main(String[] args) throws Throwable {
        Object obj = System.currentTimeMillis() % 2 == 0 ? System.out : new ClassA();

        // 第一个参数是返回值类型，后面的是参数类型
        MethodType type = MethodType.methodType(void.class, String.class);
        // 第一个参数是指引用类型，第二个参数是指方法名，最后一个参数是类型
        // bindTo用来绑定
        MethodHandles.lookup().findVirtual(obj.getClass(), "println", type).bindTo(obj).invokeExact("Hello");
    }
}
```

实际执行了哪个方法是不确定的，这种不确定不是多态的后确定，而是完全不确定。MethodHandle和反射类似，但是也不一样：前者是站在字节码的角度实现的，它的findVirtual、findStatic和findSpecial方法对应字节码中的invokevirtual、invokestatic和invokespecial方法，并且支持字节码层面的调用优化，是轻量级的方法；而后者是站在Java语言层面上的重量级方法，是建立在Java语言之上，对方法的全面映像。

##### invokedynamic

invokedynamic的一个很重要的作用就是消除固化在内部的分派机制，我们知道invokespecial、invokestatic以及final方法的invokevirtual都是静态分派，分派机制固化在编译器中，其它invokevirtual方法的动态分派机制则固化在虚拟机中，用户无法改变。所以invokedynamic的作用就是消除这种固化的分派机制，将分派转移到用户代码中，invokedynamic的作用实际上和上面的方法句柄机制的作用是一样的。

每一个含有invokedynamic指令的位置被称为动态调用点，这条指令的第一个参数不再是传统invoke*方法使用的CONSTANT_Methodref_info常量，而是一个新的常量，类型为CONSTANT_InvokeDynamic_info，从这个新常量中可以得到三个信息：引导方法（Bootstrap Method）、方法类型和名称（具体可以查看常量池一节中关于该常量的描述）。

引导方法是固有的参数，并且返回值规定为java.lang.invoke.CallSite对象，这个对象代表了真正要执行的目标方法调用，虚拟机就是通过执行引导方法获得实际方法的调用点，从而执行实际方法。（有点类似用引导方法包装了一层，引导方法决定了方法的分派逻辑）

invokedynamic在JDK8之后的一个典型用途就是Lambda表达式，具体例子可以看：

> https://blog.csdn.net/zxhoo/article/details/38387141

还有一个很典型的用途就是字符串拼接，在JDK6以前，字符串的拼接是通过调用StringBuffer的拼接函数实现的，JDK6到8是通过调用StringBuilder的拼接函数实现的，但是JDK8开始，字符串拼接通过invokedynamic指令调用一个名为`makeConcatWithConstants`的方法实现，其签名为：

```
makeConcatWithConstants:(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
```

引导方法为：

```
REF_invokeStatic java/lang/invoke/StringConcatFactory.makeConcatWithConstants:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/invoke/CallSite;
```

关于String的内容详见后续String源码分析的文章。