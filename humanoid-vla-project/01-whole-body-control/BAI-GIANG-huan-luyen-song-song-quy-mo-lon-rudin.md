# Bài giảng: Huấn luyện song song quy mô lớn (Rudin et al. 2022)

*(Thuộc mảng: Whole-Body Control (WBC))*

## 🎯 Mục tiêu bài học

- Mô tả chính xác vấn đề tốc độ huấn luyện RL cho robot chân trước 2021, và đóng góp cụ thể của Rudin et al. 2022.
- Tính tay được ví dụ so sánh thời gian huấn luyện tuần tự vs song song cho một số lượng environment step cụ thể.
- Giải thích được "curriculum lấy cảm hứng từ game" hoạt động ra sao và vì sao nó cải thiện hiệu quả huấn luyện.
- Phân biệt được đóng góp của paper này ("phân tích thành phần thuật toán nào quan trọng khi song song hoá") với đóng góp thuần kỹ thuật ("chạy nhiều instance hơn").
- Nêu được chính xác Rudin et al. 2022 dùng robot nào (ANYmal, quadruped) — và ý nghĩa của việc dự án dùng lại kỹ thuật này cho humanoid.
- Liên hệ được đây là "bài chuyển giao" trực tiếp sang Isaac Gym/Isaac Lab (05) và SONIC (06).

## 🧭 Vì sao cần học cái này? (bối cảnh)

Bài trước giải thích *tại sao* nên chuyển sang WBC học sâu (RL). Nhưng có một trở ngại thực tế lớn: RL truyền thống rất **chậm** để huấn luyện — nếu mỗi lần thử phải chạy mô phỏng tuần tự, học được một hành vi tốt có thể mất hàng ngày. Rudin et al. 2022 là paper giải quyết đúng trở ngại này, và là cầu nối trực tiếp tới hạ tầng đã học ở `05-simulation-mujoco-isaaclab/` (Isaac Gym/Isaac Lab, MuJoCo Playground) — hiểu paper này giúp bạn thấy rõ **nguồn gốc lịch sử** của kỹ thuật "chạy hàng nghìn instance song song" mà giờ đã trở thành chuẩn mực, thay vì coi đó là điều "luôn vốn dĩ như vậy".

## 🧠 Trực giác

### Góc nhìn 1: Học đàn piano bằng cách thuê 4096 giáo viên dạy 4096 học sinh giống hệt nhau cùng lúc, rồi gộp kinh nghiệm lại

Nếu một học sinh piano thực hành 1 giờ mỗi ngày, sau 1000 ngày mới tích luỹ đủ 1000 giờ luyện tập. Nhưng nếu bạn có thể "nhân bản" học sinh đó thành 4096 phiên bản, cho tất cả luyện tập song song trong 1 giờ, rồi **gộp kinh nghiệm học được** từ cả 4096 phiên bản lại thành một bộ kỹ năng chung — bạn có được tương đương 4096 giờ luyện tập chỉ trong 1 giờ đồng hồ thực tế. Rudin et al. làm đúng việc này với robot ảo: hàng nghìn "robot học sinh" luyện tập song song trên GPU, kinh nghiệm của tất cả được gộp lại để cập nhật MỘT policy chung.

**Giới hạn của loại suy này:** 4096 học sinh piano thật sẽ có sự khác biệt cá nhân (tốc độ học, phong cách) khiến việc "gộp kinh nghiệm" phức tạp; 4096 robot ảo trong mô phỏng là **các bản sao giống hệt nhau về vật lý cơ bản** (chỉ khác ở nhiễu ngẫu nhiên/domain randomization), nên việc gộp gradient học được về mặt toán học đơn giản hơn nhiều (trung bình cộng gradient qua các mẫu, đúng cơ chế PPO đã học ở `05-simulation-mujoco-isaaclab/BAI-GIANG-rsl-rl-ppo-isaac-lab.md`).

### Góc nhìn 2: Trò chơi điện tử tăng độ khó theo màn (level), không ném người chơi vào màn khó nhất ngay từ đầu

