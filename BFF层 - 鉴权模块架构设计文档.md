概述：

BFF层鉴权模块需求 - 实现客户端用户权限隐式授权，避免用户多次显示授权


架构设计图：

<img src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/%E5%9B%BE%E7%89%871.png" />

流程步骤：
- 一、客户端发送授权请求 - 并附带Client_id，userMsg等验证参数
- 二、BFF层转发授权请求，向验证服务器发送授权请求
- 三、验证服务器校验数据，成功则返回Token，失败则通知BFF层，转发至用户登录
- 四、数据校验成功，BFF层将Token与用户信息发送至验证服务器获取Access_Token
- 五、验证服务器校验Token与用户信息确认无误，返回Access_Token
- 六、BFF层将Access_Token写入Session会话
- 七、客户端与服务器通过Access_Token进行数据交流
- 八、若Access_Token失效则根据是否存在Expire_Token来决定重新验证或者重新授权


存在问题：
- 对请求参数与返回参数不明确，目前没有明确目标开始流程制作
- 对于验证失败需重新登录流程不确定	
- 对于整个隐式鉴权流程需要再次实际确认方能完全确认
