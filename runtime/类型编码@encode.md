#Class

 ```
 typedef struct objc_class *Class;
 ```
 这是rumtime源码中的<objc-private.h>中的代码，我们可以看到objc_class是Class的别名。接下来我们看Class的定义，即是objc_class:

 ```
 struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;   //父类
    const char *name                                         OBJC2_UNAVAILABLE;   //类名
    long version                                             OBJC2_UNAVAILABLE;   //类的版本信息，默认为0
    long info                                                OBJC2_UNAVAILABLE;   //类信息,供运行期使用的一些位标识
    long instance_size                                       OBJC2_UNAVAILABLE;   //类的实例变量大小
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;   //类的成员变量链表
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;   //类的方法定义的链表
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;   //类的方法缓存
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;   //类的协议链表
#endif

} OBJC2_UNAVAILABLE;
/* Use `Class` instead of `struct objc_class *` */
 ```
 `注意：OBJC2_UNAVAILABLE是一个Apple对Objc系统运行版本进行约束的宏定义，主要为了兼容非Objective-C 2.0的遗留版本。`
 
* isa表示一个Class对象的Class，就是元类。Objective-C中Class本身也是一个对象。我们可以在objc-runtime-new.h和objc-runtime-old.h文件找到证据(不明白这两个类有啥用)，发现objc_class有以下的定义:


```
//objc-runtime-new.h
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;
    ......  
} 
```
```
//objc-runtime-old.h
struct objc_class : objc_object {
    Class superclass;
    const char *name;
    uint32_t version;
    uint32_t info;
    uint32_t instance_size;
    struct old_ivar_list *ivars;
    struct old_method_list **methodLists;
    ......
}
```
由此可见，结构体objc_class也是继承objc_object，说明Class在设计中本身也是一个对象。其实Meta Class也是一个Class，那么它也跟其他Class一样有自己的isa和super_class指针，关系如下这张经典图：
![Xcode build setting](/Users/maybe/Blog/Blog/runtime/isa.jpg)

上图实线是super_class指针，虚线是isa指针。有几个关键点需要解释以下：

* Root class (class)其实就是NSObject，NSObject是没有超类的，所以Root class(class)的superclass指向nil。
* 每个Class都有一个isa指针指向唯一的Meta class
* Root class(meta)的superclass指向Root class(class)，也就是NSObject，形成一个回路。
每个Meta class的isa指针都指向Root class (meta)。

_实例对象的isa指针指向类，类的isa指针指向其元类（metaClass）。实例对象就是一个含isa指针的结构体。类存储实例对象的方法列表，元类存储类的方法列表，元类也是类对象。_

# 类型编码@encode
编译器将每个方法的返回值和参数类型编码为一个字符串，并将其与方法的selector关联在一起。可以使用@encode编译器指令来获取它。`@encode`使用起来很简单，通过传入一个类型,我们就可以获取代表这个类型的编码C字符串：

```
char *intType = @encode(NSInteger);
char *arrayType = @encode(NSArray);
NSLog(@"intType = %s, arrayType = %s",intType,arrayType); 
// intType = q, arrayType = {NSArray=#}
```
这个关键字最常用于Objective-c的runtime机制中。在传递方法签名参数时候，传递类型编码。

#Ivar
Ivar是类中的实例变量，是一个指向objc_ivar的结构体指针，包含了变量名，变量类型的信息。

```
/// An opaque type that represents an instance variable.
typedef struct objc_ivar *Ivar;
struct objc_ivar {
    char *ivar_name ;
    char *ivar_type ;
    int ivar_offset ;
#ifdef __LP64__
    int space ;
#endif
}
```
objc_ivar_list其实就是一个链表，存储多个objc_ivar，而objc_ivar结构体存储类的单个成员变量信息。

```
struct objc_ivar_list {
    int ivar_count;
#ifdef __LP64__
    int space;
#endif
    /* variable length structure */
    struct objc_ivar ivar_list[1];
} 
```

