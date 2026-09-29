# 订单与库存的扣减边界：下单扣、支付扣还是预占

> 本文属于 `architecture/` 方向，讨论订单模块与库存模块之间的职责边界与事务边界。
> 文中标注 **Recommended Design** 或 **Example** 的部分是设计方案与示例代码，不代表某个已上线产品的既有实现。

## Problem

商城系统里最早暴露、也最容易反复踩的架构问题，不是"库存怎么扣"，而是**库存应该在哪个环节扣、由哪个模块负责扣、扣减失败时谁来兜底**。

先看两种典型的失败序列。

**失败序列 A：先查后扣，中间有窗口**

1. 用户 A 下单，订单服务查库存，剩余可售 `10`，校验通过，订单创建成功；
2. 同一毫秒用户 B 下单，同样查到 `10`，也创建成功；
3. 两个订单都支付成功，仓库只有 10 件，超卖 2 件；
4. 运营在活动结束后对账才发现，只能人工联系用户取消并赔付。

**失败序列 B：支付时才扣，钱已收**

1. 下单不占用库存，订单先建出来；
2. 支付回调里扣库存，条件更新影响行数为 0（库存不足）；
3. 钱已经收到，货发不出去，只能走退款流程。

两种失败指向同一件事：**"检查库存"和"占用库存"如果不在同一个原子操作里，中间就存在一个一定会被并发穿透的窗口**；而"占用"如果晚于收款，失败代价就从"少卖一单"变成"收了钱没货"。

这个问题的代价分布很不均：出问题的是活动高峰那几分钟，但处理成本要摊到客服、财务和运营身上。

## Business Scenario

同一个"扣库存"，在不同业务下的正确边界并不一样。

| 场景 | 业务特征 | 库存失效的代价 | 对库存精度的要求 |
| --- | --- | --- | --- |
| 日常 B2C 零售 | 流量平稳，SKU 多，单量大 | 少卖 / 赔付 | 偏高，但可容忍秒级延迟 |
| 限时活动、秒杀 | 极短时间高并发同一 SKU | 超卖，舆情风险 | 极高，必须强一致 |
| B2B 合同备货 | 单笔数量大，允许部分发货 | 交期违约，影响合作关系 | 精度要求低于数量准确性 |
| 多商户平台 | 库存归属商户，平台只做聚合展示 | 商户间纠纷，平台担责 | 依赖商户同步时效 |

这四类场景的差异点集中在两个问题上：

- **谁承担"占用"的最终决策权**：平台库存中心，还是商户自己的仓储系统；
- **占用要多快失效**：用户下单后 15 分钟不付款必须自动释放，否则活动期间库存会被"僵尸订单"锁死。

先明确这两点，才能选扣减时机。选错时机，后面用再多补偿逻辑都补不回来。

## Why It Happens

把原因拆开，会发现大多数超卖不是"程序员写错了"，而是几个结构性因素叠加。

**1. 校验与写入不在同一个原子操作内。**
`SELECT` 出库存判断够不够，再 `UPDATE` 扣减，两条语句之间是开放的窗口。哪怕只隔 1 毫秒，活动期间也足够挤进几百个请求。这不是并发量的函数，是并发量的存在性函数——只要并发存在，窗口就会被穿透。

**2. 库存数据被放在两个存储里。**
Redis 放一份用于扛读，MySQL 放一份用于持久化。两边的扣减顺序、失败处理、补偿逻辑稍有差异，就会出现"缓存显示有货、下单失败"或者反过来"缓存显示没货、实际有货"。真正的坑不在双写本身，在于**双写失败后没有人负责让两边重新一致**。

**3. 扣减动作被拆到了另一个模块。**
订单模块建单，库存模块扣减，中间隔着一次服务调用或一条消息。跨模块就没有本地事务，只能靠"先做完 A 再通知 B"，而通知会失败、会重试、会乱序。这时"做了什么"和"记录了什么"之间的差异就变成了数据不一致。

**4. 支付回调不是只来一次。**
支付渠道的回调是"至少一次投递"。同一个支付单的回调可能重复到达，也可能比预期晚到几分钟。如果扣库存写在回调里且没有幂等键，重复回调就会重复扣减。

