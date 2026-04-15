# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Vũ Trung Lập
**Vai trò:** Monitoring / Docs Owner (Phụ trách Quy trình & Báo cáo chất lượng)
**Ngày nộp:** 2026-04-15
**Số từ:** ~580 từ

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

Trong giai đoạn hoàn thiện, tôi chịu trách nhiệm thiết lập "hàng rào kiểm soát" cuối cùng để đảm bảo tính minh bạch và độ tin cậy của dữ liệu trước khi nộp bài.

**File / module cụ thể:**
- `reports/group_report.md`: Tổng hợp bảng `metric_impact`, phân tích hệ quả của việc inject lỗi và soạn thảo 3 câu hỏi Peer Review.
- `eval_retrieval.py` & `grading_run.py`: Thực thi đánh giá tự động trên môi trường `.venv` để tạo ra các artifact: `grading_run.jsonl` và các file `eval.csv`.
- `instructor_quick_check.py`: Sử dụng script của giảng viên để kiểm tra tính hợp lệ của manifest và kết quả grading trước khi commit.

**Kết nối:** Tôi tiếp nhận đầu ra từ team Ingestion và Cleaning, đóng vai trò "người thẩm định" để chuyển hóa các kết quả kỹ thuật thành báo cáo chất lượng đúng barem chấm điểm.

---

## 2. Một quyết định kỹ thuật (100–150 từ)

**Quyết định: Sử dụng `hits_forbidden` làm chỉ số chặn (Halt metric) thay vì chỉ tin vào `contains_expected`.**

Trong quá trình thiết kế báo cáo chất lượng, tôi nhận thấy rủi ro **Context Poisoning (Nhiễm độc ngữ cảnh)**: Agent có thể tìm đúng thông tin ở Top-1 (SLA 7 ngày), nhưng nếu Context (Top-k) chứa đồng thời thông tin cũ (SLA 14 ngày), LLM sẽ bị "nhiễu" và có thể tư vấn sai.

Tôi quyết định sử dụng cờ `hits_forbidden=true` làm điều kiện tiên quyết để đánh giá một run là FAIL. Thay vì chỉ mừng rỡ khi thấy từ khóa đúng xuất hiện, tôi yêu cầu hệ thống phải quét toàn bộ Top-k để đảm bảo "nội dung cấm" (thông tin stale) hoàn toàn biến mất. Việc phối hợp giữa `contains_expected` (độ phủ) và `hits_forbidden` (độ sạch) giúp tầng Observability thực sự bảo vệ được sự an toàn cho AI Agent phía sau.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

**Anomaly: "Silent Failure" (Lỗi âm thầm) khi thực thi grading trên Python Global.**

Triệu chứng: Khi chạy `grading_run.py`, lệnh thực hiện rất nhanh và báo thành công, nhưng file `grading_run.jsonl` lại thiếu dữ liệu hoặc báo lỗi `ModuleNotFoundError` về `chromadb`. Qua kiểm tra, tôi phát hiện hệ thống đang gọi Python hệ thống thay vì môi trường ảo `.venv` đã cài đặt đủ thư viện.

Đây là một lỗi nguy hiểm vì nếu không kiểm tra artifact, nhóm sẽ nộp tài liệu rỗng. Tôi đã xử lý bằng cách chuẩn hóa lại đường dẫn thực thi: `.venv\Scripts\python.exe grading_run.py`. Việc này đảm bảo tính nhất quán (consistency) giữa các lần chạy và giúp script `instructor_quick_check.py` có thể đọc đúng metadata từ manifest. Bài học rút ra là luôn phải kiểm chứng (validate) đầu ra của artifact sau mỗi lệnh run tự động.

---

## 4. Bằng chứng trước / sau

**Log:** `expectation[refund_no_stale_14d_window] OK (halt) :: violations=0` trong `run_final_sprint4.log` xác nhận pipeline đã tự động sửa lỗi text stale 14 ngày.

**CSV (Standard):** Dòng `q_refund_window` có `hits_forbidden=no` trong `artifacts/eval/before_after_eval.csv`, minh chứng cho ngữ cảnh sạch sau khi qua pipeline.

**CSV (Injected):** Dòng `q_refund_window` có `hits_forbidden=yes` trong `artifacts/eval/after_inject_eval.csv`, chứng minh hệ thống phát hiện được dữ liệu stale lọt vào khi tắt cơ chế clean.

**Grading:** File `grading_run.jsonl` trả về `top1_doc_matches=true` cho câu hỏi `gq_d10_03`, khẳng định ranking versioning (chọn bản 2026 thay vì 2025) hoạt động hoàn hảo.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ xây dựng một **Auto-Alert Dashboard** bằng Streamlit. Dashboard này sẽ tự động parse `grading_run.jsonl` và manifest để hiển thị radar chart về Data Quality (độ tươi, độ sạch, độ phủ) ngay khi pipeline kết thúc, giúp team xử lý sự cố ngay lập tức mà không cần đọc log thủ công.
