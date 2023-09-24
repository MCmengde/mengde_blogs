# JVM 内存模型

众所周知，Java 以 WOTA (Write once, run anywhere)闻名。为了实现这一点，Sun Microsystems 创造了 Java 虚拟机，它是对底层操作系统的一种抽象，可以解释执行编译的 Java 代码。**JVM**(Java Vritual Machine) 是 JRE(Java Runtime Environment) 的核心组件，它原本是用来运行 Java 代码的，但是现在有一些其他的语言也在使用它(Scala, Groovy, JRuby, Closure)。

本文将着重讲述 Java 虚拟机规范中定义的**运行时数据区**(Runtime Data Areas)，该区域是用来存储应用程序或者 JVM 自身所需的数据的。我将首先给出一个 JVM 概览，然后解释什么是字节码，最后讨论各个不同的数据区。

## 概览

JVM 是对底层操作系统的抽象，无论 JVM 运行在什么的操作系统或硬件上，JVM 中的代码行为都是一致的。例如：

- 无论 JVM 所在的操作系统是 16位、32位或者64位，基本类型`int`一直都是 32 位有符号整型，范围是 $-2^{31}$ 到 $2^{31}$ 。
- 无论底层操作系统是大字节序还是小字节序，JVM 存储和使用内存中的数据都是大字节序。

注意：可能有时候 JVM 具体实现会有差异，但是大体上是一样的。

