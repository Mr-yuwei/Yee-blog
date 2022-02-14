## 在现有App中接入flutter，如何改造路由层代价最小

我能想到的

*  与原生模块隔离
*  与原生公用一套页面跳转逻辑
*  flutter如果接入了多个模块的多功能，消息规范可能还不一样，如何兼容。


###  从页面跳转说起

webView，reactNative,flutter页面与原生交互无非就这两种。

 * flutter打开原生页面
 * 原生页面跳转flutter页面

###  现有条件

* BCRouter路由以及拦截器
* flutter端 使用的了flutterBoost组件

####  BCRouter路由以及拦截器


`BCRouterService`是以URL-Block的方式实现的路由,将URL与页面建立联系，页面跳转的时候通过url找到对应的映射关系。

主要方法如下：

```
//以scheme区分模块或者不同的app
+ (instancetype)routesForScheme:(NSString *)scheme;
//以serviceName【URL】与Block绑定，进行页面跳转
- (void)registerServiceName:(NSString*)serviceName
                   impClass:(Class)impClass
                    handler:(nullable BCRouterCustomHandler)handler;
    //URL查询对应的Block，进行页面的跳转   
 + (BOOL)openURL:(NSURL *)URL
     withParams:(NSDictionary<NSString *, NSDictionary<NSString *, id> *> *)params;

```



```
   //URL查询对应的Block，进行页面的跳转       
+ (BOOL)openURL:(NSURL *)URL
     withParams:(nullable NSDictionary<NSString *, NSDictionary<NSString *, id> *> *)params
        andThen:(nullable void(^)(NSString *pathComponentKey, id obj, id returnValue))then completion:(void(^)(NSError *error, id response))completion{
    if (![self canOpenURL:URL]) {
        return NO;
    }
    NSString *scheme = URL.scheme;
    BCRouterService *router = [self routesForScheme:scheme];
    NSString *serviceImpl   = [[router servicesDict] objectForKey:[serviceName lowercaseString]];
    BCServicePathComponent *component = [[router servicesDict] objectForKey:[serviceName lowercaseString]];
    //找到对应的实现Class
    Class mClass = NSClassFromString(component.impClass);
}
```



拦截器:通过URL查找的时候，首先询问拦截器是否能够处理 URL，如果能处理就将URL交给拦截器处理。
比如登录拦截器，将需要登录的页面对应的URL都丢到loginInterceptor中，页面跳转时，url会在登录拦截器中响应。



拦截器部分方法

```
@protocol BCRouterInterceptorProtocol <NSObject>

//拦截器优先级 越大越优先
- (NSInteger)priority;
//查询是否已经注入到拦截器
-  (BOOL)canOpenURL:(NSURL *)URL;
//依赖注入
- (void)registerServiceName:(NSString*)serviceName
                   impClass:(Class)impClass;

//处理响应【返回NO代表不需要响应，返回YES说需要进行其他处理】
//返回YES之后，会执行内部处理逻辑
- (BOOL)dealWithURL:(NSURL*)URL parms:(NSDictionary<NSString *, id>*)parms
                    andThen:(void(^)(void))then;
- (BOOL)dealWithURL:(NSURL*)URL andThen:(void(^)(void))then;
```



 拦截器处理URL，不能处理，再交给路由去处理。

```
    BOOL canDealURL = BCGlobal_RouteInterceptorListDealURL(URL,allParams);
    if (canDealURL) {
        return YES;
    }
```


```
+ (void)registerInterceptor:(id<BCRouterInterceptorProtocol>)interceptor{
    BCGlobal_RouteInterceptorListAdd(interceptor);
}
static BOOL BCGlobal_RouteInterceptorListDealURL(NSURL *URL,NSDictionary*parms) {
    BCGlobal_RouteInterceptorListInit();
    BOOL canDeal = NO;
    dispatch_semaphore_wait(BCGlobal_InterCeptorlistLock, DISPATCH_TIME_FOREVER);
    for (id<BCRouterInterceptorProtocol>object in BCGlobal_routeInterceptorList) {
        canDeal = [object dealWithURL:URL parms:parms andThen:^{}];
        if (canDeal) break;
    }
    dispatch_semaphore_signal(BCGlobal_InterCeptorlistLock);
    return canDeal;
}
```

