# Báo cáo Day 09 — Vũ Văn Long, 2A202603312

## 1. Kết luận trước
Pipeline kết luận **Hold** (mã thoát 10). Nhận 224 ảnh, loại 22 ở cửa vào (5 rò rỉ test, 8 trùng hệt, 4 gần trùng, 3 quá nhỏ, 2 hỏng), còn 202 ảnh vào QC và 202 ảnh xuất ra `export/clean_v1` (bản nháp, chưa phát hành). Sau sửa tự động và người duyệt, không còn lỗi bắt buộc (0 lỗi ở v3), nhưng 38/202 ảnh (18,8%) còn cảnh báo chưa người duyệt, vượt ngưỡng 10% của cổng.

## 2. Lỗi và phân việc
Số liệu từ `02_error_list_v1` (lô raw, 224 ảnh quét; mỗi dòng là một lỗi/cảnh báo):

| Mã lỗi | Số dòng | Ai sửa | Vì sao |
|---|---:|---|---|
| LEAK_TEST | 5 | máy (xoá ảnh) | ảnh trùng với tập test, giữ lại làm điểm test cao giả |
| DUP_IMAGE_EXACT / DUP_IMAGE_NEAR | 8 / 4 | máy (xoá ảnh) | ảnh trùng, chỉ giữ một bản |
| BAD_SIZE / CORRUPT_IMAGE | 3 / 2 | máy (xoá ảnh) | ảnh nhỏ hơn 250 px hoặc không mở được |
| BOX_OUT_OF_IMAGE | 8 | máy (cắt box, `clip_box`) | sai tọa độ rõ ràng, cắt về [0,1] an toàn |
| DUP_BOX, BAD_CLASS_ID, BOX_ZERO, BAD_FORMAT | 7, 6, 4, 3 | máy (xoá dòng, `drop_line`) | dòng hỏng, không có cách đoán nhãn đúng |
| BAD_CLASS_NAME | 2 | máy (`rename_classes`) | ghi lại `names` đúng `cfg["classes"]` |
| MISSING_LABEL_FILE | 6 | **người** | máy không bịa nhãn, `03_fixes.csv` ghi `needs_label` |
| POSSIBLE_MISSING / FALSE_POS / LOOSE_BOX / WRONG_CLASS | 53 / 29 / 28 / 18 | **người** | model lớn và nhãn bất đồng, model cũng có thể sai |
| BOX_TINY, BOX_HUGE, EMPTY_LABEL, CLASS_RARE | 6, 3, 12, 4 | **người** | cảnh báo, cần nhìn ý nghĩa |

Máy thực hiện 35 thay đổi trong `03_fixes.csv`: 20 `drop_line`, 8 `clip_box`, 6 `needs_label`, 1 `rename_classes`.

Hàng đợi `05_review_queue.csv` có 125 ảnh, xếp theo điểm ưu tiên: ảnh thiếu file nhãn đứng đầu (điểm 10, vì không có nhãn thì không dùng được), tiếp theo là ảnh có nhiều cảnh báo cộng trọng số (WRONG_CLASS 3, MISSING 2, FALSE_POS 2, LOOSE_BOX 1,5). Ví dụ img_0020 đứng đầu với 11,5 điểm vì có cả dòng bị xoá, thiếu box và nghi sai lớp. Ngân sách 80 ảnh chỉ duyệt được 80/125 (64%) ảnh trong hàng đợi, còn 45 ảnh chưa ai xem.

## 3. Calibration (2 ngưỡng)
Số liệu từ `submission/calibration/calibration_report.md` (39 ảnh calib, có đáp án):

