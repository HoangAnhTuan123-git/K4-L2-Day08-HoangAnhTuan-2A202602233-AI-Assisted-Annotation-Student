# Vì sao chọn lô này?

## 1. Top 5 frame ưu tiên nếu chỉ có ngân sách rà 5 ảnh
Nếu ngân sách chỉ đủ để rà 5 ảnh, ta chọn các frame sau từ 50 dòng đứng đầu `outputs/selection_round1.csv`:

1. **`frame_0182.jpg`** (Rank 1, t = 72.8s, Score = 0.9591, U = 0.9182, A = 1.0, D = 1.0, 28 boxes): Có điểm tổng hợp cao nhất toàn pool, độ bất định cao (U=0.918) và số box mơ hồ tối đa (A=1.0).
2. **`frame_0369.jpg`** (Rank 2, t = 147.6s, Score = 0.9324, U = 0.9315, A = 0.8889, D = 1.0, 43 boxes): Mật độ xe cao (43 boxes), độ bất định cao, đại diện cho đoạn video ban đêm đông đúc.
3. **`frame_0099.jpg`** (Rank 8, t = 39.6s, Score = 0.9063, U = 0.9460, A = 0.7778, D = 1.0, 29 boxes): Đại diện cho đoạn đầu video (t=39.6s, cách xa frame_0182 và frame_0369 về mặt thời gian), độ bất định cao (U=0.946).
4. **`frame_0227.jpg`** (Rank 11, t = 90.8s, Score = 0.8915, U = 0.9164, A = 0.7778, D = 1.0, 37 boxes): Nằm ở khoảng giữa t=90.8s, đảm bảo đa dạng thời gian và không bị trùng lặp với các cụm khác.
5. **`frame_0002.jpg`** (Rank 18, t = 0.8s, Score = 0.8658, U = 0.8983, A = 0.7222, D = 1.0, 27 boxes): Đại diện cho những giây đầu tiên của video (t=0.8s), giúp mô hình học bối cảnh ánh sáng và góc nhìn ngay từ đầu chuỗi.

*Quyết định xét ảnh gần trùng:* Bỏ qua `frame_0380.jpg` (t=152.0s), `frame_0372.jpg` (t=148.8s) và `frame_0326.jpg` (t=130.4s) / `frame_0331.jpg` (t=132.4s) vì các frame này có thời gian quá gần với `frame_0369.jpg` (cách nhau dưới vài giây), xe và bối cảnh chuyển động gần như trùng lặp (redundancy cao), làm tốn ngân sách rà nhãn mà không bổ sung nhiều thông tin mới.

## 2. Ba frame thuộc lô 12 ảnh model chọn và bằng chứng
1. **`frame_0182.jpg`** (Score = 0.9591, Rank 1): Điểm cao nhất, chứa nhiều xe bị che khuất và độ tự tin của model dao động mạnh.
2. **`frame_0331.jpg`** (Score = 0.9154, Rank 5, t = 132.4s): Model phát hiện 20 box nhưng sau khi người rà thì có tới 36 box (thêm 21 box FN, xóa 5 box FP do nhiễu đèn), cho thấy độ mơ hồ A=1.0 rất chính xác.
3. **`frame_0099.jpg`** (Score = 0.9063, Rank 8, t = 39.6s): Bằng chứng trên contact sheet và file diff cho thấy model chỉ đề xuất 13 box nhưng thực tế có 25 box (nhiều xe nhỏ ở xa bị bỏ sót).

## 3. Một frame có điểm cao nhưng không chọn hoặc điểm thấp vẫn nên xem
- **`frame_0372.jpg`** (Rank 6, Score = 0.9101, t = 148.8s) có điểm rất cao nhưng **không được chọn** vào lô 12 ảnh vì cơ chế `MIN_GAP_S` đã kích hoạt do trước đó đã chọn `frame_0369.jpg` (t = 147.6s, chênh lệch chỉ 1.2s < ngưỡng khoảng cách tối thiểu). Việc loại trừ này giúp tránh lãng phí chi phí gán nhãn cho các ảnh gần như nhân bản.
- Hoặc một frame có điểm thấp hơn như `frame_0002.jpg` (t = 0.8s) vẫn nên xem vì mang đặc trưng bối cảnh khởi đầu video mà các frame điểm cao ở giữa video không có.

## 4. Điều phép chọn này chưa chứng minh về chất lượng mô hình
- Phép chọn dựa trên độ bất định (Uncertainty) và tính đa dạng (Diversity) chỉ chỉ ra các mẫu mà **mô hình hiện tại đang phân vân nhất hoặc thiếu thông tin nhất**, chứ **không bảo đảm 100% rằng học trên các mẫu này sẽ tăng hiệu năng trên mọi tập dữ liệu**.
- Nếu các mẫu được chọn chứa quá nhiều nhiễu bất khả kháng (như quá tối, chói đèn nặng, xe ngoài tầm nhìn rõ ràng của nhãn tham chiếu), việc gán nhãn có thể gây mâu thuẫn nhãn (label noise) và không giúp cải thiện chất lượng mô hình thực tế.
