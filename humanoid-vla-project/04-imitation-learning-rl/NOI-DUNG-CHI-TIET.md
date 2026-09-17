# Nội dung chi tiết — Imitation Learning & Reinforcement Learning cho Humanoid Motion Tracking

> File này giải thích **cơ chế** đằng sau 7 khái niệm ở mục A của `README.md`. Mục tiêu: đọc xong hiểu được *tại sao* các paper này hoạt động, không cần tự lần mò paper gốc để nắm ý chính. Với phần RL nền tảng (PPO/SAC), xem `../../resources/08-core-reading-list.md` Track C3 để đọc sâu — ở đây chỉ tóm tắt đủ dùng.

---

## 1. Tóm tắt nhanh RL cơ bản cần nhớ

Trong reinforcement learning (RL), một **policy** π(a|s) là hàm ánh xạ từ trạng thái quan sát được (state/observation — ví dụ góc khớp, vận tốc, dữ liệu IMU) sang hành động (action — ví dụ mô-men lực đặt vào từng khớp). Robot/agent tương tác với môi trường theo policy, tạo ra một **trajectory** (τ) — chuỗi (state, action, reward) theo thời gian: s₀, a₀, r₀, s₁, a₁, r₁, ... Mục tiêu huấn luyện là chỉnh policy để **tổng reward tích lũy kỳ vọng** trên trajectory là lớn nhất.

**PPO (Proximal Policy Optimization)** là thuật toán **on-policy**: dữ liệu huấn luyện phải được thu thập bằng chính policy hiện tại (hoặc rất gần nó), không tái sử dụng dữ liệu cũ một cách tự do. PPO cập nhật policy bằng gradient ascent nhưng giới hạn (clip) mức độ thay đổi giữa policy mới và policy cũ trong mỗi bước cập nhật (**clipped surrogate objective**) — tránh việc một bước cập nhật quá lớn làm policy sụp đổ. Đây là thuật toán RL chủ lực trong hầu hết các paper motion-tracking humanoid (DeepMimic, AMP, ASAP, SONIC đều dùng PPO hoặc biến thể của nó) vì nó ổn định, dễ scale với huấn luyện song song hàng nghìn môi trường mô phỏng.

**SAC (Soft Actor-Critic)** khác PPO ở hai điểm: (1) **off-policy** — có thể học lại từ dữ liệu cũ lưu trong replay buffer, không bắt buộc phải luôn dùng dữ liệu mới nhất, nên thường "sample-efficient" hơn; (2) **entropy-regularized** — mục tiêu tối ưu không chỉ là tổng reward mà còn cộng thêm entropy của policy (mức độ "ngẫu nhiên/đa dạng" của hành động), khuyến khích policy khám phá nhiều chiến lược khác nhau thay vì hội tụ sớm vào một hành vi duy nhất. SAC ít phổ biến hơn PPO trong các paper motion-tracking on-simulation-song-song (vì PPO song song hoá tốt hơn trên GPU), nhưng vẫn xuất hiện trong một số công trình manipulation/locomotion.

---

## 2. Reward thủ công vs. Motion-tracking reward — cơ chế chi tiết

### Reward thủ công (hand-designed reward)

Cách RL "cổ điển" cho robot locomotion: kỹ sư tự viết công thức reward theo tay, kiểu:

```
reward = w1 * (vận_tốc_tiến_thực_tế) 
       - w2 * (hình_phạt_năng_lượng, ví dụ tổng bình phương mô-men lực)
       - w3 * (hình_phạt_ngã / lệch tư thế)
       - w4 * (hình_phạt_giật lắc, thay đổi hành động đột ngột)
       + ...
```

