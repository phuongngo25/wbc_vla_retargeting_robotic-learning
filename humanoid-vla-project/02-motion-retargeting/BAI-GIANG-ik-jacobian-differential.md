# Bài giảng: Inverse Kinematics per-frame — thuật toán Jacobian-based/differential IK

*(Thuộc mảng: Motion Retargeting)*

## 🎯 Mục tiêu bài học

- Viết được công thức cập nhật `Δθ = J⁺·Δx` và giải thích được ý nghĩa từng thành phần.
- Tính tay được một bước lặp Newton–Raphson cho một chain 2 khớp phẳng đơn giản.
- Giải thích được vì sao "differential IK" hiện đại (mink) hình thức hoá mỗi bước lặp thành một bài toán QP thay vì chỉ tính giả nghịch đảo trần.
- Phân biệt được IK Jacobian-based với FABRIK (bài giảng riêng) về mặt cơ chế và chi phí tính toán.
- Nêu được ít nhất 2 hạn chế thực tế của phương pháp này (kỳ dị Jacobian, cực tiểu địa phương) và cách hệ thống hiện đại (QP có ràng buộc) giảm nhẹ chúng.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Đây là một trong hai lớp thuật toán (lớp còn lại là FABRIK — xem bài giảng riêng) dùng để giải bước "IK per-frame" trong pipeline retargeting: sau khi skeleton mapping (bài giảng riêng) đã tính ra vị trí mục tiêu đã scale cho từng đầu mút chi, ta cần một thuật toán biến vị trí mục tiêu đó thành góc khớp cụ thể. GMR — công cụ retargeting chính của dự án — chọn chính xác lớp thuật toán này (qua thư viện `mink` + MuJoCo), nên hiểu cơ chế Jacobian-based/differential IK là điều kiện để hiểu được vì sao GMR hoạt động và giới hạn của nó ở đâu.

## 🧠 Trực giác

### Góc nhìn 1: Đi bộ xuống dốc trong sương mù (gradient descent)

Hình dung bạn đứng trên một ngọn núi trong sương mù dày đặc, không nhìn thấy toàn cảnh, chỉ cảm nhận được **độ dốc ngay dưới chân**. Bạn muốn xuống tới đáy thung lũng (vị trí mục tiêu của đầu mút chi) nhưng không biết đường đi tổng thể. Chiến lược hợp lý: nhìn hướng dốc nhất ngay tại vị trí hiện tại, bước một bước nhỏ theo hướng đó, rồi đánh giá lại độ dốc, lặp lại. Đây chính là bản chất của IK Jacobian-based: `Δx` (sai số vị trí hiện tại so với mục tiêu) đóng vai trò "còn cách đáy thung lũng bao xa", ma trận Jacobian J đóng vai trò "độ dốc cục bộ" liên hệ giữa việc di chuyển khớp và việc đầu mút di chuyển, và mỗi bước lặp chỉ đi một đoạn nhỏ rồi đánh giá lại.

**Giới hạn của loại suy này:** đi bộ xuống dốc trong sương mù là bài toán 1 biến độ cao duy nhất; ở đây "hướng dốc" phải được tính đồng thời cho **nhiều biến khớp** ảnh hưởng cùng lúc tới vị trí đầu mút 3D, và quan hệ này không phải lúc nào cũng "dốc đều" — có những cấu hình khớp mà việc di chuyển khớp gần như không làm đầu mút di chuyển theo hướng cần thiết (gọi là **kỳ dị — singularity**, xem mục Sai lầm thường gặp), giống như đứng ở một chỗ bằng phẳng giữa sương mù mà không biết đi hướng nào.

### Góc nhìn 2: Điều chỉnh dây rối bằng cách kéo nhẹ từng đầu

Một phép loại suy khác: tưởng tượng một chuỗi các thanh nối khớp nhau (kinematic chain) như một cánh tay robot đồ chơi có nhiều khớp xoay, và bạn muốn đầu mút (đầu ngón tay) chạm đúng một điểm trên bàn. Thay vì tính toán chính xác góc từng khớp ngay lập tức (rất khó với chain dài), bạn **xoay nhẹ từng khớp một lượng nhỏ**, quan sát đầu mút di chuyển bao nhiêu và theo hướng nào ứng với mỗi lượng xoay nhỏ đó (đây chính là thông tin trong ma trận Jacobian — "xoay khớp i một chút thì đầu mút di chuyển theo hướng nào, bao xa"), rồi kết hợp các lượng xoay nhỏ ở tất cả khớp theo tỷ lệ hợp lý để đầu mút tiến gần mục tiêu nhất có thể trong một bước, lặp lại nhiều lần.

