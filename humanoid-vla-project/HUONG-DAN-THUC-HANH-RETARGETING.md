# Hướng dẫn thực hành: Cài đặt & chạy Motion Retargeting trên máy

> Tài liệu thực hành (không theo format bài giảng 13 mục) — mục đích: biết chính xác cần cài gì, cái gì chạy được trên máy hiện tại (Mac + UTM Ubuntu VM + ROS2), cái gì không, và các bước cụ thể để đi từ dữ liệu thô tới chuyển động robot đã retarget. Đọc song song với các bài giảng chi tiết ở `02-motion-retargeting/` và `03-human-motion-datasets/` — tài liệu này KHÔNG lặp lại lý thuyết, chỉ tập trung vào "làm thế nào trên máy thật".
>
> **Cập nhật lần cuối:** dựa trên tra cứu WebSearch/WebFetch tháng 9/2026. Số liệu phần cứng/driver có thể thay đổi theo thời gian — kiểm tra lại trang chính thức nếu tài liệu này đã cũ khi bạn đọc.

---

## 1. Tóm tắt nhanh — cái gì chạy được trên máy bạn, cái gì không

Cấu hình hiện tại: **Mac + Ubuntu chạy trong UTM (máy ảo) + ROS2**.

| Công cụ | Chạy được trên máy này? | Lý do |
|---|---|---|
| **GMR** (retargeting, CPU) | ✅ **Chạy tốt** | Chỉ cần CPU — `mink` + MuJoCo giải QP nhỏ, không cần GPU. |
| **SOMA-retargeter** (retargeting, GPU) | ⚠️ **Chạy được nhưng KHÔNG khuyến nghị** | Dùng NVIDIA Warp — Warp có CPU fallback thật (xác nhận từ tài liệu chính thức), nhưng chạy CPU-only mất hoàn toàn lợi thế GPU-batch (lý do chính để chọn công cụ này) — sẽ rất chậm. |
| **Isaac Lab / Isaac Sim** (mô phỏng để kiểm chứng/huấn luyện RL) | ❌ **Không chạy được** | Yêu cầu cứng GPU NVIDIA có RT Core (RTX 20/30/40-series trở lên, tối thiểu 8GB VRAM cho bản cũ, khuyến nghị 16GB cho bản mới) — Mac (Apple Silicon lẫn Intel đời gần đây) **không có GPU NVIDIA**, và UTM **không hỗ trợ GPU passthrough** ở tầng hypervisor (Virtualization.framework của macOS không expose GPU/OpenGL 3.3+ cho VM Linux). Đây là giới hạn phần cứng + hypervisor, không phải thứ có thể "cài thêm driver" để sửa. |
| **MuJoCo (viewer, dùng bởi GMR)** | ✅ Chạy được, có thể cần cấu hình rendering phần mềm | UTM chỉ hỗ trợ OpenGL thực nghiệm cho Linux guest — nếu viewer 3D bị lỗi/chậm, dùng chế độ headless (không viewer) của GMR để retarget, chỉ mở viewer khi cần xem trực quan. |
| **MuJoCo Playground** (JAX-based, thay thế nhẹ cho Isaac Lab) | ✅ Chạy được trên CPU (chậm hơn GPU nhiều) | JAX hỗ trợ CPU backend — phù hợp thử nghiệm nhỏ, không phù hợp huấn luyện RL quy mô lớn trên máy này. |
| **ROS2** (đã cài) | ✅ Không liên quan trực tiếp tới retargeting | Cần cho `08-real-robot-deployment/` (điều khiển robot thật/teleoperation), chưa cần cho bước retargeting dữ liệu offline — xem mục 6. |

**Kết luận thực dụng:** với máy hiện tại, hãy tập trung vào **GMR** làm công cụ retargeting chính (đúng công cụ dự án đã chọn làm ưu tiên 1). Bỏ qua việc tự chạy SOMA-retargeter và Isaac Lab tại chỗ — dùng chiến lược thay thế ở mục 4.

