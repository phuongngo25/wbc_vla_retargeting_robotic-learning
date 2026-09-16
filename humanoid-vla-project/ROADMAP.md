# ROADMAP — Humanoid VLA Project (WBC · Retargeting · GR00T/SONIC)

*Lộ trình gợi ý: ~14 tuần, giả định ~10–15 giờ/tuần. Không tuyến tính cứng nhắc — điều chỉnh theo phần cứng bạn có (có G1 thật hay chỉ chạy simulation).*

---

## Nguyên tắc

- **Đọc trước khi code, chạy demo trước khi huấn luyện, huấn luyện nhỏ trước khi scale.** Đừng nhảy thẳng vào train SONIC từ đầu — đó là 21k GPU-hour, không phải bài tập cá nhân.
- Mỗi tuần: viết log ngắn vào `logs/` (copy `../../templates/weekly-log.md`). Mỗi paper T1 đọc kỹ: dùng `../../templates/paper-reading-note.md`.
- Mục tiêu cuối cùng của dự án không phải "chạy được demo" mà là **hiểu được vì sao kiến trúc decoupled WBC + VLA lại là hướng đi đúng**, và **tìm ra một khoảng trống nghiên cứu cụ thể** (dùng `../../templates/literature-map.md`) để đề xuất với mentor.

---

## Phase 0 — Định hướng & hạ tầng (Tuần 1)

