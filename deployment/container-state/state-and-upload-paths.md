# 商城服务上容器之后：会话、上传、日志到底落在哪

> 本文属于 `deployment/` 方向，讨论商城服务容器化部署时"运行时状态"的落盘位置，以及跨层限制的对齐。
> 文中标注 **Recommended Design** 或 **Example** 的部分是设计方案与示例配置，不代表某个已上线产品的既有实现。
> 涉及的配置文件均来自公开仓库（`shsuishang/golershop`、`shsuishang/modulithshop`，均为 LGPL-3.0），仅用于说明真实默认值的形态，敏感值一律替换为占位符。

## Problem

同一个商城服务，物理机上跑了两年没出事，换成容器一个月内会同时冒出三类互不相干、但根因相同的故障：

1. **后台管理员隔一段时间被踢下线**，重启容器之后所有人一起掉登录；
2. **商品图、装修图上传"成功"，几天后 404**；滚动发布一次，之前传的图整体消失；
3. **容器磁盘写满**，但想查当时的日志，容器已经没了。

三条现象看起来是三个模块的问题，实际都是同一件事：**代码里的路径默认值相对"当前工作目录"，而容器里的当前工作目录是可写层**。可写层有三个特点——跟着容器生命周期走、不跨副本共享、不进入任何备份。

物理机上这些默认值恰好指向了持久目录，所以问题被掩盖了十年。分水岭不是"用了 Docker"，而是**副本数大于 1、且容器会被重建**。只跑一个 `docker run` 且从不重建的单机容器，也能安然无恙地把这些坑带到第一次发版。

## Business Scenario

一个典型商城部署要同时处理这几类运行时数据：

| 数据 | 来源 | 丢失/不一致的后果 |
| --- | --- | --- |
| 商品主图、详情图、店铺装修图 | 运营在后台批量上传 | 前台商品页 / 详情页图片全挂 |
| 合同、发票、对账单、导入模板 | B2B 客户与财务流程 | 对账与开票凭证丢失，不可重建 |
| 后台登录会话、购物车临时态 | 用户交互 | 掉登录、购物车串号 |
| 支付回调流水、下单日志 | 应用运行 | 出问题无据可查 |
| 定时任务（超时关单、自动收货） | 调度器 | 订单卡死、库存占用不释放 |

部署形态通常是这样演进的：单机 jar / 二进制 → docker compose 单机 → K8s 多副本。前两阶段只是"换了运行方式"，第三阶段才真正考验设计，因为多副本把"本地文件"从"持久"变成了"只对当前进程可见"。

## Why It Happens

### 框架默认值全部指向工作目录

以 `golershop`（Go + GoFrame）的 `manifest/config/config.yaml` 为例，几个与状态位置直接相关的配置项：

```yaml
server:
  address: ":8000"
  readTimeout: "60s"
  writeTimeout: "0"            # 0 = 不超时
  maxHeaderBytes: "10240"

  serverRoot: "./resource/public"      # 静态文件服务根目录（相对工作目录）
  fileServerEnabled: true

  logPath: ""                          # 空 = 不写文件
  logger:
    path: "/var/log/"                  # 文件日志目录
  accessLogEnabled: false

  sessionPath: "./resource/gsessions"  # 会话文件目录（相对工作目录）
```

对应的镜像构建方式（`manifest/docker/Dockerfile`）：

```dockerfile
FROM loads/alpine:3.8

ENV WORKDIR                 /app
ADD resource                $WORKDIR/
ADD ./temp/linux_amd64/main $WORKDIR/main
RUN chmod +x $WORKDIR/main

WORKDIR $WORKDIR
CMD ./main
```

两段放一起看，结构问题就出来了：

- `WORKDIR` 是 `/app`，所以 `serverRoot` 解析成 `/app/resource/public`；
- 而 `ADD resource $WORKDIR/` 把**前端构建产物**也放进了 `/app/resource`；
- `sessionPath` 解析成 `/app/resource/gsessions`。

也就是说，**同一个目录既要装只读资产，又要装运行时写入的会话文件**。镜像里的资产和容器里的状态被塞进了一棵树，后面所有的容错设计都会卡在这里：

1. 容器重建 → `/app/resource/gsessions` 消失 → 所有会话失效。这就是"发版后全员掉登录"。
2. 如果上传目录也落在 `/app` 下（或者像常见做法挂到 `/upload`），多副本时 A 副本写入的文件，B 副本的静态服务看不到 → **随机 404**，刷新几次又能出来。
3. `logger.path` 在容器文件系统内 → 日志随容器一起消失；如果同时开 `logStdout`，同一份日志写两遍，磁盘被可写层吃掉。

