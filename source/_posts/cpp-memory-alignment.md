---
title: C/C++ 内存对齐
date: 2020-04-03 14:46:23
tags:
- c
- c++
- 内存对齐
categories:
- c++
copyright: true
---

## 什么是内存对齐

内存对齐，也叫字节对齐。  
现代计算机中，内存空间都是按照字节划分的，从理论上讲似乎对任何类型的变量的访问可以从任何地址开始，但实际情况是在访问特定类型变量的时候经常在特定的内存地址访问，这就需要各种类型数据按照一定的规则在空间上排列，而不是顺序的一个接一个的排放，这就是对齐。
<!-- more -->

CPU每次从内存中取出数据或者指令时，并非想象中的一个一个字节取出拼接的，而是根据自己的字长，也就是CPU一次能够处理的数据长度取出内存块，比如32位处理器将取出32位也就是4个字节的内存块进行处理。

## 为什么需要内存对齐

1、平台原因(移植原因)：不是所有的硬件平台都能访问任意地址上的任意数据的；某些硬件平台只能在某些地址处取某些特定类型的数据，否则抛出硬件异常。  
2、性能原因：数据结构(尤其是栈)应该尽可能地在自然边界上对齐。原因在于，为了访问未对齐的内存，处理器需要作两次内存访问；而对齐的内存访问仅需要一次访问。

## 内存对齐的规则

每个特定平台上的编译器都有自己的默认“对齐系数”(也叫对齐模数，例如32位处理器，默认为4，64位处理器，默认为8)。程序员可以通过预编译命令#pragma pack(n)，n=1,2,4,8,16来改变这一系数，其中的n就是你要指定的“对齐系数”。  
规则：  
1、数据成员对齐规则：结构(struct)(或联合(union))的数据成员，第一个数据成员放在offset为0的地方，以后每个数据成员的对齐按照#pragma pack指定的数值和这个数据成员自身长度中，比较小的那个进行。  
2、结构(或联合)的整体对齐规则：在数据成员完成各自对齐之后，结构(或联合)本身也要进行对齐，对齐将按照#pragma pack指定的数值和结构(或联合)最大数据成员长度中，比较小的那个进行。  
3、结合1、2可推断：当#pragma pack的n值等于或超过所有数据成员长度的时候，这个n值的大小将不产生任何效果。  

根据上述规则，得到整体对齐系数= min((max(type1,type2,type3,...), n)  
其中type为结构体中的类型的字节大小，n为#pragma pack(n)设定值  

实例：

```c
// 32位系统
#pragma pack(n) /* n = 1, 2, 4, 8, 16 */

typedef struct mem_align
{
    int a;      // 4byte
    char b;     // 1byte
    short c;    // 2byte
    char d;     // 1byte
} ma;

int main()
{
    printf("ma size=%ld\n", sizeof(ma));
    return 0;
}
```

**1. #pragma pack(1)**

按照1字节对齐的结果：  
ma size=8  
分析：  

```c
typedef struct mem_align
{
    int a;      // 长度4 > 1 按1对齐；偏移量为0；存放位置区间[0,3]
    char b;     // 长度1 = 1 按1对齐；偏移量为4；存放位置区间[4]
    short c;    // 长度2 > 1 按1对齐；偏移量为5；存放位置区间[5,6]
    char d;     // 长度1 > 1 按1对齐；偏移量为7；存放位置区间[7]
    // 共8个字节
} ma;
```

   整体对齐系数=min(max(int, char, short, char), 1) = 1  
所以不需要再进行整体对齐。整体大小就为8。  

**2. #pragma pack(2)**

按照2字节对齐的结果：  
ma size=16  
分析：

```c
typedef struct mem_align
{
    int a;      // 长度4 > 1 按2对齐；偏移量为0；存放位置区间[0,3]
    char b;     // 长度1 = 1 按2对齐；偏移量为4；存放位置区间[4]
    short c;    // 长度2 > 1 按2对齐；偏移量要提升到2的倍数6；存放位置区间[6,7]
    char d;     // 长度1 > 1 按2对齐；偏移量为8；存放位置区间[8]
    // 共9个字节
} ma;
```

整体对齐系数=min(max(int, char, short, char), 2) = 2  
将9提升到2的倍数，则为10，所以最终结果为10个字节。

**3. #pragma pack(4)**

按照4字节对齐的结果：  
ma size=12  
分析：  

```c
typedef struct mem_align
{
    int a;      // 长度4 > 1 按2对齐；偏移量为0；存放位置区间[0,3]
    char b;     // 长度1 = 1 按2对齐；偏移量为4；存放位置区间[4]
    short c;    // 长度2 > 1 按2对齐；偏移量要提升到2的倍数6；存放位置区间[6,7]
    char d;     // 长度1 > 1 按2对齐；偏移量为8；存放位置区间[8]
    // 共9个字节
} ma;
```

整体对齐系数=min(max(int, char, short, char), 4) = 4  
将9提升到4的倍数，则为12，所以最终结果为12个字节。

**4. #pragma pack(8)**

按照8字节对齐的结果：
ma size=12  
分析：  

``` c
typedef struct mem_align
{
    int a;      // 长度4 > 1 按2对齐；偏移量为0；存放位置区间[0,3]
    char b;     // 长度1 = 1 按2对齐；偏移量为4；存放位置区间[4]
    short c;    // 长度2 > 1 按2对齐；偏移量要提升到2的倍数6；存放位置区间[6,7]
    char d;     // 长度1 > 1 按2对齐；偏移量为8；存放位置区间[8]
    // 共9个字节
} ma;
```

整体对齐系数=min(max(int, char, short, char), 8) = 4  
将9提升到4的倍数，则为12，所以最终结果为12个字节。

**5. #pragma pack(16)**

按照16字节对齐的结果：  
ma size=12  
分析：  

``` c
typedef struct mem_align
{
    int a;      // 长度4 > 1 按2对齐；偏移量为0；存放位置区间[0,3]
    char b;     // 长度1 = 1 按2对齐；偏移量为4；存放位置区间[4]
    short c;    // 长度2 > 1 按2对齐；偏移量要提升到2的倍数6；存放位置区间[6,7]
    char d;     // 长度1 > 1 按2对齐；偏移量为8；存放位置区间[8]
    // 共9个字节
} ma;
```

整体对齐系数=min(max(int, char, short, char), 16) = 4  
将9提升到4的倍数，则为12，所以最终结果为12个字节。
