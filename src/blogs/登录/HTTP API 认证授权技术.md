# Http Api 认证授权技术
> ref: [HTTP API 认证授权术](https://coolshell.cn/articles/19395.html)

> 我们知道，HTTP 是无状态的，所以，当我们需要获得用户是否在登录状态时，我们需要检查用户的登录状态，一般来说，用户登录成功后，服务器会
发一个登录凭证（又被称作 token）。就像你去某个公司访问，前台需要给你发送一个访客卡，其他就知道了你是授权用户，不用再去检查你。
在计算机的世界里，这个登录凭证会放在两个地方，一个是用户端，以Cookie的方式，另一个地方是放在服务端，以Session 的方式。

但是，除了用户访问，还有用户委托第三方的应用，还有企业和企业间的调用，以下是把业内常用的一些 API 认证技术相对系统的总结归纳以下，这样
可以让大家更为全面的了解这些技术。

本文会覆盖以下技术：
- HTTP Basic
- Digest Access
- App Secret Key + HMAC
- JWT - Json web tokens
- OAuth 1.0 - 3 legged & 2 legged
- OAuth 2.0 - Authentication Code & Client Credential

## HTTP Basic
Http basic 是一个非常传统的 API 认证技术，也是一个简单的技术。这个技术也就是使用 username 和 password 来进行登录。整个过程被
定义在了 [RFC 2617](http://tools.ietf.org/html/rfc2617) 中，也被描述在了 WikiPedia：Basic Access Authentication 词条中，
同时也可以参看 MDN HTTP Authentication

其技术原理如下：
    - 1.把 username 和 password 做成 username:password 的样子（用冒号分隔）
    - 2.进行 Base64 编码。Base64("username:password") 得到一个字符串（如：把zhangwentao:tiger 编码之后得到 emhhbmd3ZW50YW86dGlnZXI=）
    - 3.把 emhhbmd3ZW50YW86dGlnZXI= 放到 HTTP 头中 Authorization 字段中，形成 Authorization: Basic emhhbmd3ZW50YW86dGlnZXI=,然后发送到服务端
    - 4.服务端如果没有在头中看到认证字段，则返回401错，以及一个WWW-Authenticate：Basic Realm='' 之类的要求客户端进行认证。如果验证
    不通过，则返回一个401错误。如果验证通过，则返回 200
    
我们可以看到，使用 Base64 的目的无非是为了把一些特殊的字符弄掉，这样就可以在HTTP 协议里传输了。而这种方式的最大问题就是把用户名和密码放在
网络上传输，所以，一般要配合 TLS/SSL 的安全加密方式来使用。

但是我们要知道，这种直接把用户名和密码放在公网上进行传输的方式有点不太好，因为 Base64 不是加密协议，而是编码协议，所以就算是有HTTPS 作为
安全保护，给人的感觉还是不放心。

## Digest Access
中文名称是"HTTP摘要认证"，最初被定义在 [RFC 2069]() 文档中（后来被 RFC 2617 引入了一些安全增强选项；质量保证 qop，随机数计数器由客户端增加、
以及客户端生成的随机数）。

其基本思路是，请求方吧用户名口令和域做一个 MD5-MD5（username：realm：password），然后传给服务器，这样就不会在网上传用户名和口令了，
但是因为用户名和口令基本不会变，所以这个 MD5 字符串也是比较固定的，因此，这个认证过程在其中加入了两个事，一个是 nonce 另一个是 qop
- 首先，调用方发起一个普通的HTTP 请求。比如：GET /http/learn/ HTTP/1.1
- 服务端自然不能认证通过，服务端返回 401 错误，并且在 HTTP 头里的 WWW-Authenticate 中包含如下信息
```
WWW-Authenticate: Digest realm = "test@host.com",
qop="auth,auth-init",
nonce="afdsfasdfjsdjfoadsjfajsdf",
opaque="fhadoshfo9hadsfhads323r4"
``` 
- 其中的你once 是服务器端生成的随机数，然后，客户端做 HASH1=MD5(MD5(username:realm:password):nonce:cnonce),其中
cnonce 是客户端生成的随机数，这样就可以使得整个 MD5 的结果是不一样的。
- 如果 qop 中包含了 auth-init ，那么还得做 HASH2=MD5(method:digestURI:MD5(entityBody)),其中 entityBody 就是
HTTP 请求的整个数据体
- 然后得到 response = MD5(HASH1:nonce:nonceCount:cnonce:qop:HASH2),如果没有 qop 则 response = MD5(HA1:nonce:HA2)
- 最后，我们的客户端对服务端发起请求--注意HTTP头的 Authorization：Digest
```
GET /dir/index.html HTTP/1.0
Host: localhost
Authorization: Digest username="Mufasa",
                     realm="testrealm@host.com",
                     nonce="dcd98b7102dd2f0e8b11d0f600bfb0c093",
                     uri="%2Fcoolshell%2Fadmin",
                     qop=auth,
                     nc=00000001,
                     cnonce="0a4f113b",
                     response="6629fae49393a05397450978507c4ef1",
                     opaque="5ccc069c403ebaf9f0171e9517f40e41"
```
摘要认证这个方式会比之前的方式要好一些，因为没有在网上传递用户的密码，而只是把密码的MD5传送过去，相对比较安全，而且，并不需要 TLS/SSL
的安全链接。但是**别看这个算法复杂，最后你会发现，整个过程其实关键是用户的password，这个password如果不够复杂，其实是可以暴力破解的，
而且整个过程非常容易受到中间人攻击**--比如一个中间人告诉客户端需要的是 Basic 的认证方式 或是 老旧签名认证方式。

## App Secret Key + HMAC
先说 HMAC 技术，这个东西来自于 [MAC- Message Authentication Code](https://en.wikipedia.org/wiki/Message_authentication_code)，是
一种用于给消息签名的技术，也就是说，我们怕消息再传递的过程中被人修改，所以，我们需要对消息进行一个 MAC 算法，得到一个摘要字符串，然后，
接收方得到消息后，进行同样的计算，然后比较这个 MAC 字符串，如果一致，则表明没有被修改过（整个过程参看下图）。
而 HMac - [Hash-based Authentication Code](https://en.wikipedia.org/wiki/HMAC) 指的是利用 HASH 技术完成这一工作，
比如 SHA-256 算法。

![图片来源 wiki 词条](http://ww3.sinaimg.cn/large/006tNc79ly1g4zem9svvcj30ja0bj3yl.jpg)

我们在说说 APP ID，这个东西跟验证没有关系，只是用来区分，是谁来调用API的，就像我们每一个人的身份证一样，只是用来标注不同的人，不是用来
做身份验证的。与前面的不同之处是，这里我们需要利用APPId 来映射一个用来加密的秘钥，这样一来，我们就可以在服务端进行相关的管理，我们可以生成
若干个秘钥对（APPID，APPsecret），并可以有更细粒度的操作权限管理。

把AppId 和 HMAC 用于 API 认证，目前来说玩得最好最专业的应该是 AWS 了，我们可以通过 [S3 API 请求签名文档](https://docs.aws.amazon.com/zh_cn/general/latest/gr/sigv4-create-canonical-request.html)
看AWS是怎么玩的。整个过程非常复杂，基本上来说，分成如下几个步骤：
- 1.把HTTP 的请求（方法、URI、查询字符串、头、签名头，body）打个包叫 CanonicalRequest，做个SHA-256签名，然后再做一个Base16编码
- 2.把上面的签名和签名算法 AWS4-HMAC-SHA256、时间戳、Scop，再打一个包，叫做 StringToSign
- 3.准备签名，用 AWSSecretAccessKey 来对日期签一个 DateKey，在用 DateKey 对要操作的Region 签一个 DateRegionKey，再对相关
服务签一个 DateRegionServiceKey，最后得到 SigningKey
- 4.用第三部的 SigningKey 来对 第二部的 StringToSign 签名。
![AWS S3 签名文档](http://ww4.sinaimg.cn/large/006tNc79ly1g4zf0l0dm6j30ju0h93zq.jpg)

最后，发出 HTTP Request 时，在HTTP 头中的 Authorization 字段中放入如下的信息：
```
Authorization: AWS4-HMAC-SHA256 
               Credential=AKIDEXAMPLE/20150830/us-east-1/iam/aws4_request, 
               SignedHeaders=content-type;host;x-amz-date, 
               Signature=5d672d79c15b13162d9279b0855cfba6789a8edb4c82c400e06b5924a6f2b5d7
```
其中的 AKIDEXAMPLE 是 AWS AccessKeyID，也就是所谓的 APPID，服务器端会根据这个 APPID 来查询相关的 SecretAccessKey，然后
在验证签名。如果，你对这个过程有点没看懂的话，你可以读一读这篇文章——[《Amazon S3 Rest API with curl》](https://czak.pl/2015/09/15/s3-rest-api-with-curl.html)
这篇文章里有好些代码，代码应该是最有细节也是最准确的了。

这种认证方式的好处在于，APPID和APPSecretKey，是由服务端系统开出的，所以是可以被管理的，AWS 的IAM 就是相关的管理，其管理了用户、
权限和其对应的 APPID和AppSecretKey。但是不好的地方在于，这个东西没有标准，所以各家是实现恨不一致。
比如： Acquia 的 HMAC，微信的签名算法 （这里，我们需要说明一下，微信的API没有遵循HTTP协议的标准，把认证信息放在HTTP 头的 Authorization 里，
而是放在body里）

## JWT - JSON Web Tokens
JWT 是一个比较标准的认证解决方案，这个技术在java圈里应该是用用非常普遍的。JWT 签名也是一种 MAC 的方法。JWT的签名流程一般是下面这个样子的：
- 1.用户使用用户名和口令到服务器上请求认证。
- 2.认证服务器验证用户名和口令之后，在服务端生成JWT Token，这个token的生成过程如下：
    - 认证服务器还回生成一个 SecretKey秘钥
    - 对 JWT Header 和 JWT Payload 分别求Base64.在payLoad可能包括了用户的抽象ID 和过期时间。
    - 用秘钥对JWT 签名 HMAC-SHA256(SecretKey,Base64UrlEncode(Jwt-header)+'.'+Base64UrlEnCode(JWT-Payload))
- 3.然后把Base64(header).base64(payload).signature 作为 JWT Token 返回客户端。
- 4.客户端使用 JWT Token 向应用服务器发送相关的请求，这个 JWT Token 就像一个临时用户认证一样。

当应用服务器收到请求后：
- 1.应用服务会检查 JWT Token，确认签名是正确的。
- 2.然而，因为只有认证服务器有这个用户的 SecretKey，所以应用服务器把 JWT Token 传给认证服务器。
- 3.认证服务器通过 JWT Payload 解出用户的抽象id，然后通过抽象id查到登录时生成的 SecretKey，然后再来检查一下签名。
- 4.认证服务器检查通过之后，应用服务就可以认为这是合法的请求了。

我们可以看出，上面这个过程是在认证服务器上为用户动态生成 SecretKey，应用服务器在验签的时候，需要到认证服务器上去认证，这个过程
增加了一些网络调用，所以，JWT除了支持 HMAC-SHA256 算法外，还支持 RSA 非对称加密算法。

使用 RSA 非对称算法，在认证服务器这边放一个私钥，在应用服务器那边放一个公钥，认证服务器使用私钥加密，应用服务器使用公钥解密，这样一来，
就不需要应用服务器向认证服务器请求了。

但是 RSA 是一个很慢的算法，所以一种比较好的玩法是，如果我们把 Header 和 Payload 简单的做 SHA256，这回很快，然后，我们使用
RSA 算法就比较快了，而我们也做到了使用 RSA 签名的目的。

最后，我们只需要使用一个机制在认证服务器和应用服务器之间定期的换一下公钥私钥对就好了。

这里强烈推荐全文阅读 Angular 大学的 [JSW:The Complete Guide to Json Web Tokens](https://blog.angular-university.io/angular-jwt/)

## OAuth 1.0
OAuth 也是一个 API 认证的协议，这个协议最初在 2006年有 Twitter 的工程师在开发 OpenId 实现的时候，和社交书签网站Ma.gnolia时发现，
没有一种好的委托授权协议，后来成立了一个 OAuth 小组，知道这个消息之后，Google员工也加入进来，并完善了这个协议，在2007年底发布草案，
过一年后，在2008年将OAuth放进了IETF作进一步的标准化工作，最后在2010年4月，正式发布OAuth 1.0，即：RFC 5849 
（这个RFC比起TCP的那些来说读起来还是很轻松的），不过，如果你想了解其前身的草案，可以读一下 OAuth Core 1.0 Revision A ，我在下面做个大概的描述。

根据 RFC 5849，可以看到 OAuth 的出现，是为了用户想使用一个第三方的网络打印服务来打印他在某网站上的照片，但是用户不想把自己的用户名
和口令交给那个第三方网站，但是又想第三方网络打印服务来方位自己的照片，为了解决这个授权的问题，OAuth 这个协议就出来了。

- 这个协议有三个角色：
    - User （照片所有者-用户）
    - Consumer （第三方打印服务）
    - Service Provider （照片存储服务）
- 这个协议有三个阶段：
    - Consumer获取 Request Token
    - Service Provider 认证用户并授权 Consumer
    - Consumer 获取 AccessToken 调用 API 访问用户的照片

整个授权过程是这样的：
- 1.Consumer 需要线上 ServiceProvider 获取开发的Consumer Key 和 ConsumerSecret
- 2.当User访问Consumer时，Consumer向ServiceProvider发起请求，请求获取 RequestToken
- 3.ServiceProvider 验明 Consumer是注册过的第三方服务商后，返回Request Token（oauth_token）和 RequestTokenSecret(oauth_token_secret)
- 4.Consumer收到RequestToken后，使用Http Get 请求把User切换到 ServiceProvider的认证页面上（带上RequestToken），让用户
输入用户名和口令
- 5.Service Provider 认证User成功之后，跳回Consumer，并返回 RequestToken（oauth_token）和 VerificationCode（oauth_verifier)
- 6.接下来就是签名请求，用RequestToken和VerificationCode 换取 AccessToken（oauth_token)和AccessTokenSecret（oauth_token_secret)
- 7.最后使用 AccessToken 访问用户授权访问的资源。

下面附上一个Yahoo的流程图可以看到整个过程的相关细节：
[oauth 图1](https://coolshell.cn/wp-content/uploads/2019/05/oauth_graph.gif)

因为上面这个流程有三方：User/Consumer/ServiceProvider，所以又叫 3-legged flow，三脚流程。OAuth也有不需要用户参与的，
只有Consumer和ServiceProvider的，也就是2-legged flow两脚流程，其中省去了用户认证的事。整个过程如下：
- 1.Consumer 从 ServiceProvider 上获取开发的 ConsumerKey 和 Consumer Secret
- 2.Consumer 向 Service 发起请求，获取 RequestToken（需要对HTTP 请求签名）
- 3.Service Provider 验明Consumer是注册过的第三方服务之后，返回RequestToken和RequestTokenSecret
- 4.Consumer收到 RequestToken后，直接换取 AccessToken 和 AccessTokenSecret
- 5.最后使用 AccessToken 访问用户授权访问的资源。

最后，再来说一说 OAuth 中的签名。
- 我们可以看到有两个秘钥，一个是Consumer注册ServiceProvider时，由Provider颁发的ConsumerSecret，另一个是TokenSecret
- 签名秘钥就是由这两具秘钥拼接而成的，其中&做连接符。假设ConsumerSecret为 kkkkk，而 TokenSecret 是 oooo,签名秘钥为 kkkkk&oooo
- 在请求Request/AccessToken的时候，需要对整个HTTP请求进行签名（使用 HMAC-SHA1和 HMAC-RSA签名算法），请求头中需要包含一些OAuth
需要的字段，如：
    - ConsumerKey：也就是所谓的 APPID
    - Token：RequestToken 或 AccessToken
    - SignatureMethod：签名算法，如 HMAC-SHA1
    - Timestamp：过期时间
    - Nonce：随机字符串
    - CallBack：回调url

下面是整个签名的示意图：
![图片](http://ww4.sinaimg.cn/large/006tNc79ly1g4zkm8l4xjj30jj0myta4.jpg)

## OAuth 2.0
在前面，我们可以看到，从Digest Access ，到 Appid + HMAC ，再到 JWT，再到 OAuth1.0，这些API 认证都是要向 Client 发一个秘钥，
然后用 hash 或者是 RSA 来签名HTTP请求，**这其中主要的原因有两个，以前的http 都是明文传输，所以在传输的过程中容易被篡改，于是搞出一套
安全的签名机制**，所以，这些认证玩法是可以在HTTP 明文协议下玩。

这种使用签名的方式比较复杂，所以，对于开发者来说，也不是很友好，在组织签名的那些HTTP 报文的时候，各种 URLEncode和Base64，还要对Query
参数进行排序，然后有的方法还要层层签名，非常容易出错，另外，这种认证的安全粒度比较粗，授权也比较单一，对于有终端用户参与的移动端来说有点不够。
所以在2012年的时候，OAuth2.0的 [RFC 6749](https://tools.ietf.org/html/rfc6749) 正式放出。

**OAuth 2.0 依赖于 TLS/SSL 的链路加密技术（HTTPS），完全放弃了签名的方式，认证服务再也不返回什么token secret的秘钥了，所以，OAuth2.0
是完全不同于1.0的，也是不兼容的**。目前，Facebook 的 Graph API 只支持OAuth 2.0协议，Google 和 Microsoft Azure 也支持Auth 2.0，
国内的微信和支付宝也支持使用OAuth 2.0。

下面，我们来重点看一下OAuth2.0的两个主要Flow：
- 一个是 AuthorizationCodeFlow，这个是 3-legged flow
- 一个是 ClientCredentialFlow，这个是 2-legged flow

### Authorization Code Flow
Authorization Code 是最常用使用的 OAuth2.0 的授权许可类型，它适用于用户给第三方应用授权访问自己信息的场景。这个Flow也是OAuth2.0中，
我个人觉得最完整的一个Flow，其流程如下：
![AuthorizationCodeFlow](http://ww1.sinaimg.cn/large/006tNc79ly1g4zkz1ff06j30jt0e7jri.jpg)

下面是对这个流程的细节上的解释：
1、当用户访问第三方用户的时候，第三方应用会把用户带到认证服务器上去，主要请求的是 /authorize API,其中请求方式如下：
```
https://login.authorization-server.com/authorize?
        client_id=6731de76-14a6-49ae-97bc-6eba6914391e
        &response_type=code
        &redirect_uri=http%3A%2F%2Fexample-client.com%2Fcallback%2F
        &scope=read
        &state=xcoiv98CoolShell3kch
```
其中，
    - client_id 是客户端Appid
    - response_type=code 是告诉认证服务器，我要走 Authorization Code Flow
    - redirect_uri 是调转会的第三方应用 url
    - scope 是相关权限
    - state 是 一个随机字符串，主要用户方 CSRF 攻击。

2、当Authorization Server收到这个请求之后，会通过 client_id 来检查这个 redirect_uri 和 scope 是否合法，如果合法，弹出页面
让用户授权（如果用户没有登录，则先登录，登录完成后，出现授权访问页面）。

3、当用户授权同意访问之后，AuthorizationServer 会调转会 client，并以其中加入一个 AuthorizationCode，如下所示：    
```https://example-client.com/callback?
           code=Yzk5ZDczMzRlNDEwYlrEqdFSBzjqfTG
           &state=xcoiv98CoolShell3kch
```

4、接下来，client 就可以使用 AuthorizationCode 获得 AccessToken，其需要向 AuthorizationServer 发出如下请求
```
POST /oauth/token HTTP/1.1
Host: authorization-server.com
 
code=Yzk5ZDczMzRlNDEwYlrEqdFSBzjqfTG
&grant_type=code
&redirect_uri=https%3A%2F%2Fexample-client.com%2Fcallback%2F
&client_id=6731de76-14a6-49ae-97bc-6eba6914391e
&client_secret=JqQX2PNo9bpM0uEihUPzyrh
```

5、如果没有什么问题，Authorization 会返回如下信息：
```
{
  "access_token": "iJKV1QiLCJhbGciOiJSUzI1NiI",
  "refresh_token": "1KaPlrEqdFSBzjqfTGAMxZGU",
  "token_type": "bearer",
  "expires": 3600,
  "id_token": "eyJ0eXAiOiJKV1QiLCJhbGciO.eyJhdWQiOiIyZDRkM..."
}
```
其中，
    - access_token 就是访问请求令牌
    - refresh_token 用于刷新 access_token
    - id_token 是 JWT的token，其中一半会包含用户的 OpenID

6、接下来就是用 AccessToken 请求访问用户资源。
```
GET /v1/user/pictures
   Host: https://example.resource.com
   
   Authorization: Bearer iJKV1QiLCJhbGciOiJSUzI1NiI
```       

### Client Credential Flow
Client Credential 是一个简化版的 API 认证，主要是用于认证服务器到服务器的调用，也就是没有用户参与的认证流程，
下面是相关的流程图。
![clientCredential](http://ww3.sinaimg.cn/large/006tNc79ly1g4zldd1jusj30g90bkwei.jpg)

这个过程非常简单，本质上就是Client用自己的 client_id 和 client_secret 向Authorization Server 要一个AccessToken
，然后使用AccessToken 访问相关资源。

请求示例：
```POST /token HTTP/1.1
   Host: server.example.com
   Content-Type: application/x-www-form-urlencoded
   
   grant_type=client_credentials
   &client_id=czZCaGRSa3F0Mzpn
   &client_secret=7Fjfp0ZBr1KtDRbnfVdmIw
```

返回示例：
```
{
  "access_token":"MTQ0NjJkZmQ5OTM2NDE1ZTZjNGZmZjI3",
  "token_type":"bearer",
  "expires_in":3600,
  "refresh_token":"IwOGYzYTlmM2YxOTQ5MGE3YmNmMDFkNTVk",
  "scope":"create"
}
```
这里，容我多扯一句，微信公从平台的开发文档中，使用了OAuth 2.0 的 Client Credentials的方式（参看文档“微信公众号获取access token”），
我截了个图如下所谓。我们可以看到，微信公众号使用的是GET方式的请求，把AppID和AppSecret放在了URL中，虽然这也符合OAuth 2.0，
但是并不好，因为大多数网关代理会把整个URI请求记到日志中。我们只要脑补一下腾讯的网关的Access Log，里面的日志一定会有很多的各个
用户的AppID和AppSecret……

## 小结

### 两个概念和三个术语
- 区分两个概念：Authentication （认证）和 Authorization（授权）
    
    Authentication 是证明请求身份，重点是**身份**。主要为了证明操作人是他本人，需要提供密码、短信验证码，甚至人脸识别。
    
    Authorization 是授权，重点是 **权限**。授权是不需要在所有的请求都需要验人（验证身份），是在经过 Authorization 后
    得到一个 Token，这就是 Authorization，就像是护照和签证一样。
    
    身份是区别于别人的证明，而权限是证明自己的特权。
    
    
- 区分三个概念：编码 Base64Encode、签名 HMAC、加密 RSA
    编码是为了更好的传输，等同于明文，签名是为了信息不能被篡改，加密是为了不让别人看到是什么信息。

### 明白一下初衷
- 使用复杂的 HMAC 哈希签名算法主要是应对当前没有 TLS/SSL 加密链路的情况。
- JWT 把 uid 放在 Token 中是为了去掉状态，但是不能让用户修改，所以需要签名。
- OAuth 1.0 区分了两个事，一个是第三方的 Client，一个是真正的用户，他先拿到 Request Token，再换 Access Token 的方法
主要是为了把第三方应用和用户区分开来。
- 用户的 Password 是用户自己设置的，复杂度不可控，服务端颁发的 Secret 会很复杂，但主要目的是为了容易管理，可以随时注销掉。
- OAuth 协议有比所有认证协议更为灵活完善的配置，如果使用 APPID/APPSecret签名的方式，又需要做到可以有不同的权限和可以随时注销，
那么你得开发一个像 AWS 的 IAM 这样的账号和秘钥对管理系统。

### 相关的注意事项
- 无论哪种方式，我们都应该遵循 HTTP 规范，把认证信息放在 Authorization HTTP 头中。
- 不要使用 GET 的方式-在URL 中放入 secret 之类的东西，因为很多 proxy 或 gateway 的软件会把整个 URL 记在 AccessLog 文件中。
- 秘钥 secret 相当于 password，但是他是用来加密的，最好不要放在网络上传输，如果要传输，最好使用 TLS/SSL 的安全链路。
- HMAC 中无论是 MD5 还是 SHA1/SHA2，其计算都是非常快的，RSA的非对称加密是比较耗费 CPU 的，尤其是要加密的字符串很长的时候。
- 最好不要在程序中 HardCode 你的secret，因为在 GitHub 上有很多黑客的软件在监控各种 secret，千万小心！
这类东西应该放在你的配置系统或是部署系统中，在程序启动时设置在配置文件或环境变量中。
- 使用 APPID/AppSecret ,还是使用 OAuth 1.0 或是 OAuth 2.0，还是 JWT，个人建议使用 TLS/SSL 下的OAuth 2.0。
- 秘钥是需要被管理的，管理就是可以新增，可以撤销，可以设置账户和相关的权限。最好秘钥是可以被自动更换的。
- 认证授权服务器（Authorization Server）和应用服务器（APP Server）最好分开。
