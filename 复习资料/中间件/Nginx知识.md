## 1、Nginx 基础知识

### 1、什么是 Nginx

Nginx（发音为 “engine-x”）是一个**高性能的 HTTP 服务器（静态资源服务器）、反向代理服务器、负载均衡器 和 API 网关（限流 + 认证）**。

Nginx 采用 **事件驱动、异步非阻塞** 架构，能高效处理高并发连接，资源占用低，稳定性高，广泛用于 Web 服务、API 网关、CDN 边缘节点等场景。

#### （1）Nginx 的应用场景说明

**HTTP/静态资源服务器（Web Server）**

作用：直接响应客户端的 HTTP 请求，返回静态文件（如 HTML、CSS、JS、图片等）。

典型场景：
- 托管前端单页应用（SPA）
- 提供静态资源加速（CDN 边缘节点）

配置示例：
~~~ nginx
server {
    listen 80;
    server_name static.example.com;
    root /var/www/static;              # 静态文件根目录
    index index.html;                  # 默认首页

    # 启用 gzip 压缩
    gzip on;
    gzip_types text/css application/javascript;

    # 缓存控制：图片缓存 1 年
    location ～* \.(jpg|jpeg|png|gif)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
~~~

**反向代理服务器（Reverse Proxy）**

作用：接收客户端请求，转发给后端真实服务器，并将响应返回给客户端。客户端不知道后端是谁。(这里的功能类似于 Kong 网关)

目的：
- 隐藏后端架构（安全）
- 统一入口（API 网关）
- SSL 终止（HTTPS 卸载）
- 缓存、压缩、限流等中间处理

~~~ nginx
location /api/ {
    proxy_pass http://backend_app;  # 转发到后端
    proxy_set_header Host $host;
}
~~~

Nginx 虽然也可以做正向代理，但不推荐作为主要用途。Nginx 做正向代理有很多限制：
- 不支持 HTTPS CONNECT 隧道（除非用 stream 模块 hack，但功能有限）
- 不如 Squid、Privoxy 等专业正向代理工具完善

Nginx 实现方式（HTTP 正向代理）：
~~~ nginx
server {
    resolver 8.8.8.8;                   # 必须指定 DNS 服务器
    resolver_timeout 5s;
    listen 8080;

    location / {
        # $http_host = Host 请求头（如 www.google.com）
        # $request_uri = 完整 URI（如 /search?q=nginx）
        proxy_pass http://$http_host$request_uri;
        proxy_set_header Host $http_host;
    }
}
~~~

**正向代理（Forward Proxy）**：客户端主动配置使用代理（如浏览器设代理 IP:port），代理代表客户端去访问外部网络（如访问 Google），常用于：企业内网上网控制、翻墙（不合法场景）、缓存加速外网资源。

**负载均衡器**

作用：将请求分发到多个后端服务器，实现高可用与横向扩展。

Nginx 实现方式：通过 upstream 块定义后端池

~~~ nginx
upstream backend {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    # 支持权重、健康检查（Plus 版）、会话保持等
}

location / {
    proxy_pass http://backend;
}
~~~


**API 网关（限流 + 认证）**

配置示例：
~~~ nginx
# 限流：每 IP 每秒 10 个请求
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

server {
    listen 80;
    server_name gateway.example.com;

    location /api/ {
        # 应用限流
        limit_req zone=api_limit burst=20 nodelay;

        # 简单 Token 认证（实际建议用 Lua 或 JWT）
        if ($http_authorization != "Bearer my-secret-token") {
            return 401;
        }

        proxy_pass http://backend_api;
    }
}
~~~

burst=20：允许突发 20 个请求

nodelay：突发请求不延迟，直接处理

#### （2）Nginx 如何通过事件驱动、异步非阻塞架构实现高并发

核心思想：**一个线程处理成千上万个连接**

| 模型 | 连接处理方式 | 并发瓶颈 |
|------|-------------|--------|
| Apache（多进程） | 每个连接 → 一个进程/线程 | 内存/CPU 上下文切换开销大（C10K 问题） |
| Nginx（事件驱动） | 单线程 → 多路复用监听所有连接 | 极低资源消耗，轻松应对 C10K+ |

Nginx 的架构如下：

**Master-Worker 模型**

- Master 进程：读取配置、启动/管理 Worker、平滑 reload
- Worker 进程：实际处理网络 I/O（通常 = CPU 核数）

**事件驱动（Event-Driven）**

使用操作系统提供的 I/O 多路复用机制：
- Linux：epoll
- BSD/macOS：kqueue
- 其他：poll / select（性能差，仅 fallback）

Worker 进程调用 epoll_wait()，一次性获取所有就绪的 socket 事件（可读/可写）

**异步非阻塞 I/O（异步非阻塞 I/O）**

当读取客户端数据时：
- 如果 socket 无数据 → 立即返回，不等待
- 如果 socket 有数据 → 读取并处理

当发送响应时：
- 如果 TCP 缓冲区满 → 注册“可写”事件，稍后重试
- 不会阻塞当前线程

**状态机处理请求**
每个连接被抽象为一个状态机，Worker 在事件循环中快速切换不同连接的状态，实现“伪并行”。

💡 **结果：单个 Worker 可同时管理 数万连接，内存占用仅几 MB。**

#### （3）Nginx 如何“卸载”证书？—— 实际是 SSL/TLS 终止

**注意：“卸载证书”不是标准术语，通常指的是 在 Nginx 层终止 HTTPS 连接，即 SSL Termination。后端服务只需处理 HTTP，减轻其加解密负担。**

##### 1  准备证书文件

- fullchain.pem：服务器证书 + 中间 CA 证书（必须包含完整信任链）
- privkey.pem：私钥文件（权限应为 600，仅 root 可读）

##### 2  Nginx 配置示例

~~~ nginx
server {
    listen 443 ssl http2;               # 同时启用 SSL 和 HTTP/2
    server_name example.com;

    # 指定证书和私钥路径
    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;

    # 强制使用安全协议（禁用 SSLv3、TLS 1.0/1.1）
    ssl_protocols TLSv1.2 TLSv1.3;

    # 加密套件：优先使用 ECDHE（前向保密），禁用弱算法
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384;

    # 启用 OCSP Stapling（提升性能与隐私）
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 valid=300s;        # 用于 OCSP 查询的 DNS

    # 后端代理（此时是 HTTP）
    location / {
        # 3. 转发到后端（注意是 http://！）
        proxy_pass http://10.0.0.10:8080;

        # 4. 关键头信息传递
        proxy_set_header Host $host;                     # 保留原始 Host
        proxy_set_header X-Real-IP $remote_addr;         # 传递真实 IP
        proxy_set_header X-Forwarded-Proto $scheme;      # 告诉后端：原始是 https，如果后端不知道原始是 HTTPS，可能生成 http:// 的重定向或链接，导致安全降级或循环重定向。

        proxy_set_header X-Forwarded-Port $server_port;  # 原始端口（443）
    }
}
~~~

关键配置项含义说明
| 配置项 | 作用 |
|-------|------|
| `listen 443 ssl` | 监听 443 端口并启用 SSL |
| `ssl_certificate` | 公钥证书（PEM 格式） |
| `ssl_certificate_key` | 私钥（必须与证书匹配） |
| `ssl_protocols` | 限制 TLS 版本，避免老旧不安全协议 |
| `ssl_ciphers` | 控制加密算法强度，防止降级攻击 |
| `X-Forwarded-Proto: https` | 关键！ 让后端知道原始请求是 HTTPS（否则可能重定向到 HTTP） |

最终效果：
- 客户端 ↔ Nginx：HTTPS（加密）
- Nginx ↔ 后端应用：HTTP（明文）,后端无需配置证书，简化部署

#####  3 SSL机制解析

一个完整的 SSL/TLS 证书体系通常包含 三个关键文件（以 PEM 格式为例）：

| 文件 | 内容 | 作用 |
|------|------|------|
| 1. 服务器证书（Server Certificate）<br>`example.com.crt` | 公钥 + 域名信息 + 签发者信息 + 数字签名 | 证明“这个公钥属于 example.com” |
| 2. 私钥（Private Key）<br>`example.com.key` | 与公钥配对的私有密钥 | 用于解密客户端发来的预主密钥（Pre-Master Secret），必须严格保密 |
| 3. 中间证书（Intermediate CA Certificate）<br>`intermediate.crt` | 由根 CA 签发的中间 CA 证书 | 构建从服务器证书到受信任根证书的信任链（Chain of Trust） |

注意：很多用户只提供服务器证书，导致浏览器提示 “此证书不被信任”，就是因为缺少中间证书！

**证书是如何“运行”的？—— TLS 握手简要流程**

**1 客户端发起连接（ClientHello）**
- 支持的 TLS 版本、加密套件列表等

**2 Nginx 返回证书链（ServerHello + Certificate）**
- 发送：服务器证书 + 中间证书（按顺序：服务器 → 中间 → ...）
- 不发送根证书（客户端本地已内置）

**3  客户端验证证书链**
- 用 中间证书的公钥 验证服务器证书的签名
- 用 操作系统/浏览器内置的根证书 验证中间证书的签名
- 检查域名是否匹配、是否过期、是否被吊销（CRL/OCSP）

**4 生成会话密钥**
- 客户端生成 预主密钥（Pre-Master Secret）
- 用 服务器证书中的公钥 加密后发送给 Nginx
- Nginx 用 私钥 解密，双方各自生成相同的 会话密钥（Session Key）

**5 后续通信使用对称加密**
- 使用会话密钥进行 AES 等高效加密通信

**关键点：**
- 私钥只用于解密 Pre-Master Secret，不用于传输数据加密
- 证书本身是公开的，私钥必须保密
- 信任链断裂 = 浏览器警告

##### 4  Nginx 进行 SSL 终止时是如何运行的

Nginx 作为 TLS 终结点，负责：
- 接收 HTTPS 请求，完成 TLS 握手
- 解密请求内容，将 明文 HTTP 请求 转发给后端服务
- 后端服务 无需处理 TLS，只需监听 HTTP

工作流程图（以访问 https://api.example.com/user 为例）：
~~~ md
[Client] 
   │
   ├── HTTPS (encrypted) ────▶ [Nginx]
   │                             │
   │                             ├── TLS handshake（使用证书+私钥）
   │                             ├── Decrypt request → GET /user HTTP/1.1
   │                             └── Forward as HTTP ───▶ [Backend App]
   │                                                         │
   │                                                         └── Return HTTP response
   │
   ◀── HTTPS (encrypted) ◀────── Response (encrypted by Nginx)
~~~



### 2、Nginx 的安装方式

#### （1）包管理器安装（快速部署）

~~~ bash
# Ubuntu/Debian
sudo apt update && sudo apt install nginx

# CentOS/RHEL (需启用 EPEL)
sudo yum install epel-release
sudo yum install nginx

# 启动
sudo systemctl start nginx
~~~

#### （2）源码编译安装（推荐用于生产环境，可定制模块）

~~~ bash
# 下载源码
wget http://nginx.org/download/nginx-1.25.4.tar.gz
tar -zxvf nginx-1.25.4.tar.gz
cd nginx-1.25.4

# 安装依赖
sudo apt install build-essential libpcre3-dev zlib1g-dev libssl-dev

# 配置编译选项（关键步骤），决定哪些模块被编译进 Nginx 二进制文件
./configure \
  --prefix=/usr/local/nginx \          # 安装目录
  --with-http_ssl_module \             # 启用 HTTPS（需 OpenSSL）
  --with-http_v2_module \              # HTTP/2 支持
  --with-http_realip_module \          # 获取真实 IP
  --with-http_stub_status_module \     # 状态监控页面
  --with-stream \                      # 启用 TCP/UDP 代理
  --with-pcre \                        # 启用 rewrite（需 pcre-dev）
  --add-module=/path/to/third/module   # 添加第三方模块

# 编译 & 安装
make && sudo make install
~~~

**如何添加第三方模块？**

- 下载模块源码（通常是 GitHub 项目）
~~~ bash
git clone https://github.com/openresty/echo-nginx-module.git
~~~

- 在 configure 时通过 --add-module 指定路径
~~~ bash
./configure --add-module=./echo-nginx-module ...
~~~

- 重新编译 & 安装
~~~ bash
make && sudo make install
~~~

### 3、Nginx 模块体系详解

#### （1）核心模块
| 模块 | 功能 |
|------|------|
| `ngx_core_module` | 基础配置（worker_processes, error_log 等） |
| `ngx_errlog_module` | 错误日志 |
| `ngx_conf_module` | 配置文件解析 |

#### （2）事件模块（Events）

| 模块 | 功能 |
|------|------|
| `ngx_epoll_module` | Linux epoll 支持 |
| `ngx_kqueue_module` | BSD kqueue 支持 |

#### （3） HTTP 模块（最常用）

| 模块 | 功能 |
|------|------|
| `ngx_http_core_module` | `location`, `root`, `alias` 等基础指令 |
| `ngx_http_ssl_module` | HTTPS 支持 |
| `ngx_http_rewrite_module` | `rewrite`, `if`, `set`（依赖 PCRE） |
| `ngx_http_proxy_module` | `proxy_pass`, `proxy_set_header` |
| `ngx_http_upstream_module` | `upstream` 负载均衡 |
| `ngx_http_gzip_module` | 响应压缩 |
| `ngx_http_stub_status_module` | `/nginx_status` 监控页 |
| `ngx_http_realip_module` | 从 X-Forwarded-For 获取真实 IP |
| `ngx_http_fastcgi_module` | 与 PHP-FPM 通信 |

#### （4）Stream 模块

| 模块 | 功能 |
|------|------|
| `ngx_stream_core_module` | `listen`, `proxy_pass` for TCP/UDP |
| `ngx_stream_ssl_module` | TCP over TLS |

#### （4）常用第三方模块

| 模块 | 功能 | 适用场景 |
|------|------|--------|
| [ngx_http_lua_module](https://github.com/openresty/lua-nginx-module) | 在 Nginx 中嵌入 Lua 脚本 | API 网关、动态逻辑（OpenResty 核心） |
| [ngx_cache_purge](https://github.com/FRiCKLE/ngx_cache_purge) | 通过 URL 清除缓存 | CDN 缓存管理 |
| [nginx-module-vts](https://github.com/vozlt/nginx-module-vts) | 虚拟主机流量统计 | 监控各域名 QPS、流量 |
| [headers-more-nginx-module](https://github.com/openresty/headers-more-nginx-module) | 更灵活地操作请求/响应头 | 安全头设置、调试 |
| [nginx-upsync-module](https://github.com/weibocom/nginx-upsync-module) | 从 Consul/Etcd 动态同步 upstream | 服务发现 |
| [ModSecurity-nginx](https://github.com/SpiderLabs/ModSecurity-nginx) | 集成 WAF 引擎 | 防 SQL 注入、XSS |


## 2、Nginx 知识扩展

### 1、Nginx Plus vs 开源版 Nginx

| 功能 | Nginx 开源版 | Nginx Plus（商业版） |
|------|-------------|---------------------|
| 动态配置 API | ❌ | ✅（无需 reload 即可修改 upstream、server 等） |
| 高级负载均衡 | Round Robin, IP Hash | Least Time, Session Persistence |
| 主动健康检查 | ❌（仅被动） | ✅（HTTP/TCP 主动探测） |
| JWT 认证 | 需 Lua 实现 | 内置支持 |
| Dashboard 监控 | 需结合 Prometheus + Exporter | 内置实时仪表盘 |
| 商业支持 | 社区 | 官方 SLA 支持 |
| 价格 | 免费 | 按年订阅（数千美元起） |


### 2、Nginx 常见配置详解

#### （1）配置块层级关系

~~~ text
main               ← 全局配置（如 worker_processes）
└── events         ← I/O 模型配置
└── http           ← HTTP 协议相关
    └── upstream   ← 负载均衡池
    └── server     ← 虚拟主机（监听端口 + 域名）
        └── location ← URL 路径匹配
~~~

#### （2）nginx 配置案例

Nginx 基本配置结构：

~~~ nginx
events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    # 日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    server {
        listen 80;
        server_name example.com;

        location / {
            root /usr/share/nginx/html;
            index index.html;
        }

        location /api/ {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }

    upstream backend {
        server 10.0.0.1:8080;
        server 10.0.0.2:8080;
    }
}
~~~


nginx 接收 https 配置：
~~~ nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /path/to/fullchain.pem;
    ssl_certificate_key /path/to/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
}
~~~

完整 nginx.conf 示例 （**将内部标准 nginx 的配置贴进来**）
~~~ nginx

~~~


### 3、同类产品对比

Nginx vs Kong vs Envoy 对比（**后面这里要更加详尽一点**）

| 维度 | Nginx | Kong | Envoy |
|------|----------|---------|----------|
| 核心定位 | Web 服务器 / 反向代理 | API 网关（基于 Nginx + OpenResty） | L4/L7 代理（Service Mesh 数据平面） |
| 配置方式 | 静态配置文件（需 reload） | REST API + 数据库（PostgreSQL/Cassandra） | xDS 协议（动态配置，gRPC） |
| 动态能力 | 开源版弱，Plus 版强 | 强（插件热加载、动态 upstream） | 极强（毫秒级配置更新） |
| 插件生态 | 第三方模块（需编译） | 丰富插件（Auth, Rate-limiting, Logging） | Filter Chain（C++/Lua/WASM） |
| 协议支持 | HTTP/1.1, HTTP/2, TCP/UDP | HTTP/1.1, gRPC, WebSockets | HTTP/1.1, HTTP/2, gRPC, Dubbo, Redis, MySQL... |
| 学习曲线 | 低（配置直观） | 中（需理解插件+数据库） | 高（需掌握 xDS、Filter 架构） |
| 典型场景 | 静态资源、反向代理、简单网关 | 企业级 API 网关（认证/限流/监控） | Service Mesh（Istio）、大规模微服务 |

- 简单反向代理/静态服务 → Nginx
- 需要插件化 API 网关 → Kong
- 云原生/Service Mesh → Envoy



### 4、Nginx 高频面试题 & 参考答案

#### （1）Nginx 是如何实现高并发的

- **使用多进程**，master 进程管理多个 worker 进程，worker 进程管理网关 IO
- **事件驱动**，采用，I/O 多路复用机制，Linux 使用 epoll，mac 使用 kqueue，其他操作系统使用 poll 和 select，确保一个 worker 进程可以处理很多IO连接
- **异步非阻塞IO**：每次通过 epoll_wait() 获取就绪的 socket，如果socket 有数据则读取，没有数据立即返回，不等待，等待下一次 socket 数据就绪，不阻塞线程 

#### （2）Nginx 的 master-worker 架构有什么好处

- 热升级：master 不停机，逐个 reload worker
- 容错：一个 worker 崩溃不影响其他
- 权限分离：master 以 root 启动，worker 降权运行

#### （3）如何实现 Nginx 的平滑 reload

执行 nginx -s reload。master 会：

- 检查新配置语法（nginx -t）
- 启动新 worker 进程
- 旧 worker 处理完现有连接后退出,整个过程**零停机**。

#### （4）Nginx 负载均衡有哪些策略

- 轮询（Round Robin）：所有请求按时间顺序逐一分配到后端服务器
- 加权轮询（Weight）：给每台机器配置 weight，权重越高拿到的请求越多
- ip_hash —— 实现会话保持（黏粘会话），根据客户端 IP 的 hash 值转发到固定后端，保证同一个 IP 永远打到同一台机器
- hash $request_uri / $args / 自定义变量 —— 最推荐的会话保持方式，可以基于 uri、args、header 等任意内容做一致性哈希
- least_conn（最少连接数） —— 高并发场景神器，把新请求发给当前活跃连接数最少的那台后端
- least_time（Nginx Plus 商业版独有） —— 目前最智能的策略，选择平均响应时间最低 + 当前连接数最少的那台机器

**生产环境推荐：**
- 重量级业务（短连接HTTP）：least_conn + weight
- 缓存层/图片/静态资源：hash $request_uri consistent
- 需要会话保持的老系统：hash $cookie_jsessionid 或 hash $args_token
- 机器同构、简单业务：默认轮询 + weight 就够了
- WebSocket/Long polling：必须用 ip_hash 或 hash 方式保持连接

#### （5）如何防止 Nginx 被 CC 攻击

使用 **limit_req 限流**：
~~~ nginx
limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;
location / {
    limit_req zone=one burst=5 nodelay;
}
~~~

#### （6）Nginx 与 OpenResty 的区别

OpenResty = Nginx + LuaJIT + 一堆 Lua 库（如 lua-resty-core），允许用 Lua 脚本扩展 Nginx 功能（如**动态路由、认证、限流**），适合构建 高性能 API 网关，但调试复杂度高。

#### （7）Nginx Ingress 的作用

核心定位：**Kubernetes 的七层入口网关（L7 Ingress Controller）**

作用：将外部 HTTP/HTTPS 流量路由到 Kubernetes 集群内的 Service。

基于：Nginx（开源版或 Plus 版） + Kubernetes CRD（如 Ingress、VirtualServer）

功能：
- 域名路由（host: api.example.com → service A）
- 路径路由（/v1/* → service B）
- TLS 终止（自动集成 cert-manager）
- 负载均衡、限流、WAF、金丝雀发布等

工作流程：
~~~ md
Internet → LoadBalancer (云厂商) → Nginx Ingress Pod → Kubernetes Service → App Pods
~~~










**（1）看一下公司里面Nginx的版本分别对应哪一个版本开源的Nginx，有什么功能，如何编译为可安装的包，为什么我在 mops 上执行安装就可以安装 Nginx 包，理清楚里面的各种路径和配置。**

**（2）建议：如果你的配置平台支持“模块插件市场”或“Lua 脚本扩展”，将是巨大亮点！**

（3）Nginx plus如何实现动态API（无需 reload 即可修改 upstream、server 等）

（4）看一下内部标准化 Nginx 里面的配置

（5）内部如何监控 nginx 性能

（6）Nginx 配置平台中解决了什么痛点

