| Tham số | Khởi đầu | Đã chọn | Precision / Recall (trước → sau) | Lý do (đánh đổi) |
|---|---|---|---|---|
| `duplicates.phash_dist` | 2 | **12** | DUP_IMAGE_NEAR: 1,00 / 0,50 → 1,00 / 0,67 | Ngưỡng 2 sót một nửa ảnh gần trùng. Từ 4 đến 14 recall giữ 0,67 và precision 1,00, nên chọn 12 là giữa vùng ổn định, cách xa vực ở 20 (recall DUP_IMAGE_EXACT tụt về 0,33). Ngưỡng 16 cho F1 cao nhất (0,91) nhưng đã bị loại vì loại oan trên raw (xem dưới) |
| `second_opinion.conf` | 0.25 | **0.6** | MISSING: 0,50 / 1,00 → 0,57 / 1,00; WRONG_CLASS: 0,56 / 0,83 → 0,62 / 0,83 | Tăng conf làm bớt báo nhầm (precision lên) mà MISSING còn giữ recall 1,00. Từ 0,7 trở đi MISSING mất recall (0,88 rồi 0,75), tức sót ảnh thiếu box. Ngân sách người duyệt chỉ 80 ảnh nên bớt báo nhầm có giá trị |

**Chỗ calibration có thể đánh lừa.** Tập calib chỉ có 39 ảnh và chỉ 6 ca gần trùng (DUP_IMAGE_NEAR), nên ở `phash_dist = 16` recall tăng từ 0,67 lên 0,83 thực chất chỉ là bắt thêm **một** ảnh, và F1 0,91 trông rất đẹp. Tôi chạy thử cả hai ngưỡng trên raw: ngưỡng 16 loại 31/224 ảnh, ngưỡng 12 loại 22/224. Chênh nhau 9 ảnh (3 gần trùng, 6 rò rỉ test). Mở ghép 4 cặp trong 9 ảnh đó thì **3/4 cặp là ảnh hoàn toàn khác nhau** bị loại oan: img_0013 với test_087 (bàn làm việc và sân bóng chày), img_0033 với test_070 (biển và tiệc pizza), img_0011 với img_0008 (tennis và trượt tuyết); chỉ img_0084 với img_0074 là trùng thật. Tức ngưỡng ăn khớp calib nhưng sai trên raw, nên tôi chọn 12. Lưu ý tôi mới xem 4/9 ảnh.

## 4. Cửa chặn và điều pipeline KHÔNG bắt được
**(a) Gate chặn đúng.** Chạy `python -m pipeline.run --budget 0 --reports work/rep_nb --export-dir work/exp_nb` (không có người duyệt, ảnh thiếu nhãn còn nguyên): pipeline báo "còn 6 lỗi bắt buộc chưa sửa -> **Rework**", mã thoát **20**, log ghi "KHÔNG xuất: còn lỗi bắt buộc", và thư mục `work/exp_nb` không được tạo, tức không xuất dataset.

**(b) Điểm mù** (bằng chứng từ calibration):
1. **Box lỏng nhẹ.** `POSSIBLE_LOOSE_BOX` chỉ đạt precision 0,29 và recall 0,40 ở cấu hình khởi đầu, và tụt xuống 0,20 / 0,20 ở conf 0,6 (bỏ sót 4/5 ca chèn). Ở mức conf đã chọn, box lỏng gần như không bị bắt.
2. **Báo nhầm nhiều với lỗi ngữ nghĩa.** `POSSIBLE_FALSE_POS` có precision chỉ 0,31 (5 đúng trên 16 báo), và `POSSIBLE_MISSING` chỉ 0,57, nghĩa là hơn 40% cảnh báo là báo nhầm. Người duyệt tốn ngân sách vào những ảnh không lỗi.
3. **Ảnh trống (EMPTY_LABEL, 12 ảnh)**: máy không biết đó là ảnh nền hợp lệ hay quên gán nhãn, nên chỉ cảnh báo.
4. **Ảnh gần trùng biến đổi mạnh** (cắt, xoay, đổi màu): phash khó bắt, calib cũng chỉ chứa 6 ca.

## 5. Train: raw vs clean (tùy chọn, chấm riêng)
Train trên Colab T4: yolo26n, 20 epoch, 2 seed (0, 1) cho mỗi biến thể, cùng val/test và siêu tham số, trên bản `clean_v1` hiện tại (`phash_dist = 12`, 202 ảnh). Dataset `clean` là **bản nháp** (kết luận Hold), không phải bản phát hành.

| biến thể | #ảnh train | seed | mAP50 | mAP50-95 | precision | recall |
|---|---|---|---|---|---|---|
| raw | 224 | 2 | 0,726 ± 0,000 | 0,512 ± 0,004 | 0,764 ± 0,061 | 0,649 ± 0,028 |
| clean | 202 | 2 | 0,701 ± 0,025 | 0,487 ± 0,008 | 0,759 ± 0,003 | 0,625 ± 0,019 |

