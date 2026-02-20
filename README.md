# Codex 工程规格 + 分步 Prompt 包（最终版：含“单位不一致确认页”大字 UI 文案 & TTS）

> 目标：用 **Kotlin + Jetpack Compose + OkHttp + kotlinx.serialization + EncryptedSharedPreferences** 做一款 **语音优先、适老大字、高对比** 的烟酒食品零售店进销存安卓 App。  
> 数据：手机本地 **2 个 CSV**（Excel 可直接打开）作为唯一持久化：`purchase_records.csv` + `inventory.csv`。

---

## 0. CSV 定义（固定表头）

### 0.1 进货记录表（流水表）`purchase_records.csv`
表头（第一行必须一致）：
- `名称,数量,单位,售价,进价,进货日期`

说明：
- `数量`：**进货为正，出货为负**（由 App 落库时决定；模型输出可保持正数，App 对 sale 转负数）
- `单位`：条/包/瓶/箱/个/未知（建议只允许这几类 + 用户自定义可选）
- `进货日期`：统一 `YYYY-MM-DD`
- 允许 `售价/进价` 为空（例如出货时进价可能不填）

### 0.2 库存记录表 `inventory.csv`
表头：
- `名称,数量,单位,售价,进价`

说明：
- 每个商品（name）在库存表里 **只有一种单位**（unit）。  
- 成本 `进价`：使用 **移动加权平均成本**（仅在单位一致时更新）。

### 0.3 文件写入约束（必须）
- 编码：**UTF-8 with BOM**（文件头写入 `\uFEFF`），避免 Excel 中文乱码。
- 原子写：更新 `inventory.csv` 必须写临时文件 `inventory.tmp` → close/flush → replace，并保留 `inventory.bak`（上一版本）。
- CSV 转义：字段含 `,` 或 `"` 或换行时必须正确加引号并转义 `"` 为 `""`。

---

## 1. 适老 UI/交互硬指标（验收标准）

1) 默认字体：
- 正文 ≥ **22sp**
- 商品名 ≥ **30sp**（粗体）
- 关键数字（库存数量）≥ **40sp**
- 按钮文字 ≥ **22sp**

2) 大按钮：
- 主要按钮高度 ≥ **64dp**，触控区域 ≥ **56dp**，按钮间距 ≥ **12dp**

3) 语音优先闭环：
- “按住说话”按钮：按下开始识别，松开结束
- 每次识别后必须：**大字展示解析结果 + TTS 播报确认**
- 必须可恢复：重说 / 手动改字段 / 取消

4) 查询结果展示：
- 大字显示 商品名 + 库存（超大） + 售价/进价
- TTS 播报：“{商品} 库存 {数量}{单位}，售价 {售价} 元。”

5) 必须支持“撤销最近一条流水”：
- 撤销 = 写一条“反向流水”（quantity 取相反数）并重算库存

---

## 2. 数据模型（建议，Codex 按此实现）

### 2.1 领域模型
- `Item(name: String, qty: Double, unit: String, salePrice: Double?, buyPrice: Double?)`
- `Tx(op: Op, name: String, quantity: Double, unit: String, salePrice: Double?, buyPrice: Double?, date: LocalDate, notes: String?)`
- `Op = purchase | sale | reprice | adjust | query`

### 2.2 CSV 行模型（读写）
- `PurchaseRow(name, quantity, unit, salePrice, buyPrice, dateStr)`
- `InventoryRow(name, quantity, unit, salePrice, buyPrice)`

### 2.3 apply 结果（用于 UI 分支）
- `ApplyResult.Success(updatedInventory)`
- `ApplyResult.NeedConfirmUnitMismatch(existingUnit, incomingUnit, itemSnapshot, tx)`
- `ApplyResult.NeedConfirmStockNegative(currentQty, newQty, tx)`
- `ApplyResult.Error(message)`

---

## 3. 业务规则（最小正确实现）

