# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Tạ Vĩnh Phúc  
**Vai trò:** Monitoring / Docs Owner 
**Ngày nộp:** 2026-04-15  
**Độ dài yêu cầu:** 400–650 từ

---

## 1. Tôi phụ trách phần nào?

**File / module chính:**

- `docs/pipeline_architecture.md` — viết sơ đồ ASCII pipeline đầy đủ (các tầng ingest → clean → validate → embed → monitor), bảng ranh giới 5 component, giải thích chiến lược idempotency Snapshot Publish, liên hệ Day 09.
- `docs/runbook.md` — soạn 2 kịch bản incident thực tế: (1) agent trả lời sai refund window do chunk stale, (2) freshness FAIL do dataset cũ; mỗi kịch bản đủ 5 mục Symptom → Detection → Diagnosis → Mitigation → Prevention.
- `lab/README.md` — cập nhật đoạn Fast Quick Start với lệnh `run --run-id final_run` và `freshness --manifest`.
- `reports/group_report.md` — hoàn thiện section 3 (before/after evidence), 4 (freshness SLA), 5 (liên hệ Day 09), 6 (rủi ro + peer review 3 câu).

**Kết nối với thành viên khác:**

- Sprint 1–2 (Ingestion + Cleaning Owner): cung cấp `manifest_sprint2.json` và quarantine CSV — tôi dùng để phân tích freshness FAIL và đưa vào runbook.
- Sprint 3 (Embed + Inject Owner): cung cấp 2 file eval CSV (`before_clean_eval.csv`, `after_inject_eval.csv`) — tôi dùng làm bằng chứng before/after trong group report.

**Bằng chứng:**

- `docs/pipeline_architecture.md` và `docs/runbook.md` — authored Sprint 4 (chuyển từ template trống thành file hoàn chỉnh).
- Evidence lệnh chạy trên máy cá nhân:
  ```bash
  python etl_pipeline.py run --run-id final_run
  python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_final_run.json
  ```

---

## 2. Một quyết định kỹ thuật: Chọn Idempotency theo "Snapshot Publish" thay vì append

Khi viết `docs/pipeline_architecture.md`, tôi phải mô tả và chốt chiến lược idempotency cho embed. Có 2 lựa chọn:

- **Append-only**: mỗi run thêm vector mới, không xóa cũ → index phình dần, vector lạc hậu tồn tại lâu dài.
- **Snapshot Publish** (lựa chọn đã chốt): mỗi run là một snapshot — upsert theo `chunk_id` và **prune** các ID không còn trong cleaned batch. Index sau publish phản ánh đúng cleaned data tại thời điểm đó, không có "zombie vector".

Tôi chọn Snapshot Publish vì phù hợp với tinh thần observability: "publish boundary" rõ ràng — agent đọc đúng version tại mỗi điểm thời gian. Điều này được ghi trong `pipeline_architecture.md` mục 3 (Idempotency & Rerun) và minh chứng bằng log field `embed_prune_removed` xuất hiện trong mỗi run.

---

## 3. Một lỗi / anomaly đã xử lý: Freshness FAIL không phải do pipeline lỗi

**Triệu chứng:** Chạy `python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_final_run.json` trả về:
```
FAIL {"latest_exported_at": "2026-04-10T08:00:00", "age_hours": 121.6, "sla_hours": 24.0, "reason": "freshness_sla_exceeded"}
```

**Phát hiện:** Thoạt nhìn tưởng pipeline lỗi vì exit code 1. Thực ra đây là **expected behavior** — dataset mẫu `policy_export_dirty.csv` có `exported_at` cố định ngày 10/04, tức đã cũ 5 ngày so với thời điểm chạy lab.

**Diagnosis:** Phân biệt 2 khái niệm: "pipeline run thành công" (exit 0 từ `etl_pipeline.py run`) vs "dữ liệu tươi" (freshness PASS). Pipeline vẫn `PIPELINE_OK`, chỉ có dữ liệu nguồn stale. Freshness check đo đúng `latest_exported_at` của dữ liệu, không phải `run_timestamp` của pipeline — đây là điểm then chốt.

**Fix và tài liệu hóa:** Ghi rõ trong `docs/runbook.md` mục Prevention: với dataset mẫu static, FAIL là intentional; production cần automated refresh hoặc điều chỉnh `FRESHNESS_SLA_HOURS` env.

---

## 4. Bằng chứng trước / sau

**run_id run clean:** `sprint2` | **run_id run inject:** `inject-bad-sprint3`

Trích từ `artifacts/eval/before_clean_eval.csv` (sprint2 — pipeline chuẩn):
```
q_refund_window | contains_expected=yes | hits_forbidden=no
```

Trích từ `artifacts/eval/after_inject_eval.csv` (inject-bad-sprint3 — bỏ fix):
```
q_refund_window | contains_expected=yes | hits_forbidden=yes
```

**Diễn giải:** `contains_expected=yes` ở cả hai run (top-1 đều "7 ngày"), nhưng sau inject `hits_forbidden=yes` — chunk stale "14 ngày" xuất hiện trong top-k context. Đây là bằng chứng rằng **context poisoning** xảy ra dù top-1 trông đúng. Metric `hits_forbidden` trong `eval_retrieval.py` phát hiện anomaly này — được tôi giải thích chi tiết trong `docs/runbook.md` mục Detection.

Kết quả grading (`artifacts/eval/grading_run.jsonl`): `gq_d10_01`: `contains_expected=true`, `hits_forbidden=false` ✅ | `gq_d10_03`: `contains_expected=true`, `hits_forbidden=false`, `top1_doc_matches=true` ✅ → đạt điều kiện **Merit**.

---

## 5. Cải tiến tiếp theo (nếu có thêm 2 giờ)

Tôi sẽ thêm freshness đo tại **2 boundary**: (1) `exported_at` từ hệ nguồn (ingest boundary — đã có) và (2) `run_timestamp` publish gần nhất (publish boundary — chưa có). Thêm boundary này giúp phân biệt "data stale" vs "pipeline stale", cung cấp alert cụ thể hơn cho ops và đủ điều kiện Bonus +1 theo SCORING.
