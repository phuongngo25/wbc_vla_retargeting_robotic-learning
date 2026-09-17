# Bài giảng: Body model SMPL — cơ chế toán học

*(Thuộc mảng: Human Motion Datasets)*

## 🎯 Mục tiêu bài học

- Giải thích được vì sao mocap (motion capture) thô — chỉ là toạ độ vài chục marker — không đủ để dùng trực tiếp cho robot học hay đồ hoạ, và SMPL giải quyết vấn đề gì.
- Liệt kê và mô tả đúng vai trò của 4-5 thành phần cấu thành SMPL: template mesh, shape blend shapes (β), pose blend shapes (θ), pose-dependent corrective blend shapes, và Linear Blend Skinning (LBS).
- Tính tay được (ở mức đơn giản hoá) một bước skinning tuyến tính cho 1 vertex với 2 khớp ảnh hưởng.
- Giải thích được "candy-wrapper artifact" là gì và vì sao pose-dependent blend shapes cần thiết để sửa nó.
- Phân biệt được SMPL với các mô hình liên quan (STAR, SMPL-X) — biết SMPL là nền tảng cho các mô hình mở rộng sau này.
- Giải thích được vì sao tính khả vi (differentiable) của SMPL là lý do nó trở thành "ngôn ngữ chung" cho pipeline mocap → robot.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Toàn bộ pipeline của dự án này — từ dataset (AMASS, OMOMO) → retargeting (GMR, SOMA) → huấn luyện WBC/VLA — đều bắt đầu từ một câu hỏi: chuyển động của con người được **biểu diễn** dưới dạng gì trước khi đưa vào bất kỳ thuật toán nào? Nếu câu trả lời là "toạ độ marker thô", mỗi dataset sẽ có một số lượng marker khác nhau, gắn ở vị trí khác nhau, không thể gộp chung hay đưa vào mạng neural một cách nhất quán. SMPL là câu trả lời được cộng đồng chấp nhận rộng rãi nhất cho câu hỏi này kể từ 2015, và gần như mọi dataset/công cụ nhắc tới trong thư mục `03-human-motion-datasets/` (AMASS, OMOMO) và `02-motion-retargeting/` (GMR, SOMA-retargeter) đều dùng SMPL hoặc một biến thể mở rộng của nó (SMPL-X — xem bài giảng riêng) làm định dạng trung gian. Hiểu đúng cơ chế SMPL là điều kiện tiên quyết để hiểu vì sao AMASS "hợp nhất được" 15 bộ mocap khác nhau, và vì sao retargeting từ người sang robot lại bắt đầu từ (β, θ) chứ không phải từ toạ độ marker.

## 🧠 Trực giác

### Góc nhìn 1: Con rối rối (marionette) với lớp da co giãn

Hãy tưởng tượng một con rối gỗ có khung xương (skeleton) bên trong — đây chính là vai trò của tham số **pose θ**: mỗi sợi dây điều khiển một khớp, kéo dây để xoay khớp đó. Nhưng khác với con rối gỗ cứng, cơ thể SMPL được bọc một lớp "da" mềm dẻo bên ngoài khung xương, và lớp da này được may đo riêng cho từng người bởi tham số **shape β** — người này "may" cao 1m90 gầy, người kia "may" thấp 1m60 mập. Khi khung xương cử động (θ thay đổi), lớp da được kéo theo bởi các "dây buộc vô hình" gắn từ da vào từng khớp gần đó (đây là *skinning weights* — trọng số skinning) — da ở cổ tay bị kéo chủ yếu bởi khớp cổ tay, một phần nhỏ bởi khớp khuỷu tay.

**Giới hạn của loại suy này**: con rối gỗ có khớp là bản lề cứng (hinge) đơn giản, còn khớp người thật xoay theo 3 trục (ball joint) và da người còn co giãn/phồng do cơ bắp co lại — điều mà phép loại suy "dây buộc" không giải thích được. Đây chính là lý do SMPL cần thêm **pose-dependent blend shapes** (xem Cơ chế bên dưới) — một lớp "hiệu chỉnh" mà con rối gỗ không có tương đương.

