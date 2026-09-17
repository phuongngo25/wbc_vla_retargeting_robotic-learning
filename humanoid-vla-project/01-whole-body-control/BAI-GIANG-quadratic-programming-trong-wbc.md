# Bài giảng: Quadratic Programming (QP) trong WBC

*(Thuộc mảng: Whole-Body Control (WBC))*

## 🎯 Mục tiêu bài học

- Giải thích được vì sao công thức operational-space thuần túy (null-space projection) không đủ để xử lý ràng buộc bất đẳng thức, và QP giải quyết vấn đề đó thế nào.
- Viết đúng dạng chuẩn của một bài toán QP: biến quyết định, hàm mục tiêu bậc hai, ràng buộc đẳng thức/bất đẳng thức.
- Thiết lập được một QP đơn giản cho WBC (2 tác vụ + 1 ràng buộc bất đẳng thức) và giải tay bằng phương pháp hình học/KKT cho trường hợp nhỏ.
- Giải thích được vai trò của từng loại ràng buộc trong WBC: động lực học robot, friction cone, giới hạn khớp/mô-men.
- Phân biệt được QP một tầng (weighted-sum) với Hierarchical QP, biết ưu/nhược điểm mỗi cách.
- Nêu được tên và đặc điểm của ít nhất 2 bộ giải QP thực tế dùng trong robot học (qpOASES, OSQP, HPIPM, ProxQP).

## 🧭 Vì sao cần học cái này? (bối cảnh)

Bài giảng trước (Task-space/Operational-space control) cho thấy công thức `τ = J^T·F` là một cách đẹp để quy đổi lực tác vụ sang mô-men khớp — nhưng nó là một hệ **đẳng thức** thuần túy, không có chỗ tự nhiên để nói "lực tiếp xúc chân KHÔNG ĐƯỢC vượt quá giới hạn ma sát" hay "khớp gối KHÔNG ĐƯỢC vượt quá góc tối đa". Đây chính xác là khoảng trống mà QP lấp đầy: nó biến bài toán điều khiển thành một **bài toán tối ưu có ràng buộc**, nơi vừa có thể tối thiểu hóa sai số tác vụ (giống operational-space) vừa áp được các bất đẳng thức vật lý bắt buộc. QP là "động cơ toán học" chạy bên trong hầu như mọi WBC model-based hiện đại (mỗi bước điều khiển robot thật là một lần giải QP), và là bước đệm trực tiếp trước khi học Hierarchical QP (bài giảng tiếp theo) — nơi nhiều QP được xếp chồng có thứ tự ưu tiên.

## 🧠 Trực giác

### Góc nhìn 1: "Đơn thuốc tối ưu" — vừa đạt mục tiêu, vừa không phạm luật

Hãy tưởng tượng một bác sĩ kê đơn thuốc: mục tiêu là giảm triệu chứng nhanh nhất (hàm mục tiêu), nhưng liều lượng không được vượt ngưỡng an toàn (ràng buộc bất đẳng thức) và phải tuân theo tương tác thuốc đã biết (ràng buộc đẳng thức — ví dụ tổng liều theo công thức cố định). QP trong WBC làm đúng việc đó mỗi 1-5 mili-giây: "tối thiểu hóa sai số theo dõi tác vụ (giảm triệu chứng)" trong khi "không phá vỡ động lực học robot (tương tác thuốc)" và "không vượt giới hạn lực/khớp (ngưỡng an toàn)".

**Giới hạn của loại suy này:** bác sĩ ra quyết định một lần cho một đợt điều trị; QP trong WBC phải giải LẠI toàn bộ bài toán ở MỌI bước điều khiển (không phải giải một lần rồi áp dụng mãi) — vì trạng thái robot, và do đó cả ma trận ràng buộc, thay đổi liên tục.

### Góc nhìn 2: Cái bát và viên bi — hình học của bài toán bậc hai có ràng buộc

