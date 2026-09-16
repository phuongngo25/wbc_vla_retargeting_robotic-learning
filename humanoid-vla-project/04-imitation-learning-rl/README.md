# 04 — Imitation Learning & Reinforcement Learning

> Đây là "động cơ" biến dữ liệu chuyển động đã retarget (`02-motion-retargeting/`) thành một policy điều khiển thực sự chạy được (`01-whole-body-control/`). File này **không lặp lại** danh sách paper RL/IL tổng quát đã có ở `../../resources/08-core-reading-list.md` Track C2/C3 — mà tập trung vào phần **đặc thù cho humanoid motion tracking**.

---

## A. Khái niệm cần nắm, theo thứ tự

1. **Ôn nhanh nền tảng (nếu chưa vững):** Policy Gradient, PPO, SAC — đã có ở Track C3 trong reading list gốc. Nếu chưa từng cài đặt PPO, làm 1 bài tập nhỏ (CartPole/gym) trước khi đọc phần dưới.
2. **Reward thủ công vs. Motion-tracking reward:** RL cổ điển cho robot cần kỹ sư *thiết kế tay* reward (giữ thăng bằng, tốc độ tiến...) — rất khó cho chuyển động tự nhiên. **Motion tracking** thay reward thủ công bằng "độ giống" giữa pose robot và pose tham chiếu (đã retarget) mỗi frame — biến bài toán RL thành **imitation learning có giám sát dày đặc**.
3. **DeepMimic (2018):** paper khai sinh ý tưởng dùng RL để "bắt chước" 1 clip mocap cụ thể cho nhân vật ảo/robot — nền tảng khái niệm cho toàn bộ hướng motion-tracking sau này.
4. **AMP — Adversarial Motion Priors (2021):** thay vì reward = khoảng cách pose chính xác (dễ overfit 1 clip), dùng discriminator (giống GAN) để đánh giá "chuyển động này có tự nhiên/giống người không" — cho phép policy tổng quát hoá qua nhiều clip, không cần bám chính xác từng frame.
5. **Từ 1 clip → nhiều clip → generalist:** PHC, OmniH2O, ASAP, và cuối cùng SONIC — đường phát triển từ "bắt chước 1 động tác" tới "một policy theo dõi bất kỳ chuyển động nào trong bộ dữ liệu khổng lồ, tổng quát hoá sang chuyển động chưa thấy".
6. **Sim-to-real cho humanoid:** domain randomization (đã có ở Track C3), nhưng riêng cho humanoid còn có: residual learning để sửa lỗi mô hình động lực học, và fine-tuning ngắn trên robot thật (ASAP làm chính xác việc này — "aligning simulation and real-world physics").
7. **Imitation learning phi-RL (behavior cloning hiện đại):** Diffusion Policy, ACT (đã có Track C2) — dùng khi có dữ liệu **teleoperation trực tiếp trên robot thật** (không qua simulation), liên hệ `08-real-robot-deployment/`.

---

## B. Tài liệu bổ sung (humanoid-specific, ngoài Track C2/C3)

