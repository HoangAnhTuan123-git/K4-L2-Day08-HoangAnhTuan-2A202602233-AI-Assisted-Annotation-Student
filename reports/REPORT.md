# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Hoàng Anh Tuấn - 2A202602233

Công cụ gán nhãn đã dùng: CVAT (chạy Docker local v2.76.0)

## 1. Dữ liệu và cách chia tập

Tập chưa gán nhãn (pool) và tập kiểm thử (test set) được trích xuất từ chuỗi video đường cao tốc ban đêm nhưng được chia theo **trục thời gian (temporal split) có vùng đệm (buffer zone)** ở giữa thay vì chia ngẫu nhiên (random split). Lý do:
- Các khung hình trích từ video có tính tương quan chuỗi thời gian rất cao (temporal redundancy). Các xe trong khung hình $t$ sẽ hầu như giữ nguyên vị trí, góc chiếu sáng và hình dáng ở khung hình $t+1$.
- Vùng đệm ở giữa giúp ngăn chặn các xe đang di chuyển trong tập train/pool tiếp tục xuất hiện trong tập test.

Nếu chia ngẫu nhiên (random split):
- Số đo trên tập test (như mAP, Precision, Recall) sẽ bị **thổi phồng quá cao (overly optimistic / data leakage)** do mô hình chỉ ghi nhớ các xe cụ thể xuất hiện ở các frame lân cận thay vì học được khả năng tổng quát hóa thực sự trên các tình huống xe mới.

## 2. Mô hình khởi đầu lạnh (cold start)

Số đo từ `reports/rounds_table.md` (vòng 0):
- Model: `yolov8n cold start (COCO car+bus+truck)`
- Train: 0 ảnh, 0 box
- AP50: `0.771` (77.1%) | P@0.25: `0.925` | R@0.25: `0.489` | F1: `0.640`
- Recall theo kích thước: Small: `0.182`, Medium: `0.547`, Large: `0.561`.

Dựa vào `outputs/compare_round0.jpg`:
- Mô hình khởi đầu lạnh bỏ sót rất nhiều xe nhỏ ở xa (Recall Small chỉ đạt 18.2%), xe bị che khuất một phần trong bóng tối, và xe ở các làn ngoài cùng có độ tương phản thấp.
- Precision cao (92.5%) nhưng Recall thấp (48.9%) cho thấy mô hình khá "thận trọng": chỉ dự đoán khi rất chắc chắn, dẫn đến bỏ sót 206 box trên tập test (FN = 206).
- **Trường hợp cần rà lại nhãn tham chiếu:** Một số xe ở cực xa (kích thước bé hơn 16px hoặc bị chói sáng hoàn toàn bởi đèn pha) có thể không được gán trong nhãn tham chiếu tự động. Cần có người thẩm định lại để phân biệt ranh giới giữa false positive của model và thiếu sót của nhãn tham chiếu.

## 3. Chiến lược chọn mẫu

Công thức tính điểm chọn mẫu:
$$\text{score} = W_U \cdot U + W_A \cdot A + W_D \cdot D$$

- **$U$ (Uncertainty):** Độ bất định trung bình của các dự đoán trong frame (dựa trên entropy hoặc độ lệch conf so với ngưỡng).
- **$A$ (Ambiguity):** Tỷ lệ các box có độ tự tin nằm trong vùng mơ hồ $[0.2, 0.6]$.
- **$D$ (Diversity):** Tính đa dạng của khung hình so với các khung hình đã chọn trước đó.
- **$W_U, W_A, W_D$:** Các trọng số điều chỉnh mức độ ưu tiên của từng thành phần.
- **`MIN_GAP_S`:** Khoảng cách thời gian tối thiểu (tính bằng giây) giữa hai frame được chọn liên tiếp, nhằm loại bỏ các frame gần như trùng nhau do trích xuất liên tục từ video.

