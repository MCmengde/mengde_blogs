# CSAPP实验笔记一：data lab

本实验中涉及的知识点均包含于原书第二章

#### 1.使用`~`和`&`表示`^`

题目要求：

```
/* 
 * bitXor - x^y using only ~ and & 
 *   Example: bitXor(4, 5) = 1
 *   Legal ops: ~ &
 *   Max ops: 14
 *   Rating: 1
 */
```

即使用**非**和**与**运算符表示**异或**运算符，下面给出异或的真值表：

|  A   |  B   | A^B  |
| :--: | :--: | :--: |
|  0   |  0   |  0   |
|  1   |  0   |  1   |
|  0   |  1   |  1   |
|  1   |  1   |  0   |

**异或**可以理解为**不进位的加法**，**异或**和**或**仅仅是在 A 和 B 都为真的时候不同，所以可以将**异或**表示为：

```latex
L = a ^ b = (a | b) & (~a | ~b)
```

根据反演规则。可以推得：

```
L = ~(~a & ~b) & ~(a & b)
```

即用`~`和`&`成功表示了`^`，故完整代码为：

```c
int bitXor(int x, int y) {
  return ~(~x & ~y) & ~(x & y);
}
```

#### 2.求最小的整数的二进制补码

题目要求：

```
/* 
 * tmin - return minimum two's complement integer 
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 4
 *   Rating: 1
 */
```

对于32位的整型，最小值的补码是符号位为1，其余位皆为0，即`0x8000_0000`，故答案为：

```c
int tmin(void) {
  return 1 << 31;
}
```

#### 3.判断给定的参数是否是最大的二进制补码数

题目要求：

```
/*
 * isTmax - returns 1 if x is the maximum, two's complement number,
 *     and 0 otherwise 
 *   Legal ops: ! ~ & ^ | +
 *   Max ops: 10
 *   Rating: 1
 */
```

为了验证传入的参数是否是该数，我们需要利用该数一些特有的性质。

倘若将最大的二进制补码数加一，会得到最小的二进制补码数，例如，对于 4 bit 位的整数有：

最大的正数为`0b0111`，也就是整数`7`，对它加一我们会得到`0x1000`，也就是`-8`的补码表示，且很容易看出，**`-8`的补码是`7`**

**的补码取反的结果**。

利用这一点，我们可以写出以下逻辑表达式来判断传入的数是否为最大整数：

```c
!(~((x + 1) ^ x))
```

由内而外，首先将`x + 1`与`x`做异或，由以上所述，若`x`为最大整数，则异或的结果应该为`-1`，其二进制补码表示为全`1`，即`0xFFFFFFFF`，然后将该值取反可得`0`，即逻辑值**非**，然后再做非运算即可。

然而，还有一个特殊值`-1`，`x`为`-1`时，上述表达式也为真，这里将`-1`排除即可。

最终结果为：

```c
int isTmax(int x) {
  return !(~((x + 1) ^ x)) & !!(x + 1);
}
```

值得一提的是，这里使用`!!`将数值转化为逻辑表达式。

#### 4.判断二进制表示奇数位是否全为`1`

题目要求：

```
/* 
 * allOddBits - return 1 if all odd-numbered bits in word set to 1
 *   where bits are numbered from 0 (least significant) to 31 (most significant)
 *   Examples allOddBits(0xFFFFFFFD) = 0, allOddBits(0xAAAAAAAA) = 1
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 12
 *   Rating: 2
 */
```

为了判断所给数的奇数位是否都为`1`，需要将其奇数位先取出来，将该数与`0xAAAAAAAA`相**与**即可；若该数奇数位都为`1`，则相与的结果必和`0xAAAAAAAA`相等，可以使用**异或**运算判断两数是否相等。

结果为：

```c
int allOddBits(int x) {
  int a = 0xAA;
  int b = (0xAA << 8) | a;    // 0xAAAA
  int c = (0xAA << 16) | b;   // 0xAAAAAA
  int d = (0xAA << 24) | c;   // 0xAAAAAAAA
  return ! ((x & d) ^ d);
}
```

#### 5.求相反数

题目要求：

```
/* 
 * negate - return -x 
 *   Example: negate(1) = -1.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 5
 *   Rating: 2
 */
```

首先给出：

```
A + ~A = -1
```

即任意数与其取反结果相加，得`-1`，`-1`的二进制补码表示始终为全`1`，例如四位二进制表示中`-1`表示为`0x1111`。故有：

```
neg(A) = ~A + 1
```

最终结果为：

```c
int negate(int x) {
    return ~x + 1;
}
```

#### 6.判断是否为ASCII码中的数字

题目要求：

