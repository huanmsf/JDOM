# Nginx 详解

> 从零到精通：Nginx 架构原理、安装配置、反向代理、负载均衡、HTTPS、高可用全攻略

---

## 目录

- [Part 1: Nginx 简介与架构](#part-1)
- [Part 2: Nginx 安装与管理](#part-2)
- [Part 3: nginx.conf 核心配置](#part-3)
- [Part 4: 静态资源服务](#part-4)
- [Part 5: 反向代理](#part-5)
- [Part 6: 负载均衡](#part-6)
- [Part 7: HTTPS 配置](#part-7)
- [Part 8: 动静分离](#part-8)
- [Part 9: 限流与安全](#part-9)
- [Part 10: 高可用（Keepalived）](#part-10)
- [Part 11: 常见功能配置](#part-11)
- [Part 12: 性能调优](#part-12)
- [Part 13: 完整实战案例](#part-13)
- [Part 14: 高频面试题](#part-14)

---

## Part 1: Nginx 简介与架构 {#part-1}

### 1.1 Nginx 是什么？

Nginx（发音：engine X）是一款高性能的 HTTP 和反向代理服务器，由俄罗斯工程师 Igor Sysoev 于 2004 年发布。

**核心特点**：
- **高并发**：采用事件驱动的异步非阻塞架构，单机可处理数万并发连接
- **低内存**：处理 10000 个并发连接，内存消耗仅约 2.5MB
- **热部署**：可以在不停服务的情况下更新配置和升级版本
- **高扩展性**：模块化设计，丰富的第三方模块

**主要应用场景**：
```
┌────────────────────────────────────────────────────┐
│              Nginx 应用场景                         │
├─────────────┬──────────────┬───────────────────────┤
│  静态资源   │   反向代理   │    负载均衡           │
│  服务器     │   服务器     │    服务器             │
│  (HTML/CSS/ │  (隐藏后端  │  (将请求分发到        │
│  JS/图片)   │   服务地址)  │   多台后端)           │
├─────────────┼──────────────┼───────────────────────┤
│  HTTPS      │   动静分离   │    API 网关           │
│  终止点     │   加速       │    限流/鉴权          │
└─────────────┴──────────────┴───────────────────────┘
```

### 1.2 Nginx vs Apache

| 对比维度 | Nginx | Apache |
|---------|-------|--------|
| 并发模型 | 事件驱动（异步非阻塞） | 进程/线程（同步阻塞） |
| 并发性能 | 极高（10W+ 并发） | 较低（数千并发） |
| 内存消耗 | 极低 | 较高 |
| 静态文件 | 极快 | 快 |
| 动态内容 | 需要反向代理 | 内置（mod_php等） |
| 配置语法 | 简洁的 block 语法 | 分散的 .htaccess |
| 模块加载 | 编译时静态加载 | 运行时动态加载 |
| 适用场景 | 高并发前端代理 | 功能丰富的 Web 服务器 |

### 1.3 Nginx 架构原理

```
                    ┌──────────────────────────┐
                    │       Nginx 进程模型      │
                    └──────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  Master Process（主进程）                               │
│  - 读取并验证配置文件                                   │
│  - 管理 Worker 进程（创建/终止/信号传递）               │
│  - 监听端口（root 权限）                                │
└──────────────────────┬──────────────────────────────────┘
                       │ fork
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Worker 进程1  │ │ Worker 进程2  │ │ Worker 进程N  │
│              │ │              │ │              │
│  Event Loop  │ │  Event Loop  │ │  Event Loop  │
│  处理请求    │ │  处理请求    │ │  处理请求    │
└──────────────┘ └──────────────┘ └──────────────┘
         │                                │
         └──────────── 共享内存 ──────────┘
                  （缓存、会话等）
```

**关键设计**：
- **Master-Worker 模型**：Master 进程管理，Worker 进程处理请求
- **Worker 数量**：通常设置为 CPU 核数，充分利用多核
- **事件驱动**：每个 Worker 使用 epoll（Linux）/ kqueue（macOS）事件驱动
- **无阻塞 I/O**：一个 Worker 可同时处理数千个连接

### 1.4 Nginx 请求处理流程

```
客户端请求
    │
    ▼
[Worker 接收连接]
    │
    ▼
[解析 HTTP 请求行和请求头]
    │
    ▼
[查找匹配的 server 块（基于 Host 头）]
    │
    ▼
[查找匹配的 location 块（基于 URI）]
    │
    ▼
┌──┴──────────────────────────────┐
│  根据 location 类型分发处理      │
├────────────┬────────────────────┤
│ 静态文件   │   反向代理          │
│ 直接返回   │   转发给后端        │
└────────────┴────────────────────┘
    │
    ▼
[生成 HTTP 响应]
    │
    ▼
客户端
```

---

## Part 2: Nginx 安装与管理 {#part-2}

### 2.1 Linux 安装

**CentOS/RHEL（yum）**：
```bash
# 添加官方仓库
cat > /etc/yum.repos.d/nginx.repo << 'EOF'
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
EOF

yum install -y nginx
```

**Ubuntu/Debian（apt）**：
```bash
# 安装依赖
apt install -y curl gnupg2 ca-certificates lsb-release debian-archive-keyring

# 添加签名密钥
curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor     | tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null

# 添加仓库
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/ubuntu `lsb_release -cs` nginx"     | tee /etc/apt/sources.list.d/nginx.list

apt update && apt install -y nginx
```

### 2.2 编译安装（可定制模块）

```bash
# 安装编译依赖
yum install -y gcc gcc-c++ make pcre pcre-devel zlib zlib-devel openssl openssl-devel

# 下载源码
wget http://nginx.org/download/nginx-1.24.0.tar.gz
tar -zxvf nginx-1.24.0.tar.gz
cd nginx-1.24.0

# 配置编译选项
./configure \
    --prefix=/usr/local/nginx \
    --with-http_ssl_module \
    --with-http_gzip_static_module \
    --with-http_stub_status_module \
    --with-http_v2_module \
    --with-stream \
    --with-threads

# 编译安装
make && make install
```

### 2.3 Nginx 管理命令

```bash
# 启动
nginx                           # 前台启动
systemctl start nginx           # systemd 方式
/usr/local/nginx/sbin/nginx     # 自定义安装路径

# 停止
nginx -s stop                   # 强制停止（立即）
nginx -s quit                   # 优雅停止（等待连接处理完）
systemctl stop nginx

# 重载配置（不中断服务）
nginx -s reload
systemctl reload nginx

# 重新打开日志文件（日志切割后使用）
nginx -s reopen

# 检查配置文件语法
nginx -t
nginx -T                        # 检查并打印完整配置

# 查看版本
nginx -v                        # 版本号
nginx -V                        # 版本号 + 编译参数

# 查看 Master 进程 PID
cat /run/nginx.pid

# 发送信号给 Master 进程（等同于 nginx -s）
kill -HUP  $(cat /run/nginx.pid)   # 重载配置
kill -TERM $(cat /run/nginx.pid)   # 快速停止
kill -QUIT $(cat /run/nginx.pid)   # 优雅停止
```

### 2.4 目录结构

```
/etc/nginx/                      # 配置目录（apt/yum 安装）
├── nginx.conf                   # 主配置文件
├── conf.d/                      # 子配置目录（*.conf 自动加载）
│   └── default.conf             # 默认站点配置
├── mime.types                   # MIME 类型映射
├── fastcgi_params               # FastCGI 参数
└── snippets/                    # 可复用的配置片段

/usr/local/nginx/                # 编译安装目录
├── conf/
│   └── nginx.conf
├── html/                        # 默认网站根目录
├── logs/
│   ├── access.log               # 访问日志
│   └── error.log                # 错误日志
└── sbin/
    └── nginx                    # 可执行文件
```

---

## Part 3: nginx.conf 核心配置 {#part-3}

### 3.1 配置文件结构总览

```nginx
# nginx.conf 整体结构

# ─── 全局块 ───────────────────────────────────────────
# 影响 Nginx 全局行为的指令

user  nginx;                    # 运行用户
worker_processes  auto;         # Worker 进程数
error_log  /var/log/nginx/error.log warn;
pid        /run/nginx.pid;

# ─── events 块 ────────────────────────────────────────
events {
    worker_connections  1024;   # 每个 Worker 最大连接数
    use epoll;                  # 事件驱动模型
    multi_accept on;            # 一次接受多个新连接
}

# ─── http 块 ──────────────────────────────────────────
http {
    include       mime.types;
    default_type  application/octet-stream;

    # 日志格式
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    keepalive_timeout  65;

    # ─── server 块（虚拟主机）────────────────────────
    server {
        listen       80;
        server_name  example.com;

        # ─── location 块（路由匹配）────────────────
        location / {
            root   /usr/share/nginx/html;
            index  index.html;
        }

        location /api/ {
            proxy_pass http://backend:8080;
        }
    }
}

# ─── stream 块（TCP/UDP 代理）─────────────────────────
stream {
    server {
        listen 3306;
        proxy_pass mysql_backend;
    }
}
```

### 3.2 全局块配置详解

```nginx
# 运行 Nginx 的用户和组
user nginx nginx;

# Worker 进程数（auto = CPU 核数）
worker_processes auto;

# 绑定 Worker 到 CPU 核（减少缓存失效）
worker_cpu_affinity auto;

# Worker 进程最大打开文件数（需同时设置系统 ulimit）
worker_rlimit_nofile 65535;

# 错误日志（可选级别：debug|info|notice|warn|error|crit）
error_log /var/log/nginx/error.log warn;

# PID 文件
pid /run/nginx.pid;

# 加载动态模块
load_module modules/ngx_http_image_filter_module.so;
```

### 3.3 events 块配置详解

```nginx
events {
    # 每个 Worker 进程最大并发连接数
    # 系统最大并发 = worker_processes × worker_connections
    worker_connections 10240;

    # 事件驱动模型（Linux 推荐 epoll）
    use epoll;

    # 允许一个 Worker 一次 accept 多个新连接（高并发推荐开启）
    multi_accept on;

    # accept 互斥锁（防止惊群效应，Nginx 1.11.3+ 默认关闭）
    accept_mutex off;
}
```

### 3.4 http 块常用配置

```nginx
http {
    # ── MIME 类型 ──────────────────────────────────────
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # ── 字符集 ─────────────────────────────────────────
    charset utf-8;

    # ── 访问日志 ───────────────────────────────────────
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for" '
                      'rt=$request_time uct=$upstream_connect_time '
                      'uht=$upstream_header_time urt=$upstream_response_time';

    access_log  /var/log/nginx/access.log  main  buffer=16k  flush=5s;

    # ── 性能优化 ───────────────────────────────────────
    sendfile        on;         # 零拷贝传输文件
    tcp_nopush      on;         # 优化数据包发送（sendfile=on 时有效）
    tcp_nodelay     on;         # 小数据包立即发送（keepalive 时有效）

    # ── 连接超时 ───────────────────────────────────────
    keepalive_timeout    65;    # 长连接超时（0 禁用）
    keepalive_requests  100;    # 长连接最大请求数
    client_header_timeout 15;  # 读取请求头超时
    client_body_timeout   15;  # 读取请求体超时
    send_timeout          15;  # 向客户端发送超时

    # ── 请求体 ─────────────────────────────────────────
    client_max_body_size 100m;     # 最大请求体大小（文件上传）
    client_body_buffer_size  128k; # 请求体缓冲区大小

    # ── Gzip 压缩 ──────────────────────────────────────
    gzip  on;
    gzip_min_length  1k;
    gzip_comp_level  6;
    gzip_types  text/plain text/css application/json application/javascript
                text/xml application/xml image/svg+xml;
    gzip_vary  on;
    gzip_disable "MSIE [1-6]\.";

    # ── 隐藏版本号 ─────────────────────────────────────
    server_tokens off;

    # ── 引入子配置 ─────────────────────────────────────
    include /etc/nginx/conf.d/*.conf;
}
```

### 3.5 server 块与虚拟主机

```nginx
# 默认站点（捕获不匹配的请求）
server {
    listen 80 default_server;
    server_name _;
    return 444;  # 直接断开连接
}

# 基于域名的虚拟主机
server {
    listen 80;
    server_name www.example.com example.com;

    # 重定向 www 到非 www
    if ($host = www.example.com) {
        return 301 https://example.com$request_uri;
    }

    root /var/www/example.com;
    index index.html;

    access_log /var/log/nginx/example.access.log main;
    error_log  /var/log/nginx/example.error.log warn;
}

# 基于 IP 的虚拟主机
server {
    listen 192.168.1.100:8080;
    server_name _;
    # ...
}

# 多个端口
server {
    listen 8080;
    listen 8443 ssl;
    server_name api.example.com;
    # ...
}
```

### 3.6 location 匹配规则

```
location 匹配优先级（从高到低）：

1. = （精确匹配）     最高优先级，精确相等才匹配
2. ^~ （前缀匹配）    前缀匹配，一旦匹配则停止搜索正则
3. ~ （正则匹配）     大小写敏感的正则
4. ~* （正则匹配）    大小写不敏感的正则
5. / （普通匹配）     最长前缀匹配（兜底）
```

```nginx
server {
    # 1. 精确匹配（最高优先级）
    location = / {
        return 200 "精确匹配根路径";
    }

    # 2. 前缀匹配（匹配后不再搜索正则）
    location ^~ /images/ {
        root /var/www/static;
    }

    # 3. 大小写敏感的正则
    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
    }

    # 4. 大小写不敏感的正则
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 30d;
        add_header Cache-Control "public, no-transform";
    }

    # 5. 普通前缀匹配（兜底）
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

**匹配示例**：

| 请求 URI | 匹配 location |
|---------|--------------|
| `/` | `= /` |
| `/images/logo.png` | `^~ /images/` |
| `/upload.php` | `~ \.php$` |
| `/style.CSS` | `~* \.(css)$` |
| `/api/users` | `/` |

### 3.7 Nginx 内置变量

```nginx
# 请求相关
$request_method     # GET, POST, ...
$request_uri        # 完整 URI（包含参数），如 /path?a=1
$uri                # 规范化后的 URI（不含参数）
$args               # 查询字符串，如 a=1&b=2
$arg_name           # 特定参数，如 $arg_token
$is_args            # 有参数时为 "?"，否则为 ""
$query_string       # 同 $args
$request_body       # 请求体
$request_filename   # 请求文件的完整路径

# 客户端相关
$remote_addr        # 客户端 IP
$remote_port        # 客户端端口
$http_host          # Host 请求头
$http_user_agent    # User-Agent
$http_referer       # Referer
$http_x_forwarded_for  # X-Forwarded-For

# 响应相关
$status             # HTTP 状态码
$body_bytes_sent    # 发送的响应体字节数

# 连接相关
$server_name        # server_name 指令的值
$server_port        # 监听端口
$scheme             # http 或 https

# 时间相关
$time_local         # 本地时间
$request_time       # 请求处理时长（秒）

# 上游相关
$upstream_addr              # 上游服务器地址
$upstream_status            # 上游响应状态码
$upstream_response_time     # 上游响应时间
$upstream_connect_time      # 连接上游时间
```

---

## Part 4: 静态资源服务 {#part-4}

### 4.1 基本静态文件配置

```nginx
server {
    listen 80;
    server_name static.example.com;

    # 网站根目录
    root /var/www/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
        # try_files 依次尝试：
        # 1. $uri：查找文件
        # 2. $uri/：查找目录下的 index 文件
        # 3. =404：返回 404
    }
}
```

### 4.2 缓存控制

```nginx
server {
    # 图片、字体等资源：长期缓存
    location ~* \.(jpg|jpeg|png|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 365d;
        add_header Cache-Control "public, immutable";
        add_header Vary "Accept-Encoding";
    }

    # CSS/JS：中期缓存（带版本号时可用长期缓存）
    location ~* \.(css|js)$ {
        expires 30d;
        add_header Cache-Control "public, max-age=2592000";
    }

    # HTML：不缓存（内容频繁变化）
    location ~* \.html$ {
        expires -1;
        add_header Cache-Control "no-cache, no-store, must-revalidate";
        add_header Pragma "no-cache";
    }
}
```

### 4.3 目录浏览

```nginx
location /files/ {
    root /var/www;
    autoindex on;                   # 开启目录浏览
    autoindex_exact_size off;       # 显示人类可读的文件大小
    autoindex_localtime on;         # 显示本地时间
    autoindex_format html;          # 输出格式：html/json/xml/jsonp
}
```

### 4.4 文件下载优化

```nginx
location /downloads/ {
    root /var/www;

    # 大文件下载优化
    sendfile on;
    aio on;             # 异步 I/O
    directio 512;       # 大于 512KB 的文件使用 directio

    # 限速（避免单用户占满带宽）
    limit_rate 1m;      # 限速 1MB/s
    limit_rate_after 5m; # 前 5MB 不限速

    # 断点续传支持
    # Nginx 默认支持，客户端发 Range 头即可
}
```

### 4.5 SPA（单页应用）配置

```nginx
server {
    listen 80;
    server_name app.example.com;
    root /var/www/app/dist;

    # 所有路由都返回 index.html（前端路由）
    location / {
        try_files $uri $uri/ /index.html;
    }

    # API 请求反向代理
    location /api/ {
        proxy_pass http://backend:8080/api/;
    }

    # 静态资源缓存
    location ~* \.(js|css|png|jpg|svg|ico)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

---

## Part 5: 反向代理 {#part-5}

### 5.1 基本反向代理

```nginx
server {
    listen 80;
    server_name api.example.com;

    location / {
        # 代理到后端服务
        proxy_pass http://127.0.0.1:8080;

        # 传递客户端真实 IP
        proxy_set_header Host              $http_host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 5.2 proxy_pass 路径处理

```nginx
# 规则：proxy_pass 末尾带 / 会替换 location 路径，不带 / 会拼接

server {
    # 情况1：不带斜杠（URI 直接拼接）
    location /api/ {
        proxy_pass http://backend:8080;
        # 请求 /api/users → 转发到 http://backend:8080/api/users
    }

    # 情况2：带斜杠（替换 location 前缀）
    location /api/ {
        proxy_pass http://backend:8080/;
        # 请求 /api/users → 转发到 http://backend:8080/users
    }

    # 情况3：带路径前缀
    location /api/ {
        proxy_pass http://backend:8080/v1/;
        # 请求 /api/users → 转发到 http://backend:8080/v1/users
    }
}
```

### 5.3 完整的反向代理配置

```nginx
# 推荐提取为通用代理头配置
# /etc/nginx/snippets/proxy-params.conf
proxy_set_header Host              $http_host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host  $server_name;

proxy_connect_timeout   60s;     # 连接后端超时
proxy_send_timeout      60s;     # 向后端发送数据超时
proxy_read_timeout      60s;     # 从后端读取数据超时

proxy_buffering         on;
proxy_buffer_size       4k;
proxy_buffers           8 4k;
proxy_busy_buffers_size 8k;

proxy_http_version      1.1;
proxy_set_header Connection "";   # 支持 HTTP/1.1 长连接

# 使用方式
server {
    location /api/ {
        include snippets/proxy-params.conf;
        proxy_pass http://backend:8080/;
    }
}
```

### 5.4 代理缓存

```nginx
http {
    # 定义缓存区域
    proxy_cache_path /var/cache/nginx
        levels=1:2                  # 两层目录结构
        keys_zone=api_cache:10m     # 共享内存名称和大小
        max_size=1g                 # 最大磁盘使用量
        inactive=60m                # 未访问60分钟后清除
        use_temp_path=off;

    server {
        location /api/ {
            proxy_pass http://backend:8080/;

            # 启用缓存
            proxy_cache api_cache;
            proxy_cache_key "$scheme$request_method$host$request_uri";
            proxy_cache_valid 200 302 10m;   # 200/302 响应缓存10分钟
            proxy_cache_valid 404     1m;    # 404 缓存1分钟
            proxy_cache_valid any     5m;    # 其他状态码缓存5分钟

            # 缓存绕过条件
            proxy_cache_bypass $http_pragma;
            proxy_no_cache     $http_pragma;

            # 添加缓存状态头（调试用）
            add_header X-Cache-Status $upstream_cache_status;
        }
    }
}
```

---

## Part 6: 负载均衡 {#part-6}

### 6.1 负载均衡策略

```
┌─────────────────────────────────────────────────────────┐
│                    Nginx 负载均衡                        │
├──────────────┬──────────────────────────────────────────┤
│  策略名称    │  说明                                     │
├──────────────┼──────────────────────────────────────────┤
│  轮询        │  默认，依次分配（Round Robin）             │
│  weight      │  加权轮询，性能强的服务器分配更多请求      │
│  ip_hash     │  同一 IP 始终访问同一后端（会话保持）      │
│  least_conn  │  分配给当前连接数最少的服务器              │
│  hash        │  自定义哈希键（URL、Header等）             │
│  random      │  随机分配（1.15.1+）                      │
└──────────────┴──────────────────────────────────────────┘
```

### 6.2 轮询（默认）

```nginx
http {
    upstream backend {
        server 192.168.1.101:8080;
        server 192.168.1.102:8080;
        server 192.168.1.103:8080;
    }

    server {
        listen 80;
        location / {
            proxy_pass http://backend;
        }
    }
}
```

### 6.3 加权轮询

```nginx
upstream backend {
    server 192.168.1.101:8080 weight=5;   # 处理 5/8 的请求
    server 192.168.1.102:8080 weight=2;   # 处理 2/8 的请求
    server 192.168.1.103:8080 weight=1;   # 处理 1/8 的请求
}
```

### 6.4 IP Hash（会话保持）

```nginx
upstream backend {
    ip_hash;   # 同一客户端 IP 始终路由到同一后端
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    server 192.168.1.103:8080;
}
```

### 6.5 最少连接数

```nginx
upstream backend {
    least_conn;   # 分配给当前活跃连接数最少的服务器
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}
```

### 6.6 server 参数详解

```nginx
upstream backend {
    server 192.168.1.101:8080 weight=3 max_fails=3 fail_timeout=30s;
    server 192.168.1.102:8080 weight=3 max_fails=3 fail_timeout=30s;
    server 192.168.1.103:8080 backup;    # 备用服务器（其他全挂时启用）
    server 192.168.1.104:8080 down;      # 暂时下线（不接受请求）

    # max_fails: 在 fail_timeout 内失败 max_fails 次，则标记为不可用
    # fail_timeout: 不可用持续时间（之后重新尝试）
    # backup: 备用服务器
    # down: 永久下线
}
```

### 6.7 长连接复用（提升性能）

```nginx
upstream backend {
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    keepalive 32;   # 每个 Worker 保持 32 个到上游的长连接
}

server {
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;           # 必须使用 HTTP/1.1
        proxy_set_header Connection "";    # 清除 Connection: close
    }
}
```

### 6.8 健康检查（商业版 nginx-plus 或第三方模块）

```nginx
# nginx_upstream_check_module（第三方）
upstream backend {
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;

    check interval=3000 rise=2 fall=3 timeout=1000 type=http;
    check_http_send "HEAD /health HTTP/1.0\r\n\r\n";
    check_http_expect_alive http_2xx http_3xx;
}

# 查看健康状态页面
server {
    location /upstream_status {
        check_status;
        access_log off;
        allow 127.0.0.1;
        deny all;
    }
}
```

### 6.9 完整负载均衡配置示例

```nginx
http {
    # 上游服务器组
    upstream api_servers {
        least_conn;

        server 10.0.0.1:8080 weight=2 max_fails=2 fail_timeout=10s;
        server 10.0.0.2:8080 weight=2 max_fails=2 fail_timeout=10s;
        server 10.0.0.3:8080 weight=1 max_fails=2 fail_timeout=10s;
        server 10.0.0.4:8080 backup;

        keepalive 100;
    }

    server {
        listen 80;
        server_name api.example.com;

        location / {
            proxy_pass http://api_servers;
            proxy_http_version 1.1;
            proxy_set_header Connection "";

            proxy_set_header Host              $http_host;
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_connect_timeout 5s;
            proxy_read_timeout   30s;

            # 失败时切换到下一个上游
            proxy_next_upstream error timeout invalid_header http_500 http_502;
            proxy_next_upstream_tries 3;
        }

        # 健康检查端点（直接访问某台后端）
        location = /health {
            proxy_pass http://10.0.0.1:8080/health;
            access_log off;
        }
    }
}
```

---

## Part 7: HTTPS 配置 {#part-7}

### 7.1 SSL/TLS 基础

```
HTTPS = HTTP + SSL/TLS

握手过程（TLS 1.2 简化版）：
┌────────┐                        ┌────────┐
│ 客户端 │                        │ 服务端 │
└────────┘                        └────────┘
     │─── 1. Client Hello ────────────►│
     │        （支持的加密套件）        │
     │◄─── 2. Server Hello ────────────│
     │        （选定加密套件）          │
     │◄─── 3. 证书（公钥）─────────────│
     │─── 4. 验证证书，发送密钥────────►│
     │◄─── 5. 确认，握手完成 ──────────│
     │═══ 6. 加密通信 ════════════════│
```

### 7.2 自签名证书（测试用）

```bash
# 生成自签名证书（生产环境不要用）
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/nginx/ssl/nginx.key \
    -out /etc/nginx/ssl/nginx.crt \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=Example/CN=example.com"
```

### 7.3 Let's Encrypt 免费证书（Certbot）

```bash
# 安装 Certbot
apt install certbot python3-certbot-nginx

# 自动获取并配置证书（会自动修改 nginx.conf）
certbot --nginx -d example.com -d www.example.com

# 测试自动续期
certbot renew --dry-run

# 添加自动续期 cron
echo "0 12 * * * /usr/bin/certbot renew --quiet" | crontab -
```

### 7.4 HTTPS 完整配置

```nginx
# HTTP → HTTPS 重定向
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    # 支持 ACME 证书验证
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS 主配置
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com www.example.com;

    # 证书文件
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;

    # TLS 版本（禁用不安全的 TLS 1.0/1.1）
    ssl_protocols TLSv1.2 TLSv1.3;

    # 加密套件（Mozilla 推荐配置）
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # Diffie-Hellman 参数（增强 Forward Secrecy）
    ssl_dhparam /etc/nginx/ssl/dhparam.pem;

    # SSL 会话缓存（避免重复握手）
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    # OCSP Stapling（加速证书验证）
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;

    # HSTS（强制客户端使用 HTTPS，max-age=1年）
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    # 其他安全头
    add_header X-Frame-Options SAMEORIGIN always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    root /var/www/example.com;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### 7.5 生成 DH 参数

```bash
# 生成 2048 位 DH 参数（耗时约1-2分钟）
openssl dhparam -out /etc/nginx/ssl/dhparam.pem 2048
```

---

## Part 8: 动静分离 {#part-8}

### 8.1 动静分离的意义

```
传统架构（动静不分离）：
客户端 → Tomcat → 处理所有请求（静态+动态）
                   缺点：Tomcat 处理静态文件效率低

动静分离架构：
                        ┌─► Nginx 直接返回（高效）
客户端 → Nginx ──────────┤  （/static/ → 本地文件）
                        └─► Tomcat 处理（动态业务）
                           （/api/ → 后端服务）
```

### 8.2 动静分离配置

```nginx
server {
    listen 80;
    server_name www.example.com;

    # ── 静态资源：Nginx 直接处理 ──────────────────────
    location /static/ {
        root /var/www;
        # 请求 /static/logo.png → 读取 /var/www/static/logo.png

        expires 30d;
        add_header Cache-Control "public";
        access_log off;  # 关闭静态资源访问日志（减少 I/O）
    }

    location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff2?)$ {
        root /var/www/html;
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # ── 动态请求：代理到后端 ───────────────────────────
    location /api/ {
        proxy_pass http://tomcat:8080;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # ── 页面：SPA 或后端渲染 ───────────────────────────
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### 8.3 CDN + 动静分离

```
完整的动静分离架构：

用户
 │
 ├──►[CDN节点]───►[Nginx 静态服务器]──► /var/www/static
 │    （静态资源）
 │
 └──►[Nginx 反向代理]──►[后端集群]
      （动态请求）
```

---

## Part 9: 限流与安全 {#part-9}

### 9.1 请求限流（limit_req）

```nginx
http {
    # 定义限流区域
    # key: $binary_remote_addr（按IP限流）
    # zone=name:size（共享内存区名称和大小）
    # rate=10r/s（每秒10个请求）
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=login_limit:10m rate=1r/s;

    # 全局限流日志级别
    limit_req_log_level warn;
    limit_req_status 429;  # 超限返回 429 Too Many Requests

    server {
        # API 接口限流
        location /api/ {
            limit_req zone=api_limit burst=20 nodelay;
            # burst=20：允许突发20个请求排队
            # nodelay：排队的请求立即处理（不延迟），超出 burst 则拒绝
            proxy_pass http://backend;
        }

        # 登录接口严格限流
        location /api/auth/login {
            limit_req zone=login_limit burst=5;
            proxy_pass http://backend;
        }
    }
}
```

### 9.2 并发连接限制（limit_conn）

```nginx
http {
    # 限制每个IP的并发连接数
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;
    limit_conn_status 503;

    server {
        location / {
            limit_conn conn_limit 20;  # 每个IP最多20个并发连接
            proxy_pass http://backend;
        }

        # 下载接口限速
        location /download/ {
            limit_conn conn_limit 5;   # 每个IP最多5个并发下载
            limit_rate 512k;           # 每个连接限速 512KB/s
            root /var/www/downloads;
        }
    }
}
```

### 9.3 IP 访问控制

```nginx
# 白名单
location /admin/ {
    allow 192.168.1.0/24;  # 允许内网
    allow 10.0.0.0/8;
    allow 127.0.0.1;
    deny all;              # 拒绝其他所有
    proxy_pass http://backend;
}

# 黑名单
location / {
    deny 1.2.3.4;           # 封禁特定 IP
    deny 5.6.7.0/24;        # 封禁 IP 段
    allow all;
}
```

### 9.4 防止常见攻击

```nginx
server {
    # 禁止访问隐藏文件（.htaccess, .git 等）
    location ~ /\. {
        deny all;
        return 404;
    }

    # 禁止访问备份文件
    location ~* \.(bak|sql|tar|gz|log|sh)$ {
        deny all;
        return 404;
    }

    # 限制请求方法（只允许 GET、HEAD、POST）
    if ($request_method !~ ^(GET|HEAD|POST)$) {
        return 405;
    }

    # 防止 SQL 注入（基本过滤，建议用 WAF）
    if ($query_string ~* "union.*select|insert.*into|delete.*from") {
        return 403;
    }

    # 防止目录遍历
    if ($request_uri ~* "\.\./") {
        return 403;
    }

    # 限制请求头大小（防止头部攻击）
    large_client_header_buffers 4 8k;
    client_header_buffer_size 1k;
}
```

### 9.5 防盗链

```nginx
server {
    location ~* \.(jpg|jpeg|png|gif|mp4|mp3)$ {
        # 只允许来自自己网站的请求
        valid_referers none blocked server_names
                       *.example.com example.com;

        if ($invalid_referer) {
            return 403;
            # 或返回防盗链图片
            # rewrite ^ /images/forbidden.jpg last;
        }
    }
}
```

### 9.6 请求体大小与超时

```nginx
server {
    client_max_body_size 10m;      # 最大请求体 10MB
    client_body_timeout  30s;      # 读取请求体超时
    client_header_timeout 10s;     # 读取请求头超时
    send_timeout         30s;      # 发送响应超时

    # 慢连接攻击防护
    # 如果客户端发送请求体速度太慢，超时断开
}
```

---

## Part 10: 高可用（Keepalived + Nginx）{#part-10}

### 10.1 高可用架构

```
传统单点架构（单点故障）：
用户 ──► Nginx（单点）──► 后端集群

高可用架构（VIP 漂移）：
                      ┌─► Nginx 主（192.168.1.101）
用户 ──► VIP ──────────┤         │ 心跳检测
（192.168.1.100）      └─► Nginx 备（192.168.1.102）

当主机故障，VIP 自动漂移到备机：
用户 ──► VIP ────────────► Nginx 备（接管服务）
```

### 10.2 Keepalived 安装配置

```bash
# 安装 Keepalived
yum install -y keepalived
```

**主节点配置** (`/etc/keepalived/keepalived.conf`)：

```
! Configuration File for keepalived

global_defs {
    notification_email {
        ops@example.com          # 故障通知邮件
    }
    notification_email_from keepalived@example.com
    smtp_server 127.0.0.1
    smtp_connect_timeout 30
    router_id NGINX_MASTER       # 路由器标识（唯一）
}

# 检测 Nginx 健康的脚本
vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2                   # 每2秒检查一次
    weight -20                   # 检查失败则优先级降低20
    fall 3                       # 连续失败3次认为不可用
    rise 2                       # 连续成功2次认为可用
}

vrrp_instance VI_1 {
    state MASTER                 # 主节点
    interface eth0               # 监听网卡
    virtual_router_id 51         # VRRP 虚拟路由ID（主备相同）
    priority 100                 # 优先级（主 > 备）
    advert_int 1                 # 心跳间隔（秒）
    preempt                      # 抢占模式（主恢复后重新接管）

    authentication {
        auth_type PASS
        auth_pass nginx123       # 认证密码（主备相同）
    }

    virtual_ipaddress {
        192.168.1.100/24 dev eth0  # VIP
    }

    track_script {
        check_nginx              # 绑定健康检查脚本
    }
}
```

**备节点配置**（仅差异部分）：
```
vrrp_instance VI_1 {
    state BACKUP                 # 备节点
    priority 90                  # 优先级低于主节点
    nopreempt                    # 非抢占模式（可选）
    # 其余与主节点相同
}
```

**Nginx 健康检查脚本** (`/etc/keepalived/check_nginx.sh`)：

```bash
#!/bin/bash
# 检查 Nginx 进程是否存在
if ! pgrep -x "nginx" > /dev/null; then
    # 尝试重启 Nginx
    systemctl start nginx
    sleep 2
    # 再次检查
    if ! pgrep -x "nginx" > /dev/null; then
        exit 1  # 重启失败，让 VIP 漂移
    fi
fi
exit 0
```

```bash
chmod +x /etc/keepalived/check_nginx.sh
systemctl start keepalived
systemctl enable keepalived
```

---

## Part 11: 常见功能配置 {#part-11}

### 11.1 URL 重写（rewrite）

```nginx
server {
    # 语法：rewrite regex replacement [flag]
    # flag: last（继续匹配）、break（停止）、redirect（302）、permanent（301）

    # 旧 URL 永久重定向到新 URL
    rewrite ^/old-path/(.*)$ /new-path/$1 permanent;

    # 规范化 URL（去掉多余斜杠）
    rewrite ^//(.*)$ /$1 last;

    # 强制添加 www
    if ($host !~ ^www\.) {
        rewrite ^ https://www.$host$request_uri permanent;
    }

    # 去掉 .html 后缀
    rewrite ^(/.+)\.html$ $1 last;
}
```

### 11.2 跨域（CORS）配置

```nginx
# 方案1：简单 CORS（所有接口允许）
server {
    add_header 'Access-Control-Allow-Origin' '*' always;
    add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
    add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type' always;

    # 预检请求处理
    if ($request_method = 'OPTIONS') {
        add_header 'Access-Control-Max-Age' 1728000;
        add_header 'Content-Length' 0;
        return 204;
    }
}

# 方案2：精细化 CORS（按来源白名单）
map $http_origin $cors_origin {
    default "";
    "https://app.example.com" $http_origin;
    "https://admin.example.com" $http_origin;
}

server {
    location /api/ {
        add_header 'Access-Control-Allow-Origin' $cors_origin always;
        add_header 'Access-Control-Allow-Credentials' 'true' always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
        add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type' always;

        if ($request_method = 'OPTIONS') {
            return 204;
        }

        proxy_pass http://backend;
    }
}
```

### 11.3 WebSocket 代理

```nginx
server {
    location /ws/ {
        proxy_pass http://websocket_backend;

        # WebSocket 必须升级协议
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # WebSocket 长连接超时（根据业务需要设置）
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;

        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 11.4 gRPC 代理

```nginx
server {
    listen 50051 http2;

    location / {
        grpc_pass grpc://backend:50051;

        # gRPC 超时
        grpc_connect_timeout 5s;
        grpc_read_timeout 60s;
        grpc_send_timeout 60s;
    }
}
```

### 11.5 错误页面自定义

```nginx
server {
    # 自定义错误页面
    error_page 404              /404.html;
    error_page 500 502 503 504  /50x.html;

    location = /404.html {
        root /var/www/errors;
        internal;  # 只允许内部重定向访问
    }

    location = /50x.html {
        root /var/www/errors;
        internal;
    }

    # 重定向到外部错误页面
    error_page 403 https://example.com/forbidden;
}
```

### 11.6 访问日志定制

```nginx
http {
    # JSON 格式日志（便于 ELK 解析）
    log_format json_combined escape=json
        '{'
            '"time":"$time_iso8601",'
            '"remote_addr":"$remote_addr",'
            '"method":"$request_method",'
            '"uri":"$request_uri",'
            '"status":$status,'
            '"body_bytes":$body_bytes_sent,'
            '"request_time":$request_time,'
            '"upstream_time":"$upstream_response_time",'
            '"http_referer":"$http_referer",'
            '"user_agent":"$http_user_agent",'
            '"x_forwarded_for":"$http_x_forwarded_for"'
        '}';

    access_log /var/log/nginx/access.json.log json_combined;

    # 按天切割日志（配合 logrotate）
    # /etc/logrotate.d/nginx
}
```

### 11.7 Nginx 状态监控

```nginx
server {
    location /nginx_status {
        stub_status;
        access_log off;
        allow 127.0.0.1;
        allow 192.168.0.0/16;
        deny all;
    }
}

# 返回内容示例：
# Active connections: 291
# server accepts handled requests
#  16630948 16630948 31070465
# Reading: 6 Writing: 179 Waiting: 106
```

---

## Part 12: 性能调优 {#part-12}

### 12.1 系统层面优化

```bash
# 修改系统最大文件描述符数
echo "* soft nofile 65535" >> /etc/security/limits.conf
echo "* hard nofile 65535" >> /etc/security/limits.conf

# 修改内核参数（/etc/sysctl.conf）
cat >> /etc/sysctl.conf << 'EOF'
# 最大套接字缓冲区
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216

# TCP 连接队列
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535

# 快速回收 TIME_WAIT 状态的连接
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 30

# 本地端口范围
net.ipv4.ip_local_port_range = 1024 65535
EOF

sysctl -p
```

### 12.2 Nginx 配置层面优化

```nginx
# nginx.conf 性能优化完整配置

user nginx;
worker_processes auto;
worker_cpu_affinity auto;
worker_rlimit_nofile 65535;

error_log /var/log/nginx/error.log warn;
pid /run/nginx.pid;

events {
    use epoll;
    worker_connections 10240;
    multi_accept on;
    accept_mutex off;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # ── 性能关键配置 ────────────────────────────────────
    sendfile            on;        # 零拷贝
    tcp_nopush          on;        # 合并数据包
    tcp_nodelay         on;        # 禁用 Nagle 算法

    # ── 长连接 ─────────────────────────────────────────
    keepalive_timeout   65;
    keepalive_requests  1000;

    # ── 缓冲区 ─────────────────────────────────────────
    client_body_buffer_size     128k;
    client_header_buffer_size   1k;
    large_client_header_buffers 4 8k;
    output_buffers              2 32k;
    postpone_output             1460;

    # ── Open File Cache（缓存文件句柄）─────────────────
    open_file_cache          max=10000 inactive=30s;
    open_file_cache_valid    60s;
    open_file_cache_min_uses 2;
    open_file_cache_errors   on;

    # ── Gzip ────────────────────────────────────────────
    gzip              on;
    gzip_min_length   1k;
    gzip_comp_level   5;
    gzip_types        text/plain text/css text/xml text/javascript
                      application/json application/javascript
                      application/xml image/svg+xml;
    gzip_vary         on;
    gzip_buffers      16 8k;
    gzip_http_version 1.1;

    # ── 隐藏 Nginx 版本 ─────────────────────────────────
    server_tokens     off;

    include /etc/nginx/conf.d/*.conf;
}
```

### 12.3 负载均衡性能优化

```nginx
upstream backend {
    keepalive 300;                # 连接池大小（重要！）
    keepalive_timeout 60s;
    keepalive_requests 1000;

    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}

server {
    location / {
        proxy_pass http://backend;

        # 必须开启 HTTP/1.1 才能使用 keepalive
        proxy_http_version 1.1;
        proxy_set_header Connection "";

        # 关闭代理缓冲（针对流式响应）
        # proxy_buffering off;

        # 调整代理缓冲区大小（根据响应大小）
        proxy_buffer_size    128k;
        proxy_buffers        4 256k;
        proxy_busy_buffers_size 256k;
    }
}
```

---

## Part 13: 完整实战案例 {#part-13}

### 案例 1：Spring Boot 微服务网关

**架构**：
```
Internet
   │
   ▼
Nginx（80/443）
   │
   ├──► /api/user/**    → user-service:8081
   ├──► /api/order/**   → order-service:8082
   ├──► /api/product/** → product-service:8083
   └──► /**              → Vue 前端静态资源
```

**完整配置**：

```nginx
# /etc/nginx/nginx.conf

user nginx;
worker_processes auto;
worker_rlimit_nofile 65535;

events {
    use epoll;
    worker_connections 10240;
    multi_accept on;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format main '$remote_addr - [$time_local] "$request" '
                    '$status $body_bytes_sent rt=$request_time '
                    'urt=$upstream_response_time';

    sendfile on;
    tcp_nopush on;
    keepalive_timeout 65;
    gzip on;
    gzip_types text/plain application/json application/javascript text/css;
    server_tokens off;

    # ── 上游服务 ────────────────────────────────────────
    upstream user_service {
        least_conn;
        server 10.0.0.10:8081 max_fails=2 fail_timeout=10s;
        server 10.0.0.11:8081 max_fails=2 fail_timeout=10s;
        keepalive 20;
    }

    upstream order_service {
        least_conn;
        server 10.0.0.20:8082 max_fails=2 fail_timeout=10s;
        server 10.0.0.21:8082 max_fails=2 fail_timeout=10s;
        keepalive 20;
    }

    upstream product_service {
        least_conn;
        server 10.0.0.30:8083 max_fails=2 fail_timeout=10s;
        server 10.0.0.31:8083 max_fails=2 fail_timeout=10s;
        keepalive 20;
    }

    # ── 限流配置 ────────────────────────────────────────
    limit_req_zone $binary_remote_addr zone=api_rate:10m rate=100r/s;
    limit_req_zone $binary_remote_addr zone=login_rate:10m rate=5r/m;
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

    # ── HTTP → HTTPS 重定向 ─────────────────────────────
    server {
        listen 80;
        server_name app.example.com;
        return 301 https://$host$request_uri;
    }

    # ── 主站点 ──────────────────────────────────────────
    server {
        listen 443 ssl http2;
        server_name app.example.com;

        ssl_certificate     /etc/letsencrypt/live/app.example.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;
        ssl_protocols       TLSv1.2 TLSv1.3;
        ssl_session_cache   shared:SSL:10m;
        ssl_session_timeout 1d;

        add_header Strict-Transport-Security "max-age=31536000" always;
        add_header X-Frame-Options SAMEORIGIN always;
        add_header X-Content-Type-Options nosniff always;

        # 全局连接数限制
        limit_conn conn_limit 50;

        # ── 前端静态资源 ─────────────────────────────
        location / {
            root /var/www/app/dist;
            index index.html;
            try_files $uri $uri/ /index.html;

            location ~* \.(js|css|png|jpg|svg|ico|woff2?)$ {
                expires 1y;
                add_header Cache-Control "public, immutable";
                access_log off;
            }
        }

        # ── 用户服务 API ──────────────────────────────
        location /api/user/ {
            limit_req zone=api_rate burst=50 nodelay;

            proxy_pass http://user_service/;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_connect_timeout 5s;
            proxy_read_timeout 30s;
            proxy_next_upstream error timeout http_500;
        }

        # 登录接口额外限流
        location = /api/user/auth/login {
            limit_req zone=login_rate burst=3 nodelay;

            proxy_pass http://user_service/auth/login;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # ── 订单服务 API ──────────────────────────────
        location /api/order/ {
            limit_req zone=api_rate burst=50 nodelay;

            proxy_pass http://order_service/;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_connect_timeout 5s;
            proxy_read_timeout 60s;  # 订单处理时间可能较长
        }

        # ── 商品服务 API（带缓存）─────────────────────
        location /api/product/ {
            limit_req zone=api_rate burst=100 nodelay;

            proxy_pass http://product_service/;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;

            # GET 请求缓存
            proxy_cache api_cache;
            proxy_cache_methods GET HEAD;
            proxy_cache_valid 200 5m;
            proxy_cache_key "$scheme$host$request_uri";
            proxy_cache_bypass $http_cache_control;
            add_header X-Cache-Status $upstream_cache_status;
        }

        # ── WebSocket（实时通知）──────────────────────
        location /ws/ {
            proxy_pass http://notification_service;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_read_timeout 3600s;
        }

        # ── Nginx 监控 ────────────────────────────────
        location /nginx_status {
            stub_status;
            access_log off;
            allow 127.0.0.1;
            allow 10.0.0.0/8;
            deny all;
        }

        # 禁止访问隐藏文件
        location ~ /\. {
            deny all;
        }

        error_page 404 /404.html;
        error_page 502 503 504 /50x.html;
        location ~ ^/(404|50x)\.html$ {
            root /var/www/errors;
            internal;
        }

        access_log /var/log/nginx/app.access.log main;
        error_log  /var/log/nginx/app.error.log warn;
    }

    # 代理缓存区
    proxy_cache_path /var/cache/nginx/api
        levels=1:2
        keys_zone=api_cache:10m
        max_size=500m
        inactive=30m
        use_temp_path=off;
}
```

### 案例 2：Nginx 日志分析与监控

```bash
# 统计访问量最多的 URL Top 10
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# 统计状态码分布
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# 统计慢请求（超过2秒）
awk '$NF > 2 {print $0}' /var/log/nginx/access.log | wc -l

# 统计每分钟请求量
awk '{print $4}' /var/log/nginx/access.log | cut -d: -f1-3 | uniq -c

# 查看实时访问量
tail -f /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c
```

### 案例 3：Docker Compose + Nginx 反向代理

```yaml
# docker-compose.yml
version: '3.8'

services:
  nginx:
    image: nginx:1.24-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./frontend/dist:/var/www/html:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - nginx_logs:/var/log/nginx
    depends_on:
      - app1
      - app2
    restart: unless-stopped

  app1:
    image: my-app:latest
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/mydb
    restart: unless-stopped

  app2:
    image: my-app:latest
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/mydb
    restart: unless-stopped

  mysql:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=secret
      - MYSQL_DATABASE=mydb
    volumes:
      - mysql_data:/var/lib/mysql
    restart: unless-stopped

volumes:
  nginx_logs:
  mysql_data:
```

```nginx
# nginx/conf.d/app.conf
upstream app_backend {
    server app1:8080;
    server app2:8080;
    keepalive 20;
}

server {
    listen 80;
    server_name _;

    location / {
        root /var/www/html;
        index index.html;
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://app_backend/api/;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

---

## Part 14: 高频面试题 {#part-14}

### Q1: Nginx 为什么高并发性能好？

**答**：
1. **事件驱动架构**：采用 epoll（Linux）多路复用，一个线程可处理数千连接，而不是每连接一个线程
2. **异步非阻塞 I/O**：不会因为某个连接的 I/O 操作而阻塞其他连接
3. **Master-Worker 模型**：多个 Worker 进程充分利用多核 CPU
4. **内存高效**：处理连接不需要大量内存，10000 连接约 2.5MB
5. **零拷贝**：`sendfile` 系统调用直接在内核空间传输文件，减少数据拷贝

### Q2: Nginx 的 Master 进程和 Worker 进程分别做什么？

**答**：
- **Master 进程**：读取配置文件、验证配置、管理 Worker 进程（创建/杀死/重启）、监听端口（需要 root 权限）、处理信号（reload/stop 等）
- **Worker 进程**：实际处理客户端请求、执行反向代理/负载均衡/静态文件服务等

### Q3: Nginx 的 location 匹配优先级？

**答**（从高到低）：
1. `=` 精确匹配
2. `^~` 前缀匹配（匹配后不再检查正则）
3. `~` 大小写敏感正则
4. `~*` 大小写不敏感正则
5. 普通前缀匹配（最长匹配优先）

### Q4: proxy_pass 带斜杠和不带斜杠的区别？

```nginx
# 不带斜杠：URI 直接拼接到 proxy_pass 后
location /api/ {
    proxy_pass http://backend:8080;
    # /api/users → http://backend:8080/api/users
}

# 带斜杠：替换 location 匹配的前缀
location /api/ {
    proxy_pass http://backend:8080/;
    # /api/users → http://backend:8080/users
}
```

### Q5: Nginx 负载均衡有哪些策略？

| 策略 | 配置关键字 | 适用场景 |
|------|-----------|---------|
| 轮询 | 默认 | 后端无状态服务 |
| 加权轮询 | `weight` | 后端性能不均衡 |
| IP Hash | `ip_hash` | 需要会话保持 |
| 最少连接 | `least_conn` | 处理时间差异大 |
| 自定义 Hash | `hash` | 按 URL/Header 路由 |

### Q6: 如何实现 Nginx 的热部署？

```bash
# 步骤1：修改配置后验证
nginx -t

# 步骤2：发送 HUP 信号（优雅重载，不中断现有连接）
nginx -s reload
# 或
kill -HUP $(cat /run/nginx.pid)

# 升级 Nginx 版本的热部署：
# 1. 启动新版本 Nginx（使用同一配置）
# 2. 向旧 Master 发送 SIGUSR2（旧 Master 派生新 Master）
# 3. 向旧 Master 发送 SIGWINCH（旧 Worker 进入退出流程）
# 4. 验证新版本正常运行后，向旧 Master 发送 SIGQUIT
```

### Q7: 如何用 Nginx 实现 HTTP 跳转到 HTTPS？

```nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

### Q8: limit_req 中的 burst 和 nodelay 参数有什么区别？

**答**：
- `burst=N`：允许突发 N 个请求排队等待处理，超出 N+rate 的请求返回 503
- `nodelay`：队列中的请求立即处理（不按速率延迟），超出队列的返回 503
- 不加 `nodelay`：队列中的请求按照速率均匀处理（可能等待较长时间）

```nginx
limit_req zone=api_rate burst=20 nodelay;
# 含义：正常允许10r/s，突发时可有20个请求立即被处理
# 超过 20 个突发的请求立即返回 503
```

### Q9: Nginx 如何处理 WebSocket？

WebSocket 需要从 HTTP 升级到 ws 协议，关键配置：

```nginx
location /ws/ {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;    # 升级协议
    proxy_set_header Connection "upgrade";     # 告知 keep-alive
    proxy_read_timeout 3600s;                  # 长连接超时
}
```

### Q10: worker_processes 设置多少合适？

**答**：通常设置为 `auto`（自动检测 CPU 核数），或显式设置为 CPU 核数。

```nginx
worker_processes auto;     # 推荐
worker_processes 4;        # 4核CPU
```

原因：一个 Worker 是单线程的，绑定一个 CPU 核可减少上下文切换。设置超过 CPU 核数没有意义，可能因为上下文切换反而降低性能。

### Q11: Nginx 如何实现灰度发布？

```nginx
# 按 Cookie 灰度
map $cookie_canary $upstream_pool {
    "true"  canary;
    default production;
}

upstream production {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}

upstream canary {
    server 10.0.0.10:8080;  # 新版本服务
}

server {
    location / {
        proxy_pass http://$upstream_pool;
    }
}

# 按流量比例灰度（结合 split_clients）
split_clients $remote_addr $variant {
    10%  "canary";     # 10% 流量到新版本
    *    "production";
}
```

### Q12: 如何防止 Nginx 被 DDoS 攻击？

1. **限制请求速率**：`limit_req_zone` 限制每 IP 请求频率
2. **限制并发连接**：`limit_conn_zone` 限制每 IP 并发数
3. **过滤异常请求**：拒绝无 User-Agent、特殊请求方法等
4. **连接超时**：设置合理的 `client_header_timeout`、`client_body_timeout`
5. **隐藏版本号**：`server_tokens off` 减少信息泄露
6. **结合上游防护**：配合云防护/WAF（如阿里云 DDoS 防护）

### Q13: Nginx 的 sendfile 指令有什么作用？

**答**：`sendfile on` 使 Nginx 使用 Linux 的 `sendfile()` 系统调用来传输静态文件，实现**零拷贝**。

```
未启用 sendfile（4次数据拷贝）：
磁盘 → 内核缓冲区 → 用户缓冲区 → Socket 缓冲区 → 网卡

启用 sendfile（2次数据拷贝，0次用户态/内核态切换）：
磁盘 → 内核缓冲区 ─────────────► Socket 缓冲区 → 网卡
```

### Q14: upstream 中 max_fails 和 fail_timeout 如何配合使用？

```nginx
upstream backend {
    server 10.0.0.1:8080 max_fails=3 fail_timeout=30s;
}
```

**工作原理**：
- 在 `fail_timeout=30s` 时间窗口内
- 如果连接该服务器失败 `max_fails=3` 次
- 则在接下来 `fail_timeout=30s` 内将该服务器标记为不可用
- 30秒后 Nginx 重新尝试将请求发送到该服务器

### Q15: Nginx 如何获取客户端真实 IP？

```nginx
# 问题：经过多层代理后 $remote_addr 是最后一个代理的 IP

# 方案1：使用 X-Real-IP 头（只经过一层代理）
proxy_set_header X-Real-IP $remote_addr;

# 后端获取：request.getHeader("X-Real-IP")

# 方案2：使用 X-Forwarded-For（多层代理）
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
# $proxy_add_x_forwarded_for 会在现有头上追加 $remote_addr

# 后端获取真实 IP：取 X-Forwarded-For 的第一个值

# 方案3：使用 realip 模块（最可靠）
set_real_ip_from 10.0.0.0/8;      # 信任的代理 IP 段
real_ip_header X-Forwarded-For;   # 从哪个头获取真实 IP
real_ip_recursive on;             # 递归解析
# 配置后 $remote_addr 就是真实客户端 IP
```

---

## 附录 A：Nginx 常用命令速查

```bash
# 启停管理
systemctl start|stop|restart|reload nginx
nginx -s stop|quit|reload|reopen

# 配置检查
nginx -t              # 检查语法
nginx -T              # 检查并显示配置
nginx -v              # 版本
nginx -V              # 版本+编译参数

# 进程管理
ps aux | grep nginx   # 查看进程
cat /run/nginx.pid    # Master PID

# 日志查看
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
```

## 附录 B：Nginx 内置变量速查

```
变量                    说明
─────────────────────────────────────────────────
$remote_addr           客户端 IP
$remote_port           客户端端口
$server_name           server_name 值
$server_port           监听端口
$request_method        HTTP 方法（GET/POST）
$request_uri           完整请求 URI（含参数）
$uri                   规范化 URI（不含参数）
$args                  查询字符串
$arg_NAME              具体参数值
$http_HOST             请求头 HOST
$http_user_agent       User-Agent
$http_referer          Referer
$http_x_forwarded_for  X-Forwarded-For
$scheme                http 或 https
$status                响应状态码
$body_bytes_sent       发送响应体字节数
$request_time          请求处理时间
$upstream_addr         上游服务器地址
$upstream_status       上游响应状态码
$upstream_response_time 上游响应时间
$upstream_cache_status  缓存状态（HIT/MISS/BYPASS）
$time_local            本地时间
$time_iso8601          ISO 8601 格式时间
```

---

*Nginx 详解 — 完*


---

## 附录 C：完整 nginx.conf 生产参考配置

```nginx
# ======================================================
# Nginx 生产环境完整配置参考
# 适用于：Spring Boot 微服务 + Vue/React 前端
# ======================================================

user nginx;
worker_processes auto;
worker_cpu_affinity auto;
worker_rlimit_nofile 65535;

error_log /var/log/nginx/error.log warn;
pid /run/nginx.pid;

events {
    use epoll;
    worker_connections 10240;
    multi_accept on;
    accept_mutex off;
}

http {
    # ── 基础 ────────────────────────────────────────────
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    charset       utf-8;
    server_tokens off;

    # ── 日志 ────────────────────────────────────────────
    log_format main
        '$remote_addr - $remote_user [$time_local] "$request" '
        '$status $body_bytes_sent "$http_referer" '
        '"$http_user_agent" "$http_x_forwarded_for" '
        'rt=$request_time urt=$upstream_response_time';

    access_log /var/log/nginx/access.log main buffer=16k flush=5s;

    # ── 性能 ────────────────────────────────────────────
    sendfile           on;
    tcp_nopush         on;
    tcp_nodelay        on;
    keepalive_timeout  65;
    keepalive_requests 1000;

    # ── 请求限制 ────────────────────────────────────────
    client_max_body_size      100m;
    client_body_buffer_size   128k;
    client_header_buffer_size 2k;
    large_client_header_buffers 4 8k;
    client_header_timeout     15s;
    client_body_timeout       15s;
    send_timeout              30s;

    # ── 文件缓存 ────────────────────────────────────────
    open_file_cache          max=10000 inactive=30s;
    open_file_cache_valid    60s;
    open_file_cache_min_uses 2;
    open_file_cache_errors   on;

    # ── Gzip 压缩 ────────────────────────────────────────
    gzip              on;
    gzip_vary         on;
    gzip_min_length   1k;
    gzip_comp_level   6;
    gzip_buffers      16 8k;
    gzip_http_version 1.1;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        text/x-component
        application/json
        application/javascript
        application/x-javascript
        application/xml
        application/xhtml+xml
        application/rss+xml
        application/atom+xml
        application/vnd.ms-fontobject
        application/x-font-ttf
        application/x-web-app-manifest+json
        image/svg+xml
        font/truetype
        font/opentype;

    # ── 限流区域 ─────────────────────────────────────────
    limit_req_zone $binary_remote_addr zone=perip:10m    rate=100r/s;
    limit_req_zone $binary_remote_addr zone=login:10m    rate=5r/m;
    limit_req_zone $server_name        zone=perserver:10m rate=1000r/s;
    limit_conn_zone $binary_remote_addr zone=conn:10m;
    limit_req_status  429;
    limit_conn_status 429;

    # ── 代理缓存 ─────────────────────────────────────────
    proxy_cache_path /var/cache/nginx/static
        levels=1:2
        keys_zone=static_cache:10m
        max_size=1g
        inactive=1d
        use_temp_path=off;

    proxy_cache_path /var/cache/nginx/api
        levels=1:2
        keys_zone=api_cache:10m
        max_size=500m
        inactive=10m
        use_temp_path=off;

    # ── 上游服务器 ────────────────────────────────────────
    upstream backend_api {
        least_conn;
        server 10.0.0.1:8080 weight=3 max_fails=2 fail_timeout=10s;
        server 10.0.0.2:8080 weight=3 max_fails=2 fail_timeout=10s;
        server 10.0.0.3:8080 weight=1 max_fails=2 fail_timeout=10s;
        server 10.0.0.4:8080 backup;
        keepalive 50;
        keepalive_timeout 60s;
        keepalive_requests 1000;
    }

    # ── HTTP 跳转 HTTPS ───────────────────────────────────
    server {
        listen 80 default_server;
        listen [::]:80 default_server;
        server_name _;

        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        location / {
            return 301 https://$host$request_uri;
        }
    }

    # ── 主 HTTPS 站点 ─────────────────────────────────────
    server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name www.example.com example.com;

        # SSL 证书
        ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

        # SSL 优化
        ssl_protocols       TLSv1.2 TLSv1.3;
        ssl_ciphers         ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
        ssl_prefer_server_ciphers off;
        ssl_session_cache   shared:SSL:10m;
        ssl_session_timeout 1d;
        ssl_session_tickets off;
        ssl_stapling        on;
        ssl_stapling_verify on;
        resolver 8.8.8.8 8.8.4.4 valid=300s;

        # 安全头
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
        add_header X-Frame-Options            SAMEORIGIN always;
        add_header X-Content-Type-Options     nosniff always;
        add_header X-XSS-Protection           "1; mode=block" always;
        add_header Referrer-Policy            "strict-origin-when-cross-origin" always;
        add_header Permissions-Policy         "camera=(), microphone=(), geolocation=()" always;

        # 全局限流
        limit_conn conn 100;
        limit_req zone=perserver burst=200 nodelay;

        # 前端静态资源
        root /var/www/dist;

        location / {
            try_files $uri $uri/ /index.html;

            # 缓存控制
            location ~* \.(js|css|woff2?|ttf|eot|svg|ico)$ {
                expires 1y;
                add_header Cache-Control "public, immutable";
                access_log off;
            }
            location ~* \.(png|jpg|jpeg|gif|webp)$ {
                expires 30d;
                add_header Cache-Control "public";
                access_log off;
            }
            location = /index.html {
                expires -1;
                add_header Cache-Control "no-cache, no-store, must-revalidate";
            }
        }

        # API 接口
        location /api/ {
            limit_req zone=perip burst=50 nodelay;

            proxy_pass http://backend_api/api/;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host              $http_host;
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_connect_timeout 5s;
            proxy_read_timeout   30s;
            proxy_send_timeout   30s;

            proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;
            proxy_next_upstream_tries 2;
            proxy_next_upstream_timeout 10s;

            proxy_buffer_size    8k;
            proxy_buffers        8 8k;
        }

        # 登录接口严格限流
        location = /api/auth/login {
            limit_req zone=login burst=5 nodelay;

            proxy_pass http://backend_api/api/auth/login;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host            $http_host;
            proxy_set_header X-Real-IP       $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        # WebSocket
        location /ws/ {
            proxy_pass http://backend_api/ws/;
            proxy_http_version 1.1;
            proxy_set_header Upgrade    $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host       $http_host;
            proxy_set_header X-Real-IP  $remote_addr;
            proxy_read_timeout 3600s;
            proxy_send_timeout 3600s;
        }

        # 上传接口
        location /api/upload/ {
            client_max_body_size 200m;
            client_body_timeout  120s;

            proxy_pass http://backend_api/api/upload/;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $http_host;
            proxy_read_timeout 120s;
            proxy_request_buffering off;  # 大文件禁用请求缓冲
        }

        # 健康检查（不记录日志）
        location = /health {
            access_log off;
            proxy_pass http://backend_api/health;
        }

        # Nginx 状态
        location = /nginx_status {
            stub_status;
            access_log off;
            allow 127.0.0.1;
            allow 10.0.0.0/8;
            deny all;
        }

        # 禁止访问敏感文件
        location ~ /\. {
            deny all;
            return 404;
        }
        location ~* \.(bak|conf|log|sql|sh|env)$ {
            deny all;
            return 404;
        }

        # 自定义错误页面
        error_page 400 /errors/400.html;
        error_page 403 /errors/403.html;
        error_page 404 /errors/404.html;
        error_page 429 /errors/429.html;
        error_page 500 502 503 504 /errors/50x.html;
        location ^~ /errors/ {
            root /var/www;
            internal;
        }

        access_log /var/log/nginx/www.access.log main buffer=16k;
        error_log  /var/log/nginx/www.error.log  warn;
    }
}
```

---

## 附录 D：Nginx 故障排查指南

### D.1 常见错误及解决方法

**502 Bad Gateway**：
```bash
# 原因：后端服务宕机或无法连接
# 排查步骤：
# 1. 检查后端服务是否启动
systemctl status app-service

# 2. 检查 Nginx 错误日志
tail -100 /var/log/nginx/error.log

# 3. 直接测试后端连通性
curl -v http://127.0.0.1:8080/health

# 4. 检查防火墙
firewall-cmd --list-all
```

**504 Gateway Timeout**：
```bash
# 原因：后端响应超时
# 解决：增加 proxy_read_timeout
location /api/slow-endpoint/ {
    proxy_read_timeout 120s;  # 根据实际情况调整
    proxy_pass http://backend;
}
```

**413 Request Entity Too Large**：
```nginx
# 解决：增大 client_max_body_size
server {
    client_max_body_size 100m;  # 允许最大 100MB
}
```

**403 Forbidden**：
```bash
# 常见原因1：文件权限问题
ls -la /var/www/html
chmod -R 755 /var/www/html
chown -R nginx:nginx /var/www/html

# 常见原因2：SELinux 阻止
# 临时关闭 SELinux 测试
setenforce 0
# 永久配置
setsebool -P httpd_can_network_connect 1
```

### D.2 性能分析

```bash
# 查看 Nginx 活跃连接数
curl http://127.0.0.1/nginx_status

# 查看 Worker 进程 CPU/内存
ps aux --sort=-%cpu | grep nginx

# 实时监控请求
tail -f /var/log/nginx/access.log | awk '{print $9}' | sort | uniq -c

# 统计响应时间慢的请求
awk '$NF > 1' /var/log/nginx/access.log | wc -l

# 查看当前 TCP 连接状态
ss -s
netstat -an | awk '/tcp/ {print $6}' | sort | uniq -c
```

### D.3 配置验证清单

```bash
# 1. 语法检查
nginx -t

# 2. 检查监听端口
ss -tlnp | grep nginx

# 3. 检查证书有效期
openssl x509 -in /etc/letsencrypt/live/example.com/cert.pem -noout -dates

# 4. 测试 HTTP/HTTPS 重定向
curl -I http://example.com
curl -I https://example.com

# 5. 测试反向代理
curl -v https://example.com/api/health

# 6. 测试限流
for i in $(seq 1 20); do curl -s -o /dev/null -w "%{http_code}\n" http://api.example.com/api/test; done

# 7. 检查 SSL 评级
# 访问 https://www.ssllabs.com/ssltest/ 检查 SSL 配置评级
```

---

## 附录 E：Nginx 与 OpenResty

OpenResty 是基于 Nginx + Lua 的高性能 Web 平台，可以用 Lua 脚本扩展 Nginx：

```nginx
# OpenResty 示例：动态限流（基于 Redis）
location /api/ {
    access_by_lua_block {
        local redis = require "resty.redis"
        local red = redis:new()
        red:set_timeout(1000)

        local ok, err = red:connect("127.0.0.1", 6379)
        if not ok then
            ngx.exit(500)
        end

        local ip = ngx.var.remote_addr
        local key = "rate:" .. ip
        local count = red:incr(key)

        if tonumber(count) == 1 then
            red:expire(key, 1)  -- 1秒过期
        end

        if tonumber(count) > 100 then
            ngx.status = 429
            ngx.header["Content-Type"] = "application/json"
            ngx.say('{"error":"Too Many Requests"}')
            ngx.exit(429)
        end
    }

    proxy_pass http://backend;
}
```

---

*Nginx 详解附录完*


---

## 附录 F：Nginx 模块说明

### F.1 核心常用模块

| 模块名 | 功能 | 常用指令 |
|--------|------|---------|
| ngx_http_core_module | 核心 HTTP 功能 | `server`, `location`, `root`, `index` |
| ngx_http_proxy_module | 反向代理 | `proxy_pass`, `proxy_set_header` |
| ngx_http_upstream_module | 负载均衡 | `upstream`, `server`, `keepalive` |
| ngx_http_ssl_module | HTTPS | `ssl_certificate`, `ssl_protocols` |
| ngx_http_gzip_module | Gzip 压缩 | `gzip`, `gzip_comp_level` |
| ngx_http_limit_req_module | 请求限流 | `limit_req_zone`, `limit_req` |
| ngx_http_limit_conn_module | 连接限流 | `limit_conn_zone`, `limit_conn` |
| ngx_http_rewrite_module | URL 重写 | `rewrite`, `return`, `if` |
| ngx_http_headers_module | 响应头管理 | `add_header`, `expires` |
| ngx_http_log_module | 访问日志 | `access_log`, `log_format` |
| ngx_http_auth_basic_module | HTTP Basic 认证 | `auth_basic`, `auth_basic_user_file` |
| ngx_http_stub_status_module | 状态统计 | `stub_status` |
| ngx_http_v2_module | HTTP/2 支持 | `listen ... http2` |
| ngx_stream_module | TCP/UDP 代理 | `stream`, `proxy_pass` |

### F.2 常用第三方模块

| 模块 | 功能 |
|------|------|
| nginx-module-vts | 流量统计（可配合 Grafana） |
| nginx_upstream_check_module | 主动健康检查 |
| ngx_http_lua_module | Lua 脚本扩展（OpenResty） |
| nginx-http-concat | 合并 CSS/JS 请求 |
| ngx_http_image_filter_module | 图片动态裁剪缩放 |
| ModSecurity | WAF（Web 应用防火墙） |

---

## 附录 G：stream 块 TCP/UDP 代理

```nginx
# stream 块用于 TCP/UDP 四层代理（MySQL、Redis 等）
stream {
    log_format basic '$remote_addr [$time_local] '
                     '$protocol $status $bytes_sent $bytes_received '
                     '$session_time';

    access_log /var/log/nginx/stream.access.log basic;
    error_log  /var/log/nginx/stream.error.log;

    # MySQL 负载均衡
    upstream mysql_cluster {
        server 10.0.0.10:3306;
        server 10.0.0.11:3306;
    }

    server {
        listen 3306;
        proxy_pass mysql_cluster;
        proxy_connect_timeout 5s;
        proxy_timeout        60s;
    }

    # Redis 代理
    upstream redis_cluster {
        server 10.0.0.20:6379;
        server 10.0.0.21:6379;
    }

    server {
        listen 6379;
        proxy_pass redis_cluster;
        proxy_timeout 60s;
    }

    # SSL 终止（TLS Passthrough）
    server {
        listen 443;
        proxy_pass backend_ssl;
        ssl_preread on;  # 读取 SNI 信息进行路由

        proxy_protocol on;  # 传递客户端真实 IP
    }
}
```

---

## 附录 H：map 指令高级用法

```nginx
http {
    # map 用于按条件设置变量值

    # 1. 根据 User-Agent 判断设备类型
    map $http_user_agent $device_type {
        default         "desktop";
        "~*Mobile"      "mobile";
        "~*Tablet"      "tablet";
    }

    server {
        location / {
            # 根据设备类型重定向
            if ($device_type = "mobile") {
                return 302 https://m.example.com$request_uri;
            }
        }
    }

    # 2. 维护模式开关
    map $uri $maintenance {
        default          0;
        "~^/api/"        0;  # API 不维护
        "~^/"            1;  # 前端维护中
    }

    server {
        if ($maintenance = 1) {
            return 503;
        }
        error_page 503 /maintenance.html;
    }

    # 3. CORS 白名单
    map $http_origin $allow_origin {
        default             "";
        "https://app.example.com"   $http_origin;
        "https://admin.example.com" $http_origin;
        "http://localhost:3000"     $http_origin;
    }

    server {
        add_header Access-Control-Allow-Origin $allow_origin always;
    }

    # 4. 根据请求参数路由到不同上游
    map $arg_version $upstream_server {
        default     "backend_v1";
        "v2"        "backend_v2";
        "canary"    "backend_canary";
    }

    server {
        location /api/ {
            proxy_pass http://$upstream_server;
        }
    }
}
```

---

## 附录 I：Nginx 安全加固清单

```nginx
# 完整的 Nginx 安全加固配置

server {
    # 1. 隐藏版本号
    server_tokens off;

    # 2. 限制 HTTP 方法
    if ($request_method !~ ^(GET|HEAD|POST|PUT|DELETE|PATCH|OPTIONS)$) {
        return 405;
    }

    # 3. 防止点击劫持
    add_header X-Frame-Options "SAMEORIGIN" always;

    # 4. 防止 MIME 类型嗅探
    add_header X-Content-Type-Options "nosniff" always;

    # 5. XSS 保护（现代浏览器已内置，但仍推荐）
    add_header X-XSS-Protection "1; mode=block" always;

    # 6. 控制 Referer 信息泄露
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # 7. HSTS（仅 HTTPS）
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # 8. 内容安全策略（CSP）
    add_header Content-Security-Policy
        "default-src 'self'; "
        "script-src 'self' 'unsafe-inline' https://cdn.example.com; "
        "style-src 'self' 'unsafe-inline'; "
        "img-src 'self' data: https:; "
        "connect-src 'self' https://api.example.com; "
        "font-src 'self'; "
        "frame-src 'none'; "
        "object-src 'none'" always;

    # 9. 禁用不必要的 HTTP 方法
    dav_methods off;

    # 10. 限制请求体大小
    client_max_body_size 10m;

    # 11. 限制请求头大小
    client_header_buffer_size 1k;
    large_client_header_buffers 4 8k;

    # 12. 连接和请求超时
    client_header_timeout 15s;
    client_body_timeout   15s;
    send_timeout          30s;
    keepalive_timeout     65s;

    # 13. 禁止访问隐藏文件和目录
    location ~ /\. {
        deny all;
        return 404;
        access_log off;
        log_not_found off;
    }

    # 14. 禁止访问备份和配置文件
    location ~* \.(bak|sql|tar|gz|zip|rar|log|ini|conf|env|sh|py|rb)$ {
        deny all;
        return 404;
    }

    # 15. 防止恶意 User-Agent
    if ($http_user_agent ~* (nikto|sqlmap|nessus|masscan|zgrab|burpsuite)) {
        return 403;
    }

    # 16. 防止空 User-Agent（爬虫）
    if ($http_user_agent = "") {
        return 403;
    }
}
```

---

## 附录 J：日志轮转配置

```bash
# /etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily               # 每天轮转
    missingok           # 文件不存在时不报错
    rotate 30           # 保留 30 份
    compress            # 压缩旧日志
    delaycompress       # 延迟压缩（当天的不压缩）
    notifempty          # 空文件不轮转
    create 640 nginx adm  # 新建日志文件权限
    sharedscripts
    postrotate
        # 轮转后通知 Nginx 重新打开日志文件
        if [ -f /run/nginx.pid ]; then
            kill -USR1 `cat /run/nginx.pid`
        fi
    endscript
}
```

---

*全文完*


