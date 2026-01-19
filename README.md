# Lab Notes — 科研实验记录系统（Web + API）

Lab Notes 是一个面向科研记录与数据分析的实验记录系统：
- 前端提供区块化工作区、原始数据表、统计分析、图表可视化、扩展模块与设置中心。
- 后端提供数据集管理、行数据、统计分析、图表推荐与图表数据准备能力。
- 前端默认使用后端 API，同时保留本地模式作为 fallback。

## 功能概览

- 区块化工作区：原始数据表、统计分析、图表、附件等可组合拖拽
- 统计分析与图表：基于 tidy data 自动计算并返回可渲染数据
- 用户与数据隔离：数据与用户绑定，支持本地 / 后端模式切换
- 设置中心：外观、工作区、帮助、数据与隐私
- 扩展模块：按需启用，动态加入工作区

## 技术栈

- 前端：React + Vite + Chart.js + react-beautiful-dnd
- 后端：FastAPI + SQLAlchemy 2.0 + Pydantic v2 + SQLite
- 部署：Vercel（web）/ Render（server）/ Docker + Nginx（可选）

## 项目结构

```
Lab Notes/
├── deploy/              # Nginx 配置（可选）
├── docker-compose.yml   # Docker 部署编排（可选）
├── web/                 # React + Vite 前端
├── server/              # FastAPI 后端
└── README.md            # 总说明文档
```

```
server/app/
├── api/v1/          # v1 路由
├── core/            # 配置、日志、异常
├── db/              # ORM engine/session
├── models/          # ORM 模型
├── schemas/         # Pydantic schemas
├── services/        # 业务逻辑
├── utils/           # tidy/版本/推断工具
└── scripts/         # 初始化/示例数据脚本
```

## 本地开发

### 环境要求

- Node.js 18+（建议 18/20 LTS）
- Python 3.11+
- 建议使用 `venv` 或 `conda` 管理 Python 依赖

### 启动后端（FastAPI）

```bash
cd server
pip install -r requirements.txt
```

初始化数据库（建议首次执行）：

```bash
python -m app.scripts.init_db
```

可选：写入示例数据

```bash
python -m app.scripts.seed_demo
```

Windows 本地开发环境（推荐）：

```bash
uvicorn app.main:app --reload
```

Linux / Render / Railway 生产环境：

```bash
gunicorn -k uvicorn.workers.UvicornWorker app.main:app --bind 0.0.0.0:$PORT --workers 2
```

> 注意：gunicorn 不支持 Windows（依赖 fcntl）。在 Windows 上出现 `ModuleNotFoundError: fcntl` 属于正常现象。

默认监听地址：`http://127.0.0.1:8000`

### 启动前端（Vite）

```bash
cd web
npm install
```

配置后端地址（可选）：

```bash
# Windows PowerShell
$env:VITE_API_BASE="http://127.0.0.1:8000"

# macOS/Linux
export VITE_API_BASE=http://127.0.0.1:8000
```

启动前端：

```bash
npm run dev
```

默认访问地址：`http://127.0.0.1:5173`

## 环境变量

前端：
- `VITE_API_BASE`：后端 API base URL
- 示例文件：`web/.env.example`

后端：
- `APP_ENV`：运行环境（默认 development）
- `DATABASE_URL`：数据库连接（默认 sqlite:///./data.db）
- `LOG_LEVEL`：日志级别
- `CORS_ORIGINS`：允许的前端域名（逗号分隔）

## 一键部署到 Vercel + Render

### Render（后端）

创建 Web Service：
- Root Directory: `server`
- Build Command: `pip install -r requirements.txt`
- Start Command: `gunicorn -k uvicorn.workers.UvicornWorker app.main:app --bind 0.0.0.0:$PORT --workers 2`

建议环境变量：
- `DATABASE_URL`（默认 `sqlite:///./data.db`）
- `CORS_ORIGINS`（逗号分隔，例如 `https://your-vercel-domain.vercel.app`）

### Vercel（前端）

创建项目：
- Root Directory: `web`
- Framework Preset: `Vite`
- Build Command: `npm run build`
- Output Directory: `dist`

环境变量：
- `VITE_API_BASE` 设置为 Render 的后端地址，例如 `https://lab-notes-api.onrender.com`

## 可选部署（Docker + Nginx）

```bash
cd web
npm install
npm run build
```

```bash
docker compose up -d --build
```

默认暴露端口：Nginx `80`，API 内部 `8000`。

## API 概览（简明版）

统一响应格式：

成功：
```json
{
  "ok": true,
  "data": {},
  "meta": {
    "requestId": "uuid",
    "timestamp": "2026-01-01T12:00:00Z",
    "datasetVersion": 12
  }
}
```

失败：
```json
{
  "ok": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "错误描述",
    "detail": {}
  },
  "meta": {
    "requestId": "uuid",
    "timestamp": "2026-01-01T12:00:00Z",
    "datasetVersion": 12
  }
}
```

常用接口：
- `POST /api/v1/datasets`
- `GET  /api/v1/datasets?ownerId={userId}`
- `GET  /api/v1/datasets/{dataset_id}`
- `PUT  /api/v1/datasets/{dataset_id}`
- `DELETE /api/v1/datasets/{dataset_id}`
- `POST /api/v1/datasets/{dataset_id}/rows:batchUpsert`
- `GET  /api/v1/datasets/{dataset_id}/rows?limit=50&offset=0&filter=colA:foo`
- `POST /api/v1/datasets/{dataset_id}/analysis/run`
- `GET  /api/v1/datasets/{dataset_id}/charts/recommend`
- `POST /api/v1/datasets/{dataset_id}/charts/prepare`

## 前后端字段映射（核心）

### Record → Dataset

| 前端字段 | 后端字段 | 说明 |
| --- | --- | --- |
| record.id | dataset.id | 主键一致 |
| currentUser.id | dataset.owner_id | 归属用户 |
| record.name | dataset.name | 记录名称 |
| record（完整对象） | dataset.payload | 记录详情（tables/blocks/etc） |

### Table 结构（前端）

```json
{
  "id": "table-id",
  "name": "Raw Data",
  "columns": [
    { "id": "c1", "name": "Index", "type": "index" },
    { "id": "c2", "name": "Temperature", "type": "numeric" }
  ],
  "rows": [
    ["1", "36.8"],
    ["2", "37.1"]
  ]
}
```

后端分析/图表接口会从 `payload.tables` 自动解析为 tidy data：
- `index` 列作为样本编号，不参与统计
- `numeric` 列参与统计与图表
- `category` 列可作为分组

### ChartConfig 结构（示例）

```json
{
  "chartType": "line | bar | scatter | boxPlot | histogram | heatmap",
  "tableId": "table-id",
  "xKey": "columnKey",
  "yKeys": ["columnKey1", "columnKey2"],
  "groupBy": "columnKey",
  "yKey": "columnKey (heatmap)",
  "valueKey": "columnKey (heatmap)"
}
```

## 常见问题

- 前端打不开或请求失败：确认后端 `/health` 正常，检查 `VITE_API_BASE`。
- 模型变更后异常：删除 `server/data.db` 后重新初始化。
- Windows 无法启动 gunicorn：`ModuleNotFoundError: fcntl` 属于正常现象，请使用 `uvicorn`。

---

