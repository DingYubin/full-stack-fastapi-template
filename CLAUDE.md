# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

Full Stack FastAPI Template — FastAPI 后端 + React 前端的全栈模板，使用 Docker Compose 部署。

- **后端**: FastAPI + SQLModel (ORM) + PostgreSQL + Alembic（迁移）+ Pydantic
- **前端**: React + TypeScript + Vite + TanStack Query + TanStack Router + Tailwind CSS + shadcn/ui
- **反向代理**: Traefik（自动 HTTPS）
- **开发**: Docker Compose（本地）、GitHub Actions（CI/CD）

---

## 常用命令

### 启动开发环境

```bash
# 全部启动（带热重载）
docker compose watch

# 仅启动后端（本地前端可单独运行）
docker compose up -d backend

# 本地单独运行前端
cd frontend && bun install && bun run dev
```

### 后端开发

```bash
cd backend

# 安装依赖（使用 uv）
uv sync

# 激活虚拟环境
source .venv/bin/activate

# 运行开发服务器（热重载）
fastapi dev app/main.py

# 交互式进入运行中的容器
docker compose exec backend bash
```

### 数据库迁移（Alembic）

```bash
# 创建迁移
alembic revision --autogenerate -m "描述迁移内容"

# 执行迁移
alembic upgrade head

# 回滚
alembic downgrade -1
```

> 注意：`prestart` 服务会在启动时自动执行 `alembic upgrade head`，无需手动运行。

### 测试

```bash
# 完整测试（构建 stack 并运行）
bash ./scripts/test.sh

# 在已运行的 stack 中运行测试
docker compose exec backend bash scripts/tests-start.sh

# 带参数运行（如 -x 遇到第一个错误就停止）
docker compose exec backend bash scripts/tests-start.sh -x
```

### 代码检查

```bash
# prek hooks（提交前自动运行）
uv run prek install -f    # 安装 hooks
uv run prek run --all-files  # 手动运行全部检查

# ruff lint
uv run ruff check .

# ruff format
uv run ruff format .

# mypy 类型检查
uv run mypy .

# 前端 Biome 检查
cd frontend && bun run biome check .
```

### 生成前端 API 客户端

```bash
# 方式一：自动（需要先激活后端虚拟环境）
bash ./scripts/generate-client.sh

# 方式二：手动
# 1. 启动 stack：`docker compose up -d backend`
# 2. 下载 OpenAPI JSON 到 `frontend/openapi.json`
# 3. `cd frontend && bun run generate-client`
```

### Playwright E2E 测试

```bash
docker compose up -d --wait backend
bunx playwright test      # headless
bunx playwright test --ui  # 有 UI 模式
docker compose down -v   # 清理
```

---

## 架构

### 后端结构 `backend/app/`

```
app/
├── main.py              # FastAPI 实例，CORS 配置，路由注册
├── models.py            # SQLModel 模型（User、Item 等），所有数据库表
├── crud.py              # CRUD 通用函数（create/get/update/delete）
├── api/
│   ├── main.py          # 路由聚合（include_router）
│   └── routes/
│       ├── deps.py      # 依赖注入（get_db、get_current_user 等）
│       ├── login.py     # 登录（token）
│       ├── users.py     # 用户 CRUD
│       ├── items.py     # Item CRUD（示例）
│       ├── utils.py     # 健康检查等工具接口
│       └── private.py   # 仅 local 环境的私有接口
├── core/
│   ├── config.py        # Settings（Pydantic Settings），所有环境变量
│   ├── db.py            # 数据库引擎、会话、SQLModel.metadata.create_all
│   └── security.py      # 密码哈希、JWT token 创建/验证
├── email-templates/      # MJML 源码（src/）和编译后的 HTML（build/）
└── alembic/             # 数据库迁移文件
```

**核心开发模式**：改 `models.py` 加字段 → 用 Alembic 生成迁移 → 路由在 `app/api/routes/` 下，每个路由文件注册到 `crud.py`。

### 前端结构 `frontend/src/`

```
src/
├── client/              # OpenAPI 自动生成的 API 客户端（不要手动修改）
├── components/         # 组件（Admin、Common、Items、Sidebar、ui 等）
│   └── ui/              # shadcn/ui 组件
├── hooks/              # 自定义 hooks（useAuth、useCustomToast 等）
├── routes/             # TanStack Router 文件路由（_layout.tsx 为布局）
│   ├── _layout/
│   ├── login.tsx
│   ├── signup.tsx
│   └── ...
├── lib/                # 工具函数
└── main.tsx            # 应用入口
```

**前端 API 调用**：所有 API 调用通过 `src/client/` 下的生成客户端，无需手动 fetch。

### Docker Compose 架构

- `compose.yml` — 生产/通用配置，所有服务的核心定义
- `compose.override.yml` — 开发覆盖（卷挂载、watch、热重载命令）
- `compose.traefik.yml` — Traefik HTTPS 生产配置

---

## 关键约定

- **Python 包管理**: `uv`（不要用 pip）
- **前端包管理**: `Bun`（不要用 npm/yarn）
- **提交前**: 必须通过 `prek` 检查（自动在 `.git/hooks/pre-commit`）
- **邮箱模板**: `src/*.mjml` 用 MJML 扩展导出为 `build/*.html`，不要直接写 HTML
- **前端 CSS 类名**: 使用 Tailwind CSS，shadcn/ui 组件使用小驼峰 className
- **Secret Key**: 部署前必须在 `.env` 中修改 `SECRET_KEY`、`POSTGRES_PASSWORD`、`FIRST_SUPERUSER_PASSWORD`
- **生成客户端**: 后端 API 变更后必须重新运行 `generate-client.sh` 并提交

---

## URL 开发环境

| 服务 | URL |
|------|-----|
| 前端 | <http://localhost:5173> |
| 后端 API | <http://localhost:8000> |
| Swagger UI | <http://localhost:8000/docs> |
| ReDoc | <http://localhost:8000/redoc> |
| Adminer (DB) | <http://localhost:8080> |
| Traefik | <http://localhost:8090> |
| MailCatcher | <http://localhost:1080> |

生产部署时：API → `api.{DOMAIN}`，前端 → `dashboard.{DOMAIN}`
