# 优惠分摊与退款尾差：订单金额为什么对不平

## Problem

金额对不平，几乎不会以"算错了"的形式出现。它通常以这几种现象冒出来：

- 订单明细的实付金额逐行加起来，比订单实付少 `0.01`；
- 同一订单分三次部分退款，累计退出去的钱比客户实付多 `0.01` ~ `0.02`；
- 多商户拆单后，各子单金额之和与原订单差几分，分账时缺一个尾差没有归属；
- 财务对账时渠道流水与系统订单差 `0.01`，退款侧长期挂一笔调不平的差额；
- 商品行退了一部分，系统算出的"可退金额"和客户自己算的不一样，客服说不清。

这类问题的共同点是：**差异极小、复现困难、责任难判定**。它不会让服务挂掉，但会让财务每个月手工调账，让客服每次退款都要人工确认。

定位成本高的原因往往不在算法，而在数据：日志里只打了"订单金额 268.40、退款金额 88.00"，没有打这一笔优惠是怎么分摊到每一行的。等到出问题再想回溯，只能靠猜。

## Business Scenario

一个足够普通的订单就能触发：

订单 `SO20261020001`，三行商品，一张店铺满减券，一笔运费：

| 商品 | 单价（元） | 数量 | 行金额（元） |
| --- | --- | --- | --- |
| A | 33.33 | 3 | 99.99 |
| B | 59.90 | 1 | 59.90 |
| C | 19.90 | 2 | 39.80 |

- 商品合计：199.69
- 店铺满减：满 199 减 15 → `-15.00`
- 平台优惠券：`-10.00`
- 运费：`+12.00`
- 客户实付：186.69

接下来会发生四件事，每一件都需要"把 25 元的优惠拆到行上"：

1. **下单**：要算出每行的实付，因为后续按行退款、按行结算佣金。
2. **部分退款**：客户退掉商品 B（59.90），退款金额必须落在"B 分摊后的实付"上，而不是"B 的原价"。
3. **拆单**：三行商品分属两个商户，两个子单金额之和必须等于 186.69，运费也得有人承担。
4. **结算**：平台按行实付抽佣，佣金之和要能对上订单实付乘以费率。

四个环节里任何一处用自己的方式"四舍五入到分"，结果就可能各不相同。25 元优惠分到 3 行，`25 / 3 = 8.3333...`，逐行取两位小数就是 `8.33 × 3 = 24.99`，凭空少 1 分。这 1 分如果不显式归位，它会一直存在于系统的某个角落。

## Why It Happens

### 1. 用浮点存金额

`float64`（IEEE 754 double）的尾数是 53 位，能精确表示的整数上限约 `9e15`。所以"以分为单位的整数用 float64 存不会丢精度"这个说法本身是对的——但现实中金额是以**元**为单位、带两位小数的浮点数存进去的：

```
0.1 + 0.2 = 0.30000000000000004
```

`0.1`、`0.2`、`33.33` 在二进制里都是无限循环小数，存进 float 的那一刻就已经是近似值。后果不是"存错了"，而是：

- 累加会漂移：100 行 `33.33` 累加，结果未必等于 `3333.00`；
- 相等判断会失效：`a == b` 对两个数学上相等的金额可能返回 false；
- 序列化与回读不一致：写入 `19.9`，某条链路上被格式化成 `19.90`，再解析回来参与计算，末位已经不同；
- 取整方向被误差左右：`round(0.145 * 100)` 到底落在 14 还是 15，取决于那个数实际存成了 `14.499999999999998` 还是 `14.500000000000002`。

公开仓库 `shsuishang/golershop`（LGPL-3.0，只读参考）里，`order_item` 的金额字段目前是 `float64`，例如 `order_item_payment_amount`、`order_item_discount_amount`、`order_item_return_subtotal`。这是很多存量商城系统的现状，不是个例——历史包袱无法一夜改写，但**新字段、新表、新服务不应该继续沿用**。

### 2. 逐行取整，不做尾差归集

最直觉的写法是：

```text
每行优惠 = round(优惠总额 × 该行金额 / 商品合计)
```

三行等额时，每行 `round(2500 / 3) = 833` 分，合计 `2499`，比 `2500` 少 1 分。这个 1 分不会报错，只会让"明细之和 ≠ 总额"，然后在某个下游环节（结算、对账、拆单）暴露出来。

