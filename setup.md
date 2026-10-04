# Hướng dẫn cài đặt môi trường – Rice Classification

Project phân loại 5 giống gạo (Arborio, Basmati, Ipsala, Jasmine, Karacadag) bằng CNN, dùng **TensorFlow 2.10 + Python 3.10**.
Hướng dẫn này viết cho **Windows 10/11** (PowerShell). Có hai nhánh: máy **có GPU NVIDIA** và máy **không có GPU** (chạy bằng CPU).

> **Vì sao Python 3.10 và TensorFlow 2.10?** Đây là phiên bản TensorFlow cuối cùng hỗ trợ GPU trực tiếp trên Windows, và nó chỉ chạy với Python 3.7–3.10. Nếu máy bạn đang có Python 3.11 / 3.12 thì **không dùng được**, hãy làm theo Bước 3 để tạo môi trường Python 3.10 riêng (không ảnh hưởng Python hiện có).

---

## Bước 1. Lấy code về máy

```powershell
git clone <link-repo-github>
cd Rice_Classification
```

Từ đây, mọi lệnh đều chạy trong thư mục gốc `Rice_Classification`.

---

## Bước 2. Tạo folder `data/raw` và tải dataset

Git **không** chứa dữ liệu (thư mục `data/` nằm trong `.gitignore`), nên sau khi clone bạn sẽ chưa có folder này. Tự tạo:

```powershell
New-Item -ItemType Directory -Force data\raw
```

**Link tải dataset:**

<!-- Dán link tải dataset vào dòng trống phía trên -->

Tải về, giải nén, rồi đặt thư mục `Rice_Image_Dataset` vào trong `data\raw`. Cấu trúc đúng phải như sau:

```
Rice_Classification/
├── data/
│   └── raw/
│       └── Rice_Image_Dataset/
│           ├── Arborio/       (15.000 ảnh)
│           ├── Basmati/       (15.000 ảnh)
│           ├── Ipsala/        (15.000 ảnh)
│           ├── Jasmine/       (15.000 ảnh)
│           ├── Karacadag/     (15.000 ảnh)
│           └── Rice_Citation_Request.txt
├── notebooks/
├── requirements.txt
└── setup.md
```

Kiểm tra nhanh: 5 thư mục lớp phải nằm **trực tiếp** trong `Rice_Image_Dataset`. Đôi khi giải nén bị lồng thêm một lớp `Rice_Image_Dataset\Rice_Image_Dataset\...`; nếu gặp trường hợp này, hãy kéo 5 thư mục lớp ra một cấp.

---

## Bước 3. Tạo môi trường Python 3.10

Chọn **một** trong hai cách.

### Cách 1 (khuyên dùng): dùng `uv`

`uv` tự tải Python 3.10 cho môi trường này, không cần gỡ hay đổi Python đang cài.

```powershell
winget install --id=astral-sh.uv -e        # hoặc: pip install uv
uv venv .venv-tf --python 3.10
.\.venv-tf\Scripts\Activate.ps1
```

### Cách 2: dùng Python 3.10 cài sẵn

Cần cài Python 3.10 từ python.org trước.

```powershell
py -3.10 -m venv .venv-tf
.\.venv-tf\Scripts\Activate.ps1
```

> Nếu PowerShell báo lỗi không cho chạy script, chạy `Set-ExecutionPolicy -Scope Process Bypass` rồi kích hoạt lại.

Sau khi kích hoạt, đầu dòng lệnh sẽ có `(.venv-tf)`.

> **Lưu ý các bước sau:** nếu dùng Cách 1, dùng `uv pip install ...`. Nếu dùng Cách 2, thay `uv pip install` bằng `pip install`.

---

## Bước 4. Cài thư viện

Chọn nhánh theo máy của bạn. Bước 4A là **bắt buộc cho mọi máy**; Bước 4B chỉ làm thêm nếu máy có GPU NVIDIA.

### 4A. Thư viện chung (mọi máy)

```powershell
uv pip install -r requirements.txt
```

Nếu máy **không có GPU**, bạn đã xong phần cài đặt, chuyển thẳng sang Bước 5. TensorFlow sẽ tự chạy bằng CPU.

### 4B. Thêm hỗ trợ GPU (chỉ máy có GPU NVIDIA)

**Điều kiện:** có card NVIDIA và đã cài driver đủ mới để hỗ trợ CUDA 11.8 (khoảng bản 522.06 trở lên). Kiểm tra bằng lệnh `nvidia-smi`; nếu lệnh in ra bảng thông tin GPU là ổn.

Bạn **không cần** cài CUDA Toolkit hay cuDNN thủ công. Các thư viện này được cài qua pip (dung lượng tải khá lớn, ước tính 1–2 GB):

```powershell
uv pip install nvidia-cuda-runtime-cu11==11.8.89 nvidia-cuda-nvrtc-cu11==11.8.89 nvidia-cuda-nvcc-cu11==11.8.89 nvidia-cublas-cu11==11.11.3.6 nvidia-cufft-cu11==10.9.0.58 nvidia-curand-cu11==10.3.0.86 nvidia-cusolver-cu11==11.4.1.48 nvidia-cusparse-cu11==11.7.5.86 nvidia-cudnn-cu11==8.9.5.29
```

Các file DLL của những gói này nằm trong `.venv-tf\Lib\site-packages\nvidia\...\bin`, nhưng TensorFlow trên Windows không tự tìm ra chúng. Cần tạo thêm hai file nhỏ để Python tự nạp DLL mỗi lần khởi động. Chạy nguyên khối lệnh sau trong thư mục gốc project:

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

