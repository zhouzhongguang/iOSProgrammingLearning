# 类型编码（Type Encodings）

为了协助运行时系统，编译器为每个方法编码返回和参数类型到一个字符串并且将字符串和方法选择器关联起来。它使用的编码方案在其他上下文也很有用，因此可以通过@encode\(\)编译器指令公开获得。当给出一个类型说明时，@encode\(\)返回一个字符串编码该类型。类型可以是基础类型比如int，指针，一个标记的结构或联合，或一个类名--实际上可以用作C sizeof\(\)运算符参数的任何类型。

```
char *buf1 = @encode(int **);
char *buf2 = @encode(struct key);
char *buf3 = @encode(Rectangle);
```

下面的表格列出了类型代码。注意它们中的许多与用于存档或分发目时编码对象的代码重叠。然而，这里列出的有些代码你不能用作编写编码器，并且你想用作编写编码器的代码有些不是通过@encode\(\)产生的。（有关编码用于归档或分发的编码对象的更多信息，请参阅Foundation Framework参考中的NSCoder类规范。）

| Code | Meaning |
| :--- | :--- |
| c | A char |
| i | An int |
| s | A short |
| l | A long. 1 is treated as a 32-bit quantity on 64-bit programs. |
| q | A long long |
| C | An unsigned char |
| I | An unsigned int |
| S | An unsigned short |
| L | An unsigned long |
| Q | An unsigned long long |
| f | A float |
| d | A double |
| B | A C++ bool or a C99 \_Bool |
| v | A void |
| \* | A character string \(char \*\) |
| @ | An object \(whether statically typed or typed id\) |
| \# | A class object \(Class\) |
| : | A method selector \(SEL\) |
| \[array type\] | An array |
| {name=type...} | A structure |
| \_b\_num | A bit field of num bits |
| ^type | A pointer to type |
| ? | An unknown type \(among other things, this code is used for fucntion pointers\) |

> 重要：Objective-C不支持long double类型。@encode\(long double\)返回d，跟double的编码一样。

数组的类型代码包括在方括号内；数组中元素的个数在开括号的之后立即给出，在数组类型之前。例如，一个包含12个指向浮点数的数组将会这样编码：

```
[12^f]
```

结构体在大括号内指定，而联合在括号内。首先列出结构体标签，然后是等号和序列中列出的结构体字段的代码。例如结构：

```
typedef struct example {
    id   anObject;
    char *aString;
    int  anInt;
} Example;
```

将会这样编码：

```
{example=@*i}
```

不管定义的类型名称（Example）还是结构标记（example）都传递给@encode（），相同的编码都会生成。 结构指针的编码携带有关结构字段的相同数量的信息：

```
^{example=@*i}
```

但是，另外一个间接级别删除了内部类型规范：

```
^^{example}
```

对象被视为结构体。例如，将NSObject类名称传递给@encode\(\)会产生以下编码：

```
{NSObject=#}
```

NSObject类只声明一个类型为Class的实例变量isa。

请注意，虽然@encode\(\)指令不会返回它们，但是运行时系统用于声明协议中的方法时，它使用表6-2中列出的其他编码作为类型限定符。

表6-2 Objective-C方法编码

| Code | Meaning |
| :--- | :--- |
| r | const |
| n | in |
| N | inout |
| o | out |
| O | bycopy |
| R | byref |
| V | oneway |



