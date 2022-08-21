# Objc Runtime

Objective-C是一门动态性比较强的语言，区别于C/C++，可以在运行过程中动态修改已经编译好的一些行为（包括动态添加方法、属性setter/getter方法生成、KVO等），由Runtime API来支撑，接口基本是C语言，源码由C/C++/汇编编写。

## 一、类、实例对象底层结构

### 1. isa指针

NSObject定义，如下：
```
@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}
```
结合源码 [objc4-824](https://opensource.apple.com/tarballs/objc4/objc4-824.tar.gz) 搜索 *Class，查询到如下的代码：
```
typedef struct objc_class *Class;
typedef struct objc_object *id;

struct objc_object {
private:
    isa_t isa;
}
```
isa_t是联合体，定义如下：
```
union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    uintptr_t bits;
private:
    // Accessing the class requires custom ptrauth operations, so
    // force clients to go through setClass/getClass by making this
    // private.
    Class cls;
public:
#if defined(ISA_BITFIELD)
#   define ISA_MASK        0x0000000ffffffff8ULL
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL
    struct {
        ISA_BITFIELD;  // defined in isa.h
        uintptr_t nonpointer        : 1; /*0,代表普通指针，存储Class、Meta-Class对象内存地址 1,使用位域优化过 */                                  
        uintptr_t has_assoc         : 1; /*是否设置过关联对象*/                                          
        uintptr_t has_cxx_dtor      : 1; /*是否有C++析构函数*/                                      
        uintptr_t shiftcls          : 33;/*存储着Class、Meta-Class对象内存地址信息*/ 
        uintptr_t magic             : 6; /*用于在调试时分辨对象是否未完成初始化*/                                     
        uintptr_t weakly_referenced : 1; /*是否被弱引用指向过*/                                      
        uintptr_t unused            : 1;                                       
        uintptr_t has_sidetable_rc  : 1; /*extra_rc引用计数是否满足*/                                   
        uintptr_t extra_rc          : 19;/*引用计数减一*/
    };
}
```
实例对象 ISA & ISA_MASK 可以得到类对象实际地址，类结构Class定义如下：
```
typedef struct objc_class *Class;
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
}

/* 
获取类对象数据
#define FAST_DATA_MASK          0x00007ffffffffff8UL
class_rw_t *data = (class_rw_t *)(bits & FAST_DATA_MASK)
*/  

struct class_rw_t {

    Class firstSubclass;
    Class nextSiblingClass;

	const class_ro_t *ro;
	const method_list_t *methods; //此处实际为 method_array_t<method_t, method_list_t>二维数组，这里简化为 方法列表
	const property_list_t *properties; // 属性列表
	const protocol_list_t *protocols; // 协议
}

struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;

    const uint8_t * ivarLayout;
    
    const char * name; // 类名
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars; // 成员变量列表
}
```

### 2.方法结构
```
struct method_t {
    SEL name; // 函数名
    const char *types; // 编码
    IMP imp; // 函数指针
};
```
types 是包含了函数返回值、参数编码的字符串，如下：
```
- (void)eat:(NSString *)food;

// 获取方法编码
Method method = class_getInstanceMethod(person.class, @selector(eat:));
const char *vars = method_getTypeEncoding(method); // "v24@0:8@16"

/*
v --- 方法无返回值
@ --- 参数id类型
: --- 参数SEL类型
@ --- 参数id类型

24 --- 参数占用字节和
0 --- 第一个参数起始
8 --- 第二个参数起始
16 --- 第三个参数起始
*/
```
可以通过 @encode() 获取，具体类型编码参见[Runtime Type Encodings](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100)

### 3. 方法缓存

Class内部结构中有个方法缓存(cache_t)，用 散列表 来缓存曾经调用过的方法，结构如下：
```
struct cache_t {
	struct bucket_t *buckets; // 散列表
	uint32_t mask; // 散列表长度 -1
	uint32_t occupied; // 已经占用的数量
}

struct bucket {
	uintptr_t _imp; // 方法实现
	SEL _sel; // 方法名
}
```
为什么说上面提到的 buckets 是以散列表结构存储，objc-cache.mm 中查询 insert 函数，如下：
```
void cache_t::insert(SEL sel, IMP imp, id receiver) {
	// 1. 根据occupied + 1判定当前容量是否满足，否则重新开辟
	......
	// 2. 计算hash值存储
	bucket_t *b = buckets();
    mask_t m = capacity - 1;
    mask_t begin = cache_hash(sel, m);
    mask_t i = begin;

	// Scan for the first unused slot and insert there.
    // There is guaranteed to be an empty slot.
    do {
        if (fastpath(b[i].sel() == 0)) {
            incrementOccupied();
            b[i].set<Atomic, Encoded>(b, sel, imp, cls());
            return;
        }
        if (b[i].sel() == sel) {
            // The entry was added to the cache by some other thread
            // before we grabbed the cacheUpdateLock.
            return;
        }
    } while (fastpath((i = cache_next(i, m)) != begin));
}

// hash函数，获取地址值
static inline mask_t cache_hash(SEL sel, mask_t mask) 
{
    uintptr_t value = (uintptr_t)sel;
    return (mask_t)(value & mask);
}
// 寻找下一个位置
static inline mask_t cache_next(mask_t i, mask_t mask) {
    return i ? i-1 : mask;
}
```
方法查找是同样的原理，底层是汇编实现，IMP cache_getImp(Class cls, SEL sel)

## 二、objc_msgSend执行流程

### 1. 方法查找

OC方法调用在底层都转换为objc_msgSend函数调用，id objc_msgSend(id self, SEL _cmd, ...);，源码查询如下：
```
objc-msg-arm64.s文件，汇编语言

ENTRY _objc_msgSend // 函数入口
b.le LNilOrTagged  // 向nil对象发送消息，直接ret
CacheLookup NORMAL, _objc_msgSend, __objc_msgSend_uncached // 调用 imp 或者 objc_msgSend_uncached
MethodTableLookup // 查找方法表 汇编
_lookUpImpOrForward // c函数，lookUpImpOrForward(obj, sel, cls, LOOKUP_INITIALIZE | LOOKUP_RESOLVER)


objc-runtime-new.mm文件
lookUpImpOrForward(obj, sel, cls, LOOKUP_INITIALIZE | LOOKUP_RESOLVER)
resolveMethod_locked() // 动态解析
lookUpImpOrForwardTryCache() // 再次进行缓存方法查找
```
详细看一下 lookUpImpOrForward 函数实现：
```
/***********************************************************************
* 精简代码，反应大致逻辑，
* 查找缓存,包括本类及父类，查找到返回 IMP，否则查找当前类及父类methodlist，至到父类为nil，返回
**********************************************************************/
IMP lookUpImpOrForward(id inst, SEL sel, Class cls, int behavior)
{
    const IMP forward_imp = (IMP)_objc_msgForward_impcache;
    IMP imp = nil;
    Class curClass;

    curClass = cls;
	// 
	for (unsigned attempts = unreasonableClassCount();;) {
    	if (curClass->cache.isConstantOptimizedCache(/* strict */true)) {
            imp = cache_getImp(curClass, sel);
            if (imp) goto done_unlock;
            curClass = curClass->cache.preoptFallbackClass();
        } else {
            // curClass method list.
            Method meth = getMethodNoSuper_nolock(curClass, sel);
            if (meth) {
                imp = meth->imp(false);
                goto done;
            }
            if ((curClass = curClass->getSuperclass()) == nil) {
                // No implementation found, and method resolver didn't help.
                // Use forwarding.
                imp = forward_imp;
                break;
            }
        }
	}

    // No implementation found. 进入动态方法解析
    if (behavior & LOOKUP_RESOLVER) {
        behavior ^= LOOKUP_RESOLVER;
        return resolveMethod_locked(inst, sel, cls, behavior);
    }

 done:
    if ((behavior & LOOKUP_NOCACHE) == 0) {
        log_and_fill_cache(cls, imp, sel, inst, curClass);
    }
 done_unlock:
    if (slowpath((behavior & LOOKUP_NIL) && imp == forward_imp)) {
        return nil;
    }
    return imp;
}
```
方法查找流程如下：
![image2021-5-29_16-14-26](https://user-images.githubusercontent.com/15702940/185773290-a573d3eb-f089-43fb-8964-e4ec7d5a0621.png)

## 2. 动态方法解析
```
/***********************************************************************
* resolveMethod_locked
* Call +resolveClassMethod or +resolveInstanceMethod.
**********************************************************************/
static NEVER_INLINE IMP
resolveMethod_locked(id inst, SEL sel, Class cls, int behavior)
{
	// 当前类是否元类对象，
    if (! cls->isMetaClass()) {
        // 尝试实例方法 动态解析
        resolveInstanceMethod(inst, sel, cls);
    } 
    else {
        // try [nonMetaClass resolveClassMethod:sel]
        // and [cls resolveInstanceMethod:sel]
        resolveClassMethod(inst, sel, cls);
        if (!lookUpImpOrNilTryCache(inst, sel, cls)) {
            resolveInstanceMethod(inst, sel, cls);
        }
    }
    // chances are that calling the resolver have populated the cache
    // so attempt using it
    return lookUpImpOrForwardTryCache(inst, sel, cls, behavior);
}
lookUpImpOrForwardTryCache内部实现如下，
static IMP _lookUpImpTryCache(id inst, SEL sel, Class cls, int behavior)
{
    if (!cls->isInitialized()) {
        // see comment in lookUpImpOrForward
        return lookUpImpOrForward(inst, sel, cls, behavior);
    }

    IMP imp = cache_getImp(cls, sel);
    if (imp != NULL) goto done;
    if (cls->cache.isConstantOptimizedCache(/* strict */true)) {
        imp = cache_getImp(cls->cache.preoptFallbackClass(), sel);
    }
    if (slowpath(imp == NULL)) {
		// 方法查找
        return lookUpImpOrForward(inst, sel, cls, behavior);
    }
done:
    if ((behavior & LOOKUP_NIL) && imp == (IMP)_objc_msgForward_impcache) {
        return nil;
    }
    return imp;
}
```
当前类缓存中如果找不到，再次通过lookUpImpOrForward进行方法查找，依旧没找到返回imp = _objc_msgForward_impcache进行消息转发，流程总结如下：
![image2021-5-29_18-31-7](https://user-images.githubusercontent.com/15702940/185773309-8f0ff716-ca5c-487b-bdd4-f94d1c75965f.png)


## 3. 消息转发

_objc_msgForward_impcache是汇编，实现如下：
```
STATIC_ENTRY __objc_msgForward_impcache

	// 跳转到消息转发
	b	__objc_msgForward

END_ENTRY __objc_msgForward_impcache


ENTRY __objc_msgForward

	adrp	x17, __objc_forward_handler@PAGE
	ldr	p17, [x17, __objc_forward_handler@PAGEOFF]
	TailCallFunctionPointer x17
	
END_ENTRY __objc_msgForward
```
如何进行消息转发是不开源的，反汇编可以得出大致的实现流程(前人种树，后人乘凉)，如下：
<img width="535" alt="image" src="https://user-images.githubusercontent.com/15702940/185773418-3964b3b3-444e-4f86-a7b0-315a095ed71e.png">

注意：forwardingTargetForSelector、methodSignatureForSelector有对应的实例方法和类方法








