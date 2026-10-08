# 马帮 ERP 昨日订单 → 企业微信智能表格（每天下午 3 点自动同步）

用 GitHub Actions 当免费定时器：**每天北京时间 15:00** 自动从马帮拉取**昨天**的订单，把
`订单号 / SKU / 产品数量 / 出单时间 / 店铺名称` 五行数据写进**企业微信群里的智能表格**，并可选在群里发一条同步提醒。

---

## 0. 好消息：智能表格有官方写入接口，不用建自建应用

企业微信智能表格自带一个叫 **「接收外部数据」** 的能力：

- 在智能表格里点几下就能开启，系统生成该工作表专属的 Webhook 地址
- 外部系统直接 `POST` 就能新增记录，**不需要 access_token、不需要申请自建应用**
- 单个工作表 **3000 条/分钟**，整个文档 10000 条/分钟
- 需要企业微信客户端 **v5.0.6 及以上**

```
   马帮 ERP                  GitHub Actions                  企业微信智能表格
┌──────────────┐  取数   ┌────────────────────────┐  写入   ┌──────────────────┐
│ get-order-list│ ─────► │ 每天 15:00 触发          │ ─────► │ 「接收外部数据」   │
│ HMAC-SHA256   │        │ 昨天00:00~今天00:00      │        │ Webhook 地址      │
│ 签名鉴权       │        │ 展开到 订单×SKU 粒度      │        └──────────────────┘
└──────────────┘        │ 分批写入 + 本地备份CSV    │  通知   ┌──────────────────┐
                        └────────────────────────┘ ─────► │ 群里发一条提醒     │
                                                          └──────────────────┘
```

---

## 1. 目录说明

```
mabang-wecom-smartsheet/
├── GITHUB_SETUP.md                      ⭐ GitHub 网页操作图文指南（不知道从哪下手先看这个）
├── .github/workflows/daily-orders.yml   定时任务（cron "0 7 * * *" = 北京 15:00）
├── scripts/
│   ├── main.py          主流程：取数 → 汇总 → 写入表格 → 群通知
│   ├── mabang.py        马帮开放平台客户端（HMAC-SHA256 签名、分页、字段映射）
│   └── smartsheet.py    智能表格 Webhook 写入 + 企业微信群通知
├── tools/
│   └── setup_wizard.py  配置向导：填 Webhook / 字段 ID + 探针测试反查
├── config.example.yaml  配置模板（复制成 config.yaml 再改）
├── requirements.txt
└── .gitignore           已排除 config.yaml / output
```

> **字段 ID 可以在 GitHub 网页上配**（Settings → Variables），不用改代码：
> `SS_COL_ORDER_NO` / `SS_COL_SKU` / `SS_COL_QTY` / `SS_COL_TIME` / `SS_COL_SHOP`

**签名算法已经用马帮官方文档给出的两组样例逐字符验证通过**（v1、v2 都对得上），拿到 AppKey 后可直接用。

---

## 2. 部署步骤

### Step 1 · 拿到马帮开放平台凭证

1. **AppKey 与密钥是按企业主账号生成的**，需要联系马帮客服开通开放平台权限（部分能力是付费的）
2. 开通后你会得到两个值：`AppKey`（例如 `10000`）和 `App 密钥`（例如 `c5540d63...`）
3. 同时跟客服确认：**订单查询接口（`get-order-list`）已开通**

> 马帮网关：`https://gwapi.mabangerp.com/api/v2`（v2，推荐）；老网关 `http://openapi.mabangerp.com`（v1）

### Step 2 · 建智能表格并开启「接收外部数据」

1. 企业微信里新建**智能表格**，建好表头这 5 列：
   `订单号`（文本）、`SKU`（文本）、`产品数量`（数字）、`出单时间`（日期）、`店铺名称`（文本）
2. 点智能表格右上角 **「...」** → **接收外部数据** → **开始配置** → 选择目标工作表
   （或者点工作表名字旁的三个点 → 接收外部数据）
3. 复制生成的 **Webhook 地址**，形如
   `https://qyapi.weixin.qq.com/cgi-bin/wedoc/smartsheet/webhook?key=QIikPDZRKD7tw2m4AdZ...`
4. **只需要 `key=` 后面那一串**，这就是 `SMARTSHEET_WEBHOOK_KEY`
5. 配置页里会显示 **「示例数据」** —— 里面的 key 就是**字段 ID**，抄下来填到 `config.yaml` 的 `columns` 里

