# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** Nhóm 10 — Lớp 403  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Dương Mạnh Kiên | Ingestion / Raw Owner | duongkien50@gmail.com |
| Nguyễn Văn Hiếu | Cleaning & Quality Owner | nguyenhieu6732@gmail.com |
| Bùi Quang Hải | Embed & Idempotency Owner | buiquanghai2k3@gmail.com |
| Vũ Trung Lập & Tạ Vĩnh Phúc | Monitoring / Docs Owner | trunglap923@gmail.com & tavinhphuc123@gmail.com |

**Ngày nộp:** 2026-04-15  
**Repo:** github.com/dmk1en/Nhom10-403-Day10  
**Độ dài khuyến nghị:** 600–1000 từ

---

> **Nộp tại:** `reports/group_report.md`  
> **Deadline commit:** xem `SCORING.md` (code/trace sớm; report có thể muộn hơn nếu được phép).  
> Phải có **run_id**, **đường dẫn artifact**, và **bằng chứng before/after** (CSV eval hoặc screenshot).

---

## 1. Pipeline tổng quan

**Nguồn raw:** `data/raw/policy_export_dirty.csv` — file CSV mô phỏng export từ hệ thống nguồn (CRM/ERP) chứa 13 dòng với nhiều lỗi cố ý: duplicate, thiếu ngày, doc_id lạ, ngày không ISO, version HR xung đột (10 vs 12 ngày phép), chunk stale refund (14 vs 7 ngày), và chunk text quá ngắn.

Pipeline chạy theo chuỗi **ingest → clean → validate → embed** được điều phối bởi `etl_pipeline.py`. Mỗi run sinh ra một `run_id` duy nhất (UTC timestamp hoặc tên tùy chỉnh), gắn với toàn bộ artifact: log, manifest, cleaned CSV, quarantine CSV.

**Lệnh chạy một dòng:**

```bash
python etl_pipeline.py run
```

**run_id** xuất hiện ở dòng đầu tiên của log (`artifacts/logs/run_<run_id>.log`) và trong manifest JSON (`artifacts/manifests/manifest_<run_id>.json`).

**Các run đã thực hiện:**

| run_id               | Mục đích                                            | Kết quả                                           |
| -------------------- | --------------------------------------------------- | ------------------------------------------------- |
| `sprint1`            | Run đầu, baseline data 10 rows                      | raw=10, cleaned=6, quarantine=4                   |
| `sprint2`            | Sau khi thêm Rule 7, 8, 9 + data expanded 13 rows   | raw=13, cleaned=7, quarantine=6                   |
| `inject-bad-sprint3` | Inject corruption (--no-refund-fix --skip-validate) | raw=13, cleaned=7, quarantine=6; expectation FAIL |

---

## 2. Cleaning & Expectation

Baseline đã có sẵn 6 rule (allowlist doc_id, chuẩn hoá ngày ISO, quarantine HR stale version, fix refund 14→7, quarantine text rỗng, dedupe). Nhóm bổ sung thêm **3 rule mới** và **2 expectation mới**.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn)                  | Trước (số liệu)                                              | Sau (số liệu)                                                                                                                  | Chứng cứ (log / CSV)                                                                    |
| -------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------- |
| `rule_quarantine_missing_exported_at` (Rule 7)     | row 12 (sla_p1_2026, exported_at rỗng) sẽ vào cleaned        | `quarantine_records` sprint1=4 → sprint2=6 (+1 từ row 12)                                                                      | `quarantine_sprint2.csv`: row 12 reason=`missing_exported_at`                           |
| `rule_quarantine_short_chunk_below_20` (Rule 8)    | row 11 "OK." (3 ký tự) sẽ vào cleaned và warn expectation E4 | `quarantine_records` tăng thêm 1; E4 không cần warn nữa                                                                        | `quarantine_sprint2.csv`: row 11 reason=`short_chunk_text`, `chunk_text_len=3`          |
| `rule_normalize_whitespace_in_chunk_text` (Rule 9) | row 13 "Nhân viên được..." với 15+ double-space              | `cleaned_sprint2.csv`: row 13 → "Nhân viên được phép đăng ký nghỉ phép qua hệ thống HR Portal ít nhất 3 ngày trước." (1 space) | `cleaned_sprint2.csv` dòng cuối                                                         |
| `E7 all_doc_ids_in_allowlist` (halt)               | Nếu rule 1 bỏ sót doc_id lạ → vector store nhiễm             | Sprint 3 inject `--skip-validate` bỏ qua, nhưng run chuẩn: `unknown_doc_id_count=0 OK`                                         | `run_sprint2.log`: `expectation[all_doc_ids_in_allowlist] OK (halt)`                    |
| `E8 max_chunk_text_length_2000` (warn)             | Không giới hạn độ dài → DB export nối chunk nhầm lọt qua     | Sprint 3: inject row > 2000 ký tự → expectation warn xuất hiện trong log                                                       | `run_sprint2.log`: `expectation[max_chunk_text_length_2000] OK (warn) :: long_chunks=0` |

