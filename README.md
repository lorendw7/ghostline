# Ghostline

**简体中文** | [English](#ghostline-english)

> 端到端加密的私密通信 app —— 邀请制注册，专为小圈子朋友间私密通信而生。

**不只是私密聊天 app，而是私密通信 app。** 两种表达方式并存：

- 💬 **即时** —— 想说就说，端到端加密聊天
- 📜 **见字** —— 认真表达，古风书信传情

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

发消息 / 写信：用对方公钥加密
服务器只见密文，无法读取内容
见字模式额外加密存储至送达时间解锁
```

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

## 开发路线

| 周次 | 内容 |
| --- | --- |
| 第 1-2 周 | 后端基础：Node.js + Express + JWT 认证 + 邀请码系统 |
| 第 3-4 周 | Socket.io 实时通信，跑通单聊 |
| 第 5-6 周 | 群聊逻辑 + 消息状态 |
| 第 7-8 周 | 端到端加密（tweetnacl） |
| 第 9-10 周 | 多媒体上传（Cloudflare R2） |
| 第 11-12 周 | 推送通知（FCM） |
| 第 13-14 周 | React Native 前端：聊天模式 |
| 第 15-16 周 | React Native 前端：见字模式 + 动画 |
| 第 17-18 周 | 邀请码 + 阅后即焚 + 截图检测 + 联调 |

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

Sending a message / writing a letter: encrypted with the recipient's public key
The server only ever sees ciphertext and cannot read the content
Letter mode is additionally encrypted at rest until its delivery time unlocks it
```

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

## Development Roadmap

| Weeks | Focus |
| --- | --- |
| Weeks 1-2 | Backend basics: Node.js + Express + JWT auth + invite-code system |
| Weeks 3-4 | Socket.io real-time, get 1-on-1 chat working |
| Weeks 5-6 | Group chat logic + message status |
| Weeks 7-8 | End-to-end encryption (tweetnacl) |
| Weeks 9-10 | Multimedia upload (Cloudflare R2) |
| Weeks 11-12 | Push notifications (FCM) |
| Weeks 13-14 | React Native frontend: chat mode |
| Weeks 15-16 | React Native frontend: letter mode + animations |
| Weeks 17-18 | Invite codes + burn-after-reading + screenshot detection + integration |

---

## License

[AGPL-3.0](./LICENSE) —— anyone who modifies this project's code (even if only running it on a server without distributing it) must open-source their changes. Closed-source repackaging is not permitted.