### Góc nhìn 2: Hàm số 2 biến đầu vào, 1 biến đầu ra (góc nhìn toán học/kỹ sư)

Nhìn thuần tuý như một kỹ sư phần mềm: SMPL là một **hàm số** `M(β, θ) → mesh` nhận vào 2 vector số (β ~10 chiều, θ ~72 chiều cho 23 khớp × 3 góc axis-angle + 3 cho root) và trả về toạ độ 3D của 6,890 vertex. Hàm này được "học" (fit) từ hàng nghìn bản scan 3D người thật, chứ không phải được lập trình bằng luật hình học thủ công (như cách một số engine đồ hoạ cũ dùng công thức lượng giác cố định cho từng khớp). Vì hàm này khả vi, ta có thể dùng gradient descent để đi ngược: "cho trước một đám mây điểm 3D quan sát được, tìm (β, θ) nào sinh ra mesh khớp nhất với nó" — đây chính là cách MoSh/MoSh++ hoạt động (xem bài giảng AMASS).

**Giới hạn của loại suy này**: nhìn SMPL thuần tuý như "hàm số học được" dễ khiến người mới nghĩ nó là hộp đen (black box) không có cấu trúc — trong khi thực ra nó có cấu trúc rất tường minh (4-5 thành phần cộng dồn tuyến tính + 1 bước skinning phi tuyến), không phải một mạng neural end-to-end. Cấu trúc tường minh này là lý do SMPL nhẹ, nhanh, và dễ diễn giải hơn nhiều so với một mô hình sinh mesh bằng neural network thuần tuý.

## 📐 Định nghĩa chính xác

SMPL (*Skinned Multi-Person Linear model*, Loper, Mahmood, Romero, Pons-Moll, Black — SIGGRAPH Asia 2015, ACM ToG 34(6)) định nghĩa một hàm:

```
M(β, θ) = W( T_P(β, θ), J(β), θ, 𝒲 )
```

Trong đó:

- **T_P(β, θ)** = mesh cơ sở ở tư thế trung tính (T-pose) sau khi cộng dồn các blend shape:
  `T_P(β, θ) = T̄ + B_S(β) + B_P(θ)`
  - `T̄`: template mesh trung bình, **N = 6,890 vertex**, topology cố định.
  - `B_S(β)`: **shape blend shapes** — tổ hợp tuyến tính của các hướng biến đổi hình dáng học từ PCA (Principal Component Analysis) trên hàng nghìn bản scan 3D; β thường lấy **10 chiều đầu** (có thể mở rộng).
  - `B_P(θ)`: **pose-dependent (corrective) blend shapes** — hiệu chỉnh hình dạng phụ thuộc góc khớp hiện tại, sửa lỗi méo mesh mà LBS thuần tuý gây ra.
- **J(β)**: hàm hồi quy tuyến tính từ β ra vị trí **K = 23 khớp nội bộ** (joint regressor) — vị trí khớp cũng phụ thuộc hình dáng cơ thể (người cao thì khớp vai ở vị trí khác người thấp).
- **θ**: vector góc khớp dạng **axis-angle**, kích thước 3×(K+1) = 3×24 = 72 (23 khớp + 1 root).
- **W(·)**: phép toán **Linear Blend Skinning (LBS)** — di chuyển từng vertex theo tổ hợp tuyến tính của các phép biến đổi cứng (rigid transform) tại từng khớp, có trọng số **𝒲** (skinning weights, ma trận thưa N×K cố định, học từ dữ liệu).

Công thức LBS cho một vertex `v` cụ thể:

```
v' = Σ_k  w_{v,k} · G_k(θ, J) · v̄
```

