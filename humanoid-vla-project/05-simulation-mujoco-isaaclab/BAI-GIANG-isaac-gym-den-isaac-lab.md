# Bài giảng: Isaac Gym → Isaac Lab — kiến trúc GPU-based simulation của NVIDIA

*(Thuộc mảng: Simulation: MuJoCo & Isaac Lab)*

## 🎯 Mục tiêu bài học

- Giải thích được điểm nghẽn CPU↔GPU transfer mà Isaac Gym (Makoviychuk et al. 2021) giải quyết, khác với điểm nghẽn "tuần tự một instance" mà MJX/MuJoCo Playground giải quyết.
- Tính tay được ước lượng thời gian tiết kiệm khi loại bỏ round-trip CPU↔GPU cho một khối lượng dữ liệu cụ thể.
- Mô tả được quan hệ kế thừa chính xác: Isaac Gym (2021, nghiên cứu, deprecated) → Isaac Sim/Omniverse (nền tảng tổng quát) → Isaac Lab (framework robot learning hiện hành, xây trên Isaac Sim).
- Liệt kê được 4 điểm Isaac Lab bổ sung so với ý tưởng gốc Isaac Gym: USD, Warp/CUDA-graphable environments, đa thư viện RL, WBC/teleoperation (2.3+).
- Phân biệt được rõ ràng Isaac Gym với Isaac Sim và Isaac Lab — ba cái tên dễ gây nhầm lẫn.
- Nêu được cập nhật cụ thể của Isaac Lab 2.3 (2026) về teleoperation và tại sao nó liên quan trực tiếp tới `08-real-robot-deployment/`.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Bài trước (MuJoCo Playground) giải quyết bài toán "làm sao chạy hàng nghìn instance mô phỏng song song". Isaac Gym/Isaac Lab giải quyết một bài toán **liên quan nhưng khác**: ngay cả khi bạn đã có hàng nghìn instance chạy song song, nếu physics engine và neural network training nằm trên hai thiết bị khác nhau (CPU vs GPU), việc **truyền dữ liệu qua lại** giữa chúng vẫn có thể là điểm nghẽn. Học bài này để hiểu đúng lịch sử phát triển (tránh nhầm lẫn ba cái tên Isaac Gym/Isaac Sim/Isaac Lab rất dễ gây rối cho người mới), và vì sao NVIDIA khuyến nghị Isaac Lab là "nền chính thức mà hệ sinh thái GR00T dùng" cho pipeline retargeting → training → deploy.

## 🧠 Trực giác

### Góc nhìn 1: Nhà bếp và phòng ăn ở hai toà nhà khác nhau, so với nhà bếp mở ngay trong phòng ăn

Pipeline RL truyền thống (trước Isaac Gym) giống một nhà hàng có **bếp nấu ở một toà nhà** (CPU tính vật lý) và **phòng ăn ở toà nhà khác** (GPU huấn luyện mạng) — mỗi món ăn nấu xong phải được **vận chuyển** giữa hai toà nhà (truyền dữ liệu qua PCIe bus) trước khi thực khách ăn được. Với vài món, việc vận chuyển không đáng kể; nhưng với hàng nghìn món mỗi phút (hàng nghìn environment step mỗi giây), thời gian vận chuyển cộng dồn lại có thể **vượt xa** thời gian nấu ăn thực tế. Isaac Gym giống việc thiết kế lại nhà hàng với **bếp mở ngay trong phòng ăn** (physics + training cùng trên GPU) — loại bỏ hoàn toàn bước vận chuyển.

**Giới hạn của loại suy này:** trong ví dụ nhà hàng, "nấu" và "ăn" là hai hoạt động về bản chất khác nhau; trong khi ở đây "tính vật lý" và "cập nhật mạng nơ-ron" đều có thể biểu diễn dưới cùng dạng toán học (phép toán tensor) — đây chính là lý do việc "gộp chung một nơi" (GPU) khả thi về mặt kỹ thuật, khác với việc gộp bếp/phòng ăn vốn chỉ là thay đổi vị trí vật lý, không thay đổi bản chất công việc.

### Góc nhìn 2: Viết code trên máy tính rồi mỗi lần chạy phải upload lên server khác để test, so với chạy thẳng trên máy đang gõ code

