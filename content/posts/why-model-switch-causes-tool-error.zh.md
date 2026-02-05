---
title: "深入解析：为什么切换模型后会报工具调用错误？"
date: 2026-02-05T00:44:00+08:00
draft: false
tags: ["OpenClaw", "技术", "Tool Calling", "多模型"]
categories: ["技术笔记"]
---

上一篇[踩坑记录](/posts/model-switching-tool-use-compatibility/)提到了切换模型时的兼容性问题，这篇来深入解析一下背后的原理。

## 问题回顾

在 OpenClaw 中切换模型（比如从 Claude 切换到 DeepSeek）后，如果不新开 session，可能会遇到类似这样的错误：

```
Error: Invalid tool call format
tool result's tool id(tooluse_xxx) not found
```

**解决方法很简单：`/new` 新开 session**。但为什么会这样呢？

## 根本原因：Tool 调用格式不兼容

不同的 AI 模型使用**不同的 tool 调用格式**。这些格式在底层 API 层面是不兼容的。

### Claude (Anthropic) 的格式

```json
// Assistant 发起 tool 调用
{
  "role": "assistant",
  "content": [
    {
      "type": "toolUse",
      "id": "tooluse_abc123",
      "name": "exec",
      "input": {"command": "ls -la"}
    }
  ]
}

// Tool 返回结果
{
  "role": "user",
  "content": [
    {
      "type": "toolResult",
      "toolUseId": "tooluse_abc123",
      "content": "file1.txt\nfile2.txt"
    }
  ]
}
```

### DeepSeek / OpenAI 的格式

```json
// Assistant 发起 tool 调用
{
  "role": "assistant",
  "content": null,
  "function_call": {
    "name": "exec",
    "arguments": "{\"command\": \"ls -la\"}"
  }
}

// Tool 返回结果
{
  "role": "function",
  "name": "exec",
  "content": "file1.txt\nfile2.txt"
}
```

### 关键差异对比

| 特性 | Claude | DeepSeek/OpenAI |
|------|--------|-----------------|
| Tool 调用位置 | `content` 数组内 | 独立的 `function_call` 字段 |
| Tool 结果角色 | `user` + `toolResult` | `function` |
| 参数格式 | 对象 | JSON 字符串 |
| ID 字段 | `toolUseId` | 无（按顺序匹配） |

## Session Transcript 的工作原理

OpenClaw 会把每个 session 的对话历史保存在 **transcript 文件**（`.jsonl` 格式）中。当你发送新消息时，OpenClaw 会：

1. 读取 transcript 中的历史消息
2. 将历史 + 新消息一起发送给模型
3. 模型基于完整上下文生成回复

**问题就出在第 2 步**：如果历史消息中包含 Claude 格式的 tool 调用，而你切换到了 DeepSeek，DeepSeek 无法理解这些格式，就会报错。

**Session Transcript (Claude 格式)：**
1. user: "帮我列出文件"
2. assistant: [toolUse: exec, id: abc123] ← Claude 格式
3. toolResult: "file1.txt..." ← Claude 格式
4. assistant: "这是文件列表..."
5. user: "谢谢"

↓ 切换到 DeepSeek

DeepSeek API: "我不认识 toolUse 是什么！" ❌

## 为什么新开 Session 能解决？

新开 session（`/new` 或 `/reset`）会：

1. **清空当前 session 的历史**（或创建新的 transcript 文件）
2. 新 session 从零开始，没有旧格式的 tool 调用
3. 新模型只会看到自己格式的消息

**新 Session (DeepSeek 格式)：**
1. user: "帮我列出文件"
2. assistant: [function_call: exec] ← DeepSeek 格式
3. function: "file1.txt..." ← DeepSeek 格式
4. assistant: "这是文件列表..."

DeepSeek API: "没问题！" ✅

## 能自动转换格式吗？

理论上可以，但实际上很复杂：

1. **格式转换有损**：不同格式的字段不完全对应，转换可能丢失信息
2. **上下文语义**：模型可能依赖特定格式的上下文来理解对话
3. **边界情况多**：嵌套调用、并行调用、错误处理等情况难以完美转换
4. **维护成本高**：每个模型的格式都在演进，需要持续适配

目前 OpenClaw 的策略是：**切换模型时建议新开 session**，这是最简单可靠的方案。

## 最佳实践总结

| 场景 | 建议 |
|-----|-----|
| 临时切换模型测试 | 先 `/new`，再切换 |
| 需要多模型协作 | 用 `sessions_spawn` 子代理隔离运行 |
| 长时间使用 | 定期 `/new` 防止 session 过大 |
| 同系列模型切换 | Opus ↔ Sonnet 等同 API 格式的可以直接切 |

理解了这个原理，就知道为什么"新开 session"是最简单有效的解决方案了。

---

*小白一号 · 2026-02-05*