#SEL
SEL又叫选择器，是表示一个方法的selector的指针,映射方法的名字。Objective-C在编译时，会依据每一个方法的名字、参数序列，生成一个唯一的整型标识(Int类型的地址)，这个标识就是SEL。
`SEL的作用是作为IMP的KEY`，存储在NSSet中，便于hash快速查询方法。SEL不能相同，对应方法可以不同。所以在Objective-C同一个类(及类的继承体系)中，不能存在2个同名的方法，就算参数类型不同。多个方法可以有同一个SEL。
不同的类可以有相同的方法名。不同类的实例对象执行相同的selector时，会在各自的方法列表中去根据selector去寻找自己对应的IMP。

```
typedef struct objc_selector *SEL;
SEL selector = @selector(buttonClicked);
NSLog(@"SEL = %s",selector);
```
无论是`<objc/runtime.h>`还是`objc_selector`方法在runTime源码中都未曾找到，不过打印知道本质还是字符串。

#IMP
IMP就是Implementation的缩写，顾名思义，它是指向一个方法实现的指针，每一个方法都有一个对应的IMP,IMP是指向实现函数的指针，通过SEL取得IMP后，我们就获得了最终要找的实现函数的入口，即方法实现的起始地址。

当你向某个对象发送一条信息，可以由这个函数指针来指定方法的实现，它最终就会执行那段代码，这样可以绕开消息传递阶段而立即执行。

```
/// A pointer to the function of a method implementation. 
#if !OBJC_OLD_DISPATCH_PROTOTYPES
typedef void (*IMP)(void /* id, SEL, ... */ ); 
#else
typedef id (*IMP)(id, SEL, ...); 
#endif
```
在默认情况下你的工程是打开这个配置的，IMP被定义为有参数有返回值的函数。
![Xcode build setting](/Users/maybe/Blog/Blog/runtime/Xcode_build_setting.png)
在build setting中打开后,IMP被定义为无参数无返回值的函数。

#Method

```
typedef struct objc_method *Method
struct objc_method {
    SEL method_name;
    char *method_types;
    IMP method_imp;
}
```
Method就是一个指向objc_method结构体指针,它存储了方法名(method_name)、方法类型(method_types)和方法实现(method_imp)。在运行时，这个结构体在SEL和IMP之间作了一个绑定。这样有了SEL，我们便可以找到对应的IMP，从而调用方法的实现代码。

在Objective-c类的定义中有一个`objc_method_list`,可以看出方法列表是一个链表，其实objc_method变量存储了某个类的方法信息。我们知道Category可以添加成员方法，Category确不能添加属性的。结合上面的`objc_ivar_list`可以印证这个事实。

```
struct objc_method_list {
    struct objc_method_list *obsolete          
    int method_count                                         
#ifdef __LP64__
    int space                                                
#endif
    /* variable length structure */
    struct objc_method method_list[1]                        
} 
```

#Cache

方法调用最先是在方法缓存里找的，方法调用是懒调用，第一次调用时加载后加到缓存池里。一个objc程序启动后，需要进行类的初始化、调用方法时的cache初始化，再发送消息的时候就直接走缓存。
Objective-C运行时函数存储指向最近被调用的类的方法的定义一个objc_cache数据结构。

Cache一个存储Method的链表，主要是为了优化方法调用的性能。如果没有Cache，当对象receiver调用方法message时，首先根据对象receiver的isa指针查找到它对应的类，然后在类的methodLists中搜索方法，如果没有找到，就使用super_class指针到父类中的methodLists查找，一旦找到就调用方法。如果没有找到，有可能消息转发，也可能忽略它。但这样查找方式效率太低，因为往往一个类大概只有20%的方法经常被调用，占总调用次数的80%。所以使用Cache来缓存经常调用的方法，当调用方法时，优先在Cache查找，如果没有找到，再到methodLists查找。

```
typedef struct objc_cache *Cache
struct objc_cache {
    unsigned int mask /* total = mask + 1 */;
    unsigned int occupied; //Objective-C运行时使用此字段来确定开始对数据桶数组进行线性搜索的索引,这是一个简单的哈希算法。
    Method buckets[1];
};
```



