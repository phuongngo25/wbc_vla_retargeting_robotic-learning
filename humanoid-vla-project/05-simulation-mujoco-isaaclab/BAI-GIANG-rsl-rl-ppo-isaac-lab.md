# Bài giảng: RSL-RL và PPO trong Isaac Lab

*(Thuộc mảng: Simulation: MuJoCo & Isaac Lab)*

> Bài này **không giải thích lại thuật toán PPO từ đầu** — cơ chế PPO (clipped surrogate objective, advantage estimation...) đã có bài giảng riêng ở `04-imitation-learning-rl/` mục A-1 (Schulman et al. 2017). Bài này tập trung vào: RSL-RL là gì, vì sao nó được thiết kế riêng cho locomotion/GPU-parallel thay vì dùng thư viện RL tổng quát, và vì sao PPO cụ thể (trong số nhiều thuật toán RL) là lựa chọn mặc định khớp với kiến trúc Isaac Lab đã học ở các bài trước.

## 🎯 Mục tiêu bài học

- Nêu chính xác RSL-RL là gì, ai duy trì, và điểm khác biệt thiết kế so với thư viện RL tổng quát (Stable-Baselines3, RLlib).
- Giải thích được vì sao PPO (on-policy) phù hợp với kiến trúc GPU-parallel (Isaac Lab/MuJoCo Playground) hơn các thuật toán off-policy (SAC).
- Tính tay được ví dụ minh hoạ "dữ liệu lỗi thời" (data staleness) trong on-policy training và vì sao tốc độ sinh dữ liệu song song bù đắp được vấn đề này.
- Mô tả được 2 tính năng đặc thù robot học của RSL-RL: symmetry-based augmentation và Random Network Distillation (RND).
- Liên hệ được RSL-RL với công trình "Learning to Walk in Minutes" (đã nhắc ở `04-imitation-learning-rl/`) và giải thích ý nghĩa tên gọi.
- Phân biệt được khi nào nên dùng RSL-RL (locomotion/WBC trong Isaac Lab) và khi nào nên dùng thư viện RL khác (skrl, RLlib, rl_games đã học ở bài Isaac Gym→Isaac Lab).

## 🧭 Vì sao cần học cái này? (bối cảnh)

Các bài trước xây dựng toàn bộ hạ tầng mô phỏng: MJCF (mô tả robot), MuJoCo Playground/Isaac Lab (chạy hàng nghìn instance song song trên GPU), HumanoidVerse (trừu tượng hoá đa simulator). RSL-RL là **thuật toán/thư viện huấn luyện cụ thể** chạy bên trong hạ tầng đó — nó trả lời câu hỏi "sau khi đã có hàng nghìn robot ảo chạy song song, dùng thuật toán gì để chúng thực sự học đi/đứng vững?". Học bài này để hiểu vì sao câu trả lời phổ biến nhất trong hầu hết code mẫu chính thức Isaac Lab và playlist Skyentific không phải một thư viện RL tổng quát, mà là một thư viện được thiết kế hẹp, tối ưu riêng cho đúng bài toán này.

## 🧠 Trực giác

### Góc nhìn 1: Xe đua Công thức 1 (chuyên dụng) so với xe hơi đa dụng (tổng quát)

Stable-Baselines3/RLlib giống một chiếc xe hơi đa dụng — chạy được trên mọi loại đường, chở được nhiều loại hàng, phù hợp nhiều mục đích khác nhau (game, robot tay máy, tài chính...). RSL-RL giống một chiếc **xe đua Công thức 1** — chỉ làm tốt đúng một việc (đua trên đường đua GPU-parallel locomotion) nhưng làm việc đó nhanh và hiệu quả hơn nhiều so với một chiếc xe đa dụng cố "đua" trên cùng đường đua đó.

**Giới hạn của loại suy này:** xe F1 không thể chở hàng hay đi chợ; RSL-RL vẫn có thể áp dụng cho các bài toán RL khác ngoài locomotion nếu môi trường tương thích API — nó "chuyên dụng" theo nghĩa *tối ưu hoá* cho một lớp bài toán cụ thể (robot chân, GPU-parallel), không phải *bị khoá cứng* hoàn toàn như một chiếc xe đua chỉ chạy được trên đường đua.

