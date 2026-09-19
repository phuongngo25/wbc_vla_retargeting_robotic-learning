# Bài giảng: MuJoCo Playground — kiến trúc GPU-accelerated (MJX/JAX)

*(Thuộc mảng: Simulation: MuJoCo & Isaac Lab)*

## 🎯 Mục tiêu bài học

- Giải thích được điểm nghẽn tốc độ cốt lõi của MuJoCo CPU gốc khi dùng cho RL, và vì sao multiprocessing CPU không giải quyết triệt để.
- Mô tả được MJX khác MuJoCo CPU gốc thế nào: biểu diễn hàng nghìn instance thành 1 tensor, tính toán vector hoá song song trên GPU/TPU.
- Tính tay được một ví dụ so sánh wall-clock time giữa thu thập dữ liệu tuần tự (CPU) và song song (GPU/MJX) cho cùng một số lượng environment step.
- Giải thích được vì sao "song song hoá tăng tốc RL" không phải vì GPU tính nhanh hơn CPU cho MỘT phép tính, mà vì số mẫu thu thập mỗi bước tăng theo cấp số nhân.
- Mô tả được cấu trúc một task humanoid có sẵn trong MuJoCo Playground (MJCF + reward + termination condition).
- Nêu được vị trí của MuJoCo Warp (dự án Newton) trong bức tranh MJX/Isaac Lab hiện tại — một cập nhật kiến trúc quan trọng 2025–2026.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Bài "MJCF" vừa học cho thấy file mô tả robot có thể tái sử dụng giữa MuJoCo CPU và một runtime GPU khác — MuJoCo Playground chính là runtime GPU đó. Nếu bạn đã cài GMR và retarget được vài file (theo `HUONG-DAN-THUC-HANH-RETARGETING.md`), bước tiếp theo tự nhiên là **huấn luyện một policy** để robot thực sự dùng chuyển động đã retarget đó (theo dõi/tracking nó) trong mô phỏng vật lý — và huấn luyện RL cần hàng triệu tới hàng tỷ bước mô phỏng, một khối lượng không thể chạy tuần tự trên CPU trong thời gian hợp lý. MuJoCo Playground là câu trả lời "nhẹ, không cần hạ tầng NVIDIA-GPU-datacenter" cho bài toán này — quan trọng đặc biệt nếu máy bạn dùng để nghiên cứu không có sẵn cụm Isaac Lab (như đã phân tích trong `HUONG-DAN-THUC-HANH-RETARGETING.md`).

## 🧠 Trực giác

### Góc nhìn 1: Một đầu bếp nấu 1 nồi lớn phục vụ 4096 người cùng lúc, thay vì nấu 4096 nồi nhỏ tuần tự

MuJoCo CPU gốc giống một đầu bếp nấu từng suất ăn một cách tuần tự — muốn phục vụ 4096 người, phải nấu 4096 lần, mỗi lần một nồi nhỏ. MJX giống việc thiết kế lại quy trình nấu để **một nồi cực lớn** (GPU) nấu đồng thời phần ăn cho cả 4096 người trong cùng một mẻ — vì các phần ăn giống hệt nhau về công thức (cùng loại robot, cùng luật vật lý), việc "nấu chung một mẻ lớn" khả thi và nhanh hơn nhiều so với 4096 lần nấu riêng.

**Giới hạn của loại suy này:** một nồi ăn vật lý thật có giới hạn kích thước cứng; "kích thước nồi" của GPU (số instance song song tối đa) bị giới hạn bởi bộ nhớ GPU (VRAM), không phải giới hạn vật lý cố định — có thể tăng/giảm tuỳ cấu hình phần cứng, khác với một cái nồi thật không thể "co giãn" theo nhu cầu.

### Góc nhìn 2: In hàng loạt bằng máy in offset (in nhiều bản cùng lúc từ một khuôn) so với in từng tờ bằng máy in laser

