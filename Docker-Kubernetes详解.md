# Docker 与 Kubernetes 完全指南

> 从容器基础到云原生编排，系统掌握现代应用部署与运维

---

## 目录

- Part 1:  Docker 基础与架构
- Part 2:  Docker 镜像深度解析
- Part 3:  Docker 容器操作实战
- Part 4:  Docker Compose 实战
- Part 5:  Docker 仓库与镜像管理
- Part 6:  Kubernetes 整体架构
- Part 7:  Kubernetes 核心资源对象
- Part 8:  Kubernetes 网络原理
- Part 9:  Kubernetes 高可用与调度
- Part 10: Kubernetes 存储深度解析
- Part 11: Kubernetes 安全
- Part 12: Helm 包管理
- Part 13: 完整实战案例
- Part 14: kubectl 命令大全
- Part 15: 常见面试题 FAQ

---

# Part 1: Docker 基础与架构

## 1.1 什么是 Docker

Docker 是一个开源的容器化平台，允许开发者将应用程序及其所有依赖项打包到一个轻量级、可移植的容器中。容器可以在任何支持 Docker 的环境中运行，确保"在我机器上能跑"的问题成为历史。

### 虚拟机 vs 容器

```
+---------------------------+       +---------------------------+
|       虚拟机架构           |       |       容器架构             |
+---------------------------+       +---------------------------+
|  App A  |  App B  | App C |       |  App A  |  App B  | App C |
+---------------------------+       +---------------------------+
| Guest OS| Guest OS|Guest OS       |  Libs   |  Libs   | Libs  |
+---------------------------+       +---------------------------+
|       Hypervisor          |       |     Docker Engine         |
+---------------------------+       +---------------------------+
|       Host OS             |       |       Host OS             |
+---------------------------+       +---------------------------+
|       Hardware            |       |       Hardware            |
+---------------------------+       +---------------------------+

虚拟机：每个VM包含完整OS，资源开销大（GB级）
容器：共享宿主机OS内核，资源开销小（MB级）
```

| 特性 | 虚拟机 | 容器 |
|------|--------|------|
| 启动时间 | 分钟级 | 秒级 |
| 存储占用 | GB 级 | MB 级 |
| 性能损耗 | 中等 | 极低 |
| 隔离性 | 强（硬件级） | 中（OS级） |
| 可移植性 | 一般 | 极强 |

## 1.2 Docker 核心架构

```
+---------------------------------------------------------------+
|                    Docker 架构全览                             |
+---------------------------------------------------------------+
|                                                               |
|   Client              Docker Host           Registry          |
|  +--------+          +------------+        +-----------+      |
|  | docker |--build-->|            |        |           |      |
|  | docker |--pull--->|   Docker   |<-pull->|  Docker   |      |
|  | docker |--run---->|   Daemon   |        |    Hub    |      |
|  +--------+          |  (dockerd) |        +-----------+      |
|                      |            |                           |
|                      | +--------+ |                           |
|                      | |Containers|                           |
|                      | |  C1  C2  |                           |
|                      | +--------+ |                           |
|                      | +--------+ |                           |
|                      | | Images  ||                           |
|                      | | img1 2  ||                           |
|                      | +--------+ |                           |
|                      +------------+                           |
+---------------------------------------------------------------+
```

### 核心组件说明

- **Docker Client**：用户与 Docker 交互的命令行工具（`docker` 命令）
- **Docker Daemon (dockerd)**：后台服务，负责构建、运行、管理容器
- **Docker Registry**：镜像仓库，存储和分发镜像（如 Docker Hub、Harbor）
- **Docker Images**：只读模板，用于创建容器
- **Docker Containers**：镜像的运行实例

## 1.3 Linux 内核技术基础

Docker 并非虚拟化技术，而是基于 Linux 内核的以下特性实现隔离：

### 1.3.1 Namespace（命名空间）

Namespace 提供进程隔离，让每个容器认为自己独占系统资源。

```
+------------------------------------------+
|           Linux Namespace 类型            |
+------------+-----------------------------+
|  Namespace |         隔离内容            |
+------------+-----------------------------+
|  PID       | 进程ID，容器内进程从1开始    |
|  NET       | 网络设备、IP、端口、路由表   |
|  MNT       | 文件系统挂载点               |
|  UTS       | 主机名和域名                 |
|  IPC       | 进程间通信资源               |
|  USER      | 用户和组ID映射               |
|  CGROUP    | cgroup根目录（Linux 4.6+）   |
+------------+-----------------------------+
```

**PID Namespace 示例：**

```bash
# 宿主机查看容器内的进程
docker run -d --name myapp nginx
docker exec myapp ps aux
# PID 1 是 nginx master process

# 宿主机上看到的是不同的 PID
ps aux | grep nginx
# 可能是 PID 12345
```

**NET Namespace 示例：**

```bash
# 每个容器有独立网络栈
docker run --rm busybox ip addr
# 只显示容器内的网卡：lo 和 eth0

# 与宿主机完全隔离
ip addr
# 显示宿主机所有网卡
```

### 1.3.2 Cgroups（控制组）

Cgroups 限制容器使用的资源量（CPU、内存、I/O 等）。

```
+----------------------------------------------+
|              Cgroups 层级结构                 |
+----------------------------------------------+
|  /sys/fs/cgroup/                              |
|  ├── memory/                                  |
|  │   ├── docker/                              |
|  │   │   ├── <container-id>/                  |
|  │   │   │   ├── memory.limit_in_bytes        |
|  │   │   │   ├── memory.usage_in_bytes        |
|  │   │   │   └── memory.max_usage_in_bytes    |
|  ├── cpu/                                     |
|  │   ├── docker/                              |
|  │   │   ├── <container-id>/                  |
|  │   │   │   ├── cpu.shares                   |
|  │   │   │   └── cpu.cfs_quota_us             |
|  └── blkio/                                   |
|      └── docker/                              |
+----------------------------------------------+
```

**资源限制示例：**

```bash
# 限制内存为 512MB，CPU 使用率最大 50%
docker run -d \
  --memory="512m" \
  --memory-swap="1g" \
  --cpus="0.5" \
  --name limited-app \
  nginx

# 查看容器资源使用
docker stats limited-app

# 查看 cgroup 实际限制
cat /sys/fs/cgroup/memory/docker/<container-id>/memory.limit_in_bytes
# 输出: 536870912 (512 * 1024 * 1024)
```

### 1.3.3 UnionFS 与 OverlayFS

Union File System 允许将多个目录层叠挂载，是 Docker 镜像分层的核心。

```
+------------------------------------------+
|          OverlayFS 层结构                 |
+------------------------------------------+
|                                          |
|   Container Layer (Read-Write)           |  <-- 容器可写层
|   +------------------------------------+ |
|   | 新增/修改的文件                    | |
|   +------------------------------------+ |
|                  |                       |
|                  v (merged view)         |
|   Image Layer N  (Read-Only)             |  <-- 镜像层N
|   +------------------------------------+ |
|   | /app/config.yml                    | |
|   +------------------------------------+ |
|                  |                       |
|   Image Layer 2  (Read-Only)             |  <-- 镜像层2
|   +------------------------------------+ |
|   | /usr/local/bin/app                 | |
|   +------------------------------------+ |
|                  |                       |
|   Image Layer 1  (Read-Only)             |  <-- 镜像层1（Base）
|   +------------------------------------+ |
|   | Ubuntu 22.04 基础文件系统          | |
|   +------------------------------------+ |
+------------------------------------------+
```

**OverlayFS 工作原理：**

```bash
# OverlayFS 挂载示例
mount -t overlay overlay \
  -o lowerdir=/lower1:/lower2,\   # 只读层（可多个，用冒号分隔）
     upperdir=/upper,\            # 读写层（容器可写层）
     workdir=/work \              # 工作目录（OverlayFS 内部使用）
  /merged                         # 合并后的视图

# Docker 实际使用路径
ls /var/lib/docker/overlay2/
# 每个镜像层和容器层都在这里
```

**Copy-on-Write (CoW) 机制：**

```
读取文件：直接从最上层有该文件的层读取
修改文件：
  1. 检查文件是否在可写层
  2. 若不在，从只读层复制到可写层（copy-up）
  3. 修改可写层中的副本
删除文件：在可写层创建 whiteout 文件（遮蔽下层文件）
```

## 1.4 Docker 安装与配置

### Linux（Ubuntu/Debian）安装

```bash
# 1. 卸载旧版本
sudo apt-get remove docker docker-engine docker.io containerd runc

# 2. 安装依赖
sudo apt-get update
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# 3. 添加 Docker GPG 密钥
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 4. 添加 Docker 仓库
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 5. 安装 Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 6. 启动并设置开机自启
sudo systemctl start docker
sudo systemctl enable docker

# 7. 将当前用户加入 docker 组（避免每次都用 sudo）
sudo usermod -aG docker $USER
newgrp docker

# 8. 验证安装
docker version
docker run hello-world
```

### Docker daemon 配置（/etc/docker/daemon.json）

```json
{
  "registry-mirrors": [
    "https://mirror.ccs.tencentyun.com",
    "https://hub-mirror.c.163.com"
  ],
  "insecure-registries": ["192.168.1.100:5000"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "data-root": "/data/docker",
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 64000,
      "Soft": 64000
    }
  },
  "live-restore": true,
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```

```bash
# 重启 Docker 使配置生效
sudo systemctl daemon-reload
sudo systemctl restart docker
```

---

# Part 2: Docker 镜像深度解析

## 2.1 镜像基本概念

Docker 镜像是一个只读的模板，包含运行应用程序所需的所有内容：代码、运行时、库、环境变量和配置文件。

```
+------------------------------------------+
|            镜像命名规范                   |
+------------------------------------------+
|                                          |
|  [registry/][namespace/]name[:tag]       |
|                                          |
|  示例：                                  |
|  nginx                                   |
|  nginx:1.24                              |
|  nginx:1.24-alpine                       |
|  docker.io/library/nginx:latest          |
|  myregistry.com:5000/myapp:v1.0.0        |
|  gcr.io/google-containers/pause:3.9      |
+------------------------------------------+
```

## 2.2 Dockerfile 完整指令详解

### 基础结构

```dockerfile
# 注释以 # 开头
# escape=\ (可选，设置转义字符)

# 第一条指令必须是 FROM（除 ARG 外）
FROM ubuntu:22.04

# 设置维护者信息（已废弃，推荐用 LABEL）
LABEL maintainer="admin@example.com"
LABEL version="1.0"
LABEL description="My Application"
```

### FROM 指令

```dockerfile
# 基本用法
FROM ubuntu:22.04

# 使用 SHA256 摘要固定版本（生产推荐）
FROM ubuntu@sha256:67211c14fa74f070d27cc59d69a7fa9aeff8e28ea118ef3babc295a0428a6d21

# 多阶段构建命名
FROM golang:1.21 AS builder
FROM alpine:3.18 AS runner

# 空镜像（用于构建最小化镜像）
FROM scratch
```

### ARG 和 ENV 指令

```dockerfile
# ARG：构建时变量，不保留到镜像中
ARG BUILD_VERSION=1.0.0
ARG TARGETARCH

# ENV：运行时环境变量，保留到镜像中
ENV APP_HOME=/app \
    APP_PORT=8080 \
    JAVA_OPTS="-Xms256m -Xmx512m"

# ARG 可以在 FROM 之前使用（用于参数化基础镜像）
ARG BASE_VERSION=22.04
FROM ubuntu:${BASE_VERSION}

# 构建时传入 ARG
# docker build --build-arg BUILD_VERSION=2.0.0 .
```

### COPY 和 ADD 指令

```dockerfile
# COPY：推荐用于复制本地文件
COPY src/ /app/src/
COPY package.json package-lock.json /app/
COPY --chown=node:node . /app/

# 多阶段构建中从其他阶段复制
COPY --from=builder /app/dist /app/dist

# ADD：支持 URL 和自动解压 tar（不推荐用于普通文件复制）
ADD https://example.com/file.tar.gz /tmp/
# 等价于：
# RUN curl -o /tmp/file.tar.gz https://example.com/file.tar.gz && \
#     tar -xzf /tmp/file.tar.gz -C /tmp/ && \
#     rm /tmp/file.tar.gz

# ADD 自动解压
ADD archive.tar.gz /app/
```

### RUN 指令

```dockerfile
# Shell 形式（默认使用 /bin/sh -c）
RUN apt-get update && apt-get install -y curl

# Exec 形式（不经过 shell，推荐）
RUN ["/bin/bash", "-c", "apt-get update && apt-get install -y curl"]

# 最佳实践：合并 RUN 指令减少层数，清理缓存
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        wget \
        vim \
        git \
    && rm -rf /var/lib/apt/lists/*

# 使用 BuildKit 缓存挂载（加速构建）
RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update && apt-get install -y curl

# 使用 BuildKit 密钥挂载（安全传递敏感信息）
RUN --mount=type=secret,id=mysecret \
    cat /run/secrets/mysecret
```

### CMD 和 ENTRYPOINT 指令

```dockerfile
# CMD：容器默认执行命令，可被 docker run 覆盖
# Exec 形式（推荐）
CMD ["nginx", "-g", "daemon off;"]

# Shell 形式
CMD nginx -g "daemon off;"

# ENTRYPOINT：容器入口点，不可轻易被覆盖
ENTRYPOINT ["java"]
CMD ["-jar", "/app/app.jar"]
# docker run myapp 执行: java -jar /app/app.jar
# docker run myapp -jar /other/app.jar 执行: java -jar /other/app.jar

# 结合使用示例
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]
```

**CMD vs ENTRYPOINT 对比：**

```
+------------------+------------------+------------------+
|  Dockerfile      | docker run 参数  | 实际执行命令      |
+------------------+------------------+------------------+
| CMD ["cmd","p1"] | (无)             | cmd p1           |
| CMD ["cmd","p1"] | run_cmd          | run_cmd          |
| ENTRY ["ep","p"] | (无)             | ep p             |
| ENTRY ["ep","p"] | run_cmd          | ep p run_cmd     |
| ENTRY+CMD        | (无)             | ep cmd           |
| ENTRY+CMD        | run_cmd          | ep run_cmd       |
+------------------+------------------+------------------+
```

