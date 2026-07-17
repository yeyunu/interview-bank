# interview-bank 面试题库

个人 Agent / RAG / MCP 面试题库，配合 ChatGPT Custom GPT「面试陪练」使用。每天新增的题目可以由 GPT Action 通过 GitHub Contents API 读取，适合在开车或碎片时间里语音抽问。

## 资料导航

| 目录 | 内容 | 使用场景 |
| --- | --- | --- |
| `questions/` | 按日期记录的每日新增题目 | 日常录题、抽当天题目 |
| `archive/Agent面试题_按方向整理.md` | 67 道清洗题，按 RAG、Agent、工具、评估等方向分类 | 主复习材料，建议先看 |
| `archive/腾讯PCG候选1_套MindClaw回答.md` | 8 道结合 MindClaw / ClawCodex 的回答稿 | 项目追问与表达训练 |
| `archive/Agent面试题_宽松版_少漏题.md` | 233 条疑似题目或片段 | 查漏补缺 |
| `archive/Agent面试题_OCR全文.md` | 16 位候选、63 张图片的 OCR 全文 | 回溯原始上下文 |
| `gpt/` | Custom GPT 的 Instructions 与 Action Schema | 配置面试陪练 |

更详细的历史资料说明见 [`archive/README.md`](archive/README.md)。

## 推荐复习顺序

1. 用按方向整理版建立知识框架。
2. 用腾讯 PCG 回答稿练习如何把问题落到自己的项目经验上。
3. 用宽松版查漏补缺。
4. 只有发现语义不清或疑似漏题时，再回看 OCR 全文。

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