Máy in laser in từng tờ một, tuần tự — nhanh cho một tờ, chậm khi cần hàng nghìn bản giống hệt nhau. Máy in offset chuẩn bị một khuôn in (giống việc MJX "biên dịch" một môi trường MuJoCo thành dạng vector hoá phù hợp GPU) rồi in ra hàng nghìn bản **gần như đồng thời** từ đúng khuôn đó. Chi phí "làm khuôn" ban đầu (biên dịch JAX, compile kernel) cao hơn in một tờ đơn lẻ, nhưng khi cần số lượng lớn, offset áp đảo hoàn toàn về tốc độ trung bình mỗi bản.

**Giới hạn của loại suy này:** máy in offset in ra các bản **giống hệt nhau tuyệt đối**; các instance robot trong MJX tuy chạy cùng một "khuôn" vật lý nhưng mỗi instance có trạng thái (qpos/qvel) độc lập, hành động độc lập theo policy — chúng "giống nhau về luật chơi" chứ không "giống nhau về kết quả", khác hẳn các bản in offset thực sự đồng nhất.

## 📐 Định nghĩa chính xác

**Vấn đề gốc:** MuJoCo (bản gốc) là thư viện C/C++ chạy trên **CPU**, mô phỏng **một instance tại một thời điểm**. RL cần thu thập hàng triệu/tỷ bước mô phỏng (environment steps) để policy hội tụ — chạy tuần tự là điểm nghẽn lớn nhất; chạy đa tiến trình CPU (multiprocessing) vẫn bị giới hạn bởi số nhân CPU (thường vài chục, hiếm khi hơn 100).

**MJX** là bản cài đặt lại phần lớn thuật toán vật lý MuJoCo bằng **JAX** — thư viện tính toán số của Google hỗ trợ vector hoá song song trên GPU/TPU và tự động vi phân (autodiff). Khác biệt cốt lõi:

```text
MuJoCo CPU:  1 vòng lặp = 1 instance robot, chạy tuần tự trên 1 nhân CPU
MJX (GPU):   N instance robot (mỗi cái là 1 bản sao độc lập cùng 1 môi trường)
             được xếp thành MỘT TENSOR LỚN
             → toàn bộ phép tính vật lý (tích phân động lực học, giải contact)
               thực hiện ĐỒNG THỜI trên tất cả N instance
               nhờ kiến trúc SIMD/song song hàng loạt của GPU
```