**5. 释放逻辑没有对应的所有者。**
下单占用库存很容易想到；下单后不付款、订单取消、支付失败、部分退款，这些路径要不要回补库存，回补多少，谁触发，往往没人明确负责。缺少这些路径，库存会随时间单向减少，直到运营手动修数据。

## Design

### 三种扣减时机

**Recommended Design** —— 对绝大多数商城业务，推荐第三种：下单预占 + 支付确认扣减。

| 方案 | 何时占用 | 何时真正扣减 | 优点 | 代价 |
| --- | --- | --- | --- | --- |
| A. 下单即扣 | 创建订单时 | 同时完成 | 实现最简单，不会超卖 | 未支付订单锁死库存，需要超时释放机制；活动期间可售量虚低 |
| B. 支付成功扣 | 支付回调时 | 同时完成 | 库存不被未付款订单占用，成单率看起来更好 | 必然出现"付了钱没货"，只能退款；回调并发下仍可能超卖 |
| C. 预占 + 确认 | 创建订单时预占（`reserved`） | 支付回调时由预占转实扣（`deducted`） | 下单时即锁定，不超卖；未支付可自动释放；支付环节不再依赖实时库存 | 需要预占台账与超时释放任务，状态多一层 |

方案 B 的问题不在于实现难度，而在于**它把失败挪到了不可逆的位置**：下单失败用户会重试，付款后失败用户只会投诉。只要业务不能接受"收了钱再退"，B 就不该作为主方案。

### 预占模型的状态流转

**Recommended Design** —— 库存台账按"可售 / 预占 / 已扣"三个量管理，而不是一个库存数字加减。

```
                    ┌────────────────────────────────────────────┐
                    │  available 可售量 = 实物库存 - reserved - deducted │
                    └────────────────────────────────────────────┘

下单 ──▶ RESERVE ──▶ reserved +n, available -n
          │
          ├── 支付成功 ──▶ CONFIRM ──▶ reserved -n, deducted +n
          │
          ├── 订单取消 / 支付失败 ──▶ RELEASE ──▶ reserved -n, available +n
          │
          └── 超时未支付 ──▶ 定时任务 RELEASE ──▶ reserved -n, available +n
                        （必须幂等：已确认的预占不可释放）

退款（已确认）──▶ RESTOCK ──▶ deducted -n, available +n
```

三条约束值得单独写出来：

1. **预占是可释放的，实扣是不可释放的。**释放动作必须校验当前状态仍是 `reserved`，否则会释放掉已经支付的那部分。
2. **所有流转都是幂等的。**同一个 `reserve_no` 重复 RESERVE、重复 CONFIRM、重复 RELEASE 都必须只生效一次。
3. **可用量是推导值，不是维护值。**`available` 由三个量算出或由条件更新维护，禁止出现"直接给 available 赋值"的代码路径，否则并发下必然丢更新。

### 模块边界

**Recommended Design** —— 库存模块是唯一有权修改库存台账的模块，订单模块只持有"预占凭证"。

- 订单模块负责：决定要占多少、什么时候占、什么时候释放；保存 `reserve_no`。
- 库存模块负责：台账的原子增减、状态机校验、幂等、流水记录；对外只暴露 `reserve / confirm / release / restock` 四个语义化操作。
- 订单模块**不允许**直接 `UPDATE stock`。一旦允许，任何新写的业务分支都会绕过幂等和流水，最终无法对账。

这条边界带来的直接好处是：库存是否一致，只需要审计库存模块一个地方。

## Data Model / Flow

### 表结构

**Example** —— 库存台账与预占台账分离。台账保存当前量，流水保存"为什么变成这个量"。