```
/* 
 * isAsciiDigit - return 1 if 0x30 <= x <= 0x39 (ASCII codes for characters '0' to '9')
 *   Example: isAsciiDigit(0x35) = 1.
 *            isAsciiDigit(0x3a) = 0.
 *            isAsciiDigit(0x05) = 0.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 15
 *   Rating: 3
 */
```

首先判断该数`5~8`位是否表示`0x3`，即将这几位取出来，然后判断是否等于`0x3`，可以写为

```
!((x >> 4) ^ 0x3)
```

然后判断末四位是否在所给区间中，只需要将末尾四位取出来，然后减去`0xA`，判断结果正负即可，判断正负时，检查符号位。

最终结果为：

```c
int isAsciiDigit(int x) {
    int a = !((x >> 4) ^ 0x3);
    int b = x & 0xF;
    int c = !!((b + (~0xA + 1)) & (0x80 << 8));
    return a & c;
}
```

#### 7.实现三元运算符

题目要求：

```
/* 
 * conditional - same as x ? y : z 
 *   Example: conditional(2,4,5) = 4
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 16
 *   Rating: 3
 */
```

可以这样理解三元运算符，即返回结果为 $ay + bz$，其中系数 $a$ 和 $b$ 由 `x` 的真假决定，有以下真值表：

|  x   |  a   |  b   |
| :--: | :--: | :--: |
|  0   |  0   |  1   |
|  1   |  1   |  0   |

这里没办法用乘法，所以我们用加法重新表示返回结果为 $y + z + a + b$，则：

|  x   |  a   |  b   |
| :--: | :--: | :--: |
|  0   | $-y$ |  0   |
|  1   |  0   | $-z$ |

最终结果为：

```c
int conditional(int x, int y, int z) {
    int a = !!(x ^ 0x0);
    // x == 0  ==>  b == 0;
    // x != 0  == > b == -1;
    int b = ~a + 1;
    int c = ~(y & ~b) + 1;
    int d = ~(z & b) + 1;
    return y + z + c + d;
}
```

值得注意的是：`0`取反为`-1`，反之亦然。

#### 8.实现小于等于判断

题目要求：

```
/* 
 * isLessOrEqual - if x <= y  then return 1, else return 0 
 *   Example: isLessOrEqual(4,5) = 1.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 24
 *   Rating: 3
 */
```

对于通常的比较大小而言，只需要求`y - x`，判断结果正负即可，但是要注意排除溢出的情况。只有当`x`与`y`异号时，才可能发生溢出，这里分情况讨论。

若`x`为正，`y`为负，则以下表达式为真，需要进一步判断求差的结果：

```c
~((x >> 31) & 0x1) & ((y >> 31) & 0x1)
```

以上表达式为真，且`y - x`符号位为`0`时，发生溢出。

若`x`为负，`y`为正，则以下表达式为真，此时直接返回真即可：

```c
((x >> 31) & 0x1) & ~((y >> 31) & 0x1)
```

最终结果为：

```c
int isLessOrEqual(int x, int y) {
    int a = ~((x >> 31) & 0x1) & ((y >> 31) & 0x1);
    int b = ((x >> 31) & 0x1) & ~((y >> 31) & 0x1);
    // 计算 y - x
    int rst y + (~x + 1);
    int flag = rst >> 31;
    
    return b | (!a & !flag);
}
```

#### 9.实现逻辑取反

题目要求：

```
/* 
 * logicalNeg - implement the ! operator, using all of 
 *              the legal operators except !
 *   Examples: logicalNeg(3) = 0, logicalNeg(0) = 1
 *   Legal ops: ~ & ^ | + << >>
 *   Max ops: 12
 *   Rating: 4 
 */
```

由题意，即`x == 0`时，返回`1`；为其他值时，返回`0`。可比较`x`和`-x`的符号位。

最终答案为：

```c
int logicalNeg(int x) {
    return ((x | (~x + 1)) >> 31) + 1;
}
```

值得注意：`>>`为算数右移，`x`不等于零时，`((x | (~x + 1)) >> 31)`的值为`-1`。

#### 10.求二进制补码表示一个整数最少需要几位

题目要求：

```
/* howManyBits - return the minimum number of bits required to represent x in
 *             two's complement
 *  Examples: howManyBits(12) = 5
 *            howManyBits(298) = 10
 *            howManyBits(-5) = 4
 *            howManyBits(0)  = 1
 *            howManyBits(-1) = 1
 *            howManyBits(0x80000000) = 32
 *  Legal ops: ! ~ & ^ | + << >>
 *  Max ops: 90
 *  Rating: 4
 */
```

采用**二分查找**，对于正数而言，找到最左边的`1`；对于负数而言，找到最左边的`0`。

具体代码如下：

