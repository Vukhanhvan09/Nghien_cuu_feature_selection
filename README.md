# Feature Selection cho dự đoán hành vi mua hàng

> Đồ án liên ngành: Nghiên cứu Feature Selection cho bài toán phân tích và dự đoán hành vi mua hàng của khách hàng.

Dự án sử dụng dữ liệu **Online Retail II** để xây dựng đặc trưng ở cấp khách hàng, so sánh các phương pháp lựa chọn đặc trưng và dự đoán **FutureSpending — tổng chi tiêu sản phẩm trong 30 ngày tiếp theo** bằng mô hình hồi quy Ridge.

Phương pháp được lựa chọn là **Hybrid Mutual Information → Sequential Forward Selection (MI → SFS)**, rút gọn từ **14 xuống 4 đặc trưng**, dựa trên sự đánh đổi giữa sai số dự đoán, thời gian lựa chọn và độ ổn định.

## Mục lục

- [Mục tiêu](#mục-tiêu)
- [Công nghệ sử dụng](#công-nghệ-sử-dụng)
- [Dữ liệu](#dữ-liệu)
- [Quy trình thực hiện](#quy-trình-thực-hiện)
- [Xây dựng đặc trưng](#xây-dựng-đặc-trưng)
- [Phương pháp lựa chọn đặc trưng](#phương-pháp-lựa-chọn-đặc-trưng)
- [Thiết lập thực nghiệm](#thiết-lập-thực-nghiệm)
- [Kết quả](#kết-quả)
- [Các tệp chính](#các-tệp-chính)
- [Hướng dẫn chạy](#hướng-dẫn-chạy)
- [Sử dụng mô hình đã lưu](#sử-dụng-mô-hình-đã-lưu)
- [Giới hạn và hướng phát triển](#giới-hạn-và-hướng-phát-triển)
- [Nguồn dữ liệu](#nguồn-dữ-liệu)

## Mục tiêu

- Khám phá và tiền xử lý dữ liệu giao dịch bán lẻ.
- Xây dựng bộ đặc trưng mô tả lịch sử mua hàng của khách hàng.
- So sánh các phương pháp Feature Selection thuộc nhóm Filter, Wrapper, Embedded và Hybrid.
- Đánh giá ảnh hưởng của lựa chọn đặc trưng đến sai số dự đoán, chi phí tính toán và độ ổn định.
- Lưu mô hình cuối cùng và kiểm chứng trên tập Test.

## Công nghệ sử dụng

| Công nghệ / thư viện | Vai trò |
| --- | --- |
| Python | Ngôn ngữ lập trình |
| pandas, NumPy | Xử lý dữ liệu và tính toán |
| python-calamine | Đọc dữ liệu Excel |
| SciPy | Phân tích thống kê |
| scikit-learn | Tiền xử lý, lựa chọn đặc trưng và hồi quy |
| Matplotlib, Seaborn | Trực quan hóa |
| joblib | Lưu và tải mô hình |
| Jupyter Notebook / Google Colab | Thực hiện thí nghiệm |
| Visual Studio Code | Chạy notebook trên máy cá nhân |

## Dữ liệu

Dự án sử dụng bộ dữ liệu **Online Retail II** từ UCI Machine Learning Repository.

Mỗi dòng dữ liệu biểu diễn một mặt hàng trong hóa đơn. Một hóa đơn có thể chứa nhiều dòng sản phẩm.

### Quy mô dữ liệu ban đầu

| Thành phần | Số dòng | Số cột gốc |
| --- | ---: | ---: |
| Year 2009–2010 | 525.461 | 8 |
| Year 2010–2011 | 541.910 | 8 |
| Tích hợp hai sheet | 1.067.371 | 8 |

Các trường dữ liệu được sử dụng trong notebook:

| Trường | Ý nghĩa |
| --- | --- |
| `Invoice` | Mã hóa đơn |
| `StockCode` | Mã sản phẩm |
| `Description` | Mô tả sản phẩm |
| `Quantity` | Số lượng sản phẩm |
| `InvoiceDate` | Thời điểm giao dịch |
| `Price` | Đơn giá |
| `Customer ID` | Mã khách hàng |
| `Country` | Quốc gia |

Giá trị của từng dòng giao dịch được tính bằng:

```python
Amount = Quantity * Price
```

### Định nghĩa bài toán dự đoán

Với mỗi khách hàng đã xuất hiện trong cửa sổ quan sát:

```text
FutureSpending = Tổng Amount của các giao dịch sản phẩm hợp lệ
                 trong khoảng (cutoff, cutoff + 30 ngày]
```

Khách hàng không phát sinh mua hàng trong cửa sổ tương lai được gán `FutureSpending = 0`.

| Thiết lập | Giá trị |
| --- | --- |
| Cutoff | 2011-11-09 12:50:00 |
| Observation Window | Các giao dịch hợp lệ có thời gian ≤ cutoff |
| Future Window | 30 ngày sau cutoff |
| Số khách hàng | 5.691 |
| Train | 4.552 khách hàng |
| Test | 1.139 khách hàng |
| Khách hàng trùng giữa Train và Test | 0 |
| Biến mục tiêu | `FutureSpending` |

Observation/Future được tách theo thời gian để xây dựng đặc trưng và nhãn. Sau đó, bảng khách hàng được chia ngẫu nhiên thành Train/Test theo tỷ lệ 80/20.

## Quy trình thực hiện

| Phần | Nội dung |
| --- | --- |
| **A. Khám phá và tiền xử lý** | Tích hợp dữ liệu; kiểm tra thiếu dữ liệu, trùng lặp, hóa đơn hủy, giá và số lượng; áp dụng chính sách mã sản phẩm. |
| **B. Xây dựng đặc trưng** | Tạo đặc trưng RFM, Temporal, Behavioral và biến mục tiêu FutureSpending. |
| **C. Nghiên cứu Feature Selection** | Xây dựng baseline; so sánh phương pháp; phân tích sai số, thời gian, độ ổn định và độ nhạy cấu hình; lưu mô hình cuối. |
| **D. Kiểm chứng** | Kiểm tra dữ liệu và mô hình; đánh giá Test; so sánh đối chứng và phân tích sai số. |

Các đặc trưng được xây dựng từ Observation Window. Future Window chỉ dùng để tạo biến mục tiêu.

## Xây dựng đặc trưng

Bộ đầu vào gồm **14 đặc trưng**, chia thành ba nhóm.

### RFM Features

| Đặc trưng | Cách xây dựng |
| --- | --- |
| `Recency` | Số ngày từ lần mua gần nhất đến cutoff |
| `Frequency` | Số hóa đơn mua hàng duy nhất |
| `Monetary` | Tổng giá trị mua sản phẩm hợp lệ |

### Temporal Features

| Đặc trưng | Cách xây dựng |
| --- | --- |
| `AveragePurchaseHour` | Giờ mua hàng trung bình, mỗi hóa đơn được tính một lần |
| `PreferredDayOfWeek` | Ngày trong tuần có nhiều hóa đơn nhất |
| `ActiveMonths` | Số tháng năm khác nhau có phát sinh mua hàng |
| `AverageInterpurchaseTime` | Khoảng cách trung bình giữa các lần mua liên tiếp |
| `PurchaseTrend` | Hệ số góc của đường hồi quy chi tiêu theo tháng, chỉ dùng các tháng đã hoàn tất |

### Behavioral Features

| Đặc trưng | Cách xây dựng |
| --- | --- |
| `AverageOrderValue` | Monetary / Frequency |
| `AverageQuantityPerInvoice` | Tổng số lượng sản phẩm trung bình trên mỗi hóa đơn |
| `AveragePrice` | Đơn giá trung bình trên các dòng giao dịch |
| `TotalQuantity` | Tổng số lượng sản phẩm đã mua |
| `UniqueProducts` | Số mã sản phẩm khác nhau đã mua |
| `ProductDiversity` | Shannon entropy chuẩn hóa từ tỷ trọng số lượng từng sản phẩm |

`AverageInterpurchaseTime` chưa xác định khi khách hàng có dưới hai lần mua. `PurchaseTrend` chưa xác định khi chưa đủ hai tháng quan sát hoàn chỉnh. Những giá trị thiếu này được xử lý bằng imputation trong pipeline.

## Phương pháp lựa chọn đặc trưng

| Nhóm | Phương pháp |
| --- | --- |
| Filter | F-test, Mutual Information |
| Wrapper | Recursive Feature Elimination (RFE), Sequential Forward Selection (SFS) |
| Embedded | Lasso lựa chọn đặc trưng, sau đó Ridge dự đoán |
| Hybrid | Mutual Information → SFS |

Các mô hình đối chứng gồm:

- Dummy mean.
- Dummy median.
- Ridge sử dụng toàn bộ 14 đặc trưng.

### Hybrid MI → SFS

Phương pháp Hybrid được thực hiện như sau:

1. Tính Mutual Information giữa từng đặc trưng và FutureSpending.
2. Giữ lại **m = 7** đặc trưng ứng viên có điểm MI cao nhất.
3. Áp dụng SFS tiến để chọn **k = 4** đặc trưng.
4. Đánh giá các tập ứng viên bằng Ridge và MAE trong inner CV.
5. Huấn luyện Ridge trên tập đặc trưng được chọn.
6. Sau khi chốt cấu hình, lựa chọn lại trên toàn bộ Train và lưu mô hình cuối.

`PreferredDayOfWeek` được khai báo là đặc trưng rời rạc khi tính Mutual Information.

## Thiết lập thực nghiệm

| Thành phần | Cấu hình |
| --- | --- |
| Mô hình hồi quy | Ridge |
| Ridge alpha | 1.0 |
| Xử lý thiếu | SimpleImputer, strategy="median" |
| Giữ cột rỗng | keep_empty_features=True |
| Chuẩn hóa | StandardScaler |
| Outer CV | KFold 5 fold, shuffle=True |
| Inner CV của SFS | 5 fold |
| Random state | 42 |
| n_jobs | 1 |
| Chỉ số đánh giá | MAE, MSE, RMSE, R² |
| Tiêu chí sai số chính | MAE |
| Độ ổn định | Tần suất lựa chọn và Jaccard giữa các fold |
| Độ lệch chuẩn | ddof=0 |

Pipeline dự đoán:

```python
Pipeline([
    (
        "imputer",
        SimpleImputer(
            strategy="median",
            keep_empty_features=True
        )
    ),
    ("scaler", StandardScaler()),
    ("model", Ridge(alpha=1.0))
])
```

Trong đánh giá CV, các bước học từ dữ liệu được thực hiện trên phần huấn luyện tương ứng của từng fold.

## Kết quả

Các số liệu dưới đây lấy từ output đã lưu trong notebook `Đồ_án_liên_ngành(6).ipynb`. Dấu chấm trong bảng kết quả là dấu thập phân.

### Kết quả 5-fold CV trên Train

| Phương pháp | Số đặc trưng | MAE trung bình | Jaccard trung bình |
| --- | ---: | ---: | ---: |
| Ridge full | 14 | 294.4401 | 1.0000 |
| F-test | 3 | 256.3655 | — |
| Mutual Information | 7 | 264.0290 | — |
| RFE | 2 | 259.6997 | 0.5333 |
| SFS | 4 | 252.9966 | 0.5943 |
| **Hybrid MI → SFS** | **4** | **253.6166** | **0.8400** |
| MI-only — ablation | 4 | 268.7550 | 0.8400 |
| Dummy median | Không áp dụng | 232.1873 | Không áp dụng |

MI 7 đặc trưng và MI-only 4 đặc trưng là hai cấu hình khác nhau. MI-only 4 đặc trưng được dùng để phân tích đóng góp của bước SFS.

Jaccard của Ridge full bằng 1 vì phương pháp này luôn giữ toàn bộ đặc trưng.

### Căn cứ lựa chọn Hybrid

Quy tắc lựa chọn chấp nhận Hybrid khi:

- MAE tăng không quá **0,5%** so với SFS.
- Thời gian lựa chọn giảm ít nhất **20%**.
- Jaccard trung bình tăng ít nhất **0,15**.

Kết quả so với SFS:

| Chỉ tiêu | Kết quả |
| --- | ---: |
| MAE tăng | 0,2451% |
| Jaccard trung bình | 0,5943 → 0,8400 |
| Mức tăng Jaccard | 0,2457 |
| Thời gian lựa chọn ở bảng so sánh chính | 9,4615 → 4,9027 giây |
| Mức giảm thời gian ở bảng so sánh chính | 48,1824% |

Trong lần đo riêng của khảo sát độ nhạy, cấu hình `(m=7, k=4)` giảm **40,8420%** thời gian lựa chọn so với `(m=14, k=4)`.

Thời gian được đo riêng theo từng thí nghiệm, phụ thuộc môi trường và lần chạy; không đại diện cho thời gian chạy toàn bộ notebook.

### Tập đặc trưng cuối cùng

```python
final_features = [
    "Frequency",
    "Monetary",
    "Recency",
    "AverageInterpurchaseTime"
]
```

Tập cuối giảm từ **14 xuống 4 đặc trưng**, tương đương mức rút gọn **71,43%**.

### Kết quả trên Test

| Mô hình | MAE | RMSE | R² |
| --- | ---: | ---: | ---: |
| Dummy mean | 294.7021 | 991.9036 | -0.0051 |
| Dummy median | **161.6678** | 1002.5137 | -0.0267 |
| Ridge full — 14 đặc trưng | 239.5096 | **844.5495** | **0.2714** |
| Ridge — 4 đặc trưng cuối | 210.7397 | 933.8040 | 0.1092 |

So với Ridge full, mô hình 4 đặc trưng:

- Giảm MAE **12,0120%**.
- Tăng RMSE **10,5683%**.
- Có R² thấp hơn.

Việc lựa chọn đặc trưng cải thiện MAE trong thiết lập hiện tại, nhưng chưa cải thiện đồng thời mọi chỉ số.

Mô hình cuối chưa vượt Dummy median về MAE. Trong Test, **75,68% khách hàng có FutureSpending bằng 0**, giúp dự đoán trung vị bằng 0 đạt MAE thấp.

## Các tệp chính

| Tệp / thư mục | Nội dung |
| --- | --- |
| `README.md` | Giới thiệu và hướng dẫn sử dụng |
| `Đồ_án_liên_ngành(6).ipynb` | Notebook chính gồm phần A–D |
| `ket_qua_feature_selection/` | Thư mục kết quả được tạo khi chạy |
| `ket_qua_feature_selection/eda_figures/` | Các hình được lưu qua hàm hỗ trợ EDA |
| `ket_qua_feature_selection/wrapper_results/` | Kết quả và pipeline Wrapper |
| `ket_qua_feature_selection/embedded_results/` | Kết quả Embedded |
| `ket_qua_feature_selection/hybrid_results/` | Kết quả, cấu hình và độ ổn định của Hybrid |
| `ket_qua_feature_selection/section6_results/` | Các bảng tổng hợp độ ổn định |
| `ket_qua_feature_selection/final_feature_selection_ridge.joblib` | Gói mô hình cuối và metadata |

Một số biểu đồ được lưu trực tiếp tại thư mục làm việc theo lệnh `savefig` trong notebook.

## Hướng dẫn chạy

### 1. Chuẩn bị dữ liệu

Tải bộ dữ liệu tại:

[Online Retail II — UCI Machine Learning Repository](https://archive.ics.uci.edu/dataset/502/online+retail+ii)

Notebook sử dụng:

- File nén: `online+retail+ii.zip`.
- File Excel: `online_retail_II.xlsx`.
- Hai sheet: `Year 2009-2010` và `Year 2010-2011`.

### 2. Chạy trên Google Colab

1. Mở notebook chính bằng Google Colab.
2. Đưa file ZIP vào Google Drive.
3. Cập nhật `zip_path` theo vị trí dữ liệu:

   ```python
   zip_path = "/content/drive/MyDrive/FeatureSelection/online+retail+ii.zip"
   ```

4. Kiểm tra các cell mount Drive, giải nén và thiết lập `file_path`.
5. Chạy notebook theo thứ tự **A → B → C → D**.
6. Lưu các kết quả cần thiết trong `ket_qua_feature_selection/`.

**Lưu ý:** Notebook có nhiều cell gán đường dẫn; cần cập nhật nhất quán để tránh ghi đè đường dẫn đã sửa.

### 3. Chạy trên VS Code

#### Tạo môi trường ảo và cài thư viện

Trên Windows:

```powershell
py -m venv .venv

.\.venv\Scripts\python.exe -m pip install --upgrade pip

.\.venv\Scripts\python.exe -m pip install numpy pandas scipy scikit-learn matplotlib seaborn joblib python-calamine jupyter ipykernel
```

Trên macOS / Linux:

```bash
python3 -m venv .venv

.venv/bin/python -m pip install --upgrade pip

.venv/bin/python -m pip install numpy pandas scipy scikit-learn matplotlib seaborn joblib python-calamine jupyter ipykernel
```

Mở notebook trong VS Code và chọn kernel Python của `.venv`.

#### Điều chỉnh các cell dành cho Colab

Bỏ hoặc chú thích:

```python
from google.colab import drive

drive.mount("/content/drive")
```

Giải nén dữ liệu vào thư mục `data/`, sau đó thay các cell thiết lập đường dẫn và giải nén của Colab bằng:

```python
from pathlib import Path

file_path = Path("data") / "online_retail_II.xlsx"

assert file_path.is_file(), (
    f"Không tìm thấy dữ liệu: {file_path.resolve()}"
)
```

Thư mục `data/` cần nằm dưới thư mục làm việc của notebook.

Sau khi điều chỉnh, chạy lần lượt từ **A → D**. Không chạy lại các cell gán đường dẫn `/content/...`.

Để tái lập thí nghiệm, cần lưu phiên bản Python và các thư viện trong môi trường đã chạy thành công. Phiên bản scikit-learn phải hỗ trợ `root_mean_squared_error` và `keep_empty_features`.

## Sử dụng mô hình đã lưu

Mô hình được lưu tại:

```text
ket_qua_feature_selection/final_feature_selection_ridge.joblib
```

File này chứa dictionary gồm mô hình, cấu hình và thứ tự đặc trưng.

Ví dụ sử dụng với `X_test` đã được tạo trong notebook:

```python
import joblib

bundle = joblib.load(
    "ket_qua_feature_selection/final_feature_selection_ridge.joblib"
)

model = bundle["model"]
feature_order = bundle["feature_order"]

X_predict = X_test.loc[:, feature_order].copy()
predictions = model.predict(X_predict)

print("Phương pháp:", bundle["method"])
print("Đặc trưng:", feature_order)
print("Dự đoán:", predictions[:5])
```

Đối với khách hàng mới, cần xây dựng bảng đặc trưng theo cùng định nghĩa và chính sách dữ liệu của phần B trước khi dự đoán.

Pipeline đã lưu thực hiện imputation, chuẩn hóa và hồi quy. Đầu vào của pipeline là bảng đặc trưng khách hàng.

## Giới hạn và hướng phát triển

- Thực nghiệm sử dụng một cutoff và một cách chia dữ liệu; cần kiểm chứng tại nhiều mốc thời gian.
- Việc lựa chọn cấu hình từ kết quả CV có thể khiến đánh giá trên Train lạc quan.
- Test đã được xem ở một số bước trước phần D, nên chưa phải holdout hoàn toàn chưa từng được quan sát.
- Mô hình cuối chưa vượt Dummy median về MAE.
- Ridge có thể tạo dự đoán âm; kết quả hiện tại sử dụng dự đoán gốc.
- Một số khách hàng có chi tiêu lớn ảnh hưởng mạnh đến RMSE và R².
- Cần khóa phiên bản môi trường và đo thời gian nhiều lần để tăng khả năng tái lập.
- Có thể nghiên cứu mô hình hai giai đoạn: dự đoán khả năng mua hàng và dự đoán mức chi tiêu khi có mua.

## Nguồn dữ liệu

Chen, D. (2012). **Online Retail II** [Dataset]. UCI Machine Learning Repository.

- Dataset: https://archive.ics.uci.edu/dataset/502/online+retail+ii
- DOI: https://doi.org/10.24432/C5CG6D
- Giấy phép dữ liệu: **CC BY 4.0**

Giấy phép dữ liệu áp dụng cho bộ dữ liệu Online Retail II. Giấy phép mã nguồn của repository cần được tác giả quy định riêng.
