# Bài giảng: Inverse Kinematics per-frame — thuật toán FABRIK

*(Thuộc mảng: Motion Retargeting)*

## 🎯 Mục tiêu bài học

- Giải thích được hai pha của FABRIK (forward reaching, backward reaching) và vì sao chỉ cần phép chiếu điểm lên đoạn thẳng, không cần góc/ma trận xoay.
- Tính tay được 1 vòng lặp FABRIK đầy đủ (cả 2 pha) cho một chain 3 điểm.
- Giải thích được vì sao FABRIK phù hợp cho retargeting/IK real-time hơn Jacobian-based IK về mặt chi phí tính toán, và trong trường hợp nào điều này không còn đúng.
- Phân biệt được FABRIK với Jacobian-based IK (bài giảng riêng) về biểu diễn trạng thái và cách xử lý ràng buộc.
- Nêu được ít nhất 2 hạn chế của FABRIK gốc (không xử lý giới hạn góc/hướng trực tiếp, không tối ưu đa mục tiêu tự nhiên) và cách các biến thể hiện đại khắc phục.

## 🧭 Vì sao cần học cái này? (bối cảnh)

FABRIK là lớp thuật toán IK per-frame thứ hai (bên cạnh Jacobian-based/differential IK — bài giảng riêng) dùng để biến vị trí mục tiêu đã scale (từ bước skeleton mapping) thành góc khớp cụ thể cho robot. Dù GMR — công cụ chính của dự án — chọn hướng Jacobian/QP (`mink`), FABRIK vẫn là nền tảng của **rất nhiều bộ giải IK real-time** trong game/animation/teleoperation khác, và là công cụ tư duy quan trọng để hiểu vì sao có nhiều hơn một cách hợp lý để giải cùng một bài toán IK — mỗi cách có đánh đổi khác nhau giữa tốc độ, độ chính xác, và khả năng xử lý nhiều ràng buộc.

## 🧠 Trực giác

### Góc nhìn 1: Kéo một sợi dây chuỗi hạt có độ dài cố định

Hình dung một chuỗi hạt cườm nối bằng dây cứng có độ dài cố định giữa các hạt (mỗi đoạn dây = một đoạn xương). Bạn muốn hạt cuối cùng (đầu mút) chạm đúng một điểm trên bàn. Cách làm tự nhiên nhất **không cần biết góc gì cả**: cầm hạt cuối, kéo thẳng nó tới điểm mục tiêu. Vì các hạt trước đó bị "dây kéo theo" nhưng độ dài dây không đổi, bạn lần lượt kéo từng hạt về **đúng trên đường thẳng nối nó với hạt kế tiếp (đã di chuyển)**, cách hạt đó đúng bằng độ dài đoạn dây — cứ thế đi ngược từ đầu mút về gốc. Sau đó, vì hạt gốc bị kéo lệch khỏi vị trí cố định của nó, bạn giữ chặt hạt gốc về đúng chỗ và lặp lại thao tác — lần này đi xuôi từ gốc tới đầu mút. Đây chính là forward/backward reaching của FABRIK.

**Giới hạn của loại suy này:** chuỗi hạt cườm là mô hình 2D/3D thuần hình học không có khối lượng hay giới hạn góc xoay tại từng hạt; trong khi robot thật có **giới hạn góc khớp** (một khớp không thể xoay quá 1 vòng, và một số khớp có giới hạn góc hẹp) — FABRIK gốc (được mô tả trong bài này) không xử lý trực tiếp việc "hạt không được xoay quá góc X so với đoạn trước", cần bổ sung riêng (xem mục Sai lầm thường gặp).

### Góc nhìn 2: Origami — gấp giấy từ hai đầu vào giữa

Một phép loại suy khác, nhấn vào tính chất "hai pha" của thuật toán: hình dung bạn có một dải giấy dài với các nếp gấp cố định (khớp), và bạn muốn đầu dải giấy chạm một điểm cụ thể trong khi đầu kia vẫn dính chặt vào bàn (gốc — root cố định). Bạn gấp lại dải giấy **từ đầu tự do vào trong** trước (forward reaching — ưu tiên đạt mục tiêu trước, chấp nhận gốc bị "trôi"), rồi **gấp lại từ điểm dính bàn ra ngoài** (backward reaching — kéo gốc về đúng chỗ, chấp nhận đầu tự do hơi lệch mục tiêu một chút), lặp lại vài lần cho tới khi cả hai đầu đều gần đúng vị trí mong muốn.

