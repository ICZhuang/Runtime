# **七、**Runtime 实践部分之 YYModel 源码跟读（二）NSObject+YYModel

在YYKit中，模型字典的互转，以NSObject的分类来实现，这个分类就是NSObject+YYModel。

作为分类方法，任何继承自NSObject的 类都可以直接调用 +modelWithDictionary:  并将一个字典作为参数，快速创建一个包含与字典中匹配键值的属性的实例对象。

```objective-c
+ (instancetype)modelWithDictionary:(NSDictionary *)dictionary {
    if (!dictionary || dictionary == (id)kCFNull) return nil;
    if (![dictionary isKindOfClass:[NSDictionary class]]) return nil;
    /// 1.creat model meta
    Class cls = [self class];
    _YYModelMeta *modelMeta = [_YYModelMeta metaWithClass:cls];
    if (modelMeta->_hasCustomClassFromDictionary) {
        cls = [cls modelCustomClassForDictionary:dictionary] ?: cls;
    }
    /// 2.set value
    NSObject *one = [cls new];
    if ([one moelSetWithDictionary:dictionary]) return one;
    return nil;
}
```

这个方法分两个部分，一是创建一个_YYModelMeta实例对象，二是创建一个本类实例对象并根据字典对应赋值。

------



## _YYModelMeta

这是内部的一个私有类，顾名思义，这个类就是用来封装，更贴切的说是记录（或者引用）模型类中的一些数据的，与模型类是一一对应的。

目的就是为了后面对模型更快跟准确赋值用的。所以有一个缓存专门存储这个模型类的_YYModelMeta。（就称需要用到模型字段互转的类为模型类，后面也用这个称呼）。

> 开始笔者认为 _YYModelMeta 并不准确，meta 是指元数据，就是这个类是存字段那些信息的，所以笔者在纠结 _YYModelMeta 和  _YYClassMeta 哪个名字更准确，后来是笔者想岔了，这个类不就是存储model的meta数据么，class meta 岂不是类的元数据？类的元数据应该在元类(meta class)吧..  况且 _YYModelMeta  类同时会遍历并存储其父类的属性信息。。

```objective-c
@interface _YYModelMeta : NSObject {
    /// 该类的ClassInfo
    YYClassInfo *_classInfo;
    /// 用来索引
    /// 字典的键默认用属性名作为键, 否则如果用 +modelCustomPropertyMapper 映射的，则用映射后的做为键
    /// 字典的值是对应的_YYModelPropertyMeta实例对象
    NSDictionary *_mapper;
    /// 用来存储所有的_YYModelPropertyMeta实例对象
    NSArray *_allPropertyMetas;
    /// 用来存储需要用keyPath(xxx.xxx.xxx)索引的_YYModelPropertyMeta实例对象
    NSArray *_keyPathPropertyMetas;
    /// 用来存储有多个key索引的_YYModelPropertyMeta实例对象
    NSArray *_multiKeysPropertyMetas;
    /// 所有编入索引的_YYModelPropertyMeta实例对象的数量 _allPropertyMetas.count
    NSUInteger _keyMappedCount;
    /// 该类对象的NS类型，辨别该类是不是属于Foundation框架的子类
    YYEncodingNSType _nsType;
    /// 是否在转换之前对原dic做了中间处理
    BOOL _hasCustomWillTransformFromDictionary;
    /// 是否有定义要不要返回(或处理)dic转换之后的对象
    BOOL _hasCustomTransformFromDictionary;
    /// model转json中，是否有定义要不要处理转换后的对象
    BOOL _hasCustomTransformToDictionary;
    /// 是否有定义实际对象的类型，参考协议方法 +modelCustomClassForDictionary:
    /// 根据dictionary出现的不同字段值，判定究竟应该返回什么类型的对象
    BOOL _hasCustomClassFromDictionary;
}
```

_YYModelPropertyMeta 顾名思义，也可以知道它封装了property 的元数据，后续将介绍它的数据结构。可见YYModel 是核心是对属性的遍历和操作，并不涉及对 Ivar （成员变量）。

_YYModelMeta 通过类方法 +initWithClass: 创建对象。它里面是在对模型对象的YYClassInfo做工作。

------

### 1、白名单（white list）/ 黑名单（black list）

应该更准确的称呼它们，那就是属性白名单和属性黑名单，这名单是用来区分在转换过程中哪些属性是需要处理的和哪些是需要忽略的，那就是白则需要处理，黑则不需要处理。

