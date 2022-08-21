## hook实例、类方法通用方案
通常对于Public类及对应方法，可以直接通过`method_exchangeImplementations(arg1, arg2)`进行交换，但是对于Private类该如何处理呢，直接上代码：
假设Student类为私有类，如下：
```
@interface Student : NSObject
/// 类方法
+ (void)listen;
/// 实例方法
- (void)study;
@end

@implementation Student

+ (void)listen {
    NSLog(@"Student listen");
}

- (void)study {
    NSLog(@"Student study");
}

@end
```
提供一个SwizzleTool类来hook对应实例、类方法：
```
@implementation SwizzleTool

+ (void)load {
    // 1. 通过映射方式来获取对应类及方法标识Selector，得到 初始方法
    Method origin = class_getInstanceMethod(NSClassFromString(@"Student"), NSSelectorFromString(@"study"));
    // 2. 获取需要交换的方法
    Method swizzle = class_getInstanceMethod([self class], @selector(swizzleStudy));
    // 3. Student类添加 swizzleStudy 方法，但是实现是指向了 study，就是为了保证在方法交换之后，执行[self swizzleStudy]可以执行 初始study方法
    class_addMethod(NSClassFromString(@"Student"), @selector(swizzleStudy), method_getImplementation(origin), method_getTypeEncoding(origin));
    // 4. 
    BOOL result = class_addMethod(NSClassFromString(@"Student"),  NSSelectorFromString(@"study"), method_getImplementation(swizzle), method_getTypeEncoding(swizzle));
    method_exchangeImplementations(origin, swizzle);
}

+ (void)swizzleStart {
    Method origin = class_getInstanceMethod(Student.class, @selector(study));
    Method swizzle = class_getInstanceMethod([self class], @selector(swizzleStudy));
    // 交换类添加对应方法
    class_addMethod(Student.class, @selector(swizzleStudy), method_getImplementation(origin), method_getTypeEncoding(origin));
    method_exchangeImplementations(origin, swizzle);
    
    Method originCls = class_getClassMethod(objc_getMetaClass([NSStringFromClass(Student.class) UTF8String]), @selector(listen));
    Method swizzleCls = class_getClassMethod(objc_getMetaClass([NSStringFromClass([self class]) UTF8String]), @selector(swizzleListen));
}

#pragma mark - swizzle implemention
- (void)swizzleStudy {
    [self swizzleStudy];
    NSLog(@"swizzleStudy");
}

+ (void)swizzleListen {
    [self swizzleListen];
    NSLog(@"swizzleListen");
}

@end
```