**Giới hạn của loại suy này:** gấp giấy có các nếp gấp góc cố định biết trước; FABRIK xử lý các "khớp" như điểm tự do hoàn toàn có thể nằm ở bất kỳ đâu trên đường thẳng nối 2 điểm lân cận — nên loại suy "nếp gấp cố định góc" hơi sai lệch so với cơ chế thật, chỉ nên dùng để hình dung "đi từ 2 đầu vào giữa" chứ không phải cơ chế hình học chính xác.

## 📐 Định nghĩa chính xác

**FABRIK (Forward And Backward Reaching Inverse Kinematics)** — Aristidou & Lasenby (2011) — giải IK cho một kinematic chain gồm n điểm khớp **p₁, p₂, ..., pₙ** (p₁ = gốc/root, pₙ = đầu mút/end-effector), với độ dài đoạn cố định **dᵢ = ‖pᵢ₊₁ − pᵢ‖** giữ nguyên trong suốt quá trình giải. Cho vị trí mục tiêu **t** cho đầu mút, thuật toán **không thao tác trên góc khớp hay ma trận xoay**, mà trực tiếp di chuyển các điểm pᵢ trong không gian Cartesian, bảo toàn độ dài đoạn bằng phép chiếu tuyến tính:

```
Điểm mới pᵢ' nằm trên đoạn thẳng nối pᵢ (vị trí hiện tại) và điểm neo (anchor)
q (điểm lân cận đã được cập nhật), cách q đúng khoảng dᵢ:

pᵢ' = q + dᵢ · (pᵢ − q) / ‖pᵢ − q‖
```

**Thuật toán gồm 2 pha, lặp lại tới khi hội tụ:**

1. **Forward reaching** (từ đầu mút về gốc): đặt pₙ = t (đầu mút = mục tiêu). Với i từ n−1 xuống 1: pᵢ' = pᵢ₊₁' + dᵢ · (pᵢ − pᵢ₊₁') / ‖pᵢ − pᵢ₊₁'‖ (dùng vị trí **cũ** của pᵢ để xác định hướng, neo tại pᵢ₊₁' **mới**).
2. **Backward reaching** (từ gốc về đầu mút): đặt p₁' = p₁_gốc (vị trí gốc ban đầu, cố định). Với i từ 2 đến n: pᵢ' = pᵢ₋₁' + dᵢ₋₁ · (pᵢ − pᵢ₋₁') / ‖pᵢ − pᵢ₋₁'‖.

Lặp lại 2 pha cho tới khi `‖pₙ − t‖ < ε` (dung sai hội tụ) hoặc đạt số vòng lặp tối đa. Nếu mục tiêu nằm **ngoài tầm với** (‖t − p₁_gốc‖ > tổng độ dài chain), FABRIK không hội tụ đúng vị trí nhưng vẫn cho một tư thế duỗi thẳng hướng về mục tiêu — hành vi hợp lý về mặt thị giác.

## ⚙️ Cơ chế hoạt động — từng bước

```
Chain gốc: p1(root, cố định) — p2 — p3 — ... — pn(end-effector)
Độ dài đoạn cố định: d1=‖p2-p1‖, d2=‖p3-p2‖, ..., d(n-1)=‖pn-p(n-1)‖
Mục tiêu: t

┌─────────────────────────────────────────────────────────────┐
│ LẶP cho tới khi ‖pn − t‖ < ε:                                  │
│                                                                 │
│  PHA 1 — FORWARD REACHING (đầu mút → gốc)                      │
│   pn ← t                                                        │
│   for i = n-1 downto 1:                                         │
│       hướng = (pi − p(i+1)) / ‖pi − p(i+1)‖                     │
│       pi ← p(i+1) + di · hướng      # bảo toàn độ dài di        │
│   (sau pha này, p1 thường bị lệch khỏi vị trí gốc ban đầu)       │
│                                                                 │
│  PHA 2 — BACKWARD REACHING (gốc → đầu mút)                      │
│   p1 ← p1_gốc_ban_đầu               # ghim lại gốc               │
│   for i = 2 to n:                                                │
│       hướng = (pi − p(i-1)) / ‖pi − p(i-1)‖                     │
│       pi ← p(i-1) + d(i-1) · hướng                               │
│   (sau pha này, pn thường gần t hơn nhưng chưa chắc = t)         │
│                                                                 │
└─────────────────────────────────────────────────────────────┘
              │
              ▼
     Trả về p1..pn → suy ra góc khớp (nếu cần) bằng cách tính
     góc giữa các đoạn liên tiếp pi−p(i-1) và p(i+1)−pi
```

Lưu ý quan trọng: FABRIK tự nó chỉ cho ra **vị trí điểm khớp**, không trực tiếp cho góc khớp. Nếu hệ điều khiển robot cần góc khớp (θ), một bước hậu kỳ tính góc giữa các đoạn liên tiếp (dùng arccos của tích vô hướng chuẩn hoá, hoặc atan2 cho 2D) là cần thiết sau khi FABRIK hội tụ.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số tự chọn để dễ hình dung, dùng 2D cho đơn giản)*

