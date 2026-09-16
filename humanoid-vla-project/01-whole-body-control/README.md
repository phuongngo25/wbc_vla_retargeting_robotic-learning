# 01 — Whole-Body Control (WBC)

> WBC là bài toán điều khiển **đồng thời toàn bộ các khớp** của robot humanoid (chân, thân, tay, đầu) để vừa giữ thăng bằng/di chuyển, vừa thực hiện tác vụ tay — thay vì điều khiển từng phần tách rời (chân đi, tay với riêng).

---

## A. Khái niệm cần nắm, theo thứ tự

1. **WBC cổ điển (model-based):** task-space control, operational space control, QP (quadratic programming) giải đồng thời nhiều ràng buộc (cân bằng, giới hạn khớp, tiếp xúc chân), ZMP (Zero Moment Point), MPC (Model Predictive Control) cho dáng đi. Đây là cách robot humanoid được điều khiển trong ~20 năm trước khi RL phổ biến.
2. **WBC học sâu (learning-based):** thay vì viết tay bộ điều khiển, huấn luyện một policy (RL) học cách tái tạo (track) chuyển động tham chiếu từ dữ liệu người. Reward = "giống chuyển động mẫu" thay vì reward thủ công.
3. **Kiến trúc "decoupled WBC"** (SONIC): tách một policy motion-tracking cấp thấp (biết cách di chuyển tự nhiên) khỏi một planner/policy cấp cao (quyết định *làm gì*: đi đâu, cầm gì). Cấp cao có thể là con người (teleop), một kinematic planner, hoặc một VLA (GR00T N1.x). Đây là lý do một policy WBC duy nhất phục vụ được cả teleoperation lẫn VLA.
4. **Token space thống nhất:** SONIC biểu diễn lệnh điều khiển bằng một không gian token chung, để cả tín hiệu từ VR teleop và tín hiệu từ VLA đều "nói cùng một ngôn ngữ" với policy cấp thấp.

---

## B. Tài liệu chính