### EXPOSE、VOLUME、WORKDIR、USER

```dockerfile
# WORKDIR：设置工作目录（不存在则自动创建）
WORKDIR /app

# USER：设置运行用户（安全最佳实践：非 root）
RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser

# EXPOSE：声明容器监听的端口（仅文档作用，需 -p 才能实际映射）
EXPOSE 8080
EXPOSE 8443/tcp
EXPOSE 9090/udp

# VOLUME：声明匿名卷挂载点
VOLUME ["/data", "/logs"]
```

### HEALTHCHECK 指令

```dockerfile
# 健康检查
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# 禁用健康检查
HEALTHCHECK NONE
```

### ONBUILD 和 STOPSIGNAL

```dockerfile
# ONBUILD：当此镜像被用作基础镜像时执行的指令
ONBUILD COPY . /app/src
ONBUILD RUN make /app/src

# STOPSIGNAL：设置容器停止信号
STOPSIGNAL SIGTERM
```

## 2.3 多阶段构建

多阶段构建是减小镜像体积的最重要技术之一。

### Java Spring Boot 多阶段构建

```dockerfile
# ================================
# Stage 1: Build
# ================================
FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /build

# 先复制 pom.xml，利用 Docker 缓存
COPY pom.xml .
RUN mvn dependency:go-offline -B

# 复制源码并构建
COPY src ./src
RUN mvn package -DskipTests -B

# ================================
# Stage 2: Extract layers (JDK 17+)
# ================================
FROM eclipse-temurin:17-jre AS extractor
WORKDIR /extracted
COPY --from=builder /build/target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# ================================
# Stage 3: Runtime
# ================================
FROM eclipse-temurin:17-jre-alpine

# 安全：创建非 root 用户
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# 按变更频率从低到高复制层（优化缓存）
COPY --from=extractor /extracted/dependencies/ ./
COPY --from=extractor /extracted/spring-boot-loader/ ./
COPY --from=extractor /extracted/snapshot-dependencies/ ./
COPY --from=extractor /extracted/application/ ./

USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=30s \
    CMD wget -q --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
```

### Go 应用多阶段构建

```dockerfile
# Stage 1: Build
FROM golang:1.21-alpine AS builder

# 安装依赖工具
RUN apk add --no-cache git ca-certificates tzdata

WORKDIR /build

# 利用模块缓存
COPY go.mod go.sum ./
RUN go mod download

COPY . .

# 静态编译，不依赖 libc
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-w -s" -o /app/server ./cmd/server

# Stage 2: Minimal runtime
FROM scratch

# 从 builder 复制必要文件
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=builder /app/server /server

EXPOSE 8080

ENTRYPOINT ["/server"]
```

### 前端 React 多阶段构建

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder

WORKDIR /app

COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile

COPY . .
RUN yarn build

# Stage 2: Nginx serve
FROM nginx:1.25-alpine

# 复制自定义 nginx 配置
COPY nginx.conf /etc/nginx/conf.d/default.conf

# 复制构建产物
COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

## 2.4 镜像优化技巧

### .dockerignore 文件

```
# .dockerignore
.git
.gitignore
node_modules
npm-debug.log
Dockerfile*
docker-compose*
.env
*.md
.DS_Store
target/
dist/
*.log
coverage/
.idea/
.vscode/
```

### 镜像瘦身对比

```bash
# 未优化版本
FROM ubuntu:22.04
RUN apt-get update
RUN apt-get install -y python3
RUN apt-get install -y pip
COPY app.py .
RUN pip install flask
# 镜像大小：约 450MB

# 优化版本
FROM python:3.11-alpine
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
# 镜像大小：约 50MB
```

### 常见基础镜像选择

| 镜像 | 大小 | 适用场景 |
|------|------|----------|
| ubuntu:22.04 | ~80MB | 需要完整 apt 工具链 |
| debian:bookworm-slim | ~75MB | 轻量 Debian |
| alpine:3.18 | ~7MB | 极致精简，musl libc |
| distroless/base | ~20MB | 安全，无 shell |
| scratch | 0MB | 静态编译二进制 |
| busybox | ~5MB | 调试用，含基本命令 |

## 2.5 镜像管理命令

```bash
# 拉取镜像
docker pull nginx:1.25
docker pull nginx@sha256:abc123...   # 使用摘要

# 查看本地镜像
docker images
docker image ls --filter "dangling=true"   # 悬空镜像
docker image ls --format "{{.Repository}}:{{.Tag}}\t{{.Size}}"

# 查看镜像层
docker history nginx:1.25
docker image inspect nginx:1.25

# 标记镜像
docker tag nginx:1.25 myregistry.com/nginx:1.25

# 推送镜像
docker push myregistry.com/nginx:1.25

# 删除镜像
docker rmi nginx:1.25
docker image rm $(docker image ls -q --filter "dangling=true")  # 删除悬空镜像
docker image prune -a   # 删除所有未使用镜像

# 导出/导入镜像
docker save -o nginx.tar nginx:1.25
docker load -i nginx.tar

# 从容器导出文件系统（不含元数据）
docker export container_id -o container.tar
docker import container.tar myimage:latest

# 构建镜像
docker build -t myapp:1.0 .
docker build -t myapp:1.0 -f Dockerfile.prod .
docker build --no-cache -t myapp:1.0 .
docker build --build-arg VERSION=1.0 -t myapp:1.0 .

# 使用 BuildKit（推荐）
DOCKER_BUILDKIT=1 docker build -t myapp:1.0 .
# 或在 daemon.json 中设置 "features": {"buildkit": true}
```

---

# Part 3: Docker 容器操作实战

## 3.1 容器生命周期

```
+-------+    docker create    +---------+
|       |-------------------->|         |
| 镜像  |                     | Created |
|       |    docker run       |         |
+-------+-------------------->+---------+
                                   |
                              docker start
                                   |
                                   v
+----------+  docker pause  +---------+  docker stop  +---------+
|          |<---------------|         |-------------->|         |
| Paused   |                | Running |               | Stopped |
|          |--------------->|         |<--------------|         |
+----------+ docker unpause +---------+  docker start +---------+
                                |                          |
                           docker kill               docker rm
                                |                          |
                                v                          v
                           +---------+              (容器删除)
                           |  Dead   |
                           +---------+
```

## 3.2 容器运行命令详解

```bash
# 基本运行
docker run nginx

# 后台运行
docker run -d nginx

# 交互式运行
docker run -it ubuntu:22.04 bash
docker run -it --rm ubuntu:22.04 bash   # 退出后自动删除

# 命名容器
docker run -d --name my-nginx nginx

# 端口映射
docker run -d -p 8080:80 nginx             # 宿主机:容器
docker run -d -p 127.0.0.1:8080:80 nginx   # 只绑定本地回环
docker run -d -P nginx                      # 自动映射所有 EXPOSE 端口

# 环境变量
docker run -d \
    -e MYSQL_ROOT_PASSWORD=secret \
    -e MYSQL_DATABASE=mydb \
    mysql:8.0

# 从文件读取环境变量
docker run -d --env-file .env myapp

# 卷挂载
docker run -d \
    -v /host/data:/container/data \
    -v myvolume:/app/data \
    myapp

# 网络设置
docker run -d --network mynetwork myapp
docker run -d --network host nginx         # 使用宿主机网络
docker run -d --network none myapp         # 无网络

# 资源限制
docker run -d \
    --memory="512m" \
    --memory-swap="1g" \
    --cpus="1.5" \
    --cpu-shares=512 \
    myapp

# 重启策略
docker run -d --restart=always nginx
docker run -d --restart=on-failure:3 myapp
# 可选值: no | always | on-failure[:max-retries] | unless-stopped

# 特权模式与能力
docker run --privileged myapp           # 完全特权（生产慎用）
docker run --cap-add SYS_PTRACE myapp   # 添加特定能力
docker run --cap-drop ALL myapp         # 删除所有能力

# 安全选项
docker run --security-opt=no-new-privileges myapp
docker run --read-only myapp            # 只读根文件系统

# 设置主机名和 DNS
docker run -d --hostname myserver --dns 8.8.8.8 myapp

# 挂载配置
docker run -d \
    --mount type=bind,source=/host/path,target=/container/path \
    --mount type=volume,source=myvolume,target=/data \
    --mount type=tmpfs,target=/tmp \
    myapp
```

## 3.3 容器管理命令

```bash
# 查看容器
docker ps                    # 运行中的容器
docker ps -a                 # 所有容器
docker ps -q                 # 只显示 ID
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# 容器详细信息
docker inspect container_id
docker inspect -f '{{.NetworkSettings.IPAddress}}' container_id
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_id

# 容器日志
docker logs container_id
docker logs -f container_id           # 实时跟踪
docker logs --tail=100 container_id   # 最后100行
docker logs --since="2024-01-01" container_id
docker logs --since=1h container_id   # 最近1小时

# 在容器内执行命令
docker exec -it container_id bash
docker exec container_id ls /app
docker exec -u root container_id bash  # 以 root 执行

# 容器资源统计
docker stats
docker stats container_id
docker stats --no-stream   # 只输出一次

# 停止/启动/重启
docker stop container_id           # SIGTERM，超时后 SIGKILL（默认10s）
docker stop -t 30 container_id     # 自定义超时
docker kill container_id           # 直接 SIGKILL
docker start container_id
docker restart container_id

# 暂停/恢复（SIGSTOP/SIGCONT）
docker pause container_id
docker unpause container_id

# 删除容器
docker rm container_id
docker rm -f container_id           # 强制删除运行中的容器
docker rm $(docker ps -aq)          # 删除所有停止的容器
docker container prune              # 清理所有停止的容器

# 复制文件
docker cp container_id:/app/log.txt ./log.txt   # 容器→宿主机
docker cp ./config.yml container_id:/app/        # 宿主机→容器

# 查看容器文件变更
docker diff container_id
# A=Added, C=Changed, D=Deleted

# 提交容器为镜像（不推荐生产使用）
docker commit -m "add config" container_id myimage:v2

# 容器端口
docker port container_id
docker port container_id 80
```

## 3.4 Docker 网络

### 网络模式

```
+------------------------------------------------+
|              Docker 网络模式                    |
+------------------+-----------------------------+
|  网络模式        |  说明                       |
+------------------+-----------------------------+
|  bridge          | 默认模式，虚拟网桥           |
|  host            | 共享宿主机网络栈             |
|  none            | 无网络连接                  |
|  container:<id>  | 共享另一个容器的网络栈       |
|  overlay         | 跨主机容器网络（Swarm用）    |
|  macvlan         | 容器拥有独立MAC地址          |
+------------------+-----------------------------+
```

```bash
# 查看网络
docker network ls
docker network inspect bridge

# 创建自定义网络
docker network create mynetwork
docker network create \
    --driver bridge \
    --subnet 172.20.0.0/16 \
    --ip-range 172.20.240.0/20 \
    --gateway 172.20.0.1 \
    mynetwork

# 连接/断开容器网络
docker network connect mynetwork container_id
docker network disconnect mynetwork container_id

# 删除网络
docker network rm mynetwork
docker network prune   # 清理未使用网络
```

**自定义 bridge 网络的优势：**

```bash
# 创建自定义网络后，容器间可以通过名称互相访问
docker network create appnet

docker run -d --name mysql --network appnet mysql:8.0
docker run -d --name webapp --network appnet myapp

# webapp 可以直接通过 "mysql" 主机名访问数据库
# 默认 bridge 网络不支持此功能
```

## 3.5 Docker 数据持久化

### Volume（卷）

```bash
# 创建卷
docker volume create myvolume
docker volume create \
    --driver local \
    --opt type=nfs \
    --opt o=addr=192.168.1.1,rw \
    --opt device=:/path/to/dir \
    nfsvolume

# 查看卷
docker volume ls
docker volume inspect myvolume
# 数据位置：/var/lib/docker/volumes/myvolume/_data

# 使用卷
docker run -d -v myvolume:/data myapp
docker run -d --mount source=myvolume,target=/data myapp

# 删除卷
docker volume rm myvolume
docker volume prune   # 删除所有未使用卷
```

### Bind Mount（绑定挂载）

```bash
# 绑定挂载
docker run -d \
    -v /host/absolute/path:/container/path \
    myapp

# 只读绑定挂载
docker run -d \
    -v /host/config:/app/config:ro \
    myapp
```

### tmpfs 挂载

```bash
# tmpfs：内存文件系统，容器停止后数据消失
docker run -d \
    --mount type=tmpfs,target=/tmp,tmpfs-size=100m \
    myapp
```

---

# Part 4: Docker Compose 实战

## 4.1 Docker Compose 概述

Docker Compose 是用于定义和运行多容器 Docker 应用的工具，通过 YAML 文件描述服务配置。

```yaml
# docker-compose.yml 基本结构
version: '3.9'   # Compose 文件版本

services:        # 服务定义
  web:
    image: nginx:1.25
  db:
    image: mysql:8.0

volumes:         # 卷定义
  dbdata:

networks:        # 网络定义
  appnet:
```

## 4.2 完整 Spring Boot + MySQL + Redis 案例

### 项目结构

```
my-project/
├── docker-compose.yml
├── docker-compose.override.yml    # 开发环境覆盖
├── docker-compose.prod.yml        # 生产环境配置
├── .env                           # 环境变量文件
├── app/
│   ├── Dockerfile
│   └── src/
├── nginx/
│   ├── Dockerfile
│   └── nginx.conf
└── mysql/
    └── init/
        └── 01-schema.sql
```

### .env 文件

