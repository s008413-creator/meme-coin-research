---
name: meme-coin-research
description: Meme Coin 投研分析 — 输入 CA 地址，输出固定格式研报。叙事驱动+注意力定价+传播链分析框架。
triggers:
  - 分析这个meme
  - meme研报
  - 投研
  - /analyze
  - 看看这个币
  - meme coin research
  - 帮我分析
---

# Meme Coin 投研分析

输入一个 CA（合约地址），输出一份固定格式的投研报告。

## 触发方式

用户提供 CA 地址并要求分析时自动触发。典型说法：
- "分析这个meme: `<CA>`"
- "/analyze `<CA>`"
- "帮我做一下 `<CA>` 的投研"
- "/quick `<CA>`" — 快速定性

---

## 前置依赖

本 skill 需要以下命令行工具已安装并可调用：

| 工具 | 安装方式 | 验证 |
|------|---------|------|
| `gmgn-cli` | `npm install -g gmgn-cli` | `gmgn-cli --version` |
| `twitter` | 通过 Agent-Reach：`pipx install twitter-cli`（**包名**是 `twitter-cli`，但**命令名**是 `twitter`） | `twitter --version` |

以及以下辅助工具（通常系统自带或 Agent 环境已有）：
- `curl` — HTTP 请求（读网页/推文）
- 搜索引擎/Exa — 全网搜索（Agent 环境通常内置）

> ⚠️ **命令名陷阱**：Agent-Reach 的 Twitter 工具 `pipx install` 的包叫 `twitter-cli`，但注册出来的可执行命令是 `twitter`（不是 `twitter-cli`）。本文件所有示例统一用 `twitter`。

---

## 核心认知

```
meme coin 价值 = 叙事驱动的注意力定价
链上数据是结果，不是原因
执行铁律：link字段 → 传播链 → 盘面数据
不能因为链上脏就 pass，脏也要列出来
```

---

## 执行流程

收到 CA 后，严格按以下 3 轮执行。不跳步，不颠倒，不来回确认。

### 第 1 轮：并行数据采集

**同时**执行以下查询。注意所有命令在 shell 中执行，解析 JSON/文本输出提取所需字段。

> **链识别（先确认 `--chain` 参数）**：`gmgn-cli` 的 `--chain` 取值为 `sol` / `bsc` / `base` / `robinhood`。
> - **用户已明确指定链**：直接用用户指定的链（例如用户说"这是 robinhood 链的"→ 立刻用 `robinhood`）。**绝不要覆盖用户的指定**，也不要另起炉灶去试别的链。
> - **Solana**：CA 为 base58 编码，约 44 字符，不含 `0x` → 用 `sol`。
> - **0x 开头的地址**：可能是 **BSC / Base / Robinhood 三者之一**——它们的代币地址都是 `0x` + 40 位十六进制格式（**牢记：0x ≠ 仅 BSC/Base**）。判定顺序：
>   1. 若用户或来源（DexScreener 等）标注了具体网络 → 直接用该网络。
>   2. 若无法判定 → **依次试 `robinhood` → `bsc` → `base`**。（gmgn 对 robinhood 的支持常被忽略，但它是 0x 地址且 gmgn 原生支持，**必须优先探测**；命中标志：`token info` 返回了 `symbol`/`name` 等真实字段。）
> - **Ethereum 主网 / Arbitrum / Avalanche / Optimism / Polygon 等**：`gmgn-cli` **不支持**。不要做 RPC 探测、不要查 Etherscan、不要做多链扫描——既无解又浪费时间。这些链上的 meme 不在本 skill 覆盖范围内。
> - **判定成功的唯一标准**：`gmgn-cli token info` 返回了 `symbol`/`name`/`price` 等真实字段即命中；返回空或报错则换下一链重试。
>
> 下文命令中的 `--chain sol` 仅为示例，务必按实际 CA 替换为对应链。

#### ① 代币信息（gmgn-cli）

```bash
gmgn-cli token info --chain sol --address <CA>
```

必须提取以下字段：

