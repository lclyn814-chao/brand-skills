---
name: brand
description: |
  品牌管理工作流总入口 — 三阶段架构：spawn 独立进程搜情报 → 主 Agent 做评估 → delegate_task 并行出四维分发。
  当用户提到"品牌全流程""品牌工作流""品牌一套走完""品牌谈判准备"时触发。
  有子 skill 具体描述触发关键词时，优先直接路由到对应子 skill。
version: 2.0.0
related_skills:
  - brand-deep-dive
  - brand-agency-intel
  - brand-3layer
---

# 品牌管理工作流 v2

## 设计原则

三种 Agent 调用方式，各司其职：

| 方式 | 工具范围 | 适合什么 | 不适合什么 |
|------|---------|---------|-----------|
| **spawn 独立进程**（terminal + hermes chat） | 全套工具，无限制 | 搜索、爬取、长任务 | 需要等结果才能继续的任务要加 timeout |
| **主 Agent**（你自己） | 全套工具 | 分析、汇总、决策 | 重复性的并行生成工作 |
| **delegate_task** | 只有你传的工具子集，无 web_search（结果会被摘要截断） | 纯生成任务（材料已齐） | 搜索、需要反问用户的任务 |

## 三阶段架构

```
用户输入：品牌名 + 品类 + 代理区域
                │
    ┌───────────┼───────────┐
    ▼           ▼           ▼
┌─────────┐ ┌─────────┐ ┌─────────┐
│spawn A  │ │spawn B  │ │spawn C  │   阶段1：搜索
│品牌深扒 │ │竞品深扒 │ │区域调研│   独立进程并行
│全工具✅ │ │全工具✅ │ │全工具✅ │   结果→文件
└────┬────┘ └────┬────┘ └────┬────┘
    │           │           │
    └───────────┼───────────┘
                ▼
    ┌─────────────────────┐
    │    主 Agent 汇总     │             阶段2：分析
    │ brand-agency-intel  │             主 Agent 自己跑
    │ 十维评估 + 打分     │             全工具✅
    └──────────┬──────────┘
               ▼
    ┌─────────────────────┐
    │  delegate_task batch │             阶段3：分发
    │  brand-3layer 四维   │             并行生成
    │  ①谈判 ②a对方 ②b我方│             无搜索✅
    │  ③终端               │
    └──────────┬──────────┘
               ▼
          全部就绪，上桌
```

## 阶段1：并行搜索（spawn 独立进程）

### 为什么不用 delegate_task

- 子 Agent 虽然可以带 `web_search` 工具，但搜索结果经 orchestrator 摘要层后会被截断
- 实测：delegate_task + web_search 不可用（2026-05-27，连续 6+ 次尝试全部失败）
- 唯一可行路径：spawn 独立 Hermes 进程，拥有全套工具

### 执行方式

**单品牌全流程（一条命令链）：**

```bash
# 同时启动三个后台进程，结果写到统一目录
mkdir -p ~/brand-research/{品牌名}/

# Agent A: 品牌深扒
terminal(
  command="hermes chat -q '加载 brand-deep-dive skill，深扒 {品牌名}（{品类}），代理区域 {区域}。结果保存到 ~/brand-research/{品牌名}/deep-dive.md' --timeout 600",
  background=true,
  timeout=600
)

# Agent B: 如果有明确竞品，同时跑竞品深扒
terminal(
  command="hermes chat -q '加载 brand-deep-dive skill，深扒 {竞品名}（{品类}），作为 {品牌名} 的对标参考。结果保存到 ~/brand-research/{品牌名}/competitor.md' --timeout 600",
  background=true,
  timeout=600
)

# Agent C: 区域市场调研
terminal(
  command="hermes chat -q '调研 {代理区域} 的 {品类} 市场现状：消费力、主流品牌、价格带、用户偏好、空白机会。结果保存到 ~/brand-research/{品牌名}/region.md' --timeout 600",
  background=true,
  timeout=600
)
```

**等待策略：**
- 用 `process(action='wait')` 等三个进程都结束，或 `process(action='poll')` 轮询进度
- 全部完成后，`read_file` 读取三份报告

### 如果竞品未知

先不启动 Agent B。Agent A 跑完从 deep-dive 中提取竞品名，再单独跑 Agent B。

## 阶段2：汇总分析（主 Agent）

三份报告到齐后，主 Agent 依次执行：

