### MVC

图解

![MVC.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c1a5c3174bd24bf39be1a6b9e85e6736~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

- View：负责与用户交汇，显示界面。
- Controller：负责接收来自view的请求，处理业务逻辑。
- Model：负责数据逻辑，网络请求数据以及本地数据库操作数据等

在MVC架构中，Controller是业务的主要承载者，几乎所有的业务逻辑都在Controller中进行编写。而View主要负责UI逻辑，而Model是数据逻辑，彼此分工。

在Android中，view一般使用xml进行编写，但xml的能力不全面，需要Activity进行一些UI逻辑的编写，因而MVC中的`V`即为xml+Activity。Model数据层，在Android中负责网络请求和数据库操作，并向外暴露接口。Controller是争议比较多的写法：一种是直接把Activity当成Controller；一种是独立出Controller类，进行逻辑分离。比较符合MVC思想的笔者认为是后者。因为前者直接在Activity中进行书写业务逻辑就会和UI逻辑混合在一起了，达不到模块分工的效果。MVC架构的处理流程一般是：

- view接收用户的点击
- view请求controller进行处理或直接去model获取数据
- controller请求model获取数据，进行其他的业务操作
- 这一步可以有多种做法：
  - 利用callBack从controller进行回调
  - 把view实例给controller，让controller进行处理
  - 通知view去model获取数据

改进方向

- 对模块进行更加彻底的分离，不要让view和model直接联系。
- 对controller进行减压。



### MVP

图解

![MVP.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0d0184fa2f93476980967e464b4c5d80~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

- View：UI模块，负责界面显示和与用户交汇。
- Presenter：负责业务逻辑，起着连接View和Model桥梁的作用。
- Model：专注于数据逻辑。

为了解决MVC中代码的耦合严重性，把业务逻辑都抽离到了Presenter中。这样**View和Model完全被隔离，实现了单向依赖，大大减少了耦合度**。view和prensenter之间通过接口来通信，只要定义好接口

在Android中，需要让Activity提供控件的更新接口，prensenter提供业务逻辑接口，Activity持有presenter的实例，prensenter持有Activity的弱引用（不用直接引用是为了避免内存泄露），Activity直接调用prensenter的方法更新界面，prensenter去model获取数据之后，通过view的接口更新view。如下图：

![MVP-android.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7a2f9d30da054d009f605994ef3df179~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

不同的view可以通过实现相同的接口来共享prensenter。presenter也可以通过实现接口来实现动态更换逻辑。Model是完全独立开发的，向外暴露的方法参数中含有callBack参数，可以直接调用callBack进行回调。

- MVP通过模块职责分工，抽离业务逻辑，降低代码的耦合性
- 实现模块间的单向依赖，代码思路清晰，提高可维护性
- 模块间通过接口进行通信，降低了模块间的耦合度，可以实现不同模块独立开发或动态更换

MVP的最大特点就是接口通信，接口的作用是为了实现模块间的独立开发，模块代码复用以及模块的动态更换。但是我们会发现后两个特性，在Android开发中使用的机会非常少。presenter的作用就是接受view的请求，然后再model中获取数据后调用view的方法进行展示，但是每个界面都是不同的，很少可以共用模块的情景出现。这就导致了每个Activity/Fragment都必须写一个IView接口，然后还需要再写个IPresenter接口，从而产生了非常多的接口，需要编写大量的代码来进行解耦。如果在小型的项目，这样反而会大大降低了开发效率。 其次，prensenter并没有真正解耦，他还需要调用view的接口进行UI操作，解耦没有彻底。MVP也没有解决MVC中Controller代码臃肿的问题，甚至还把部分的UI操作带到了Presenter中。

因此，由于MVP有：

> - 过度设计导致接口过多，编写大量的代码来实现模块解耦，降低了开发效率
> - 并没有彻底进行解耦，prensenter需要同时处理UI逻辑和业务逻辑，presenter臃肿



### MVVM

图解

![MVVM.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/70f01ff0a9f04a6aacd1d5b9731089ae~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

- View：和前面的MVP、MVC中的View一样，负责UI界面的显示以及与用户的交汇。
- Model：同样是负责网络数据获取或者本地数据库数据获取。
- ViewModel：负责存储view的数据映像以及业务逻辑。

MVVM的view和model和前面的两种架构模式是差不多的，重点在ViewModel。viewModel通过将数据和view进行绑定，修改数据会直接反映到view上，通过**数据驱动型思想**，彻底把MVP中的Presenter的UI操作逻辑给去掉了。而viewModel是绑定于单独的view的，也就不需要进行编写接口了。但viewModel中依旧有很多的业务逻辑，但是因为把view和数据进行绑定，这样可以让view和业务彻底的解耦了。view可以专注于UI操作，而viewModel可以专注于业务操作。因而：

> - MVVM通过数据驱动型思想，彻底把业务和UI逻辑进行解耦，各模块分工职责明确。

View只需要关注Viewmodel的数据部分，而无需知道数据是怎么来的；而ViewModel只需要关注数据逻辑，而不需要知道UI是如何实现的。View可以随意更换UI实现，但ViewModel却完全不需要改变。

- MVVM的viewModel依旧很臃肿。
- MVVM需要学习数据绑定框架，具有一定的上手难度。

![谷歌推荐android开发架构.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d85544fbcc944084ad223072e4869734~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

简单的解析：

- View对应的就是Activity和Fragment，在这里进行UI操作。
- ViewModel中包含了LiveData，这是一种可观察数据类型框架。View通过向LIveData注册观察者，当LiveData发生改变时，就会直接调用观察者的逻辑把数据更新到view上。
- ViewModel完全不需要关心UI操作，只需要专注于数据与业务操作。
- Repository代表了Model层，Repository对ViewModel进行了减压，把业务操作般到了Repository中，避免了viewModel臃肿。
- Repository对请求进行判断是要到本地数据库获取还是网络请求获取分别调用不同的模块。



DataBinding：

- 解基于数据驱动思想，决视图调用一致性问题，实现双向绑定
- 避免编写样板式代码，提高效率

LiveData：

- 通过唯一可信源获取数据，正确分发数据
- 与Lifecycle结合，拥有生命周期感知能力，配合viewModel实现作用域可控
- 实现模块的单向依赖，抛弃接口回调

ViewModel：

- 托管界面状态，解决状态管理问题
- 实现跨页面数据分享，并为数据设置作用域，做到作用域可控
- 实现单向依赖，避免内存泄露

Lifecycle：

- 以简便地方式解决生命周期管理的一致性问题
- 避免内存泄露的情况下让第三方组件随时获取生命周期状态，追踪事故所在的生命周期源，对错过时机的异步操作及时停止

Navigation

- 通过遵循导航定则实现对Fragment的管理



作者：一只修仙的猿
链接：https://juejin.cn/post/6905592834611478535
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。