# Báo Cáo Lab Day 21 - CI/CD cho AI Systems

| | |
|---|---|
| Họ và tên | Nguyễn Tiến Đạt |
| MSSV | 2A202602970 |
| Lớp / Khóa | K4 |
| Repo GitHub | https://github.com/DatTienNguyenn/K4-L3-DAY21-NguyenTienDat-2A202602970-CI-CD-for-AI-Systems |
| Ngày nộp | 07/10/2026 |

---

## 1. Bộ Siêu Tham Số Đã Chọn và Lý Do

| Lần chạy | n_estimators | learning_rate | max_depth | f1_score | accuracy |
|---|---|---|---|---|---|
| 1 | 100 | 0.1 | 3 | 0.7109 | 0.8780 |
| 2 | 50 | 0.05 | 2 | 0.6051 | 0.8460 |
| 3 | 200 | 0.1 | 5 | 0.7149 | 0.8740 |

**Bộ siêu tham số đã chọn:** `n_estimators=200`, `learning_rate=0.1`, `max_depth=5`.

**Lý do:** Bộ siêu tham số lần 3 đạt `f1_score = 0.7149`, cao nhất trong cả 3 lần thử nghiệm và vượt qua ngưỡng yêu cầu 0.65. Mặc dù lần chạy 1 có `accuracy` cao nhất (0.8780 so với 0.8740 của lần 3), nhưng `f1_score` của lần 3 vượt trội hơn (0.7149 so với 0.7109), cho thấy mô hình sâu hơn nắm bắt và phân loại lớp dương (thu nhập > 50K) hiệu quả hơn trên dữ liệu mất cân bằng. Việc lần chạy có accuracy cao nhất không trùng với lần có f1_score cao nhất chứng minh accuracy bị chi phối bởi lớp đa số (<= 50K) và không phản ánh đúng chất lượng phát hiện lớp thiểu số. Ngoài ra, có sự đánh đổi rõ rệt giữa `n_estimators` và `learning_rate`: ở lần 2 khi giảm tốc độ học xuống 0.05 cùng `max_depth=2` nhưng số cây quá ít (`n_estimators=50`), mô hình chưa kịp hội tụ khiến F1 tụt dốc còn 0.6051. Ngược lại, cấu hình `n_estimators=200` và `max_depth=5` cho phép các cây sau sửa sai hiệu quả cho các cây trước, mang lại mô hình tối ưu nhất.

---

## 2. Vì Sao Ngưỡng Chất Lượng Đặt Trên F1 Chứ Không Phải Accuracy

Tập dữ liệu Adult có phân bố lớp mất cân bằng nghiêm trọng khi tỷ lệ người có thu nhập cao (> 50K USD) chỉ chiếm khoảng 24.8%, trong khi lớp thu nhập thấp chiếm tới 75.2%. Nếu một mô hình ngây thơ luôn luôn dự đoán mọi cá nhân đều có "thu nhập thấp", nó vẫn dễ dàng đạt được accuracy 0.752 (75.2%) dù hoàn toàn vô dụng và không phân loại được bất kỳ mẫu dương tính nào. 

Chỉ số F1 trên lớp dương là trung bình điều hòa giữa Precision và Recall, đo lường chính xác mức độ hiệu quả trong việc nhận diện đúng nhóm thu nhập cao mà không bị phân tán bởi lớp chiếm đa số. Khi đánh giá, ta tuyệt đối không sử dụng `average="weighted"` hay `average="macro"` vì các tham số này sẽ bị lớp đa số kéo điểm số lên cao một cách giả tạo, làm sai lệch năng lực nhận diện thực sự của mô hình đối với nhóm mục tiêu quan trọng.

---

## 3. Khó Khăn Gặp Phải và Cách Giải Quyết

| Khó khăn | Nguyên nhân | Cách giải quyết |
|---|---|---|
| GCP từ chối tạo Service Account key file `sa-key.json` | Chính sách tổ chức GCP kích hoạt ràng buộc `disableServiceAccountKeyCreation` nhằm bảo mật | Sử dụng tệp Application Default Credentials của tài khoản owner để cung cấp xác thực cho DVC và GitHub Secrets |
| Dịch vụ trên VM gặp lỗi `AttributeError` khi nạp mô hình | Khác biệt phiên bản `scikit-learn` giữa máy ảo (1.7.2) và runner huấn luyện (1.4.2) | Đồng bộ chính xác phiên bản `scikit-learn==1.4.2` trên máy ảo theo đúng tệp `requirements.txt` |
| Pipeline không tự kích hoạt khi commit cập nhật dữ liệu | Cú pháp đường dẫn `data/**.dvc` trong GitHub Actions không khớp với định dạng tệp DVC ở thư mục con | Cập nhật cấu hình trigger trong workflow thành `data/*.dvc` và chuyển sang đẩy qua SSH cá nhân |

---

## 4. So Sánh Bước 2 và Bước 3 (bắt buộc, 2 - 3 câu)

| | f1_score | accuracy |
|---|---|---|
| Bước 2 (chỉ `train_batch1`) | 0.7149 | 0.8740 |
| Bước 3 (thêm `train_batch2`) | 0.7354 | 0.8820 |

**Nhận xét:** Khi bổ sung 22.361 mẫu từ `train_batch2` (nâng tổng số dữ liệu huấn luyện lên 44.722 mẫu), điểm F1 của lớp dương tăng từ 0.7149 lên 0.7354 và accuracy tăng nhẹ từ 0.8740 lên 0.8820. Dữ liệu bổ sung đã cung cấp thêm các trường hợp đặc trưng giúp mô hình GradientBoosting xác định ranh giới quyết định chính xác hơn đối với nhóm thu nhập cao mà không bị overfitting. Quan trọng nhất, toàn bộ chu trình Continuous Training đã vận hành trơn tru: hệ thống tự động phát hiện thay đổi dữ liệu từ DVC, kích hoạt pipeline CI/CD kiểm thử, huấn luyện lại và triển khai thẳng lên API mà không cần can thiệp thủ công.
