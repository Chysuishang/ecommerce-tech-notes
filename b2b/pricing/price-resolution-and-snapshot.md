# B2B 商城的多价格体系：价格解析与下单价格快照

> 本文为 **Recommended Design**，整理自 B2B 商城项目的价格模块设计取舍。文中的表名、字段名、接口与优先级顺序均为示例方案，不代表某个具体已上线产品的既有实现。

## Problem

B2B 商城里，同一个 SKU 会同时命中多种价格来源：基础价、客户等级折扣、客户专属合同价、阶梯量价、促销活动价、区域或渠道价。这本身是业务需要，不是设计缺陷——企业采购的价格本来就是「一客一价、一单一议」。

真正的故障来自另外三件事：

1. **同一个价格在链路上被算了多次，结果不一致。** 商品列表页、详情页、购物车、结算页、下单接口各有一份取价代码。客户看到 ¥88.00，提交后订单是 ¥92.00，客服无法解释。
2. **优先级没有显式建模。** 谁覆盖谁写在 `if/else` 的顺序里。新加一个「限时促销」就覆盖了签过合同的客户价，接着就是商务纠纷。
3. **订单只存了最终单价，没存这个价是怎么来的。** 三个月后客户问「这单为什么是 88」，唯一能回答的人是当初写这段代码的人。

B2C 里价格错一分钱用户未必发现；B2B 的单个订单金额可能是几十万，合同价被活动价覆盖是可以进入正式争议流程的问题。

## Business Scenario

一个典型组合：

经销商 A 的状态：

- 客户等级 = 金牌，等级折扣 92 折；
- 与商家签了年度合同价：SKU-X 固定 ¥88.00，合同期 2026-01-01 ~ 2026-12-31；
- 平台当前在跑活动：该 SKU 满 100 件立减 ¥500。

A 下单 SKU-X 共 120 件。

需要回答四个问题：

1. 这 120 件按 ¥88.00 还是 ¥88.00 × 0.92 算？
2. 满 100 件立减能不能叠加在合同价之上？
3. 数量跨过 100 件门槛，是整行按一个价，还是前 99 件一个价、后 21 件另一个价？
4. 下完单第二天商家改了合同价，这张订单的金额会不会跟着变？

这四个问题在同一个「取价函数」里没有答案，因为它们分属四个不同的设计决策。

## Why It Happens

### 1. 把「价格」当成一个值，而不是一次计算结果

价格不是 SKU 上的一个字段，而是一次计算的输出，输入至少包括：客户、商户、数量、时间、渠道、活动状态。只要输入在链路上有任何一处不同，结果就会不同。

所以「详情页显示 ¥88，下单变 ¥92」通常不是随机的 bug，而是两处输入的差异：详情页没带客户身份，下单带了。

### 2. 优先级靠代码顺序表达

```javascript
// 反例：优先级 = 代码顺序
let price = sku.basePrice;
if (activity) price = activity.price;
if (customerLevel) price = price * levelDiscount;
if (contract) price = contract.price;   // 谁写在后面谁生效
```

这段代码不是逻辑错，而是**不可配置、不可解释、不可测试**。每加一个规则都要改一次代码，改完没人说得清影响面。

### 3. 没有快照

价格规则表是会变的：合同会续签、活动会结束、等级会调整。如果订单行只引用了规则 ID，而没有把当时的单价与命中条件固化下来，历史订单金额就会随规则变更而漂移，对账、退款、开票同时失效。

### 4. 金额精度与分摊

用浮点存金额、每一行独立四舍五入，会出现 Σ 行小计 ≠ 订单总额。剩下的一分钱在对账环节是灾难，因为它是「每次都不一样」的那种错误。

## Design

核心思路：**把价格计算收敛到一个入口，把结果固化成快照。**

### 先把三个概念分开

| 概念 | 含义 | 例子 |
| --- | --- | --- |
| 价格来源 Source | 价格从哪里来 | 合同价、等级价、阶梯价、活动价、基础价 |
| 解析策略 Policy | 多个来源同时命中时谁生效、能否叠加 | 优先级、堆叠性、决胜规则 |
| 价格快照 Snapshot | 下单那一刻的计算结果与依据 | 最终单价、命中规则、数量档位、算价时间 |

很多实现把这三件事混在一个函数里，于是既改不动，也解释不了。

### 单一解析入口

**Recommended Design**：所有价格出口都经过同一个解析能力，输入输出一次定义清楚。

```text
resolve(PriceContext) -> PriceDecision[]
```

- `PriceContext`：`{ customerId, merchantId, channel, customerLevel, at, items[] }`
- `items[]`：`{ skuId, qty }`
- `PriceDecision`：`{ skuId, qty, basePrice, finalUnitPrice, lineAmount, source, matchedRuleId, tierId, stackableApplied, reason }`

