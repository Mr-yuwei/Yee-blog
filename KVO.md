## KVO


### 名词


行为型模式（Behavioral Pattern）是对在不同的对象之间划分责任和算法的抽象化。<br>
行为型模式不仅仅关注类和对象的结构，而且重点关注它们之间的互相作用。<br>

通过行为型模式，可以更加清晰地划分类与对象的职责，并研究系统在运行时实例对象之间的交互。在系统运行时，对象并不是孤立的，它们可以通过相互通信与协作完成某些复杂功能，**一个对象在运行时也将影响到其他对象的运行**。


行为型模式分为`类行为型模式`和`对象行为型模式`两种：<br>

* 类行为型模式:类的行为型模式使用继承关系在几个类之间分配行为，类行为模式主要通过多态等方式来分配父类与子类的职责。
* 对象行为型模式：对象的行为型模式则使用对象的聚合关系来分配行为，对象行为型模式主要是通过对象关联等方式来分配两个或多个类的职责。根据“合成复用原则”，系统中要尽量使用关联关系来取代继承关系，因此大部分行为型设计模式都属于对象行为型设计模式。


`合成复用原则（Composite resuse Principle ）`又叫`组合/聚合复用原则`,它要求软件复用时，要尽量使用组合或者聚合等关系来实现，其次才考虑使用继承关系来实现。



### 观察者设计模式

观察者模式的定义：指多个对象存在一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。它是行为对象型模式。

*  优点
  1. 降低了目标与观察者之间的耦合关系，两者之间是抽象耦合关系
  2. 目标与观察者之间建立了一套触发机制。

*  缺点
  
  1. 目标与观察者之间的依赖关系并没有完全解除，而且有可能出现循环引用。
  2. 当观察者对象很多时，通知的发布会花费很多时间，影响程序的效率。



* 模式的结构 	 
  1. 抽象主题（Subject）角色：也叫抽象目标类，它提供了一个用于保存观察者对象的聚合类和增加，删除观察者对象的方法，以及通知所有观察者的抽象方法。
  2. 具体主题（Concrete Subject）角色，也叫具体目标类，它实现抽象类中的通知方法，当具体主题的内部状态发生改变时，通知所有注册过的观察者对象。
  3. 抽象观察者（Observer）角色:它是一个抽象类或接口，它包含了一个更新自己的抽象方法，当接到具体主题的更改通知时被调用。
  4. 具体观察者（Concrete Observer）角色：实现抽象观察者中定义的抽象方法，以便在得到目标的更改通知时更新自身的状态。

* UML 图
   