| 字段 | 用途 |
|------|------|
| `link.twitter_username` | ★ 叙事钥匙 — 可能是推文路径、用户名、或空 |
| `link.website` | ★ 叙事钥匙 — 可能是新闻链接、trending 链接、或空 |
| `price` / `market_cap` / `ath` | 盘面数据 |
| `creation_timestamp` | 创建时间（UTC → 北京时间 +8h） |
| `holders_count` | 持币地址数 |
| `top_10_holder_rate` | Top 10 持仓占比 |
| `dev_hold_rate` | Dev 持仓占比 |
| `bundler_trader_amount_rate` | Bundler 占比 |
| `rat_trader_amount_rate` | 老鼠仓比率 |
| `sniper_count` | 狙击手数量 |
| `smart_degen_count` | 聪明钱数量 |
| `is_on_curve` | 是否还在 bonding curve |
| `rug_risk_score` | Rug 风险评分 |
| `honeypot_detected` | 蜜罐检测 |
| `mint_authority_revoked` | Mint 权限是否放弃 |
| `freeze_authority_revoked` | Freeze 权限是否放弃 |

#### ② Dev 历史（gmgn-cli）

从 token info 输出中提取 dev 钱包地址后执行：

```bash
gmgn-cli portfolio created-tokens --chain sol --wallet <DEV_WALLET>
```

提取：历史币数量、每个币的最高MC、存活时间、是否 rug。

#### ③ 读原始叙事（根据 link 字段分四种情况）

**情况①：twitter_username 是推文路径**
```
例: "0xScimpy/status/2053306587721252940"
→ 提取末尾数字作为 tweet_id
→ 执行: twitter tweet <tweet_id>
→ 获取：推文正文、作者粉丝数、likes/RTs/replies/views、评论区内容
```

**情况②：twitter_username 是正常用户名**
```
例: "ScamAltmanCoin"
→ 执行: twitter user <username>
→ 获取：账号创建时间、粉丝数、最近推文列表
```

**情况③：website 是无厘头链接**
```
例: "https://www.dailymail.co.uk/news/xxx"
→ 执行: curl -sL "https://r.jina.ai/<URL>"（Jina Reader，免费）
→ 或直接用内建网页阅读能力
→ 从网页内容中提取真正的叙事来源
```

**情况④：空值**
```
→ 标注"无推特/网站"
→ 跳过此步，但后续必须用全网搜索弥补
```

> ⚠️ link 字段是叙事的钥匙。跳过一个字段就可能 miss 一个 20x。

#### 聪明钱 / KOL 链上取证（第 1 轮补充）

> **链支持**：仅 `sol` / `bsc` / `base` / `eth` 支持；`robinhood` 链 gmgn 不提供 track 接口 → 跳过本步，在报告标注"链不支持，聪明钱/KOL 链上取证缺失"。
> **拿地址的正确姿势**：`token info` 的 `holders` 数组**恒为空**、`wallet_tags_stat` 只给数量，**不能从中提取钱包地址**。必须拉 `track` 交易流，反查 `base_address == <CA>` 的 `maker` 钱包。

**曾经持仓（历史交易）**——拉该链最近的聪明钱 / KOL 交易流，筛出涉及本 CA 的记录：

```bash
# 聪明钱交易流 → 筛 base_address == <CA>
gmgn-cli track smartmoney --chain <chain> --limit 200 --raw | python3 -c "
import sys,json
ca='<CA>'.lower()
for r in json.load(sys.stdin).get('list',[]):
    if str(r.get('base_address','')).lower()==ca:
        print('SM', r.get('maker'), r.get('side'), 'usd=%.0f'%r.get('amount_usd',0), 't=%d'%r.get('timestamp',0), 'close=%s'%r.get('is_open_or_close'))
"
# KOL 交易流 → 同上筛选（把 smartmoney 换成 kol）
gmgn-cli track kol --chain <chain> --limit 200 --raw | python3 -c "
import sys,json
ca='<CA>'.lower()
for r in json.load(sys.stdin).get('list',[]):
    if str(r.get('base_address','')).lower()==ca:
        print('KOL', r.get('maker'), r.get('side'), 'usd=%.0f'%r.get('amount_usd',0), 't=%d'%r.get('timestamp',0), 'close=%s'%r.get('is_open_or_close'))
"
```

提取：哪些聪明钱 / KOL 钱包交易过本币、买卖方向（buy/sell）、金额、时间、是否平仓（`is_open_or_close`=1 表示平仓）。

**当前持仓**——对上面出现的关键钱包（尤其买入且未平仓的），查其在本币上的当前余额：

