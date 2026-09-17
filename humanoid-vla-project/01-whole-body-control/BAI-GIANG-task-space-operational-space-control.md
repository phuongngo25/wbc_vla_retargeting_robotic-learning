# Bài giảng: Task-space control / Operational-space control là gì

*(Thuộc mảng: Whole-Body Control (WBC))*

## 🎯 Mục tiêu bài học

- Giải thích được sự khác biệt giữa joint space và task space, và vì sao humanoid buộc phải nghĩ bằng task space.
- Viết được quan hệ vi phân `ẋ = J·q̇` và giải thích ý nghĩa vật lý của Jacobian `J`.
- Suy ra được quan hệ lực-mô-men `τ = J^T·F` từ nguyên lý công ảo (virtual work), không chỉ nhớ công thức.
- Tính được ma trận quán tính không gian tác vụ `Λ(x) = (J·M⁻¹·J^T)⁻¹` cho một ví dụ số đơn giản (robot 2 khớp phẳng).
- Giải thích được vai trò của phép chiếu null-space `(I − J^T·J^#)` trong việc thực hiện tác vụ phụ mà không phá tác vụ chính.
- Phân biệt được operational-space control (Khatib 1987) với các cách tiếp cận thay thế (joint-space PD, QP-based WBC) và biết khi nào mỗi cách phù hợp.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Đây là khái niệm nền tảng nhất trong toàn bộ mảng Whole-Body Control (WBC) — mọi kỹ thuật khác trong thư mục này (QP, Hierarchical QP, thậm chí cả cách một policy RL học "đuổi theo" một target pose) ngầm định đứng trên vai của ý tưởng: **mục tiêu điều khiển của humanoid không phải là góc khớp, mà là các đại lượng ở không gian tác vụ** (vị trí tay, trọng tâm, hướng nhìn...). Trước khi có thể hiểu vì sao WBC cần giải một bài toán tối ưu (QP) hay vì sao cần phân tầng ưu tiên (HQP), bạn cần hiểu bài toán gốc mà những kỹ thuật đó đang giải quyết: **làm sao biến một mục tiêu vật lý (ví dụ "bàn tay tới điểm P") thành mô-men khớp cụ thể**, trong khi robot có hàng chục bậc tự do dư thừa.

## 🧠 Trực giác

### Góc nhìn 1: Người lái xe cần cùi (marionette) — bạn muốn điều khiển "đầu ngón tay của con rối", không phải "từng sợi dây"

Hãy tưởng tượng một con rối (marionette) có hàng chục sợi dây, mỗi sợi nối tới một khớp. Nếu bạn phải nghĩ "kéo dây 1 thêm 3cm, dây 2 thêm 1cm, dây 3 giữ nguyên..." để tay con rối chạm vào một điểm cụ thể, đó là **joint-space thinking** — cực kỳ khó vì bạn phải tự tính ngược từ "tôi muốn tay ở đâu" ra "mỗi sợi dây phải kéo bao nhiêu". Operational-space control giống như có một bộ điều khiển thông minh đứng giữa bạn và các sợi dây: bạn chỉ cần nói "tay tới điểm P với lực F", còn bộ điều khiển tự tính ra cách kéo từng sợi dây tương ứng.

**Giới hạn của loại suy này:** con rối không có khái niệm "động lực học" (quán tính, lực quán tính khi di chuyển nhanh) — nó chỉ là hình học tĩnh. Operational-space control còn phải xử lý phần "động" (quán tính, Coriolis) mà loại suy con rối không thể hiện được.

### Góc nhìn 2: Đổi "đơn vị đo" bằng ma trận biến đổi — giống đổi hệ tọa độ trong vật lý

Ở góc độ toán học thuần túy hơn: hãy nghĩ Jacobian `J` như một "bộ chuyển đổi đơn vị" giữa hai hệ tọa độ — một hệ là không gian khớp (q, đơn vị: radian/khớp), một hệ là không gian tác vụ (x, đơn vị: mét ở đầu công tác). Giống như đổi từ độ F sang độ C có công thức tuyến tính cố định tại một thời điểm, `J` cho biết "tại cấu hình robot hiện tại, một thay đổi nhỏ ở khớp tương ứng với thay đổi nhỏ nào ở tác vụ". Vì `J` biến đổi vận tốc theo một chiều, chuyển vị `J^T` của nó biến đổi lực theo chiều ngược lại — đây thuần túy là hệ quả của **nguyên lý công ảo** (virtual work: công sinh ra phải bằng nhau dù tính ở không gian nào).