### 上传这件事，限制写在四个地方

图片上传失败排查起来最费力，因为同一个 413/500 可能来自四个完全不同的层，报错信息还各不相同。

`modulithshop`（Java / Spring Boot）的 `src/main/resources/application.yml` 里，与上传相关的三段配置是并存的：

```yaml
server:
  port: 8101
  max-http-header-size: 4048576
  tomcat:
    max-http-form-post-size: 1000MB
    connection-timeout: 60000

spring:
  servlet:
    multipart:
      enabled: true
      max-file-size: 100MB
      max-request-size: 100MB
```

`max-http-form-post-size` 是 1000MB，`multipart` 是 100MB，两者不一致本身不会报错，但会造出一个非常难受的排查局面：**你以为是 A 层拦的，实际是 B 层拦的**；改了一处不生效，改了另一处又"修好了"，但没人知道真正起作用的是哪一层。再加上前置的 Nginx（`client_max_body_size` 默认 1m）和业务侧的单文件大小校验，一共四层。

| 层 | 配置点 | 常见默认/示例值 | 触发后表现 |
| --- | --- | --- | --- |
| Nginx / Ingress | `client_max_body_size` | 默认 `1m` | 413 Request Entity Too Large |
| 应用服务器 | `server.tomcat.max-http-form-post-size` | Spring Boot 默认 2MB | 表单体被丢弃或连接重置 |
| 框架上传解析 | `spring.servlet.multipart.max-file-size` | 默认 1MB / 10MB | `MaxUploadSizeExceededException` |
| 业务校验 | 自定义单文件/总大小 | 由业务定 | 业务错误码，最友好但最晚触发 |

### 时区、调度、监控台：部署时最容易"默认就好"的三处

- **时区。** 容器默认 UTC。应用不做任何设置时，`new Date()` 是对的（时间戳无时区），但 `LocalDateTime` 与 JDBC 写入会差 8 小时。表现是"下单时间和实际差 8 小时""当日对账报表少 8 小时数据"。`modulithshop` 的 `docker-compose.yml` 里给每个容器都加了 `TZ=Asia/Shanghai` 并挂载 `/etc/localtime`，这是常见做法，但注意**容器时区不等于数据库时区**：数据库自己的 `time_zone` 还得单独设。
- **定时任务。** 同一份 `application.yml` 里：

  ```yaml
  spring:
    quartz:
      job-store-type: jdbc
      jdbc:
        initialize-schema: always
  ```

  配置项旁边的注释原文是"每次启动项目都重新清空定时任务"。单机 + 单副本时代这只是一个"重启后任务要重新挂"的小麻烦；多副本后它变成**互相覆盖**：两个副本启动时都在初始化 `QRTZ_` 表，调度数据被反复重建，任务是重复执行还是不执行，取决于启动顺序。
- **监控台。** Druid 的 `stat-view-servlet` 在这个项目里是开着的，走 `/druid/*` 路径。这类管理端点在生产环境必须显式关掉或限制来源，而不是靠"没人知道这个路径"。

## Design

**Recommended Design**：不要按"目录"分类，按"**丢了会怎样**"分类。四类状态、四种落点，规则一旦定下来，新加功能时不用每次重新讨论。

| 类型 | 典型内容 | 允许丢吗 | 落点 | 多副本要求 |
| --- | --- | --- | --- | --- |
| 只读资产 | 前端 dist、模板、字体、默认图 | 丢 = 页面崩，可重建 | 打进镜像，只读 | 无 |
| 可丢缓存 | 缩略图、商品详情缓存、验证码 | 丢 = 变慢 | tmpfs / Redis | 无 |
| 必须持久 | 商品图、合同、对账单、导入原件 | 丢 = 不可恢复 | 对象存储 或 PVC/NAS | 必须共享 |
| 必须共享 | 登录会话、购物车、幂等令牌 | 丢 = 掉登录、串号 | Redis | 必须共享 |
| 需外送 | 应用日志、回调流水 | 丢 = 无据可查 | stdout + 采集 | 无（但需可聚合） |

三条原则：

1. **镜像内一律只读。** 容器根文件系统 `read_only: true`，只有显式声明的挂载点可写。让"意外写入"在部署阶段就暴露，而不是在半年后磁盘满的时候暴露。
2. **共享状态要么进 Redis，要么进对象存储，不要"共享一个 NFS 目录当会话盘"。** 文件锁在 NFS 上不可靠，网络抖动一次就掉一批会话，而且写放大很难看。
3. **日志只留一个出口。** 要么只 stdout 交给采集，要么只落文件并挂载出去；同时开两个，等于把磁盘和排障成本都翻倍。