---

## 2. Vì sao Mac không thể chạy CUDA/Isaac Lab dù dùng máy ảo (giải thích ngắn)

Đây không phải vấn đề "chưa cài đúng driver" — mà là giới hạn kiến trúc:

```text
Phần cứng Mac (Apple Silicon HOẶC Intel đời gần đây)
        │
        ▼
  KHÔNG có GPU NVIDIA (Apple dùng GPU riêng/AMD, không phải NVIDIA
  từ nhiều năm nay) → không tồn tại CUDA ở tầng phần cứng
        │
        ▼
  UTM (dựa trên QEMU + Virtualization.framework của macOS)
        │
        ▼
  Virtualization.framework KHÔNG expose GPU/OpenGL 3.3+ cho VM Linux
  → dù VM Ubuntu chạy trong UTM, nó vẫn không "thấy" được GPU thật
    (vì làm gì có GPU NVIDIA nào trên máy để expose)
        │
        ▼
  Isaac Sim/Isaac Lab kiểm tra cứng: yêu cầu GPU NVIDIA có RT Core
  → cài đặt sẽ báo lỗi / không khởi động được, bất kể cấu hình VM
```

**Có nghiên cứu thử nghiệm GPU passthrough NVIDIA qua QEMU trên Apple Silicon** (dùng eGPU Thunderbolt + QEMU 8.2, đạt kết quả trên M2 Max với RTX 6000 Ada) — nhưng đây là setup nghiên cứu phức tạp, không phải giải pháp chuẩn/ổn định, và vẫn cần **có sẵn một GPU NVIDIA vật lý** (qua eGPU) — không giải quyết được nếu bạn không có phần cứng NVIDIA nào cả.

---

## 3. Cài đặt GMR trên Ubuntu (trong UTM) — các bước cụ thể

### 3.1. Yêu cầu hệ thống (theo README chính thức)

- **OS**: Ubuntu 22.04 hoặc 20.04 (kiểm tra `lsb_release -a` trong VM — nếu bạn cài UTM với Ubuntu 24.04, thử vẫn cài được nhưng chưa được liệt kê chính thức, nên ưu tiên 22.04 nếu có thể chọn lại).
- **Python**: 3.10 (bắt buộc dùng conda/miniconda để quản lý version riêng, không dùng Python hệ thống).
- **RAM**: không có con số chính thức, nhưng vì chỉ chạy CPU/QP nhỏ, 8GB RAM cấp cho VM là đủ cho retargeting (không cần nhiều như Isaac Lab).

### 3.2. Cài Miniconda (nếu chưa có) trong Ubuntu VM

```bash
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
source ~/.bashrc
```

*(Lưu ý kiến trúc: UTM trên Apple Silicon thường chạy Ubuntu ARM64, không phải x86_64 — nếu VM của bạn là ARM64, đổi link tải thành bản `Miniconda3-latest-Linux-aarch64.sh` tương ứng.)*

### 3.3. Tạo môi trường và cài GMR

```bash
conda create -n gmr python=3.10 -y
conda activate gmr

git clone https://github.com/YanjieZe/GMR.git
cd GMR
pip install -e .

# Sửa lỗi rendering thường gặp trên Linux (thư viện C++ chuẩn)
conda install -c conda-forge libstdcxx-ng -y
```

### 3.4. Tải body model SMPL-X (bắt buộc cho input dạng SMPL-X)

