# Báo Cáo Lab Day 21 - CI/CD cho AI Systems

| | |
|---|---|
| Họ và tên | Nguyễn Việt Phong |
| MSSV | 2A202601975 |
| Lớp / Khóa | K4 |
| Repo GitHub | https://github.com/phongviet/K4-Track2-Day21-2A202601975-NguyenVietPhong |
| Ngày nộp | 21/08/2026 |

---

## 1. Bộ Siêu Tham Số Đã Chọn và Lý Do

| Lần chạy | n_estimators | learning_rate | max_depth | f1_score | accuracy |
|---|---|---|---|---|---|
| 1 | 100 | 0.10 | 3 | 0.7109 | 0.8780 |
| 2 | 50 | 0.05 | 2 | 0.6051 | 0.8460 |
| 3 | 200 | 0.10 | 5 | 0.7149 | 0.8740 |

**Bộ siêu tham số đã chọn:** `n_estimators=200`, `learning_rate=0.1`, `max_depth=5`.

**Lý do:** Bộ siêu tham số ở lần chạy 3 đạt điểm F1-score cao nhất (0.7149) trên tập holdout và vượt qua ngưỡng Quality Gate (0.65). Mặc dù lần chạy 1 đạt Accuracy cao nhất (0.8780), F1-score của nó lại thấp hơn (0.7109), điều này chứng minh Accuracy không phản ánh đúng năng lực phân loại trên lớp thiểu số. Ngoài ra, giữa n_estimators và learning_rate có sự đánh đổi rõ rệt: khi giảm đồng thời cả hai ở lần chạy 2, mô hình bị underfitting khiến F1 tụt mạnh xuống 0.6051.

---

## 2. Vì Sao Ngưỡng Chất Lượng Đặt Trên F1 Chứ Không Phải Accuracy

Tập dữ liệu Adult bị mất cân bằng lớp nghiêm trọng khi chỉ có khoảng 24.8% mẫu thuộc lớp thu nhập cao (>50K). Một mô hình ngây thơ luôn dự đoán "thu nhập thấp" cho mọi trường hợp vẫn có thể đạt Accuracy 75.2%, nhưng hoàn toàn vô dụng trong thực tế vì bỏ sót toàn bộ khách hàng thu nhập cao (F1 = 0). Do đó, chỉ số F1-score của lớp dương (harmonic mean giữa Precision và Recall) là thước đo then chốt để đánh giá đúng chất lượng mô hình. Khi tính F1, bắt buộc tính trực tiếp trên lớp dương (target=1) mà không dùng `average="weighted"` hay `"macro"` để tránh bị lớp đa số thổi phồng điểm số.

---

## 3. Khó Khăn Gặp Phải và Cách Giải Quyết

| Khó khăn | Nguyên nhân | Cách giải quyết |
|---|---|---|
| DVC pull lỗi 401 trên GitHub Actions | File `.dvc/config` ghi cứng đường dẫn `credentialpath` cục bộ không tồn tại trên CI runner | Loại bỏ `credentialpath` trong config và dùng biến môi trường `GOOGLE_APPLICATION_CREDENTIALS` |
| Lỗi unpickle model trên VM | Phiên bản `scikit-learn` trên VM (1.7+) khác với phiên bản huấn luyện (1.4.2) | Cài đặt chính xác phiên bản `scikit-learn==1.4.2` và `numpy<2` trên VM |
| Health check thất bại ở Job Release | Server cần vài giây để tải model từ GCS khi khởi động, vượt quá thời gian `sleep 5` cố định | Viết vòng lặp retry trong script deploy (kiểm tra mỗi 3s, tối đa 10 lần) |

---

## 4. So Sánh Bước 2 và Bước 3

| | f1_score | accuracy |
|---|---|---|
| Bước 2 (chỉ `train_batch1`) | 0.7149 | 0.8740 |
| Bước 3 (thêm `train_batch2`) | 0.7149 | 0.8740 |

**Nhận xét:** Khi tăng gấp đôi dữ liệu từ 22.361 lên 44.722 mẫu, các chỉ số F1-score và Accuracy giữ nguyên (hoặc dao động rất nhỏ) do dữ liệu bổ sung có cùng phân phối thống kê với tập ban đầu và mô hình đã học trọn vẹn đặc trưng từ batch 1. Mục tiêu cốt lõi đạt được ở Bước 3 là chứng minh quy trình Continuous Training tự động hóa hoàn toàn: chỉ cần commit dữ liệu DVC mới, toàn bộ pipeline CI/CD tự động kích hoạt, kiểm định và triển khai phiên bản mới lên VM thành công.