**Giới hạn của loại suy này:** khác với đổi độ F↔C (hệ số cố định vĩnh viễn), Jacobian `J` **thay đổi theo cấu hình robot** — nó chỉ đúng cục bộ (tuyến tính hóa quanh q hiện tại), không phải một phép biến đổi tuyến tính cố định toàn cục. Đây là lý do operational-space control phải tính lại `J` liên tục ở mỗi bước điều khiển.

## 📐 Định nghĩa chính xác

Gọi **x** ∈ ℝᵐ là tọa độ tác vụ (ví dụ vị trí 3D đầu công tác, m = 3), **q** ∈ ℝⁿ là tọa độ khớp tổng quát (n = số bậc tự do, với humanoid thường n ≫ m). Quan hệ vi phân bậc nhất:

```
ẋ = J(q)·q̇        (J ∈ ℝ^(m×n) là Jacobian của tác vụ tại cấu hình q)
```

Đạo hàm thêm một bậc để có quan hệ gia tốc:

```
ẍ = J·q̈ + J̇·q̇
```

Theo nguyên lý công ảo, nếu tác vụ chịu một lực mong muốn **F** ∈ ℝᵐ (lực/mô-men ảo tại đầu công tác), mô-men khớp tương ứng để tạo ra đúng lực đó (không tạo thêm chuyển động phụ) là:

```
τ = J^T·F
```

Phương trình động lực học khớp tổng quát của robot (Newton-Euler dạng gộp):

```
M(q)·q̈ + h(q, q̇) = τ
```

trong đó **M** là ma trận quán tính khớp (n×n), **h** gộp các số hạng Coriolis/ly tâm/trọng lực. Khatib (1987) định nghĩa **ma trận quán tính không gian tác vụ**:

```
Λ(x) = (J·M⁻¹·J^T)⁻¹        (m×m)
```

và **lực suy rộng tác vụ tương đương**:

```
μ = Λ·J·M⁻¹·h − Λ·J̇·q̇
```

sao cho động lực học không gian tác vụ được viết gọn thành:

```
Λ·ẍ + μ = F
```

— tức là một phương trình Newton bậc hai đơn giản (giống điều khiển một khối lượng đơn `Λ`), tách bạch hoàn toàn khỏi cấu trúc khớp phức tạp bên dưới. Với robot dư bậc tự do (n > m, humanoid luôn ở trường hợp này), mô-men khớp tổng quát cho phép thêm một tác vụ phụ **τ₀** không ảnh hưởng đến tác vụ chính:

```
τ = J^T·F + N^T·τ₀,        N = (I − J^T·J^#)
```

trong đó **J^#** là nghịch đảo suy rộng có trọng số quán tính (`J^# = M⁻¹·J^T·Λ`), và **N** là **ma trận chiếu null-space** — chiếu bất kỳ vector mô-men nào vào không gian con không gây gia tốc tác vụ chính.

**Ghi chú quan trọng — vì sao J^# phải là nghịch đảo có trọng số quán tính (không phải Moore-Penrose thường):** nghịch đảo suy rộng `J^#` trong công thức null-space **không phải** nghịch đảo Moore-Penrose thông thường (`J^+ = J^T(JJ^T)⁻¹`), mà là **nghịch đảo có trọng số quán tính** `J^# = M⁻¹J^TΛ`. Lý do: nếu dùng Moore-Penrose thường, phép chiếu null-space chỉ "trực giao" theo nghĩa hình học Euclid thuần túy ở không gian khớp — nhưng động lực học thực của robot bị chi phối bởi quán tính `M`, không phải hình học Euclid. Dùng `J^#` có trọng số `M` đảm bảo phép chiếu là **"dynamically consistent"** (nhất quán động lực học): tác vụ phụ thực thi ở null-space sẽ **không tạo ra bất kỳ lực phản lực nào** tác động lên tác vụ chính, kể cả khi tính đến quán tính — đây là điều Moore-Penrose thường không đảm bảo được. Đây chính là thuật ngữ "dynamically consistent null-space projection" xuất hiện xuyên suốt các paper của Khatib.