不是"舍入有问题"，而是**分摊算法缺了最后一步**：任何按比例拆分整数的算法，都必须显式处理余数，并保证 `sum(结果) == 总额` 作为后置条件。

### 3. 舍入规则不统一

同一个系统里，不同语言、不同库的默认舍入不一样：

| 位置 | 常见默认行为 |
| --- | --- |
| Java `Math.round(2.5)` | `3`（等价于 `floor(x + 0.5)`） |
| Java `BigDecimal.setScale(0)` 不传模式 | 抛 `ArithmeticException` |
| Java `new BigDecimal(0.1)` | 带入二进制误差，得到 `0.1000000000000000055511151231257827...` |
| Java `BigDecimal.valueOf(0.1)` | 走 `Double.toString`，得到 `0.1` |
| Go `math.Round(2.5)` | `3`（远离零） |
| MySQL `ROUND(2.5)` | `3`（对精确值为远离零） |
| Python `round(2.5)` | `2`（银行家舍入，四舍六入五成双） |

下单用一个规则，退款用另一个规则，两边的"几分"就会朝相反方向走。退款额比可退额多 1 分，是超退；少 1 分，是客户投诉。两种都要人工处理。

更隐蔽的一种：`int(x * 100)` 做"元转分"。对 `1.15` 这种值，`1.15 * 100` 可能得到 `114.99999999999999`，截断后是 `114` 分。少 1 分，且只在部分金额上出现。

### 4. 汇总字段与明细分配字段是两套口径

这在"优惠"上最容易出事。同一个 `order_item` 上，优惠既可能有一个汇总字段（这一行总共减了多少），又可能有按活动类型拆开的明细字段（券分了多少、满减分了多少）。

`shsuishang/golershop` 的 `order_item` 实体就是这种结构：有 `order_item_discount_amount`（优惠金额，注释写明"只考虑单品的，订单及店铺总活动优惠不影响"），也有 `order_item_voucher`（分配优惠券额度）、`order_item_reduce`（分配满减额度）两个明细字段；另有一个非持久化字段 `order_item_can_refund_amount`，注释给出的口径是：

```text
order_item_can_refund_amount = order_item_payment_amount
                             - order_item_voucher
                             - order_item_reduce
```

这里不做对错的判定，只说明风险：**当汇总字段已经包含了明细字段时，任何"再减一次明细"的公式都会把同一笔优惠扣两遍**，可退金额就会偏小；反过来，如果汇总字段只含单品优惠、订单级优惠另存，那不减明细就会超退。同一套字段，两种理解，结果差一个优惠全额。

结论很硬：金额字段的口径不能靠字段名推断，必须以**写入那段代码**为准，并且在文档里写清楚"这个字段含什么、不含什么"。跨字段再计算之前，先确认包含关系。

### 5. 退款按"本次重算"而不是"累计剩余"

第二种常见写法：

```text
本次可退 = 该行实付 × 本次退货数量 / 该行购买数量
```

单次看没问题。但这个客户先退 1 件、再退 1 件、最后退 1 件，假设该行优惠后实付是 100.00（原价 99.99，被加价或运费调整过），每件 `round(100.00 / 3) = 33.33`，三次累计退 99.99，永远剩 1 分退不掉；反过来，如果该行实付是 99.98，`round(33.326) = 33.33`，三次累计 99.99，就多退了 1 分。

正确做法是**只记剩余可退，每次扣减**：

```text
本次可退 = 当前剩余可退（从库里读）  -  退款后剩余可退
退款后剩余可退 = 该行实付 - 累计已退 - 本次退款
```

也就是把"可退上限"当成一个有状态的余额，而不是每次用比例重新推导。这样才能保证：无论退几次、按什么顺序退，累计退款永远不超过该行实付。

### 6. 拆单时运费和订单级优惠没有分摊规则

拆单的输入是"母单金额"，输出是"N 个子单金额"。如果拆分只按商品行金额做权重，运费和订单级优惠（满减、平台券）就没有落点：

