# 多商户商城的订单拆单设计

> 本文为 **Recommended Design**，整理自多商户商城项目的设计取舍，描述的是推荐方案，不代表某个具体已上线产品的既有实现。

## Problem

多商户平台里，一个购物车里往往装着来自不同商户的商品。用户点一次「提交订单」，平台却需要面向 N 个商户分别履约、分别结算、分别开票。

如果订单模型不考虑这件事，最常见的做法是「先存一张订单，出问题再说」。等真正接入商户结算、拆包发货和售后退款时，才发现这张订单既算不清钱，也拆不动货。

## Business Scenario

一个典型的场景：

1. 用户在平台下单：A 商户的 2 件商品 + B 商户的 1 件商品 + 平台自营的 1 件商品；
2. 用户只付了一次钱，一个支付单；
3. 但需要产生三份发货任务，分属三个主体；
4. 每份发货任务要单独计算商户应收、平台佣金、优惠分摊；
5. 用户可能只退 B 商户那一件；
6. 商户结算周期到了，平台要按已经确认收货的部分给商户打款。

这四个动作（支付、履约、结算、退款）各自的口径都不一样，却都挂在同一笔用户支付上。

## Why It Happens

问题通常不是「忘了拆单」，而是在错误的时机拆：

- **在支付前拆**：用户看到三笔待支付，体验割裂，且任一笔支付失败都会让整单处于半死不活的状态。
- **在支付后立刻拆**：此时还没有履约信息，拆出来的子单只是把钱按商品金额切开，一旦发生部分退款、运费调整、优惠重算，子单金额就对不上了。
- **不拆，靠结算时算**：结算逻辑会变成对着一张大订单反复做分组聚合，退款一次就要重算一次，越改越不敢动。

根本原因在于：**用户视角的「一次交易」和商户视角的「一次履约」不是同一个聚合根。** 用一张表同时满足两个视角，必然要不断打补丁。

## Design

核心是把「用户订单」和「商户履约单」分成两层，并明确各自的职责边界。

### 分层

| 层 | 聚合根 | 职责 | 金额口径 |
| --- | --- | --- | --- |
| 交易层 | 主订单 `order_main` | 对用户负责：下单、支付、整单取消、整单退款入口 | 用户实付金额 |
| 支付层 | 支付单 `order_payment` | 对支付渠道负责：一次支付、一次回调 | 渠道实收金额 |
| 履约层 | 子订单 `order_sub` | 对商户负责：发货、收货、售后、结算 | 商户应收 + 平台佣金 + 优惠分摊 |

### 拆分时机

推荐在**支付成功回调之后**拆，而不是下单时：

- 下单时只创建主订单 + 支付单，保持「一次交易一个支付单」；
- 支付回调成功后，在同一事务（或可靠消息）内按商户维度生成子订单；
- 拆分是幂等的，以 `order_main.id` + `shop_id` 为幂等键。

这样做的代价是：支付成功到子订单生成之间有一个短暂窗口，前端需要能处理「已支付但子单还在生成」的状态。好处是拆分逻辑可以独立重试，不会因为拆单失败而丢钱。

### 金额归属

拆分时必须把三类金额明确落到子订单上：

```
子订单商户应收 = Σ(商品成交价 × 数量) + 该商户分摊运费 - 该商户分摊优惠
平台佣金       = 子订单商户应收 × 佣金比例（按商户/类目配置）
商户实收       = 子订单商户应收 - 平台佣金 - 其他扣款
```

优惠分摊建议**按商品成交金额比例分摊到分**，余数用最大余额法补给金额最大的那个子订单，保证 Σ子订单优惠 = 主订单优惠。否则长期累积会出现对账差 1 分钱的问题，而且很难查。

## Data Model / Flow