## ⚙️ Cơ chế hoạt động — từng bước

```
┌─────────────────────────────────────────────────────────────┐
│  1. Đo trạng thái hiện tại: q, q̇ (encoder khớp)                │
└───────────────────────────┬─────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  2. Tính Jacobian J(q), J̇(q,q̇) tại cấu hình hiện tại           │
│     (forward kinematics + đạo hàm)                             │
└───────────────────────────┬─────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  3. Tính Λ(x) = (J·M⁻¹·J^T)⁻¹  và  μ  từ M(q), h(q,q̇)           │
└───────────────────────────┬─────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  4. Chọn gia tốc tác vụ mong muốn ẍ* (thường từ luật PD:         │
│     ẍ* = ẍ_ref + Kp·(x_ref − x) + Kd·(ẋ_ref − ẋ))                │
└───────────────────────────┬─────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  5. Tính lực tác vụ cần thiết: F = Λ·ẍ* + μ                     │
└───────────────────────────┬─────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  6. Quy đổi sang mô-men khớp: τ = J^T·F + N^T·τ₀ (tác vụ phụ)   │
└───────────────────────────┬─────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  7. Gửi τ xuống động cơ (torque control loop, thường >1kHz)     │
└─────────────────────────────────────────────────────────────┘
                             │
                             └──── lặp lại từ bước 1 mỗi chu kỳ
```

Điểm mấu chốt: bước 5-6 là "bản dịch" hai chiều — bước 5 làm việc hoàn toàn trong không gian tác vụ (như điều khiển một khối lượng đơn `Λ`), bước 6 mới quy đổi trở lại không gian khớp. Toàn bộ độ phức tạp của cấu trúc động học nhiều khớp bị "giấu" trong `J`, `Λ`, `μ` — người thiết kế bộ điều khiển tác vụ không cần nghĩ về từng khớp riêng lẻ nữa.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh họa, số tự chọn để dễ hình dung — dùng cánh tay phẳng 2 khớp, bỏ qua trọng lực để đơn giản hóa phép tính.)*

Robot tay máy phẳng 2 khớp (2-link planar arm), mỗi khâu dài L₁ = L₂ = 0.5 m, khối lượng tập trung ở đầu mỗi khâu m₁ = m₂ = 1 kg. Tại thời điểm xét, θ₁ = 0°, θ₂ = 90° (khâu 2 vuông góc khâu 1).

**Bước 1 — Jacobian vị trí đầu công tác** (công thức chuẩn tay máy phẳng 2 khớp):

```
J = [ −L₁sinθ₁ − L₂sin(θ₁+θ₂)     −L₂sin(θ₁+θ₂) ]
    [  L₁cosθ₁ + L₂cos(θ₁+θ₂)      L₂cos(θ₁+θ₂) ]
```

Thay θ₁ = 0, θ₂ = 90°: sin(θ₁)=0, cos(θ₁)=1, sin(θ₁+θ₂)=sin90°=1, cos(θ₁+θ₂)=cos90°=0.

```
J = [ 0 − 0.5·1     −0.5·1 ]   =  [ −0.5   −0.5 ]
    [ 0.5·1 + 0.5·0   0.5·0 ]      [  0.5    0   ]
```

**Bước 2 — giả sử ma trận quán tính khớp rút gọn (đơn giản hóa, chỉ để minh họa cơ chế, không phải công thức Lagrangian đầy đủ):**

```
M ≈ [ 1.5   0.25 ]      M⁻¹ ≈ [ 0.72   −0.72 ]
    [ 0.25  0.5  ]              [ −0.72   2.15 ]
```

**Bước 3 — tính `J·M⁻¹·J^T`** (nhân ma trận 2×2 · 2×2 · 2×2, kết quả là ma trận 2×2):

Trước hết `M⁻¹·J^T`:

```
J^T = [ −0.5    0.5 ]
      [ −0.5    0   ]

M⁻¹·J^T ≈ [ 0.72·(−0.5)+(−0.72)·(−0.5)   0.72·0.5+(−0.72)·0 ]   = [ 0.0    0.36 ]
           [ −0.72·(−0.5)+2.15·(−0.5)     −0.72·0.5+2.15·0  ]     [ −0.715  −0.36]
```

