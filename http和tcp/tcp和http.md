1. tcp 连接

   tcp连接的建立需要经过“三次握手”

2. http协议：

   HTTP全称是HyperText Transfer Protocal，即：超文本传输协议，HTTP连接最显著的特点是客户端发送的每次请求都需要服务器回送响应，在请求结束后，会主动释放连接。从建立连接到关闭连接的过程称为“一次连接”

3. https通信原理：

   https安全超文本传输协议，它是一个安全通信通道；https是http over ssl/tls，HTTP是应用层协议，TCP是传输层协议，在应用层和传输层之间，增加了一个安全套接层ssl/tls

   SSL (Secure Socket Layer，安全套接字层)

   TLS (Transport Layer Security，传输层安全协议)

   SSL使用40 位关键字作为RC4流加密算法

4. Https的作用

   内容加密 建立一个信息安全通道，来保证数据传输的安全；

   身份认证 确认网站的真实性

   数据完整性 防止内容被第三方冒充或者篡改

5. cookie和sessions token

   cookie可以理解成会话的跟踪机制，它可以弥补HTTP协议无状态的不足。在Session出现之前，基本上所有的网站都采用Cookie来跟踪会话  

   https://www.cnblogs.com/wxinyu/p/9154178.html