- 运费留在母单 → 母单与子单之和差一笔运费；
- 运费按金额比例摊到子单 → 每个商户承担一部分运费，结算口径要跟着改；
- 订单级优惠不摊 → 子单金额之和大于母单实付，分账总额超过收款总额，账上凭空多钱。

这三个选择都能自洽，但必须**在拆单时一次定死并落库**，不能在下游各算一遍。

## Design

**Recommended Design**：以下是一套可以落地的口径，不是某个既有产品的实现说明。

### 原则 1：内部一律用最小货币单位整数

- 数据库：`BIGINT`（分）或 `DECIMAL(18,2)`，禁止 `FLOAT` / `DOUBLE`；
- 内存：`int64` / `long` / `BigDecimal`（必须指定 scale 与 rounding mode）；
- 传输：JSON 里用整数分（字段名带 `_cents`）或字符串 `"186.69"`，禁止裸浮点；
- 展示：只在最外层渲染时转成元，且格式化函数只有一个。

`DECIMAL` 与整数分二选一即可，不要混用。混用的项目里，`DECIMAL` 列被读成 `double` 再写回去，是最常见的静默精度损失。

### 原则 2：总额 → 明细只用一种算法，且带后置断言

推荐最大余额法（largest remainder method）：

1. 计算目标权重（行金额、行数量或自定义权重）；
2. `基数 = total × w_i / sum(w)` 向下取整（整数除法）；
3. 余数 `total - Σ基数` 按 `(total × w_i) mod sum(w)` 从大到小补 1 分；
4. 余数相同时按下标顺序补，保证同样输入永远得到同样输出；
5. 断言 `Σ结果 == total`，不成立直接 panic/抛异常——这种错误不该被吞掉。

如果业务上要求"尾差固定给某一行"（比如给金额最大的行、或给最后一行），也可以，但必须写进文档并保持稳定。规则可以简单，不能随机。

### 原则 3：分摊结果落库，下游只读不算

分摊是一次性决策，应该落成数据：

- 每一行记录：`discount_alloc_cents`、`coupon_alloc_cents`、`freight_alloc_cents`、`payable_cents`；
- 订单级记录优惠的来源与总额，便于反查；
- 退款、结算、佣金、拆单**全部从这些落库字段派生**，不再重新按比例算一遍。

一旦下游重新计算比例，就等于存在第二套口径；两套口径迟早会分叉。

### 原则 4：可退金额是余额，不是公式

`line.refundable_cents` 表示"这一行现在还最多能退多少"，随每次退款递减，并配合 `refunded_cents`（已退）与 `refunding_cents`（退款中，占位防并发）：

```text
refundable = payable - refunded - refunding
```

调用渠道退款前先用条件更新占位（`refunding` 增加），渠道返回成功或失败后再结算（`refunding` 减少、`refunded` 增加或不变）。这样并发退款、重复提交、渠道超时重试都不会超退。

### 原则 5：金额只在一处产生，其余都是搬运

下单时算一次金额，之后所有环节（支付、退款、结算、对账）都只搬运这份结果。任何"为了方便，这里再算一下"的写法，都是未来的差异来源。

## Data Model / Flow

**Example**：一套满足上述原则的最小表结构（字段名为示例命名，不代表任何既有系统）。

```sql
CREATE TABLE order_main (
  order_id            VARCHAR(32)  NOT NULL PRIMARY KEY,
  goods_amount_cents  BIGINT       NOT NULL,           -- 商品合计（分摊前）
  discount_cents      BIGINT       NOT NULL DEFAULT 0, -- 订单级优惠合计（满减+平台券）
  freight_cents       BIGINT       NOT NULL DEFAULT 0,
  payable_cents       BIGINT       NOT NULL,           -- 应付 = 商品 - 优惠 + 运费
  paid_cents          BIGINT       NOT NULL DEFAULT 0,
  version             INT          NOT NULL DEFAULT 0
);

CREATE TABLE order_line (
  order_line_id        BIGINT      NOT NULL PRIMARY KEY,
  order_id             VARCHAR(32) NOT NULL,
  sku_id               BIGINT      NOT NULL,
  unit_price_cents     BIGINT      NOT NULL,
  quantity             INT         NOT NULL,
  line_amount_cents    BIGINT      NOT NULL,           -- unit_price × quantity
  discount_alloc_cents BIGINT      NOT NULL DEFAULT 0, -- 订单级优惠分摊到这里
  coupon_alloc_cents   BIGINT      NOT NULL DEFAULT 0, -- 券分摊到这里（属于 discount 的明细）
  freight_alloc_cents  BIGINT      NOT NULL DEFAULT 0,
  payable_cents        BIGINT      NOT NULL,           -- 分摊后实付
  refunded_cents       BIGINT      NOT NULL DEFAULT 0,
  refunding_cents      BIGINT      NOT NULL DEFAULT 0,
  KEY idx_order (order_id)
);
```