### 3.1 单位一致性（强制）
- 若库存已有该商品且 `incomingUnit != existingUnit`：**不允许自动落库**，必须进入“单位不一致确认页”。
- 选择项（详见组件说明）：
  1) **按库存单位改写**：本次 tx 的 `unit` 改成 `existingUnit`（数量数字不变）
  2) **更新库存单位为本次单位**：库存该商品 unit 改为 `incomingUnit`（数量数字不变）
  3) **取消并重说**：不写入任何 CSV

> 注意：不做自动换算（条↔包、箱↔瓶），避免错误库存。

### 3.2 进货 purchase（quantity > 0）
- 追加一行到 `purchase_records.csv`（数量为正）
- 更新库存：
  - `newQty = oldQty + qty`
  - 成本进价（单位一致时）：`newCost = (oldCost*oldQty + buyPrice*qty) / newQty`
  - 若 tx 提供 salePrice，则库存售价覆盖为新 salePrice

### 3.3 出货 sale（quantity > 0，落库写负数）
- 追加一行到 `purchase_records.csv`（数量写 **-quantity**）
- 更新库存：`newQty = oldQty - quantity`
- 若 `newQty < 0`：返回 `NeedConfirmStockNegative`（默认不允许负库存；由 UI 确认是否继续）

### 3.4 改价 reprice（quantity = 0）
- 追加一行流水（quantity=0）可选（建议记录）
- 更新库存售价

### 3.5 盘点 adjust
- quantity 表示差值（可正可负）
- 更新库存数量（单位一致前提）

### 3.6 查询 query
- 不写入 CSV，不改库存
- 只展示 + TTS

---

## 4. DeepSeek 解析协议（JSON Output）

### 4.1 模型输出 JSON schema（必须严格解析）
```json
{
  "op": "purchase|sale|reprice|adjust|query",
  "name": "string",
  "quantity": 0,
  "unit": "条|包|瓶|箱|个|未知",
  "sale_price": null,
  "buy_price": null,
  "date": "YYYY-MM-DD",
  "confidence": 0.0,
  "notes": ""
}
```

### 4.2 System Prompt 模板（直接复制使用）
> 让 App 在 system prompt 或单独消息里注入 `today=YYYY-MM-DD`。

```text
你是一个进销存语音指令抽取引擎。你必须只输出 JSON（json_object），不要输出任何解释文字。

【数据表字段（非常重要）】
进货记录表列：名称,数量,单位,售价,进价,进货日期
库存记录表列：名称,数量,单位,售价,进价
因此你必须输出 unit（单位）。

【单位习惯（优先使用）】
- 烟：条 / 包 / 瓶
- 酒：箱
- 包装礼品（食品）：个
若用户未说单位：
- 按商品类型推断一个最可能单位；无法推断则 unit="未知" 并在 notes 说明需要确认。

【操作类型 op】
- purchase：进货
- sale：出货/卖出
- reprice：改售价（quantity=0）
- adjust：盘点/校正库存（quantity=差值或用户说的调整量）
- query：查询库存/询价（不落库）

【输出 JSON Schema（必须严格遵守，字段齐全）】
{
  "op": "purchase|sale|reprice|adjust|query",
  "name": "string",
  "quantity": number,
  "unit": "条|包|瓶|箱|个|未知",
  "sale_price": number|null,
  "buy_price": number|null,
  "date": "YYYY-MM-DD",
  "confidence": number,
  "notes": "string"
}

【解析规则】
1) name 尽量完整准确（如“红塔山硬”“可乐500ml”“某某礼盒”）。
2) quantity 只输出数字（把中文数字转阿拉伯数字：十=10，十二=12，一百=100）。
3) 用户说“今天/昨天/前天”要转换成 date（系统会传入 today=YYYY-MM-DD；昨天=today-1）。
4) query：quantity=0，sale_price/buy_price 为空。
5) 用户说“卖出/出货/出了”：op="sale"，quantity 输出正数，单位照说法输出。
6) 价格识别：
   - “售价/卖价/零售” -> sale_price
   - “进价/成本” -> buy_price
7) confidence：
   - 字段齐全且一致：>=0.80
   - 缺单位/缺数量/名称模糊：0.50~0.79，并在 notes 写明缺失点
   - 无法确定关键字段：<0.50

【示例】
输入：进货 红塔山硬 10条 进价18 售价22 今天
输出：{"op":"purchase","name":"红塔山(硬)","quantity":10,"unit":"条","sale_price":22,"buy_price":18,"date":"2026-02-19","confidence":0.88,"notes":""}

输入：卖出 红塔山硬 2包
输出：{"op":"sale","name":"红塔山(硬)","quantity":2,"unit":"包","sale_price":null,"buy_price":null,"date":"2026-02-19","confidence":0.82,"notes":""}

输入：查询 茅台 还有多少
输出：{"op":"query","name":"茅台","quantity":0,"unit":"箱","sale_price":null,"buy_price":null,"date":"2026-02-19","confidence":0.75,"notes":"未明确单位，按酒类默认箱推断；若店内按瓶管理请确认"}
```