> ⚠️ **不要开启 IP 白名单**。GitHub Actions 的出口 IP 是变动的，开了白名单会写不进去。
> 如果你一定要开白名单，就得把任务迁到有固定出口 IP 的国内服务器上跑。

### Step 3 · 拿群机器人 key（可选，只用来发提醒）

企业微信 **PC 端** → 目标**内部群** → 右上角 `...` → **消息推送** → **添加** → 新建 → 复制 Webhook 里的 `key`。

> ⚠️ 老规矩：**消息推送机器人只能加到企业内部群**，群里有个人微信用户的外部群加不了。
> 不过注意——这只影响"发提醒"，**不影响往智能表格写数据**。

### Step 4 · 建仓库、配 Secrets

```bash
cd mabang-wecom-smartsheet
git init && git add . && git commit -m "init: 马帮订单同步"
git branch -M main
git remote add origin https://github.com/<你的用户名>/mabang-wecom-smartsheet.git
git push -u origin main
```

**仓库选 Private**（订单数据不上公开网，且 Private 仓库每月 2000 分钟 Actions 免费额度完全够用）。

然后 **Settings → Secrets and variables → Actions → New repository secret**：

| Secret 名称 | 值 |
|---|---|
| `MABANG_APPKEY` | 马帮 AppKey |
| `MABANG_SECRET` | 马帮 App 密钥 |
| `SMARTSHEET_WEBHOOK_KEY` | 智能表格 Webhook 的 key |
| `WECOM_BOT_KEY` | 群消息推送的 key（可选） |

### Step 5 · 先核对字段名，再试运行

马帮官方文档**没有完整列出响应字段名**，所以第一次务必先核对。

**5-1 本地跑一遍**：

```bash
pip install -r requirements.txt
cp config.example.yaml config.yaml

# 用假数据看整条链路
python scripts/main.py --config config.yaml --dry-run --mock
```

**5-2 核对马帮真实返回的字段名**：

先在本机（或直接去 Actions 手动跑）执行一次，把第一条原始记录打出来：

```bash
export MABANG_APPKEY=你的AppKey
export MABANG_SECRET=你的密钥
python scripts/main.py --config config.yaml --dry-run --dump
```

日志里会打印：

```
===== 第一条原始记录（用于核对字段名）=====
{ "platformOrderId": "221008...", "shopId": "...", "orderItem": [ {...} ], ... }
=========================================
```

**对照着把真实字段名补进 `scripts/mabang.py` 顶部这几个列表**（如果字段名和默认不同的话）：

```python
ORDER_NO_KEYS   = ["platformOrderId", "orderId", ...]   # 订单号
SHOP_NAME_KEYS  = ["shopName", "shop_name", ...]        # 店铺名称
ORDER_TIME_KEYS = ["paidTime", "createDate", ...]       # 出单时间
ITEM_LIST_KEYS  = ["orderItem", "orderItems", ...]      # SKU 明细数组
ITEM_SKU_KEYS   = ["platformSku", "stockSku", ...]      # 明细里的 SKU
ITEM_QTY_KEYS   = ["quantity", "qty", ...]              # 明细里的数量
```

> 已知字段（来自马帮文档与公开对接资料）：
> `platformOrderId` 订单编号、`salesRecordNumber` 销售记录号、`shopId` 店铺编号、`orderStatus` 订单状态、
> `paidTime` 付款时间、`createDate` 创建时间、`expressTime` 发货时间、`orderFee` 订单金额、`platformId` 来源平台；
> 明细数组 `orderItem[]` 里：`stockSku` 马帮SKU、`platformSku` 平台SKU、`quantity` 数量、`itemId`、`sellPrice`。
>
> **店铺名称**：马帮订单接口里通常只给 `shopId`。脚本会自动尝试调 `get-shop-list` 把 `shopId` 换成店铺名；
> 如果那个接口没权限，就会**直接用 shopId 填进去**（不会报错中断）。

**5-3 填字段映射**：把 Step 2 抄下来的**字段 ID** 填进 `config.yaml`：

```yaml
smartsheet:
  columns:
    - source: order_no
      target: "fldABCDEF123"   # ← 换成智能表格里「订单号」的字段 ID
      type: text
    - source: sku
      target: "fldGHIJKL456"   # ← 「SKU」
      type: text
    - source: quantity
      target: "fldMNOPQR789"   # ← 「产品数量」
      type: number
    - source: order_time
      target: "fldSTUVWX012"   # ← 「出单时间」
      type: date
    - source: shop_name
      target: "fldYZABCD345"   # ← 「店铺名称」
      type: text
```

