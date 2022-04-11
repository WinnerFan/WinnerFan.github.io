---
layout: post
title: User Register
tags: Java
---
## User Register
### Cookie
报文头中通过response: Set-Cookie，request: Cookie添加 
```
// 添加Cookie
Cookie cookie = new Cookie("xxx", yyy);
cookie.setPath("/testCookie"); //仅此URL带Cookie，默认/
cookie.setDomain("localhost"); //仅此域名带Cookie
cookie.setMaxAge(10); //过期时间秒数，0表示删除Cookie，负数表示临时Cookie，关闭浏览器即失效，默认-1
cookie.setHttpOnly(true); //Http协议传输
cookie.setSecure(true); //Https协议传输，禁止Http
(HttpServletResponse)response.addCookie(cookie);

获取tokenId，tokenId存在redis，key：tokenId；value：useId；expire：60 * 60

// 获取Cookie
Cookie[] cookies = (HttpServletRequest)request.getCookies();
for (Cookie c : cookies) {
    if("tokenId".equals(c.getName()))
        tokenId = c.getValue();
}


// 删除cookie
Cookie cookie = new Cookie("xxx", "");
cookie.setPath("/");
cookie.setDomain("localhost");
cookie.setMaxAge(0);
cookie.setHttpOnly(true);
cookie.setSecure(true);
(HttpServletResponse)response.addCookie(cookie);
```

- Cookie不能跨域。Google也只能操作Google的Cookie，而不能操作百度的Cookie
- images.google.com与网站www.google.com不同Cookie。
- 二级域名都可以使用同一Cookie，可以设置cookie.setDomain(".google.com")


### Session
Cookie是存储在**客户端方**，Session是存储在**服务端方**，客户端只存储SessionId。

如果将账户的一些信息都存入Cookie中的话，一旦信息被拦截，那么我们所有的账户信息都会丢失掉。所以就出现了Session，在一次会话中将重要信息保存在Session中，浏览器只记录SessionId，一个SessionId对应一次会话请求。
```
(HttpSession)session.setArribute("testSession","yyy")
```
报文头中response: Set-Cookie: JSESSIONID=，request: Cookie: JSESSIONID=

- 客户端过期：和Cookie过期一致，如果未设置，默认是关闭浏览器
- 服务端过期：默认30分钟
- 服务端默认ConcurrentHashMap保存，为了多机共享可放入Redis


### Token
Session是存放在服务器端的，采用空间换时间的策略来进行身份识别；Token是放在客户端存储的，采用了时间换空间策略，无状态所以在分布式环境中应用广泛。**Session储存问题、安全问题、跨域问题**出现了Token。

第一次会根据例如UserId和密钥生成Token，第二次携带Token再次根据UserId和密钥生成Token，校验Token是否相同。

- 无状态、可扩展、简洁：可以通过URL、POST参数或者是在HTTP头参数发送
- 安全性：CSRF（Cross-site request forgery）跨站请求伪造，只能借用cookie，且不可预知，无法伪造cookie。可以放cookie，但是并不使用cookie验证，而仅仅是用作储存；后台http请求体中的token参数值和cookie中的csrftoken值进行比对
- 多站点：CORS跨域支持
- 支持移动平台

```
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.9.1</version>
</dependency>

```
生成tokenID，解析message，判断cookie过期
```
private final static long TTL = 30 * 60 * 1000;
private final static String USER_ID = "useId";
private final static Key key = MacProvider.generateKey();

public CustomResponse<String> test(@RequestParam(value = "useId") String useId, HttpServletResponse response) {
    long now = System.currentTimeMillis();
    Date exp = new Date(now + TTL);
    String tokenId = Jwts.builder().compressWith(CompressCodecs.DEFLATE).signWith(SignatureAlgorithm.HS512, key).claim(USER_ID, message).setExpiration(exp).compact();
    
    //Jws<Claims> claim = Jwts.parser().setSigningKey(key).parseClaimsJws(tokenId);
    //String message = (String) claim.getBody().get(USER_ID);
    
    //Jws<Claims> claim = Jwts.parser().setSigningKey(key).parseClaimsJws(tokenId);
    //Date date = claim.getBody().getExpiration();
    //if (date.getTime() >= System.currentTimeMillis()) return false;
    //else return true;

    CustomResponse customResponse = new CustomResponse();
    customResponse.setUserId(useId);
    return customResponse;
}

```

### 请求头
HTTP/1.0长连接Connection: keep-alive，HTTP/1.1规定所有都是，否则显式加上Connection: close。
短连接通过请求关闭界定边界，长连接通过Content-Length，长度小于实体会被截断，长度长于实体pending。
当Transfer-Encoding不为identity，Content-Length将被忽略。
长度不好获取，buffer等内容生成好后再计算，但需要内存开销且客户端慢，需要Transfer-Encoding: chunked分块编码。每个分块包含一个16进制内容长度值加`\r\n`和换行后的内容加`\r\n`，最后一个分块长度值为0，内容为空。
Content-Encoding先对内容编码（压缩）后再进行传输编码（分块）。