约束写成断言，比写在注释里可靠：

```sql
-- 明细实付之和必须等于订单应付
SELECT o.order_id,
       o.payable_cents,
       SUM(l.payable_cents)                        AS line_sum,
       o.payable_cents - SUM(l.payable_cents)      AS diff
FROM order_main o
JOIN order_line l ON l.order_id = o.order_id
GROUP BY o.order_id, o.payable_cents
HAVING diff <> 0;

-- 优惠分摊之和必须等于订单级优惠
SELECT o.order_id,
       o.discount_cents,
       SUM(l.discount_alloc_cents) AS alloc_sum
FROM order_main o
JOIN order_line l ON l.order_id = o.order_id
GROUP BY o.order_id, o.discount_cents
HAVING o.discount_cents <> SUM(l.discount_alloc_cents);

-- 任何一行的可退余额不得为负
SELECT order_line_id, order_id, payable_cents, refunded_cents, refunding_cents
FROM order_line
WHERE payable_cents - refunded_cents - refunding_cents < 0;
```

如果手上是 `shsuishang/golershop` 这类字段命名的库（字段真实存在，`float64`），可以直接跑现有数据的偏差统计，先量化问题规模：

```sql
SELECT i.order_id,
       SUM(i.order_item_payment_amount)                        AS item_sum,
       SUM(i.order_item_return_subtotal)                       AS refund_sum,
       SUM(i.order_item_payment_amount)
         - SUM(i.order_item_return_subtotal)                   AS remain_sum
FROM order_item i
GROUP BY i.order_id
HAVING SUM(i.order_item_return_subtotal) > SUM(i.order_item_payment_amount);
```

金额流转的顺序（每一步只搬运上一步的结果）：

```text
下单
 ├─ 1. 计算商品合计 goods_amount          （每行 unit_price × quantity）
 ├─ 2. 计算订单级优惠 discount             （满减 / 券 / 活动，只在订单级算）
 ├─ 3. 按行金额权重分摊 discount 与 freight（最大余额法，尾差归位）
 ├─ 4. 写入每行 payable_cents，断言 Σ行 == 订单应付
 └─ 5. 写入 order_main.payable_cents
支付
 └─ 6. 校验支付金额 == payable_cents，落 paid_cents（回调幂等见 api/payment-callback/）
退款
 ├─ 7. 读该行 refundable = payable - refunded - refunding
 ├─ 8. 条件更新占位 refunding（乐观锁 version）
 ├─ 9. 调渠道退款 → 成功：refunded += 本次，refunding -= 本次
 │                  失败：refunding -= 本次
 └─ 10. 全单退完时校验 Σrefunded <= paid_cents
拆单
 ├─ 11. 按行分组，子单金额 = Σ该组行 payable_cents
 ├─ 12. 运费按规则整体归某个子单（或按行 freight_alloc 摊分），一次定死
 └─ 13. 断言 Σ子单 == 母单，尾差显式归到指定子单
```

## Example

**Example**：整数分分摊，带尾差归集与后置断言（Go）。