**5-4 去 Actions 手动验证**：
仓库 → **Actions** → 左侧选本 workflow → **Run workflow**：
选 `dry_run = true`、`dump = true` → 跑一次，看日志里打印出来的 JSON 结构对不对。
确认无误后，再选 `dry_run = false` 跑一次 —— 这时智能表格里应该真的多出昨天的记录了。

### Step 6 · 打开定时

前 5 步全部验证通过后，定时任务自动生效（`cron: "0 7 * * *"` = 北京时间每天 15:00）。

想改时间，记住换算规则 —— **cron 用 UTC，北京时间 = UTC + 8**：

| 北京时间 | cron |
|---|---|
| 15:00 | `0 7 * * *` |
| 09:00 | `0 1 * * *` |
| 18:30 | `30 10 * * *` |
| 工作日 15:00 | `0 7 * * 1-5` |

---

## 3. 必读的坑

### 马帮接口侧

| 坑 | 说明 |
|---|---|
| **查询窗口限制** | 7:00~19:00 只支持"近一天"范围查询；其余时段开始与结束间隔不能超过 24 小时。脚本默认取 `[昨天00:00:00, 今天00:00:00]` —— **正好 24 小时，两个时段都合法** |
| **必须传时间范围** | 不传 `paidtimeStart` 等时间参数，接口拿不到数据 |
| **签名是对 Body 原始字符串** | 代码里已用 `data=body_str.encode()` 发原始字节，**不要改成 `json=`**，否则序列化差异会导致签名失败 |
| **时间戳容差 10 分钟** | 服务器时间要准，GitHub runner 没问题 |
| **频率 300 次/分钟** | 代码已读 `X-RateLimit-*` 响应头并做退避重试 |
| **多状态去重** | `status=6`（所有未发货）和 `7`（所有非未发货）结果会重叠，脚本按 `platformOrderId` 去重 |

### 智能表格侧

| 坑 | 说明 |
|---|---|
| **不要开 IP 白名单** | GitHub Actions 出口 IP 不固定，开了就写不进去 |
| **Webhook 地址 = 写权限密码** | 只能放 GitHub Secrets，泄露了别人能往你表里写垃圾数据 |
| **单次 500 行内** | 代码自动分批（500 行/批 + 间隔 0.3 秒） |
| **日期格式** | 日期字段用 `YYYY-MM-DD HH:MM:SS` 字符串，脚本已统一格式化 |
| **字段 ID 优先** | `target` 填字段 ID 最稳；填字段标题在某些情况下会因重名而失败 |

### GitHub Actions 侧

| 坑 | 说明 |
|---|---|
| **cron 是 UTC** | 写错会半夜跑。北京 15:00 → `0 7 * * *` |
| **定时会延迟** | 免费 runner 高峰可能延迟 5~20 分钟，不是精确闹钟 |
| **只跑默认分支** | schedule 只在仓库默认分支生效 |
| **公有仓库 60 天无活动会暂停** | 选 Private 没这问题 |
| **runner 在海外** | 调用马帮国内网关可能偶发超时，代码已加 3 次重试。若长期不稳，把同一份代码搬到腾讯云 SCF / 阿里云 FC 即可，基本不用改 |

---

## 4. 常见问题

**Q：为什么用了 `status: ["6","7"]` 而不是一个状态？**
文档里没有"全部订单"选项。6 是"所有未发货"、7 是"所有非未发货"，两个并集 = 除"已作废"外的全部订单。脚本会自动去重。如果你只想看未发货的，改成 `["6"]` 即可。

**Q：能不能顺便把产品图片也写进表格？**
可以，但有两个前提：① 马帮返回里要有图片 URL；② 智能表格那一列要建成**图片字段**，且图片需要先通过接口上传拿到 `image_url`。这比文本字段麻烦不少，建议先用文本字段存图片链接，需要时再升级。

**Q：昨天的订单里有今天才付款的，会统计进去吗？**
不会。脚本按 `paidtime`（付款时间）筛选，并且有 `filter_to_target: true` 做二次过滤，只保留出单时间落在 `[昨天00:00, 今天00:00)` 的行。

**Q：想改成导出"今天"的订单？**
把 `scripts/mabang.py` 的 `resolve_window()` 改成返回 `(today0, now)` 即可。注意马帮的窗口限制：如果跑在 19:00 之后，跨度别超过 24 小时。