Nếu bạn viết code trên laptop nhưng phải upload lên một server khác để chạy thử mỗi lần sửa (như làm việc qua SSH chậm hoặc CI/CD nặng), chi phí "chờ upload + chờ kết quả trả về" có thể chiếm phần lớn thời gian làm việc thực tế, dù bản thân code chạy rất nhanh trên server. Isaac Gym giống việc chuyển sang **chạy thẳng trên máy đang gõ code** — không còn bước "upload rồi chờ" nào cả.

**Giới hạn của loại suy này:** upload code là một hành động rời rạc, có thể đo bằng giây; round-trip CPU↔GPU trong RL diễn ra **liên tục, hàng nghìn lần mỗi giây** trong suốt quá trình huấn luyện — độ "phiền toái" tích luỹ theo cấp số nhân so với việc chỉ upload một lần trước khi chạy.

## 📐 Định nghĩa chính xác

**Vấn đề Isaac Gym giải quyết (Makoviychuk et al. 2021, [arXiv:2108.10470](https://arxiv.org/abs/2108.10470)):** trước Isaac Gym, pipeline RL chuẩn là:

```text
Physics simulation → chạy trên CPU
Neural network training (forward/backward pass) → chạy trên GPU
Mỗi bước: dữ liệu (trạng thái robot, reward) phải TRUYỀN QUA LẠI
          giữa CPU và GPU (qua PCIe bus)
```

Với robot phức tạp và hàng nghìn môi trường song song, chi phí truyền dữ liệu CPU↔GPU trở thành **điểm nghẽn lớn hơn cả** chi phí tính vật lý hay tính mạng nơ-ron.

**Đóng góp cốt lõi của Isaac Gym:** chạy **cả physics simulation lẫn neural network training trên cùng GPU**, dữ liệu ở dạng **PyTorch tensor nằm sẵn trên GPU memory**, không bao giờ rời khỏi GPU giữa bước mô phỏng và bước cập nhật mạng — loại bỏ hoàn toàn round-trip CPU↔GPU khỏi vòng lặp huấn luyện. Paper báo cáo tăng tốc **2-3 bậc độ lớn (orders of magnitude)** so với setup CPU-sim + GPU-train truyền thống.

**Isaac Lab** — theo tài liệu chính thức NVIDIA (developer.nvidia.com/isaac/lab) — là *"a lightweight, open-source framework built on top of Isaac Sim, specifically optimized for robot learning workflows"*: không phải một engine vật lý độc lập mới, mà một lớp framework xây **trên nền Isaac Sim/Omniverse** (nền tảng mô phỏng robot tổng quát của NVIDIA — dùng cho cả sinh dữ liệu tổng hợp, kiểm định thiết kế, không chỉ RL). Isaac Lab kế thừa ý tưởng gốc "physics + training cùng GPU" từ Isaac Gym, bổ sung 4 điểm:

1. **USD (Universal Scene Description)** thay vì chỉ URDF/MJCF thuần (xem bài giảng riêng "MJCF vs URDF vs USD").
2. **GPU-optimized simulation paths built on Warp và CUDA-graphable environments** — Warp đóng vai trò tương tự JAX với MJX, hỗ trợ chạy từ máy trạm cá nhân tới cloud data-center, multi-GPU/multi-node.
3. **Tích hợp sẵn nhiều thư viện RL** — không khoá cứng vào 1 thuật toán: skrl, RLlib, rl_games, và đặc biệt **RSL-RL** (bài giảng riêng) cho locomotion/WBC.
4. **Isaac Lab 2.3** bổ sung **whole-body control (WBC) và teleoperation nâng cao** chính thức — lý do được gọi là "nền chính thức mà hệ sinh thái GR00T dùng".

```text
Isaac Gym (2021, paper nghiên cứu, ĐÃ DEPRECATED)
   → chứng minh ý tưởng "all-GPU pipeline"
        │
        ▼
Isaac Sim / Omniverse (nền tảng mô phỏng tổng quát, dựa trên USD)
        │
        ▼
Isaac Lab (framework production/nghiên cứu HIỆN HÀNH,
           xây trên Isaac Sim, mở rộng USD + đa thư viện RL + WBC/teleop)
```

NVIDIA khuyến nghị người dùng Isaac Gym cũ chuyển sang Isaac Lab để có tính năng mới nhất — **Isaac Gym không còn được phát triển tiếp**.

## ⚙️ Cơ chế hoạt động — từng bước

```text
┌────────────────────────────────────────────────────────────┐
│ TRƯỚC ISAAC GYM (pipeline truyền thống)                        │
│                                                                 │
│  CPU: tính vật lý N environment ──┐                            │
│                                    │ TRUYỀN QUA PCIe (chậm,     │
│                                    │ lặp lại MỖI BƯỚC)          │
│                                    ▼                            │
│  GPU: forward/backward pass mạng nơ-ron                        │
│                                    │                            │
│                                    │ TRUYỀN NGƯỢC LẠI action    │
│                                    ▼                            │
│  CPU: áp action, tính bước vật lý tiếp theo                    │
│  (lặp lại — mỗi vòng đều có 2 lượt truyền CPU↔GPU)             │
└──────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│ ISAAC GYM / ISAAC LAB (all-GPU pipeline)                       │
│                                                                 │
│  GPU: tính vật lý N environment (PyTorch tensor, tại chỗ)      │
│         │ (không rời GPU)                                      │
│         ▼                                                       │
│  GPU: forward/backward pass mạng nơ-ron (CÙNG GPU memory)      │
│         │ (không rời GPU)                                      │
│         ▼                                                       │
│  GPU: áp action, tính bước vật lý tiếp theo                    │
│  (toàn bộ vòng lặp KHÔNG BAO GIỜ rời khỏi GPU)                 │
└──────────────────────────────────────────────────────────────┘
```

Isaac Lab thêm một lớp trên cùng kiến trúc này:

```text
Isaac Sim / Omniverse (USD scene, PhysX solver, rendering PBR)
        │
        ▼
Isaac Lab (Python API robot-learning: task, env, reward wrapper,
           tích hợp RSL-RL/skrl/RLlib/rl_games)
        │
        ▼
Script huấn luyện của bạn (gần giống Gym/Gymnasium API quen thuộc)
```

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số tự chọn để định lượng hoá lợi ích loại bỏ round-trip CPU↔GPU — không phải số liệu thật từ paper Isaac Gym)*

Giả sử một pipeline huấn luyện có **N = 4096** environment song song, mỗi environment có vector trạng thái (observation) kích thước **D = 200** số thực (float32, 4 byte/số).

**Kích thước dữ liệu cần truyền mỗi bước (round-trip):**

```text
dữ liệu quan sát (obs) truyền GPU→CPU (nếu policy được đánh giá trên CPU)
  hoặc CPU→GPU (nếu vật lý trên CPU, mạng trên GPU):
  N × D × 4 byte = 4096 × 200 × 4 = 3.276.800 byte ≈ 3.2 MB

dữ liệu action truyền ngược lại (giả sử action_dim = 40):
  N × 40 × 4 byte = 4096 × 40 × 4 = 655.360 byte ≈ 0.64 MB

Tổng mỗi bước (2 chiều): ≈ 3.84 MB
```

**Giả sử băng thông PCIe hiệu dụng ≈ 8 GB/s** (một con số thực tế dè dặt cho PCIe khi có overhead giao thức, thấp hơn băng thông lý thuyết đỉnh):

```text
thời gian truyền mỗi bước = 3.84 MB / 8.000 MB/s = 0.00048 giây = 0.48 ms
```

**Với 100 triệu bước huấn luyện** (chia N=4096 → số vòng lặp = `100.000.000/4096 ≈ 24.414` vòng):

```text
tổng thời gian CHỈ RIÊNG truyền dữ liệu = 24.414 × 0.00048 ≈ 11.72 giây
```

**Ý nghĩa:** với ví dụ này, riêng chi phí truyền dữ liệu (chưa tính độ trễ khởi động truyền — latency, thường lớn hơn nhiều so với thời gian truyền thuần tuý cho gói tin nhỏ) đã cộng thêm gần 12 giây. Trong thực tế, **độ trễ (latency)** của mỗi lần gọi truyền dữ liệu qua PCIe (thường vài chục đến vài trăm microsecond mỗi lần gọi, không phụ thuộc kích thước gói tin nhỏ) thường áp đảo thời gian truyền thuần tuý đã tính ở trên — đây chính xác là lý do paper Isaac Gym mô tả chi phí này "trở thành điểm nghẽn lớn hơn cả tính toán vật lý hay mạng nơ-ron": không phải vì lượng dữ liệu lớn, mà vì **số lần gọi truyền lặp lại hàng chục nghìn lần**, mỗi lần đều tốn một khoản chi phí cố định (overhead) bất kể gói tin nhỏ tới đâu. *(Số liệu ở trên là ước tính minh hoạ đơn giản hoá, không phải benchmark chính thức của Isaac Gym — mục đích chỉ để cụ thể hoá cơ chế "vì sao round-trip lặp lại nhiều lần lại đắt".)*

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Isaac Gym (2021, deprecated) | Isaac Lab (hiện hành) | MuJoCo Playground/MJX (bài trước) |
|---|---|---|---|
| Trạng thái phát triển | Đã deprecated, không phát triển tiếp | Đang phát triển tích cực (2.3+ tại thời điểm học) | Đang phát triển tích cực |
| Nền tảng bên dưới | PhysX (độc lập) | Isaac Sim/Omniverse (PhysX + USD) | MuJoCo (MJX/JAX hoặc MJWarp) |
| Định dạng scene/robot | Chủ yếu URDF | USD | MJCF |
| Runtime GPU | PyTorch tensor trực tiếp | Warp + CUDA-graphable environments | JAX (MJX) hoặc NVIDIA Warp (MJWarp) |
| Thư viện RL tích hợp | Hạn chế, tự viết nhiều | Đa dạng: RSL-RL, skrl, RLlib, rl_games | Tự chọn (thường PPO/SAC viết bằng JAX/Flax) |
| WBC/Teleoperation chính thức | Không | Có (từ 2.3+) | Không (cần tự xây) |
| Hạ tầng cần thiết | GPU NVIDIA | GPU NVIDIA (thường mạnh hơn, do Omniverse nặng hơn) | GPU cá nhân tầm trung, hoặc CPU (chậm) |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "Isaac Gym, Isaac Sim, Isaac Lab là ba tên gọi khác nhau của cùng một sản phẩm qua các phiên bản".** Vì sao sai: đây là ba thứ khác nhau về bản chất — Isaac Gym là một **framework nghiên cứu độc lập** (2021, đã deprecated, không còn phát triển); Isaac Sim là **nền tảng mô phỏng tổng quát** của NVIDIA (Omniverse, dùng cho nhiều mục đích ngoài RL); Isaac Lab là **framework robot-learning** xây *trên nền* Isaac Sim. Isaac Lab không phải "Isaac Gym phiên bản mới", mà là một kiến trúc khác hẳn (xây trên Isaac Sim thay vì độc lập). **Hiểu đúng:** khi tài liệu/tutorial nói "dùng Isaac Gym", cần kiểm tra thời điểm viết — nếu là tài liệu cũ (2021-2023), rất có thể nội dung đã lỗi thời và nên tìm hướng dẫn tương đương cho Isaac Lab.
2. **Hiểu nhầm: "lợi ích tốc độ của Isaac Gym/Isaac Lab giống hệt lợi ích của MJX — đều là 'chạy nhiều instance song song trên GPU'".** Vì sao sai: cả hai đều tận dụng chạy nhiều instance song song, nhưng đóng góp *cốt lõi* mà mỗi công trình nhấn mạnh là khác nhau — MJX/MuJoCo Playground giải quyết bài toán "MuJoCo CPU chỉ chạy 1 instance tuần tự" (đã học ở bài trước); Isaac Gym giải quyết bài toán "dù đã chạy song song, dữ liệu vẫn phải di chuyển qua lại giữa CPU và GPU nếu vật lý và mạng nơ-ron nằm ở hai nơi khác nhau". **Hiểu đúng:** hai đóng góp bổ sung cho nhau (song song hoá + loại bỏ round-trip), không phải cùng một ý tưởng diễn đạt hai lần.

## 🏗️ Ví dụ minh hoạ trong dự án này

```text
GMR retarget dữ liệu (02-motion-retargeting/) → output .pkl (SMPL-X/BVH → G1)
        │
        ▼
Convert model robot: URDF gốc Unitree → USD (Isaac Sim URDF Importer,
                                              xem bài "MJCF vs URDF vs USD")
        │
        ▼
Isaac Lab: định nghĩa task motion-tracking, chạy N=4096+ instance G1
           song song trên GPU, dùng RSL-RL + PPO (bài giảng riêng)
        │
        ▼
Policy đã huấn luyện → triển khai qua Isaac Lab 2.3 teleoperation
  (Apple Vision Pro / Meta Quest / Manus gloves) hoặc lên robot thật
  (08-real-robot-deployment/)
```

Đây chính là con đường mà README của dự án gọi là "nền chính thức mà hệ sinh thái GR00T dùng" — pipeline retargeting → training → deploy của NVIDIA cho robot G1 dựa trực tiếp trên Isaac Lab, khác với con đường "nhẹ" hơn (MuJoCo Playground) phù hợp khi không có hạ tầng GPU NVIDIA mạnh.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **Isaac Lab 2.3 (2026), xây trên Isaac Sim 5.1, mở rộng mạnh về teleoperation và dexterous data collection.** Theo NVIDIA Technical Blog chính thức, bản 2.3 bổ sung: hỗ trợ chính thức teleoperation qua **Apple Vision Pro** để thu thập dữ liệu tay khéo léo chất lượng cao, thêm **bi-manual teleoperation** và workflow imitation learning qua **Isaac Lab Mimic**, mở rộng hỗ trợ thiết bị (Meta Quest VR, Manus gloves) để tăng tốc tạo dataset trình diễn (demonstration dataset), và **hỗ trợ teleoperation nâng cao cho Unitree G1** với dexterous retargeting (dịch chuyển động bàn tay người sang robot) — liên hệ trực tiếp tới `02-motion-retargeting/` (retargeting không chỉ cho cơ thể mà giờ cả bàn tay) và `08-real-robot-deployment/`. [NVIDIA Technical Blog](https://developer.nvidia.com/blog/streamline-robot-learning-with-whole-body-control-and-enhanced-teleoperation-in-nvidia-isaac-lab-2-3)
2. **Tích hợp SkillGen + Isaac Lab Mimic + cuRobo** cho phép sinh dữ liệu kỹ năng (skill-based data generation) với motion planning tăng tốc GPU — mở rộng khả năng "tự sinh thêm dữ liệu huấn luyện" thay vì chỉ dựa vào teleoperation thủ công, một hướng bổ sung trực tiếp cho vấn đề khối lượng dữ liệu đã thảo luận ở `03-human-motion-datasets/` (BONES-SEED).
3. **Xu hướng chung:** Isaac Lab đang dịch chuyển trọng tâm từ "framework RL thuần cho locomotion" (kế thừa Isaac Gym) sang một **nền tảng toàn diện cho robot learning** bao gồm cả teleoperation, imitation learning, và data generation — phản ánh đúng xu hướng ngành đang chuyển từ "RL thuần từ đầu" sang các pipeline lai kết hợp dữ liệu người (teleoperation, retargeting) với RL/imitation learning, đúng tinh thần đã thấy xuyên suốt các bài giảng `02-motion-retargeting/` và `03-human-motion-datasets/`.

## ❓ Câu hỏi tự kiểm tra

1. Điểm nghẽn mà Isaac Gym giải quyết khác gì với điểm nghẽn mà MJX/MuJoCo Playground giải quyết?
   <details><summary>Gợi ý đáp án</summary>MJX giải quyết việc MuJoCo CPU chỉ mô phỏng được 1 instance tuần tự (chưa song song hoá); Isaac Gym giải quyết việc dù đã song song hoá, dữ liệu vẫn phải truyền qua lại giữa CPU (vật lý) và GPU (mạng nơ-ron) nếu chúng nằm ở hai thiết bị khác nhau — Isaac Gym gộp cả hai vào cùng GPU để loại bỏ round-trip này.</details>
2. Isaac Gym, Isaac Sim, Isaac Lab khác nhau ở điểm nào? Cái nào đã deprecated?
   <details><summary>Gợi ý đáp án</summary>Isaac Gym (2021, framework nghiên cứu độc lập, đã deprecated) → Isaac Sim (nền tảng mô phỏng tổng quát Omniverse/USD) → Isaac Lab (framework robot-learning xây trên Isaac Sim, hiện hành). Isaac Gym đã deprecated, không phát triển tiếp.</details>
3. Trong ví dụ tính tay, vì sao chi phí "latency" của mỗi lần gọi truyền dữ liệu thường quan trọng hơn kích thước gói tin?
   <details><summary>Gợi ý đáp án</summary>Vì round-trip lặp lại hàng chục nghìn lần trong một quá trình huấn luyện (mỗi bước một lần) — mỗi lần gọi đều tốn một khoản chi phí cố định (overhead khởi tạo truyền) bất kể gói tin nhỏ tới đâu, nên tổng chi phí latency tích luỹ theo SỐ LẦN GỌI, không chỉ theo tổng lượng dữ liệu.</details>
4. Isaac Lab bổ sung 4 điểm gì so với ý tưởng gốc của Isaac Gym?
   <details><summary>Gợi ý đáp án</summary>(1) USD thay vì chỉ URDF/MJCF; (2) GPU-optimized simulation paths qua Warp và CUDA-graphable environments; (3) tích hợp đa thư viện RL (RSL-RL, skrl, RLlib, rl_games); (4) whole-body control và teleoperation nâng cao chính thức (từ bản 2.3).</details>
5. Vì sao Isaac Lab 2.3 (dexterous retargeting cho Unitree G1 qua teleoperation) liên quan trực tiếp tới nội dung đã học ở `02-motion-retargeting/`?
   <details><summary>Gợi ý đáp án</summary>Vì đây là một dạng retargeting khác — không phải retarget một chuyển động mocap có sẵn theo batch/offline (như GMR/SOMA-retargeter), mà retarget chuyển động bàn tay người theo thời gian thực (real-time) trong lúc teleoperation, cùng chung bài toán cấu trúc (DoF mismatch, tỷ lệ khác nhau) nhưng ở ngữ cảnh online thay vì offline.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** với N=8192 environment, D=150 (observation dim), action_dim=30, giả sử băng thông PCIe hiệu dụng 12 GB/s. Tính lại kích thước dữ liệu round-trip mỗi bước và tổng thời gian truyền thuần tuý cho 50 triệu bước huấn luyện — so sánh với kết quả trong bài (N=4096, D=200).
2. **Đọc tài liệu thật:** mở [Isaac Lab Release Notes chính thức](https://isaac-sim.github.io/IsaacLab/main/source/refs/release_notes.html), tìm mục thay đổi giữa 2 phiên bản gần nhau bất kỳ (ví dụ 2.2 → 2.3) — liệt kê ít nhất 3 thay đổi cụ thể và phân loại chúng theo 4 nhóm đã học ở mục Định nghĩa (USD, GPU-runtime, thư viện RL, WBC/teleoperation) hoặc ghi rõ nếu thay đổi đó không thuộc nhóm nào trong 4 nhóm này.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

Isaac Gym (Makoviychuk et al. 2021) giải quyết một điểm nghẽn khác với MJX: thay vì chỉ song song hoá instance, nó loại bỏ hoàn toàn round-trip truyền dữ liệu CPU↔GPU bằng cách chạy cả vật lý lẫn huấn luyện mạng nơ-ron trên cùng GPU dưới dạng PyTorch tensor — như ví dụ tính tay minh hoạ, chi phí round-trip tích luỹ chủ yếu từ số lần gọi lặp lại (latency), không chỉ từ lượng dữ liệu, giải thích vì sao loại bỏ nó mang lại tăng tốc 2-3 bậc độ lớn theo paper gốc. Isaac Gym nay đã deprecated, được kế thừa bởi Isaac Lab — framework robot-learning xây trên nền Isaac Sim/Omniverse, bổ sung USD, Warp/CUDA-graphable environments, tích hợp đa thư viện RL (đặc biệt RSL-RL cho locomotion), và từ bản 2.3 (2026) có whole-body control cùng teleoperation nâng cao (Apple Vision Pro, dexterous retargeting cho Unitree G1) — trở thành nền chính thức cho pipeline retargeting→training→deploy của hệ sinh thái GR00T, dù đòi hỏi hạ tầng GPU NVIDIA mạnh hơn nhiều so với lựa chọn "nhẹ" là MuJoCo Playground.
