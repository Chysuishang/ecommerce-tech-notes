# 商品规格与 SKU 落库：组合怎么存、怎么查、改规格为什么会删掉库存

## Problem

同一个商品，后台编辑一次规格，线上库存清零、活动价失效、订单里的 SKU 变成「已删除商品」——这类问题在商城项目里反复出现，原因几乎都指向同一处：**规格组合没有独立的主键，而是靠字段拼出来的**。

规格的难点不在「商品有多个规格」这件事本身，而在下面四个具体问题上：

1. 用户选中的一组规格值（红 / L / 棉），怎么唯一对应到一条 SKU 记录；
2. 规格值改名、排序、增删之后，这个对应关系还成不成立；
3. 后台保存商品时，规格变更应该落到「更新已有 SKU」还是「删除重建」；
4. 商品详情页要一次返回全部规格、价格、库存、图片，这些数据放在一张表里还是拆开放。

第 3 点决定数据会不会丢，第 1 点决定会不会卖错货，两者都跟表结构直接相关，等到业务量上来再改代价很高。

## Business Scenario

一个典型的服装类目商品：

- 规格一「颜色」：黑色、白色、藏青
- 规格二「尺码」：S、M、L、XL
- 规格三「材质」：纯棉、混纺

后台需要给每个组合单独定价、单独设库存、单独挂图。运营的实际动作是：先建 3×4×2 = 24 个组合，然后只对其中 6 个真正会卖的组合配图配价格，剩下的先留空。

同时还有另外两类商品要兼容：

- 单规格商品（只有颜色，或者干脆没有规格）——多数商城里这类占大头；
- 带辅助属性的商品（品牌、产地、适用季节），这些属性只用于筛选，**不参与组合**，也不应该生成 SKU。

一套表结构要同时撑住这三种形态，还不能让「没有规格的商品」退化成一个特例分支。

## Why It Happens

### 1. 用「拼接字符串」当组合主键

最常见的做法是给 SKU 表加一个 `spec` 字段，存 `"黑色,L,纯棉"` 这样的字符串，查询时按字符串匹配。它的问题不在性能，而在**顺序和可读名都不稳定**：

- 前端按用户点击顺序提交，后台按规格排序提交，同一个组合会得到两个不同的键；
- 规格值改名（「藏青」改成「藏青色」）之后，历史键全部对不上，等于所有 SKU 的规格信息失效；
- 名称里出现逗号、空格、斜杠（「S/M」「L(加长)」）时，拼接结果不再可逆。

### 2. 商品表和 SKU 表职责混在一起

把价格、库存、规格全部塞进商品主表，再额外开一个 `sku_json` 字段存全部 SKU，写的时候很省事，读的时候只有一种用法是安全的：整段读出来解析。一旦要做「按价格区间筛选」「按库存过滤」「只查有货的 SKU」，就只能全表读出来在内存里过滤——商品上万之后这条路走不通。

### 3. 编辑保存时「先删后建」

后台保存商品的实现通常是：删掉该商品下所有 SKU，再把提交上来的 SKU 全部插入。这对新商品没问题，对老商品是灾难：

- `item_id` 全部变化，历史订单、活动配置、购物车、收藏、库存流水里的 SKU 引用全部悬空；
- 库存被重置成提交值，如果提交的是「不在本次编辑范围内的字段」，库存直接归零；
- 每次编辑都产生一批无意义的删除记录，做数据核对时完全无法追溯。

### 4. 缓存字段和实时字段没有分开

商品详情页为了少查表，会把价格、库存冗余到商品主表或一个 JSON 里。冗余本身没问题，问题是没人定义「这个字段什么时候刷新」。库存每次下单都在变，冗余值必然过期；如果下单校验读的是冗余值，就会超卖或者误判无货。

## Design

### 参考实现的表分层

公开项目 golershop（Go，LGPL-3.0）把商品数据拆成了下面几张表，这个分层方式值得单独拿出来看，因为它把上面四个问题分别放到了不同的表里：

| 表 | 职责 | 关键字段 |
| --- | --- | --- |
| `product_base` | 商品基础信息、主图、限购、运费模板 | `product_id`、`product_image`、`product_buy_limit` |
| `product_info` | 详情、规格定义快照、SKU 指纹索引 | `product_spec`、`product_uniqid`、`spec_ids` |
| `product_item` | SKU：价格、库存、编码、规格归属 | `item_id`、`item_unit_price`、`item_quantity`、`item_quantity_frozen`、`item_spec`、`spec_item_ids`、`color_id`、`item_enable` |
| `product_spec` / `product_spec_item` | 规格与规格值字典 | `spec_id`、`spec_name`、`spec_item_id`、`spec_item_name` |
| `product_image` | 按规格值挂图 | `product_id` + `color_id`、`item_image_default` |
| `product_index` | 检索冗余表 | `product_unit_price_min/max`、`product_assist_data`、`product_state_id` |

