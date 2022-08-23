## 背景
鉴于第三方合作商与业务发展的需要，Javascipt SDK包的开发被提上日程，用于为第三方合作商提供一个开箱即用的能力，目前Javascipt SDK需要提供两种功能，其一提供Javascipt API接口用于第三方接入Wooshpay完成支付，其二通过更为简单化的UI自动生成让第三方合作商快速、便捷接入Wooshpay。

## 实现思路

鉴于整个SDK能力的需要，首先我们根据合作商的需求场景，简单绘制需求场景图：

<img src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/tapd_62821792_1650945631_12.png" />

这里Wooshpay SDK提供了两种使用场景：
- 需要Wooshpay UI，由WooshPay渲染UI界面，商户仅需接入SDK，配置参数即可。商户可根据自身需要自定义修改UI界面
- 商户自行收集支付信息，调用SDK相关接口完成支付流程，这里商户自身需要关注支付流程与结果，并根据自身需求做出对应操作

一期Wooshpay UI的制作将会比较简单，仅提供基本的表单展示，数据校验功能，因此需要商户根据自身需要自行修改表单UI样式。

在完成UI层渲染后，接下来便是SDK API的设计，首先我们明确Wooshpay SDK能力范围：
- 支持3DS安全校验
- 支持用户支付功能
- 支持交易取消
- 支持交易单查询
- 更新交易单信息

然后我们在接下来的环节中确认整个SDK的交互逻辑与交易系统的实现逻辑，确保系统功能稳定无错漏。

## SDK交互流程设计

在SDK的交互中角色包括用户、SDK、Server三者，因此整个交互流程也仅会在此三者间流转，具体如下图：

<img src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/tapd_62821792_1651817455_36.png" />

## 3Ds支付流程设计

<img src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/tapd_62821792_1651996486_20.png" />

## 工程化设计
考虑到SDK系统的复杂性，为保证系统的稳定使其良好运行并支持其日后的可扩展能力，这里需要着重强调一下整个SDK工程化的重要性。
工程化将主要围绕项目开发脚手架、CI/CD自动化部署、性能分析与错误监控三个方面来进行，具体工程化结构图如下：

<img src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/tapd_62821792_1650943651_36.png" />

- 开发脚手架：
	负责开发环境启动、代码调试等功能
- 项目编译：
	自动化完成项目编译打包，实现SDK代码压缩、加密、资源优化等功能
- 单元测试：
	对SDK包所提供的API进行单元测试，确保其功能的上线稳定
- 部署发布：
	由Jekins或Gitlab CI等CI自动化化工具实现项目的自动发布与回滚功能
- 错误收集：
	收集线上环境错误，用于排查线上BUG、及时发现、修复问题
- 性能分析：
	对整个SDK UI渲染、接口调用延时等数据进行分析，提升用户使用体验


## 风险防控

## 错误与性能监控
技术实现方案：


