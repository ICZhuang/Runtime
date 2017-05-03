# **三、**Runtime  概念部分 之 Method

## **Method**

顾名思义，Method 即方法，在 Runtime 中，通过 class_getInstanceMethod 获得类中的对象方法

```objective-c
Method class_getInstanceMethod(Class cls, SEL name);
```

通过 class_getClassMethod 获取类方法

```objective-c
Method class_getClassMethod(Class cls, SEL name);

```

Method 是 objc_method 结构体的指针，objc_method 如下定义，它包含了三个字段：方法名，返回值类型和参数类型的编码和方法实现

```objective-c
struct objc_method {
    SEL method_name                                          OBJC2_UNAVAILABLE;
    char *method_types                                       OBJC2_UNAVAILABLE;
    IMP method_imp                                           OBJC2_UNAVAILABLE;
}                                                            OBJC2_UNAVAILABLE;
```

字段中 method_types，类比 Ivar 和 property，可以猜想它应该是返回值和参数的类型编码字符串，只不过这个结构体的字段值不能 直接通过常规的结构体字段访问方法去获取，而是通过 Runtime 提供的方法获得各个部分的type encoding。

```objective-c
const char *method_getTypeEncoding(Method m);
char *method_copyReturnType(Method m);
char *method_copyArgumentType(Method m, unsigned int index);
void method_getReturnType(Method m, char *dst, size_t dst_len);
void method_getArgumentType(Method m, unsigned int index, char *dst, size_t dst_len); 
```

> 注意：Runtime中的某些函数使用之后，是需要调用 free() 释放返回的字符串。记得参考注释

字段中 method_name，顾名思义，是方法的名字，但特别的是这不是char *类型，而是 SEL 类型，通过  method_getName 获得

```objective-c
SEL method_getName(Method m);
```

字段中 IMP 则为 方法的实现 

```objective-c
IMP method_getImplementation(Method m);
```



## SEL

method selector (方法选择器，名字暂这样取)。依代码来看， SEL 也是一个指针，这是一个 objc_selector 结构体的指针。

```objective-c
typedef struct objc_selector *SEL;
```

问题在于 objc_selector 是怎样定义的？头文件没有找到定义，而查阅网络资料时找到几点说法汇总如下：

1. objc_selector 的定义取决于使用的是 GNU Runtime 还是 Apple Runtime
2. 在 Apple 中 SEL 可以理解为 char *
3. 另一种则是 SEL 可以理解为函数指针，C 语言中可以直接取还是指针，但在OC中依靠@selector()来去