**Rule chính (baseline + mở rộng):**

- **Baseline**: unknown_doc_id, missing/invalid effective_date, stale_hr_policy_effective_date, missing_chunk_text, duplicate_chunk_text, fix_stale_refund_14→7
- **Sprint 2 mới**: missing_exported_at (Rule 7), short_chunk_text < 20 chars (Rule 8), normalize_whitespace (Rule 9)

**Kết quả sprint2:** raw=13, cleaned=7 (+1 row 13 normalized), quarantine=6 (+2 so với sprint1)

**Ví dụ 1 lần expectation fail và cách xử lý:**

Sprint 3 inject (`--no-refund-fix --skip-validate`): `expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=1` — pipeline bị halt nếu không có `--skip-validate`. Fix: chạy lại pipeline chuẩn (không có flag inject) → expectation pass, refund window = 7 ngày.

---

## 3. Before / After ảnh hưởng retrieval

**Kịch bản inject (Sprint 3):**

Chạy `python etl_pipeline.py run --run-id inject-bad-sprint3 --no-refund-fix --skip-validate`:

- `--no-refund-fix`: Giữ nguyên chunk "14 ngày làm việc" (không sửa → 7 ngày)
- `--skip-validate`: Bỏ qua halt → embed chunk stale vào vector store

Expectation `refund_no_stale_14d_window` báo FAIL với `violations=1` nhưng pipeline tiếp tục embed (chỉ dùng cho demo).

**Kết quả định lượng (từ CSV eval):**

| Câu hỏi           | Metric               | Before (sprint2 clean) | After (inject-bad-sprint3) | Kết luận                                                   |
| ----------------- | -------------------- | ---------------------- | -------------------------- | ---------------------------------------------------------- |
| `q_refund_window` | `contains_expected`  | `yes`                  | `yes`                      | Cả hai đều trả về "7 ngày" ở top-1                         |
| `q_refund_window` | **`hits_forbidden`** | **`no`**               | **`yes`**                  | ⚠️ Inject làm chunk "14 ngày" xuất hiện trong top-k        |
| `q_leave_version` | `contains_expected`  | `yes`                  | `yes`                      | HR versioning vẫn ổn (bản 2025 đã quarantined từ Sprint 1) |
| `q_leave_version` | `hits_forbidden`     | `no`                   | `no`                       | HR stale chunk không bị ảnh hưởng bởi inject flag này      |
| `q_leave_version` | `top1_doc_expected`  | `yes`                  | `yes`                      | Top-1 vẫn từ `hr_leave_policy` đúng source                 |

**Ý nghĩa:** Dù top-1 vẫn hiển thị "7 ngày", nhưng context gợi ý (top-2 hoặc top-3) chứa chunk stale "14 ngày" → agent/LLM có thể bị lầm lẫn. `hits_forbidden=yes` phát hiện trước khi agent ra quyết định sai — đây là giá trị cốt lõi của observability layer.

