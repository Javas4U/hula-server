# HuLa-Server 项目深度分析报告

## 1. 项目概览

HuLa-Server 是一款基于 **Spring Cloud 2024** & **Spring Boot 3** 构建的高性能即时通讯（IM）服务端系统。它采用微服务架构，核心设计目标是高并发、高可用和易扩展。

- **JDK 版本**: Java 21
- **构建工具**: Maven (核心工程位于 `luohuo-cloud` 目录)
- **核心架构**: Spring Cloud Alibaba (Nacos, RocketMQ) + Netty + WebFlux

## 2. 核心技术栈

| 分类           | 技术组件                           | 用途                             |
| :------------- | :--------------------------------- | :------------------------------- |
| **基础框架**   | Spring Boot 3.x, Spring Cloud 2024 | 微服务基础设施                   |
| **网络通信**   | Netty, Reactor-Netty (WebFlux)     | 高性能长连接、异步 I/O           |
| **消息中间件** | RocketMQ                           | 消息解耦、削峰填谷、顺序消息保证 |
| **缓存/存储**  | Redis, MySQL, MyBatis-Plus         | 会话缓存、持久化存储             |
| **服务治理**   | Nacos                              | 服务注册发现、配置中心           |
| **鉴权安全**   | Sa-Token, OAuth2                   | 统一认证、权限控制               |

## 3. 模块结构分析 (`luohuo-cloud`)

项目采用标准的 Maven 多模块结构，根模块为 `luohuo-cloud`。

| 模块名称                   | 路径                                   | 功能说明               | 二次开发核心关注点                                     |
| :------------------------- | :------------------------------------- | :--------------------- | :----------------------------------------------------- |
| **luohuo-gateway**         | `luohuo-gateway/luohuo-gateway-server` | **API 网关**           | 统一入口，负责路由转发、全局鉴权、跨域配置、限流熔断。 |
| **luohuo-im**              | `luohuo-im/luohuo-im-server`           | **IM 业务中心**        | 处理单聊/群聊逻辑、消息持久化、群组管理、好友关系。    |
| **luohuo-ws**              | `luohuo-ws/luohuo-ws-server`           | **接入层 (WebSocket)** | 维护用户长连接 (Netty)，负责消息的实时推送和协议解析。 |
| **luohuo-oauth**           | `luohuo-oauth/luohuo-oauth-server`     | **认证中心**           | 处理用户登录、Token 签发与校验、多端登录互踢逻辑。     |
| **luohuo-user** / **base** | `luohuo-base/luohuo-base-server`       | **基础用户服务**       | 用户基础信息管理、租户管理。                           |
| **luohuo-ai**              | `luohuo-ai/luohuo-ai-server`           | **AI 能力中心**        | 集成 LLM 大模型，提供智能助手、文生图等 AI 功能。      |
| **luohuo-presence**        | `luohuo-presence`                      | **状态服务**           | 维护用户全局在线状态 (Online/Offline)。                |
| **luohuo-common**          | `luohuo-public/luohuo-common`          | **公共组件**           | 全局异常处理、通用工具类、常量定义。                   |

## 4. 代码分层规范

多数业务服务（如 `luohuo-im`）遵循以下标准分层：

- **`*-server`**: **启动层**。包含 `StandardApplication` 启动类和 `application.yml` 配置。
- **`*-biz`**: **业务层**。包含 Service 实现类，是业务逻辑的核心聚集地。
- **`*-controller`**: **接口层**。对外暴露 RESTful API。
- **`*-entity`**: **领域模型层**。包含 DO (数据库实体), DTO (传输对象), VO (视图对象)。
- **`*-api`**: **Feign 接口层**。提供给其他微服务调用的 RPC 接口定义。

## 5. 消息流转核心流程

1.  **发送**: 客户端 -> `luohuo-gateway` -> `luohuo-im` (业务校验、持久化)。
2.  **分发**: `luohuo-im` -> MQ / RPC -> `luohuo-ws` (查找目标用户连接)。
3.  **推送**: `luohuo-ws` -> Netty Channel -> 客户端。

## 6. 二次开发指引

### 环境准备

在本地启动开发前，请确保基础设施已就绪：

1.  **Nacos**: 必须启动，用于配置和注册。
2.  **MySQL / Redis**: 数据存储与缓存。
3.  **RocketMQ**: 消息队列，核心通信依赖。

### 推荐启动顺序

1.  `luohuo-gateway-server` (网关)
2.  `luohuo-oauth-server` (依赖它获取 Token)
3.  `luohuo-ws-server` (接入层)
4.  `luohuo-im-server` (业务层)

### 常见场景修改点

- **新增 API 接口**: 在对应模块的 `xxx-controller` 下新建 Controller，并在 `xxx-biz` 实现 Service。
- **修改消息协议**: 关注 `luohuo-ws` 模块下的 Netty Handler 链。
- **定制业务逻辑**: 如 "发送消息前先调用第三方审核"，请在 `luohuo-im` 模块的消息发送 Service 中介入。