Chênh mAP50 (clean − raw) = **−0,025**.

**So với nhiễu.** Chênh −0,025 ở mAP50 xấp xỉ bằng độ lệch chuẩn giữa 2 seed của chính clean (0,025), nên chưa vượt khỏi nhiễu. Chỉ có 2 seed nên độ lệch chuẩn không đáng tin (raw ra ± 0,000 vì hai seed cho cùng kết quả làm tròn), cộng với tập test chỉ khoảng 100 ảnh nên nhiễu thật còn lớn hơn. mAP50-95 cũng thấp hơn 0,025 (độ lệch 0,008) và recall thấp hơn 0,024; precision gần bằng nhau (0,759 so với 0,764). Mọi chỉ số đều nghiêng nhẹ về phía raw, nhưng mức chênh đều nhỏ.

**Kết luận trung thực:** chưa kết luận được clean tốt hơn raw. Kết quả còn nghiêng nhẹ về raw, nhưng nằm trong nhiễu, nên không thể nói clean làm model tệ hơn.

**Nguyên nhân có thể và cách kiểm:**
1. *Clean ít ảnh hơn raw* (202 so với 224, ít hơn 10%): với dataset nhỏ, mất 22 ảnh có thể làm giảm mAP, bất kể chất lượng nhãn. Kiểm: train raw trên tập con ngẫu nhiên 202 ảnh để so công bằng về số lượng.
2. *Clean còn nhãn chưa được người duyệt.* 38/202 ảnh còn cảnh báo, 45 ảnh trong hàng đợi chưa ai xem, nên nhãn nhiễu vẫn còn trong clean. Kiểm: duyệt hết hàng đợi, rồi train lại.
3. *Rò rỉ test của raw làm điểm raw cao hơn.* Raw chứa 5 ảnh trùng với tập test, nên raw được "xem trước" một phần test, đẩy mAP lên. Kiểm: bỏ riêng 5 ảnh này khỏi raw rồi train lại, xem mAP raw giảm bao nhiêu.
4. *Test chỉ 100 ảnh và chỉ 2 seed.* Kiểm: tăng lên ≥ 5 seed, hoặc dùng tập test lớn hơn.

**Việc tiếp theo:** duyệt hết hàng đợi để dataset đủ điều kiện Release, rồi train lại với ≥ 5 seed và đối chứng raw cùng số ảnh.

## 6. Quyết định và ghi chú AI
**(a) Quyết định cuối: Hold.** Không còn lỗi bắt buộc, nhưng 38/202 ảnh (18,8%) còn cảnh báo chưa người duyệt, vượt ngưỡng 10%. Để thành Release cần: duyệt thêm 45 ảnh còn lại trong hàng đợi (ngân sách duyệt hiện 80/125), hoặc nâng ngân sách để tỉ lệ cảnh báo dưới 10% (≤ 20 ảnh). Ngoài ra cần có người xác nhận 6 ảnh thiếu nhãn.

**(b) Dùng AI:** Có. Claude viết 4 module (`label_format`, `gate`, `autofix`, `second_opinion`) theo docstring và test trong repo, và hướng dẫn các bước làm.
- **Prompt chính:** "viết cho t" (kèm file stub và test của từng module).
- **Cách kiểm:** chạy `pytest` (79 test xanh), rồi đối chiếu 3 ảnh đã biết lỗi trong `02_error_list_v1`: img_0020 (BAD_CLASS_ID), img_0001 (DUP_BOX), img_0030 (BOX_OUT_OF_IMAGE) đều được bắt. Số liệu trong báo cáo lấy từ `submission/reports/`.
- **Một lần AI sai:** AI nói "giá trị cuối phải khác giá trị khởi đầu" cho cả 2 ngưỡng. Đọc lại `check_submission.py` thì chỉ báo lỗi khi cả hai cùng còn nguyên (2, 0.25), nên điều kiện chặt hơn thực tế. Giá trị cuối được chọn theo bảng sweep chứ không chỉ để khác giá trị khởi đầu.
