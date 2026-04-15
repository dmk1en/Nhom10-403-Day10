# Quality Report — Lab Day 10 (nhóm)

**run_id:** sprint2 (clean), inject-bad-sprint3 (corrupted)  
**Ngày:** 2026-04-15

---

## 1. Tóm tắt số liệu

| Chỉ số | Trước (sprint2) | Sau (inject-bad-sprint3) | Ghi chú |
|--------|-----------------|--------------------------|---------|
| raw_records | 13 | 13 | Giống — cùng dataset |
| cleaned_records | 7 | 7 | Giống — cùng cleaning logic |
| quarantine_records | 6 | 6 | Giống — cùng failure mode |
| Expectation halt? | **NO** | **YES** (refund_no_stale_14d_window FAIL) | Inject ghi đè refund 14→7 fix |

**Chi tiết expectation sprint2 (PASS):**
- ✅ min_one_row: 7 cleaned rows
- ✅ no_empty_doc_id: 0 empty
- ✅ **refund_no_stale_14d_window: 0 violations** ← FIX applied
- ✅ all_doc_ids_in_allowlist: 0 unknown

**Chi tiết expectation inject (FAIL):**
- ✅ min_one_row: 7 cleaned rows
- ✅ no_empty_doc_id: 0 empty
- ❌ **refund_no_stale_14d_window: 1 violation** ← FIX NOT applied (--no-refund-fix)
- ✅ all_doc_ids_in_allowlist: 0 unknown

---

## 2. Before / after retrieval (bắt buộc)

### 2.1 Câu hỏi then chốt: refund window (`q_refund_window`)

**Before (clean sprint2):**
```
question_id: q_refund_window
top1_doc_id: policy_refund_v4
top1_preview: "Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng."
contains_expected: yes
hits_forbidden: NO ← ✅ Không phát hiện chunk stale 14 ngày
```

**After (inject-bad-sprint3):**
```
question_id: q_refund_window
top1_doc_id: policy_refund_v4
top1_preview: "Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng."
contains_expected: yes
hits_forbidden: YES ← ⚠️ PHÁT HIỆN chunk stale 14 ngày trong top-k!
```

**Ý nghĩa:** Khi bỏ `--no-refund-fix`, pipeline không xử lý chunk policy thừa có "14 ngày làm việc". Dù top-1 vẫn hiển thị "7 ngày", nhưng context gợi ý (top-2 hoặc top-3) chứa câu trả lời sai "14 ngày" → agent/LLM có thể bị lầm lẫn. **Đây là ví dụ hoàn hảo về "observability" — metric `hits_forbidden` phát hiện anomaly trước khi agent ra quyết định sai.**

---

### 2.2 Merit (khuyến nghị): versioning HR — `q_leave_version`

**Before:**
```
question_id: q_leave_version  
top1_doc_id: hr_leave_policy
top1_preview: "Nhân viên dưới 3 năm kinh nghiệm được 12 ngày phép năm theo chính sách 2026."
contains_expected: yes ← ✅ Có "12 ngày" (chính sách mới)
hits_forbidden: no ← ✅ Không có "10 ngày" (chính sách cũ)
top1_doc_expected: yes ← ✅ Top-1 từ hr_leave_policy (đúng source)
```

**After:**
```
question_id: q_leave_version
top1_doc_id: hr_leave_policy
top1_preview: "Nhân viên dưới 3 năm kinh nghiệm được 12 ngày phép năm theo chính sách 2026."
contains_expected: yes
hits_forbidden: no ← ✅ Vẫn không phát hiện chunk 10 ngày
top1_doc_expected: yes
```

**Nhận xét:** HR versioning vẫn ổn (pipeline tách bản 2025 khỏi cleaned data ngay ở Sprint 1). Injection flag không ảnh hưởng trường hợp này vì HR cũ đã được quarantined—không có cách bỏ fix để restore stale HR version trong injection scenario hiện tại.

---

## 3. Freshness & monitor

**Sprint 2 manifest freshness:**
```
freshness_check=FAIL {
  "latest_exported_at": "2026-04-10T08:00:00",
  "age_hours": 120.75,
  "sla_hours": 24.0,
  "reason": "freshness_sla_exceeded"
}
```