Về mặt hình học: một hàm mục tiêu bậc hai (quadratic, dạng `‖Ax − b‖²`) trông giống một cái bát parabol lồi trong không gian nhiều chiều — có duy nhất một đáy thấp nhất (nghiệm tối ưu không ràng buộc). Các ràng buộc bất đẳng thức giống như "tường chắn" cắt vào cái bát đó — nếu đáy bát nằm trong vùng cho phép (feasible region), nghiệm tối ưu chính là đáy bát; nếu đáy bát nằm ngoài vùng cho phép, viên bi (nghiệm tối ưu có ràng buộc) sẽ "lăn" tới điểm thấp nhất CÓ THỂ đạt được mà vẫn chạm vào bức tường gần nhất — đây chính là trực giác của điều kiện KKT (nghiệm tối ưu nằm ở biên, với "áp lực" từ ràng buộc active cân bằng đúng với gradient của hàm mục tiêu).

**Giới hạn của loại suy này:** loại suy "viên bi lăn" gợi ý một quá trình động (như gradient descent), nhưng QP với bộ giải hiện đại (active-set, interior-point) không "lăn" theo thời gian thực — nó giải trực tiếp bằng đại số tuyến tính/lặp số học trong một lần gọi hàm, không mô phỏng vật lý viên bi.

## 📐 Định nghĩa chính xác

Dạng chuẩn của một bài toán QP:

```
minimize (over z)     ½ zᵀQz + cᵀz
subject to             A_eq·z = b_eq        (ràng buộc đẳng thức)
                       A_ineq·z ≤ b_ineq     (ràng buộc bất đẳng thức)
```

trong đó **z** là vector biến quyết định, **Q** là ma trận đối xứng nửa xác định dương (positive semi-definite — đảm bảo bài toán lồi, có nghiệm tối ưu toàn cục duy nhất hoặc tập nghiệm lồi).

Áp dụng cho WBC, biến quyết định thường là:

```
z = [ q̈ ; F_c ; τ ]     (gia tốc khớp, lực tiếp xúc, mô-men khớp — có thể giải đồng thời cả ba)
```

**Hàm mục tiêu** — tổng bình phương sai số theo dõi gia tốc tác vụ mong muốn, có trọng số **w_i** cho từng tác vụ i:

```
minimize   Σᵢ wᵢ · ‖Jᵢ·q̈ + J̇ᵢ·q̇ − ẍᵢ*‖²
```

**Ràng buộc đẳng thức** — phương trình động lực học robot (Newton-Euler đầy đủ, không đơn giản hóa như operational-space):

```
M(q)·q̈ + h(q,q̇) = τ + Σⱼ J_cⱼᵀ·F_cⱼ
```

**Ràng buộc bất đẳng thức** điển hình:

```
τ_min ≤ τ ≤ τ_max                                  (giới hạn mô-men động cơ)
q_min ≤ q + q̇·dt + ½q̈·dt² ≤ q_max                  (giới hạn khớp, dạng rời rạc hóa)
F_c trong friction cone: ‖F_t‖ ≤ μ·F_n, F_n ≥ 0     (ma sát Coulomb, không kéo sàn)
```

Vì `Q` ở đây là tổng các ma trận `wᵢ·Jᵢᵀ·Jᵢ` (luôn nửa xác định dương), bài toán WBC-QP luôn là một **bài toán lồi** — đây là lý do nó giải được nhanh và ổn định (không có vấn đề "kẹt ở cực tiểu địa phương" như tối ưu phi lồi).

## ⚙️ Cơ chế hoạt động — từng bước

```
┌───────────────────────────────────────────────────────────────┐
│ 1. Đo trạng thái: q, q̇ (encoder), lực tiếp xúc đo được (F/T sensor)│
└─────────────────────────────┬─────────────────────────────────┘
                               ▼
┌───────────────────────────────────────────────────────────────┐
│ 2. Với mỗi tác vụ i, tính Jᵢ, J̇ᵢ, ẍᵢ* (từ luật PD tham chiếu)      │
└─────────────────────────────┬─────────────────────────────────┘
                               ▼
┌───────────────────────────────────────────────────────────────┐
│ 3. Lắp ráp ma trận Q = Σ wᵢ·Jᵢᵀ·Jᵢ  và  c = −Σ wᵢ·Jᵢᵀ·(ẍᵢ*−J̇ᵢq̇)     │
└─────────────────────────────┬─────────────────────────────────┘
                               ▼
┌───────────────────────────────────────────────────────────────┐
│ 4. Lắp ràng buộc đẳng thức (động lực học) và bất đẳng thức        │
│    (friction cone, giới hạn khớp/mô-men) thành A_eq, A_ineq       │
└─────────────────────────────┬─────────────────────────────────┘
                               ▼
┌───────────────────────────────────────────────────────────────┐
│ 5. Gọi bộ giải QP (qpOASES/OSQP/HPIPM/ProxQP) → z* = [q̈*,F_c*,τ*] │
└─────────────────────────────┬─────────────────────────────────┘
                               ▼
┌───────────────────────────────────────────────────────────────┐
│ 6. Gửi τ* xuống động cơ (torque control, thường 200Hz–1kHz)       │
└─────────────────────────────┬─────────────────────────────────┘
                               │
                               └──── lặp lại từ bước 1 mỗi chu kỳ
```

