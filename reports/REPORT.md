# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Phan Ngọc Hòa

Công cụ gán nhãn đã dùng: CVAT

Sao chép file này thành `reports/REPORT.md` rồi hoàn thiện các mục. Mọi con số phải truy được
từ `reports/rounds_table.md`, `outputs/selection_round1.csv`, `outputs/metrics_round*.json` hoặc
`outputs/round*_diff.md`. Không coi nhãn test do mô hình tạo là chân lý tuyệt đối.

## 1. Dữ liệu và cách chia tập

Tại sao tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian, có vùng
đệm ở giữa, thay vì chia ngẫu nhiên? Nếu chia ngẫu nhiên, số đo trên tập kiểm thử sẽ bị lệch theo
hướng nào, và vì sao?

- Camera đặt cố định trên cầu vượt, xe di chuyển qua khung hình mất nhiều giây, 2 frame liên tiếp gần như y hệt. Nếu chia ngẫu nhiên, cùng một chiếc xe sẽ xuất hiện ở cả tập train và test gây rò rỉ dữ liệu. Vùng đệm 4.4s loại bỏ các frame trung gian giúp hai tập độc lập hoàn toàn.
- Xu hướng lệch: Số đo trên tập test sẽ bị lệch lạc quan, vì mô hình chỉ ghi nhớ các xe cụ thể đã thấy thay vì học cách khái quát hóa sang thời điểm và xe mới.

## 2. Mô hình khởi đầu lạnh (cold start)

Chép dòng vòng 0 từ `rounds_table.md`. Dựa vào `outputs/compare_round0.jpg`, cho biết mô hình khởi
đầu lạnh không khớp nhãn tham chiếu ở những loại xe nào. Độ phủ (recall) theo kích thước xe cho
thấy điều gì? Một trường hợp nào cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai?

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

- **Không khớp nhãn tham chiếu:** Mô hình khởi đầu lạnh bỏ sót nhiều xe ở xa, xe tối màu, xe bị che khuất và xe làn ngược chiều bị đèn pha chói (FN = 206 box).
- **Recall theo kích thước:** $R_{small} = 0.182$, $R_{medium} = 0.547$, $R_{large} = 0.561$. Cho thấy mô hình nhận diện tốt xe vừa và lớn, nhưng bỏ sót hơn 80% xe nhỏ ở xa.
- **Trường hợp cần rà lại nhãn tham chiếu:** Xe ở sát đường chân trời chỉ thấy đốm sáng mờ hoặc vùng sáng phản chiếu trên mặt đường; do nhãn tham chiếu sinh tự động bởi AI chưa được người duyệt, có thể chính nhãn tham chiếu bị box ảo (false positive).

## 3. Chiến lược chọn mẫu

Giải thích bằng lời công thức `score = W_U·U + W_A·A + W_D·D` và vai trò của `MIN_GAP_S`.
Dẫn ba frame trong `reports/SELECTION.md` và một frame khác để chứng minh cách bạn cân nhắc
độ bất định, ảnh gần trùng và công gán nhãn. Điểm bất định có chứng minh ảnh đó sẽ cải thiện
mô hình không? Vì sao?

- **Giải thích công thức:**
  - $U$: Độ bất định trung bình của 5 box khó nhất (conf gần 0.5).
  - $A$: Mật độ box mập mờ ($0.15 \le conf < 0.50$) chuẩn hoá theo max pool.
  - $D$: Khoảng cách thời gian tới frame đã gán gần nhất (chia cho mốc chặn 10s) để tăng tính đa dạng.
  - `MIN_GAP_S`: Khoảng cách thời gian tối thiểu giữa 2 frame được chọn để loại bỏ các ảnh gần trùng lặp.
- **Dẫn chứng 4 frame:**
  - 3 frame được chọn: `frame_0182.jpg` (Rank 1, score 0.9591, A=1.0, 18 box mập mờ), `frame_0369.jpg` (Rank 2, 43 box phát hiện, mật độ cao), `frame_0099.jpg` (Rank 8, U=0.9460, đại diện dải đầu video).
  - 1 frame bị loại: `frame_0372.jpg` (Rank 6, score 0.9101) bị loại vì cách `frame_0369.jpg` chỉ 1.2s ($< MIN\_GAP\_S$); bỏ qua để tránh trùng cảnh và tiết kiệm công gán.
- **Điểm bất định có chứng minh cải thiện mô hình không:** **Không.** Bất định chỉ đo độ phân vân của mô hình, có thể do nhiễu, chói đèn pha hoặc xe quá nhỏ dưới 16px; gán nhãn những ảnh này chưa chắc giúp mô hình tăng AP50 trên tập test.

## 4. Các vòng học chủ động (active learning)