trong đó `G_k(θ, J)` là ma trận biến đổi đồng nhất (4×4, world transform) của khớp k tính từ chuỗi động học thuận (forward kinematics) theo θ, và `w_{v,k}` là trọng số skinning của vertex v đối với khớp k (Σ_k w_{v,k} = 1 với mỗi v).

## ⚙️ Cơ chế hoạt động — từng bước

```
                     β (shape, ~10 chiều)         θ (pose, 72 chiều)
                            │                            │
                            ▼                            ▼
                   ┌──────────────────┐         ┌──────────────────┐
                   │ Shape blend      │         │ Pose-dependent   │
                   │ shapes B_S(β)    │         │ blend shapes     │
                   │ (PCA directions) │         │ B_P(θ)           │
                   └────────┬─────────┘         └────────┬─────────┘
                            │                            │
                            ▼                            ▼
         T̄ (template) ──►  cộng dồn  T_P = T̄ + B_S(β) + B_P(θ)
                            │
                            ▼
                   ┌──────────────────┐
                   │ Joint regressor  │◄── β
                   │ J(β) → vị trí    │
                   │ 23 khớp          │
                   └────────┬─────────┘
                            │
                            ▼
                   ┌──────────────────┐
                   │ Forward kinematics│◄── θ (chuỗi động học)
                   │ → G_k(θ,J) mỗi   │
                   │ khớp k           │
                   └────────┬─────────┘
                            │
                            ▼
                   ┌──────────────────┐
                   │ Linear Blend     │◄── skinning weights 𝒲
                   │ Skinning (LBS)   │    (cố định, học sẵn)
                   └────────┬─────────┘
                            │
                            ▼
                    Mesh cuối M(β,θ)
                 (N = 6,890 vertex 3D)
```

**Bước 1 — Xác định hình dáng cá nhân (β → shape)**: β là 10 (hoặc nhiều hơn) hệ số nhân với các "hướng biến dạng chính" (principal components) học từ tập dữ liệu quét 3D CAESAR (hàng nghìn người, đa dạng chủng tộc/giới tính/vóc dáng). Mỗi chiều của β không có nghĩa "vật lý" trực tiếp (không phải "chiều cao", "cân nặng" riêng biệt) mà là một tổ hợp thống kê — chiều β₁ có thể tương quan mạnh với chiều cao tổng thể, β₂ với tỷ lệ vai/hông, v.v.

**Bước 2 — Xác định vị trí khớp theo hình dáng**: khớp vai của một người cao 1m90 nằm ở vị trí khác khớp vai của người cao 1m60, dù cùng một θ. Hàm `J(β)` (một phép hồi quy tuyến tính đã học sẵn từ dữ liệu) tính vị trí 23 khớp dựa trên mesh hình dáng đã có ở bước 1 — đảm bảo skeleton "vừa khít" với mesh, không bị lệch.

**Bước 3 — Áp dụng tư thế (θ → pose blend correction)**: trước khi skinning, mesh được hiệu chỉnh thêm bởi `B_P(θ)` — ví dụ khi khuỷu tay gập 90°, các vertex quanh vùng cơ nhị đầu được "phồng" lên trước, mô phỏng hiệu ứng co cơ. Nếu bỏ qua bước này (chỉ dùng LBS thuần), mesh sẽ bị hiện tượng **"candy-wrapper artifact"** — vùng khớp gập bị tóp lại giống vỏ kẹo bị vặn xoắn, vì phép nội suy tuyến tính giữa 2 phép biến đổi cứng của 2 khớp lân cận làm mất thể tích ở vùng giao nhau.

**Bước 4 — Skinning (LBS)**: mỗi vertex được di chuyển theo tổ hợp trọng số của các khớp ảnh hưởng nó (thường 1-4 khớp gần nhất có trọng số khác 0, theo ma trận 𝒲 thưa đã học sẵn). Đây là bước cuối cùng biến (β, θ) thành mesh 3D thực tế trong không gian.