```go
// Allocate 把 total（分）按 weights 拆成等长结果，保证 sum(result) == total。
// 负数总额（退款冲正）先取绝对值再恢复符号，避免 Go 整数除法向零截断导致余数为负。
func Allocate(total int64, weights []int64) []int64 {
	out := make([]int64, len(weights))
	if len(weights) == 0 {
		return out
	}

	sign := int64(1)
	if total < 0 {
		sign, total = -1, -total
	}

	var sumW int64
	for _, w := range weights {
		if w < 0 {
			panic("Allocate: weight must be non-negative")
		}
		sumW += w
	}
	if sumW == 0 {
		out[0] = sign * total // 权重全零：全部落在第一行，避免除零
		return out
	}

	type remainder struct {
		idx  int
		frac int64
	}
	rems := make([]remainder, 0, len(weights))

	var assigned int64
	for i, w := range weights {
		base := total * w / sumW
		out[i] = base
		assigned += base
		rems = append(rems, remainder{idx: i, frac: total * w % sumW})
	}

	rest := total - assigned // 0 <= rest < len(weights)
	sort.SliceStable(rems, func(a, b int) bool {
		if rems[a].frac != rems[b].frac {
			return rems[a].frac > rems[b].frac // 余数大的先补
		}
		return rems[a].idx < rems[b].idx // 余数相同按下标，保证幂等
	})
	for i := int64(0); i < rest; i++ {
		out[rems[i].idx]++
	}

	for i := range out {
		out[i] *= sign
	}

	var check int64
	for _, v := range out {
		check += v
	}
	if check != sign*total {
		panic("Allocate: sum mismatch") // 不可恢复，直接暴露
	}
	return out
}
```

调用方（下单时把 2500 分优惠摊到三行，权重取行金额）：

```go
lineAmounts := []int64{9999, 5990, 3980} // 33.33×3, 59.90×1, 19.90×2，单位：分
alloc := Allocate(2500, lineAmounts)    // [1252 750 498]，合计 2500
```

同理，退款时不要重新算比例，只读余额：

```go
// 条件更新占位，返回受影响行数，0 表示余额不足或被并发抢走
func ReserveRefund(ctx context.Context, lineID int64, amount int64) (bool, error) {
	res, err := db.Exec(ctx,
		`UPDATE order_line
		    SET refunding_cents = refunding_cents + ?, version = version + 1
		  WHERE order_line_id = ?
		    AND payable_cents - refunded_cents - refunding_cents >= ?`,
		amount, lineID, amount)
	if err != nil {
		return false, err
	}
	n, err := res.RowsAffected()
	return n == 1, err
}
```

**Example**：Java 侧的三种写法差异（同样的 15 元优惠摊到 3 行）。

```java
// 不推荐：double 参与金额计算
double payable = 199.69 - 15.00;
long cents = (long) (payable * 100);          // 可能得到 18468 而不是 18469

// 不推荐：BigDecimal 用 double 构造
new BigDecimal(0.1);                          // 0.1000000000000000055511151231257827...
new BigDecimal(0.1).setScale(2, RoundingMode.HALF_UP); // 0.10，侥幸对，但不可依赖

// 推荐：整数分
long totalCents = 1500L;
long[] weights = {9999L, 5990L, 3980L};
long[] alloc = AmountAllocator.allocate(totalCents, weights); // Σ == 1500

// 推荐：必须用 BigDecimal 时，字符串构造 + 显式舍入（单位统一为分）
BigDecimal totalCents = new BigDecimal("1500");                 // 15.00 元
BigDecimal share = totalCents.multiply(new BigDecimal("9999"))
                             .divide(new BigDecimal("19969"), 0, RoundingMode.DOWN); // 1251
```

## Edge Cases

- **单行订单**：分摊退化为"全给它"，尾差逻辑仍要跑，避免出现单行专用分支（两套代码就意味着两种结果）。
- **零元行（赠品）**：权重为 0，按上面的算法分不到优惠，也可能分不到运费。要明确赠品行是否承担运费，以及退货时是否退运费。
- **优惠大于单行金额**：某行 5 元、优惠摊到 6 元，会出现负实付。要么在分摊前限制（优惠不超过商品合计），要么允许负值但在支付时归零并单独记账——不能默默截断到 0，因为截断掉的金额会让总额对不平。
- **运费参与分摊 vs 不参与**：运费不参与优惠分摊是常见口径（优惠只作用于商品），但运费要参与拆单。这两件事必须分别定义，不能合并成一句"按比例摊"。
- **全额退款后剩 1 分**：`payable=100.00`，两次部分退款分别 49.99、49.99，剩 1 分。余额式可退金额能自然兜住（最后一次可退 = 1 分），比例式会算成 50.00 而超退。
- **多商户拆单 + 平台券**：平台承担的券不该从商户结算额里扣，商户券才扣。分摊时要把"谁出这笔钱"作为维度之一，否则结算金额会错。
- **退款中的二次申请**：客户在退款审核中又发起一笔，`refunding` 占位必须挡住，否则两笔同时占用同一份余额。
- **跨币种/多分站**：不同币种最小单位不同（有的货币没有小数位）。整数分的前提是"最小单位"，不是"分"，字段名与注释要写清楚。
- **存量 float 数据迁移**：加整数列 → 双写 → 离线比对（元值四舍五入到分是否一致）→ 灰度读新列 → 停写旧列。不要一次性 `ALTER` 改类型后直接切换，历史行的漂移会在切换当天集中暴露。