```bash
gmgn-cli portfolio token-balance --chain <chain> --wallet <WALLET_ADDR> --token <CA>
```

返回该钱包当前持有本币的数量 / 价值；空值或 0 表示已清仓（`robinhood` 链此处 `--chain` 可照常使用）。

**分析要点**：
- 聪明钱 / KOL 是**建仓加仓**（真看涨背书）还是**清仓跑路**（看跌信号）？
- 建仓成本区间、当前盈亏——亏损硬扛 vs 盈利落袋，含义相反
- **社交喊单 vs 链上背离**：KOL 在 Twitter 喊单但链上钱包零持仓 → 纯嘴炮无真金白银，可信度打折（呼应 caller 互割信号）
- 多少比例的聪明钱钱包"用脚投票"离场 = 撤退强度

### 第 2 轮：传播链深度分析

#### ④ Twitter 搜索

```bash
twitter search "<TOKEN_NAME>"
twitter search "$<TICKER>"
```

从搜索结果中分析并回答：

- 讨论量有多少？趋势是上升还是下降？
- 有哪些 KOL 在讨论？（记录粉丝量级、发帖内容质量）
- 讨论是自然传播还是付费推广？
  - **付费信号**：多个中型 KOL（10k-50k粉）在 30 分钟内集中发帖；不同 KOL 用相同框架和关键词
  - **Caller 互割信号**：KOL 发帖 5-15 分钟后出现大额卖单；发帖后 holder 增但 MC 不涨
- 有没有形成争议/对立面？（有争议的传播更快）
- 评论区的质量如何？（真人在讨论 vs 机器人刷 "LFG 🚀🌕"）

#### ⑤ 全网搜索

使用 Agent 内建的搜索能力，搜索：
- `"<TOKEN_NAME> meme coin"`
- `"<TOKEN_NAME> news"`
- `"<TICKER> token"`

获取：是否有传统媒体报道、是否有新闻覆盖、叙事是否在破圈。

#### ⑥ 跨平台检测（代币已存在 >6h 时执行）

- Reddit：搜索 `r/CryptoMoonShots` 等板块
- B站/小红书：检测中文社区传播
- 判断是否已跨文化传播

### 第 3 轮：综合分析 → 输出固定格式研报

基于前两轮的所有数据，按以下框架执行 LLM 分析。**严禁跳过任何章节。**

---

## 分析框架

### 一、安全检查（分两级，勿混淆）

> ⚠️ **链差异必读**：
> - **Solana 链**：检查 Mint Authority / Freeze Authority 是否 revoke。
> - **EVM 链（BSC / Base / Robinhood）**：**不存在 Mint/Freeze Authority 概念**，这两项为 `N/A`（非失败、非通过，直接标 N/A）。EVM 改查：合约 Owner 是否 renounce、是否仍可 mint、是否 honeypot、税率是否可改。
> - **Dev 持仓低是正向信号**：`dev_hold_rate` < 5% 表示 Dev 几乎不持币、砸盘风险低，**不是安全检查失败项**；仅当 > 10% 或未锁定且集中时才计入风险。

逐条检查，分两级处理：

**🔴 硬否决（任一命中 → 标记"安全不通过"，只输出安全检查章节，不输出后续）**：
```
☐ H1. Honeypot 检测命中（买得进卖不出）— 无例外
☐ H2. [Solana] Mint Authority 未 revoke — 无例外
☐ H3. [Solana] Freeze Authority 未 revoke — 无例外
☐ H4. [EVM] 合约 Owner 仍可 mint / 未 renounce 且保留铸币函数 — 无例外
☐ H5. 有未披露的 Transfer Fee 修改权限（Dev 可随时改成 99% 税率）— 无例外
☐ H6. 前 3 地址（排除CEX/DEX/销毁）持仓 > 30% 且未锁定 — 无例外
☐ H7. Dev 在创建后 24h 内卖出 > 初始持仓 10%
```

**🟡 软警告（命中 → 继续完整分析，但在安全检查章节标红 + 态势判断中提示风险）**：
```
☐ S1. LP 未锁定/未销毁 — 常见但需提示（Dev 可撤池 Rug）
☐ S2. Dev 持仓偏高（> 10%）或集中未锁定
☐ S3. 税率 > 5% 或未明确
```

