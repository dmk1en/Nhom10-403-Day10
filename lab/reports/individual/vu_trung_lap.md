# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Vũ Trung Lập
**Vai trò:** Monitoring / Docs Owner (Phụ trách Quy trình & Báo cáo chất lượng)
**Ngày nộp:** 2026-04-15
**Độ dài yêu cầu:** 400–650 từ

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

Trong đợt Submit 2 (Phần hoàn thiện), tôi chịu trách nhiệm chính trong việc kiểm soát chất lượng đầu ra của toàn bộ pipeline và đối chiếu với các yêu cầu chấm điểm trong `SCORING.md`.

**File / module cụ thể:**

- `reports/group_report.md`: Tổng hợp bảng `metric_impact`, phân tích kết quả Before/After Retrieval và soạn thảo 3 câu hỏi Peer Review.
- `eval_retrieval.py` & `grading_run.py`: Trực tiếp thực thi các lệnh đánh giá trên môi trường máy ảo để sinh ra các artifact quan trọng (`grading_run.jsonl`, `before_after_eval.csv`).

**Kết nối với thành viên khác:**
Tôi là "trạm cuối" tiếp nhận `run_id` và các tệp log từ Dương Mạnh Kiên (Ingestion Owner) để chuyển hóa các con số khô khan thành bằng chứng về độ tin cậy của dữ liệu cho giảng viên.

---

## 2. Một quyết định kỹ thuật (100–150 từ)

**Quyết định: Sử dụng chỉ số `hits_forbidden` làm tiêu chuẩn "Halt" thay vì chỉ dựa vào `contains_expected`.**

Trong quá trình thiết kế báo cáo chất lượng, tôi nhận thấy một vấn đề lớn: Một hệ thống RAG có thể trả về câu trả lời đúng ở Top-1 (tức là `contains_expected=true`), nhưng nếu trong Top-k (context) vẫn chứa các thông tin cũ/sai lệch (stale data), Agent có nguy cơ bị "poisoned" và nội suy sai ở các bước tiếp theo.

Tôi đã quyết định nhấn mạnh vào chỉ số `hits_forbidden` trong báo cáo nhóm. Thay vì chỉ mừng rỡ khi thấy Agent tìm ra "7 ngày làm việc", tôi yêu cầu hệ thống phải báo động (FAIL) nếu phát hiện thấy bất kỳ chữ "14 ngày làm việc" nào lọt vào Context. Quyết định này giúp tầng Data Observability thực sự bảo vệ được tầng AI Agent phía sau, tránh việc Agent tự tin khẳng định một thông tin đúng nhưng lại kẹp thêm dẫn chứng sai vào câu trả lời.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

**Anomaly: Hiện tượng "Silent Failure" khi chạy grading trên Global Python.**

Khi thực hiện lệnh `python grading_run.py`, tôi gặp lỗi `ModuleNotFoundError` dù đã cài đặt các package. Qua kiểm tra, tôi phát hiện hệ thống đang gọi Python toàn cục thay vì môi trường ảo `.venv` chứa `chromadb` và `sentence-transformers`.

Thay vì chỉ cài đặt lại package vào máy (cách làm không ổn định), tôi đã xử lý bằng cách chuẩn hóa kịch bản chạy trong README và trực tiếp thực thi qua đường dẫn tuyệt đối của venv: `.venv\Scripts\python.exe`. Lỗi này tuy nhỏ nhưng nếu không phát hiện, các artifact sinh ra sẽ bị rỗng hoặc lỗi format, dẫn đến việc `instructor_quick_check.py` báo FAIL. Sau khi chuyển sang venv, tôi đã thu được file `grading_run.jsonl` hoàn chỉnh với đầy đủ 3 câu hỏi đạt chuẩn Merit.

---

## 4. Bằng chứng trước / sau (80–120 từ)

Tôi đã sử dụng kết quả từ `inject-bad-sprint3` để chứng minh hiệu quả của hàng rào bảo vệ:

**So sánh kết quả Retrieval:**

- **Trước khi Ingest lỗi (sprint2):** `hits_forbidden = no`. Không có tài liệu cũ nào lọt vào.
- **Sau khi Inject lỗi (--no-refund-fix):** `hits_forbidden = YES`. Hệ thống đo lường đã ngay lập tức bắt được chunk "14 ngày" xuất hiện trong top-5 context.

**Trích đoạn Grading JSONL (`gq_d10_01`):**

```json
{
  "id": "gq_d10_01",
  "contains_expected": true,
  "hits_forbidden": false,
  "grading_criteria": [
    "Trả lời đúng số ngày (7 ngày)",
    "Không khẳng định 14 ngày"
  ]
}
```

Kết quả thực tế cho thấy hệ thống đã chặn thành công dữ liệu bẩn, đảm bảo `hits_forbidden` luôn là `false` trong điều kiện pipeline chuẩn.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ xây dựng một **Dashboard Giám sát thời gian thực** (sử dụng Streamlit hoặc bảng HTML đơn giản) để tự động hiển thị biểu đồ `metric_impact` ngay sau mỗi lệnh `run`. Điều này giúp Team Monitoring không phải mở file CSV thủ công mà có thể quan sát trực quan sự biến động của tỷ lệ dữ liệu bị Quarantine và các cảnh báo Freshness ngay trên trình duyệt.
