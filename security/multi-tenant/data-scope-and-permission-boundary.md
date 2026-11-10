# 多商户后台的数据隔离边界：身份、权限点、数据范围

## Problem

一个 B2B2C 平台给某商家开了八个客服账号，权限配得很干净：菜单里只有「订单」「售后」，没有「商户管理」，也没有「结算」。

三个月后对账，商家发现自己的客户名单和别家的重合度过高。查日志，其中一个客服账号访问过其他商户的订单详情，方式是把订单列表页地址里的一个参数改成了另一个商家的编号。

事后复盘有三句话：

- 菜单里确实没有别的商户，前端也确实点不到；
- 接口也确实要求登录，未登录访问会被拦；
- 所以问题不在"权限配置"，在于**这条查询允许命中哪些行，从来不是服务端决定的**。

后台权限其实有三个层次，混在一层做，迟早出事：

| 层次 | 回答的问题 | 典型实现 | 漏掉的后果 |
| --- | --- | --- | --- |
| 认证 Authentication | 你是谁 | token / session | 谁都能进 |
| 功能权限 Authorization | 你能不能调这个接口 | 权限点、角色 | 换个 URL 就绕过菜单 |
| 数据范围 Data Scope | 这次查询能命中哪些行 | 归属字段 + 服务端强制过滤 | 同一接口看到别家的数据 |

三层里只有第三层和"多商户"直接相关，也是自研后台最容易整层缺失的一层。前两层通常有人做，因为漏了会立刻打不开页面；第三层漏了，用总管理员账号测试永远发现不了。

## Business Scenario

一个平台同时承载平台自营、入驻商家、线下门店，三类身份共用一个后台前端：

| 身份 | `role_id` | 可见数据范围 |
| --- | --- | --- |
| 平台运营 | 9 | 全部商户 |
| 入驻商家 | 2 | 自己的 `store_id` |
| 门店 | 3 | 自己的 `chain_id`，且该门店必须属于所在商家 |

约束条件，也是这类系统真实的样子：

1. 三类身份共用同一个 `/manage` 路由前缀、同一套菜单、同一批列表接口。不是三套后台。
2. 数据源是同一批表：商品、订单、售后单、结算单、优惠券。差异只体现在行上的归属字段，不同业务表的归属字段还不一样（`store_id` / `chain_id` / `supplier_id`）。
3. 平台运营需要"代商家操作"：替商家改价、替商家发货。代操作必须留痕。
4. 商家员工离职，撤权限必须立即生效，不能等到 token 过期。

第 1 条决定了不能靠"给商家一套独立接口"来隔离；第 4 条决定了"把范围写进 token"这个看起来最省事的做法会留下一个撤不掉的窗口。

## Why It Happens

这一节用一个公开仓库（`shsuishang/golershop`，LGPL-3.0）的实际代码结构做对照。下面引用的是公开源码里能读到的**现状**，不是推荐做法；它的价值在于把"分层落点"这件事说明白——这种结构在自研商城后台里非常常见。

### 1. 把菜单当成权限

角色表里存的权限是一个菜单集合：

```go
// internal/model/entity/user_role.go（公开源码，节选字段）
type UserRole struct {
    UserRoleId   uint   `json:"user_role_id"`   // 角色编号
    UserRoleName string `json:"user_role_name"` // 角色名称
    UserRoleCode string `json:"user_role_code"` // 角色标识
    MenuIds      string `json:"menu_ids"`       // 请求列表(DOT)
}
```

`menu_ids` 的注释是「请求列表(DOT)」，但它的实际作用域是**菜单渲染**：登录后把可见菜单吐给前端，前端按它渲染侧边栏。

菜单和接口的差别是：菜单是"看不见"，接口是"调不到"。前端隐藏一个菜单只影响鼠标点击，不影响 `curl`。两者之间没有必然联系，除非你在服务端把菜单项和接口权限点绑定，并且每个接口都真的校验一遍。

### 2. 鉴权链只到"是否登录"

公开代码里的全局中间件注册顺序：

