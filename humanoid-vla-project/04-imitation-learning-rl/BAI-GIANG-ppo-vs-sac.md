# Bài giảng: Tóm tắt RL cơ bản — PPO vs SAC

*(Thuộc mảng: Imitation Learning & Reinforcement Learning)*

## 🎯 Mục tiêu bài học

- Giải thích được policy, trajectory, reward tích lũy nghĩa là gì trong RL, không cần nhớ công thức toán hàn lâm.
- Phân biệt chính xác **on-policy vs off-policy**, và vì sao PPO thuộc nhóm đầu còn SAC thuộc nhóm sau.
- Giải thích được cơ chế **clipped surrogate objective** của PPO — vì sao nó ngăn được policy "sụp đổ" sau một bước cập nhật tệ.
- Giải thích được **entropy regularization** trong SAC dùng để làm gì, khác gì so với việc PPO khuyến khích khám phá.
- Tính tay được một ví dụ số nhỏ minh hoạ PPO clip hoạt động ra sao khi ratio xác suất vượt ngưỡng.
- Giải thích được vì sao gần như toàn bộ paper motion-tracking humanoid (DeepMimic, AMP, ASAP, SONIC) chọn PPO thay vì SAC, và tình huống nào SAC vẫn có chỗ đứng.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Toàn bộ các bài giảng tiếp theo trong thư mục này (Reward thủ công vs motion-tracking, DeepMimic, AMP, PHC→SONIC, Domain Randomization, Residual learning) đều giả định bạn đã hiểu "vòng lặp RL cơ bản" đang chạy bên dưới. Không có bài nào trong số đó giải thích lại PPO/SAC — chúng chỉ nói "huấn luyện bằng PPO" rồi đi thẳng vào phần thú vị (reward, discriminator, kiến trúc). Bài này là nền tảng bắt buộc: nếu bạn chưa từng cài PPO/SAC trước đây, đọc kỹ bài này rồi mới sang các bài sau — nếu đã quen thuộc, đây vẫn là nơi để nắm chắc **vì sao PPO gần như là lựa chọn mặc định duy nhất** trong pipeline huấn luyện humanoid ở dự án này (chạy hàng nghìn môi trường song song trên GPU trong `05-simulation-mujoco-isaaclab/`).

## 🧠 Trực giác

### Góc nhìn 1: Học lái xe với giáo viên luôn ngồi cạnh (PPO) vs xem lại băng ghi hình cũ (SAC)

PPO giống một học viên lái xe chỉ được phép luyện tập với **chiếc xe và tuyến đường hiện tại**, dưới sự giám sát trực tiếp — mỗi lần muốn thử một kiểu lái mới, học viên phải thực sự lái thử ngay bây giờ, không được "tưởng tượng lại" các buổi tập cũ vì tuyến đường/điều kiện đã thay đổi (chính sách đã cập nhật). Ngược lại, SAC giống một học viên có **camera hành trình ghi lại mọi buổi tập trước đây** (replay buffer) — kể cả những buổi tập từ vài tuần trước với một phong cách lái hơi khác, học viên vẫn có thể lôi lại các đoạn băng đó ra phân tích và học thêm, không cần lái lại từ đầu.

*Đúng ở đâu:* nắm bắt đúng bản chất "dữ liệu có được tái sử dụng hay không" — điểm khác biệt cốt lõi giữa on-policy và off-policy.
*Giới hạn:* phép loại suy này không giải thích được **tại sao** việc tái sử dụng dữ liệu cũ lại nguy hiểm về mặt toán học (dữ liệu cũ được thu bằng một policy khác, nên ước lượng gradient/giá trị dựa trên nó có thể bị lệch — gọi là vấn đề **distribution shift**). SAC giải quyết việc này bằng công thức off-policy actor-critic (Q-function) chịu được dữ liệu lệch phân phối ở mức độ nhất định, chứ không phải "học được từ băng cũ" một cách miễn phí, ngây thơ.

### Góc nhìn 2: Dây cương ngựa non (PPO clip) vs người huấn luyện khuyến khích thử nhiều cách (SAC entropy)