```bash
# .env
COMPOSE_PROJECT_NAME=myapp

# App
APP_VERSION=1.0.0
APP_PORT=8080
SPRING_PROFILES_ACTIVE=docker

# MySQL
MYSQL_ROOT_PASSWORD=rootsecret
MYSQL_DATABASE=myappdb
MYSQL_USER=appuser
MYSQL_PASSWORD=appsecret
MYSQL_PORT=3306

# Redis
REDIS_PASSWORD=redissecret
REDIS_PORT=6379

# Nginx
NGINX_PORT=80
NGINX_HTTPS_PORT=443
```

### docker-compose.yml（完整版）

```yaml
version: '3.9'

services:
  # ================================
  # Nginx 反向代理
  # ================================
  nginx:
    image: nginx:1.25-alpine
    container_name: myapp-nginx
    restart: unless-stopped
    ports:
      - "${NGINX_PORT:-80}:80"
      - "${NGINX_HTTPS_PORT:-443}:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - nginx_logs:/var/log/nginx
    depends_on:
      app:
        condition: service_healthy
    networks:
      - frontend
    labels:
      - "com.example.service=nginx"

  # ================================
  # Spring Boot 应用
  # ================================
  app:
    build:
      context: ./app
      dockerfile: Dockerfile
      args:
        - APP_VERSION=${APP_VERSION:-1.0.0}
    image: myapp:${APP_VERSION:-latest}
    container_name: myapp-backend
    restart: unless-stopped
    environment:
      - SPRING_PROFILES_ACTIVE=${SPRING_PROFILES_ACTIVE:-docker}
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/${MYSQL_DATABASE}?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
      - SPRING_DATASOURCE_USERNAME=${MYSQL_USER}
      - SPRING_DATASOURCE_PASSWORD=${MYSQL_PASSWORD}
      - SPRING_REDIS_HOST=redis
      - SPRING_REDIS_PORT=6379
      - SPRING_REDIS_PASSWORD=${REDIS_PASSWORD}
      - JAVA_OPTS=-Xms256m -Xmx512m -XX:+UseG1GC
    volumes:
      - app_logs:/app/logs
      - app_uploads:/app/uploads
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s
    networks:
      - frontend
      - backend
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 768M
        reservations:
          cpus: '0.5'
          memory: 256M
    labels:
      - "com.example.service=app"

  # ================================
  # MySQL 数据库
  # ================================
  mysql:
    image: mysql:8.0
    container_name: myapp-mysql
    restart: unless-stopped
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
      - TZ=Asia/Shanghai
    volumes:
      - mysql_data:/var/lib/mysql
      - ./mysql/init:/docker-entrypoint-initdb.d:ro
      - ./mysql/conf/my.cnf:/etc/mysql/conf.d/my.cnf:ro
    ports:
      - "127.0.0.1:${MYSQL_PORT:-3306}:3306"   # 只暴露给本地
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s
    networks:
      - backend
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --default-authentication-plugin=mysql_native_password
      - --max_connections=500
    labels:
      - "com.example.service=mysql"

  # ================================
  # Redis 缓存
  # ================================
  redis:
    image: redis:7.2-alpine
    container_name: myapp-redis
    restart: unless-stopped
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
      --appendonly yes
      --appendfsync everysec
    volumes:
      - redis_data:/data
    ports:
      - "127.0.0.1:${REDIS_PORT:-6379}:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend
    labels:
      - "com.example.service=redis"

  # ================================
  # 数据库管理工具（可选）
  # ================================
  adminer:
    image: adminer:4.8.1
    container_name: myapp-adminer
    restart: unless-stopped
    ports:
      - "8090:8080"
    depends_on:
      - mysql
    networks:
      - backend
    profiles:
      - debug   # 只在 debug profile 时启动

volumes:
  mysql_data:
    driver: local
  redis_data:
    driver: local
  app_logs:
    driver: local
  app_uploads:
    driver: local
  nginx_logs:
    driver: local

networks:
  frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24
  backend:
    driver: bridge
    internal: true   # 内部网络，不能访问外网
    ipam:
      config:
        - subnet: 172.20.1.0/24
```

### nginx.conf

```nginx
upstream backend {
    server app:8080;
    keepalive 32;
}

server {
    listen 80;
    server_name _;

    # 静态文件
    location /static/ {
        alias /usr/share/nginx/html/static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # API 代理
    location /api/ {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 10s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # 前端应用
    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ /index.html;
    }
}
```

## 4.3 Docker Compose 命令

```bash
# 启动服务
docker compose up           # 前台启动
docker compose up -d        # 后台启动
docker compose up -d app    # 只启动指定服务

# 停止服务
docker compose down             # 停止并删除容器、网络
docker compose down -v          # 同时删除卷
docker compose down --rmi all   # 同时删除镜像

# 构建/重新构建
docker compose build
docker compose build --no-cache app
docker compose up -d --build    # 构建后启动

# 查看状态
docker compose ps
docker compose logs
docker compose logs -f app
docker compose top

# 扩展服务实例
docker compose up -d --scale app=3

# 执行命令
docker compose exec app bash
docker compose run --rm app python manage.py migrate

# 使用不同配置文件
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# 使用 profile
docker compose --profile debug up -d

# 验证配置
docker compose config
```

---

# Part 5: Docker 仓库与镜像管理

## 5.1 Docker Hub

```bash
# 登录 Docker Hub
docker login
docker login -u myusername

# 推送镜像
docker tag myapp:1.0 myusername/myapp:1.0
docker push myusername/myapp:1.0

# 退出登录
docker logout
```

## 5.2 搭建私有仓库（Registry）

### 使用官方 Registry

```bash
# 启动私有仓库
docker run -d \
    --name registry \
    -p 5000:5000 \
    --restart=always \
    -v /data/registry:/var/lib/registry \
    -e REGISTRY_STORAGE_DELETE_ENABLED=true \
    registry:2

# 推送到私有仓库
docker tag myapp:1.0 localhost:5000/myapp:1.0
docker push localhost:5000/myapp:1.0

# 从私有仓库拉取
docker pull localhost:5000/myapp:1.0

# 查看仓库镜像列表（API）
curl http://localhost:5000/v2/_catalog
curl http://localhost:5000/v2/myapp/tags/list
```

## 5.3 Harbor 企业级镜像仓库

Harbor 是 CNCF 毕业项目，提供企业级镜像仓库功能。

### Harbor 架构

```
+-----------------------------------------------------+
|                  Harbor 架构                         |
+-----------------------------------------------------+
|                                                     |
|   Client (Browser/Docker CLI)                       |
|          |                                          |
|          v                                          |
|   +-------------+    +----------+                  |
|   |  Nginx      |--->| Portal   | (Web UI)          |
|   |  (Proxy)   |    +----------+                  |
|   +------+------+         |                        |
|          |                v                        |
|   +------+------+    +----------+                  |
|   |  Core       |    | JobService|                  |
|   |  Service    |    | (扫描/复制)|                  |
|   +------+------+    +----------+                  |
|          |                                         |
|   +------+---+--------+----------+                 |
|   | Registry | DB     | Redis    |                 |
|   | (镜像存储)| (Postgres)|(缓存)  |                |
|   +----------+--------+----------+                 |
|          |                                         |
|   +------+------+                                  |
|   |  Storage    | (local/S3/OSS/GCS)               |
|   +-------------+                                  |
+-----------------------------------------------------+
```

### Harbor 安装（docker-compose）

```bash
# 1. 下载 Harbor 安装包
wget https://github.com/goharbor/harbor/releases/download/v2.9.0/harbor-online-installer-v2.9.0.tgz
tar xvf harbor-online-installer-v2.9.0.tgz
cd harbor

# 2. 配置 harbor.yml
cp harbor.yml.tmpl harbor.yml
```

```yaml
# harbor.yml 关键配置
hostname: harbor.example.com

http:
  port: 80

https:
  port: 443
  certificate: /your/certificate/path
  private_key: /your/private/key/path

harbor_admin_password: Harbor12345

database:
  password: root123
  max_idle_conns: 100
  max_open_conns: 900

data_volume: /data

log:
  level: info
  local:
    rotate_count: 50
    rotate_size: 200M
    location: /var/log/harbor
```

```bash
# 3. 安装并启动
sudo ./install.sh --with-trivy   # 带漏洞扫描

# 4. 配置 Docker 使用 Harbor（insecure registry 或配置证书）
# /etc/docker/daemon.json
{
  "insecure-registries": ["harbor.example.com"]
}
sudo systemctl restart docker

# 5. 使用 Harbor
docker login harbor.example.com
docker tag myapp:1.0 harbor.example.com/myproject/myapp:1.0
docker push harbor.example.com/myproject/myapp:1.0
```

### Harbor 功能特性

| 功能 | 说明 |
|------|------|
| 多项目隔离 | 按项目划分镜像存储空间和访问权限 |
| RBAC | 基于角色的访问控制（管理员/开发者/游客） |
| 镜像扫描 | 集成 Trivy/Clair 扫描漏洞 |
| 内容信任 | Docker Notary 镜像签名 |
| 镜像复制 | 跨 Harbor 实例同步镜像 |
| Webhook | 事件通知（推送/扫描完成等） |
| 垃圾回收 | 清理未引用的镜像层 |
| 审计日志 | 记录所有操作 |

---

# Part 6: Kubernetes 整体架构

## 6.1 Kubernetes 简介

Kubernetes（简称 K8s）是 Google 开源的容器编排系统，用于自动化容器的部署、扩展和管理。

```
+------------------------------------------------------------------+
|                    Kubernetes 核心能力                            |
+------------------------------------------------------------------+
|                                                                  |
|  服务发现与负载均衡    自动装箱（bin packing）    自我修复          |
|  +--------------+    +--------------------+    +-------------+  |
|  | DNS/Service  |    | 根据资源请求调度容器|    | 重启失败容器 |  |
|  +--------------+    +--------------------+    +-------------+  |
|                                                                  |
|  水平自动扩展         密钥与配置管理          存储编排             |
|  +--------------+    +--------------------+    +-------------+  |
|  | HPA/VPA      |    | Secret/ConfigMap   |    | PV/PVC/CSI  |  |
|  +--------------+    +--------------------+    +-------------+  |
|                                                                  |
|  滚动更新与回滚       批量执行               服务网格集成          |
|  +--------------+    +--------------------+    +-------------+  |
|  | Deployment   |    | Job/CronJob        |    | Istio/Linkerd|  |
|  +--------------+    +--------------------+    +-------------+  |
+------------------------------------------------------------------+
```

## 6.2 集群架构全览

```
+------------------------------------------------------------------+
|                   Kubernetes 集群架构                             |
+------------------------------------------------------------------+
|                                                                  |
|  +--------------------------+  +--------------------------+      |
|  |     Control Plane        |  |      Worker Node 1       |      |
|  |  (Master Node)           |  |                          |      |
|  |                          |  |  +--------------------+  |      |
|  |  +--------------------+  |  |  |      kubelet       |  |      |
|  |  |   kube-apiserver   |<-+--+->|  (node agent)      |  |      |
|  |  +--------------------+  |  |  +--------------------+  |      |
|  |  +--------------------+  |  |  +--------------------+  |      |
|  |  | kube-controller-   |  |  |  |   kube-proxy       |  |      |
|  |  |     manager        |  |  |  |  (iptables/ipvs)   |  |      |
|  |  +--------------------+  |  |  +--------------------+  |      |
|  |  +--------------------+  |  |  +--------------------+  |      |
|  |  |  kube-scheduler    |  |  |  | Container Runtime  |  |      |
|  |  +--------------------+  |  |  | (containerd/CRI-O) |  |      |
|  |  +--------------------+  |  |  +--------------------+  |      |
|  |  |       etcd         |  |  |  +--------------------+  |      |
|  |  |  (分布式KV存储)     |  |  |  |   Pods             |  |      |
|  |  +--------------------+  |  |  | [C1][C2] [C1][C2]  |  |      |
|  |  +--------------------+  |  |  +--------------------+  |      |
|  |  | cloud-controller-  |  |  +--------------------------+      |
|  |  |     manager        |  |                                    |
|  |  +--------------------+  |  +--------------------------+      |
|  +--------------------------+  |      Worker Node 2       |      |
|                                |  (same components)       |      |
|                                +--------------------------+      |
+------------------------------------------------------------------+
```

## 6.3 Control Plane 组件详解

### kube-apiserver

API Server 是 Kubernetes 的前端，所有操作都通过它进行。

```
+-------------------------------------------------------+
|                  kube-apiserver 请求流程               |
+-------------------------------------------------------+
|                                                       |
|  kubectl/SDK    -->   Authentication                  |
|                       (认证: JWT/X.509/Bearer)        |
|                            |                          |
|                            v                          |
|                       Authorization                   |
|                       (授权: RBAC/ABAC/Webhook)       |
|                            |                          |
|                            v                          |
|                    Admission Control                  |
|                    (准入控制: 变更/验证)                |
|                            |                          |
|                            v                          |
|                    Validation & Storage               |
|                    (验证资源规范，存入 etcd)            |
|                            |                          |
|                            v                          |
|                    Watch/Notify                       |
|                    (通知各控制器)                      |
+-------------------------------------------------------+
```

**常用准入控制器：**

| 准入控制器 | 功能 |
|-----------|------|
| NamespaceLifecycle | 阻止在终止中的命名空间创建资源 |
| LimitRanger | 为未设置资源限制的 Pod 添加默认值 |
| ResourceQuota | 强制执行命名空间资源配额 |
| ServiceAccount | 自动挂载 ServiceAccount token |
| PodSecurity | 执行 Pod 安全标准 |
| MutatingAdmissionWebhook | 调用外部 webhook 修改资源 |
| ValidatingAdmissionWebhook | 调用外部 webhook 验证资源 |

### kube-scheduler

调度器负责将 Pod 分配到合适的节点。