几个可以直接从代码里看出来的设计取舍：

**规格定义在商品里存了一份快照。** `product_info.product_spec` 不是规格 id 列表，而是一段 JSON，把规格名和它包含的规格值名一起存下来。这样全局规格字典改名不会影响已发布商品的展示，代价是同一个规格在不同商品里可能显示不同名字（后面 Edge Cases 会讲这个代价）。

**组合指纹是「规格值 id 排序后拼接」。** `product_info.product_uniqid` 是一个 map，键由该 SKU 的规格值 id 升序排序后用 `-` 连接而成，值是一个数组，按 `[item_id, 单价, 库存, 启用状态, color_id, 图片]` 的顺序存放。用 id 而不是名称，就绕开了改名失效和分隔符冲突两个问题；排序则让「先选颜色再选尺码」和「先选尺码再选颜色」落到同一个键上。

**SKU 自己也存一份规格归属。** `product_item.item_spec` 与 `spec_item_ids` 让单条 SKU 能独立回答「我是什么规格」，不必反查商品表的 JSON。

### Recommended Design：把指纹做成数据库约束

上面这套结构里，`product_uniqid` 只是一个 JSON 字段，重复组合、缺组合都只能靠应用层保证。要在自己的项目里做扎实，建议再加一条数据库层的约束：

```
UNIQUE KEY uk_product_spec (product_id, spec_key)
```

`spec_key` 用定长、可比较的形式存，例如规格值 id 升序后补零拼接（`0003-0011-0027`），而不是原始名称。有了唯一索引，重复提交同一组合会直接报冲突，写入逻辑可以用 upsert 处理，天然幂等。

### Recommended Design：只落存在的组合

24 个理论组合，实际只卖 6 个。两种落库方式：

| 方式 | 写入量 | 空组合处理 | 适用 |
| --- | --- | --- | --- |
| 全量笛卡尔积预生成 | 大 | 大量 `quantity=0` 的僵尸行，筛选、统计都要额外过滤 | 组合数很少（<50）且运营确实要逐条配置 |
| 只落运营提交的组合 | 小 | 不存在即不可售，前端按指纹查不到就置灰 | 组合数可能过百、或多规格混合的类目 |

第二种更常见。代价是前端必须能处理「某个组合查不到 SKU」的情况，也就是规格按钮要能单独置灰（后面 Flow 里给了做法）。

### Recommended Design：编辑走 diff，不重建

保存商品时，把流程改成三段：

1. 按 `product_id` 读出全部现存 SKU，建立 `item_id` 集合；
2. 提交的 SKU 里带 `item_id` 的走 update，不带的走 insert（新组合）；
3. 现存集合减掉提交集合的差集，**不删除**，改为置 `item_enable = 下架` 并保留行。

第 3 步是关键。物理删除只应该在「商品彻底删除」或「运营明确确认丢弃库存」时发生。保留行的成本是一条记录，收益是所有历史引用都还能解析出「这个 SKU 当时是什么规格」。

golershop 当前的保存逻辑是物理删除差集（`product_base.go` 中按 `product_id` 查出旧 SKU，把提交列表里不存在的 `item_id` 从集合里剔除后，剩余 id 直接 `Remove`）。这是「编辑商品会丢库存」的直接原因，也是二次开发时最该先改的一处。

### Recommended Design：缓存字段与实时字段分离

`product_uniqid` 里带了库存和单价，但它只在保存商品时写入——`product_index.go` 补全商品详情时是直接透传这个字段，不做刷新，库存变动（下单、取消、补货）都不会回写。所以它的定位只能是**结构索引**：告诉前端「这个组合对应哪个 item_id、大概多少钱、有没有配图」。

下单校验必须回到 `product_item` 表读实时值。这条边界不写清楚，前端很容易拿 `uniqid` 里的库存做「是否可买」判断。

## Data Model / Flow

### 表结构（Example）

下面是按上述设计整理的一份最小结构，字段命名与 golershop 保持相近便于对照，但去掉了该项目业务特有的字段：