硬约束：商品列表页、详情页、购物车、结算页、下单接口**全部调用这一个解析接口**，前端不允许自己乘折扣。

列表页可以只传 `skuId`、不带客户身份，拿到的是基础价，但必须显式标注为「参考价」，不能把 `resolve()` 在无客户上下文下的结果当成客户的成交价展示。

### 优先级用配置表达

每个价格来源用两个属性描述：

- `priority`：谁赢，数值越大越优先；
- `stackable`：赢家是否允许低优先级的规则继续在其上叠加。
  - `stackable = false`：命中即终止解析；
  - `stackable = true`：继续向下匹配，在已得价格上再计算。

一个可用的初始顺序（示例，应按业务调整）：

```text
合同价 (100)  >  阶梯价 (80)  >  客户等级价 (60)  >  促销活动价 (40)  >  基础价 (0)
```

常见配置取舍：

- 合同价通常 `stackable = false`——签了合同就按合同执行，不再参与平台活动；
- 等级折扣通常 `stackable = true`——允许在活动价上继续享受等级优惠；
- 活动价是否可再叠加，是纯粹的商务决策，不是技术细节，所以它必须存在配置里，而不是代码里。

### 解析流程

```text
1. 收集候选规则
   - 按 merchantId 过滤
   - 按 scope 匹配：SKU / 类目 / 全店
   - 按客户范围匹配：customerId / customerLevel / 全部客户
   - 按有效期过滤：at ∈ [start_at, end_at)
2. 按 priority 降序排序，同优先级按 ruleId 升序（保证结果确定）
3. 逐条判定命中
   - 阶梯价：用整行 qty 确定唯一档位
   - 命中后记录 matchedRuleId 与 tierId
   - 若 stackable = false，终止；否则继续判定下一条
4. 生成 PriceDecision；下单时写入价格快照
```

第 2 步的「同优先级决胜规则」必须有，而且必须确定。缺失时数据库返回顺序看似稳定，但语义上没有保证：换存储引擎、加一个索引、改一次执行计划，价格就可能变。价格在这种地方飘一下，代价很高。

**时间参数必须显式传入。** 解析函数接收 `at`，而不是内部调用 `now()`。这样接口可重放、可测试，异步补单也能按业务时间算价。

### 阶梯价口径必须二选一，并全链路统一

| 口径 | 规则 | 适用场景 |
| --- | --- | --- |
| 整行适用 | 整行 qty 落在哪一档，整行都按该档单价 | 批发、按量议价（推荐默认） |
| 分段累加 | 前 99 件按第 1 档，后 21 件按第 2 档 | 阶梯式计价，商城场景少见 |

推荐**整行适用**：用户能预期，财务能核对，订单行不需要再拆数量区间。

选哪一种都可以，重要的是详情页、购物车、结算页、下单、退款、对账六处口径一致。口径不一致时，同一个订单在不同页面显示不同金额，问题会被归因为「系统不稳定」，而不是「规则没定清楚」。

### 快照

下单时把决策结果写入订单行（或其关联快照表），至少包含：

- `final_unit_price`：最终成交单价（整数分）；
- `price_source`：命中的来源类型；
- `price_rule_id` / `tier_id`：命中的规则与档位；
- `base_price`：解析前基础价，用于展示「已优惠 X」；
- `calc_qty`：参与算价的数量；
- `calc_at`：算价时间。

订单行的金额字段一旦写入，**不允许 UPDATE**。价格变更、退款、对账全部以快照为唯一口径。

## Data Model / Flow

以下表结构为 **Recommended Design**，字段名按团队习惯调整。金额统一用整数分（`BIGINT`）。