```
+-----------------------------------------------+
|           kube-scheduler 调度流程              |
+-----------------------------------------------+
|                                               |
|  未调度的 Pod                                  |
|       |                                       |
|       v                                       |
|  +------------------+                         |
|  |  过滤阶段 (Filter)|                         |
|  | 找出可用节点       |                         |
|  | - NodeSelector    |                         |
|  | - NodeAffinity    |                         |
|  | - Taints/Tolerations|                      |
|  | - 资源是否满足     |                         |
|  | - 端口是否冲突     |                         |
|  +--------+---------+                         |
|           |                                   |
|           v  (可用节点列表)                    |
|  +------------------+                         |
|  |  打分阶段 (Score)|                          |
|  | 为每个节点打分     |                         |
|  | - 资源均衡        |                         |
|  | - 亲和性优先      |                         |
|  | - 镜像本地化      |                         |
|  +--------+---------+                         |
|           |                                   |
|           v  (最高分节点)                      |
|  +------------------+                         |
|  |  绑定 (Bind)      |                        |
|  | 写入 Pod.spec.    |                        |
|  | nodeName          |                        |
|  +------------------+                         |
+-----------------------------------------------+
```

### kube-controller-manager

控制器管理器运行各种控制器，确保集群状态符合期望状态。

```
+----------------------------------------------+
|       kube-controller-manager 内置控制器      |
+----------------------------------------------+
|                                              |
|  Node Controller          管理节点状态        |
|  ReplicaSet Controller    确保副本数量        |
|  Deployment Controller    管理滚动更新        |
|  StatefulSet Controller   管理有状态应用      |
|  DaemonSet Controller     确保每节点运行      |
|  Job Controller           管理批处理任务      |
|  CronJob Controller       管理定时任务        |
|  Service Controller       管理服务（LoadBalancer）|
|  Endpoint Controller      维护 Endpoints     |
|  Namespace Controller     管理命名空间生命周期 |
|  PersistentVolume Controller  管理 PV 生命周期 |
|  ServiceAccount Controller  自动创建默认SA    |
+----------------------------------------------+
```

### etcd

etcd 是 Kubernetes 的数据存储后端，是一个高可用的分布式键值存储。

```bash
# etcd 生产部署建议
# 1. 使用奇数个节点（3 或 5）
# 2. 使用 SSD 存储
# 3. 定期备份

# etcd 备份
ETCDCTL_API=3 etcdctl snapshot save backup.db \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key

# etcd 恢复
ETCDCTL_API=3 etcdctl snapshot restore backup.db \
    --data-dir=/var/lib/etcd-restore \
    --initial-cluster=master=https://127.0.0.1:2380 \
    --initial-advertise-peer-urls=https://127.0.0.1:2380 \
    --name=master

# 查看 etcd 集群状态
ETCDCTL_API=3 etcdctl member list \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key
```

## 6.4 Worker Node 组件

### kubelet

kubelet 是运行在每个节点上的 agent，负责管理 Pod 生命周期。

```
+---------------------------------------------+
|              kubelet 工作流程                |
+---------------------------------------------+
|                                             |
|  API Server                                 |
|      |                                      |
|      v  (PodSpec)                           |
|  kubelet                                    |
|      |                                      |
|      +---> 调用 CRI (Container Runtime)     |
|      |     创建/删除容器                     |
|      |                                      |
|      +---> 调用 CNI (Network Plugin)        |
|      |     配置 Pod 网络                     |
|      |                                      |
|      +---> 调用 CSI/Volume Plugin           |
|      |     挂载存储卷                        |
|      |                                      |
|      +---> 执行健康检查                      |
|      |     (liveness/readiness/startup)     |
|      |                                      |
|      +---> 上报节点和 Pod 状态               |
+---------------------------------------------+
```

### kube-proxy

kube-proxy 维护节点上的网络规则，实现 Service 的负载均衡。

```
iptables 模式（默认）：
  Service ClusterIP --> iptables DNAT 规则 --> Pod IP

ipvs 模式（推荐生产）：
  Service ClusterIP --> IPVS 虚拟服务 --> Pod IP
  优势：O(1) 规则查找（vs iptables 的 O(n)），支持多种 LB 算法
```

```bash
# 查看 kube-proxy 模式
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode

# 查看 ipvs 规则
ipvsadm -ln

# 查看 iptables 规则
iptables -t nat -L KUBE-SERVICES -n | head -30
```

## 6.5 kubectl 工作原理

```
+----------------------------------------------------------+
|                  kubectl 请求流程                         |
+----------------------------------------------------------+
|                                                          |
|  1. 用户执行: kubectl get pods                           |
|                                                          |
|  2. kubectl 读取 ~/.kube/config                          |
|     - 确定 cluster (API Server 地址)                     |
|     - 确定 user (证书/token)                             |
|     - 确定 context (cluster + user + namespace)          |
|                                                          |
|  3. kubectl 发送 HTTP 请求                               |
|     GET https://api-server:6443/api/v1/namespaces/       |
|          default/pods                                    |
|     Authorization: Bearer <token>                        |
|                                                          |
|  4. API Server 处理:                                     |
|     认证 -> 授权 -> 准入控制 -> etcd 查询                 |
|                                                          |
|  5. 返回 JSON 响应                                       |
|                                                          |
|  6. kubectl 格式化输出                                   |
+----------------------------------------------------------+
```

```bash
# kubeconfig 管理
kubectl config view
kubectl config get-contexts
kubectl config use-context my-context
kubectl config set-context --current --namespace=myns

# 多集群管理
export KUBECONFIG=~/.kube/config:~/.kube/config-prod
kubectl config view --merge --flatten > ~/.kube/config-merged
```

## 6.6 Kubernetes 安装方式

### kubeadm 安装（推荐生产）

```bash
# === 所有节点执行 ===

# 1. 禁用 swap
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

# 2. 配置内核模块
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# 3. 配置内核参数
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# 4. 安装 containerd
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
# 修改 SystemdCgroup = true
sudo systemctl restart containerd

# 5. 安装 kubeadm, kubelet, kubectl
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | \
    sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | \
    sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# === 仅 Master 节点执行 ===

# 6. 初始化集群
sudo kubeadm init \
    --pod-network-cidr=10.244.0.0/16 \
    --service-cidr=10.96.0.0/12 \
    --apiserver-advertise-address=192.168.1.100 \
    --kubernetes-version=v1.28.0

# 7. 配置 kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 8. 安装网络插件（以 Flannel 为例）
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# === 仅 Worker 节点执行 ===

# 9. 加入集群（使用 kubeadm init 输出的命令）
sudo kubeadm join 192.168.1.100:6443 \
    --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:xxxx

# 验证集群
kubectl get nodes
kubectl get pods -A
```

---

# Part 7: Kubernetes 核心资源对象

## 7.1 Pod

Pod 是 Kubernetes 的最小部署单元，封装一个或多个容器。

```yaml
# pod.yaml - 完整示例
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  namespace: default
  labels:
    app: myapp
    version: "1.0"
  annotations:
    description: "My application pod"
spec:
  # 初始化容器（在主容器前运行）
  initContainers:
  - name: init-db
    image: busybox:1.35
    command: ['sh', '-c', 'until nc -z mysql 3306; do echo waiting for mysql; sleep 2; done']

  containers:
  - name: myapp
    image: myapp:1.0
    ports:
    - containerPort: 8080
      name: http
    - containerPort: 9090
      name: metrics

    # 环境变量
    env:
    - name: DB_HOST
      value: "mysql"
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    - name: MY_POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name   # Downward API

    # 资源限制
    resources:
      requests:
        memory: "128Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "1000m"

    # 存活探针
    livenessProbe:
      httpGet:
        path: /actuator/health/liveness
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
      failureThreshold: 3

    # 就绪探针
    readinessProbe:
      httpGet:
        path: /actuator/health/readiness
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 3

    # 启动探针（K8s 1.16+）
    startupProbe:
      httpGet:
        path: /actuator/health
        port: 8080
      failureThreshold: 30
      periodSeconds: 10

    # 卷挂载
    volumeMounts:
    - name: config
      mountPath: /app/config
    - name: logs
      mountPath: /app/logs

    # 安全上下文
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL

    # 生命周期钩子
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo container started"]
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 5"]  # 优雅停机

  # 卷定义
  volumes:
  - name: config
    configMap:
      name: myapp-config
  - name: logs
    emptyDir: {}

  # Pod 级别安全上下文
  securityContext:
    fsGroup: 2000
    runAsNonRoot: true

  # 调度相关
  nodeSelector:
    kubernetes.io/os: linux
    node-type: worker

  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-type
            operator: In
            values:
            - high-mem
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: myapp
          topologyKey: kubernetes.io/hostname

  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"

  # 重启策略
  restartPolicy: Always   # Always | OnFailure | Never

  # 终止宽限期
  terminationGracePeriodSeconds: 30

  # DNS 配置
  dnsPolicy: ClusterFirst
  dnsConfig:
    nameservers:
    - 1.2.3.4
    options:
    - name: ndots
      value: "2"
```

## 7.2 Deployment

Deployment 管理无状态应用的滚动更新和回滚。

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
  labels:
    app: myapp
spec:
  replicas: 3

  selector:
    matchLabels:
      app: myapp

  # 更新策略
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 最多超出 replicas 的 Pod 数
      maxUnavailable: 0  # 最多不可用 Pod 数（0 = 零停机更新）

  # Pod 模板
  template:
    metadata:
      labels:
        app: myapp
        version: "1.0"
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5

  # 历史版本保留数（用于回滚）
  revisionHistoryLimit: 10

  # 最短就绪时间（超过此时间才认为可用）
  minReadySeconds: 5

  # 进度截止时间
  progressDeadlineSeconds: 600
```

```bash
# Deployment 操作
kubectl apply -f deployment.yaml
kubectl get deployment myapp
kubectl describe deployment myapp

# 扩缩容
kubectl scale deployment myapp --replicas=5
kubectl autoscale deployment myapp --min=2 --max=10 --cpu-percent=80

# 更新镜像
kubectl set image deployment/myapp myapp=myapp:2.0
kubectl set image deployment/myapp myapp=myapp:2.0 --record  # 记录变更原因

# 查看滚动更新状态
kubectl rollout status deployment/myapp

# 查看版本历史
kubectl rollout history deployment/myapp
kubectl rollout history deployment/myapp --revision=2

# 回滚
kubectl rollout undo deployment/myapp             # 回滚到上一版本
kubectl rollout undo deployment/myapp --to-revision=2  # 回滚到指定版本

# 暂停/继续滚动更新
kubectl rollout pause deployment/myapp
kubectl rollout resume deployment/myapp

