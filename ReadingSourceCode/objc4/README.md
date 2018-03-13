关于 objc4 源码的一些说明：     

- objc4 的源码不能直接编译，需要配置相关环境才能运行。可以在[这里](https://github.com/RetVal/objc-runtime)下载可调式的源码。
- objc 运行时源码的入口在 `void _objc_init(void)` 函数。


### 1. Objective-C 对象是什么？Class 是什么？id 又是什么？

所有的类都继承 NSObject 或者 NSProxy，先来看看这两个类在各自的公开头文件中的定义：

```
@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}
```

```
@interface NSProxy <NSObject> {
    Class	isa;
}
```

在 objc.h 文件中，对于 Class，id 以及 objc_object 的定义：

```
/// An opaque type that represents an Objective-C class.
typedef struct objc_class *Class;

/// Represents an instance of a class.
struct objc_object {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};

/// A pointer to an instance of a class.
typedef struct objc_object *id;

```

runtime.h 文件中对 objc_class 的定义：

```
struct objc_class {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class _Nullable super_class                              OBJC2_UNAVAILABLE;
    const char * _Nonnull name                               OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list * _Nullable ivars                  OBJC2_UNAVAILABLE;
    struct objc_method_list * _Nullable * _Nullable methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache * _Nonnull cache                       OBJC2_UNAVAILABLE;
    struct objc_protocol_list * _Nullable protocols          OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
```

在 Objective-C 中，每一个对象是一个结构体，每个对象都有一个 isa 指针，类对象 Class 也是一个对象。所以，我们说，凡是包含 isa 指针的，都可以被认为是 Objective-C 中的对象。运行时可以通过 isa 指针，查找到该对象是属于什么类(Class)。



### 2. isa 是什么？为什么要有 isa？

在 Runtime 源码中，对于 objc_object 和 objc_class 的定义分别如下：

```
struct objc_object {
private:
    isa_t isa;  // isa 是一个 union 联合体，其包含这个对象所属类的信息

public:
    Class ISA();     // ISA() assumes this is NOT a tagged pointer object
    Class getIsa();  // getIsa() allows this to be a tagged pointer object
    
	...
};
```

```
struct objc_class : objc_object {
    // 这里没写 isa，其实继承了 objc_object 的 isa , 在这里 isa 是一个指向元类的指针
    // Class ISA;
    Class superclass;           // 指向当前类的父类
    cache_t cache;              // formerly cache pointer and vtable
                                // 用于缓存指针和 vtable，加速方法的调用
    class_data_bits_t bits;     // class_rw_t * plus custom rr/alloc flags
                                // 相当于 class_rw_t 指针加上 rr/alloc 的标志
                                // bits 用于存储类的方法、属性、遵循的协议等信息的地方

    // 针对 class_data_bits_t 的 data() 函数的封装，最终返回一个 class_rw_t 类型的结构体变量
    // Objective-C 类中的属性、方法还有遵循的协议等信息都保存在 class_rw_t 中
    class_rw_t *data() { 
        return bits.data();
    }
    
    ...
};
```


objc_class 继承于 objc_object，所以 objc_class 也是一个 objc_object，objc_object 和 objc_class 都有一个成员变量 isa。isa 变量的类型是 isa_t，这个 isa_t 其实是一个联合体（union），其中包括成员量 cls。也就是说，每个 objc_object 通过自己持有的 isa，都可以查找到自己所属的类，对于 objc_class 来说，就是通过 isa 找到自己所属的元类（meta class）。

```
#define ISA_MASK        0x00007ffffffffff8ULL
#define ISA_MAGIC_MASK  0x001f800000000001ULL
#define ISA_MAGIC_VALUE 0x001d800000000001ULL
#define RC_ONE   (1ULL<<56)
#define RC_HALF  (1ULL<<7)

// isa 的类型是一个 isa_t 联合体
union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;			// 所属的类
    uintptr_t bits;

    struct {
        uintptr_t nonpointer        : 1;  // 表示 isa_t 的类型，0 表示 raw isa，也就是没有结构体的部分，访问对象的 isa 会直接返回一个指向 cls 的指针，也就是在 iPhone 迁移到 64 位系统之前时 isa 的类型。1 表示当前 isa 不是指针，但是其中也有 cls 的信息，只是其中关于类的指针都是保存在 shiftcls 中。
        uintptr_t has_assoc         : 1;  // 对象含有或者曾经含有关联引用，没有关联引用的可以更快地释放内存
        uintptr_t has_cxx_dtor      : 1;  // 表示当前对象有 C++ 或者 ObjC 的析构器(destructor)，如果没有析构器就会快速释放内存。
        uintptr_t shiftcls          : 44; // MACH_VM_MAX_ADDRESS 0x7fffffe00000
        uintptr_t magic             : 6;  // 用于调试器判断当前对象是真的对象还是没有初始化的空间
        uintptr_t weakly_referenced : 1;  // 对象被指向或者曾经指向一个 ARC 的弱变量，没有弱引用的对象可以更快释放
        uintptr_t deallocating      : 1;  // 对象正在释放内存
        uintptr_t has_sidetable_rc  : 1;  // 对象的引用计数太大了，存不下
        uintptr_t extra_rc          : 8;  // 对象的引用计数超过 1，会存在这个这个里面，如果引用计数为 10，extra_rc 的值就为 9
    };
};
```

而在 Objective-C 中，对象的方法都是存储在类中，而不是对象中（如果每一个对象都保存了自己能执行的方法，那么对内存的占用有极大的影响）。

```
// Objective-C 类中的属性、方法还有遵循的协议等信息都保存在 class_rw_t 中
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;               // 一个指向常量的指针，其中存储了当前类在编译期就已经确定的属性、方法以及遵循的协议

    method_array_t methods;             // 方法列表
    property_array_t properties;        // 属性列表
    protocol_array_t protocols;         // 所遵循的协议的列表

    Class firstSubclass;
    Class nextSiblingClass;
    
    ...
    
};
```

当一个对象的实例方法被调用时，它要通过自己持有的 isa 来查找对应的类，然后在这里的 class_data_bits_t 结构体中查找对应方法的实现（每个对象可以通过 `cls->data()-> methods` 来访问所属类的方法）。同时，每一个 objc_class 也有一个指向自己的父类的指针 super_class 用来查找继承的方法。

因为在 Objective-C 中，类其实也是一个对象，每个类也有一个 isa 指向自己所属的元类。所以无论是类还是对象都能通过相同的机制查找方法的实现。

**isa 在方法调用时扮演的角色：**

- 调用一个对象的实例方法时，通过对象的 isa 在类中获取方法的实现
- 调用一个类的类方法时，通过类的 isa 在元类中获取方法的实现
- 如果在当前类/元类中没找到，就会通过类/元类的 superclass 在继承链中一级一级往上查找


![](http://7ni3rk.com1.z0.glb.clouddn.com/Runtime/class-diagram.jpg)
<div align='center'>图 1. 对象，类与元类之间的关系</div>

**isa_t 中包含什么：**

isa 的类型 isa_t 是一个 union 类型的结构体，也就是说其中的 isa_t、cls、 bits 还有结构体共用同一块地址空间，而 isa 总共会占据 64 位的内存空间（决定于其中的结构体）。其中包含的信息见上面的代码注释。

现在直接访问对象（objc_object）的 isa 已经不会返回类指针了，取而代之的是使用 `ISA()` 方法来获取类指针。其中 ISA_MASK 是宏定义，这里通过掩码的方式获取类指针。

#### 结论
（1）isa 的作用：用于查找对象（或类对象）所属类（或元类）的信息，比如方法列表。

（2）isa 是什么：isa 的数据结构是一个 isa_t 联合体，其中包含其所属的 Class 的地址，通过访问对象的 isa，就可以获取到指向其所属 Class 的指针（针对 tagged pointer 的情况，也就是 non-pointer isa，有点不一样的是，除了指向 class 的指针，isa 中还会包含对象本身的一些信息，比如对象是否被弱引用）。


- [Non-pointer isa](http://www.sealiesoftware.com/blog/archive/2013/09/24/objc_explain_Non-pointer_isa.html)

### 3. 为什么在 Objective-C 中，所以的对象都用一个指针来追踪？

内存中的数据类型分为两种：值类型和引用类型。指针就是引用类型，struct 类型就是值类型。


值类型在传值时需要拷贝内容本身，而引用类型在传递时，拷贝的是对象的地址。所以，一方面，值类型的传递占用更多的内存空间，使用引用类型更节省内存开销；另一方面，也是最主要的原因，很多时候，我们需要把一个对象交给另一个函数或者方法去修改其中的内容（比如说一个 Person 对象的 age 属性），显然如果我们想让修改方获取到这个对象，我们需要的传递的是地址，而不是复制一份。

对于像 `int` 这样的基本数据类型，拷贝起来更快，而且数据简单，方便修改，所以就不用指针了。

另一方面，对象的内存是分配在堆上的，而值类型是分配到栈上的。所以一般对象的生命周期会比普通的值类型要长，而且创建和销毁对象以及内存管理是要消耗性能的，所以通过指针来引用一个对象，比直接复制和创建对象要更有效率、更节省性能。

参考：

- [Understanding pointers?](https://stackoverflow.com/questions/9746683/understanding-pointers)
- [need of pointer objects in objective c](https://stackoverflow.com/questions/17992127/need-of-pointer-objects-in-objective-c)
- [Why "Everything" in Objective C is pointers. I mean why I should declare NSArray instance variables in pointers.](https://teamtreehouse.com/community/why-everything-in-objective-c-is-pointers-i-mean-why-i-should-declare-nsarray-instance-variables-in-pointers)
- [Why do all objects in Objective-C have to use pointers?](https://www.quora.com/Why-do-all-objects-in-Objective-C-have-to-use-pointers#)

### 4. Objective-C 对象是如何被创建（alloc）和初始化（init）的？

**整个对象的创建过程其实就做了两件事情：为对象分配内存空间，以及初始化 isa（一个联合体）。**

（1）创建 NSObject 对象的过程

```
+ (id)alloc {
    return _objc_rootAlloc(self);
}


id
_objc_rootAlloc(Class cls)
{
    return callAlloc(cls, false/*checkNil*/, true/*allocWithZone*/);
}


static ALWAYS_INLINE id
callAlloc(Class cls, bool checkNil, bool allocWithZone=false)
{
    id obj = class_createInstance(cls, 0);
    if (slowpath(!obj)) return callBadAllocHandler(cls);
    return obj;
}


id 
class_createInstance(Class cls, size_t extraBytes)
{
    return _class_createInstanceFromZone(cls, extraBytes, nil);
}
```

最核心的逻辑就在 `_class_createInstanceFromZone` 函数中：

```
static id _class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone, bool cxxConstruct = true, size_t *outAllocatedSize = nil) {

    // 实例变量内存大小，实例大小 instanceSize 会存储在类的 isa_t 结构体中，经过对齐最后返回
    size_t size = cls->instanceSize(extraBytes);  

    // 给对象申请内存空间
    id obj = (id)calloc(1, size);
    if (!obj) return nil;
    
    // 初始化 isa
    obj->initInstanceIsa(cls, hasCxxDtor);

    return obj;
}

```

获取对象内存空间大小：

```
size_t instanceSize(size_t extraBytes) {
    // Core Foundation 需要所有的对象的大小至少有 16 字节。
    size_t size = alignedInstanceSize() + extraBytes;
    if (size < 16) size = 16;
    return size;
}

uint32_t alignedInstanceSize() {
    return word_align(unalignedInstanceSize());
}

uint32_t unalignedInstanceSize() {
    assert(isRealized());
    return data()->ro->instanceSize;
}
```

初始化 isa，isa 是一个 isa_t 联合体：

```
inline void objc_object::initIsa(Class cls, bool indexed, bool hasCxxDtor) {
    if (!indexed) {
        isa.cls = cls;
    } else {
        isa.bits = ISA_MAGIC_VALUE;
        isa.has_cxx_dtor = hasCxxDtor;
        isa.shiftcls = (uintptr_t)cls >> 3;
    }
}
```

（2）NSObject 对象初始化的过程

NSObject 对象的初始化实际上就是返回 `+alloc` 执行后得到的对象本身：

```
- (id)init {
    return _objc_rootInit(self);
}

id _objc_rootInit(id obj) {
    return obj;
}
```

### 5. Objective-C 对象的实例变量是什么？为什么不能给 Objective-C 对象动态添加实例变量？


（1）两个注意点：

- Objective-C 的 `->` 操作符不是C语言指针操作
- Objective-C 对象不能简单对应于一个 C struct，访问成员变量不等于访问 C struct 成员

（2）Non Fragile ivars

在 Runtime 的现行版本中，最大的特点就是健壮的实例变量。

当一个类被**编译**时，实例变量的布局也就形成了，它表明访问类的实例变量的位置。用旧版OSX SDK 编译的 MyObject 类成员变量布局是这样的，MyObject的成员变量依次排列在基类NSObject 的成员后面。

当苹果发布新版本OSX SDK后，NSObject增加了两个成员变量。如果没有Non Fragile ivars特性，我们的代码将无法正常运行，因为MyObject类成员变量布局在编译时已经确定，有两个成员变量和基类的内存区域重叠了。此时，我们只能重新编译MyObject代码，程序才能在新版本系统上运行。

现在有了 Non Fragile ivars 之后，问题就解决了。**在程序启动后，runtime加载MyObject类的时候，通过计算基类的大小，runtime 动态调整了 MyObject 类成员变量布局，把MyObject成员变量的位置向后移动8个字节。于是我们的程序无需编译，就能在新版本系统上运行。**

（3）Non Fragile ivars 是如何实现的呢？

当成员变量布局调整后，静态编译的native程序怎么能找到变量的新偏移位置呢？


根据 Runtime 源码可知，一个变量实际上就是一个 ivar_t 结构体。而每个 Objective-C 对象对应于 `struct objc_object`，后者的 isa 指向类定义，即 struct objc_class：

```
typedef struct ivar_t *Ivar;

struct objc_object {
private:
    isa_t isa;
    //...
};

struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

    class_rw_t *data() { 
        return bits.data();
    }
    //...
};


```

沿着 objc_class 的 `data()->ro->ivars` 找下去，struct ivar_list_t是类所有成员变量的定义列表。通过 `first` 变量，可以取得类里任意一个类成员变量的定义。

```
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;               // 一个指向常量的指针，其中存储了当前类在编译期就已经确定的属性、方法以及遵循的协议
    //...
    
};

struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;           // 实例变量大小，决定对象创建时要分配的内存
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;                // 类名
    method_list_t * baseMethodList;   // （编译时确定的）方法列表
    protocol_list_t * baseProtocols;  // （编译时确定的）所属协议列表
    const ivar_list_t * ivars;        // （编译时确定的）实例变量列表

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;  // （编译时确定的）属性列表

    method_list_t *baseMethods() const {
        return baseMethodList;
    }
};

struct ivar_list_t {
    uint32_t entsize;
    uint32_t count;
    ivar_t first;
};

struct ivar_t {
    int32_t *offset;
    const char *name;
    const char *type;
    //...
};
```

这里的 offset，应该就是用来记录这个成员变量在对象中的偏移位置。也就是说，runtime在发现基类大小变化时，通过修改offset，来更新子类成员变量的偏移值。

假如我们现在有一个继承于 NSError 的类 MyClass，在编译时，LLVM计算出基类 NSError 对象的大小为40字节，然后记录在MyClass的类定义中，如下是对应的C代码。在编译后的可执行程序中，写死了“40”这个魔术数字，记录了在此次编译时MyClass基类的大小。

```
class_ro_t class_ro_MyClass = {
    .instanceStart = 40,
    .instanceSize = 48,
    //...
}
```

现在假如苹果发布了OSX 11 SDK，NSError类大小增加到48字节。当我们的程序启动后，runtime加载MyClass类定义的时候，发现基类的真实大小和MyClass的instanceStart不相符，得知基类的大小发生了改变。于是runtime遍历MyClass的所有成员变量定义，将offset指向的值增加8。具体的实现代码在runtime/objc-runtime-new.mm的 `moveIvars()`函数中。

并且，MyClass类定义的instanceSize也要增加8。这样runtime在创建MyClass对象的时候，能分配出正确大小的内存块。

```
static void moveIvars(class_ro_t *ro, uint32_t superSize)
{
    uint32_t diff;

    diff = superSize - ro->instanceStart;
    if (ro->ivars) {
        // Find maximum alignment in this class's ivars
        uint32_t maxAlignment = 1;
        for (const auto& ivar : *ro->ivars) {
            if (!ivar.offset) continue;  // anonymous bitfield

            uint32_t alignment = ivar.alignment();
            if (alignment > maxAlignment) maxAlignment = alignment;
        }

        // Compute a slide value that preserves that alignment
        uint32_t alignMask = maxAlignment - 1;
        diff = (diff + alignMask) & ~alignMask;

        // Slide all of this class's ivars en masse
        for (const auto& ivar : *ro->ivars) {
            if (!ivar.offset) continue;  // anonymous bitfield

            uint32_t oldOffset = (uint32_t)*ivar.offset;
            uint32_t newOffset = oldOffset + diff;
            *ivar.offset = newOffset;

            if (PrintIvars) {
                _objc_inform("IVARS:    offset %u -> %u for %s "
                             "(size %u, align %u)", 
                             oldOffset, newOffset, ivar.name, 
                             ivar.size, ivar.alignment());
            }
        }
    }

    *(uint32_t *)&ro->instanceStart += diff;
    *(uint32_t *)&ro->instanceSize += diff;
}


```

（4）为什么Objective-C类不能动态添加成员变量

runtime 提供了一个 class_addIvar() 函数用于给类添加成员变量，但是根据文档中的注释，这个函数只能在“构建一个类的过程中”调用。一旦完成类定义，就不能再添加成员变量了。经过编译的类在程序启动后就被runtime加载，没有机会调用addIvar。程序在运行时动态构建的类需要在调用 objc_allocateClassPair 之后，调用 objc_registerClassPair 之前才可以添加成员变量。


这样做会带来严重问题，为基类动态增加成员变量会导致所有已创建出的子类实例都无法使用。那为什么runtime允许动态添加方法和属性，而不会引发问题呢？

因为方法和属性并不“属于”实例，而成员变量“属于”实例。我们所说的“类实例”概念，指的是一块内存区域，包含了isa指针和所有的成员变量。所以假如允许动态修改类成员变量布局，已经创建出的实例就不符合类定义了，变成了无效对象。但方法定义是在objc_class中管理的，不管如何增删类方法，都不影响实例的内存布局，已经创建出的实例仍然可正常使用。

美团的技术博客中给出的解释比较简单，其中“破坏破坏类的内部布局”这句话本身也是有些问题的：
> extension在**编译期决议**，它就是类的一部分，在编译期和头文件里的@interface以及实现文件里的@implement一起形成一个完整的类，它伴随类的产生而产生，亦随之一起消亡。extension一般用来隐藏类的私有信息，你必须有一个类的源码才能为一个类添加extension，所以你无法为系统的类比如NSString添加extension。
>
> 但是category则完全不一样，它是在**运行期决议的**。
就category和extension的区别来看，我们可以推导出一个明显的事实，extension可以添加实例变量，而category是无法添加实例变量的（**因为在运行期，对象的内存布局已经确定，如果添加实例变量就会破坏类的内部布局，这对编译型语言来说是灾难性的**）。




参考：

- [Objective-C类成员变量深度剖析](http://quotation.github.io/objc/2015/05/21/objc-runtime-ivar-access.html)
- [深入理解Objective-C：Category](https://tech.meituan.com/DiveIntoCategory.html)
- [Non-fragile ivars ](http://www.sealiesoftware.com/blog/archive/2009/01/27/objc_explain_Non-fragile_ivars.html)

### 6. Objective-C 对象的属性是什么？属性跟实例变量的区别？


### 7. Objective-C 对象的方法是什么？Objective-C 对象的方法在内存中的存储结构是什么样的？

objc_class 有一个 class_data_bits_t 类型的变量 bits，Objective-C 类中的属性、方法还有遵循的协议等信息都保存在 class_rw_t 中，通过调用 objc_class 的 class_rw_t *data() 方法，可以获取这个 class_rw_t 类型的变量。



```
// Objective-C 类是一个结构体，继承于 objc_object
struct objc_class : objc_object {
    // 这里没写 isa，其实继承了 objc_object 的 isa , 在这里 isa 是一个指向元类的指针
    // Class ISA;
    Class superclass;           // 指向当前类的父类
    cache_t cache;              // formerly cache pointer and vtable
                                // 用于缓存指针和 vtable，加速方法的调用
    class_data_bits_t bits;     // class_rw_t * plus custom rr/alloc flags
                                // 相当于 class_rw_t 指针加上 rr/alloc 的标志
                                // bits 用于存储类的方法、属性、遵循的协议等信息的地方
                                

    // 针对 class_data_bits_t 的 data() 函数的封装，最终返回一个 class_rw_t 类型的结构体变量
    // Objective-C 类中的属性、方法还有遵循的协议等信息都保存在 class_rw_t 中
    class_rw_t *data() { 
        return bits.data();
    }

	 ...
}
```

class_rw_t 中还有一个指向常量的指针 ro，其中存储了当前类在编译期就已经确定的属性、方法以及遵循的协议。

```
/ Objective-C 类中的属性、方法还有遵循的协议等信息都保存在 class_rw_t 中
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;               // 一个指向常量的指针，其中存储了当前类在编译期就已经确定的属性、方法以及遵循的协议

    method_array_t methods;             // 方法列表
    property_array_t properties;        // 属性列表
    protocol_array_t protocols;         // 所遵循的协议的列表

	...

}
```

```
// 用于存储一个 Objective-C 类在编译期就已经确定的属性、方法以及遵循的协议
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;
    method_list_t * baseMethodList;   // （编译时确定的）方法列表
    protocol_list_t * baseProtocols;  // （编译时确定的）所属协议列表
    const ivar_list_t * ivars;        // （编译时确定的）实例变量列表

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;  // （编译时确定的）属性列表

    method_list_t *baseMethods() const {
        return baseMethodList;
    }
};

```

加载 ObjC 运行时的过程中在 `realizeClass()` 方法中：

1. 从 class_data_bits_t 调用 data 方法，将结果强制转换为 class_ro_t 指针；
2. 初始化一个 class_rw_t 结构体；
3. 设置结构体 ro 的值以及 flag。
4. 最后重新将这个 class_rw_t 设置给 class_data_bits_t 的 data。

```
...
const class_ro_t *ro = (const class_ro_t *)cls->data();
class_rw_t *rw = (class_rw_t *)calloc(sizeof(class_rw_t), 1);
rw->ro = ro;
rw->flags = RW_REALIZED|RW_REALIZING;
cls->setData(rw);
...
```

在上面这段代码运行之后 `class_rw_t` 中的方法，属性以及协议列表均为空。这时需要 `realizeClass` 调用 `methodizeClass` 方法来将类自己实现的方法（包括分类）、属性和遵循的协议加载到 methods、 properties 和 protocols 列表中。

方法的结构，与类和对象一样，方法在内存中也是一个结构体。

```
struct method_t {
    SEL name;
    const char *types;
    IMP imp;
};
```

结论：

1. 在 runtime 初始化之后，realizeClass 之前，从 class_data_bits_t 结构体中获取的 class_rw_t 一直都不是 class_rw_t 结构体，而是class_ro_t。因为类的一些方法、属性和协议都是在编译期决定的（baseMethods 等成员以及类在内存中的位置都是编译期决定的）。

2. 类在内存中的位置是在编译期间决定的，在之后修改代码，也不会改变内存中的位置。
类的方法、属性以及协议在编译期间存放到了“错误”的位置，直到 realizeClass 执行之后，才放到了 class_rw_t 指向的只读区域 class_ro_t，这样我们即可以在运行时为 class_rw_t 添加方法，也不会影响类的只读结构。

3. 在 class_ro_t 中的属性在运行期间就不能改变了，再添加方法时，会修改 class_rw_t 中的 methods 列表，而不是 class_ro_t 中的 baseMethods。
    
### 7. 什么是选择器 selector ？什么是 IMP？
1. 向不同的类发送相同的消息时，其生成的选择子是完全相同的
2. 通过 @selector(方法名) 就可以返回一个选择子，通过 (void *)@selector(方法名)， 就可以读取选择器的地址
3. 推断 selector 的特性：
   - Objective-C 为我们维护了一个巨大的选择子表
   - 在使用 @selector() 时会从这个选择子表中根据选择子的名字查找对应的 SEL。如果没有找到，则会生成一个 SEL 并添加到表中
   - 在编译期间会扫描全部的头文件和实现文件将其中的方法以及使用 @selector() 生成的选择子加入到选择子表中
   
### 8. 关于消息发送和消息转发
> 具体过程查看源码中 `lookUpImpOrForward()` 函数部分的注释

1. 发送 hello 消息后，编译器会将上面这行 [obj hello]; 代码转成 objc_msgSend()（注：objc_msgSend 是一个私有方法，而且是用汇编实现的，我们没有办法进入它的实现，但是我们可以通过 lookUpImpOrForward 函数断点拦截）
2. 到当前类的缓存中去查找方法实现，如果找到了直接 done
3. 如果没找到，就到当前类的方法列表中去查找，如果找到了直接 done
4. 如果还没找到，就到父类的缓存中去查找方法实现，如果找到了直接 done
5. 如果没找到，就到父类的方法列表中去查找，如果找到了直接 done
6. 如果还没找到，就进行方法决议
7. 最后还没找到的话，就走消息转发

### 9. Method Swizzling 的原理是什么？

### 10. Objective-C 中的 Category 是什么？

### 11.  Associated Objects 的原理是什么？到底能不能给 Objective-C 类添加属性和实例变量？

### 12. Objective-C 中的 Protocol 是什么？


### 13. self 和 super 的本质


self和super两个是不同的，self是类的一个隐藏参数，每个方法的实现的第一个参数即为self。而super不是一个隐藏参数，它实际上只是一个”编译器标示符”，它负责告诉编译器，当调用viewDidLoad方法时，去调用父类的方法，而不是本类中的方法。我们可以看看super的定义。

```
#ifndef OBJC_SUPER
#define OBJC_SUPER
/// Specifies the superclass of an instance. 
struct objc_super {
    /// Specifies an instance of a class.
    __unsafe_unretained id receiver;
    /// Specifies the particular superclass of the instance to message. 
#if !defined(__cplusplus)  &&  !__OBJC2__
    /* For compatibility with old objc-runtime.h header */
    __unsafe_unretained Class class;
#else
    __unsafe_unretained Class super_class;
#endif
    /* super_class is the first class to search */
};
#endif
```

objc_super 包含了两个变量，receiver是消息的实际接收者，super_class是指向当前类的父类。当我们使用super来接收消息时，编译器会生成一个objc_super结构体。发送消息时，就不是使用objc_msgSend方法了，而是objc_msgSendSuper

- [Objc Runtime](http://iostangtang.com/2017/05/20/Runtime/)

### 14. load 方法和 initialize 方法

### 延伸

1. clang 命令的使用（比如 `clang -rewrite-objc test.m`），`clang -rewrite-objc` 的作用是什么？clang rewrite 出来的文件跟 objc runtime 源码的实现有什么区别吗？


### 参考：

  - [Objective-C 中的类和对象](https://blog.ibireme.com/2013/11/25/objc-object/)
  - Draveness 出品的 runtime 源码阅读系列文章（强烈推荐）
     - [对象是如何初始化的（iOS）](https://draveness.me/object-init)：介绍了 Objective-C 对象初始化的过程
     - [从 NSObject 的初始化了解 isa](https://draveness.me/isa)：深入剖析了 isa 的结构和作用
     - [深入解析 ObjC 中方法的结构](https://draveness.me/method-struct)：介绍了在 ObjC 中是如何存储方法的
     - [从源代码看 ObjC 中消息的发送](https://draveness.me/message) ：通过逐步断点调试 objc 源码的方式，从 Objc 源代码中分析并合理地推测一些关于消息传递的过程
  - [从 ObjC Runtime 源码分析一个对象创建的过程](https://www.jianshu.com/p/8e4887a43bd7)
  - [Objective-C 对象模型](http://blog.leichunfeng.com/blog/2015/04/25/objective-c-object-model/)
  - [Objc 对象的今生今世](https://www.jianshu.com/p/f725d2828a2f)
  - [Runtime源码 —— 概述和调试环境准备](https://www.jianshu.com/u/43bb8b1a9d39)：作者写了一个系列的文章，内容很详细
  - [Objective-C runtime - 系列开始](http://vanney9.com/2017/06/03/objective-c-runtime-summarize/)：简单介绍了学习 objc 源代码的方法