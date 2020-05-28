# seriesTechnique
## Git
## iOS Object-C
## 

### 消息转发
`_objc_msgForward`,为了让替换的方法走 forwardInvocation, 把它指向一个不存在的IMP:`calss_getMethodImplementation(cls, @selector(__xxxSelector))`,实际上这样痴线是多余的，若`class_getMethodImplementation`找不到class/selector对应的IMP,会返回`_objc_msgForward`这个IMP,所以更直接的方式是把要替换的方法都指向`_objc_msgForward`,省去查找方法的时间
