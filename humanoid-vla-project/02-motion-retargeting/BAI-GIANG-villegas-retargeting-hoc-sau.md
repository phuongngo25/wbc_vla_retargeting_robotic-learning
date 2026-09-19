# Bài giảng: Retargeting học sâu/residual (Villegas et al. 2018)

*(Thuộc mảng: Motion Retargeting)*

## 🎯 Mục tiêu bài học

- Giải thích được ý tưởng cốt lõi của Neural Kinematic Networks (NKN, Villegas et al. 2018): mạng hồi quy + lớp Forward-Kinematics (FK layer) + huấn luyện không giám sát bằng cycle-consistency.
- Giải thích được vì sao "học không giám sát" là bắt buộc ở đây, không phải một lựa chọn tuỳ ý — vì sao không có "nhãn đúng" cho retargeting.
- Tính tay được một ví dụ số đơn giản cho thấy FK layer giúp mạng "không cần học lại hình học đã biết", và một ví dụ số minh hoạ cycle-consistency loss.
- Phân biệt được ba lớp phương pháp: retargeting hình học thuần (GMR/SOMA, đã học), residual/hiệu chỉnh học được, và sinh chuyển động end-to-end bằng mạng.
- Nêu được vì sao trong bối cảnh humanoid VLA hiện đại, hướng "học sâu" thường không thay thế IK hình học mà bổ sung dưới dạng residual hoặc mở rộng sang correspondence học được.
- Liệt kê được ít nhất 2 công trình 2020–2026 kế thừa/mở rộng trực tiếp tư tưởng NKN.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Hai bài trước (GMR, SOMA-retargeter) là hai công cụ retargeting **hình học thuần** — giải trực tiếp bài toán IK mỗi frame dựa trên công thức động học đã biết, không cần dữ liệu huấn luyện. Đây là "trường phái thứ ba" trong ba trường phái retargeting mà `NOI-DUNG-CHI-TIET.md` liệt kê: dùng **mạng nơ-ron để học retargeting**. Học bài này để hiểu: khi nào và tại sao người ta muốn thay một phần (hoặc toàn bộ) bài toán retargeting bằng một mạng học được, thay vì luôn giải IK hình học tường minh — và vì sao hướng này, dù xuất hiện từ 2018, tới nay (2025–2026) chủ yếu tồn tại dưới dạng **bổ sung/residual** cho IK hình học chứ chưa thay thế hoàn toàn nó trong các pipeline humanoid production (GMR/SOMA vẫn là lựa chọn chính của dự án).

## 🧠 Trực giác

### Góc nhìn 1: Học lái xe với một GPS luôn đúng đường, chỉ cần học cách rẽ vô-lăng

Hãy tưởng tượng dạy một người mới học lái xe. Nếu bắt họ vừa phải tự tính toán góc rẽ hợp lý để tới đúng điểm đến, vừa phải điều khiển vô-lăng chính xác, nhiệm vụ sẽ khó hơn nhiều so với việc cho họ một GPS luôn chỉ đúng đường (không cần học) và họ chỉ cần học kỹ năng "làm sao xoay vô-lăng để đi theo chỉ dẫn GPS đó". FK layer trong NKN đóng đúng vai trò GPS này: công thức động học thuận (từ góc khớp ra vị trí đầu mút) đã biết chính xác 100% bằng toán học, nên mạng không cần "học lại" quan hệ hình học đó — nó chỉ cần học phần khó thật sự: đoán góc khớp nào (bài toán IK ngược) sẽ tạo ra chuyển động hợp lý trên khung xương mới.

**Giới hạn của loại suy này:** GPS chỉ cho một tuyến đường, còn bài toán IK có **vô số nghiệm** góc khớp khác nhau có thể tạo cùng một vị trí đầu mút — mạng vẫn phải "chọn" trong không gian nghiệm này theo cách tạo ra chuyển động tự nhiên, một việc phức tạp hơn việc "đi theo một tuyến đường duy nhất" mà loại suy GPS ngụ ý.

