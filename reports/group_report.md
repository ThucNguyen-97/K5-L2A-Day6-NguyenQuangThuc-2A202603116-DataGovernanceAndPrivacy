# Group Report — Day 06 Data Lab

## 00 · Header

| Mục | Giá trị |
|---|---|
| Group ID | G029 |
| Challenge | T02 · Raw QC Triage |
| Git URL | https://github.com/ThucNguyen-97/K5-L2A-Day6-NguyenQuangThuc-2A202603116-DataGovernanceAndPrivacy |
| Thành viên | Nguyễn Quang Thức — Lead/Integrator; Nguyễn Quốc Thắng — Data & QA |
| Seed | 42 |
| Chế độ chạy | Offline; không đọc `gt/`; không gọi network |
| Code commit | `5f86be6` — `feat: implement T02 raw QC pipeline` |
| Review/output commit | `8134ebd` — `chore: record final T02 review run` |

## 01 · Problem statement

T02 xếp hạng các ảnh raw có nguy cơ lỗi chất lượng để reviewer kiểm tra trước. Các lỗi mục tiêu là blur, dark, glare và compression. Ảnh không sử dụng được cho mục đích dataset được đánh dấu reject/RECAPTURE; ảnh vẫn dùng được được approve/KEEP. Pipeline không tự xóa ảnh gốc.

Input là `fixtures/t02_raw_qc/images.csv` và 96 ảnh trong `fixtures/t02_raw_qc/images/`. Output prediction là `out/poc/ranked_review.csv` với `image_id,score`, trong đó score cao hơn nghĩa là nghi lỗi hơn.

Decision policy của PoC: nếu queue có ứng viên score ≥ 0.35 thì batch là `HOLD` để human review; nếu không thì `GO`. Owner là Nguyễn Quang Thức.

## 02 · Data & provenance

| Split | N | Nguồn | License | Synthetic |
|---|---:|---|---|---|
| dev | 56 | fixture offline của lab | CC0 theo README | Có |
| eval | 40 | fixture offline của lab | CC0 theo README | Có |

Tổng cộng 96 ảnh, 24 cảnh gốc; camera gồm `cam_A` (64) và `cam_B` (32); light gồm day (56), dusk (20), night (20). Manifest được kiểm tra bằng preflight và các hash fixture đều match. Pipeline không đọc `fixtures/t02_raw_qc/gt/`.

## 03 · Research & design

Data flow:

```text
images.csv + images/*.jpg
        ↓ load + manifest verification
blur / brightness / dark fraction / saturated fraction
        ↓ process: calibrated risk score
top-B ranking, B=20
        ↓ contact_sheet.html + reviewer CSV
review decisions + audit log
        ↓ decision.json
GO / HOLD
```

PoC dùng `laplacian_variance` làm proxy sharpness, `brightness_stats` cho mean brightness, dark fraction và saturated fraction. Score là tổ hợp deterministic: blur risk 0.55, dark risk 0.25 và glare risk 0.20; calibration dùng phân phối dev không nhãn. Không dùng pretrained model, không download và không có model license cần khai báo.

Trade-off: feature CPU nhanh và dễ tái lập nhưng có thể báo nhầm ảnh night hữu ích, bỏ sót glare cục bộ và nhạy với camera có quang học mềm hơn. Đây là lý do top-B vẫn cần human review.

## 04 · Baseline vs PoC

Baseline là random seeded baseline trong `src/examples/random_baseline.py`, chạy cùng pool 96 ảnh, budget 20 và seed 42. PoC là `src.pipeline`, chạy cùng input, budget và seed. Cả hai đều sinh prediction đúng hai cột `image_id,score`.

Metric chính thức chưa thể điền vì ground truth `gt/labels.csv` nằm trong overlay riêng của giảng viên và không có trong repo học viên. Không tự tạo hoặc tự điền số metric. Khi có overlay, chạy:

```powershell
.venv\Scripts\python.exe -m src.evaluate --track t02 --gt fixtures/t02_raw_qc/gt/labels.csv --pred out/baseline/ranked_review.csv --run-name baseline --out reports/metrics_baseline.json
.venv\Scripts\python.exe -m src.evaluate --track t02 --gt fixtures/t02_raw_qc/gt/labels.csv --pred out/poc/ranked_review.csv --run-name poc --out reports/metrics_poc.json
.venv\Scripts\python.exe -m src.compare reports/metrics_baseline.json reports/metrics_poc.json --out reports/table1_baseline_vs_poc.md
```

## 05 · Evaluation protocol

