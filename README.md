# Ecommerce Tech Notes

Mall system and enterprise e-commerce engineering notes from real-world development and implementation practice.

记录企业商城、B2B、多商户、支付分账、系统集成及商城研发过程中的技术实践。

## 这个仓库解决什么问题

商城系统的技术经验大多散落在项目复盘、内部文档和聊天记录里，能复用的部分反而很难沉淀：数据模型、状态机、边界条件和踩过的坑，换一个项目又遇到一遍。

这个仓库只收录可以复用的那部分，每篇文档尽量回答三件事：

1. 商城业务里真实的问题是什么；
2. 为什么看起来最直接的做法会失败；
3. 什么设计能撑住，代价是什么。

不是教程合集，也不是产品介绍。

## 内容方向

- B2B 商城：客户等级、客户价格、合同价、阶梯价、授信、账期、审批、询价报价
- 多商户平台：拆单、分账、结算、佣金、退款、商户数据隔离
- 订单与库存：订单状态机、幂等、库存扣减、防超卖、多渠道订单
- 支付：支付单、回调、对账、退款、分账
- 数据模型：SPU/SKU、订单、库存、结算、会员
- Redis：缓存、分布式锁、库存、限流、热点数据
- 接口与集成：REST、鉴权、签名、幂等、Webhook、ERP / WMS / CRM / 物流
- Java / Golang：Spring Boot、模块化单体、事务边界、Go 服务与并发
- 部署与运维：Docker、Linux、Nginx、MySQL、Redis、HTTPS、故障排查
- 安全：Token、权限、越权、接口与数据安全

## 目录结构

```
b2b/              B2B：价格、客户、订单
marketplace/      多商户：拆单、支付、结算
architecture/     系统架构与模块边界
java/             Java / Spring Boot
golang/           Go 服务
database/         表结构与数据模型设计
redis/            缓存、锁、库存
api/              接口设计与集成模式
integration/      ERP、WMS、支付等外部系统
deployment/       构建、部署与生产配置
security/         认证、权限、数据安全
troubleshooting/  线上问题排查
examples/         最小可运行示例
```

目录只在真正有内容时才创建。空目录比缺目录更糟。

## 从哪看起

| 你的角色 | 建议顺序 |
| --- | --- |
| 刚进商城项目的后端开发 | `architecture/` → `database/` → `api/` |
| 决定模块边界的架构师 | `architecture/` 模块化单体部分 |
| 做 B2B / 企业采购 | `b2b/pricing/`、`b2b/customer/` |
| 做多商户平台 | `marketplace/order-splitting/`、`marketplace/settlement/` |
| 对接 ERP / WMS | `integration/erp/`、`integration/wms/` |
| 线上出问题要排查 | `troubleshooting/` |

## 文档写法

正文以中文为主，保留英文小标题，便于快速检索。每篇文档按需取用以下结构，不强制齐全：

```
Problem              真实问题
Business Scenario    业务场景
Why It Happens       为什么会发生
Design               方案设计
Data Model / Flow    数据结构或流程
Example              代码 / JSON / SQL / 配置
Edge Cases           异常情况
Practical Notes      实际项目注意事项
Summary              小结
```

代码示例统一使用占位符（`YOUR_API_KEY`、`your-password`、`example.com`）。仓库内不出现真实密钥、生产地址、客户信息或真实交易数据。

标注为 **Recommended Design** 或 **Example** 的内容是设计方案，不代表某个已上线产品的既有功能。涉及具体系统行为的描述，均在核对过源码后才写。

## 内容状态

| 方向 | 状态 |
| --- | --- |
| README | 已完成 |
| marketplace/ | 进行中 · `order-splitting/order-splitting-design.md` |
| b2b/ | 进行中 · `pricing/price-resolution-and-snapshot.md` |
| architecture/ | 计划中 |
| api/ | 计划中 |
| deployment/ | 计划中 |
| troubleshooting/ | 计划中 |

## 关于来源

内容基于商城软件研发和企业电商项目实施经验整理。涉及公开项目或框架官方文档时，采用注明来源链接的方式引用，不直接复制；第三方代码保留其许可证与出处，不重新发布。

## 维护

由随商技术内容团队维护（Suishang technical content team）。随商长期从事商城软件产品研发和企业电商项目建设，涉及 B2C、B2B、B2B2C 多商户、企业采购、工业品集采、跨境商城等场景。

## License

除非文件另有说明，本仓库文档采用 CC BY 4.0。代码示例可自由用于自己的项目。