> Bộ phiên bản trên là bộ đã chạy được trên máy của tác giả (RTX 4050). Chưa kiểm tra trên các dòng card khác.

---

## Bước 5. Kiểm tra cài đặt

```powershell
python -c "import tensorflow as tf; print('TensorFlow', tf.__version__); print('GPU:', tf.config.list_physical_devices('GPU'))"
```

| Máy | Kết quả đúng |
|---|---|
| Có GPU | `TensorFlow 2.10.1` và `GPU: [PhysicalDevice(name='/physical_device:GPU:0', device_type='GPU')]` |
| Không GPU | `TensorFlow 2.10.1` và `GPU: []` (kèm vài dòng cảnh báo "Could not load dynamic library", đây là bình thường) |

Nếu máy có GPU mà ra `GPU: []`, xem phần "Xử lý lỗi" ở cuối.

---

## Bước 6. Mở notebook

Đăng ký môi trường thành một kernel Jupyter:

```powershell
python -m ipykernel install --user --name rice-tf --display-name "Python 3.10 (TF 2.10)"
```

Mở `notebooks/model_CNN.ipynb` bằng VS Code (hoặc `jupyter lab`) và chọn kernel **Python 3.10 (TF 2.10)**. Trong VS Code, bạn cũng có thể chọn trực tiếp `.venv-tf` trong phần "Select Kernel → Python Environments".

---

## Bước 7. Sửa đường dẫn trong notebook

Notebook đang ghi đường dẫn tuyệt đối theo máy của tác giả. **Bạn phải sửa 2 dòng sau thành đường dẫn trên máy mình:**

1. Cell ở mục *2. Loading and explore the dataset*:
   ```python
   dataset_path = r"D:\Rice_DL_Project\Rice_Classification\data\raw\Rice_Image_Dataset"
   ```
2. Cell ở mục *6. Model Training*:
   ```python
   models_dir = r"D:\Rice_DL_Project\Rice_Classification\models"
   ```

Ví dụ: `r"C:\Users\Ten\Rice_Classification\data\raw\Rice_Image_Dataset"`. Thư mục `models` sẽ được notebook tự tạo.

---

## Bước 8. Chạy notebook

Chạy lần lượt từ trên xuống dưới. Thứ tự quan trọng:

1. Cell kiểm tra dữ liệu (mục *Data quality check*) chạy **trước** cell chia dữ liệu. Cell này đọc 75.000 ảnh nên mất vài phút.
2. Cell chia dữ liệu 70/15/15, rồi xây model, compile, train, đánh giá.

**Về việc train:**
- Máy **có GPU**: train 10 epoch. Trên RTX 4050, khoảng 2 phút mỗi epoch ở trạng thái ổn định.
- Máy **không GPU**: vẫn chạy được nhưng **chậm hơn nhiều** (chưa đo; dự kiến rất lâu với ảnh 224×224).
- Sau khi train xong, notebook tự lưu `models/model1_cnn.h5` và `models/model1_cnn_history.json`. Lần chạy sau, nếu hai file này đã có thì notebook **nạp lại luôn, không train lại**. Muốn train lại thì đặt `FORCE_RETRAIN = True` trong cell train, hoặc xoá hai file đó.
- **Mẹo cho máy không GPU:** nhờ một bạn có GPU gửi hai file trên (qua Drive, Zalo...) và bỏ vào thư mục `models/` của bạn. Khi đó notebook sẽ nạp model có sẵn và đánh giá luôn, không cần train. Điều kiện: dataset giống nhau, để test set giống nhau.

---

## Xử lý lỗi thường gặp

**Máy có GPU nhưng kết quả `GPU: []`**
1. Chạy `nvidia-smi`. Nếu lỗi, cài hoặc cập nhật driver NVIDIA.
2. Kiểm tra đang dùng đúng môi trường: `python --version` phải là 3.10.x.
3. Kiểm tra hai file `_nvidia_dll_path.pth` và `_nvidia_dll_path.py` có trong `.venv-tf\Lib\site-packages`.
4. Kiểm tra các gói NVIDIA đã cài: `uv pip list | findstr nvidia` (hoặc `pip list | findstr nvidia`).
5. **Khởi động lại kernel/terminal.** Hai file `.pth` chỉ có tác dụng khi Python khởi động.

**Lỗi liên quan `protobuf` hoặc `numpy`**
Không tự nâng cấp các gói này. `requirements.txt` đã ghim phiên bản tương thích với TensorFlow 2.10 (`protobuf<3.20`, `numpy<1.24`). Nếu lỡ nâng cấp, cài lại: `uv pip install -r requirements.txt`.

**Hết bộ nhớ GPU (Out of Memory)**
Giảm `batch_size` từ 32 xuống 16 trong hàm `make_generator` ở mục *3. Data Preprocessing*.

**Notebook báo "FileNotFoundError" hoặc không thấy ảnh**
Kiểm tra lại cấu trúc thư mục ở Bước 2 và đường dẫn `dataset_path` ở Bước 7.

**Không thấy kernel `Python 3.10 (TF 2.10)`**
Chạy lại lệnh đăng ký kernel ở Bước 6 khi môi trường `.venv-tf` đang được kích hoạt, rồi tải lại cửa sổ VS Code.

---

## Ghi chú

- Hướng dẫn này chưa được kiểm tra trên Linux và macOS. Máy macOS có thể làm theo nhánh "không GPU".
- Thư mục `.venv-tf/`, `data/` và `models/` không được đưa lên git; mỗi người tự tạo trên máy mình.
