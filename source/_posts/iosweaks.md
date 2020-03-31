---
title: iOS大解密之Weak引用
date: 2017-04-06 17:23:32
updated: 2017-04-06 17:23:32
categories: iOS
tags: 
 - iOS
 - Runtime
---

## 引言
assign与weak有什么区别？面试的时候常常会被问到此问题，我们会回答weak修饰在对象释放时会自动变为nil,那么底层是怎么实现的呢？今天让我们来探讨一下，本文使用的runtime源码为大神提供的可编译版本[objc4-706](https://github.com/isaacselement/objc4-706)

## storeWeak...

与weak相关的操作，应该都调用了次方法，稍后看一下这个方法的实现，首先做一下准备工作

我首先在项目中定义了两个类Person、Like实现如下

``` C++
@interface Person : NSObject
@property (atomic,weak) Like *like;
- (void)logLikeAddress;
@end

@implementation Person
- (void)logLikeAddress
{
    printf("%p",&_like);
}
@end

// Like其实什么也没实现😂
@interface Like : NSObject
@end
@implementation Like
@end

```

其中main函数为

``` C++
void addLike(Person* person)
{
    Like *like = [[Like alloc] init];
    printf("%p\n",like);
    [person logLikeAddress];
    person.like = like;
}

int main(int argc, const char * argv[])
{
    @autoreleasepool {
        // insert code here...
        Person *person = [[Person alloc] init];
        addLike(person);

    }
    return 0;
}
```

准备工作已经就绪了，我们在`Person`的中`weak`修饰的属性下断点，并单步调试，会发现如下的函数调用栈

![](http://mainimage.hi996.com/b2.png)

我们发现了关键的函数`objc_storeWeak(...)`,下面我们要详细的探讨这一函数

``` C++
template <bool HaveOld, bool HaveNew, bool CrashIfDeallocating>
static id 
storeWeak(id *location, objc_object *newObj)
{
    assert(HaveOld  ||  HaveNew);
    if (!HaveNew) assert(newObj == nil);

    Class previouslyInitializedClass = nil;
    id oldObj;
    // 新旧引用表创建
    SideTable *oldTable;
    SideTable *newTable;

    // Acquire locks for old and new values.
    // Order by lock address to prevent lock ordering problems. 
    // Retry if the old value changes underneath us.
 retry:
    if (HaveOld) {
        // 获取旧引用表
        oldObj = *location;
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
    if (HaveNew) {
        // 获取新引用表
        newTable = &SideTables()[newObj];
    } else {
        newTable = nil;
    }
    // 加锁
    SideTable::lockTwo<HaveOld, HaveNew>(oldTable, newTable);
    // location 应该与 oldObj 保持一致，否则重新获取
    if (HaveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<HaveOld, HaveNew>(oldTable, newTable);
        goto retry;
    }

    // Prevent a deadlock between the weak reference machinery
    // and the +initialize machinery by ensuring that no 
    // weakly-referenced object has an un-+initialized isa.
    if (HaveNew  &&  newObj) {
        Class cls = newObj->getIsa();
        if (cls != previouslyInitializedClass  &&  
            !((objc_class *)cls)->isInitialized()) 
        {
            SideTable::unlockTwo<HaveOld, HaveNew>(oldTable, newTable);
            _class_initialize(_class_getNonMetaClass(cls, (id)newObj));

            // If this class is finished with +initialize then we're good.
            // If this class is still running +initialize on this thread 
            // (i.e. +initialize called storeWeak on an instance of itself)
            // then we may proceed but it will appear initializing and 
            // not yet initialized to the check above.
            // Instead set previouslyInitializedClass to recognize it on retry.
            previouslyInitializedClass = cls;

            goto retry;
        }
    }

    // Clean up old value, if any. 清除旧值
    if (HaveOld) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

    // Assign new value, if any. 设置新值
    if (HaveNew) {
        newObj = (objc_object *)weak_register_no_lock(&newTable->weak_table, 
                                                      (id)newObj, location, 
                                                      CrashIfDeallocating);
        // weak_register_no_lock returns nil if weak store should be rejected

        // Set is-weakly-referenced bit in refcount table.
        if (newObj  &&  !newObj->isTaggedPointer()) {
            // 标记该对象是一个弱引用
            newObj->setWeaklyReferenced_nolock();
        }

        // Do not set *location anywhere else. That would introduce a race.
        *location = (id)newObj;
    }
    else {
        // No new value. The storage is not changed.
    }
    
    SideTable::unlockTwo<HaveOld, HaveNew>(oldTable, newTable);

    return (id)newObj;
}

```

在此方法中，我们可以看见一些关键点， `SideTable`， `weak_unregister_no_lock(...)`， `weak_register_no_lock（...）`，下面我们会逐一介绍

## 引用计数和弱引用依赖表 SideTable
SideTable 是一个结构体，主要用于管理对象的引用计数和 weak 表。在 NSObject.mm 中声明其数据结构

``` C++
struct SideTable {
    //自旋锁
    spinlock_t slock;
    // 引用计数表
    RefcountMap refcnts;
    // 弱引用表
    weak_table_t weak_table;

    SideTable() {
        memset(&weak_table, 0, sizeof(weak_table));
    }

    ~SideTable() {
        _objc_fatal("Do not delete SideTable.");
    }

    void lock() { slock.lock(); }
    void unlock() { slock.unlock(); }

    // Address-ordered lock discipline for a pair of side tables.

    template<bool HaveOld, bool HaveNew>
    static void lockTwo(SideTable *lock1, SideTable *lock2);
    template<bool HaveOld, bool HaveNew>
    static void unlockTwo(SideTable *lock1, SideTable *lock2);
};
```

对于该结构体中的`slock`,`refcnts`,暂时我们不做讨论，我们主要讨论与弱引用相关的`weak_table`作用，其中`weak_table_t`的结构如下

``` C++
struct weak_table_t {
    // 保存了所有指向指定对象的 weak 指针
    weak_entry_t *weak_entries;
    // 存储空间
    size_t    num_entries;
    // 参与判断引用计数辅助量
    uintptr_t mask;
    // hash key 最大偏移值
    uintptr_t max_hash_displacement;
};
```

这是一个全局弱引用表。使用不定类型对象的地址作为 key ，用`weak_entry_t`类型结构体对象作为value。其中的`weak_entries`成员，从字面意思上看，即为弱引用表入口。其实现也是这样的

``` C++
#define WEAK_INLINE_COUNT 4
#define REFERRERS_OUT_OF_LINE 2
struct weak_entry_t {
    DisguisedPtr<objc_object> referent;
    union {
        struct {
            weak_referrer_t *referrers;
            uintptr_t        out_of_line_ness : 2;
            uintptr_t        num_refs : PTR_MINUS_2;
            uintptr_t        mask;
            uintptr_t        max_hash_displacement;
        };
        struct {
            // out_of_line_ness field is low bits of inline_referrers[1]
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };

    bool out_of_line() {
        return (out_of_line_ness == REFERRERS_OUT_OF_LINE);
    }

    weak_entry_t& operator=(const weak_entry_t& other) {
        memcpy(this, &other, sizeof(other));
        return *this;
    }

    weak_entry_t(objc_object *newReferent, objc_object **newReferrer)
        : referent(newReferent)
    {
        inline_referrers[0] = newReferrer;
        for (int i = 1; i < WEAK_INLINE_COUNT; i++) {
            inline_referrers[i] = nil;
        }
    }
};
```

让我们回到`storeWeak(...)`函数中，获取`oldTable`,与`newTable`,发现调用的函数为`&SideTables()[newObj]`,`&SideTables()[oldObj]`,其中该方法的实现为

``` C++
alignas(StripedMap<SideTable>) static uint8_t SideTableBuf[sizeof(StripedMap<SideTable>)];

static void SideTableInit() {
    new (SideTableBuf) StripedMap<SideTable>();
}

static StripedMap<SideTable>& SideTables() {
    return *reinterpret_cast<StripedMap<SideTable>*>(SideTableBuf);
}
```

``` C++
template<typename T>
class StripedMap {

    enum { CacheLineSize = 64 };

#if TARGET_OS_EMBEDDED
    enum { StripeCount = 8 };
#else
    enum { StripeCount = 64 };
#endif

    struct PaddedT {
        T value alignas(CacheLineSize);
    };

    PaddedT array[StripeCount];

    static unsigned int indexForPointer(const void *p) {
        uintptr_t addr = reinterpret_cast<uintptr_t>(p);
        return ((addr >> 4) ^ (addr >> 9)) % StripeCount;
    }

 public:
    T& operator[] (const void *p) { 
        return array[indexForPointer(p)].value; 
    }
    const T& operator[] (const void *p) const { 
        return const_cast<StripedMap<T>>(this)[p]; 
    }
};
```

在上面我们可以看出`StripedMap`是一个模板类（Template Class），通过传入类（结构体）参数，会动态修改在该类中的一个 array 成员存储的元素类型，并且其中提供了一个针对于地址的 hash 算法，用作存储key。可以说， `StripedMap` 提供了一套拥有将地址作为 key 的 hash table 解决方案，而该方案采用了模板类，是拥有泛型性的,在这个类中有一个 array 成员，用来存储 `PaddedT` 对象，并且其中对于 [] 符的重载定义中，会返回这个 PaddedT 的 value 成员，这个 value 就是我们传入的 T 泛型成员，也就是 `SideTable` 对象。在 array 的下标中，这里使用了 `indexForPointer` 方法通过位运算计算下标，实现了静态的 Hash Table。而在 `weak_table` 中，其成员 `weak_entry `会将传入对象的地址加以封装起来，并且其中也有访问全局弱引用表的入口

## weak_register_no_lock(...)
该方法将弱引用对象注册到弱引用表中，我们看一下它的具体实现

``` C++
id 
weak_register_no_lock(weak_table_t *weak_table, id referent_id, 
                      id *referrer_id, bool crashIfDeallocating)
{
    // weak对象的引用 即like
    objc_object *referent = (objc_object *)referent_id;
    // 指向weak对象的引用的指针 即Person对象中的&like 可理解为&(person.like)
    objc_object **referrer = (objc_object **)referrer_id;

    //判断TaggedPointer
    if (!referent  ||  referent->isTaggedPointer()) return referent_id;

    // ensure that the referenced object is viable 保证对象是可以访问的
    bool deallocating;
    if (!referent->ISA()->hasCustomRR()) {
        deallocating = referent->rootIsDeallocating();
    }
    else {
        BOOL (*allowsWeakReference)(objc_object *, SEL) = 
            (BOOL(*)(objc_object *, SEL))
            object_getMethodImplementation((id)referent, 
                                           SEL_allowsWeakReference);
        if ((IMP)allowsWeakReference == _objc_msgForward) {
            return nil;
        }
        deallocating =
            ! (*allowsWeakReference)(referent, SEL_allowsWeakReference);
    }

    if (deallocating) {
        if (crashIfDeallocating) {
            _objc_fatal("Cannot form weak reference to instance (%p) of "
                        "class %s. It is possible that this object was "
                        "over-released, or is in the process of deallocation.",
                        (void*)referent, object_getClassName((id)referent));
        } else {
            return nil;
        }
    }

    // now remember it and where it is being stored
    weak_entry_t *entry;
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        append_referrer(entry, referrer);
    } 
    else {
        // 该对象第一次插入
        weak_entry_t new_entry(referent, referrer);
        // 是否需要扩容表长度
        weak_grow_maybe(weak_table);
        // 插入表
        weak_entry_insert(weak_table, &new_entry);
    }

    // Do not set *referrer. objc_storeWeak() requires that the 
    // value not change.

    return referent_id;
}
```

总结一下，我们weak对象的引用,与指向weak对象的引用，传入此函数中，假设是首次保存此weak对象的引用，构建出`weak_entry_t`的实例
`new_entry`,判断当前的弱引用表是否需要增加长度，然后将`new_entry`插入到弱引用表`weak_table`中；如若不是第一次保存，首先取出旧的`entry`，并将指向weak对象的引用（即&(person.like)）保存在`entry`中。
所以可以看出，一个`entry`中保存了所有指向该weak对象的弱引用

## weak_unregister_no_lock(...)

``` C++
void
weak_unregister_no_lock(weak_table_t *weak_table, id referent_id, 
                        id *referrer_id)
{
    objc_object *referent = (objc_object *)referent_id;
    objc_object **referrer = (objc_object **)referrer_id;

    weak_entry_t *entry;

    if (!referent) return;

    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        remove_referrer(entry, referrer);
        bool empty = true;
        if (entry->out_of_line()  &&  entry->num_refs != 0) {
            empty = false;
        }
        else {
            for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
                if (entry->inline_referrers[i]) {
                    empty = false; 
                    break;
                }
            }
        }

        if (empty) {
            weak_entry_remove(weak_table, entry);
        }
    }

    // Do not set *referrer = nil. objc_storeWeak() requires that the 
    // value not change.
}
```

其中该方法就是`remove_referrer(entry, referrer);`,在`entry`中移除了`referrer`,弱引用表中不再保存指向该weak对象的引用

## weak对象释放

![](http://mainimage.hi996.com/weak2.png)
我们可以看见当一个被弱引用指向的对象释放时，会调用`weak_clear_no_lock(...)`方法

``` C++
void 
weak_clear_no_lock(weak_table_t *weak_table, id referent_id) 
{
    objc_object *referent = (objc_object *)referent_id;

    weak_entry_t *entry = weak_entry_for_referent(weak_table, referent);
    if (entry == nil) {
        /// XXX shouldn't happen, but does with mismatched CF/objc
        //printf("XXX no entry for clear deallocating %p\n", referent);
        return;
    }

    // zero out references
    weak_referrer_t *referrers;
    size_t count;
    
    if (entry->out_of_line()) {
        referrers = entry->referrers;
        count = TABLE_SIZE(entry);
    } 
    else {
        referrers = entry->inline_referrers;
        count = WEAK_INLINE_COUNT;
    }
    
    for (size_t i = 0; i < count; ++i) {
        objc_object **referrer = referrers[i];
        if (referrer) {
            if (*referrer == referent) {
               //weak指向的对象，在释放时会被置nil的关键
                *referrer = nil;
            }
            else if (*referrer) {
                _objc_inform("__weak variable at %p holds %p instead of %p. "
                             "This is probably incorrect use of "
                             "objc_storeWeak() and objc_loadWeak(). "
                             "Break on objc_weak_error to debug.\n", 
                             referrer, (void*)*referrer, (void*)referent);
                objc_weak_error();
            }
        }
    }
    
    weak_entry_remove(weak_table, entry);
}
```

如方法中，通过`referent`,找到保存指向该对象的所有引用的实例`entry`，通过循环，将`entry`中的所有引用置为`nil`即可

## weak引用表图解 
![](http://mainimage.hi996.com/weak5.png)


参考资料</br>
[http://www.jianshu.com/p/ef6d9bf8fe59](http://www.jianshu.com/p/ef6d9bf8fe59)</br>
[http://kylinroc.github.io/objc-retain-release.html](http://kylinroc.github.io/objc-retain-release.html)</br>
[http://ios.jobbole.com/89012/](http://ios.jobbole.com/89012/)


