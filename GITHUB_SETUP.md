# GitHub 操作指南（从零到跑通）

> 你已经拿到智能表格 Webhook 了，所以这篇从**建仓库**开始。全程只需要浏览器，不用装 Git、不用装 Python。

---

## 先搞清楚三个概念

| 名词 | 是什么 | 在这个项目里对应 |
|---|---|---|
| **仓库（Repository）** | 存代码的文件夹，放在 GitHub 云上 | 存这个项目的 10 个文件 |
| **Actions** | GitHub 免费送的"定时服务器" | 每天 15:00 自动跑脚本 |
| **Secrets / Variables** | 给 Actions 传参数的两个抽屉 | 放马帮密钥、Webhook 地址、字段 ID |

**Secrets 和 Variables 的区别**（很重要）：

- **Secrets** = 加密抽屉。存密码密钥，日志里会自动打码成 `***`。**放马帮密钥和 Webhook 地址**
- **Variables** = 明文抽屉。存不敏感配置，日志里能看见，方便排查。**放字段 ID**

> ⚠️ 反了会出事：密钥放 Variables 会明文出现在日志里，泄露风险高；
> 字段 ID 放 Secrets 则排查问题时看不见值，很难定位。

---

## 路径 A：纯网页操作（推荐，零安装）

### 第 1 步 · 建一个 Private 仓库

1. 登录 GitHub，点右上角 **`+`** → **New repository**
2. **Repository name** 填 `mabang-wecom-smartsheet`
3. 勾选 **Private** ← 必选！订单数据不能公开
4. 下面的 "Add a README file" 等选项**全部不勾**
5. 点 **Create repository**

> Private 仓库每月有 **2000 分钟** Actions 免费额度。这个任务每天跑不到 1 分钟，够用几十年。

---

### 第 2 步 · 上传项目文件

仓库建成后，你会看到一个空仓库页面。总共要上传 **10 个文件**：

- **5 个扁平文件**（不带目录）→ 直接拖拽上传
- **5 个带子目录的文件** → 网页上传建不了目录，需要一个个手动建