```go
// internal/cmd/cmd.go（公开源码，节选）
s.Use(service.Middleware().CORS)
s.Use(service.Middleware().NeverDoneCtx)
s.Use(service.Middleware().Ctx)
s.Use(service.Middleware().MiddlewareErrorHandler)
s.Use(service.Middleware().MiddlewareHandlerResponse)
s.Use(service.Middleware().CheckLogin)
```

`CheckLogin` 的判定只有一句：

```go
// internal/logic/shopsuite/middleware/middleware.go（公开源码，节选）
if !gstr.InArray(urlItem, u.Path) {
    if service.Login().IsSignedIn(r.Context()) {
        r.Middleware.Next()
    } else {
        // 返回 401 需要登录
    }
}
```

而 `IsSignedIn` 的实现是：

```go
// internal/logic/account/login/login.go（公开源码，节选）
func (s *sLogin) IsSignedIn(ctx context.Context) bool {
    if v := service.BizCtx().Get(ctx); v != nil && v.User != nil {
        return true
    }
    return false
}
```

也就是说这一层只回答一个问题：**请求里有没有一个能解析通过的 token**。它不回答"这个 token 属于谁、能做什么"。功能权限和范围判断，全压在业务代码里。

### 3. 数据范围从来没有进登录态

`Ctx` 中间件解析 token 后写入上下文的结构是四个字段：

```go
// internal/logic/shopsuite/middleware/middleware.go（公开源码，节选）
service.BizCtx().SetUser(r.Context(), &model.ContextUser{
    UserId:      claim.UserId,
    UserAccount: claim.UserAccount,
    UserSalt:    claim.UserSalt,
    ClientId:    clientId,
})
```

`ClientId` 这一行值得多看一眼：

```go
clientId := uint(0)
if strings.HasPrefix(r.RequestURI, "/manage") {
    clientId = 1
} else {
    if strings.HasPrefix(r.RequestURI, "/front/account/user/info") {
        clientId = 1
    }
}
```

`ClientId` 是**按请求路径前缀算出来的**，不是按账号身份算的。这意味着在中间件层，"平台运营"和"前台注册买家"如果都持有一个有效 token，走 `/manage` 路径时看起来完全一样。

要看这个人是哪个商户的，只能去 `user_admin` 表：

```go
// internal/model/entity/user_admin.go（公开源码，节选字段）
type UserAdmin struct {
    UserId           uint `json:"user_id"`            // 用户编号
    UserRoleId       uint `json:"user_role_id"`       // 权限角色
    UserIsSuperadmin bool `json:"user_is_superadmin"` // 是否超管
    RoleId           uint `json:"role_id"`            // 角色编号: 0-用户; 2-商家; 3-门店; 9-平台
    ChainId          uint `json:"chain_id"`           // 所属门店
    StoreId          uint `json:"store_id"`           // 店铺编号
}
```

表和字段都在，信息是齐的。缺的是"**每个查询都必须带上它**"这条约束——靠每个开发者自觉在每处 `Where` 里补一行，是撑不住的。

### 4. 查询条件由请求参数决定

管理端列表接口的典型写法是把请求结构体直接扫进查询条件：

```go
// internal/controller/trade/manage_order_base.go（公开源码，节选）
func (c *cOrderBase) List(ctx context.Context, req *trade.OrderListReq) (res *trade.OrderListRes, err error) {
    input := do.OrderInfoListInput{}
    gconv.Scan(req, &input)
    ml.ConvertReqToInputWhere(req, &input.Where, &input.WhereExt)
    result, error := service.Order().List(ctx, &input)
    ...
}
```

这种"请求即条件"的映射本身没有错，它是后台 CRUD 的标准做法。问题在于：如果归属字段（`store_id`）也在请求结构体里，那它就成了一个普通筛选条件——传什么查什么，不传就查全部。

这一刻，"越权"不需要攻击，只要改一个数字。

### 5. 入口太多，只加固了一处

同一份数据在后台至少有六个入口：

```
列表页查询     GET  /manage/order/list
详情页         GET  /manage/order/detail?id=
导出 Excel     GET  /manage/order/export
批量操作       POST /manage/order/cancel（body 里是 id 数组）
异步任务       MQ consumer / 定时对账
对外开放接口   /openapi、Webhook 回调
```