**Giới hạn của loại suy này:** loại suy này gợi ý việc "thử từng khớp một cách tuần tự", trong khi Jacobian-based IK thực chất giải **đồng thời** tất cả khớp trong một phép toán đại số tuyến tính (nghịch đảo ma trận) — không phải thử tuần tự — nên nhanh hơn nhiều so với việc thực sự thử từng khớp một như loại suy gợi ý.

## 📐 Định nghĩa chính xác

Cho một kinematic chain với vector góc khớp **θ** ∈ ℝⁿ (n bậc tự do) và vị trí đầu mút (end-effector) **x** = FK(θ) ∈ ℝᵐ (m chiều không gian Cartesian, thường m=3 hoặc 6 nếu tính cả hướng), quan hệ vi phân giữa vận tốc khớp và vận tốc đầu mút là:

```
ẋ = J(θ) · θ̇
```

trong đó **J(θ) ∈ ℝ^(m×n)** là **ma trận Jacobian**, với phần tử J_ij = ∂x_i/∂θ_j — đạo hàm riêng của toạ độ đầu mút thứ i theo góc khớp thứ j.

**Bài toán IK vi phân (differential IK)** tại mỗi bước lặp: cho sai số vị trí `Δx = x_mục_tiêu − x_hiện_tại`, tìm bước cập nhật góc khớp `Δθ` sao cho `J·Δθ ≈ Δx`. Vì J thường không vuông (m ≠ n) hoặc suy biến, nghiệm xấp xỉ bình phương tối thiểu (least-squares) dùng **giả nghịch đảo Moore-Penrose** J⁺:

```
Δθ = J⁺ · Δx,   với J⁺ = Jᵀ(J·Jᵀ)⁻¹  (khi m < n, dư thừa bậc tự do — redundant)
```

Cập nhật `θ ← θ + Δθ`, tính lại `x = FK(θ)`, lặp lại cho tới khi `‖Δx‖` đủ nhỏ. Đây chính là một dạng **Newton–Raphson** áp dụng cho hệ phương trình phi tuyến `FK(θ) = x_mục_tiêu`, vì J đóng vai trò đạo hàm (Jacobian) của hàm phi tuyến FK trong khai triển Newton.

**Differential IK hiện đại (dạng QP)**: thay vì tính J⁺ trực tiếp, mỗi bước lặp được hình thức hoá thành một **bài toán quy hoạch toàn phương (Quadratic Program — QP)**:

```
minimize_{θ̇}   ‖J·θ̇ − Δx‖²  +  (các số hạng regularization)
subject to      θ_min ≤ θ + θ̇·dt ≤ θ_max      (joint limit)
                |θ̇| ≤ θ̇_max                    (velocity limit)
                J_task_khác · θ̇ = Δx_task_khác  (nhiều mục tiêu task-space đồng thời)
```

Đây là cách thư viện **`mink`** (mà GMR sử dụng, kết hợp MuJoCo) giải IK — nó xử lý luôn ràng buộc giới hạn khớp/tốc độ **ngay trong vòng lặp giải**, thay vì xử lý như bước hậu kỳ (post-processing) tách rời.

## ⚙️ Cơ chế hoạt động — từng bước

