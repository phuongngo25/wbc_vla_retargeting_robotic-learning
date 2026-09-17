# Bài giảng: Reward thủ công vs. Motion-tracking reward

*(Thuộc mảng: Imitation Learning & Reinforcement Learning)*

## 🎯 Mục tiêu bài học

- Giải thích được vì sao reward thủ công (hand-designed reward) cho robot locomotion khó mở rộng để tạo dáng đi tự nhiên.
- Định nghĩa chính xác công thức motion-tracking reward (Gaussian kernel trên sai lệch pose/vận tốc/root) và giải thích ý nghĩa từng số hạng.
- Phân biệt được **dense reward** và **sparse reward**, giải thích vì sao motion-tracking reward biến RL thành "imitation learning có giám sát dày đặc".
- Tính tay được giá trị reward tổng hợp cho một frame cụ thể, với số liệu giả định.
- Nhận diện được hiện tượng **reward hacking** trong reward thủ công và giải thích vì sao motion-tracking reward giảm thiểu (nhưng không loại bỏ hoàn toàn) vấn đề này.
- So sánh được khi nào nên dùng reward thủ công, khi nào nên dùng motion-tracking reward.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Đây là "bản lề" giữa hai thế giới: RL locomotion cổ điển (thiết kế reward bằng tay, như Rudin et al. 2022 trong `../01-whole-body-control/`) và toàn bộ chuỗi paper motion-tracking humanoid mà thư mục này sẽ nói tiếp (DeepMimic, AMP, ASAP, SONIC). Trước khi hiểu DeepMimic dùng reward gì, bạn cần hiểu chính xác vấn đề mà motion-tracking reward giải quyết — nếu không, các bài giảng sau sẽ chỉ là "công thức từ trên trời rơi xuống" thay vì một giải pháp có động cơ rõ ràng cho một vấn đề cụ thể.

## 🧠 Trực giác

### Góc nhìn 1: Chấm điểm bài luận theo tiêu chí tự đặt (reward thủ công) vs chấm theo văn mẫu (motion-tracking)

Reward thủ công giống việc giáo viên tự đặt ra một danh sách tiêu chí chấm điểm cho bài luận: "có mở bài không", "độ dài câu hợp lý không", "dùng từ nối chưa"... Danh sách càng dài, càng cố "vá" các trường hợp học sinh viết kỳ quặc nhưng vẫn đạt điểm cao theo đúng tiêu chí (ví dụ học sinh nhồi nhét từ nối vô nghĩa để đạt tiêu chí "dùng từ nối"). Motion-tracking reward giống việc đưa cho học sinh một bài văn mẫu cụ thể và chấm điểm dựa trên **mức độ giống bài mẫu ở từng câu, từng đoạn** — không cần liệt kê tiêu chí trừu tượng, chỉ cần so khớp trực tiếp với ví dụ cụ thể đã có sẵn.

*Đúng ở đâu:* nắm đúng bản chất "tiêu chí trừu tượng, dễ bị lách" (reward thủ công) vs "so khớp cụ thể với mẫu thật" (motion-tracking).
*Giới hạn:* phép loại suy không phản ánh đúng rằng motion-tracking reward vẫn cần *một chút* thiết kế thủ công (chọn hằng số k1..k4, trọng số w trong công thức Gaussian kernel) — nó không hoàn toàn "miễn phí thiết kế", chỉ là gánh nặng thiết kế chuyển từ "nghĩ ra tiêu chí gì" sang "cân trọng số giữa các thành phần khoảng cách đã có công thức cố định".

### Góc nhìn 2: GPS chỉ đường theo toạ độ đích (reward thủ công dạng sparse/mục tiêu) vs GPS bám theo vệt đường có sẵn (motion-tracking)