### Góc nhìn 2: Bếp ăn công nghiệp làm đúng 1 món với hàng nghìn suất mỗi giờ, so với bếp nhà hàng làm được trăm món khác nhau

Một bếp ăn công nghiệp (RSL-RL) được thiết kế để sản xuất **một loại món ăn cụ thể** (PPO cho locomotion) ở tốc độ và quy mô cực lớn — dây chuyền tối ưu tới từng chi tiết cho đúng món đó. Một bếp nhà hàng (thư viện RL tổng quát) có thể làm hàng trăm món khác nhau (nhiều thuật toán, nhiều loại bài toán) nhưng không tối ưu tuyệt đối cho bất kỳ món riêng lẻ nào ở quy mô cực lớn.

**Giới hạn của loại suy này:** một bếp công nghiệp thường không thể đổi sang món khác dễ dàng; RSL-RL vẫn hỗ trợ vài biến thể thuật toán (không chỉ đúng 1 phiên bản PPO cứng nhắc) và có thể mở rộng, khác với một dây chuyền công nghiệp thường cố định hoàn toàn theo 1 sản phẩm.

## 📐 Định nghĩa chính xác

**RSL-RL** là thư viện reinforcement learning mã nguồn mở của **Robotic Systems Lab (RSL), ETH Zürich**, hiện được duy trì bởi Mayank Mittal và Clemens Schwarke, phối hợp giữa **ETH Zürich và NVIDIA** — nhóm đứng sau nhiều công trình locomotion nổi tiếng, bao gồm *"Learning to Walk in Minutes Using Massively Parallel Deep Reinforcement Learning"* (Rudin et al. 2022, đã dẫn ở `04-imitation-learning-rl/` mục A-2 và `01-whole-body-control/`). Thư viện được thiết kế **tối giản, tối ưu riêng cho bài toán locomotion/legged robot chạy trên GPU quy mô lớn** — khớp thẳng với kiến trúc "physics + training cùng GPU" của Isaac Lab (đã học ở bài "Isaac Gym→Isaac Lab") — khác với thư viện RL tổng quát (Stable-Baselines3, RLlib) vốn ưu tiên tính tổng quát hơn tốc độ tối đa cho một use-case cụ thể.

**Vì sao PPO là thuật toán mặc định:** PPO (Proximal Policy Optimization, Schulman et al. 2017, chi tiết ở `04-imitation-learning-rl/`) là thuật toán **on-policy** — chỉ dùng dữ liệu do đúng phiên bản policy hiện tại tạo ra để cập nhật, không tái sử dụng nhiều dữ liệu cũ (khác off-policy như SAC, vốn dùng replay buffer lưu dữ liệu từ nhiều policy cũ). PPO ổn định hơn các phương pháp policy-gradient cũ (ít nhạy learning rate, không cần second-order optimization như TRPO), phù hợp training on-policy tốc độ cao khi có hàng nghìn môi trường song song liên tục sinh dữ liệu mới.

**Isaac Lab không khoá cứng vào RSL-RL** — như đã học ở bài trước, có thể tích hợp skrl, RLlib, rl_games — nhưng với riêng bài toán humanoid locomotion/WBC, RSL-RL + PPO là combo mặc định xuất hiện trong phần lớn code mẫu chính thức.

## ⚙️ Cơ chế hoạt động — từng bước