```sql
-- 主订单：对用户负责
CREATE TABLE order_main (
  id            BIGINT       NOT NULL COMMENT '主订单号',
  user_id       BIGINT       NOT NULL,
  pay_amount    DECIMAL(12,2) NOT NULL COMMENT '用户实付',
  status        TINYINT      NOT NULL COMMENT '10待支付 20已支付 30已完成 40已取消',
  create_time   DATETIME     NOT NULL,
  PRIMARY KEY (id),
  KEY idx_user (user_id, create_time)
);

-- 支付单：一次支付一个
CREATE TABLE order_payment (
  id            BIGINT       NOT NULL,
  order_id      BIGINT       NOT NULL,
  channel       VARCHAR(16)  NOT NULL COMMENT 'wechat/alipay',
  out_trade_no  VARCHAR(64)  NOT NULL COMMENT '渠道交易号，唯一',
  amount        DECIMAL(12,2) NOT NULL,
  status        TINYINT      NOT NULL COMMENT '10待支付 20成功 30失败',
  notify_time   DATETIME     NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_channel_trade (channel, out_trade_no),
  KEY idx_order (order_id)
);

-- 子订单：一个商户一条
CREATE TABLE order_sub (
  id            BIGINT       NOT NULL,
  order_id      BIGINT       NOT NULL COMMENT '关联主订单',
  shop_id       BIGINT       NOT NULL,
  goods_amount  DECIMAL(12,2) NOT NULL COMMENT '商品成交金额',
  freight       DECIMAL(12,2) NOT NULL DEFAULT 0,
  discount      DECIMAL(12,2) NOT NULL DEFAULT 0 COMMENT '分摊到的优惠',
  shop_amount   DECIMAL(12,2) NOT NULL COMMENT '商户应收',
  commission    DECIMAL(12,2) NOT NULL DEFAULT 0,
  status        TINYINT      NOT NULL COMMENT '10待发货 20已发货 30已收货 40已退款',
  PRIMARY KEY (id),
  UNIQUE KEY uk_order_shop (order_id, shop_id),
  KEY idx_shop_status (shop_id, status)
);
```

`uk_order_shop (order_id, shop_id)` 是拆单幂等的关键：回调重复触发时，重复插入会被唯一键挡住，而不是生成第二份子订单。

流程：

```
用户提交订单
  └─ 创建 order_main(待支付) + order_payment(待支付)
       │
       └─ 调渠道下单，返回支付参数
            │
        用户支付
            │
   渠道异步回调 ── 验签 ── 幂等校验(out_trade_no)
            │
      更新 order_payment = 成功
      更新 order_main = 已支付
            │
      按 shop_id 分组拆单（幂等键 order_id + shop_id）
            │
      写入 N 条 order_sub，同时写拆单事件
            │
      通知各商户发货 / 进入结算池
```

## Example

支付成功后的拆单逻辑（Go，只表达关键点）：

```go
// SplitOrder 支付成功后按商户拆单，幂等。
func SplitOrder(ctx context.Context, orderID int64) error {
    main, err := repo.GetMainOrder(ctx, orderID)
    if err != nil {
        return err
    }
    // 幂等：已经拆过就直接返回
    if ok, _ := repo.HasSubOrders(ctx, orderID); ok {
        return nil
    }

    items, err := repo.ListItems(ctx, orderID)
    if err != nil {
        return err
    }

    groups := groupByShop(items)              // map[shopID][]Item
    shares := allocateDiscount(main.Discount, items) // 按成交金额比例分摊，最大余额法处理余数

    subs := make([]*model.OrderSub, 0, len(groups))
    for shopID, list := range groups {
        goodsAmount := sumGoods(list)
        freight := allocateFreight(main.Freight, items, shopID)
        discount := shares[shopID]
        shopAmount := goodsAmount + freight - discount
        commission := calcCommission(ctx, shopID, list, shopAmount)

        subs = append(subs, &model.OrderSub{
            OrderID:     orderID,
            ShopID:      shopID,
            GoodsAmount: goodsAmount,
            Freight:     freight,
            Discount:    discount,
            ShopAmount:  shopAmount,
            Commission:  commission,
            Status:      model.SubStatusPendingShip,
        })
    }

    // 唯一键 uk_order_shop 兜底并发重复回调
    return repo.BatchInsertSubOrders(ctx, subs)
}
```

分摊余数处理的思路（伪代码）：

