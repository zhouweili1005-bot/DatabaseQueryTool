---
description: 一键执行 SQL 查询并将结果导出为 CSV / JSON 文件
argument-hint: <数据库名> <SQL 或查询描述> [csv|json]
---

# 导出查询结果（Execute + Export）

将「执行查询」和「导出结果」两个步骤合并为一次命令自动完成。

用户输入（`$ARGUMENTS`）格式：`<数据库名> <SQL 或查询描述> [csv|json]`
- 第一个参数：数据库连接名（如 `sales`）
- 第二个参数：要执行的 SQL，或自然语言查询描述（如 "查询所有用户"）
- 第三个参数（可选）：导出格式 `csv` / `json`，默认 `csv`

## Agent 任务分解

把「导出数据」这个大任务拆成三个子任务，按顺序执行：

### 子任务 1：获取查询结果
- 解析 `$ARGUMENTS`，校验数据库连接存在：调用 `GET /api/v1/dbs` 列出连接，确认用户给的数据库名有效。
- 如果第二个参数是自然语言描述，先调用 `POST /api/v1/dbs/{name}/query/natural` 生成 SQL。
- 执行 SQL。优先走现成的 CLI 一键脚本（复用同一套服务层逻辑），也可以调用 API：`POST /api/v1/dbs/{name}/export`（body: `{"sql": "...", "format": "csv"}`，返回文件流）。

### 子任务 2：格式化数据
- CSV：RFC 4180 转义（逗号/引号/换行），None 输出为空字符串，日期输出 ISO 8601，UTF-8 带 BOM（Excel 打开中文不乱码）。
- JSON：`ensure_ascii=False` 保留中文，2 空格缩进，datetime 转字符串。
- 大数据集（>1 万行）在导出前明确提示用户，并建议提高 `--limit` 或确认 SQL 自带的 LIMIT。

### 子任务 3：创建文件
- 快速路径（推荐）：运行后端 CLI
  ```bash
  cd backend && uv run python scripts/export_query.py \
    --db "$DB" --sql "$SQL" --format "$FORMAT" --limit 100000
  ```
  文件默认写到 `backend/exports/<db>_<时间戳>.<ext>`。
- 或调用 API 端点后把响应保存为文件（curl 场景）：
  ```bash
  curl -X POST http://localhost:8000/api/v1/dbs/$DB/export \
    -H "Content-Type: application/json" \
    -d "{\"sql\": \"$SQL\", \"format\": \"$FORMAT\"}" \
    -o "./exports/$DB.$FORMAT"
  ```
- 完成后向用户汇报：导出行数、文件路径、执行耗时。

## 约定
- 只允许 SELECT 语句（后端 sqlglot 校验会拦截其它类型）。
- SQL 没有显式 LIMIT 时，后端自动附加行数上限（默认 1000，导出可用 `--limit` 调高）。
- 导出失败时检查：数据库连接是否已添加、后端是否在 8000 端口运行、SQL 是否正确。
