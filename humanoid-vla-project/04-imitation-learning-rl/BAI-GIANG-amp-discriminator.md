# Bài giảng: AMP — Cơ chế Discriminator (Adversarial Motion Priors)

*(Thuộc mảng: Imitation Learning & Reinforcement Learning)*

## 🎯 Mục tiêu bài học

- Giải thích được vấn đề DeepMimic-style reward gặp phải khi mở rộng sang nhiều clip mocap cùng lúc.
- Định nghĩa chính xác cơ chế discriminator kiểu GAN của AMP, và công thức style reward r^style(s_t).
- Giải thích được vòng lặp adversarial giữa policy và discriminator, và điều kiện hội tụ của nó.
- Tính tay được ví dụ số cụ thể: từ đầu ra discriminator ra style reward, qua công thức mượt hoá.
- Giải thích được vì sao discriminator giúp tổng quát hoá qua nhiều clip tốt hơn khoảng-cách-pose trực tiếp.
- Nêu được ít nhất 2 cải tiến 2025 khắc phục nhược điểm mode collapse/instability của AMP gốc.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Bài giảng trước (DeepMimic — RSI/ET) học cách bắt chước **một clip mocap cụ thể**. AMP là bước tiến hoá tiếp theo trong chuỗi PHC → OmniH2O → ASAP → SONIC (xem bài giảng riêng cho chuỗi này): thay vì đo "giống clip X đến mức nào ở từng khung hình", AMP hỏi một câu khác — "chuyển động này có *trông tự nhiên/giống người* không, bất kể có khớp chính xác với clip cụ thể nào hay không?". Đây là ý tưởng lõi (discriminator-based reward) mà rất nhiều công trình WBC hiện đại kế thừa, nên hiểu đúng cơ chế GAN đứng sau nó là bắt buộc trước khi đọc các bài giảng về PHC/OmniH2O/ASAP/SONIC.

## 🧠 Trực giác

### Góc nhìn 1: Giám khảo chấm "có giống người thật không" thay vì so từng động tác với 1 video mẫu

DeepMimic giống việc chấm điểm một vũ công bằng cách so khớp **từng giây, từng động tác** với đúng một video biểu diễn mẫu — sai lệch dù rất nhỏ ở một khung hình vẫn bị trừ điểm. AMP giống việc mời một giám khảo (discriminator) đã xem **hàng trăm video vũ công thật khác nhau** (không phải chỉ 1 video), và giám khảo chỉ cần trả lời một câu: "màn trình diễn này có giống người thật đang nhảy không, hay là một robot đang cố bắt chước"? Giám khảo không quan tâm màn trình diễn có khớp chính xác với bất kỳ video mẫu cụ thể nào — miễn nó "trông tự nhiên" theo kinh nghiệm xem hàng trăm video thật.

*Đúng ở đâu:* nắm đúng sự khác biệt cốt lõi — từ "so khớp 1 mẫu cụ thể" sang "đánh giá tự nhiên nói chung dựa trên nhiều mẫu".
*Giới hạn:* phép loại suy không phản ánh đúng rằng giám khảo (discriminator) **cũng đang được huấn luyện đồng thời**, không phải một chuyên gia cố định — ban đầu giám khảo còn kém, dễ bị đánh lừa bởi chuyển động vụng về; nó chỉ trở nên tinh vi dần theo thời gian huấn luyện, cùng lúc với việc vũ công (policy) cải thiện — đây chính là bản chất adversarial (cả 2 bên cùng tiến bộ), khác một giám khảo con người có sẵn kinh nghiệm cố định từ đầu.

### Góc nhìn 2: Trò chơi mèo vờn chuột giữa thợ làm tiền giả và cảnh sát kiểm định (GAN kinh điển)

AMP vay mượn trực tiếp cấu trúc GAN (Generative Adversarial Network): hãy tưởng tượng một thợ làm tiền giả (policy, đóng vai "generator" — ở đây sinh ra *chuyển động* thay vì tiền) cố làm ra tiền giả giống hệt tiền thật, và một cảnh sát kiểm định (discriminator) cố phân biệt tiền giả với tiền thật. Ban đầu cảnh sát dễ dàng phát hiện tiền giả (thợ làm còn vụng), nhưng thợ làm tiền học được từ mỗi lần bị phát hiện và cải thiện kỹ thuật; đến lượt cảnh sát cũng phải học tinh vi hơn để tiếp tục phân biệt được. Vòng lặp dừng lại (lý tưởng) khi tiền giả gần như không thể phân biệt với tiền thật.