![img](https://mengde-pic-bed.oss-cn-hangzhou.aliyuncs.com/img/jvm_overview.jpg)

上图给出了 JVM 的概览图：

- 类的源码编译之后得到字节码，然后 JVM 解释执行字节码。虽然 JVM 的含义是 Java 虚拟机，它也可以运行其他语言的代码，比如 scala 以及 groovy，只要这些代码可以被编译为 java 字节码，JVM 就可以运行它们。
- 为了避免磁盘 I/O，字节码由运行时数据区中的类加载器加载到 JVM 中，并且这些代码一直存在与内存中，直到 JVM 停掉，或者加载它的类加载器被销毁。
- 被加载的代码，之后会由执行引擎解释执行。
- 执行引擎需要存储一些数据，比如指向正在执行的代码的指针，它还需要存储开发者代码中处理的数据。

- 执行引擎还需要兼顾处理底层的操作系统。

注意：字节码并非总是被解释执行的，许多虚拟机的执行引擎会把经常使用的字节码编译成本地代码，这种技术被称作即时编译(**JIT**)，在很大程度上加快了 JVM 的速度。被编译的代码存储在通常被称为代码缓存区的区域，因为该区域并未在 Java 虚拟机规范中明确做出规定，我在后面不会再提及这个概念。

## 基于栈的架构

JVM 使用一种基于栈的架构，尽管该架构对开发者是不可见的，但是它对于生成的字节码以及 JVM 结构有很大影响，所以我在这里简要说明这一概念。

JVM 通过执行 Java 字节码（将在下一节中详细说明）中的基础指令来完成开发者代码中的操作，操作数是指令操作的值。根据 JVM 规范，这些操作必须使用操作数栈来传参。

![example of the state of a java operand stack during the iadd operation](https://mengde-pic-bed.oss-cn-hangzhou.aliyuncs.com/img/state_of_java_operand_stack.jpg)

就以两数相加举例，该操作被称为`iadd`（Integer addition），倘若想在字节码中完成 3 加 4：

- 首先，将 3 和 4 压入操作数栈。
- 然后调用`iadd`指令。
- `iadd`指令会把之前的两个值出栈。
- 3 + 4 的结果将被压入操作数栈中，以供其他操作使用。

这种实现函数的方式被称作基于栈的架构，除此之外也有一些其他的处理基础操作指令的方式。比如基于寄存器的架构，操作数存在寄存器中，而非栈。桌面端和服务端的处理器都是使用的寄存器架构，之前的安卓虚拟机 Dalvik 也是使用的这种架构。

## 字节码

因为 JVM 解释执行的是字节码，所以在深入学习之前，我们先来弄清楚字节码是什么。

Java 字节码是由 Java 源码转换而来的一系列基础操作指令，每个指令由以下部分组成：一个字节表示待执行指令的类型（称作 **opcode** 或者 **operation code**），紧接着是一些参数，也可能没有（大多数指令是通过操作数栈传参的）。对于一字节长的 **opcode**，可能会有 256 种不同的指令（从 0x00 到 0xFF），而在 Java 8 中，一共有 204 种指令被使用。

下面列出字节码指令的分类，对于每一种类型，后面给出了相应的说明以及 **opcode** 范围：

| 类型               | 说明                                                         | 指令范围       |
| ------------------ | ------------------------------------------------------------ | -------------- |
| <u>Constants</u>   | 将**常量池中的值**或**已知的值**压入操作数栈                 | 0x00 - 0x14    |
| <u>Loads</u>       | 将**本地变量**加载到操作数栈                                 | 0x15 - 0x35    |
| <u>Stores</u>      | 将操作数栈中的值存到**本地变量**中                           | 0x36 - 0x56    |
| <u>Stack</u>       | 管理**操作数栈**                                             | 0x57 - 0x5f    |
| <u>Math</u>        | 对操作数栈中的数做基本的**数学运算**                         | 0x60 - 0x84    |
| <u>Conversions</u> | **类型转换**                                                 | 0x85 - 0x93    |
| <u>Comparisons</u> | 对两个值做基础的**比较**                                     | 0x94 - 0xa6    |
| <u>Control</u>     | 例如`goto`、`return`的一些基础操作，也包括一些高级的操作，例如循环和带返回值的方法 | 0xa7 - 0xb1    |
| <u>References</u>  | **分配**对象或者数组，**获取**或**检查**对象、方法以及静态方法的引用，也被用来调用静态方法。 | 0xb2 - 0xc3    |
| <u>Extended</u>    | 其他后来添加的指令                                           | 0xc4 - 0xc9    |
| <u>Reserved</u>    | 保留字段，留作不同的 Java 虚拟机内部使用                     | 0xca/0xfe/0xff |

这 204 个操作指令并不复杂，例如：

- `ifeq`(0x99)：比较两个值是否相等
- `iadd`(0x60)：将两数相加
- `i2l`(0x85)：将整型转变为长整型
- `arraylength`(0xbe)：返回数组的长度
- `pop`(0x57)：将操作数栈中的栈顶的元素出栈

字节码由编译器产生，JDK 中内置的标准 Java 编译器是 **javac**。

让我们看看简单的两数相加的例子：

```java
public class Test {
	public static void main(String[] args) {
		int a = 1;
		int b = 15;
		int result = add(a, b);
	}

	public static int add(int a, int b) {
		int result = a + b;
		return result;
	}
}
```

使用`javac Test.java`命令生成字节码文件 Test.class，然而 Java 字节码是二进制码，读起来不方便。Oracle 在 JDK 中提供了一种将字节码转化一系列可读的指令的工具，也就是 javap 工具。

运行`javap -verbose Test.class`命令将返回一下结果：

```asm
Classfile /E:/code/Test.class
  Last modified 2021年9月16日; size 348 bytes
  MD5 checksum b960189a1fb901ac54a2a428efe8611e
  Compiled from "Test.java"
public class Test
  minor version: 0
  major version: 55
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #3                          // Test
  super_class: #4                         // java/lang/Object
  interfaces: 0, fields: 0, methods: 3, attributes: 1
Constant pool:
   #1 = Methodref          #4.#15         // java/lang/Object."<init>":()V
   #2 = Methodref          #3.#16         // Test.add:(II)I
   #3 = Class              #17            // Test
   #4 = Class              #18            // java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = Utf8               Code
   #8 = Utf8               LineNumberTable
   #9 = Utf8               main
  #10 = Utf8               ([Ljava/lang/String;)V
  #11 = Utf8               add
  #12 = Utf8               (II)I
  #13 = Utf8               SourceFile
  #14 = Utf8               Test.java
  #15 = NameAndType        #5:#6          // "<init>":()V
  #16 = NameAndType        #11:#12        // add:(II)I
  #17 = Utf8               Test
  #18 = Utf8               java/lang/Object
{
  public Test();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=4, args_size=1
         0: iconst_1
         1: istore_1
         2: bipush        15
         4: istore_2
         5: iload_1
         6: iload_2
         7: invokestatic  #2                  // Method add:(II)I
        10: istore_3
        11: return
      LineNumberTable:
        line 3: 0
        line 4: 2
        line 5: 5
        line 6: 11

  public static int add(int, int);
    descriptor: (II)I
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=2
         0: iload_0
         1: iload_1
         2: iadd
         3: istore_2
         4: iload_2
         5: ireturn
      LineNumberTable:
        line 9: 0
        line 10: 4
}
SourceFile: "Test.java"
```

从可读的字节码文件中可以看出，字节码文件不仅仅是对源代码的转录，它包含了以下信息：

- 对该类常量池的描述，常量池是 JVM 中的一个数据区，它主要存储类的元数据，例如类中的方法名，以及方法的参数。当一个类被加载到 JVM 中的时候，这些数据就会存到常量池中。
- `LineNumberTable`和`LocalVariable`表示 Java 源码中的行到字节码中的行的映射。
- Java 源码的转录（加上了隐含的构造方法）。
- 指明处理操作数栈的指令，以及更多传入和获取参数的方式。

以下是 .class 文件中的简要信息，供参考：

```java
ClassFile {
  u4 magic;
  u2 minor_version;
  u2 major_version;
  u2 constant_pool_count;
  cp_info constant_pool[constant_pool_count-1];
  u2 access_flags;
  u2 this_class;
  u2 super_class;
  u2 interfaces_count;
  u2 interfaces[interfaces_count];
  u2 fields_count;
  field_info fields[fields_count];
  u2 methods_count;
  method_info methods[methods_count];
  u2 attributes_count;
  attribute_info attributes[attributes_count];
}
```

## 运行时数据区

运行时数据区是用来存储数据的内存区域，这些数据会在开发者的程序中或者 JVM 内部使用。

![overview of the different runtime memory data areas of a JVM](https://mengde-pic-bed.oss-cn-hangzhou.aliyuncs.com/img/jvm_memory_overview.jpg)

上图展示了 JVM 中不同的运行时数据区，有一些区域是每个线程私有的。

### 堆

堆是 Java 虚拟机中所有的线程共享的，它在虚拟机启动时被创建。所有类的实例以及数组都在堆中（使用`new`创建的）。

```java
MyClass myVariable = new MyClass();
MyClass[] myArrayClass = new MyClass[1024];
```

堆由垃圾回收器管理，当开发者创建的实例对象不再被使用时，垃圾回收器会回收这些实例所占用的内存。至于回收的策略，各个虚拟机的实现都不尽相同（Oracle 的 Hotspot 虚拟机提供了多种垃圾回收算法）。

堆是可以动态拓展和压缩的，也可以设定固定的最大最小值。例如，在 Hotspot 虚拟中，用户可以使用`Xms`和`Xmm`参数来规定堆的最值，具体用法为`java -Xms=512m -Xmm=1024m`。

==注意：==堆的大小有一个不能超过的最大值，如果超出了这个最大值，JVM 就会抛出 **OutOfMemoryError**。

### 方法区

方法区也是 JVM 中所有线程共享的，同样，它在虚拟机启动时被创建，并由类加载器从字节码中加载。，只要加载它的类加载器还存在，方法区的数据会一直保持在内存中。

方法区主要保存：

- 类信息（成员变量和成员方法的数量，父类名，接口名，版本号）
- 成员方法和构造方法的字节码
- 每个类加载的运行时常量池

虚拟机规范并没有强制要求方法区要放在堆中，在 Java 7 之前，Hotspot 虚拟机的方法区在永生代中。永生代在堆之后，JVM 以管理堆内存的方式管理永生代内存，永生代默认大小是 64M （可以使用`-XX:MaxPermSize`参数修改）。从 Java 8 开始，Hotspot 将方法区放到了独立的本地内存中，称作元空间，最大可以用空间为系统内存大小。

==注意：==方法区大小也有最大值，如果超出了这个最大值，JVM 就会抛出 **OutOfMemoryError**。

### 运行时常量池

运行时常量池是方法区的一部分，因为它是元数据中很重要的一部分，Oracle 规范中对它进行了单独的描述。每当有类或者接口被加载的时候，运行时常量池就会增加，它就像传统编程语言中的符号表。换句话说，每当某个类、方法或者成员变量被引用时，JVM 都会搜索运行时常量池，来确定它在内存中的真实地址。运行时常量池中也会保存字符串和基本类型的常量。

```java
String myString1 = “This is a string litteral”;
static final int MY_CONSTANT=2;
```

### 程序计数器（线程线程独有）

每一个线程都有它自己的程序计数器，它和线程同时被创建。每任何时刻，每个线程都在执行单个方法的代码，我们称之为该线程的**当前方法**，程序计数器存储着当前正在执行的指令在内存中的地址（在方法区）。

==注意：==如果当前虚拟机执行的是本地方法，程序计数器的状态将是未定义。JVM 的程序计数器的位宽足够大，可以存储返回地址，或者特定平台的本地指针。

### 虚拟机栈（线程独有）

栈区存有很多不同的框架，所以在讨论栈之前，我们先来看看这些框架：

#### 帧

所谓帧就是保存了多个数据的数据结构，这些数据表示了该线程的**当前方法**的状态：

- **操作数栈**：前面以及提到过了，操作数栈被用来存储字节码指令的参数，也可以在 Java 方法调用时传递参数，以及存储调用方法的结果（在栈的顶部）。
- **本地变量数组**：该数组保存当前方法范围内所有的本地变量，数组中可以保存基本类型，引用或者返回地址。该数组的大小在编译时计算确定。Java 虚拟机在方法调用时使用本地变量传参，被调方法的本地变量数组根据调用方法的操作数栈创建。
- **运行时常量池引用**：当前正在执行的方法所在的类的常量池的引用，JVM 利用它来讲方法和变量的符号引用转化为实际的内存地址（例如：`myInstance.method()`）。

#### 栈

每一个线程有一个私有的虚拟机栈，和线程同时产生。虚拟机栈保存上面提到的这些帧，每当方法调用发生时，就会有新的帧加入到栈中，当方法调用完成后，该帧就会被销毁，无论方法是正常完成还是异常完成（异常时会抛出一个不可捕获的异常）。

每一个时刻，只有正在执行的方法的帧是活跃的，该帧被称作**当前帧**，它所对应的方法也就是**当前方法**，当前方法所在的类被称作**当前类**。对本地变量和操作数栈的操作都和当前帧有关。

让我们看看下面具体的例子：

```java
public int add(int a, int b){
  return a + b;
}
 
public void functionA(){
// some code without function call
  int result = add(2,3); //call to function B
// some code without function call
}
```

下面给出`functionA()`在 JVM 中是怎样运行的：

![example of the state of a jvm method stack during after and before an inner call](https://mengde-pic-bed.oss-cn-hangzhou.aliyuncs.com/img/state_of_jvm_method_stack.jpg)

在`functionA()`中，`Frame A`是栈顶的帧，也就是当前帧。当内部调用`add()`方法时，新的帧(`Frame B`)入栈，`Frame B`成了当前帧，`Frame B`中的本地变量数组，是由`Frame A`中的操作数栈产生的。当`add()`方法执行完成之后，`Frame B`被销毁，`Frame A`重新成为当前帧，`add()`方法调用的结果被放在了`Frame A`的操作数栈的栈顶，`functionA()`就可以从它的操作数栈中取用`add()`方法的返回结果。

==注意：==栈的功能决定了它会动态的增强或压缩。栈的大小也会有个最大值，该值限制了递归调用的次数，如果栈的大小超过了限制值，JVM 会抛出 StackOverflowError。在 Hotspot 的中，该最大值可以用`-Xss`参数设置。

### 本地方法栈（线程独有）

本地方法栈是为本地代码服务的，所谓本地代码，就是非 Java 语言的，由 JNI(Java Native Interface) 调用的代码。本地栈的行为完全是由下层的操作系统决定的。

## 总结

希望本文可以帮你更好的理解 JVM。在我看来，最难理解的时虚拟机栈，因为它和 JVM 的内部功能紧紧相关。

如果你想深入学习，可以阅读 Java 虚拟机规范：[The Java® Virtual Machine Specification (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)

