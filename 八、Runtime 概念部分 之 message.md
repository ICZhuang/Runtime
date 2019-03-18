# 八、Runtime 概念部分 之 message

> 本位github地址 [https://github.com/ICZhuang/Runtime](https://github.com/ICZhuang/Runtime)

我们通常定义一个类的时候，会直接将方法写到类中，给予一种调用改对象方法就一定会找到该对象的方法实现的错觉。其实不然。我们说调用一个对象的某一个对象方法，其实更准确的描述应该是向对象发送一个消息，比如[receiver message]，应该描述成像receiver这个对象发送一个message消息。



## objc_msgSend

在Objective-C语言中，消息并没有跟方法的实现绑定的，直到运行状态下，才知道响应消息时应该执行的方法。比如编译器会将下面表达式

```objective-c
[receiver message]
```

转换成消息发送方法 objc_msgSend 方式。这个方法将消息接受者和方法selector作为两个最重要的参数

```objective-c
objc_msgSend(receiver, selector)
```

任何消息传递过来的参数都会以可变形参的形式拼接在selector后面

```objective-c
objc_msgSend(receiver, selector, arg1, arg2, ...)
```

在消息发送方法中，就是在做动态绑定

1. 第一步查找selector的方法实现，相同的selector在不同的类有不同的实现，但它会在 receiver的class中找；
2. 然后执行程序，将receiver和其它参数作为实参传递过去；
3. 最后将程序的返回值作为自己的返回值返回

查找的key就存在类的结构中。类结构中有两个重要的元素：

- superclass 指针
- 分发表(dispatch table)，这个分发表将selector和方法的实现地址关联了起来

当一个对象被创建，分配内存，并且给成员变量做初始化。第一个变量就是isa指针，指向它的class。通过class便可找到所有在继承链中的类。

![image_01](https://github.com/ICZhuang/Runtime/blob/master/image/08-01.png?raw=false){:height="100"}

当给一个对象发送消息，消息发送方法就会沿着对象的isa指针在类的分发表里找方法的selector，如果在类中没有找到，那么继续在superclass的分发表里找，直到查找到NSObject class。一旦查找到，消息发送方法就调起该方法实现并把对象的数据结构传递过去。

为了提高查找速度，Runtime 会缓存那些使用过的方法。每个class都有一个cache，它可以缓存继承而来的方法和自身定义的方法。在从分发表里查找之前，消息查找机制会先从cache里查找。这个缓存会动态的增长以存放新的消息指定执行过的方法。



### 隐式参数

当objc_msgSend找到了方法的实现，它就会调起实现，并把消息传递的参数全部传递过去，其中包括两个默认就会传递的参数

- 消息接收对象
- method对应的selector

通常在写Objective-C代码的时候，通常会省去这两个参数，当被编译的时候，参会插入到方法实现中。虽然在编码时不显示声明，但我们仍然可以应用到它们。self 则为消息接收对象，  _cmd 则为消息的 selector。比如下面示例，_cmd 是strange的selector，self 则是接收strange消息的接受者

```objective-c
- strange {
    id  target = getTheReceiver();
    SEL method = getTheMethod();
 
    if ( target == self || method == _cmd )
        return nil;
    return [target performSelector:method];
}
```



### 方法地址

唯一可以避开动态绑定的方法就是获得方法(实现)的地址，直接调用。一个方法被多次连续调用，而又想避开每次都要查找的消耗，可以用这种方法避开动态绑定。

通过NSObject class中声明的 methodForSelector: 可以获得方法的地址(IMP)，就是一个函数指针。通过函数指针唤起它的实现。但是得注意函数指针与IMP的转换，返回值和参数都需要包含到

```objective-c
void (*setter)(id, SEL, BOOL);
int i;
 
setter = (void (*)(id, SEL, BOOL))[target methodForSelector:@selector(setFilled:)];
for ( i = 0 ; i < 1000 ; i++ )
    setter(targetList[i], @selector(setFilled:), YES);
```

前面两个参数是用来接收 self 和 _cmd 的，转成函数形式需要加上。 

注意 methodForSelector: 是 Cocoa runtime 提供的，并非Objective-C语言提供的。

------

给一个对象发送消息后，如果对应的方法没找到，则进入下面步骤：

1. 动态方法解析：唤起对象的resolveInstanceMethod: 或者 resolveClassMethod: ，询问是否能够处理消息的selector，如果不能则进入下一步；
2. 备援对象：唤起对象的forwardingTargetForSelector:，返回能够处理该selector的对象，如果返回nil或者self，则进入下一步；
3. 消息转发：实现forwardInvocation:将消息转发给其它能处理该消息的对象，该方法的参数会通过methodSignatureForSelector:创建，所以还应该实现methodSignatureForSelector:方法


> 参考：[理解消息传递机制](https://www.zybuluo.com/MicroCai/note/64270)  [[Objective-C 消息发送与转发机制原理](http://blog.csdn.net/wangweijjj/article/details/51888750)]



## 动态方法解析

有些情况你可能希望动态的提供方法实现。比如声明了一个动态属性

```objective-c
@dynamic propertyName;
```

这说明与改属性相关的方法(setter&getter)要动态添加

可以实现 resolveInstanceMethod: 和 resolveClassMethod: 动态的为对象或者类的selector添加实现。

一个Objective-C的方法其实在底层就是一个简单的C函数，这函数带着至少两个参数，self 和 _cmd。通过class_addMethod添加一个函数作为类的方法。因此，先添加一个函数

```c
void dynamicMethodIMP(id self, SEL _cmd) {
    // implementation ....
}
```

然后在 resolveInstanceMethod: 中动态的将它作为方法(如resolveThisMethodDynamically)添加到class中

```objective-c
@implementation MyClass
+ (BOOL)resolveInstanceMethod:(SEL)aSEL
{
    if (aSEL == @selector(resolveThisMethodDynamically)) {
          class_addMethod([self class], aSEL, (IMP) dynamicMethodIMP, "v@:");
          return YES;
    }
    return [super resolveInstanceMethod:aSEL];
}
@end
```

消息转发和动态方法解析是顺序的，在进入消息转发机制之前，可以通过动态方法解析来处理方法。如果respondsToSelector: 和 instancesRespondToSelector: 被唤起，通过动态方法处理为selector提供一个IMP。如果实现了resolveInstanceMethod: 但又想让selector进入消息转发机制，那么在遇到该selector时返回NO。

经试验，如果实现了resolveInstanceMethod: 并且在里面往类添加了selector关联的方法实现，则不会再走备援对象和消息转发机制。



## 消息转发

在出现崩溃之前，runtime 给对象发送 forwardInvocation: 消息，并将一个NSInvocation对象作为唯一的参数传递过去，这个NSInvocation对象囊括了最初的消息和参数。

你可以实现 forwardInvocation: 方法提供对消息的默认处理，或者过滤消息避免出现错误。顾名思义，方法forwardInvocation:通常是用来将转发消息给另外的对象。

为了了解消息转发的意图，想象这样一种场景：假设需要设计一个对象可以响应negotiate消息，并且希望它的响应期间可以带动另外一种对象响应negotiate消息。完成这需求很简单，可以在你的negotiate方法实现中将消息传递给另外一个对象。

进一步假设，你设计的对象对negotiate的响应实际上其实就是执行另外一个类的实现，其中一种方法就是让你的类继承自该类。然而，这可能并不合理。原因在于，你的类和实现了negotiate的类可能在不同的继承树分支。就是说它们可能不存在直接或间接的继承关系。

即使你不能继承negotiate方法，你可以简单地实现negotiate，将negotiate消息传递给另外一个对象。如此一来你的对象就响应了negotiate，就像是从其它对象"借用"到了negotiate。

```objective-c
- (id)negotiate {
    if ( [someOtherObject respondsTo:@selector(negotiate)] )
        return [someOtherObject negotiate];
    return self;
}
```

这种方法有些笨拙，特别是当有多种消息需要传递给其它对象的情况。你需要在一个方法中覆盖所有其它对象应该响应的方法。此外，在写代码的时候，有一些消息的转发是不可预见的。这些消息是取决于runtime，它们可能被新方法替代，或被新类重新实现。

forwardInvocation: 提供的是一种动态解决方案。它工作的原理就想这样：当一个对象没有一个方法可以匹配消息中的selector，这个对象是不能响应该消息的，这个时候，runtime 会给对象发送一个 forwardInvocation: 消息。所有的对象都从NSObject类中继承了改方法。然而，NSObject中的该方法只是简单的调起doesNotRecognizeSelector: 。通过重写 forwardInvocation: 方法，将消息转发给另外的对象。

转发一个消息，需要在forwardInvocation:方法中：

- 判断消息应该转发给谁，还有
- 转发的时候带上它原始的参数

消息可以通过 invokeWithTarget: 方法发送：

```objective-c
- (void)forwardInvocation:(NSInvocation *)anInvocation {
    if ([someOtherObject respondsToSelector:[anInvocation selector]])
        [anInvocation invokeWithTarget:someOtherObject];
    else
        [super forwardInvocation:anInvocation];
}
```

如果有返回值，则返回值会返给最初的发送者，包括任何类型 ids, structures, and double-precision floating-point numbers.

forwardInvocation: 方法就像是未知消息的分发中心，将它们投递到指定的接受者。由或者是中转站，将消息发送给同一目标。它可以将一个消息转换成另一个消息，或者简单的就把消息"吞下"，让消息得不到响应。forwardInvocation: 所干的事儿就看它是怎么被实现的。

> 注意：forwardInvocation: 只有在消息的接受者不能够响应的情况下才会去处理该消息。举例来说，如果希望一个对象转发negotiate消息给其它的对象，那么这个对象不能有negotiate方法，如果有，消息永远也不可能被forwardInvocation:处理。



## 转发和多继承

如下图，一个Warrior实例对象转发一个negotiate消息给一个Diplomat实例对象。Warrior就好像Diplomat一样能响应negotiate消息

![image-02](https://github.com/ICZhuang/Runtime/blob/master/image/08-02.png?raw=true)

这样看来，转发消息的对象就"继承"了来自两个继承树分支中的方法 —  本类所在继承分支和消息真正响应的对象所在的分支。按白话说，转发消息的对象就想继承了多个对象，2️而这些对象不存在直接或间接的继承关系。

消息转发提供了多继承中的许多特性。但是，他们两这有一点不同的是：多继承是将多个不同的行为合并到一个对象里面。它趋向更大，更丰富的对象的定义。相反，转发趋向将任务分割派给不同的对象。它将问题分解成不同的对象，又将这些对象以一种方式关联起来，但这种方式对消息发送者来说是透明的。



### 转发和继承

虽说转发很像继承，但NSObject class却能清楚的区分它们。像respondsToSelector:和 isKindOfClass: 这样的方法只会是从继承树种查找，并不会在转发链中查找。举例来说，一个Warrior object想知道自己是否响应negotiate消息

```objective-c
if ( [aWarrior respondsToSelector:@selector(negotiate)] )
    ...
```

答案是NO，即使它能接收negotiate消息，某种意义上，它只是转发消息给Diplomat object。

除非，有这样一种情况，你希望它表现出来的是仿佛继承了消息转发的目的对象一样，你需要重新实现respondsToSelector: 和 isKindOfClass: 

```objective-c
- (BOOL)respondsToSelector:(SEL)aSelector {
    if ( [super respondsToSelector:aSelector] )
        return YES;
    else {
      /// 这里，检测 aSelector 消息是否可以被转发到另一个对象，或者另一个对象是否
      /// 是否可以响应 aSelector 消息， 如果可以retrun YES
    }
    return NO;
}
```

如果一个对象可以转发消息给它的代理对象，你需要像下面一样实现methodSignatureForSelector:方法

```objective-c
- (NSMethodSignature*)methodSignatureForSelector:(SEL)selector {
    NSMethodSignature* signature = [super methodSignatureForSelector:selector];
    if (!signature) {
       signature = [surrogate methodSignatureForSelector:selector];
    }
    return signature;
}
```
