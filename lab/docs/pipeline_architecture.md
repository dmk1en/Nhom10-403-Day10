# Kiến trúc Pipeline — Lab Day 10: Data Pipeline & Data Observability

**Nhóm:** Nhóm 10 — Lớp 403  
**Cập nhật:** 2026-04-15

---

## 1. Sơ đồ luồng

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                          ETL PIPELINE — DAY 10                                │
│                                                                               │
│  [Source: CSV/API]                                                            │
│       │                                                                       │
│       ▼  exported_at ──────────────────────────────────► [Freshness Check]   │
│  ┌─────────┐  raw_records                                     │              │
│  │  INGEST │──────────────────── run_id ──────────────────────┤              │
│  └────┬────┘                                                  │              │
│       │                                                     PASS/FAIL        │
│       ▼ raw_records                                                           │
│  ┌──────────┐  cleaned_records                                                │
│  │  CLEAN   │──────────────────────────────────────────────────────►log      │
│  │(rules.py)│  quarantine_records                                             │
│  └────┬─────┘                                                                 │
│       │            ┌──────────────────────┐                                  │
│       │            │  artifacts/quarantine │  ◄─ unknown_doc_id              │
│       │            │  /quarantine_<run>.csv│  ◄─ missing_date               │
│       │            └──────────────────────┘  ◄─ stale_hr / short_chunk      │
│       │                                                                       │
│       ▼ cleaned rows                                                          │
│  ┌────────────┐                                                               │
│  │  VALIDATE  │── expectation FAIL (halt) ──► PIPELINE_HALT (exit 2)        │
│  │(expect.py) │── expectation FAIL (warn) ──► log WARN + continue           │
│  └─────┬──────┘── all PASS ────────────────► continue                       │
│        │                                                                      │
│        ▼ cleaned_csv                                                          │
│  ┌──────────────┐                                                             │
│  │  EMBED       │  upsert chunk_id → Chroma collection: day10_kb            │
│  │  (Chroma)    │  prune stale ids  (embed_prune_removed)                   │
│  └──────┬───────┘                                                             │
│         │                                                                     │
│         ▼                                                                     │
│  ┌────────────────┐    ┌──────────────────────┐                              │
│  │ manifest_*.json│───►│ Freshness Check (SLA) │  PASS / FAIL               │
│  └────────────────┘    └──────────────────────┘                              │
│                                                                               │
│  ► artifacts/logs/run_<run_id>.log  (mọi bước đều ghi log)                  │
│  ► artifacts/manifests/manifest_<run_id>.json                                │
└───────────────────────────────────────────────────────────────────────────────┘
```

**Điểm đo freshness:** trường `latest_exported_at` từ `exported_at` tối đa trong cleaned rows — ghi vào manifest, kiểm tra ngay sau embed với SLA=24h.

**run_id:** sinh từ UTC timestamp (hoặc `--run-id` flag), xuất hiện trong log, manifest, cleaned CSV, quarantine CSV — thống nhất truy xuất toàn bộ artifact của 1 run.

---

## 2. Ranh giới trách nhiệm

| Thành phần | Input | Output | Module | Owner nhóm |
|------------|-------|--------|--------|------------|
| **Ingest** | `data/raw/policy_export_dirty.csv` (CSV bẩn từ hệ nguồn) | `List[Dict]` raw rows | `transform/cleaning_rules.py::load_raw_csv` | Ingestion Owner |
| **Transform / Clean** | Raw rows | `(cleaned_rows, quarantine_rows)` | `transform/cleaning_rules.py::clean_rows` | Cleaning & Quality Owner |
| **Quality / Validate** | Cleaned rows | `(results, should_halt)` | `quality/expectations.py::run_expectations` | Cleaning & Quality Owner |
| **Embed** | `artifacts/cleaned/cleaned_<run>.csv` | Chroma collection `day10_kb` (upsert + prune) | `etl_pipeline.py::cmd_embed_internal` | Embed Owner |
| **Monitor** | `artifacts/manifests/manifest_<run>.json` | `PASS / WARN / FAIL` + detail JSON | `monitoring/freshness_check.py` | Monitoring / Docs Owner |

---

## 3. Idempotency & Rerun

Pipeline sử dụng chiến lược **upsert theo `chunk_id`** để đảm bảo idempotency:

1. **chunk_id** được tính bằng SHA-256(16 ký tự đầu) của chuỗi `doc_id|chunk_text|seq` — cùng nội dung → cùng ID.
2. Khi rerun, `col.upsert(ids, documents, metadatas)` ghi đè vector cũ nếu ID tồn tại, thêm mới nếu chưa có.
3. **Prune stale IDs:** Trước khi upsert, pipeline truy vấn `col.get(include=[])` để lấy danh sách ID hiện tại trong collection, tính phần bù (IDs có trong collection nhưng không còn trong cleaned run này) → `col.delete(ids=drop)`. Đây là cơ chế **publish boundary**: index sau mỗi run phản ánh đúng cleaned snapshot, không có vector lạc hậu.
4. Rerun 2 lần với cùng input → collection count không phình, `embed_prune_removed` = 0 lần 2.

---

## 4. Liên hệ Day 09

Pipeline Day 10 xử lý **lớp dữ liệu** (ingest → clean → validate → embed) cho cùng corpus CS + IT Helpdesk đã dùng ở Day 09:

- **Cùng `data/docs/`**: 4 file `.txt` (policy_refund_v4, sla_p1_2026, it_helpdesk_faq, hr_leave_policy) làm ground truth; pipeline Day 10 không đọc lại file txt này mà đọc CSV export đại diện cho "bản sao DB".
- **Khác collection**: Day 10 dùng Chroma collection `day10_kb` (tách với collection Day 09 nếu có), tránh nhiễm vector cũ.
- **Tích hợp Day 09**: Agent retriever Day 09 có thể mount lại `day10_kb` thay cho collection cũ để tận dụng dữ liệu đã qua pipeline observability — tức là "agent đọc đúng version" như mô tả ở README.

---

## 5. Rủi ro đã biết

| Rủi ro | Mức độ | Biện pháp hiện tại |
|--------|--------|-------------------|
| `exported_at` lỗi thời → freshness FAIL dù pipeline chạy đúng | Trung bình | Ghi trong runbook: phân biệt SLA của "data snapshot" vs "pipeline run"; có thể đổi `FRESHNESS_SLA_HOURS` env hoặc cập nhật timestamp có chủ đích |
| Injection `--skip-validate` làm vector store bị "poison" | Cao (chỉ trong demo) | Chỉ dùng cho Sprint 3 demo; production không bao giờ dùng flag này; eval `hits_forbidden` phát hiện |
| Chroma collection không có snapshot/backup | Trung bình | Artifact `cleaned_*.csv` + manifest đủ để rebuild; production cần artifact registry |
| Cutoff HR date hard-code trong code | Thấp (đã cải thiện) | `hr_leave_min_effective_date` đọc từ `contracts/data_contract.yaml` (Merit) |