---

## 5. “单位不一致确认页”组件说明（大字 UI 文案 + TTS 播报词）

> 这段可以原样复制给 Codex，让它实现 `UnitMismatchConfirmScreen`（或 `UnitMismatchDialogFullScreen`）。

### 5.1 触发条件
当 `InventoryService.applyTx(tx)` 返回：
- `NeedConfirmUnitMismatch(existingUnit, incomingUnit, itemSnapshot, tx)`  
则导航进入本页。

### 5.2 页面结构（Compose，建议全屏）
**页面标题（超大字，粗体）**
- 文案：`单位不一致，需要确认`

**核心提示区（大字、分段、易读）**
- 第一行：`商品：{name}`
- 第二行：`库存单位：{existingUnit}`
- 第三行：`你刚才说的单位：{incomingUnit}`
- 第四行（警示，仍用大字，不用小字）：
  - `为了避免库存混乱，请选择一种处理方式。`

**三按钮区（超大按钮，竖排，每个按钮高度>=72dp，间距>=12dp）**
1) 主按钮（推荐）：`按库存单位改写（推荐）`
   - 说明（按钮下方一行大字）：`把本次记录的“单位”改成 {existingUnit}（数量不变）`
2) 次按钮：`更新库存单位为 {incomingUnit}`
   - 说明：`把库存的单位改成 {incomingUnit}（数量不变）`
3) 危险/退出按钮：`取消并重说`
   - 说明：`不保存这次记录`

**底部辅助信息（可选，大字 20-22sp）**
- `提示：本 App 不自动换算条/包/箱/瓶，请按实际单位录入。`

### 5.3 TTS 播报词（进入页面后自动播报一次）
进入页面立刻播报（建议分句、语速稍慢）：
- `注意，单位不一致。`
- `库存中 {name} 的单位是 {existingUnit}。`
- `你刚才说的是 {incomingUnit}。`
- `请选择：一，按库存单位改写（推荐）。二，更新库存单位。三，取消并重说。`

> 若用户停留超过 8 秒无操作，可再次轻提示（可选）：  
> `请点击一个按钮继续。`

### 5.4 点击按钮后的行为（必须）
- 点击【按库存单位改写】：
  - 将 `tx.unit = existingUnit`（quantity 不变）
  - 返回 ConfirmScreen（或直接继续保存流程）
  - TTS：`已按库存单位改写，请确认保存。`
- 点击【更新库存单位】：
  - 将库存 `item.unit = incomingUnit`（qty 数字不变）
  - 返回 ConfirmScreen
  - TTS：`已更新库存单位，请确认保存。`
- 点击【取消并重说】：
  - 丢弃本次 tx，不写 CSV
  - 返回 MainScreen
  - TTS：`已取消，请重新说一遍。`

