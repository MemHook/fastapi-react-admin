# p01 本地运行（不用 Docker，连接 MySQL）

p01 项目在 `p01` 分支上的本地运行方式：不用 Docker，数据库和 Redis 用局域网里已有的服务。
官方的 Docker 起法见 [CONTRIBUTING.md](../../CONTRIBUTING.md#本地开发)。

> 🔴 这个 fork 是公开仓库，**密码一律不写进任何提交的文件**。只写在 `apps/api/backend/.env` 里（已被 `.gitignore` 忽略）。

## 前置

| 工具 | 版本 |
|---|---|
| Node | ≥ 22.13 |
| pnpm | 由 `package.json` 的 `packageManager` 锁定，执行 `pnpm` 时会自动切换到这个版本 |
| Python | ≥ 3.10 |
| [uv](https://docs.astral.sh/uv/) | 管理后端 Python 依赖 |

## 外部服务

| 服务 | 地址 | 账号 / 库 |
|---|---|---|
| MySQL 8.4 | `192.168.2.40:3308` | 用户 `p01`，库 `p01_fastapi_react_admin`（`utf8mb4`） |
| Redis | `192.168.2.40:6379` | 业务用 db `3`，Celery broker 用 db `4`（db `2` 已被 hotgo 占用） |

## 首次安装

在仓库根目录执行：

```bash
# 1. 前端依赖
pnpm install

# 2. 后端依赖
pnpm --filter api exec uv sync

# 3. 后端配置：从模板复制，再按下表修改
cp apps/api/backend/.env.example apps/api/backend/.env
```

`apps/api/backend/.env` 需要改的项：

```ini
DATABASE_TYPE='mysql'
DATABASE_HOST='192.168.2.40'
DATABASE_PORT=3308
DATABASE_USER='p01'
DATABASE_PASSWORD='<MySQL 密码>'
DATABASE_PK_MODE='snowflake'          # 保持 snowflake，种子数据只有雪花版
DATABASE_SCHEMA='p01_fastapi_react_admin'   # 模板里没有这一项，需要手动加上；不加时默认库名是 fba

REDIS_HOST='192.168.2.40'
REDIS_PORT=6379
REDIS_PASSWORD='<Redis 密码>'
REDIS_DATABASE=3
CELERY_BROKER_REDIS_DATABASE=4

TOKEN_SECRET_KEY='<python3 -c "import secrets;print(secrets.token_urlsafe(32))" 的输出>'
```

```bash
# 4. 建表 + 写入种子数据 + 标记迁移版本
#    ⚠️ 会 DROP 并重建所有表，只在首次运行或需要重置时执行
#    过程中会要求确认（输入 y），并设置 admin 的新密码（种子数据里的 123456 不允许保留）
pnpm db:init
```

## 日常启动

```bash
pnpm dev        # 同时启动 api :8088 · web :8888 · celery worker
```

单独启动其中一个：`pnpm --filter api dev` / `pnpm --filter web dev` / `pnpm --filter worker dev`。

表结构有更新时（比如从上游合并后），运行 `pnpm db:upgrade`，**不要**再执行 `db:init`。

## 访问地址

| 用途 | 地址 |
|---|---|
| 管理后台 | <http://127.0.0.1:8888>（端口固定，被占用时直接报错） |
| 后端 API | <http://127.0.0.1:8088> |
| API 文档（Swagger） | <http://127.0.0.1:8088/docs> |

账号 `admin`，密码是执行 `pnpm db:init` 时设置的那个。

## 验证码

登录默认需要图形验证码。本地开发想跳过：登录后在「系统管理 → 参数配置」里把 `LOGIN_CAPTCHA_ENABLED` 改成 `false`，修改后立即生效。

要照常输入验证码时，可以在 Redis db `3` 里查答案：先从 `GET /api/v1/auth/captcha` 响应里取 `uuid`，再执行

```bash
redis-cli -h 192.168.2.40 -p 6379 -a '<Redis 密码>' -n 3 --raw GET "fba:login:captcha:<uuid>"
```
