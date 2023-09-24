# C语言中字符串的初始化

字符串是最常用的数据类型之一。

C语言中，是没有`String`类型来存储字符串的，字符串被看做是一组**连续**的`char`类型。

所以，字符串有两种表示方法，分别是**字符数组**和**字符指针**，而这两种表示的初始化却又不尽相同。

为了方便比较结果，定义全局变量`LENGTH`为`15`，定义输出函数`print`如下：

```c
/** Display the outputs.
 * args: chars[], The char array to print.
 *       length, The size of the char array you wanna print.
 *       type, Tht format you wanna print those chars.
 *              16: hexadecimal
 *               0: chars
 */
void print(char* chars, int length, int type){
    // printf("%ld:", sizeof(chars));
    if (type == 16)
    {
        for (int i = 0; i < length; i++)
        {   
            printf("0x%x ", *(chars + i));
        }
    } else if (type == 0)
    {
        for (int i = 0; i < length; i++)
        {   
            printf("%c", *(chars + i));
        }
    }
    
    printf("\n");
}
```

其中`type`为`0`时直接输出字符串，为`16`时，以16进制输出。

## 字符数组

### 以字符串赋值

```c
char array_1[LENGTH] = "array_1";
print(array_1, LENGTH, 16);
printf("%ld\n", sizeof(array_1));
```

先来看结果

```shell
0x61 0x72 0x72 0x61 0x79 0x5f 0x31 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 
15
```

这种情况下，字符串以`0`结尾，并且没有赋值的部分，也都已经初始化为`char`类型的`0`。

### 用`strcpy`赋值

```c
char array_2[LENGTH];
strcpy(array_2, "array_2");
print(array_2, LENGTH, 16);
printf("%ld\n", sizeof(array_1));
```

结果为

```shell
0x61 0x72 0x72 0x61 0x79 0x5f 0x32 0x0 0xfffffff0 0x6c 0x7f 0x0 0x0 0xffffffc0 0xfffffff3
15
```

可见，使用`strcpy`函数赋值，字符串到`\0`结束，之后的数据，**类型**和**值**都是随机的。

和方法一对比可知，字符数组在声明时，是没有初始化为全零的。

### 以字符数组赋值

在C语言中，一个字符串结束的标志位是`\0`， 那么在用于初始化的字符数组最后一位要不要写`\0`呢？

不加`\0`的情况：

```c
char array_3[LENGTH] = {'a', 'r', 'r', 'a', 'y', '_', '3'};
print(array_3, LENGTH, 16);
printf("%ld\n", sizeof(array_1));
```

结果为

```
0x61 0x72 0x72 0x61 0x79 0x5f 0x33 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0
15
```

不加`\0`的情况

```c
char array_4[LENGTH] = {'a', 'r', 'r', 'a', 'y', '_', '4', '\0'};
print(array_4, LENGTH, 16);
printf("%ld\n", sizeof(array_1));
```

结果为

```
0x61 0x72 0x72 0x61 0x79 0x5f 0x34 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0
15
```

可见，这两种方法并没有什么区别。直接赋值时，数组会按声明的长度，把每个位置都初始化为`\0`。

### 值得一提

- 声明为数组的变量，是不能先申明，再用常量赋值的。以下两种写法都**不能通过编译**。

```c
char array_5[LENGTH];
array_5 = "test";
array_5 = {'t', 'e', 's', 't'};
```

- 另外，用`sizeof`实际上是求的指针变量的大小，也就是数组声明的长度。但是用`strlen`函数可以求出字符串的有效长度，也就是到`\0`的长度，**可自行验证**。

## 字符指针

在将变量声明为指针时，只有两种赋值方法。

### 以字符串直接赋值

```c
char* pointer_1 = "pointer_1";
print(pointer_1, LENGTH, 16);
printf("%ld\n", sizeof(pointer_1));
```

结果为

```shell
0x70 0x6f 0x69 0x6e 0x74 0x65 0x72 0x5f 0x31 0x0 0x70 0x6f 0x69 0x6e 0x74
8
```

值得注意的是，此时，`sizeof`便可求出字符串的**有效长度**了。

### 以字符串间接赋值

```c
char* pointer_5;
pointer_5 = "pointer_5";
print(pointer_5, LENGTH, 16);
printf("%ld\n", sizeof(pointer_1));
```

结果为

```shell
0x70 0x6f 0x69 0x6e 0x74 0x65 0x72 0x5f 0x35 0x0 0x0 0x1 0x1b 0x3 0x3b
8
```

### 其他方法

- 在使用**`strcpy`函数赋值**时，编译器会提示指针变量未初始化。

    可见以**数组声明**时，虽然数组中的值没有初始化为0，但是地址空间已经得到了。

    而用**指针声明**时，没有指明长度，自然没办法申请空间。

- 至于用**字符数组赋值**的方法，由于现在变量是声明为字符指针的，类型不同，自然是行不通的。

## 完整测试代码

```c
#include<stdlib.h>
#include<string.h>
#include<stdio.h>

#define LENGTH 15


/** Display the outputs.
 * args: chars[], The char array to print.
 *       length, The size of the char array you wanna print.
 *       type, Tht format you wanna print those chars.
 *              16: hexadecimal
 *               0: chars
 */
void print(char* chars, int length, int type){
    // printf("%ld:", sizeof(chars));
    if (type == 16)
    {
        for (int i = 0; i < length; i++)
        {   
            printf("0x%x ", *(chars + i));
        }
    } else if (type == 0)
    {
        for (int i = 0; i < length; i++)
        {   
            printf("%c", *(chars + i));
        }
    }
    
    printf("\n");
}


int main()
{
    // Define as array;
    char array_1[LENGTH] = "array_1";
    print(array_1, LENGTH, 16);
    printf("%ld\n", sizeof(array_1));

    char array_2[LENGTH];
    strcpy(array_2, "array_2");
    print(array_2, LENGTH, 16);
    printf("%ld\n", sizeof(array_1));


    char array_3[LENGTH] = {'a', 'r', 'r', 'a', 'y', '_', '3'};
    print(array_3, LENGTH, 16);
    printf("%ld\n", sizeof(array_1));

    char array_4[LENGTH] = {'a', 'r', 'r', 'a', 'y', '_', '4', '\0'};
    print(array_4, LENGTH, 16);
    printf("%ld\n", sizeof(array_1));


    // char array_5[LENGTH];
    // array_5 = "test";

    // Define as pointer.
    char* pointer_1 = "pointer_1";
    print(pointer_1, LENGTH, 16);
    printf("%ld\n", sizeof(pointer_1));

    // char* pointer_2;
    // strcpy(pointer_2, "pointer_2");
    // print(pointer_2, LENGTH, 16);

    // char* pointer_3 = {'p', 'o', 'i', 'n', 't', 'e', 'r', '_', '3'}; 
    // print(pointer_3, LENGTH, 16);

    // char* pointer_4 = {'p', 'o', 'i', 'n', 't', 'e', 'r', '_', '3', '\0'};
    // print(pointer_4, LENGTH, 16);

    char* pointer_5;
    pointer_5 = "pointer_5";
    print(pointer_5, LENGTH, 16);
    printf("%ld\n", sizeof(pointer_1));

    return 0;
}

```