### 5.5 视觉/可用性要求（适老）
- 所有信息都必须在单屏可见（尽量不滚动）；若必须滚动，按钮固定在底部。
- 禁止仅用颜色表达风险：按钮需有明确文字（如“推荐”“取消”）。
- 不用小号灰字；行距至少 1.3 倍。

---

## 6. 工程结构建议（Codex 按此组织文件）

- `ui/theme/`：Typography、Dimens（按钮高度/间距）、ColorScheme
- `ui/screens/`：Main、Confirm、Inventory、History、Settings、UnitMismatchConfirm
- `data/csv/`：CsvCodec（转义/解析）、CsvRepository
- `domain/`：models、InventoryService、UseCases
- `llm/`：DeepSeekClient、prompt templates
- `speech/`：SpeechRecognizerController、TtsController
- `settings/`：SecurePrefs（EncryptedSharedPreferences）

---

## 7. 分步 Prompt 包（让 Codex 一步步落地，建议顺序执行）

> 用法：把每一步 Prompt 单独发给 Codex，让它“生成/修改工程”，确保能编译运行后再进入下一步。  
> 你要求的技术栈：Kotlin + Compose + OkHttp + kotlinx.serialization + EncryptedSharedPreferences（不引入额外复杂依赖）。

### Prompt A（总 System Prompt：放在最前面）
```text
你生成的是一个可安装到真机运行的 Android App（Kotlin + Jetpack Compose），面向 50+ 老花眼用户：
- 默认大字体（正文>=22sp，标题>=30sp，关键数字>=40sp）、大按钮（高度>=64dp）、高对比
- 语音优先交互：按住说话（SpeechRecognizer），每次关键结果必须 TextToSpeech 播报
- 本地数据只用 2 个 CSV（App 私有目录）：inventory.csv + purchase_records.csv（Excel 兼容 UTF-8 BOM）
- 写文件必须原子写入：写 tmp -> fsync/close -> replace，并保留 .bak
- 调用 DeepSeek API（OkHttp）把语音文本解析为 JSON 指令（kotlinx.serialization 严格解析）；错误要可恢复（重说/手动改/取消）
- API Key 必须用 EncryptedSharedPreferences 保存；不能硬编码；release 不打印敏感日志
输出必须是完整可编译运行的代码（不是伪代码），包含 Gradle 配置、Manifest 权限、运行所需的全部文件。
```

### Prompt 1（工程骨架 + 大字主题 + 导航 + 页面空壳）
```text
创建单模块 Android app（Kotlin + Compose + Material3 + MVVM + Coroutines）。
页面：MainScreen / InventoryScreen / ConfirmScreen / HistoryScreen / SettingsScreen。
要求：
1) Theme 高对比浅色；Typography：正文 22sp，标题 30sp，关键数字 40sp；按钮高度>=64dp。
2) MainScreen：底部中央一个超大按钮“按住说话”（先不接语音）。
3) 所有图标必须配文字（适老）。
4) Navigation Compose 路由完成，可运行到真机。
输出完整工程结构与关键代码。
```

### Prompt 2（CSV 数据层 + 原子写 + 业务规则 + 单测）
```text
在现有工程上实现本地 CSV 持久化（仅 App 私有目录 context.filesDir）。
CSV：
- purchase_records.csv 表头：名称,数量,单位,售价,进价,进货日期
- inventory.csv 表头：名称,数量,单位,售价,进价

要求实现：
A) CsvCodec：实现 CSV 读写（最小正确转义，支持引号/逗号/换行）。写入 UTF-8 BOM。
B) CsvRepository：
- loadInventory(): Map<String, Item>
- appendPurchaseRecord(row)
- saveInventoryAtomic(items)：写 inventory.tmp -> close -> replace inventory.csv；保留 inventory.bak
C) InventoryService.applyTx(tx):
- purchase/sale/reprice/adjust/query
- 单位不一致：返回 NeedConfirmUnitMismatch(existingUnit, incomingUnit,...)
- 出货导致负库存：返回 NeedConfirmStockNegative
- 成本：移动加权平均（仅单位一致且 buy_price 非空时更新）
D) UI 接入：ConfirmScreen 的“确认保存”按钮可触发保存（先用假 Tx），并在 HistoryScreen 展示流水。
E) 单元测试：加权平均、出货扣减、单位不一致分支、撤销最近一条（反向流水）
```

