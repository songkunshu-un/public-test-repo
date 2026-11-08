# 多平台公开粘贴文本采集系统

本项目持续发现 Pastebin、GitHub Gist 和 Paste.gg 上的公开文本，获取正文并保存到 PostgreSQL。Discovery 定期发现内容，Worker 独立处理正文任务；公共框架负责请求限速、任务去重、失败重试、断点恢复、多文件和内容版本管理。系统通过 Docker Compose 运行，数据库端口仅绑定 VPS 本机。

## 已实现的平台能力

| 平台 | 发现方式 | 正文处理 |
| --- | --- | --- |
| Pastebin | Scraping API 最新公开列表 | Scraping API 获取正文 |
| GitHub Gist | 官方公开列表 API，支持分页和更新时间重叠 | 官方详情 API 获取正文，支持多文件 |
| Paste.gg | 官方 API 最新公开列表，最多 100 条 | 官方详情 API 获取文本文件，支持多文件 |

只有平台公开的数据会进入采集流程。当前情报筛选和分析不在采集阶段执行。

## 数据在哪里

PostgreSQL 保存 Paste 元数据、发现来源、正文文件、内容哈希、版本和任务状态。持久化数据位于项目目录的 `pgdata/`。不要手动修改或删除这个目录；停止采集容器不会删除数据。

- `app/`：采集程序和通用框架
- `migrations/`：数据库结构迁移
- `tests/`：自动化测试
- `.env`：本机数据库密码和平台配置，不要提交或分享
- `data/`：导入工具使用的输入数据

旧版 `/home/sks/pastebin` 和 SQLite 数据单独保留，没有迁入新数据库。

## 首次部署

在项目目录创建本机配置文件：

```bash
cp .env.example .env
nano .env
```

设置 `POSTGRES_PASSWORD` 和 `DATABASE_URL` 中相同的密码。如果密码含 URL 保留字符，需要在 `DATABASE_URL` 中进行编码。

启动 PostgreSQL 并应用数据库迁移：

```bash
docker compose up -d db
docker compose --profile tools run --build --rm tools python -m paste_site.migrate
```

## 启动、查看和停止采集

每个平台的 Discovery 和 Worker 独立运行。常驻服务使用 `unless-stopped` 重启策略；启动后会持续采集，直到手动停止。执行普通的 `docker compose up` 不会启动采集服务。

### 常驻命令说明

以下命令在项目目录中执行，三个平台的命令含义相同：

- `up -d`：在后台启动 Discovery 和 Worker，开始持续采集。
- `logs -f`：持续显示实时日志；按 Ctrl+C 只退出日志查看，不会停止采集。
- `<platform>-stats`：查询发现、正文获取和任务队列统计，执行完即退出。
- `stop`：停止对应平台的 Discovery 和 Worker；PostgreSQL 及已采集数据保留。

### Pastebin


默认每 60 秒检查一次最新列表，每次最多 100 条。列表没有历史分页游标。

```bash
docker compose --profile continuous up -d pastebin-discovery pastebin-worker
docker compose --profile continuous logs -f pastebin-discovery pastebin-worker
docker compose --profile tools run --rm tools python -m paste_site.cli pastebin-stats
docker compose --profile continuous stop pastebin-discovery pastebin-worker
```

### GitHub Gist

默认每 300 秒发现一次，每页 30 条、最多 2 页，并使用 10 分钟更新时间重叠窗口。可在服务器的 `.env` 配置 `GITHUB_TOKEN` 提高 API 配额；Token 不要写入命令、日志或仓库。修改 `.env` 后需重建或重新创建服务使配置生效。

```bash
docker compose --profile gist-continuous up -d github-gist-discovery github-gist-worker
docker compose --profile gist-continuous logs -f github-gist-discovery github-gist-worker
docker compose --profile tools run --rm tools python -m paste_site.cli github-gist-stats
docker compose --profile gist-continuous stop github-gist-discovery github-gist-worker
```

### Paste.gg

默认每 120 秒检查一次最新列表，最多 100 条。API 没有可用分页或时间游标，因此只能扫描最新固定窗口。公开内容读取不需要 API Key。正文支持文本格式；其他编码格式会记录为不支持。

```bash
docker compose --profile pastegg-continuous up -d pastegg-discovery pastegg-worker
docker compose --profile pastegg-continuous logs -f pastegg-discovery pastegg-worker
docker compose --profile tools run --rm tools python -m paste_site.cli pastegg-stats
docker compose --profile pastegg-continuous stop pastegg-discovery pastegg-worker
```

## 在 DBeaver 查看正文

DBeaver 通过 SSH 隧道连接 VPS 上的 PostgreSQL。在 SQL 编辑器执行：

```sql
SELECT d.platform_key, d.platform_id, d.title, d.author,
       d.fetched_at, f.file_name, f.content
FROM documents AS d
JOIN document_versions AS v ON v.document_id = d.id
JOIN document_files AS f ON f.version_id = v.id
WHERE d.platform_key IN ('pastebin', 'github_gist', 'pastegg')
ORDER BY d.fetched_at DESC
LIMIT 50;
```

数据库名、用户名和端口读取项目 `.env` 配置。Compose 默认端口为 5432，仅监听 VPS 本机。

## 调整采集配置

复制 `.env.example` 为 `.env` 后按需修改：

- Pastebin：`PASTEBIN_POLL_INTERVAL_SECONDS`、`PASTEBIN_LIST_LIMIT`、`PASTEBIN_REQUEST_GAP_MS`
- GitHub Gist：`GITHUB_TOKEN`、`GITHUB_GIST_POLL_INTERVAL_SECONDS`、`GITHUB_GIST_PER_PAGE`、`GITHUB_GIST_MAX_PAGES`
- Paste.gg：`PASTEGG_POLL_INTERVAL_SECONDS`、`PASTEGG_LIST_LIMIT`、`PASTEGG_REQUEST_GAP_MS`
- 各平台的正文大小、文件数和重试次数：`*_MAX_CONTENT_BYTES`、`*_MAX_FILES`、`*_MAX_ATTEMPTS`

## 备份与迁移

数据库备份写入项目的 `backups/` 目录：

```bash
mkdir -p backups
sudo docker compose exec -T db sh -c 'pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB"' > backups/paste_site.sql
```

迁移到新机器时，复制项目文件、备份和本机 `.env`，在新机器启动空数据库后导入备份：

```bash
docker compose up -d db
cat backups/paste_site.sql | sudo docker compose exec -T db sh -c 'psql -v ON_ERROR_STOP=1 -U "$POSTGRES_USER" "$POSTGRES_DB"'
```

恢复备份前，目标数据库应为空。