Nếu chỉ thưởng robot khi "đi tới đích" hoặc "đạt vận tốc tiến mong muốn" (một dạng reward thủ công đơn giản, gần sparse), điều đó giống việc chỉ cho tài xế biết toạ độ đích mà không chỉ đường — tài xế (agent) phải tự dò đường, có thể đi theo tuyến kỳ lạ miễn tới đích đúng lúc. Motion-tracking reward giống việc có sẵn một vệt GPS ghi lại đúng lộ trình một tài xế giỏi đã đi qua trước đó, và mỗi giây đều được chấm "bạn đang lệch bao xa khỏi vệt này ngay bây giờ" — tín hiệu phản hồi có ở **mọi thời điểm**, không chỉ khi tới đích.

*Đúng ở đâu:* làm rõ trực giác "dense vs sparse" — vì sao motion-tracking reward giúp học nhanh hơn nhờ phản hồi liên tục.
*Giới hạn:* phép loại suy khiến người học tưởng rằng reward thủ công luôn sparse — thực tế reward thủ công (như trong Rudin et al. 2022) thường **cũng dense** (thưởng mỗi bước dựa vào vận tốc tức thời, hình phạt năng lượng mỗi bước...), chỉ là nó dense theo tiêu chí tự đặt (không tham chiếu một quỹ đạo cụ thể nào), khác với motion-tracking dense theo tham chiếu một trajectory mocap cụ thể.

## 📐 Định nghĩa chính xác

**Reward thủ công (hand-designed reward):** hàm reward được thiết kế trực tiếp bởi kỹ sư dưới dạng tổ hợp tuyến tính có trọng số của các đại lượng đo được, không tham chiếu tới bất kỳ dữ liệu chuyển động mẫu nào:

```
r = w1·(vận_tốc_tiến_thực_tế) − w2·(Σ τ_joint²) − w3·(chỉ_báo_ngã) − w4·(‖a_t − a_{t-1}‖²) + ...
```

trong đó τ_joint là mô-men lực từng khớp (hình phạt năng lượng), a_t là hành động tại bước t (hình phạt giật/jerk). Số lượng số hạng và trọng số w_i hoàn toàn do kỹ sư quyết định, thường qua thử-sai (trial and error).

**Motion-tracking reward:** reward tại mỗi timestep t được định nghĩa là hàm khoảng cách dạng **Gaussian kernel âm mũ** giữa trạng thái mô phỏng của robot và trạng thái tham chiếu (reference) lấy từ một frame tương ứng trong clip mocap đã retarget:

```
r_pose      = exp(−k1·‖q_robot − q_reference‖²)
r_velocity  = exp(−k2·‖q̇_robot − q̇_reference‖²)
r_root_pose = exp(−k3·‖root_pos/rot_robot − root_ref‖²)
r_root_vel  = exp(−k4·‖root_vel_robot − root_vel_ref‖²)
r_total     = w_p·r_pose + w_v·r_velocity + w_rp·r_root_pose + w_rv·r_root_vel
```

với q là vector góc khớp, q̇ là vận tốc góc khớp, root_pos/rot là vị trí/hướng của pelvis (gốc cơ thể), k1..k4 là hằng số kiểm soát "độ dốc" của Gaussian kernel (k lớn → phạt nặng khi lệch nhỏ), w_p, w_v, w_rp, w_rv là trọng số tương đối giữa 4 thành phần.

Vì r ∈ (0, 1] luôn dương (đặc tính của hàm exp(−x²) với x thực), motion-tracking reward tại mọi thời điểm luôn là một tín hiệu "độ giống" liên tục — bằng 1 khi khớp hoàn hảo với reference, tiến về 0 khi lệch rất xa.

## ⚙️ Cơ chế hoạt động — từng bước