- Đọc `README.md` của thư mục này tới cuối — nắm sơ đồ pipeline.
- Đọc toàn bộ mục **"Getting Started"** trên [nvlabs.github.io/GR00T-WholeBodyControl](https://nvlabs.github.io/GR00T-WholeBodyControl/getting_started/).
- Cài môi trường: Ubuntu + CUDA, clone repo `NVlabs/GR00T-WholeBodyControl`, cài MuJoCo (nhẹ, làm trước).
- Đọc lướt (pass-1) hai paper trụ cột: SONIC ([2511.07820](https://arxiv.org/abs/2511.07820)) và GR00T N1 ([2503.14734](https://arxiv.org/abs/2503.14734)) — chưa cần hiểu sâu, chỉ cần nắm bức tranh.

**Done khi:** môi trường chạy được `import mujoco` không lỗi, và bạn viết được 3–5 câu mô tả sơ đồ pipeline bằng lời của mình.

---

## Phase 1 — Dữ liệu chuyển động người (Tuần 2)

→ `03-human-motion-datasets/`

- Hiểu body model SMPL/SMPL-X là gì, khác BVH/FBX ở điểm nào.
- Đăng ký tài khoản AMASS, tải một subset nhỏ (ví dụ CMU hoặc HumanEva).
- Tải LAFAN1 (public, không cần đăng ký) — thử mở bằng BVH viewer.
- Đọc mô tả OMOMO và BONES-SEED để biết dataset human-object-interaction và dataset độc quyền của NVIDIA khác gì AMASS/LAFAN1.

**Done khi:** bạn visualize được ít nhất 1 sequence AMASS và 1 sequence LAFAN1 trên máy mình.

---

## Phase 2 — Retargeting (Tuần 3–4)

→ `02-motion-retargeting/`

- Cài `GMR` (YanjieZe), retarget 1 sequence AMASS sang Unitree G1, xem trong viewer.
- Retarget tiếp 1 sequence LAFAN1 (định dạng BVH) — so sánh chất lượng.
- Đọc code `soma-retargeter` để hiểu retargeting bằng tối ưu hoá IK trên GPU (Warp) khác gì cách tiếp cận CPU real-time của GMR.
- Viết note ngắn: retargeting thất bại ở loại chuyển động nào (nhảy, ngã, tương tác vật thể)? Vì sao?

**Done khi:** bạn có ≥3 file quỹ đạo G1 retarget từ 3 nguồn dữ liệu khác nhau, và giải thích được sự khác nhau giữa retargeting bằng IK tối ưu hoá và retargeting bằng học sâu (residual/RL-based).

---

## Phase 3 — Simulation & nền tảng RL/IL (Tuần 5–7)

→ `05-simulation-mujoco-isaaclab/` song song `04-imitation-learning-rl/`

- Nếu chưa vững RL: học nhanh CS285 (Levine) chương policy gradient + imitation learning — xem `../resources/03-robotics.md` mục A.
- Cài Isaac Lab, chạy tutorial humanoid có sẵn (H1/G1 flat-terrain walking, RSL-RL/PPO).
- Cài MuJoCo Playground, chạy `humanoid-walk` task mẫu (GPU-accelerated, JAX).
- Đọc paper T1: DeepMimic, AMP — hiểu vì sao "motion tracking reward" thay thế reward thủ công.
- Đọc kỹ SONIC (pass-3 lần này) — chú ý phần "unified token space" nối motion tracking với VLA.

**Done khi:** bạn huấn luyện được (hoặc chạy inference) một policy walking đơn giản trong ít nhất 1 simulator, và giải thích được decoupled WBC nghĩa là gì trong kiến trúc SONIC.

---

## Phase 4 — VLA: GR00T N1 → N1.7 (Tuần 8–9)

→ `06-vla-groot-sonic/`

- Đọc kỹ GR00T N1 (pass-3). So sánh N1.5/N1.6/N1.7 (đổi gì, vì sao).
- Tải checkpoint `nvidia/GR00T-N1.7-3B` từ HuggingFace, chạy inference trên 1 task mẫu (theo hướng dẫn chính thức).
- Đọc phần "VLA workflows" trong tutorials của `GR00T-WholeBodyControl` — hiểu cách N1.x (ngôn ngữ+ảnh → task cấp cao) gọi SONIC (WBC cấp thấp) thực thi.
- So sánh nhanh với OpenVLA/RT-2/π₀ (đã có trong `../resources/08-core-reading-list.md` Track C2) — điểm khác biệt của GR00T là gì (humanoid whole-body vs tay máy cố định)?

**Done khi:** bạn chạy được inference GR00T N1.x trên máy mình (hoặc trên Colab/cloud GPU nếu máy yếu), và vẽ được sơ đồ luồng dữ liệu ngôn ngữ→hành động→WBC.

---

## Phase 5 — Policy Evaluation (Tuần 10)

→ `07-policy-evaluation/`

- Thiết kế bộ benchmark nhỏ để đánh giá policy đã có (tracking error, success rate, độ mượt chuyển động).
- Đọc `../resources/07-experiments-and-rigor.md` — áp dụng nguyên tắc thiết kế thí nghiệm/thống kê vào đánh giá robot policy (không chỉ báo cáo 1 con số).
- Tìm hiểu "sim-to-real gap" được đo thế nào trong SONIC (họ báo cáo zero-shot 100% success trên 50 trajectory thật — đọc kỹ phần thiết kế thí nghiệm này).

**Done khi:** bạn có 1 bảng metric đánh giá tự thiết kế, áp dụng được lên policy Phase 3.

---

## Phase 6 — Teleoperation & Deployment (Tuần 11–13, cần phần cứng)

→ `08-real-robot-deployment/`

- Cài XRoboToolkit, thử teleoperation trong simulation trước (không cần robot thật).
- Nếu có G1: theo sát 2 tutorial chính thức — [VR Teleop Setup](https://nvlabs.github.io/GR00T-WholeBodyControl/getting_started/vr_teleop_setup.html) và [VR Whole-Body Teleop](https://nvlabs.github.io/GR00T-WholeBodyControl/tutorials/vr_wholebody_teleop.html).
- Thu một bộ dữ liệu teleop nhỏ (vài phút), dùng nó để fine-tune hoặc kiểm tra policy.
- Xuất policy sang ONNX, thử deploy pipeline (kể cả trong sim trước khi lên robot thật).

**Done khi:** bạn hoàn thành 1 vòng teleop → thu dữ liệu → deploy thử, dù chỉ trong simulation.

---

## Phase 7 — Tổng hợp & đề xuất hướng nghiên cứu (Tuần 14)

- Hoàn thiện `templates/literature-map.md` riêng cho dự án này (nhóm paper theo: classical WBC / learning-based WBC / retargeting / VLA integration).
- Xác định 2–3 khoảng trống nghiên cứu cụ thể (ví dụ: retargeting cho tương tác vật thể phức tạp, đánh giá robustness của VLA khi robot gặp địa hình chưa thấy, chi phí teleop data để fine-tune N1.7...).
- Trình bày với mentor: sơ đồ pipeline + 2–3 hướng đề xuất + lý do chọn.

**Done khi:** bạn có 1 slide/note trình bày được với mentor về hướng đi tiếp theo.