**Giải thích:** Dataset policy_export_dirty.csv xuất từ 2026-04-10 08:00 UTC; tính tới 2026-04-15 08:00 UTC là **120+ giờ = 5 ngày** — vượt SLA 24 giờ. Trạng thái **FAIL** là cảnh báo hợp lệ: dữ liệu trong vector store có thể lỗi thời. Trong production, cần:
- **Action:** Trigger rerun pipeline nếu freshness FAIL
- **Alert:** Ghi log / gửi team khi dữ liệu >24h
- **Prevention:** Automated scheduled refresh hoặc event-driven ingest

---

## 4. Corruption inject (Sprint 3)

### 4.1 Kịch bản inject

**Flag sử dụng:** `python etl_pipeline.py run --run-id inject-bad-sprint3 --no-refund-fix --skip-validate`

- `--no-refund-fix`: Không áp dụng transformation sửa chữa chunk policy từ "14 ngày làm việc" → "7 ngày làm việc"
- `--skip-validate`: Bỏ qua việc halt pipeline khi expectation FAIL (cho phép embed dữ liệu hỏng)

**Hiệu ứng:**
- Cleaned CSV vẫn 7 rows (logic clean_rows không thay đổi)
- **Nhưng** chunk nội dung không được sửa → chunk chứa "14 ngày" vẫn được embed
- Expectation `refund_no_stale_14d_window` FAIL (detect violation)
- Dù expectation FAIL, pipeline vẫn chạy embed (skip-validate) → vector store có chunk sai

### 4.2 Phát hiện & bằng chứng

| Phương pháp | Kết quả Before | Kết quả After | Kỳ vọng |
|------------|---|---|---|
| Expectation suite (automated) | ✅ refund_no_stale_14d_window: 0 violations | ❌ refund_no_stale_14d_window: 1 violation | FAIL detected ✓ |
| Retrieval eval (`hits_forbidden`) | ❌ No stale chunk in top-k | ✅ **Stale chunk found in top-k** | Corruption visible ✓ |
| Log message | `PIPELINE_OK` | `WARN: expectation failed but --skip-validate → continue embed` | Gap acknowledged ✓ |

**Kết luận:** Injection thành công — vector store bị "poison" với chunk stale, và `hits_forbidden` metric phát hiện. Đây là chứng minh rằng **observability layer (eval + expectation + logging) hoạt động** — khác với pipeline "im lặng chạy sai".

---

## 5. Hạn chế & việc chưa làm

- **SLA 24h gặp FAIL:** Dataset mẫu `policy_export_dirty.csv` cũ, nên freshness luôn red flag — trong production cần mock latest `exported_at` hoặc automate scheduled refresh.
- **Chỉ test 1 injection scenario:** Có thể thêm scenario khác (duplicate record, missing chunk_text) để kiểm test rule/expectation mở rộng từ Sprint 2.
- **Versioning HR still manual:** Bản cũ (2025) được quarantine bằng date check, nhưng nếu export ghi lại `2025-12-31` cho cả 2 phiên bản → cần thêm rule detection logic hoặc metadata flag.
- **Chroma collection không snapshot:** Dữ liệu embed vẫn giữ trên disk (`chroma_db/`), không backup — cần integration với artifact registry (MLflow, etc.) cho production.

---

## 6. Liên hệ & tích hợp tiếp theo

Sprint 3 hoàn thành chứng minh **before/after data layer ảnh hưởng retrieval**; ready để:
- **Sprint 4 (monitoring/runbook):** Thêm automation trigger remediation khi freshness FAIL hoặc expectation alert.
- **Day 11 (guardrail):** Wrap retrieval eval vào agent orchestration — nếu `hits_forbidden=yes`, agent yêu cầu re-rank hoặc skip source.
- **Re-integration Day 09:** Vector collection `day10_kb` có thể được mount lại agent Day 09 để test multi-agent + data layer mới.

**Công thức quan trọng nhất từ Day 10:**
```
Data Quality (expectation + eval) = Before Agent trả lời
```

Nếu dữ liệu không observability, agent chỉ có thể "confident sai" — Sprint 3 chứng minh rằng một dòng `hits_forbidden: yes` có thể cứu cuộc hành động.