Một game thiết kế tốt không bắt người chơi đối mặt màn cuối (boss cực khó) ngay từ đầu — nó bắt đầu bằng màn dễ, chỉ tăng độ khó khi người chơi đã "qua màn" hiện tại. Curriculum của Rudin et al. áp dụng đúng triết lý này cho địa hình: robot ảo bắt đầu tập đi trên địa hình phẳng (dễ), chỉ khi robot đã đi vững mới tăng dần độ gồ ghề của địa hình — thay vì ném thẳng robot vào địa hình khó nhất ngay từ episode đầu tiên (khi robot còn chưa biết đứng vững).

**Giới hạn của loại suy này:** một game thường có các "màn" rời rạc, rõ ràng (level 1, level 2...); curriculum địa hình của Rudin et al. thường **liên tục** (độ gồ ghề tăng dần mượt mà theo hiệu năng đo được của robot, không phải các bước nhảy rời rạc cố định) — tinh vi hơn một chút so với cấu trúc "màn chơi" cứng nhắc trong game truyền thống.

## 📐 Định nghĩa chính xác

**Vấn đề trước 2021:** huấn luyện RL cho robot chân thường mất **hàng giờ đến hàng ngày** vì mô phỏng chạy tuần tự trên CPU, mỗi lần chỉ một (hoặc một vài) robot ảo.