# 重启 Deployment（触发滚动更新）
kubectl rollout restart deployment/myapp
```

## 7.3 StatefulSet

StatefulSet 管理有状态应用，提供稳定的网络标识和持久存储。

```yaml
# statefulset.yaml - MySQL 主从示例
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: "mysql"    # 必须对应一个 Headless Service
  replicas: 3
  podManagementPolicy: OrderedReady  # Parallel | OrderedReady

  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0  # 从第 0 个 Pod 开始更新

  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:8.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # 根据 Pod 序号设置 server-id
          [[ $HOSTNAME =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          # 0 号 Pod 为 master，其余为 slave
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/master.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/slave.cnf /mnt/conf.d/
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map

      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1

      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql-config

  # 为每个 Pod 自动创建 PVC
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "fast-ssd"
      resources:
        requests:
          storage: 20Gi

---
# StatefulSet 需要的 Headless Service
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None   # Headless Service
  selector:
    app: mysql
  ports:
  - port: 3306
    name: mysql
```

**StatefulSet Pod 标识：**

```
Pod 名称: mysql-0, mysql-1, mysql-2
DNS 记录: mysql-0.mysql.default.svc.cluster.local
          mysql-1.mysql.default.svc.cluster.local
PVC 名称: data-mysql-0, data-mysql-1, data-mysql-2
```

## 7.4 DaemonSet

DaemonSet 确保每个节点（或指定节点）上运行一个 Pod 副本。

```yaml
# daemonset.yaml - 日志收集示例
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: filebeat
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        name: filebeat
    spec:
      tolerations:
      # 允许在 master 节点运行
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: filebeat
        image: elastic/filebeat:8.10.0
        args: ["-c", "/etc/filebeat.yml", "-e"]
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat.yml
          subPath: filebeat.yml
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
      volumes:
      - name: config
        configMap:
          name: filebeat-config
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      terminationGracePeriodSeconds: 30
```

## 7.5 Job 和 CronJob

```yaml
# job.yaml - 数据库迁移任务
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  completions: 1          # 需要成功完成的次数
  parallelism: 1          # 并行 Pod 数
  backoffLimit: 3         # 失败重试次数
  activeDeadlineSeconds: 600  # 超时时间
  ttlSecondsAfterFinished: 3600  # 完成后保留时间

  template:
    spec:
      restartPolicy: Never  # Job 必须是 Never 或 OnFailure
      containers:
      - name: migration
        image: myapp:1.0
        command: ["java", "-jar", "migration.jar"]
        env:
        - name: DB_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url

---
# cronjob.yaml - 定时备份任务
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"   # 每天凌晨2点
  timeZone: "Asia/Shanghai"
  concurrencyPolicy: Forbid    # Allow | Forbid | Replace
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 200  # 错过启动时间的最大容忍秒数

  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: backup-tool:latest
            command: ["/backup.sh"]
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
```

## 7.6 Service

Service 为 Pod 提供稳定的网络端点和负载均衡。

```yaml
# ClusterIP Service（默认，集群内访问）
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
  - name: http
    port: 80        # Service 端口
    targetPort: 8080  # Pod 端口
    protocol: TCP
  sessionAffinity: None  # None | ClientIP

---
# NodePort Service（集群外通过节点端口访问）
apiVersion: v1
kind: Service
metadata:
  name: myapp-nodeport
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080   # 范围: 30000-32767，不指定则自动分配

---
# LoadBalancer Service（云环境，创建外部负载均衡器）
apiVersion: v1
kind: Service
metadata:
  name: myapp-lb
  annotations:
    # AWS 示例
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080

---
# Headless Service（直接返回 Pod IP，用于 StatefulSet）
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306

---
# ExternalName Service（映射外部服务）
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.example.com
```

## 7.7 Ingress

Ingress 提供 HTTP/HTTPS 路由，将外部流量路由到集群内 Service。

```yaml
# ingress.yaml - Nginx Ingress 示例
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    - api.example.com
    secretName: myapp-tls

  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80

  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 8080
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2-svc
            port:
              number: 8080
```

```bash
# 安装 Nginx Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml

# 查看 Ingress
kubectl get ingress
kubectl describe ingress myapp-ingress
```

## 7.8 ConfigMap 和 Secret

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  # 键值对
  DB_HOST: "mysql"
  DB_PORT: "3306"
  LOG_LEVEL: "INFO"

  # 文件内容
  application.yml: |
    server:
      port: 8080
    spring:
      datasource:
        url: jdbc:mysql://${DB_HOST}:${DB_PORT}/mydb
      redis:
        host: redis

  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://backend:8080;
      }
    }

---
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque   # Opaque | kubernetes.io/tls | kubernetes.io/dockerconfigjson
data:
  # 值必须 base64 编码
  username: YWRtaW4=   # echo -n 'admin' | base64
  password: c2VjcmV0   # echo -n 'secret' | base64
stringData:
  # stringData 不需要 base64（更方便）
  api-key: "my-secret-api-key"

---
# TLS Secret
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
```

**在 Pod 中使用 ConfigMap 和 Secret：**

```yaml
spec:
  containers:
  - name: myapp
    image: myapp:1.0

    # 方式1：单个键作为环境变量
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: myapp-config
          key: DB_HOST
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password

    # 方式2：所有键作为环境变量
    envFrom:
    - configMapRef:
        name: myapp-config
    - secretRef:
        name: db-secret
        optional: true

    volumeMounts:
    # 方式3：挂载为文件
    - name: config-vol
      mountPath: /app/config
    - name: secret-vol
      mountPath: /app/secrets
      readOnly: true

  volumes:
  - name: config-vol
    configMap:
      name: myapp-config
      items:
      - key: application.yml
        path: application.yml
  - name: secret-vol
    secret:
      secretName: db-secret
      defaultMode: 0400  # 文件权限
```

## 7.9 PersistentVolume 和 PersistentVolumeClaim

```yaml
# persistentvolume.yaml - 静态供应
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-001
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteMany    # RWO | ROX | RWX | RWOP
  persistentVolumeReclaimPolicy: Retain  # Retain | Recycle | Delete
  storageClassName: nfs-storage
  mountOptions:
  - hard
  - nfsvers=4.1
  nfs:
    path: /exports/data
    server: 192.168.1.200

---
# persistentvolumeclaim.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-pvc
spec:
  accessModes:
  - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 20Gi
  storageClassName: fast-ssd   # 留空则使用默认 StorageClass
  # selector:                   # 可以绑定到特定 PV
  #   matchLabels:
  #     type: fast

---
# StorageClass - 动态供应
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # 设为默认
provisioner: kubernetes.io/aws-ebs   # 或 rancher.io/local-path 等
parameters:
  type: gp3
  fsType: ext4
  iops: "3000"
  throughput: "125"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

---

# Part 8: Kubernetes 网络原理

## 8.1 Kubernetes 网络模型

Kubernetes 网络模型要求：
1. 每个 Pod 有唯一 IP 地址
2. 所有 Pod 可以不经 NAT 直接通信
3. 节点可以和 Pod 直接通信
4. Pod 内部通信通过 localhost

```
+----------------------------------------------------------+
|                  K8s 网络通信层级                         |
+----------------------------------------------------------+
|                                                          |
|  同 Pod 内容器通信                                        |
|  Container A <--> Container B (通过 localhost)           |
|  共享同一个 network namespace                             |
|                                                          |
|  同节点 Pod 通信                                          |
|  Pod1 <--> veth0 <--> cbr0(bridge) <--> veth1 <--> Pod2 |
|                                                          |
|  跨节点 Pod 通信（以 Flannel VXLAN 为例）                 |
|  Pod1 --> veth --> cbr0 --> flannel.1(VTEP)              |
|       --> UDP 封装 --> 物理网络                           |
|       --> flannel.1(VTEP) --> cbr0 --> veth --> Pod2     |
|                                                          |
|  Pod 访问 Service                                        |
|  Pod --> iptables/IPVS --> Pod (随机选择)                 |
|                                                          |
|  外部访问 Service                                        |
|  Internet --> LB/NodePort --> iptables/IPVS --> Pod      |
+----------------------------------------------------------+
```

## 8.2 CNI（容器网络接口）

CNI 是 Kubernetes 网络插件的标准接口。

```
+---------------------------------------------+
|            CNI 工作流程                      |
+---------------------------------------------+
|                                             |
|  kubelet 创建 Pod                           |
|       |                                     |
|       v                                     |
|  调用 CNI 插件 (ADD 命令)                   |
|       |                                     |
|       +---> 创建 veth pair                  |
|       |     一端放入容器 (eth0)              |
|       |     一端留在宿主机 (vethXXX)         |
|       |                                     |
|       +---> 分配 IP 地址                    |
|       |     (从 IPAM 插件获取)               |
|       |                                     |
|       +---> 配置路由规则                    |
|       |                                     |
|       +---> 连接到网桥/VTEP                 |
|                                             |
|  kubelet 删除 Pod                           |
|       |                                     |
|       v                                     |
|  调用 CNI 插件 (DEL 命令)                   |
|       清理网络资源                           |
+---------------------------------------------+
```

### 主流 CNI 插件对比

| 插件 | 网络模式 | 性能 | 网络策略 | 适用场景 |
|------|---------|------|---------|---------|
| Flannel | VXLAN/Host-GW | 中 | 不支持 | 简单场景，易部署 |
| Calico | BGP/IPIP | 高 | 支持 | 生产，需要网络策略 |
| Cilium | eBPF | 极高 | 支持 | 高性能，可观测性 |
| Weave | VXLAN/Mesh | 中 | 支持 | 跨多云/多集群 |
| Canal | Flannel+Calico | 中高 | 支持 | 兼顾易用性和策略 |

## 8.3 Flannel

```yaml
# Flannel 使用 VXLAN 模式安装
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Flannel ConfigMap 配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-flannel-cfg
  namespace: kube-flannel
data:
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan",
        "VNI": 1,
        "Port": 8472
      }
    }
```

**VXLAN 数据包封装：**

```
原始数据包:
[原始以太网帧: src=PodA-MAC, dst=PodB-MAC]
[IP头: src=PodA-IP, dst=PodB-IP]
[数据]

VXLAN 封装后:
[外层以太网帧: src=Node1-MAC, dst=Node2-MAC]
[外层IP头: src=Node1-IP, dst=Node2-IP]
[UDP头: dst-port=8472]
[VXLAN头: VNI=1]
[原始以太网帧]
[IP头: src=PodA-IP, dst=PodB-IP]
[数据]
```

## 8.4 Calico

```bash
# 安装 Calico
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml

# 安装 calicoctl
curl -L https://github.com/projectcalico/calico/releases/download/v3.26.1/calicoctl-linux-amd64 -o calicoctl
chmod +x calicoctl
mv calicoctl /usr/local/bin/

# 查看 Calico 节点状态
calicoctl node status

# 查看 BGP 对等体
calicoctl get bgppeers
```

**Calico BGP 模式（无封装，最高性能）：**

```
+------------------------------------+
|  BGP 路由模式                      |
+------------------------------------+
|  Node1 (10.1.0.0/24)              |
|    Pod: 10.1.0.100                 |
|         |                          |
|    BGP 广播: 10.1.0.0/24 via Node1|
+------------------------------------+
             |
     物理路由器（学习路由）
             |
+------------------------------------+
|  Node2 (10.1.1.0/24)              |
|    Pod: 10.1.1.100                 |
|    收到路由: 10.1.0.0/24 via Node1 |
+------------------------------------+
```

## 8.5 Cilium 与 eBPF

Cilium 使用 Linux eBPF 技术，在内核层面实现网络功能，性能极高。

```
+--------------------------------------------------+
|          eBPF vs iptables 性能对比                |
+--------------------------------------------------+
|                                                  |
|  iptables (O(n) 规则匹配):                       |
|  数据包 -> 遍历所有链 -> 匹配规则 -> 转发         |
|  规则数越多，延迟越高                              |
|                                                  |
|  eBPF (O(1) 哈希查找):                           |
|  数据包 -> eBPF map 查找 -> 直接转发              |
|  规则数量对性能影响极小                            |
+--------------------------------------------------+
```

```bash
# 安装 Cilium CLI
curl -L --remote-name-all \
    https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz
tar xzvf cilium-linux-amd64.tar.gz
mv cilium /usr/local/bin/

# 安装 Cilium
cilium install --version 1.14.2

# 检查 Cilium 状态
cilium status
cilium connectivity test
```

## 8.6 CoreDNS

CoreDNS 是 Kubernetes 集群内的 DNS 服务器，为服务发现提供支持。

```
+--------------------------------------------------+
|           Kubernetes DNS 解析规则                 |
+--------------------------------------------------+
|                                                  |
|  Service DNS 格式:                               |
|  <service>.<namespace>.svc.cluster.local         |
|                                                  |
|  Pod DNS 格式:                                   |
|  <pod-ip-with-dashes>.<namespace>.pod.cluster.local|
|  例: 10-244-0-100.default.pod.cluster.local      |
|                                                  |
|  短名解析（同命名空间）:                           |
|  mysql --> mysql.default.svc.cluster.local       |
|                                                  |
|  跨命名空间:                                      |
|  mysql.production --> mysql.production.svc.cluster.local|
+--------------------------------------------------+
```

```yaml
# CoreDNS ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
            max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

## 8.7 NetworkPolicy（网络策略）

NetworkPolicy 控制 Pod 间的网络访问（需要 CNI 支持，如 Calico、Cilium）。

```yaml
# networkpolicy.yaml - 限制 myapp 只能被 frontend 访问
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: myapp-netpol
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: myapp

  policyTypes:
  - Ingress
  - Egress

  ingress:
  # 只允许来自 frontend 命名空间中带有 role=frontend 标签的 Pod
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
      podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 8080

  egress:
  # 允许访问 mysql
  - to:
    - podSelector:
        matchLabels:
          app: mysql
    ports:
    - protocol: TCP
      port: 3306
  # 允许 DNS 查询
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53

---
# 默认拒绝所有入站流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}   # 选择所有 Pod
  policyTypes:
  - Ingress

---
# 允许所有入站（显式）
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
spec:
  podSelector: {}
  ingress:
  - {}   # 空规则 = 允许所有
  policyTypes:
  - Ingress
```

---

# Part 9: Kubernetes 高可用与调度

## 9.1 HPA（水平 Pod 自动扩缩容）

```yaml
# hpa.yaml - 基于 CPU 和自定义指标
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp

  minReplicas: 2
  maxReplicas: 20

  metrics:
  # CPU 利用率
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

  # 内存利用率
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 400Mi

  # 自定义指标（需要 Prometheus Adapter）
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"

  # 外部指标（如消息队列深度）
  - type: External
    external:
      metric:
        name: sqs_queue_depth
        selector:
          matchLabels:
            queue: myapp-queue
      target:
        type: AverageValue
        averageValue: "50"

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100     # 每次最多翻倍
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # 缩容前等待5分钟
      policies:
      - type: Pods
        value: 2       # 每次最多缩容2个
        periodSeconds: 60
```

```bash
# 查看 HPA 状态
kubectl get hpa
kubectl describe hpa myapp-hpa

# 手动触发测试（模拟负载）
kubectl run -it --rm load-generator --image=busybox \
    --restart=Never -- /bin/sh -c \
    "while true; do wget -q -O- http://myapp-svc; done"
```

## 9.2 VPA（垂直 Pod 自动扩缩容）

```yaml
# 安装 VPA
kubectl apply -f https://github.com/kubernetes/autoscaler/raw/master/vertical-pod-autoscaler/deploy/vpa-v1-crd-gen.yaml
kubectl apply -f https://github.com/kubernetes/autoscaler/raw/master/vertical-pod-autoscaler/deploy/vpa-rbac.yaml

# vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp

  updatePolicy:
    updateMode: "Auto"   # Off | Initial | Recreate | Auto

  resourcePolicy:
    containerPolicies:
    - containerName: myapp
      minAllowed:
        cpu: 100m
        memory: 50Mi
      maxAllowed:
        cpu: 4
        memory: 4Gi
      controlledResources: ["cpu", "memory"]
```

## 9.3 Cluster Autoscaler

Cluster Autoscaler 自动增减集群节点数量。

```yaml
# cluster-autoscaler.yaml (AWS EKS 示例)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.28.0
        name: cluster-autoscaler
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
        - --balance-similar-node-groups
        - --scale-down-delay-after-add=10m
        - --scale-down-unneeded-time=10m
```

## 9.4 Pod 调度进阶

### 节点亲和性（Node Affinity）

```yaml
spec:
  affinity:
    nodeAffinity:
      # 硬性要求（必须满足）
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/arch
            operator: In
            values: [amd64]
          - key: node-type
            operator: In
            values: [high-mem, gpu]

      # 软性偏好（尽量满足）
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values: [cn-east-1a]
      - weight: 20
        preference:
          matchExpressions:
          - key: disk-type
            operator: In
            values: [ssd]
```

### Pod 亲和性和反亲和性

```yaml
spec:
  affinity:
    # Pod 亲和性：与带有 app=cache 标签的 Pod 放在同一节点
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: cache
        topologyKey: kubernetes.io/hostname

    # Pod 反亲和性：不与同一应用的 Pod 放在同一节点（高可用）
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: myapp
          topologyKey: kubernetes.io/hostname
```

### Taints 和 Tolerations

```bash
# 给节点添加污点
kubectl taint nodes node1 dedicated=gpu:NoSchedule
kubectl taint nodes node1 team=infra:PreferNoSchedule
kubectl taint nodes node1 maintenance=true:NoExecute

# 查看节点污点
kubectl describe node node1 | grep Taint

# 删除污点
kubectl taint nodes node1 dedicated=gpu:NoSchedule-
```