Hãy tưởng tượng đang huấn luyện một con ngựa non (policy) học một bài nhảy. PPO giống việc bạn **giữ dây cương chặt vừa đủ**: mỗi lần điều chỉnh hướng đi, bạn chỉ được kéo dây một góc giới hạn (clip) — nếu ngựa đang đi đúng hướng, bạn không kéo mạnh hơn nữa để "thưởng" nó (tránh làm nó hoảng và đổi hành vi đột ngột); nếu ngựa lệch hướng, bạn cũng chỉ kéo tới một mức giới hạn rồi dừng, không kéo giật cho tới khi ngựa "gãy" thói quen cũ hoàn toàn. SAC giống một người huấn luyện khác: ngoài việc thưởng cho hành vi đúng, còn **chủ động thưởng thêm cho sự đa dạng** trong cách ngựa thử các bước nhảy (entropy bonus) — khuyến khích ngựa không chỉ lặp lại một kiểu nhảy duy nhất quá sớm, để tránh nó "chốt" vào một chiến lược cục bộ trước khi khám phá đủ.

*Đúng ở đâu:* nắm đúng vai trò cơ chế của "clip" (giới hạn thay đổi mỗi bước) và "entropy bonus" (khuyến khích đa dạng hành vi).
*Giới hạn:* phép loại suy không phản ánh đúng rằng PPO **cũng có khám phá** (thông qua độ ngẫu nhiên vốn có của policy stochastic và advantage estimate), chỉ là nó không có số hạng entropy tường minh trong objective như SAC theo mặc định (dù nhiều triển khai PPO thực tế, bao gồm cả trong Isaac Lab/RSL-RL, vẫn thêm entropy bonus phụ trợ) — ranh giới giữa hai thuật toán không tuyệt đối "PPO không khám phá, SAC mới khám phá".

## 📐 Định nghĩa chính xác

Cho một Markov Decision Process (MDP) với trạng thái s, hành động a, policy π_θ(a|s) tham số hoá bởi θ. Mục tiêu RL tổng quát là tối đa hoá:

```
J(θ) = E_τ~π_θ [ Σ_t γ^t r_t ]
```

với γ ∈ (0,1] là discount factor, τ là trajectory (s₀,a₀,r₀,s₁,...) sinh ra khi chạy π_θ trong môi trường.

