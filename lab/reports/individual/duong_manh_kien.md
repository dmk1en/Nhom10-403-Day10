# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Dương Mạnh Kiên
**Vai trò:** Ingestion Owner (Sprint 1) & Cleaning / Quality Owner (Sprint 2)
**Ngày nộp:** 4/15/2026
**Độ dài yêu cầu:** 400–650 từ

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

- `data/raw/policy_export_dirty.csv` — mở rộng thêm 3 dòng dữ liệu bẩn mới (row 11–13) để tạo kịch bản kiểm chứng cho rule mới.
- `transform/cleaning_rules.py` — thêm Rule 7 (`missing_exported_at`), Rule 8 (`short_chunk_text < 20`), Rule 9 (`normalize_whitespace`).
- `quality/expectations.py` — thêm E7 (`all_doc_ids_in_allowlist`, halt) và E8 (`max_chunk_text_length_2000`, warn).
- `docs/data_contract.md` — điền source map đầy đủ: 2 nguồn, failure mode, metric/alert.
- `contracts/data_contract.yaml` — điền `owner_team` và `alert_channel`.

**Kết nối với thành viên khác:**

Sprint 1 bàn giao log và manifest (`run_id=sprint1`) cho Embed Owner để xác nhận 6 record đầu vào trước khi upsert Chroma. Sprint 2 cung cấp file `cleaned_sprint2.csv` (7 records) và bảng `metric_impact` trong `group_report.md` cho Monitoring / Docs Owner tổng hợp quality report.

**Bằng chứng:**

`transform/cleaning_rules.py` — comment docstring từng rule (7), (8), (9) ghi rõ `metric_impact`. `quality/expectations.py` — comment E7, E8 ghi severity và kịch bản chứng minh.

---

## 2. Một quyết định kỹ thuật (100–150 từ)

**Quyết định: đặt Rule 7 (`missing_exported_at`) là quarantine thay vì chỉ warn.**

Ban đầu tôi cân nhắc chỉ log warning khi `exported_at` rỗng, vì trường này không ảnh hưởng trực tiếp đến nội dung chunk được embed. Tuy nhiên sau khi xem xét luồng `freshness_check` trong `monitoring/freshness_check.py`, tôi nhận ra `latest_exported_at` trong manifest được tính bằng `max(exported_at)` của toàn bộ cleaned rows. Nếu để row thiếu `exported_at` lọt vào cleaned, giá trị `max()` vẫn đúng (Python bỏ qua chuỗi rỗng khi so sánh theo lexicographic), nhưng một số batch khác có thể không có row nào có timestamp hợp lệ — khi đó freshness check trả về `WARN: no_timestamp_in_manifest`, che khuất lỗi thật. Vì vậy tôi chọn **quarantine** (severity cao hơn) để buộc data team phải fix tại nguồn trước khi record được publish vào vector store. Quyết định này khớp với nguyên tắc "fail fast at the boundary" trong pipeline observability.

---

## 3. Một lỗi / anomaly đã xử lý (100–150 từ)

**Anomaly: Rule 9 (normalize whitespace) không thay đổi `quarantine_records` nhưng thay đổi `chunk_id`.**

Khi chạy `python etl_pipeline.py run --run-id sprint2` lần đầu, tôi quan sát thấy `quarantine_records=6` (tăng 2 đúng như dự kiến từ Rule 7 và 8), nhưng `cleaned_records=7` — tức row 13 vẫn vào cleaned. Kiểm tra `cleaned_sprint2.csv` cho thấy dòng cuối có `chunk_id=hr_leave_policy_7_19b04bf71ceb912d`, khác hoàn toàn với chunk_id từ sprint1 (nếu row đó tồn tại). Nguyên nhân: `_stable_chunk_id` hash theo `chunk_text` sau khi normalize — text đã thay đổi (double-space → single-space) nên hash khác. Điều này là **đúng về mặt logic** (chunk đã được làm sạch nên id mới là chính xác hơn), nhưng cần ghi chú trong runbook: nếu cùng một row xuất hiện với whitespace khác nhau giữa hai batch, Chroma sẽ tạo thêm entry mới thay vì upsert vào entry cũ. Fix: pipeline đã có bước `embed_prune_removed` xóa id cũ sau mỗi publish — xác nhận bằng log `embed_prune_removed=N` khi chạy lại.

---

## 4. Bằng chứng trước / sau (80–120 từ)

**So sánh sprint1 vs sprint2 (run_id ghi trong log tương ứng):**

| Metric | sprint1 | sprint2 | Delta |
|--------|---------|---------|-------|
| `raw_records` | 10 | 13 | +3 (row 11–13 thêm vào) |
| `cleaned_records` | 6 | 7 | +1 (row 13 normalize pass) |
| `quarantine_records` | 4 | 6 | +2 (Rule 7: row 12, Rule 8: row 11) |
| Số expectation | 6 | 8 | +2 (E7 halt, E8 warn) |

**Trích log sprint2** (`artifacts/logs/run_sprint2.log`):
```
quarantine_records=6
expectation[all_doc_ids_in_allowlist] OK (halt) :: unknown_doc_id_count=0
expectation[max_chunk_text_length_2000] OK (warn) :: long_chunks=0
embed_upsert count=7 collection=day10_kb
PIPELINE_OK
```

Toàn bộ 8 expectation pass, pipeline exit 0. `embed_upsert count=7` khớp với `cleaned_records=7`.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ đọc cutoff versioning (`hr_leave_min_effective_date`) trực tiếp từ `contracts/data_contract.yaml` thay vì hard-code chuỗi `"2026-01-01"` trong `cleaning_rules.py`. Điều này đáp ứng tiêu chí Distinction (d): "rule versioning không hard-code ngày cố định". Khi contract thay đổi (ví dụ HR cập nhật lên policy 2027), chỉ cần sửa YAML — pipeline tự áp dụng đúng cutoff mà không cần chạm code.