```
Bắt đầu: θ₀ (góc khớp khởi tạo, thường = kết quả frame trước đó)
x_mục_tiêu (vị trí đầu mút đã scale từ bước skeleton mapping)

┌─────────────────────────────────────────────────────────┐
│  LẶP (k = 0, 1, 2, ...) cho tới khi hội tụ:                │
│                                                             │
│  1. Tính FK:      x_k = FK(θ_k)                            │
│  2. Tính sai số:  Δx_k = x_mục_tiêu − x_k                   │
│  3. Nếu ‖Δx_k‖ < ε  →  DỪNG, trả về θ_k                     │
│  4. Tính Jacobian: J_k = ∂FK/∂θ tại θ_k                     │
│  5a. [Bản cổ điển]  Δθ_k = J_k⁺ · Δx_k                      │
│  5b. [Bản QP/mink]  giải QP:                                │
│        min ‖J_k·θ̇ − Δx_k‖² s.t. joint/velocity limits       │
│        → Δθ_k = θ̇* · dt                                    │
│  6. Cập nhật:      θ_{k+1} = θ_k + Δθ_k                     │
│                                                             │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
             θ_final → dùng cho frame retarget này
             (θ_final trở thành θ₀ khởi tạo cho frame tiếp theo
              → "warm start", giúp hội tụ nhanh hơn vì hai frame
              liên tiếp thường gần nhau)
```

Với retargeting nhiều đầu mút chi cùng lúc (tay trái, tay phải, chân trái, chân phải, đầu — tức nhiều "task" task-space đồng thời), Jacobian được ghép (stack) từ Jacobian của từng đầu mút, và bài toán QP giải đồng thời tất cả — đây chính là điểm khác biệt so với giải riêng lẻ từng bone chain một cách độc lập.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số tự chọn để dễ hình dung)*

Xét chain 2 khớp phẳng (2D) giống bài giảng "Vì sao không thể copy trực tiếp góc khớp": vai→khuỷu (L1=22cm)→cổ tay (L2=18cm), góc khớp θ = (α, β) với α là góc vai, β là góc khuỷu (tương đối so với cánh tay trên).

**Forward Kinematics:**
```
x = L1·cos(α) + L2·cos(α+β)
y = L1·sin(α) + L2·sin(α+β)
```

**Jacobian** (đạo hàm riêng theo α, β):
```
∂x/∂α = −L1·sin(α) − L2·sin(α+β)
∂x/∂β = −L2·sin(α+β)
∂y/∂α =  L1·cos(α) + L2·cos(α+β)
∂y/∂β =  L2·cos(α+β)

J = [ ∂x/∂α  ∂x/∂β ]
    [ ∂y/∂α  ∂y/∂β ]
```

**Bước lặp cụ thể:** giả sử khởi tạo θ₀ = (α=60°, β=30°). Tính FK:
- α+β = 90°
- x₀ = 22·cos(60°) + 18·cos(90°) = 22×0.5 + 18×0 = 11.0 cm
- y₀ = 22·sin(60°) + 18·sin(90°) = 22×0.866 + 18×1 = 19.05 + 18 = 37.05 cm

Mục tiêu (lấy từ ví dụ ở bài giảng trước): x_mục_tiêu = −12.73 cm, y_mục_tiêu = 34.73 cm.

Sai số: Δx = (−12.73 − 11.0, 34.73 − 37.05) = (−23.73, −2.32) cm.

Tính Jacobian tại θ₀ (α=60°, β=30°, α+β=90°):
```
∂x/∂α = −22·sin(60°) − 18·sin(90°) = −22×0.866 − 18×1 = −19.05 − 18 = −37.05
∂x/∂β = −18·sin(90°) = −18
∂y/∂α =  22·cos(60°) + 18·cos(90°) = 22×0.5 + 18×0 = 11.0
∂y/∂β =  18·cos(90°) = 0

J = [ −37.05   −18 ]
    [  11.0      0 ]
```

Vì J vuông (2×2) trong ví dụ này (m=n=2, không dư thừa), ta dùng nghịch đảo thường thay vì giả nghịch đảo. Định thức: det(J) = (−37.05×0) − (−18×11.0) = 0 + 198 = 198.

Nghịch đảo: J⁻¹ = (1/198) × [ 0    18  ]
                              [ −11.0  −37.05 ]

Δθ = J⁻¹ · Δx = (1/198) × [ 0×(−23.73) + 18×(−2.32) ]
                          [ −11.0×(−23.73) + (−37.05)×(−2.32) ]

= (1/198) × [ −41.76 ]
            [ 261.03 + 85.96 ]

= (1/198) × [ −41.76 ]
            [ 346.99 ]

= [ −0.211 rad ]
  [  1.753 rad ]

