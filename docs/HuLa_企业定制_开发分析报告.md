# 企业内部通信定制 - 开发分析报告与落地计划

**生成时间**: 2025-12-14
**版本**: v1.1 (增加 VPS 部署计划)

## 1. 核心目标声明

本报告基于将 HuLa 改造为 **企业内部私有化即时通讯系统** 的目标进行分析。
重点关注：**内网部署**、**数据安全**、**组织架构对接**。
排除项：**公网支付模块**、**Web 网页版适配**。

## 2. 现有架构与企业级适配度分析

| 系统模块                 | 现有能力                                             | 企业定制需改造点                                                                                                                           | 适配难度    |
| :----------------------- | :--------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------- | :---------- |
| **服务端 (hula-server)** | Spring Cloud Alibaba 微服务 (Nacos, RocketMQ, Netty) | **内网穿透/IP 配置**: 需确保 Nacos 和服务注册 IP 对内网可达。<br>**鉴权集成**: 默认是手机号/账密登录，企业可能需要 LDAP/AD 域集成。        | ⭐⭐ (中等) |
| **客户端 (hula-app)**    | Tauri (Rust + Vue3) 跨平台桌面/移动端                | **打包分发**: 企业内网缺乏 App Store，需搭建简单的下载站或通过 MDM 推送。<br>**更新机制**: 需修改 `tauri.conf.json` 中的更新源为内网地址。 | ⭐ (低)     |
| **管理端 (hula-admin)**  | Vue3 + NaiveUI                                       | **组织架构管理**: 现有的手动录入不适合大企业，需开发 "Excel 导入" 或 "钉钉/飞书组织架构同步" 功能。                                        | ⭐⭐ (中等) |
| **数据存储**             | MySQL, Redis, MinIO                                  | **数据备份**: 需增加定时备份策略。<br>**MinIO**: 需确保文件服务端口对所有客户端开放。                                                      | ⭐ (低)     |

## 3. 全流程跑通路线图 (Roadmap)

为了快速验证系统可行性，建议按照 **部署 -> 配置 -> 联调** 的顺序执行。

### 第一阶段：基础设施部署 (Infrastructure)

此阶段目标是让所有中间件在本地或服务器上运行起来。

- **依赖环境**: Docker, Docker Compose。
- **关键文件**: `hula-server/docs/install/docker/docker-compose.yml`。
- **注意**: 配置文件中的 `CANDIDATE` 环境变量 (在 SRS 服务中) 需要修改为**宿主机的局域网 IP**，否则音视频通话将无法连接。

### 第二阶段：服务端配置与启动 (Backend)

此阶段目标是启动无状态服务并注册到 Nacos。

1.  **数据库初始化**: 运行 `docs/sql/` 下的 SQL 脚本，创建库表。
2.  **配置修改**:
    - 修改 Nacos 配置 (或 `application.yml`) 中的中间件地址，指向 Docker 容器暴露的端口。
    - 确保 `luohuo-gateway` 暴露的端口 (如 8080/8888)未被防火墙拦截。

### 第三阶段：客户端构建与连接 (Client)

此阶段目标是打包出一个能用的 Exe/Apk。

1.  **API 地址指向**: 修改 `hula-app/.env` 或常量文件，将 `VITE_BASE_URL` 指向局域网网关地址 (例如 `http://192.168.1.100:8080`)。
2.  **Tauri 权限**: 检查 `tauri.conf.json` 的 `csp` 配置，允许连接内网 IP。
3.  **打包**: 运行 `pnpm tauri build` 生成安装包并安装测试。

## 4. VPS 部署与打包计划 (新增)

针对 VPS (Virtual Private Server, Linux 环境) 的生产级部署方案。

### 4.1 服务端部署架构 (服务端)

推荐采用 **"中间件 Docker 化 + 业务服务 Jar 包运行"** 或 **"全 Docker 化"** 方案。
考虑到运维便捷性，推荐全 Docker 化。

#### 步骤 A: 中间件部署

1.  **准备 VPS**: 建议 Linux (CentOS 7+ / Ubuntu 20.04+), 推荐 4 核 8G 以上配置。
2.  **上传文件**: 将 `hula-server/docs/install/docker/` 目录上传至 VPS `/opt/hula/docker`。
3.  **修改配置 (关键)**:
    - `docker-compose.yml` 中的 `srs (CANDIDATE)`: 必须修改为 **VPS 公网 IP**。
    - `broker.conf` (RocketMQ): 必须添加 `brokerIP1=VPS公网IP`，否则客户端连不上 MQ。
4.  **启动**: `docker-compose up -d`。

#### 步骤 B: 业务服务构建与镜像化

在本地(或打包机)执行：

1.  **Maven 编译**:
    ```bash
    cd hula-server
    mvn clean package -DskipTests
    ```
2.  **构建 Docker 镜像** (为每个微服务):
    创建通用 `Dockerfile`:
    ```dockerfile
    FROM openjdk:21-jre-slim
    COPY target/luohuo-*.jar app.jar
    ENTRYPOINT ["java", "-jar", "/app.jar"]
    ```
3.  **上传/推送镜像**: 将镜像 push 到私有仓库 (如阿里云 ACR) 或 save/load 到 VPS。
4.  **启动业务容器**: 编写 `docker-compose-app.yml` 编排 5 个核心服务，注意网络要与中间件在同一 Docker Network (`docker_default`)。

### 4.2 客户端打包 (客户端)

客户端打包依赖于目标平台，通常需要在对应系统上执行。

1.  **准备打包机**:
    - **Windows 电脑**: 用于打包 Windows Exe 和 Android Apk。
    - **Mac 电脑**: 用于打包 macOS App 和 iOS Ipa (必须)。
2.  **环境变量配置**:
    - 在打包机上新建 `.env.production`:
      ```properties
      VITE_BASE_URL=http://<VPS_IP>:8080
      VITE_WS_URL=ws://<VPS_IP>:8888
      ```
3.  **执行打包**:
    - Windows/Android: `pnpm tauri build`
    - iOS/Mac: `pnpm tauri build --target ios` (需 XCode 环境)

### 4.3 自动化流水线 (CI/CD 建议)

虽然初期可手动打包，通过 Jenkins (已在 `docker-compose` 中包含) 可实现自动化：

1.  **Git 触发**: 代码 push 到 `main` 分支。
2.  **Jenkins 流水线**:
    - 拉取代码。
    - 运行 `mvn package`。
    - 运行 `docker build` & `docker push`。
    - SSH 连到 VPS 执行 `docker-compose pull && docker-compose up -d`。

## 5. 潜在风险预警

- **推送问题**: 纯内网环境下，手机端的系统级推送 (APNs/FCM) 可能失效，App 必须常驻后台才能收到消息 (保活是难点)。
- **证书问题**: 如果企业要求 HTTPS，需要自签证书并让客户端信任，Tauri 对自签证书的处理需要额外配置 (`http` vs `https` 作用域)。

## 6. 建议立即执行的步骤

1.  **启动 Docker 环境**: 在目录下运行 `docker-compose up -d`。
2.  **IP 修正**: 检查 `srs` 服务的 `CANDIDATE` 是否为本机局域网 IP。