Khác biệt lớn nhất so với operational-space control thuần túy: bước 5 không còn là một phép tính giải tích đóng (closed-form) như `F=Λẍ*+μ`, mà là một **lần gọi thuật toán tối ưu số** — chậm hơn về lý thuyết, nhưng bù lại xử lý được đồng thời nhiều tác vụ + ràng buộc bất đẳng thức trong CÙNG một bài toán, điều mà công thức đóng không làm được.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh họa, số tự chọn để dễ hình dung — đơn giản hóa về một biến vô hướng để có thể giải bằng tay, không dùng ma trận đầy đủ.)*

Giả sử ta đơn giản hóa cực độ: robot có 1 "biến điều khiển" `u` (thay cho q̈ đầy đủ), có 2 tác vụ đối lập nhau:

- Tác vụ 1 (ưu tiên thấp hơn, trọng số w₁=1): muốn `u = 5`.
- Tác vụ 2 (ưu tiên cao hơn, trọng số w₂=4): muốn `u = 2`.
- Ràng buộc bất đẳng thức vật lý: `u ≤ 3` (ví dụ giới hạn mô-men động cơ).

**Bước 1 — hàm mục tiêu weighted-sum (QP một tầng):**

```
minimize f(u) = w₁(u−5)² + w₂(u−2)² = (u−5)² + 4(u−2)²
```

**Bước 2 — tìm cực tiểu không ràng buộc** bằng đạo hàm = 0:

```
f'(u) = 2(u−5) + 8(u−2) = 2u−10+8u−16 = 10u−26 = 0
⟹ u* = 2.6
```

**Bước 3 — kiểm tra ràng buộc:** `u* = 2.6 ≤ 3` — thỏa mãn ràng buộc, nên đây chính là nghiệm tối ưu của QP có ràng buộc (ràng buộc không "active", không cần điều chỉnh thêm).

**Bước 4 — thử lại với trọng số khác để thấy ràng buộc active:** nếu đổi w₂ = 20 (tác vụ 2 áp đảo mạnh hơn):

```
f'(u) = 2(u−5) + 40(u−2) = 2u−10+40u−80 = 42u−90 = 0
⟹ u* = 90/42 ≈ 2.14
```

Vẫn ≤ 3, ràng buộc vẫn không active trong trường hợp này. Để minh họa ràng buộc **active thật sự**, đổi ràng buộc thành `u ≤ 2.0` (chặt hơn nghiệm không ràng buộc u*=2.6 hoặc 2.14):

```
Nghiệm không ràng buộc (2.6 hoặc 2.14) đều > 2.0 → VI PHẠM ràng buộc
⟹ nghiệm QP có ràng buộc phải nằm ĐÚNG trên biên: u* = 2.0
```

Đây là bản chất của "active constraint" trong QP: khi cực tiểu tự do nằm ngoài vùng khả thi, nghiệm tối ưu bị "ép" về đúng biên gần nhất — về mặt KKT, tại điểm này gradient của hàm mục tiêu song song (và ngược hướng phù hợp) với gradient của ràng buộc, với một hệ số Lagrange dương biểu diễn "áp lực" ép vào biên.

**Diễn giải bằng KKT (Karush-Kuhn-Tucker):** trong trường hợp ràng buộc active `u=2.0`, điều kiện tối ưu có ràng buộc yêu cầu tồn tại một hệ số Lagrange `λ ≥ 0` sao cho `f'(u*) + λ = 0` (đạo hàm hàm mục tiêu cộng "áp lực" từ ràng buộc bằng không). Với w₂=4: `f'(2.0) = 2(2−5)+8(2−2) = −6`, nên `λ = 6 > 0` — dấu dương của λ xác nhận ràng buộc đang thực sự "đẩy" nghiệm ra khỏi điểm mà nó muốn tới (tức ràng buộc active hợp lệ). Nếu tính ra λ âm, đó là dấu hiệu ràng buộc đó KHÔNG nên active — bộ giải QP thực tế (active-set method) dùng chính logic này để lặp thử/loại các ràng buộc active ở mỗi vòng lặp cho tới khi mọi λ đều không âm.