## 排查步骤

线上出现金额差异时，按这个顺序查，比直接读代码快：

1. **确认差异量级**：跑一次"明细之和 vs 主单"的 SQL。差 1 分的集中在特定活动/特定商户，说明是分摊口径问题；差几元的，先怀疑重复退款或状态机问题。
2. **拉这一单的完整金额链**：下单时的 `goods_amount / discount / freight / payable`，每行的 `alloc / payable`，支付金额，退款记录。缺哪一段就先补日志。
3. **对齐舍入规则**：把下单、退款、结算三处的取整代码找出来，看是否是同一个函数。多数 1 分差异在这里就结束了。
4. **检查字段包含关系**：确认"汇总优惠字段"是否已含"明细优惠字段"。这一步能解释"可退金额整体偏小/偏大一笔优惠"的批量问题。
5. **重放**：用该订单的输入（数量、单价、活动、券）在测试环境重跑分摊，看是否能复现。不能复现的，通常是活动配置在事后被改过——所以优惠配置必须版本化。
6. **定责与挂账**：1 分以内的差异建议自动挂"尾差"科目，不做人工追；超过阈值的告警。追每一分钱的成本远高于那一分钱。

## Practical Notes

- **日志要打分摊明细**，不只是总额：`order_id`、每行 `alloc`、`rest`（尾差几分）、补到了哪一行。没有这段日志，事后只能靠猜。
- **单测写成对拍**：随机生成 1 万组（总额、权重）跑 `Allocate`，断言 `Σ == total` 且同输入同输出。这比手写十几个用例有效。
- **禁止在 SQL 里做金额分摊**：`ROUND` 的行为随版本与类型变化，且无法加断言。分摊放在应用层，SQL 只做校验。
- **禁止前端算钱**：前端只展示服务端给的金额；任何"前端算好了传给后端"的字段都必须被后端重新校验或忽略。
- **手工改价必须重算分摊**：后台改单、客服补差、运费改价，都会让已落库的分摊失效。改价接口要做成"改输入 → 重新跑分摊 → 覆盖落库"，而不是"直接改 payable"。
- **优惠配置版本化**：活动金额改动后，历史订单要能还原当时的规则。至少保留 `activity_snapshot`，否则复现永远做不出来。
- **对账差额分级**：`0` 正常；`|diff| <= 1 分` 自动挂账并计数；`> 1 分` 告警到人。指标化之后，"这个月尾差挂了多少笔"本身就是系统健康度。
- **新老系统并存期**：老系统 float、新系统整数分，中间层转换只允许在边界发生一次（读老库转分、写新库），不要在中间层来回转。

## Summary

金额对不平，原因通常不是某个公式写错，而是三件事叠加：用浮点存金额、分摊不做尾差归集、同一笔优惠存在多套口径。

能撑住的做法不复杂：

1. 内部只用最小货币单位整数，元只在展示层出现；
2. 任何"总额拆明细"都用最大余额法，尾差显式归位，并断言合计相等；
3. 分摊结果落库，退款、结算、拆单、佣金全部从落库值派生，不重新计算比例；
4. 可退金额是余额（`payable - refunded - refunding`），不是每次用比例重算；
5. 字段口径以写入代码为准，跨字段再计算前先确认包含关系。

代价是前期要多写一张表、多几个断言、多一层分摊代码。收益是财务不用每月手工调账，客服不用每次退款都人工确认，拆单和分账的总额永远对得上。

---

本文整理自随商商城项目和企业电商研发实践。示例代码与表结构为设计示例，不代表任何既有产品的实现；文中对公开仓库字段的引用仅用于说明口径问题。