```objective-c
// Get black list
NSSet *blacklist = nil;
if ([cls respondsToSelector:@selector(modelPropertyBlacklist)]) {
    NSArray *properties = [(id<YYModel>)cls modelPropertyBlacklist];
    if (properties) {
        blacklist = [NSSet setWithArray:properties];
    }
}
// Get white list
NSSet *whitelist = nil;
if ([cls respondsToSelector:@selector(modelPropertyWhitelist)]) {
    NSArray *properties = [(id<YYModel>)cls modelPropertyWhitelist];
    if (properties) {
        whitelist = [NSSet setWithArray:properties];
    }
}
```

可以通过实现 Protocol YYModel 的两个方法来提供这两个名单。（如果名单有交集，可以试试哪个优先）

```objective-c
+ (nullable NSArray<NSString *> *)modelPropertyBlacklist;
+ (nullable NSArray<NSString *> *)modelPropertyWhitelist;
```

> NSObject+YYModel.h中对方法都做了注释，虽说是英文，但都易懂。应该看看注释。



### 2、Container Property's Generic Class

同样根据 Protocol YYModel 提供的协议方法来定义容器类型属性中元素类型，捋顺了讲，就是说如果属性是容器类型，通过这个方法来映射该属性应该装什么类型的对象。使用方法参考它的注释。

```objective-c
+ (nullable NSDictionary<NSString *, id> *)modelContainerPropertyGenericClass;
```

协议方法的返回值是 NSDictionary<NSString *, id> *，但是内部会进行过滤不合格的 id。得出来的结果就是一个{key<property name> : value<class>} 形式的 NSDictionary。

```objective-c
NSDictionary *genericMapper = nil;
if ([cls respondsToSelector:@selector(modelContainerPropertyGenericClass)]) {
    genericMapper = [(id<YYModel>)cls modelContainerPropertyGenericClass];
    if (genericMapper) {
        NSMutableDictionary *tmp = [NSMutableDictionary new];
        [genericMapper enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
            if (![key isKindOfClass:[NSString class]]) return;
            Class meta = object_getClass(obj);
            if (!meta) return;
            if (class_isMetaClass(meta)) { /// Means that 'obj' is the 'Class' type
                tmp[key] = obj; /// The value is still 'obj'
            } else if ([obj isKindOfClass:[NSString class]]) {
                Class cls = NSClassFromString(obj); /// Get 'Class' if 'obj' is a NSString
                if (cls) {
                    tmp[key] = cls;
                }
            }
        }];
        genericMapper = tmp;
    }
}
```

> 在声明容器属性的时候，加上泛型并不能起到规范类型效果。它只是在使用该属性时比较方便。



### 3、Property Metas

循环向上追溯遍历类与父类（除了NSObject/NSProxy）的各个属性，利用类中的property info 对象创建_YYModelPropertyMeta，如果不存在setter或者getter方法，则跳过。如果子父类有相通名字的属性，采用子类的_YYModelPropertyMeta。

```objective-c
NSMutableDictionary *allPropertyMetas = [NSMutableDictionary new];
YYClassInfo *curClassInfo = classInfo;
// recursive parse super class, but ignore root class (NSObject/NSProxy)
while (curClassInfo && curClassInfo.superCls != nil) {   
    for (YYClassPropertyInfo *propertyInfo in curClassInfo.propertyInfos.allValues) {
        if (!propertyInfo.name) continue;
      
        if (blacklist && [blacklist containsObject:propertyInfo.name]) continue;
        if (whitelist && ![whitelist containsObject:propertyInfo.name]) continue;
      
        _YYModelPropertyMeta *meta = [_YYModelPropertyMeta 
                                      metaWithClassInfo:classInfo
                                      propertyInfo:propertyInfo
                                      generic:genericMapper[propertyInfo.name]];
      
        if (!meta || !meta->_name) continue;
        if (!meta->_getter || !meta->_setter) continue;
        if (allPropertyMetas[meta->_name]) continue;
        allPropertyMetas[meta->_name] = meta;
    }
    curClassInfo = curClassInfo.superClassInfo;
}
if (allPropertyMetas.count) _allPropertyMetas = allPropertyMetas.allValues.copy;
```



### 4、Mapper

根据对 _YYModelMeta 的 _mapper 字段描述，先需检查是否有通过 +modelCustomPropertyMapper 协议方法来返回属性对应在 NSDictionary 的键，如果有，则先要把这些 _YYModelPropertyMeta 加入 mapper 中。

> 这里可能要结合 _YYModelPropertyMeta 一节看，因为涉及到了它里面的字段设置了

