# interview-bank 面试题库

个人面试题库，配合 ChatGPT Custom GPT（面试陪练）使用：每天新增的题目会被 GPT 通过 Action 读取，开车时语音抽问。

## 目录规范

```
questions/          每日新增题目，一天一个文件
  2026-07-15.md     文件名固定为 YYYY-MM-DD.md
archive/            历史整理的完整题集（一次性迁入，不参与"每日新增"逻辑）
```

## questions/ 文件格式

每题一个二级标题，题目下写答案要点。GPT 会按 `##` 切分逐题提问。

```markdown
## 题目：Agent 的规划能力如何评测？

- 要点1：...
- 要点2：...

## 题目：function calling 和 MCP 的区别？

- 要点1：...
```

## 使用方式

- 平时（vibecoding 等待间隙）：把当天遇到/想到的题追加到 `questions/今天日期.md`，commit + push。
- 车上：打开 ChatGPT 的「面试陪练」GPT，进语音模式，说"抽今天的题"。
