## 15. Cookie和session和token的区别是什么？
### cookie
cookie 是一个非常具体的东西，指的就是浏览器里面能永久存储的一种数据，仅仅是浏览器实现的一种数据存储功能。  
**cookie由服务器生成，发送给浏览器，浏览器把cookie以kv形式保存到某个目录下的文本文件内，下一次请求同一网站时会把该cookie发送给服务器**。由于**cookie是存在客户端上**的，
所以浏览器加入了一些限制确保cookie不会被恶意使用，同时不会占据太多磁盘空间，所以每个域的cookie数量是有限的。

### session
当用户第一次通过浏览器使用用户名和密码访问服务器时，服务器会验证用户数据，验证成功后在服务器端写入session数据，向客户端浏览器返回sessionid，**浏览器将sessionid保存在cookie中，
当用户再次访问服务器时，会携带sessionid，服务器会拿着sessionid从数据库获取session数据，然后进行用户信息查询，查询到，就会将查询到的用户信息返回，从而实现状态保持**。
这种用户信息存储方式相对cookie来说更安全。

#### 弊端：
1. **服务器压力增大**  
> 通常session是存储在内存中的，每个用户通过认证之后都会将session数据保存在服务器的内存中，而当用户量增大时，服务器的压力增大。
2. **CSRF跨站伪造请求攻击**  
> session是基于cookie进行用户识别的, cookie如果被截获，用户就会很容易受到跨站请求伪造的攻击。
3. **扩展性不强**  
> 如果将来搭建了多个服务器，虽然每个服务器都执行的是同样的业务逻辑，但是session数据是保存在内存中的（不是共享的），用户第一次访问的是服务器1，当用户再次请求时可能访问的是另外一台服务器2，
> 服务器2获取不到session信息，就判定用户没有登陆过。

### token
**Token是服务端生成的一串字符串**，以作客户端进行请求的一个令牌，当第一次登录后，服务器生成一个Token便将此Token返回给客户端，以后客户端只需带上这个Token前来请求数据即可，**无需再次带上用户名和密码**。

最简单的token组成:uid(用户唯一的身份标识)、time(当前时间的时间戳)、sign(签名，由token的前几位+盐以哈希算法压缩成一定长的十六进制字符串，可以防止恶意第三方拼接token请求服务器)。

使用Token的目的：T**oken的目的是为了减轻服务器的压力，节省服务器内存，提高服务器的扩展能力；数字签名仿伪造攻击**。

**使用session时，鉴权状态在服务器中处理，而token则在客户端管理; 有了token，访问特定页面，用户就不需要每次都输入账号和密码登录**


#### Token的实现原理：JSON Web Token（JWT）方式
1） 将荷载payload，以及Header信息进行Base64加密，形成密文payload密文，header密文。  
2） 将形成的密文用句号链接起来，用服务端秘钥进行HS256加密，生成签名.  
3） 将前面的两个密文后面用句号链接签名形成最终的token返回给服务端  