把"只在列表页加了 `store_id` 条件"当成修复，是最典型的半成品：导出没加，批量操作没加，异步任务压根没有登录态，于是数据从别的口子照样流出去。

## Design

**Recommended Design**。三层分开，各自落点明确，任何一层缺失都能被另一层的约束暴露出来。

```
Layer 1  认证         token → userId               中间件
Layer 2  功能权限     permission code → 接口       中间件 / 路由声明
Layer 3  数据范围     scope（服务端推导）→ 查询     仓储层唯一出口
```

三条原则，比具体实现更重要：

1. **范围只能由服务端推导。** 前端可以指定"我按门店筛选"，但不能指定"我属于哪个门店"。前者是查询条件，后者是身份属性。
2. **范围是查询的必填参数，不是可选条件。** 让 "不传 scope" 在类型层面就编译不过，比任何 code review 规则都可靠。
3. **默认拒绝。** 查不到归属信息的账号，返回空集，不返回全量。

### 三者职责的划分

| 关注点 | 归属层 | 判断依据 | 失败返回 |
| --- | --- | --- | --- |
| 是否登录 | 认证 | token 有效性、账号状态 | 401 |
| 能不能调这个接口 | 功能权限 | 角色 → 权限点集合 | 403 |
| 能看哪些行 | 数据范围 | `role_id` + `store_id` / `chain_id` | 空集（不返回 403） |

第三条失败时返回空集而不是 403，是有意为之：403 会告诉对方"这个资源存在，但你没权限"，在多商户场景下等于泄露了资源存在性。空集什么都不说。

## Data Model

**Recommended Design**。把"菜单可见性"和"接口可调用性"拆成两张表，别再用一个 `menu_ids` 字段同时承担两件事。

```sql
-- 权限点：一个 code 对应一组接口，粒度按"业务动作"而不是按"表"
CREATE TABLE `sys_permission` (
  `perm_id`     INT UNSIGNED NOT NULL AUTO_INCREMENT,
  `perm_code`   VARCHAR(64)  NOT NULL COMMENT '权限点标识，如 order:view / order:refund',
  `perm_name`   VARCHAR(64)  NOT NULL COMMENT '中文名，用于后台勾选',
  `perm_group`  VARCHAR(32)  NOT NULL DEFAULT '' COMMENT '分组，仅用于展示',
  `perm_kind`   TINYINT      NOT NULL DEFAULT 1 COMMENT '1-菜单 2-动作',
  PRIMARY KEY (`perm_id`),
  UNIQUE KEY `uk_code` (`perm_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='权限点';

