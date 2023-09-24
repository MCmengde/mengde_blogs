# Java中只有值传递

## 参数传递

在我们日常编写代码的过程中，调用函数可能是最常见的操作了。那么，在调用函数时，参数是怎么样传递的呢？

### 值传递

相信有很多人都是学C语言入门的，刚开始写代码时，用的最多的就是值传递了。

```c
void plus_one(int a){
    a++;
    printf("a: %d", a);
}

int main(){
    int n = 10;
    plus_one(n);
    printf("n:%d", n);
    return 0;
}
```

这是一个简单的值传递的例子，无需多言，`plus_one`函数的作用就是将传进来的数加一，然后输出。所谓值传递，就是直接将实参`n`的值赋给形参`a`，赋值完成之后，两者再无瓜葛。

因此，上面的代码可以等效为：

```c
int main(){
    int n = 10;

    // plus_one start
    int a;
    a = n;
    a++;
    printf("a: %d", a);
    // plus_one end

    printf("n:%d", n);
    return 0;
}
```

可以看到，值传递简单直观，然而，调用函数并不能改变实参`n`的值。

### 指针传递

那么，当我们需要改变实参的值的时候，我们就会想到使用指针传递，也就是所谓的地址传递。

```c
void plus_one(int* p){
    *p = *p + 1;
}

int main(){
    int n = 10;
    plus_one(&n);
    printf("The result is %d", n);
    return 0;
}
```

这里，我们将实参`n`的地址传入`plus_one`函数，在函数中，直接对指针`p`所指向的值，也就是`n`做操作，自然就可以改变实参`n`的值了。

实际上，指针传递也是值传递。我们将上面的代码改写：

```c
int main(){
    int n = 10;

    // plus_one start
    int* p;
    p = &n;
    *p = *p + 1;
    printf("The result is %d", n);
    // plus_one end

    return 0;
}
```

可以看到，所谓的指针传递，也只不过是将变量`n`的地址值赋给指针变量`p`，实际上也是值传递。

所以，可以不负责任的概括为，C语言中只有值传递；

### 引用传递

指针固然强大，但是由于代码不易读，难以理解等问题，也是广为诟病。C++作为C语言的超大杯，引入了引用传递来简化指针传递的写法。

```c++
void plus_one(int& a){
    a++;
}

int main(){
    int n;
    plus_one(n);
    printf("The result is %d", n);
    return 0;
}
```

C++中，对&运算符进行了重载，实现了引用传递。具体实现为，在调用`plus_one`函数时，在函数调用栈中存变量`n`的地址，而不是n的值。因此，`plus_one`中的变量`a`就相当于是`n`的"别名"，对`a`操作时，自然会改变`n`的值。

可见，引用传递的底层也是赋值操作。

## Java中的参数传递

那么，在Java中，究竟是引用传递，还是值传递呢？

Java中变量分为基本变量和对象，我们不妨分别讨论。

### 基本变量类型

首先，对于`int`、`char`等基本类型，Java是使用值传递的，很容易验证。

```java
static void plusOne(int a){
    a++;
    System.out.println("a: " + a);
}

public static void main(String[] args){
    int n = 10;
    plusOne(n);
    System.out.println("n: " + n);
}
```

显然，与C语言中一样，这里`n`的值是不会改变的。

### 对象

```java
public class PassObject {
    public static void main(String[] args) {
        Dog myDog = new Dog("Test");
        foo(myDog);
        System.out.println(myDog.getName());// TestPlus
    }

    public static void foo(Dog dog) {
        dog.setName("TestPlus");
    }
}

class Dog{

    private String name;

    public Dog(String name) {
        this.name = name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }
}
```

通过上面的例子可以看到，传入对象的引用时，是可以改变对象的属性变量的。那么Java在传递对象作为参数时，是引用传递吗？

实际上并非如此，Java中，对象的引用，实际上相当于对象的指针。在Java中操作对象，只有通过引用操作这一种途径。某种意义上，Java中是不能直接操作对象的。

也就是说，在上例中传参时，没有对`myDog`对象实例做任何操作，只是把`myDog`引用值赋给了`foo`函数中的本地变量`dog`。并没有像引用传递一样，传入对象实体，但是只在栈中保存对象引用的操作。所以，Java中传递对象时，也是值传递。

所以，**Java中只有值传递**。

## 值得一提

然而，还是会有一些特殊情况，会让人怀疑上述结论。

### 数组

上面只分析了基本变量类型和对象，数组呢？

实际上，Java中的数组也是一种对象，数组类也是继承自Object类。在将数组作为参数时，也是传递的数组的引用，并没有传递数组的实体。

```java
public static void changeContent(int[] arr) {
    arr[0] = 10;
}

public static void changeRef(int[] arr) {
    arr = new int[2];
    arr[0] = 15;
}

public static void main(String[] args) {
    int [] arr = new int[2];
    arr[0] = 4;
    arr[1] = 5;

    changeContent(arr);
    System.out.println(arr[0]); // 10
    changeRef(arr);
    System.out.println(arr[0]); // 10
}
```

在上例中可以看到，将传入的数组引用赋给一个新的数组后，这个引用就不能操作之前的数组了。

关于**引用**，英文是`reference`，实际上，我自认为，翻译为**句柄**是更为贴切的，引用就像是一个**柄**，一个`Handler`，你可以用它操作实体，但他并不是实体本身。就像手柄可以操控游戏机，但不是游戏机本身，当你将这个手柄连接到另一个游戏机的时候， 它就不能操控之前的游戏机了。

### 包装类和String

```java
public static void main(String[] args) {
    Integer n = 1;
    plusOne(n);
    System.out.println(n); // 1
}

private static void plusOne(Integer n) {
    n = n + 1;
    System.out.println(n);// 2
}
```

在这段代码中，`n`作为`Integer`类型实例的句柄，却并没有成功改变对象的值，这是为什么呢？

在`Integer`类中，存对应值的属性是`value`，其声明如下:

```java
private final int value;
```

可见，value值是不能改的，那加的操作是怎么实现的呢？

在上述加一的过程中，会重新`new`一个`Integer`对象，让后将这个对象赋给引用`n`。这样以来，之前的对象自然是不会改变的。

实际上，包装类以及`String`类的值，都是`final`的，所以在执行`+`的过程中，都会重新生成一个对象，然后对它赋值。
