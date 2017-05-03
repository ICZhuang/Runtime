# **一、** Runtime 概念部分 之 Ivar

> 笔者取名概念部分，是因为笔者认为理解 Runtime 的基础就是先熟悉各组织，这些组织包括 Ivar、Property、Method、Category、Protocol、Class、Object 和 Message
>
> 本文githud地址 [https://github.com/ICZhuang/Runtime](https://github.com/ICZhuang/Runtime)



## **Ivar**

指类中的一个成员变量，如类RTModel中的  _ivar_01、  _ivar_02、 _ivar_03，包括  _property_01  /  _property_02 

```objective-c
@interface RTModel : NSObject {
    @public NSString *_ivar_01;
}
@property (nonatomic,  assign,  readonly)   NSInteger    i_property_01;
@property (nonatomic,  strong)              NSArray     *a_property_02;
@end

@interface RTModel () {
    NSString    *_ivar_02;
}
@end
 
@implementation RTModel {
    NSInteger   *_ivar_03;
}
@end
```

通过 class_copyIvarList 可以将成员获得类的 Ivar 列表，但是父类的Ivar 没有包含在里面，注意使用完成后需要通过 free() 将结果释放掉。

```objective-c
Ivar *class_copyIvarList(Class cls, unsigned int *outCount)
```



## 访问Ivar

获得了Ivar列表即可遍历每个Ivar，通过以下方法取得Ivar的特定信息

1、取name

```objective-c
const char *ivar_getName(Ivar v); 
```

2、取 Type Encoding

```
const char *ivar_getTypeEncoding(Ivar v);
```

3、取 offset 偏移量

```objective-c
ptrdiff_t ivar_getOffset(Ivar v); 
```



## 类型编码（Type Encodings）

编译器可将 <u>类型</u> 编码成一个字符串，这个字符串就是类型编码（Type Encodings）用于描述变量类型。通过@encode()即可获得某个类型的编码，任何能在sizeof()使用的参数都可以传递给@encode()。详细请参考 ‘Objective-C Runtime Programming Guide > Type Encodings’

比如RTModel中各Ivar的type encoding,   _ivar_03 的类型是NSInteger指针， encoding表示成`^q`

![01-01](https://github.com/ICZhuang/Runtime/blob/master/image/01_01.png?raw=true)

> 关于类型编码也是一个基础概念，Runtime 中很多地方依靠这个概念实现的。有必要先了解一下。
>
> 英文文档 [Objective-C Runtime Programming Guide > Type Encoding](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1)
>
> 笔者译文 [Type Encoding & Declared Property](https://github.com/ICZhuang/Runtime/blob/master/附、Type%20Encoding%20%26%20Declared%20Property.md)



## 偏移量

每个Ivar在类中是顺序排列的，它们有自己的位置，偏移量就是用来记录它的位置的。比如RTModel中的Ivar的偏移量就如图所示，对象在内存中通过指针引用者，假设这个指针指向的地址为A,那么 _ivar_01 距离A的距离就是8个字节，_ivar_02 距离A为16字节。

![01-02](https://github.com/ICZhuang/Runtime/blob/master/image/01_02.png?raw=true)

但是在类的category中声明的则不是Ivar，它只是声明，并不分配空间，所以并不是Ivar。



## 遍历

遍历类中所有的Ivar，并获取它们的信息。

```objective-c
- (void)getIvars {
    unsigned int count = 0;
    Ivar *ivars = class_copyIvarList(RTModel.class, &count);
    
    Ivar ivar;
    for (int index = 0; index < count; ++index) {
        ivar = ivars[index];
        
        printf("%s - %s\n", ivar_getName(ivar), ivar_getTypeEncoding(ivar));
    }
}
```



## 扩展认识

Ivar 其实是结构体objc_ivar，这么说不准确，准确来说它是一个objc_ivar结构体的指针，objc_ivar就代表着一个成员变量，以上所列关于取得Ivar信息的方法都是Runtime内部在操作这个结构体，取得的信息也是 objc_ivar结构体中的某个字段的值。以上方法取得的值分别对应ivar_name/ivar_type/ivar_offset字段的值。

(space 猜想是Ivar内存对齐的后的剩余字节，没去证实，暂且不说)

```objective-c
struct objc_ivar {
    char *ivar_name                                          OBJC2_UNAVAILABLE;
    char *ivar_type                                          OBJC2_UNAVAILABLE;
    int ivar_offset                                          OBJC2_UNAVAILABLE;
#ifdef __LP64__
    int space                                                OBJC2_UNAVAILABLE;
#endif
} 
```

事实上，Runtime 的大部分方法都是在操作某一个结构体，找到对应的结构体，就可以看出该结构体具备什么字段，进而知道方法是干什么用的了。比如以下结构体

```objective-c
typedef struct objc_method *Method;
typedef struct objc_ivar *Ivar;
typedef struct objc_category *Category;
typedef struct objc_property *objc_property_t;
```