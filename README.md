# DuckDB Parquet AI Analytics Agent

Hệ thống trả lời câu hỏi phân tích dữ liệu bằng Gemini, DuckDB và Parquet.
PostgreSQL chỉ là nguồn cho pipeline export; Agent và MCP không kết nối PostgreSQL.

## Kiến trúc

```text
PostgreSQL nguồn
    -> pipelines.export_*
    -> Parquet + manifest + Dataset Catalog

Client UI
    -> MCP Server
    -> AI Planner + Semantic Catalog
    -> DuckDB SQL Guard
    -> DuckDB Executor
    -> analytics.* views trên Parquet
    -> AI Answerer
```

Hệ thống hiện không sử dụng `monthly_kpi.parquet`. Mọi KPI được tính trực tiếp từ
`orders`, `sales_items`, `stores` và `products`.

## Cấu trúc chính

```text
pipelines/                 Kết nối nguồn và export PostgreSQL -> Parquet
data_lake/                 Dữ liệu Parquet cục bộ (không commit Git)
semantics/dataset_catalog.yaml
                           Đường dẫn, schema và quan hệ vật lý
semantics/demo_catalog.yaml
                           Định nghĩa nghiệp vụ và KPI
src/duckdb_runtime.py      Tạo các view analytics.* trên Parquet
src/duckdb_sql_guard.py    Chỉ cho phép SELECT an toàn
src/duckdb_executor.py     Thực thi SQL có giới hạn
src/duckdb_schema_tools.py Metadata DuckDB cho Planner
src/planner.py             Gemini lập kế hoạch SQL DuckDB
src/agent.py               Điều phối Planner -> Executor -> Answerer
src/mcp_server.py          Cổng MCP cho client
analytics_client_ui/       Giao diện người dùng
```

## Cài đặt

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
Copy-Item .env.example .env
```

Cấu hình runtime tối thiểu:

```env
GEMINI_API_KEY=
GEMINI_MODEL=gemini-3.5-flash
SEMANTIC_PATH=semantics/demo_catalog.yaml
MCP_HOST=127.0.0.1
MCP_PORT=8000
MAX_ROWS=100
```

`SOURCE_DATABASE_URL` chỉ cần khi chạy export hoặc kiểm tra PostgreSQL nguồn.

## Chuẩn bị dữ liệu

```powershell
.\.venv\Scripts\python.exe -m pipelines.export_stores
.\.venv\Scripts\python.exe -m pipelines.export_products
.\.venv\Scripts\python.exe -m pipelines.export_orders
.\.venv\Scripts\python.exe -m pipelines.export_sales_items
```

Kiểm tra Parquet:

```powershell
.\.venv\Scripts\python.exe -m scripts.test_dataset_catalog
.\.venv\Scripts\python.exe -m scripts.test_duckdb_runtime
.\.venv\Scripts\python.exe -m scripts.test_executor
```

## Chạy Agent

Chạy trực tiếp:

```powershell
.\.venv\Scripts\python.exe -m scripts.ask_direct `
  "Tổng doanh thu tháng 10/2026 là bao nhiêu?"
```

Chạy MCP Server:

```powershell
.\.venv\Scripts\python.exe -m src.mcp_server
```

Chạy backend UI ở terminal khác:

```powershell
Set-Location analytics_client_ui\backend
..\..\.venv\Scripts\python.exe -m uvicorn app.main:app `
  --host 127.0.0.1 --port 8200 --reload
```

## Kiểm thử

```powershell
.\.venv\Scripts\python.exe -m pytest -q
```

Các nguyên tắc an toàn:

- Gemini chỉ được truy vấn `analytics.<view>`.
- Không cho Gemini gọi `read_parquet`, `ATTACH`, `COPY`, `INSTALL` hoặc đường dẫn file.
- DuckDB chỉ được đọc trong `data_lake`.
- Runtime khóa cấu hình, giới hạn RAM, thread, thời gian và số dòng.
- PostgreSQL không nằm trong luồng trả lời người dùng.