```yaml
# 容忍污点
spec:
  tolerations:
  # 精确匹配
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"

  # 只匹配 key
  - key: "team"
    operator: "Exists"
    effect: "PreferNoSchedule"

  # 容忍 NoExecute（带宽限驱逐）
  - key: "maintenance"
    operator: "Equal"
    value: "true"
    effect: "NoExecute"
    tolerationSeconds: 3600   # 3600秒后驱逐
```

## 9.5 PodDisruptionBudget（PDB）

PDB 保证在维护操作期间应用的可用性。

```yaml
# pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2     # 最少保持2个可用（或用百分比: "80%"）
  # maxUnavailable: 1  # 最多允许1个不可用（二选一）
  selector:
    matchLabels:
      app: myapp
```

```bash
# 查看 PDB
kubectl get pdb
kubectl describe pdb myapp-pdb

# 模拟节点维护（drain会遵守PDB）
kubectl drain node1 --ignore-daemonsets --delete-emptydir-data
kubectl uncordon node1   # 恢复调度
```

## 9.6 QoS（服务质量等级）

Kubernetes 根据资源请求和限制自动为 Pod 分配 QoS 等级，影响 OOM Kill 优先级。

```
+---------------------------------------------------+
|              QoS 等级                              |
+---------------------------------------------------+
|                                                   |
|  Guaranteed（最高优先级，最后被 OOM Kill）          |
|  条件: requests == limits (CPU 和 内存都设置)       |
|  resources:                                       |
|    requests: { cpu: 500m, memory: 512Mi }         |
|    limits:   { cpu: 500m, memory: 512Mi }         |
|                                                   |
|  Burstable（中等优先级）                           |
|  条件: 至少设置一个资源的 requests 或 limits       |
|  resources:                                       |
|    requests: { cpu: 100m, memory: 128Mi }         |
|    limits:   { cpu: 500m, memory: 512Mi }         |
|                                                   |
|  BestEffort（最低优先级，最先被 OOM Kill）          |
|  条件: 不设置任何 resources                        |
+---------------------------------------------------+
```

```bash
# 查看 Pod QoS 等级
kubectl get pod myapp -o jsonpath='{.status.qosClass}'
```

---

# Part 10: Kubernetes 存储深度解析

## 10.1 存储架构概览

```
+--------------------------------------------------------------+
|                   K8s 存储架构                                |
+--------------------------------------------------------------+
|                                                              |
|  应用层:   Pod -> PVC -> PV -> StorageClass                  |
|                                                              |
|  CSI 层:   PV Controller -> CSI Controller (external)       |
|            kubelet -> CSI Node Plugin                        |
|                                                              |
|  存储后端: 本地存储 | NFS | Ceph | AWS EBS | GCP PD | ...    |
+--------------------------------------------------------------+
```

## 10.2 CSI（容器存储接口）

CSI 是 Kubernetes 存储插件的标准接口。

```
+----------------------------------------------+
|             CSI 组件架构                      |
+----------------------------------------------+
|                                              |
|  Kubernetes                                  |
|  +-------------------+                       |
|  | PV Controller     |--gRPC--> CSI Controller|
|  | (external)        |                       |
|  +-------------------+    +--external-provisi-+
|                           |oner/attacher      |
|  +-------------------+    +--CSI Driver-------+
|  | kubelet           |--gRPC--> CSI Node Plugin|
|  +-------------------+    +-------------------+
|                                              |
|  CSI Driver 组件:                             |
|  - Controller Plugin (Deployment)            |
|    * CreateVolume / DeleteVolume             |
|    * ControllerPublishVolume (attach)        |
|  - Node Plugin (DaemonSet)                  |
|    * NodeStageVolume (mount to node)         |
|    * NodePublishVolume (mount to pod)        |
+----------------------------------------------+
```

## 10.3 动态存储供应

```yaml
# StorageClass - Ceph RBD 示例
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd
provisioner: rbd.csi.ceph.com
parameters:
  clusterID: <ceph-cluster-id>
  pool: kubernetes
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
  csi.storage.k8s.io/provisioner-secret-namespace: ceph-csi
  csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
  csi.storage.k8s.io/controller-expand-secret-namespace: ceph-csi
  csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
  csi.storage.k8s.io/node-stage-secret-namespace: ceph-csi
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate

---
# StorageClass - Local Path Provisioner（测试/开发用）
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
```

## 10.4 卷扩容

```bash
# 前提：StorageClass 设置了 allowVolumeExpansion: true

# 编辑 PVC 增加容量
kubectl edit pvc myapp-pvc
# 修改 spec.resources.requests.storage 的值

# 查看扩容状态
kubectl describe pvc myapp-pvc
# 等待 status.capacity.storage 更新
```

## 10.5 本地持久卷

```yaml
# 适合数据库等需要高I/O的场景
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-node1
spec:
  capacity:
    storage: 500Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/data
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node1

---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner   # 不自动供应
volumeBindingMode: WaitForFirstConsumer     # 等到 Pod 调度后再绑定
```

---

# Part 11: Kubernetes 安全

## 11.1 RBAC（基于角色的访问控制）

```
+----------------------------------------------+
|              RBAC 概念关系图                  |
+----------------------------------------------+
|                                              |
|  Subject (主体)     Role/ClusterRole         |
|  +-----------+      +----------------+       |
|  | User      |      | 定义权限规则    |       |
|  | Group     |<--+  | - apiGroups    |       |
|  | ServiceAcc|   |  | - resources    |       |
|  +-----------+   |  | - verbs        |       |
|                  |  +----------------+       |
|                  |         |                 |
|        RoleBinding/ClusterRoleBinding        |
|        将主体绑定到角色                       |
+----------------------------------------------+
```

```yaml
# role.yaml - 命名空间级别权限
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]    # "" 表示核心 API 组
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update", "patch"]

---
# rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane    # 用户名
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: devteam
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: myapp-sa
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

---
# clusterrole.yaml - 集群级别权限
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "list"]
- nonResourceURLs: ["/healthz", "/metrics"]
  verbs: ["get"]

---
# clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-reader-binding
subjects:
- kind: ServiceAccount
  name: monitoring-sa
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

**常用 RBAC verbs：**

| Verb | HTTP 方法 | 说明 |
|------|-----------|------|
| get | GET | 获取单个资源 |
| list | GET | 列举资源 |
| watch | GET+watch | 监听资源变化 |
| create | POST | 创建资源 |
| update | PUT | 完整更新 |
| patch | PATCH | 部分更新 |
| delete | DELETE | 删除资源 |
| deletecollection | DELETE | 批量删除 |

## 11.2 ServiceAccount

```yaml
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/my-role   # IRSA（AWS）

---
# 在 Pod 中使用 ServiceAccount
spec:
  serviceAccountName: myapp-sa
  automountServiceAccountToken: false   # 禁止自动挂载 token（安全最佳实践）
```

```bash
# 查看 ServiceAccount 的 token
kubectl create token myapp-sa --duration=1h

# 测试权限
kubectl auth can-i get pods --as=system:serviceaccount:default:myapp-sa
kubectl auth can-i list deployments --as=jane
```

## 11.3 Pod Security Standards（Pod 安全标准）

```yaml
# 在命名空间级别启用 Pod 安全标准
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # 三种策略级别: privileged | baseline | restricted
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.28
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**三种安全级别：**

| 级别 | 说明 | 限制 |
|------|------|------|
| Privileged | 无限制 | 无 |
| Baseline | 防止已知提权 | 禁止特权容器、hostPath等 |
| Restricted | 最严格 | 必须以非root运行，只读根文件系统等 |

```yaml
# restricted 级别的 Pod 示例
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 65534
    fsGroup: 65534
    seccompProfile:
      type: RuntimeDefault

  containers:
  - name: myapp
    image: myapp:1.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

## 11.4 镜像安全最佳实践

```dockerfile
# 安全 Dockerfile 示例
FROM eclipse-temurin:17-jre-alpine

# 不以 root 运行
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# 扫描基础镜像漏洞（使用 trivy）
# trivy image eclipse-temurin:17-jre-alpine

WORKDIR /app

# 只复制需要的文件
COPY --chown=appuser:appgroup target/app.jar .

USER appuser

# 只读根文件系统（配合 tmpfs 挂载 /tmp）
VOLUME ["/tmp"]

HEALTHCHECK --interval=30s --timeout=3s \
    CMD wget -q --spider http://localhost:8080/health || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
# 使用 trivy 扫描镜像
trivy image myapp:1.0
trivy image --severity CRITICAL,HIGH myapp:1.0

# 使用 trivy 扫描 K8s 集群
trivy k8s --report summary cluster
```

---

# Part 12: Helm 包管理

## 12.1 Helm 简介

Helm 是 Kubernetes 的包管理器，类似于 Linux 的 apt/yum。

```
+------------------------------------------+
|             Helm 核心概念                 |
+------------------------------------------+
|                                          |
|  Chart     = 应用的打包格式（类似 rpm）   |
|  Release   = Chart 的一次部署实例         |
|  Repository= Chart 的存储仓库             |
|  Values    = Chart 的配置参数             |
|                                          |
|  helm install myrelease ./mychart        |
|       |                                  |
|       v                                  |
|  Template 渲染（values 注入）             |
|       |                                  |
|       v                                  |
|  kubectl apply 到 Kubernetes             |
+------------------------------------------+
```

## 12.2 Helm 安装与常用命令

```bash
# 安装 Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 添加 Helm 仓库
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 搜索 Chart
helm search repo mysql
helm search hub wordpress

# 查看 Chart 信息
helm show chart bitnami/mysql
helm show values bitnami/mysql

# 安装 Chart
helm install my-mysql bitnami/mysql
helm install my-mysql bitnami/mysql -f values.yaml
helm install my-mysql bitnami/mysql \
    --set auth.rootPassword=secret \
    --set primary.persistence.size=20Gi \
    --namespace database \
    --create-namespace

# 查看 Release
helm list
helm list -A    # 所有命名空间
helm status my-mysql

# 升级 Release
helm upgrade my-mysql bitnami/mysql -f values.yaml
helm upgrade --install my-mysql bitnami/mysql   # 不存在则安装

# 回滚
helm history my-mysql
helm rollback my-mysql 1

# 卸载
helm uninstall my-mysql
helm uninstall my-mysql --keep-history   # 保留历史记录

# 导出渲染后的 YAML（不实际安装）
helm template my-mysql bitnami/mysql -f values.yaml
helm install my-mysql bitnami/mysql --dry-run
```

## 12.3 Chart 目录结构

```
mychart/
├── Chart.yaml          # Chart 元数据
├── values.yaml         # 默认配置值
├── values.schema.json  # values 的 JSON Schema 验证（可选）
├── charts/             # 依赖 Chart
├── crds/               # CRD 定义（在其他资源前安装）
├── templates/          # Kubernetes 资源模板
│   ├── _helpers.tpl    # 模板辅助函数
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── serviceaccount.yaml
│   ├── hpa.yaml
│   └── NOTES.txt       # 安装后显示的提示信息
└── .helmignore         # 类似 .gitignore
```

## 12.4 自定义 Chart 开发

### Chart.yaml

```yaml
apiVersion: v2
name: myapp
description: My Spring Boot Application Helm Chart
type: application    # application | library
version: 0.1.0       # Chart 版本
appVersion: "1.0.0"  # 应用版本

maintainers:
- name: DevTeam
  email: dev@example.com

dependencies:
- name: mysql
  version: "9.14.x"
  repository: "https://charts.bitnami.com/bitnami"
  condition: mysql.enabled
- name: redis
  version: "18.x.x"
  repository: "https://charts.bitnami.com/bitnami"
  condition: redis.enabled
```

### values.yaml

```yaml
replicaCount: 2

image:
  repository: myapp
  pullPolicy: IfNotPresent
  tag: ""   # 默认使用 Chart.appVersion

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations: {}
podSecurityContext:
  fsGroup: 2000

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000
  capabilities:
    drop:
    - ALL

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: false
  className: "nginx"
  annotations: {}
  hosts:
  - host: myapp.example.com
    paths:
    - path: /
      pathType: Prefix
  tls: []

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

nodeSelector: {}
tolerations: []
affinity: {}

config:
  springProfile: production
  logLevel: INFO

mysql:
  enabled: true
  auth:
    rootPassword: ""
    database: myappdb
    username: myapp
    password: ""
  primary:
    persistence:
      size: 20Gi

redis:
  enabled: true
  auth:
    password: ""
```

### templates/_helpers.tpl

```
{{/*
Expand the name of the chart.
*/}}
{{- define "myapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "myapp.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
      - name: {{ .Chart.Name }}
        securityContext:
          {{- toYaml .Values.securityContext | nindent 12 }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: {{ .Values.config.springProfile }}
        {{- if .Values.mysql.enabled }}
        - name: SPRING_DATASOURCE_URL
          value: jdbc:mysql://{{ include "myapp.fullname" . }}-mysql:3306/{{ .Values.mysql.auth.database }}
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ include "myapp.fullname" . }}-mysql
              key: mysql-password
        {{- end }}
        ports:
        - name: http
          containerPort: {{ .Values.service.targetPort }}
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: http
          initialDelaySeconds: 10
          periodSeconds: 5
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

```bash
# 开发 Chart 工作流
helm create myapp           # 脚手架创建
helm lint myapp             # 语法检查
helm template myapp ./myapp # 预览渲染结果
helm install myapp ./myapp --dry-run  # 模拟安装
helm install myapp ./myapp  # 正式安装

# 打包和发布
helm package myapp
helm repo index . --url https://mycharts.example.com
```

---

# Part 13: 完整实战案例

## 案例一：Spring Boot 微服务部署到 Kubernetes

### 应用架构

```
+----------------------------------------------------------+
|                  微服务架构                               |
+----------------------------------------------------------+
|                                                          |
|  Internet                                                |
|      |                                                   |
|  Ingress (nginx)                                         |
|      |                                                   |
|  +---+--------+  +------------+  +------------------+   |
|  | user-svc   |  | order-svc  |  | product-svc      |   |
|  | (3 replicas)|  | (3 replicas)|  | (3 replicas)     |   |
|  +---+--------+  +-----+------+  +--------+---------+   |
|      |                 |                  |              |
|  +---+-----------------+------------------+----------+  |
|  |              MySQL            Redis               |  |
|  +---------------------------------------------------+  |
+----------------------------------------------------------+
```

### 完整部署清单

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: microservices
  labels:
    pod-security.kubernetes.io/enforce: baseline

---
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: microservices
data:
  SPRING_PROFILES_ACTIVE: "kubernetes"
  LOG_LEVEL: "INFO"
  REDIS_HOST: "redis-svc"
  REDIS_PORT: "6379"

---
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: microservices
type: Opaque
stringData:
  DB_PASSWORD: "your-db-password"
  REDIS_PASSWORD: "your-redis-password"
  JWT_SECRET: "your-jwt-secret-key-here"

---
# user-service deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: microservices
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: user-service
        version: v1
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: user-service
              topologyKey: kubernetes.io/hostname
      containers:
      - name: user-service
        image: registry.example.com/user-service:1.0.0
        ports:
        - containerPort: 8080
        envFrom:
        - configMapRef:
            name: app-config
        env:
        - name: DB_URL
          value: jdbc:mysql://mysql-svc:3306/userdb
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: DB_PASSWORD
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]
      terminationGracePeriodSeconds: 60

