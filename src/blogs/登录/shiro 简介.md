### shiro 概念

- Subject ：主体，
代表了当前“用户”，这个用户不一定是一个具体的人，与当前应用交互的任何东西都是subject，如网络爬虫、机器人等；
	
    是一个抽象概念，所有subject 都绑定到 SecurityManager ， 与 Subject 的所有交互都委托给 SecurityManager ； 可以把 Subject 认为是一个门面； SecurityManager 才是实际的执行者。

- SecurityManager：安全管理器，
即所有与安全有关的操作都会与 SecurityManager 交互，他管理者所有的 Subject；可以看出他才是 Shiro 的核心，它负责与后边介绍的其他组件进行交互。相当于SpringMVC 中的 DispatcherServlet 前端控制器。
	
**它是shiro的核心，负责有其他所有的组件进行交互。**	

- Realm：域，可以有一个或多个
	shiro 从 Realm 获取安全数据（如用户、权限、角色等），也就是说 SecurityManager 要验证用户的身份，那么他需要从 Realm 获取相应的用户进行比较以确定用户身份是否合法；验证是否有权限进行操作，可以把 Realm 看成是 DataSource ，即安全数据源

	Realm是安全数据源，所有与验证有关的信息从这里获取.用户、权限、角色等。
	这里是需要用户自定义的地方。

- Authenticator：认证器
	负责subject认证，这里是一个扩展点，如果用户觉得shiro实现的不好，可以自定义实现；需要一个认证策略（Authentication Strategy ） ，表示什么情况下用户认证通过 

- Authrizer：授权器，访问控制器
	用来决定subject是否有权限进行相应的操作；即控制用户能访问应用中有哪些功能。

- SessionManager：管理session 的生命周期
	shiro 抽象了一个自己的session来管理 subject 与应用之间交互的数据；这样的话，比如我们在 web 环境用，刚开始是一台 web 服务器；接着又上了台 EJB 服务器；这是想把两台服务器的会话数据放在一个地方，这个时候就可以实现自己的分布式会话。

- SessionDAO： 用于会话的CRUD。

- CacheManager：缓存控制器
	用来管理用户、角色、权限等的缓存，这些数据很少去改变，放到缓存之后可以提高访问的性能。

- Cryptography：密码模块，shiro 提高了一下常见的加密组件用于密码的加解密。

- principals：身份 
	subject 的身份表示，用户名或邮箱都可。一个主体可以有多个principals，但是只有一个Primaryprincipals，一般是用户名/密码/手机号。

- credentials：证明/凭证
	只有subject 知道的安全值，如密码、数字证书等。

	最常见的 principals credentials 组合就是用户名和密码。


### shiro 集成 web
1、第一个过滤器-AbstractShiroFilter
subject 是后续动作的主体。
首先构造 subject：
- WebSubject
- DefaultSecurityManager
- CasSubjectFactory
- DefaultWebSubjectFactory
- DefaultSubjectContext 判断 subject 是否登录字段
    - 通过session判断 
    
2、其他过滤

获取 subject之后，通过 
executeChain(request, response,chain);
进行后续过滤。

**其中比较重要的是 PathMatchingFilter 过滤器，判断是否有权限（首先会判断是否登录，org.apache.shiro.web.filter.AccessControlFilter#onPreHandle 判断是否登录）
分为两个方法：onAccessAllow 和 onAccessDeny。如果subject已经登录，则不走登录；否则重新构造subject ，去server判断是否登录。**