Chain 3 điểm: p1 (vai, gốc, cố định tại gốc toạ độ (0,0)), p2 (khuỷu), p3 (cổ tay, đầu mút). Độ dài đoạn: d1 = ‖p2−p1‖ = 22 cm (cánh tay trên), d2 = ‖p3−p2‖ = 18 cm (cẳng tay).

**Vị trí ban đầu** (tư thế duỗi thẳng theo trục x): p1=(0,0), p2=(22,0), p3=(40,0).

**Mục tiêu:** t = (−12.73, 34.73) (giống mục tiêu ở bài giảng "Jacobian-based IK" để dễ đối chiếu). Khoảng cách từ gốc tới mục tiêu: ‖t−p1‖ = √(12.73² + 34.73²) = √(162.05+1206.2) = √1368.3 ≈ 36.99 cm — nhỏ hơn tổng độ dài chain (22+18=40cm), nên mục tiêu **trong tầm với**, FABRIK sẽ hội tụ đúng vị trí.

**Vòng lặp 1 — Pha Forward reaching:**
- p3' = t = (−12.73, 34.73)
- Tính p2': hướng từ p3' tới p2 (dùng p2 cũ = (22,0)): vector = p2 − p3' = (22−(−12.73), 0−34.73) = (34.73, −34.73), độ dài = √(34.73²+34.73²) = 34.73×√2 ≈ 49.11. Hướng chuẩn hoá = (0.7072, −0.7072).
  p2' = p3' + d2 × hướng = (−12.73, 34.73) + 18×(0.7072, −0.7072) = (−12.73+12.73, 34.73−12.73) = (0.0, 22.0)
- Tính p1' (dùng p1 cũ = (0,0)): vector = p1 − p2' = (0−0.0, 0−22.0) = (0, −22), độ dài = 22. Hướng chuẩn hoá = (0, −1).
  p1' = p2' + d1×hướng = (0.0, 22.0) + 22×(0,−1) = (0.0, 0.0)

  (Tình cờ p1' = (0,0) đúng bằng gốc ban đầu trong ví dụ này — nhưng nói chung sau pha forward, p1' thường lệch khỏi gốc; ở đây do hình học đối xứng của ví dụ nên trùng khớp.)

**Pha Backward reaching:**
- p1'' = p1_gốc = (0,0) (ghim lại — trong ví dụ này không đổi vì đã trùng)
- Tính p2'': hướng từ p1'' tới p2' = (0.0−0, 22.0−0)/22 = (0,1). p2'' = p1'' + d1×(0,1) = (0,22.0)
- Tính p3'': hướng từ p2'' tới p3' = (−12.73−0, 34.73−22.0)/‖·‖ = (−12.73, 12.73)/√(162+162)=(−12.73,12.73)/17.99≈(−0.7076,0.7076). p3''=p2''+d2×hướng = (0,22)+18×(−0.7076,0.7076) = (0−12.74, 22+12.74) = (−12.74, 34.74)