> **只有 H1–H7 任一命中才"只输出安全章节"**；S1–S3 命中不阻断后续分析。

### 二、叙事分类与判断

先判断属于哪种类型（只能一种），再套对应框架：

**A. 事件驱动 — 已发生**
- 事件距今多久？（0-24h 极强 / 1-3天 强 / 3-7天 中 / 7天+ 弱）
- 主流媒体覆盖量？
- 独立传播 hook 有几个？
- 名字与事件的关联直觉明显吗？
- 对标：同类事件之前的 meme 跑到多少 MC？

**B. 事件驱动 — 预期炒作**
- 事件发生的确定性？会不会取消？
- 距离事件还有多久？1-2周最佳
- 全球关注度？
- 同事件有多少个盘在竞争？
- 有没有"利好出尽"风险？

**C. 文化梗型**
- 梗新鲜度？（第一次 vs 翻炒）
- 理解门槛？（越低越好）
- 视觉符号强度？（能做表情包？48x48 可辨识？）
- 二创空间？（能被 remix 吗？）
- 原生社区与 crypto 用户重叠度？
- 历史上有无同类成功先例？

**D. 情感叙事型**
- 触发哪种原始情绪？（保护弱小/逆袭/愤怒/搞笑）
- 有无具体情感载体（人物/动物）？
- 跨文化共鸣度？
- 首爆平台？（TikTok > Twitter > Reddit）

**E. 技术包装型**
- 剥掉包装，底层叙事是什么？
- 与技术事件的时间距离？
- 命名竞争密度？
- 需要持续交付维持信仰吗？

**F. 纯 Meme 型**
- 名字够短够顺口？ticker 念出来顺吗？
- 0.5 秒能 get 到吗？
- 视觉够强吗？
- 有口号/slogan 吗？
- 历史先例？

### 三、传播链分析

1. **源头**：谁第一个发的？粉丝量级：K(<10K) / 万级(10-100K) / 十万级(100K-1M) / 百万级(1M+)
2. **传播层级（L0-L6）**：
   - L0-L1: 私域发酵（4chan/Reddit/Discord）
   - L2: 早期传播 — 小KOL发帖 ★ 最优窗口
   - L3: 叙事放大 — 中型KOL跟进
   - L4: 引爆 — 大KOL集中讨论
   - L5: FOMO — 头部KOL跟进 ⚠ 接盘风险
   - L6: 破圈/衰退
3. **传播量级**：views 总量、views/followers（>1=出圈）、大V接力层数
4. **传播质量**：自然传播 vs 付费推广 vs caller互割
5. **争议驱动**：有无对立面（加速传播 1.5-2x）
6. **持续性**：一次性 vs 持续讨论？有无第二波催化剂？

### 四、阶段判定

| 阶段 | 条件 |
|------|------|
| 内盘早期 | is_on_curve=true, MC < $30K |
| 内盘中段 | is_on_curve=true, MC $30K-$69K |
| 毕业窗口 | 刚毕业(<5min) 或即将毕业 |
| 外盘早期 | 已毕业, 5-30min |
| 外盘趋势 | 毕业后 30min-6h |
| 外盘后期 | 毕业后 6h+ |

毕业窗口额外判断：
- 自然买盘推动 → 毕业后大概率继续涨
- 1-3 个钱包硬拉 → 毕业即巅峰
- Dev 自己拉 → 短期波动大，有 DexScreener 流量但风险高

### 五、注意力定价

```
步骤1: 找对标 — 同类叙事的历史天花板 MC 是多少？
步骤2: 打折 — 源头权威性、传播层级、持续力、竞争密度各打折扣
步骤3: 对照当前 MC：
  - 当前 MC << 预期 → 定价错误，有空间
  - 当前 MC ≈ 预期 → 性价比一般
  - 当前 MC >> 预期 → 已透支
```

### 六、筹码与验证

**脏盘量化**（所有脏东西加在一起）：
- Dev 持仓 ___%（可接受 ≤5%，致命 >10%）
- 狙击手/Bundler ___%（可接受 ≤20%，致命 >30%）
- 老鼠仓 ___%（可接受 ≤3%，致命 >10%）
- Top 10（排除已知地址）___%（可接受 ≤30%）
- **总量** ___%（总原则：不超过 30-35%）