## Data Model / Flow

一次上传请求经过的层，以及每层各自决定的东西：

```
client
  │  上传 12MB 商品主图 / 导出对账单
  ▼
Nginx / Ingress
  │   client_max_body_size       ← 请求体大小
  │   client_body_timeout        ← 上传耗时
  │   proxy_read_timeout         ← 后端处理耗时（导出常超 60s）
  ▼
app 容器（根文件系统只读）
  ├── /app/resource/public      只读资产（镜像层）
  ├── /tmp                      可写临时（tmpfs，上传中转、图片压缩）
  ├── session   ──► Redis       必须共享
  ├── /data/upload ─► 对象存储 / PVC   必须持久
  └── stdout    ──► 采集         需外送
```

请求一旦落到"不可写的位置"，失败点会出现在**框架内部**而不是业务代码里，报错通常是 `Permission denied`、`No such file or directory`、`Read-only file system` 这类与"上传"毫无字面关系的信息。这就是为什么这类问题排查成本高。

## Example

以下均为 **Example**，占位符按仓库惯例使用 `example.com`、`YOUR_*`。

### 1) 只读根文件系统 + 显式挂载（compose）

```yaml
services:
  mall-api:
    image: registry.example.com/mall-api:1.0.0
    read_only: true
    tmpfs:
      - /tmp:size=512m
    volumes:
      - mall-upload:/data/upload
    environment:
      - TZ=Asia/Shanghai
      - SPRING_PROFILES_ACTIVE=prod
      - MALL_UPLOAD_DIR=/data/upload
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/mall?serverTimezone=Asia/Shanghai
```

开 `read_only` 之后最常见的坑是**忘了给 `/tmp`**。图片压缩、Excel 导入、OpenOffice 转 PDF 都会写临时文件，失败时业务日志往往只有一行"导出失败"，真正的 `Read-only file system` 在框架日志里。

### 2) 会话与日志外置（Go 侧配置形态，示意）

```yaml
server:
  serverRoot: "/app/resource/public"   # 绝对路径，明确只读资产
  sessionPath: ""                     # 不使用文件会话
  logPath: ""                         # 不落文件，只走 stdout
  logger:
    path: ""                          # 关闭文件日志，避免双写
```

把 `serverRoot` 写成绝对路径，是为了让"这是只读资产"这件事在配置里可见；写成相对路径时，它和 `sessionPath` 长得一样，没人会注意一个可写一个不可写。Redis 会话的具体键名随框架版本不同，这里只表达"文件会话关掉、会话进 Redis"这个意图，**不要照抄键名**。

### 3) K8s：把"可丢"和"必须持久"分开

```yaml
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: main
          image: registry.example.com/mall-api:1.0.0
          env:
            - name: TZ
              value: Asia/Shanghai
          volumeMounts:
            - name: upload
              mountPath: /data/upload
            - name: tmp
              mountPath: /tmp
          readinessProbe:
            httpGet: { path: /healthz, port: 8000 }
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /healthz, port: 8000 }
            periodSeconds: 20
            failureThreshold: 6
          resources:
            requests: { cpu: 200m, memory: 512Mi }
            limits:   { memory: 1Gi }
      volumes:
        - name: upload
          persistentVolumeClaim: { claimName: mall-upload }
        - name: tmp
          emptyDir: { sizeLimit: 2Gi }
```

注意 `emptyDir` 的 `sizeLimit`：不设上限时，一个跑飞的导出任务可以把节点磁盘写满，进而影响同节点上的其他 Pod。这在商城场景里很容易发生——运营一次导出十万行订单。

另外，公开仓库里的 kustomize base（`manifest/deploy/kustomize/base/deployment.yaml`）只有 `replicas: 1`、镜像和 `imagePullPolicy`，**没有探针、没有资源限制、没有挂载**。这类模板适合起步，直接搬到生产会同时缺三样东西：滚动发布时的流量摘除、内存打爆时的自保、以及上面说的状态落点。

### 4) Nginx 侧把限制写成显式值

