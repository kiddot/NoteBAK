# Http怎么处理长连接

http://huachao1001.github.io/article.html?3C6fV99L

`http`长连接：

> `http1.1`默认保持长连接，数据传输完成后，`TCP`连接不断开，等等同域名下继续使用这个通道传输数据。

`HTTP`首部`Connection`：

> `keep-alive`是`HTTP1.0`的扩展，如果`HTTP1.1`版本的`HTTP`请求不希望使用长连接，则要在`HTTP1.0`请求首部加`Connection:close`.

另外，`HTTP`头部有`Keep-Alive`这个值并不代表一定会使用长连接，客户端和服务端都可以无视这个值，也就是不按标准来。正常情况下浏览器和`Web`服务器都实现这个标准。

长连接不可能无限期保持，会有一个超时时间，服务器有时会告诉客户端超时时间，比如，服务器会在在应答包的`HTTP`头加上:

> Connection:Keep-Alive
>
> Keep-Alive:time=20

还可能有`max=XXX`,表示这个长连接最多接收`XXX`次请求就断开。对于客户端，如果服务器没有告诉超时时间也没关系，服务端可能主动发起四次挥手断开`TCP`连接，客户端能够知道该`TCP`无效；另外，`TCP`还有心跳包来检测当前连接是否活着，方法很多。

使用长连接后，客户端、服务端如何知道本次传输结束？

> - 判断传输数据是否达到了`Content-Length`指示的大小
> - 动态生成的文件没有`Content-Length`,它是分块传输（`chunked`），这时候要根据`chunked`编码来判断，`chunked`编码的数据在最后有一个空的`chunked`块，表明本次传输数据结束。