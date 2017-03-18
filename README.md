#websocket
服务器运行 
```shell
php websocket.php  
```
## demo
<http://www.yxsss.com/ui/sk.html>

# 即时通讯（instant-messaging）

## 开始之前(overview)

> 使用`web`来构建实时展示信息，类似股票信息，火车票余票，或是即时聊天信息等等
## 传统的实现方式：
> 轮询？长轮询？ ...
> 不管以上什么方式，在使用的时候都有缺点或是值得改进的地方。

### ajax轮询
>`ajax`轮询的原理非常简单，让浏览器隔个几秒就发送一次请求，询问服务器是否有新信息。

**场景再现：**

客户端：啦啦啦，有没有新信息(Request)

服务端：没有（Response）

客户端：啦啦啦，有没有新信息(Request)

服务端：没有。。（Response）

客户端：啦啦啦，有没有新信息(Request)

服务端：你好烦啊，没有啊。。（Response）

客户端：啦啦啦，有没有新消息（Request）

服务端：好啦好啦，有啦给你。（Response）

客户端：啦啦啦，有没有新消息（Request）

服务端：。。。。。没。。。。没。。。没有（Response） —- loop
###  长轮询（ long poll ）
>`long poll` 其实原理跟 ajax轮询 差不多，都是采用轮询的方式，不过采取的是阻塞模型（一直打电话，没收到就不挂电话），也就是说，客户端发起连接后，如果没消息，就一直不返回Response给客户端。直到有消息才返回，返回完之后，客户端再次建立连接，周而复始。

**场景再现：**

客户端：啦啦啦，有没有新信息，没有的话就等有了再返回给我吧（Request）

服务端：额。。 等待到有消息的时候。。。有了，来吧， 给你（Response）

客户端：啦啦啦，有没有新信息，没有的话就等有了再返回给我吧（Request） -loop
### 结论
>从上面可以看出其实这两种方式，都是在不断地建立HTTP连接，然后等待服务端处理，可以体现HTTP协议的另外一个特点，被动性
**被动性呢，其实就是，服务端不能主动联系客户端，只能由客户端发起。**

从上面很容易看出来，不管怎么样，上面这两种都是非常消耗资源的。

`ajax`轮询 需要服务器有很快的处理速度和资源。（速度）
`long poll` 需要有很高的并发，也就是说同时接待客户的能力。（场地大小）

>所以 `ajax`轮询 和 `long poll` 都有可能发生这种情况。

客户端：啦啦啦啦，有新信息么？

服务端：月线正忙，请稍后再试（503 Server Unavailable）

客户端：。。。。好吧，啦啦啦，有新信息么？

服务端：月线正忙，请稍后再试（503 Server Unavailable）

客户端：然后服务端在一旁忙的要死：服务器，我要更多的服务器！更多。。更多。。

## websocket实现方式
### websocket与http
> `websocket`是`html5`新出的东西（协议），但是`HTTP`协议并没有变化，或者说是`websocket`与`http`协议没有多大关系，`HTTP`协议是不支持持久链接的（长链接，循环链接的不算）。

首先HTTP协议有`1.1`和`1.0`之说，也就是所谓的`keep-alive`，把多个`HTTP`请求合并成为一个，但是`websocket`其实是一个新协议，和`http`协议基本没有关系，只是为了兼容现有的浏览器的握手规范而已，也就是说它是`http`协议上的一种补充，可以通过这样一张图来了解，