```sql
-- 价格规则主表：合同价 / 等级价 / 阶梯价 / 活动价共用
CREATE TABLE mall_price_rule (
  id              BIGINT       NOT NULL AUTO_INCREMENT,
  merchant_id     BIGINT       NOT NULL,
  source_type     VARCHAR(16)  NOT NULL COMMENT 'BASE|LEVEL|CONTRACT|TIER|ACTIVITY',
  scope_type      VARCHAR(16)  NOT NULL COMMENT 'SKU|CATEGORY|SHOP',
  scope_id        BIGINT       NOT NULL COMMENT '按 scope_type 解释，SHOP 时为 0',
  target_type     VARCHAR(16)  NOT NULL COMMENT 'ALL|LEVEL|CUSTOMER',
  target_id       BIGINT       NOT NULL DEFAULT 0 COMMENT '等级 ID 或客户 ID，ALL 时为 0',
  price_mode      VARCHAR(16)  NOT NULL COMMENT 'FIXED_PRICE|DISCOUNT_RATE',
  price_value     BIGINT       NOT NULL COMMENT 'FIXED_PRICE=单价(分)；DISCOUNT_RATE=万分比，9200 表示 92 折',
  priority        INT          NOT NULL DEFAULT 0,
  stackable       TINYINT(1)   NOT NULL DEFAULT 0,
  start_at        DATETIME     NOT NULL,
  end_at          DATETIME     NOT NULL,
  status          TINYINT      NOT NULL DEFAULT 1 COMMENT '1 启用 0 停用',
  PRIMARY KEY (id),
  KEY idx_lookup (merchant_id, scope_type, scope_id, status, start_at, end_at)
) COMMENT '价格规则';

-- 阶梯价档位，仅 source_type = TIER 的规则使用
CREATE TABLE mall_price_tier (
  id          BIGINT NOT NULL AUTO_INCREMENT,
  rule_id     BIGINT NOT NULL,
  min_qty     INT    NOT NULL,
  max_qty     INT    NOT NULL COMMENT '极大值表示无上限',
  price_value BIGINT NOT NULL COMMENT '该档单价(分)或万分比折扣',
  PRIMARY KEY (id),
  KEY idx_rule (rule_id, min_qty)
) COMMENT '阶梯价档位';

-- 订单行价格快照：只写不改
CREATE TABLE order_item_price_snapshot (
  id               BIGINT   NOT NULL AUTO_INCREMENT,
  order_id         BIGINT   NOT NULL,
  order_item_id    BIGINT   NOT NULL,
  sku_id           BIGINT   NOT NULL,
  customer_id      BIGINT   NOT NULL,
  base_price       BIGINT   NOT NULL COMMENT '解析前基础价(分)',
  final_unit_price BIGINT   NOT NULL COMMENT '最终成交单价(分)',
  calc_qty         INT      NOT NULL,
  line_amount      BIGINT   NOT NULL COMMENT '行小计 = final_unit_price * calc_qty',
  price_source     VARCHAR(16) NOT NULL,
  price_rule_id    BIGINT   NOT NULL DEFAULT 0,
  tier_id          BIGINT   NOT NULL DEFAULT 0,
  calc_at          DATETIME NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_item (order_item_id),
  KEY idx_order (order_id)
) COMMENT '订单行价格快照';
```

折扣率用**整数万分比**（9200 = 92 折）而不是小数存储：既避免 `0.92` 这类浮点在乘法后再取整时的边界抖动，也避免「92 折」被配置界面存成 `0.9199999`。

### 流程

```text
下单请求
  └─ PricingService.resolve(ctx)
        ├─ 读规则（一次批量查询，按 skuId 分组，避免逐行 N+1）
        ├─ 排序 + 命中判定
        └─ 返回 PriceDecision[]
              └─ 订单落库（同一事务）
                    ├─ order_main / order_item
                    └─ order_item_price_snapshot  ← 固化单价与依据
```

解析本身是纯计算，不写库；快照写入与订单创建在同一事务内完成，避免出现「订单已创建但没有价格依据」的中间状态。

## Example

### 请求

```json
{
  "customerId": 100238,
  "merchantId": 5012,
  "customerLevel": "GOLD",
  "channel": "WEB",
  "at": "2026-09-22T10:00:00+08:00",
  "items": [
    { "skuId": 88001, "qty": 120 }
  ]
}
```

### 响应

```json
{
  "merchantId": 5012,
  "decisions": [
    {
      "skuId": 88001,
      "qty": 120,
      "basePrice": 12000,
      "finalUnitPrice": 8800,
      "lineAmount": 1056000,
      "source": "CONTRACT",
      "matchedRuleId": 77102,
      "tierId": 0,
      "stackableApplied": false,
      "reason": "命中客户专属合同价，该规则不可叠加，不参与平台活动"
    }
  ]
}
```

金额单位为分：`8800` 表示 ¥88.00，`1056000` 表示 ¥10560.00。`basePrice` 为 ¥120.00。

`reason` 字段是给客服和商务看的，不是调试日志——它回答的就是「为什么是这个价」。

### 解析伪代码

```text
function resolve(ctx):
  rules = loadRules(ctx.merchantId, ctx.at, ctx.customerId, ctx.customerLevel, ctx.items[].skuId)
  out = []
  for item in ctx.items:
    candidates = filterByScopeAndTarget(rules, item, ctx)
    candidates.sortBy(priority desc, ruleId asc)

    decision = { basePrice: basePriceOf(item.skuId), finalUnitPrice: null }
    for r in candidates:
      if not hit(r, item, ctx): continue
      price = applyRule(r, item.qty)     # FIXED_PRICE 直接取；DISCOUNT_RATE 用 basePrice 乘万分比
      decision.finalUnitPrice = price
      decision.matchedRuleId  = r.ruleId
      decision.tierId         = (r.sourceType == TIER) ? tierOf(r, item.qty) : 0
      decision.source         = r.sourceType
      decision.stackableApplied = r.stackable
      if not r.stackable: break          # 不可叠加，终止解析

    if decision.finalUnitPrice == null:
      decision.finalUnitPrice = decision.basePrice
      decision.source = "BASE"

    decision.lineAmount = decision.finalUnitPrice * item.qty
    out.push(decision)

  return out
```

