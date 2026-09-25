# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét
ảnh gần trùng hoặc trường hợp model không dự đoán được box:
- **Top 5 đề xuất theo ngân sách 5 ảnh:**
  1. `frame_0182.jpg` (Rank 1): Điểm cao nhất toàn tập, độ mập mờ tối đa (A = 1.0, 18 box mập mờ trên 28 box), model rất phân vân.
  2. `frame_0369.jpg` (Rank 2): U và A đều rất cao, 43 box phát hiện, đại diện cho dải thời gian cuối video.
  3. `frame_0380.jpg` (Rank 3): U cao thứ ba, cách frame 369 khoảng 4.4s, dòng xe thay đổi vị trí đáng kể.
  4. `frame_0326.jpg` (Rank 4): Khai thác dải thời gian quanh 130s, mật độ xe dày (39 box), 15 box mập mờ.
  5. `frame_0312.jpg` (Rank 7): **Quyết định xét trùng lặp:** Bỏ qua Rank 5 (`frame_0331.jpg`, t = 132.4s) vì cách Rank 4 chỉ 2.0s (quá gần, cảnh gần trùng). Bỏ qua Rank 6 (`frame_0372.jpg`, t = 148.8s) vì cách Rank 2 chỉ 1.2s. Chọn Rank 7 cách xa hơn (t = 124.8s), có A = 1.0 (18 box mập mờ), tăng tính đa dạng cảnh và tiết kiệm chi phí gán.

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet:
1. `frame_0182.jpg` (Rank 1, t = 72.8s, score = 0.9591): `selected = True`. Contact sheet cho thấy xe chạy thành hàng dài, đèn pha gây lóa nhiều xe phía xa, model dao động quanh ngưỡng 0.5.
2. `frame_0369.jpg` (Rank 2, t = 147.6s, score = 0.9324): `selected = True`. 43 box với 16 box mập mờ; contact sheet thể hiện mật độ giao thông cao, nhiều xe bị che khuất một phần.
3. `frame_0099.jpg` (Rank 8, t = 39.6s, score = 0.9063, U = 0.9460): `selected = True`. Đại diện cho dải thời gian đầu video (t < 50s), có U rất cao (0.9460) chứng minh model phân vân nặng ở các xe ngược chiều.

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do:
- **`frame_0372.jpg` (Rank 6, score = 0.9101, U = 0.9202, A = 0.8333):** Điểm nằm trong top 6 nhưng `selected = False`. Lý do: mốc thời gian t = 148.8s chỉ cách `frame_0369.jpg` (Rank 2, t = 147.6s) đúng 1.2s (< `MIN_GAP_S`). Hai frame liên tiếp chứa các đối tượng gần như y hệt. Loại bỏ giúp tránh lãng phí ngân sách gán nhãn mà không làm mất thông tin học.

Điều phép chọn này chưa chứng minh về chất lượng mô hình:
- Điểm bất định (`score`) cao chỉ chứng minh model đang thiếu tin cậy hoặc dao động mạnh tại các frame đó; nó không đảm bảo việc gán nhãn và train thêm trên các frame này sẽ làm tăng AP50 trên tập test. Nếu sự bất định bắt nguồn từ nhiễu (vệt đèn pha phản chiếu, sương mù, xe quá xa <16px bị test set lọc bỏ), việc học thêm các frame này không cải thiện hiệu năng thực tế.