```
┌───────────────────────────────────────────────────────────────┐
│ Pipeline reward thủ công (hand-designed)                        │
│                                                                   │
│  Kỹ sư quan sát dáng đi robot trong sim                          │
│         │                                                        │
│         ▼                                                        │
│  Viết công thức reward (v tiến, phạt năng lượng, phạt ngã...)    │
│         │                                                        │
│         ▼                                                        │
│  Huấn luyện PPO → xem robot học được gì                          │
│         │                                                        │
│         ▼                                                        │
│  Dáng đi kỳ quặc/reward hacking? ──Yes──► Thêm số hạng reward     │
│         │                                 mới, quay lại bước viết │
│         No                                công thức               │
│         ▼                                                        │
│  Chấp nhận dáng đi hiện tại                                       │
└───────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────┐
│ Pipeline motion-tracking reward                                   │
│                                                                   │
│  Clip mocap người thật ──► Retarget (xem 02-motion-retargeting/) │
│         │                    ──► trajectory tham chiếu cho robot  │
│         ▼                                                        │
│  Tại mỗi timestep t trong episode huấn luyện:                    │
│    - Lấy state robot mô phỏng s_t = (q, q̇, root_pos, root_vel)   │
│    - Lấy state tham chiếu cùng thời điểm s_ref,t từ clip mocap    │
│    - Tính r_total(t) = Σ w_i · exp(−k_i·‖s_t,i − s_ref,t,i‖²)     │
│         │                                                        │
│         ▼                                                        │
│  Cộng r_total(t) vào tổng reward episode, dùng PPO cập nhật θ     │
│         │                                                        │
│         ▼                                                        │
│  Không cần chỉnh sửa công thức reward khi đổi clip mocap mới —   │
│  chỉ cần đổi trajectory tham chiếu s_ref,t                        │
└───────────────────────────────────────────────────────────────┘
```

Điểm khác biệt cơ chế cốt lõi: pipeline reward thủ công là một **vòng lặp thiết kế lại công thức liên tục** (nằm ngoài vòng lặp RL), còn pipeline motion-tracking là một **công thức cố định, chỉ thay dữ liệu đầu vào** (đổi clip mocap) — gánh nặng kỹ thuật chuyển từ "nghĩ lại công thức" sang "thu thập/retarget đủ dữ liệu mocap đa dạng".

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số liệu tự chọn để dễ hình dung — đơn giản hoá xuống 1 khớp gối để dễ theo dõi, thực tế công thức áp dụng trên toàn bộ vector nhiều chiều.)*

Giả sử tại timestep t, robot mô phỏng có góc khớp gối q_robot = 0.62 rad, vận tốc khớp gối q̇_robot = 1.1 rad/s. Clip mocap tham chiếu tại cùng thời điểm có q_reference = 0.55 rad, q̇_reference = 0.9 rad/s. Chọn hằng số minh hoạ k1 = 10 (cho pose), k2 = 2 (cho vận tốc), bỏ qua root pose/vel để đơn giản (giả định robot đứng yên tại chỗ, root khớp hoàn hảo nên r_root_pose = r_root_vel = 1).

Bước 1 — tính sai lệch bình phương:
```
‖q_robot − q_reference‖² = (0.62 − 0.55)² = 0.07² = 0.0049
‖q̇_robot − q̇_reference‖² = (1.1 − 0.9)² = 0.2² = 0.04
```

Bước 2 — tính từng thành phần reward:
```
r_pose     = exp(−10 × 0.0049) = exp(−0.049) ≈ 0.9522
r_velocity = exp(−2 × 0.04)    = exp(−0.08)  ≈ 0.9231
```

Bước 3 — tổng hợp với trọng số minh hoạ w_p = 0.5, w_v = 0.3, w_rp = 0.15, w_rv = 0.05 (r_root_pose = r_root_vel = 1):
```
r_total = 0.5×0.9522 + 0.3×0.9231 + 0.15×1 + 0.05×1
        = 0.4761 + 0.2769 + 0.15 + 0.05
        = 0.9530
```