**Tính khả vi**: mọi bước trên đều là phép toán tuyến tính hoặc trơn (differentiable) theo (β, θ) — kể cả forward kinematics qua axis-angle → ma trận xoay (dùng công thức Rodrigues, cũng khả vi). Đây là lý do có thể dùng gradient-based optimization (MoSh++) hoặc backprop qua mạng neural để "tìm" hoặc "sinh" (β, θ) từ dữ liệu quan sát được.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số liệu tự chọn đơn giản hoá để dễ hình dung — không phải số liệu thật từ SMPL, vì ma trận skinning thật của SMPL có ~6,890×24 trọng số không thể tính tay.)*

Giả sử ta chỉ xét **1 vertex `v`** nằm giữa khớp khuỷu tay (elbow, k=1) và khớp cổ tay (wrist, k=2), với trọng số skinning đã học sẵn: `w_{v,1} = 0.7`, `w_{v,2} = 0.3` (tổng = 1, đúng ràng buộc chuẩn hoá).

Ở tư thế T-pose (θ=0), vị trí cục bộ của vertex là `v̄ = (0.10, 0.00, 0.00)` m (tính từ gốc toạ độ cục bộ, đơn vị mét).

Bây giờ giả sử cẳng tay xoay quanh khớp khuỷu tay một góc 90° quanh trục z (θ_elbow = [0,0,π/2] dạng axis-angle rút gọn), còn khớp cổ tay không xoay thêm (θ_wrist = 0). Ma trận xoay 90° quanh z:

```
R_z(90°) = [ 0  -1   0 ]
           [ 1   0   0 ]
           [ 0   0   1 ]
```

**Bước A — tính G_1 (world transform của khớp khuỷu tay)**: giả sử khớp khuỷu tay nằm tại gốc cục bộ, không tịnh tiến thêm ngoài xoay → `G_1 · v̄ = R_z(90°) · (0.10, 0, 0) = (0, 0.10, 0)`.

**Bước B — tính G_2 (world transform của khớp cổ tay)**: vì θ_wrist = 0, cổ tay không xoay thêm so với hệ quy chiếu của khuỷu tay đã xoay — theo chuỗi động học, `G_2 = G_1` trong ví dụ đơn giản hoá này (cổ tay "thừa hưởng" phép xoay của khuỷu tay vì nó là khớp con) → `G_2 · v̄ = (0, 0.10, 0)` (giống G_1 trong trường hợp đặc biệt này).

**Bước C — tổ hợp LBS**:
```
v' = w_{v,1}·(G_1·v̄) + w_{v,2}·(G_2·v̄)
   = 0.7·(0, 0.10, 0) + 0.3·(0, 0.10, 0)
   = (0, 0.10, 0)
```

Trong ví dụ đơn giản hoá này (θ_wrist=0), kết quả không phụ thuộc vào cách chia trọng số vì G_1 = G_2. Để thấy rõ vai trò của trọng số hỗn hợp, giả sử cổ tay xoay thêm 30° độc lập với khuỷu tay (một θ_wrist khác 0) khiến `G_2·v̄ = (0.02, 0.09, 0)` (số minh hoạ) trong khi `G_1·v̄` vẫn là `(0, 0.10, 0)` — khi đó:

```
v' = 0.7·(0, 0.10, 0) + 0.3·(0.02, 0.09, 0)
   = (0.006, 0.097, 0)
```