### Góc nhìn 2: Dịch qua lại giữa hai ngôn ngữ mà không có từ điển song ngữ, chỉ có người bản xứ của mỗi bên

Nếu không có cặp câu (tiếng Anh, tiếng Việt) đã dịch sẵn làm nhãn huấn luyện, làm sao dạy một hệ thống dịch máy? Một cách: dịch câu A sang B, rồi dịch ngược lại từ B về A — nếu bản dịch ngược khớp lại với câu gốc A, có cơ sở tin rằng bản dịch trung gian đã giữ đúng ý nghĩa, dù không ai xác nhận trực tiếp bản dịch A→B có đúng hay không. Đây chính xác là **cycle-consistency**: retarget chuyển động A (khung xương nguồn) sang B (khung xương đích) rồi ngược lại B→A, so sánh với A ban đầu.

**Giới hạn của loại suy này:** một bản dịch có thể "khớp lại" khi dịch ngược (round-trip) nhưng vẫn sai nghĩa ở bước trung gian nếu cả hai lỗi dịch xuôi/ngược tình cờ triệt tiêu nhau (ví dụ dịch sai theo cách đối xứng); cycle-consistency là một tín hiệu huấn luyện hữu ích nhưng **không đảm bảo tuyệt đối** bước trung gian đúng — đây là hạn chế đã biết của mọi phương pháp cycle-consistency (bao gồm CycleGAN gốc trong thị giác máy tính), không riêng gì NKN.

## 📐 Định nghĩa chính xác