1. Đăng ký tài khoản tại [smpl-x.is.tue.mpg.de](https://smpl-x.is.tue.mpg.de) (miễn phí, cần chấp nhận điều khoản sử dụng học thuật).
2. Tải file `.pkl` cho 3 giới tính (NEUTRAL/FEMALE/MALE), đặt vào:
   ```text
   GMR/assets/body_models/smplx/
   ├── SMPLX_NEUTRAL.pkl
   ├── SMPLX_FEMALE.pkl
   └── SMPLX_MALE.pkl
   ```
3. Nếu dùng file `.pkl` (không phải `.npz`), sửa `ext` trong `smplx/body_models.py` từ `npz` sang `pkl` — README chính thức có ghi bước này rõ ràng, kiểm tra lại nếu bản mới không cần nữa.

### 3.5. Chạy thử retarget một file (không cần dataset lớn ngay)

GMR có kèm dữ liệu mẫu trong repo để test trước khi tải dataset thật:

```bash
python scripts/smplx_to_robot.py \
  --smplx_file <đường_dẫn_file_smplx_mẫu_trong_repo> \
  --robot unitree_g1 \
  --save_path output/test_retarget.pkl \
  --rate_limit

python scripts/vis_robot_motion.py \
  --robot unitree_g1 \
  --robot_motion_path output/test_retarget.pkl
```

Nếu bước `vis_robot_motion.py` (mở cửa sổ MuJoCo 3D) bị treo/lỗi do giới hạn OpenGL của UTM, bỏ qua bước xem trực quan trước — xác nhận file `.pkl` output đã tạo ra và có dữ liệu hợp lệ (đọc bằng Python: `pickle.load` rồi in shape) là đủ để biết retargeting chạy đúng.

### 3.6. Xem kết quả trực quan tốt hơn: cài MuJoCo riêng trên macOS

GMR chính thức chỉ được test trên Ubuntu 22.04/20.04 (README không đề cập macOS) — nên bước **retarget** (tính toán IK, thuần CPU, không cần OpenGL) vẫn nên chạy trong Ubuntu VM như mục 3.3–3.5.

Nhưng bước **xem trực quan** (MuJoCo viewer, cần OpenGL) lại là chỗ UTM yếu nhất. Vì MuJoCo là gói Python độc lập, không phụ thuộc GMR để mở viewer, cách thực dụng hơn:

```bash
# Trên macOS (không phải trong VM)
pip install mujoco
```

1. Retarget trong Ubuntu VM như bình thường → được file `.pkl`.
2. Copy file `.pkl` (và file MJCF của robot, ví dụ `unitree_g1.xml`, lấy từ thư mục `assets/` của repo GMR) ra thư mục share giữa UTM và macOS.
3. Trên macOS, dùng lại đúng script `vis_robot_motion.py` (chỉ cần cài `mujoco` — không cần cài lại toàn bộ GMR/mink) để mở viewer — macOS có wheel `mujoco` chính thức (kể cả Apple Silicon), rendering qua OpenGL native, không bị giới hạn như UTM.

*(Chạy cả GMR trực tiếp trên macOS về lý thuyết có thể được — `mink` và `mujoco` đều pure-Python có wheel macOS — nhưng đây là đường không chính thức, không có trong README, nên nếu lỗi lạ thì không có support upstream. An toàn nhất: tính toán ở Ubuntu (official), xem hình ở macOS.)*

---

## 4. Chiến lược cho phần KHÔNG chạy được tại chỗ (SOMA-retargeter, Isaac Lab)

### 4.1. Thay vì tự chạy SOMA-retargeter: dùng dữ liệu đã retarget sẵn

BONES-SEED (đã học ở [`03-human-motion-datasets/BAI-GIANG-bones-seed.md`](03-human-motion-datasets/BAI-GIANG-bones-seed.md)) đã cung cấp sẵn bản retarget **Unitree G1 (CSV, tương thích MuJoCo)** — tức là bước "chạy SOMA-retargeter trên GPU" đã được người khác làm sẵn ở quy mô lớn. Với máy không có GPU NVIDIA, đây là lựa chọn thực dụng nhất: tải trực tiếp bản CSV đã retarget, load vào MuJoCo (CPU, không cần GPU) để xem/dùng tiếp cho các bước sau.

Nếu bắt buộc cần chạy chính SOMA-retargeter (ví dụ để retarget một nguồn dữ liệu mới chưa có bản retarget sẵn), dùng **cloud GPU** (mục 4.3) thay vì máy local.

### 4.2. Thay vì Isaac Lab: dùng MuJoCo Playground cho thử nghiệm nhẹ

[MuJoCo Playground](https://github.com/google-deepmind/mujoco_playground) (xây trên MJX/JAX, xem `05-simulation-mujoco-isaaclab/`) chạy được trên CPU (JAX hỗ trợ backend CPU) — chậm hơn GPU rất nhiều lần nhưng **có thể chạy được trên máy này**, phù hợp để:
- Kiểm tra một chuyển động đã retarget có "đứng vững" được trong mô phỏng vật lý cơ bản không (fall/không fall).
- Thử nghiệm nhỏ với một robot, một vài episode — không phù hợp huấn luyện RL quy mô lớn (hàng nghìn môi trường song song, vốn là lý do Isaac Lab cần GPU).

### 4.3. Khi thực sự cần GPU NVIDIA (SOMA-retargeter quy mô lớn, Isaac Lab huấn luyện RL)

Thuê GPU cloud theo nhu cầu, thay vì cố chạy trên máy local:

| Lựa chọn | Phù hợp cho | Ghi chú |
|---|---|---|
| Google Colab (free/Pro) | Thử nghiệm nhanh, notebook | GPU miễn phí giới hạn, không ổn định cho session dài |
| RunPod / Lambda Cloud / Vast.ai | Chạy SOMA-retargeter batch, huấn luyện Isaac Lab | Thuê theo giờ, có sẵn image Ubuntu + CUDA, giá rẻ hơn AWS/GCP cho GPU đơn lẻ |
| AWS (EC2 G/P instance) / GCP (A2/G2 instance) | Sản xuất/nghiên cứu dài hạn | Đắt hơn nhưng ổn định, tích hợp tốt nếu đã dùng hạ tầng cloud khác |

*(Đây là gợi ý phổ biến trong cộng đồng robot learning, không phải khuyến nghị thương mại — tự so sánh giá tại thời điểm cần dùng.)*

---

## 5. Dataset — nơi tải và định dạng (tóm tắt thực hành, chi tiết lý thuyết xem bài giảng)

| Dataset | Nơi tải | Cần đăng ký? | Định dạng | Dùng trực tiếp với |
|---|---|---|---|---|
| **AMASS** | [amass.is.tue.mpg.de](https://amass.is.tue.mpg.de) | Có (miễn phí, học thuật) | SMPL-X (`.npz`) — **tải bản SMPL-X, không phải SMPL+H** | GMR (`smplx_to_robot.py`) |
| **SMPL-X body models** | [smpl-x.is.tue.mpg.de](https://smpl-x.is.tue.mpg.de) | Có | `.pkl`/`.npz` | Bắt buộc để GMR đọc được dữ liệu SMPL-X |
| **LAFAN1** | [github.com/ubisoft/ubisoft-laforge-animation-dataset](https://github.com/ubisoft/ubisoft-laforge-animation-dataset) | Không (mở) | BVH | GMR (`bvh_to_robot.py` hoặc tương đương — kiểm tra tên script chính xác trong repo tại thời điểm dùng) |
| **OMOMO** | Link Google Drive trong [repo chính thức](https://github.com/lijiaman/omomo_release) | Không (link công khai) | SMPL-H/SMPL-X + object trajectory | Cần script `scripts/convert_omomo_to_smplx.py` của GMR trước khi retarget |
| **BONES-SEED** | [huggingface.co/datasets/bones-studio/seed](https://huggingface.co/datasets/bones-studio/seed) | Có (gated, license phi lợi nhuận) | SOMA (BVH) + **G1 CSV đã retarget sẵn** | Dùng trực tiếp bản G1 CSV, không cần retarget lại (xem mục 4.1) |

**Lưu ý dung lượng đĩa:** AMASS + BONES-SEED có thể chiếm hàng chục GB — kiểm tra dung lượng ổ đĩa ảo (virtual disk) đã cấp cho VM UTM trước khi tải, mở rộng nếu cần (UTM cho phép resize disk image).

---

## 6. Vai trò của ROS2 trong pipeline này — chưa cần ngay

ROS2 (bạn đã cài) **không tham gia trực tiếp** vào bước retargeting offline (GMR/SOMA-retargeter chạy độc lập, không cần ROS2). ROS2 sẽ cần thiết ở các giai đoạn sau trong pipeline dự án:

```text
Retargeting (GMR/SOMA) — KHÔNG cần ROS2, đã học/cài ở tài liệu này
        │
        ▼
Mô phỏng/huấn luyện RL (05, 04, 01) — KHÔNG cần ROS2 trực tiếp
  (Isaac Lab/MuJoCo Playground có API riêng, không bắt buộc qua ROS2)
        │
        ▼
Triển khai robot thật / Teleoperation (08-real-robot-deployment/)
  — ROS2 THƯỜNG được dùng ở đây để giao tiếp với driver robot thật,
    xử lý message giữa các node (camera, IMU, motor controller...)
```

Nếu mục tiêu trước mắt là "chạy được retargeting", ROS2 hiện chưa cần dùng tới — có thể để nguyên đã cài, sẽ dùng khi tới `08-real-robot-deployment/`.

---

## 7. Checklist cài đặt tổng hợp (theo thứ tự làm)

```text
[ ] Xác nhận kiến trúc VM: x86_64 hay ARM64 (lệnh `uname -m` trong Ubuntu VM)
[ ] Xác nhận phiên bản Ubuntu: 22.04 hoặc 20.04 (lsb_release -a)
[ ] Cài Miniconda (đúng bản kiến trúc)
[ ] conda create -n gmr python=3.10
[ ] git clone GMR + pip install -e .
[ ] conda install -c conda-forge libstdcxx-ng
[ ] Đăng ký + tải body model SMPL-X → assets/body_models/smplx/
[ ] Chạy thử retarget với dữ liệu mẫu có sẵn trong repo GMR (chưa cần tải dataset lớn)
[ ] Nếu ổn: đăng ký + tải AMASS (SMPL-X), hoặc tải LAFAN1 (mở, không cần đăng ký) để có dữ liệu thật
[ ] Retarget một file thật sang unitree_g1, kiểm tra output .pkl
[ ] (Tuỳ chọn) Đăng ký BONES-SEED nếu cần dữ liệu quy mô lớn đã retarget sẵn, tránh phải tự chạy SOMA-retargeter
[ ] KHÔNG cố cài Isaac Lab/Isaac Sim trên máy này — dùng MuJoCo Playground (CPU) cho thử nghiệm nhẹ,
    hoặc thuê cloud GPU khi cần huấn luyện RL thật
```

---

## Liên kết chéo

- Lý thuyết retargeting đầy đủ: `02-motion-retargeting/` (đặc biệt [`BAI-GIANG-gmr-kien-truc-pipeline.md`](02-motion-retargeting/BAI-GIANG-gmr-kien-truc-pipeline.md), [`BAI-GIANG-soma-retargeter-kien-truc-pipeline.md`](02-motion-retargeting/BAI-GIANG-soma-retargeter-kien-truc-pipeline.md)).
- Chi tiết dataset: `03-human-motion-datasets/` (đặc biệt [`BAI-GIANG-lafan1-dinh-dang-bvh.md`](03-human-motion-datasets/BAI-GIANG-lafan1-dinh-dang-bvh.md), [`BAI-GIANG-bones-seed.md`](03-human-motion-datasets/BAI-GIANG-bones-seed.md)).
- Mô phỏng để kiểm chứng retargeting: `05-simulation-mujoco-isaaclab/` (sẽ bổ sung bài giảng MJCF, MuJoCo Playground, Isaac Gym→Isaac Lab ở phiên tiếp theo).
- Triển khai robot thật/ROS2: `08-real-robot-deployment/`.