-- 角色 → 权限点（多对多，替代 menu_ids 里的 DOT 串）
CREATE TABLE `sys_role_permission` (
  `role_id`   INT UNSIGNED NOT NULL,
  `perm_code` VARCHAR(64)  NOT NULL,
  `ctime`     DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`role_id`, `perm_code`),
  KEY `idx_perm` (`perm_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='角色权限';
```

归属字段和索引的关系，决定了越权查询是否"顺手就能防住"：

```sql
-- 归属字段必须在业务主查询索引的最左端，否则"先按商户过滤"会退化成全表扫
ALTER TABLE `order_info`
  ADD KEY `idx_store_state_ctime` (`store_id`, `order_state_id`, `create_time`);
```

理由很直接：如果 `store_id` 不在索引前缀，开发者会觉得"加了这个条件查询变慢"，然后绕过去。**让正确的写法同时也是快的写法**，比写文档更有效。

在请求进入业务之前的身份表（沿用公开仓库的字段语义，仅说明结构）：

```
user_base   (user_id, user_account, user_password, user_salt)          -- 账号
user_info   (user_id, user_nickname, user_state)                       -- 状态：停用即失效
user_admin  (user_id, role_id, store_id, chain_id, user_is_superadmin) -- 后台身份与归属
user_role   (user_role_id, user_role_code, ...)                        -- 角色定义
```

`role_id` 的取值语义（`0` 用户 / `2` 商家 / `3` 门店 / `9` 平台）之所以重要，是因为它决定了 `store_id` 是"约束"还是"可选项"：平台身份的 `store_id` 通常是 0，此时不能套用"必须等于某个 store"的规则。

## Flow

**Recommended Design** 的请求链路，七个步骤，任何一步缺失都会让后面几步失效：

```
① 取 token           Authorization: Bearer <token>
② 校验签名与过期     失败 → 401
③ 加载身份与范围      user_id → user_admin(role_id, store_id, chain_id)
                     缓存 key 带权限版本号；账号停用 / 改角色 → 版本 +1
④ 写入请求上下文      scope = {role_id, store_id, chain_id, is_platform}
⑤ 校验权限点          route → perm_code；role 是否拥有   失败 → 403
⑥ 构造查询            repository 强制注入 scope 条件（不接受外部覆盖）
⑦ 写审计日志          operator_id + scope + 目标对象归属 + 是否代操作
```

第 ③ 步的缓存和 ④ 的落点是同一件事的两面：范围必须**在服务端、在每次请求里**重新确定，不能只在登录那一刻确定一次并写进 token。原因在 Edge Cases 里展开。

平台代操作走不同分支：

```
平台运营（is_platform=true）调用同一接口
  → scope.store_id 为空，允许显式传 target_store_id
  → 该校验 target_store_id 是否存在、是否可用
  → 审计日志记录 operator_id = 平台账号, on_behalf_of = target_store_id
```

## Example

**Example**，不要这样写。这是越权的标准形态，也是最容易通过 code review 的形态，因为它看起来完全正常：

```go
// ✗ 反例：归属字段来自请求
type OrderListReq struct {
    StoreId  uint `json:"store_id"`   // 前端传，不传就是 0
    StateId  uint `json:"order_state_id"`
    Page     int  `json:"page"`
}

func (c *OrderController) List(ctx context.Context, req *OrderListReq) (*OrderListRes, error) {
    m := dao.OrderInfo.Ctx(ctx)
    if req.StoreId > 0 {
        m = m.Where("store_id", req.StoreId)     // 不传 → 查全部
    }
    if req.StateId > 0 {
        m = m.Where("order_state_id", req.StateId)
    }
    ...
}
```

三个问题叠在一起：条件可选、来源是请求、默认全量。

**Example**，正确形态的中间件。用 GoFrame 的中间件签名，因为公开仓库用的是这套框架；换成 Gin 只是函数签名不同，分层不变：

```go
// 示例：把身份和范围一次性解析进上下文，业务代码不再自己查
func (s *sMiddleware) Authz(r *ghttp.Request) {
    ctx := r.Context()
    user := service.BizCtx().GetUser(ctx)
    if user == nil {
        r.Response.WriteStatus(http.StatusUnauthorized)
        return
    }

    // ① 身份与归属：只查一次，带缓存，缓存 key 含权限版本
    admin, err := service.AdminIdentity().Load(ctx, user.UserId)
    if err != nil || admin == nil || admin.Disabled {
        r.Response.WriteStatus(http.StatusUnauthorized)
        return
    }

    scope := model.Scope{
        RoleId:     admin.RoleId,
        StoreId:    admin.StoreId,
        ChainId:    admin.ChainId,
        IsPlatform: admin.RoleId == consts.ROLE_PLATFORM,
    }
    ctx = model.WithScope(ctx, scope)

    // ② 权限点：路由 → perm_code，角色不持有则 403
    perm := service.Permission().CodeOfRoute(r.URL.Path, r.Method)
    if perm != "" && !service.Permission().Has(admin.RoleId, perm) {
        r.Response.WriteStatus(http.StatusForbidden)
        return
    }

    r.SetCtx(ctx)
    r.Middleware.Next()
}
```

仓储层是唯一出口，`scope` 是必填参数。签名上没有 `scope` 的查询方法不允许存在——这条比任何规范文档都有效，因为漏写会编译不过：

```go
// 示例：范围在查询构造的最左边注入，外部无法覆盖
func (r *orderRepo) ListByScope(ctx context.Context, scope model.Scope, in *do.OrderListInput) ([]*entity.OrderInfo, error) {
    m := dao.OrderInfo.Ctx(ctx).Where("order_state_id", in.StateId)

    switch {
    case scope.IsPlatform:
        // 平台：允许按目标商户筛选，不筛就是全部
        if in.StoreId > 0 {
            m = m.Where("store_id", in.StoreId)
        }
    default:
        // 商家 / 门店：范围先钉死，再叠加业务条件
        if scope.RoleId == consts.ROLE_STORE {
            m = m.Where("chain_id", scope.ChainId)
        } else {
            m = m.Where("store_id", scope.StoreId)
        }
    }

    return m.OrderDesc("create_time").Scan()
}
```

单条详情是最容易漏的地方：列表加了过滤，详情按主键查，改个 `id` 就能看到别家的订单。详情必须走同一条范围约束：

```go
// 示例：按主键查也要带范围条件，查不到就是查不到
func (r *orderRepo) DetailByScope(ctx context.Context, scope model.Scope, orderId uint) (*entity.OrderInfo, error) {
    m := dao.OrderInfo.Ctx(ctx).WherePri(orderId)
    if !scope.IsPlatform {
        m = m.Where("store_id", scope.StoreId)   // 门店身份再叠加 chain_id 条件
    }
    one, err := m.One()
    if err != nil || one == nil {
        return nil, gerror.NewCode(gcode.CodeNotFound, "订单不存在")
    }
    return one, nil
}
```

批量操作按 id 数组处理时，范围条件同样不能省。校验和写入放在同一个事务里，避免"校验时属于我、写入时已改归属"：

```go
// 示例：批量取消，先按范围筛出可操作集合，再只处理这个集合
func (s *orderService) CancelByScope(ctx context.Context, scope model.Scope, ids []uint) (ok []uint, err error) {
    return ok, dao.OrderInfo.Transaction(ctx, func(ctx context.Context, tx gdb.TX) error {
        query := dao.OrderInfo.Ctx(ctx).WhereIn("order_id", ids).Where("order_state_id", consts.ORDER_STATE_UNPAID)
        if !scope.IsPlatform {
            query = query.Where("store_id", scope.StoreId)
        }
        allowed, err := query.Array("order_id")
        if err != nil {
            return err
        }
        for _, id := range allowed {
            ... // 只处理 allowed 里的，不处理入参里的
        }
        return nil
    })
}
```

## Edge Cases

这一节列的都是上线后才暴露的情形，按出现频率从高到低。

### 1. 改了角色，旧 token 还能用

token 里如果写了 `store_id` 或权限列表，撤权限必须等 token 过期。商城后台的 token 有效期通常是小时级，撤权限窗口就是小时级。

所以范围**不进 token**，进服务端缓存，缓存 key 带一个"权限版本号"。改角色、停用账号、调商户时版本 +1，下一次请求自然失效。

```sql
-- 示例：账号表加一列版本，任何权限变更都 +1
ALTER TABLE `user_info`
  ADD COLUMN `perm_version` INT UNSIGNED NOT NULL DEFAULT 1 COMMENT '权限版本，变更权限时 +1';
```

### 2. 异步任务和定时任务没有登录态

订单超时关闭、对账、结算生成、消息推送，这些任务不经过中间件，也没有 scope。

两种可行做法：

- 按商户分片跑：先查商户列表，逐个商户带 `store_id` 执行，天然隔离；
- 任务入参里显式携带 `scope`，由发起方（有登录态的请求）在投递时写入，消费方只读。

第二种更通用，但要防住"发起方伪造 scope"——投递前必须校验 scope 来源。

### 3. 导出是最容易漏的一个入口

导出通常单独实现（流式写 Excel），不复用列表查询。列表修完忘了导出，等于没修。做法上不要让导出自己写 SQL，让它复用 `ListByScope`，只换输出层。

### 4. 门店跨商户

门店表里如果只有 `chain_id` 没有 `store_id`，门店身份与商户身份的绑定关系要么靠联表，要么靠冗余。联表最容易在分页时写丢，结果就是门店看到了全平台。

建议在门店表上冗余 `store_id` 并在写入时校验一致性，查询时用 `chain_id`，鉴权时用 `store_id`。

### 5. 归属变更后的历史数据

商品从 A 商户调到 B 商户（平台代运营、商户退出、门店换签），历史订单的 `store_id` 是写死的还是跟着变的？

结论是**写死**：订单、结算单这类单据的归属字段一旦生成不再变更，变更归属走"新单据 + 原单据关闭"。否则B 商户会看到 A 商户的历史订单，财务口径也会乱。

### 6. 平台代操作的越权边界

平台运营能看全部，不等于能操作全部。代操作要单独授权（`order:ops-on-behalf` 这样的独立权限点），且必须留痕：

```
operator_id    = 平台账号
on_behalf_of   = 目标商户
reason         = 代操作原因（必填，进审计）
```

没有 `reason` 的代操作，事后无法向商户解释。

### 7. 回调与开放接口不走用户鉴权

支付回调、物流推送、ERP 回传，这些请求没有用户身份，走的是签名鉴权。它们的"范围"由业务单号本身决定（回调里带 `order_no`，只能改那一单），不要试图套用用户 scope。

但要防另一件事：回调里的 `store_id` 如果是可写的，就成了一条改归属的通道。回调只允许改状态，不允许改归属。

## Practical Notes

### 可以直接抄进测试用例的六条

1. 商家账号 A，把列表接口的 `store_id` 改成商户 B → 结果为空或仅 A。
2. 商家账号 A，把详情接口的 `id` 换成 B 的订单 → 404。
3. 商家账号 A，导出接口 → 导出的行全部属于 A。
4. 商家账号 A，批量取消里混入 B 的订单 id → B 的订单状态不变，且响应里不出现 B 的 id。
5. 门店账号，`chain_id` 属于 A 商户 → 只能看该门店；伪造 `chain_id` 属于 B → 空。
6. 停用账号后立刻再请求 → 401（不能等 token 过期）。

前五条用总管理员账号测不出来，所以测试环境必须准备至少两个商户的数据。

### code review 的一条硬规则

请求结构体里出现 `store_id`、`chain_id`、`supplier_id`、`merchant_id` 这类归属字段，就要问一句：**它作为查询条件，还是作为身份来源？**

- 作为查询条件：可以留，但只在平台身份分支里生效。
- 作为身份来源：删掉，从 `scope` 取。

这条规则能拦下绝大多数越权。

### 审计日志记什么

```
request_id     请求链路 ID
operator_id    操作人
role_id        身份
scope          推导出的范围
perm_code      命中的权限点
target_type    对象类型（order / settlement / product）
target_id      对象 ID
target_owner   对象归属（store_id）
on_behalf_of   代操作目标（可空）
result         成功 / 业务失败 / 拒绝
```

没有 `target_owner` 的审计日志，事后无法回答"这个操作动了谁的数据"，而这恰恰是商户投诉时第一个要回答的问题。

### 依赖的框架能力要确认，不要假设

公开代码里用的是 GoFrame（`ghttp`、`gdb`）加一层自研的 `mallsuite/gocore` 条件转换。换框架或升级版本时，这几点要重新确认：

- 中间件执行顺序是否与注册顺序一致（`Use` 的先后决定谁能拦到谁）；
- 请求参数扫描（`gconv.Scan`）是否会覆盖结构体里已有的默认值；
- 条件转换（`ConvertReqToInputWhere`）是否会把空值转成 `IS NULL` 语义。

这类转换层最容易出现"看起来加了条件、实际没生效"的情况，改完必须打印最终 SQL 核对，不要只看代码。

## Summary

多商户后台的隔离不是一个功能，是两个约束：

1. **身份与范围在服务端推导**，永远不接受前端传入的归属字段作为身份来源；
2. **范围是查询的必填参数**，让它在编译期就不可省略，而不是靠每个人记得补 `Where`。

菜单权限解决的是"界面长什么样"，登录校验解决的是"进不进门"。这两层都做了而数据仍然串户，是因为第三层——这次查询允许命中哪些行——从来没有被明确地放在某一段代码里。

把这一层落到仓储层的唯一出口上，多商户才算真的隔离。剩下的都是细节：导出有没有走同一条路，异步任务有没有带 scope，撤权限要不要等 token 过期。

---

本文整理自随商商城项目和企业电商研发实践。
