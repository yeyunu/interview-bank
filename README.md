# interview-bank 面试题库

个人 Agent / RAG / MCP 面试题库，配合 ChatGPT Custom GPT「面试陪练」使用。每天新增的题目可以由 GPT Action 通过 GitHub Contents API 读取，适合在开车或碎片时间里语音抽问。

## 资料导航

| 目录 | 内容 | 使用场景 |
| --- | --- | --- |
| `questions/` | 按日期记录的每日新增题目 | 日常录题、抽当天题目 |
| `archive/Agent面试题_完整去重版.md` | 179 道去重、纠错后的完整问题及连续追问 | 历史抽题主入口 |
| `archive/腾讯PCG候选1_套MindClaw回答.md` | 8 道结合 MindClaw / ClawCodex 的回答稿 | 项目追问与表达训练 |
| `gpt/` | Custom GPT 的 Instructions 与 Action Schema | 配置面试陪练 |

更详细的历史资料说明见 [`archive/README.md`](archive/README.md)。

## 推荐复习顺序

1. 用完整去重版做系统复习和随机抽题。
2. 用腾讯 PCG 回答稿练习如何把问题落到自己的项目经验上。
3. 需要核对原始 OCR 或清洗过程时，回到本地源资料或 Git 历史。

## 仓库更新规则

- 当天新遇到、想到或需要反复训练的题目，写入 `questions/YYYY-MM-DD.md`。
- 从旧笔记、面经、OCR 或其他资料批量整理出的题库，写入 `archive/`，不要伪装成当天新增。
- `questions/` 中只保留日期文件；说明、索引和批量题库放在根目录或 `archive/`。
- 新增或重构历史题库后，同步更新 `archive/README.md`；如果影响 GPT 抽题入口，同时更新 `gpt/instructions.md`。
- 精简仓库只保留可直接使用的整理成果；原始资料保存在本地源目录和 Git 历史中。

## 每日题目格式

`questions/` 中一天一个文件，文件名固定为 `YYYY-MM-DD.md`。每题使用一个二级标题，下面记录答案要点：

```markdown
# 2026-07-16

## 题目：Agent 的规划能力如何评测？

- 要点 1：……
- 要点 2：……

## 题目：Function Calling 和 MCP 有什么区别？

- 要点 1：……
```

## 使用方式

- 平时：把当天遇到或想到的题追加到 `questions/当天日期.md`，然后提交并推送。
- 车上：打开 ChatGPT 的「面试陪练」GPT，进入语音模式，说“抽今天的题”或“抽历史题”。
- GPT 配置：将 [`gpt/instructions.md`](gpt/instructions.md) 和 [`gpt/action-schema.yaml`](gpt/action-schema.yaml) 分别填入 GPT Instructions 与 Actions。

## 资料来源

历史题库来自 `mindcodex/agent面试`，其中小红书面试资料的原始采集结果位于 `outputs/tikhub_xhs_interviews/20260707-102826/`。
