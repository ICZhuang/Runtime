# 五、**Runtime **实践部分 之 Class & Object

上一章只是介绍了Class 和 Object 的概念，本来想继续介绍它们之间的一些函数的，奈何篇幅过长了。所以另开一章，介绍它们之间的一些函数，这些函数还包括对Ivar、property、Method的函数(前面几章还没学到class，所以在这补上)。作为简单实践，是笔者学习了前几篇概念之后，觉得到了要做一些实践的时候了。

> 其实Ivar、property、Method遍历也可以是实践的一部分了，这里不再做介绍



## **Adding Class**

本来是想先从 add_ivar 开始的，但这个方法是要在 objc_allocateClassPair 和 objc_registerClassPair 中间执行，即是说往一个已经存在的类中添加 Ivar 是不允许的。个人理解，如果类已经存在，再往里面添加Ivar，那添加之前创建的实例对象该不该拥有Ivar的空间。所以先介绍一下创建类。

创建一个新类分两步，allocate & register，参数 superclass 将成为该新类的父类， extraBytes 通常为0，如果传入的名字是已经存在的话，则创建失败，返回nil。

```objective-c
Class objc_allocateClassPair(Class superclass, const char *name, size_t extraBytes);
void objc_registerClassPair(Class cls); 
```

销毁一个类对象和它的元类对象，调用前提是不能仍存在该类的实例对象或者是子类，而且这个类是通objc_allocateClassPair 创建的。

```objective-c
void objc_disposeClassPair(Class cls); 
```

> 关于这个方法 objc_duplicateClass， 不应该由我们调用。它是用来辅助KVO机制用的。后面介绍。



## **Adding Ivar**

这个方法是要在 objc_allocateClassPair 和 objc_registerClassPair 中间执行，往一个已经存在的类中添加 Ivar 是不允许的。

```objective-c
BOOL class_addIvar(Class cls, const char *name, size_t size, 
                               uint8_t alignment, const char *types);
```

添加一个Ivar

```objective-c
Class allocate_clazz = objc_allocateClassPair(RTModel.class, "RTCopyModel", 0);
class_addIvar(allocate_clazz, "_ivar_04", sizeof(NSInteger), log2(_Alignof(NSInteger)), 
			  @encode(NSUInteger));
class_addIvar(allocate_clazz, "_ivar_05", sizeof(NSString *), log2(_Alignof(NSString *)), 				@encode(NSString *));
objc_registerClassPair(allocate_clazz);
```