Primary metrics là Defect Recall@B và Precision@B với B=20 trên split eval. Recall@20 là số ảnh lỗi thật trong top 20 chia cho tổng số ảnh lỗi thật; Precision@20 là số ảnh lỗi thật trong top 20 chia cho 20. Budget, seed và input pool được giữ cố định giữa baseline và PoC. Threshold/risk weights được chốt trong code trước khi có GT eval.

Metric report chính thức: **pending instructor GT overlay**. Không tuyên bố PoC đạt metric khi chưa có GT.

## 06 · Human review & decision

`out/poc/contact_sheet.html` hiển thị top 20 ảnh, score và reason. Reviewer Nguyễn Quốc Thắng (`Thang-03158`) đã review đủ 20/20 ảnh: 17 reject và 3 approve. Kết quả được lưu ở `out/poc/review_queue_reviewed.csv`.

Review decisions được đọc lại khi chạy pipeline; các thay đổi được append vào `out/poc/audit_log.jsonl`. Pipeline có hỗ trợ override score khi reviewer cần chỉnh, nhưng reviewer không sửa score trong run cuối nên không phát sinh `override_value`; nhóm không tạo override giả và không đánh giá override riêng. Decision cuối là `HOLD` với reason code `QC_REVIEW_REQUIRED` và evidence là review queue/prediction.

### 3. Failure cases thực tế

Hai trường hợp được chọn từ queue T02 cho thấy sự khác nhau giữa tín hiệu tự động và quyết định của reviewer:

| Ảnh | Tín hiệu pipeline | Reviewer | Nhận xét |
|---|---|---|---|
| `img_07_3` | Score `0.55`, reason `blur`; sharpness `4.6155` | Reject | Ảnh quá mờ, không thấy rõ vật thể. Đây là trường hợp pipeline phát hiện đúng lỗi blur. |
| `img_11_3` | Score `0.55`, reason `blur`; sharpness `4.7961` | Approve | Vẫn nhìn thấy vật thể dù chưa rõ ràng. Đây là trường hợp borderline, reviewer giữ lại để tránh loại nhầm. |

Kết luận: score và reason giúp ưu tiên ảnh cần kiểm tra, nhưng chưa đủ để tự động loại toàn bộ ảnh. Human review vẫn cần thiết cho các trường hợp mờ ở mức ranh giới.

## 07 · Reproduce & tests

```powershell
.venv\Scripts\python.exe -m src.main --config configs/lab.yaml --track t02 --output out/poc --run-name poc --offline --seed 42
.venv\Scripts\python.exe -m pytest -q
```

Run cuối trả `status=0`, xử lý 96 ảnh, seed 42, offline true và reviewer count 20. Test nhóm gồm:

- `test_t02_process_scores_images`: success path, score hợp lệ 0–1 và đủ cột.
- `test_t02_process_fails_on_missing_image`: failure/edge path, báo `FileNotFoundError` khi ảnh thiếu.

Preflight trả `0 FAIL`; pytest pass toàn bộ. Hai warning về `data/real` và `models_optional` là tùy chọn, không ảnh hưởng T02.

## 08 · Limitations & ethics

Fixture là synthetic và không chứa PII thật. Pipeline không nên được diễn giải là chứng nhận GDPR hoặc tuân thủ pháp luật. False alarm có thể tăng ở night/cam_B; false negative có thể xảy ra với glare cục bộ. Nếu scale 10× cần đo latency, kiểm tra drift theo camera/light và bổ sung nhãn reviewer.

Cải tiến tiếp theo: calibrate theo camera/light trên dev, thêm glare ROI, bổ sung compression feature và đánh giá bằng GT hidden trên eval.

## 09 · Team contribution

| Thành viên | Vai trò | Bằng chứng |
|---|---|---|
| Nguyễn Quang Thức — `nguyenquangthuc.info@gmail.com` | Lead/Integrator; pipeline; report | `5f86be6`, `8134ebd` |
| Nguyễn Quốc Thắng — `quocthang9112@gmail.com` | Data & QA; review queue; output review | `a79e698`, `89d1159`, `24c1250` |

Đối chiếu thêm bằng:

```powershell
git shortlog -sne --all
```

## 10 · Checklist

- [x] Repo, group ID G029, track T02, owner và seed 42.
- [x] Baseline và PoC output.
- [x] Pipeline offline, không đọc GT, không gọi network.
- [x] Contact sheet và human review đủ 20 ảnh.
- [x] Audit log và decision output.
- [x] Một test success và một test failure/edge.
- [ ] Metrics baseline vs PoC — chờ instructor GT overlay.
- [x] Hai thành viên có commit thực chất và được ghi trong contributions.