**Diễn giải:** robot lệch khá ít so với reference (0.07 rad ≈ 4°, 0.2 rad/s) nên nhận reward gần tối đa (0.953/1.0). Nếu sai lệch pose tăng gấp đôi (0.14 rad thay vì 0.07 rad), ‖·‖² tăng gấp 4 lần (0.0196), r_pose = exp(−10×0.0196) = exp(−0.196) ≈ 0.822 — giảm rõ rệt hơn hẳn so với mức giảm tuyến tính, minh hoạ đặc tính "phạt tăng nhanh khi lệch xa" của Gaussian kernel (khác hẳn reward tuyến tính đơn giản như −w×|lệch|, vốn phạt tuyến tính đều đặn theo độ lệch).

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Reward thủ công | Motion-tracking reward |
|---|---|---|
| Nguồn "đáp án đúng" | Trực giác/kinh nghiệm kỹ sư | Dữ liệu mocap người thật đã retarget |
| Độ dày tín hiệu (dense/sparse) | Thường dense nhưng theo tiêu chí tự đặt | Dense, theo tham chiếu quỹ đạo cụ thể |
| Rủi ro reward hacking | Cao — càng nhiều số hạng càng dễ bị lách | Thấp hơn nhưng vẫn có thể (robot "trung bình hoá" giữa các reference gần nhau nếu trộn nhiều clip không cẩn thận) |
| Chi phí thiết kế khi thêm kỹ năng mới | Cao — phải nghĩ lại/tinh chỉnh công thức | Thấp hơn — chỉ cần thêm clip mocap mới |
| Tính tự nhiên của hành vi học được | Phải mô tả "tự nhiên" bằng số hạng cụ thể (đối xứng, nhịp bước...) | Tự động thừa hưởng từ dữ liệu người thật |
| Khả năng tổng quát khi robot cần sáng tạo động tác không có trong dữ liệu | Tốt hơn (không bị ràng buộc bởi bất kỳ clip cụ thể nào) | Kém hơn ở dạng thuần DeepMimic-style (khắc phục một phần bằng AMP — xem bài giảng riêng) |
| Ví dụ điển hình | Rudin et al. 2022 (`../01-whole-body-control/`) | DeepMimic, AMP, ASAP, SONIC (các bài giảng khác trong thư mục này) |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "Motion-tracking reward không cần thiết kế thủ công gì cả, hoàn toàn tự động."** Sai — các hằng số k1..k4 và trọng số w_p, w_v, w_rp, w_rv vẫn là hyperparameter phải chọn/tinh chỉnh bằng tay (khác nhẹ giữa các paper, như chính NOI-DUNG-CHI-TIET.md của thư mục này lưu ý). Điều "tự động hoá" thực sự là phần **nội dung động tác** (không cần tự nghĩ ra thế nào là dáng đi tự nhiên), không phải toàn bộ quá trình thiết kế reward.

2. **Hiểu nhầm: "Reward hacking chỉ xảy ra với reward thủ công, motion-tracking reward miễn nhiễm."** Sai — motion-tracking reward thuần (kiểu DeepMimic, bám 1 clip cụ thể) vẫn có thể bị "lách" theo cách khác: nếu trọng số giữa các thành phần (pose/velocity/root) không cân bằng, policy có thể học cách tối đa hoá thành phần dễ (ví dụ giữ root gần đúng vị trí) mà bỏ bê thành phần khó (chi tiết góc khớp bàn tay), miễn tổng reward vẫn cao. Đây là lý do các paper sau này (AMP) chuyển sang discriminator thay vì khoảng cách trực tiếp — xem bài giảng riêng "AMP — cơ chế discriminator".