Vertex bị "kéo" một phần nhỏ về phía biến đổi của cổ tay (30% ảnh hưởng), minh hoạ đúng cơ chế "blend" trong Linear Blend Skinning — vertex không "thuộc về" tuyệt đối một khớp nào mà là trung bình có trọng số của nhiều khớp lân cận.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | SMPL (2015) | STAR (2020) | Rigging thủ công (game engine cổ điển) |
|---|---|---|---|
| Số tham số shape | ~10 (PCA toàn cục, mỗi chiều ảnh hưởng toàn mesh) | ~300 nhưng **thưa** (sparse) — mỗi khớp chỉ ảnh hưởng vertex cục bộ gần nó | Không có tham số shape thống nhất — mỗi nhân vật rig riêng |
| Kích thước model | Lớn hơn STAR (dense corrective blend shapes) | ~20% số tham số của SMPL, generalize tốt hơn với ít dữ liệu hơn | Không so sánh được (không phải model tham số hoá) |
| Học từ dữ liệu quét 3D | Có (CAESAR + các bản scan bổ sung) | Có, **cộng thêm 10,000 scan mới** so với SMPL | Không — dựa vào tay nghề riggers |
| Khả vi (differentiable) | Có | Có | Thường không (dùng công thức cố định, không tối ưu hoá được bằng gradient) |
| Cộng đồng/hệ sinh thái | Rất lớn — chuẩn de-facto, AMASS/OMOMO/hầu hết công cụ retargeting hỗ trợ sẵn | Nhỏ hơn nhiều, ít dataset lớn công bố ở định dạng STAR | Riêng biệt theo từng engine/game, không tương thích chéo |
| Khi nào dùng | Mặc định cho mọi bài toán mocap→robot hiện nay (do hệ sinh thái) | Khi cần model nhẹ hơn, cần generalize tốt với ít dữ liệu | Khi chỉ cần 1 nhân vật cố định, không cần tương thích dataset ngoài |

