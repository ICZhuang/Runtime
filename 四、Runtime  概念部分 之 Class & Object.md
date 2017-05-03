# **四、**Runtime 概念部分 之 Class & Object

> 本文githud地址 [https://github.com/ICZhuang/Runtime](https://github.com/ICZhuang/Runtime)

除了前面介绍的三部分（Ivar/Property/Method），基础的概念部分其实还应包括 Protocol 和 Category。但笔者打算暂时放下它们。这里想先学习Class & Object的原因有以下几点

- Ivar/Property/Method 三者是我们平常开发最常见的组成部分，而且足以构成一个完整类；
- 关于 Ivar/Property/Method 还有一些操作函数，这些函数实际是针对于 Class 或者 Object 的；
- Protocol、 Category 和前面三者基本是差不多的学习方法；

> 关于 Runtime 的文章太多了，基本上都会说明三个关键 类对象(Class) 元类对象(Meta Class) 和实例对象(Instance)，虽然可能会晕晕乎乎的，但怎么说也有点大概印象，那么在笔者这里只是列举笔者认为的关键点来帮助理解，说白了，也是方便笔者重拾对Runtime的理解。
>
> 还要说明一点，要完全剖析 Runtime，笔者认为看源码最合适，奈何笔者不是C、C++大牛，这重任暂时被搁下了，所以笔者写的文章也不会深入的太底层，层面还是来于从网络文章和官方文档加上自己实践验证学习而来的。



## 从两个结构体开始说起

Runtime 中对对象的定义如下所示，是一个 objc_object 结构体，里面拥有一个 Class 类型的 isa 指针

```objective-c
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};
```

> 但笔者理解，这里的对象不要狭隘的理解成是实例对象，因为接下来类的字段也有一个isa
>

相信在接触 Runtime 之初，大家都听过这么一句话：“拥有这个isa指针的都是对象”，有这样一个解释我觉得很合理：’isa’其实就是 ‘is  a’，这倒是挺形象的描述了这句话。。因为，isa 指针就是指向的该对象所属的类。

往下查找 Class 类型，它是一个 objc_class 结构体类型指针，objc_class 结构体如下所示

```objective-c
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
```

套路来了，这里笔者还是要假装地提一下这个问题，方便做牵引和说明，就是：“既然对象 isa 指向的是所属的类，怎么类里面还有一个isa？搞事情啊？”

无疑，类拥有了 isa 指针，那么类也是一个对象，称为类对象，相信大家可能听过或者已经了解过了，它在内存中就是一个单例对象。那既然 isa 是指向其类的指针，那类的 isa 指针如何指向？

答案是，类对象的 isa 指向的称为元类（meta class），可以看出，元类也是 Class 类型的。可以说类对象的类型就是它的元类

其实结构体 objc_object 只是定义了 Objective-C 中对象应该长什么样，以此说明类和元类都是对象。任何继承与NSObject的类都有一个'isa'指针，NSObject就是OC层面的‘objc_object’。



## 内存布局

实例对象也是对象，难道它的实例变量就只有一个isa？那继承NSObject后自己定义的实例变量放在哪里？

这里要先停一下了，推荐大家先看一下这篇博客，了解一个对象在内存的布局和它与类之间的关系，将会有大帮助 因为不想写同样的篇幅，这里放上博客链接 http://www.cnblogs.com/csutanyu/archive/2011/12/12/Objective-C_memory_layout.html

这里附上笔者理解后做的关于RTModel的内存布局，就是指针指来指去。(不知Heap位置放对了没有，因为不知道class对象都放在哪，Heap基于笔者对C的理解)

![image-01](https://github.com/ICZhuang/Runtime/blob/master/image/04_01.png?raw=true)

我们定义的那些类，运行时候加载到内存中， 将类中定义的对象方法存到方法列表的 methodLists 字段中，成员变量存到 ivars 中，遵循的协议存到 protocols 中，这些都是 objc_class 结构体就已经定义的

实例对象 instance 可以在内存空间允许的情况下创建多个，实例对象依据类(class)对象而生。但实例对象存放的是实例变量(当然包括了父类声明的成员变量，父类的在前，子类的在后)，因为ivar才是各个实例对象不同的地方，像方法这种共用的就放在类中，否则开辟一个实例变量就拷贝一份方法列表，那内存会吃不消的。

调用对象方法，就是给实例对象发送消息，通过在它的类中的方法列表 methodLists 找到方法并执行。

接着可以说说元类(meta class)了，如上面所述，类(class)的isa指向的就是元类。而且元类的结构跟类的结构是一样的。只是在元类的方法列表 methodLists 中，存放的是类中定义的 类方法，而类中的 methodLists 存放的是对象方法。当调用类方法的时候，通过在元类中找到方法并执行。还有关于 Cache，是关于方法的缓存，再深究的话就到讲实现上面了，笔者还没研究那么深呢，哈哈，原谅。

![image-02](https://github.com/ICZhuang/Runtime/blob/master/image/04_02.png?raw=true)

可以发现，元类既然也是 Class 类型，那它也应该是类对象，isa指向它的类。那最终isa指向谁呢？这里笔者不负责任的说，上面推荐的博客最下方的图已经比较形象清晰的了。。 或许你还见过那个文档中拷出来的图，跟博客中的是一样的意思，比如这篇博客也有极大帮助[http://blog.csdn.net/wzzvictory/article/details/8592492](http://blog.csdn.net/wzzvictory/article/details/8592492)



## 总结：

- Runtime 中对对象的概念 和我们开发中念叨的实例对象要分清，这里只是OC对对象的宏观的定义；
- 实例对象存放着各自的实例变量，但共用 class 对象，实例对象的 isa 指向它的 class 对象；
- class 的 isa 指向它的元类对象 meta class，meta class的 isa 都指向 NSObject meta class;
- class 的 super_class 指向其父类的 class，NSObject class的super_class 指向**0x0**；
- meta class 的 super_class 指向其父类的 meta class， 但 NSObject meta class 的 super_class 指向它自己的类对象(class);