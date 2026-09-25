# Quét độc lập trước khi xem pre-label
Frame: frame_0099.jpg
Số xe nhìn thấy bằng mắt: 25 xe (gồm nhiều xe ở xa kích thước rất bé)
Hai vị trí dễ bị AI bỏ sót hoặc vẽ sai, kèm mô tả xe:
1. Xe ở khoảng cách xa trên làn đường bị xe phía trước che khuất một phần đèn sau (chỉ nhìn thấy một bên đèn hậu đỏ), độ tương phản thấp trong đêm.
2. Xe ở sát rìa khung hình bị cắt một nửa thân xe ra ngoài ảnh và bị khuất cụm đèn, thiếu đặc trưng nhận diện đầy đủ khiến AI dễ bỏ sót.
Chạy `python3 tools/lock_blind.py` ngay sau khi điền. Sau đó giữ file này nguyên vẹn.