```sql
-- 规格字典
CREATE TABLE product_spec (
  spec_id     BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  spec_name   VARCHAR(32)  NOT NULL,
  spec_format VARCHAR(16)  NOT NULL DEFAULT 'text',  -- text | image
  sort_no     INT          NOT NULL DEFAULT 0,
  PRIMARY KEY (spec_id)
);

CREATE TABLE product_spec_item (
  spec_item_id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  spec_id      BIGINT UNSIGNED NOT NULL,
  item_name    VARCHAR(32)  NOT NULL,
  sort_no      INT          NOT NULL DEFAULT 0,
  enabled      TINYINT      NOT NULL DEFAULT 1,
  PRIMARY KEY (spec_item_id),
  KEY idx_spec (spec_id, sort_no)
);

-- SKU：一行一个真实组合
CREATE TABLE product_item (
  item_id        BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  product_id     BIGINT UNSIGNED NOT NULL,
  spec_key       VARCHAR(128) NOT NULL,         -- 规格值 id 升序拼接，规范键
  item_spec      JSON         NOT NULL,         -- 冗余的规格明细，供详情页直接渲染
  item_number    VARCHAR(64)  NOT NULL DEFAULT '',
  unit_price     DECIMAL(12,2) NOT NULL DEFAULT 0.00,
  quantity       INT UNSIGNED NOT NULL DEFAULT 0,
  quantity_frozen INT UNSIGNED NOT NULL DEFAULT 0,
  is_default     TINYINT      NOT NULL DEFAULT 0,
  enabled        TINYINT      NOT NULL DEFAULT 1,
  PRIMARY KEY (item_id),
  UNIQUE KEY uk_product_spec (product_id, spec_key),
  KEY idx_product (product_id, enabled)
);

-- 商品：规格定义快照
CREATE TABLE product_info (
  product_id   BIGINT UNSIGNED NOT NULL,
  product_spec JSON NOT NULL,   -- [{id,name,item:[{id,name}]}]
  PRIMARY KEY (product_id)
);
```

`spec_key` 用定长补零拼接，是为了让字符串比较与数值比较等价，避免 `10` 排在 `9` 前面这类排序错乱：

```
规格值 id: 3, 11, 27  →  spec_key = "0003-0011-0027"
```

### 发布商品流程

```
运营提交 { 规格定义, SKU 列表 }
        │
        ├─ 1. 校验：每个 SKU 的规格值必须属于已定义的规格；同一规格只能选一个值
        ├─ 2. 计算 spec_key（规格值 id 升序 → 补零 → 拼接）
        ├─ 3. 查重：同一 product_id 下 spec_key 冲突 → 报错，不静默覆盖
        ├─ 4. 落库：有 item_id 走 update，无则 insert
        ├─ 5. 差集处理：现存但未提交的 SKU 置为下架（推荐）或删除（当前实现）
        └─ 6. 重建派生数据：min/max 价格、默认 SKU、检索冗余字段
```

第 6 步最容易被漏掉。golershop 在保存商品时会把 `product_index` 的 `product_unit_price_min/max`、`product_assist_data`（辅助属性值拼接，用于 `FIND_IN_SET` 筛选）、`product_state_id` 一起刷新，并把默认 SKU 的主图回写到商品主图字段。这些派生字段一旦漏刷，列表页的价格区间和筛选结果就会和详情页对不上。

### 下单流程

```
用户选规格 → 前端拼 spec_key（同样升序、同样补零）
          → 本地用 uniqid 找到 item_id、展示价
          → 提交 item_id 给服务端
服务端 → 按 item_id 回表读 unit_price / quantity / enabled（实时值）
      → 条件更新扣库存（quantity_frozen 或 quantity 条件扣减）
      → 订单明细写入规格快照（名称，不是 id）
```

订单明细必须同时存 **item_id** 和 **规格名称快照**。只存 item_id，SKU 一旦被下架改名，历史订单就显示不出当时买的是什么；只存名称，又没法做售后按 SKU 统计。

## Example

### 规格定义快照

`product_info.product_spec` 的实际形态（结构照抄自公开源码注释，值由本文构造）：

```json
[
  {
    "id": 1,
    "name": "颜色",
    "item": [
      { "id": 3,  "name": "黑色" },
      { "id": 4,  "name": "白色" },
      { "id": 5,  "name": "藏青" }
    ]
  },
  {
    "id": 2,
    "name": "尺码",
    "item": [
      { "id": 11, "name": "S" },
      { "id": 12, "name": "M" },
      { "id": 13, "name": "L" }
    ]
  }
]
```

