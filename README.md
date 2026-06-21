# Ghostline

**简体中文** | [English](#ghostline-english)

> 端到端加密的私密通信 app —— 邀请制注册，专为小圈子朋友间私密通信而生。

**不只是私密聊天 app，而是私密通信 app。** 两种表达方式并存：

- 💬 **即时** —— 想说就说，端到端加密聊天
- 📜 **见字** —— 认真表达，古风书信传情

---

## 📚 文档导航

| 文档 | 回答的问题 |
| --- | --- |
| **README**（本文） | 这是什么、长什么样、用什么技术 |
| [ROADMAP](./ROADMAP.md) | 按什么顺序做 —— 学习优先的纵切片路线 |
| [GOALS](./GOALS.md) | 做到什么程度算「完整且实用」—— 全功能 + 优先级 + v1.0 验收清单 |
| [PRODUCTION](./PRODUCTION.md) | 工业级工程严谨度 —— 安全、可靠、可观测、合规、发布 |

---

## 两大核心模块

### 💬 聊天模式（即时）

- 端到端加密实时消息
- 群聊
- 多媒体消息（图片、语音、视频）
- 阅后即焚
- 已发送 / 已送达 / 已读 状态
- 截图提醒

### 📜 见字模式（古风书信）

- 仿古信纸界面（宣纸、竹简、锦缎、梅花笺）
- 专属印章系统（梅兰竹菊龙凤等图案）
- 封印发送（发出后不可撤回）
- 延迟送达（可设置 1 小时 / 明天 / 3 天后 送达）
- 拆信动画（信纸慢慢展开）
- 焚信功能（阅后即焚，信纸燃烧动画）
- 节气时间显示（「癸卯年 芒种」替代时间戳）
- 收信推送通知（「你收到一封来自 XX 的信」）

---

## 技术栈

### 前端

- **React Native (Expo)**
- `react-native-gifted-chat` —— 聊天模式 UI
- `tweetnacl-js` —— 端到端加密
- `react-native-mmkv` —— 本地加密存储
- `expo-screen-capture` —— 截图检测
- `Zustand` —— 状态管理
- `react-native-reanimated` —— 拆信 / 焚信动画

### 后端

- **Node.js + Express** —— HTTP 接口
- **Socket.io** —— 实时通信
- **JWT** —— 认证
- **node-cron** —— 定时任务（检查见字送达时间）
- 最小化存储原则（服务器只存密文和公钥）

### 数据库 / 存储

- **PostgreSQL** 或 **MongoDB** —— 主数据库
- **Redis (Upstash)** —— 在线状态、未读数、缓存
- **Cloudflare R2** —— 图片、语音、视频、信纸素材

### 推送

- **Firebase FCM** —— 送达提醒 + 普通消息推送

---

## 加密架构

```
注册时在设备本地生成密钥对
公钥上传服务器
私钥永远留在本地

发消息 / 写信：用对方公钥加密（NaCl box，需自己的私钥 + 对方公钥）
服务器只见密文，无法读取内容
```

> **关于见字模式的「延迟送达」**：服务器**永远**读不到明文，所以「到送达时间解锁」
> 指的是服务器在 `deliverAt` 之前**扣着那团密文不下发**，到点才推给收信人，由收信人设备解密。
> 服务器全程只是个「定时投递的密文邮筒」，而非到点解密。

> **加密成熟度**：起步用 `tweetnacl box`（静态密钥对），对朋友圈够用，但**不提供前向保密 / 多设备**。
> 升级路径（Double Ratchet 等）见 [PRODUCTION.md](./PRODUCTION.md) 的加密成熟度阶梯。

---

## 见字数据结构

```javascript
{
  content: "加密内容...",
  sealedAt: Date.now(),           // 封印时间
  deliverAt: Date.now() + 延迟,   // 送达时间
  style: "xuanzhi",               // 信纸样式
  seal: "plum",                   // 印章图案
  canBurn: true,                  // 是否可焚信
  solarTerm: "芒种",              // 节气
  lunarDate: "癸卯年五月初六"      // 农历日期
}
```

---

## 部署方案（全部免费）

| 组件 | 平台 |
| --- | --- |
| 后端 | Railway |
| 数据库 | Supabase (PostgreSQL) |
| Redis | Upstash |
| 文件存储 | Cloudflare R2 |
| 推送 | Firebase FCM |

---

## 发布计划

- **安卓** —— APK 直接发给朋友，$0 成本
- **iOS** —— Apple 开发者账号 $99/年，TestFlight 内测后上架
- **Google Play** —— $25 一次性（可选）

---

## 开发路线（纵切片，学习优先）

不按「先做完后端再做前端」的横向分层，而是**每个切片都从手机一路打通到服务器**，
先窄后全 —— 第 1 周结束就能和朋友用两台真手机实时聊天，再逐层加深。

| 切片 | 内容 | 结束时能验证 |
| --- | --- | --- |
| 0 | 环境搭建：Expo + 最小 Express | 手机连上本机后端 |
| 1 | 明文实时单聊（Socket.io） | 两台手机实时收发 ⭐ |
| 2 | 身份 + 持久化（JWT + 邀请码 + Supabase） | 离线消息、重启不丢 |
| 3 | 端到端加密（tweetnacl + secure-store） | 数据库里只有密文 ⭐ |
| 4 | 聊天打磨（状态 / 在线 / 媒体 / 推送） | 双勾已读、推送送达 |
| 5 | 见字模式（信纸 / 封印 / 延迟送达 / 动画） | 定时送达 + 拆信 ⭐ |
| 6 | 加固与发布（阅后即焚 / 截图 / 群聊 / 打包） | 朋友当日常工具用 |

> 完整切片说明见 [ROADMAP.md](./ROADMAP.md)；功能优先级与 v1.0 验收清单见 [GOALS.md](./GOALS.md)。

---

## 开源协议

[AGPL-3.0](./LICENSE) —— 任何人修改本项目代码（哪怕只在服务器上运行而不分发），也必须开源其修改，禁止闭源套壳。

---

# Ghostline (English)

[简体中文](#ghostline) | **English**

> An end-to-end encrypted private messaging app —— invite-only, built for private communication within close-knit circles of friends.

**Not just a private chat app, but a private *communication* app.** Two ways to express yourself, side by side:

- 💬 **Instant** —— say it as it comes, end-to-end encrypted chat
- 📜 **Letters (见字)** —— say it with intention, classical-style correspondence

---

## 📚 Docs

| Doc | Answers |
| --- | --- |
| **README** (this) | What it is, what it looks like, what it's built with |
| [ROADMAP](./ROADMAP.md) | What order to build it in — learning-first vertical slices |
| [GOALS](./GOALS.md) | What "complete & practical" means — full features + priorities + v1.0 checklist |
| [PRODUCTION](./PRODUCTION.md) | Production-grade rigor — security, reliability, observability, compliance, release |

---

## Two Core Modules

### 💬 Chat Mode (Instant)

- End-to-end encrypted real-time messaging
- Group chats
- Multimedia messages (images, voice, video)
- Disappearing messages (burn after reading)
- Sent / Delivered / Read status
- Screenshot alerts

### 📜 Letter Mode (Classical Correspondence)

- Antique letter-paper UI (xuan paper, bamboo slips, brocade, plum-blossom stationery)
- Personal seal system (plum, orchid, bamboo, chrysanthemum, dragon, phoenix, etc.)
- Sealed sending (cannot be recalled once sent)
- Delayed delivery (deliver in 1 hour / tomorrow / 3 days)
- Letter-opening animation (the paper slowly unfolds)
- Burn-after-reading (with a paper-burning animation)
- Solar-term timestamps (「癸卯年 芒种」instead of a numeric time)
- Delivery push notifications (“You received a letter from XX”)

---

## Tech Stack

### Frontend

- **React Native (Expo)**
- `react-native-gifted-chat` —— chat mode UI
- `tweetnacl-js` —— end-to-end encryption
- `react-native-mmkv` —— encrypted local storage
- `expo-screen-capture` —— screenshot detection
- `Zustand` —— state management
- `react-native-reanimated` —— letter-opening / burning animations

### Backend

- **Node.js + Express** —— HTTP API
- **Socket.io** —— real-time communication
- **JWT** —— authentication
- **node-cron** —— scheduled jobs (checking letter delivery times)
- Minimal-storage principle (the server only stores ciphertext and public keys)

### Database / Storage

- **PostgreSQL** or **MongoDB** —— primary database
- **Redis (Upstash)** —— presence, unread counts, caching
- **Cloudflare R2** —— images, voice, video, stationery assets

### Push

- **Firebase FCM** —— delivery alerts + regular message push

---

## Encryption Architecture

```
A key pair is generated locally on the device at registration
The public key is uploaded to the server
The private key never leaves the device

Sending a message / writing a letter: encrypted with the recipient's public key (NaCl box — needs your private key + their public key)
The server only ever sees ciphertext and cannot read the content
```

> **On letter mode's "delayed delivery":** the server can *never* read the plaintext, so "unlock at
> delivery time" means the server simply **withholds the (still-encrypted) blob until `deliverAt`**,
> then pushes it to the recipient, whose device decrypts it. The server is just a "timed ciphertext
> mailbox," never a decryptor.

> **Crypto maturity:** we start with `tweetnacl box` (static key pairs) — fine for a circle of friends,
> but it provides **no forward secrecy / multi-device**. Upgrade path (Double Ratchet, etc.) is in
> [PRODUCTION.md](./PRODUCTION.md).

---

## Letter Data Structure

```javascript
{
  content: "encrypted content...",
  sealedAt: Date.now(),           // when it was sealed
  deliverAt: Date.now() + delay,  // delivery time
  style: "xuanzhi",               // stationery style
  seal: "plum",                   // seal design
  canBurn: true,                  // burn-after-reading enabled?
  solarTerm: "芒种",              // solar term
  lunarDate: "癸卯年五月初六"      // lunar date
}
```

---

## Deployment (all free tier)

| Component | Platform |
| --- | --- |
| Backend | Railway |
| Database | Supabase (PostgreSQL) |
| Redis | Upstash |
| File storage | Cloudflare R2 |
| Push | Firebase FCM |

---

## Release Plan

- **Android** —— ship the APK directly to friends, $0 cost
- **iOS** —— Apple Developer account $99/year, TestFlight beta then App Store
- **Google Play** —— $25 one-time fee (optional)

---

## Development Roadmap (vertical slices, learning-first)

Instead of "finish all the backend, then all the frontend," **every slice cuts straight through
from phone to server** — narrow but complete first, then deepen. By end of week 1 you and a friend
are chatting on two real phones.

| Slice | Focus | Verifiable when done |
| --- | --- | --- |
| 0 | Setup: Expo + minimal Express | Phone reaches local backend |
| 1 | Plaintext real-time 1:1 chat (Socket.io) | Two phones exchange messages ⭐ |
| 2 | Identity + persistence (JWT + invites + Supabase) | Offline delivery, survives restart |
| 3 | End-to-end encryption (tweetnacl + secure-store) | DB holds only ciphertext ⭐ |
| 4 | Chat polish (status / presence / media / push) | Read receipts, push delivered |
| 5 | Letter mode (stationery / sealing / delay / animation) | Timed delivery + opening ⭐ |
| 6 | Hardening & release (burn / screenshot / group / build) | Friends use it daily |

> Full slice breakdown in [ROADMAP.md](./ROADMAP.md); feature priorities and the v1.0 acceptance
> checklist in [GOALS.md](./GOALS.md). Production-grade engineering in [PRODUCTION.md](./PRODUCTION.md).

---

## License

[AGPL-3.0](./LICENSE) —— anyone who modifies this project's code (even if only running it on a server without distributing it) must open-source their changes. Closed-source repackaging is not permitted.
