# **Type Encoding & Declared Property**

本文的出现是因为笔者在学习Runtime的学习过程中，这是一个必备的知识点。在这点上笔者也模糊了一点时间，问题在于区分类型编码、修饰符编码和访问属性的编码。在看YYClassInfo的时候差点没晕了。



## Type Encoding

> 翻译自‘Objective-C Runtime Programming Guide > Type Encodings’

为了完善 Runtime 机制，编译器将方法的返回值类型和参数的类型编码成字符串，并且将这个字符串与方法的选择器(selector)关联起来。这个字符串就是类型编码(Type Encoding)。除此之外，类型编码适用于其它一些上下文中，比如成员变量类型，属性类型等等。

通过编译指令@encode()，则可以获得对应的类型编码。传递的类型可以是基本数据类型，比如int，指针(pointer)，结构体(struct)或者联合(union), 还可以是一个类名 任何类型都可以。事实上，适用于@sizeof()的类型都适用于@encode()。

```objective-c
char *buf1 = @encode(int **);
char *buf2 = @encode(struct key);
char *buf3 = @encode(Rectangle);
```



下面列举了所有的 Type Encoding。需要注意的是，当我们编写一个coder，将对象归档或解归档的时候，都不能用@encode()可以生成的。（详细的要参考 NSCoder class 关于将对象解档归档的描述）

![image-01](https://github.com/ICZhuang/Runtime/blob/master/image/type%20encoding.png?raw=true)

> 注意：Objective-C 不支持 long double 类型， @encode(long double) 返回的是 d，跟@encode(double) 是一样的返回值



一个 C数组 的编码用 [] 包裹表示，其中数组的元素个数放在数组元素类型前面。 举例：12个 float 类型指针的一个数组，它的编码就是

```objective-c
[12^f]
```



struct 的编码则用{}，union 则用()。在{}里，struct 的 tag 放在最前面，紧接着一个 =，各个字段的编码按顺序排在后面。举例如结构体

```c
typedef struct example {
    id   anObject;
    char *aString;
    int  anInt;
} Example;
```

@encode 结果像这样

```objective-c
{example=@*i}
```

@encode(Example) 和 @encode(struct example) 结果是一样的。  如果是struct指针类型，结果同样带着个字段信息，就像

```objective-c
^{example=@*i}
```

但是如果是简介引用，则将内部各字段的类型编码去掉，如二级指针

```objective-c
^^{example}
```



Object 跟 struct 一样，例如@encode(NSObject)

```objective-c
{NSObject=#} 
```

NSObject class 只声明了一个 Class类型的 isa 变量



另外，在声明协议方法的时候，可能用到的类型修饰符如下表所示。但这些修饰符的编码却不能通过@encode()指令得到

![image-02](https://github.com/ICZhuang/Runtime/blob/master/image/qualifiers%20encoding.png?raw=true)

- in：输入参数，后续不再引用
- out：参数被引用作为返回值
- inout：输入参数，引用作为返回值
- const：常量参数
- oneway：无障碍结果返回
- bycopy：返回对象的拷贝
- byref：返回对象的代理

> 参考文章 http://blog.csdn.net/jasonjwl/article/details/51557009



## **Declared Property**

> 取自‘Objective-C Runtime Programming Guide > Declared Property’ 的后半部分 >  Property Type String

你可以使用Runtime 中 property_getAttributes 函数获得属性的名字、类型编码和其它访问属性

Property Type String 这个字符串是由 'T' 紧接着类型编码和一个逗号开始的，以 'V' 接着变量的名字作为结尾。在开头和结尾中间，访问属性以下列描述符表示，它们分别用逗号隔开

![image-03](https://github.com/ICZhuang/Runtime/blob/master/image/property%20type%20string.png?raw=true)

原文有示例，查看示例请移步[Decalred Property]( https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtPropertyIntrospection.html)

