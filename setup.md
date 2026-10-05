# Rice Classification – SETUP

Hướng dẫn cài đặt môi trường để chạy các notebook của project phân loại 5 giống gạo (Arborio, Basmati, Ipsala, Jasmine, Karacadag) bằng CNN.

## Mục lục

1. [Yêu cầu](#1-yêu-cầu)
2. [Chọn cấu hình theo máy](#2-chọn-cấu-hình-theo-máy)
3. [Cài đặt](#3-cài-đặt)
4. [Kiểm tra cài đặt](#4-kiểm-tra-cài-đặt)
5. [Chạy notebook](#5-chạy-notebook)
6. [Troubleshooting](#6-troubleshooting)
7. [Ghi chú](#7-ghi-chú)

---

## 1. Yêu cầu

| Thành phần | Yêu cầu |
|---|---|
| Hệ điều hành | Windows 10/11 64-bit (dùng PowerShell) |
| Python | 3.10 (TensorFlow 2.10 không hỗ trợ Python 3.11 trở lên) |
| Công cụ | Git |
| GPU (tuỳ chọn) | NVIDIA, driver hỗ trợ CUDA 11.8 (khuyến nghị bản 522.06 trở lên) |

Môi trường Python 3.10 được tạo riêng trong project, không ảnh hưởng Python đang cài trên máy.

---

## 2. Chọn cấu hình theo máy

Kiểm tra máy có GPU nào:

```powershell
Get-CimInstance Win32_VideoController | Select-Object Name
```

- Có dòng `NVIDIA GeForce ...` → máy có **GPU NVIDIA**.
- Chỉ có `Intel(R) ... Graphics` hoặc `AMD Radeon(TM) Graphics` → máy **không có GPU rời** (chỉ có card tích hợp).

| Loại máy | Cấu hình | Các bước thực hiện |
|---|---|---|
| Có GPU NVIDIA | TensorFlow GPU (CUDA 11.8) | 3.1 → 3.4, **3.5A**, 3.6 → 3.8 |
| Không có GPU rời | TensorFlow CPU | 3.1 → 3.4, **3.5B**, 3.6 → 3.8 |
| Không có GPU rời, muốn thử tăng tốc bằng card tích hợp | CPU + DirectML (thử nghiệm) | 3.1 → 3.4, **3.5C**, 3.6 → 3.8 |

Chỉ thực hiện **một** trong ba mục 3.5A / 3.5B / 3.5C.

---

## 3. Cài đặt

Tất cả lệnh chạy trong PowerShell, tại thư mục gốc `Rice_Classification` (từ 3.2 trở đi).

### 3.1. Clone repo

```powershell
git clone <link-repo-github>
cd Rice_Classification
```

### 3.2. Tạo folder `data/raw` và tải dataset

Thư mục `data/` nằm trong `.gitignore` nên không có sau khi clone. Tạo thủ công:

```powershell
New-Item -ItemType Directory -Force data\raw
```

**Link tải dataset:  *https://www.kaggle.com/datasets/muratkokludataset/rice-image-dataset?select=Rice_Image_Dataset*

<!-- Dán link tải dataset vào dòng trống phía trên -->

Giải nén và đặt thư mục `Rice_Image_Dataset` vào `data\raw`. Cấu trúc đúng:

```
Rice_Classification/
├── data/
│   └── raw/
│       └── Rice_Image_Dataset/
│           ├── Arborio/
│           ├── Basmati/
│           ├── Ipsala/
│           ├── Jasmine/
│           ├── Karacadag/
│           └── Rice_Citation_Request.txt
├── notebooks/
├── requirements.txt
└── SETUP.md
```

Năm thư mục lớp phải nằm **trực tiếp** trong `Rice_Image_Dataset`. Nếu bị lồng thêm một lớp `Rice_Image_Dataset\Rice_Image_Dataset\`, di chuyển năm thư mục lớp ra một cấp.

### 3.3. Tạo môi trường Python 3.10

Chọn một trong hai cách.

**Cách 1 (khuyến nghị): `uv`.** `uv` tự tải Python 3.10 cho môi trường này.

```powershell
winget install --id=astral-sh.uv -e        # hoặc: pip install uv
uv venv .venv-tf --python 3.10
```

**Cách 2: Python 3.10 cài sẵn (python.org).**

```powershell
py -3.10 -m venv .venv-tf
```

### 3.4. Kích hoạt môi trường

```powershell
.\.venv-tf\Scripts\Activate.ps1
```

Đầu dòng lệnh hiện `(.venv-tf)` là thành công. Nếu PowerShell chặn script, chạy `Set-ExecutionPolicy -Scope Process Bypass` rồi kích hoạt lại.

> Từ đây, dùng `uv pip install` nếu chọn Cách 1 ở 3.3. Nếu chọn Cách 2, thay `uv pip install` bằng `pip install`.

### 3.5A. Cài đặt cho máy có GPU NVIDIA

Cài thư viện chung:

```powershell
uv pip install -r requirements.txt
```

Kiểm tra driver (lệnh phải in ra bảng thông tin GPU):

```powershell
nvidia-smi
```

Cài CUDA 11.8 và cuDNN 8.9.5 qua pip (không cần cài CUDA Toolkit riêng; dung lượng tải ước tính 1–2 GB):

```powershell
uv pip install nvidia-cuda-runtime-cu11==11.8.89 nvidia-cuda-nvrtc-cu11==11.8.89 nvidia-cuda-nvcc-cu11==11.8.89 nvidia-cublas-cu11==11.11.3.6 nvidia-cufft-cu11==10.9.0.58 nvidia-curand-cu11==10.3.0.86 nvidia-cusolver-cu11==11.4.1.48 nvidia-cusparse-cu11==11.7.5.86 nvidia-cudnn-cu11==8.9.5.29
```

TensorFlow trên Windows không tự tìm DLL của các gói trên. Tạo hai file để Python tự nạp DLL khi khởi động:

```powershell
$sp = ".\.venv-tf\Lib\site-packages"

Set-Content -Path "$sp\_nvidia_dll_path.pth" -Value "import _nvidia_dll_path" -Encoding ascii

@'
"""Auto-register CUDA/cuDNN DLLs from pip nvidia-* wheels at interpreter startup."""
import glob
import os
import sys

if sys.platform == "win32":
    for _d in glob.glob(os.path.join(sys.prefix, "Lib", "site-packages", "nvidia", "*", "bin")):
        os.add_dll_directory(_d)
        os.environ["PATH"] = _d + os.pathsep + os.environ.get("PATH", "")
'@ | Set-Content -Path "$sp\_nvidia_dll_path.py" -Encoding ascii
```

### 3.5B. Cài đặt cho máy không có GPU rời (CPU)

Chỉ cần cài thư viện chung. TensorFlow tự chạy bằng CPU, không cần cấu hình thêm:

```powershell
uv pip install -r requirements.txt
```

### 3.5C. Máy không có GPU rời: thử tăng tốc bằng card tích hợp (DirectML, thử nghiệm)

Plugin DirectML cho phép TensorFlow dùng card tích hợp Intel/AMD trên Windows.

> **Cảnh báo:**
> - Microsoft đã **ngừng phát triển** plugin này; bản phát hành mới nhất là bản pre-release.
> - Plugin yêu cầu `tensorflow-cpu==2.10`, **không dùng chung** với gói `tensorflow` trong `requirements.txt`.
> - Cấu hình này **chưa được kiểm tra** với notebook của project, và chưa rõ có nhanh hơn CPU hay không.
> - Nếu có lỗi, quay lại 3.5B bằng cách tạo lại môi trường từ 3.3.

Cài các thư viện chung trừ `tensorflow`, sau đó cài `tensorflow-cpu` và plugin:

```powershell
Get-Content requirements.txt | Where-Object { $_ -notmatch '^tensorflow' } | Set-Content "$env:TEMP\req-no-tf.txt"
uv pip install -r "$env:TEMP\req-no-tf.txt"
uv pip install "tensorflow-cpu==2.10.*" tensorflow-directml-plugin
```

Yêu cầu của plugin: Windows 10 v1709+ hoặc Windows 11, Python 3.7–3.10, card AMD Radeon R5/R7/R9 2xx trở lên hoặc Intel HD Graphics 5xx trở lên.

### 3.6. Đăng ký kernel Jupyter

```powershell
python -m ipykernel install --user --name rice-tf --display-name "Python 3.10 (TF 2.10)"
```

Mở `notebooks/model_CNN.ipynb` bằng VS Code (hoặc `jupyter lab`) và chọn kernel **Python 3.10 (TF 2.10)**. Trong VS Code có thể chọn trực tiếp `.venv-tf` ở *Select Kernel → Python Environments*.

### 3.7. Sửa đường dẫn trong notebook

Notebook dùng đường dẫn tuyệt đối của máy tác giả. Sửa hai dòng sau cho đúng với máy của mình:

| Cell | Dòng cần sửa |
|---|---|
| Mục *2. Loading and explore the dataset* | `dataset_path = r"D:\Rice_DL_Project\Rice_Classification\data\raw\Rice_Image_Dataset"` |
| Mục *6. Model Training* | `models_dir = r"D:\Rice_DL_Project\Rice_Classification\models"` |

Ví dụ: `r"C:\Users\Ten\Rice_Classification\data\raw\Rice_Image_Dataset"`. Thư mục `models` được notebook tự tạo.

### 3.8. Chạy notebook

Chạy lần lượt từ trên xuống. Cell *Data quality check* phải chạy **trước** cell chia dữ liệu và mất vài phút do đọc 75.000 ảnh. Xem mục 5 cho phần train.

---

## 4. Kiểm tra cài đặt

```powershell
python -c "import tensorflow as tf; print('TensorFlow', tf.__version__); print('GPU:', tf.config.list_physical_devices('GPU'))"
```

| Cấu hình | Kết quả đúng |
|---|---|
| GPU NVIDIA (3.5A) | `TensorFlow 2.10.1` và `GPU: [PhysicalDevice(name='/physical_device:GPU:0', device_type='GPU')]` |
| CPU (3.5B) | `TensorFlow 2.10.1` và `GPU: []`. Các dòng cảnh báo "Could not load dynamic library" là bình thường |
| DirectML (3.5C) | `TensorFlow 2.10.x` và danh sách `GPU` có một thiết bị. Nếu là `[]`, plugin chưa hoạt động |

---

## 5. Chạy notebook

### Thời gian train

| Cấu hình | Thời gian mỗi epoch (Model 1, ảnh 224×224) |
|---|---|
| GPU NVIDIA (RTX 4050) | Khoảng 2 phút khi ổn định; vài epoch đầu có thể chậm hơn nhiều (đã gặp khoảng 10 phút) |
| CPU | Chưa đo, dự kiến chậm hơn rất nhiều. Chạy thử `epochs=1` để ước lượng trước khi chạy đủ 10 epoch |
| DirectML | Chưa đo |

### Lưu và nạp model

Sau khi train xong, notebook tự lưu `models/model1_cnn.h5` và `models/model1_cnn_history.json`. Ở lần chạy sau, nếu hai file đã tồn tại, notebook **nạp model và bỏ qua bước train**.

- Train lại: đặt `FORCE_RETRAIN = True` trong cell train, hoặc xoá hai file trên.
- File `.h5` nặng khoảng 45MB, **không commit** lên git.

### Máy không GPU: dùng model train sẵn (khuyến nghị)

1. Nhận hai file `model1_cnn.h5` và `model1_cnn_history.json` từ người đã train xong (qua Drive hoặc kênh khác).
2. Đặt vào thư mục `models/` của máy mình (đúng đường dẫn `models_dir` ở 3.7).
3. Chạy notebook từ đầu. Cell train sẽ nạp model có sẵn, cell đánh giá chạy luôn trên test set.

Điều kiện: dataset phải giống nhau để cả hai máy có cùng test set (split cố định bằng `random_state=42`).

---

## 6. Troubleshooting

**GPU NVIDIA nhưng kết quả `GPU: []`**

1. Chạy `nvidia-smi`. Nếu lỗi, cài hoặc cập nhật driver NVIDIA.
2. Kiểm tra `python --version` ra 3.10.x và đang kích hoạt `.venv-tf`.
3. Kiểm tra hai file `_nvidia_dll_path.pth` và `_nvidia_dll_path.py` có trong `.venv-tf\Lib\site-packages`.
4. Kiểm tra gói NVIDIA đã cài: `uv pip list | findstr nvidia`.
5. Khởi động lại kernel và terminal. Hai file `.pth` chỉ có tác dụng khi Python khởi động.

**Hết bộ nhớ GPU (Out of Memory)**

Giảm `batch_size` từ 32 xuống 16 trong hàm `make_generator` ở mục *3. Data Preprocessing*.

**DirectML: lỗi ở dòng `set_memory_growth` trong cell import**

Thiết bị DirectML có thể không hỗ trợ `set_memory_growth`. Đặt hai dòng `for gpu in gpus:` và `tf.config.experimental.set_memory_growth(gpu, True)` thành comment.

**Lỗi liên quan `protobuf` hoặc `numpy`**

Không nâng cấp hai gói này. `requirements.txt` đã ghim phiên bản tương thích với TensorFlow 2.10 (`protobuf<3.20`, `numpy<1.24`). Nếu lỡ nâng cấp, cài lại: `uv pip install -r requirements.txt`.

**`FileNotFoundError` hoặc không thấy ảnh**

Kiểm tra lại cấu trúc thư mục ở 3.2 và đường dẫn `dataset_path` ở 3.7.

**Không thấy kernel `Python 3.10 (TF 2.10)`**

Chạy lại lệnh đăng ký ở 3.6 khi `.venv-tf` đang được kích hoạt, rồi tải lại cửa sổ VS Code.

---

## 7. Ghi chú

- Đã kiểm tra: Windows, Python 3.10, TensorFlow 2.10.1, GPU NVIDIA RTX 4050 (cấu hình 3.5A).
- Chưa kiểm tra: cấu hình CPU (3.5B), DirectML (3.5C), card NVIDIA đời khác, Linux và macOS. Máy macOS có thể thử theo cấu hình CPU.
- `.venv-tf/`, `data/` không được đưa lên git; mỗi người tự tạo trên máy mình.