| Tài liệu | Loại | Vì sao đọc | Link |
|---|---|---|---|
| **SONIC** — Supersizing Motion Tracking for Natural Humanoid WBC | 🔴 Paper trụ cột | Định nghĩa kiến trúc decoupled WBC, scale theo 3 trục (model/data/compute), kết quả zero-shot 100% trên G1 thật. **Đọc kỹ nhất trong thư mục này.** | [arXiv:2511.07820](https://arxiv.org/abs/2511.07820) |
| **GR00T-WholeBodyControl** docs | 🔴 Tài liệu chính thức | Kiến trúc code thật, cấu hình observation, ONNX export, cách N1.5/1.6/1.7 dùng chung nền WBC này. | [Docs](https://nvlabs.github.io/GR00T-WholeBodyControl/) · [GitHub](https://github.com/NVlabs/GR00T-WholeBodyControl) |
| **`humanoid-wbc-review`** (Earl000333) | 🔴 Bản đồ tài liệu | Review có cấu trúc, chia WBC học sâu thành **4 paradigm**: command-based, motion-tracking-based, interaction-based, multimodal-based. Dùng file này để tự định vị SONIC nằm ở paradigm nào (motion-tracking + multimodal). Có sẵn danh sách ~100 paper, dataset, simulator theo từng paradigm. | [GitHub](https://github.com/Earl000333/humanoid-wbc-review) |
| **Modern Robotics** (Lynch & Park), chương kinematics/dynamics | 🟡 Nền tảng bắt buộc | Cần trước khi đọc QP-based WBC — screw theory, Jacobian, dynamics của tay máy nhiều khớp. Đã có trong `../../resources/03-robotics.md`. | [Sách miễn phí](http://hades.mech.northwestern.edu/index.php/Modern_Robotics) |
| **Underactuated Robotics** (Tedrake), chương ZMP/legged robots | 🟡 Nền tảng | Giải thích ZMP, dáng đi, điều khiển robot hai chân theo hướng model-based — bối cảnh để hiểu vì sao learning-based WBC là bước tiến. | [underactuated.mit.edu](https://underactuated.csail.mit.edu) |
| **ETH RSL — Legged Robotics lectures** | 🟡 Video bài giảng | Bài giảng công khai về dynamics/control robot chân, chuẩn academia. | [rsl.ethz.ch/education-students/lectures](https://rsl.ethz.ch/education-students/lectures.html) |

> 💡 **Cần research thêm** (mentor lưu ý phần này còn mở): các bài WBC học sâu tiền nhiệm của SONIC — **PHC (Perpetual Humanoid Control)**, **OmniH2O**, **H2O**, **ASAP** (RSS 2025, [LeCAR-Lab/ASAP](https://github.com/LeCAR-Lab/ASAP), giải quyết sim-to-real gap cho kỹ năng agile). Đọc `humanoid-wbc-review` mục "Motion Tracking-based Control" để tìm thêm — đây chính là chỗ literature map của bạn nên mở rộng.

---

## C. Video hướng dẫn

| Video/Playlist | Ngôn ngữ | Nội dung |
|---|---|---|
| **Skyentific — "How to Build Humanoid Robots Using NVIDIA Isaac Lab"** | Tiếng Anh | Series thực hành từ số 0: dựng robot 2 chân, huấn luyện policy đi trong Isaac Lab. Rất hợp để *thấy* WBC học sâu hoạt động trước khi đọc lý thuyết sâu. |
| **Steve Brunton — Control Bootcamp** (YouTube) | Tiếng Anh | Trực giác về control theory (state-space, LQR) — bổ trợ nếu Modern Robotics quá hàn lâm. Đã có trong `../../resources/03-robotics.md`. |
| **ETH Zurich RSL lecture recordings** | Tiếng Anh | Bài giảng chính quy về legged robot dynamics/control. |

> ⚠️ Chưa tìm được video/tài liệu tiếng Việt chuyên sâu về WBC học sâu — đây là mảng rất mới (SONIC công bố 11/2025). Nếu tìm được kênh Việt Nam làm nội dung này, bổ sung vào bảng trên.

---

## D. Lộ trình từng bước cho người mới bắt đầu

1. **(0.5 tuần)** Nếu chưa vững kinematics: đọc Modern Robotics chương 1–4 (vị trí, vận tốc khớp) — không cần làm hết bài tập, chỉ cần hiểu Jacobian và forward/inverse kinematics là gì.
2. **(0.5 tuần)** Đọc `humanoid-wbc-review` — chỉ đọc README + taxonomy 4 paradigm, chưa cần đọc hết 100 paper. Ghi vào literature map 1 dòng cho mỗi paradigm.
3. **(1 tuần)** Đọc SONIC pass-2 (nắm ý chính: motion tracking = task học được, không phải reward thủ công; decoupled = tách planner khỏi motor skill). Ghi chú câu hỏi chưa hiểu.
4. **(3–5 ngày)** Xem 2–3 tập đầu playlist Skyentific, làm theo để có cảm giác thực tế robot 2 chân "học đi" trông như thế nào trong Isaac Lab.
5. **(1 tuần)** Đọc SONIC pass-3 đầy đủ, đặc biệt phần kiến trúc token space nối với teleop/VLA — đây là cầu nối sang `06-vla-groot-sonic/`.
6. **(checkpoint)** Đọc code (không cần chạy train từ đầu) trong `GR00T-WholeBodyControl` repo, tìm phần định nghĩa observation/action space của WBC — đối chiếu với những gì paper mô tả.

---

## Liên kết chéo

- Retargeting output (`02-motion-retargeting/`) chính là **input** cho việc huấn luyện WBC (motion tracking cần quỹ đạo robot tham chiếu).
- Huấn luyện thực tế nằm ở `04-imitation-learning-rl/` (thuật toán RL) và `05-simulation-mujoco-isaaclab/` (nơi chạy).
- Kết nối với VLA ở `06-vla-groot-sonic/`.