Chép bảng từ `rounds_table.md`. Với mỗi vòng, trình bày:

- mức độ bạn đã sửa nhãn gợi ý (số box giữ nguyên, chỉnh sửa, xoá, thêm mới, lấy từ
  `outputs/round*_diff.md`);
- AP50 thay đổi bao nhiêu so với khởi đầu lạnh và so với vòng trước;
- nhóm xe nào tốt lên hoặc xấu đi theo số đo trên cùng tập test.

Dựa vào các ảnh `compare_round*.jpg`, chỉ ra một ca kết quả đổi sau fine-tune (tốt hơn hoặc xấu
đi), cùng lý do có thể kiểm. Dùng `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `round1_diff.md` phân biệt
quan sát độc lập, lỗi pre-label đã sửa và kết quả mô hình sau train. Mô tả một ca khó theo guideline.

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 282 | 0.363 | -0.408 | 1.000 | 0.055 | 0.103 | 0.000 | 0.034 | 0.293 |

- **Mức độ sửa pre-label (từ `outputs/round1_diff.md`):** 12 ảnh; AI gợi ý 169 box, sau khi sửa thành 282 box: `accepted` 141 (83%), `edited` 12, `deleted` 16 (FP của AI), `added` 129 (FN của AI).
- **Biến thiên AP50:** AP50 giảm từ 0.771 xuống 0.363 ($\Delta = -0.408$). Precision đạt 1.000 (0 FP) nhưng Recall giảm mạnh từ 0.489 xuống 0.055.
- **Nhóm xe tốt/xấu:** Tất cả nhóm xe đều giảm recall trên tập test ($R_{small}: 0.182 \to 0$, $R_{medium}: 0.547 \to 0.034$, $R_{large}: 0.561 \to 0.293$). Mô hình trở nên cực kỳ dè dặt, chỉ dự đoán 22 box rõ nhất.
- **Ca đổi sau fine-tune:** Trên `outputs/compare_round1.jpg`, mô hình vòng 1 triệt tiêu được các box ảo ở dải phân cách nhưng bỏ sót hầu hết xe ở xa do fine-tune 50 epochs trên 12 ảnh bị overfitting và quên đặc trưng pretrain COCO.
- **Đối chiếu 3 nguồn:** `BLIND_SCAN.md` (`frame_0099.jpg`) thấy 22 xe; `round1_diff.md` và `REVIEW_LOG.csv` ghi nhận AI chỉ đoán 13 box, người gán sửa thành 21 box sạch; sau train mô hình co cụm dự đoán do tập train quá nhỏ.
- **Ca khó theo guideline:** Tại `frame_0312.jpg`, AI gán box khổng lồ trùm vệt đèn chói và 3 xe; đã xóa (`deleted`) và vẽ lại từng box riêng sát thân xe theo quy tắc.

## 5. Kết luận và giới hạn

Kết quả vòng này so với cold start ra sao? Vì sao bạn dừng hoặc tiếp tục? Đề xuất hai ca còn yếu
hoặc bất định cho vòng sau, kèm chi phí rà nhãn và nguy cơ ảnh gần trùng. Tập kiểm thử chỉ 20 ảnh,
có luật bỏ qua xe quá nhỏ và nhãn tham chiếu do mô hình tạo chưa được rà thủ công; các giới hạn đó
ảnh hưởng thế nào đến kết luận? Nếu AP50 giảm, bạn sẽ kiểm tra điều gì trước khi train thêm?

- **So với cold start:** AP50 giảm sâu từ 0.771 về 0.363; model quá cẩn trọng (P=1.0, R=0.055) do học 12 ảnh mà không đóng băng backbone dẫn đến overfitting.
- **Quyết định dừng/tiếp tục:** Tạm dừng, không làm vòng 2; cần tinh chỉnh tham số huấn luyện (giảm learning rate, đóng băng backbone) trước khi tiếp tục gán nhãn.
- **Đề xuất 2 ca vòng sau:** `frame_0270.jpg` ($t=108.0s$, 35 box, bổ sung xe tải) và `frame_0187.jpg` ($t=74.8s$, 39 box, khúc cua). Cả hai đều đông xe nên tốn công rà; frame 187 cách frame 182 (vòng 1) chỉ 2.0s nên có nguy cơ trùng lặp cảnh, cần giãn cách.
- **Giới hạn:** Tập test nhỏ (20 ảnh, 403 box), bỏ qua xe $<16px$ và nhãn tham chiếu do AI tạo chưa kiểm định thủ công khiến số đo chỉ mang tính tương đối.
- **Kiểm tra khi AP50 giảm:** Kiểm tra định dạng nhãn đã pack, kiểm tra learning rate và số epoch, kiểm tra độ lệch phân bố giữa 12 ảnh train và 20 ảnh test.