```nginx
server {
    client_max_body_size 20m;         # 与业务侧单文件上限保持一致
    client_body_timeout  60s;

    location /api/ {
        proxy_pass http://mall_api;
        proxy_read_timeout 120s;      # 导出对账单类请求超过默认 60s
        proxy_send_timeout 120s;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # 上传目录只读暴露，禁止脚本执行
    location /upload/ {
        alias /data/upload/;
        add_header X-Content-Type-Options nosniff;
        add_header Content-Disposition "attachment";
        location ~* \.(php|jsp|sh|exe)$ { deny all; }
    }
}
```

`alias` 结尾的斜杠必须和 `location` 前缀对齐，否则会出现目录穿越（`/upload../`）。这一条和上传大小无关，但在同一个 `location` 里配，一起改最省事。

## Edge Cases

**1. 只读根文件系统下的隐式写入。** 除了上传和会话，很多组件会往工作目录或家目录写东西：字体缓存、`~/.cache`、临时解压目录、JDBC 驱动的本地元数据。上线前把 `read_only` 打开跑一遍主流程（上传、导出、支付回调、缩略图生成），比读文档有效得多。

**2. 多副本 + 会话私存，表现不是"掉登录"而是"隔几次掉一次"。** 用户投诉随机、开发复现不了，是因为负载均衡把连续请求分到了不同副本。判据很简单：在响应头里带上实例标识，连续刷新 20 次看 `Set-Cookie` 的变化。

**3. 上传目录共享、缩略图缓存也共享。** 两个副本同时为同一张图生成缩略图，用户会看到半张图或损坏文件。要么"先写临时文件再原子 rename"，要么干脆不共享缩略图缓存（放 tmpfs，丢了自动重建）。

**4. 时区三处不一致。** 排查一次就知道自己错在哪：

```sql
SELECT @@global.time_zone, @@session.time_zone, NOW(), UTC_TIMESTAMP();
```

应用侧 `TZ=Asia/Shanghai`、连接串 `serverTimezone=Asia/Shanghai`、数据库 `time_zone='+08:00'`，三处一起看。跨时区业务（跨境商城）建议统一存 UTC、只在展示层转换，否则"日切"逻辑会被时区反复坑。

**5. 调度器多副本。** `quartz.jdbc.initialize-schema: always` 在多副本下会互相覆盖调度表。两个选择：单副本专门跑调度（把这个进程独立出去），或者开集群模式 `org.quartz.jobStore.isClustered=true` 并用数据库锁协调。不要"两个副本都跑调度，靠业务幂等兜底"——超时关单这类任务重复执行会直接改订单状态。

**6. 上传文件迁移。** 从旧机器迁到对象存储时不要"迁完就切"。先双写、再双读比对、最后切读；校验项至少包括文件大小与 MD5（或 ETag）。少这一步，线上会出现一批 0 字节文件，而且是用户先发现的。

## Practical Notes

上线前按顺序过一遍，每条都能在 10 分钟内完成：

1. `docker diff <container>` 看容器可写层到底被写了什么。写了不该写的地方，这里会直接列出来。
2. 打开 `read_only: true`，跑完主流程：后台上传三张不同大小的图（1MB / 8MB / 19MB）、导出一次对账单、走一次支付回调。
3. 上传落点用两个副本交叉验证：在副本 A 上传，立刻从副本 B 读 URL。读不到就是共享没做对。
4. 会话进 Redis 后，登录 + 连续刷新 20 次不掉；再把一个副本杀掉，会话应仍然有效。
5. 把三层上传限制（Nginx / 应用 / 业务）写成一张表放进部署文档，**改的时候一起改**。这张表比任何注释都有用。
6. 时区三处对齐（容器 / 连接串 / 数据库），并跑一次"当天订单数"统计核对。
7. 日志只保留一个出口；保留天数 × 单日峰值 要小于磁盘容量。
8. 探针职责分清：`readiness` 只判断"能不能接流量"，不要查第三方支付；`liveness` 不要查数据库——数据库抖一下会把整个集群重启，把一次小故障放大成全站不可用。

## Summary

容器化不改业务代码，改的是"当前工作目录"的语义。商城系统里最容易出事的三处运行时状态是**会话、上传、日志**——它们的默认值都写在容器里，而且在配置文件里看起来完全正常。

判断标准只有一句话：**这份数据，容器删掉重建之后，还能不能恢复？** 不能，就别让它留在容器里。

顺带一句：上传失败、时间差 8 小时、发版后掉登录，这三类"线上问题"里，有相当一部分不是代码缺陷，而是部署时各层的默认值没有人负责对齐。把限制写成显式配置、把状态放进显式挂载点，比事后排查便宜得多。

---

本文整理自随商商城项目和企业电商研发实践。示例配置中的路径、镜像地址、密钥均为占位符。
