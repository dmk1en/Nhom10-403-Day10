# Runbook — Lab Day 10: Data Pipeline & Observability

**Nhóm:** Nhóm 10 — Lớp 403  
**Cập nhật:** 2026-04-15  
**Lệnh một dòng chạy pipeline:**
```bash
cd lab && python etl_pipeline.py run --run-id sprint4
```

---

## Incident 1: Agent / retrieval trả lời sai cửa sổ hoàn tiền ("14 ngày" thay vì "7 ngày")

---

### Symptom

User hoặc agent nhận được câu trả lời chứa **"14 ngày làm việc"** cho câu hỏi về refund window — trong khi chính sách hiện hành (`policy_refund_v4`) quy định **7 ngày**.

---

### Detection

Các tín hiệu cảnh báo theo thứ tự ưu tiên:

| Tín hiệu | Metric / Check | Vị trí |
|----------|---------------|--------|
| Expectation FAIL | `expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=1` | `artifacts/logs/run_<run_id>.log` |
| Retrieval eval | `hits_forbidden=yes` cho câu `q_refund_window` | `artifacts/eval/before_clean_eval.csv` hoặc `after_inject_eval.csv` |
| Pipeline không halt (nếu dùng `--skip-validate`) | Log có dòng `WARN: expectation failed but --skip-validate` | `artifacts/logs/run_<run_id>.log` |
| Freshness FAIL | `freshness_check=FAIL {"age_hours": X, "sla_hours": 24}` | `artifacts/manifests/manifest_<run_id>.json` |

---

### Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|-----------------|
| 1 | Kiểm tra `artifacts/manifests/manifest_<run_id>.json` | Phải có `"no_refund_fix": false`; nếu `true` → nguyên nhân là inject flag |
| 2 | Mở `artifacts/logs/run_<run_id>.log` | Tìm dòng `expectation[refund_no_stale_14d_window]`; nếu `FAIL` → chunk stale đang được embed |
| 3 | Mở `artifacts/quarantine/quarantine_<run_id>.csv` | Kiểm tra xem chunk `policy_refund_v4` với text "14 ngày" đã được quarantine chưa |
| 4 | Chạy `python eval_retrieval.py --out artifacts/eval/check_now.csv` | Kiểm tra `hits_forbidden` — nếu `yes` → vector store bị nhiễm chunk stale |
| 5 | Đếm vector trong collection | `col.count()` trong Python; nếu nhiều hơn `cleaned_records` bình thường → prune chưa chạy |

**Debug order (theo slide Day 10):**
```
Freshness / version → Volume & errors → Schema & contract → Lineage / run_id → mới đến model/prompt
```

---

### Mitigation

1. **Rerun pipeline chuẩn** (không có flag inject):
   ```bash
   python etl_pipeline.py run --run-id recovery-$(date +%Y%m%d)
   ```
   Pipeline tự động fix chunk "14 ngày → 7 ngày" và prune vector stale.

2. **Xác nhận fix:**
   ```bash
   python eval_retrieval.py --out artifacts/eval/after_recovery.csv
   # Kiểm tra: hits_forbidden=no cho q_refund_window
   ```

3. **Tạm thời** (nếu không thể rerun ngay): thông báo cho team sử dụng agent tạm suspend câu hỏi về refund window cho đến khi pipeline rerun xong.

4. **Rollback embed**: Nếu cần rollback về version trước, dùng `cleaned_<run_id_cũ>.csv` để rebuild:
   ```bash
   # Xóa và rebuild collection từ CSV cũ
   python etl_pipeline.py run --run-id rollback --raw artifacts/cleaned/cleaned_<run_ok>.csv
   ```

---

### Prevention

1. **Expectation suite**: `expectation[refund_no_stale_14d_window]` severity `halt` — pipeline sẽ không embed nếu chunk stale tồn tại (trừ khi dùng `--skip-validate` một cách có chủ đích).
2. **Freshness SLA**: Đặt `FRESHNESS_SLA_HOURS=24` trong `.env`; khi `FAIL`, trigger rerun pipeline hoặc alert Slack `#data-alerts`.
3. **Eval định kỳ**: Chạy `eval_retrieval.py` sau mỗi deploy — `hits_forbidden=yes` → incident.
4. **Không dùng `--skip-validate` trong production**: Chỉ dùng cho demo Sprint 3.
5. **Giám sát Day 11**: Nếu tích hợp guardrail, agent kiểm tra `hits_forbidden` trước mỗi response — nếu `yes` thì re-rank hoặc skips source đó.

---

## Incident 2: Freshness FAIL — Dữ liệu cũ hơn SLA

---

### Symptom

Log pipeline hoặc `python etl_pipeline.py freshness --manifest ...` trả về:
```
FAIL {"latest_exported_at": "2026-04-10T08:00:00", "age_hours": 120.75, "sla_hours": 24.0, "reason": "freshness_sla_exceeded"}
```

---

### Detection

- `freshness_check=FAIL` trong `artifacts/logs/run_<run_id>.log`
- Hoặc chạy thủ công: `python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_<run_id>.json`

---

### Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|-----------------|
| 1 | Kiểm tra `latest_exported_at` trong manifest | So sánh với `now`; nếu nguồn cũ → cần reexport từ hệ nguồn |
| 2 | Phân biệt: SLA cho "data snapshot" hay "pipeline run"? | Nếu dataset là snapshot cũ cố định (lab) → FAIL là hợp lệ, không cần action |
| 3 | Kiểm tra `FRESHNESS_SLA_HOURS` trong `.env` | Có thể điều chỉnh SLA cho phù hợp với bối cảnh |

---

### Mitigation

- **Production**: Trigger ingest lại từ hệ nguồn → rerun `etl_pipeline.py run`.
- **Lab (dataset mẫu cố định)**: Ghi chú rõ trong runbook rằng `exported_at=2026-04-10` là intentional (dataset mẫu), FAIL là expected behavior cho bối cảnh lab. Có thể đổi `FRESHNESS_SLA_HOURS=200` tạm thời để test PASS nếu cần demo.

---

### Prevention

- Automated scheduled refresh (cron / Airflow) rerun pipeline mỗi 24h.
- Event-driven ingest: khi hệ nguồn cập nhật policy → trigger pipeline ngay.
- Đo freshness tại **2 boundary** (ingest timestamp + publish timestamp) để phân biệt "data stale" vs "pipeline stale".
