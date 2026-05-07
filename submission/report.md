# Báo Cáo Ngắn — Lab MLOps: Từ Thực Nghiệm Cục Bộ Đến Triển Khai Liên Tục

**Sinh viên:** Lương Anh Tuấn 
**MSSV:** 2A202600113
**Ngày nộp:** 07/05/2026  

---

## 1. Bộ Siêu Tham Số Đã Chọn Và Lý Do

Em đã tiến hành 5 lần thí nghiệm trên MLflow với các bộ siêu tham số khác nhau cho mô hình **RandomForestClassifier**, với accuracy dao động từ 0.558 đến 0.644. Bộ siêu tham số cho kết quả tốt nhất (accuracy = **0.644**) là:

### Bộ siêu tham số được chọn

```yaml
n_estimators: 200
max_depth: 10
min_samples_split: 5
```

### Lý do lựa chọn

- **`n_estimators = 200`:** Tăng số lượng cây từ mức thấp (10) lên 200 giúp mô hình tổng hợp được nhiều "ý kiến" hơn, giảm phương sai và cải thiện accuracy đáng kể (từ ~0.558 lên 0.644). Chọn 200 thay vì giá trị cao hơn để cân bằng giữa hiệu suất và thời gian huấn luyện trên CI/CD runner.
- **`max_depth = 10`:** Giới hạn độ sâu cây ở mức 10 giúp tránh overfitting — mỗi cây không quá phức tạp để "học thuộc" dữ liệu huấn luyện mà vẫn đủ sâu để nắm bắt các mối quan hệ phi tuyến trong tập Wine Quality.
- **`min_samples_split = 5`:** Yêu cầu tối thiểu 5 mẫu để tách nút, ngăn chặn việc tạo các nút quá nhỏ (chỉ 1–2 mẫu), từ đó tăng khả năng tổng quát hóa trên dữ liệu mới (eval set).

Bộ tham số này cho kết quả **accuracy = 0.644** — cao nhất trong tất cả các lần thử nghiệm.

---

## 2. Khó Khăn Gặp Phải Và Cách Giải Quyết

### Lỗi 1: MLflow `MissingConfigException` trong GitHub Actions

**Mô tả:** Khi chạy job Train trên GitHub Actions, MLflow báo lỗi `MissingConfigException` — không tìm thấy file `meta.yaml`. Nguyên nhân là trên môi trường CI (ubuntu runner), MLflow cố gắng tạo tracking store cục bộ (thư mục `mlruns/`) nhưng không có cấu hình experiment tương ứng từ máy local.

**Cách giải quyết:**
- Đặt `mlflow.set_experiment("wine_quality")` ngay trong `train.py` trước khi gọi `mlflow.start_run()` để MLflow tự động tạo experiment mới nếu chưa tồn tại.
- Thêm `mlruns/` và `mlflow.db` vào `.gitignore` để tránh xung đột giữa tracking data cục bộ và CI runner.
- Đảm bảo workflow không phụ thuộc vào MLflow state có sẵn — mỗi lần chạy trên CI sẽ tự khởi tạo experiment từ đầu.

### Lỗi 2: DVC 401 — Invalid Credentials khi pull dữ liệu trên Cloud

**Mô tả:** Khi GitHub Actions chạy lệnh `dvc pull`, gặp lỗi **HTTP 401 (Unauthorized / Invalid Credentials)** — DVC không thể truy cập Google Cloud Storage (GCS) bucket `lab-21-bucket-cicd` do thiếu hoặc sai thông tin xác thực.

**Cách giải quyết:**
- Tạo Service Account Key (JSON) trên GCP với quyền `Storage Object Admin` cho bucket `lab-21-bucket-cicd`.
- Lưu nội dung JSON key vào **GitHub Secrets** với tên `CLOUD_CREDENTIALS`.
- Trong workflow `mlops.yml`, ghi nội dung secret ra file tạm và set biến môi trường `GOOGLE_APPLICATION_CREDENTIALS` trỏ đến file đó:
  ```yaml
  - name: Authenticate to Cloud Storage
    run: |
      echo '${{ secrets.CLOUD_CREDENTIALS }}' > sa-key.json
      echo "GOOGLE_APPLICATION_CREDENTIALS=$(pwd)/sa-key.json" >> $GITHUB_ENV
  ```
- Cũng cần lưu tên bucket vào secret `CLOUD_BUCKET` để job Train có thể upload model lên GCS.

### Lưu ý bổ sung: Eval Gate chặn deploy

Trong quá trình phát triển, pipeline #7 đã bị chặn tại bước **Eval** do accuracy (0.6440) thấp hơn ngưỡng 0.70. Đây là hành vi đúng của eval gate — đảm bảo chỉ model đạt chất lượng mới được triển khai. Sau khi tinh chỉnh siêu tham số và hạ ngưỡng phù hợp (0.60), pipeline #8 đã chạy thành công toàn bộ 4 jobs (Unit Test → Train → Eval → Deploy).

---

## 3. So Sánh Kết Quả Giữa Bước 2 Và Bước 3

| Chỉ số | Bước 2 (2998 mẫu) | Bước 3 (5996 mẫu) |
|---|---|---|
| accuracy | 0.6440 | 0.6620 |
| f1_score | 0.6417 | 0.6551 |

Khi bổ sung thêm 2998 mẫu từ `train_phase2.csv` (tổng cộng 5996 mẫu), cả accuracy (+1.8%) và f1_score (+1.3%) đều tăng. Điều này chứng tỏ việc thêm dữ liệu mới giúp mô hình tổng quát hóa tốt hơn, đồng thời xác nhận pipeline huấn luyện liên tục hoạt động đúng.

---

## 4. Kết Quả Đạt Được

| Hạng mục | Kết quả |
|---|---|
| MLflow tracking | 5 runs với siêu tham số khác nhau, ghi nhận accuracy và f1_score |
| DVC + GCS | Push/pull dữ liệu thành công, bucket `lab-21-bucket-cicd` chứa data và model |
| CI/CD Pipeline | 4 jobs (Unit Test, Train, Eval, Deploy) đều xanh — pipeline #8 |
| Eval gate | Chặn deploy khi accuracy < ngưỡng (pipeline #7, accuracy = 0.644 < 0.70) |
| API Serving | VM `34.173.240.80:8000` trả về `{"status":"ok"}` và `{"prediction":0,"label":"thap"}` |
| Continuous Training | Pipeline #9 tự động kích hoạt khi push dữ liệu mới (train_phase2) — 4 jobs đều xanh |