**Đóng góp của Rudin, Hoeller, Reist, Hutter (2022)** ([arXiv:2109.11978](https://arxiv.org/abs/2109.11978), Proceedings of the 5th Conference on Robot Learning, PMLR 164:91-100):

1. Chứng minh có thể **mô phỏng hàng nghìn robot ảo song song trên MỘT GPU** (nhờ simulator GPU-native — tiền thân trực tiếp của Isaac Gym đã học ở `05-simulation-mujoco-isaaclab/`), rút ngắn thời gian huấn luyện từ hàng giờ xuống **dưới 4 phút** cho địa hình phẳng và **khoảng 20 phút** cho địa hình gồ ghề — nhanh hơn các phương pháp trước đó **nhiều bậc độ lớn**.
2. Đóng góp **không chỉ là "chạy song song nhiều hơn"** mà còn **phân tích các thành phần thuật toán nào thực sự quan trọng** khi huấn luyện ở chế độ song song quy mô lớn (ví dụ cách chuẩn hoá observation, cách thiết kế reward phù hợp với tốc độ thu thập dữ liệu cực cao).
3. Đề xuất một **curriculum lấy cảm hứng từ game** (độ khó địa hình tăng dần khi robot "qua màn" tốt) để huấn luyện hiệu quả hơn so với huấn luyện trên độ khó cố định ngay từ đầu.

**Robot thử nghiệm:** paper huấn luyện robot **bốn chân ANYmal** (quadruped, không phải humanoid) đi trên địa hình thử thách — điểm quan trọng cần nhớ: kỹ thuật huấn luyện song song quy mô lớn này **ra đời cho robot chân bốn chân**, sau đó mới được áp dụng/mở rộng sang humanoid (hai chân, dư bậc tự do phức tạp hơn nhiều) trong các công trình sau này (PHC, OmniH2O, ASAP, SONIC — `04-imitation-learning-rl/`).

**Ý nghĩa "bài chuyển giao":** kỹ thuật huấn luyện song song quy mô lớn này là nền tảng trực tiếp mà **Isaac Gym/Isaac Lab** (`05-simulation-mujoco-isaaclab/`) và **SONIC** kế thừa để scale huấn luyện lên quy mô còn lớn hơn nữa — một trong 3 trục scale mà SONIC nhấn mạnh (model/data/**compute**, bài giảng "Kiến trúc decoupled WBC của SONIC").

## ⚙️ Cơ chế hoạt động — từng bước

```text
┌────────────────────────────────────────────────────────────┐
│ TRƯỚC 2021: huấn luyện tuần tự                                │
│  1 (hoặc vài) robot ảo chạy trên CPU                           │
│  → hàng giờ/hàng ngày để tích luỹ đủ environment step           │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ RUDIN ET AL. 2022: huấn luyện song song trên GPU               │
│                                                                 │
│  1. Khởi tạo N=4096+ bản sao robot ANYmal, MỖI bản sao ở một    │
│     ô địa hình riêng (bố trí lưới trên cùng một GPU)            │
│  2. Mỗi episode: robot bắt đầu ở địa hình có ĐỘ KHÓ hiện tại     │
│     của CURRICULUM (ban đầu: phẳng)                             │
│  3. Thu thập N mẫu (obs, action, reward) mỗi bước, TẤT CẢ song  │
│     song trên GPU (đúng cơ chế MJX/Isaac Gym đã học)             │
│  4. Cập nhật policy (PPO) sau mỗi rollout                        │
│  5. ĐÁNH GIÁ hiệu năng từng instance — nếu robot ở một ô địa      │
│     hình "qua màn tốt" (ví dụ đi được quãng đường đủ xa mà       │
│     không ngã), TĂNG độ khó địa hình cho ô đó ở episode sau       │
│     (curriculum thích ứng theo hiệu năng, không cố định)          │
│  6. Lặp lại — độ khó trung bình toàn bộ quần thể robot ảo tăng    │
│     dần theo thời gian huấn luyện                                 │
└──────────────────────────────────────────────────────────────┘
```

Sơ đồ minh hoạ curriculum "qua màn":

```text
Episode đầu:  [robot 1: phẳng] [robot 2: phẳng] ... [robot N: phẳng]
                    │ đi tốt          │ ngã              │ đi tốt
                    ▼                 ▼                  ▼
Episode sau:  [robot 1: hơi gồ ghề] [robot 2: VẪN phẳng] [robot N: hơi gồ ghề]
              (tăng độ khó vì       (giữ nguyên vì       (tăng độ khó)
               qua màn tốt)          chưa qua màn)
```

Mỗi robot ảo trong quần thể **tiến bộ theo tốc độ riêng** — không phải cả quần thể cùng tăng độ khó đồng loạt theo một lịch trình cố định, mà **thích ứng theo hiệu năng cá nhân từng instance**.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Dùng lại đúng cấu trúc tính toán đã học ở `05-simulation-mujoco-isaaclab/BAI-GIANG-mujoco-playground-mjx-jax.md`, áp dụng cho đúng ngữ cảnh/con số của Rudin et al. 2022)*

Giả sử một policy cần **300 triệu environment step** để học đi vững trên địa hình phẳng (một con số điển hình hợp lý cho locomotion đơn giản).

**Trước 2021 — tuần tự, 1 robot, giả sử 1.500 step/giây (CPU đơn):**

```text
thời gian = 300.000.000 / 1.500 = 200.000 giây ≈ 55.6 giờ
```

**Rudin et al. 2022 — N=4096 robot song song trên GPU, giả sử mỗi bước vật lý cho cả 4096 instance mất 10ms:**

```text
số bước cần = 300.000.000 / 4096 ≈ 73.242 bước
thời gian = 73.242 × 0.010 ≈ 732,4 giây ≈ 12.2 phút
```

**So sánh với con số paper công bố ("dưới 4 phút cho địa hình phẳng"):** ví dụ tính tay ở trên cho ra ~12.2 phút — chênh lệch với con số thật của paper (dưới 4 phút) là hợp lý vì đây chỉ là **ước tính minh hoạ** với giả định đơn giản hoá (thời gian mỗi bước, số step cần thiết đều là số tự chọn); điểm quan trọng cần rút ra không phải khớp đúng con số tuyệt đối, mà là **bậc độ lớn của tăng tốc**:

```text
tăng tốc ≈ 200.000 giây / 732,4 giây ≈ 273 lần
```

**Ý nghĩa:** dù con số chính xác phụ thuộc giả định, bậc độ lớn "hàng trăm lần" khớp với mô tả "nhanh hơn các phương pháp trước đó nhiều bậc độ lớn" của paper gốc — biến một quá trình huấn luyện "hàng giờ tới hàng ngày" (55.6 giờ trong ví dụ) thành "vài phút" (12.2 phút trong ví dụ), đúng tên gọi "Learning to Walk in **Minutes**".

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Trước Rudin et al. 2022 (RL tuần tự/CPU) | Rudin et al. 2022 (GPU-parallel + curriculum) | Isaac Lab/MuJoCo Playground hiện tại (05, kế thừa) |
|---|---|---|---|
| Số instance song song | 1 (hoặc vài chục qua multiprocessing) | Hàng nghìn (trên 1 GPU) | Hàng nghìn tới hàng chục nghìn |
| Thời gian huấn luyện locomotion cơ bản | Hàng giờ đến hàng ngày | Dưới 4 phút (địa hình phẳng) | Tương tự hoặc nhanh hơn (phần cứng mới hơn) |
| Curriculum địa hình | Thường cố định hoặc thiết kế thủ công | Thích ứng theo hiệu năng ("qua màn") | Kế thừa/mở rộng ý tưởng này |
| Robot thử nghiệm | Đa dạng, thường robot đơn giản hơn | ANYmal (quadruped) | Cả quadruped lẫn humanoid (G1, H1...) |
| Phân tích thành phần thuật toán quan trọng | Ít hệ thống hoá | Có, là đóng góp riêng biệt của paper | Kế thừa các bài học này làm best practice |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "đóng góp của Rudin et al. 2022 chỉ đơn thuần là 'chạy nhiều robot ảo hơn cùng lúc' — một cải tiến kỹ thuật thuần tuý".** Vì sao sai: chính paper nhấn mạnh đóng góp thứ hai — **phân tích hệ thống các thành phần thuật toán nào thực sự quan trọng** khi huấn luyện ở chế độ song song quy mô lớn (ví dụ cách observation normalization, reward shaping cần điều chỉnh khác đi khi tốc độ thu thập dữ liệu tăng vọt) — đây là đóng góp khoa học, không chỉ kỹ thuật. **Hiểu đúng:** chỉ đơn giản "thêm nhiều instance song song" mà không điều chỉnh đúng các thành phần thuật toán có thể không đạt được hiệu quả huấn luyện tốt — quy mô lớn đòi hỏi thiết kế thuật toán phù hợp đi kèm, không tự động "miễn phí" chỉ vì có nhiều GPU hơn.
2. **Hiểu nhầm: "Rudin et al. 2022 đã trực tiếp huấn luyện humanoid đi được".** Vì sao sai: paper gốc thử nghiệm trên **ANYmal — robot bốn chân (quadruped)**, không phải humanoid hai chân. Robot bốn chân có đế chân rộng hơn, dễ giữ thăng bằng tĩnh hơn nhiều so với humanoid hai chân (vốn luôn ở trạng thái "gần ngã" khi đi, cần kiểm soát động lực học tinh vi hơn — liên hệ ZMP/DCM đã học). **Hiểu đúng:** kỹ thuật huấn luyện song song quy mô lớn là **nền tảng chung** áp dụng được cho cả quadruped lẫn humanoid, nhưng bản thân paper 2022 chỉ chứng minh trên quadruped — việc mở rộng thành công sang humanoid là đóng góp của các công trình sau này (PHC, OmniH2O, ASAP, SONIC).

## 🏗️ Ví dụ minh hoạ trong dự án này

```text
Rudin et al. 2022 (ANYmal, quadruped, 2021-2022)
        │  kỹ thuật GPU-parallel training + curriculum
        ▼
Isaac Gym (05-simulation-mujoco-isaaclab/BAI-GIANG-isaac-gym-den-isaac-lab.md)
  → tổng quát hoá kỹ thuật này thành một framework độc lập
        │
        ▼
Isaac Lab (kế thừa Isaac Gym) + MuJoCo Playground (hướng song song khác, MJX)
  → hạ tầng huấn luyện song song mà 04-imitation-learning-rl/ và
    06-vla-groot-sonic/ (SONIC) dùng trực tiếp
        │
        ▼
SONIC: 9.000+ giờ GPU, huấn luyện trên 128 GPU trong 3 ngày, cho
  HUMANOID (không phải quadruped) — quy mô lớn hơn Rudin et al. 2022
  nhiều bậc, nhưng vẫn dựa trên đúng nguyên lý "chạy hàng nghìn
  instance song song trên GPU + curriculum" đã chứng minh từ 2022
```

Đây chính là lý do bài này được gọi là "bài chuyển giao quan trọng" — nếu không hiểu Rudin et al. 2022, con số "9.000 giờ GPU, 128 GPU trong 3 ngày" của SONIC (bài giảng tiếp theo) sẽ chỉ là một con số ấn tượng không có ngữ cảnh, thay vì hiểu đúng đây là **sự mở rộng quy mô** (từ 1 GPU/vài phút lên hàng trăm GPU/vài ngày) của đúng một nguyên lý kỹ thuật đã được chứng minh từ 2022.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **Kỹ thuật này đã trở thành hạ tầng chuẩn, được trích dẫn rộng rãi trong nghiên cứu robot chân 2025-2026.** Các công trình gần đây như *StairMaster* (leo cầu thang gồ ghề cho quadruped nhanh nhẹn) và nhiều nghiên cứu locomotion khác đều xây dựng trực tiếp trên hạ tầng GPU-parallel + curriculum mà Rudin et al. 2022 tiên phong — cho thấy đây không phải một kỹ thuật "một lần dùng" mà đã trở thành **thành phần hạ tầng tiêu chuẩn** của toàn bộ lĩnh vực.
2. **FastTD3 (2025)** ([arXiv:2505.22642](https://arxiv.org/pdf/2505.22642), đã nhắc ở `05-simulation-mujoco-isaaclab/`) tiếp tục hướng "đơn giản, nhanh" mà Rudin et al. khởi xướng nhưng áp dụng cho thuật toán off-policy (TD3) thay vì on-policy (PPO) — cho thấy nguyên lý "tối ưu hoá cho tốc độ huấn luyện GPU-parallel" đang được áp dụng rộng ra ngoài phạm vi PPO ban đầu.
3. **Xu hướng chung:** quy mô huấn luyện tiếp tục tăng theo cấp số nhân — từ N≈4096 instance/1 GPU (Rudin 2022) tới hàng trăm GPU (SONIC 2025) — nhưng nguyên lý cốt lõi (song song hoá + curriculum thích ứng + phân tích kỹ lưỡng thành phần thuật toán quan trọng) vẫn giữ nguyên giá trị, không bị thay thế bởi một nguyên lý khác, chỉ được **mở rộng quy mô** liên tục.

## ❓ Câu hỏi tự kiểm tra

1. Hai đóng góp của Rudin et al. 2022 là gì, và vì sao đóng góp thứ hai (phân tích thành phần thuật toán) quan trọng không kém đóng góp thứ nhất (song song hoá)?
   <details><summary>Gợi ý đáp án</summary>(1) Chứng minh mô phỏng hàng nghìn robot song song trên 1 GPU khả thi; (2) phân tích thành phần thuật toán nào thực sự quan trọng khi song song hoá quy mô lớn. Đóng góp thứ hai quan trọng vì chỉ "thêm instance song song" mà không điều chỉnh đúng thuật toán (observation normalization, reward shaping...) không tự động đạt hiệu quả huấn luyện tốt.</details>
2. Robot nào được dùng trong paper gốc Rudin et al. 2022, và điều này có ý nghĩa gì khi áp dụng kỹ thuật này cho humanoid?
   <details><summary>Gợi ý đáp án</summary>ANYmal — robot bốn chân (quadruped). Việc mở rộng thành công sang humanoid (hai chân, khó giữ thăng bằng hơn) là đóng góp của các công trình sau này (PHC, OmniH2O, ASAP, SONIC), không phải bản thân paper 2022.</details>
3. Curriculum "lấy cảm hứng từ game" hoạt động theo nguyên tắc gì, khác gì với việc tăng độ khó theo một lịch trình cố định cho mọi robot ảo?
   <details><summary>Gợi ý đáp án</summary>Độ khó địa hình tăng dần THEO HIỆU NĂNG CÁ NHÂN của từng instance robot ("qua màn" thì mới tăng độ khó cho đúng instance đó) — khác với lịch trình cố định áp dụng đồng loạt cho mọi instance bất kể instance đó đang thực sự đi vững hay chưa.</details>
4. Trong ví dụ tính tay, vì sao con số tính được (~12.2 phút) không khớp chính xác với con số paper công bố (dưới 4 phút), và tại sao điều đó không quan trọng?
   <details><summary>Gợi ý đáp án</summary>Vì ví dụ dùng các giả định tự chọn (số step cần, thời gian mỗi bước vật lý) không phải số liệu thật từ paper — điều quan trọng cần rút ra là BẬC ĐỘ LỚN của tăng tốc (hàng trăm lần, biến hàng giờ thành vài phút), không phải khớp đúng số phút tuyệt đối.</details>
5. Vì sao SONIC (9.000+ giờ GPU) được coi là "mở rộng quy mô" của Rudin et al. 2022 chứ không phải một nguyên lý hoàn toàn khác?
   <details><summary>Gợi ý đáp án</summary>Vì SONIC vẫn dựa trên đúng nguyên lý cốt lõi "chạy hàng nghìn instance song song trên GPU" mà Rudin et al. 2022 chứng minh lần đầu — chỉ khác về QUY MÔ (từ 1 GPU/vài phút lên hàng trăm GPU/vài ngày) và ĐỐI TƯỢNG (humanoid thay vì quadruped), không phải một cơ chế huấn luyện khác về bản chất.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** với 500 triệu environment step cần thiết, N=8192 instance, mỗi bước vật lý cho cả N instance mất 15ms — tính số bước cần và tổng thời gian huấn luyện (phút), so sánh tăng tốc so với kịch bản tuần tự 1.500 step/giây.
2. **Đọc paper thật:** đọc phần "Curriculum" hoặc "Terrain Curriculum" trong [arXiv:2109.11978](https://arxiv.org/abs/2109.11978) — mô tả bằng lời (3-5 câu) tiêu chí cụ thể mà paper dùng để quyết định "robot đã qua màn" (ví dụ dựa trên quãng đường đi được, số lần ngã, hay tiêu chí khác) và cách độ khó địa hình được điều chỉnh sau đó.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

Rudin, Hoeller, Reist, Hutter (2022) giải quyết trở ngại tốc độ huấn luyện RL cho robot chân bằng cách mô phỏng hàng nghìn robot ANYmal (quadruped) song song trên một GPU, kết hợp phân tích hệ thống các thành phần thuật toán quan trọng khi song song hoá quy mô lớn và một curriculum địa hình thích ứng theo hiệu năng từng instance ("qua màn" mới tăng độ khó) — như ví dụ tính tay minh hoạ, kết quả là tăng tốc hàng trăm lần, biến quá trình huấn luyện từ hàng giờ/hàng ngày thành vài phút, đúng tên gọi "Learning to Walk in Minutes". Đây là "bài chuyển giao" trực tiếp cho Isaac Gym/Isaac Lab (đã học ở `05-simulation-mujoco-isaaclab/`) và là nền tảng nguyên lý mà SONIC (bài giảng tiếp theo) mở rộng quy mô lên hàng trăm GPU để huấn luyện humanoid — chứng minh nguyên lý cốt lõi từ 2022 vẫn giữ nguyên giá trị dù đối tượng (quadruped→humanoid) và quy mô (1 GPU→hàng trăm GPU) đã thay đổi đáng kể.