**Vai trò của biến chùng (slack variables) trong thực tế:** một chi tiết quan trọng khi triển khai QP-WBC thật là không phải mọi ràng buộc đều nên "cứng" tuyệt đối. Ví dụ ràng buộc động lực học `M·q̈+h=τ+J_cᵀF_c` về lý thuyết là đẳng thức chính xác, nhưng do sai số mô hình (M, h không bao giờ khớp 100% với robot thật) và nhiễu số học, ép nó "cứng tuyệt đối" có thể khiến QP thường xuyên infeasible. Giải pháp thực dụng: thêm một biến chùng `s` nhỏ vào ràng buộc (`M·q̈+h = τ+J_cᵀF_c + s`) và phạt `s` rất nặng trong hàm mục tiêu (`+ w_s·‖s‖²` với `w_s` cực lớn) — về bản chất QP vẫn cố "tôn trọng gần như tuyệt đối" ràng buộc đó, nhưng không sụp đổ hoàn toàn khi có sai số nhỏ không tránh khỏi.

**Ví dụ thứ hai — QP hai biến với ràng buộc bất đẳng thức dạng vector (gần với thực tế WBC hơn):** *(ví dụ minh họa, số tự chọn)*. Giả sử biến quyết định là `z = [z₁, z₂]ᵀ` (ví dụ gia tốc của 2 khớp), với một tác vụ duy nhất mong muốn đạt gia tốc tác vụ `ẍ* = 4` thông qua quan hệ `ẍ = z₁ + z₂` (một Jacobian đơn giản hàng-vector `J=[1,1]`), và ràng buộc bất đẳng thức riêng cho từng khớp: `z₁ ≤ 1.5` và `z₂ ≤ 1.5` (giới hạn mô-men/gia tốc từng động cơ).

Hàm mục tiêu: `minimize (z₁+z₂−4)²`. Không ràng buộc, có vô số nghiệm thỏa `z₁+z₂=4` (ví dụ z₁=2,z₂=2) — đây là bài toán dư biến (underdetermined), thường gặp thật trong WBC vì số bậc tự do khớp luôn nhiều hơn số chiều tác vụ. Để chọn MỘT nghiệm cụ thể, QP-WBC thực tế luôn thêm một số hạng điều hòa nhỏ (regularization) vào hàm mục tiêu, ví dụ `+ ε·(z₁²+z₂²)` với ε rất nhỏ — về bản chất là "trong số các nghiệm đạt tác vụ chính hoàn hảo, chọn nghiệm có z nhỏ nhất (tốn ít năng lượng/mô-men nhất)". Với ε→0⁺, nghiệm điều hòa tối thiểu-chuẩn (minimum-norm) đối xứng cho bài toán này là `z₁=z₂=2` — nhưng z₂=2 > 1.5, vi phạm ràng buộc! Áp dụng ràng buộc `z₂≤1.5` làm active: đặt `z₂=1.5`, khi đó để giữ tác vụ chính tốt nhất có thể, `z₁` được đẩy lên bù: nghiệm mới `z₁=2.5` — nhưng lúc này lại vi phạm `z₁≤1.5`! Vậy CẢ HAI ràng buộc đều active: `z₁=z₂=1.5`, tổng đạt được `ẍ = 3.0` — **thấp hơn mục tiêu ẍ*=4**, nghĩa là QP chấp nhận một sai số theo dõi tác vụ (residual = 4−3=1) vì không còn cách nào khác để tôn trọng cả hai giới hạn khớp. Đây là minh họa rất thực tế: khi ràng buộc vật lý quá chặt, QP-WBC **hy sinh độ chính xác theo dõi tác vụ** (chứ không vi phạm ràng buộc cứng) — hành vi này khác biệt rõ với null-space projection thuần túy (vốn giả định luôn đạt được tác vụ chính hoàn hảo).

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | QP một tầng (weighted-sum) | Operational-space control (giải tích) | Hierarchical QP (bài riêng) |
|---|---|---|---|
| Xử lý ràng buộc bất đẳng thức | Tự nhiên, trực tiếp | Không tự nhiên | Tự nhiên, từng cấp |
| Nhiều tác vụ đồng thời | Có, qua trọng số cố định | Có, qua null-space thủ công | Có, qua thứ tự ưu tiên cứng |
| Rủi ro chính | Phải tay chỉnh trọng số — dễ vỡ khi tình huống mới | Khó mở rộng khi nhiều ràng buộc | Phức tạp hơn khi lập trình/giải |
| Tốc độ giải mỗi bước | Nhanh (1 QP) | Rất nhanh (đóng, không lặp số) | Chậm hơn (nhiều QP tuần tự) |
| Đảm bảo ưu tiên tuyệt đối | Không — trọng số không đảm bảo 100% | Không có khái niệm ưu tiên định lượng | Có — cấp cao không bao giờ bị hy sinh |
| Công cụ giải phổ biến | qpOASES, OSQP, HPIPM, ProxQP | Không cần bộ giải (đóng) | HQP solver chuyên dụng (eiquadprog, HiQP) |