#### flutterBoost在iOS端的相关实现


`FlutterBoostDelegate`协议实现，flutter端打开、关闭原生页面，会执行到下面的具体实现。


```
///如果框架发现您输入的路由表在flutter里面注册的路由表中找不到，那么就会调用此方法来push一个纯原生页面
- (void) pushNativeRoute:(NSString *) pageName arguments:(NSDictionary *) arguments;

///当框架的withContainer为true的时候，会调用此方法来做原生的push
- (void) pushFlutterRoute:(FlutterBoostRouteOptions *)options;

///当pop调用涉及到原生容器的时候，此方法将会被调用
- (void) popRoute:(FlutterBoostRouteOptions *)options;

```


原生打开flutter页面，我们需要借助 ` [[FlutterBoost instance]open:options]`这样的操作。

```
 FlutterBoostRouteOptions *options = [[FlutterBoostRouteOptions alloc] init];
 options.pageName = @"flutter/testPage2";
 [[FlutterBoost instance]open:options];
```


### 思考问题


如果遇到如下问题，我们该怎么办？？ 

*  app已经唤醒，停留在具体的某一页面  推送通知打开app内的任意页面 
*  某一个具体页面，点击cell时候跳转的链接由后台配置


容易想到的解决方法是这样

```
if (type == 原生) {
   if (id  == "页面1"){
      vc = ...
   }else if (id  == "页面2"){
      vc = ...
   }else if (id  == "页面3"){
      vc = ...
   }else if (id  == "..."){
      vc = ...
   }
   //做页面跳转
   [self.currentVC.navigationController pushViewController: vc animated:YES];
}else{
  flutter页面跳转 
   FlutterBoostRouteOptions *options = [[FlutterBoostRouteOptions alloc] init];
 options.pageName = @"flutter/testPage2";
 [[FlutterBoost instance]open:options];
}
```

想想这些代码散在app的各个角落，新增或修改的时候成本只会越来越大。<br>
但是别忘了我们的路由层可以帮我们解决原生那一堆判断操作。改成下面的代码如下面。

```
if (type == 原生) {
 NSURL *URL = ...
 [BCRouterService openURL: URL]
}else{
// flutter页面跳转 
 FlutterBoostRouteOptions *options = [[FlutterBoostRouteOptions alloc] init];
 options.pageName = @"flutter/testPage2";
 [[FlutterBoost instance]open:options];
}
```

还是有问题，只要遇到原生与flutter都要这样写还是很麻烦，还没法扩展。<br>
如何可以写成下面这样的，既做到多端统一，也解决了硬编码的问题。

```
 NSURL *URL = ...
 [BCRouterService openURL: URL]
```

这样写，就代表着我们需要在router层里面这样写.

```
if (URL == 原生) {
  //处理原生部分
}else{
// flutter页面跳转 
 FlutterBoostRouteOptions *options = [[FlutterBoostRouteOptions alloc] init];
 options.pageName = @"URL";
 [[FlutterBoost instance]open:options];
}
```
上面的写法，已经解决了我们90%的问题，代码只会在路由层，维护路由层就行。<br>
但是我们要思考这样写是不是：用到路由组件的应用都要导入flutter组件，显然这违反了一个公共组件上下依赖关系【怎么形容，就是不应该这样写】。那我们又不能省略这段代码，有没有方法把这段代码隔离出来，有的，我们的拦截器。



#### flutter拦截器

我刚开始写的时候想到这个组件肯定要满足下面的条件：
 
 1. 将flutter业务与原生业务隔离，可插拔
 2. flutter业务可继续扩展，flutter中的多个模块可细分，只需要在这个全局的拦截器中做文章


flutter牵涉到原生的逻辑该怎么办？？,比如登录逻辑，我要写在那个模块合适？

flutter模块的全部功能原则上都是独立，登录必须写在这个拦截器里面。不能与原生的拦截器搞在一块。当然写在一块功能肯定能实现，但是后续的维护成本可是不小，必须在开始的时候就把规则定好。




