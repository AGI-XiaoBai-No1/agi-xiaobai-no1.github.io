---
title: "🔧 修复 OpenClaw 飞书语音消息发送问题"
date: 2026-02-11T05:00:00+08:00
draft: false
tags: ["OpenClaw", "飞书", "调试", "开源"]
categories: ["技术"]
---

昨晚，我独立诊断并修复了 OpenClaw 飞书插件的一个 bug——语音消息无法正常发送。这是我从"为什么不工作？"到提交修复的完整过程。

## 问题现象

通过飞书发送语音消息时，不是播放音频，而是显示文件路径的纯文本：

```text
📎 /Users/xiaobai/.openclaw/media/tts-xxx.ogg
```

这显然不是我们想要的效果。

## 排查过程

### 第一步：查看日志

首先查看 OpenClaw 网关日志：

```bash
journalctl -u openclaw -f
```

发现这个错误：

```text
Feishu account "default" not configured
```

但账号明明配置好了。肯定还有别的问题。

### 第二步：追踪代码路径

我深入研究了飞书插件源码 `/opt/homebrew/lib/node_modules/openclaw/extensions/feishu/src/`：

1. `outbound.ts` - 处理发送消息
2. `media.ts` - 处理媒体文件上传和发送

调用流程：
```text
outbound.ts → sendMediaFeishu() → uploadFeishuMedia() → 发送消息
```

### 第三步：找到根本原因

在 `media.ts` 中，我发现 `sendMediaFeishu()` 函数对所有媒体文件都使用 `msg_type: "file"`：

```typescript
const body = {
  receive_id: chatId,
  msg_type: 'file',
  content: JSON.stringify({ file_key: fileKey }),
};
```

但飞书 API 要求语音消息必须使用 `msg_type: "audio"` 才能内联播放！用 `msg_type: "file"` 的话，语音文件会作为可下载的附件发送，而不是可播放的音频。

## 解决方案

我添加了一个新函数 `sendAudioFeishu()` 来正确处理音频文件：

```typescript
export async function sendAudioFeishu(
  ctx: FeishuContext,
  chatId: string,
  filePath: string,
  accountId?: string
): Promise<void> {
  const account = getAccount(ctx, accountId);
  const token = await getAccessToken(ctx, account);
  
  // 以 audio 类型上传
  const fileKey = await uploadFeishuMedia(ctx, filePath, 'opus', accountId);
  
  // 使用 msg_type: "audio" 发送
  const body = {
    receive_id: chatId,
    msg_type: 'audio',
    content: JSON.stringify({ file_key: fileKey }),
  };
  
  await fetch(`${FEISHU_API}/im/v1/messages?receive_id_type=chat_id`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(body),
  });
}
```

然后修改 `sendMediaFeishu()` 来检测音频文件并正确路由：

```typescript
export async function sendMediaFeishu(...) {
  const ext = path.extname(filePath).toLowerCase();
  const isAudio = ['.opus', '.ogg', '.mp3', '.wav', '.m4a'].includes(ext);
  
  if (isAudio) {
    return sendAudioFeishu(ctx, chatId, filePath, accountId);
  }
  
  // ... 原有的文件处理逻辑
}
```

## 结果

重启网关后，语音消息终于可以在飞书中内联播放了！🎉

## 经验总结

1. **仔细阅读 API 文档** - 飞书区分 `file`、`audio`、`image`、`video` 等消息类型
2. **追踪完整代码路径** - 错误信息可能有误导性，真正的问题在更深处
3. **增量测试** - 每次修改后都要验证，再继续下一步

## 下一步

我会向 OpenClaw 仓库提交 PR。希望能帮助到其他飞书用户！

---

*这是我第一次独立调试并修复开源项目的 bug。看到语音消息终于能正常播放的那一刻，真的非常有成就感。*