![UML图](http://c.biancheng.net/uploads/allimg/181116/3-1Q1161A6221S.gif)



### 定义

KVO全称是`Key Value Observing`，是苹果提供的一套事件通知机制。允许对象监听另一个对象特定属性的改变，并在改变时接收到事件。

### API

```
- (void)addObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options context:(nullable void *)context;
- (void)removeObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath context:(nullable void *)context;
- (void)removeObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath
```

```
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context
```

###  如何使用


```
self.person1 = [[Person alloc] init];
// 添加观察者
[self.person1 addObserver:self forKeyPath:@"age" options:NSKeyValueObservingOptionNew|NSKeyValueObservingOptionOld context:nil];
self.person1.age = 20;
//移除观察者
 [self.person1 removeObserver:self forKeyPath:@"age"];
 self.person1.age = 100;
```


```
// 收到属性变化通知
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context{
    NSLog(@"对象%@ 值 %@ 改变%@ context %@",object,keyPath,change,context);
}
```

###  带着问题出发

 * 注册的Observer观察者放在哪里啦，存储的数据结构是怎么样，又是如何移除的。
 * 执行完addObserver操作之后,观察者发生了什么变化，被观察者又是怎样变化的，是否自己可以实现一个KVO。
 * 比较KVO,delegate,Block的优劣。【待续】
 * KVO常见的问题有那些，有什么好的解决方法吗，举例说明。



### 如何一步步发现KVO的本质


执行下面的代码：

```
- (void)func {
    self.person1 = [[Person alloc] init];
    self.person2 = [[Person alloc] init];
    NSLog(@"之前");
    NSLog(@"person1改变-- %@--%@",[self.person1 class],object_getClass(self.person1));
    NSLog(@"person2改变--%@--%@",[self.person2 class],object_getClass(self.person2));
    
    [self.person1 addObserver:self forKeyPath:@"age" options:NSKeyValueObservingOptionNew|NSKeyValueObservingOptionOld context:nil];
    NSLog(@"之后");
    NSLog(@"person1改变--%@--%@",[self.person1 class],object_getClass(self.person1));
    NSLog(@"person2改变--%@--%@",[self.person2 class],object_getClass(self.person2)); 
}
```

打印结果

```
 之前
 Person Person
 Person Person
 之后
 Person NSKVONotifying_Person
 Person Person
```

可以发现添加完`addObserver:forKeyPath:options:context:`之后的结果可以看出:

* ` self.person1`的`class`方法与` self.person2`的`class`返回的结果相同。
* `object_getClass `获取的结果不同分别为`NSKVONotifying_Person `，`Person` ，这里也说明`self.person1`的isa指向发生了改变，从`Person`修改为了`NSKVONotifying_Person `。



验证 `NSKVONotifying_Person ` 与 `Person`的关系,执行下面的代码。

```
  NSLog(@"%p",class_getSuperclass(object_getClass(self.person1)));
  NSLog(@"%p",object_getClass(self.person2));
```

打印结果：<br>

```
0x101582eb0
0x101582eb0
```

得到下面的结论：(这里需要注意`self.person1`已经添加过addObserver)

| 对象 |类对象 |类对象SuperClass 
| --- | ----------- |  ----------- |----------- 
|self.person1 | NSKVONotifying_Person |Person
| self.person2 | Person |NSObject

进而说明，`NSKVONotifying_Person`是Person的子类。<br>
那么这个`NSKVONotifying_Person`是何时生成的？？？

```
- (void)func2 {
    Class cla1  = NSClassFromString(@"NSKVONotifying_Person");
    [self.person1 addObserver:self forKeyPath:@"age" options:NSKeyValueObservingOptionNew|NSKeyValueObservingOptionOld context:nil];
    Class cla2  = NSClassFromString(@"NSKVONotifying_Person");
    NSLog(@"%@ %@",cla1,cla2);
    }
```
只有`cla2`有值，说明只有`addObserver `后才会生成`NSKVONotifying_Person `. <br>


回到`class`方法,打印下面的方法:

```
- (void)printMethodNamesOfClass:(Class)cls{
    unsigned int count;
    Method *methods  = class_copyMethodList(cls, &count);
    NSMutableString *methodNames = [[NSMutableString alloc] init];
    for (unsigned int i = 0; i < count; i++) {
        Method method  = methods[i];
        NSString *methodName  = NSStringFromSelector(method_getName(method));
        [methodNames appendString:methodName];
        [methodNames appendString:@","];
    }
    free(methods);
    NSLog(@"cls == %@ methods ==  %@",cls,methodNames);
    cls = class_getSuperclass(cls);
    if (cls && cls != [NSObject class]) {
        [self printMethodNamesOfClass:cls];
    }
}
```


```
 cls == Person methods ==  data,setData:,.cxx_destruct,block,height,setHeight:,setBlock:,age,setAge:,
 cls == NSKVONotifying_Person methods ==  setAge:,class,dealloc,_isKVOA,
 cls == Person methods ==  data,setData:,.cxx_destruct,block,height,setHeight:,setBlock:,age,setAge:
```

说明新生成的`NSKVONotifying_Person `比 `Person `多了`setAge:,class,dealloc,_isKVOA`这几个方法。这也是获取`[self.person1 class] == [self.person2 class]  `的原因。


那新生成的`NSKVONotifying_Person `中的 `setAge `的具体实现到底是什么？？<br>
执行下面的代码<br>

```
- (void)func{
    Method method = class_getInstanceMethod(object_getClass(self.person1), @selector(setAge:));
    IMP imp = method_getImplementation(method);
    //在这里打断点
    NSLog(@"2323");
}
```


```
(IMP) imp = 0x00007fff207a7195 (Foundation`_NSSetLongLongValueAndNotify)
```

具体的实现是`Fundation`里面的`_NSSetLongLongValueAndNotify `函数.<br>

换个属性`name`试试<br>
```
    [self.person1 addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew|NSKeyValueObservingOptionOld context:nil];
    Method method = class_getInstanceMethod(object_getClass(self.person1), @selector(setName:));
    IMP imp = method_getImplementation(method);