**Neural Kinematic Networks (NKN)** — Villegas, Yang, Ceylan, Lee, *"Neural Kinematic Networks for Unsupervised Motion Retargetting"* (CVPR 2018, [arXiv:1804.05653](https://arxiv.org/abs/1804.05653)) — là kiến trúc gồm ba thành phần chính:

1. **Mạng hồi quy (recurrent network)** nhận chuỗi chuyển động nguồn `θ_A(t)` (góc khớp trên khung xương A) và dự đoán chuỗi góc khớp trên khung xương đích `θ_B(t)`.
2. **Lớp Forward-Kinematics (FK layer)** được nhúng trực tiếp vào kiến trúc mạng: nhận `θ_B(t)` và chiều dài xương đã biết của khung B, tính ra vị trí toàn cục các đầu mút chi bằng đúng công thức FK toán học (không có tham số học được trong lớp này) — cho phép lan truyền ngược (backpropagation) gradient của sai số *vị trí không gian* trở lại thành gradient cập nhật *góc khớp dự đoán*.
3. **Mục tiêu huấn luyện dạng cycle-consistency, không giám sát:**

```text
θ_A(t) --[mạng, A→B]--> θ_B(t) --[mạng, B→A]--> θ_A'(t)

Loss_cycle = ‖θ_A(t) − θ_A'(t)‖²   (hoặc theo vị trí FK tương ứng)
```

Không cần cặp `(θ_A, θ_B_đúng)` làm nhãn — điều gần như không thể thu thập ở quy mô lớn (không tồn tại "nghiệm đúng duy nhất" cho retargeting để làm ground truth). Ngoài cycle-consistency, paper gốc còn dùng thêm mục tiêu đối kháng (adversarial, theo tinh thần GAN) để khuyến khích chuyển động sinh ra "trông giống thật" trên khung xương đích, và hoạt động **online** — thích ứng theo từng frame khi dữ liệu tới, không cần xử lý toàn bộ clip trước.

## ⚙️ Cơ chế hoạt động — từng bước

```text
┌────────────────────────────────────────────────────────────┐
│ INPUT: chuỗi góc khớp nguồn θ_A(t), khung xương A            │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ Mạng hồi quy (RNN/LSTM) A→B                                  │
│  học ánh xạ θ_A(t) ↦ θ_B(t) trên khung xương đích B          │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ FK layer (không tham số, công thức toán học cố định)         │
│  θ_B(t) ↦ vị trí toàn cục đầu mút chi trên khung B            │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
              ┌─────────────┴─────────────┐
              ▼                           ▼
   Mạng hồi quy B→A (dùng lại            Discriminator (adversarial)
   kiến trúc tương tự, hướng ngược)       đánh giá θ_B(t) "giống thật"?
              │                                     │
              ▼                                     │
   θ_A'(t) (bản dịch ngược)                          │
              │                                     │
              ▼                                     ▼
   So sánh θ_A(t) vs θ_A'(t)              Cập nhật qua adversarial loss
   → Loss_cycle                                     
              │                                     │
              └───────────────┬─────────────────────┘
                              ▼
                Backpropagation cập nhật cả hai mạng A→B, B→A
                (không cần nhãn θ_B đúng nào)
```

Điểm mấu chốt về *vì sao* kiến trúc này hoạt động được mà không cần nhãn: FK layer đảm bảo rằng sai số được lan truyền ngược **đã đi qua đúng hình học thật** của khung xương đích — nếu mạng dự đoán góc khớp sai, sai số thể hiện rõ ở vị trí đầu mút chi tính qua FK (một khớp sai một chút ở vai có thể khiến cả cánh tay lệch xa), tạo tín hiệu gradient có ý nghĩa vật lý rõ ràng thay vì chỉ so sánh trực tiếp các con số góc trừu tượng.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số tự chọn để dễ hình dung — không phải số liệu thực nghiệm từ paper Villegas)*

### Phần A — FK layer giúp gradient "biết" hậu quả không gian của một sai số góc nhỏ

Xét chain 2 khớp phẳng (2D) như ở các bài trước: vai→khuỷu (`L1 = 0.30 m`)→cổ tay (`L2 = 0.25 m`). Giả sử mạng dự đoán góc vai `α = 90°`, góc khuỷu `β = 45°`, nhưng giá trị đúng (theo hướng huấn luyện mong muốn) là `α* = 92°`, `β = 45°` — sai số nhỏ `Δα = 2°`.

Vị trí cổ tay dự đoán (dùng công thức FK đã học ở bài "Vì sao không thể copy góc khớp"):

```text
α=90°: x_cổ_tay = L1·cos(90°) + L2·cos(135°) = 0 + 25×(−0.7071) = −17.68 cm
       y_cổ_tay = L1·sin(90°) + L2·sin(135°) = 30 + 25×0.7071 = 47.68 cm

α*=92°: x*_cổ_tay = 30·cos(92°) + 25·cos(137°) = 30×(−0.0349) + 25×(−0.7314)
                   = −1.047 + (−18.285) = −19.33 cm
        y*_cổ_tay = 30·sin(92°) + 25·sin(137°) = 30×0.9994 + 25×0.6820
                   = 29.98 + 17.05 = 47.03 cm
```

Sai lệch vị trí do sai số góc `2°` gây ra:

```text
Δx = −19.33 − (−17.68) = −1.65 cm
Δy = 47.03 − 47.68 = −0.65 cm
‖Δp‖ = √(1.65² + 0.65²) = √(2.7225+0.4225) = √3.145 ≈ 1.77 cm
```

**Ý nghĩa:** nếu mạng chỉ được huấn luyện bằng sai số góc thuần tuý (`|Δα| = 2°`), nó không "biết" 2° này gây ra 1.77 cm lệch ở đầu ngón tay — quan trọng hay không phụ thuộc vào chain nào bị lệch. FK layer buộc loss được tính *sau khi* đã quy đổi qua hình học thật (ở đây là 1.77 cm), khiến mạng tự động học được rằng sai số góc ở khớp có đòn bẩy dài (gần gốc, ảnh hưởng nhiều khớp con) cần được ưu tiên sửa hơn so với sai số góc tương đương ở một khớp ít ảnh hưởng — đây là lợi ích cụ thể, định lượng được của việc nhúng FK vào kiến trúc thay vì chỉ so khớp góc trực tiếp.

### Phần B — Cycle-consistency loss cho một cặp giá trị đơn giản

Giả sử với một chiều duy nhất của góc khớp (để đơn giản hoá), giá trị gốc `θ_A = 40°`. Mạng A→B dự đoán `θ_B = 25°` (khung xương đích có tỷ lệ khác). Mạng B→A (dịch ngược) từ `θ_B = 25°` cho ra `θ_A' = 37°` (không hoàn hảo vì mạng chưa hội tụ).

```text
Loss_cycle = (θ_A − θ_A')² = (40 − 37)² = 3² = 9 (đơn vị: độ²)
```

Nếu sau một bước cập nhật gradient, mạng cải thiện thành `θ_A' = 39°`:

```text
Loss_cycle_mới = (40 − 39)² = 1² = 1
```

**Ý nghĩa:** loss giảm từ 9 xuống 1 cho thấy chu trình A→B→A ngày càng nhất quán hơn — đây là tín hiệu huấn luyện *duy nhất* mà mô hình có, không có bước nào so sánh trực tiếp `θ_B = 25°` với một "góc B đúng" nào cả, vì giá trị đó không tồn tại trong dữ liệu huấn luyện không giám sát.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | NKN/residual học sâu (Villegas 2018 và kế thừa) | GMR/SOMA-retargeter (IK hình học thuần) |
|---|---|---|
| Cần dữ liệu huấn luyện? | Có — cần tập chuyển động đa dạng để mạng khái quát hoá | Không — thuật toán hình học/toán học thuần |
| Cách xử lý DoF/tỷ lệ khác nhau | Học ngầm qua dữ liệu (không cần config tường minh mỗi robot) | Cấu hình tường minh (`ik_match_table`, scale table) theo từng robot |
| Đảm bảo ràng buộc vật lý cứng (giới hạn khớp/tốc độ) | Khó đảm bảo tường minh — thường cần lớp hậu xử lý riêng | Áp trực tiếp trong ràng buộc IK/QP (đã học ở bài GMR) |
| Khái quát hoá qua robot/nguồn mới chưa thấy | Tốt hơn nếu huấn luyện đa dạng, nhưng cần retrain/finetune | Cần viết config mới hoàn toàn cho robot mới |
| Độ minh bạch/dễ debug | Thấp hơn (hộp đen mạng nơ-ron) | Cao (từng bước toán học tường minh) |
| Vai trò trong dự án này hiện tại | Hướng nghiên cứu mở rộng, dùng làm residual hoặc mô-đun sửa lỗi bổ sung | Công cụ chính (GMR) và thứ hai (SOMA-retargeter), mentor chỉ định |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "học sâu retargeting nghĩa là mạng thay thế hoàn toàn IK, không cần công thức động học nữa".** Vì sao sai: chính kiến trúc NKN đã chứng minh điều ngược lại — nó *giữ nguyên* FK layer (công thức động học thuận chính xác, không tham số) và chỉ dùng mạng để học phần khó (nghiệm IK ngầm định qua dữ liệu). Nhiều công trình kế thừa (Aberman et al. 2020) cũng giữ nguyên tinh thần "dùng cấu trúc hình học đã biết, chỉ học phần chưa biết" chứ không vứt bỏ hoàn toàn kiến thức động học. **Hiểu đúng:** retargeting học sâu hiện đại thường là *lai* (hybrid) giữa cấu trúc hình học đã biết và phần học được, không phải "mạng học tất cả từ đầu".
2. **Hiểu nhầm: "không giám sát (unsupervised) nghĩa là không cần dữ liệu gì cả".** Vì sao sai: NKN vẫn cần một **tập lớn chuyển động đa dạng** trên nhiều khung xương khác nhau (paper gốc dùng Mixamo) để huấn luyện — "không giám sát" chỉ có nghĩa là không cần **cặp nhãn** (chuyển động nguồn, chuyển động đích đã biết đúng), chứ không có nghĩa "không cần dữ liệu". **Hiểu đúng:** đây vẫn là phương pháp học máy tốn dữ liệu, chỉ khác ở loại nhãn cần thiết.

## 🏗️ Ví dụ minh hoạ trong dự án này

Trong pipeline hiện tại của dự án, GMR và SOMA-retargeter (hai công cụ hình học thuần) vẫn là lựa chọn chính. Hướng "học sâu/residual" theo tinh thần Villegas xuất hiện ở dự án dưới dạng **mô-đun bổ sung sau một bước retarget hình học thô**, không thay thế hoàn toàn nó:

```text
AMASS/LAFAN1 (chuyển động người)
        │
        ▼
GMR hoặc SOMA-retargeter (retarget hình học thô, đã học)
        │
        ▼
   [Tuỳ chọn] residual/hiệu chỉnh học được:
   - sửa lỗi cân bằng còn sót lại
   - học trực tiếp một policy tracking chuyển động đã retarget
        │
        ▼
Reference motion cho WBC (01-whole-body-control/)
hoặc training RL/imitation (04-imitation-learning-rl/)
```

Đây chính là ranh giới nối sang hai mảng khác của dự án: khi một policy RL (ví dụ dùng motion-tracking reward, xem `04-imitation-learning-rl/`) được huấn luyện để *theo dõi* (track) một chuyển động đã retarget hình học thô, bản thân policy đó đang đóng vai trò một dạng "residual học được" — sửa các artefact còn sót lại (foot sliding, mất cân bằng nhẹ) mà GMR/SOMA-retargeter chưa xử lý triệt để, thông qua tương tác thực tế với vật lý mô phỏng thay vì một mạng retargeting độc lập kiểu NKN.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **Aberman et al. (2020), "Skeleton-Aware Networks for Deep Motion Retargeting"** (ACM ToG/SIGGRAPH 2020, [arXiv:2005.05732](https://arxiv.org/pdf/2005.05732)) là bước phát triển trực tiếp và có ảnh hưởng lớn nhất kế thừa tinh thần NKN: thay vì coi khung xương như một chuỗi phẳng, nhóm tác giả biểu diễn skeleton như một **đồ thị (graph)** và định nghĩa các toán tử tích chập/pooling/unpooling khả vi (differentiable) chuyên biệt cho cấu trúc đồ thị khớp nối — cho phép nhúng nhiều khung xương có **số khớp khác nhau nhưng đồng cấu về mặt tô pô** (homeomorphic) vào cùng một không gian latent chung thông qua một "primal skeleton" (khung xương gộp chung). Đây là bước tiến trực tiếp giải quyết hạn chế của NKN gốc (vốn xử lý tuần tự theo chain, chưa khai thác cấu trúc đồ thị đầy đủ của toàn bộ cơ thể).
2. **G-DReaM — Cao, Liu, Li, Zhang, Chen (2025/2026), "Graph-conditioned Diffusion Retargeting across Multiple Embodiments"** ([arXiv:2505.20857](https://arxiv.org/abs/2505.20857)) đưa tư tưởng "học retargeting" sang kiến trúc **diffusion model có điều kiện theo đồ thị robot** — một mô hình sinh duy nhất có thể retarget một chuyển động tham chiếu sang **nhiều robot khác nhau cùng lúc** (số khớp, chiều dài đoạn, tô pô khác nhau), kể cả khi không có dữ liệu chuyển động cho robot đích, bằng cách điều kiện hoá bộ sinh theo mô tả đồ thị của từng khung xương và hướng dẫn quá trình lấy mẫu (sampling) bằng một hàm năng lượng khớp vị trí đầu mút chi (sau khi đã scale theo kích thước cơ thể) với chuyển động tham chiếu. Đây là bước mở rộng đáng kể so với NKN 2018 (vốn chỉ xử lý một cặp khung xương A↔B cố định) sang bài toán **đa embodiment tổng quát**.
3. **ReActor (2026), "Reinforcement Learning for Physics-Aware Motion Retargeting"** ([arXiv:2605.06593](https://arxiv.org/pdf/2605.06593)) đại diện cho hướng "residual qua RL" đã nhắc ở mục Ví dụ minh hoạ: thay vì học một mạng retargeting kinematic thuần như NKN, ReActor dùng RL để học một policy sửa lỗi/theo dõi chuyển động đã retarget sao cho **khả thi về vật lý** (physics-aware) trên robot thật — xác nhận đúng nhận định trong `NOI-DUNG-CHI-TIET.md` rằng hướng học sâu hiện đại thường "học một phần hiệu chỉnh (residual) bổ sung sau một bước retarget hình học thô" thay vì thay thế hoàn toàn IK.
4. **Xu hướng chung 2020–2026:** từ NKN (2018, chain-based, một cặp khung xương cố định) → Aberman (2020, graph-based, nhiều khung xương đồng cấu) → G-DReaM (2025, diffusion, đa embodiment tổng quát, không cần dữ liệu cho robot đích) → ReActor (2026, RL-residual, physics-aware) — hướng phát triển nhất quán là **mở rộng phạm vi khái quát hoá** (từ một cặp cố định sang nhiều embodiment bất kỳ) và **tăng dần tính vật lý** (từ kinematic thuần sang physics-aware qua RL), trong khi tư tưởng lõi "dùng cấu trúc đã biết (FK/graph/kích thước cơ thể) làm điều kiện, để mạng chỉ học phần chưa biết" của NKN vẫn được giữ lại xuyên suốt.

## ❓ Câu hỏi tự kiểm tra

1. Vì sao NKN cần FK layer thay vì để mạng tự học luôn cả quan hệ hình học giữa góc khớp và vị trí đầu mút?
   <details><summary>Gợi ý đáp án</summary>Vì quan hệ đó (forward kinematics) đã biết chính xác bằng công thức toán học — để mạng tự học lại từ dữ liệu vừa lãng phí năng lực học, vừa có thể học sai/xấp xỉ không chính xác; nhúng công thức đã biết giúp mạng chỉ tập trung học phần thật sự chưa biết (nghiệm IK ngầm).</details>
2. Trong ví dụ tính tay Phần A, nếu sai số góc xảy ra ở khớp khuỷu (`β`) thay vì khớp vai (`α`), sai lệch vị trí đầu mút có xu hướng lớn hơn hay nhỏ hơn so với cùng độ lớn sai số ở vai? Vì sao?
   <details><summary>Gợi ý đáp án</summary>Nhỏ hơn hoặc bằng, vì khớp vai nằm gần gốc chain và ảnh hưởng tới toàn bộ cánh tay phía sau nó (đòn bẩy dài hơn), trong khi khớp khuỷu chỉ ảnh hưởng tới đoạn cẳng tay phía sau nó — đây là hiệu ứng khuếch đại sai số dọc kinematic chain đã học ở bài "Vì sao không thể copy góc khớp".</details>
3. Vì sao cycle-consistency loss không đảm bảo tuyệt đối rằng bước retarget trung gian (θ_B) là đúng?
   <details><summary>Gợi ý đáp án</summary>Vì về lý thuyết, một lỗi ở mạng A→B có thể bị "triệt tiêu" bởi một lỗi tương ứng ở mạng B→A khiến chu trình khớp lại dù bước trung gian sai — cycle-consistency là tín hiệu hữu ích nhưng không phải bằng chứng toán học chặt chẽ về tính đúng đắn của bước trung gian.</details>
4. Aberman et al. (2020) giải quyết hạn chế nào của NKN 2018?
   <details><summary>Gợi ý đáp án</summary>NKN xử lý theo chain tuần tự trên một cặp khung xương cố định; Aberman biểu diễn skeleton như đồ thị và định nghĩa toán tử tích chập/pooling chuyên biệt, cho phép xử lý nhiều khung xương có số khớp khác nhau (nhưng đồng cấu tô pô) trong cùng một không gian latent chung.</details>
5. Vì sao G-DReaM (2025/2026) được coi là mở rộng đáng kể so với NKN, dù cả hai đều "học retargeting"?
   <details><summary>Gợi ý đáp án</summary>NKN chỉ xử lý một cặp khung xương A↔B cố định, cần huấn luyện lại cho cặp khác; G-DReaM điều kiện hoá theo đồ thị robot nên một mô hình sinh có thể retarget sang nhiều robot khác nhau, kể cả robot chưa có dữ liệu chuyển động, mở rộng phạm vi từ "một cặp" sang "đa embodiment tổng quát".</details>
6. Vì sao trong dự án này, hướng học sâu/residual hiện được dùng như một lớp bổ sung sau GMR/SOMA-retargeter thay vì thay thế chúng?
   <details><summary>Gợi ý đáp án</summary>Vì IK hình học thuần vẫn đảm bảo minh bạch, dễ debug, và áp trực tiếp ràng buộc vật lý cứng (giới hạn khớp/tốc độ) mà không cần dữ liệu huấn luyện; hướng học sâu hiện chủ yếu giá trị ở việc sửa các artefact còn sót (residual) hoặc khái quát hoá qua nhiều embodiment, chưa đủ minh bạch/đáng tin cậy để thay thế hoàn toàn bước retarget hình học cốt lõi trong production.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** lặp lại ví dụ Phần A với `L1 = 0.35 m`, `L2 = 0.20 m`, `α = 60°`, `α* = 65°` (giữ `β = 30°` không đổi ở cả hai trường hợp). Tính vị trí cổ tay cho cả hai giá trị `α`, sai lệch `‖Δp‖`, và so sánh với kết quả trong bài (1.77 cm) — sai số góc như nhau (5° so với 2° trong bài) có tạo sai lệch vị trí tỷ lệ thuận không?
2. **Đọc paper/code thật:** đọc phần "Method" của paper Aberman et al. 2020 ([arXiv:2005.05732](https://arxiv.org/pdf/2005.05732)) hoặc trang project ([peizhuoli.github.io/publication/skeleton-aware](https://peizhuoli.github.io/publication/skeleton-aware/)), mô tả bằng lời của bạn (3-5 câu) cơ chế "skeletal pooling" biến nhiều khung xương khác số khớp thành một "primal skeleton" chung — so sánh khái niệm này với "bone chain" đã học ở bài Skeleton mapping.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

Retargeting học sâu, khởi đầu từ Neural Kinematic Networks của Villegas et al. (2018), giải quyết bài toán bằng một mạng hồi quy kết hợp lớp Forward-Kinematics không tham số (để tận dụng hình học đã biết chính xác) và huấn luyện không giám sát qua cycle-consistency (retarget xuôi rồi ngược, so khớp với chuyển động gốc) vì không tồn tại nhãn "đúng" cho retargeting; ví dụ tính tay cho thấy FK layer giúp gradient phản ánh đúng hậu quả không gian của sai số góc theo từng vị trí trên kinematic chain. Khác với GMR/SOMA-retargeter (IK hình học thuần, minh bạch, không cần dữ liệu), hướng học sâu cần dữ liệu huấn luyện đa dạng và khó đảm bảo tường minh ràng buộc vật lý cứng — nhưng đã phát triển liên tục qua Aberman et al. 2020 (graph-based, nhiều khung xương đồng cấu), G-DReaM 2025/2026 (diffusion đa embodiment, không cần dữ liệu cho robot đích) và ReActor 2026 (RL-residual physics-aware); trong dự án này và các pipeline humanoid production hiện tại, hướng này chủ yếu đóng vai trò một lớp bổ sung/residual sau bước retarget hình học thô, không thay thế hoàn toàn GMR/SOMA-retargeter.