**Nhận xét quan trọng:** bước Δθ tính ra rất lớn (1.753 rad ≈ 100°) — đây là dấu hiệu điển hình của việc bước Newton–Raphson "thô" (không giới hạn bước) có thể **overshoot** mạnh khi sai số ban đầu lớn và Jacobian gần suy biến (det(J)=198 không quá nhỏ nhưng cấu hình α+β=90° khiến ∂y/∂β=0, một dạng gần-kỳ dị cục bộ theo trục β). Trong thực hành, differential IK luôn **giới hạn kích thước bước** (step size / damping, hoặc qua ràng buộc velocity limit trong QP ở mục Cơ chế) để tránh dao động/overshoot — đây chính là lý do bản "QP có ràng buộc" (mink) ổn định hơn nhiều so với công thức `Δθ = J⁺Δx` trần trụi ở trên khi áp dụng thực tế cho nhiều frame liên tiếp.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Jacobian-based/differential IK | FABRIK (xem bài giảng riêng) |
|---|---|---|
| Biểu diễn trạng thái | Góc khớp (joint space) | Vị trí điểm khớp trong không gian (Cartesian) |
| Phép toán lõi | Nghịch đảo/giả nghịch đảo ma trận mỗi bước | Chiếu điểm lên đoạn thẳng (không lượng giác, không nghịch đảo ma trận) |
| Xử lý nhiều mục tiêu task-space cùng lúc | Tự nhiên — ghép nhiều hàng Jacobian, giải QP chung | Khó hơn — thiết kế gốc cho 1 chain/1 mục tiêu, cần mở rộng cho multi-chain |
| Tích hợp ràng buộc (joint/velocity limit) | Trực tiếp trong bài toán QP cùng lúc với giải IK | Thường xử lý hậu kỳ (post-processing) sau khi hội tụ hình học |
| Chi phí tính toán/bước lặp | Cao hơn (nghịch đảo ma trận m×n) | Rất thấp (vài phép cộng/trừ vector, chuẩn hoá) |
| Công cụ tiêu biểu | `mink` + MuJoCo (dùng trong GMR) | Nhiều engine game/animation real-time |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "chỉ cần tính `J⁺·Δx` một lần là ra ngay góc khớp đúng, không cần lặp".** Vì sao sai: FK là hàm **phi tuyến** (chứa sin/cos), trong khi `Δθ = J⁺Δx` chỉ là xấp xỉ **tuyến tính cục bộ** (đạo hàm bậc nhất) quanh θ hiện tại — chỉ đúng khi Δx đủ nhỏ. Với sai số lớn (như ví dụ tính tay ở trên, Δθ tính ra tới 100°), một bước duy nhất sẽ overshoot xa khỏi nghiệm thật. **Hiểu đúng:** phải lặp nhiều bước, mỗi bước tính lại Jacobian tại vị trí mới (hoặc dùng bước lặp có giới hạn kích thước/damping), không bao giờ tin một bước "nhảy thẳng tới đích".
2. **Hiểu nhầm: "Jacobian luôn khả nghịch, càng nhiều khớp thì giải IK càng dễ".** Vì sao sai: chain càng nhiều khớp (dư thừa bậc tự do — redundant DoF) thì J càng "gầy" (m<n), không có nghịch đảo thường mà phải dùng giả nghịch đảo, và tại một số **cấu hình đặc biệt** (ví dụ cánh tay duỗi thẳng hoàn toàn), J bị **suy biến/kỳ dị (singular)** — det(JJᵀ) tiến về 0, khiến J⁺ chứa số rất lớn, gây bước cập nhật bùng nổ (numerical instability). **Hiểu đúng:** cần kỹ thuật ổn định số (damped least-squares / Levenberg-Marquardt, hoặc ràng buộc QP như mink) để xử lý vùng gần-kỳ dị, không thể giả định Jacobian "luôn tốt" chỉ vì có nhiều khớp hơn.
3. **Hiểu nhầm: "differential IK chỉ giải được 1 đầu mút chi tại 1 thời điểm".** Vì sao sai: đây là hiểu nhầm phổ biến do nhầm với công thức `Δθ=J⁺Δx` cơ bản cho 1 chain; nhưng bản chất bài toán QP hiện đại (mục Định nghĩa) cho phép **ghép nhiều hàng Jacobian** của nhiều đầu mút (tay trái, tay phải, chân trái, chân phải) thành một bài toán tối ưu chung, giải đồng thời — đây chính xác là cách GMR giải retarget toàn thân trong một lần gọi solver mỗi frame, không phải giải tuần tự từng chi.