**Khi nào dùng cái nào:** QP một tầng phù hợp khi số tác vụ ít và trọng số có thể tinh chỉnh thủ công tốt (ví dụ robot công nghiệp cố định); humanoid hai chân với tác vụ thăng bằng CHẮC CHẮN phải quan trọng hơn tác vụ tay trong MỌI tình huống thường chuyển sang Hierarchical QP để có đảm bảo cứng.

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "QP luôn tìm được nghiệm, chỉ cần đưa đủ ràng buộc vào."** Sai — nếu các ràng buộc mâu thuẫn nhau (ví dụ giới hạn khớp và ràng buộc động lực học không thể đồng thời thỏa mãn ở trạng thái hiện tại), bài toán trở nên **infeasible** (vô nghiệm), và bộ giải QP sẽ trả về lỗi thay vì một nghiệm gần đúng. Trong thực tế, kỹ sư WBC thường phải thêm "biến chùng" (slack variables) có phạt (penalty) lớn cho một số ràng buộc mềm để tránh infeasibility hoàn toàn, đặc biệt với ràng buộc động lực học đôi khi được nới thành "gần đúng" thay vì đẳng thức cứng tuyệt đối.

2. **Hiểu nhầm: "Trọng số wᵢ càng lớn thì tác vụ đó càng được ưu tiên tuyệt đối, giống Hierarchical QP."** Sai một phần — trọng số lớn hơn chỉ làm tác vụ đó được nghiệm tối ưu "chiều theo" nhiều hơn, nhưng KHÔNG đảm bảo tuyệt đối 100% giống HQP. Ví dụ trong walkthrough ở trên, dù w₂=20 ≫ w₁=1, nghiệm vẫn không hoàn toàn bằng 2 (mục tiêu của tác vụ 2) — nó vẫn bị "kéo" một chút bởi tác vụ 1. Chỉ có giới hạn `w₂ → ∞` mới tiệm cận hành vi ưu tiên cứng của HQP.

3. **Hiểu nhầm: "QP giải một lần rồi dùng lại nhiều bước để tiết kiệm tính toán."** Sai — vì `J`, `M`, `h` đều phụ thuộc cấu hình `q, q̇` hiện tại, QP phải được lắp ráp và giải lại ở **mọi bước điều khiển** (thường 200Hz-1kHz). Đây là lý do tốc độ bộ giải QP (qpOASES/OSQP/HPIPM) là yếu tố sống còn cho khả năng chạy real-time trên robot thật.

4. **Hiểu nhầm: "Nếu QP không đạt được tác vụ chính hoàn hảo (residual > 0 như ở ví dụ hai biến trên), nghĩa là công thức bị lỗi."** Sai — đây là hành vi ĐÚNG và MONG MUỐN của QP khi ràng buộc vật lý (giới hạn khớp/mô-men/ma sát) thực sự không cho phép đạt tác vụ hoàn hảo. QP luôn ưu tiên tôn trọng ràng buộc cứng (nếu được khai báo là hard constraint) hơn là đạt chính xác tuyệt đối hàm mục tiêu — residual dương ở đây phản ánh đúng giới hạn vật lý thật của robot, không phải lỗi lắp ráp bài toán.