> 如果上面说的词不达意，请参考
>
> [http://www.cnblogs.com/geek6/p/4106199.html](http://www.cnblogs.com/geek6/p/4106199.html)
>
> [http://blog.csdn.net/jeffasd/article/details/52084639](http://blog.csdn.net/jeffasd/article/details/52084639)

虽然亲测将以上说的类型强转并输出成功，但笔者对于这几点还是没能了然，综合上面几点和以下所述，可以大胆猜想，在iOS 中， objc_selector 结构体`至少`包含了方法名这一个字段，然后这个结构体指针作为 objc_method 的 method_name

因为其中关于 SEL (objc_selector *) 的关键方法中有以下两个

```objective-c
/** 
 * Registers a method with the Objective-C runtime system, maps the method 
 * name to a selector, and returns the selector value.
 * 
 * @param str A pointer to a C string. Pass the name of the method you wish to register.
 * 
 * @return A pointer of type SEL specifying the selector for the named method.
 * 
 * @note You must register a method name with the Objective-C runtime system to obtain the
 *  method’s selector before you can add the method to a class definition. If the method name
 *  has already been registered, this function simply returns the selector.
 */
OBJC_EXPORT SEL sel_registerName(const char *str);

/** 
 * Returns a Boolean value that indicates whether two selectors are equal.
 * 
 * @param lhs The selector to compare with rhs.
 * @param rhs The selector to compare with lhs.
 * 
 * @return \c YES if \e lhs and \e rhs are equal, otherwise \c NO.
 * 
 * @note sel_isEqual is equivalent to ==.
 */
OBJC_EXPORT BOOL sel_isEqual(SEL lhs, SEL rhs); 
```

根据注释， sel_getUid 与 sel_registerName 的实现是相同的，都是用于注册方法。它们返回值的注释是这么说明

```objective-c
// A pointer of type SEL specifying the selector for the named method
```

可以确定，SEL 指向的是一个叫 ‘selector’ 的东西，暂就称它为选择器。传递的参数是一个字符串，难道返回值还是简单的字符串么？（这个怀疑理由牵强了，但笔者是这样猜想的）

再者，看下面这段注释：

```objective-c
// Registers a method with the Objective-C runtime system, maps the method name to a selector, and returns the selector value.
```

意思则是 sel_registerName 作用是注册 method, 让程序运行过程中可以识别并调用到它，所谓注册，就是将 method 映射到 一个 selector 上，在添加一个方法前，必须先注册方法名以获得方法的 selector，如果这个方法名曾经注册过，那么这个方法纯粹返回该方法的selector。 

笔者猜想，通过 selector 可以找到对应的 method，这相当于 method 的名字，只不过这个名字是一个包含了方法名的一个结构体而已。

对于 SEL 还有以下几个访问方法

```objective-c
const char *sel_getName(SEL sel);
BOOL sel_isEqual(SEL lhs, SEL rhs);
BOOL sel_isMapped(SEL sel);
```

> 只是笔者不理解的是不同类的两个方法，如果方法名相同，他们的selector也是一样的。(后续补上)



## IMP

方法分 声明和定义(实现)， 只有声明而没实现，强行调用该方法会崩溃。 方法中{}中的代码可以认为是方法的实现。IMP 就是指向实现的首地址(函数指针)，因为方法的实现放在内存代码区，也是有相应入口地址的。

通过 method_getImplementation 获得就可以获得一个方法(Method)的 实现入口地址(IMP)

```objective-c
IMP method_getImplementation(Method m);
```

后者通过类来获取某个方法的IMP

```objective-c
IMP class_getMethodImplementation(Class cls, SEL name);
IMP class_getMethodImplementation_stret(Class cls, SEL name);
```

IMP 也是一个指针，这个指针指向方法入口

```objective-c
#if !OBJC_OLD_DISPATCH_PROTOTYPES
typedef void (*IMP)(void /* id, SEL, ... */ ); 
#else
typedef id (*IMP)(id, SEL, ...); 
#endif
```

调用一个方法，则是给这个对象发送一个消息，参数 id 传入的就是调用方法的对象，该方法为类方法则传递类对象，实例方法则传递当前调用对象； 参数 SEL 为该方法的选择器； 可变参数则问改方法的参数列表。方法的实现都是以这个格式定义的。

> 常规方法调用通过 method selector 来找到该方法的实现入口指针(IMP)。如果事先已经知道了IMP，则直接可调用IMP，而跳过方法查找那一步的消耗。



关于方法 IMP 的调用示例(当然在平常开发中不是这样干的)：

![image-01](https://github.com/ICZhuang/Runtime/blob/master/image/03_01.png?raw=true)

除了可以从已知的 method 中获取它的入口指针 IMP，也可以通过传递一个 block 凭空创建一个函数实现

```objective-c
IMP imp_implementationWithBlock(id block);
```

block 按照 method_return_type ^(id self, method_args...) 定义，其中 method_args… 则是你要声明的参数列表当调用到该 IMP 时，就会调用这个 block。注意在销毁的时候调用 imp_removeBlock 移除，一旦移除，再次调用方法，则崩溃。

```objective-c
BOOL imp_removeBlock(IMP anImp);
```

另外这有一个方法可获得 IMP 的 block

```objective-c
id imp_getBlock(IMP anImp);
```

但不要以为所有实现都可以通过 imp_getBlock 获得 block 形式的实现，已经存在的 IMP 获取到的 block 是NULL只有 block 关联的 IMP 才能获得，通常是以 imp_implementationWithBlock 创建的。

凭空创建一个实现后，我们就可以利用 Runtime 替换已有的方法的实现

```objective-c
IMP method_setImplementation(Method m, IMP imp);
```



## 遍历Method

```objective-c
- (void)getMethods {
    unsigned count = 0;
    Method *methods = class_copyMethodList(RTModel.class, &count);
    
    Method method;
    for (int index = 0; index < count; ++index) {
        method = methods[index];
        
        printf("%s\n", sel_getName(method_getName(method)));
    }
}
```



## Mthod 拾遗

method 相关属性的访问方法都是 method_ 开头的，（SEL 则是以 sel_ 开头的，IMP, Ivar, Property 也是一样的）除了上面三种（method_types，method_name，method_imp）外，还有以下信息可以访问

1、参数个数

```objective-c
unsigned int method_getNumberOfArguments(Method m);
```

2、method description

```objective-c
struct objc_method_description *method_getDescription(Method m);
```