---
# user-service service
apiVersion: v1
kind: Service
metadata:
  name: user-svc
  namespace: microservices
spec:
  selector:
    app: user-service
  ports:
  - port: 80
    targetPort: 8080

---
# HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service-hpa
  namespace: microservices
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

---
# PDB
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: user-service-pdb
  namespace: microservices
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: user-service

---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
  namespace: microservices
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api/users
        pathType: Prefix
        backend:
          service:
            name: user-svc
            port:
              number: 80
      - path: /api/orders
        pathType: Prefix
        backend:
          service:
            name: order-svc
            port:
              number: 80
      - path: /api/products
        pathType: Prefix
        backend:
          service:
            name: product-svc
            port:
              number: 80
```

## 案例二：有状态应用 - Redis Cluster 部署

```yaml
# redis-cluster StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
spec:
  selector:
    matchLabels:
      app: redis-cluster
  serviceName: redis-cluster
  replicas: 6   # 3 master + 3 slave
  template:
    metadata:
      labels:
        app: redis-cluster
    spec:
      containers:
      - name: redis
        image: redis:7.2-alpine
        ports:
        - containerPort: 6379
          name: client
        - containerPort: 16379
          name: gossip
        command: ["/conf/update-node.sh", "redis-server", "/conf/redis.conf"]
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        volumeMounts:
        - name: conf
          mountPath: /conf
          readOnly: false
        - name: data
          mountPath: /data
          readOnly: false
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
      volumes:
      - name: conf
        configMap:
          name: redis-cluster-config
          defaultMode: 0755
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 5Gi

---
# Headless Service
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster
spec:
  type: ClusterIP
  clusterIP: None
  selector:
    app: redis-cluster
  ports:
  - port: 6379
    targetPort: 6379
    name: client
  - port: 16379
    targetPort: 16379
    name: gossip
```

```bash
# 初始化 Redis Cluster
kubectl exec -it redis-cluster-0 -- redis-cli --cluster create \
    $(kubectl get pods -l app=redis-cluster -o jsonpath='{range .items[*]}{.status.podIP}:6379 {end}') \
    --cluster-replicas 1 \
    --cluster-yes
```

## 案例三：CI/CD Pipeline（GitLab CI + Kubernetes）

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - scan
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  IMAGE_NAME: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  LATEST_IMAGE: $CI_REGISTRY_IMAGE:latest

# 构建 Docker 镜像
build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build
        --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')
        --build-arg VCS_REF=$CI_COMMIT_SHA
        -t $IMAGE_NAME
        -t $LATEST_IMAGE .
    - docker push $IMAGE_NAME
    - docker push $LATEST_IMAGE
  only:
    - main
    - /^release\/.*$/

# 单元测试
test:
  stage: test
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn test -B
  artifacts:
    reports:
      junit:
        - target/surefire-reports/TEST-*.xml

# 漏洞扫描
scan:
  stage: scan
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image
        --exit-code 1
        --severity CRITICAL
        --no-progress
        $IMAGE_NAME
  allow_failure: true   # 发现漏洞不阻塞部署（可按需改为 false）

# 部署到开发环境
deploy-dev:
  stage: deploy
  image: bitnami/kubectl:latest
  environment:
    name: development
    url: https://dev.example.com
  script:
    - kubectl config use-context dev-cluster
    - kubectl set image deployment/myapp
        myapp=$IMAGE_NAME
        -n development
    - kubectl rollout status deployment/myapp -n development
  only:
    - main

# 部署到生产环境（需要手动审批）
deploy-prod:
  stage: deploy
  image: bitnami/kubectl:latest
  environment:
    name: production
    url: https://prod.example.com
  script:
    - kubectl config use-context prod-cluster
    - helm upgrade --install myapp ./helm/myapp
        --namespace production
        --set image.tag=$CI_COMMIT_SHA
        --wait
        --timeout 10m
  only:
    - /^release\/.*$/
  when: manual
```

## 案例四：监控栈部署（Prometheus + Grafana）

```bash
# 使用 kube-prometheus-stack Helm Chart
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# 创建自定义 values 文件
cat > monitoring-values.yaml << 'EOF'
grafana:
  enabled: true
  adminPassword: "admin123"
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
    - grafana.example.com

prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: fast-ssd
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: fast-ssd
          resources:
            requests:
              storage: 10Gi

nodeExporter:
  enabled: true

kubeStateMetrics:
  enabled: true
EOF

helm install monitoring prometheus-community/kube-prometheus-stack \
    --namespace monitoring \
    --create-namespace \
    -f monitoring-values.yaml
```

```yaml
# 应用自定义 ServiceMonitor（让 Prometheus 抓取应用指标）
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  namespace: monitoring
  labels:
    release: monitoring   # 必须匹配 Prometheus 的 serviceMonitorSelector
spec:
  selector:
    matchLabels:
      app: myapp
  namespaceSelector:
    matchNames:
    - default
  endpoints:
  - port: http
    path: /actuator/prometheus
    interval: 30s
    scrapeTimeout: 10s
```

## 案例五：多环境配置管理（Kustomize）

```
overlays/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
├── dev/
│   ├── kustomization.yaml
│   └── patch-replicas.yaml
├── staging/
│   ├── kustomization.yaml
│   └── patch-resources.yaml
└── prod/
    ├── kustomization.yaml
    ├── patch-replicas.yaml
    └── patch-resources.yaml
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- configmap.yaml

commonLabels:
  app.kubernetes.io/managed-by: kustomize

---
# dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: dev

resources:
- ../base

namePrefix: dev-

patches:
- path: patch-replicas.yaml

images:
- name: myapp
  newTag: dev-latest

configMapGenerator:
- name: app-config
  behavior: merge
  literals:
  - LOG_LEVEL=DEBUG
  - SPRING_PROFILES_ACTIVE=dev

---
# dev/patch-replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1

---
# prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
- ../base

namePrefix: prod-

replicas:
- name: myapp
  count: 5

images:
- name: myapp
  newTag: "1.2.3"

patches:
- path: patch-resources.yaml

---
# prod/patch-resources.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 2000m
            memory: 2Gi
```

```bash
# 使用 Kustomize
kubectl apply -k overlays/dev/
kubectl apply -k overlays/prod/
kubectl diff -k overlays/prod/   # 预览变更
kustomize build overlays/prod/ | kubectl apply -f -
```

---

# Part 14: kubectl 命令大全

## 14.1 基础命令

```bash
# ===== 查看资源 =====
kubectl get pods                         # 当前命名空间
kubectl get pods -n kube-system          # 指定命名空间
kubectl get pods -A                      # 所有命名空间
kubectl get pods -o wide                 # 显示更多信息（IP、节点）
kubectl get pods -o yaml                 # YAML 格式
kubectl get pods -o json                 # JSON 格式
kubectl get pods --show-labels           # 显示标签
kubectl get pods -l app=myapp            # 标签过滤
kubectl get pods --field-selector=status.phase=Running  # 字段过滤
kubectl get pods --watch                 # 实时监听变化
kubectl get pods --sort-by='.metadata.creationTimestamp'  # 排序

# 自定义输出格式
kubectl get pods -o custom-columns=\
  "NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName"

# JSONPath 输出
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pod mypod -o jsonpath='{.spec.containers[0].image}'

# ===== 查看详情 =====
kubectl describe pod mypod
kubectl describe node node1
kubectl describe service myapp-svc

# ===== 创建/更新资源 =====
kubectl apply -f manifest.yaml
kubectl apply -f ./manifests/           # 应用目录下所有文件
kubectl apply -f https://example.com/manifest.yaml
kubectl create -f manifest.yaml         # 只创建，已存在则报错

# 命令式创建
kubectl create deployment nginx --image=nginx --replicas=3
kubectl create service clusterip nginx --tcp=80:80
kubectl create configmap myconfig --from-file=config.yaml
kubectl create secret generic mysecret --from-literal=password=123456
kubectl create namespace myns

# ===== 删除资源 =====
kubectl delete pod mypod
kubectl delete -f manifest.yaml
kubectl delete pods,services -l app=myapp
kubectl delete pods --all -n myns
kubectl delete namespace myns   # 删除命名空间及其所有资源

# 强制立即删除（谨慎使用）
kubectl delete pod mypod --grace-period=0 --force

# ===== 编辑资源 =====
kubectl edit deployment myapp
kubectl edit cm myconfig
```

## 14.2 调试命令

```bash
# ===== 日志 =====
kubectl logs mypod
kubectl logs mypod -c mycontainer      # 多容器 Pod 指定容器
kubectl logs mypod --previous          # 上一个崩溃的容器日志
kubectl logs -f mypod                  # 实时跟踪
kubectl logs mypod --tail=100          # 最后100行
kubectl logs mypod --since=1h          # 最近1小时
kubectl logs -l app=myapp --all-pods   # 标签选择器

# ===== 进入容器 =====
kubectl exec -it mypod -- bash
kubectl exec -it mypod -c mycontainer -- sh
kubectl exec mypod -- ls /app

# ===== 端口转发（本地调试神器） =====
kubectl port-forward pod/mypod 8080:8080
kubectl port-forward svc/myapp-svc 8080:80
kubectl port-forward deployment/myapp 8080:8080

# ===== 复制文件 =====
kubectl cp mypod:/app/log.txt ./log.txt
kubectl cp ./config.yml mypod:/app/config.yml

# ===== 调试网络 =====
# 临时运行调试容器
kubectl run tmp-shell --rm -it --image=nicolaka/netshoot -- bash
kubectl run tmp-curl --rm -it --image=curlimages/curl -- sh

# 调试现有 Pod（K8s 1.23+）
kubectl debug mypod -it --image=busybox --target=mycontainer
kubectl debug node/node1 -it --image=ubuntu

# ===== 事件 =====
kubectl get events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events -n myns --field-selector type=Warning
```

## 14.3 节点管理

```bash
# ===== 节点操作 =====
kubectl get nodes
kubectl get nodes -o wide
kubectl describe node node1

# 标记节点
kubectl label node node1 disk=ssd
kubectl label node node1 disk-                   # 删除标签
kubectl taint node node1 key=value:NoSchedule
kubectl taint node node1 key=value:NoSchedule-   # 删除污点

# 节点维护
kubectl cordon node1        # 禁止调度新 Pod 到 node1
kubectl uncordon node1      # 恢复调度
kubectl drain node1 \       # 驱逐所有 Pod（用于维护）
    --ignore-daemonsets \
    --delete-emptydir-data \
    --grace-period=120

# 查看节点资源使用
kubectl top nodes
kubectl top pods
kubectl top pods --containers  # 显示容器级别
```

## 14.4 集群管理

```bash
# ===== 命名空间 =====
kubectl get namespaces
kubectl create namespace myns
kubectl delete namespace myns

# ===== 上下文切换 =====
kubectl config get-contexts
kubectl config current-context
kubectl config use-context prod-cluster
kubectl config set-context --current --namespace=myns

# ===== 资源配额 =====
kubectl get resourcequota -A
kubectl describe resourcequota -n myns

# ===== 角色和权限 =====
kubectl get roles -A
kubectl get clusterroles
kubectl get rolebindings -A
kubectl auth can-i create pods
kubectl auth can-i create pods --as=jane
kubectl auth can-i '*' '*'   # 是否是集群管理员

# ===== 证书 =====
kubectl get csr   # Certificate Signing Requests
kubectl certificate approve mycsr
kubectl certificate deny mycsr

# ===== 集群信息 =====
kubectl cluster-info
kubectl version
kubectl api-resources    # 列出所有资源类型
kubectl api-versions     # 列出所有 API 版本
kubectl explain pod.spec.containers   # 查看字段文档
```

## 14.5 高效技巧

```bash
# ===== 别名设置（加入 ~/.bashrc 或 ~/.zshrc）=====
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgn='kubectl get nodes'
alias kdp='kubectl describe pod'
alias kaf='kubectl apply -f'
alias kdf='kubectl delete -f'

# ===== 自动补全 =====
source <(kubectl completion bash)   # bash
source <(kubectl completion zsh)    # zsh

# ===== 使用 kubectx/kubens 快速切换集群/命名空间 =====
kubectx                  # 列出所有 context
kubectx prod-cluster     # 切换到 prod-cluster
kubens                   # 列出所有命名空间
kubens myapp             # 切换默认命名空间

# ===== stern：多 Pod 日志聚合 =====
stern myapp              # 所有匹配 "myapp" 的 Pod 日志
stern -l app=myapp       # 标签选择
stern myapp --since 10m  # 最近10分钟

# ===== kubectl neat：简化 YAML 输出 =====
kubectl get pod mypod -o yaml | kubectl neat
```

