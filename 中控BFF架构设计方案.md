
# 中控BFF-架构设计


## 1. 应用架构

### 访问流程：

 1. 应用加载动态配置并初始化应用相关项完成应用启动
 2. 前端发送API请求，请求将被分为客户端请求与管理端请求两个部分：
	
	- 客户端：仅开放数据访问权限，并开放跨域以支持不同域名下请求访问数据
	- 管理端：同时开放读、写操作权限，并进行用户权限、数据格式校验以及SSO登陆鉴权，同时对用户的操作进行记录
 3. 经过控制器处理后开始访问服务层，服务层根据具体请求开始访问数据库并返回相应数据或操作结果
 4. 请求经过处理后返回相关数据或相关错误情况，同时访问将会被日志系统记录


### 流程图：

![图片描述](https://pubimg.xingren.com/c3c92736-f9e1-4209-8052-ed5480885958.png)


 
## 2. 流程与模块介绍

### 1. 配置加载与应用初始化
- 配置的加载：
默认配置将包括应用基本信息、中间件、错误处理等，对于SSO鉴权插件及数据库连接将根据环境
动态配置或通过Appllo在项目启动时进行动态配置。
- 应用初始化：
初始化将包括项目启动、辅助函数注入、Appllo配置动态加载等，在项目启动时进行统一初始化操作。


### 2. Middleware中间件

- 日志记录：
日志将通过logger中间件进行统一的记录，包括访问日志与错误日志等。
日志使用[egg-logger](https://github.com/eggjs/egg-logger)插件进行记录并遵循[BFF日志规范](https://www.tapd.cn/39130625/markdown_wikis/?#1139130625001009701)
日志保存包括本地文件与控制台输出,本地文件将记录在./los下目录同时根据日志等级记录在对应文件内，
同时日志将会被打印输出到stdout以便运维同事统一记录日志。


- 操作记录：
操作记录中间件用于记录Admin用户所执行的__增、删、查、改__ 操作记录，
以便监控和查询具体的操作执行人与操作时间等数据，增强系统的安全稳定性。


- 错误处理：
错误处理将包含BFF错误、网络错误、MySQL数据库错误等，所有错误将通过统一包装并返回如下格式，
对于errorCode将遵守[restify-error](https://github.com/restify/errors)规范，对错误码进行规范。

```
{
	success: false,
	errorCode: xxx,
	errorMessage: '~~~'
}
```

- Utils辅助函数
utils函数用于帮助快速开发、减少开发工作量以提升开发效率。
目前utils包括返回数据格式封装、日期格式化、mysql查询语句序列化等函数。


### 3. 数据库设计与使用
- 描述：
数据库延续目前JAVA后台使用MySQL数据库，并依旧分为开发、测试、生产三个独立数据库。
生产环境数据库将定期将生产的数据同步到测试环境，以减免不必要的额外操作。
数据库使用[egg-mysql](https://eggjs.org/zh-cn/tutorials/mysql.html)进行连接，依据不同环境采用不同配置

- 配置示例：
```
{
  mysql: {
    client: {
      host: '589d8f49e9299.gz.cdb.myqcloud.com',
      port: '3450',
      user: 'cdb_outerroot',
      password: 'xxxxxxxx',
      database: 'offcialsite'
    },
    app: true
  }
}
```

- ER图：
![图片描述](https://pubimg.xingren.com/8d3eb8f6-2e93-4c00-8b8a-43f1d0c492ac.png)


### 4. 控制器与服务层
- 控制器
控制器负责与相应的路由相匹配，解析用户的输入，处理后返回相应的结果。
官网中控的控制器将分为客户端与管理端两个部分：
	- 客户端：仅开放数据访问权限，并开放跨域以支持不同域名下请求访问数据
	- 管理端：同时开放读、写操作权限，并进行用户权限、数据格式校验以及SSO登陆鉴权，同时对用户的操作进行记录
- 服务层
服务层负责将逻辑和展现分离，保持业务逻辑的独立性。
主要内容包括与数据库的交互并对数据库的查询与操作进行统一的封装和错误处理，将结果返回控制器层。


### 5. 安全处理与风险防范

- SSO登陆插件
SSO登陆流程图：
![图片描述](https://pubimg.xingren.com/7bb8cbcc-5c53-4d4e-9f1e-02fa7037c36e.png)
SSO登陆流程描述：
1.客户端发送API请求，SSO插件进行白名单校验，白名单路由直接校验通过
2.非白名单路由进入Token校验，对`Token`进行 `jwt` 解析，校验其合法性以及是否过期, 校验失败则正式进入SSO登陆流程
3.登陆流程开始浏览器重定向到 {sphinx}/sphinx/oauth/authorize?${params}, sphinx检测用户未登陆跳转企微扫码页面
4.扫码成功返回code, 跳转重定向页面。BFF接收code，开始向sphinx发送Token请求
5.成功获取Access_Token通过token开始向normandy请求用户数据
6.用户数据请求成功，将Token与用户信息写入cookie，跳转至首页，登陆成功
SSO插件配置示例：
````
{
  whiteLists: ['/404', '/401', `/front`], // 路由登陆白名单[]则为全部打开, 路由path遵循 path-to-regex原则
  sso: {
    authHost: 'https://sphinx.aihaisi.com', // SSO sphinx 地址
    authPath: '/sphinx/oauth/authorize', // SSO sphinx 验证接口
    tokenPath: '/oauth/token', // SSO sphinx token 获取接口
    redirectHost: 'https://offcialsite.aihaisi.com', // SSO 回调域名
    loginPath: `${BASE_URL}/admin/sso`, // SSO sphinx 回调地址
    verifyPath: 'http://normandy2.aihaisi.com/api/token/verify', // normandy 用户信息接口
    appId: 'officialsite', // SSO sphinx appID
    appCode: 89, // normandy 应用ID
    appSecret: 'officialsite', // SSO sphinx appSecret
    loginRedirectPath: '/' // 登录成功跳转页面
  }
}
```


- 用户权限控制
用户权限控制是对于用户具体对于系统的某个模块或功能的权限控制，目前用户权限控制的设置流程一共分为三步：
1. 在[normandy](http://normandy2.aihaisi.com/#/manage/application/)上创建应用与用户角色
2. 将拥有系统操作权限人员加入此角色并与上面创建好的应用相绑定并配置好权限表
3. 在Admin系统中读取用户权限表依据不同权限展示不同模块或功能。用户权限配置完成


- 安全风险防范
安全风险防范是指BFF层对于网络安全防范采取的具体措施：
1. [CSRF](https://www.owasp.org/index.php/CSRF)(Cross-site request forgery跨站请求伪造)，CSRF防范将在生产环境启用，防止cookie伪造而造成的安全问题，
在测试及开发环境为了方便开发调试将会关闭。
2. [XSS](https://www.owasp.org/index.php/Cross-site_Scripting_(XSS))（cross-site scripting跨域脚本攻击)， 此防范将会在全部环境下开启，对于来自的前端的请求进行escape字符串过滤。
3. [Cors](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)（跨域资源请求），此防范将会在应用端关闭以便前端资源实现跨域资源请求，在Admin管理端此防范开会开启，防止任何恶意的跨域资源或操作请求。


## 6. 辅助与支持模块
- Swagger接口文档生成
接口文档遵守[Swagger](https://www.gitbook.com/book/huangwenchao/swagger/details)规范，默认Swagger文档将会在build时生成，同时通过`yapi import`命令进行文档生成与上线

- Appllo配置读取
Appllo配置读取用于动态的加载某些重要的配置，例如线上数据库配置密码。
Node使用[node-appllo](https://github.com/Quinton/node-apollo)进行Appllo配置的加载与使用

- 应用测活
BFF层专门提供了/alive接口用于服务的测活，测活失败服务器将重启Node应用，以防止应用崩溃。

- 静态资源转发
由于前后端在一个域名下，对于Admin前端而已这是一个SPA的单页服务，所以需要将资源进行转发，
目前所有请求将先经过BFF层，对于静态资源请求BFF将会将其重定向到静态文件`/static/index.html`来实现前端SPA应用。

- 单测与压测
(〃'▽'〃)持续跟进中