```
total := Σ 子单分摊值
diff  := 主订单应分摊总额 - total
if diff != 0 {
    把 diff 加到金额最大的那个子单上
}
```

## Edge Cases

| 情况 | 处理方式 |
| --- | --- |
| 支付回调重复推送 | `out_trade_no` 幂等 + `uk_order_shop` 唯一键，双保险 |
| 拆单过程中进程崩溃 | 拆单可重入；定时任务扫描「已支付但无子订单」的主订单补拆 |
| 全平台自营、只有一个商户 | 仍然生成子订单，层数不变，避免出现两种代码路径 |
| 用户只退一个商户的商品 | 只影响对应 `order_sub`；主订单状态按子单状态聚合推导，不单独维护 |
| 部分子单已结算后发生退款 | 走逆向结算（下期扣回），不直接改已结算数据 |
| 优惠券是平台券还是商户券 | 平台券按比例分摊到各子单；商户券只落到该商户子单，不参与跨商户分摊 |
| 子单金额出现负数 | 兜底校验：`shop_amount < 0` 直接拒绝拆单并报警，说明分摊逻辑有问题 |

## 拆单与结算的衔接

子订单是结算的输入，所以拆单时就要想清楚"什么状态下这笔钱可以给商户"。

常见做法是分两步，而不是一步到位：

```
order_sub.status = 30（已收货）
  └─ 生成一条 settlement_item（待结算）
       └─ 过了售后期（如 7 天无退款）
            └─ 进入结算池，按账期生成结算单
                 └─ 打款后回写 settlement_item.status = 已结算
```

这样处理的好处：

- 收货和可结算解耦，售后期长度可以按类目调整，不用改订单状态；
- 退款只需要把对应的 `settlement_item` 作废或标记待扣回，不影响已生成的结算单；
- 对账时能明确回答"这笔钱现在在哪一步"，而不是只有一个模糊的"未结算"。

对应表结构（同样属于 **Recommended Design**）：

```sql
CREATE TABLE settlement_item (
  id           BIGINT       NOT NULL,
  sub_order_id BIGINT       NOT NULL COMMENT '关联子订单',
  shop_id      BIGINT       NOT NULL,
  amount       DECIMAL(12,2) NOT NULL COMMENT '商户应收',
  commission   DECIMAL(12,2) NOT NULL,
  settle_amount DECIMAL(12,2) NOT NULL COMMENT '实际结算给商户',
  status       TINYINT      NOT NULL COMMENT '10待结算 20已入池 30已结算 40已作废',
  settle_time  DATETIME     NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_sub_order (sub_order_id),
  KEY idx_shop_status (shop_id, status)
);
```

`uk_sub_order` 保证一个子订单只能产生一条结算项，避免重复入池。

## Practical Notes

1. **拆单幂等键要在建表时就定好。** 事后加唯一键，历史上已经产生的重复子订单要先清洗，成本高得多。
2. **运费分摊别忽略。** 运费按商户分包计算还是按商品金额比例分摊，会直接影响商户到账金额，两种做法商户都能接受，但必须提前说清楚。
3. **佣金比例要有版本。** 商户改佣金规则时，历史订单必须按当时的比例结算，所以子订单要落一份比例快照，而不是每次结算时去查当前配置。
4. **主订单状态由子订单推导，不要手工改。** 一旦允许手工改主订单状态，子单和主单就会长期不一致，售后和结算都会出问题。
5. **退款入口在子订单，整单退款只是批量操作。** 反过来设计会很难支持部分退款。
6. 前端需要能展示「一单多包裹、多物流」；如果前端模型只支持一个物流单号，后端做得再对，用户也看不懂。

## Summary

多商户订单的关键不是「怎么把订单拆成几份」，而是先接受一个事实：用户的一次交易，和商户的一次履约，本来就该是两套模型。

主订单承载交易，支付单承载资金，子订单承载履约与结算。拆分放在支付成功之后，用 `order_id + shop_id` 保证幂等，用比例分摊保证金额守恒，用比例快照保证历史可复算。

把这几件事在建模阶段定下来，后面的退款、对账、结算才不会一路补丁。

---

本文整理自随商商城项目和企业电商研发实践。