> 💡 **嫌手动建 10 个文件麻烦？** 有个更快的办法：装一个 [GitHub Desktop](https://desktop.github.com/)，
> 克隆这个空仓库到本地，把整个项目文件夹拖进去，点一下 Publish 就完事了，
> 目录结构、`.github` 隐藏文件夹全部自动保留。下面的网页操作法只是"什么都不装"的兜底方案。

#### 2-1 上传普通文件（5 个）

在空仓库页面点 **uploading an existing file**（或 **Add file → Upload files**），把这 5 个文件拖进去：

```
README.md
GITHUB_SETUP.md          ← 就是本篇
requirements.txt
.gitignore
config.example.yaml
```

> 一共 **5 个**扁平文件（不带目录的）。注意 `.gitignore` 前面有个点，别漏。
> 另外 `config.yaml` **不需要上传**——它是你本地的私有配置（含 Webhook 地址），已被 `.gitignore` 排除；
> 仓库里只放 `config.example.yaml` 这个模板，GitHub 上跑任务时会自动复制一份。

然后**还要建 5 个带子目录的文件**——网页上传没法建目录，所以要一个一个手动建：

- 点 **Add file → Create new file**
- 文件名框里输入完整路径，**输入 `/` 会自动创建目录**：

| 要建的文件 | 在文件名框里输入 |
|---|---|
| 主流程 | `scripts/main.py` |
| 马帮客户端 | `scripts/mabang.py` |
| 智能表格写入 | `scripts/smartsheet.py` |
| 配置向导 | `tools/setup_wizard.py` |
| **定时任务** | `.github/workflows/daily-orders.yml` |

建每个文件的方法：文件名框输入路径 → 把文件内容粘到下面的大编辑框 → 拉到底点 **Commit changes**。

> ⚠️ **`.github/workflows/` 这个路径最容易被漏掉或拼错。**
> 必须是 `.github`（前面有个点）、`workflows`（复数）、文件名 `daily-orders.yml`。
> 拼错了定时任务就不会出现。

#### 2-2 检查是否上传成功

回到仓库首页，应该看到**完整的 10 个文件**（`.github` 是隐藏不了的，GitHub 会显示）：

```
.github/workflows/daily-orders.yml    ← 定时任务（最关键）
tools/setup_wizard.py                 ← 配置向导
scripts/main.py                       ← 主流程
scripts/mabang.py                     ← 马帮客户端
scripts/smartsheet.py                 ← 智能表格写入
GITHUB_SETUP.md
README.md
config.example.yaml
requirements.txt
.gitignore
```

**对照着数一遍，必须是 10 个。** 少一个都可能在运行时才报错。
如果目录结构不对，直接删掉重新建——Actions 找不到 workflow 文件是不会报错的，只会静默不运行，很容易白等一天。

---

### 第 3 步 · 配置 Secrets（3 个）

**Settings → Secrets and variables → Actions → Secrets 标签页 → New repository secret**

| Name | Secret 填什么 | 从哪来 |
|---|---|---|
| `MABANG_APPKEY` | 马帮 AppKey | 马帮客服给你的 |
| `MABANG_SECRET` | 马帮 App 密钥 | 马帮客服给你的 |
| `SMARTSHEET_WEBHOOK_KEY` | 智能表格 Webhook 的 **key** | 你已拿到 ✅ |

**`SMARTSHEET_WEBHOOK_KEY` 怎么填？**

你拿到的地址长这样：

```
https://qyapi.weixin.qq.com/cgi-bin/wedoc/smartsheet/webhook?key=QIikPDZRKD7tw2m4AdZjZ4aqcfzxeqwGKL3USxhZ9yehr3
```

**只需要填 `key=` 后面那一串**：

```
QIikPDZRKD7tw2m4AdZjZ4aqcfzxeqwGKL3USxhZ9yehr3
```

> 直接粘完整地址也行（脚本会自动提取 key），但只填 key 更干净。
> 填完点 **Add secret**。Secrets 保存后**无法再查看明文**，只能覆盖——这是防泄露设计，正常。

---

### 第 4 步 · 配置 Variables（5 个字段 ID）

**还是在 Settings → Secrets and variables → Actions，切到 Variables 标签页 → New repository variable**

这一步是**填智能表格里每个字段的 ID**。字段 ID 长这样：`fldXXXXXXXXXXXX`（一串字母数字）。

**字段 ID 在哪找？**

打开你的智能表格 → 进入「接收外部数据」配置页 → 里面有一块 **「示例数据」**，长这样：

```json
{
  "add_records": [
    {
      "values": {
        "fldAAA111": [{ "type": "text", "text": "文本内容" }],
        "fldBBB222": [{ "type": "text", "text": "文本内容" }],
        "fldCCC333": 100,
        "fldDDD444": "2024-06-01 12:00:00",
        "fldEEE555": [{ "type": "text", "text": "文本内容" }]
      }
    }
  ]
}
```

每个 key（`fldAAA111` 这种）就是一个字段 ID。**按顺序抄下来**——不过你很可能不知道哪个 ID 对应哪一列，别急，第 6 步的探针测试会帮你反查。

按下表新建 5 个 Variable：

| Name | Value |
|---|---|
| `SS_COL_ORDER_NO` | 订单号那一列的字段 ID |
| `SS_COL_SKU` | SKU 那一列的字段 ID |
| `SS_COL_QTY` | 产品数量那一列的字段 ID |
| `SS_COL_TIME` | 出单时间那一列的字段 ID |
| `SS_COL_SHOP` | 店铺名称那一列的字段 ID |

> 不想在网页上一个个配？也可以直接把 ID 填进仓库里的 `config.example.yaml`
> 的 `columns` 里（网页上点开文件 → 铅笔图标 → 编辑 → Commit）。
> **两种方式选一种就行**，用 Variables 的好处是以后改配置不用动代码。

---

### 第 5 步 · 跑第一次：核对马帮字段名

马帮官方文档**没有完整列出订单接口的返回字段名**，所以第一次必须先看一眼真实返回。

1. 点仓库顶部的 **Actions** 标签页
2. 左侧列表点 **马帮订单同步到企业微信智能表格**
3. 右侧点 **Run workflow** 按钮（下拉框出现）
4. 按这样选：
   - `dry_run` 选 **`true`** ← 千万别选 false，第一次只试运行
   - `dump` 选 **`true`** ← 打印马帮返回的原始数据
5. 点绿色 **Run workflow**
6. 等几秒刷新页面，会出现一条运行记录，点进去，再点左上角 **sync** 步骤
7. 在日志里找到这一段：

```
===== 第一条原始记录（用于核对字段名）=====
{
  "platformOrderId": "221008...",
  "shopId": "12345",
  "paidTime": "2026-10-07 14:32:00",
  "orderItem": [
    { "platformSku": "HALLOWEEN-TEE-M", "quantity": 2 }
  ]
}
=========================================
```

**对照着检查日志里的这几行**：

```
共 N 行（订单 × SKU 粒度）
N 单 / N 件 / N 个店铺
```

**如果数字是 0，或者日志显示找不到字段**：说明马帮返回的字段名和脚本默认的不一样。
在网页上打开 `scripts/mabang.py` → 铅笔图标编辑 → 找到顶部这几个列表，把真实字段名加进去：

```python
ORDER_NO_KEYS   = ["platformOrderId", "orderId", ...]   # 订单号
SHOP_NAME_KEYS  = ["shopName", "shop_name", ...]        # 店铺名称
ORDER_TIME_KEYS = ["paidTime", "createDate", ...]       # 出单时间
ITEM_LIST_KEYS  = ["orderItem", "orderItems", ...]      # SKU 明细数组
ITEM_SKU_KEYS   = ["platformSku", "stockSku", ...]      # 明细里的 SKU
ITEM_QTY_KEYS   = ["quantity", "qty", ...]              # 明细里的数量
```

改完拉到底 **Commit changes**，再跑一次。

---

### 第 6 步 · 反查字段 ID 对不对（探针测试）

把字段 ID 配好后，**这一步能一眼看出有没有填错**。

在你自己的电脑上（如果装了 Python），或者直接看第 5 步日志里打印的 JSON，把内容拿去和表格对：

其实更省事的办法是**手动发一条测试数据**：

1. 在仓库 Actions 页面 → Run workflow
2. `dry_run` 选 **`false`**（这次是真的写）
3. 跑完后打开智能表格，看看有没有多出一行数据
4. **重点核对**：订单号列显示的是不是订单号、SKU 列显示的是不是 SKU
   - 如果**串列了**（比如订单号跑到了 SKU 列）→ 说明两个字段 ID 填反了
   - 去 Settings → Variables 把两个 ID 对调，再跑一次

> 想用更精确的探针（带 ①②③ 编号标记，一眼就能定位）：
> 本地装好 Python 后运行 `python tools/setup_wizard.py`，按引导走完三步即可。
> 它会往表格写一行 `①订单号-PROBE` `②SKU-PROBE` 这样的数据，你打开表格一看就知道哪个 ID 对应哪一列。

---

### 第 7 步 · 确认定时任务已生效

前 6 步都验证通过后，定时任务**自动就生效了**，不需要额外操作。

Workflow 里写的是：

```yaml
on:
  schedule:
    - cron: "0 7 * * *"    # UTC 07:00 = 北京时间 15:00
```

**每天北京时间下午 3 点，自动跑一次，把昨天的订单写进智能表格。**

怎么确认它真的跑了：Actions 页面会按时间留下运行记录，每天多一条。
（注意 GitHub 免费 runner 在整点高峰期可能延迟 5~20 分钟，不是精确到秒的闹钟。）

---

## 路径 B：本地先验证（可选，但推荐）

如果你电脑上有 Python，我强烈建议**先在本地跑通，再传 GitHub**。本地调试比看 Actions 日志快得多。

```bash
cd mabang-wecom-smartsheet
pip install -r requirements.txt
cp config.example.yaml config.yaml

# 1. 用假数据看整条链路长什么样（不需要任何凭证）
python scripts/main.py --config config.yaml --dry-run --mock

# 2. 配好 Webhook 和字段 ID（交互式，会写进 config.yaml）
python tools/setup_wizard.py

# 3. 探针测试，核对字段 ID
python tools/setup_wizard.py --no-input --probe

# 4. 核对马帮真实字段名（需要 MABANG_APPKEY / MABANG_SECRET 环境变量）
export MABANG_APPKEY=你的AppKey
export MABANG_SECRET=你的密钥
python scripts/main.py --config config.yaml --dry-run --dump

# 5. 真跑一次（会真的写进智能表格）
export SMARTSHEET_WEBHOOK_KEY=你的key
python scripts/main.py --config config.yaml
```

> 本地跑 `config.yaml` 是 gitignore 的，**不会被提交到 GitHub**，
> 所以你在本地填的 Webhook、字段 ID 都是安全的，仓库里只有 `config.example.yaml` 模板。

---

## 排错对照表

| 现象 | 原因 | 解决 |
|---|---|---|
| Actions 页面左侧**找不到**这个 workflow | `.github/workflows/` 路径拼错，或文件名不对 | 检查路径必须是 `.github/workflows/daily-orders.yml` |
| `Run workflow` 按钮是灰的 | workflow 不在默认分支上 | 确认文件在 `main` 分支 |
| 报 `缺少马帮 AppKey / AppSecret` | Secrets 名字拼错或没建 | 检查 `MABANG_APPKEY` / `MABANG_SECRET` 拼写 |
| 报 `缺少智能表格 Webhook 地址` | 同上 | 检查 `SMARTSHEET_WEBHOOK_KEY` |
| `errcode 840001 invalid webhook` | Webhook key 错、被停用、或表格没开「接收外部数据」 | 重新去表格里复制一次 Webhook 地址 |
| 日志显示 `共 0 行` | 昨天真的没订单，或马帮接口窗口/字段问题 | 先确认昨天有订单；再用 `dump=true` 看原始返回 |
| 表格里**多了一堆空的列** | 字段 ID 填错，写到了不存在的字段 | 用探针测试反查，改 Variables |
| 马帮报签名错误 | 密钥错了，或系统时间偏差超 10 分钟 | 检查 `MABANG_SECRET`；GitHub runner 时间通常是准的 |
| 表格里店铺列全是数字 | 马帮订单接口只给 `shopId`，没权限查店铺名 | 正常降级行为；如需店名，找马帮开通 `get-shop-list` 权限 |

---

## 一句话总结操作顺序

```
建 Private 仓库
   → 传 10 个文件（5 个扁平 + 5 个带目录，注意 .github 路径！）
      → 配 3 个 Secrets（马帮密钥 + Webhook key）
         → 配 5 个 Variables（字段 ID）
            → 手动跑 dry_run=true + dump=true 核对字段名
               → 改用 dry_run=false 跑一次，看表格里有没有数据
                  → 完事，每天 15:00 自动跑
```
