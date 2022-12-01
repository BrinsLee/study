## 一、原理概述

ARouter 的`路由`、`参数`和`拦截器`都是用注解来标注的，跳转基于路由表 `RouterMap` 实现的，负责`生成路由表`的是 `RouteProcessor` ，负责`加载路由表`的是 `LogisticsCenter` 或 `RegisterTransform` 。

当我们调用ARouter#build()返回一个Postcard对象，调用的 navigation() 方法就是 Postcard 的 navigation() 方法，Postcard 的 navigation() 方法会调用到  _ARoute 的 `navigation()` 方法中，在这方法中，首先会处理`预处理服务`，然后会让 `LogisticsCenter` 填充 `Postcard` 中的信息，如果 LogisticsCenter 没有找到对应的路由信息的话，就会走

1. `降级策略`的逻辑，如果 LogisticsCenter 找到对应的路由信息的话，就会判断是不是走`绿色通道`，如果不走绿色通道的话就由`拦截器链`决定要不要跳转。如果走绿色通道的话，就直接按 Fragment 和 Activity 等不同的类型进行跳转，在跳转完成后，如果设置了`跳转回调`， LogisticsCenter 就会调用这个回调。
2. `预处理服务`具体就是一个 `PretreatmentService` 接口，只要定义一个实现了这个接口的类，并给这个类加一个 `@Route` 注解就可以使用了，预处理服务的作用，是做一些跳转的时候，在加载路由表前的判断。
3. `降级策略`的作用是跳转路由的信息缺失的时候，要做的事情，比如说给用户弹一个错误提示或记录错误日志等，降级策略对应的是一个 DegradeService 接口，定义一个实现这个接口的类，并添加上 @Route 注解就可以使用降级策略了。
4. `绿色通道`的作用就是判断要不要走拦截器链，比如说我们定义了一个登陆拦截器，但是某个页面不需要做这个判断，就可以走绿色通道，走绿色通道只要在调用 `build()` 方法后调用 `greenChannel()` 方法就可以了。
5. `拦截器`具体就是一个添加了 `@Interceptor` 注解并实现了 `IInterceptor` 接口的类，通过拦截器我们能做一些类似登录态判断等逻辑。
6. `跳转回调`具体就是一个传到 navigation() 方法中的 NavigationCallback 接口或 NavCallback 抽象类。

## 二、ARouter架构

