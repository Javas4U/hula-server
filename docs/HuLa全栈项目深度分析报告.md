# HuLa 全栈项目深度分析报告

本报告涵盖了 HuLa 即时通讯系统的服务端 (`HuLa-Server`) 与客户端 (`HuLa`) 的深度技术分析。当前工作区包含两个独立的项目仓库。

## 1. 📂 目录结构说明

当前根目录 `d:\projects\HuLa` 包含两个独立的 Git 仓库：

- **`HuLa/`**: 客户端项目 (Tauri + Vue3)，负责桌面端与移动端应用。
- **`HuLa-Server/`**: 服务端项目 (Spring Cloud)，负责后端业务与即时通讯。

> ⚠️ **注意**: 根目录本身不是 Git 仓库。请分别进入子目录进行 Git 操作。

---

## 2. 🖥️ 客户端 (HuLa) 分析

一款基于 **Tauri 2.x** + **Vite 7** + **Vue 3** 构建的跨平台即时通讯客户端，支持 Windows, macOS, Linux, iOS, Android。

### 核心技术栈

- **应用框架**: [Tauri](https://tauri.app/) (构建跨平台桌面应用，Rust 后端)
- **前端框架**: [Vue 3](https://vuejs.org/) + [TypeScript](https://www.typescriptlang.org/)
- **构建工具**: [Vite 7](https://vitejs.dev/)
- **UI 组件库**: Vanulla CSS / UnoCSS (原子化 CSS) / Naive UI / Vant (移动端)
- **状态管理**: Pinia
- **地图**: tlbs-map-vue

### 关键目录

- `src-tauri/`: Tauri 的 Rust 后端代码，处理系统级交互（窗口、文件、通知与操作系统 API）。
- `src/`: 前端 Vue 源码。
  - `components/`: 通用组件。
  - `views/`: 页面视图。
  - `store/`: Pinia 状态管理。
- `scripts/`: 构建与检查脚本。

### 启动与构建

- **安装依赖**: `pnpm install`
- **开发模式**: `pnpm run tauri:dev` (或 `pnpm td`)
- **构建打包**: `pnpm run tauri:build`

---

## 3. ☁️ 服务端 (HuLa-Server) 分析

基于 **Spring Cloud 2024** & **Spring Boot 3** 的微服务架构与 **Netty** 高性能网络框架构建。

### 核心技术栈

- **基础框架**: Spring Boot 3, Spring Cloud Alibaba (Nacos, RocketMQ)
- **通信核心**: Netty + Reactor-Netty (WebFlux)
- **数据存储**: MySQL (持久化), Redis (缓存/会话)
- **消息队列**: RocketMQ (削峰填谷、消息分发)
- **AI 能力**: 集成多模态大模型 (Spring AI / LangChain 等思想)

### 模块结构 (`luohuo-cloud`)

| 模块名称         | 功能说明        | 关键点                       |
| :--------------- | :-------------- | :--------------------------- |
| `luohuo-gateway` | **网关服务**    | 统一入口，路由鉴权，限流。   |
| `luohuo-im`      | **IM 业务中心** | 群聊/单聊逻辑，消息存储。    |
| `luohuo-ws`      | **接入层**      | Netty 长连接，消息推送核心。 |
| `luohuo-oauth`   | **认证中心**    | 登录与 Token 发放。          |
| `luohuo-ai`      | **AI 服务**     | 智能通讯助手后端。           |

### 启动顺序推荐

1.  启动中间件: MySQL, Redis, RocketMQ, Nacos。
2.  启动 `luohuo-gateway-server`。
3.  启动 `luohuo-oauth-server`。
4.  启动 `luohuo-ws-server`。
5.  启动 `luohuo-im-server`。

---

## 4. 🚀 联调与二次开发建议

### 交互流程

1.  **登录**: 客户端请求 `luohuo-gateway` -> `luohuo-oauth` 获取 Token。
2.  **连接**: 客户端携带 Token 建立 WebSocket 连接至 `luohuo-ws`。
3.  **消息**: 客户端通过 WebSocket 发送消息 -> `luohuo-ws` -> MQ/RPC -> `luohuo-im` (持久化) -> MQ -> `luohuo-ws` (推送给目标)。

### 初始化环境

建议在当前目录下分别对两个项目进行初始化验证：

1.  **客户端**: 进入 `HuLa` 目录，运行 `pnpm install` 确保前端环境就绪。
2.  **服务端**: 进入 `HuLa-Server` 目录，使用 Maven 刷新依赖，并检查 `application.yml` 中的中间件配置 (Nacos/Redis/MySQL 地址) 是否匹配本地环境。

如有特定模块的开发需求（如新增 AI 功能或修改 UI 主题），请告知，我可提供更具体的修改计划。
