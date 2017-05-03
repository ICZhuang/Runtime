# **二、**Runtime概念部分 之 Property

## Property

从代码中看，它就是一个带着property attribute和自带访问器的Ivar，类似于Ivar，Property 在Runtime中的表示则为 objc_property_t，它是一个 objc_property 结构体的指针(笔者没找到它的定义，后续找到后补上，不过各位看官可以类比Ivar的访问方法，可以猜测 objc_property `可能`长什么样)。

诸如 nonatomic/readonly/assign/strong/weak 等这些则为property attribute。（两个单词皆有属性的意思，如果写成 属性的属性 不免拗口，在这直接写上英文原文，下面将它们述为 attribute。）

那么属性（property）带有的信息 有attributes、type、name，当然同时还有setter或者getter。



## 获取属性

同样，通过class_copyPropertyList可获取类中的属性列表

```objective-c
objc_property_t *class_copyPropertyList(Class cls, unsigned int *outCount);
```

或者直接通过属性名获得对应的属性

```objective-c
objc_property_t class_getProperty(Class cls, const char *name);
```



## 访问属性

对于属性的结构体objc_property_t，同样有对应的相关字段（或者成为信息）的操作方法

1、获取name

```objective-c
const char *property_getName(objc_property_t property);
```

2、取属性的type encoding

```objective-c
const char *property_getAttributes(objc_property_t property);
```

3、上面方法是把 attributes 以字符串返回，通过下面方法可以取得 attribute 列表，进而通过遍历可以取得每个attribute的信息

```objective-c
objc_property_attribute_t *property_copyAttributeList(objc_property_t property, unsigned int *outCount);
```

这里涉及到另一个结构体objc_property_attribute_t，每个 attribute 通过结构体 objc_property_attribute_t 定义

```objective-c
typedef struct {
    const char *name;           /**< The name of the attribute */
    const char *value;          /**< The value of the attribute (usually empty) */
} objc_property_attribute_t;
```

4、通过指定一个attribute的name，获得对应的value

```objective-c
char *property_copyAttributeValue(objc_property_t property, const char *attributeName);
```

> 相当于获得一个attribute，然后直接按结构体访问方法访问它的字段值



## 遍历

```objective-c
- (void)getProperties {
    /// get properties 
    unsigned int count = 0;
    objc_property_t *properties = class_copyPropertyList(RTModel.class, &count);
    
    objc_property_t property;
    for (int index = 0; index < count; ++index) {
        property = properties[index];
        
        printf("%s - %s\n", property_getName(property), property_getAttributes(property));
        /// get attribute's name and it's value
        unsigned int out_count = 0;
        objc_property_attribute_t *list = property_copyAttributeList(property,&out_count);
        
        objc_property_attribute_t attribute;
        for (int i = 0; i < out_count; ++i) {
            attribute = list[i];
            
            printf("%s - %s\n", attribute.name, attribute.value);
        }
    }
}
```



## Type Encoding

属性的type encoding，除了变量类型的，还应带上各个attribute的编码，可以说属性的类型是由这两者组成。比如RTModel的属性的type encodings如下 

![image-01](https://github.com/ICZhuang/Runtime/blob/master/image/02_01.png?raw=true)

> 关于property attribute的编码详细可参阅Objective-C Runtime Programming Guide > Declared Properties
>



另外通过property_02的获得attributes列表， attribute.name 和 attribute.value打印如下。如‘T’就是attribute的name字段，对应的value是‘NSArray’

![image-02](https://github.com/ICZhuang/Runtime/blob/master/image/02_02.png?raw=true)