## 🏗️ Ví dụ minh họa trong dự án này

Trong 20 năm trước SONIC (theo README của thư mục này), WBC model-based dựa trên QP chính là công nghệ chuẩn cho robot humanoid thương mại (ví dụ các dòng robot của Boston Dynamics, ANYmal của ETH). Dự án này (theo `NOI-DUNG-CHI-TIET.md` mục 2.1) chọn hướng learning-based (RL + motion tracking) thay vì QP-WBC cổ điển — nhưng hiểu QP là nền tảng quan trọng để hiểu **TẠI SAO** cộng đồng chuyển sang RL: ba hạn chế của WBC cổ điển (phải mô hình hóa tường minh động lực học, phải viết tay hàm mục tiêu/ràng buộc, khó tổng quát hóa) chính là hạn chế cố hữu của MỌI công thức QP, dù có tinh vi đến đâu — vì QP luôn cần `M(q)`, `h(q,q̇)` chính xác và các ràng buộc được liệt kê tường minh trước, trong khi policy RL "học" các mối quan hệ đó từ dữ liệu mà không cần công thức tường minh.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **Khảo sát toàn diện về bộ giải QP thời gian thực cho robot chân (2025):** "Real-Time QP Solvers: A Concise Review and Practical Guide Towards Legged Robots" (arXiv:2510.21773, 2025) tổng hợp và so sánh các bộ giải QP hiện đại (active-set, interior-point, ADMM-based như OSQP/ProxQP) theo tiêu chí tốc độ, độ ổn định số, khả năng "warm-start" (khởi tạo từ nghiệm bước trước để hội tụ nhanh hơn) — trực tiếp phục vụ WBC/legged robot chạy real-time ở tần số cao. ([arXiv:2510.21773](https://arxiv.org/pdf/2510.21773))

2. **Đơn giản hóa QP-WBC bằng centroidal dynamics để giảm chi phí tính toán:** "Efficient Computation of Whole-Body Control Utilizing Simplified Whole-Body Dynamics via Centroidal Dynamics" (arXiv:2409.10903, 2024) đề xuất dùng động lực học trọng tâm (centroidal dynamics) rút gọn thay vì động lực học đầy đủ nhiều khớp trong một số thành phần của QP, giảm đáng kể chi phí tính toán mỗi bước mà vẫn giữ được độ chính xác cần thiết cho các tác vụ chính. ([arXiv:2409.10903](https://arxiv.org/pdf/2409.10903))

3. **Tích hợp cảm biến xúc giác (tactile) trực tiếp vào QP-WBC:** nghiên cứu công bố trên *Advanced Intelligent Systems* (Armleder et al., 2025) đưa dữ liệu lực/khoảng cách từ da nhân tạo (artificial skin) trực tiếp thành ràng buộc/tác vụ bổ sung trong QP-WBC thời gian thực, cho phép robot phản ứng an toàn khi có tiếp xúc ngoài kế hoạch (ví dụ va chạm bất ngờ với người) — mở rộng QP-WBC ra khỏi phạm vi "chỉ tiếp xúc chân với sàn" truyền thống. ([Wiley — Advanced Intelligent Systems](https://advanced.onlinelibrary.wiley.com/doi/10.1002/aisy.202500149))

4. **Xu hướng chung:** phần lớn hệ thống WBC model-based hiện đại (2024-2026) vẫn dùng QP làm "tầng thực thi tức thời" (instantaneous execution layer) bên dưới một tầng MPC hoạch định trước (xem bài giảng "MPC cho dáng đi") — kiến trúc hai tầng MPC+QP-WBC này được xác nhận là "state-of-practice" phổ biến nhất cho locomotion model-based tính đến 2025 theo khảo sát "Humanoid Locomotion and Manipulation: Current Progress and Challenges" (arXiv:2501.02116).

## ❓ Câu hỏi tự kiểm tra

1. Vì sao ma trận `Q` trong QP của WBC luôn nửa xác định dương (positive semi-definite)?
<details><summary>Gợi ý đáp án</summary>Vì Q là tổng các số hạng dạng `wᵢ·Jᵢᵀ·Jᵢ` — mỗi số hạng `Jᵢᵀ·Jᵢ` luôn nửa xác định dương (dạng Gram matrix, `zᵀJᵢᵀJᵢz = ‖Jᵢz‖² ≥ 0` với mọi z), và tổng của các ma trận nửa xác định dương (trọng số dương) vẫn nửa xác định dương.</details>

2. Trong ví dụ tính tay, nếu cả hai tác vụ có cùng trọng số w₁=w₂=1 (thay vì 1 và 4), nghiệm không ràng buộc u* sẽ là bao nhiêu?
<details><summary>Gợi ý đáp án</summary>f'(u)=2(u−5)+2(u−2)=4u−14=0 ⟹ u*=3.5 — đúng giữa hai mục tiêu 5 và 2 vì trọng số bằng nhau.</details>

3. Vì sao dùng trọng số wᵢ rất lớn cho một tác vụ KHÔNG hoàn toàn tương đương với Hierarchical QP?
<details><summary>Gợi ý đáp án</summary>Vì QP một tầng luôn là một sự thỏa hiệp (trade-off) liên tục giữa các tác vụ tùy theo tỷ lệ trọng số — không có "ngưỡng cứng" nào đảm bảo tác vụ ưu tiên thấp không ảnh hưởng dù chỉ một chút tới tác vụ ưu tiên cao, trong khi HQP giải tuần tự và ràng buộc cấp dưới trong null-space đúng nghĩa của cấp trên.</details>

4. Nếu ràng buộc bất đẳng thức trong QP mâu thuẫn với ràng buộc đẳng thức (động lực học), điều gì xảy ra với bộ giải QP?
<details><summary>Gợi ý đáp án</summary>Bài toán trở nên infeasible — bộ giải QP không tìm được nghiệm thỏa mãn tất cả ràng buộc, thường trả về lỗi hoặc cần cơ chế "soft constraint"/slack variable để tránh dừng đột ngột.</details>

5. Tại sao "warm-start" (dùng nghiệm bước trước làm điểm khởi tạo) lại quan trọng với QP-WBC chạy ở 1kHz?
<details><summary>Gợi ý đáp án</summary>Vì trạng thái robot thay đổi rất ít giữa hai bước liên tiếp ở tần số cao, nghiệm QP bước trước thường gần với nghiệm bước sau — khởi tạo từ đó giúp thuật toán (đặc biệt active-set) hội tụ nhanh hơn nhiều so với khởi tạo từ đầu, điều cốt yếu để giữ được deadline thời gian thực.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể:** với ví dụ 1-biến ở trên, đổi ràng buộc thành `u ≥ 4` (thay vì `u ≤ 3`), và giữ trọng số gốc (w₁=1, w₂=4, mục tiêu 5 và 2). Xác định nghiệm không ràng buộc, kiểm tra nó có vi phạm ràng buộc mới không, và nếu có, tìm nghiệm QP có ràng buộc.
2. **Đọc code/tài liệu thật:** tìm tài liệu hoặc mã nguồn mở của một bộ giải QP thực tế (OSQP hoặc qpOASES) — đọc phần "warm start" hoặc "active set" trong tài liệu chính thức, ghi lại 2-3 câu tóm tắt cách họ triển khai ý tưởng "khởi tạo từ nghiệm trước" đã nói ở câu hỏi tự kiểm tra #5.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

Quadratic Programming (QP) biến bài toán WBC từ một phép tính đóng (operational-space control thuần túy) thành một bài toán tối ưu lồi có ràng buộc: tối thiểu hóa tổng bình phương sai số theo dõi nhiều tác vụ đồng thời (hàm mục tiêu bậc hai), trong khi bắt buộc thỏa mãn động lực học robot (ràng buộc đẳng thức) và các giới hạn vật lý như friction cone, giới hạn khớp/mô-men (ràng buộc bất đẳng thức) — điều mà công thức null-space cổ điển không xử lý tự nhiên được. Mỗi bước điều khiển robot thật (thường 200Hz-1kHz) là một lần giải QP mới, dùng các bộ giải chuyên dụng như qpOASES/OSQP/HPIPM để đạt tốc độ real-time. QP một tầng vẫn cần tay chỉnh trọng số giữa các tác vụ — hạn chế này chính là động lực dẫn tới Hierarchical QP (bài giảng tiếp theo), nơi thứ tự ưu tiên được đảm bảo cứng thay vì chỉ là một sự thỏa hiệp theo trọng số.