## 🏗️ Ví dụ minh hoạ trong dự án này

GMR (công cụ retargeting chính của dự án) dùng chính xác lớp thuật toán này: theo tài liệu chính thức, GMR xây dựng bộ giải IK vi phân trên thư viện **`mink`** kết hợp **MuJoCo** làm mô hình động học. Với mỗi frame chuyển động người, GMR set nhiều mục tiêu task-space đồng thời (vị trí/hướng của pelvis, hai bàn tay, hai bàn chân, đầu — tuỳ cấu hình robot) và giải một bài toán QP duy nhất mỗi bước lặp để tìm vận tốc khớp tối ưu thoả mãn tất cả mục tiêu, đồng thời áp giới hạn tốc độ khớp (velocity limit mặc định khoảng 3π rad/s) ngay trong ràng buộc QP. Nhờ đây, GMR đạt tốc độ 35–45 FPS trên CPU laptop (Intel i9) và 60–70 FPS trên CPU máy trạm mạnh (AMD Threadripper) — đủ nhanh để chạy real-time streaming từ thiết bị mocap, phục vụ cả retargeting offline (từ AMASS/LAFAN1) lẫn teleoperation trực tiếp (`08-real-robot-deployment/`).

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **`mink` là "spiritual successor" của Pink, chuyển hệ sinh thái từ Pinocchio sang MuJoCo.** Theo tài liệu và bài viết kỹ thuật 2026, `mink` là thư viện differential IK xây trên MuJoCo, hình thức hoá mỗi bước điều khiển thành một QP để giải vận tốc khớp tối ưu cục bộ — là "direct spiritual successor to Pink (by Stéphane Caron), which used Pinocchio", giúp cộng đồng nghiên cứu vốn đã dùng hệ sinh thái MuJoCo/DeepMind có thể dùng differential IK mà không cần chuyển sang thư viện động học khác. [mink documentation](https://kevinzakka.github.io/mink/) · [BrightCoding blog 2026](https://www.blog.brightcoding.dev/2026/06/04/stop-wrestling-with-robot-math-use-mink-instead)
2. **MJINX (2025) mở rộng hướng này sang GPU/JAX.** MJINX là thư viện IK số khả vi tự động (auto-differentiable) xây trên JAX + MuJoCo MJX, lấy cảm hứng trực tiếp từ Pink và mink — cho thấy xu hướng 2025 là đưa differential IK từ CPU (mink) sang GPU có thể vi phân được (MJINX), phục vụ các pipeline cần giải IK cho hàng loạt (batch) chuyển động song song — gần với hướng mà SOMA-retargeter (bài giảng riêng) theo đuổi nhưng qua nền tảng khác (Newton+Warp thay vì JAX). [MJINX GitHub](https://github.com/based-robotics/mjinx)
3. **Hướng kết hợp IK giải tích và tối ưu hoá (2025–2026).** Paper *"A Framework for Combining Optimization-Based and Analytic Inverse Kinematics"* (arXiv:2602.05092) và *"HJCD-IK: GPU-Accelerated Inverse Kinematics through Batched Hybrid Jacobian Coordinate Descent"* (arXiv:2510.07514) cho thấy xu hướng hiện tại: thay vì chỉ dùng thuần QP dựa trên Jacobian, các phương pháp mới kết hợp lời giải giải tích (closed-form cho các chain đơn giản) làm điểm khởi tạo tốt cho vòng lặp tối ưu hoá số, giảm số bước lặp cần thiết và tránh các vùng gần-kỳ dị đã nêu ở mục Sai lầm thường gặp. [arXiv:2602.05092](https://arxiv.org/pdf/2602.05092) · [arXiv:2510.07514](https://arxiv.org/pdf/2510.07514)

## ❓ Câu hỏi tự kiểm tra

1. Viết công thức cập nhật cơ bản của Jacobian-based IK và giải thích ý nghĩa của J⁺.
   <details><summary>Gợi ý đáp án</summary>Δθ = J⁺·Δx; J⁺ là giả nghịch đảo Moore-Penrose của Jacobian, dùng để giải xấp xỉ bình phương tối thiểu khi J không vuông/không khả nghịch, ánh xạ sai số vị trí Cartesian ngược về không gian khớp.</details>
2. Trong ví dụ tính tay ở trên, vì sao bước Δθ tính ra rất lớn (β thay đổi hơn 100°)? Nêu 1 cách khắc phục.
   <details><summary>Gợi ý đáp án</summary>Vì sai số ban đầu Δx lớn trong khi công thức J⁺Δx chỉ là xấp xỉ tuyến tính cục bộ — cách khắc phục: giới hạn kích thước bước (step size), dùng damped least-squares, hoặc ràng buộc velocity limit trong bài toán QP.</details>
3. Kỳ dị Jacobian (singularity) là gì và vì sao nó nguy hiểm cho thuật toán này?
   <details><summary>Gợi ý đáp án</summary>Là cấu hình khớp mà det(J·Jᵀ) tiến về 0 (ví dụ chain duỗi thẳng hoàn toàn) — J⁺ khi đó chứa số rất lớn, gây bước cập nhật Δθ bùng nổ, mất ổn định số.</details>
4. Vì sao "differential IK dạng QP" (mink) được coi là cải tiến so với công thức `Δθ=J⁺Δx` cổ điển?
   <details><summary>Gợi ý đáp án</summary>Vì QP cho phép tích hợp trực tiếp ràng buộc joint/velocity limit và nhiều mục tiêu task-space đồng thời ngay trong bài toán giải, thay vì xử lý hậu kỳ hoặc chỉ giải 1 mục tiêu 1 lúc — ổn định và thực tế hơn cho robot thật.</details>
5. So với FABRIK, vì sao Jacobian-based IK "tự nhiên" hơn khi cần giải nhiều đầu mút chi cùng lúc (ví dụ 5 điểm: 2 tay, 2 chân, đầu)?
   <details><summary>Gợi ý đáp án</summary>Vì Jacobian của nhiều đầu mút có thể ghép (stack) thành một ma trận lớn và giải chung một bài toán tối ưu (QP) duy nhất mỗi bước, trong khi FABRIK vốn thiết kế cho một chain/một mục tiêu, cần cơ chế mở rộng riêng để xử lý nhiều chain phối hợp.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể:** lặp lại ví dụ tính tay ở trên với θ₀ khác (α=45°, β=45°) và cùng mục tiêu (x=−12.73, y=34.73). Tính FK tại θ₀, sai số Δx, Jacobian tại θ₀, và bước Δθ đầu tiên. So sánh độ lớn Δθ với ví dụ trong bài — điểm khởi tạo tốt hơn có làm bước đầu tiên "hợp lý" hơn không?
2. **Đọc code thật:** clone thư viện `mink` (`github.com/kevinzakka/mink`) hoặc GMR (`github.com/YanjieZe/GMR`), tìm phần định nghĩa bài toán QP cho IK (thường trong file liên quan đến `solve_ik` hoặc `tasks`), xác định chính xác các thành phần: hàm mục tiêu (cost), các ràng buộc joint/velocity limit được viết dưới dạng bất đẳng thức nào.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

IK Jacobian-based/differential IK giải bài toán "tìm góc khớp cho vị trí đầu mút mục tiêu" bằng cách lặp lại một bước Newton–Raphson cục bộ: dùng ma trận Jacobian (liên hệ vận tốc khớp và vận tốc đầu mút) và giả nghịch đảo của nó để suy ra một bước cập nhật góc khớp nhỏ, tính lại sai số, lặp lại tới khi hội tụ — như ví dụ tính tay ở trên cho thấy, bước cập nhật này có thể rất lớn nếu sai số ban đầu lớn hoặc Jacobian gần kỳ dị, nên các hệ thống hiện đại như `mink` (dùng trong GMR) hình thức hoá mỗi bước thành một bài toán quy hoạch toàn phương (QP) tích hợp sẵn ràng buộc giới hạn khớp/tốc độ và nhiều mục tiêu task-space đồng thời, thay vì dùng công thức giả nghịch đảo trần trụi — đây là lý do GMR đạt được cả độ chính xác lẫn tốc độ real-time (35–70 FPS) trên CPU thuần.
