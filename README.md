# Meme Coin 投研分析 Skill

> 输入一个 CA（合约地址），输出一份固定格式的投研报告。

## 核心理念

```
meme coin 价值 = 叙事驱动的注意力定价
链上数据是结果，不是原因
```

这个 skill 不做喊单 bot 的事。它帮你回答：**叙事是什么、传播到哪了、注意力值多少钱、当前处于什么态势。**

## 安装

```bash
# 复制 SKILL.md 到你的 CodeBuddy skills 目录
mkdir -p ~/.codebuddy/skills/meme-coin-research
cp SKILL.md ~/.codebuddy/skills/meme-coin-research/
```

## 前置依赖

本 skill 依赖以下两个 skill 提供数据采集：

| Skill | 仓库 | 用途 |
|-------|------|------|
| **GMGN Skills** | [GMGNAI/gmgn-skills](https://github.com/GMGNAI/gmgn-skills) | 链上数据：代币信息、持仓、Dev历史、安全检测 |
| **Agent-Reach** | [Panniantong/agent-reach](https://github.com/Panniantong/agent-reach) | 社交数据：Twitter、Reddit、全网搜索、跨平台追踪 |

请确保这两个 skill 已在你的环境中安装并配置好。

## 使用

```
/analyze <CA>     → 完整投研报告（8个章节）
/quick <CA>       → 快速定性（叙事类型 + 安全 + 态势一句话）
```

也可以自然语言触发：`分析这个meme`、`帮我看看这个币`，后跟 CA 地址。

## 研报结构

| 章节 | 内容 |
|------|------|
| 🔒 安全检查 | 6条一票否决清单 |
| 📖 叙事分析 | A-F六类分类 + 强度评分 + 生命周期 |
| 📡 传播链 | L0-L6传播层级 + KOL图谱 + 传播质量 |
| 📈 注意力定价 | 对标找天花板 + 折扣计算 |
| 📊 盘面数据 | 核心指标 + 脏盘评估 |
| 🔗 叙事→资金验证 | 5个正向/负向信号 |
| 🎯 态势判断 | 一句话结论 + 预期天花板 + 风险 |

## 设计文档

详见 [DESIGN.md](./DESIGN.md) — 包含完整的设计大纲、专家分析汇总、六类叙事判断框架、pump.fun 内外盘博弈分析等。
