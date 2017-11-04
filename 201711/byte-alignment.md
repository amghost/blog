# 字节对齐
网络上对字节对齐已经有很多详细的介绍文档了，比如[C语言字节对齐、结构体对齐最详细的解释](http://blog.csdn.net/lanzhihui_10086/article/details/44353381)，所以这里只描述我总结的一些思路。

## 核心思想

访问任何对象都不要有多余的内存访问操作，换言之，任何对象的起始地址值为其对齐值的整数倍

整数倍的原因是，这样访问一个对象就不存在多余的内存访问操作，比如系统字长为8字节，`double`类型对齐值为8字节，系统是按字长寻址，且按字长读取的，如果`double`类型的对象内存地址是8的整数倍，那一次内存读取就可以把该对象读出来。

这里的对象包括：
- 单独声明的基本类型
- 结构体实例
- 结构体成员
- 数组成员

对齐值是我对 **x-byte aligned** 的翻译

摘引wikipedia的描述:

for 32-bit x86
- A char (one byte) will be 1-byte aligned.
- A short (two bytes) will be 2-byte aligned.
- An int (four bytes) will be 4-byte aligned.
- A long (four bytes) will be 4-byte aligned.
- A float (four bytes) will be 4-byte aligned.
- A double (eight bytes) will be 8-byte aligned on Windows and 4-byte aligned on Linux (8-byte with `-malign-double` compile time option).
- A long long (eight bytes) will be 4-byte aligned.
- A long double (ten bytes with C++Builder and DMC, eight bytes with Visual C++, twelve bytes with GCC) will be 8-byte aligned with C++Builder, 2-byte aligned with DMC, 8-byte aligned with Visual C++, and 4-byte aligned with GCC.
- Any pointer (four bytes) will be 4-byte aligned. (e.g.: char*, int*)

for 64-bit system:
- A long (eight bytes) will be 8-byte aligned.
- A double (eight bytes) will be 8-byte aligned.
- A long long (eight bytes) will be 8-byte aligned.
- A long double (eight bytes with Visual C++, sixteen bytes with GCC) will be 8-byte aligned with Visual C++ and 16-byte aligned with GCC.
- Any pointer (eight bytes) will be 8-byte aligned.

可以看到同一个类型在不同的系统下对齐值不同，但都遵循一个原则：对象地址必须是其类型对齐值的倍数，比如64位系统中，double对象的内存地址默认是8的倍数

## 结构体字节对齐
一般内置对象的字节对齐还是比较简单的，按各自对齐值就可以确认其地址为哪个值的倍数。但是对于结构体就不一样了，因为对象都有字节对齐，所以导致结构体内存在padding

```
struct MyStruct
{
    char c1;
    int i1;
    char c2;
};

MyStruct my_struct;
```

因为要满足 `my_struct.c1`、`my_struct.i1`、`my_struct.c2` 字节对齐，它们的内存地址要为各自对齐值的倍数，这里我们先假设`my_struct`的起始地址是0(注意这个假设)
1. c1 对齐值为1，必然是对齐的
2. i1 对齐值为4，内存地址必须是4的倍数，所以 c1 到 i1 中间空出(padding)了 3个字节
3. c2 对齐值为1，必然是对齐的
所以**至少**就会有填充 3 + 0 = 3，但是这是不够的。

如果我们还要表示 MyStruct的数组 `MyStruct[10] my_struct_array`，基于核心思想，我们要让数组每一个元素的每一个字段都满足内存对齐，但是显然不可能每个数组元素的起始地址都是0，如果不对`my_struct_array[0].c2`进行padding，那么`my_struct_array[1]`紧接着就不能保证其字段都是字节对齐的(这里稍微想象一下就好了)

所以需要对MyStruct结构体做padding，使其大小为其最大字段成员类型对齐值的整数倍，比如这里就是int类型对齐值4的整数倍。
所以在 `my_struct.c2` 后还要再补3个字节.
总padding字节数为：3 + 0 + 3 = 6
结构体大小：1 + 8 + 1 + 6 = 16(4的整数倍)

为什么是最大字段的对齐值？其实这基于这样的推导：
1. **所有基本类型的对齐值都成倍数关系**，如果最大字段是内存对齐的，则必然其他字段也可以是内存对齐的。
2. 由于每个结构体实例是字节对齐的，若结构体大小(也是对齐值)为最大字段对齐值的整数倍，则结构体起始地址是最大字段对齐值的整数倍，这个与起始地址为0是等价的(取模为0)，由于我们已经保证了在结构体内部各字段是字节对齐的，所以必然其最大字段也可以是字节对齐的；
所有条件都可以满足，Happy ending。

总结：结构体内各字段以结构体起始地址为0各自padding，然后结构体以最大字段对齐值的最小整数倍为大小，padding补足

## pragma pack(x)
有时候我们不想用默认的对齐方式(牺牲内存读取优化)，而希望充分利用内存，那我们可以通过 `#pragma pack(1)` 来消除对齐导致的padding。

首先 `x` 必须的较小的2的次方：1、2、4、8、16，在我的机器上最大是16。

然后它的工作方式：
1. 如果 `x` 比原类型的默认对齐值小，则以`x`作为类型的对齐值
2. 如果 `x` 大于等于原类型的默认对齐值，仍旧以默认对齐值为对齐值

也就是说，如果我用 `#pragma pack(16)`，那就相当于没效果了，因为我系统上没有默认对齐值超过16的(最多等于16)

举个例子吧

```
struct MyStruct
{
    char c1;
    double d1;
    char c2;
};

#pragma pack(2)
struct MyStructPack2
{
    char c1;
    double d1;
    char c2;
};

#pragma pack(4)
struct MyStructPack4
{
    char c1;
    double d1;
    char c2;
};

int main()
{
    cout << "MyStruct:" << sizeof(MyStruct) << endl;
    cout << "MyStructPack2:" << sizeof(MyStructPack2) << endl;
    cout << "MyStructPack4:" << sizeof(MyStructPack4) << endl;
}
```

输出结果为
```
MyStruct:24
MyStructPack2:12
MyStructPack4:16
```

`MyStructPack2` 中，`c1`对齐值还是1，`d1`对齐值为2，`c2`对齐值还是1，所以`c1`后padding 1个字节，最后结构体以最大字段对齐值(`d1`的对齐值2)的倍数补足，也就是在 `c2`后padding 1个字节，结果为 12
`MyStructPack4` 中，`c1`对齐值还是1，`d1`对齐值为4，`c2`对齐值还是1，所以`c1`后padding 3个字节，最后结构体以最大字段对齐值(`d1`的对齐值4)的倍数补足，也就是在 `c2`后padding 3个字节，结果为 16