**Kết quả sau 1 vòng lặp đầy đủ:** p3'' ≈ (−12.74, 34.74) — sai số so với mục tiêu (−12.73, 34.73) chỉ khoảng 0.01–0.02 cm, đã **hội tụ gần như hoàn hảo chỉ sau 1 vòng lặp** trong ví dụ này (vì chain 2 đoạn với mục tiêu trong tầm với thường hội tụ rất nhanh, đúng như đặc tính "chi phí mỗi vòng lặp cực thấp, số vòng lặp cần thiết cũng rất ít" đã nêu ở phần Trực giác). Với chain nhiều đoạn hơn hoặc mục tiêu khó hơn, cần vài vòng lặp bổ sung, nhưng mỗi vòng vẫn chỉ tốn các phép cộng/trừ/chuẩn hoá vector — không có nghịch đảo ma trận nào.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | FABRIK | Jacobian-based/differential IK (bài giảng riêng) |
|---|---|---|
| Biểu diễn trạng thái | Vị trí điểm khớp (Cartesian) | Góc khớp (joint space) |
| Phép toán lõi/bước lặp | Cộng/trừ/chuẩn hoá vector (chiếu điểm lên đoạn thẳng) | Nghịch đảo/giả nghịch đảo ma trận Jacobian |
| Chi phí tính toán mỗi bước | Rất thấp, O(n) đơn giản | Cao hơn, phụ thuộc kích thước ma trận (m×n) |
| Tốc độ hội tụ (số vòng lặp thực tế) | Thường rất ít vòng cho chain đơn/mục tiêu trong tầm với | Phụ thuộc độ "tuyến tính hoá tốt" quanh điểm khởi tạo |
| Xử lý nhiều mục tiêu task-space đồng thời | Không tự nhiên — thiết kế gốc cho 1 chain/1 mục tiêu | Tự nhiên — ghép nhiều hàng Jacobian, giải QP chung |
| Xử lý giới hạn góc khớp trực tiếp trong vòng lặp | Không có sẵn trong bản gốc — cần bổ sung constraint riêng | Có thể tích hợp trực tiếp trong bài toán QP (như `mink`) |
| Công cụ tiêu biểu trong dự án | Không phải lựa chọn của GMR, nhưng nền tảng phổ biến cho IK real-time game/VR/teleoperation khác | GMR (qua `mink` + MuJoCo) |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "FABRIK tự động tôn trọng giới hạn góc khớp của robot vì nó chỉ dịch chuyển điểm".** Vì sao sai: bản FABRIK gốc (Aristidou & Lasenby 2011) chỉ bảo toàn **độ dài đoạn**, hoàn toàn không kiểm tra góc giữa hai đoạn liên tiếp có nằm trong giới hạn cơ học của khớp hay không — một chain có thể "gập ngược" theo hình học hoàn toàn hợp lệ (bảo toàn độ dài) nhưng góc khớp tương ứng lại vượt xa giới hạn vật lý thật của robot. **Hiểu đúng:** cần thêm bước hậu kỳ hoặc biến thể FABRIK có ràng buộc góc (constrained FABRIK) để loại các nghiệm hình học hợp lệ nhưng vô nghĩa về cơ khí — đây chính là lý do bài giảng "Joint limit clamping" vẫn cần thiết ngay cả khi dùng FABRIK.
2. **Hiểu nhầm: "FABRIK luôn nhanh hơn Jacobian-based IK trong mọi trường hợp, nên luôn nên chọn FABRIK".** Vì sao sai: FABRIK nhanh **trên mỗi chain đơn lẻ, mục tiêu đơn**, nhưng khi cần giải **nhiều đầu mút chi phối hợp đồng thời** (ví dụ giữ cân bằng toàn thân trong khi cả hai tay đều có mục tiêu — một ràng buộc chéo giữa các chain), FABRIK không có cơ chế tự nhiên để cân bằng nhiều mục tiêu xung đột trong cùng một vòng lặp như bài toán QP của Jacobian-based IK. **Hiểu đúng:** lựa chọn phụ thuộc vào bài toán cụ thể — GMR (cần giải đồng thời nhiều task-space trên toàn thân với ràng buộc vật lý) chọn Jacobian/QP, trong khi nhiều ứng dụng animation/VR (mỗi tay/chân là một chain độc lập, không cần phối hợp chặt) chọn FABRIK vì tốc độ.

## 🏗️ Ví dụ minh hoạ trong dự án này