```objective-c
NSMutableDictionary *mapper = [NSMutableDictionary new];
NSMutableArray *keyPathPropertyMetas = [NSMutableArray new];
NSMutableArray *multiKeysPropertyMetas = [NSMutableArray new];

if ([cls respondsToSelector:@selector(modelCustomPropertyMapper)]) {
    NSDictionary *customMapper = [(id <YYModel>)cls modelCustomPropertyMapper];
    [customMapper enumerateKeysAndObjectsUsingBlock:^(NSString *propertyName,
                                                      NSString *mappedToKey,
                                                      BOOL *stop) {
        _YYModelPropertyMeta *propertyMeta = allPropertyMetas[propertyName];
        if (!propertyMeta) return;
      
      /// 如果属性有被自定义映射，则将它从allPropertyMetas移除
      /// 至于剩余的那些会在后面循坏形式统一加到mapper中
        [allPropertyMetas removeObjectForKey:propertyName];
      
        if ([mappedToKey isKindOfClass:[NSString class]]) {
            if (mappedToKey.length == 0) return;
			/// 设置它的_mappedToKey
            propertyMeta->_mappedToKey = mappedToKey; 
            /// 看它的 mapper key是不是keyPath(xxx.xxx.xxx)
            NSArray *keyPath = [mappedToKey componentsSeparatedByString:@"."];
            for (NSString *onePath in keyPath) {
                if (onePath.length == 0) {
                    NSMutableArray *tmp = keyPath.mutableCopy;
                    [tmp removeObject:@""];
                    keyPath = tmp;
                    break;
                }
            }
            if (keyPath.count > 1) {
              /// 是keyPath，则把property meta 加进keyPathPropertyMetas中
                propertyMeta->_mappedToKeyPath = keyPath;
                [keyPathPropertyMetas addObject:propertyMeta];
            }
           /// 众所周知，NSDictionary 不可能存在同样的 key，如果多个属性同时映射一个 key
           /// 这个时候就将前面的那个 meta 作为后面的 met a的 next，就如同链表一样。
            propertyMeta->_next = mapper[mappedToKey] ?: nil;
           /// 将 property meta 存进mapper中
            mapper[mappedToKey] = propertyMeta;

        } else if ([mappedToKey isKindOfClass:[NSArray class]]) {
			/// mappedToKey 也可能是数组(参考 +modelCustomPropertyMapper 用法)
          	/// 代表多个一个属性映射多个 key
            NSMutableArray *mappedToKeyArray = [NSMutableArray new];
            for (NSString *oneKey in ((NSArray *)mappedToKey)) {
                if (![oneKey isKindOfClass:[NSString class]]) continue;
                if (oneKey.length == 0) continue;

                NSArray *keyPath = [oneKey componentsSeparatedByString:@"."];
                if (keyPath.count > 1) {
                    /// 如果是keyPath，将分割后的数组存进mappedToKeyArray
                    [mappedToKeyArray addObject:keyPath];
                } else {
                    /// 如果只是key，则直接把key存进mappedToKeyArray
                    [mappedToKeyArray addObject:oneKey];
                }
				/// 存储它的 key 和 keyPath
                if (!propertyMeta->_mappedToKey) {
                    propertyMeta->_mappedToKey = oneKey;
                    propertyMeta->_mappedToKeyPath = keyPath.count > 1 ? keyPath : nil;
                }
            }
            if (!propertyMeta->_mappedToKey) return;
			/// 设置 property meta 的mappedToKeyArray
            propertyMeta->_mappedToKeyArray = mappedToKeyArray;
            [multiKeysPropertyMetas addObject:propertyMeta];
			
            propertyMeta->_next = mapper[mappedToKey] ?: nil;
          	/// 将 property meta 存进mapper中
            mapper[mappedToKey] = propertyMeta;
        }
    }];
}
/// 遍历那些没有经过自定义映射的，设置它们的字段，并加入mapper中
[allPropertyMetas enumerateKeysAndObjectsUsingBlock:^(NSString *name, 
                                                      _YYModelPropertyMeta *propertyMeta, 
                                                      BOOL *stop) {
    propertyMeta->_mappedToKey = name;
    propertyMeta->_next = mapper[name] ?: nil;
    mapper[name] = propertyMeta;
}];
```



### 5、_hasCustomClassFromDictionary

Protocol YYModel中 +modelCustomClassForDictionary: 设计巧妙，可以由开发者根据开发者通过dictionary出现的不同字段值，判定字典转模型的结果应该返回的是什么类型的对象。通常出现在父子类关系中。具体参见该方法注释。

```objective-c
_hasCustomClassFromDictionary = 
([cls respondsToSelector:@selector(modelCustomClassForDictionary:)]);
```

------



## _YYModelPropertyMeta

这也是内部一个私有类，封装的一个属性的 meta 数据。与属性也是一一对应的。