```c
int howManyBits(int x) {
    int b16, b8, b4, b2, b1;
    
    // 符号位
    int flag = x >> 31;
    
    // 正数不变，负数取反
    x = (flag & ~x) | (~flag & x);
    
    // 判断高16位是否为零，不为零时取高16位，继续判断
    b16 = (!!(x >> 16)) << 4;
    x >>= b16;
    
    b8 = (!!(x >> 8)) << 3;
    x >>= b8;
    
    b4 = (!!(x >> 4)) << 2;
    x >>= b4;
    
    b2 = (!!(x >> 2)) << 1;
    x >>= b2;
    
    b1 = (!!(x >> 1));
    x >>= b1;
    
    b0 = x;
    // 最后加上符号位
    return b0 + b1 + b2 + b4 + b8 + b16 + 1;
}
```

#### 11.浮点数乘以2

题目要求：

```
/* 
 * floatScale2 - Return bit-level equivalent of expression 2*f for
 *   floating point argument f.
 *   Both the argument and result are passed as unsigned int's, but
 *   they are to be interpreted as the bit-level representation of
 *   single-precision floating point values.
 *   When argument is NaN, return argument
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 30
 *   Rating: 4
 */
```

分情况讨论

对于`NaN`和无限大，直接返回输入值即可。

对于规格化数，只需要将阶码加一即可。

对于非规格化数，将尾码乘以`2`即可，有些情况尾码可能会溢出到阶码，但是不影响结果的正确性。这一点得益于非规格化数编码时，`E = 1 - bias`，使用`1`去减偏置，使得非规格化数到规格化数的过度是平滑的，可自行验证。

最终结果如下：

```c
unsigned floatScale2(unsigned uf) {
	unsigned sign = (uf >> 31) & 0x1;
	unsigned exp = (uf & 0x7FFFFFFF) >> 23;
	unsigned frac = uf & 0x7FFFFF;
	unsigned rst;
	if (exp == 0xFF) {
		return uf;
	} else if (exp == 0) {
		frac <<= 1;
		rst = (sign << 31) | farc;
	} else {
		++exp;
		rst = (sign << 31) | (exp << 23) | farc;
	}

	return rst;
}
```

#### 12.浮点数转整型

题目要求：

```
/* 
 * floatFloat2Int - Return bit-level equivalent of expression (int) f
 *   for floating point argument f.
 *   Argument is passed as unsigned int, but
 *   it is to be interpreted as the bit-level representation of a
 *   single-precision floating point value.
 *   Anything out of range (including NaN and infinity) should return
 *   0x80000000u.
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 30
 *   Rating: 4
 */
```

对于非规格化数，直接返回零即可；

对于无限大，以及超过整型数范围的值，返回`0x80000000u`；

对于其他的值，根据解码对尾码进行截取即可。

具体代码：

```c
int floatFloat2Int(unsigned uf) {
  unsigned exp = (uf & 0x7F800000) >> 23;
  int sign = uf >> 31 & 0x1;
  unsigned frac = uf & 0x7FFFFF;
  int E = exp - 127;
  if (E < 0) {
    // 值小于1，包含非规格数
    return 0;
  } else if (E >= 31) {
    // 值大于int型的最大值
    return 0x80000000u;
  } else {
    // 加上省去的1
    frac = frac | 1 << 23;

    if (E < 23) {
      // 指数较小，需要舍去部分尾数
      frac >>= (23 - E);
    } else {
      // 指数较大，可以包含全部位数
      frac <<= (E - 23);
    }

  }

  if (sign) {
    return -frac;
  } else {
    return frac;
  }
  
}
```

#### 13.求指数 $2.0 ^ x$ 的值

题目要求：

```
/* 
 * floatPower2 - Return bit-level equivalent of the expression 2.0^x
 *   (2.0 raised to the power x) for any 32-bit integer x.
 *
 *   The unsigned value that is returned should have the identical bit
 *   representation as the single-precision floating-point number 2.0^x.
 *   If the result is too small to be represented as a denorm, return
 *   0. If too large, return +INF.
 * 
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. Also if, while 
 *   Max ops: 30 
 *   Rating: 4
 */
```

对于极大和极小的数不能表示，

对于 8 位阶码的浮点数，阶码码最大为`0xFFFFFFFE`，为`254`，偏置为`127`，故阶码最大能表示 $2 ^ {127}$；

其尾码为 23 位，故最小的非规格数是尾码为`0x00000...001`，此时表示的浮点数为 $2 ^ {-149}$；

故可得`x`的合法区间为 $[-149, 127]$。

对于合法的值，按照偏置区别规格数和非规格数即可。

代码如下：

```c
unsigned floatPower2(int x) {
  if (x > 127) {
    return 0xFF << 23;
  } else if (x < -149) {
    return 0;
  } else if (x > -126) {
    int exp = x + 127;
    return (exp << 23);
  } else {
    int t = 149 + x;
    return (1 << t);
  }
  return 2;
}
```

#### 参考目录：

[CSAPP:Lab1 -DataLab 超详解 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/339047608)