| Paper/Tài liệu | Tầng | Vì sao đọc | Link |
|---|---|---|---|
| **DeepMimic** — Example-Guided Deep RL of Physics-Based Character Skills (SIGGRAPH 2018) | 🔴 | Paper gốc của motion-tracking-as-RL-task. Đọc trước AMP/SONIC. | Tìm "DeepMimic Peng SIGGRAPH 2018" |
| **AMP** — Adversarial Motion Priors for Stylized Physics-Based Character Control (SIGGRAPH 2021) | 🔴 | Discriminator-based reward — ý tưởng lõi được nhiều WBC hiện đại kế thừa. | Tìm "AMP Adversarial Motion Priors Peng 2021" |
| **ASAP** — Aligning Simulation and Real-World Physics for Learning Agile Humanoid Whole-Body Skills (RSS 2025) | 🔴 | Giải quyết trực diện sim-to-real gap cho kỹ năng agile (nhảy, xoay người) trên humanoid thật. | [GitHub LeCAR-Lab/ASAP](https://github.com/LeCAR-Lab/ASAP) |
| **PHC / OmniH2O / H2O** | 🟡 | Các bước trung gian giữa DeepMimic/AMP và SONIC — tìm qua `humanoid-wbc-review` mục Motion Tracking-based Control để có danh sách và link đầy đủ. | Xem `../01-whole-body-control/` |
| **SONIC** (đọc lại, lần này tập trung phần thiết kế reward/training) | 🔴🔴 | Đã giới thiệu ở `01-whole-body-control/` — ở đây đọc kỹ phần Method để hiểu **cụ thể thuật toán RL** họ dùng để scale tới 42M tham số. | [arXiv:2511.07820](https://arxiv.org/abs/2511.07820) |

> 💡 **Danh sách Track C2/C3 gốc để ôn lại song song:** Diffusion Policy ([arXiv:2303.04137](https://arxiv.org/abs/2303.04137)), ACT/ALOHA ([arXiv:2304.13705](https://arxiv.org/abs/2304.13705)), PPO ([arXiv:1707.06347](https://arxiv.org/abs/1707.06347)), SAC ([arXiv:1801.01290](https://arxiv.org/abs/1801.01290)), Domain Randomization ([arXiv:1703.06907](https://arxiv.org/abs/1703.06907)) — xem `../../resources/08-core-reading-list.md`.

---

## C. Video hướng dẫn

| Video/Khoá học | Ngôn ngữ | Nội dung |
|---|---|---|
| **CS285 — Deep RL (Sergey Levine, Berkeley)** | Tiếng Anh | Đặc biệt chương Imitation Learning (đầu khoá) và Policy Gradient/Actor-Critic. Video + slide + bài tập công khai. Đã có ở `../../resources/03-robotics.md`. | [rail.eecs.berkeley.edu/deeprlcourse](https://rail.eecs.berkeley.edu/deeprlcourse/) |
| **Xu Peng (tác giả DeepMimic/AMP) — talk/demo videos** | Tiếng Anh | Tìm "DeepMimic SIGGRAPH talk" và "AMP adversarial motion priors talk" trên YouTube — có video demo trực quan rất dễ hiểu trước khi đọc paper. |
| **HuggingFace LeRobot course/tutorials** | Tiếng Anh | Thực hành imitation learning (ACT, Diffusion Policy) end-to-end trên robot tay máy — kỹ năng nền để hiểu phần behavior cloning khi bạn tới `08-real-robot-deployment/`. |

---

## D. Lộ trình từng bước cho người mới bắt đầu

1. **(3–5 ngày, nếu cần)** Cài đặt PPO từ đầu (hoặc dùng stable-baselines3) trên CartPole/LunarLander — sanity check để chắc bạn hiểu vòng lặp RL cơ bản trước khi vào bài toán phức tạp.
2. **(2–3 ngày)** Xem video demo DeepMimic + AMP (không cần đọc paper trước), để có trực giác "reward giống chuyển động mẫu" trông như thế nào khi robot học.
3. **(1 tuần)** Đọc DeepMimic pass-3, AMP pass-2. Ghi chú: AMP giải quyết vấn đề gì mà DeepMimic không giải quyết được?
4. **(3–5 ngày)** Đọc ASAP pass-2 — tập trung câu hỏi: sim-to-real gap được đo và sửa bằng cách nào (không cần hiểu hết chi tiết toán).
5. **(1 tuần)** Đọc lại SONIC, lần này pass-3 phần Method — đối chiếu với DeepMimic/AMP: SONIC kế thừa gì, cải tiến gì để scale lên 42M tham số và 700 giờ dữ liệu.
6. **(checkpoint, thực hành)** Trong `05-simulation-mujoco-isaaclab/`, chạy 1 task motion-tracking đơn giản (ví dụ MuJoCo Playground `humanoid-walk`) và thử đổi reward từ "tốc độ tiến" sang "khoảng cách tới 1 pose tham chiếu" — tự tay cảm nhận sự khác biệt giữa RL cổ điển và motion-tracking RL.

---

## Liên kết chéo

- Input huấn luyện lấy từ `02-motion-retargeting/` (quỹ đạo robot đã retarget).
- Nơi chạy thực nghiệm: `05-simulation-mujoco-isaaclab/`.
- Kết quả huấn luyện chính là "policy WBC" mô tả ở `01-whole-body-control/`.
- Behavior cloning trên dữ liệu teleop thật liên hệ `08-real-robot-deployment/`.