---

# Part 15: 常见面试题 FAQ

## Docker 相关

**Q1: Docker 容器与虚拟机的核心区别是什么？**

A: 核心区别在于隔离层级和资源使用：
- 虚拟机在 Hypervisor 上运行完整的 Guest OS，每个 VM 占用 GB 级内存和存储
- Docker 容器共享宿主机 OS 内核，通过 Linux Namespace（隔离）和 Cgroups（限制资源）实现隔离，仅需 MB 级资源
- 容器启动时间秒级，VM 需要分钟级
- 隔离性：VM 是硬件级隔离（更安全），容器是 OS 级隔离（可能存在内核漏洞逃逸风险）

---

**Q2: Docker 镜像是如何分层存储的？写时复制（CoW）是如何工作的？**

A: Docker 使用 OverlayFS 实现分层：
- 镜像由多个只读层堆叠，每条 Dockerfile 指令通常创建一层
- 容器运行时在最上层添加一个可写层（Container Layer）
- 读操作：从最上层向下搜索，找到文件即返回
- 写操作（CoW）：
  1. 检查可写层是否已有该文件
  2. 若没有，从只读层 "copy-up" 一份到可写层
  3. 修改可写层中的副本
- 优点：多个容器共享相同镜像层，节省磁盘空间

---

**Q3: Dockerfile 中 CMD 和 ENTRYPOINT 的区别？**

A:
- `ENTRYPOINT`：定义容器的主执行程序，不可被 `docker run` 末尾的参数替换（除非使用 `--entrypoint`）
- `CMD`：提供默认参数，会被 `docker run` 末尾的参数替换
- 最佳实践：ENTRYPOINT 指定固定的执行程序，CMD 提供默认参数，两者结合使用
- 例：`ENTRYPOINT ["java"] CMD ["-jar", "app.jar"]`，可以通过 `docker run myapp -Xmx1g -jar app.jar` 覆盖 CMD 但保留 ENTRYPOINT

---

**Q4: 如何优化 Docker 镜像大小？**

A: 多种策略：
1. **多阶段构建**：构建阶段使用完整 SDK，运行阶段只用 JRE/Alpine
2. **选择精简基础镜像**：alpine（7MB）vs ubuntu（80MB）
3. **合并 RUN 指令**：减少层数，且在同一层清理缓存（`rm -rf /var/lib/apt/lists/*`）
4. **.dockerignore**：排除不必要文件（node_modules, .git 等）
5. **不安装不必要的工具**：`apt-get install --no-install-recommends`
6. **使用 distroless 镜像**：无 shell、包管理器，更安全更小

---

**Q5: Docker 网络 bridge 模式的工作原理？**

A:
- Docker 创建 `docker0` 虚拟网桥（默认 172.17.0.1/16）
- 每个容器启动时，创建一对 veth pair：一端放入容器（命名为 eth0），一端连接到 docker0
- 容器间通信通过 docker0 网桥转发
- 容器访问外网：通过 iptables MASQUERADE（SNAT）规则，将容器 IP 转换为宿主机 IP
- 自定义 bridge 网络比默认 docker0 多一个功能：**容器名 DNS 解析**（通过 Docker 内置 DNS 服务器）

---

## Kubernetes 相关

**Q6: Kubernetes 中 Pod 的状态有哪些？各自代表什么？**

A:
- `Pending`：Pod 已创建，但容器尚未启动（可能在调度中、拉取镜像或等待资源）
- `Running`：至少一个容器正在运行
- `Succeeded`：所有容器成功退出（适用于 Job）
- `Failed`：至少一个容器以失败状态退出
- `Unknown`：无法获取 Pod 状态（通常是节点通信问题）
- `CrashLoopBackOff`（非官方，常见 Condition）：容器反复崩溃重启，退避时间递增

---

**Q7: Kubernetes 中 Service 有哪几种类型？各自使用场景？**

A:
- **ClusterIP**（默认）：集群内部虚拟 IP，只能集群内访问。适合服务间内部通信
- **NodePort**：在每个节点上开放固定端口（30000-32767），外部通过 `节点IP:NodePort` 访问。适合开发测试
- **LoadBalancer**：云环境中创建外部负载均衡器，获得公网 IP。适合生产环境对外暴露
- **ExternalName**：将 Service 映射为外部 DNS 名，返回 CNAME。适合访问集群外部服务
- **Headless**（clusterIP: None）：不分配 VIP，DNS 直接返回 Pod IP 列表。适合 StatefulSet 和自定义服务发现

---

**Q8: Deployment、StatefulSet、DaemonSet 的适用场景？**

A:
- **Deployment**：管理无状态应用，支持滚动更新、回滚。适合 Web 服务器、API 服务等
- **StatefulSet**：管理有状态应用，提供稳定的网络标识（`pod-0`, `pod-1`）和持久存储。适合数据库、消息队列、ZooKeeper 等
- **DaemonSet**：每个节点运行一个 Pod 副本，新节点加入自动部署。适合日志收集（Filebeat）、监控（Node Exporter）、网络插件（Calico）等

---

**Q9: 什么是 liveness probe 和 readiness probe？区别是什么？**

A:
- **Liveness Probe（存活探针）**：检测容器是否存活。失败则重启容器。用于：应用死锁但进程未退出的场景
- **Readiness Probe（就绪探针）**：检测容器是否准备好接收流量。失败则从 Service 的 Endpoints 中移除（不重启）。用于：应用启动慢、数据库连接未就绪的场景
- **Startup Probe（启动探针，K8s 1.16+）**：应用启动期间使用，成功后才启用 liveness/readiness。避免慢启动应用被 liveness 误杀

---

**Q10: 什么是 RBAC？如何给一个 ServiceAccount 授予只读访问 Pod 的权限？**

A: RBAC（基于角色的访问控制）通过 Role/ClusterRole 定义权限，通过 RoleBinding/ClusterRoleBinding 将权限赋给用户/组/ServiceAccount。

```yaml
# 1. 创建 Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

---
# 2. 创建 ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-sa

---
# 3. 绑定
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
subjects:
- kind: ServiceAccount
  name: my-sa
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

**Q11: etcd 在 Kubernetes 中的作用是什么？如何高可用？**

A:
- etcd 是 K8s 的唯一数据存储后端，保存所有集群状态（资源对象、配置等）
- 使用 Raft 共识算法，需要奇数个节点（3 或 5 个）实现高可用
- 数据写入：只有 leader 接受写请求，写成功需要多数节点（quorum）确认
- 生产建议：使用 SSD、独立节点部署、定期备份、监控 etcd 延迟

---

**Q12: Kubernetes 调度器是如何工作的？**

A: 调度器的工作分为两个阶段：
1. **过滤（Filtering）**：找出所有满足 Pod 需求的节点
   - 资源是否充足（CPU/内存）
   - NodeSelector/NodeAffinity 是否匹配
   - Taints/Tolerations 是否匹配
   - 端口是否冲突
   - Volume 亲和性
2. **打分（Scoring）**：为每个可用节点打分
   - 资源均衡（LeastAllocated/MostAllocated）
   - Pod 拓扑分布
   - 镜像是否在本地
   - 亲和性优先度

选择最高分节点，写入 `pod.spec.nodeName`。

---

**Q13: 什么是 ConfigMap 和 Secret？Secret 是安全的吗？**

A:
- **ConfigMap**：存储非敏感配置数据（键值对或文件），可作为环境变量或挂载为文件
- **Secret**：存储敏感数据（密码、Token、密钥），base64 编码（注意：base64 不是加密！）
- **Secret 的安全性**：
  - 默认 Secret 以明文存储在 etcd 中（base64 解码即可获得原始值）
  - 加强措施：启用 etcd 静态加密（`EncryptionConfiguration`）、使用外部密钥管理（HashiCorp Vault、AWS Secrets Manager）、限制 RBAC 权限
  - 在生产环境，应结合 sealed-secrets 或 External Secrets Operator 使用

---

**Q14: 如何实现 Kubernetes 集群的高可用（HA）？**

A: HA 需要从多个层面考虑：
1. **Control Plane HA**：多个 Master 节点（3 个），API Server 前面加负载均衡器，etcd 3/5 节点集群
2. **应用 HA**：Deployment replicas >= 2，设置 PodDisruptionBudget，Pod 跨节点/可用区分布（podAntiAffinity）
3. **节点 HA**：跨可用区部署节点，使用 Cluster Autoscaler 自动替换故障节点
4. **网络 HA**：多个 Ingress 副本，Service 使用 LoadBalancer 类型

---

**Q15: 什么是 PV、PVC 和 StorageClass？它们的关系？**

A:
- **PV（PersistentVolume）**：集群管理员预先创建的存储资源（或动态供应），描述实际存储的配置
- **PVC（PersistentVolumeClaim）**：用户对存储的请求（多少容量、访问模式），K8s 自动将 PVC 绑定到合适的 PV
- **StorageClass**：定义存储的"类别"（provisioner、参数、回收策略），支持动态供应（创建 PVC 时自动创建 PV）
- 绑定流程：`PVC(请求) --> StorageClass(动态创建) --> PV(实际存储) --> 挂载到 Pod`

---

**Q16: 什么是 Helm？Chart 的目录结构是什么？**

A:
- Helm 是 Kubernetes 的包管理器，将相关 K8s 资源打包为 Chart
- Chart 由 `Chart.yaml`（元数据）、`values.yaml`（默认配置）、`templates/`（K8s 资源模板）组成
- Release 是 Chart 在集群中的一次部署实例，支持版本管理和回滚
- 核心价值：参数化配置（不同环境用不同 values）、版本管理、依赖管理

---

**Q17: 描述 kubectl apply 和 kubectl create 的区别？**

A:
- `kubectl create`：命令式操作，资源不存在时创建，已存在时报错
- `kubectl apply`：声明式操作，资源不存在时创建，已存在时对比差异并更新（三方合并：当前状态、上次应用配置、新配置）
- 生产推荐使用 `apply`，因为它是幂等的，适合 GitOps 工作流

---

**Q18: 如何对 Kubernetes 中的应用进行滚动更新和回滚？**

A:
```bash
# 触发滚动更新
kubectl set image deployment/myapp myapp=myapp:v2
kubectl rollout status deployment/myapp   # 监控进度

# 如果出现问题，立即回滚
kubectl rollout undo deployment/myapp
kubectl rollout undo deployment/myapp --to-revision=3

# 查看历史版本
kubectl rollout history deployment/myapp
```
- Deployment 通过 `maxSurge` 和 `maxUnavailable` 控制滚动速度
- `revisionHistoryLimit` 控制保留历史版本数量（默认10）

---

**Q19: 什么是 Kubernetes 的 Namespace？如何做资源隔离？**

A:
- Namespace 提供资源的逻辑隔离，同一 Namespace 内资源名不能重复
- 通过 ResourceQuota 限制命名空间的资源使用（CPU、内存、Pod 数量等）
- 通过 LimitRange 设置容器默认资源限制
- 通过 NetworkPolicy 限制命名空间间的网络访问
- 通过 RBAC 限制命名空间的访问权限
- 注意：Node、PV、StorageClass、ClusterRole 等是集群级别资源，不属于任何 Namespace

---

**Q20: Kubernetes 中的 Service Mesh 是什么？Istio 解决了什么问题？**

A:
- **Service Mesh** 是处理服务间通信的基础设施层，在每个 Pod 中注入 sidecar 代理（Envoy）
- **Istio 解决的问题**：
  1. **流量管理**：金丝雀发布、A/B 测试、故障注入、熔断、限流、重试
  2. **可观测性**：请求追踪（Jaeger）、指标收集（Prometheus）、服务拓扑图（Kiali）
  3. **安全**：mTLS 加密服务间通信、证书自动轮换、细粒度授权策略
- 代价：每个 Pod 注入 sidecar，增加资源消耗（约 0.5 vCPU, 50MB 内存）和延迟

---

**Q21: K8s 中如何处理敏感配置？生产最佳实践？**

A: 生产最佳实践：
1. 不要将密码硬编码在镜像中
2. 使用 Secret（配合 etcd 加密）而非 ConfigMap 存储敏感数据
3. 使用 External Secrets Operator 对接 Vault/AWS SM 等外部密钥管理系统
4. 限制 Secret 的 RBAC 权限
5. 通过 Vault Agent Injector 在运行时动态注入 Secret（Secret 不落 etcd）
6. 使用 Sealed Secrets（Bitnami）将加密的 Secret 存储到 Git 仓库

---

**Q22: 什么是 Pod Topology Spread Constraints？**

A: 拓扑分布约束控制 Pod 在集群拓扑域（节点、可用区等）的分布，比 podAntiAffinity 更灵活：

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1                    # 最大偏差
    topologyKey: topology.kubernetes.io/zone  # 按可用区分布
    whenUnsatisfiable: DoNotSchedule         # 无法满足时不调度
    labelSelector:
      matchLabels:
        app: myapp
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname      # 按节点分布
    whenUnsatisfiable: ScheduleAnyway        # 无法满足时尽力而为
    labelSelector:
      matchLabels:
        app: myapp
```

---

## 附录：常用资源速查

```
核心资源缩写:
  po  = pods
  svc = services
  deploy = deployments
  ds  = daemonsets
  sts = statefulsets
  cm  = configmaps
  sa  = serviceaccounts
  ns  = namespaces
  no  = nodes
  pv  = persistentvolumes
  pvc = persistentvolumeclaims
  ep  = endpoints
  ing = ingresses
  rs  = replicasets
  cj  = cronjobs
  hpa = horizontalpodautoscalers
  pdb = poddisruptionbudgets
  sc  = storageclasses
  rb  = rolebindings
  crb = clusterrolebindings
  cr  = clusterroles
```

---

*文档完成 - Docker & Kubernetes 完全指南*
*涵盖: Docker 基础、镜像、容器、Compose、Harbor、K8s 架构、资源对象、网络、调度、存储、安全、Helm、实战案例、命令大全、面试题*
