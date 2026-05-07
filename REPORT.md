# Báo Cáo Lab Day 21 - CI/CD Cho AI Systems

## 1. Kết quả thí nghiệm cục bộ với MLflow

Em đã chạy 5 thí nghiệm huấn luyện mô hình `RandomForestClassifier` với các bộ siêu tham số khác nhau và theo dõi bằng MLflow. Mỗi lần chạy đều ghi nhận đầy đủ `accuracy`, `f1_score` và các tham số `n_estimators`, `max_depth`, `min_samples_split`.

Bộ siêu tham số tốt nhất ở giai đoạn dữ liệu ban đầu là:

```yaml
n_estimators: 300
max_depth: null
min_samples_split: 2
```

Kết quả tốt nhất trên tập `eval.csv` ở Bước 1:

```text
accuracy: 0.682
f1_score: khoảng 0.681
```

Em chọn bộ tham số này vì nó đạt accuracy cao nhất trong các lần chạy MLflow, đồng thời f1-score cũng cao và ổn định hơn các cấu hình có cây quá nông hoặc số lượng cây thấp.

## 2. DVC, Cloud Storage và CI/CD

Em đã dùng DVC để version hóa dữ liệu và đẩy dữ liệu lên Google Cloud Storage. Bucket sử dụng cho lab là:

```text
mlops-day21-hoangthaiduong-494008
```

Trong bucket, dữ liệu được lưu dưới prefix:

```text
dvc/
```

Pipeline GitHub Actions gồm 4 job:

```text
Unit Test -> Train -> Eval -> Deploy
```

Ở lần chạy đầu với `train_phase1.csv`, job `Unit Test` và `Train` chạy thành công, nhưng `Eval` chặn deploy vì accuracy chưa đạt ngưỡng `0.70`. Đây là hành vi mong muốn của eval gate.

Sau đó em chạy `add_new_data.py` để gộp thêm `train_phase2.csv` vào `train_phase1.csv`, cập nhật DVC bằng `dvc add`, chạy `dvc push` trước rồi mới `git push`. Commit dữ liệu mới đã tự động kích hoạt lại GitHub Actions. Lần chạy sau đó hoàn thành thành công cả 4 job, chứng minh pipeline có khả năng huấn luyện liên tục khi dữ liệu thay đổi.

## 3. Triển khai mô hình

Mô hình sau huấn luyện được upload lên Google Cloud Storage tại:

```text
models/latest/model.pkl
```

Em đã triển khai FastAPI trên VM `mlops-serve` và cấu hình service `mlops-serve` bằng systemd. API cung cấp hai endpoint:

```text
GET /health
POST /predict
```

Kết quả kiểm tra:

```text
GET /health -> {"status":"ok"}
POST /predict -> trả về prediction và label hợp lệ
```

## 4. Khó khăn và cách xử lý

Trong quá trình làm lab, em gặp một số vấn đề:

- Lỗi cài thư viện với `uv` do `mlflow` cần `pkg_resources`; em xử lý bằng cách dùng Python 3.10 và cài lại `setuptools`.
- MLflow trong unit test ban đầu bị lỗi tracking URI; em đặt default tracking URI là `sqlite:///mlflow.db`.
- GitHub Actions ban đầu bị chặn vì repo fork chưa enable Actions; em đã bật Actions thủ công.
- Deploy thất bại do `VM_SSH_KEY` bị paste sai; em tạo lại và kiểm tra SSH key bằng lệnh `ssh -i ... "echo ok"`.
- GCP service account key từng bị lộ trong quá trình thao tác; em đã tạo key mới, cập nhật `CLOUD_CREDENTIALS`, copy key mới lên VM và xóa key cũ trên GCP.

## 5. Bằng chứng nộp kèm

Các ảnh chụp màn hình đã lưu trong thư mục `screenshots/`:

```text
01-mlflow-runs.png
02-cloud-storage-dvc.png
03-eval-gate-blocked.png
04-github-actions-step3-green.png
05-api-health-predict.png
```