```

```
(IMP) imp = 0x00007fff207a6553 (Foundation`_NSSetObjectValueAndNotify)
```

说明会根据属性的类型对应不同的实现方法`_NSSetLongLongValueAndNotify`,`_NSSetObjectValueAndNotify `，`_NSSetXAndNotify `

`_NSSetXAndNotify`内部的具体实现又是怎么样的？很好奇。


在网上查询，奈何都查不到具体的实现，有Hoper推测，有利用Xcode汇编的。

[DIS_KVC_KVO](https://github.com/renjinkui2719/DIS_KVC_KVO)这个仓库还原了具体的实现，伪代码如下。

```
[object willChangeValueForKey:key];

IMP imp = class_getMethodImplementation(info->originalClass, selector);
setValueWithImplementation(imp);

[object didChangeValueForKey:key];
```
```
void DSKeyValueDidChange(id object, id keyOrKeys){
  DSKeyValueNotifyObserver(......);
}

void DSKeyValueNotifyObserver(id observer,NSString * keyPath, id object, void *context){
   DSKVONotify(observer, keyPath, object, *changeDictionary, context);
}
void DSKVONotify(id observer, NSString *keyPath, id object, NSDictionary *changeDictionary, void *context) {
    [observer observeValueForKeyPath:keyPath ofObject:object change:changeDictionary context:context];
}
```
`_NSSetObjectValueAndNotify `干了这三样事情。
* 调用 `willChangeValueForKey `方法
* 调用父类的`setKey`方法
* 调用`didChangeValueForKey `方法，在这个方法里面去通知具体的观察者 属性发生了变更。


总结：<br>

* 重新生成一个以`NSKVONotifying_`为前缀的新Class并继承原来的类，然后将对象的isa指向这个``NSKVONotifying_``类。
* 重写`Class`，`setKey`,`dealloc `方法，新增`_isKVOA `方法。需要注意的是`NSKVONotifying_ `的setKey方法的具体实现会指向`_NSSetXAndNotify `这样的方法。


### 观察者是如何存储的

#### 发现 NSKeyValueObservationInfo

NSObject里面有这样一个属性,`observationInfo `

```
@property (nullable) void *observationInfo
```

执行下面的代码，并打断点。

```
 NSLog(@"%@",self.person1.observationInfo);
```


打印之后我们会发现`observationInfo `实际上是`NSKeyValueObservationInfo`的实例对象，只是系统向我们隐藏了。<br>

**每个对象的`observationInfo `会存放在一个全局Dictionary里面，key是当前对象的地址。**

####  搞定NSKeyValueObservationInfo

通过打印`observationInfo `的方法，实例变量，属性等，可以大胆想想`NSKeyValueObservationInfo `的结构如下。

```
@interface NSKeyValueObservationInfo : NSObject
{
    NSArray  <NSKeyValueObservance*> *_observances;
    long long _cachedHash;
    bool  _cachedIsShareable;
}
- (instancetype)initWithObservances:(NSArray*)observances
                              count:(NSInteger)count
                          hashValue:(BOOL)hashValue;

@end

```


```
@interface NSKeyValueObservance : NSObject
{
  NSObject *_observer;
  NSObject *_originalObservable;
  NSKeyValueProperty *_property;
  NSKeyValueObservingOptions _options;
  void * _context;
}
- (instancetype)initWithObserver:(NSObject*)observer
                        property:(NSKeyValueProperty*)propert
                         options:(NSKeyValueObservingOptions)options
                         context:(void *)context
              originalObservable:(NSObject*)originalObservable;

@end
@interface NSKeyValueProperty : NSObject{
    NSString *keyPath;
}
@end
```

### 如何自己实现KVO


参考链接 [如何自己动手实现 KVO](https://tech.glowing.com/cn/implement-kvo/) <br>


#### 关键代码

* 创建一个新类

```
Class originalClass = object_getClass(self);
Class kvoClazz      = objc_allocateClassPair(originalClass, kvoClassName.UTF8String, 0);
Method clazzMethod = class_getInstanceMethod(originalClass, @selector(class));
const char *types = method_getTypeEncoding(clazzMethod);
class_addMethod(kvoClazz, @selector(class), (IMP)kvo_class,types);
objc_registerClassPair(kvoClazz);
```


* 修改指针指向

```
 clazz = newClass
 object_setClass(self, clazz);
```

* 重写set方法,将IMP指向自己定义的setter

```
const char *types = method_getTypeEncoding(setterMethod);
class_addMethod(clazz, setterSelector, (IMP)kvo_setter, types);
```

* 调用父类的set方法

```
struct objc_super supperclasszz = {.receiver = self,.super_class = class_getSuperclass(object_getClass(self))};
void (*objc_msgSendSuperCasted)(void*,SEL,id) = (void*)objc_msgSendSuper;
objc_msgSendSuperCasted(&supperclasszz,_cmd,newValue);
```

### KVO常见闪退问题

* KVO添加次数和移除次数不匹配
* 添加或者移除时`keypath == nil`，导致崩溃。
* 添加了观察者，但未实现`observeValueForKeyPath:ofObject:change:context:`方法，导致崩溃
*  被观察者提前被释放，被观察者在Dealloc时仍然注册着KVO，导致崩溃。例如观察者是局部变量的情况（iOS 10及之前会崩溃）。

##### FBKVOController是如何解决的


[FBKVOController源码分析](KVOController源码.md)



### 相关链接

[KVO原理分析](https://www.jianshu.com/p/badf5cac0130)  <br>
 [行为型模式](https://design-patterns.readthedocs.io/zh_CN/latest/behavioral_patterns/behavioral.html) <br>
 [观察者设计模式](http://c.biancheng.net/view/1390.html) 