Rồi `J·(M⁻¹·J^T)`:

```
Λ⁻¹ = J·M⁻¹·J^T ≈ [ −0.5·0.0+(−0.5)·(−0.715)    −0.5·0.36+(−0.5)·(−0.36) ]
                   [  0.5·0.0+0·(−0.715)          0.5·0.36+0·(−0.36)     ]

      ≈ [ 0.358    0.0 ]
        [ 0.0      0.18 ]
```

**Bước 4 — nghịch đảo để ra Λ:**

```
Λ = (J·M⁻¹·J^T)⁻¹ ≈ [ 2.79   0    ]
                     [ 0      5.56 ]
```

**Diễn giải:** đây là "khối lượng hiệu dụng" mà đầu công tác cảm nhận theo từng trục x, y — trục x "nhẹ" hơn (2.79) so với trục y (5.56) tại chính cấu hình này, nghĩa là cùng một lực F tác dụng theo x sẽ tạo gia tốc lớn hơn theo x so với theo y. Nếu muốn đầu công tác đạt gia tốc mong muốn `ẍ* = [1, 0]ᵀ m/s²` (bỏ qua μ vì không có trọng lực/Coriolis trong ví dụ này), lực tác vụ cần thiết là `F = Λ·ẍ* ≈ [2.79, 0]ᵀ N`, và mô-men khớp tương ứng `τ = J^T·F ≈ [−0.5·2.79, −0.5·2.79]ᵀ = [−1.395, −1.395]ᵀ N·m`.

**Kiểm tra chéo bằng cảm nhận vật lý:** với cấu hình θ₂=90° (khâu 2 vuông góc khâu 1, hướng thẳng đứng), trục x là hướng "vươn ngang" — chỉ cần xoay khớp 1 một chút là đầu công tác di chuyển đáng kể theo x, nên "cảm giác nhẹ" (Λ nhỏ = dễ gia tốc) là hợp lý về mặt trực giác. Ngược lại trục y (hướng "vươn lên") đòi hỏi phối hợp cả hai khớp phức tạp hơn để tạo cùng một lượng chuyển vị, nên "nặng" hơn (Λ lớn hơn). Đây là lý do tại sao dùng trực tiếp một hệ số PD cố định cho cả hai trục x, y (bỏ qua Λ) sẽ khiến robot phản ứng nhanh/chậm khác nhau tùy hướng — đúng như sai lầm #2 ở mục dưới.

**Ví dụ số thứ hai — null-space projection:** giả sử ngoài tác vụ chính (đầu công tác), ta muốn thêm tác vụ phụ "giữ khớp 2 gần góc 90° nhất có thể" (ví dụ để tránh giới hạn khớp), với mô-men phụ mong muốn (trước khi chiếu) `τ₀ = [0, −2]ᵀ N·m` (mô-men âm ở khớp 2 để kéo về phía góc mong muốn). Ma trận chiếu null-space `N = I − J^T·J^#` — vì J ở đây có hạng đầy đủ (2 hàng độc lập tuyến tính, m=n=2 trong ví dụ 2 khớp), null-space thực chất có số chiều 0, nghĩa là **không còn dư bậc tự do nào cho tác vụ phụ** — mọi τ₀ đưa vào sẽ bị chiếu về gần 0 (N ≈ ma trận không). Đây chính là lý do trong thực tế, null-space projection chỉ thực sự "có chỗ để dùng" khi robot có **nhiều khớp hơn số chiều tác vụ chính** (n > m) — với humanoid (n ≈ 20-30, một tác vụ tay chỉ chiếm m=6), null-space còn dư rất nhiều chiều cho các tác vụ phụ như giữ tư thế, tránh giới hạn khớp, hay theo dõi tác vụ ưu tiên thấp hơn.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Operational-space control (Khatib) | Joint-space PD control | QP-based WBC (xem bài riêng) |
|---|---|---|---|
| Mục tiêu điều khiển | Trực tiếp ở x (vị trí/lực tác vụ) | Trực tiếp ở q (góc khớp) | Nhiều tác vụ + ràng buộc đồng thời |
| Cần inverse kinematics tường minh? | Không — dùng J, Λ trực tiếp | Có (tính trước offline/online) | Không, giải trong 1 bài toán tối ưu |
| Xử lý dư bậc tự do | Qua null-space projection | Không có khái niệm này | Qua trọng số/ràng buộc trong QP |
| Xử lý ràng buộc bất đẳng thức (giới hạn khớp, ma sát) | Khó, không tự nhiên | Khó | Tự nhiên — là một phần bài toán QP |
| Nhiều tác vụ đồng thời có ưu tiên cứng | Có (qua null-space lồng nhau, thủ công) | Không hỗ trợ | Có, đặc biệt tốt với HQP |
| Độ phức tạp tính toán mỗi bước | Trung bình (vài phép nhân ma trận) | Thấp | Cao hơn (giải QP) |
| Vai trò lịch sử | Nền tảng lý thuyết (1987) | Cổ điển nhất | Chuẩn công nghiệp hiện đại cho WBC model-based |