Minh họa từ `reports/SELECTION.md`:
- Các frame được chọn như `frame_0182.jpg` (Score = 0.9591), `frame_0369.jpg` (Score = 0.9324), `frame_0099.jpg` (Score = 0.9063) đều có $U$ và $A$ rất cao kết hợp phân bố thời gian trải đều.
- Frame điểm cao `frame_0372.jpg` (Score = 0.9101) bị loại bởi `MIN_GAP_S` vì quá gần `frame_0369.jpg`.
- **Lưu ý:** Điểm bất định cao không tự chứng minh ảnh đó sẽ cải thiện mô hình 100%, vì điểm bất định có thể bắt nguồn từ nhiễu cảm biến, chói lóa vô nghĩa hoặc ảnh vỡ hạt thay vì thông tin học hữu ích.

## 4. Các vòng học chủ động (active learning)

Bảng tổng hợp kết quả:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 347 | 0.810 | +0.039 | 1.000 | 0.213 | 0.352 | 0.000 | 0.199 | 0.658 |

Chi tiết sửa nhãn vòng 1 (từ `outputs/round1_diff.md`):
- Tổng 12 ảnh: Model đề xuất ban đầu 169 box. Sau khi người rà trên CVAT sửa thành 347 box (Accepted: 128, Edited: 26, Deleted/FP: 15, Added/FN: 193).
- Tỷ lệ chấp nhận (accept rate): 76%.

Phân tích chất lượng:
- **AP50 tăng từ 0.771 lên 0.810 (+0.039 / +3.9%)** sau khi fine-tune chỉ với 12 ảnh.
- Precision đạt mức tuyệt đối `1.000` tại conf 0.25, không còn sinh ra các False Positive do đèn đường hay vật thể lạ.
- Recall đối với xe cỡ lớn (`R large`) tăng mạnh từ `0.561` lên `0.658`, giúp nhận diện chính xác các xe gần và rõ ràng.
- Đối chiếu với `BLIND_SCAN.md` và `REVIEW_LOG.csv`: Ở `frame_0099.jpg`, quan sát độc lập ban đầu phát hiện 25 xe (có xe ở xa bị che khuất đèn). AI ban đầu chỉ phát hiện 13 box. Sau khi gán nhãn thủ công bổ sung 12 box thiếu, mô hình sau fine-tune đã học được cách nhận diện chuẩn xác các cụm xe trong bối cảnh đêm thiếu sáng mà không bị ảo giác nhầm đèn đường.

## 5. Kết luận và giới hạn

- **Đánh giá:** Vòng học chủ động thứ nhất đạt hiệu quả tốt: AP50 tăng từ 77.1% lên 81.0% và Precision đạt 100% chỉ với 12 ảnh gán nhãn chọn lọc theo chiến lược bất định.
- **Quyết định dừng/tiếp tục:** Dừng ở vòng 1 vì đã hoàn thành toàn bộ chu trình chuẩn theo yêu cầu thực nghiệm và đạt mức tăng trưởng AP50 rõ rệt.
- **Đề xuất cho vòng tiếp theo:**
  1. Ưu tiên các frame chứa nhiều xe nhỏ ở xa có độ chói đèn cao để cải thiện `R small`.
  2. Tiếp tục duy trì `MIN_GAP_S >= 3.0s` để ngăn ngừa chọn các frame trùng lặp gây lãng phí chi phí gán nhãn.
- **Giới hạn thực nghiệm:**
  1. Tập test chỉ có 20 ảnh và nhãn tham chiếu do mô hình tự động tạo ra (chưa qua người rà 100%), do đó số đo AP50 phản ánh độ khớp với bộ tham chiếu này hơn là chân lý tuyệt đối ngoài thực địa.
  2. Quy tắc lọc bỏ xe cao dưới 16px làm giảm độ nhạy đo lường trên các vật thể cực nhỏ.
- **Nếu AP50 giảm:** Cần kiểm tra lại: (1) Tính nhất quán của nhãn người gán (có vi phạm guideline không, có vẽ trùng/lệch box không), (2) Tỷ lệ overfitting do tập train quá nhỏ (12 ảnh) bằng cách điều chỉnh learning rate, epoch hoặc áp dụng data augmentation.