注意这里存的是名称而不只是 id。规格值改名后，已发布商品仍显示订货时的名字，这是刻意的：改全局字典不应该批量改写历史商品。

### 规范键计算

```go
// Example
func SpecKey(specItemIDs []uint64) string {
    sorted := make([]uint64, len(specItemIDs))
    copy(sorted, specItemIDs)
    sort.Slice(sorted, func(i, j int) bool { return sorted[i] < sorted[j] })

    parts := make([]string, len(sorted))
    for i, id := range sorted {
        parts[i] = fmt.Sprintf("%04d", id)
    }
    return strings.Join(parts, "-")
}
```

要点只有两个：**排序**和**不依赖名称**。前端必须用同一套规则算键，否则「选了两个规格查不到 SKU」这类问题会以「偶尔复现」的形式出现在线上。

### 幂等写入

```sql
-- Example：同一 product_id + spec_key 重复提交不会产生第二行
INSERT INTO product_item
  (product_id, spec_key, item_spec, item_number, unit_price, quantity, quantity_frozen, is_default, enabled)
VALUES
  (1001, '0003-0011', '{"specs":[{"id":1,"name":"颜色","item":{"id":3,"name":"黑色"}},{"id":2,"name":"尺码","item":{"id":11,"name":"S"}}]}',
   'SKU-1001-0003-0011', 199.00, 100, 0, 0, 1)
ON DUPLICATE KEY UPDATE
  item_spec  = VALUES(item_spec),
  unit_price = VALUES(unit_price),
  enabled    = VALUES(enabled);
```

`quantity` 故意不放进 `ON DUPLICATE KEY UPDATE`。库存的写入通道只有下单扣减、取消回补、入库三类，商品编辑不应该成为第四个入口——把库存放进商品保存的更新列表，就是「编辑商品把库存覆盖掉」的典型成因。

### 查询可售 SKU

```sql
-- Example：详情页需要的全部可售组合
SELECT item_id, spec_key, unit_price, quantity - quantity_frozen AS available, is_default
FROM product_item
WHERE product_id = 1001
  AND enabled = 1
  AND quantity > quantity_frozen
ORDER BY is_default DESC, item_id ASC;
```

列表页的价格区间不要现算，用检索表的冗余字段：

```sql
-- Example
UPDATE product_index pi
JOIN (
  SELECT product_id, MIN(unit_price) AS pmin, MAX(unit_price) AS pmax
  FROM product_item
  WHERE enabled = 1 AND quantity > quantity_frozen
  GROUP BY product_id
) s ON s.product_id = pi.product_id
SET pi.product_unit_price_min = s.pmin,
    pi.product_unit_price_max = s.pmax
WHERE pi.product_id = 1001;
```

### 前端规格按钮置灰

组合稀疏时，规格按钮不能只按「这个规格值是否存在」来判断，要按「加上这个值之后还有没有可售 SKU」判断：

```js
// Example
function isSelectable(selected, candidate, skuKeys) {
  const next = { ...selected, [candidate.specId]: candidate.specItemId };
  const ids = Object.values(next).sort((a, b) => a - b);
  // 已选规格不足以构成完整键时，退化为前缀匹配
  return skuKeys.some(key => ids.every(id => key.split('-').includes(String(id).padStart(4, '0'))));
}
```

这段逻辑的正确性依赖 `skuKeys` 里只包含可售组合。如果把下架或零库存的组合也放进去，用户会点到「能选但下单失败」的组合。

## Edge Cases

**规格值减少导致 SKU 行消失。** 运营把「XL」从尺码里删掉，差集逻辑会删掉所有带 XL 的 SKU，连同库存。如果这个 SKU 还有在途订单或售后，行没了，售后按 item_id 查不到规格信息。处置：差集只下架不删除；真要删除，先检查是否存在未完结订单。

**删除后重新添加，item_id 会变。** 主键来自序列表（golershop 用 `NumberSeq().GetNextSeqInt` 取 `product_id` / `item_id`），不复用旧值。所以「删掉再加回来」在系统眼里是一个全新的 SKU，原来的销量、评价、活动绑定全部断掉。

**默认 SKU 缺货。** 列表页价格和主图都取默认 SKU。如果没有任何 SKU 被标记为默认，golershop 的实现是把提交数组里的**第一个**设为默认——这意味着默认 SKU 取决于提交顺序。一旦这个 SKU 缺货或下架，列表页会展示一个买不到的价格。处置：保存时校验默认 SKU 必须可售，缺货时自动切换到最低价可售 SKU。

