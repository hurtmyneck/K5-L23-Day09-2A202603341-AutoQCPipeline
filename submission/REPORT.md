# Báo cáo Day 09 — Dương Trí Dũng, 2A202603341

## 1. Kết luận trước
Pipeline kết luận **Hold**. Vào 224 ảnh, ra dataset `clean_v1` 202 ảnh (loại 22 ở cửa vào: 8 trùng byte, 5 rò rỉ test, 4 trùng gần đúng, 3 quá nhỏ, 2 hỏng). Không còn lỗi bắt buộc, nhưng **29/202 ảnh (14,4%)** còn cảnh báo chưa được người duyệt, vượt ngưỡng 10% của cổng. Hàng đợi duyệt có 118 ảnh mà ngân sách chỉ 80. `clean_v1` vì vậy là bản nháp, chưa phát hành.

## 2. Lỗi và phân việc
Số liệu từ `02_error_list_v1` (số ảnh; số dòng ghi riêng khi khác).

| Mã lỗi | Số ảnh/dòng | Ai sửa | Vì sao |
|---|---:|---|---|
| `DUP_IMAGE_EXACT` / `DUP_IMAGE_NEAR` | 8 / 4 | máy (loại ở cửa vào) | trùng là quy tắc đo được (md5, pHash) |
| `LEAK_TEST` | 5 | máy (loại ở cửa vào) | trùng ảnh kho test/val: phải bỏ để không làm hỏng phép đo |
| `BAD_SIZE` / `CORRUPT_IMAGE` | 3 / 2 | máy (loại ở cửa vào) | ảnh nhỏ hơn 250px hoặc không mở được |
| `BAD_FORMAT`, `BAD_CLASS_ID`, `BOX_ZERO`, `DUP_BOX` | 3 / 6 / 4 / 7 | máy (`drop_line`, 20 dòng) | dòng hỏng về cú pháp, xoá là an toàn |
| `BOX_OUT_OF_IMAGE` | 8 | máy (`clip_box`, 8 dòng) | cắt box về trong ảnh, giữ nguyên lớp |
| `BAD_CLASS_NAME` | 1 file (2 dòng) | máy (`rename_classes`) | ghi lại `names` đúng `cfg["classes"]` |
| `MISSING_LABEL_FILE` | 6 | **người** (`needs_label`) | máy không được bịa nhãn |
| `POSSIBLE_MISSING` / `POSSIBLE_WRONG_CLASS` | 43 / 16 | **người** | lỗi ngữ nghĩa, model chỉ nghi ngờ |
| `POSSIBLE_LOOSE_BOX` / `POSSIBLE_FALSE_POS` | 21 / 27 | **người** | box lỏng, box thừa: cần nhìn ảnh |
| `EMPTY_LABEL`, `BOX_TINY`, `BOX_HUGE` | 12 / 4 / 3 | **người** | "ảnh nền" hay thiếu nhãn là việc của người |

**Hàng đợi `05_review_queue.csv`** xếp theo điểm ưu tiên cộng từ trọng số của từng mã. Đầu hàng là `img_0020` (11,5 điểm: có cả `FIXED_DROP_LINE`, `POSSIBLE_MISSING`, `POSSIBLE_WRONG_CLASS`), kế đó là 6 ảnh `NEEDS_LABEL` (10 điểm), vì thiếu nhãn là lỗi nặng nhất. Trong 80 ảnh đầu, người xem đủ **43/43** ảnh `POSSIBLE_MISSING` và **16/16** ảnh `POSSIBLE_WRONG_CLASS`. Họ chỉ xem 13/21 ảnh `LOOSE_BOX` và 11/23 ảnh `FALSE_POS`. Còn 38 ảnh ngoài ngân sách, chủ yếu mang cảnh báo nhẹ.

