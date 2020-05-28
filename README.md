# seriesTechnique
## Git
## iOS Object-C

## 知识点汇总
   1. Runtime专题 -- https://halfrost.com/objc_runtime_isa_class/
   2. Runloop专题 -- https://blog.ibireme.com/2015/05/18/runloop/#more-41710
   3. 多线程讲解 -- https://segmentfault.com/a/1190000002712678
   4. 启动加载
   5. 崩溃专题
   6. 编译、链接与加载
   7. clang学习，llvm

### 消息转发
`_objc_msgForward`,为了让替换的方法走 forwardInvocation, 把它指向一个不存在的IMP:`calss_getMethodImplementation(cls, @selector(__xxxSelector))`,实际上这样痴线是多余的，若`class_getMethodImplementation`找不到class/selector对应的IMP,会返回`_objc_msgForward`这个IMP,所以更直接的方式是把要替换的方法都指向`_objc_msgForward`,省去查找方法的时间