![tokengenerate](https://img-blog.csdnimg.cn/20191026104926702.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0F3YXlGdXR1cmU=,size_16,color_FFFFFF,t_70)

说明：  
（1）用户请求时携带此token（分为三部分，header密文，payload密文，签名）到服务端，服务端解析第一部分（header密文），用Base64解密，可以知道用了什么算法进行签名，此处解析发现是HS256。  
（2）服务端使用原来的秘钥与密文(header密文+"."+payload密文)同样进行HS256运算，然后用生成的签名与token携带的签名进行对比，若一致说明token合法，不一致说明原文被修改。  
（3）判断是否过期，客户端通过用Base64解密第二部分（payload密文），可以知道荷载中授权时间，以及有效期。通过这个与当前时间对比发现token是否过期。  

https://blog.csdn.net/AwayFuture/article/details/102753627

使用Token验证的优势：  
* 无状态、可扩展
> 在客户端存储的Tokens是无状态的，并且能够被扩展。基于这种无状态和不存储Session信息，负载负载均衡器能够将用户信息从一个服务传到其他服务器上。

* 安全性
> 请求中发送token而不再是发送cookie能够防止CSRF(跨站请求伪造)。即使在客户端使用cookie存储token，cookie也仅仅是一个存储机制而不是用于认证。不将信息存储在Session中，更加安全。  
token是有时效的，一段时间之后用户需要重新验证。我们也不一定需要等到token自动失效，token有撤回的操作，通过token revocataion可以使一个特定的token或是一组有相同认证的token无效。

https://blog.csdn.net/weixin_43549578/article/details/90576846


    个人理解：token主要靠签名来检查是否登录过，base64加解密很好得到，但是因为没有HS256加密用的秘钥，所以没办法伪造，因此没办法得到正确的签名。
    反过来说，如果签名验证通过，说明曾经登陆过，签名是服务端返回的。
    
    使用token也是有安全问题的，比方说token被窃取了，劫持者使用窃取的token和服务端进行通信。在网络层面上token明文传输的话会非常的危险，
    所以建议一定要使用HTTPS，并且把token放在post body里。可以提供一个让用户可以主动expire一个过去的token类似的机制，在被盗的时候能远程止损。

### cookie与session的区别
1. Session保存在服务器端，客户端不知道其中的信息，Cookie保存在客户端，服务器能够知道其中的信息；
2. cookie不是很安全，别人可以分析存放在本地的COOKIE并进行COOKIE欺骗；考虑到安全应当使用session。
3. session会在一定时间内保存在服务器上。当访问增多，会比较占用你服务器的性能。考虑到减轻服务器性能方面，应当使用COOKIE。
4. **session是基于cookie实现的（透传sessionId）**。当客户端存储的cookie失效后，服务端的session不会立即销毁，会有一个延时，服务端会定期清理无效session，不会造成无效数据占用存储空间的问题。
5. 单个cookie保存的数据不能超过4K，很多浏览器都限制一个站点最多保存20个cookie。所以建议**将登陆信息等重要信息存放为SESSION；其他信息如果需要保留，可以放在COOKIE中**

cookie和session的共同之处在于：**cookie和session都是用来跟踪浏览器用户身份的会话方式**，因为HTTP协议是无状态的，这种无状态意味着程序需要验证每一次请求，从而辨别客户端的身份。

https://www.cnblogs.com/ella-li/p/12008069.html

### token与session的区别：
1. token和session其实都是为了身份验证，session一般翻译为会话，而token更多的时候是翻译为令牌；
session服务器会保存一份，可能保存到缓存，文件，数据库；同样，session和token都是有过期时间一说，都需要去管理过期时间；
其实token与session的问题是一种时间与空间的博弈问题，**session是空间换时间，而token是时间换空间**。两者的选择要看具体情况而定。

2. **基于 Token 的身份验证方法，在服务端不需要存储用户的登录记录。而session需要在服务端保存用户数据**：
虽然确实都是“客户端记录，每次访问携带”，但 token 很容易设计为**自包含**的，也就是说，后端不需要记录什么东西，每次一个无状态请求，每次解密验证，每次当场得出合法 /非法的结论。这一切判断依据，除了固化在 CS 两端的一些逻辑之外，整个信息是自包含的。这才是真正的无状态。 
而 sessionid ，一般都是一段随机字符串，需要到后端去检索 id 的有效性。万一服务器重启导致内存里的 session 没了呢？万一 redis 服务器挂了呢？

3. 作为身份认证 token安全性比session好，因为**每个请求都有签名还能防止监听以及重放攻击**，而session就必须靠链路层来保障通讯安全了。Token是通过加密算法（如：HMAC-SHA256算法）来实现session对象验证的，这样使得攻击者无法伪造token来达到攻击或者其他对服务器不利的行为；

https://blog.csdn.net/mydistance/article/details/84545768  
https://www.cnblogs.com/jinbuqi/p/10347800.html