```sql
-- 库存台账：一行 = 一个 SKU 在一个仓库的量
CREATE TABLE stock_ledger (
  id            BIGINT       NOT NULL AUTO_INCREMENT,
  sku_id        BIGINT       NOT NULL COMMENT 'SKU 主键',
  warehouse_id  BIGINT       NOT NULL COMMENT '仓库主键',
  physical_qty  INT          NOT NULL DEFAULT 0 COMMENT '实物在库量',
  reserved_qty  INT          NOT NULL DEFAULT 0 COMMENT '已预占未确认',
  deducted_qty  INT          NOT NULL DEFAULT 0 COMMENT '已确认实扣',
  version       INT          NOT NULL DEFAULT 0 COMMENT '乐观锁版本',
  updated_at    DATETIME(3)  NOT NULL DEFAULT CURRENT_TIMESTAMP(3)
                             ON UPDATE CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  UNIQUE KEY uk_sku_wh (sku_id, warehouse_id),
  CONSTRAINT ck_qty_non_negative
    CHECK (physical_qty >= 0 AND reserved_qty >= 0 AND deducted_qty >= 0)
) ENGINE = InnoDB COMMENT '库存台账';

-- 预占台账：一行 = 一次预占，是幂等的唯一依据
CREATE TABLE stock_reservation (
  id             BIGINT      NOT NULL AUTO_INCREMENT,
  reserve_no     VARCHAR(64) NOT NULL COMMENT '预占单号，业务侧唯一',
  order_no       VARCHAR(64) NOT NULL COMMENT '来源订单号',
  sku_id         BIGINT      NOT NULL,
  warehouse_id   BIGINT      NOT NULL,
  qty            INT         NOT NULL COMMENT '预占数量',
  state          TINYINT     NOT NULL DEFAULT 1
                 COMMENT '1=预占中 2=已确认 3=已释放 4=已回补',
  expire_at      DATETIME(3) NOT NULL COMMENT '预占过期时间',
  created_at     DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  UNIQUE KEY uk_reserve_no (reserve_no),
  KEY idx_state_expire (state, expire_at)
) ENGINE = InnoDB COMMENT '库存预占台账';

-- 库存流水：只追加，用于对账与追溯
CREATE TABLE stock_movement (
  id            BIGINT      NOT NULL AUTO_INCREMENT,
  biz_no        VARCHAR(64) NOT NULL COMMENT '业务单号（预占单号/退款单号）',
  sku_id        BIGINT      NOT NULL,
  warehouse_id  BIGINT      NOT NULL,
  change_type   VARCHAR(16) NOT NULL
                COMMENT 'RESERVE / CONFIRM / RELEASE / RESTOCK',
  qty           INT         NOT NULL COMMENT '变动数量，正负均可',
  created_at    DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  UNIQUE KEY uk_biz_type (biz_no, change_type, sku_id) COMMENT '防重复记账',
  KEY idx_sku_time (sku_id, created_at)
) ENGINE = InnoDB COMMENT '库存流水';
```

`stock_movement` 上的 `uk_biz_type` 是这套模型里最便宜的一道保险：即使上层幂等判断漏了，重复记账也会在数据库层面被唯一键拦掉。

### 预占流程

```
Client ──▶ OrderService.createOrder
              │
              ├─ 1. 生成 reserve_no（订单号 + SKU序号，可重入）
              │
              ├─ 2. 调用 StockService.reserve(order_no, items[])
              │        │
              │        ├─ 开启本地事务
              │        ├─ 按 sku_id 升序逐个加锁（避免交叉死锁）
              │        ├─ 条件更新台账：available 足够才扣
              │        ├─ 写入 stock_reservation(state=1, expire_at=now+15m)
              │        ├─ 写入 stock_movement(RESERVE)
              │        └─ 提交
              │
              ├─ 3. 落库订单 + 保存 reserve_no
              └─ 4. 返回下单结果

支付回调 ──▶ PaymentService
              │
              ├─ 1. 按 payment_no 幂等判重（已处理则直接返回成功）
              ├─ 2. 更新支付单状态
              ├─ 3. 调用 StockService.confirm(reserve_no)
              │        └─ 更新 reservation: state 1→2（条件更新，影响行数 0 视为已处理）
              ├─ 4. 写入 stock_movement(CONFIRM)
              └─ 5. 推进订单状态
```

第 2 步的"按 `sku_id` 升序加锁"不是优化，是必需项。两个订单包含相同两个 SKU 但顺序不同时，会互相等待形成死锁；统一升序后，所有事务的加锁顺序一致，死锁概率降为零。

## Example

### 条件更新扣减（核心 SQL）