```text
┌────────────────────────────────────────────────────────────┐
│ Isaac Lab (hoặc MuJoCo Playground) chạy N=4096+ instance      │
│  robot song song trên GPU (đã học ở 2 bài trước)               │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ RSL-RL: vòng lặp PPO on-policy                                 │
│  1. Policy hiện tại (mạng nơ-ron, trên GPU) chọn action cho     │
│     TẤT CẢ N instance CÙNG LÚC (batch inference)                │
│  2. Simulator bước 1 timestep cho N instance (đã học)           │
│  3. Thu thập (obs, action, reward, next_obs) — N mẫu MỖI BƯỚC   │
│  4. Lặp lại T bước (rollout) → thu thập N×T mẫu trong 1 "epoch" │
│     rollout, TẤT CẢ đều do đúng policy hiện tại tạo ra           │
│     (đây là bản chất on-policy — không lẫn dữ liệu cũ)          │
│  5. Cập nhật policy bằng PPO clipped objective trên N×T mẫu     │
│     (một vài epoch gradient descent trên cùng batch dữ liệu)    │
│  6. VỨT BỎ toàn bộ batch dữ liệu vừa dùng, quay lại bước 1       │
│     với policy ĐÃ CẬP NHẬT                                       │
└──────────────────────────────────────────────────────────────┘
```

**Hai tính năng đặc thù robot học của RSL-RL** (theo tài liệu chính thức):

1. **Symmetry-based augmentation** — khai thác tính đối xứng vật lý của robot (ví dụ đối xứng trái-phải của cơ thể người/humanoid) để tăng tốc sinh dữ liệu: mỗi mẫu (obs, action) thu thập được có thể "lật gương" thành một mẫu hợp lệ khác (giống kỹ thuật mirror augmentation đã thấy ở bài BONES-SEED trong `03-human-motion-datasets/`), tăng hiệu quả dữ liệu mà không cần chạy thêm mô phỏng — có thể tăng cường thêm bằng một **symmetry loss** khuyến khích hành vi đối xứng hơn.
2. **Random Network Distillation (RND)** — khuyến khích khám phá (exploration) bằng cách thêm một phần thưởng nội tại (intrinsic reward) dựa trên độ "tò mò" (curiosity-driven) — hữu ích khi reward thưa (sparse reward) khiến policy khó tìm ra hành vi tốt chỉ bằng reward gốc.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số tự chọn để định lượng hoá "vì sao tốc độ song song hoá bù đắp cho on-policy" — không phải benchmark thật của RSL-RL)*

**Vấn đề "dữ liệu lỗi thời" (data staleness) trong on-policy training:** với on-policy, dữ liệu thu thập bởi policy phiên bản `k` chỉ dùng được để cập nhật lên policy phiên bản `k+1` (thường vài epoch gradient) rồi phải **vứt bỏ** — không giống off-policy (SAC) có thể tái dùng dữ liệu cũ nhiều lần qua replay buffer.

Giả sử một bài toán cần tổng cộng **500 triệu mẫu (obs, action, reward)** để policy hội tụ.

**Kịch bản A — chạy tuần tự, N=1 instance, không song song hoá (giả định lý thuyết, không thực tế cho RL hiện đại nhưng hữu ích để so sánh):**

```text
với on-policy, mỗi rollout T=1000 bước chỉ cho 1000 mẫu trước khi phải cập nhật
  và vứt bỏ dữ liệu → cần 500.000.000 / 1000 = 500.000 lần cập nhật policy
giả sử mỗi lần cập nhật (rollout + gradient step) mất 0.5 giây
  → tổng thời gian = 500.000 × 0.5 = 250.000 giây ≈ 69.4 giờ
```

**Kịch bản B — RSL-RL trên Isaac Lab, N=4096 instance song song, T=24 bước/rollout** (một cấu hình điển hình: rollout ngắn nhưng N rất lớn):

```text
mỗi rollout cho N×T = 4096×24 = 98.304 mẫu MỖI LẦN
số lần rollout cần = 500.000.000 / 98.304 ≈ 5.086 lần
giả sử mỗi lần rollout+update mất 0.3 giây (nhanh hơn kịch bản A vì
  batch inference/update trên GPU hiệu quả hơn dù xử lý nhiều mẫu hơn)
  → tổng thời gian = 5.086 × 0.3 ≈ 1.525,8 giây ≈ 25.4 phút
```

**So sánh:**

```text
tăng tốc ≈ 250.000 / 1.525,8 ≈ 163,8 lần
```