3. **Hiểu nhầm: "Vì motion-tracking reward tốt hơn, reward thủ công đã lỗi thời, không ai dùng nữa."** Sai — reward thủ công vẫn là lựa chọn hợp lý khi không có dữ liệu mocap phù hợp cho tác vụ (ví dụ các hành vi mới hoàn toàn không có trong bất kỳ dataset mocap nào), hoặc khi mục tiêu là tối ưu một chỉ số vật lý cụ thể (tiết kiệm năng lượng, tốc độ tối đa) hơn là bắt chước con người. Hai cách tiếp cận thường được **kết hợp** trong thực tế (task reward + imitation reward, xem bài giảng DeepMimic).

## 🏗️ Ví dụ minh hoạ trong dự án này

Trong pipeline của dự án, `02-motion-retargeting/` tạo ra chính xác "s_ref,t" — trajectory robot đã retarget từ mocap người — là input trực tiếp cho công thức motion-tracking reward ở đây. Không có bước retargeting chất lượng tốt, motion-tracking reward sẽ tính khoảng cách so với một reference "méo mó" (do lỗi retargeting), khiến robot học theo một chuẩn sai. README của thư mục 04 (mục D, bước 6) đề xuất bài tập thực hành trực tiếp: chạy 1 task motion-tracking đơn giản trong MuJoCo Playground (`humanoid-walk`) và thử đổi reward từ dạng "tốc độ tiến" (thuần thủ công, kiểu Rudin et al. 2022) sang dạng "khoảng cách tới 1 pose tham chiếu" (công thức ở bài này) — đây chính là cách trực quan nhất để cảm nhận sự khác biệt đã học trong bài giảng này.

## 🔥 Cập nhật hiện đại / SOTA gần đây