### Prompt 3（DeepSeekClient + JSON Output + 安全存 Key）
```text
接入 DeepSeek Chat Completions（OkHttp + kotlinx.serialization）。
要求：
1) DeepSeekClient.parseCommand(userText, today): ParseResult
- 使用 JSON Output：response_format={"type":"json_object"}
- system prompt 使用我提供的模板（包含单位习惯）
- temperature=0.1；超时、网络错误可恢复
2) 解析对象包含：op,name,quantity,unit,sale_price,buy_price,date,confidence,notes
3) SettingsScreen：用户输入 API Key，保存到 EncryptedSharedPreferences；无 key 时引导设置（大字提示+可播报）
4) 低 confidence / 字段缺失：进入 ConfirmScreen 的“大字表单模式”让用户改字段
输出完整代码与集成点。
```

### Prompt 4（SpeechRecognizer 按住说话 + TTS 确认闭环）
```text
实现语音优先闭环：
A) SpeechRecognizer：MainScreen 按住开始，松开结束，得到识别文本（大字显示）。
B) 调用 DeepSeekClient.parseCommand -> 导航到 ConfirmScreen。
C) ConfirmScreen 打开后 TTS 播报：“已识别：{op}{name}{quantity}{unit}… 要保存吗？”
D) 保存成功播报“已保存”；失败播报原因。
E) 权限：RECORD_AUDIO 运行时权限；拒绝/失败要“大字+TTS+重试/手动输入/取消”三个大按钮。
```

### Prompt 5（单位不一致确认页：按本文件第 5 节实现）
```text
实现 UnitMismatchConfirmScreen（全屏），触发于 NeedConfirmUnitMismatch。
UI 文案与 TTS 播报词严格按我提供的“单位不一致确认页组件说明”实现。
按钮动作：按库存单位改写 / 更新库存单位 / 取消并重说，并正确回到 Confirm 或 Main。
```

### Prompt 6（语音查询库存 query + 大字结果页）
```text
新增语音查询模式：当 DeepSeek 返回 op="query" 时：
- 打开 InventoryScreen，大字展示 商品名 + 库存数量（超大）+ 单位 + 售价/进价
- TTS 播报库存与售价
- 提供大按钮：出货 / 进货 / 改价（预填 name+unit 到 ConfirmScreen）
query 不写入 CSV、不改库存。
```

### Prompt 7（收尾：撤销最近一条 + 错误恢复 + 真机可用性）
```text
实现“撤销最近一条流水”（HistoryScreen 顶部一个超大按钮）：
- 读取最后一条 purchase_records 记录，生成反向记录追加进去，并重算库存（或 apply 反向 tx）。
- TTS 播报“已撤销上一条记录”。
另外完善：
- 首次启动自动创建 CSV 并写表头
- 全局错误提示大字化+TTS
- release 构建关闭敏感日志
```

---

## 8. 运行与验收清单（真机）
- 首次启动：自动生成两份 CSV（带表头）
- 语音说“进货 红塔山硬 10条 进价18 售价22 今天” → Confirm → 保存 → 库存增加 + 流水追加
- 语音说“卖出 红塔山硬 2包” 若库存单位=条 → 进入“单位不一致确认页”
- 语音说“查询 红塔山硬 还有多少” → InventoryScreen + TTS
- 撤销最近一条：库存回滚 + 流水追加反向记录

---

> 你可以把本文件直接交给 Codex，从 Prompt A 开始逐步执行。

---

## 9. GitHub Actions 使用说明（从零建仓库 → 自动编译 → 下载 APK）