1. **read_file** 读取全部三份报告
2. **加载 brand-agency-intel**，按十维模型执行：
   - 综合三份报告的原始数据
   - 补齐品牌底盘、产品、客户、痛点、风格、舆情
   - 地域迁移适配（Agent C 的区域数据为核心输入）
   - 对标品牌差距分析（Agent B 的竞品数据为核心输入）
   - 企业风险扫描
   - 多维打分 + Go/No-Go 判断
3. 输出完整的代理谈判情报报告
4. 保存到 `~/brand-research/{品牌名}/agency-intel.md`

## 阶段3：四维分发（delegate_task 并行）

前两阶段产出齐了（deep-dive.md + agency-intel.md），进入纯生成阶段。四维之间互不依赖，用 delegate_task batch 并行。

```python
delegate_task(tasks=[
  {
    "goal": "生成 {品牌名} 谈判层速查卡",
    "context": "基于以下品牌情报，按 brand-3layer ①谈判层模板输出：\n{deep-dive.md 和 agency-intel.md 的核心数据摘要}",
    "toolsets": ["file"]
  },
  {
    "goal": "生成 {品牌名} 对方管理层心理建设包",
    "context": "基于以下品牌情报，按 brand-3layer ②a 模板输出。注意禁忌词：禁用'原单/尾货/仿品/品控不行'等表述。\n{核心数据摘要}",
    "toolsets": ["file"]
  },
  {
    "goal": "生成 {品牌名} 我方管理层内部对齐包",
    "context": "基于以下品牌情报，按 brand-3layer ②b 模板输出：财务预期、品牌矩阵关系、团队培训、风险前置、一句话决策。\n{核心数据摘要}",
    "toolsets": ["file"]
  },
  {
    "goal": "生成 {品牌名} 终端话术结构+互动脚本",
    "context": "基于以下品牌情报，按 brand-3layer ③终端层模板输出：客户5问+犹豫信号对照表+互动脚本+AI高销模板。\n{核心数据摘要}",
    "toolsets": ["file"]
  }
])
```

> ⚠️ `context` 中传的是数据摘要，不是"请读文件"。子 Agent 不能读文件路径——必须把核心数据直接写进 context。

主 Agent 收齐四个结果后，整合输出，保存为 `~/brand-research/{品牌名}/four-dimension.md`。

## 完整执行指令

用户只需一句：

> "品牌全流程：{品牌名}，{品类}，代理 {区域}"

触发后自动按三阶段执行：

1. **阶段1**：spawn 2-3 个独立 Hermes 进程并行搜索 → 等待全部完成 → 读取结果
2. **阶段2**：主 Agent 汇总 → brand-agency-intel 十维评估
3. **阶段3**：delegate_task batch → brand-3layer 四维分发

每阶段完成后告知用户进度，最终交付完整四维报告。

## 单步模式

用户只需某一步，直接触发对应子 skill：

| 用户说 | 执行 |
|--------|------|
| "扒一下XX" | 只跑 brand-deep-dive（spawn 独立进程） |
| "评估代理XX" | 只跑 brand-agency-intel（主 Agent） |
| "生成话术" / "拆四维" | 只跑 brand-3layer（前提材料已齐，可用 delegate_task） |

## 成本意识

| 阶段 | 消耗 | 何时用 |
|------|------|--------|
| 阶段1（spawn 进程） | 高（独立 LLM 调用 ×3） | 需要完整调研时 |
| 阶段2（主 Agent） | 中（一次长上下文调用） | 必有 |
| 阶段3（delegate） | 中（4 个子 Agent × 较短调用） | 需要完整四维输出时 |

**节省策略：**
- 如果已经有竞品数据，跳过 Agent B
- 如果是本土品牌（原区域=代理区域），跳过 Agent C 的区域差异部分
- 如果只需要谈判筹码、不需要终端话术，阶段3 只跑对应维度

## 注意事项

- **阶段1 的进程是异步的**——用 `process(action='wait')` 等，不要盲等
- **阶段2 依赖阶段1 的三份文件**——确认文件存在再开始
- **阶段3 的 delegate_task context 要传数据摘要，不传文件路径**——子 Agent 读不了文件
- **语言底线贯穿全程**：禁用"原单/尾货/仿品/跟单/品控不行/你们卖不动了"
- 数据可信度标注每一阶段都带，不丢失