- **SONIC** ([arXiv:2511.07820](https://arxiv.org/abs/2511.07820), Nvidia/CMU, 2025) chỉ ra rằng thiết kế reward locomotion truyền thống thường cần **hơn 20 số hạng** reward-shaping (tracking kinematic, posture regularizer, penalty khớp, các shaping term khác) để có hành vi ổn định — nhưng các công trình gần đây (bao gồm chính SONIC) cho thấy hành vi **robust và tự nhiên có thể xuất hiện từ một tập mục tiêu đơn giản hơn nhiều (dưới 10 số hạng)** khi dựa trên motion-tracking reward với dữ liệu mocap đủ đa dạng và quy mô đủ lớn — củng cố luận điểm cốt lõi của bài này: motion-tracking reward giảm gánh nặng thiết kế reward thủ công một cách có hệ thống, không chỉ là một trick cục bộ.
- **"Learning Sim-to-Real Humanoid Locomotion in 15 Minutes"** ([arXiv:2512.01996](https://arxiv.org/pdf/2512.01996), 2025) — báo cáo rằng pipeline huấn luyện hiện đại (kết hợp motion-tracking reward tinh gọn + domain randomization) có thể huấn luyện chính sách đi bộ sim-to-real thành công trong khoảng 15 phút huấn luyện, một minh chứng thực nghiệm cho việc đơn giản hoá reward giúp tăng tốc hội tụ đáng kể so với các pipeline reward thủ công phức tạp truyền thống.
- **"Two-Layered Reward Reinforcement Learning in Humanoid Robot Motion Tracking"** (MDPI Mathematics, 2025) — đề xuất kiến trúc reward hai lớp, tách biệt lớp reward "theo dõi động học" (giống motion-tracking reward ở bài này) và một lớp reward bổ sung điều chỉnh hành vi cấp cao hơn, cho thấy xu hướng hiện tại không dừng ở việc chọn một trong hai cách (thủ công hoặc tracking) mà đang phát triển các kiến trúc reward phân tầng kết hợp ưu điểm của cả hai.

## ❓ Câu hỏi tự kiểm tra

1. Vì sao motion-tracking reward được gọi là "imitation learning có giám sát dày đặc" thay vì RL thông thường?
<details><summary>Gợi ý đáp án</summary>Vì mỗi timestep đều có tín hiệu phản hồi rõ ràng (khoảng cách tới pose tham chiếu ngay lúc đó), gần giống bài toán regression theo từng bước, thay vì phải khám phá rộng để tìm ra hành vi tốt như RL với sparse reward.</details>

2. Nếu k1 trong công thức r_pose = exp(−k1·‖q_robot−q_reference‖²) được tăng gấp 10 lần, điều gì xảy ra với độ "khắt khe" của reward?
<details><summary>Gợi ý đáp án</summary>k1 lớn hơn khiến reward giảm nhanh hơn nhiều khi có sai lệch nhỏ (hàm exp dốc hơn quanh 0) — reward trở nên khắt khe hơn, đòi hỏi robot bám sát reference chính xác hơn mới nhận reward cao, có thể khiến việc học khó hơn nếu k1 quá lớn (reward gần như luôn ~0 trừ khi cực kỳ chính xác).</details>

3. Trong ví dụ tính tay, nếu chỉ có r_pose và r_velocity thay đổi còn r_root_pose/r_root_vel giữ nguyên = 1, tại sao r_total vẫn không giảm về 0 dù r_pose và r_velocity giảm đáng kể?
<details><summary>Gợi ý đáp án</summary>Vì r_total là tổng có trọng số của 4 thành phần, hai thành phần root (chiếm 0.15+0.05=0.2 trọng số) vẫn đóng góp đầy đủ giá trị tối đa; chỉ có 0.8 trọng số (0.5 pose + 0.3 velocity) chịu ảnh hưởng của sự suy giảm.</details>

4. Vì sao SONIC (2025) lại giảm số lượng số hạng reward-shaping từ 20+ xuống dưới 10 mà vẫn giữ được hành vi tự nhiên, trong khi lẽ ra ít ràng buộc hơn thường dẫn tới hành vi kém kiểm soát hơn?
<details><summary>Gợi ý đáp án</summary>Vì phần "tự nhiên" không còn dựa vào các ràng buộc thủ công (đối xứng, nhịp bước...) mà dựa trực tiếp vào dữ liệu mocap đa dạng, quy mô lớn — bản thân dữ liệu đã "mã hoá" các ràng buộc tự nhiên đó, nên không cần thêm số hạng reward để ép buộc chúng một cách tường minh.</details>

## 📝 Bài tập thực hành

1. Trong MuJoCo Playground hoặc Isaac Lab (`../05-simulation-mujoco-isaaclab/`), tìm một task locomotion có sẵn dùng reward thủ công (ví dụ velocity tracking đơn giản). Đọc code reward và liệt kê từng số hạng — xác định số hạng nào có nguy cơ bị "lách" (reward hacking) nếu bỏ qua các số hạng khác.
2. Tính lại ví dụ ở mục "Ví dụ tính tay" nhưng với sai lệch vận tốc lớn hơn nhiều: q̇_robot = 2.5 rad/s, q̇_reference = 0.9 rad/s (giữ nguyên các số khác). Tính r_velocity mới và r_total mới — so sánh mức giảm với ví dụ gốc để cảm nhận độ "phạt nặng" phi tuyến của Gaussian kernel khi sai lệch vượt một ngưỡng nhất định.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

Reward thủ công đòi hỏi kỹ sư tự nghĩ ra và tinh chỉnh từng số hạng (vận tốc, năng lượng, ngã, giật) để mô tả "hành vi tốt", dễ rơi vào vòng lặp vá lỗi vô tận và reward hacking khi muốn có dáng đi tự nhiên; motion-tracking reward thay thế bằng một công thức Gaussian kernel cố định đo khoảng cách pose/vận tốc/root giữa robot mô phỏng và một clip mocap người thật đã retarget, biến bài toán thành "imitation learning có giám sát dày đặc" — tự động thừa hưởng sự tự nhiên từ dữ liệu thay vì phải mô tả nó bằng công thức, và đây chính là nền tảng reward mà DeepMimic, AMP, ASAP, SONIC (các bài giảng tiếp theo) đều xây dựng lên trên.
