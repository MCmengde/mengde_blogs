# C语言中变长数组的陷阱

本文为译文[原文链接]([The (too) many pitfalls of VLA in C | Jorengarenar](https://blog.joren.ga/vla-bad))

>  相比于定长数组，变长数组会产生额外的代码，使代码运行速度更慢，鲁棒性更差 ~ [Linus Torvalds](https://lkml.org/lkml/2018/3/7/621)

变长数组缩写为**VLA**(variable-length array)，它是一种在运行时才确定长度的数组（地址空间连续的数组，并不是表现得像数组的多段内存组成的数据结构），而非编译期。

```
以一种或多种方式提供VLAs支持的语言包括：Ada, Algol 68, APL, C, C#, COBOL, 	Fortran, J, Object Pascal。正如你所见，除了C和C#，其他的都不是现在主流的语言。
```

VLA在C99版本中出现。一开始，它们看起来似乎既方便又高效，然而这都是假象。事实上，他们往往是不断出现问题的根源。如果没有这个污点，C99本应该是一个很好的版本。

```
正如你在文章开头的引用中所见，Linux内核曾经是一个广泛使用VLA的项目。开发者付出了巨大的努力去摆脱VLA，终于在2018年的4.20版本中得尝所愿，去掉了全部的VLA。
```

## 在栈上分配空间

VLAs通常在栈上分配内存空间，而这正是大多数问题的根源所在。让我们来看一个简单的例子：

```c
#include <stdio.h>

int main(void) {
    int n;
    scanf("%d", &n);
    long double arr[n];
    printf("%Lf", arr[0]);
    return 0;
}
```

这里获取一个用户输入，作为数组的长度。试着把他跑起来，看看具体多大的数会使程序由于栈溢出导致的`segmentation fault`而报错。在我这里，最大可以到50万。这只是对于原始类型的数据，如果是结构体数组，这个上限会更小。又或者这个数组不是在`main()`里面，而是在递归调用中，上限会急剧减小。

然而，对于栈溢出，你并没有什么好的办法去补救，因为程序已经崩溃了。所以，你必须要在声明数组之前严格检查数组大小，或者你可以指望用户不要输入太大的数（这种赌博的结局显而易见）。

程序员必须保证变长数组的大小不会超过一个安全的最大值，但实际上，如果有人能知道这个安全的最大值的话，他没有任何理由不使用它（也就是说这个值是不可知的^译者注^）。

## 更糟糕的是

事实上，在VLA处理不当时，`segmentation fault`已经是最好的结果了。最坏的情况是，这是一个可利用的漏洞，攻击者可以选择一个合适的数组大小，利用数组覆盖其他的地址空间，以让他们控制这些地址空间。这简直就是安全噩梦。

```
以牺牲性能为代价，你可以在GCC中使用 -fstack-clash-protection 参数。该参数作用为，在变长栈空间分配前后添加额外的指令，以在分配时探测每一页内存。这样可以减轻“栈冲突”攻击的作用，该指令保证所有的栈内存分配都是有效的，如果存在无效的，就直接丢出 segementation fault异常，这样就把一个可能的代码攻击变成了拒绝服务。
```

## 改进之前的例子

如果确实需要用户输入数组大小，而又不想浪费空间去提前申请大数组，应该怎么做呢？使用`malloc()`！：

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int n;
    scanf("%d", &n);
    long double* arr = malloc(n * sizeof (*arr));
    printf("%Lf", arr[0]);
    free(arr);
    return 0;
}
```

在这个例子中，我可以最多可以输入13亿，不出现`segementation fault`的前提下，差不多比之前多了2500倍，然而还是会有一个导致

`segementation fault`的上限。不同的是，这里可以检查`malloc()`函数的返回值，以知晓地址空间分配是否成功。

```c
 	long double* arr = malloc(n * sizeof (*arr));
    if (arr == NULL) {
        perror("malloc()"); // 输出: "malloc(): Cannot allocate memory"
    }
```

```
有这样一个相反的观点，C语言通常被用作编写系统或者嵌入式系统，在这种情况下，可能用不了 malloc()。
我必须要在这里重复一遍我的看法，因为这真的很重要。
在这些设备上，你所拥有的栈空间也不会很多。所以，你应该确定你需要多少空间，然后使用定长数组，而不是在栈上动态的分配空间。
当在栈很小的系统上使用动态数组时，很容易出现，虽然看起来一切正常，但是由于较深的函数调用、大量的数据分配而造成栈崩溃的情况。
如果你总是分配一个固定大小的栈空间，测试时就不会出现这些问题。如果你在栈上动态分配空间，你需要用最大的分配空间去测试你的代码，这是非常难的，而且很容易出错。不要做对自己没有好处的事。

```

## 在意料之外产生

不同于其他危险的C语言功能，VLA是广为人知的。许多新手通过反复试错学会使用VLA，但是并不了解陷阱。有时候即使是经验丰富的程序员也会在不经意间使用VLA。以下代码就会悄无声息的产生一个不必要的VLA:

```c
const int n = 10;
int A[n];
```

值得庆幸的是，编译器会察觉并优化这样的VLA，但是万一没有察觉到呢？又或者基于其他的考虑（比如安全）没有优化呢？大概不会有更糟的情况了吧？

## 比定长更慢

没有编译器优化的情况下，在传入数组之前，[使用 VLA 的代码](https://godbolt.org/z/c7nPvGGcP)的汇编指令数是[使用定长数组的代码](https://godbolt.org/z/jx94vx84T)的7倍。实际上，优化之后，情况也是一样的。见[下例](https://godbolt.org/z/vnf174eej)：

```c
#include <stdio.h>
void bar(int*, int);

#if 1 // 1 for VLA, 0 for VLA-free

void foo(int n) {
    int A[n];
    for (int i = n; i--;) {
        scanf("%d", &A[i]);
    }
    bar(A, n);
}

#else

void foo(int n) {
    int A[1000];  // Let's make it bigger than 10! (or there won't be what to examine)
    for (int i = n; i--;) {
        scanf("%d", &A[i]);
    }
    bar(A, n);
}

#endif

int main(void) {
    foo(10);
    return 0;
}

void bar(int* B, int n) {
    for (int i = n; i--;) {
        printf("%d %d", i, B[i]);
    }
}
```

为了更好的说明情况，`-01`级别的优化更合适(汇编会更清楚，另外`-02`级别的优化对 VLA 的优化并不明显)

编译VLA的版本后，在`for`循环对应的指令之前，我们可以看到：

```asm
push    rbp
mov     rbp, rsp
push    r14
push    r13
push    r12
push    rbx
mov     r13d, edi
movsx   r12, edi       ; "VLA"在这里开始
sal     r12, 2         ;
lea     rax, [r12+15]  ;
and     rax, -16       ;
sub     rsp, rax       ;
mov     r14, rsp       ; 这里结束
```

而非VLA的版本是这样的：

```asm
push    r12
push    rbp
push    rbx
sub     rsp, 4000      ; 这里是数组的定义
mov     r12d, edi
```

可见，定长数组的代码更简短。为什么使用VLA会造成这么多的函数头部开销呢？我们也许不必考虑所有的事情，但这绝不仅仅是指针碰撞。

这些区别必然是值得关心的。

## 不允许初始化

为了减少无意间使用VLA时的麻烦，以下操作是不允许的：

```c
int n = 10;
int A[n] = { 0 };
```

即使有编译器的优化，初始化VLAs也是不允许的。所以尽管我们希望编译器能在技术上提供一个定长的数组，这种操作也是不允许的。

## 编译器作者的麻烦事

几个月前，我保存了Reddit上的一个评论，是关于编译器作者如何看待VLA带来的问题的。在此引用：

> - A VLA applies to a type, not an actual array. So you can create a `typedef` of a VLA type, which "freezes" the value of the expression used, even if elements of that expression change at the time the VLA type is applied
> - VLAs can occur inside blocks, and inside loops. This means allocating and deallocating variable-sized data on the stack, and either screwing up all the offsets, or needing to do things indirectly via pointers.
> - You can use `goto` into and out of blocks with active VLAs, with some things restricted and some not, but the compiler needs to keep track of the mess.
> - VLAs can be used with multi-dimensional arrays.
> - VLAs can be used as pointer targets (so no allocation is done, but it still needs to keep track of the variable size).
> - Some compilers allow VLAs inside structure definitions (I really have no idea how that works, or at what point the VLA size is frozen, so that all instances have the same VLA(s) sizes.)
> - A function can have dozens of VLAs active at any one time, with some being created or destroyed at different times, or conditionally, or in loops.
> - `sizeof` needs to be specially implemented for VLAs, and all the necessary info (for actual VLAs, VLA-types, and hybrid VLA/fixed-size types and arrays and pointed-to VLAs).
> - 'VLA' is also the term used for multi-dimensional array parameters, where the dimensions are passed by other parameters.
> - On Windows, with some compilers (GCC at least), declaring local arrays which make the stack frame size over 4 KiB, mean calling a special allocator (`__chkstk()`), as the stack can only grow a page at a time. When a VLA is declared, since the compiler doesn't know the size, it needs to call `__chkstk` for every such function, even if the size turns out to be small.

你在浏览其他的C语言论坛时，肯定也见过更多不同的抱怨。

## 减少支持

由于上文提到的这些问题，一些编译器提供者决定不完全支持 C99，一开始是微软的 MSVC。C语言标准协会也注意到这个问题，并且在 C11 版本中，VLAs 是可选的（大多数都选择弃用）。

这就意味着，使用 VLA 的代码不一定可以用 C11 的编译器编译，所以使用时需要检查编译器是否支持`_SRDC_NO_VLA_`宏，并且编写不适用 VLA 的版本作为备用。既然需要写不使用 VLA 的版本，那为什么还要写使用 VLA 的版本呢？

```
值得一提，C++没有 VLA，也没有迹象表明以后会支持。C++并非破坏者，却任然反对C语言中的 VLA 。
```

## （挑剔的理由）打破了惯例

也许显得苛刻，却也是一个不喜欢 VLA 的理由。以下为广泛使用的传入二维数组的传参方式，我们习惯于先传入数组：

```c
void foo(int** arr, int n, int m) { /* arr[i][j] = ... */ }
```

C99 中，当函数参数列表中有数组时，数组大小会被立即解析。也就意味着，使用 VLA ，就不能使用下面一种传参方式了

```
void foo(int arr[n][m], int n, int m) { /* arr[i][j] = ... */ } // INVALID!
```

你只能选择下面的方式：

- 打破常规：

    ```c
    void foo(int n, int m, int arr[n][m]) { /* arr[i][j] = ... */ }
    ```

- 使用过时的语法

    ```c
    void foo(int[*][*], int, int);
    void foo(arr, n, n)
        int n;
        int m;
        int arr[n][m]
    {
        // arr[i][j] = ...
    }
    ```

## 某些情况下还有点用

有一种需要使用 VLA 的情景：动态分配多维数组，数组的内层维度要到运行时才知道。这里甚至没有安全问题，因为没有随意分配栈空间。

```c
int (* A)[m] = malloc(n * sizeof (*A)); // m 和 n 是数组维度
if (A) {
    // A[i][j] = ...;
    free(A);
}
```

不使用 VLA，可以有以下替代方式：

- 一行一行使用`malloc()`申请：

    ```c
    int** A = malloc(n * sizeof (*A));
    if (A) {
        for (int i = 0; i < m; ++i) {
            A[i] = malloc(m * sizeof (*A[i]));
        }
        // A[i][j] = ...
        for (int i = 0; i < m; ++i) {
            free(A[i]);
        }
        free(A);
    }
    ```

- 一维数组加上偏置：

    ```c
    int* A = malloc(n * m * sizeof (*A));
    if (A) {
        // A[i*n + j] = ...
        free(A);
    }
    ```

- 使用大的定长数组：

    ```c
    int A[SAFE_SIZE][SAFE_SIZE]; // SAFE_SIZE must be safe for SAFE_SIZE*SAFE_SIZE
    // A[i][j] = ...;
    ```

## 总结

简而言之，避免使用 VLA。它带来了危险，却没有带来任何好处。如果你真的想使用的话，请牢记它的限制。

```
值得一提的是，VLA 是问题更多的`alloca()`的解决方案（并非标准)。
```