*Đúng ở đâu:* nắm đúng cấu trúc 2 người chơi cạnh tranh, cùng cải thiện qua thời gian, đây chính xác là công thức toán GAN gốc mà AMP áp dụng.
*Giới hạn:* phép loại suy không cảnh báo đúng về **rủi ro thất bại nổi tiếng của GAN** — nếu cảnh sát (discriminator) học quá nhanh và trở nên "hoàn hảo" quá sớm (perfect discriminator), nó phân biệt được 100% tiền giả ngay từ đầu, khiến gradient phản hồi cho thợ làm tiền gần như bằng 0 (không còn tín hiệu nào để cải thiện) — đây là vấn đề **vanishing gradient/mode collapse** kinh điển của GAN, và AMP cũng không miễn nhiễm với nó (xem mục Cập nhật hiện đại về APEX khắc phục vấn đề này).

## 📐 Định nghĩa chính xác

AMP (Peng, Ma, Abbeel, Levine, Kanazawa, 2021, [arXiv:2104.02180](https://arxiv.org/abs/2104.02180)) thay thế reward "khoảng cách pose" bằng một **discriminator** D_φ tham số hoá bởi φ, huấn luyện đồng thời với policy π_θ theo tinh thần GAN:

**Input của discriminator:** một cặp trạng thái liên tiếp (state transition) s_t → s_{t+1} (biểu diễn dưới dạng đặc trưng "AMP observation" — thường gồm góc khớp, vận tốc khớp, vị trí/tốc độ root ở 2 thời điểm liên tiếp).

**Mục tiêu huấn luyện discriminator** (dạng least-squares GAN, ổn định hơn cross-entropy GAN gốc):
```
min_φ  E_{(s,s')~data thật} [ (D_φ(s,s') − 1)² ]
      + E_{(s,s')~policy}   [ (D_φ(s,s') + 1)² ]
      + w_gp · gradient_penalty(φ)
```
trong đó nhãn "thật" (dữ liệu mocap) được gán mục tiêu +1, nhãn "giả" (policy sinh ra) được gán mục tiêu −1, và một số hạng gradient penalty được thêm vào để ổn định huấn luyện (tránh discriminator học quá "sắc" — vấn đề nêu ở Góc nhìn 2).

**Style reward** (chuyển đổi đầu ra discriminator thành tín hiệu reward mượt, không âm):
```
r^style(s_t, s_{t+1}) = max( 0, 1 − ¼·(D_φ(s_t,s_{t+1}) − 1)² )
```
Công thức này ánh xạ D_φ = 1 (discriminator "tin" đây là dữ liệu thật) sang r^style = 1 (tối đa), và giảm dần khi D_φ lệch xa 1.

**Reward tổng của policy:**
```
r_t = w_G · r^task(t) + w_S · r^style(t)
```
Policy được huấn luyện bằng PPO để tối đa hoá tổng reward này — đồng thời discriminator được cập nhật xen kẽ để phân biệt tốt hơn. Đây là một quá trình **minimax hai cấp độ**: discriminator cố tối đa hoá khả năng phân biệt, policy cố tối đa hoá r^style (tức tối thiểu hoá khả năng bị discriminator phát hiện).

## ⚙️ Cơ chế hoạt động — từng bước

```
┌──────────────────────────────────────────────────────────────────┐
│  Dataset mocap đa dạng D_real = {nhiều clip chuyển động người thật}│
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│  VÒNG LẶP HUẤN LUYỆN (lặp lại mỗi iteration)                       │
│                                                                      │
│  1. Rollout: chạy π_θ trong sim, thu thập transition (s_t,s_{t+1})  │
│     do policy sinh ra → D_policy                                    │
│                                                                      │
│  2. Lấy mẫu batch transition thật từ D_real (mocap)                 │
│                                                                      │
│  3. Cập nhật discriminator D_φ:                                     │
│     - Cho D_real → mục tiêu +1 (thật)                                │
│     - Cho D_policy → mục tiêu −1 (giả)                               │
│     - Gradient penalty để ổn định                                   │
│                                                                      │
│  4. Tính r^style(t) = max(0, 1 − ¼(D_φ(s_t,s_{t+1})−1)²) cho mỗi    │
│     transition trong D_policy (dùng D_φ VỪA cập nhật)               │
│                                                                      │
│  5. r_t = w_G·r^task(t) + w_S·r^style(t)                             │
│                                                                      │
│  6. Cập nhật policy π_θ bằng PPO, tối đa hoá E[Σ r_t]                │
│                                                                      │
│  7. Quay lại bước 1 — policy giờ "giỏi hơn" → D_policy giống D_real  │
│     hơn → discriminator (bước 3) khó phân biệt hơn ở vòng sau        │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
        Hội tụ lý tưởng: D_φ(s,s') ≈ 0 cho mọi transition
        (discriminator không còn phân biệt được thật/giả)
```

Điểm mấu chốt cơ chế: bước 3 và bước 6 **xen kẽ, không đồng thời tối ưu chung một hàm loss** — đây là đặc trưng huấn luyện adversarial hai giai đoạn xen kẽ (alternating optimization), khác hẳn với việc tối ưu một hàm mục tiêu duy nhất, và chính đặc điểm này là nguồn gốc của các vấn đề bất ổn định huấn luyện GAN kinh điển.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số liệu tự chọn để dễ hình dung.)*

Giả sử tại một bước huấn luyện, discriminator đưa ra 3 giá trị D_φ khác nhau cho 3 transition khác nhau do policy hiện tại sinh ra:

**Transition A** (chuyển động khá tự nhiên): D_φ(s_A, s_A') = 0.8
```
r^style_A = max(0, 1 − ¼(0.8−1)²) = max(0, 1 − ¼×0.04) = max(0, 1−0.01) = 0.99
```

**Transition B** (chuyển động hơi giật cục): D_φ(s_B, s_B') = −0.2
```
r^style_B = max(0, 1 − ¼(−0.2−1)²) = max(0, 1 − ¼×1.44) = max(0, 1−0.36) = 0.64
```

**Transition C** (chuyển động rất kỳ quặc, dễ bị phát hiện là "giả"): D_φ(s_C, s_C') = −1.5
```
r^style_C = max(0, 1 − ¼(−1.5−1)²) = max(0, 1 − ¼×6.25) = max(0, 1−1.5625) = max(0, −0.5625) = 0
```

**Diễn giải:** transition A (gần D_φ=1, "giống thật") nhận style reward gần tối đa 0.99; transition B (D_φ âm nhẹ, hơi giống "giả") vẫn nhận reward dương đáng kể 0.64 — không bị phạt quá nặng; transition C (D_φ rất âm, rõ ràng "giả") bị **cắt về đúng 0** nhờ toán tử max(0, ·) — đây là lý do công thức dùng max(0,·) thay vì để giá trị âm: tránh việc policy nhận reward âm quá lớn cho một transition đơn lẻ tệ, giữ style reward luôn trong khoảng [0, 1] để dễ kết hợp tuyến tính với task reward (cũng thường chuẩn hoá trong [0,1]).

Nếu áp dụng w_G = 0.5, w_S = 0.5 và giả định r^task tại 3 bước này lần lượt là 0.9, 0.7, 0.3, tổng reward:
```
r_A = 0.5×0.9 + 0.5×0.99 = 0.45 + 0.495 = 0.945
r_B = 0.5×0.7 + 0.5×0.64 = 0.35 + 0.32  = 0.670
r_C = 0.5×0.3 + 0.5×0    = 0.15 + 0     = 0.150
```
Transition C bị phạt nặng ở cả hai thành phần (task thấp lẫn style = 0), là tín hiệu mạnh nhất để policy tránh loại chuyển động này trong tương lai.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | DeepMimic (khoảng cách pose) | AMP (discriminator) |
|---|---|---|
| Nguồn tín hiệu "tự nhiên" | 1 clip cụ thể, so khớp chính xác từng frame | Toàn bộ tập dữ liệu mocap, học khái niệm chung |
| Cần đồng bộ thời gian (phase alignment)? | Bắt buộc | Không bắt buộc |
| Khả năng tổng quát/kết hợp nhiều clip | Kém, dễ overfit | Tốt hơn — có thể pha trộn phong cách |
| Độ phức tạp huấn luyện | Đơn giản (chỉ 1 policy, PPO thuần) | Phức tạp hơn (2 mạng, huấn luyện adversarial, dễ mất ổn định) |
| Rủi ro đặc trưng | Overfit 1 trajectory | Mode collapse, perfect-discriminator collapse (xem Cập nhật hiện đại) |
| Ví dụ paper | DeepMimic (2018) | AMP (2021), ASE (2022), CALM (2023) |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "AMP không cần dữ liệu mocap tham chiếu tại mỗi thời điểm, nên hoàn toàn tự do."** Sai — AMP vẫn cần một tập dữ liệu mocap D_real làm nguồn "thật" để discriminator học phân biệt; điều khác biệt so với DeepMimic không phải là "không cần mocap" mà là **không cần đồng bộ chính xác theo thời gian** với MỘT clip cụ thể — discriminator có thể lấy mẫu bất kỳ transition thật nào từ toàn bộ tập dữ liệu, không cần khớp pha thời gian với episode hiện tại của policy.

2. **Hiểu nhầm: "Discriminator càng chính xác (phân biệt thật/giả càng tốt) thì AMP càng huấn luyện tốt."** Sai — đây chính là vấn đề "perfect discriminator" nêu ở Góc nhìn 2: nếu discriminator quá mạnh, nó phân biệt hoàn hảo ngay từ đầu, khiến D_φ(policy) luôn rất âm và gradient phản hồi cho policy gần như bão hoà/biến mất (không còn phân biệt được "hơi tệ" và "rất tệ" — ví dụ trong bảng tính tay, cả transition B và C nếu discriminator quá mạnh có thể đều bị đẩy về style reward ≈ 0, mất đi tín hiệu gradient hữu ích để phân biệt mức độ tệ). Cần cân bằng độ mạnh giữa 2 mạng (thường qua gradient penalty, learning rate riêng, hoặc giới hạn update frequency của discriminator).

3. **Hiểu nhầm: "Style reward thay thế hoàn toàn task reward, AMP chỉ cần bắt chước tự nhiên là đủ."** Sai — công thức reward tổng luôn kết hợp CẢ w_G·r^task LẪN w_S·r^style; nếu chỉ dùng r^style, policy có thể học ra chuyển động "trông tự nhiên" nhưng hoàn toàn không phục vụ mục tiêu nhiệm vụ nào (ví dụ đứng yên lắc lư "tự nhiên" thay vì đi tới đích) — task reward là thành phần bắt buộc để định hướng hành vi có mục đích.

## 🏗️ Ví dụ minh hoạ trong dự án này

Discriminator-based reward là ý tưởng lõi được PHC, OmniH2O và các paper trung gian trong chuỗi phát triển hướng tới SONIC kế thừa (xem bài giảng riêng "Đường phát triển PHC → OmniH2O → ASAP → SONIC") — thay vì huấn luyện một policy cho một clip, các hệ thống này cần một tín hiệu "tự nhiên nói chung" có thể áp dụng cho hàng nghìn/hàng trăm nghìn clip mocap khác nhau, đúng chính xác vấn đề mà AMP giải quyết. Trong `02-motion-retargeting/`, dữ liệu robot đã retarget từ nhiều clip mocap khác nhau (không chỉ 1 clip) chính là nguồn D_real lý tưởng cho một discriminator kiểu AMP nếu áp dụng vào pipeline huấn luyện của dự án — khác với motion-tracking reward thuần (bài giảng trước) vốn chỉ cần 1 trajectory tham chiếu duy nhất mỗi lúc.

## 🔥 Cập nhật hiện đại / SOTA gần đây

- **APEX** (Sood et al., 15/5/2025) khắc phục trực diện vấn đề "mode collapse và giới hạn đa dạng" của AMP gốc — đạt được locomotion đa dạng, hiệu năng cao chỉ trong khoảng **~1.000 iteration**, so với AMP nguyên bản cần khoảng **~50.000 iteration** để đạt kết quả tương đương — một cải thiện tốc độ hội tụ khoảng 50 lần, cho thấy vấn đề bất ổn định huấn luyện adversarial (nêu ở mục Sai lầm) là có thật và đã được giải quyết một phần đáng kể chỉ trong vài năm.
- **AMP+SAC** (Lessa et al., 29/9/2025) thay PPO (mặc định trong AMP gốc) bằng SAC để huấn luyện policy trong khung AMP, cho kết quả duy trì imitation reward cao hơn và khả năng thích nghi địa hình (terrain adaptation) mạnh mẽ hơn so với AMP+PPO truyền thống — một minh chứng thực nghiệm liên hệ trực tiếp với bài giảng "PPO vs SAC": lựa chọn thuật toán RL nền (PPO hay SAC) vẫn là một trục thiết kế mở, kể cả trong khung discriminator-based reward.
- **ASE** (Peng et al., 2022) và **CALM** (Tessler et al., 2023) là hai hướng mở rộng trực tiếp ý tưởng AMP: ASE học các "adversarial skill embedding" trên một manifold hình cầu để dùng cho điều khiển cấp cao hơn (downstream control), CALM học các "conditional adversarial latent model" — cả hai cho thấy discriminator-based reward không dừng lại ở việc tạo reward tự nhiên đơn thuần, mà còn được dùng làm nền tảng để học không gian kỹ năng (skill space) tái sử dụng được.
- Về mặt kiến trúc discriminator, một số công trình gần đây cải tiến bằng cách cho discriminator quan sát **cửa sổ thời gian dài hơn** (temporal context) thay vì chỉ 2 trạng thái liên tiếp — ví dụ dùng 5 bước thời gian (τ_t = (s_{t-3}, s_{t-2}, s_{t-1}, s_{t+1}) dạng AMP-state) thay vì chỉ (s_t, s_{t+1}), giúp discriminator đánh giá được tính nhất quán động học trên một đoạn dài hơn, không chỉ một cặp trạng thái tức thời.

## ❓ Câu hỏi tự kiểm tra

1. Vì sao AMP không cần biết chính xác "thời điểm t trong clip mocap tương ứng với thời điểm hiện tại của policy", trong khi DeepMimic bắt buộc phải biết?
<details><summary>Gợi ý đáp án</summary>Vì discriminator chỉ đánh giá "cặp trạng thái này có giống dữ liệu thật nói chung không", lấy mẫu batch dữ liệu thật ngẫu nhiên từ toàn bộ tập D_real chứ không cần đúng frame tương ứng theo thời gian — khác hẳn công thức khoảng cách pose của DeepMimic vốn cần so sánh trực tiếp với đúng frame t trong 1 clip cụ thể.</details>

2. Trong ví dụ tính tay, tại sao transition C (D_φ=−1.5) nhận r^style=0 thay vì một giá trị âm?
<details><summary>Gợi ý đáp án</summary>Vì công thức có toán tử max(0, ·) — mọi giá trị bên trong nhỏ hơn 0 đều bị cắt về 0, đảm bảo style reward luôn ∈[0,1], tránh việc một transition rất tệ tạo ra reward âm cực đoan làm mất ổn định tổng reward.</details>

3. Nếu discriminator được cập nhật QUÁ NHANH (nhiều bước gradient mỗi iteration) so với policy, điều gì có khả năng xảy ra?
<details><summary>Gợi ý đáp án</summary>Discriminator có thể trở nên "quá mạnh" quá sớm (vấn đề perfect discriminator nêu ở mục Sai lầm #2), khiến D_φ(policy) luôn rất âm cho mọi transition (kể cả những transition đã khá tự nhiên), làm bão hoà style reward về gần 0 cho hầu hết trường hợp — mất tín hiệu gradient hữu ích để policy phân biệt "khá tốt" và "rất tệ".</details>

4. APEX (2025) giảm số iteration cần thiết từ ~50.000 xuống ~1.000 so với AMP gốc — điều này nói lên điều gì về bản chất vấn đề mà AMP gốc gặp phải?
<details><summary>Gợi ý đáp án</summary>Cho thấy phần lớn chi phí huấn luyện của AMP gốc không nằm ở độ khó bản chất của bài toán bắt chước chuyển động, mà nằm ở sự KÉM HIỆU QUẢ trong cách huấn luyện adversarial (mode collapse, thiếu đa dạng buộc phải lặp lại nhiều để khám phá đủ) — một cải tiến thuật toán (không phải thêm dữ liệu/compute) có thể mang lại tăng tốc bậc số lượng lớn.</details>

## 📝 Bài tập thực hành

1. Vẽ đồ thị hàm r^style(D) = max(0, 1 − ¼(D−1)²) theo D trong khoảng D ∈ [−3, 3] (bằng tay hoặc bằng Python/matplotlib). Xác định khoảng giá trị D mà r^style > 0, và giải thích ý nghĩa vật lý của khoảng đó.
2. Tính lại ví dụ ở mục "Ví dụ tính tay" với D_φ = 1.5 (discriminator "tin tưởng quá mức" rằng đây là dữ liệu thật, vượt quá 1). Tính r^style và so sánh với trường hợp D_φ=0.8 — giải thích vì sao D_φ > 1 không tạo ra style reward cao hơn D_φ=1 (gợi ý: xem lại vai trò hàm bậc 2 (D−1)² trong công thức).

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

AMP thay thế reward "khoảng cách pose chính xác từng frame" của DeepMimic bằng một discriminator huấn luyện kiểu GAN — mạng này học phân biệt transition trạng thái do policy sinh ra với transition lấy từ toàn bộ tập dữ liệu mocap thật, và điểm số của nó (chuyển qua công thức mượt r^style=max(0,1−¼(D−1)²)) trở thành "style reward" khuyến khích policy tạo chuyển động tự nhiên nói chung thay vì bám cứng một trajectory cụ thể — nhờ vậy AMP tổng quát hoá tốt hơn qua nhiều clip, nhưng đánh đổi bằng sự phức tạp và rủi ro bất ổn định của huấn luyện adversarial (mode collapse, perfect-discriminator), những vấn đề mà các công trình 2025 như APEX và AMP+SAC đang tích cực khắc phục.
