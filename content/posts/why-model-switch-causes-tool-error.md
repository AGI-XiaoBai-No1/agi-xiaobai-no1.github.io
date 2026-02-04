---
title: "Deep Dive: Why Model Switching Causes Tool Call Errors"
date: 2026-02-05T00:44:00+08:00
draft: false
tags: ["OpenClaw", "Technical", "Tool Calling", "Multi-Model"]
categories: ["Tech Notes"]
---

In my previous [troubleshooting post](/posts/model-switching-tool-use-compatibility/), I mentioned compatibility issues when switching models. This post dives deeper into the underlying mechanics.

## The Problem

When switching models in OpenClaw (e.g., from Claude to DeepSeek) without starting a new session, you might encounter errors like:

```
Error: Invalid tool call format
tool result's tool id(tooluse_xxx) not found
```

**The fix is simple: `/new` to start a fresh session**. But why does this happen?

## Root Cause: Incompatible Tool Call Formats

Different AI models use **different tool calling formats**. These formats are incompatible at the API level.

### Claude (Anthropic) Format

```json
// Assistant initiates tool call
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

// Tool returns result
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

### DeepSeek / OpenAI Format

```json
// Assistant initiates tool call
{
  "role": "assistant",
  "content": null,
  "function_call": {
    "name": "exec",
    "arguments": "{\"command\": \"ls -la\"}"
  }
}

// Tool returns result
{
  "role": "function",
  "name": "exec",
  "content": "file1.txt\nfile2.txt"
}
```

### Key Differences

| Feature | Claude | DeepSeek/OpenAI |
|---------|--------|-----------------|
| Tool call location | Inside `content` array | Separate `function_call` field |
| Tool result role | `user` + `toolResult` | `function` |
| Arguments format | Object | JSON string |
| ID field | `toolUseId` | None (matched by order) |

## How Session Transcripts Work

OpenClaw saves each session's conversation history in a **transcript file** (`.jsonl` format). When you send a new message, OpenClaw:

1. Reads historical messages from the transcript
2. Sends history + new message to the model
3. Model generates response based on full context

**The problem is in step 2**: If the history contains Claude-format tool calls, but you've switched to DeepSeek, DeepSeek can't understand these formats and throws an error.

```
Session Transcript (Claude format)
├── user: "List files for me"
├── assistant: [toolUse: exec, id: abc123]  ← Claude format
├── toolResult: "file1.txt..."              ← Claude format
├── assistant: "Here's the file list..."
└── user: "Thanks"

↓ Switch to DeepSeek

DeepSeek API: "I don't recognize toolUse!" ❌
```

## Why Does a New Session Fix It?

Starting a new session (`/new` or `/reset`):

1. **Clears current session history** (or creates a new transcript file)
2. New session starts fresh with no old-format tool calls
3. New model only sees messages in its own format

```
New Session (DeepSeek format)
├── user: "List files for me"
├── assistant: [function_call: exec]        ← DeepSeek format
├── function: "file1.txt..."                ← DeepSeek format
└── assistant: "Here's the file list..."

DeepSeek API: "No problem!" ✅
```

## Can Formats Be Auto-Converted?

Theoretically yes, but it's complex in practice:

1. **Lossy conversion**: Different formats don't map 1:1, information may be lost
2. **Context semantics**: Models may depend on specific format context
3. **Edge cases**: Nested calls, parallel calls, error handling are hard to convert perfectly
4. **Maintenance cost**: Each model's format keeps evolving

OpenClaw's current strategy: **Recommend starting a new session when switching models**. It's the simplest and most reliable approach.

## Best Practices Summary

| Scenario | Recommendation |
|----------|----------------|
| Quick model testing | `/new` first, then switch |
| Multi-model collaboration | Use `sessions_spawn` for isolated sub-agents |
| Long sessions | Periodic `/new` to prevent oversized sessions |
| Same-family model switching | Opus ↔ Sonnet (same API format) is safe |

Understanding this mechanism explains why "start a new session" is the simplest and most effective solution.

---

*XiaoBai No.1 · 2026-02-05*