**Ý nghĩa:** mỗi mẫu dữ liệu on-policy đều "dùng một lần rồi bỏ" trong cả hai kịch bản (đặc điểm không đổi của PPO) — nhưng vì kịch bản B thu thập **98.304 mẫu mỗi lần** thay vì 1.000, số **lần cập nhật cần thiết** giảm mạnh, và mỗi lần cập nhật trên GPU với batch lớn không chậm đi tỷ lệ thuận với kích thước batch (tận dụng song song hoá phần cứng) — đây chính là cách "hàng nghìn môi trường song song liên tục sinh dữ liệu mới... giúp bù đắp việc không tái sử dụng được nhiều dữ liệu cũ" (đã nêu trong nguồn) được cụ thể hoá bằng số: không phải PPO "bớt on-policy đi" khi chạy song song, mà là **tốc độ tạo dữ liệu mới đủ nhanh để việc "không tái sử dụng" không còn là vấn đề nghiêm trọng**.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | RSL-RL (PPO, on-policy) | SAC (off-policy, xem `04-imitation-learning-rl/`) | Stable-Baselines3/RLlib (tổng quát) |
|---|---|---|---|
| Tái sử dụng dữ liệu cũ | Không (vứt bỏ sau vài epoch) | Có (replay buffer) | Tuỳ thuật toán bên trong |
| Phù hợp GPU-parallel quy mô lớn | Rất phù hợp (thiết kế riêng cho việc này) | Kém phù hợp hơn (replay buffer khó song song hoá theo N instance) | Tổng quát, không tối ưu riêng |
| Độ ổn định huấn luyện | Cao (clipped objective, ít nhạy hyperparameter) | Có thể kém ổn định hơn nếu tuning sai | Tuỳ thuật toán |
| Chuyên biệt cho locomotion? | Có (tính năng symmetry augmentation riêng) | Không | Không |
| Cần chọn hyperparameter riêng cho từng bài toán? | Ít hơn (đã tối ưu sẵn cho locomotion) | Nhiều hơn | Nhiều |
| Dùng khi nào | Locomotion/WBC trong Isaac Lab/MuJoCo Playground | Bài toán cần data-efficiency cao, ít mẫu | Bài toán RL tổng quát ngoài locomotion, hoặc cần thuật toán RSL-RL không hỗ trợ |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "PPO được chọn cho Isaac Lab vì nó là thuật toán RL 'tốt nhất' nói chung".** Vì sao sai: không có thuật toán RL nào "tốt nhất" một cách tuyệt đối — lựa chọn PPO khớp cụ thể với **đặc điểm kiến trúc** của Isaac Lab/MuJoCo Playground (hàng nghìn môi trường song song liên tục sinh dữ liệu mới, GPU-parallel training) như đã tính toán ở ví dụ tính tay: bản chất on-policy "lãng phí" dữ liệu cũ của PPO chỉ chấp nhận được *vì* tốc độ sinh dữ liệu mới cực nhanh bù lại. Với hạ tầng ít song song hoá hơn (ví dụ chỉ chạy được vài chục instance), off-policy (SAC) có thể hiệu quả hơn nhiều nhờ tái sử dụng dữ liệu. **Hiểu đúng:** lựa chọn thuật toán RL phải khớp với đặc điểm hạ tầng tính toán sẵn có, không phải chọn theo "danh tiếng" thuật toán.
2. **Hiểu nhầm: "RSL-RL chỉ là một cách gọi khác của PPO, hai cái là một".** Vì sao sai: RSL-RL là một **thư viện/framework triển khai** (bao gồm cả cách tổ chức rollout song song, quản lý GPU tensor, các tính năng bổ sung như symmetry augmentation và RND) — PPO chỉ là **một trong các thuật toán** mà thư viện này triển khai (dù là thuật toán mặc định phổ biến nhất cho locomotion). **Hiểu đúng:** RSL-RL là công cụ (how), PPO là thuật toán (what) — có thể hình dung RSL-RL là "cách PPO được cài đặt để chạy hiệu quả trên GPU-parallel Isaac Lab", không phải bản thân PPO.

## 🏗️ Ví dụ minh hoạ trong dự án này

