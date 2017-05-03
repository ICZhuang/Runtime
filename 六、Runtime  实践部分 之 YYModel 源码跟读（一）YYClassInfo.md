# **六、**Runtime 实践部分 之 YYModel 源码跟读（一）YYClassInfo

> 想着继续介绍Runtime 的概念部分 关于Protocol，Category，还有objc_xxx API，但又想到这跟之前介绍的都有相通的地方，所以就不再拉长篇幅了。其中就“关联对象”还需要提一下，然而网络博客已经一大堆了，有些东西，毕竟没有去实际应用过，不好做总结，就在真正用到Runtime再详细的补上吧。
>
> MJExtension 和 YYKit中的模型对象互转，是Runtime体现出目前最为广泛使用的方式之一。至于这里选择读YYKit没什么特殊原因，就是因为项目中使用的是YYKit，我想，看懂一个，在去看懂另外一个不是难事儿。。



## YYClassInfo

看头文件，就知道，这个就是在获取一个类中的各部分的信息。该头文件声明了四个类YYClassInfo、YYClassPropertyInfo、 YYClassIvarInfo、YYClassMethodInfo，它们就是对应着将objc_class、objc_property、objc_ivar、objc_method封装成OC对象。

YYClassInfo的核心方法就是_update方法，在这个方法里，遍历class对象的各个部分，将它们封装成了对象放进属性中。

笔者认为，在有了前面的基础，看懂这个方法在做什么绝不是难事，这里笔者只提一下本人认为的比较巧妙的方法。就是对类型编码的处理方法。

```objective-c
YYEncodingType YYEncodingGetType(const char *typeEncoding);
```

> 关于类型编码，可参考官方文档 
>
> ‘Objective-C Runtime Programming Guide > Type Encodings’
>
> ‘Objective-C Runtime Programming Guide > Declared Properties’
>
> 笔者译文 [Type Encoding & Declared Propery](https://github.com/ICZhuang/Runtime/blob/master/附、Type%20Encoding%20%26%20Declared%20Property.md)

对编码的处理，YY大神将其字符串表示的编码转成用枚举标识，而理解的关键点也就在于枚举的运算，就是枚举的 &、|  操作。另外有一点需要知晓的是使用过的Class info会被缓存的。



在 Type Encodings 文档中谈及的有 类型编码和类型描述符编码，而在 Declared Properties 文档中谈及的则有访问属性的attribute的编码。关于这三类编码都在头文件中 YYEncodingType 的枚举中对应找到，YYKit就是将 C 字符串的形式转成用枚举来表示类型编码。

```objective-c
/**
 Type encoding's type.
 */
typedef NS_OPTIONS(NSUInteger, YYEncodingType) {
    YYEncodingTypeMask       = 0xFF, ///< mask of type value
    YYEncodingTypeUnknown    = 0, ///< unknown
    YYEncodingTypeVoid       = 1, ///< void
    YYEncodingTypeBool       = 2, ///< bool
    YYEncodingTypeInt8       = 3, ///< char / BOOL
    YYEncodingTypeUInt8      = 4, ///< unsigned char
    YYEncodingTypeInt16      = 5, ///< short
    YYEncodingTypeUInt16     = 6, ///< unsigned short
    YYEncodingTypeInt32      = 7, ///< int
    YYEncodingTypeUInt32     = 8, ///< unsigned int
    YYEncodingTypeInt64      = 9, ///< long long
    YYEncodingTypeUInt64     = 10, ///< unsigned long long
    YYEncodingTypeFloat      = 11, ///< float
    YYEncodingTypeDouble     = 12, ///< double
    YYEncodingTypeLongDouble = 13, ///< long double
    YYEncodingTypeObject     = 14, ///< id
    YYEncodingTypeClass      = 15, ///< Class
    YYEncodingTypeSEL        = 16, ///< SEL
    YYEncodingTypeBlock      = 17, ///< block
    YYEncodingTypePointer    = 18, ///< void*
    YYEncodingTypeStruct     = 19, ///< struct
    YYEncodingTypeUnion      = 20, ///< union
    YYEncodingTypeCString    = 21, ///< char*
    YYEncodingTypeCArray     = 22, ///< char[10] (for example)
    
    YYEncodingTypeQualifierMask   = 0xFF00,   ///< mask of qualifier
    YYEncodingTypeQualifierConst  = 1 << 8,  ///< const
    YYEncodingTypeQualifierIn     = 1 << 9,  ///< in
    YYEncodingTypeQualifierInout  = 1 << 10, ///< inout
    YYEncodingTypeQualifierOut    = 1 << 11, ///< out
    YYEncodingTypeQualifierBycopy = 1 << 12, ///< bycopy
    YYEncodingTypeQualifierByref  = 1 << 13, ///< byref
    YYEncodingTypeQualifierOneway = 1 << 14, ///< oneway
    
    YYEncodingTypePropertyMask         = 0xFF0000, ///< mask of property
    YYEncodingTypePropertyReadonly     = 1 << 16, ///< readonly
    YYEncodingTypePropertyCopy         = 1 << 17, ///< copy
    YYEncodingTypePropertyRetain       = 1 << 18, ///< retain
    YYEncodingTypePropertyNonatomic    = 1 << 19, ///< nonatomic
    YYEncodingTypePropertyWeak         = 1 << 20, ///< weak
    YYEncodingTypePropertyCustomGetter = 1 << 21, ///< getter=
    YYEncodingTypePropertyCustomSetter = 1 << 22, ///< setter=
    YYEncodingTypePropertyDynamic      = 1 << 23, ///< @dynamic
};
```

很明显了，最上面YYEncodingTypexxx 是代表变量的类型编码，YYEncodingTypeQualifierxxx 是代表类型修饰符编码，YYEncodingTypePropertyxxx 则是代表属性的 attributes 的编码。其中属性的 attributes 的编码处理在YYPropertyInfo初始化的时候就已经处理了，YYEncodingGetType 这个方法只是用来处理类型的编码。