## HTTP介绍

HyperText Transfer Protocol超文本协议，属于应用层协议，用于定义Web浏览器与Web服务器之间交换数据的过程以及数据本身的格式，底层是靠TCP进行可靠的信息传输

![在这里插入图片描述](https://img-blog.csdnimg.cn/7d06296ca33d4a4e9f5170f8d23d1045.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBARG9udGxh,size_20,color_FFFFFF,t_70,g_se,x_16)

## HTTP协议发展史

### HTTP/0.9

HTTP 于 1990 年问世。那时的 HTTP 并没有作为正式的标准被建立。
现在的 HTTP 其实含有 HTTP1.0 之前版本的意思，因此被称为 HTTP/0.9。

该版本极其简单：

- 只有一个命令 GET；
- 没有 Header 等描述数据的信息；
- 服务器在发送数据完毕后，就关闭 TCP 连接。

### HTTP/1.0

HTTP/1.0 版本与 HTTP/0.9 相比，主要有：

- 增加了很多命令，如 POST，PUT，HEAD；
- 增加了 Status Code 和 Header 相关内容；
- 增加了多字符集支持、多部分发送(multi-part type)、权限(authorization)、缓存(cache)、内容编码(content encoding)等。

### HTTP/1.1

HTTP/1.1 是目前主流的 HTTP 版本，有比较完善的功能。

- 增加了 PATCH、OPTIONS、DELETE 命令。
- 持久连接，即 TCP 连接默认不关闭，可以被多个请求复用，提高了请求性能。
- 管道机制(pipeline)，即在同一个 TCP 连接里面，客户端可以同时发送多个请求。例如，浏览器同时发出 A 请求和 B 请求，但是服务器还是按照顺序，先回应 A 请求，完成后再回应 B 请求。
- 增加 Host 字段，可以将请求发往同一个服务器的不同网站，为虚拟主机打下了基础。这个字段增加的好处就是在同一个物理服务器中可以同时部署多个 Web 服务，这样可以提高物理服务器的使用效率。

### HTTP2

HTTP2 目前还没有普及，但肯定是未来的主流。HTTP2 主要解决了传输性能的问题。

- 所有数据以二进制传输。在 HTTP/1.1 版本中大部分数据是以文本形式传输，在 HTTP2 版本中，所有数据以二进制传输，统称为“`帧`”。
- 多工。因为有了以二进制传输的好处，同一个连接里面发送多个请求不再需要按照顺序来进行返回处理，而是同时返回。在返回第一个请求的同时也可以返回第二个请求，这样它就是一个并行的效率，可以更大限度地让整个 Web 应用的传输效率有一个质的提升。（[为什么以二进制传输就能并行？](https://www.zhihu.com/question/528616001)）
- 头信息压缩。在 HTTP/1.1 中，每次发送请求和返回请求，它的 HTTP 头信息总是要完整发送和返回，而这部分头信息内容是以字符串形式保存，所以它占用的带宽量是很大的。而 HTTP2 中，对头信息进行了压缩，减少了对带宽的占用。
- 服务器推送。HTTP/2 允许服务器未经请求，主动向客户端发送资源。常见场景是客户端请求一个网页，这个网页里面包含很多静态资源。正常情况下，客户端必须收到网页后，解析 HTML 源码，发现有静态资源，再发出静态资源请求。其实，服务器可以预期到客户端请求网页后，很可能会再请求静态资源，所以就主动把这些静态资源随着网页一起发给客户端了。

## HTTP 三次握手

![在这里插入图片描述](https://img-blog.csdnimg.cn/40c35b9e39334f9c9aab765f5d60bdf2.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBARG9udGxh,size_20,color_FFFFFF,t_70,g_se,x_16)

为了准确无误地将数据送达目标处，TCP 协议采用了三次握手 (three-way handshaking) 策略。用 TCP 协议把数据包送出去后，TCP 不会对传送后的情况置之不理，它一定会向对方确认是否成功送达。握手过程中使用了 TCP 的标志——SYN(synchronize) 和 ACK(acknowledgement)。

发送端首先发送一个带 SYN 标志的数据包给对方。接收端收到后，回传一个带有 SYN/ACK 标志的数据包以示传达确认信息。最后，发送端再回传一个带 ACK 标志的数据包，代表“握手”结束

## URI、URL 和 URN

### URI（统一资源标识符 Uniform Resource Identifier）

URI（统一资源标识符）是 Uniform Resource Identifier 的缩写。它主要用于定位某一类特定的资源而设计，用来唯一标识互联网上的信息资源。它包括 URL 和 URN。

### URL（统一资源定位符 Uniform Resource Locator）

URL（统一资源定位符）是 Uniform Resource Locator 的缩写。它用来找到资源所在的位置，并且去访问和得到资源。

URL 格式：

![在这里插入图片描述](https://img-blog.csdnimg.cn/be069cfbce0a4ddf901a649a90478bac.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBARG9udGxh,size_20,color_FFFFFF,t_70,g_se,x_16)

### 协议

获取资源时要指定协议类型，比如http、https、ftp等协议

### 登录信息（认证）

指定用户和密码作为从服务端获取资源时必要的登录信息，这种方式在现在的 Web 应用开发中不太会使用到。如果用户每次访问资源，都需要在 URL 中填写用户名和密码，是很不方便的，也是很不安全的做法

### 服务器地址

指定资源所在服务器在互联网中的位置，ip地址，也可以是DNS可解析的地址

### 服务器端口号

指定服务器连接的网络端口号。每一台服务器都有很多的端口，在这台服务器上可以运行很多软件的 Web 服务，这些 Web 服务可以监听不同的端口。如果我们要找这台服务器上某一个 Web 服务里面的资源，就要指定要找的是哪个 Web 服务，也就是说端口是用来定位服务器上的某个 Web 服务的。

### 资源路径

#### 查询字符串（这个不太懂，关于查询字符串与json？）

针对已指定的文件路径内的资源，可以使用查询字符串传入任意参数。

#### 片段标识符（？）

使用片段标识符通常可标记出已获取资源中的子资源(文档内的某个位置)。

### URN（永久统一资源定位符 Uniform Resource Name）（目前用得不多，主要用URL）

URN（永久统一资源定位符）是 Uniform Resource Name 的缩写。作为 HTTP 服务，如果某一类资源改变了位置，导致它的 URL 链接无法访问到资源，那么 URN 就解决了这个问题。也就是说，即便是资源改变了位置，通过 URN 还是可以访问到。