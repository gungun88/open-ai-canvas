# 二开说明（本 fork 专属）

> 本目录是 fork 二开的文档，放在根目录避开上游 `docs/*` 的 gitignore 规则。

## 仓库关系

| remote | 地址 | 用途 |
| --- | --- | --- |
| `origin` | `gungun88/open-ai-canvas` | 你的仓库，可写 |
| `upstream` | `ddcat-ai/open-ai-canvas` | 上游，只读（push 已禁用） |

## 分支模型

| 分支 | 职责 | 铁律 |
| --- | --- | --- |
| `main` | 上游镜像 | 只允许 fast-forward，**永不提交二开代码** |
| `dev` | 二开主线 | 所有二开提交在这里，前缀 `[custom]` |
| `feat/*` | 单个功能 | 从 `dev` 切，`--no-ff` 合回 |
| tag `v*-custom.N` | 部署点 | 服务器只认 tag |

## 同步上游

```bash
git fetch upstream --tags

git checkout main
git merge --ff-only upstream/main
git push origin main

git checkout dev
git rebase main
# 冲突：解 → git add -A → git rebase --continue
git push --force-with-lease origin dev
```

用 `rebase` 不用 `merge`：二开永远是一叠干净补丁叠在最新上游之上。

## 二开落点（按冲突成本从低到高）

1. **插件层（零冲突）**：`plugin-packages/<your>.yingce-plugin`
   - 加模型渠道、加支付方式，完全不用改上游代码
2. **前端新功能**：`web/src/pages/<your>/` 或 `web/src/components/<your>/`
   - 只在 `web/src/router.tsx` 做 1-2 行注册
3. **后端新模块**：`backend/internal/<your>/`
   - 在 `backend/cmd/server/main.go` 注册路由（1-2 行）
   - 需要新表就写新 migration 文件，不改旧的
4. **配置与编排**：自己的 `docker-compose.fork.yml`、`.env`
   - `.env` 已被 `.gitignore`，只放 `.env.example`
5. **最后才直接改上游文件**：改动尽量小、成块

## 冲突热点

| 文件 | 处理 |
| --- | --- |
| `web/bun.lock` | 不 merge：`git checkout --theirs` + `bun install` |
| `backend/go.sum` | 同上：`go mod tidy` |
| `VERSION` | 保持上游值，自己的版本用 tag |
| `docker-compose*.yml` | 走 `*.fork.yml` 覆盖 |
| `docs/**` | 上游的 gitignore 忽略大部分，二开文档放本目录 |
| 大范围格式化 | **禁止**对上游文件跑 `format` |

## 项目结构（来自 AGENTS.md）

| 单元 | 技术栈 | 入口 |
| --- | --- | --- |
| `web/` | Vite + React 19 + TS + React Router + Ant Design + Tailwind + Zustand + TanStack Query | `web/src/application.tsx`、`web/src/router.tsx` |
| `backend/` | Go 1.25 + Gin + GORM + SQLite/PostgreSQL + Redis | `backend/cmd/server/main.go` |
| `docs/` | Next.js + Fumadocs + MDX | `docs/content/docs/` |

**关键目录**：
- `web/src/pages/` 路由页面
- `web/src/layouts/` 路由级布局
- `web/src/components/` 跨页面复用组件
- `web/src/services/api/` 业务 API、模型渠道协议
- `backend/internal/` 内部模块
- `backend/payment-plugins/` 支付插件
- `plugin-packages/` 124 个模型/支付插件

## 部署

```bash
cd /srv/repos/open-ai-canvas
git checkout dev && git pull origin dev

# 构建（用 Dockerfile，不需要本机装 Go/Bun）
docker compose -f docker-compose.build.yml build

# 打 tag
docker tag open-ai-canvas-backend:latest open-ai-canvas-backend:custom-$(date +%Y%m%d)
docker tag open-ai-canvas-web:latest     open-ai-canvas-web:custom-$(date +%Y%m%d)

# 切换生产（改 /opt/open-ai-canvas/.env 的 image）
cd /opt/open-ai-canvas && ./dc.sh up -d
```

## 回滚

```bash
# 改回官方镜像
cd /opt/open-ai-canvas
cp /srv/backup-canvas-upgrade-*/.env.bak .env
./dc.sh up -d
```