Đây là cách tiếp cận của các paper như Rudin et al. 2022 (*"Learning to Walk in Minutes"*, [arXiv:2109.11978](https://arxiv.org/abs/2109.11978)) — hiệu quả cho đi bộ đơn giản, nhưng có nhược điểm lớn: để có dáng đi **tự nhiên, giống người**, kỹ sư phải thêm ngày càng nhiều số hạng reward (đối xứng trái-phải, độ cao chân, nhịp bước...) — mỗi số hạng lại cần tinh chỉnh trọng số bằng tay, dễ tạo ra hành vi "đúng metric nhưng kỳ quặc" (reward hacking) vì policy tìm ra cách lách công thức reward mà không thực sự tự nhiên.

### Motion-tracking reward

Ý tưởng thay thế: thay vì tự nghĩ ra công thức, dùng **dữ liệu mocap người thật đã retarget sang robot** (từ `02-motion-retargeting/`) làm "đáp án mẫu" (reference motion). Reward tại mỗi frame/timestep được định nghĩa là **hàm khoảng cách** giữa trạng thái hiện tại của robot mô phỏng và trạng thái tham chiếu tại cùng thời điểm trong clip mocap, thường có dạng mũ âm (Gaussian kernel):

```
r_pose      = exp(-k1 * ||q_robot - q_reference||²)        # vị trí góc khớp
r_velocity  = exp(-k2 * ||q̇_robot - q̇_reference||²)        # vận tốc khớp
r_root_pose = exp(-k3 * ||root_pos/rot_robot - root_ref||²) # vị trí & hướng gốc (pelvis)
r_root_vel  = exp(-k4 * ||root_vel_robot - root_vel_ref||²) # vận tốc góc/tuyến tính của gốc
r_total     = w_p*r_pose + w_v*r_velocity + w_rp*r_root_pose + w_rv*r_root_vel
```

(Hằng số k1..k4 và trọng số w là hyperparameter; công thức cụ thể khác nhau nhẹ giữa các paper nhưng ý tưởng lõi giống nhau.)

**Tại sao đây là "imitation learning có giám sát dày đặc" (dense supervision)?** Trong RL cổ điển, tín hiệu reward thường **sparse** (thưa) — ví dụ chỉ nhận reward khi hoàn thành mục tiêu (đi được 1m, không ngã), agent phải tự khám phá rất nhiều để tìm ra hành vi tốt. Với motion-tracking reward, **mỗi frame** đều có một tín hiệu reward rõ ràng: "bạn đang lệch bao xa so với pose mẫu ngay lúc này". Điều này biến bài toán RL thăm dò khó thành bài toán gần giống **supervised learning theo từng bước** (regression tới pose mục tiêu, nhưng học qua tương tác vật lý thay vì học trực tiếp trên nhãn) — giúp huấn luyện nhanh hội tụ hơn nhiều so với thiết kế reward thủ công, đồng thời tự động thừa hưởng "phong cách tự nhiên" từ dữ liệu người thật mà không cần kỹ sư tự mô tả thế nào là "tự nhiên".

---

## 3. DeepMimic (2018) — cơ chế chi tiết

DeepMimic (Peng, Abbeel, Levine, van de Panne, 2018, [arXiv:1804.02717](https://arxiv.org/abs/1804.02717)) là paper đặt nền móng cho toàn bộ hướng motion-tracking-as-RL. Ba cơ chế cốt lõi:

### 3.1 Reward pha trộn: imitation reward + task reward

Reward tổng = kết hợp có trọng số giữa:
- **Imitation reward** — đúng như mô tả ở mục 2: khoảng cách pose so với clip mocap tham chiếu (khớp, vận tốc, gốc).
- **Task reward** — mục tiêu bổ sung tùy bài toán cụ thể, ví dụ: ném bóng trúng đích, đi theo hướng mong muốn, đạt vận tốc mục tiêu. Task reward cho phép nhân vật/robot **không chỉ sao chép y hệt clip mẫu** mà còn thích nghi động tác để hoàn thành mục tiêu (ví dụ: động tác đá bóng được điều chỉnh hướng đá theo vị trí bóng thực tế, thay vì lặp lại chính xác góc đá trong clip gốc).

### 3.2 Reference State Initialization (RSI)

Thay vì luôn bắt đầu episode huấn luyện từ trạng thái đứng yên ban đầu cố định của clip mocap, RSI **lấy mẫu ngẫu nhiên một frame bất kỳ trong clip mocap** làm trạng thái khởi tạo cho mỗi episode. Điều này quan trọng vì: với các động tác động học phức tạp (nhào lộn, đá, xoay người có tiếp đất ngắt quãng), nếu chỉ luôn học từ đầu clip, agent phải "sống sót" qua toàn bộ chuỗi hành động dài trước khi chạm tới các đoạn giữa/cuối khó — rất kém hiệu quả để khám phá. RSI cho phép agent học "song song" ở nhiều điểm vào khác nhau của clip, tăng tốc học đáng kể, đặc biệt cho động tác khó.

### 3.3 Early Termination (ET)

Episode bị **kết thúc sớm** ngay khi phát hiện thất bại rõ ràng — ví dụ thân/torso chạm đất (ngã). Lý do: nếu không chấm dứt sớm, agent sẽ tiếp tục tích lũy trải nghiệm ở trạng thái ngã/thất bại (trạng thái vô nghĩa để học tiếp), làm loãng dữ liệu huấn luyện bằng các mẫu không hữu ích, thậm chí khiến policy học cách "chấp nhận nằm im" vì vẫn nhận được một phần reward nhỏ. Chấm dứt sớm buộc agent tập trung học các trajectory dẫn tới trạng thái thành công.

> Theo chính paper: RSI và ET là **thiết yếu** để học các kỹ năng động học khó (spins, kicks, flips có ngắt quãng tiếp đất) — với lựa chọn mặc định thông thường (trạng thái khởi tạo cố định + episode có độ dài cố định), việc bắt chước các động tác khó này thường thất bại.

---

## 4. AMP — Adversarial Motion Priors (2021) — cơ chế discriminator chi tiết

### 4.1 Vấn đề mà DeepMimic-style reward gặp phải

Reward dạng "khoảng cách pose chính xác từng frame" (mục 2–3) hoạt động tốt khi học **một clip cụ thể**, nhưng có nhược điểm: nó buộc policy phải bám sát *chính xác từng khung hình* của đúng một trajectory tham chiếu. Khi muốn học từ **nhiều clip cùng lúc** hoặc tổng quát hoá sang chuyển động chưa từng thấy, cách tính khoảng cách frame-by-frame trở nên cứng nhắc — rất dễ overfit vào từng clip riêng lẻ, và không có khái niệm rõ ràng nào cho "sự tự nhiên nói chung" tách khỏi một quỹ đạo cụ thể.

### 4.2 Cơ chế discriminator (giống GAN)

AMP (Peng, Ma, Abbeel, Levine, Kanazawa, 2021, [arXiv:2104.02180](https://arxiv.org/abs/2104.02180)) thay thế reward "khoảng cách pose" bằng một **discriminator** — một mạng phân loại được huấn luyện đồng thời với policy, theo tinh thần Generative Adversarial Network (GAN):

- Discriminator nhận vào một **cặp trạng thái liên tiếp** (state transition, s_t → s_{t+1}) và phải phân loại: cặp này đến từ **dữ liệu mocap thật** (nhãn "thật") hay từ **chuyển động do policy hiện tại sinh ra** (nhãn "giả")?
- Discriminator được huấn luyện để tối đa hoá khả năng phân biệt đúng hai nguồn này.
- Ngược lại, **policy được huấn luyện để "đánh lừa" discriminator** — tức là sinh ra các chuyển động mà discriminator không phân biệt được với dữ liệu thật. Điểm số/tín hiệu đầu ra của discriminator (biến đổi qua một hàm mượt, dạng r^style(s_t) = α·max(0, 1 − ¼(d−1)²) trong paper) được dùng trực tiếp làm **reward phong cách (style reward)** cho policy.
- Reward tổng thường kết hợp: r = w_G · r_task (task reward, ví dụ đi tới đích) + w_S · r_style (điểm discriminator).

Đây là một vòng lặp **adversarial**: khi policy cải thiện, các transition nó sinh ra ngày càng giống dữ liệu thật hơn, khiến discriminator khó phân biệt hơn (độ chính xác giảm dần) — buộc discriminator phải học tinh vi hơn, tạo áp lực ngược lại buộc policy tiếp tục cải thiện độ "tự nhiên". Quá trình hội tụ khi discriminator không còn phân biệt được policy-generated motion với dữ liệu thật.

### 4.3 Vì sao điều này cho phép tổng quát hoá qua nhiều clip

Vì discriminator học từ **toàn bộ tập dữ liệu mocap** (nhiều clip, không phải một clip cụ thể), nó học được khái niệm chung "thế nào là chuyển động tự nhiên/giống người" (ví dụ: nhịp điệu hợp lý của các khớp, phân bố trọng lượng hợp lý khi chuyển động) thay vì "khớp chính xác với đúng từng khung hình của một quỹ đạo". Điều này giải phóng policy khỏi việc phải bám sát tuyệt đối một reference trajectory — policy có thể tự do kết hợp/biến tấu các động tác miễn là "trông tự nhiên" theo đánh giá của discriminator, đồng thời vẫn hoàn thành task reward (ví dụ đi theo hướng mong muốn). Đây chính là lý do AMP tổng quát hoá tốt hơn DeepMimic thuần khi mở rộng sang tập dữ liệu chuyển động đa dạng.

---

## 5. Đường phát triển PHC → OmniH2O → ASAP → SONIC

Đây là chuỗi tiến hoá ý tưởng từ "bắt chước một động tác" tới "một policy tổng quát theo dõi bất kỳ chuyển động nào". Vì PHC và OmniH2O **chưa có citation kiểm chứng trong repo này**, phần dưới chỉ mô tả **xu hướng chung**, không đi sâu chi tiết toán học của từng bài (tránh bịa thông tin chưa xác minh) — chi tiết đầy đủ của PHC/OmniH2O/H2O nên tìm ở `../01-whole-body-control/` mục Motion Tracking-based Control.

**Xu hướng chung, theo 4 giai đoạn:**

1. **Bắt chước 1 clip (DeepMimic, 2018):** policy được huấn luyện để tái tạo chính xác một động tác cụ thể, thường phải huấn luyện lại từ đầu (hoặc fine-tune nặng) cho mỗi clip mới.
2. **Tổng quát hoá phong cách qua nhiều clip (AMP, 2021):** dùng discriminator để học "tự nhiên nói chung" thay vì bám khung hình — một bước tiến tới việc dùng nhiều dữ liệu hơn, nhưng vẫn thường giới hạn ở một tập kỹ năng/phong cách nhất định.
3. **Theo dõi tổng quát nhiều clip, quy mô lớn hơn (giai đoạn PHC/OmniH2O, cần xác minh thêm chi tiết):** hướng phát triển tiếp theo được cộng đồng humanoid WBC theo đuổi là huấn luyện **một policy duy nhất** có khả năng theo dõi (track) **bất kỳ clip nào trong một tập dữ liệu mocap rất lớn**, thay vì một policy cho một clip/phong cách. Đây là bước chuyển từ "character animation trong simulation thuần" (DeepMimic/AMP ban đầu nhắm tới nhân vật hoạt hình) sang "điều khiển humanoid robot thật" với các ràng buộc động lực học và giới hạn actuator thực tế.
4. **Sim-to-real cho kỹ năng agile (ASAP, RSS 2025) → Scale 3 trục (SONIC, 2025):**
   - **ASAP** ([GitHub LeCAR-Lab/ASAP](https://github.com/LeCAR-Lab/ASAP), arXiv:2502.01143) giải quyết trực diện khoảng cách sim-to-real (*"aligning simulation and real-world physics"*) cho các kỹ năng agile (nhảy, xoay người) — xem chi tiết cơ chế ở mục 6.
   - **SONIC** ([arXiv:2511.07820](https://arxiv.org/abs/2511.07820)) đẩy hướng "theo dõi tổng quát nhiều clip" lên quy mô rất lớn — mở rộng đồng thời theo **3 trục: model (kích thước mạng, từ 1.2M tới 42M tham số), data (khối lượng dữ liệu mocap, hơn 100 triệu frame từ khoảng 700 giờ mocap), và compute (khoảng 21.000 GPU-hours huấn luyện)**. Kết quả cho thấy hiệu năng cải thiện đều đặn khi tăng compute và độ đa dạng dữ liệu, và policy học được tổng quát hoá sang cả những chuyển động chưa từng thấy trong huấn luyện — củng cố ý tưởng "motion tracking ở quy mô lớn" như một nền tảng thực tế cho điều khiển humanoid tổng quát (tương tự tinh thần "scaling law" trong huấn luyện mô hình ngôn ngữ lớn).

Tóm lại: mạch phát triển là **từ chính xác-1-clip (DeepMimic) → tự nhiên-đa-clip nhờ discriminator (AMP) → tổng-quát-nhiều-clip ở quy mô lớn hơn cho robot thật (PHC/OmniH2O — cần đọc thêm) → sửa sai lệch sim-thật cho kỹ năng khó (ASAP) → scale cực lớn 3 trục thành generalist controller (SONIC)**.

---

## 6. Sim-to-real cho humanoid — cơ chế chi tiết

Policy huấn luyện trong mô phỏng (simulation) luôn có một khoảng cách với robot thật gọi là **"sim-to-real gap"** — do mô phỏng không mô hình hoàn hảo ma sát, độ trễ, độ đàn hồi dây/động cơ, sai số cảm biến... Hai kỹ thuật chính để thu hẹp khoảng cách này:

### 6.1 Domain Randomization (DR)

Ý tưởng (Tobin et al. 2017, [arXiv:1703.06907](https://arxiv.org/abs/1703.06907)): thay vì cố mô phỏng vật lý *chính xác tuyệt đối*, chủ động **ngẫu nhiên hoá** các tham số vật lý và cảm biến **trong lúc huấn luyện**, ví dụ:
- Hệ số ma sát giữa chân robot và mặt sàn (thay đổi ngẫu nhiên mỗi episode hoặc mỗi môi trường song song).
- Khối lượng và vị trí trọng tâm của từng khớp/link (mô phỏng sai số chế tạo, tải trọng thêm).
- Độ trễ (latency) của tín hiệu cảm biến (sensor) và tín hiệu điều khiển tới động cơ (actuator) — vì robot thật luôn có độ trễ truyền tín hiệu/xử lý mà mô phỏng lý tưởng không có.
- Nhiễu (noise) thêm vào quan sát (giả lập sai số cảm biến thật).

Khi policy được huấn luyện để hoạt động tốt trên **một dải rộng các biến thể vật lý ngẫu nhiên** như vậy, nó buộc phải học một chiến lược **robust** — không phụ thuộc quá chặt vào một bộ tham số vật lý cụ thể — nên khi chuyển sang robot thật (vốn nằm đâu đó trong dải biến thể đã huấn luyện), policy vẫn hoạt động tốt dù chưa từng thấy chính xác tham số vật lý thật.

### 6.2 Residual learning (bù sai lệch động lực học)

DR giúp policy robust hơn nhưng không "sửa" được sai lệch mô hình một cách chủ động — nó chỉ chấp nhận sự bất định. **Residual/delta action learning** là cách tiếp cận bổ sung: học một mô hình phụ (**delta/residual action model**) chuyên biệt để **hiệu chỉnh hành động** mà policy chính đưa ra, sao cho khi hành động đã hiệu chỉnh này chạy trên robot thật, kết quả trạng thái tiếp theo khớp với kết quả mà mô phỏng "mong đợi". Nói cách khác: thay vì cố làm mô phỏng giống thật hơn (sửa simulator), cách này học một lớp bù trực tiếp trên hành động để **thu hẹp khoảng cách động lực học giữa sim và thật ở đầu ra**.

### 6.3 Cách ASAP fine-tune ngắn trên robot thật

ASAP dùng chính xác chiến lược residual learning theo 2 giai đoạn (đã xác minh qua nguồn chính thức):
1. **Giai đoạn 1 (huấn luyện trong sim):** huấn luyện policy motion-tracking như thông thường (PPO + motion-tracking reward, trên dữ liệu mocap đã retarget) hoàn toàn trong mô phỏng.
2. **Giai đoạn 2 (thu thập dữ liệu thật + học residual):** triển khai policy đã huấn luyện lên **robot thật** (Unitree G1 trong paper gốc), thu thập dữ liệu về cách trạng thái thật diễn tiến so với trạng thái mà mô phỏng dự đoán cho cùng hành động. Từ chênh lệch này, huấn luyện một **mô hình delta action** — mô hình học cách điều chỉnh hành động đầu ra sao cho khi áp dụng lên robot thật, kết quả khớp gần hơn với những gì mô phỏng kỳ vọng, tức là **giảm thiểu chênh lệch giữa trạng thái thật và trạng thái mô phỏng**. Mô hình delta action này sau đó được đưa ngược vào huấn luyện trong sim (fine-tune lại policy với mô phỏng đã "được hiệu chỉnh" bởi residual model) trước khi triển khai lại lên robot thật.

Theo kết quả công bố, cách tiếp cận này giúp giảm sai số tracking trên robot thật tới khoảng 52.7% so với không dùng hiệu chỉnh, cho phép Unitree G1 thực hiện các kỹ năng agile (nhảy, xoay người) khó mà chỉ dùng DR thuần không đạt được.

---

## 7. Behavior cloning hiện đại: Diffusion Policy và ACT

Nhánh này **không dùng RL** — không có reward, không có mô phỏng vật lý lặp lại hàng nghìn lần. Thay vào đó, học trực tiếp từ **dữ liệu demonstration** (thường là teleoperation — con người điều khiển robot thật hoặc thiết bị teleop, ghi lại cặp quan sát–hành động), theo kiểu **supervised learning**: học một hàm ánh xạ từ quan sát sang hành động sao cho khớp với hành động con người đã thực hiện.

### 7.1 Diffusion Policy

Diffusion Policy (Chi et al. 2023, [arXiv:2303.04137](https://arxiv.org/abs/2303.04137)) biểu diễn policy điều khiển (visuomotor policy) như một **quá trình khử nhiễu (denoising diffusion) có điều kiện**, dựa trên Denoising Diffusion Probabilistic Models (DDPM):
- Thay vì mạng nơ-ron trực tiếp hồi quy (regress) ra một hành động cụ thể từ quan sát, Diffusion Policy học cách **biến một chuỗi nhiễu ngẫu nhiên (Gaussian noise) thành một chuỗi hành động hợp lý** (action chunk) qua nhiều bước khử nhiễu tuần tự, với mỗi bước khử nhiễu được **điều kiện hoá (conditioned)** trên quan sát hiện tại (hình ảnh camera, trạng thái robot...).
- Về bản chất, mạng học "hàm score" (gradient của log mật độ xác suất) của phân bố hành động có điều kiện theo quan sát, thay vì học trực tiếp một hành động duy nhất.
- Ưu điểm lớn: có thể biểu diễn **phân bố hành động đa phương thức (multi-modal)** — ví dụ khi có nhiều cách hợp lý để hoàn thành một thao tác (đẩy vật sang trái hoặc phải đều được), các phương pháp hồi quy trực tiếp (regression) thường bị "trung bình hoá" hai lựa chọn thành một hành động vô nghĩa ở giữa, còn diffusion model có thể học đúng cả hai "đỉnh" phân bố. Đánh đổi: suy luận (inference) cần nhiều bước khử nhiễu tuần tự, nên tốc độ suy luận tiêu chuẩn thường chỉ đạt khoảng 1–2 Hz trên phần cứng robot thông thường (có các biến thể tăng tốc sau này).

### 7.2 ACT (Action Chunking with Transformers) / ALOHA

ACT (Zhao et al. 2023, [arXiv:2304.13705](https://arxiv.org/abs/2304.13705)) giải quyết vấn đề **compounding error** kinh điển của behavior cloning: khi policy dự đoán từng hành động một cách độc lập tại mỗi bước thời gian, một sai số nhỏ ở bước này sẽ đẩy quan sát tiếp theo hơi lệch khỏi phân bố dữ liệu huấn luyện, khiến sai số bước sau lớn hơn, tích luỹ dần theo cấp số nhân qua nhiều bước ("compounding").

Cơ chế **action chunking**: thay vì dự đoán 1 hành động tại mỗi bước, ACT dự đoán một **chuỗi hành động dài k bước liên tiếp** (action chunk) cho mỗi lần suy luận. Vì policy chỉ cần "quyết định lại" sau mỗi k bước thay vì mỗi bước, **horizon hiệu dụng** của bài toán giảm xuống k lần — giảm đáng kể số lần sai số có cơ hội tích luỹ. ACT còn dùng thêm **temporal ensembling** — tại mỗi bước thời gian thực tế, kết hợp (trung bình có trọng số) các dự đoán từ nhiều action chunk chồng lấn (được sinh ra ở các thời điểm suy luận khác nhau) để làm mượt chuyển động, giảm giật cục. Về kiến trúc, ACT huấn luyện policy dưới dạng một **conditional VAE (variational autoencoder)** để mô hình hoá tốt sự đa dạng/nhiễu trong cách con người thực hiện demonstration (cùng một thao tác nhưng con người không lặp lại y hệt mỗi lần).

### 7.3 Khi nào dùng nhánh này so với RL/motion-tracking

- **Dùng RL/motion-tracking (mục 2–6)** khi: có dữ liệu mocap người (không cần chạy trên robot thật), muốn huấn luyện trong mô phỏng song song quy mô lớn, và/hoặc mục tiêu là các kỹ năng toàn thân (whole-body) như đi, nhảy, giữ thăng bằng — nơi mô phỏng vật lý đủ tin cậy và tốc độ huấn luyện song song là lợi thế lớn.
- **Dùng behavior cloning hiện đại (Diffusion Policy/ACT)** khi: đã có dữ liệu **teleoperation thật trên robot thật** (ví dụ qua hệ thống như Open-TeleVision — Cheng et al. 2024, [arXiv:2407.01512](https://arxiv.org/abs/2407.01512), xem `../08-real-robot-deployment/`), đặc biệt cho các thao tác tay khéo léo (dexterous manipulation) mà mô phỏng vật lý khó mô hình chính xác (tiếp xúc, ma sát vật thể nhỏ, biến dạng...) — nơi học trực tiếp từ dữ liệu thật đáng tin cậy hơn học qua simulation.
- Hai nhánh **không loại trừ nhau**: nhiều hệ thống humanoid thực tế dùng RL/motion-tracking cho whole-body locomotion (chân, thân) và behavior cloning cho tay/thao tác khéo léo (manipulation), kết hợp trong cùng một hệ thống điều khiển.

---

## Bảng thuật ngữ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| **Policy (π)** | Hàm ánh xạ từ quan sát (state) sang hành động (action); là "bộ não điều khiển" cần huấn luyện. |
| **Reward** | Tín hiệu số đo "tốt/xấu" của một hành động/trạng thái, dùng để huấn luyện policy trong RL. |
| **Trajectory (τ)** | Chuỗi (state, action, reward) theo thời gian mà agent trải qua trong một episode. |
| **On-policy** | Kiểu thuật toán RL chỉ dùng được dữ liệu thu thập bởi chính policy hiện tại (ví dụ PPO). |
| **Off-policy** | Kiểu thuật toán RL có thể tái sử dụng dữ liệu cũ từ replay buffer (ví dụ SAC). |
| **Clipped surrogate objective** | Cơ chế của PPO giới hạn mức thay đổi policy mỗi bước cập nhật để tránh mất ổn định. |
| **Entropy regularization** | Cộng thêm entropy của policy vào mục tiêu tối ưu (như trong SAC) để khuyến khích khám phá. |
| **Reference motion / Motion-tracking reward** | Dữ liệu mocap tham chiếu dùng làm "đáp án mẫu"; reward = hàm khoảng cách (thường dạng exp(-distance)) giữa pose robot và pose tham chiếu mỗi frame. |
| **Dense reward** | Tín hiệu reward xuất hiện dày đặc (mỗi bước/frame), đối lập với sparse reward (chỉ ở cuối/khi đạt mục tiêu). |
| **Reference State Initialization (RSI)** | Kỹ thuật của DeepMimic: khởi tạo episode từ một frame ngẫu nhiên trong clip mocap thay vì luôn từ đầu clip. |
| **Early Termination (ET)** | Kết thúc episode sớm khi phát hiện thất bại (ví dụ ngã), tránh lãng phí dữ liệu huấn luyện vô ích. |
| **Discriminator (trong AMP)** | Mạng phân loại "chuyển động này thật hay do policy sinh ra", huấn luyện kiểu adversarial (GAN) để tạo reward phong cách (style reward). |
| **Style reward** | Reward sinh ra từ điểm số discriminator, đo mức độ "tự nhiên/giống người" của chuyển động, không cần khớp chính xác từng khung hình. |
| **Sim-to-real gap** | Khoảng cách giữa hành vi của policy trong mô phỏng và trên robot thật, do sai lệch mô hình vật lý. |
| **Domain Randomization (DR)** | Ngẫu nhiên hoá tham số vật lý/cảm biến (ma sát, khối lượng, độ trễ...) khi huấn luyện trong sim để policy robust hơn với sự bất định thật. |
| **Residual/Delta action learning** | Học một mô hình bù (hiệu chỉnh) hành động để giảm chênh lệch động lực học giữa sim và robot thật, dùng trong ASAP. |
| **Behavior Cloning (BC)** | Học policy trực tiếp từ dữ liệu demonstration bằng supervised learning, không dùng reward/RL. |
| **Compounding error** | Sai số nhỏ tích luỹ theo cấp số nhân qua nhiều bước dự đoán liên tiếp trong behavior cloning. |
| **Action chunking** | Dự đoán một chuỗi hành động dài (thay vì từng bước) để giảm horizon hiệu dụng và giảm compounding error (dùng trong ACT). |
| **Denoising diffusion process** | Quá trình sinh dữ liệu (ở đây là hành động) bằng cách khử nhiễu dần từ nhiễu ngẫu nhiên thành mẫu hợp lý, có điều kiện theo quan sát (dùng trong Diffusion Policy). |
| **Temporal ensembling** | Kỹ thuật của ACT: trung bình có trọng số các dự đoán hành động từ nhiều action chunk chồng lấn để làm mượt chuyển động. |

---

## Nguồn tham khảo đã xác minh (WebSearch, tháng 9/2026)

- DeepMimic — RSI, Early Termination: xác nhận qua [arXiv:1804.02717](https://arxiv.org/pdf/1804.02717) và các tóm tắt liên quan (emergentmind, alphaXiv).
- AMP — cơ chế discriminator và công thức reward: xác nhận qua [arXiv:2104.02180](https://arxiv.org/pdf/2104.02180) và rofunc docs.
- ASAP — cơ chế 2 giai đoạn + delta/residual action model, kết quả giảm 52.7% sai số: xác nhận qua [GitHub LeCAR-Lab/ASAP](https://github.com/LeCAR-Lab/ASAP) và arXiv:2502.01143.
- SONIC — scale 3 trục (model 1.2M→42M tham số, data 100M+ frame/700 giờ mocap, compute ~21k GPU-hours): xác nhận qua [arXiv:2511.07820](https://arxiv.org/abs/2511.07820).
- Diffusion Policy — DDPM, action chunk, đa phương thức: xác nhận qua [arXiv:2303.04137](https://arxiv.org/abs/2303.04137).
- ACT/ALOHA — action chunking giảm horizon k-lần, temporal ensembling, conditional VAE: xác nhận qua [arXiv:2304.13705](https://arxiv.org/pdf/2304.13705) và emergentmind.
- PHC/OmniH2O: **chưa xác minh chi tiết cơ chế trong lần tra cứu này** — phần mục 5 chỉ mô tả xu hướng chung, không khẳng định chi tiết thuật toán cụ thể. Nên bổ sung khi có citation đầy đủ trong repo.