## 3. Calibration (2 ngưỡng)
| Tham số | Khởi đầu | Đã chọn | Precision / Recall (trước → sau) | Lý do (đánh đổi) |
|---|---|---|---|---|
| `duplicates.phash_dist` | 2 | **10** | `DUP_IMAGE_NEAR`: 1,00 / 0,50 → 1,00 / 0,67 (`DUP_IMAGE_EXACT` giữ 1,00 / 1,00) | Recall `NEAR` đi ngang ở 4–14, 10 nằm giữa vùng ổn định. Precision vẫn 1,00. Chọn 16 (recall 0,83) bị loại, xem dưới. |
| `second_opinion.conf` | 0.25 | **0.7** | `MISSING`: 0,50 / 1,00 → 0,58 / 0,88; `WRONG_CLASS`: 0,56 / 0,83 → 0,62 / 0,83; `LOOSE_BOX`: 0,29 / 0,40 → 0,50 / 0,20 | Tăng conf làm precision tốt hơn và hàng đợi nhẹ hơn, nhưng recall `MISSING` giảm (sót 1/8 ảnh thiếu nhãn trên calib) và recall `LOOSE_BOX` giảm còn 0,20. Chọn 0,7 thay vì 0,8 để giữ recall `MISSING`. Ở 0,8 pipeline ra Release (7,9%) nhưng một phần là do máy ít nghi hơn, không phải dữ liệu sạch hơn. |

**Chỗ calibration có thể đánh lừa.** Bảng sweep trên 39 ảnh calib cho `phash_dist = 16` recall `DUP_IMAGE_NEAR` cao nhất (0,83, F1 0,91), nên rất dễ bị chọn vì "điểm cao nhất". Nhưng calib chỉ có 6 cặp trùng gần đúng, và 16 nằm sát vách: ở 20, recall `DUP_IMAGE_EXACT` rơi từ 1,00 xuống 0,33. Kiểm trên lô raw thật: từ 10 lên 16, số ảnh bị loại ở bước 1 tăng từ 22 lên 31 (thêm 3 ảnh `DUP_IMAGE_NEAR` và 6 ảnh `LEAK_TEST`). Mở xem các cặp trùng gần đúng:
- Khoảng cách 6 (`img_0077`–`img_0223`) và 10 (`img_0061`–`img_0186`): cùng một cảnh, chỉ khác độ sáng hoặc khung cắt. Trùng thật.
- Khoảng cách 16, 3 cặp thêm: `img_0008`–`img_0011` (người trượt tuyết và vận động viên tennis) và `img_0113`–`img_0177` (người đàn ông ngồi bàn và cô gái lướt sóng) là **hai ảnh khác hẳn nhau**. Chỉ `img_0074`–`img_0084` là trùng thật. Tức 2/3 ảnh bị loại thêm là loại nhầm.

Vì vậy chọn 10. Calib nhỏ có thể cho đúng một cặp tốt hơn khi đổi ngưỡng, và điều đó không đáng đổi lấy việc mất ảnh khác nhau. Với 6 ảnh `LEAK_TEST` thêm ở ngưỡng 16, không mở xem (không dùng ảnh `data/test/` để chọn ngưỡng), nên không kết luận chúng đúng hay sai.

## 4. Cửa chặn và điều pipeline KHÔNG bắt được
**(a) Gate chặn đúng.** Chạy `python -m pipeline.run --budget 0 --reports work/rep_nb --export-dir work/exp_nb` (không có người duyệt, 6 ảnh thiếu file nhãn còn nguyên): kết luận **Rework** ("còn 6 lỗi bắt buộc chưa sửa"), mã thoát **20**, và pipeline in "KHÔNG xuất": thư mục `work/exp_nb` không được tạo.

**(b) Điểm mù, có bằng chứng:**
1. **Box lỏng nhẹ.** `POSSIBLE_LOOSE_BOX` ở `conf = 0.7` chỉ có recall 0,20 trên calib (bắt 1/5 ca chèn). Box lệch vừa phải vẫn có IoU ≥ 0,5 với dự đoán model nên không bị cờ.
2. **Ảnh trùng biến đổi mạnh.** `DUP_IMAGE_NEAR` chỉ đạt recall 0,67 ở `phash_dist = 10` (sót 2/6 cặp trên calib). pHash không bắt được ảnh bị cắt hoặc đổi nhiều.
3. **Nhãn rỗng không phân biệt được "ảnh nền" và "thiếu nhãn".** `img_0004` có file nhãn rỗng, trong khi ảnh có tàu hỏa và xe ô tô (nhìn bằng mắt qua `viz_labels.py`). Máy chỉ gắn `EMPTY_LABEL` (cảnh báo, 12 ảnh), phải có người xem mới biết.
4. **`POSSIBLE_FALSE_POS` ồn.** Precision chỉ 0,31 trên calib (11 FP so với 5 TP) và không phụ thuộc `conf`. Đây là mã cảnh báo nhiều nhất còn lại ở `07_error_list_v3` (21 ảnh).