**叙事→资金验证**：
- 正向：新地址净买入>0、持币时间增加、聪明钱叙事后 2-6h 入场
- 负面：喊单激增但交易量不涨、热度高但 holder 不涨、MC 涨但持币地址降

---

## 输出格式（⚠️ 强制执行，不可简化、不可替代）

> **绝对禁止**输出口语化摘要、对话式总结、要点列表等非模板格式。
> 无论分析过程多复杂、数据多或少，最终输出**必须且只能**是以下 Markdown 研报模板。
> 不允许省略任何章节（安全不通过时除外），不允许用"以下是简要结论"代替完整报告。
> 模板中的 `{占位符}` 替换为实际数据，`数据不足` 标注缺失项，绝不编造。

```markdown
# $TICKER 投研报告
> {一句话核心叙事钩子 — 为什么能起来}

---

## 🔒 安全检查

> EVM 链（BSC/Base/Robinhood）：Mint/Freeze Authority 标记为 `N/A`（概念不适用）。
> Dev 持仓低（<5%）为 🟢 正向，不计入失败。

**🔴 硬否决项**：
| 检查项 | 状态 |
|--------|------|
| Honeypot 检测 | ✅/❌ |
| [Sol] Mint 权限放弃 / [EVM] 合约可 mint | ✅/❌/N/A |
| [Sol] Freeze 权限放弃 / [EVM] 可 pause | ✅/❌/N/A |
| 税率可修改 | ✅/❌ |
| Top3 持仓集中（>30% 未锁定） | ✅/❌ |
| Dev 早期卖出（>10%） | ✅/❌ |

**🟡 软警告项**：
| 检查项 | 状态 |
|--------|------|
| LP 锁定/销毁 | ✅/❌ |
| Dev 持仓偏高（>10%） | ✅/❌ |
| 税率偏高（>5%） | ✅/❌ |

**结论**：✅ 通过 / 🟡 有软警告（继续分析） / ❌ 硬否决（{具体原因}，仅输出本章节）

---

## 📖 叙事分析

**类型**：{A/B/C/D/E/F} — {类型名称}
**强度评分**：{xx}/100
**生命周期**：{萌芽/发酵/爆发/成熟/衰退}

**叙事拆解**：
- 来源：{谁发的、粉丝量级、互动数据}
- 传播逻辑：{源头→接力→量级，大V接力层数}
- 情绪驱动：{人群共鸣点、争议性}
- 护城河：{同叙事竞争格局、独立传播引擎判断}

---

## 📡 传播链

**当前层级**：L{x} — {层级名称}
**传播量级**：{强/中/弱}，{具体数据支撑}
**传播质量**：{自然传播/付费推广/caller互割}，{判断依据}
**跨文化状态**：{仅英文CT / 已传到中文CT / 已传到韩文CT / 已破圈}

---

## 📈 注意力定价

**对标项目**：{同类叙事标杆}，天花板 MC {金额}
**折扣因子**：{源头折扣 + 传播折扣 + 持续力折扣 + 竞争折扣}
**预期天花板**：{金额}
**当前 MC**：{金额}
**判断**：{定价错误有空间 / 性价比一般 / 已透支}

---

## 📊 盘面数据

| 指标 | 数值 |
|------|------|
| 当前 MC | {金额} |
| ATH | {金额} |
| 创建时间 | {UTC时间}（{相对时间}前，北京时间 {CST时间}） |
| 阶段 | {内盘早期/内盘中段/毕业窗口/外盘早期/外盘趋势/外盘后期} |
| Holder 数 | {数量} |
| Dev 持仓 | {百分比} |
| Bundler 占比 | {百分比} |
| 老鼠仓比率 | {百分比} |
| Top 10 占比 | {百分比}（排除已知地址后） |
| 聪明钱数量 | {数量} |

**脏盘评估**：{总量}% — {可接受/偏高/致命}

---

## 💡 聪明钱 / KOL 链上取证

> 若链不支持（robinhood）或 track 流中无本币记录，标注"数据不足"，不编造。

**曾经持仓（历史交易）**：
| 钱包 | 类型 | 方向 | 金额 | 时间 | 平仓 |
|------|------|------|------|------|------|
| {addr} | 聪明钱/KOL | buy/sell | {$} | {相对时间} | 是/否 |

**当前持仓**：
| 钱包 | 类型 | 当前持仓 | 占本币比 | 盈亏 |
|------|------|---------|---------|------|
| {addr} | 聪明钱/KOL | {数量/$} | {%} | {盈/亏/%} |

**结论**：{建仓加仓看涨 / 清仓跑路看跌 / 社交喊单但链上零持仓（嘴炮） / 数据不足}

---

## 🔗 叙事→资金验证

| 信号 | 状态 |
|------|------|
| 新地址净买入 | ✅/❌ |
| 持币时间增长 | ✅/❌ |
| 聪明钱/KOL 链上取证印证（建仓/清仓） | ✅/❌ |
| 喊单≠交易背离 | ✅/❌ |
| 热度≠holder背离 | ✅/❌ |

**验证结论**：{叙事正在转化为买盘 / 叫好不叫座 / 数据不足}

---

## 🎯 态势判断

**{一句话态势}**

{具体说明：龙头已锁定 / 独立叙事空间 / 跟风盘 / 叙事无升级空间 / 等待第二波催化剂 / 叙事被证伪 / 叙事被更强替代}

**预期天花板**：{金额}
**核心风险**：{最关键的风险点，不超过两条}
```