**Khi nào dùng cái nào:** operational-space control thuần túy (không QP) phù hợp cho tay máy đơn, ít ràng buộc bất đẳng thức phức tạp; humanoid hiện đại hầu như luôn dùng phiên bản mở rộng thành QP/HQP (xem hai bài giảng riêng) vì cần xử lý đồng thời nhiều tác vụ + ràng buộc tiếp xúc chân + giới hạn động cơ.

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "Jacobian là ma trận cố định của robot, tính một lần là dùng mãi."** Sai — `J(q)` phụ thuộc cấu hình khớp hiện tại, phải tính lại ở MỌI bước điều khiển (thường >200Hz). Nhầm lẫn này dẫn đến bug kinh điển: bộ điều khiển "đúng" lúc khởi động nhưng dần lệch khi robot di chuyển xa cấu hình ban đầu.

2. **Hiểu nhầm: "`τ = J^T·F` luôn đúng cho MỌI mục tiêu gia tốc mong muốn, không cần quan tâm quán tính."** Sai — công thức `τ = J^T·F` chỉ đúng khi F đã được tính từ `F = Λ·ẍ* + μ` (đã tính đến quán tính không gian tác vụ Λ và số hạng phi tuyến μ). Nếu bỏ qua Λ mà lấy F tỷ lệ thuận trực tiếp với sai số vị trí (kiểu PD đơn giản ở x rồi nhân J^T), robot sẽ phản ứng "méo" theo hướng — nhanh/chậm khác nhau tùy trục, đúng như ví dụ tính tay ở trên (trục x nhẹ hơn trục y).

3. **Hiểu nhầm: "Null-space projection nghĩa là tác vụ phụ hoàn toàn không ảnh hưởng gì tới tác vụ chính, ở mọi thời điểm."** Đúng về mặt tức thời (instantaneous — tại đúng thời điểm chiếu), nhưng vì `J` thay đổi liên tục theo cấu hình, một chuỗi hành động ở null-space theo thời gian VẪN có thể làm robot trôi dần ra khỏi vùng mà null-space đó còn "an toàn" — cần tái tính chiếu liên tục, không phải chiếu một lần.

## 🏗️ Ví dụ minh họa trong dự án này

Trong pipeline WBC → SONIC của dự án này, operational-space control không được dùng trực tiếp ở dạng thuần túy Khatib 1987 — nhưng **tư duy "tách task-space khỏi joint-space" là nền tảng khái niệm** cho toàn bộ chuỗi: retargeting (`02-motion-retargeting/`) sinh ra quỹ đạo tham chiếu ở dạng vị trí/hướng khớp task-space (bàn tay, gốc thân, bàn chân) → policy WBC học sâu (mục 2 trong `NOI-DUNG-CHI-TIET.md`) học cách ánh xạ các tác vụ task-space đó thành mô-men khớp, đúng vai trò mà `τ = J^T·F` từng làm một cách tường minh trong công thức cổ điển, nhưng giờ được một mạng nơ-ron "học ngầm" thay vì tính giải tích. Hiểu operational-space control giúp hiểu TẠI SAO observation/action space của policy RL trong Isaac Lab (`05-simulation-mujoco-isaaclab/`) thường biểu diễn mục tiêu ở dạng vị trí/vận tốc task-space thay vì chỉ góc khớp thô.

