---
title: 『底层探索』13 - KVC 底层探索
toc: true
donate: false
tags: []
date: 2020-10-29 15:38:40
categories: [底层探索]
---

在 iOS 项目开发中，我们经常用 `setValue:forKey:` 和 `value:forKey:` 来访问对象的属性或成员变量，那么这两个方法的底层执行流程是怎样的呢？

<!-- more -->

# KVC 探究

KVC (Key Value Coding) 是 Apple 的一种通过 key 或者 name 来直接访问对象成员变量的一种机制。

对于继承 `NSObject` 的类，我们可以调用：

- `setValue(Any?, forKey: String)` 设置成员变量的值。
- `setValue(Any?, forKeyPath: String)` 设置成员对象中的成员变量的值。
- `value(forKey: String)` 获取成员变量的值。
- `value(forKeyPath: String)` 获取成员对象的成员变量值。
- [更多用法](https://developer.apple.com/documentation/objectivec/nsobject/nskeyvaluecoding)

## KVC 底层原理

虽然 KVC 的源代码没有开源，但也拦不住我们的探索，我们可以通过官方文档 [ Key-Value Coding Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueCoding/index.html#//apple_ref/doc/uid/10000107i) 来了解 KVC 的原理细节。

### 代码准备

首先我们定义一个 `Person` 对象和 `Pet` 对象。

```objc
@interface Pet : NSObject {
    @public
    NSString * _name;
}

@end

@implementation Pet

@end
```

```objc
@interface Person : NSObject {
    @public
    NSString *isName;
//    NSString *name;
//    NSString *_isName;
//    NSString *_name;
    
    NSInteger  _age;
}


@property (nonatomic,assign) double  weight;
@property (nonatomic,strong) Pet *pet;
@property (nonatomic,strong) NSArray * hobbies;

@end

@implementation Person

@end
```

### Set

对于如下的代码，当我们通过 `setValue:forKey` 设置对象的值的时候，底层会经历哪些流程呢。

```objc
Person *person = [Person alloc];
[person setValue: @"muhlenXi" forKey:@"name"];
```

在 [Accessor Search Patterns](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueCoding/SearchImplementation.html#//apple_ref/doc/uid/20000955-CJBBBFFA) 章节中，我们找到了答案, 主要会经历如下的流程：

- 1、首先会查找当前类是否实现了 `set<Key>` 或者 `_set<Key>` 方法，如果实现了，就调用该方法并返回，否则跳转 2。
- 2、判断 `accessInstanceVariablesDirectly` 方法的返回值是否为 YES。这个方法的默认实现返回的是 YES。如果是 NO 则调用 `setValue:forUndefinedKey:` 方法进行报错。是 YES，跳转 3。
- 3、在成员变量列表中依次查找如下的变量 `_<key>`, `_is<Key>`, `<key>`, `is<Key>`，查找到则进行赋值。如果仍然查找不到，则调用 `setValue:forUndefinedKey:` 方法进行报错。

### Get

那么，当我们调用 `valueForKey` 取值的时候，底层会经历哪些流程呢？

```objc
NSString *readName = [person valueForKey:@"name"];
```

文档上是这样描述的：

- 1、首先查找当前类是否实现了 `get<Key>`, `<key>`, `is<Key>`, `_<key>` 方法，如果其中的某个有实现，就调用该方法并返回，否则跳转 2。
- 2、检查当前类是否可以响应 `countOf<Key>` 和 `objectIn<Key>AtIndex` 方法，如果可以响应，将集合类中的对象进行组装后返回。（这主要针对的是 NSArray 类的成员变量。） 如果不能响应则跳转 3。
- 3、检查当前类是否可以响应 `countOf<Key>`, `enumeratorOf<Key>` 和 `memberOf<Key>` 方法，如果可以响应，将集合类中的对象进行组装后返回。（这主要针对的是 NSSet 类的成员变量。） 如果不能响应则跳转 4。
- 4、判断 `accessInstanceVariablesDirectly` 方法的返回值是否为 YES。这个方法的默认实现返回的是 YES。如果是 NO 则调用 `valueForUndefinedKey:` 方法进行报错。是 YES，跳转 5。
- 5、在成员变量列表中依次查找如下的变量 `_<key>`, `_is<Key>`, `<key>`, `is<Key>`，查找到则进行取值后返回。如果仍然查找不到，则调用 `valueForUndefinedKey:` 方法进行报错。

通过本文的 Demo [mock-KVC-KVO](https://github.com/muhlenXi/mock-KVC-KVO) 可以验证上述的流程是否正确。

## 尝试去实现 KVC

搞清楚了上面 set 和 get 的流程，我们可以尝试自己去实现简易版的 KVC 机制。

我们可以创建一个 NSObject 的 Category，然后实现我们的方法。

```objc
- (void) mx_setValue:(id)value forKey:(NSString *)key;
- (id)mx_valueForKey:(NSString *)key;
```

### mx_setValue

简单起见我们仅仅考虑一种情况，就是 NSString 类型的成员变量。在 `mx_setValue` 方法中，我们完成如下的四个流程就可以了。

- 1、key 安全校验
- 2、setter 方法检查和调用
- 3、检查 accessInstanceVariablesDirectly
- 4、查找成员变量并赋值

简易代码如下，完整代码参考 demo。

```
- (void)mx_setValue:(id)value forKey:(NSString *)key {
    // 1 - 安全校验
    if (key == nil || key.length == 0) {
        return;
    }
    
    // 2 - setter 方法检查和调用
    NSString *setKey = [NSString stringWithFormat:@"set%@:", key.capitalizedString];
    NSString *_setKey = [NSString stringWithFormat:@"_set%@:", key.capitalizedString];
    
    if ([self mx_performSelector:setKey withObject:value]) {
        return;
    } else if ([self mx_performSelector:_setKey withObject:value]) {
        return;
    }
    // 3 - 检查 accessInstanceVariablesDirectly
    if (![self.class accessInstanceVariablesDirectly]) {
        @throw [NSException exceptionWithName:@"MXUnknownKeyException" reason:[NSString stringWithFormat:@"[%@ valueForUndefinedKey:]: this class is not key value coding-compliant for the key name.", self] userInfo:nil];
    }
    
    // 4 - 查找成员变量并赋值
    NSArray *ivarNames = [self getIvarListName];
    NSString *_key = [NSString stringWithFormat:@"_%@", key];
    NSString *_isKey = [NSString stringWithFormat:@"_is%@", key.capitalizedString];
    NSString *isKey = [NSString stringWithFormat:@"is%@", key.capitalizedString];
    
    NSArray *keys = @[_key, _isKey, key, isKey];
    for(NSString *key in keys) {
        if ([ivarNames containsObject:key]) {
            Ivar ivar = class_getInstanceVariable([self class], key.UTF8String);
            object_setIvar(self, ivar, value);
            return;
        }
    }
    
    // 异常提醒
    @throw  [NSException exceptionWithName:@"MXUnknownKeyException" reason:[NSString stringWithFormat:@"[%@ %@]: this class is not key value coding-compliant for the key name.", self, NSStringFromSelector(_cmd)] userInfo:nil];
}

```

### mx_valueForKey

在这个方法中，我们实现下面的主要流程即可。

- 1、安全校验
- 2、getter 方法检查和调用
- 3、检查 accessInstanceVariablesDirectly
- 4、成员变量查找

简易代码如下，完整代码参考 demo。

```objc
- (id)mx_valueForKey:(NSString *)key {
    // 1 - 安全校验
    if (key == nil || key.length == 0) {
        return nil;
    }
    
    // 2 - getter 方法检查和调用
    NSString *getKeyM = [NSString stringWithFormat:@"get%@", key.capitalizedString];
    NSString *isKeyM = [NSString stringWithFormat:@"is%@", key.capitalizedString];
    NSString *_keyM = [NSString stringWithFormat:@"_%@", key];
    NSString *countOfKey = [NSString stringWithFormat:@"countOf%@", key.capitalizedString];
    NSString *objectInKeyAtIndex = [NSString stringWithFormat:@"ObjectIn%@AtIndex:", key.capitalizedString];
    
    
    NSArray *methodNames = @[getKeyM, isKeyM, key, _keyM];
    
    for(NSString *name in methodNames) {
        id value = [self checkPerformSelector: name];
        if (value != nil) {
            return value;
        }
    }
    if ([self respondsToSelector:NSSelectorFromString(countOfKey)]) {
        if ([self respondsToSelector:NSSelectorFromString(objectInKeyAtIndex)]) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
            int num = (int)[self performSelector:NSSelectorFromString(countOfKey)];
            NSMutableArray *mArray = [NSMutableArray arrayWithCapacity:1];
            for(int i = 0; i < num; i++) {
                id object = [self performSelector:NSSelectorFromString(objectInKeyAtIndex) withObject:@(i)];
#pragma clang diagnostic pop
                [mArray addObject:object];
            }
        }
    }
    
    // 3 - 检查 accessInstanceVariablesDirectly
    if (![self.class accessInstanceVariablesDirectly]) {
        @throw [NSException exceptionWithName:@"MXUnknownKeyException" reason:[NSString stringWithFormat:@"[%@ valueForUndefinedKey:]: this class is not key value coding-compliant for the key name.", self] userInfo:nil];
    }
    
    // 4 - 成员变量查找
    
    NSString *_key = [NSString stringWithFormat:@"_%@", key];
    NSString *_isKey = [NSString stringWithFormat:@"_is%@", key.capitalizedString];
    NSString *isKey = [NSString stringWithFormat:@"is%@", key.capitalizedString];
    
    NSArray *keys = @[_key, _isKey, key, isKey];
    NSArray *names = [self getIvarListName];
    for(NSString *key in keys) {
        if ([names containsObject:key]) {
            Ivar ivar = class_getInstanceVariable([self class], key.UTF8String);
            return object_getIvar(self, ivar);
        }
    }
    
    return nil;
}
```

## 总结

如果想详细的了解 Apple 的 KVC 源码实现细节，下面的两个资料是值得推荐的。掌握了 KVC 的底层原理，在使用 KVC 的时候，我们可以更加的得心应手、游刃有余。

- [GNUstep](http://www.gnustep.org/resources/downloads.php) GNU 计划的项目之一。它将 Cocoa（前身为 NeXT 的 OpenStep ）Objective-C软件库，部件工具箱（widget toolkits）以及其上的应用软件，以自由软件方式重新实现。它能够运行在类Unix 操作系统上，也能运作在 Microsoft Windows 上。
- [DIS_KVC_KVO](https://github.com/renjinkui2719/DIS_KVC_KVO), 这是根据IOS Foundation 框架汇编反写的 KVC,KVO 实现，可以在 mac, iOS 环境下运行调试。

## 后记

我是穆哥，卖码维生的一朵浪花。我们下期见。

## 参考资料

- [1、Key-Value Coding Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueCoding/index.html)