> Nguồn STAR: Osman, Bolkart, Black (2020), *"STAR: A Sparse Trained Articulated Human Body Regressor"*, ECCV 2020. [star.is.tue.mpg.de](https://star.is.tue.mpg.de)

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "β chính là các chỉ số vật lý như chiều cao, cân nặng."**
   Vì sao sai: β là hệ số của các thành phần PCA học từ dữ liệu, không có ánh xạ 1-1 sang đại lượng vật lý con người thường nghĩ tới. Một chiều β có thể đồng thời ảnh hưởng nhẹ tới chiều cao VÀ tỷ lệ vai.
   Hiểu đúng: β là toạ độ trong "không gian hình dáng" học được — muốn suy luận "người này cao bao nhiêu" phải tính từ mesh sinh ra (ví dụ đo khoảng cách đỉnh đầu-gót chân), không đọc trực tiếp từ β.

2. **Hiểu nhầm: "SMPL có mesh + skeleton nghĩa là nó đã sẵn sàng để robot bắt chước trực tiếp."**
   Vì sao sai: 23 khớp và tỷ lệ xương của SMPL là của **cơ thể người**, khác hoàn toàn số bậc tự do, giới hạn khớp, và tỷ lệ chi của một robot humanoid cụ thể (ví dụ Unitree G1 có số khớp và giới hạn góc khác). Copy trực tiếp θ từ SMPL sang robot sẽ gây va chạm hoặc vượt giới hạn khớp.
   Hiểu đúng: cần một bước **retargeting** riêng (xem `02-motion-retargeting/`) để ánh xạ (β, θ) sang không gian khớp của robot cụ thể, có tính đến giới hạn vật lý.

3. **Hiểu nhầm: "Skinning weights (𝒲) được tính lại mỗi lần render, phụ thuộc vào θ hiện tại."**
   Vì sao sai: 𝒲 là ma trận **cố định**, học một lần từ dữ liệu, không đổi theo θ hay β — thứ thay đổi theo θ là *vị trí* các khớp (qua forward kinematics), không phải trọng số ảnh hưởng của từng khớp lên từng vertex.
   Hiểu đúng: 𝒲 tương tự "cấu hình rigging cố định", còn θ là "input điều khiển" chạy qua cấu hình đó ở mỗi khung hình.

## 🏗️ Ví dụ minh hoạ trong dự án này

Trong pipeline của dự án (xem `README.md` gốc `03-human-motion-datasets/`), SMPL là **định dạng trung gian bắt buộc phải đi qua** trước khi tới bất kỳ bước huấn luyện robot nào: AMASS lưu chuyển động dưới dạng chuỗi (β, θ) SMPL/SMPL-X; khi GMR hoặc SOMA-retargeter (thư mục `02-motion-retargeting/`) nhận một file AMASS làm input, việc đầu tiên chúng làm là **giải mã (β, θ) → mesh 3D + vị trí 23 khớp** bằng đúng công thức `M(β,θ)` ở trên, sau đó mới bắt đầu bài toán IK để map các khớp đó sang skeleton của Unitree G1. Nếu không hiểu cơ chế SMPL, sẽ không hiểu tại sao bước retargeting cần "vị trí khớp 3D" làm input trung gian thay vì dùng thẳng θ của SMPL cho robot (vì θ của SMPL không có ý nghĩa trực tiếp trên skeleton robot khác cấu trúc).

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **Retargeting neural học trực tiếp từ SMPL, bỏ qua tối ưu hoá lặp**: paper *"Make Tracking Easy: Neural Motion Retargeting for Humanoid Whole-body Control"* (arXiv 2603.22201, 2026) đề xuất học một mạng ánh xạ trực tiếp từ chuỗi chuyển động SMPL sang chuyển động robot khả thi, tránh các điểm cực trị cục bộ (local optima) thường gặp ở phương pháp tối ưu hoá cổ điển (fit θ_SMPL → θ_robot bằng gradient descent lặp mỗi khung hình). Đây là minh chứng cho việc SMPL vẫn là định dạng đầu vào chuẩn ngay cả khi phương pháp retargeting chuyển sang học sâu.

2. **World-Coordinate Human Motion Retargeting via SAM 3D Body** (arXiv 2512.21573, 2025/2026): xây dựng pipeline dùng body model 3D (họ SMPL) với "trajectory-level identity và skeleton-scale locking" để giữ nhất quán hình học khi retarget toàn bộ quỹ đạo dài, báo cáo kết quả trên Unitree G1: **zero joint jumps, giảm 54% frame tự va chạm (self-collision), giảm 61% vi phạm giới hạn khớp** so với baseline — cho thấy các cải tiến gần đây tập trung vào việc khai thác tốt hơn cấu trúc hình học của SMPL/SMPL-X trong không gian toàn cục (world coordinate), không phải thay thế bản thân body model.

3. **STAR (Osman, Bolkart, Black, ECCV 2020)** vẫn là ứng viên thay thế được công bố có kiểm chứng academic rõ ràng nhất cho SMPL (tham số thưa hơn ~80%, generalize tốt hơn) — nhưng tính đến thời điểm tra cứu, hệ sinh thái dataset lớn (AMASS, OMOMO, BONES-SEED) và công cụ retargeting (GMR, SOMA) trong lĩnh vực humanoid robot vẫn **chưa chuyển sang STAR làm mặc định** — SMPL/SMPL-X vẫn là chuẩn de-facto vì hiệu ứng mạng lưới (network effect) của dữ liệu và công cụ đã xây dựng sẵn xung quanh nó.

## ❓ Câu hỏi tự kiểm tra

1. Vì sao cần cả shape blend shapes (β) lẫn pose blend shapes (θ) — không thể dùng chung 1 vector tham số?
   <details><summary>Gợi ý đáp án</summary>β mã hoá đặc điểm KHÔNG đổi theo thời gian trong một phiên mocap (hình dáng cơ thể), θ mã hoá cái THAY ĐỔI mỗi khung hình (tư thế). Tách riêng giúp AMASS/MoSh++ chỉ cần ước lượng β một lần cho cả chuỗi, tối ưu θ riêng cho từng khung — hiệu quả hơn nhiều so với ước lượng lại toàn bộ mesh mỗi khung hình.</details>

2. "Candy-wrapper artifact" là gì, và thành phần nào của SMPL được thiết kế để sửa nó?
   <details><summary>Gợi ý đáp án</summary>Là hiện tượng mesh bị tóp/méo phi thực tế ở vùng khớp gập khi chỉ dùng Linear Blend Skinning thuần tuý (nội suy tuyến tính giữa 2 phép biến đổi cứng làm mất thể tích). Được sửa bởi pose-dependent (corrective) blend shapes B_P(θ).</details>

3. Trong ví dụ tính tay ở trên, nếu đổi trọng số skinning thành `w_{v,1}=0.5, w_{v,2}=0.5` (thay vì 0.7/0.3), kết quả `v'` ở trường hợp θ_wrist≠0 sẽ thay đổi thế nào?
   <details><summary>Gợi ý đáp án</summary>v' sẽ dịch gần trung điểm giữa (0,0.10,0) và (0.02,0.09,0) hơn, cụ thể = 0.5·(0,0.10,0)+0.5·(0.02,0.09,0) = (0.01, 0.095, 0) — thiên về ảnh hưởng cổ tay nhiều hơn so với 0.7/0.3 ban đầu (0.006, 0.097, 0).</details>

4. Vì sao nói SMPL là "khả vi" lại quan trọng cho MoSh/MoSh++ (xem bài giảng AMASS)?
   <details><summary>Gợi ý đáp án</summary>Vì MoSh/MoSh++ cần tối ưu hoá (β,θ) bằng gradient descent sao cho marker ảo trên mesh khớp với marker thật đo được — nếu hàm M(β,θ) không khả vi, không thể tính gradient để lan truyền ngược và tối ưu.</details>

5. Nếu một kỹ sư muốn dùng trực tiếp θ_SMPL của khớp khuỷu tay để điều khiển góc khớp khuỷu tay của robot G1, vì sao cách này thường không đúng?
   <details><summary>Gợi ý đáp án</summary>Vì tỷ lệ xương, số bậc tự do, và giới hạn góc khớp của G1 khác cấu trúc SMPL — cần bước retargeting (IK per-frame, xem `02-motion-retargeting/`) ánh xạ vị trí khớp 3D (không phải θ trực tiếp) sang không gian khớp robot, có ràng buộc vật lý riêng.</details>

## 📝 Bài tập thực hành

1. **Đọc code thật**: clone repo chính thức [`vchoutas/smplx`](https://github.com/vchoutas/smplx) (thư viện Python triển khai cả SMPL và SMPL-X), tìm hàm `lbs()` trong `smplx/lbs.py`. Đối chiếu từng bước trong hàm đó (`blend_shapes`, `vertices2joints`, `batch_rigid_transform`, `skinning`) với 4 bước mô tả ở mục "Cơ chế hoạt động" trên — ghi lại tên biến code tương ứng với B_S(β), B_P(θ), J(β), G_k(θ).

2. **Tính tay biến thể khác**: lặp lại ví dụ tính tay ở trên nhưng với 3 khớp ảnh hưởng 1 vertex (thêm khớp vai với `w_{v,shoulder}=0.1`, giảm `w_{v,elbow}=0.6`, `w_{v,wrist}=0.3`), giả sử vai xoay thêm 10° quanh trục y khiến `G_shoulder·v̄ = (0.005, 0.099, 0.01)` (số minh hoạ) — tính `v'` cuối cùng.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

SMPL biến một vài trăm con số (β cho hình dáng, θ cho tư thế) thành một mesh 3D đầy đủ của cơ thể người thông qua 4 bước: cộng shape blend shapes vào template, cộng pose-dependent blend shapes để sửa lỗi biến dạng khớp, tính vị trí khớp từ hình dáng, rồi áp dụng Linear Blend Skinning để di chuyển từng vertex theo tổ hợp trọng số các khớp lân cận — toàn bộ quá trình khả vi nên có thể tối ưu ngược (β,θ) từ dữ liệu quan sát được, biến SMPL thành "ngôn ngữ chung" mà AMASS dùng để hợp nhất 15 bộ mocap khác nhau và mọi công cụ retargeting hiện đại (GMR, SOMA) dùng làm định dạng đầu vào chuẩn trước khi ánh xạ sang skeleton robot cụ thể.