![ARouter 架构-路由表.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb3dbf571371406188d35673fa893507~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

## 三、基本用法

### 1、添加依赖

![添加依赖.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/56beee0caeea4e65ad0e1c3e064147fe~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

### 2、声明路径

ARouter要用@Route注解声明跳转目标的路径，最前面 / 不能少，而且路径最少有两级

group 是可选的，ARouter 内部会对 path 进行分组，以下面这段代码为例，如果不传 group 的话，那 ARouter 会把 goods 作为该路径的 group ，否则 taobao 就是该路径所属的 group 。

```kotlin
@Route(path = "/goods/details", group = "taobao")
class GoodsDetailsActivity: AppCompatActivity() {
    //...
}
```

### 3、初始化ARouter

打印日志和开启调试模式必须写 在init() 之前，否则这些配置在初始化过程中将无效。

```kotlin
private fun initARouter() {
    if(BuildConfig.DEBUG) {
        ARouter.openLog()
        ARouter.openDebug()
    }
    ARouter.init(application)
}
```

### 4、跳转

```kotlin
ARouter.getInstance()
	// 构建明信片 设定path 与 group
	.build("/goods/details","taobao")
	.navigation()
```

### 5、明信片

当我们调用 `ARouter.getinstance().build()` 时，其实是在`创建一个明信片 Postcard` 对象，withXXX() 和 navigation() 等方法就是它的方法，Postcard 的 navigation() 方法最终调用的是 _ARouter 的 navigation() 方法

Postcard 继承了 RouteMeta，RouteMeta 是路由表的内容，而 Postcard 则包含了在跳转时的传参和动画

#### 5.1、RouteMeta

| 属性名       | 类型                   | 说明         |
| ------------ | ---------------------- | ------------ |
| type         | RouteType              | 路线类型     |
| rawType      | Element                | 路线原始类型 |
| destination  | Class<?>               | 终点         |
| group        | String                 | 路径组名     |
| path         | String                 | 路径         |
| priority     | int                    | 优先级       |
| extra        | int                    | 标志         |
| paramsType   | Map<String, Integer>   | 参数类型     |
| name         | String                 | 路线名称     |
| injectConfig | Map<String, Autowired> | 注入配置     |

#### 1. RouteType

枚举类

| 属性     |                  |
| -------- | ---------------- |
| ACTIVITY | CONTENT_PROVIDER |
| SERVICE  | BOARDCAST        |
| PROVIDER | METHOD           |
| FRAGMENT | UNKNOWN          |

#### 2. Element

原始路线类型rawType类型为Element，包含了跳转目标的 Class 信息，是由路由处理器 RouteProcessor 设定的。

#### 3. 终点

destination就是声明了@Route的跳转目标Class，比如目标Activity和Fragment的Class，这个信息也是由RouteProcessor设定的

#### 4. 路径和路线组

如果我们在 @Route 中只设定了路径，比如 path = /goods/details ，那么 goods 就是 group ，details 就是路径 path 
如果我们设置了 @Route 的 group 的值，比如 group = taobao ，那么 path 的 group 就是 taobao。

#### 5.优先级

优先级在@Route中无法设定，是给拦截器使用的，priority的值越小，拦截器的优先级越高。

拦截器链的实现类为 InterceptorServiceImpl，它在调用拦截器的 onInteceptor() 方法时，就是按照 priority 的顺序来获取拦截器，然后逐个调用的。

#### 6. 标志

这个 extra 是路由文档 RouteDoc 的标志，文档中包含了路由和服务相关的信息 ，默认不会生成对应的文件，如果想生成的话，可以在注解处理参数中设置生成文档。

- 添加参数

  如果我们想查看路由文档，可以 AROUTER_MODULE_NAME 后面添加：

  AROUTER_GENERATE_DOC: "enable"

- 文档路径

  build/generated/source/apt/(debug or release)/com/alibaba/android/arouter/docs/arouter-map-of-${moduleName}.json

路由文档大致如下：

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dc73a1cd267b415fbee84f0498b60787~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp" alt="img" style="zoom:50%;" />

#### 7. 参数类型

paramsType也就是参数类型，对于我们跳转时设定的参数，ARouter会根据不同类型给他设定一个枚举值，然后取值时，再根据不同类型调用Intent#getXXXExtra()等方法

### 5.2 Postcard

| 属性                 | 类型                 | 说明           |
| -------------------- | -------------------- | -------------- |
| uri                  | Uri                  | 统一资源标识符 |
| tag                  | Object               | 标签           |
| mBundle              | Bundle               | 传递参数       |
| flags                | int                  | 标志           |
| timeout              | int                  | 超时时间       |
| provider             | IProvider            | 自定义服务     |
| greenChannel         | Boolean              | 绿色通道       |
| serializationService | SerializationService | 序列化服务     |
| optionsCompat        | Bundle               | 共享元素动画   |
| enterAnim            | int                  | 进入动画       |
| exitAnim             | int                  | 退出动画       |

#### 1.uri

在 ARouter 中，我们可以用 URI 作为路径跳转，ARouter 会用路径替换服务 PathReplaceService 替换路径，这个服务没有默认实现，如果我们需要为不同的协议和主机替换不同的路径时，就要自己实现这个服务。

```kotlin
val uri = Uri.parse("demo://www.xxx.com/moduleA/secomd?name=aa)
ARouter.getInstance()
       .build(uri)
       .navigation()
```

#### 2.tag

用于在拦截器NavigationCallback#interrypt()中获取异常信息

```kotlin
@Interceptor(priority = 8, name = "测试拦截器")
class TestInterceptor: IInterceptor {
    override fun init(context: Context?){
        //初始化拦截器，会在SDK初始化的时候调用该方法，仅会调用一次
    }
    
    override fun process(postcard: Postcard?, callback: InterceptorCallback) {
        callback.onInterrupt(RuntimeException("糟糕，出错了"))
    }
}

ARouter.getInstance()
	  .build(path)
	  .navigation(this, object: NavigationCallback {
          override fun onInterrupt(postcard: Postcard?) {
              // 糟糕，出错了
              val errorMsg = postcard?.tag
          }
      })
```

#### 3. mBundle

调用withString等方法传入的参数要传递给跳转目标的数据时候，都是存放在这个mBundle中的

#### 4.flags

调用withFlags设定Activity启动标志时，这个标志就会赋值给flags

#### 5.timeout

拦截器处理跳转事件是放在CountDownLatch中执行的，默认超时时间是300s，也就是5分钟，所以我妈在拦截器中不要进行太过耗时的操作

#### 6.provider

实现自定义服务时，参数注解处理器AutowiredProcessor会为各个路径创建一个实现注射器ISyringe接口的类，这个类的inject()方法中，调用了ARouter.getInstance().navigation(XXXService.class)，当LogisticsCenter发现是一个Provider时，就会反射创建一个Provicer实例，然后设置给Postcard，再进行跳转

#### 7.greenChannel

所谓绿色通道，就是不会被拦截器链处理的通道，自定义服务 IProvider 和 Fragment 就是走的绿色通道。如果我们想让某个跳转操作跳过拦截器，可以在 navigation() 前调用 greenChannel() 方法。

#### 8.serializationService

当我们调用 withObject() 方法时，ARouter 就会获取我们自己自定义的序列化服务 SerializationService，然后调用该服务的 object2Json() 方法，再把数据转化为 String 放入 bundle 中。

### 6、ARouter路由表生成原理

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c88b64b69e8b4983950862860244ae4f~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp" alt="注解处理流程.png" style="zoom:60%;" />

#### 1.路由解析流程

RouteProcessor.process()

process ---> 获取路由元素 ---> 创建路由元信息 ---> 把路由元信息进行分组 ---> 通过JavaPoet生成路由文件

1. 获取路由元素

   这里的元素指的是 javax 包中的 Element ，Element 表示 Java 语言元素，比如字段、包、方法、类以及接口。

   process() 方法会接收到 annotations 和 roundEnv 两个参数，annotations 就是当前处理器要处理的注解，roundEnv 就是运行环境 JavacRoundEnvironment。

   JavacRoundEnvironment 有一个 getElementsAnnotatedWith() 方法，RouteProcessor 在处理注解时首先会用这个方法获取元素。

2. 创建路由元信息

   ![创建路由元信息.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3e6563af2b3f4ba48ab0c91dcf5e2d25~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

​	路由元信息指的是 RouteMeta，RouteProcessor 会把声明了 @Route 注解的的 Activity、Provider、Service 或 	Fragment 和一个 RouteMeta 关联起来。

​	当元素类型为 Activity 时，RouteProcessor 会遍历获取 Activity 的子元素，也就是 Activity 的成员变量，把它们放	入注入配置 RouteMeta 的 injectConfig 中，当我们配置了要生成路由文档时，RouteProcessor 就会把 		injectConfig 写入到文档中，然后就对元信息进行分组。

​	3.把路由元信息进行分组
​	在 RouteProcessor 中有一个 groupMap，在 RouteMeta 创建好后，RouteProcessor 会把不同的 RouteMeta 进	行分组，放入到 groupMap 中。

​	拿路径 /goods/details 来说，如果我们在 @Route 中没有设置 group 的值，那么 RouteProcessor 就会把 goods 	作为 RouteMeta 的 group 。

​	4.生成路由表
​	当 RouteProcessor 把 RouteMeta 分组好后，就会用 JavaPoet 生成 Group、Provider 和 Root 路由文件，路由表	就是由这些文件组成的，JavaPoet 是 Square 开源的代码生成框架。

​	生成路由文件后，物流中心 LogisticsCenter 需要用这些文件来填充仓库 Warehouse 中的 routes 和 providerIndex 	等索引，然后在跳转时根据 routes 和索引来跳转。



### 7、ARouter跳转原理

_ARouter 的 navigation() 方法有下面两种重载。

```java
navigation(final Context context, final Postcard postcard, final int requestCode, final 				  NavigationCallback callback)
  
navigation(Class<? extends T> service)
```

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1fa3f4eff94a4458a641c372563a1dcb~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp" alt="_ARouter.navigation().png" style="zoom:80%;" />

大致流程

**1.预处理服务**
根据我们实现的预处理服务判断是否需要继续处理

**2.加载路由表**
如果预处理服务返回true，则navigation方法会去加载路由表，把RouteMeta的信息填充到Postcard中，比如destination等信息，此操作是在LogisticsCenter中完成的

**3.降级策略**
如果在完善postcard过程中出现异常，就会调用降级策略，可在降级策略中显示错误提示信息

**4.拦截器链**
完善postcard后，如果不是走绿色通道，则会把事件交由拦截器链进行处理

**5.按类型跳转**

在拦截器链处理完成后，并且没有出现中断，navigation就会按照路径类型跳转到不同的页面或调用自定义服务

接下来我们来具体展开

### 1.实现预处理服务

```kotlin
// 实现PretreatmentService接口，并加上一个Path内容注解即可
@Route(path = "/xx/xx")
class PretreatmentServiceImpl : PretreatmentService  {
    // 初始化回调
    override fun init(context: Context?) {
        TODO("Not yet implemented")
    }
	// 预处理回调
    override fun onPretreatment(context: Context?, postcard: Postcard?): Boolean {
        // 返回false 表示自行处理跳转
        return false
    }
} 
```

在 navigation() 中，首先会调用预处理服务的 onPretreamtn() 方法，判断是否要继续往下处理，如果返回结果为 false ，则不再往下处理，也就是不会进行跳转等操作。

### 2.完善明信片

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c78d7b3ed51a46b897b6864e733ace40~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp" alt="LogisticsCenter.completion().png" style="zoom:67%;" />

调用完预处理服务后，_ARouter 就会用物流中心 LogisticsCenter 来加载路由表，路由表也就是 RouteProcessor 生成的路由文件。

**1.获取路由元信息**

| 属性              | 类型                                            | 说明         |
| ----------------- | ----------------------------------------------- | ------------ |
| groupsIndex       | HashMap<String, Class<? extends IRouteGroup>>   | Group索引    |
| routes            | HashMap<String, RouteMeta>                      | 路由信息映射 |
| providersIndex    | HashMap<String, RouteMeta>                      | Provider索引 |
| providers         | HashMap<Class, IProvider>                       | Provider映射 |
| interceptorsIndex | HashMap<Integer, Class<? extends IInterceptor>> | 拦截器索引   |
| interceptors      | ArrayList<Interceptors>                         | 拦截器列表   |

在 _ARouter 初始化时，会把 LogisticsCenter 也进行初始化，而 LogisticsCenter 的初始化方法中，会读取 RouteProcessor 创建好的路由表，然后放到对应的索引 index 中。

有了索引，当 _ARouter 调用 LogisticsCenter 的 completion() 方法时，就可以用索引从 Warehouse 的 routes 中获取路由元信息。

如果 LogisticsCenter 根据索引查找不到对应的 RouteMeta，那就说明 routes 还没有被填充，这时 LogisticsCenter 就会获取 group 的 RouteMeta，然后把 group 下的路径填充到 routes 中，然后再调用一次 completion() ，这时就可以取填充明信片的信息了。



