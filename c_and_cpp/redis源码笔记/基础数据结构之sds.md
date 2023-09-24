# [Redis源码笔记] 基础数据结构之sds

Redis源码笔记系列将主要记录总结Redis源码相关的，C语言生僻知识点以及优秀的编程设计。源码版本为Redis 3.0，主要参考黄健宏老师的《Redis设计与实现》，这里给出[仓库链接](https://github.com/huangz1990/redis-3.0-annotated)。

本篇讨论Redis重要的基础数据结构sds(Simple Dynamic String)实现相关的知识点，sds类型的定义如下：

```c
typedef char *sds;
```

可见sds就是字符指针，该定义使sds可以直接使用C语言中字符串相关的接口。

## 关于 flexible array member

Redis中定义了sdshdr结构来保存字符串的长度信息，其定义如下：

```c
struct sdshdr {
    
    // buf 中已占用空间的长度
    int len;

    // buf 中剩余可用空间的长度
    int free;

    // 数据空间
    char buf[];
};
```

其定义简介明了，无需多言，但其相关的，获取sds长度的方法`sdslen()`值得讨论，其实现如下：

```c
static inline size_t sdslen(const sds s) {
    struct sdshdr *sh = (void*)(s-(sizeof(struct sdshdr)));
    return sh->len;
}
```

该方法入参是sds类型实例，返回值的是其关联的sdshdr类型实例的len字段。显然，该方法的实参应该是一个sdshdr实例的buf字段，根据入参s求偏移，就可以找到len字段的位置。该sdshdr实例内存分布如下图（假定int为4字节）：

![image-20230917123345390](https://mengde-pic-bed.oss-cn-hangzhou.aliyuncs.com/img/image-20230917123345390.png)

可见用s减去两个int的长度，就可以求得该sdshdr实例的首地址。但`sdslen()`的实现中却直接用s减掉了`sizeof(struct sdshdr)`，然而`struct sdshdr`中除了前两个int类型的字段，还有第三个`char buf[]`，那么为什么`sizeof(struct sdshdr)`会等于前两个int类型字段的长度呢？

在C99标准中，对类似`char buf[]`这种结构的描述如下：

> As a special case, the last element of a structure with more than one named member may have an incomplete array type; this is called a flexible array member. In most situations, the flexible array member is ignored. In particular, the size of the structure is as if the flexible array member were omitted except that it may have more trailing padding than the omission would imply.

也就是说，在求结构体大小的时候，这种声明为**结构体最后一个元素的、长度未指定的**的flexible array member是被忽略的。同样，flexible array member的地址空间也需要显示分配，`sdsnewlen()`方法中展示了这种结构体通常的初始化方法为：

```c
struct sdshdr *sh = malloc(sizeof(struct sdshdr) + initlen + 1)；
```

其中`initlen`即为buf数组申请的空间(最后的1放置`'\0'`)，申请成功之后即可使用`->`操作符按下标正常索引`buf`，未申请空间之前对其索引是未定义行为。

关于flexible array member，还有一点不容忽视，就是结构体对齐对它的影响，不妨看下面一个例子：

```c
#include<stdarg.h>
#include<stdio.h>
#include <stddef.h>

struct sdshdr {
    int len;
    short free;
    char buf[];
};

int main() {
    struct sdshdr sh;
    sh.buf[1] = 'c';
    printf("%c\n", sh.buf[1]);
    printf("%d\n", offsetof(struct sdshdr, buf));
    printf("%d\n", sizeof(struct sdshdr));
    return 0;
}
```

这里，我们将sdshdr结构体的free字段改为short类型，该程序输出结果为：

```shell
c
6
8
```

可见，结构体字段对其之后，整个结构体的大小为8字节，但前两个整数字段只占了6个字节，而flexible array member的首地址直接从第6个字节开始，这种情况下，直接在栈上声明一个sdshdr实例，其最后两个补全字节是可以为flexible array member所用的。

## sds的binary safe

二进制安全是sds的关键特性之一，与C语言原生字符串相关函数中，默认以`'\0'`空字符作为字符串的结尾相比，sds可以在buf的中间位置存储`\0`空字符，sds相关的方法可以有效处理`'\0'`空字符；sds不以`'\0'`作为结束标志，而是在sdshdr用len字段记录当前buf数组已使用的空间，即当前字符串的长度。然而，buf数组任然以`'\0'`，以兼容原生字符串相关的接口。

## 关于可变长度参数表

sds有一个格式化追加的方法`sdscatprintf`，其函数头如下：

```c
sds sdscatprintf(sds s, const char *fmt, ...)；
```

其入参声明为可变长度，关于变长参数表的函数，有以下例子可以参考：

```c
#include<stdarg.h>
#include<stdio.h>
#include <stddef.h>
#include <stdarg.h>

int sum(int args_num, ...) {
    va_list ap;
    va_start(ap, args_num);

    int sum = 0;
    int i;
    for (i = 0; i < args_num; ++i) {
        sum += va_arg(ap, int);
    }
    va_end(ap);

    return sum;
}

int main() {
    int s = sum(4, 1, 2, 3, 4);
    printf("%d", s);
    return 0;
}
```

其中sum函数可以用来求任意个数的整形变量的和，但仍需在第一个参数指明其个数。

变长参数表相关的宏函数定义在stdarg.h中，需要用到的宏函数有：

- va_list
- va_start
- va_arg
- va_end
- va_copy，使用va_arg取出参数之后，va_list会变化，若有重复使用va_list的需求，可以先使用va_copy复制一份

## \_\_attribute\_\_宏的使用

另外，`sdscatprintf`函数的定义如下：

```c
#ifdef __GNUC__
sds sdscatprintf(sds s, const char *fmt, ...)
    __attribute__((format(printf, 2, 3)));
#else
sds sdscatprintf(sds s, const char *fmt, ...);
#endif
```

`__attribute__`宏是GNU C语言中的关键字，这里用来说明`sdscatprintf`的第2个和第三个参数的格式与`printf`函数一致，编译器在编译器会对其调用进行检查。

C语言中该关键字的用法有以下几种：

- [函数属性](https://gcc.gnu.org/onlinedocs/gcc-4.9.4/gcc/Function-Attributes.html#Function-Attributes)
    - `__attribute__((format(...))`，规定函数参数
    - `__attribute__((noreturn))`，说明函数永远不会返回
    - `__attribute__((const))`，说明函数为纯函数
- [变量属性](https://gcc.gnu.org/onlinedocs/gcc-4.9.4/gcc/Variable-Attributes.html#Variable-Attributes)
    - `__attribute__((unused))`，说明变量可能未使用
    - `__attribute__(aligned(n))`，指明变量的对齐长度，n需要是2的幂次
- [类型属性](https://gcc.gnu.org/onlinedocs/gcc-4.9.4/gcc/Type-Attributes.html#Type-Attributes)