---

## 格式规范

- 金额用 K/M/B：`$12.3K`、`$1.2M`、`$45M`
- CA 纯文本，不发链接
- 时间用相对时间 + 北京时间（CST）：`3小时前（北京时间 14:30）`
- 不用"首先/其次/最后/此外"等过渡词
- 善用加粗、表格、分割线区分层级
- 百分比精确到小数点后一位
- 缺失数据标注 `数据不足`，不编造，不留空
- **安全不通过时只输出安全检查章节**

---

## 反直觉提醒

分析时必须记住以下经验：

1. 链上脏不是 pass 理由 — Bot/Bundler 高可能是做市商在铺流动性
2. Bot 率突然下降比 Bot 率高更危险 — 说明做市商撤了
3. TG/Discord 刷屏越快越危险 — 大户不在里面聊天
4. CEX 上币往往是顶部 — 只有 Binance 现货是例外
5. "涨太多了"不是合理判断 — 用注意力定价，不用价格否定叙事
6. 负面 FUD 可能是买入信号 — 洗纸手 + 筹码集中到信仰者
7. 英文 CT → 中文 CT 时，早期玩家通常在考虑出货
8. 越"傻"越传播 — 0.5秒能get的梗比需要3秒的梗传播力强10倍
9. 争议是传播燃料 — 有对立面的叙事传播快3-5倍
10. 内盘看速度，外盘看叙事 — 不在错误的阶段用错误的逻辑

---

## 安装

本 skill 是一个纯 Markdown 文件，放入 Agent 的 skills 目录即可：

```bash
# WorkBuddy（本机）
mkdir -p ~/.workbuddy/skills/meme-coin-research
cp SKILL.md ~/.workbuddy/skills/meme-coin-research/

# Hermes
mkdir -p ~/.hermes/skills/meme-coin-research
cp SKILL.md ~/.hermes/skills/meme-coin-research/

# 其他 Agent — 放入对应 skills 目录
```

前置工具：
```bash
npm install -g gmgn-cli        # 链上数据（已装则跳过）
pipx install twitter-cli        # Twitter 查询（Agent-Reach）；包名 twitter-cli，命令名 twitter
```

GMGN API Key 配置：
```bash
# 无需手动 export。gmgn-cli 会自动读取 ~/.config/gmgn/.env 中的 GMGN_API_KEY。
# 该 .env 可能同时含 GMGN_PRIVATE_KEY（交易私钥）。研究用途不需要私钥。
# ⚠️ 切勿将 .env（或任何含私钥的内容）提交到公开仓库。建议在仓库根目录加 .gitignore：
#     .env
#     .env.*
#     *.env
#     *.pem
#     *.key
# 并用 .env.example 占位（全为假值）替代真实密钥。
```

GMGN API Key 如果尚未配置，去 GMGN 官网获取后写入：
```bash
mkdir -p ~/.config/gmgn
printf 'GMGN_API_KEY=your_api_key_here\n# 研究用途请勿填入真实私钥\nGMGN_PRIVATE_KEY=\n' > ~/.config/gmgn/.env
```
