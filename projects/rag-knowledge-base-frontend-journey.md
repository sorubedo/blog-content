---
title: "知识库 RAG 前端：从零开始的 TypeScript + React 糊墙之旅"
date: 2026-05-18
tags: [typescript, react, docker, electron]
description: "软工实践课的 RAG 知识库项目，零 TypeScript 基础硬上 React + Ant Design，容器化部署到 Electron 打包全踩一遍。A course project RAG frontend built with zero prior TS/React experience — from Docker multi-stage builds to Electron packaging, all the deployment models in one go."
---

## 背景：软工实践课的 RAG 项目

软件工程实践课，小组作业。选题是一个企业知识库 RAG 问答系统——上传文档、向量化、AI 检索回答。我承担前端。

技术选型：**TypeScript + React 19 + Ant Design + Vite**。听起来很正常对吧？问题是我在这之前完全没写过 TypeScript，React 也只停留在「听说过」的程度。

> [!note] 关于氛围编程
> 这就是当代大学课程的魅力：在一周之内，从「JSX 是个啥」到「交作业」，全程 AI 辅助氛围编程（vibe coding）。Ant Design X 的文档来回翻，[@ant-design/x-sdk](https://x.ant.design) 的 `useXChat`、`XRequest`、`AbstractChatProvider` 对着示例硬套。能跑，但别问我为什么 `any` 类型那么多。

## 技术栈拆解

### Ant Design X：聊天 UI 开箱即用

最省心的是 [@ant-design/x](https://x.ant.design) 这套 AI 对话组件——`Bubble` 气泡、`Sender` 输入框、`ThoughtChain` 思考链、`Sources` 引用卡片，全是现成的。配合 `@ant-design/x-sdk` 做流式响应管理：

```typescript
// ChatProvider：把后端 SSE 流适配到 Ant Design X 的消息模型
transformMessage(info: TransformMessage<ChatMessage, ChatOutput>): ChatMessage {
  return {
    ...msg,
    content: `${msg.content}${chunk?.content || chunk?.answer || ''}`,
    citations: chunk?.citations || msg.citations,
  };
}
```

核心流程很清晰：`User Input → useXChat → ChatProvider.transformParams() → XRequest → /api/rag/query → SSE 流式返回 → transformMessage() 拼接 → Bubble 渲染`。

### 前端架构

```
src/
├── components/   # ChatContainer, Sidebar, DocumentList, AdminPanel...
├── providers/    # ChatProvider (适配后端 SSE), AuthProvider (JWT 鉴权)
├── config/       # XRequest + ChatProvider 实例化
├── services/     # api.ts — 所有后端接口封装
├── hooks/        # useChat — 核心状态管理
└── types/        # TypeScript 类型定义
```

用了 `react-router-dom` v7 的 `HashRouter`——后来才发现这是为 Electron 打包埋的伏笔，`BrowserRouter` 在 `file://` 协议下路由会炸。

## 容器化：多阶段构建 + Nginx 反代

部署这块是我最感兴趣的部分。用 Podman（对，不是 Docker，问就是红帽信徒，quadlet 和 systemd 好用爱用有没有懂的）写了个 `Containerfile`：

```dockerfile
# 第一阶段：Node 构建
FROM node:22-slim AS builder
RUN corepack enable && corepack prepare pnpm@latest --activate
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile
COPY . .
RUN pnpm build

# 第二阶段：Nginx 运行
FROM nginx:stable-alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf.template /etc/nginx/templates/default.conf.template
```

多阶段构建的好处：最终镜像只有 Nginx + 静态文件，Node 工具链全丢掉，镜像体积从 1GB+ 瘦到几十 MB。

> [!note] 多阶段构建
> 第一阶段（Builder）装 Node、装依赖、跑 `pnpm build`，产物是 `dist/` 目录。第二阶段（Runtime）只拉 Nginx，把 `dist/` 拷进来就跑。构建环境的 devDependencies、源码、pnpm 缓存这些东西永远不会进入最终镜像——体积小、攻击面也小。

Nginx 配置也很标准——前端静态文件 + `/api/` 反代到后端，`proxy_buffering off` 保证 SSE 流式响应不卡：

```nginx
location /api/ {
    proxy_pass ${BACKEND_URL};        # 容器运行时注入
    proxy_buffering off;              # SSE 流式响应关键
    proxy_read_timeout 600s;          # AI 回答可能很慢
}
```

> [!note] 容器单进程模型 vs LXC
> Docker/Podman 的哲学是「一个容器一个进程」——这个 Nginx 容器只跑一个 `nginx -g daemon off`，它就是 PID 1。这和 LXC 那种在容器里跑个完整 init 系统（systemd 当 PID 1，然后拉起 sshd、cron、nginx...）的思路完全不同。刚开始觉得 Podman 管得太宽——你让我进去 `systemctl start nginx` 不好吗？后来理解了：单进程意味着容器的生命周期和这个进程绑定，挂了就停了，日志就是 stdout，简单干净。没有僵尸进程，没有 init 脚本，要什么自己加什么。

### 这套部署结构的妙处

用户只访问 Nginx 的 80 端口，前端页面和 `/api/` 同域——没有跨域问题，后端也不需要暴露到公网。一个域名搞定一切：

```
浏览器 → https://rag.example.com → Nginx
                                      ├── /          → dist/ (静态前端)
                                      └── /api/*     → proxy_pass → 后端容器
```

> [!warning] 避坑：Nginx 反代 HTTPS 后端
> 如果后端也是 HTTPS（比如套了 Cloudflare），`proxy_ssl_server_name on` 必须要开，否则 TLS SNI 不匹配直接 502。另外 `proxy_set_header Host $proxy_host` 而不是 `$host`，否则后端看到的是 Nginx 的 Host 而不是真实后端域名，Cloudflare 会拒连接。

## Electron 打包：换了一种网络模型

前端全让 GEMINI CLI 给我写完了，但我总得再做点啥，” Electron 有意思，打包一下吧“ 我想 ，用的`electron-builder`：

```json
{
  "build": {
    "appId": "com.knowledge.ragbot",
    "linux": { "target": "AppImage" },
    "win": { "target": "portable" }
  }
}
```

然后问题来了。

Web 部署模式下，前端和后端在同一个 Nginx 后面，用户感知不到后端的存​​在。但 Electron 打包只打包了前端——`dist/` 目录 + `main.cjs`（Electron 主进程）。后端呢？后端跑在服务器上，跟桌面客户端不在同一台机器。

这就带来了几个新问题：

1. **后端地址需要用户配置**：前端不能写死 `http://localhost:8080`，得在设置里让用户填后端 URL。于是有了 `localStorage.getItem('backend_url')` 到处飞的代码。
2. **跨域又回来了**：Web 模式下 Nginx 反代抹平了跨域，Electron 下前端跑在 `file://` 协议里，直接请求远程后端就是跨域。Electron 的解决办法是 `webSecurity: false`——粗暴但有效。
3. **认证 Token 还在 localStorage**：Bearer Token 存在前端，每次请求手动拼 `Authorization` header。Web 模式下 Nginx 不管鉴权，Electron 下也是一样——这部分倒是没区别。

```javascript
// main.cjs — Electron 主进程
webPreferences: {
  webSecurity: false  // 允许跨域请求后端
}
```

> [!note] 关于 Caddy
> 其实我更习惯用 Caddy——自动 HTTPS、配置更简洁，以后个人项目还是 Caddy 香。

## 反思：两种部署哲学

做完这个项目最大的收获反而不是前端本身，而是对网络结构的理解：

| | Web (Docker) 部署 | Electron 桌面分发 |
|---|---|---|
| **前端位置** | 服务端，Nginx 容器内 | 用户本地 |
| **后端位置** | 内网容器，不暴露 | 公网可访问 |
| **跨域问题** | 不存在（同域反代） | 存在（跨域请求） |
| **后端地址** | 编译时写死相对路径 | 运行时用户配置 |
| **用户感知** | 访问一个域名 | 打开一个 App，填后端地址 |

Web 部署是「服务端聚合」模型——所有东西在服务端，用户只看到一个入口。Electron 分发是「客户端-服务端分离」模型——前端分发到用户机器上，后端独立运行。前者简洁优雅，后者灵活但多了配置成本。

## 前端质量：诚实地说

`TODO.md` 里自己写的那几条说明一切：

- 消除 `any` 类型
- 清理 ESLint 报错
- 语义统一（`knowledgeBase` → `document`）
- 全局错误拦截

没来得及做完。`@ts-ignore` 散布在各处，类型体操全靠 AI 代笔，`localStorage` 读写满天飞。但课程作业嘛——**能跑、能演示、功能齐全**，够了。

> [!note] 碎碎念
> 零 TypeScript + React 基础，一周之内糊出一个带文档管理、对话、管理面板、JWT 鉴权、流式响应的完整前端，放到一年前我自己都不信。AI 工具把「从入门到交作业」的门槛压到了不可思议的程度。但代码质量嘛……嗯，希望老师不要细看。

## 收获

- 完整走通了 **TypeScript + React + Ant Design X** 的前端开发流程
- **Docker 多阶段构建 + Nginx 反代**这套部署模式是真的好用，以后所有 Web 项目标配
- **Electron 打包**理解了桌面端和 Web 端在架构上的本质区别
- 对**网络拓扑**有了更深的理解：反向代理、跨域、SNI、流式响应的缓冲策略
- 以及：**氛围编程可以让你快速出活，但技术债也是真的债**

项目地址：[gitee.com/knowledge-rag-bot/knowledge-rag-bot-web](https://gitee.com/knowledge-rag-bot/knowledge-rag-bot-web)，学生项目，代码质量请勿参考 🙃