**Example** —— 用一条带条件的 `UPDATE` 完成"检查 + 占用"，杜绝先查后扣的窗口：

```sql
-- RESERVE：仅当可用量足够时才预占
UPDATE stock_ledger
SET reserved_qty = reserved_qty + #{qty},
    version      = version + 1
WHERE sku_id       = #{skuId}
  AND warehouse_id = #{warehouseId}
  AND physical_qty - reserved_qty - deducted_qty >= #{qty};

-- 影响行数 = 1 → 预占成功；= 0 → 可用量不足，整单回滚
```

```sql
-- CONFIRM：预占转实扣，只允许 state=1 → 2
UPDATE stock_reservation
SET state = 2
WHERE reserve_no = #{reserveNo}
  AND state = 1;

-- 影响行数 = 0 说明已被确认或已释放，按幂等直接返回成功

UPDATE stock_ledger
SET reserved_qty = reserved_qty - #{qty},
    deducted_qty = deducted_qty + #{qty}
WHERE sku_id = #{skuId} AND warehouse_id = #{warehouseId}
  AND reserved_qty >= #{qty};
```

```sql
-- RELEASE：超时释放，同样是条件更新
UPDATE stock_reservation
SET state = 3
WHERE reserve_no = #{reserveNo}
  AND state = 1;          -- 关键：只释放"预占中"的

-- 影响行数 = 1 时才回补台账
UPDATE stock_ledger
SET reserved_qty = reserved_qty - #{qty}
WHERE sku_id = #{skuId} AND warehouse_id = #{warehouseId}
  AND reserved_qty >= #{qty};
```

三条语句共用同一个模式：**把判断条件写进 `WHERE`，用影响行数作为唯一判据**。不要先查再判断，也不要在应用层用读到的值做决策。

### 幂等入口

**Example** —— 幂等判断放在更新之后、以影响行数为准，而不是先读后判：

```java
// 伪代码：confirm 的幂等处理
public void confirm(String reserveNo, List<Item> items) {
    for (Item it : items) {
        int rows = reservationMapper.markConfirmed(reserveNo, it.getSkuId());
        if (rows == 0) {
            // 已被确认或已释放，属于重复回调：跳过，不报错
            continue;
        }
        int ledgerRows = ledgerMapper.moveReservedToDeducted(
                it.getSkuId(), it.getWarehouseId(), it.getQty());
        if (ledgerRows == 0) {
            // 台账异常：预占存在但台账不足，必须告警而不是静默
            throw new StockInconsistentException(reserveNo, it.getSkuId());
        }
        movementMapper.insert(reserveNo, it.getSkuId(),
                              it.getWarehouseId(), "CONFIRM", it.getQty());
    }
}
```

这里刻意区分了两种"影响行数为 0"：**预占行数为 0 是正常幂等**，静默跳过；**台账行数为 0 是数据不一致**，必须告警。把两者都当成功处理，是线上问题被长期掩盖的常见原因。

### 超时释放

**Example** —— 用"扫状态 + 限量"的方式释放，不做全表扫描：

```sql
-- 分批捞取过期预占，每次 200 条，避免长事务
SELECT reserve_no, qty, sku_id, warehouse_id
FROM stock_reservation
WHERE state = 1
  AND expire_at < NOW(3)
ORDER BY expire_at
LIMIT 200;
```

```sql
-- 单条释放，即使任务重复执行也安全
UPDATE stock_reservation SET state = 3
WHERE reserve_no = #{reserveNo} AND state = 1;
```

释放任务的设计要点：

- **幂等**：任务重跑必须无副作用，靠 `state = 1` 的条件保证。
- **可观测**：每次执行的释放条数、失败条数要进监控；成功率突降通常意味着上游下单逻辑变了。
- **与支付回调竞争**：释放和确认可能同时到达。上面两条 `UPDATE` 都带 `state = 1` 条件，数据库行锁保证只有一个成功，另一个拿到 0 行并走幂等分支。

## Edge Cases

**1. 重复支付回调。**支付渠道"至少一次"投递，同一 `payment_no` 的回调可能来 2 次以上。处理原则：以支付单状态做第一层幂等，以预占 `state` 做第二层，以流水唯一键 `uk_biz_type` 做第三层。三层都过不去才会重复扣减。

