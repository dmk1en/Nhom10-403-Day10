# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** ___________  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| ___ | Ingestion / Raw Owner | ___ |
| ___ | Cleaning & Quality Owner | ___ |
| ___ | Embed & Idempotency Owner | ___ |
| ___ | Monitoring / Docs Owner | ___ |

**Ngày nộp:** ___________  
**Repo:** ___________  
**Độ dài khuyến nghị:** 600–1000 từ

---

> **Nộp tại:** `reports/group_report.md`  
> **Deadline commit:** xem `SCORING.md` (code/trace sớm; report có thể muộn hơn nếu được phép).  
> Phải có **run_id**, **đường dẫn artifact**, và **bằng chứng before/after** (CSV eval hoặc screenshot).

---

## 1. Pipeline tổng quan (150–200 từ)

> Nguồn raw là gì (CSV mẫu / export thật)? Chuỗi lệnh chạy end-to-end? `run_id` lấy ở đâu trong log?

**Tóm tắt luồng:**

_________________

**Lệnh chạy một dòng (copy từ README thực tế của nhóm):**

_________________

---

## 2. Cleaning & expectation (150–200 từ)

> Baseline đã có nhiều rule (allowlist, ngày ISO, HR stale, refund, dedupe…). Nhóm thêm **≥3 rule mới** + **≥2 expectation mới**. Khai báo expectation nào **halt**.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn) | Trước (số liệu) | Sau (số liệu) | Chứng cứ (log / CSV) |
|-----------------------------------|-----------------|---------------|----------------------|
| `rule_quarantine_missing_exported_at` (Rule 7) | row 12 (sla_p1_2026, exported_at rỗng) sẽ vào cleaned | `quarantine_records` sprint1=4 → sprint2=6 (+1 từ row 12) | `quarantine_sprint2.csv`: row 12 reason=`missing_exported_at` |
| `rule_quarantine_short_chunk_below_20` (Rule 8) | row 11 "OK." (3 ký tự) sẽ vào cleaned và warn expectation E4 | `quarantine_records` tăng thêm 1; E4 không cần warn nữa | `quarantine_sprint2.csv`: row 11 reason=`short_chunk_text`, `chunk_text_len=3` |
| `rule_normalize_whitespace_in_chunk_text` (Rule 9) | row 13 "Nhân  viên  được..." với 15+ double-space | `cleaned_sprint2.csv`: row 13 → "Nhân viên được phép đăng ký nghỉ phép qua hệ thống HR Portal ít nhất 3 ngày trước." (1 space) | `cleaned_sprint2.csv` dòng cuối |
| `E7 all_doc_ids_in_allowlist` (halt) | Nếu rule 1 bỏ sót doc_id lạ → vector store nhiễm | Sprint 3 inject `--skip-validate` bỏ qua, nhưng run chuẩn: `unknown_doc_id_count=0 OK` | `run_sprint2.log`: `expectation[all_doc_ids_in_allowlist] OK (halt)` |
| `E8 max_chunk_text_length_2000` (warn) | Không giới hạn độ dài → DB export nối chunk nhầm lọt qua | Sprint 3: inject row > 2000 ký tự → expectation warn xuất hiện trong log | `run_sprint2.log`: `expectation[max_chunk_text_length_2000] OK (warn) :: long_chunks=0` |

**Rule chính (baseline + mở rộng):**

- **Baseline**: unknown_doc_id, missing/invalid effective_date, stale_hr_policy_effective_date, missing_chunk_text, duplicate_chunk_text, fix_stale_refund_14→7
- **Sprint 2 mới**: missing_exported_at (Rule 7), short_chunk_text < 20 chars (Rule 8), normalize_whitespace (Rule 9)

**Kết quả sprint2:** raw=13, cleaned=7 (+1 row 13 normalized), quarantine=6 (+2 so với sprint1)

**Ví dụ 1 lần expectation fail và cách xử lý:**

Sprint 3 inject (`--no-refund-fix --skip-validate`): `expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=1` — pipeline bị halt nếu không có `--skip-validate`. Fix: chạy lại pipeline chuẩn (không có flag inject) → expectation pass, refund window = 7 ngày.

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

> Bắt buộc: inject corruption (Sprint 3) — mô tả + dẫn `artifacts/eval/…` hoặc log.

**Kịch bản inject:**

_________________

**Kết quả định lượng (từ CSV / bảng):**

_________________

---

## 4. Freshness & monitoring (100–150 từ)

> SLA bạn chọn, ý nghĩa PASS/WARN/FAIL trên manifest mẫu.

_________________

---

## 5. Liên hệ Day 09 (50–100 từ)

> Dữ liệu sau embed có phục vụ lại multi-agent Day 09 không? Nếu có, mô tả tích hợp; nếu không, giải thích vì sao tách collection.

_________________

---

## 6. Rủi ro còn lại & việc chưa làm

- …