> 目标：你本机没有 Android 开发环境，也能在每一步 Prompt 之后 **确保能编译**，并且 **下载 APK 到手机安装验证**。  
> 方法：把代码托管到 GitHub，使用 GitHub Actions 自动跑 `./gradlew testDebugUnitTest assembleDebug` 并上传 `app-debug.apk` 构建产物。

### 9.1 从零创建 GitHub 仓库（只需做一次）
1) 登录 GitHub → 右上角 `+` → `New repository`
2) 仓库名建议：`android-inventory-voice`
3) 选择 `Public` 或 `Private` 都可以（Private 也支持 Actions）
4) 勾选 `Add a README file`（可选，但建议勾上）
5) 点击 `Create repository`

### 9.2 把 Codex 生成的工程代码放进仓库（两种方式）
**方式 A：用 Git（推荐）**
1) 在你电脑任意位置打开终端：
   - `git clone <你的仓库地址>`
2) 把 Codex 生成的工程文件（含 `gradlew`、`settings.gradle`、`app/` 等）复制到仓库目录
3) 提交并推送：
   - `git add .`
   - `git commit -m "step1: project skeleton"`
   - `git push`

**方式 B：网页直接上传（没有 Git 也能用）**
1) 打开仓库页面 → `Add file` → `Upload files`
2) 拖拽整个工程文件夹内容上传（注意要包含 `gradlew`）
3) 填写 commit message → `Commit changes`

> 说明：网页上传适合初期，但后面文件多、频繁更新时，建议还是用 Git。

### 9.3 添加 GitHub Actions Workflow（只需做一次）
在仓库根目录创建文件：  
- `.github/workflows/android-ci.yml`

把下面内容原样粘贴进去：

```yaml
name: Android CI

on:
  push:
    branches: [ "main" ]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Set up Gradle
        uses: gradle/actions/setup-gradle@v3

      - name: Build & Unit Test
        run: |
          chmod +x gradlew
          ./gradlew --no-daemon testDebugUnitTest assembleDebug

      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: app-debug-apk
          path: app/build/outputs/apk/debug/*.apk
```

然后提交并推送（或网页提交）。

### 9.4 如何确认“编译通过”
1) 打开 GitHub 仓库 → 顶部点 `Actions`
2) 找到最新一次运行（Run）
3) 若显示绿色 ✅：表示 **编译与单测通过**（等价于你要求的“确保能编译运行（至少能编译）”）
4) 若失败（红色 ❌）：点进去看日志，通常是：
   - Gradle 版本/AGP/JDK 不匹配
   - 依赖版本冲突
   - Kotlin/Compose 语法错误
   - 单元测试失败

> 建议：你每完成一步 Prompt，就 push 一次。只有 Actions 绿了再进入下一步。

### 9.5 在哪里下载 APK（app-debug.apk）
1) 进入仓库 → `Actions`
2) 点击最新一次 **成功** 的 Run
3) 在页面下方找到 `Artifacts`
4) 下载 `app-debug-apk`
5) 解压后得到 `app-debug.apk`

### 9.6 把 APK 安装到手机上（验证“能运行”）
1) 把 `app-debug.apk` 发送到手机（微信/网盘/数据线/手机浏览器直接下载均可）
2) 手机设置中允许“安装未知来源应用”（不同品牌入口略有差异）
3) 点击 APK 安装
4) 打开 App 做最小冒烟检查（每一步 30 秒）：
   - 能启动不闪退
   - 页面大字、大按钮显示正常
   - 到语音那一步：麦克风权限弹窗正常、TTS 能播报
   - CSV 那一步：首次启动会生成两份 CSV（带表头）

### 9.7（可选）每一步的推荐 commit message
- `step1: project skeleton`
- `step2: csv repository + inventory service`
- `step3: deepseek client + secure api key`
- `step4: speech recognizer + tts flow`
- `step5: unit mismatch confirm screen`
- `step6: voice query inventory`
- `step7: undo last record + polish`

---
