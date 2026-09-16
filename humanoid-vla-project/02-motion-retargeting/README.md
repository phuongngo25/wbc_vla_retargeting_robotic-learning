# 02 — Motion Retargeting

> Retargeting là bài toán: cho một chuyển động của **người** (khác hình dạng, số bậc tự do, tỷ lệ cơ thể), tính ra chuyển động tương ứng cho **robot humanoid** (G1, H1...) sao cho vẫn giữ được "ý nghĩa" của động tác (dáng đi, tư thế, tiếp xúc chân/tay) mà không vi phạm giới hạn vật lý của robot.

---

## A. Khái niệm cần nắm, theo thứ tự

1. **Vì sao không thể copy trực tiếp góc khớp:** người có ~200+ bậc tự do (mô hình SMPL/SMPL-X), robot G1 có ~23–43 khớp tuỳ cấu hình; tỷ lệ tay/chân/thân khác nhau; giới hạn góc khớp và tốc độ motor khác nhau.
2. **Skeleton mapping:** ánh xạ các khớp tương ứng (vai người ↔ vai robot...), scale theo tỷ lệ cơ thể.
3. **Inverse Kinematics (IK) per-frame:** với mỗi frame chuyển động, giải IK để tìm góc khớp robot khớp với vị trí đầu mút (bàn tay, bàn chân, đầu) đã scale.
4. **Ràng buộc vật lý:** ổn định tiếp xúc chân (foot contact), không vi phạm giới hạn khớp, giới hạn tốc độ motor (velocity limiting) — nếu bỏ qua, chuyển động retarget được nhưng robot thật không thực thi nổi.
5. **Hai trường phái retargeting:**
   - **Tối ưu hoá theo từng frame (analytic/IK-based):** nhanh, chạy real-time trên CPU — đại diện: **GMR**.
   - **Tối ưu hoá batch bằng GPU / kinodynamic:** chính xác hơn, tính cả động lực học, chạy trên GPU — đại diện: **SOMA-retargeter** (dùng NVIDIA Warp).
   - **Học sâu (residual/RL-based):** học một policy sửa lỗi retargeting hoặc học trực tiếp tracking — đây là ranh giới nối sang `01-whole-body-control/` và `04-imitation-learning-rl/`.

---

## B. Tài liệu & công cụ chính

| Công cụ | Vai trò | Điểm mạnh | Link |
|---|---|---|---|
| **GMR** (General Motion Retargeting, YanjieZe) | 🔴 Công cụ chính, mentor chỉ định | Real-time trên CPU (35–70 FPS), hỗ trợ input đa dạng: SMPL-X (AMASS, OMOMO), BVH (LAFAN1, Nokov), FBX (OptiTrack), streaming trực tiếp (Xsens, OptiTrack), cả video đơn mắt qua GVHMR. Output tương thích 17+ robot (Unitree G1/H1, Booster, Fourier, Talos...). Chấp nhận tại ICRA 2026. | [GitHub](https://github.com/YanjieZe/GMR) |
| **SOMA-retargeter** (NVIDIA) | 🔴 Công cụ thứ hai, mentor chỉ định | Chuyển BVH (định dạng SOMA-skeleton) → animation khớp robot bằng 5 bước: đọc BVH → scale khớp → giải IK từng frame (GPU, Newton + NVIDIA Warp) → ổn định tiếp xúc chân + giới hạn khớp → xuất CSV (root pose + joint values). Hỗ trợ sẵn G1, H2, Booster T1, AGIBot X2Ultra/A3T3. | [GitHub](https://github.com/NVIDIA/soma-retargeter) |
| **GR00T-WholeBodyControl — data prep docs** | 🟡 Tham khảo | Mô tả cách dữ liệu retarget (từ BONES-SEED) được dùng làm target huấn luyện SONIC — đọc để hiểu retargeting không phải bước "để có video đẹp" mà là **bước sinh nhãn** cho RL. | [Docs](https://nvlabs.github.io/GR00T-WholeBodyControl/) |

> 💡 **Cần research thêm** (mentor lưu ý): retargeting học sâu / residual (không chỉ IK thuần) — tìm trong `humanoid-wbc-review` (`../01-whole-body-control/`) mục datasets & retargeting tools, và tìm citation của GMR trên Semantic Scholar để xem hướng nào đang phát triển tiếp (ví dụ xử lý tương tác vật thể phức tạp từ OMOMO, nơi retargeting phải giữ cả quan hệ tay–vật thể, không chỉ tư thế cơ thể).

---

## C. Video hướng dẫn

Hiện chưa có video hướng dẫn chính thức dài cho GMR/SOMA-retargeter (cả hai đều là repo mã nguồn, tài liệu chủ yếu ở README + docs). Cách học thực hành hiệu quả nhất:

- Đọc kỹ README + thư mục `examples/` của GMR (có GIF minh hoạ input→output).
- Xem playlist Skyentific (đã nêu ở `01-whole-body-control/`) — một số tập có đoạn minh hoạ retargeting trước khi train policy.
- Nếu cần trực giác IK trước: xem lại bài giảng kinematics của Cyrill Stachniss (`../../resources/03-robotics.md`) — retargeting về bản chất là IK ứng dụng.

---

## D. Lộ trình từng bước cho người mới bắt đầu

1. **(2–3 ngày)** Cài GMR: `conda create -n gmr python=3.10 && conda activate gmr && pip install -e .`. Chạy thử một demo có sẵn trong repo trên dữ liệu mẫu đi kèm.
2. **(2–3 ngày)** Tải một sequence AMASS nhỏ (xem `03-human-motion-datasets/`), chạy GMR retarget sang Unitree G1, xem kết quả trong viewer.
3. **(2–3 ngày)** Lặp lại với một file LAFAN1 (BVH) — so sánh: động tác phức tạp (nhảy, đánh nhau) có bị méo khớp hoặc mất cân bằng không?
4. **(1 tuần)** Cài SOMA-retargeter, chạy cùng 1 sequence qua cả hai công cụ, so sánh trực quan: tốc độ, độ mượt, độ chính xác tiếp xúc chân.
5. **(checkpoint)** Viết một note ngắn (dùng `../../templates/paper-reading-note.md` hoặc tự do): liệt kê 3 loại chuyển động mà retargeting IK-thuần thất bại hoặc cho kết quả kém — đây là dấu hiệu đầu tiên của "khoảng trống nghiên cứu" (điền vào `literature-map.md`).

---

## Liên kết chéo

- Input lấy từ `03-human-motion-datasets/`.
- Output retarget là dữ liệu huấn luyện cho `01-whole-body-control/` và `04-imitation-learning-rl/`.
- Retargeting real-time (GMR) cũng dùng trực tiếp trong teleoperation — xem `08-real-robot-deployment/`.