**PPO (Proximal Policy Optimization — Schulman et al. 2017, [arXiv:1707.06347](https://arxiv.org/abs/1707.06347))** là thuật toán **on-policy, policy-gradient**. Với tỉ số xác suất (probability ratio):

```
r_t(θ) = π_θ(a_t|s_t) / π_θ_old(a_t|s_t)
```

PPO tối ưu **clipped surrogate objective**:

```
L^CLIP(θ) = E_t [ min( r_t(θ)·Â_t , clip(r_t(θ), 1-ε, 1+ε)·Â_t ) ]
```

trong đó Â_t là ước lượng advantage (thường tính bằng GAE — Generalized Advantage Estimation) tại bước t, và ε là hyperparameter clip (thường 0.1–0.2). Vì phải dùng π_θ_old để tính r_t(θ), dữ liệu huấn luyện **phải mới** (thu bằng policy gần với policy hiện tại) — đây là lý do PPO là on-policy: sau vài epoch cập nhật trên cùng một batch dữ liệu, dữ liệu đó bị "cũ" và phải thu lại từ đầu.

**SAC (Soft Actor-Critic — Haarnoja et al. 2018, [arXiv:1801.01290](https://arxiv.org/abs/1801.01290))** là thuật toán **off-policy, actor-critic, entropy-regularized**. Mục tiêu tối ưu (maximum entropy RL objective):

```
J(θ) = E_τ~π_θ [ Σ_t γ^t ( r_t + α·H(π_θ(·|s_t)) ) ]
```

với H(π_θ(·|s_t)) = -E_a~π_θ[log π_θ(a|s_t)] là entropy của policy tại trạng thái s_t, và α là **temperature coefficient** kiểm soát trọng số của số hạng entropy (nhiều triển khai hiện đại tự động điều chỉnh α trong lúc huấn luyện). SAC học đồng thời: một hoặc nhiều **Q-function** (soft Q-value, ước lượng bằng Bellman backup có cộng entropy) từ dữ liệu lấy ngẫu nhiên trong **replay buffer** (không cần dữ liệu mới nhất), và một **policy actor** được cập nhật để tối đa hoá Q-value cộng entropy. Vì Q-function có thể học từ transition thu thập bởi bất kỳ policy nào trong quá khứ (miễn còn trong buffer), SAC là off-policy.

## ⚙️ Cơ chế hoạt động — từng bước

**Vòng lặp PPO** (mỗi iteration huấn luyện):

```
┌─────────────────────────────────────────────────────────┐
│ 1. Rollout: chạy π_θ_old trên N môi trường song song      │
│    (ví dụ 4096 robot ảo trong Isaac Lab) trong T bước,    │
│    thu thập batch (s,a,r,s') — batch size N×T             │
├─────────────────────────────────────────────────────────┤
│ 2. Tính advantage Â_t bằng GAE, dùng critic (value        │
│    function V(s)) ước lượng                                │
├─────────────────────────────────────────────────────────┤
│ 3. Lặp K epoch (thường K=3–10) trên CÙNG batch dữ liệu:   │
│    - Tính r_t(θ) = π_θ(a|s)/π_θ_old(a|s)                  │
│    - Cập nhật θ bằng gradient ascent trên L^CLIP           │
│    - Đồng thời cập nhật critic (giảm MSE giữa V(s) và     │
│      giá trị mục tiêu)                                      │
├─────────────────────────────────────────────────────────┤
│ 4. Sau K epoch: BỎ toàn bộ batch cũ (không dùng lại),      │
│    đặt π_θ_old ← π_θ, quay lại bước 1                       │
└─────────────────────────────────────────────────────────┘
```

Điểm mấu chốt: bước 4 — dữ liệu bị vứt bỏ hoàn toàn sau K epoch, đây chính là bản chất "on-policy". Việc lặp K epoch trên cùng batch (thay vì 1 epoch rồi vứt ngay) là lý do PPO cần cơ chế clip: nếu không clip, sau vài epoch cập nhật liên tiếp trên cùng dữ liệu, θ có thể trôi quá xa θ_old khiến ước lượng advantage (tính từ θ_old) không còn chính xác cho θ mới — clip ngăn r_t(θ) đi quá xa 1 (tức ngăn π_θ khác quá xa π_θ_old) trong mỗi epoch.

**Vòng lặp SAC:**

```
┌─────────────────────────────────────────────────────────┐
│ 1. Tương tác với môi trường bằng π_θ hiện tại, lưu mỗi    │
│    transition (s,a,r,s') vào replay buffer D (buffer lớn, │
│    ví dụ hàng triệu transition, KHÔNG xoá sau mỗi bước)   │
├─────────────────────────────────────────────────────────┤
│ 2. Lấy mẫu MINIBATCH NGẪU NHIÊN từ D (có thể là transition│
│    thu thập từ rất lâu trước, bởi policy cũ hơn nhiều)     │
├─────────────────────────────────────────────────────────┤
│ 3. Cập nhật Q-function(s) bằng soft Bellman backup:        │
│    Q(s,a) ← r + γ·E_a'~π[Q(s',a') - α·log π(a'|s')]        │
├─────────────────────────────────────────────────────────┤
│ 4. Cập nhật policy actor để tối đa hoá E[Q(s,a) - α·log π] │
├─────────────────────────────────────────────────────────┤
│ 5. (tuỳ chọn) Cập nhật α tự động để giữ entropy trung bình │
│    gần một target entropy mong muốn                         │
│    Quay lại bước 1 (không cần vứt buffer)                  │
└─────────────────────────────────────────────────────────┘
```

Khác biệt cấu trúc rõ nhất: PPO **xen kẽ** thu-dữ-liệu-mới rồi vứt, SAC **liên tục tích luỹ** dữ liệu và học lại nhiều lần từ dữ liệu cũ trong mọi bước — đây là lý do SAC thường "sample-efficient" hơn (cần ít tương tác môi trường hơn để đạt cùng hiệu năng) khi môi trường tương tác đắt đỏ (robot thật), nhưng lại tốn công tính toán hơn trên mỗi bước huấn luyện khi so trên hạ tầng mô phỏng song song rẻ.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số liệu tự chọn để dễ hình dung.)*

Giả sử policy đang huấn luyện một khớp gối của robot, hành động a là mô-men lực chuẩn hoá ∈ [-1,1]. Tại một state s cụ thể, policy cũ π_θ_old chọn hành động a=0.5 với xác suất mật độ π_θ_old(a=0.5|s) = 0.20. Advantage ước lượng tại bước này là Â = +2.0 (hành động này tốt hơn kỳ vọng trung bình).

Sau một bước cập nhật gradient, policy mới π_θ giờ gán xác suất mật độ π_θ(a=0.5|s) = 0.55 cho cùng hành động đó (policy đã "tự tin" hơn vào hành động này). Tính:

```
r_t(θ) = π_θ(a|s) / π_θ_old(a|s) = 0.55 / 0.20 = 2.75
```

Với ε = 0.2 (giá trị chuẩn phổ biến), khoảng clip là [1-0.2, 1+0.2] = [0.8, 1.2]. Vì r_t(θ) = 2.75 > 1.2, ta clip:

```
clip(r_t(θ), 0.8, 1.2) = 1.2
```

Vì Â_t = +2.0 > 0, ta lấy **min** của hai giá trị:

```
L^CLIP = min( r_t(θ)·Â_t , clip(r_t(θ))·Â_t )
       = min( 2.75×2.0 , 1.2×2.0 )
       = min( 5.5 , 2.4 )
       = 2.4
```

**Diễn giải:** dù ratio thực tế (2.75) cho thấy policy đã tăng xác suất hành động này rất mạnh, gradient thực tế chỉ "được thưởng" như thể ratio chỉ là 1.2 — objective bị **giới hạn trần**, ngăn không cho gradient tiếp tục đẩy θ đi xa hơn nữa theo hướng này trong cùng một bước cập nhật (dù advantage rất tốt). Đây chính xác là cơ chế ngăn "một bước cập nhật quá lớn làm policy sụp đổ" nói ở phần định nghĩa — nếu không có min/clip, một advantage dương lớn kết hợp ratio lớn sẽ tạo gradient khổng lồ, đẩy θ nhảy quá xa.

Ngược lại nếu Â_t = -2.0 (hành động này *tệ* hơn kỳ vọng), công thức đổi vai trò: min của (2.75×(-2.0)=-5.5) và (1.2×(-2.0)=-2.4) là **-5.5** (min lấy giá trị nhỏ hơn/âm hơn) — trường hợp advantage âm, PPO **không** clip khi ratio lớn theo chiều tăng xác suất một hành động xấu, để đảm bảo penalty đủ mạnh kéo policy tránh xa hành động đó (chi tiết bất đối xứng này là một điểm hay bị hiểu nhầm — xem mục Sai lầm bên dưới).

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | PPO | SAC |
|---|---|---|
| Kiểu học | On-policy | Off-policy |
| Tái sử dụng dữ liệu cũ | Không (vứt sau K epoch) | Có (replay buffer) |
| Sample efficiency (số tương tác cần) | Thấp hơn — cần nhiều môi trường/bước hơn | Cao hơn — thường cần ít tương tác hơn |
| Song song hoá trên GPU (nghìn env ảo) | Rất tốt — là lý do chính được chọn cho humanoid sim | Kém tự nhiên hơn (dù có biến thể như FlashSAC cải thiện việc này) |
| Độ ổn định huấn luyện | Rất ổn định nhờ clip | Nhạy hơn với chất lượng dữ liệu trong buffer |
| Cơ chế khám phá | Ngẫu nhiên nội tại của policy stochastic (+ entropy bonus tùy triển khai) | Entropy regularization tường minh trong objective |
| Độ phổ biến trong paper motion-tracking humanoid (DeepMimic/AMP/ASAP/SONIC) | Chủ lực, gần như mặc định | Hiếm, xuất hiện ở một số bài về dexterous manipulation |
| Chi phí tính toán mỗi bước cập nhật | Thấp hơn (không cần Q-function + replay buffer lớn) | Cao hơn (2 Q-network, target network, buffer khổng lồ) |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "PPO không khám phá vì không có entropy bonus như SAC."** Sai — PPO vẫn khám phá nhờ chính bản chất stochastic của policy (policy output là một phân phối, ví dụ Gaussian, không phải giá trị xác định), và hầu hết triển khai thực tế của PPO (bao gồm RSL-RL dùng trong Isaac Lab) vẫn cộng thêm một entropy bonus phụ trợ vào loss để khuyến khích khám phá thêm. Khác biệt thực sự là SAC đưa entropy vào **chính định nghĩa mục tiêu tối ưu J(θ)** một cách hình thức (maximum entropy RL), còn ở PPO đó chỉ là một trick huấn luyện bổ sung, không nằm trong định nghĩa gốc của thuật toán.

2. **Hiểu nhầm: "Clip trong PPO luôn giới hạn ratio trong [1-ε, 1+ε], không cho vượt qua khoảng này bao giờ."** Sai một phần — công thức `min(r·Â, clip(r,1-ε,1+ε)·Â)` chỉ **có tác dụng thực sự** (tức chặn gradient) khi cả điều kiện dấu của Â và hướng lệch của r cùng phù hợp (xem ví dụ tính tay: khi Â âm và r lớn, min vẫn chọn r·Â không bị clip). Nói cách khác, cơ chế clip là **bất đối xứng theo dấu advantage** — nó chặn việc "thưởng quá đà" cho hành động tốt khi ratio đã tăng mạnh, nhưng không chặn việc "phạt mạnh" hành động tệ dù ratio tăng mạnh theo chiều xấu.

3. **Hiểu nhầm: "SAC luôn tốt hơn PPO vì sample-efficient hơn, tại sao paper humanoid không dùng SAC?"** Nhầm lẫn giữa "hiệu quả về số lượng tương tác môi trường" và "hiệu quả về thời gian thực (wall-clock time)". Khi mô phỏng song song hàng nghìn robot ảo gần như miễn phí trên GPU (đúng bối cảnh huấn luyện humanoid trong Isaac Lab/MuJoCo Playground), số lượng tương tác không còn là tài nguyên khan hiếm — tốc độ cập nhật ổn định và khả năng song song hoá của PPO quan trọng hơn. SAC chỉ có lợi thế rõ rệt khi tương tác môi trường thực sự đắt đỏ (ví dụ học trực tiếp trên robot thật, không qua sim).

## 🏗️ Ví dụ minh hoạ trong dự án này

Trong pipeline của dự án này, PPO là "động cơ tối ưu hoá" đứng sau gần như mọi bài giảng khác trong thư mục 04: DeepMimic dùng PPO để tối ưu reward pha trộn imitation+task; AMP dùng PPO để tối ưu policy đồng thời huấn luyện adversarial discriminator; ASAP dùng PPO cả ở giai đoạn huấn luyện trong sim lẫn giai đoạn fine-tune với residual model; SONIC scale PPO lên tới 42M tham số, huấn luyện song song trên nhiều GPU với hàng nghìn environment ảo mô phỏng đồng thời (xem `../05-simulation-mujoco-isaaclab/`, mục RSL-RL và PPO trong Isaac Lab). SAC hầu như không xuất hiện trong chuỗi paper cốt lõi này, nhưng có liên quan gián tiếp: khi robot cần fine-tune trực tiếp trên phần cứng thật với số lượt thử giới hạn (ví dụ bước 2 của ASAP thu dữ liệu thật để học residual model), tính "sample-efficient" là đúng loại bài toán mà SAC (hoặc các biến thể off-policy khác) thường được cân nhắc trong cộng đồng rộng hơn, dù bản thân ASAP chọn một cách tiếp cận khác (residual model học offline, không phải RL on-hardware trực tiếp).

## 🔥 Cập nhật hiện đại / SOTA gần đây

Tra cứu WebSearch (9/2026) cho thấy cục diện PPO vs SAC cho humanoid đang dịch chuyển, không còn là "PPO thắng tuyệt đối" như giai đoạn 2021-2023:

- **FlashSAC** ([arXiv:2604.04539](https://arxiv.org/abs/2604.04539), 2026) — một biến thể SAC tối ưu hoá kỹ thuật (batch xử lý, kiến trúc mạng) cho thấy lợi thế rõ rệt trên các tác vụ chiều cao (dexterous manipulation, humanoid locomotion): hội tụ tới hiệu năng tiệm cận cao hơn PPO với thời gian wall-clock ít hơn đáng kể, và trong huấn luyện sim-to-real cho dáng đi humanoid, giảm thời gian huấn luyện từ hàng giờ xuống còn vài phút trong khi vẫn triển khai ổn định trên robot thật. Đây là bằng chứng đầu tiên đáng kể cho thấy SAC (dạng tối ưu hoá tốt) có thể cạnh tranh trực tiếp với PPO ngay cả trong bối cảnh mô phỏng song song mà trước đây PPO gần như độc quyền.
- **RSL-RL-SAC** ([arXiv:2605.24975](https://arxiv.org/abs/2605.24975), "Bridging the Gap: Enabling Soft Actor Critic for High Performance Legged Locomotion") — một báo cáo kỹ thuật đi kèm bản mở rộng mã nguồn mở của thư viện RSL-RL (thư viện PPO chuẩn dùng trong Isaac Lab, xem `../05-simulation-mujoco-isaaclab/`) để **hỗ trợ SAC** cho các tác vụ locomotion chân, cho thấy cộng đồng đang chủ động đưa SAC vào cùng hạ tầng song song hoá GPU vốn trước đây chỉ tối ưu cho PPO — thu hẹp khoảng cách "SAC khó song song hoá" nói ở mục so sánh.
- **Reparameterization PPO** ([arXiv:2508.06214](https://arxiv.org/abs/2508.06214), 2025) — một hướng cải tiến trực tiếp trên chính PPO, thay đổi cách tham số hoá gradient policy để cải thiện độ ổn định/tốc độ hội tụ, cho thấy bản thân PPO cũng không "đứng yên" mà vẫn tiếp tục được tinh chỉnh song song với sự trỗi dậy của SAC.
- Xu hướng chung 2025-2026: thay vì coi PPO/SAC là lựa chọn nhị phân cố định theo loại bài toán (PPO cho sim song song, SAC cho sample-efficient), các nhóm nghiên cứu đang tối ưu hoá kỹ thuật để mỗi thuật toán "lấn sân" sang thế mạnh truyền thống của thuật toán kia — SAC được tối ưu để chạy nhanh trên GPU song song (FlashSAC, RSL-RL-SAC), trong khi PPO tiếp tục được cải tiến về mặt lý thuyết tối ưu hoá (Reparameterization PPO).

## ❓ Câu hỏi tự kiểm tra

1. Vì sao PPO không thể tái sử dụng dữ liệu thu thập từ 5 iteration trước, còn SAC thì có thể?
<details><summary>Gợi ý đáp án</summary>PPO on-policy: ratio r_t(θ)=π_θ/π_θ_old chỉ có ý nghĩa đúng khi π_θ_old gần π_θ hiện tại; dữ liệu cũ được sinh bởi một policy đã "lạc hậu" quá xa, làm ước lượng advantage/gradient sai lệch nghiêm trọng. SAC học Q-function bằng Bellman backup, về nguyên tắc có thể học đúng từ transition của bất kỳ policy nào (dù chất lượng ước lượng phụ thuộc độ đa dạng dữ liệu trong buffer).</details>

2. Nếu ε (clip range) của PPO được đặt quá nhỏ (ví dụ 0.01), điều gì xảy ra với tốc độ học?
<details><summary>Gợi ý đáp án</summary>Policy gần như không được phép thay đổi nhiều mỗi bước cập nhật → học rất chậm, an toàn nhưng tốn nhiều iteration hơn để hội tụ. Đánh đổi giữa ổn định và tốc độ hội tụ.</details>

3. Trong ví dụ tính tay ở trên, nếu π_θ(a=0.5|s) chỉ tăng nhẹ lên 0.24 (thay vì 0.55), r_t(θ) là bao nhiêu, có bị clip không?
<details><summary>Gợi ý đáp án</summary>r_t(θ) = 0.24/0.20 = 1.2 — đúng biên clip trên (1+ε=1.2 với ε=0.2), về mặt kỹ thuật đây là ranh giới, chưa vượt ngưỡng nên chưa bị cắt xuống thấp hơn giá trị này (clip chỉ có tác dụng khi vượt QUÁ 1.2).</details>

4. Vì sao entropy regularization trong SAC giúp tránh "hội tụ sớm vào một chiến lược duy nhất", còn PPO thuần (không có entropy bonus phụ trợ) dễ gặp rủi ro này hơn?
<details><summary>Gợi ý đáp án</summary>Entropy H(π) đo độ "trải rộng" của phân phối hành động — tối đa hoá thêm α·H buộc policy phải giữ một mức độ ngẫu nhiên tối thiểu tại mọi trạng thái, chống lại xu hướng tự nhiên của gradient ascent là hội tụ càng nhanh càng tốt về một hành động xác định (deterministic) khi đã tìm được vùng advantage dương. PPO thuần không có ràng buộc này nên nếu advantage estimate bị nhiễu/thiên lệch sớm, policy có thể "chốt" quá sớm vào một hành vi cục bộ.</details>

5. Tại sao FlashSAC và RSL-RL-SAC (2026) lại là tin tức đáng chú ý, nếu biết rằng từ 2021-2023 gần như mọi paper humanoid RL đều mặc định chọn PPO?
<details><summary>Gợi ý đáp án</summary>Vì chúng thách thức trực tiếp lý do chính khiến PPO được chọn (khả năng song song hoá tốt trên GPU) — nếu SAC được tối ưu kỹ thuật để cũng chạy nhanh song song, lợi thế "sample efficiency" vốn có của SAC không còn bị triệt tiêu bởi nhược điểm tốc độ, khiến bài toán chọn thuật toán không còn mặc định nghiêng hẳn về PPO nữa.</details>

## 📝 Bài tập thực hành

1. Cài đặt PPO từ đầu (không dùng thư viện có sẵn, tự viết loss L^CLIP) trên môi trường CartPole-v1 của Gymnasium. Thử 3 giá trị ε khác nhau (0.05, 0.2, 0.5) và vẽ đường cong reward theo iteration — quan sát trade-off ổn định vs tốc độ hội tụ đã nói ở câu hỏi tự kiểm tra #2.
2. Lặp lại ví dụ tính tay ở mục "Ví dụ tính tay" nhưng với một bộ số khác tự chọn: π_θ_old(a|s)=0.35, π_θ(a|s)=0.10 (policy mới giảm xác suất hành động này), Â_t=+1.5, ε=0.15. Tính r_t(θ), xác định có bị clip không, và tính L^CLIP cuối cùng — giải thích ý nghĩa của việc r_t(θ) < 1 (policy giảm xác suất) khi advantage dương.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

PPO và SAC là hai thuật toán RL nền tảng đứng sau mọi pipeline huấn luyện humanoid trong dự án này: PPO là on-policy, dùng clipped surrogate objective để giới hạn mức thay đổi policy mỗi bước cập nhật, phải vứt bỏ dữ liệu sau mỗi vài epoch, nhưng song song hoá cực tốt trên hàng nghìn môi trường mô phỏng GPU — đây là lý do gần như mọi paper motion-tracking (DeepMimic, AMP, ASAP, SONIC) chọn nó; SAC là off-policy, tái sử dụng dữ liệu qua replay buffer và tối ưu thêm entropy để khuyến khích khám phá, truyền thống "sample-efficient" hơn nhưng khó song song hoá hơn — dù các công trình 2025-2026 như FlashSAC và RSL-RL-SAC đang tích cực thu hẹp khoảng cách này, cho thấy ranh giới lựa chọn giữa hai thuật toán không còn cố định như trước.