Dù GMR (công cụ chính của dự án) không dùng FABRIK mà chọn Jacobian/QP (`mink`), FABRIK vẫn liên quan trực tiếp tới `08-real-robot-deployment/` — trong các hệ thống teleoperation real-time trên phần cứng hạn chế (ví dụ chạy trên CPU nhúng của bộ điều khiển VR, hoặc khi cần giải hàng chục chain độc lập mỗi frame ở tần số cao mà không có GPU), FABRIK là lựa chọn phổ biến vì chi phí mỗi vòng lặp cực thấp như ví dụ tính tay ở trên đã minh hoạ (hội tụ gần như hoàn hảo chỉ sau 1 vòng cho một chain 2 đoạn). Việc hiểu rõ đánh đổi giữa FABRIK và Jacobian/QP giúp giải thích được một quyết định thiết kế cụ thể: vì sao GMR — nhắm tới độ chính xác và nhiều ràng buộc vật lý tích hợp (velocity limit, nhiều task-space đồng thời) — chấp nhận chi phí tính toán cao hơn của Jacobian/QP thay vì chọn con đường "rẻ nhất" là FABRIK.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **M-FABRIK (biến thể cho robot di động, 2020–2021, vẫn được trích dẫn trong nghiên cứu 2024–2025).** *"M-FABRIK: A New Inverse Kinematics Approach to Mobile Manipulator Robots Based on FABRIK"* mở rộng FABRIK gốc cho tay máy gắn trên robot di động, với ưu điểm được xác nhận lại trong các khảo sát gần đây: "simplicity to implement, fast convergence and low computational cost, allowing real-time applications" — đúng đặc tính đã minh hoạ bằng số ở ví dụ tính tay của bài này. [ResearchGate](https://www.researchgate.net/publication/347824143_M-FABRIK_A_New_Inverse_Kinematics_Approach_to_Mobile_Manipulator_Robots_Based_on_FABRIK)
2. **FABRIK vẫn là lựa chọn phổ biến cho IK real-time trong các hệ teleoperation 2024–2025, dù không phải lựa chọn duy nhất.** Khảo sát *"Teleoperation of Humanoid Robots: A Survey"* (arXiv:2301.04317, vẫn được trích dẫn rộng rãi trong các paper 2024–2025 về teleoperation) ghi nhận: "Mainstream approaches for arm motion generation rely on Inverse Kinematics (IK) solvers or more advanced real-time motion planners" — trong đó các solver dạng lặp nhanh như FABRIK và họ hàng của nó tiếp tục là lựa chọn phổ biến khi cần chạy trên phần cứng hạn chế. [arXiv:2301.04317](https://arxiv.org/pdf/2301.04317)
3. **Xu hướng 2025–2026: kết hợp FABRIK/giải tích với tối ưu hoá để xử lý ràng buộc góc/hướng.** Đúng như hạn chế đã nêu ở mục Sai lầm thường gặp (FABRIK gốc không xử lý giới hạn góc), paper *"A Framework for Combining Optimization-Based and Analytic Inverse Kinematics"* (arXiv:2602.05092) đại diện cho xu hướng lai ghép: dùng lời giải nhanh kiểu FABRIK/giải tích làm khởi tạo tốt, rồi tinh chỉnh bằng một bước tối ưu hoá có ràng buộc — giảm số vòng lặp cần thiết của bước tối ưu hoá trong khi vẫn tôn trọng giới hạn cơ học mà FABRIK thuần không đảm bảo được. [arXiv:2602.05092](https://arxiv.org/pdf/2602.05092)

## ❓ Câu hỏi tự kiểm tra

1. Nêu đúng 2 pha của FABRIK và điểm khác biệt cốt lõi so với Jacobian-based IK về biểu diễn trạng thái.
   <details><summary>Gợi ý đáp án</summary>Forward reaching (đầu mút→gốc) và backward reaching (gốc→đầu mút); FABRIK thao tác trực tiếp trên vị trí điểm Cartesian, không dùng góc khớp/ma trận xoay như Jacobian-based IK.</details>
2. Trong ví dụ tính tay, vì sao chỉ cần 1 vòng lặp đã hội tụ gần như hoàn hảo? Điều này có luôn đúng với mọi chain không?
   <details><summary>Gợi ý đáp án</summary>Vì chain chỉ có 2 đoạn và mục tiêu nằm trong tầm với, hình học đơn giản; với chain dài hơn hoặc mục tiêu khó hơn (gần biên tầm với, nhiều ràng buộc), cần nhiều vòng lặp hơn để hội tụ.</details>
3. Vì sao FABRIK không tự động đảm bảo góc khớp nằm trong giới hạn cơ học của robot?
   <details><summary>Gợi ý đáp án</summary>Vì FABRIK chỉ bảo toàn độ dài đoạn qua phép chiếu điểm lên đoạn thẳng, không kiểm tra góc giữa các đoạn liên tiếp — cần bước ràng buộc góc bổ sung (constrained FABRIK) hoặc joint limit clamping hậu kỳ.</details>
4. Nếu mục tiêu nằm ngoài tầm với của chain (‖t−p1‖ > tổng độ dài các đoạn), FABRIK sẽ cho kết quả gì?
   <details><summary>Gợi ý đáp án</summary>Chain sẽ duỗi thẳng hoàn toàn hướng về phía mục tiêu (đầu mút không chạm đúng mục tiêu nhưng ở vị trí gần nhất có thể theo hướng đó) — hành vi hợp lý về mặt thị giác dù không đạt sai số 0.</details>
5. Vì sao GMR (dùng Jacobian/QP qua `mink`) không chọn FABRIK dù FABRIK rẻ hơn về mặt tính toán mỗi bước?
   <details><summary>Gợi ý đáp án</summary>Vì GMR cần giải đồng thời nhiều mục tiêu task-space toàn thân (nhiều đầu mút chi phối hợp) cùng ràng buộc velocity/joint limit tích hợp trực tiếp — bài toán QP của Jacobian-based IK xử lý tự nhiên yêu cầu này, trong khi FABRIK thiết kế gốc cho 1 chain/1 mục tiêu, không có cơ chế tự nhiên cho phối hợp đa mục tiêu và ràng buộc tích hợp.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể:** lặp lại ví dụ tính tay ở trên (chain p1-p2-p3, d1=22, d2=18) nhưng với mục tiêu mới t=(30, 15) — kiểm tra xem mục tiêu này có nằm trong tầm với không (so ‖t−p1‖ với d1+d2), sau đó chạy 1 vòng lặp forward+backward đầy đủ, tính p3'' cuối cùng và sai số so với t.
2. **Đọc/chạy code thật:** tìm một implementation FABRIK mã nguồn mở (nhiều bản có sẵn trên GitHub bằng Python/JavaScript cho mục đích animation/game), chạy thử với một chain 4-5 đoạn và quan sát trực quan số vòng lặp cần thiết để hội tụ khi thay đổi vị trí mục tiêu — đối chiếu với tuyên bố "chi phí mỗi vòng lặp cực thấp, số vòng lặp cần thiết cũng rất ít" đã nêu trong bài.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

FABRIK giải bài toán IK cho một kinematic chain hoàn toàn khác cách tiếp cận Jacobian-based: thay vì tính toán trên góc khớp và ma trận xoay, nó coi mỗi khớp là một điểm trong không gian và lặp lại hai pha — forward reaching (kéo đầu mút về đúng mục tiêu rồi lần lượt kéo các điểm trước đó về đúng trên đoạn thẳng, bảo toàn độ dài xương) và backward reaching (ghim lại gốc rồi lặp lại theo chiều ngược) — như ví dụ tính tay ở trên cho thấy, cách này hội tụ cực nhanh (thường chỉ vài vòng, đôi khi 1 vòng) với chi phí mỗi bước rất thấp (chỉ cộng/trừ/chuẩn hoá vector, không nghịch đảo ma trận), khiến nó trở thành nền tảng phổ biến cho IK real-time trong game/animation/teleoperation — nhưng bản gốc không tự động tôn trọng giới hạn góc khớp cơ học và không tự nhiên xử lý nhiều mục tiêu task-space phối hợp cùng lúc, đó là lý do GMR (cần cả hai điều này) chọn hướng Jacobian/QP thay vì FABRIK, còn các hệ thống hiện đại khác kết hợp FABRIK làm khởi tạo nhanh cho một bước tối ưu hoá tinh chỉnh sau đó.
