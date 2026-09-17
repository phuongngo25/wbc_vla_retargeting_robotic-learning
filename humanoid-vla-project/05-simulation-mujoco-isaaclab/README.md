# 05 — Simulation: MuJoCo & Isaac Lab

> Đây là "sân tập" — nơi bạn thực sự chạy retargeting, huấn luyện WBC, và kiểm thử policy trước khi đụng tới robot thật. Chọn công cụ đúng ngay từ đầu tiết kiệm rất nhiều thời gian.
>
> 📖 **Giải thích chi tiết đầy đủ** (kiến trúc physics engine, GPU parallelization, định dạng file): xem `NOI-DUNG-CHI-TIET.md`.

---

## A. Ba lựa chọn chính — khi nào dùng cái nào

| Công cụ | Đặc điểm | Khi nào dùng |
|---|---|---|
| **MuJoCo** (thuần, CPU/nhẹ) | Physics engine chuẩn, định dạng MJCF (XML), Python binding chính thức, viewer tương tác. Chạy tốt trên laptop không GPU. | Prototype nhanh, debug retargeting (GMR dùng MuJoCo làm viewer mặc định), học physics engine cơ bản. |
| **MuJoCo Playground** (Google DeepMind) | Bộ môi trường RL tăng tốc GPU trên nền MuJoCo MJX, dùng JAX, có sẵn PPO/SAC, có sẵn task `humanoid-stand/walk/run`. | Huấn luyện RL nhanh trên GPU của cá nhân, không cần hạ tầng Omniverse nặng. Hợp cho người mới thử motion-tracking RL lần đầu. |
| **NVIDIA Isaac Lab** (trên Isaac Sim/Omniverse) | Framework mô phỏng song song GPU quy mô lớn, tích hợp RSL-RL, hỗ trợ chính thức WBC + teleoperation nâng cao (Isaac Lab 2.3), là nền chính thức mà hệ sinh thái GR00T dùng. | Khi cần bám sát pipeline chính thức của GR00T-WholeBodyControl, huấn luyện quy mô lớn, chuẩn bị deploy lên robot thật (G1). |
| **HumanoidVerse** (CMU LeCAR Lab) | Lớp trừu tượng "multi-simulator": cùng 1 code, chạy được trên IsaacGym/Genesis/Isaac Lab, hỗ trợ sẵn Unitree H1/G1, có sẵn pipeline sim-to-sim/sim-to-real. | Khi muốn so sánh cùng 1 policy chạy trên nhiều simulator, hoặc tránh khoá cứng vào 1 framework khi mới học. |

**Gợi ý cho người mới:** bắt đầu bằng **MuJoCo thuần** (nhẹ, hiểu physics engine), sau đó **MuJoCo Playground** (thấy RL chạy nhanh trên GPU), rồi mới sang **Isaac Lab** (đúng pipeline GR00T). HumanoidVerse dùng khi bạn đã quen cả hai và muốn thực nghiệm so sánh.