**2. 释放与确认同时到达。**超时释放任务和支付回调可能几乎同时执行。必须让两者竞争同一行 `stock_reservation`，由数据库决定胜负，而不是靠应用层时间比较。**不要**在释放前再查一次订单状态然后决定——那个查询同样有窗口。

**3. 部分支付 / 部分发货。**B2B 场景常见"订单 100 件，客户先付 60 件"。此时预占应按可拆分的粒度建立，确认时按实际数量确认，剩余部分走释放而不是等超时。预占和实扣的粒度如果不一致，对账时会对不上。

**4. 退款回补。**退款不一定回补库存：未发货退款应回补，已发货且商品已损毁则不应回补。因此 `RESTOCK` 必须是显式动作，由退货单状态驱动，不能挂在"退款成功"事件上自动执行。

**5. 多仓库。**同一 SKU 分布在不同仓库，预占必须指定仓库。如果下单时不指定，就需要一层分配策略（就近仓、有货仓），而分配策略本身会引入新的并发点——建议在预占前一次性分配好，不要在下单链路里做动态选择。

**6. 组合商品 / 套装。**一个套装对应多个 SKU 的库存，预占需要展开成多条明细，并且**要么全部成功要么全部失败**。多明细的条件更新必须在同一事务内完成，任意一条影响行数为 0 就整体回滚。

**7. 库存初始化与盘点。**盘点调整的是 `physical_qty`，不应触碰 `reserved_qty` 和 `deducted_qty`。如果盘点时直接改总库存，会把进行中的预占冲掉。

## Practical Notes

**监控指标。**这套模型上线后，至少盯这几个量：

| 指标 | 含义 | 异常信号 |
| --- | --- | --- |
| `reserved_qty` 总量 / 可售量 | 预占占比 | 持续升高说明释放任务失效或用户不下单 |
| 预占平均存活时长 | 从 RESERVE 到 CONFIRM / RELEASE | 明显高于超时时间说明释放任务积压 |
| `stock_movement` 与台账差异 | 按 SKU 汇总流水变化 vs 台账当前值 | 不为 0 即数据不一致，需立即排查 |
| 下单库存失败率 | `reserve` 返回不足的比例 | 突增可能是超卖保护在起作用，也可能是库存数据有问题 |

最后一项特别容易被忽略：**库存失败率上升不一定是故障，可能是保护机制生效**。但如果它长期处于高位，说明可售量维护本身有问题。

**对账。**按 SKU 做日终对账，比较流水汇总与台账当前值。这套对账的成本很低（两条 `GROUP BY` 查询），但能在问题扩散前发现它。

**缓存。**缓存只用于展示和流量削峰，不参与扣减决策。展示层缓存可以容忍秒级偏差；下单链路的库存判断必须落到数据库的条件更新上。缓存与数据库的差异要能自愈（设置合理的过期时间），而不是依赖人工刷新。

**降级策略。**当库存服务不可用时应拒绝下单，而不是降级为"允许下单后补"。后者会把一次可用性故障转换成一次数据一致性故障，而数据问题修复成本远高于短暂的不可用。

## Summary

- 库存扣减的核心不是"怎么扣"，而是**"检查"和"占用"必须在同一条语句里完成**。把判断条件写进 `WHERE`，用影响行数做决策，是唯一不依赖应用层时序的做法。
- 扣减时机按业务选：**下单预占 + 支付确认**是大多数商城的推荐方案，它把失败留在可逆的位置（未支付可释放），而不是收钱之后。
- 模块边界要硬：只有库存模块能写库存台账，订单模块只持有预占凭证。这条边界决定了将来能不能对账。
- 状态量和流水要分开：台账管当前值，流水管变化原因。幂等最终靠数据库唯一键兜底，不靠应用层判断。
- 释放路径和占用路径同等重要。未支付的、取消的、退款的、部分的，都要有明确的负责人和触发条件。
- 顺序是先在纸上定清边界，再写代码。边界不清时补的每一条补偿逻辑，都会变成下一轮不一致的来源。

---

本文整理自随商商城项目和企业电商研发实践。
