# Data contract — Lab Day 10

> Bắt đầu từ `contracts/data_contract.yaml` — mở rộng và đồng bộ file này.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| `data/raw/policy_export_dirty.csv` — Export CSV từ hệ thống nguồn (CRM/ERP) | Đọc CSV bằng `csv.DictReader`; map schema sang `chunk_id`, `doc_id`, `chunk_text`, `effective_date`, `exported_at` | Trùng dòng (duplicate chunk_text); `doc_id` không hợp lệ (catalog lạ); `effective_date` sai format (DD/MM/YYYY thay vì ISO); chunk_text rỗng | `raw_records` - `cleaned_records` = tổng bị quarantine; alert khi `quarantine_records > 0` |
| `data/docs/*.txt` — Tài liệu policy/SLA canonical (5 file txt) | Không ingest trực tiếp vào CSV; dùng làm ground truth để kiểm tra nội dung cleaned | Version conflict: HR policy 2025 vs 2026 (10 ngày vs 12 ngày); refund window 14 ngày (v3 cũ) vs 7 ngày (v4 hiện tại) | `expectation[refund_no_stale_14d_window]` FAIL; `expectation[hr_leave_no_stale_10d_annual]` FAIL |

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| chunk_id | string | Có | ID ổn định sau clean — SHA256 hash 16 ký tự của `doc_id\|chunk_text\|seq`; đảm bảo idempotency khi upsert Chroma |
| doc_id | string | Có | Khóa logic tài liệu nguồn; phải thuộc `ALLOWED_DOC_IDS` trong cleaning_rules.py |
| chunk_text | string | Có | Nội dung chunk sau clean; tối thiểu 8 ký tự; đã fix stale refund window nếu áp dụng |
| effective_date | date | Có | Định dạng YYYY-MM-DD (ISO 8601); được chuẩn hoá từ nhiều format đầu vào (DD/MM/YYYY) |
| exported_at | datetime | Có | Timestamp xuất từ hệ nguồn; dùng để kiểm tra freshness SLA |

---

## 3. Quy tắc quarantine vs drop

Record bị flag → ghi vào `artifacts/quarantine/quarantine_<run_id>.csv` với cột `reason` bổ sung.

| Lý do quarantine | Mô tả |
|-----------------|-------|
| `unknown_doc_id` | `doc_id` không thuộc allowlist — có thể từ catalog sai hoặc export lẫn source khác |
| `missing_effective_date` | `effective_date` rỗng sau khi parse |
| `invalid_effective_date_format` | Không thể parse sang ISO (không phải YYYY-MM-DD hay DD/MM/YYYY) |
| `stale_hr_policy_effective_date` | HR leave policy có `effective_date < 2026-01-01` — bản cũ 2025, conflict version |
| `missing_chunk_text` | `chunk_text` rỗng sau strip |
| `duplicate_chunk_text` | Nội dung trùng hoàn toàn (normalized lowercase) với record đã xử lý |

**Quy trình merge lại:** Data owner xem xét từng record trong quarantine CSV → quyết định fix tại nguồn hoặc bỏ hẳn → re-run pipeline với `--run-id` mới.

---

## 4. Phiên bản & canonical

**Source of truth cho policy refund:** `data/docs/policy_refund_v4.txt` — phiên bản v4 với cửa sổ hoàn tiền **7 ngày làm việc**. Bất kỳ chunk nào chứa "14 ngày làm việc" là từ v3 (stale) và phải bị fix hoặc quarantine.

**Source of truth cho HR leave policy:** `data/docs/hr_leave_policy.txt` — chính sách 2026 với **12 ngày phép năm** cho nhân viên dưới 3 năm. Record có `effective_date < 2026-01-01` hoặc chứa "10 ngày phép năm" là bản 2025 (stale).

**Cutoff versioning:** `hr_leave_min_effective_date = 2026-01-01` được định nghĩa trong `contracts/data_contract.yaml` (không hard-code trong cleaning code).