### 快照写入

```sql
INSERT INTO order_item_price_snapshot
  (order_id, order_item_id, sku_id, customer_id, base_price, final_unit_price,
   calc_qty, line_amount, price_source, price_rule_id, tier_id, calc_at)
VALUES
  (?, ?, 88001, 100238, 12000, 8800, 120, 1056000, 'CONTRACT', 77102, 0, '2026-09-22 10:00:00');
```

## Edge Cases

**数量跨档。** 用户把数量从 99 改成 100。若口径是「整行适用」，整行 100 件都按新档单价，可能出现「买得更多、总额反而更低」的跳变。这是设计意图，但详情页必须提示「再加 1 件，整单单价降至 ¥XX」，否则会被当成 bug 投诉。

**同优先级多条命中。** 客户同时签了两份覆盖同一 SKU 的合同价。必须由决胜规则决定（`ruleId` 升序 / 生效时间最新 / 创建时间最新），并且要做保存期校验：**同一 `merchant_id + scope + target` 且有效期重叠、`priority` 相同的规则，在配置保存时就告警**，而不是等下单时才随机命中一条。

**时间边界。** 有效期统一用左闭右开 `[start_at, end_at)`，避免 `23:59:59` 与次日 `00:00:00` 之间的一秒错位。所有时间用同一时区存储。

**按哪个时间算价。** 用**下单时间**，不用支付时间。否则用户在 23:59 下单、次日 00:01 支付，跨过了活动结束时间，就会看到一个自己从未见过的价格。

**等级变更。** 客户从金牌降到银牌，未支付订单是否重算？推荐**不重算**，以下单时快照为准。否则会出现「下单 ¥88、付款前变 ¥95」。重算只能由人工或明确的业务动作触发，并且要留痕。

**零价与负价。** 折扣用乘法不会出负数，但「立减固定金额」的活动会。必须有兜底：解析结果不低于 0（或业务规定的最低成交价），并且触发告警——低于兜底价通常意味着规则配置错了，而不是捡到便宜。

**退款口径。** 部分退款按快照的 `final_unit_price` 退，不能按当前价格退。多行订单的优惠分摊要落到行，余数用最大余额法补给金额最大的行，保证 Σ 行退款不超过实付金额。

**缓存失效。** 解析结果可以缓存，但缓存 key 必须包含 `customerId + merchantId + channel + skuId`；规则变更（含活动开始与结束）时必须能精确失效。用「活动开始时全量清价格缓存」会造成尖峰，配置短 TTL 更稳。

**最小起订量与多包装。** 若 SKU 有最小起订量或按箱下单（1 箱 = 12 件），阶梯价的 `qty` 必须与算价单位统一，否则档位判定会整档偏移。

## Practical Notes

- **禁止前端算价。** 前端只展示解析接口返回的价格。允许做「划线价 / 现价」的视觉处理，不允许做折扣运算。
- **显式传入时间。** 解析函数收 `at` 参数，内部不取 `now()`。这是可测试性与补单能力的前提。
- **快照只写不改。** 任何修改订单金额的需求都走逆向单据，不 UPDATE 订单行金额。
- **可解释性优先于性能。** `reason` 和 `matchedRuleId` 看起来是额外存储，但客服与商务每天都在用。把它当需求，而不是当调试日志。
- **规则上线前要能预览。** 后台应支持「选一个客户 + 一个 SKU，直接看到他会按什么价买」，而不是发到生产上试。
- **区分参考价与成交价。** 未登录用户和列表页拿到的必须是基础价并明确标注，不能复用带客户身份的解析结果。
- **价格规则表要能审计。** 谁在什么时候把合同价从 ¥96 改成 ¥88，需要有变更记录。价格调整在 B2B 里是商务事件，不只是数据变更。
- **规则数量增长要有预期。** 逐客户签合同价会让规则表随客户数线性增长。查询必须走 `merchant_id + scope` 的复合索引批量取，不能逐 SKU 查询。

## Summary

B2B 的价格问题通常不是「折扣算错了」，而是三件工程上的事没做：

1. **单一解析入口**——价格只能有一个出口，展示价与成交价同源；
2. **优先级配置化**——用 `priority` + `stackable` 表达商务规则，而不是靠 `if/else` 的书写顺序；
3. **下单快照**——把最终单价、命中规则、数量档位、算价时间固化到订单行，且只写不改。

做到这三点之后，合同价、等级价、阶梯价、活动价同时命中就不再是事故，而是一个可以被解释、被审计、被复现的计算结果。

---

本文整理自随商商城项目和企业电商研发实践。