**图片只能挂在一个规格维度上。** `product_image` 的关联键是 `product_id + color_id`，`color_id` 是系统默认颜色规格的 `spec_item_id`，没有颜色规格时为 0。也就是说多规格商品只有「颜色」这一维能挂图，尺码、材质维度无法各自配图。这是双规格（颜色 + 尺码）时代留下的结构，三规格商品要按维度挂图必须另建关联表。

**同一规格值被重复选择。** 提交 `颜色=黑` 两次，若应用层不校验，两个 SKU 的 `spec_key` 相同，唯一索引会拦下；没有唯一索引的话就是两条重复 SKU，前端永远只能查到其中一条。这也是建议加 `uk_product_spec` 的实际原因。

**规格值 id 跨租户/门店冲突。** 多商户场景下 `product_spec_item` 带 `store_id`，不同店铺可以有同名规格但 id 不同。`spec_key` 只用规格值 id 拼接，跨店比较时不可直接互用；做平台级商品库同步时，键里要再加入来源标识。

**字段注释与实现对不上。** `product_item.item_spec` 的字段注释写的是 `{spec_id: spec_item_id}` 形式的对象，但保存逻辑里是按**数组**解析的（每个元素含 `id`、`name`、`item.{id,name}`），并从 `item.name` 拼出 SKU 名称、从 `item.id` 拼出 `spec_item_ids`。二次开发如果照注释写解析代码，会在运行时才报错。这类注释漂移在长期项目里很常见，接手时以写入侧代码为准，不要以注释为准。

## Practical Notes

1. **规格值 id 是组合的唯一依据，名称只是展示。** 任何需要「重新识别同一个组合」的地方（导入商品、跨环境同步、灰度环境导数据）都只能比对 id，比对名称一定出错。
2. **商品保存不要碰库存字段。** 库存改动走独立接口，并且要求带「期望值」或版本号，避免编辑页面拿着 10 分钟前的快照把库存写回去。
3. **派生字段要有统一的刷新入口。** 最低价、最高价、默认 SKU、主图、辅助属性串，全部由同一个 `rebuildProductIndex(productId)` 完成，不要散落在保存、上下架、改价、改库存四个地方各写一遍。
4. **SKU 数量上限要在产品层面定。** 4 个规格各 10 个值就是一万个组合，即使只落存在的组合，前端规格树和运营后台的表格都会先崩。常见做法是限制参与组合的规格数（≤3），超出时改为「选配」模式。
5. **订单明细存规格名称快照。** 名称快照 + item_id 双存，售后、对账、ERP 推送三个场景都需要，缺一个就得回表反查历史。
6. **改规格属于高危操作，要有审计。** 记录「谁在什么时候把哪个 SKU 从什么规格改成了什么规格」，出了问题能定位到操作而不是只看到结果。
7. **数量为 0 与下架是两种状态。** `enabled=1` 且库存为 0 表示「暂时无货、可到货提醒」；`enabled=0` 表示「不再销售」。前端展示和筛选都要区分，否则售罄商品会从列表里直接消失，损失的是已有的商品权重。

## Summary

规格组合的问题本质是「一个多对多的选择，需要一个稳定的主键」。用规格值 id 排序拼接出来的规范键承担这个角色，商品表存规格定义快照保证历史可读，SKU 表用唯一索引保证组合不重复，编辑时用 diff 而不是删除重建保证引用不断。

判断一套 SKU 结构是否可靠，可以问四个问题：

1. 规格值改名之后，已有商品和已有订单还能不能正确显示？
2. 两个不同顺序的选规格操作，会不会算出两个不同的键？
3. 编辑一次商品，SKU 主键会不会变、库存会不会被覆盖？
4. 有没有任何读取路径，把冗余的库存或价格当成实时值来用？

四个问题都能答清楚，这套结构才敢往上加活动、加会员价、加多商户。

---

参考来源（公开仓库，LGPL-3.0，仅阅读未复制代码）：

- [shsuishang/golershop](https://github.com/shsuishang/golershop) — `internal/model/entity/product_info.go`、`product_item.go`、`product_spec.go`、`product_spec_item.go`、`product_image.go`、`product_index.go`，`internal/logic/pt/product/product_base.go`、`product_index.go`

本文整理自随商商城项目和企业电商研发实践。