**Paths:**

- Before clean: `artifacts/eval/before_clean_eval.csv`
- After inject: `artifacts/eval/after_inject_eval.csv`
- Quality report đầy đủ: `docs/quality_report.md`

---

## 4. Freshness & Monitoring

**SLA đã chọn:** `FRESHNESS_SLA_HOURS=24` (1 ngày) — phù hợp với policy CS/IT Helpdesk cập nhật hàng ngày.

**Kết quả freshness trên manifest sprint2:**

```
freshness_check=FAIL {
  "latest_exported_at": "2026-04-10T08:00:00",
  "age_hours": 120.75,
  "sla_hours": 24.0,
  "reason": "freshness_sla_exceeded"
}
```

**Ý nghĩa PASS/WARN/FAIL:**

| Trạng thái | Điều kiện                                    | Action                                               |
| ---------- | -------------------------------------------- | ---------------------------------------------------- |
| `PASS`     | `age_hours <= sla_hours` (≤24h)              | Pipeline OK, dữ liệu tươi                            |
| `WARN`     | `latest_exported_at` không có trong manifest | Manifest thiếu timestamp — cần kiểm tra lại ingest   |
| `FAIL`     | `age_hours > sla_hours` (>24h)               | Dữ liệu stale — trigger rerun ingest hoặc alert team |

**Giải thích FAIL trên dataset mẫu:** `policy_export_dirty.csv` có `exported_at=2026-04-10T08:00:00` — cũ 5 ngày so với runtime của lab. FAIL là **hợp lý và expected** cho dataset mẫu static. Trong production, cần automated refresh hoặc event-driven ingest khi hệ nguồn cập nhật.

**Chạy freshness check thủ công:**

```bash
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_sprint2.json
```

---

## 5. Liên hệ Day 09

Pipeline Day 10 xử lý lớp dữ liệu cho **cùng corpus CS + IT Helpdesk** đã dùng ở Day 09 multi-agent. Điểm khác biệt:

- Day 09 embed trực tiếp từ `data/docs/*.txt` → collection Day 09.
- Day 10 thêm intermediary layer: **CSV export → clean → validate → embed** vào collection `day10_kb`, mô phỏng production pipeline có observability.

Agent Day 09 có thể mount lại `day10_kb` thay collection cũ để tận dụng dữ liệu đã qua pipeline quality control — tức là "agent đọc đúng version" như narrative Day 10. Tích hợp này không bắt buộc trong lab nhưng thể hiện đúng tinh thần: pipeline tốt → agent trả lời đúng.

---

## 6. Rủi ro còn lại & Việc chưa làm

- **SLA 24h luôn FAIL với dataset mẫu:** `exported_at` cố định 2026-04-10 → production cần mock hoặc automate scheduled refresh.
- **Chỉ test 1 injection scenario:** Có thể thêm: duplicate record inject, missing chunk_text inject, doc_id lạ inject để kiểm tra rule/expectation mở rộng toàn diện hơn.
- **HR versioning vẫn dùng date check:** Nếu export ghi cùng date cho 2 phiên bản → cần metadata flag hoặc thêm rule detection logic.
- **Chroma không snapshot:** Cần artifact registry (MLflow, DVC) để versioning collection trong production.
- **Peer review (slide Phần E — 3 câu hỏi):**
  1. _Nếu pipeline không có freshness check, agent sẽ gặp vấn đề gì?_ → Agent đọc chunk stale mà không biết, confident trả lời sai.
  2. _Tại sao idempotency quan trọng với vector store?_ → Không có prune → mỗi rerun phình collection, vector cũ lạc hậu ảnh hưởng retrieval.
  3. _`hits_forbidden` phát hiện gì mà `contains_expected` không phát hiện được?_ → `contains_expected` chỉ check top-1; `hits_forbidden` quét toàn bộ top-k context — phát hiện chunk sai trong context dù top-1 đúng.
