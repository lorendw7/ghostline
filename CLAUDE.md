# CLAUDE.md

给 Claude Code 的项目上下文。详细介绍见 [README.md](./README.md)。

## 项目一句话

端到端加密的私密通信 app，邀请制注册。两种通信模式并存：💬 即时聊天 + 📜 古风书信（见字）。

## 核心原则

- **端到端加密优先**：私钥永远留在设备本地，服务器只存密文和公钥，无法读取明文内容。
- **最小化存储**：服务器尽量不留可读数据。见字模式的内容额外加密，到送达时间才解锁。
- **邀请制**：注册需邀请码，不开放公开注册。

## 两大模块

- **聊天模式（即时）**：E2EE 实时消息、群聊、多媒体、阅后即焚、消息状态（已发送/送达/已读）、截图提醒。
- **见字模式（古风书信）**：仿古信纸、印章系统、封印发送（不可撤回）、延迟送达、拆信/焚信动画、节气时间显示、收信推送。

## 技术栈

- **前端**：React Native (Expo) · gifted-chat · tweetnacl-js（加密）· react-native-mmkv（本地加密存储）· expo-screen-capture（截图检测）· Zustand · react-native-reanimated（动画）
- **后端**：Node.js + Express · Socket.io（实时）· JWT · node-cron（见字送达定时任务）
- **存储**：PostgreSQL/MongoDB（主库）· Redis/Upstash（在线状态、缓存）· Cloudflare R2（媒体、信纸素材）
- **推送**：Firebase FCM

## 部署（全免费）

后端 Railway · 数据库 Supabase · Redis Upstash · 文件 Cloudflare R2 · 推送 FCM

## 见字数据结构

```javascript
{
  content,      // 加密内容
  sealedAt,     // 封印时间
  deliverAt,    // 送达时间（延迟送达）
  style,        // 信纸样式 e.g. "xuanzhi"
  seal,         // 印章图案 e.g. "plum"
  canBurn,      // 是否可焚信
  solarTerm,    // 节气 e.g. "芒种"
  lunarDate     // 农历日期 e.g. "癸卯年五月初六"
}
```

## 开发顺序（建议）

后端基础(认证+邀请码) → Socket.io 单聊 → 群聊+消息状态 → E2EE(tweetnacl) → 多媒体上传(R2) → 推送(FCM) → 前端聊天模式 → 前端见字模式+动画 → 阅后即焚/截图检测/联调。

## 协议

AGPL-3.0（见 [LICENSE](./LICENSE)）。涉及他人贡献或衍生时注意 copyleft 要求。
