# Cell Counting Hough

Đếm tế bào (và các vật tròn nói chung) trong ảnh bằng **Hough circle tự cài đặt từ đầu**, kèm giao diện web bằng Python để tải ảnh, chỉnh tham số, xem từng bước xử lý và so sánh với OpenCV.

> Bài tập lớn môn Xử lý ảnh / Computer Vision, Học viện Công nghệ Bưu chính Viễn thông (PTIT).

---

## Mục lục

- [Giới thiệu](#giới-thiệu)
- [Tính năng](#tính-năng)
- [Pipeline xử lý](#pipeline-xử-lý)
- [Cấu trúc dự án](#cấu-trúc-dự-án)
- [Cài đặt](#cài-đặt)
- [Cách chạy](#cách-chạy)
- [Dữ liệu](#dữ-liệu)
- [Đánh giá](#đánh-giá)
- [Kết quả](#kết-quả)
- [Hạn chế và các trường hợp thất bại](#hạn-chế-và-các-trường-hợp-thất-bại)
- [Hướng phát triển](#hướng-phát-triển)
- [Thành viên và đóng góp](#thành-viên-và-đóng-góp)
- [Giấy phép và tài liệu tham khảo](#giấy-phép-và-tài-liệu-tham-khảo)

---

## Giới thiệu

**Bài toán.** Cho một ảnh chứa nhiều vật gần tròn (hồng cầu, nhân tế bào nhuộm huỳnh quang, nấm men, hạt bead, đồng xu...), hệ thống cần phát hiện vị trí, bán kính và trả về số lượng vật.

**Ứng dụng thực tế.** Đếm tế bào trong xét nghiệm máu và nuôi cấy, kiểm đếm hạt hoặc viên thuốc trong công nghiệp, đếm đồng xu và vật nhỏ.

**Ràng buộc của đề.** Toàn bộ phần xử lý ảnh trong `core/` được **tự cài đặt**, chỉ dùng NumPy cho tính toán. OpenCV chỉ được dùng để đọc/ghi ảnh và làm baseline so sánh trong `baseline/`.

## Tính năng

- Tự cài đặt: chuyển xám, tích chập, Gaussian, Sobel, Canny, Hough circle (bản naive và bản dùng hướng gradient), tìm đỉnh và loại vòng trùng.
- Ước lượng dải bán kính tự động (tùy chọn), không cần người dùng nhập tay.
- Giao diện Streamlit: tải ảnh, thanh trượt tham số, hiển thị từng bước (xám, làm mịn, biên, accumulator, kết quả).
- So sánh song song với `cv2.HoughCircles` về số lượng, vị trí và thời gian chạy.
- Đánh giá định lượng: MAE số lượng, Precision/Recall/F1 theo tâm, Dice/IoU nếu có mask.
- Xuất ảnh kết quả và file CSV (x, y, r).
- Chạy hàng loạt trên thư mục ảnh để tạo bảng kết quả cho báo cáo.

## Pipeline xử lý

```
Ảnh vào
  → Chuyển xám (0.299 R + 0.587 G + 0.114 B)
  → Gaussian (kernel tự tạo, tích chập NumPy)
  → [Otsu: tách nền]                         (tùy chọn)
  → Canny: gradient Sobel → NMS → ngưỡng kép → hysteresis
  → Hough circle: accumulator theo (a, b, r), bỏ phiếu dọc hướng gradient
  → Tìm đỉnh: ngưỡng + NMS + khoảng cách tối thiểu giữa các tâm
  → Đếm và vẽ kết quả
```

Hai chế độ Hough:

| Chế độ | Mô tả | Ghi chú |
|---|---|---|
| `naive` | Mỗi điểm biên bỏ phiếu cho mọi góc θ, accumulator 3D | Dễ hiểu, chậm |
| `gradient` | Mỗi điểm biên chỉ bỏ phiếu dọc hướng gradient | Nhanh hơn đáng kể |

## Cấu trúc dự án

```
cell-counting-hough/
├── README.md
├── requirements.txt
├── core/                 # thuật toán tự cài đặt (không import cv2/streamlit/fastapi)
│   ├── params.py
│   ├── grayscale.py
│   ├── convolution.py
│   ├── gaussian.py
│   ├── sobel.py
│   ├── canny.py
│   ├── otsu.py
│   ├── hough_circle.py
│   ├── peaks.py
│   ├── radius.py
│   ├── counting.py
│   └── draw.py
├── baseline/
│   └── opencv_compare.py # cv2.HoughCircles, cv2.Canny: chỉ để so sánh
├── eval/
│   ├── metrics.py
│   ├── run_experiments.py
│   ├── sweep_params.py
│   └── failure_analysis.py
├── app/
│   └── streamlit_app.py
├── data/
│   ├── samples/          # vài ảnh mẫu nhỏ cho demo
│   ├── synthetic/        # ảnh tổng hợp biết trước số lượng
│   └── labels/           # ground_truth.csv
├── tests/
├── scripts/
│   ├── generate_synthetic.py
│   └── download_data.md
├── results/
│   ├── tables/
│   └── figures/
└── docs/
    ├── report/
    ├── slides/
    └── demo/
```

## Cài đặt

Yêu cầu: Python 3.10 trở lên.

```bash
git clone https://github.com/<tên-tài-khoản>/cell-counting-hough.git
cd cell-counting-hough

python -m venv .venv
# Windows: .venv\Scripts\activate
source .venv/bin/activate

pip install -r requirements.txt
```

`requirements.txt` gợi ý:

```
numpy
opencv-python
pillow
matplotlib
streamlit
pytest
```

## Cách chạy

**Giao diện web:**

```bash
streamlit run app/streamlit_app.py
```

Sau đó mở địa chỉ hiển thị trong terminal (thường là `http://localhost:8501`), tải ảnh hoặc chọn ảnh mẫu, chỉnh tham số ở thanh bên.

**Chạy thí nghiệm hàng loạt:**

```bash
python -m eval.run_experiments --images data/samples --labels data/labels/ground_truth.csv --out results/tables
```

**Khảo sát tham số:**

```bash
python -m eval.sweep_params --images data/synthetic --out results/tables
```

**Sinh ảnh tổng hợp:**

```bash
python scripts/generate_synthetic.py --n 50 --out data/synthetic
```

**Kiểm thử:**

```bash
pytest
```

**Dùng `core/` như thư viện:**

```python
import numpy as np
from PIL import Image
from core.counting import count_circles
from core.params import HoughParams

img = np.array(Image.open("data/samples/cells_01.png").convert("RGB"))
params = HoughParams(r_min=10, r_max=40, threshold=0.5, min_dist=20)
circles = count_circles(img, params)   # [(x, y, r, votes), ...]
print(len(circles))
```

## Dữ liệu

| Nguồn | Nội dung | Ghi chú |
|---|---|---|
| Ảnh tổng hợp (`data/synthetic/`) | Vòng tròn sinh ngẫu nhiên, thêm nhiễu và thay đổi độ sáng | Biết chính xác số lượng và vị trí |
| BBBC005 / BBBC039 (Broad Institute) | Ảnh nhân tế bào huỳnh quang | Cần kiểm tra giấy phép và dạng nhãn |
| BCCD | Ảnh máu: hồng cầu, bạch cầu, tiểu cầu | Cần kiểm tra giấy phép và dạng nhãn |
| Ảnh tự chọn | Dùng để phân tích lỗi | Ghi rõ nguồn |

Hướng dẫn tải dữ liệu nằm trong `scripts/download_data.md`. Dữ liệu nặng không đưa lên repo.

## Đánh giá

- **Số lượng:** MAE và sai số tương đối (%) so với số đếm thật.
- **Vị trí:** Precision, Recall, F1 với tâm khớp trong dung sai khoảng nửa bán kính.
- **Vùng:** Dice/IoU nếu có mask.
- **So sánh:** bản tự cài đặt (naive, gradient) với `cv2.HoughCircles`, về độ chính xác và thời gian chạy.
- **Khảo sát:** mức nhiễu, mật độ tế bào, độ mờ, các tham số Hough và Canny.

## Kết quả

> Điền sau khi chạy thí nghiệm.

| Phương pháp | MAE | F1 | Thời gian (s) |
|---|---|---|---|
| Hough naive (tự code) | | | |
| Hough gradient (tự code) | | | |
| OpenCV `HoughCircles` | | | |

Ảnh minh họa đặt trong `results/figures/` và `docs/demo/`.

## Hạn chế và các trường hợp thất bại

Phân tích chi tiết nằm trong báo cáo. Các tình huống dự kiến:

- Tế bào dính nhau hoặc chồng lên nhau.
- Tế bào méo, dạng elip do góc chụp hoặc bản chất tế bào.
- Vật bị cắt ở mép ảnh.
- Nền bẩn, nhiều texture hoặc phản chiếu.
- Ảnh mờ, nhiễu lớn, độ tương phản thấp.
- Nhiều kích thước lẫn lộn khi dải bán kính quá rộng.

## Hướng phát triển

- Xử lý tế bào dính nhau bằng distance transform và watershed tự code.
- Hough elip hoặc hiệu chỉnh phối cảnh.
- Phân loại theo kích thước (ví dụ gom nhóm bằng K-means tự code).
- Tăng tốc bỏ phiếu bằng vector hóa hoặc Numba.
- Tách backend FastAPI và frontend riêng, giữ nguyên `core/`.

## Thành viên và đóng góp

Đóng góp được tính theo nội dung xử lý, không tính công viết báo cáo và làm slide.

| Thành viên | MSSV | Phần phụ trách | % đóng góp |
|---|---|---|---|
| Nguyễn Huy Hoàng | | | |
| | | | |
| | | | |

Giảng viên hướng dẫn: _(điền tên)_

## Giấy phép và tài liệu tham khảo

- Mã nguồn: _(chọn giấy phép, ví dụ MIT)_.
- Duda, R. O., Hart, P. E. (1972). *Use of the Hough transformation to detect lines and curves in pictures.*
- Canny, J. (1986). *A computational approach to edge detection.*
- Giáo trình môn học và các bộ dữ liệu đã nêu ở phần [Dữ liệu](#dữ-liệu).