```objective-c
@interface _YYModelPropertyMeta : NSObject {
    /// 属性名
    NSString *_name;    
    /// 属性类型
    YYEncodingType _type;   
    /// 属性类型(Foundation框架中的类型)
    YYEncodingNSType _nsType;   
    /// 是否为C基本数据类型
    BOOL _isCNumber;             
    /// 属性自己的Class
    Class _cls;                  
    /// 如果该属性是容器类型，_genericCls 则是元素的类型
    Class _genericCls;           
    /// getter 方法
    SEL _getter;  
    /// setter 方法
    SEL _setter;  
    /// 是否支持KVC访问
    BOOL _isKVCCompatible;      
    /// 如果是 struct 类型，是否支持解档归档
    BOOL _isStructAvailableForKeyedArchiver; 
    /// 是否有定义实际对象的类型
    BOOL _hasCustomClassFromDictionary; 
    /// +modelCustomPropertyMapper 映射的key
    NSString *_mappedToKey;      
    /// 如果 +modelCustomPropertyMapper 映射的是keyPath(xx.xx)，以‘.’分割后存储
    NSArray *_mappedToKeyPath;   
    /// 如果 +modelCustomPropertyMapper 映射的是多个key或keyPath，都存储在_mappedToKeyArray中，
    /// 这个时候_mappedToKey存储的是第一个元素。
    /// 元素可能同时又 NSString 和 NSArray
    NSArray *_mappedToKeyArray; 
    /// 对应的 property info
    YYClassPropertyInfo *_info;  
    /// 如果多个属性映射同一个Key, 将以链表的形式将它们以next的形式链接起来
    _YYModelPropertyMeta *_next; 
}
```

_YYModelPropertyMeta 通过类方法创建，其中 generic 则是容器属性中元素的类型

```
+ (instancetype)metaWithClassInfo:propertyInfo:generic:
```



### 1、 _mappedToKey/ _mappedToKeyPath/ _mappedToKeyArray

默认字典中的key与类中属性名一一对应，就是说 _mappedToKey 默认是属性名。 但如果字典中的key找不到与之对应的属性名，那么可以通过实现+modelCustomPropertyMapper方法提供属性与字典key之间的映射，让属性能都找到真正的值。

该方法返回的是NSDictionary<NSString *, id> *，这里 id 可以是 NSString 和 NSArray 类型，其它的类型会没过滤掉。但就这两种类型，组合也可以是多种的。代码参考 _YYModelMeta > mapper creation

- 如果是key (xxx) 形式的 NSString，那么   _mappedToKey 就是 id， 这时其它两者皆为nil；
- 如果是keyPath (xxx.xxx.xxx) 形式的NSString，那么 _mappedToKey 值是 id， _mappedToKeyPath 则是 ‘.’ 分割后的数组， _mappedToKeyArray 为nil；
- 如果是数组(A)，那么 _mappedToKey 是数组的第一个元素， 如果第一个元素是第二种形式的NSString，那么 _mappedToKeyPath 为其通过 ‘.’ 分割后的结果， _mappedToKeyArray 则比较复杂，它的元素可能是NSString 或者是NSArray，遍历数组(A)，如果是keyPath形式的， 那么存进去的就是分割后的数组，如果是key 形式的直接存储。

### 2、_hasCustomClassFromDictionary

它的判定方法如下所示。如果存在generic，说明该属性是容器类型，这个字段值应该由元素的类对象判定；如果没有generic则不是容器类型，并且该属性如果是自定义类型，那么根据自身类对象判定。

```objective-c
if (generic) {
    meta->_hasCustomClassFromDictionary = 
    [generic respondsToSelector:@selector(modelCustomClassForDictionary:)];
} else if (meta->_cls && meta->_nsType == YYEncodingTypeNSUnknown) {
    meta->_hasCustomClassFromDictionary = 
    [meta->_cls respondsToSelector:@selector(modelCustomClassForDictionary:)];
}
```



## 结语

承文章开头，本文介绍的是字典转模型步骤中的第一部分，就是创建 _YYModelMeta，后半部分就是拿着这些元数据和NSDictionary去分别给属性赋值。NSObject+YYModel通篇几近两千行代码，要都往博客上粘，任重道远呐，笔者还是不干这么愚蠢的事儿了。关于后半部分，笔者想往后有时间再继续往深了读吧。

况且，笔者初衷是想对前面 Runtime 的文章做一些实践的东西，就想到拿YYModel源码做个示例。后半部分还涉及到很多东西，比如通过字符串获得时间，各类时间格式YY大神都写的很全面，笔者不认为不去查阅相关资料就可以有很好的认识。