![@有交集，但并不相等 | center | 300 * 0 ](http://img2.tuicool.com/iE3yuqA.png!web)


另外Html5是指的一系列新的API，或者说新规范，新技术。Http协议本身只有1.0和1.1，而且跟Html本身没有直接关系。。通俗来说，你可以用HTTP协议传输非Html数据，就是这样=。=

再简单来说，层级不一样。

### Websocket是什么样的协议，具体有什么优点?

首先，`Websocket`是一个持久化的协议，相对于`HTTP`这种非持久的协议来说。简单的举个例子吧，用目前应用比较广泛的`PHP`生命周期来解释。

`HTTP`的生命周期通过 Request 来界定，也就是一个 `Request` 一个 `Response` ，那么在 `HTTP` 1.0 中，这次`HTTP`请求就结束了。

在`HTTP` 1.1中进行了改进，使得有一个`keep-alive`，也就是说，在一个`HTTP`连接中，可以发送多个`Request`，接收多个`Response`。但是请记住 `Request = Response` ， 在`HTTP`中永远是这样，也就是说一个`request`只能有一个`response`。而且这个`response`也是被动的，不能主动发起。

**首先Websocket是基于HTTP协议的，或者说借用了HTTP的协议来完成一部分握手**

**首先我们来看个典型的 Websocket 握手**[@wikepedia](https://zh.wikipedia.org/wiki/WebSocket)

    GET /chat HTTP/1.1
	Host: server.example.com
	Upgrade: websocket
	Connection: Upgrade
	Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
	Sec-WebSocket-Protocol: chat, superchat
	Sec-WebSocket-Version: 13
	Origin: http://example.com

这段类似HTTP协议的握手请求中，多了几个东西

    Upgrade: websocket
    Connection: Upgrade

这个就是websocket的核心了，告诉Apache，Nginx等服务器：注意了，我发起的是`Websocket`协议，快点帮我找对应的助理处理，不是那个老土的`http`协议

    Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
	Sec-WebSocket-Protocol: chat, superchat
	Sec-WebSocket-Version: 13
首先，`sec-WebSocket-Key`是一个`base64 encode`的值，这个是浏览器随机生成的，告诉服务器，泥煤，不要忽悠窝，我要验证你是不是真正的`websocket`的助理。

然后， `Sec_WebSocket-Protocol`是一个用户定义的字符串，用来区分同URL下，不同的服务所需要的协议。简单理解：今晚我要服务A，别搞错啦~

最后， `Sec-WebSocket-Version` 是告诉服务器所使用的 `Websocket Draft`（协议版本），在最初的时候，`Websocket`协议还在 `Draft` 阶段，各种奇奇怪怪的协议都有，而且还有很多奇奇怪怪不同的东西，什么`Firefox`和`Chrome`用的不是一个版本之类的，当初`Websocket`协议太多可是一个大难题。。不过现在还好，已经定下来啦~大家都使用的一个东西~ 脱水： 服务员，我要的是13岁的噢→_→

**然后服务器会返回下列东西，表示已经接受到请求， 成功建立Websocket啦！**

    HTTP/1.1 101 Switching Protocols
	Upgrade: websocket
	Connection: Upgrade
	Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=
	Sec-WebSocket-Protocol: chat
这里开始就是HTTP最后负责的区域了，告诉客户，我已经成功切换协议啦~

    Upgrade: websocket
	Connection: Upgrade
依然是固定的，告诉客户端即将升级的是 `Websocket` 协议，而不是`mozillasocket`，`nanisocket`或者`shitsocket`。

然后， `Sec-WebSocket-Accept` 这个则是经过服务器确认，并且加密过后的 `Sec-WebSocket-Key` 。 服务器：好啦好啦，知道啦，给你看我的ID CARD来证明行了吧。。

后面的， `Sec-WebSocket-Protocol` 则是表示最终使用的协议。

**至此，HTTP已经完成它所有工作了，接下来就是完全按照Websocket协议进行了。具体的协议就不在这阐述了。**

哦对了，忘记说了HTTP还是一个状态协议。

通俗的说就是，服务器因为每天要接待太多客户了，是个健忘鬼，你一挂电话，他就把你的东西全忘光了，把你的东西全丢掉了。你第二次还得再告诉服务器一遍。

所以在这种情况下出现了，Websocket出现了。他解决了HTTP的这几个难题。首先，被动性，当服务器完成协议升级后（HTTP->Websocket），服务端就可以主动推送信息给客户端啦。所以上面的情景可以做如下修改。


**场景再现：**

服务端：ok，确认，已升级为Websocket协议（HTTP Protocols Switched）

客户端：麻烦你有信息的时候推送给我噢。。

服务端：ok，有的时候会告诉你的。

服务端：balabalabalabala

服务端：balabalabalabala

服务端：哈哈哈哈哈啊哈哈哈哈

服务端：笑死我了哈哈哈哈哈哈哈

**就变成了这样，只需要经过一次HTTP请求，就可以做到源源不断的信息传送了。（在程序设计中，这种设计叫做回调，即：你有信息了再来通知我，而不是我傻乎乎的每次跑来问你 ）**

这样的协议解决了上面同步有延迟，而且还非常消耗资源的这种情况。那么为什么他会解决服务器上消耗资源的问题呢？

其实我们所用的程序是要经过两层代理的，即`HTTP`协议在`Nginx`等服务器的解析下，然后再传送给相应的`Handler`（`PHP`等）来处理。简单地说，我们有一个非常快速的 接线员（`Nginx`） ，他负责把问题转交给相应的 客服（`Handler`） 。

本身接线员基本上速度是足够的，但是每次都卡在客服（`Handler`）了，老有客服处理速度太慢。，导致客服不够。`Websocket`就解决了这样一个难题，建立后，可以直接跟接线员建立持久连接，有信息的时候客服想办法通知接线员，然后接线员在统一转交给客户。

这样就可以解决客服处理速度过慢的问题了。

同时，在传统的方式上，要不断的建立，关闭`HTTP`协议，由于`HTTP`是非状态性的，每次都要重新传输 `identity info` `（鉴别信息）`，来告诉服务端你是谁。

虽然接线员很快速，但是每次都要听这么一堆，效率也会有所下降的，同时还得不断把这些信息转交给客服，不但浪费客服的处理时间，而且还会在网路传输中消耗过多的流量/时间。

但是`Websocket`只需要一次`HTTP`握手，所以说整个通讯过程是建立在一次连接/状态中，也就避免了`HTTP`的非状态性，服务端会一直知道你的信息，直到你关闭请求，这样就解决了接线员要反复解析`HTTP`协议，还要查看`identity info`的信息。

同时由客户主动询问，转换为服务器（推送）有信息的时候就发送（当然客户端还是等主动发送信息过来的。。），没有信息的时候就交给接线员（`Nginx`），不需要占用本身速度就慢的客服（`Handler`）了


----------
至于怎么在不支持Websocket的客户端上使用Websocket。。答案是： 不能

但是可以通过上面说的 long poll 和 ajax 轮询 来 模拟出类似的效果

