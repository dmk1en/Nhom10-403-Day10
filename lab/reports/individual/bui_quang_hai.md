# Báo Cáo Cá Nhân — Embed & Idempotency Owner

**Họ và tên:** Bùi Quang Hải  
**Vai trò:** Embed & Idempotency Owner  
**Độ dài:** ~450 từ

---

## 1. Phụ trách

Tôi chịu trách nhiệm về embed dữ liệu vào vector store và đảm bảo tính idempotency trong pipeline. Các nhiệm vụ chính bao gồm:

- **Embed:** Đảm bảo dữ liệu được nhúng chính xác, không trùng lặp, và không lỗi.
- **Idempotency:** Đảm bảo pipeline chạy nhiều lần không làm thay đổi trạng thái dữ liệu đã nhúng.
- **Kiểm tra:** Xác minh log và manifest để đảm bảo tính toàn vẹn của dữ liệu.

**Bằng chứng:**
- Log: `artifacts/logs/run_sprint2.log`
- Manifest: `artifacts/manifests/manifest_sprint2.json`

---

## 2. Quyết định kỹ thuật

- **Prune vector:** Sau khi inject lỗi, tôi quyết định prune các vector không còn trong batch để tránh top-k trả về kết quả không mong muốn.
- **Idempotency:** Đảm bảo rằng mỗi run_id là duy nhất và không xung đột, giúp pipeline có thể chạy lại mà không ảnh hưởng đến dữ liệu.

---

## 3. Sự cố / anomaly

- Khi thử bỏ prune vector, `grading_run.jsonl` báo `hits_forbidden=true` dù cleaned đã sạch. Nguyên nhân là vector cũ chưa được xóa. Fix: thêm bước prune trong `etl_pipeline.py`.

---

## 4. Before/after

- **Log:** `expectation[refund_no_stale_14d_window] OK (halt)` sau run chuẩn; trước đó với `--no-refund-fix` expectation FAIL.
- **CSV:** Dòng `q_refund_window` có `hits_forbidden=no` trong `artifacts/eval/before_clean_eval.csv` và `hits_forbidden=yes` trong `artifacts/eval/after_inject_eval.csv`.

---

## 5. Cải tiến thêm 2 giờ

- Đọc cutoff HR `2026-01-01` từ `contracts/data_contract.yaml` thay vì hard-code trong Python.
- Tối ưu hóa embed để giảm thời gian xử lý khi batch lớn.