**MuJoCo Playground** (Google DeepMind, [arXiv:2502.08844](https://www.alphaxiv.org/abs/2502.08844)) là bộ môi trường RL đóng gói sẵn trên nền MJX — không cần tự viết code JAX, chỉ gọi task có sẵn (`humanoid-stand`, `humanoid-walk`, `humanoid-run`...) cùng một thuật toán RL có sẵn (PPO/SAC), việc huấn luyện song song trên GPU diễn ra tự động.

## ⚙️ Cơ chế hoạt động — từng bước

```text
┌────────────────────────────────────────────────────────────┐
│ Khởi tạo: load MJCF của robot (đã học ở bài trước)            │
│  → biên dịch (JIT compile qua JAX) thành hàm vật lý vector hoá │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ Tạo N bản sao song song (ví dụ N=4096)                         │
│  qpos, qvel của cả N instance → xếp thành 1 tensor [N, D]      │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ VÒNG LẶP RL (mỗi bước):                                        │
│  1. Policy (mạng nơ-ron) nhận batch quan sát [N, obs_dim]      │
│     → xuất batch hành động [N, action_dim] — CŨNG song song    │
│  2. MJX thực hiện 1 bước vật lý cho CẢ N instance ĐỒNG THỜI     │
│     (tích phân động lực học + giải contact, vector hoá GPU)    │
│  3. Trả về batch [N] trạng thái mới + reward + done             │
│  4. Thu thập N mẫu (obs, action, reward, next_obs) — 1 BƯỚC     │
│     nhưng N mẫu dữ liệu cùng lúc                                │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ Cập nhật policy (PPO) định kỳ sau khi gom đủ batch dữ liệu     │
│  Instance nào "done" (robot ngã) → reset độc lập, tiếp tục      │
└──────────────────────────────────────────────────────────────┘
```

Cấu trúc một task humanoid có sẵn (`humanoid-stand`/`humanoid-walk`/`humanoid-run`) gồm 3 thành phần đóng gói sẵn: (1) file MJCF của robot chuẩn, (2) hàm reward (ví dụ `stand` thưởng giữ thân thẳng ổn định; `walk`/`run` thưởng theo tốc độ tiến + giữ thăng bằng), (3) điều kiện kết thúc episode (robot ngã → reset). Khi huấn luyện, hàng nghìn bản sao "thử" hành động ngẫu nhiên ban đầu (phần lớn ngã ngay) — nhưng vì có hàng nghìn mẫu mỗi bước, policy học rất nhanh cách đứng vững rồi tiến tới đi được.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số tự chọn để dễ hình dung quy mô lợi ích song song hoá)*

Giả sử một policy cần **100 triệu environment step** để hội tụ (một con số điển hình cho locomotion RL).

**Kịch bản A — CPU tuần tự, 1 instance, giả sử 2.000 step/giây** (một tốc độ hợp lý cho MuJoCo CPU đơn giản, chưa tính thời gian forward pass mạng):

```text
thời gian = 100.000.000 / 2.000 = 50.000 giây ≈ 13.9 giờ
```

**Kịch bản B — CPU multiprocessing, 32 tiến trình song song** (giả sử tăng tốc gần tuyến tính, một giả định lạc quan — thực tế thường kém hơn do overhead giao tiếp giữa tiến trình):

```text
step/giây hiệu dụng ≈ 2.000 × 32 = 64.000 step/giây
thời gian = 100.000.000 / 64.000 ≈ 1.562,5 giây ≈ 26 phút
```

**Kịch bản C — MJX/GPU, N=4096 instance song song, giả sử GPU xử lý 1 bước vật lý cho cả 4096 instance trong 8ms** (một con số minh hoạ hợp lý cho GPU hiện đại với robot vừa phải):

```text
số bước vật lý cần = 100.000.000 / 4096 ≈ 24.414 bước
thời gian = 24.414 × 0.008 giây ≈ 195,3 giây ≈ 3.26 phút
```

**So sánh:**

```text
tăng tốc B so với A ≈ 50.000 / 1.562,5 = 32 lần   (đúng bằng số tiến trình, lý tưởng)
tăng tốc C so với A ≈ 50.000 / 195,3 ≈ 256 lần
tăng tốc C so với B ≈ 1.562,5 / 195,3 ≈ 8 lần
```

**Ý nghĩa:** đây chính là con số cụ thể hoá cho câu nói "bài toán trước kia mất nhiều ngày/tuần huấn luyện trên CPU cluster giờ có thể xong trong vài giờ trên 1 GPU cá nhân" — không phải vì GPU tính một phép cộng/nhân nhanh hơn CPU hàng trăm lần, mà vì **số lượng instance chạy đồng thời** (N=4096 so với N=1 hoặc N=32) quyết định trực tiếp số mẫu thu thập được mỗi đơn vị thời gian thực (wall-clock).

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | MuJoCo CPU gốc | MuJoCo CPU multiprocessing | MuJoCo Playground (MJX/GPU) | Isaac Lab (GPU, xem bài riêng) |
|---|---|---|---|---|
| Phần cứng | CPU | CPU (nhiều nhân) | GPU/TPU | GPU (NVIDIA) |
| Số instance song song thực tế | 1 | Vài chục (giới hạn số nhân) | Hàng nghìn (giới hạn VRAM) | Hàng nghìn (giới hạn VRAM) |
| Cần viết code JAX? | Không | Không | Không (Playground đóng gói sẵn) | Không (dùng API Isaac Lab) |
| Physics + training cùng thiết bị? | N/A (không train song song) | Không — train NN vẫn thường trên GPU riêng | Có — cả JAX physics lẫn JAX/Flax NN đều trên GPU | Có (đúng ý tưởng gốc Isaac Gym) |
| Hạ tầng cần thiết | Máy CPU thường | Máy CPU nhiều nhân | 1 GPU cá nhân (kể cả laptop GPU tầm trung) | Thường cần GPU NVIDIA mạnh hơn (RTX 20/30/40-series+) |
| Định dạng robot | MJCF | MJCF | MJCF (dùng lại nguyên vẹn) | USD (cần convert từ URDF/MJCF) |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "MJX nhanh hơn vì JAX/GPU tính toán số học nhanh hơn CPU nhiều lần".** Vì sao sai: với **một** phép tính vật lý đơn lẻ (một instance), MuJoCo CPU thường không chậm hơn GPU bao nhiêu — thậm chí có overhead khởi tạo GPU/kernel launch. Lợi ích tốc độ thực sự đến từ việc **xử lý đồng thời hàng nghìn instance** trong cùng một lệnh gọi kernel GPU (song song hoá theo dữ liệu — data parallelism), như ví dụ tính tay ở trên cho thấy rõ (tăng tốc tỷ lệ gần với N, không phải với "độ nhanh" của một phép tính đơn). **Hiểu đúng:** lợi ích của MJX chỉ thể hiện rõ khi bạn cần **nhiều instance song song** (RL training); với một mô phỏng đơn lẻ (ví dụ chỉ xem lại 1 chuyển động đã retarget), MuJoCo CPU gốc hoàn toàn đủ dùng và đôi khi nhanh hơn.
2. **Hiểu nhầm: "MuJoCo Playground và Isaac Lab là hai lựa chọn cạnh tranh trực tiếp, chỉ nên học một cái".** Vì sao sai: hai công cụ có đánh đổi khác nhau về hạ tầng (MJX chạy được trên GPU cá nhân tầm trung; Isaac Lab thường cần GPU NVIDIA mạnh hơn và hệ sinh thái Omniverse/USD phức tạp hơn) và HumanoidVerse (bài giảng riêng) thậm chí được thiết kế để chạy **cùng một task/algorithm trên cả hai** làm bằng chứng robustness. **Hiểu đúng:** với người mới bắt đầu không có GPU NVIDIA mạnh, MuJoCo Playground là điểm khởi đầu thực dụng hơn; hai công cụ bổ sung cho nhau trong một pipeline nghiên cứu trưởng thành, không loại trừ nhau.

## 🏗️ Ví dụ minh hoạ trong dự án này

```text
GMR retarget AMASS/LAFAN1 → Unitree G1 (đã học, 02-motion-retargeting/)
        │
        ▼
File MJCF của G1 (đã học ở bài MJCF) — tái sử dụng NGUYÊN VẸN
        │
        ▼
MuJoCo Playground (MJX): định nghĩa 1 task "motion tracking"
  - reward: khoảng cách giữa pose robot hiện tại và pose reference đã retarget
  - termination: robot ngã hoặc lệch quá xa reference
        │
        ▼
Huấn luyện song song 4096 instance G1 cùng "cố gắng" theo dõi
  đúng chuyển động đã retarget — đây chính là "motion-tracking reward"
  đã nhắc tới ở 04-imitation-learning-rl/
```

Với máy không có GPU NVIDIA mạnh (như đã phân tích ở `HUONG-DAN-THUC-HANH-RETARGETING.md`), MuJoCo Playground vẫn có thể chạy trên CPU qua JAX CPU backend (rất chậm, N nhỏ hơn nhiều) — đủ để kiểm tra logic task/reward hoạt động đúng trước khi thuê GPU cloud để huấn luyện thật ở quy mô N lớn.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **MuJoCo Warp (MJWarp) — hội tụ kiến trúc giữa hệ sinh thái MuJoCo và NVIDIA, qua dự án Newton.** Theo repo chính thức [google-deepmind/mujoco_warp](https://github.com/google-deepmind/mujoco_warp), đây là "phiên bản GPU-accelerated của MuJoCo, thiết kế cho phần cứng NVIDIA", được **Google DeepMind và NVIDIA đồng phát triển như một phần của dự án Newton** — đúng dự án Newton (xây trên NVIDIA Warp) đã xuất hiện ở bài giảng SOMA-retargeter (`02-motion-retargeting/`)! Điều này có nghĩa: hệ sinh thái MuJoCo giờ có **hai đường tăng tốc GPU song song**: MJX (qua JAX, đã học ở bài này) và MJWarp (qua NVIDIA Warp, tối ưu riêng cho phần cứng NVIDIA, hỗ trợ cả batch ray-tracing rendering hàng triệu khung hình/giây). Theo tài liệu, MJX có thể **truy cập MJWarp làm backend** cho người dùng ưu tiên hệ sinh thái JAX, và MJWarp còn hỗ trợ tích hợp trực tiếp với **Isaac Lab và một công cụ mới tên "mjlab"** — cho thấy ranh giới giữa "hệ sinh thái MuJoCo" và "hệ sinh thái Isaac Lab/NVIDIA" (vốn được trình bày như hai lựa chọn tách biệt ở phần đầu bài) đang **hội tụ dần** ở tầng thấp nhất (physics kernel), dù API/workflow cấp cao vẫn khác nhau. [GitHub: google-deepmind/mujoco_warp](https://github.com/google-deepmind/mujoco_warp)
2. **FastTD3 (2025), "Simple, Fast, and Capable Reinforcement Learning for Humanoid Control"** ([arXiv:2505.22642](https://arxiv.org/pdf/2505.22642)) là một ví dụ thuật toán RL hiện đại được thiết kế/benchmark trực tiếp trên hạ tầng song song hoá kiểu MuJoCo Playground — cho thấy các thuật toán RL mới (không chỉ PPO mặc định, xem bài RSL-RL) tiếp tục được phát triển để tận dụng đúng kiến trúc "nhiều instance song song, tốc độ thu thập dữ liệu cao" mà MJX cung cấp.
3. **Xu hướng chung:** MuJoCo Playground tiếp tục mở rộng bộ task (locomotion, dexterous manipulation, non-prehensile manipulation) và duy trì vai trò "điểm vào nhẹ" cho robot learning không cần hạ tầng NVIDIA data-center — trong khi ở tầng kỹ thuật thấp hơn, việc hội tụ với NVIDIA Warp (MJWarp) cho thấy ranh giới công cụ "hệ sinh thái Google" vs "hệ sinh thái NVIDIA" đang mờ dần trong 2025–2026, dù đây là cập nhật kiến trúc phía sau hậu trường mà đa số người dùng thông thường của MuJoCo Playground/Isaac Lab (qua API cấp cao) chưa cần quan tâm trực tiếp.

## ❓ Câu hỏi tự kiểm tra

1. Vì sao chạy 4096 instance song song trên GPU không có nghĩa là mỗi instance được tính nhanh hơn 4096 lần so với CPU?
   <details><summary>Gợi ý đáp án</summary>Vì lợi ích không đến từ tốc độ tính MỘT phép tính (GPU không nhất thiết nhanh hơn CPU cho 1 instance đơn lẻ), mà đến từ việc xử lý ĐỒNG THỜI hàng nghìn instance trong cùng một lệnh kernel — tổng số mẫu thu thập mỗi đơn vị thời gian tăng theo N, không phải theo "độ nhanh" của từng phép tính.</details>
2. Trong ví dụ tính tay, nếu tăng N từ 4096 lên 8192 (giả sử thời gian mỗi bước vật lý không đổi 8ms — giả định lý tưởng chưa bão hoà VRAM), thời gian huấn luyện 100 triệu step thay đổi thế nào?
   <details><summary>Gợi ý đáp án</summary>Số bước cần giảm còn `100.000.000/8192 ≈ 12.207` bước → thời gian `12.207 × 0.008 ≈ 97.66s`, giảm gần một nửa so với `195.3s` ở N=4096.</details>
3. Vì sao một file MJCF viết cho MuJoCo CPU có thể tái sử dụng nguyên vẹn cho MuJoCo Playground (MJX)?
   <details><summary>Gợi ý đáp án</summary>Vì MJX là một runtime đọc cùng định dạng MJCF (không phải một định dạng file riêng biệt) — chỉ khác cách thực thi phép tính (vector hoá song song GPU thay vì tuần tự CPU), đã học ở bài MJCF.</details>
4. MuJoCo Warp (MJWarp) là một phần của dự án nào, và dự án đó có liên hệ gì với bài giảng SOMA-retargeter?
   <details><summary>Gợi ý đáp án</summary>MJWarp là một phần của dự án Newton (Google DeepMind + NVIDIA đồng phát triển) — cùng dự án Newton xây trên NVIDIA Warp đã xuất hiện trong pipeline của SOMA-retargeter (NVIDIA/soma-retargeter dùng Newton + NVIDIA Warp để giải IK trên GPU).</details>
5. Vì sao người không có GPU NVIDIA mạnh vẫn nên học MuJoCo Playground trước khi học Isaac Lab?
   <details><summary>Gợi ý đáp án</summary>Vì MJX/MuJoCo Playground có thể chạy trên GPU cá nhân tầm trung (thậm chí CPU, dù chậm hơn nhiều qua JAX CPU backend), trong khi Isaac Lab thường đòi hỏi GPU NVIDIA mạnh hơn (RTX 20/30/40-series+) và hệ sinh thái Omniverse/USD phức tạp hơn — MuJoCo Playground là điểm khởi đầu thực dụng hơn cho hạ tầng hạn chế.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** với 200 triệu environment step cần thiết, CPU tuần tự đạt 1.500 step/giây, MJX với N=2048 instance đạt thời gian mỗi bước vật lý là 6ms. Tính thời gian huấn luyện cho cả hai kịch bản (theo giờ) và tỷ lệ tăng tốc.
2. **Đọc code/tài liệu thật:** mở [repo MuJoCo Playground](https://github.com/google-deepmind/mujoco_playground) (hoặc trang alphaXiv của paper, [arXiv:2502.08844](https://www.alphaxiv.org/abs/2502.08844)), tìm một task humanoid có sẵn (ví dụ `HumanoidStand` hoặc tương đương) — đọc định nghĩa hàm reward và điều kiện termination trong code, mô tả lại bằng lời (3-5 câu) cách chúng khớp với mô tả khái niệm ở mục Cơ chế hoạt động của bài này.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

MuJoCo Playground giải quyết điểm nghẽn tốc độ của MuJoCo CPU gốc (mô phỏng tuần tự, một instance mỗi lúc) bằng MJX — bản viết lại vật lý MuJoCo bằng JAX, xếp hàng nghìn instance thành một tensor và tính toán vector hoá đồng thời trên GPU/TPU, tái sử dụng nguyên vẹn định dạng MJCF đã học ở bài trước; như ví dụ tính tay minh hoạ, lợi ích tốc độ (~256 lần so với CPU tuần tự trong ví dụ) đến từ số lượng mẫu thu thập đồng thời mỗi bước, không phải từ việc GPU tính một phép toán nhanh hơn. MuJoCo Playground đóng gói sẵn các task humanoid phổ biến (stand/walk/run) với reward và termination có sẵn, là điểm khởi đầu robot learning thực dụng cho hạ tầng không có GPU NVIDIA mạnh; một cập nhật kiến trúc quan trọng 2025-2026 là MuJoCo Warp (dự án Newton, Google DeepMind + NVIDIA đồng phát triển) đang hội tụ tầng physics-kernel của hệ sinh thái MuJoCo với hệ sinh thái NVIDIA Warp/Isaac Lab, dù hai workflow cấp cao vẫn tách biệt tại thời điểm hiện tại.
