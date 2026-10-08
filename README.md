# tokenrouter-lzcapp

[TokenRouter](https://github.com/TokenFlux/TokenRouter)（自托管 LLM 网关）的懒猫微服打包：**只发喵喵商店，镜像模式**（`delivery.mode: mirror`）。

## 镜像模式怎么用

- **应用镜像**：Action 只做只读校验，把 manifest 里的 `ghcr.io/tokenflux/tokenrouter:0.2.7` 改写成交付用的加速地址 `ghcr.1ms.run/tokenflux/tokenrouter:0.2.7`（多架构应用开了 `require_digest_match: true`，要求加速镜像里就是目标架构那份）。**不搬镜像到懒猫镜像源**，因此也不能上官方商店（官方商店要求 `registry.lazycat.cloud`）——正好符合"只发喵喵商店"。
- **依赖镜像**：`postgres`、`redis` 不参与版本跟踪，直接在 manifest 里写成加速地址 `docker.1ms.run/library/postgres:18.6-alpine`、`docker.1ms.run/library/redis:8.8-alpine`（注释里保留 upstream 原始 tag，升级时改一处即可）。
- **版本跟踪**：只跟应用镜像 `^[0-9]+\.[0-9]+\.[0-9]+$`（排除 `0.1.x-amd64/-arm64` 变体与 `latest`）。

## 关于 SQLite

不支持。仓库里 SQLite 只出现在测试工具 `backend/internal/testutil/sqlite/`，生产配置写死 `database.port: 5432`，官方部署文档也写明「应用依赖 PostgreSQL 和 Redis」。所以这里带了一套 Postgres + Redis。

## 其它要点

- **PG18 的 PGDATA 坑**：`postgres:18` 默认把 PGDATA 放在 `/var/lib/postgresql/18/docker`（镜像匿名卷），不显式设 `PGDATA` 的话挂载的目录接不到数据、容器重建会重新 initdb。manifest 里已按上游 compose 的写法固定 `PGDATA=/var/lib/postgresql/data`。
- **路由**：`/` → `tokenrouter:8080`，**整站公开**（`public_path: [/]`）——网关就是给外部客户端打 `/v1` 用的；鉴权靠应用自己的账号与 API Key。
- **三个服务都 `user: root`**：平台建的 `/lzcapp/var/*` 属主是 root，而三个镜像的入口脚本都以 root 起才做 chown + 降权（app → `tokenrouter`，postgres/redis → 各自的系统用户），和上游 compose 的语义一致。
- **环境**：`AUTO_SETUP=true`（无人值守初始化：迁移 + 建管理员）、`SERVER_HOST/PORT`、`DATABASE_*`、`REDIS_*`、`TZ=Asia/Shanghai`、`GOMEMLIMIT=4GiB`；`JWT_SECRET` 用 `stable_secret` 固定（否则每次启动随机生成会把所有人踢下线）。
- **管理员**：安装向导填邮箱 + 密码；密码留空则随机生成并只打印一次到应用日志。
- **已知取舍**：`TOTP_ENCRYPTION_KEY` 故意不设——它要求 32 字节（64 hex）的 AES-256 密钥，模板给不出合规值，填错会让加密功能炸掉；留空由应用自动生成，代价是**重启后已开启的双因素认证需要重新绑定**。
- **健康检查**：应用 `curl /health`；db `pg_isready`；cache `redis-cli ping`（`depends_on` 只认 healthy）。

## 商店状态

- **喵喵商店**：`cloud.lazycat.app.tokenrouter` 由 Action 发布（`stores.private`）。
- **官方商店**：不可能也不打算——镜像模式的镜像引用不是 `registry.lazycat.cloud`，官方商店不接受。
