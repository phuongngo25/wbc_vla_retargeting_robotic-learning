# Humanoid VLA Project — Whole-Body Control · Retargeting · GR00T/SONIC

> Dự án nghiên cứu do mentor giao, tách riêng khỏi trục PhD chính (`../ROADMAP.md` — medical imaging/surgical robotics). Đây là lộ trình **deep research** vào hệ sinh thái NVIDIA GR00T cho humanoid robot: từ dữ liệu chuyển động người → retargeting → whole-body control → VLA → triển khai robot thật.

---

## Bối cảnh: hệ sinh thái bạn đang bước vào

Toàn bộ dự án xoay quanh **một hệ sinh thái duy nhất** của NVIDIA, không phải các mảnh rời rạc. Hiểu sơ đồ này trước khi đọc bất cứ thứ gì khác:

```text
Dữ liệu chuyển động người (AMASS/SMPL-X, OMOMO, LAFAN1, video)
        │  (03-human-motion-datasets)
        ▼
Retargeting  (GMR / SOMA-retargeter)              ──► 02-motion-retargeting
        │  chuyển động người → quỹ đạo khớp robot (G1/H2...)
        ▼
Huấn luyện Whole-Body Control (WBC)                ──► 01-whole-body-control
        │  motion tracking bằng RL/IL trong simulation      04-imitation-learning-rl
        │  (SONIC = decoupled WBC, foundation cho toàn bộ)  05-simulation-mujoco-isaaclab
        ▼
Policy Evaluation (sim benchmark, sim-to-real gap)  ──► 07-policy-evaluation
        ▼
VLA — GR00T N1.7 dùng SONIC làm "tay chân"          ──► 06-vla-groot-sonic
        │  ngôn ngữ + ảnh → task cấp cao → SONIC thực thi động tác
        ▼
Teleoperation (thu dữ liệu qua VR) + Triển khai robot thật (G1)  ──► 08-real-robot-deployment
```

**Ba trụ cột chính, đọc theo thứ tự này khi mới bắt đầu:**

