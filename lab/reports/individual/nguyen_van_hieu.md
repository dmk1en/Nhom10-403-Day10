# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Nguyen Van Hieu  
**Vai trò:** Cleaning & Quality Owner  
**Ngày nộp:** 2026-04-15  
**Độ dài yêu cầu:** **400–650 từ** (ngắn hơn Day 09 vì rubric slide cá nhân ~10% — vẫn phải đủ bằng chứng)

---

> Viết **"tôi"**, đính kèm **run_id**, **tên file**, **đoạn log** hoặc **dòng CSV** thật.  
> Nếu làm phần clean/expectation: nêu **một số liệu thay đổi** (vd `quarantine_records`, `hits_forbidden`, `top1_doc_expected`) khớp bảng `metric_impact` của nhóm.  
> Lưu: `reports/individual/[ten_ban].md`

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

Tôi phụ trách `transform/cleaning_rules.py` (các rule clean data từ raw CSV) và `quality/expectations.py` (suite expectation để validate cleaned data). Ngoài ra, tôi quản lý logic quarantine trong `clean_rows()` để tách record lỗi ra `artifacts/quarantine/`.

**Kết nối với thành viên khác:**

Tôi phối hợp với Embed Owner để đảm bảo cleaned data được embed đúng (upsert chunk_id), và với Ingestion Owner để verify raw input format. Trong Sprint 3, tôi cung cấp data "corrupted" cho Embed Owner test eval before/after.

**Bằng chứng (commit / comment trong code):**

Trong `cleaning_rules.py`, tôi thêm comment `# Rule 7: quarantine missing exported_at` và `# Rule 8: quarantine short chunk <20 chars`. Trong `expectations.py`, tôi implement `E7 all_doc_ids_in_allowlist` với severity="halt" để prevent unknown doc_id.

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Tôi quyết định set severity="halt" cho expectation `refund_no_stale_14d_window` thay vì "warn". Lý do: refund policy là critical business rule — nếu chunk chứa "14 ngày" stale lọt vào vector store, agent có thể trả lời sai và gây mất lòng tin khách hàng. Halt đảm bảo pipeline dừng ngay khi detect violation, buộc team fix trước khi publish.

Trong Sprint 3, khi inject `--no-refund-fix`, expectation FAIL và halt pipeline. Nhưng tôi thêm flag `--skip-validate` để demo corruption mà không break flow. Production sẽ không có flag này — expectation halt là guardrail quan trọng.

Quyết định này tăng `quarantine_records` từ 4 lên 6 trong Sprint 2 (thêm 2 records từ rule mới), chứng minh impact đo được. Nếu chỉ warn, pipeline vẫn chạy và risk agent sai.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

Trong Sprint 3, tôi inject corruption bằng `python etl_pipeline.py run --run-id inject-bad-sprint3 --no-refund-fix --skip-validate`. Triệu chứng: expectation `refund_no_stale_14d_window` FAIL với violations=1, log ghi `WARN: expectation failed but --skip-validate → continue embed`.

Metric phát hiện: log expectation FAIL + `hits_forbidden=yes` trong eval CSV (phát hiện chunk stale trong top-k).

Fix: Xác nhận `--no-refund-fix` bỏ apply rule fix 14→7, dẫn đến chunk "14 ngày" vẫn embed. Để demo, tôi dùng `--skip-validate` bypass halt. Production fix: chạy lại pipeline chuẩn (không flag) → expectation PASS, `hits_forbidden=no`.

Anomaly này chứng minh expectation suite hoạt động — detect stale data trước khi agent sử dụng. Nếu không halt, vector store bị poison và retrieval quality giảm.

---

## 4. Bằng chứng trước / sau (80–120 từ)

**Before (run_id=sprint2, data sạch):**
```
q_refund_window,Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền kể từ khi xác nhận đơn?,policy_refund_v4,Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng.,yes,no,,3
```
`contains_expected=yes`, `hits_forbidden=no` — không phát hiện chunk stale.

**After (run_id=inject-bad-sprint3, data hỏng):**
```
q_refund_window,Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền kể từ khi xác nhận đơn?,policy_refund_v4,Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng.,yes,yes,,3
```
`contains_expected=yes`, `hits_forbidden=yes` — phát hiện chunk stale trong top-k!

Sự khác biệt chứng minh corruption ảnh hưởng retrieval: dù top-1 đúng, nhưng context có chunk sai → agent risk trả lời "14 ngày".

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ implement expectation mới `E9: duplicate_chunk_text_normalized` với severity="halt" để detect duplicate sau normalize lowercase + strip. Hiện tại baseline chỉ check exact duplicate, nhưng nếu chunk có typo nhỏ (vd "nhân viên" vs "nhân  viên"), vẫn lọt. Expectation này sẽ tăng `quarantine_records` và đảm bảo vector store pure hơn, giảm noise trong retrieval.