## 5. Train: raw vs clean
Thiết lập: `yolo26n`, 20 epoch, 2 seed (0, 1), trên Colab (GPU). Cùng val `gold_val` (40 ảnh) và cùng test (100 ảnh).

| biến thể | #ảnh train | số seed | map50 (TB ± độ lệch) | map50_95 (TB ± độ lệch) |
|---|---:|---:|---|---|
| raw | 224 | 2 | 0,726 ± 0,000 | 0,512 ± 0,004 |
| clean | 202 | 2 | 0,712 ± 0,012 | 0,489 ± 0,009 |

Chênh mAP50 (`clean − raw`) = **−0,013**, xấp xỉ bằng độ lệch giữa 2 seed của `clean` (0,012). Cả hai lần `clean` (0,7206 và 0,7043) đều thấp hơn cả hai lần `raw` (0,7256 và 0,7259).

**Kết luận:** dữ liệu sau pipeline **không tốt hơn** dữ liệu gốc. Với 2 seed và tập test 100 ảnh, **chưa kết luận được** `clean` kém hơn hay chỉ là nhiễu. Theo lớp, `truck` giảm (0,65 → 0,56) còn `bus` tăng (0,70 → 0,73), nhưng `truck` có rất ít mẫu (40 box) nên dao động mạnh.

Giả thuyết và cách kiểm:
1. **Rò rỉ test làm `raw` được lợi.** `raw` chứa 5 ảnh `LEAK_TEST` trùng ảnh test, nên điểm `raw` có thể bị thổi phồng. Kiểm: train lại `raw` sau khi bỏ 5 ảnh đó.
2. **`clean` còn nhãn máy chưa duyệt, và mất 22 ảnh.** 29 ảnh còn cảnh báo, và dữ liệu giảm 10%. Kiểm: so với biến thể `oracle` (nhãn người), hoặc tăng ngân sách duyệt rồi train lại.
3. **Test chỉ 100 ảnh, 2 seed.** Chênh −0,013 nằm trong tầm nhiễu. Kiểm: thêm seed (≥ 5) để ước lượng độ lệch chuẩn tốt hơn.

## 6. Quyết định và ghi chú AI
**(a) Quyết định cuối: Hold.** Để thành Release, số ảnh còn cảnh báo chưa duyệt phải xuống dưới 10% (≤ 20/202 ảnh, hiện là 29). Cần duyệt thêm ít nhất 9 ảnh trong hàng đợi (ưu tiên 21 ảnh `POSSIBLE_FALSE_POS` còn lại), tức nâng ngân sách duyệt từ 80 lên khoảng 90 trở lên. Cách khác là giảm cảnh báo thừa của `POSSIBLE_FALSE_POS`. Không hạ ngưỡng `conf` chỉ để qua cổng, vì làm vậy chỉ ẩn cảnh báo.

**(b) Dùng AI (Claude Code) để viết 4 module:**
- **Prompt chính:** yêu cầu viết từng module (`label_format`, `gate`, `autofix`, `second_opinion`), bám đúng docstring và test của repo.
- **Cách kiểm:** chạy `pytest` (79 test xanh), chạy `pipeline.run` trên dữ liệu thật rồi đối chiếu số lỗi với ảnh vẽ bằng `viz_labels.py` (ví dụ `img_0001`, `img_0006`), và kiểm quy tắc không đọc `MANIFEST`/`data/test` trong code kiểm.
- **Một lần AI sai:** lần đầu viết `autofix.py`, AI ghi nhầm ký tự xuống dòng `"\n"` thành xuống dòng thật nên file lỗi cú pháp (`SyntaxError`, pytest không chạy được); phát hiện qua pytest và sửa. AI cũng từng đọc nhầm id lớp khi nhận xét `img_0005`.