1. **SONIC** — *Supersizing Motion Tracking for Natural Humanoid Whole-Body Control* — [arXiv:2511.07820](https://arxiv.org/abs/2511.07820) · [GEAR-SONIC](https://nvlabs.github.io/GEAR-SONIC/). Đây là paper WBC chính, và cũng là paper mô tả `GR00T-WholeBodyControl`.
2. **GR00T N1** — *An Open Foundation Model for Generalist Humanoid Robots* — [arXiv:2503.14734](https://arxiv.org/abs/2503.14734). Model VLA gốc; N1.5/N1.6/N1.7 là các bản nâng cấp (N1.7 dùng pretraining EgoScale trên 20,000+ giờ video egocentric, phát hiện scaling law đầu tiên cho sự khéo léo của robot) — [HF blog N1.7](https://huggingface.co/blog/nvidia/gr00t-n1-7).
3. **GR00T-WholeBodyControl** (repo tổng, umbrella) — [nvlabs.github.io/GR00T-WholeBodyControl](https://nvlabs.github.io/GR00T-WholeBodyControl/) · [GitHub](https://github.com/NVlabs/GR00T-WholeBodyControl). Đây là **tài liệu trung tâm** — mọi getting-started, tutorial VR teleop, training guide đều nằm ở đây.

**Dữ liệu quy mô lớn dùng để huấn luyện SONIC:** **BONES-SEED** — 142,220 chuyển động người (~288 giờ, 522 diễn viên), kèm quỹ đạo G1 tương ứng đã retarget sẵn. ⚠️ *Đã kiểm chứng lại (xem `03-human-motion-datasets/`): dataset này do **Bones Studio** công bố (gated trên HuggingFace `bones-studio/seed`), KHÔNG phải dữ liệu nội bộ của NVIDIA — NVIDIA đóng góp công cụ retargeting (SOMA-retargeter) chứ không sở hữu dữ liệu gốc.* Đây là lý do vì sao "human motion dataset" và "retargeting" đứng trước "WBC" trong sơ đồ trên — WBC hiện đại được huấn luyện *từ* dữ liệu người đã retarget, không phải tay viết reward.

---

## Cấu trúc thư mục

```text
humanoid-vla-project/
├── README.md                        # File này — tổng quan & sơ đồ pipeline
├── ROADMAP.md                       # ⭐ Lộ trình học end-to-end theo tuần
├── 01-whole-body-control/           # WBC cổ điển + học sâu, kiến trúc decoupled (SONIC)
├── 02-motion-retargeting/           # GMR, SOMA-retargeter, IK, mapping người → robot
├── 03-human-motion-datasets/        # AMASS/SMPL-X, OMOMO, LAFAN1, BONES-SEED
├── 04-imitation-learning-rl/        # Motion tracking reward, AMP, PPO, DeepMimic, ASAP
├── 05-simulation-mujoco-isaaclab/   # MuJoCo, MuJoCo Playground, Isaac Lab, HumanoidVerse
├── 06-vla-groot-sonic/              # GR00T N1→N1.7, SONIC, tích hợp VLA + WBC
├── 07-policy-evaluation/            # Benchmark, sim-to-real gap, thống kê đánh giá
└── 08-real-robot-deployment/        # VR teleop, thu dữ liệu, ONNX, triển khai lên G1
```

**Cách dùng mỗi thư mục:** mỗi `README.md` con có cùng bố cục — (1) khái niệm cần nắm, (2) tài liệu/paper/repo đã kiểm chứng, (3) video hướng dẫn, (4) lộ trình từng bước cho người mới, (5) liên kết chéo.

---

## Chuẩn bị hạ tầng trước khi bắt đầu (Phase 0)

| Việc cần làm | Ghi chú |
|---|---|
| Máy Ubuntu 22.04/24.04 + GPU NVIDIA (RTX, ≥8GB VRAM để chạy Isaac Lab thoải mái) | Isaac Lab/Isaac Sim yêu cầu Linux + driver NVIDIA mới. MuJoCo thuần CPU vẫn chạy được trên máy yếu hơn. |
| Tài khoản HuggingFace | Để tải checkpoint GR00T N1.x (`nvidia/GR00T-N1.7-3B`…). |
| Clone `NVlabs/GR00T-WholeBodyControl` và đọc `getting_started/` trước tiên | [Getting Started](https://nvlabs.github.io/GR00T-WholeBodyControl/getting_started/) |
| (Nếu có phần cứng) Unitree G1 + kính VR (Quest) cho teleoperation | Xem `08-real-robot-deployment/`. Công cụ teleop mentor nhắc tới ("XToolRobotKits") khớp với **XRoboToolkit** — framework teleop OpenXR mã nguồn mở, test trên Ubuntu 22.04/24.04 — [arXiv:2508.00097](https://arxiv.org/abs/2508.00097) · [GitHub](https://github.com/XR-Robotics). |
| Zotero/Notion để quản lý paper — dùng chung với `../templates/paper-reading-note.md` | Không cần dựng hệ thống riêng, tái dùng template gốc của repo. |

> ⚠️ **Link đã kiểm tra:** `vnrobo.com/series/groot-sonic-humanoid-wbc` hiện trả về **404** — series này chưa tồn tại/đã gỡ. Đừng mất thời gian tìm; tài liệu tiếng Việt chuyên sâu cho mảng này gần như chưa có, phần lớn phải đọc tiếng Anh. `Earl000333/humanoid-wbc-review` xác nhận là repo review hợp lệ, dùng làm bản đồ paper (xem `01-whole-body-control/`).

---

## Liên kết với repo chính

Repo chính (`../resources/08-core-reading-list.md` **Track C2/C3**) đã có sẵn các paper robot learning nền tảng (Diffusion Policy, ACT, RT-2, OpenVLA, π₀, PPO, SAC, Domain Randomization) — các thư mục ở đây **không lặp lại** mà trỏ thẳng tới đó và bổ sung phần **humanoid-specific** (motion tracking, retargeting, WBC). Dùng chung `../templates/weekly-log.md` để ghi log hàng tuần cho dự án này — tạo `logs/` trong thư mục này khi bạn bắt đầu.

Xem `ROADMAP.md` để bắt đầu ngay.