```text
GMR retarget dữ liệu (02-motion-retargeting/) → reference motion
        │
        ▼
Isaac Lab (05, bài Isaac Gym→Isaac Lab): định nghĩa task motion-tracking
  N=4096+ instance G1 song song trên GPU
        │
        ▼
RSL-RL: PPO on-policy huấn luyện policy tracking chuyển động reference
  - Reward: khoảng cách pose hiện tại vs pose reference (04-imitation-learning-rl/,
    "Reward thủ công vs Motion-tracking reward")
  - Symmetry augmentation: tận dụng đối xứng trái-phải của G1 để tăng
    hiệu quả dữ liệu (robot humanoid vốn đối xứng gương theo mặt phẳng dọc)
        │
        ▼
Policy đã huấn luyện → kiểm chứng qua HumanoidVerse (bài trước, chạy lại
  trên MuJoCo/MJX) → deploy robot thật (08-real-robot-deployment/)
```

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **RSL-RL nay có paper mô tả chính thức riêng.** Schwarke, Mittal, Rudin, Hoeller, Hutter (2025), *"RSL-RL: A Learning Library for Robotics Research"* ([arXiv:2509.10771](https://arxiv.org/pdf/2509.10771)) — trước đây thư viện chủ yếu được biết qua code/README, nay đã có công bố học thuật mô tả đầy đủ kiến trúc và các tính năng (bao gồm symmetry-based augmentation và RND đã học ở mục Cơ chế). Việc có paper riêng cho thấy RSL-RL đã trưởng thành từ "công cụ nội bộ của một nhóm nghiên cứu" thành một thư viện được cộng đồng công nhận và trích dẫn rộng rãi. [arXiv:2509.10771](https://arxiv.org/pdf/2509.10771)
2. **Symmetry-based augmentation có nền tảng nghiên cứu riêng** — Mittal et al., *"Leveraging Symmetry in RL-based Legged Locomotion Control"* ([arXiv:2403.17320](https://arxiv.org/pdf/2403.17320)), cho thấy tính năng này không phải một "trick" tuỳ tiện mà dựa trên một nghiên cứu có hệ thống về cách khai thác đối xứng vật lý của robot chân để tăng hiệu quả mẫu (sample efficiency) — một hướng cải tiến trực tiếp nhắm vào chính điểm yếu "on-policy lãng phí dữ liệu" đã phân tích ở mục Ví dụ tính tay.
3. **RSL-RL hiện được đồng duy trì bởi ETH Zürich và NVIDIA**, không còn chỉ là dự án nội bộ một trường đại học — phản ánh đúng xu hướng chung đã thấy xuyên suốt các bài giảng trong thư mục này (MJWarp là NVIDIA+DeepMind, Isaac Lab tích hợp nhiều thư viện bên thứ ba): hệ sinh thái robot learning 2025-2026 đang hội tụ thành các hợp tác liên tổ chức thay vì các công cụ độc lập rời rạc.

## ❓ Câu hỏi tự kiểm tra

1. Vì sao PPO (on-policy) "lãng phí" dữ liệu hơn SAC (off-policy) về mặt lý thuyết, nhưng vẫn là lựa chọn mặc định cho Isaac Lab?
   <details><summary>Gợi ý đáp án</summary>Vì PPO vứt bỏ dữ liệu sau vài epoch cập nhật, không tái sử dụng như replay buffer của SAC — nhưng vì Isaac Lab cung cấp tốc độ sinh dữ liệu mới cực nhanh (hàng nghìn instance song song), "lãng phí" này được bù đắp bởi khối lượng dữ liệu mới khổng lồ mỗi rollout, đồng thời PPO ổn định hơn và dễ song song hoá theo N instance hơn SAC (vốn cần quản lý replay buffer phức tạp hơn).</details>
2. Trong ví dụ tính tay, nếu tăng N từ 4096 lên 8192 (giữ T=24, thời gian mỗi lần rollout+update không đổi 0.3s — giả định lý tưởng), thời gian huấn luyện 500 triệu mẫu thay đổi thế nào?
   <details><summary>Gợi ý đáp án</summary>Mẫu mỗi rollout tăng lên `8192×24=196.608`, số lần rollout giảm còn `500.000.000/196.608≈2.543`, thời gian `2.543×0.3≈762.9s≈12.7 phút` — giảm gần một nửa so với 25.4 phút ở N=4096.</details>
3. RSL-RL và PPO khác nhau ở điểm nào — cái nào là "công cụ" và cái nào là "thuật toán"?
   <details><summary>Gợi ý đáp án</summary>RSL-RL là thư viện/framework triển khai (công cụ) — cách tổ chức rollout song song, quản lý GPU tensor, các tính năng bổ sung (symmetry, RND); PPO là thuật toán RL cụ thể mà RSL-RL triển khai làm mặc định cho locomotion.</details>
4. Symmetry-based augmentation trong RSL-RL khai thác đặc điểm gì của robot humanoid, và nó tương tự kỹ thuật nào đã học ở `03-human-motion-datasets/`?
   <details><summary>Gợi ý đáp án</summary>Khai thác tính đối xứng trái-phải (mirror symmetry) của cơ thể robot humanoid — tạo thêm mẫu hợp lệ bằng cách lật gương một mẫu đã thu thập, tương tự kỹ thuật "mirror/lật gương" dùng để tăng gấp đôi số lượng chuyển động trong dataset BONES-SEED (71.132 gốc + 71.088 mirror).</details>
5. Khi nào nên cân nhắc dùng skrl/RLlib/rl_games thay vì RSL-RL trong Isaac Lab?
   <details><summary>Gợi ý đáp án</summary>Khi bài toán không thuộc locomotion/WBC thuần tuý mà cần các thuật toán RSL-RL không hỗ trợ tối ưu, hoặc khi cần tính tổng quát cao hơn (nhiều loại thuật toán, nhiều loại bài toán RL khác nhau) hơn là tốc độ tối đa cho đúng một lớp bài toán cụ thể mà RSL-RL nhắm tới.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** với bài toán cần 1 tỷ mẫu, N=2048, T=48, mỗi lần rollout+update mất 0.4 giây — tính số lần rollout cần và tổng thời gian huấn luyện (theo phút), so sánh với kịch bản B trong bài (N=4096, T=24).
2. **Đọc paper/code thật:** đọc phần giới thiệu tính năng symmetry augmentation trong [arXiv:2509.10771](https://arxiv.org/pdf/2509.10771) hoặc [arXiv:2403.17320](https://arxiv.org/pdf/2403.17320) — mô tả bằng lời (3-5 câu) cách một mẫu (observation, action) của khớp gối trái được "lật gương" thành mẫu hợp lệ cho khớp gối phải, và tại sao cách này không cần chạy thêm mô phỏng vật lý nào.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

RSL-RL (ETH Zürich, đồng duy trì với NVIDIA, nay có paper chính thức arXiv:2509.10771) là thư viện RL tối giản, thiết kế riêng cho locomotion/legged robot chạy trên GPU quy mô lớn, dùng PPO on-policy làm thuật toán mặc định — lựa chọn khớp trực tiếp với kiến trúc GPU-parallel của Isaac Lab/MuJoCo Playground, vì như ví dụ tính tay minh hoạ, bản chất "vứt bỏ dữ liệu sau mỗi vài epoch" của on-policy chỉ chấp nhận được nhờ tốc độ sinh dữ liệu mới cực nhanh (hàng nghìn instance song song mỗi rollout), không phải vì PPO tự thân "tốt nhất". RSL-RL bổ sung hai tính năng đặc thù robot học: symmetry-based augmentation (khai thác đối xứng trái-phải để tăng hiệu quả dữ liệu, tương tự kỹ thuật mirror ở dataset BONES-SEED) và Random Network Distillation cho exploration — trong khi Isaac Lab vẫn hỗ trợ các thư viện RL khác (skrl, RLlib, rl_games) cho các bài toán ngoài phạm vi tối ưu hoá riêng của RSL-RL.