> 📚 **Đọc thêm (paper gốc của từng công cụ — nền tảng kỹ thuật đứng sau, không chỉ đọc docs):**
> - **Todorov, Erez, Tassa (2012)** — *"MuJoCo: A Physics Engine for Model-Based Control"*, IROS 2012, tr. 5026–5033. Paper gốc giải thích vì sao MuJoCo tính tiếp xúc (contact) khác các physics engine game (Bullet, PhysX) — quan trọng để hiểu sai số vật lý khi retarget/huấn luyện. [Bản PDF (ResearchGate)](https://www.researchgate.net/publication/261353949_MuJoCo_A_physics_engine_for_model-based_control)
> - **MuJoCo Playground** — nhóm Google DeepMind, 2025. Kiến trúc GPU-accelerated JAX/Warp, danh sách đầy đủ task humanoid. [arXiv:2502.08844](https://arxiv.org/abs/2502.08844) (đã dẫn ở mục B).
> - **Makoviychuk et al. (NVIDIA, 2021)** — *"Isaac Gym: High Performance GPU-Based Physics Simulation For Robot Learning"*, NeurIPS Datasets & Benchmarks 2021. Isaac Gym là tiền thân trực tiếp của Isaac Lab (cùng đội NVIDIA, cùng ý tưởng "physics + neural net cùng chạy trên GPU, không qua CPU") — đọc bài này để hiểu *tại sao* Isaac Lab nhanh, trước khi đọc docs Isaac Lab. [arXiv:2108.10470](https://arxiv.org/abs/2108.10470)
> - **HumanoidVerse** chưa có paper riêng — đọc trực tiếp README kiến trúc trên GitHub (đã dẫn ở mục B).

---

## B. Tài liệu chính

| Tài liệu | Vì sao đọc | Link |
|---|---|---|
| **MuJoCo — official documentation** | Bắt buộc đọc trước: MJCF format, cách định nghĩa robot, viewer, Python API cơ bản. | [mujoco.readthedocs.io](https://mujoco.readthedocs.io) |
| **MuJoCo Playground** paper (2025) | Kiến trúc GPU-accelerated (JAX/Warp), danh sách task humanoid có sẵn, cách benchmark sim-to-real. | [arXiv:2502.08844](https://arxiv.org/abs/2502.08844) |
| **NVIDIA Isaac Lab — official docs** | Cài đặt, cấu trúc environment, RSL-RL integration. | [developer.nvidia.com/isaac/lab](https://developer.nvidia.com/isaac/lab) |
| **Isaac Lab 2.3 blog — WBC + teleoperation** | Tính năng WBC và teleop nâng cao mới nhất, liên quan trực tiếp tới pipeline GR00T. | [NVIDIA Tech Blog](https://developer.nvidia.com/blog/streamline-robot-learning-with-whole-body-control-and-enhanced-teleoperation-in-nvidia-isaac-lab-2-3) |
| **HumanoidVerse** repo | Đọc README để hiểu kiến trúc trừu tượng hoá simulator/task/algorithm. | [GitHub LeCAR-Lab/HumanoidVerse](https://github.com/LeCAR-Lab/HumanoidVerse) |

---

## C. Video hướng dẫn

| Video/Playlist | Ngôn ngữ | Nội dung |
|---|---|---|
| **Skyentific — "How to Build Humanoid Robots Using NVIDIA Isaac Lab"** (YouTube playlist) | Tiếng Anh | Series thực hành từng bước dựng và huấn luyện robot 2 chân đi bằng Isaac Lab — điểm khởi đầu thực hành tốt nhất cho người mới, đã kiểm chứng tồn tại và được cộng đồng tổng hợp lại (Class Central). |
| **MuJoCo official tutorial notebook** (Google Colab, có video giới thiệu đi kèm của nhóm DeepMind) | Tiếng Anh | Notebook tương tác chính thức — chạy trực tiếp trên trình duyệt, không cần cài gì. Tìm "MuJoCo tutorial Colab" trên trang MuJoCo GitHub. |
| **NVIDIA Isaac Lab tutorial videos** (kênh NVIDIA Robotics/Developer) | Tiếng Anh | Video chính thức từ NVIDIA giới thiệu Isaac Lab, RSL-RL workflow. |
| Cộng đồng: **10-chapter tutorial phát triển RL environment cho legged robot trong Isaac Lab** | Tiếng Anh | Tài liệu cộng đồng chi tiết hơn docs chính thức — tìm "Isaac Lab legged robot RL tutorial 10 chapters" nếu cần đi sâu từng bước xây environment tuỳ chỉnh. |

> ⚠️ Chưa tìm được video/playlist tiếng Việt hướng dẫn Isaac Lab hoặc MuJoCo Playground — mảng công cụ mô phỏng GPU cho humanoid quá mới và quá chuyên biệt, cộng đồng Việt Nam chưa có nội dung công khai đáng kể. Nếu bạn thấy kênh Việt Nam nào làm robot learning cơ bản (ROS/Gazebo), vẫn hữu ích để làm quen khái niệm simulation nói chung trước.

---

## D. Lộ trình từng bước cho người mới bắt đầu

1. **(2–3 ngày)** Cài MuJoCo (`pip install mujoco`), chạy notebook tutorial chính thức, tự tay tạo 1 file MJCF đơn giản (con lắc hoặc robot 2 khớp) và mô phỏng nó.
2. **(2–3 ngày)** Dùng MuJoCo viewer để mở file quỹ đạo G1 đã retarget ở `02-motion-retargeting/` — xác nhận pipeline dữ liệu → simulation hoạt động thông suốt.
3. **(3–5 ngày)** Cài MuJoCo Playground, chạy thử task `humanoid-walk` có sẵn với PPO mặc định — quan sát policy học đi từ đầu (random) tới ổn định.
4. **(1 tuần)** Cài Isaac Lab (chú ý yêu cầu phần cứng: Ubuntu + GPU NVIDIA driver mới). Chạy theo 3–5 tập đầu playlist Skyentific để dựng và huấn luyện 1 robot 2 chân đi.
5. **(3–5 ngày)** Đọc phần Isaac Lab 2.3 WBC + teleoperation blog — đối chiếu tính năng mới này với kiến trúc SONIC đã đọc ở `01-whole-body-control/`.
6. **(checkpoint, tuỳ chọn nâng cao)** Nếu có thời gian: thử chạy cùng 1 task walking trên cả MuJoCo Playground và Isaac Lab (qua HumanoidVerse nếu muốn code chung), so sánh tốc độ huấn luyện và chất lượng chính sách.

---

## Liên kết chéo

- Đây là nơi thực thi thuật toán mô tả ở `04-imitation-learning-rl/`.
- Kết quả huấn luyện được đánh giá ở `07-policy-evaluation/`.
- Isaac Lab là nền chính thức khi chuyển sang deploy thật — xem `08-real-robot-deployment/`.