## 🔥 Cập nhật hiện đại / SOTA gần đây

Bản thân công thức toán operational-space control (1987) là kiến thức nền tảng ổn định — không có "thay thế" theo nghĩa nó bị chứng minh sai, nhưng có hai hướng phát triển/ứng dụng hiện đại đáng chú ý dựa trên tra cứu thực tế:

1. **Mở rộng chính thức cho toàn thân với nhiều ràng buộc/tiếp xúc:** Khatib cùng cộng sự (Khatib, Jorda, Park, Sentis, Chung) công bố "Constraint-consistent task-oriented whole-body robot formulation: Task, posture, constraints, multiple contacts, and balance", *International Journal of Robotics Research*, 2022 — mở rộng công thức operational-space gốc để xử lý đồng thời nhiều tiếp xúc, tư thế, và cân bằng bằng các phép chiếu null-space "dynamically consistent" lồng nhau, chứ không dừng ở một tác vụ + một tác vụ phụ đơn giản như bản 1987. ([SAGE Journals](https://journals.sagepub.com/doi/abs/10.1177/02783649221120029))

2. **Kết hợp với Control Barrier Functions (CBF) để đảm bảo an toàn:** nghiên cứu "Safe, Task-Consistent Manipulation with Operational Space Control Barrier Functions" (arXiv:2503.06736, 2025) mở rộng khung operational-space control bằng cách thêm ràng buộc an toàn dạng CBF ngay trong không gian tác vụ, đảm bảo robot không vi phạm giới hạn an toàn (va chạm, giới hạn khớp) trong khi vẫn giữ được cấu trúc null-space phân tầng gốc của Khatib. Một hướng liên quan là "Safety-Critical Whole-Body Control for Humanoid Robots via Input-to-State Safe Control Barrier Functions" (arXiv:2605.25546). ([arXiv:2503.06736](https://arxiv.org/pdf/2503.06736))

3. **Xu hướng chung 2024-2026:** phần lớn nghiên cứu humanoid WBC hiện đại (ví dụ HOVER — ICRA 2025, JAEGER — arXiv:2505.06584) không dùng lại công thức operational-space giải tích thuần túy mà dùng nó như một "bộ khung tư duy task-space" bên trong pipeline học sâu — tức là ý tưởng cốt lõi (tách task-space khỏi joint-space, dùng Jacobian để quy đổi) vẫn sống, nhưng phần "giải" (từ F ra τ) ngày càng được một policy học thay vì tính giải tích tường minh, đặc biệt khi robot cần xử lý các ràng buộc phi tuyến phức tạp mà công thức giải tích khó biểu diễn gọn.

## ❓ Câu hỏi tự kiểm tra

1. Vì sao J^T (chứ không phải J) dùng để chuyển lực từ không gian tác vụ sang không gian khớp?
<details><summary>Gợi ý đáp án</summary>Từ nguyên lý công ảo: công ảo δW = F^T·δx = F^T·(J·δq) = (J^T·F)^T·δq, mà công ảo ở không gian khớp là τ^T·δq — so sánh hai vế suy ra τ = J^T·F.</details>

2. Λ(x) = (J·M⁻¹·J^T)⁻¹ có đơn vị/ý nghĩa vật lý gì? Vì sao không dùng trực tiếp M?
<details><summary>Gợi ý đáp án</summary>Λ là "ma trận quán tính hiệu dụng" cảm nhận được ở đầu công tác — nó khác M vì phải "lọc" quán tính khớp qua Jacobian để quy về đúng chiều của không gian tác vụ (m chiều thay vì n chiều).</details>

3. Nếu robot có n=30 bậc tự do và tác vụ chính chỉ chiếm m=6 chiều (vị trí+hướng một tay), null-space của tác vụ chính có bao nhiêu chiều (xấp xỉ, giả sử J hạng đầy đủ)?
<details><summary>Gợi ý đáp án</summary>Xấp xỉ n − m = 24 chiều — đây là không gian "tự do dư" mà các tác vụ phụ (ví dụ giữ tư thế thoải mái) có thể hoạt động mà không phá tác vụ chính.</details>

4. Tại sao operational-space control thuần túy (không QP) khó xử lý trực tiếp ràng buộc "lực tiếp xúc chân phải nằm trong friction cone"?
<details><summary>Gợi ý đáp án</summary>Vì công thức gốc giải một đẳng thức (F = Λẍ*+μ rồi τ=J^T F) chứ không có cơ chế tự nhiên để áp bất đẳng thức — cần chuyển sang dạng tối ưu có ràng buộc (QP) để biểu diễn friction cone như một tập bất đẳng thức tuyến tính/nón.</details>

5. Trong ví dụ tính tay, nếu θ₂ = 0° (cánh tay duỗi thẳng) thay vì 90°, Jacobian sẽ suy biến (mất hạng) — hãy giải thích ý nghĩa vật lý của hiện tượng này (không cần tính lại toàn bộ).
<details><summary>Gợi ý đáp án</summary>Đây là điểm kỳ dị (singularity) — khi cánh tay duỗi thẳng, chuyển động dọc theo trục cánh tay không thể tạo ra bằng bất kỳ tổ hợp vận tốc khớp nào theo hướng đó ở bậc nhất, Λ⁻¹ trở nên gần suy biến (một trị riêng tiến về 0), khiến Λ "bùng nổ" — lực cần thiết để đạt gia tốc mong muốn theo hướng đó tiến tới vô cùng.</details>

6. Vì sao J^# (nghịch đảo có trọng số quán tính) khác Moore-Penrose J^+ lại quan trọng khi humanoid thực hiện nhiều tác vụ đồng thời (ví dụ tay + cân bằng)?
<details><summary>Gợi ý đáp án</summary>Vì J^# đảm bảo tác vụ ở null-space (ví dụ giữ tư thế thoải mái) không tạo ra phản lực động lực học lên tác vụ chính (ví dụ cân bằng) — dùng J^+ thông thường chỉ đảm bảo trực giao hình học, không đảm bảo "vô hại về mặt lực/mô-men" khi robot đang chuyển động, có thể khiến tác vụ ưu tiên cao bị nhiễu ngoài ý muốn.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể:** dùng đúng cấu hình 2-khớp ở ví dụ trên nhưng đổi θ₂ = 45°, tính lại J, rồi Λ⁻¹ = J·M⁻¹·J^T (dùng cùng M⁻¹ minh họa ở trên để đơn giản hóa) — so sánh Λ thu được với kết quả θ₂=90°, giải thích vì sao "độ nhẹ" theo từng trục thay đổi.
2. **Đọc code thật:** tìm trong một thư viện điều khiển robot mã nguồn mở có cài đặt operational-space control (ví dụ `pinocchio` hoặc `RBDL` có hàm tính Jacobian + mass matrix, hoặc code tham khảo trong repo `GR00T-WholeBodyControl`) — xác định đoạn code nào tương ứng với từng bước trong sơ đồ 7 bước ở mục Cơ chế hoạt động, ghi chú lại tên hàm tương ứng.
3. **Phân tích null-space:** với robot 3 khớp phẳng (thêm một khớp nữa vào ví dụ trên, n=3) nhưng tác vụ chính vẫn chỉ là vị trí 2D đầu công tác (m=2), hãy lý luận (không cần tính số) về số chiều của null-space, và đề xuất một tác vụ phụ hợp lý có thể đặt vào không gian dư đó cho một cánh tay robot thật.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

Operational-space control (Khatib, 1987) giải quyết bài toán "làm sao điều khiển trực tiếp một đại lượng vật lý ở đầu công tác (task space) thay vì phải nghĩ bằng góc khớp", bằng cách dùng Jacobian `J` để biến đổi vận tốc/lực qua lại giữa hai không gian, định nghĩa một "ma trận quán tính tác vụ" `Λ = (J·M⁻¹·J^T)⁻¹` để tuyến tính hóa động lực học tác vụ thành dạng Newton đơn giản, và dùng phép chiếu null-space `(I − J^T·J^#)` để tận dụng bậc tự do dư cho các tác vụ phụ mà không phá tác vụ chính. Đây là nền tảng lý thuyết trực tiếp cho mọi kỹ thuật WBC hiện đại — từ QP-based WBC, Hierarchical QP, tới cách các policy RL hiện đại biểu diễn mục tiêu điều khiển ở không gian tác vụ thay vì không gian khớp thô.