> 参数 alignment 说明参考: [what-does-class-addivars-alignment-do-in-objective-c#](http://stackoverflow.com/questions/33184826/what-does-class-addivars-alignment-do-in-objective-c#)



## **Adding Property**

添加一个属性，如果已经存在则失败。

```objective-c
BOOL class_addProperty(Class cls, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount);
```

在类中进行添加属性，则显得比较麻烦了，需要创建需要赋予的attributes。

```objective-c
objc_property_attribute_t attributes[4];
/// prepare for param 'attributes'
objc_property_attribute_t attr_0 = {"T", "@\"NSArray\""};
objc_property_attribute_t attr_1 = {"&", ""};
objc_property_attribute_t attr_2 = {"N", ""};
objc_property_attribute_t attr_3 = {"V", "_property_03"};

attributes[0] = attr_0;
attributes[1] = attr_1;
attributes[2] = attr_2;
attributes[3] = attr_3;

class_addProperty(RTModel.class, "property_03", attributes, 4);
```

事实上变现不太良好。通常我们定义一个属性后，会自动带有它的Ivar，setter/getter 方法。但利用 Runtime 往一个类中添加属性后，只是声明了这个属性，但这个属性却没有对应添加它的Ivar，setter/getter。

这是可以理解的。因为添加 Ivar 是需要在类 alloc 和 register 之间的，没了Ivar这个条件， setter/getter 方法也就不可能生成了。所以我们完成添加一个属性，还需要在创建类的时候添加它的Ivar, 并且继续添加 setter/getter方法（后续介绍添加方法）。

当然我们可以在代码中就声明它的Ivar，不过这有意义么？直接定义成属性不是更快。。哈哈。所以暂时没遇到需要添加属性的需求。



## **Adding Method**

添加一个对象方法就往 class 里添加，添加一个类方法就往 metaclass 里添加

```objective-c
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types);
```

假设RTModel类已经添加了一个  _property_03 实例变量，现在往RTModel类中添加它的setter/getter （格式要注意，参考IMP）

```objective-c
void imp_setter(id obj, SEL _cmd, NSArray *value) {
    Ivar ivar = class_getInstanceVariable([RTModel class], "_property_03");
    object_setIvarWithStrongDefault(obj, ivar, value);
}

NSArray *imp_getter(id obj, SEL _cmd) {
    Ivar ivar = class_getInstanceVariable([RTModel class], "_property_03");
    return object_getIvar(obj, ivar);
}

/// ...
/// adding property

SEL setter = sel_registerName("setProperty_03:");
SEL getter = sel_registerName("property_03");
    
class_addMethod(RTModel.class, setter, (IMP)imp_setter, "v@:@");
class_addMethod(RTModel.class, getter, (IMP)imp_getter, "@@:");
  
```

其中 imp_setter/imp_getter 通过C函数实现的，可以强转。当然也可以采用 imp_implementationWithBlock 生成一个。

> 以上都是关于 Class 中的方法，可以发现基本是以‘class_’为开头的。那么以此类比可以推出 Runtime 接口中 ‘object_’ 开头的则是关于实例对象的操作方法，要理解这些方法都不是难事，笔者偷懒，不在此赘述，否则，篇幅又臭又长。
>
> 只是大家使用方法的时候，看一下方法注释。。像 object_setIvarWithStrongDefault 和 object_setIvar 都用来设置变量值，区别仅在这是 strong 的还是 unsafe_unretained。



## 黑魔法 Method Swizzling

搜索 Runtime 黑魔法，可以搜出一大堆结果，可以说黑魔法并不是什么高升的概念了。其实，就是替换掉一个方法的实现。

```objective-c
void method_exchangeImplementations(Method m1, Method m2);
```

其常见应用就是替换掉系统的方法实现。比如这么个需求，想在每个Controller每次viewDidLoad执行的时候打印当前对象。可以在自定义的Controller的时候打印一下即可，但这工作不免繁琐，更方便的办法是将原来的viewDidLoad扩展一下实现。

在替换之前，发送消息给Controller 的 viewDidLoad，最终就会找到它的实现，假设是imp_viewDidLoad，然后执行。

![image-01](https://github.com/ICZhuang/Runtime/blob/master/image/05_01.png?raw=true)

增加一个方法__viewDidLoad，它的实现是imp___viewDidLoad，准备用来替换viewDidLoad的imp，那么当前状态是这样

![image-02](https://github.com/ICZhuang/Runtime/blob/master/image/05_02.png?raw=true)

最后替换

![image-03](https://github.com/ICZhuang/Runtime/blob/master/image/05_03.png?raw=true)

图虽简单，下面做一下说明。先上代码

```objective-c
@implementation NSObject (Additions)
+ (void)load {
    Method original = class_getInstanceMethod(self, @selector(viewDidLoad));
    Method final    = class_getInstanceMethod(self, @selector(__viewDidLoad));
    method_exchangeImplementations(original, final);
}
- (void)__viewDidLoad {
    NSLog(@"viewDidLoad <%@, %p>", self, self);
    [self __viewDidLoad];
}
@end
```

理解的关键点也就在`__viewDidLoad  `的 实现里，它里面又调用了`__viewDidLoad` ，但这不会造成循环递归调用。原因在于在 load 的时候就将 viewDidLoad 和 `__viewDidLoad` 的实现替换了。调用viewDidLoad方法时，会找它的实现，它当前的实现是 `imp__viewDidLoad` ，所以它会先打印对象，然后调用`__viewDidLoad`方法，但这时`__viewDidLoad`方法的实现是`imp_viewDidLoad`，也就是在Controller中viewDidLoad 的实现，即是`imp_viewDidLoad`

如此一来，也是顺序的执行，并没有形成递归调用。至于注意点和更严谨的使用方法，笔者就不吹了，就请参考这篇博客。[http://www.jianshu.com/p/ff19c04b34d0](http://www.jianshu.com/p/ff19c04b34d0)



## 扩展

上面说到一个关于KVO的函数，作为实践，这里有一篇关于KVO的实现原理，推荐大家阅读，里面是用到了Runtime，模拟KVO实现。[KVO的奥秘](http://sindrilin.com/ios-dev/2015/12/12/KVO的奥秘.html)