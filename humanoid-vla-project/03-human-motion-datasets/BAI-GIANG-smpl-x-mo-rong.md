# Bài giảng: SMPL-X — mở rộng so với SMPL

*(Thuộc mảng: Human Motion Datasets)*

## 🎯 Mục tiêu bài học

- Giải thích được chính xác SMPL-X hợp nhất những mô hình nào (MANO, FLAME) và vì sao cần hợp nhất thay vì dùng 3 mô hình riêng biệt.
- Liệt kê đúng số liệu SMPL-X (N=10,475 vertex, K=54 khớp, 119 tham số) và giải thích ý nghĩa từng nhóm tham số.
- Phân biệt được tham số **expression** (biểu cảm) với tham số **shape** (β) — hiểu vì sao chúng phải tách biệt.
- Giải thích được vì sao AMASS/OMOMO chọn SMPL-X (hoặc SMPL-H) thay vì SMPL gốc cho bài toán robot học.
- Nhận diện được giới hạn của SMPL-X (không mô hình hoá quần áo, tóc, vẫn có artifact ở vùng ngón tay khi occlusion).
- Nêu được ít nhất 1 hướng nghiên cứu 2024-2026 mở rộng/khắc phục hạn chế của SMPL-X.

## 🧭 Vì sao cần học cái này? (bối cảnh)

SMPL (xem bài giảng riêng) chỉ mô hình hoá thân người ở mức thô — bàn tay là "khối cụt" không ngón, không có khuôn mặt biểu cảm. Với robot loco-manipulation (di chuyển + thao tác vật thể bằng tay) là trọng tâm của OMOMO và pipeline VLA trong dự án này, thông tin về **vị trí từng ngón tay khi cầm/nắm vật** là bắt buộc — đây chính là lý do tồn tại SMPL-X. Hiểu đúng SMPL-X là điều kiện để hiểu tại sao AMASS và OMOMO (2 dataset dùng trực tiếp trong pipeline) chọn định dạng này, và tại sao khi retarget sang robot (GMR, SOMA-retargeter) cần xử lý riêng phần bàn tay khác với phần thân.

## 🧠 Trực giác

### Góc nhìn 1: Lắp ghép 3 chuyên gia thành 1 đội (góc nhìn hệ thống)

Trước SMPL-X, cộng đồng có 3 mô hình chuyên biệt tách rời: **SMPL** (thân, do nhóm Loper et al. phát triển), **MANO** (bàn tay, *hand Model with Articulated and Non-rigid defOrmations*), và **FLAME** (khuôn mặt). Mỗi mô hình là một "chuyên gia" giỏi một việc nhưng không nói cùng "ngôn ngữ" với nhau — ghép kết quả của 3 mô hình riêng biệt lại (ví dụ dán tay MANO vào thân SMPL) sẽ bị lệch ở điểm nối (cổ tay, cổ) vì mỗi mô hình có hệ toạ độ và cách tham số hoá riêng. SMPL-X giống như "sáp nhập 3 phòng ban thành 1 công ty dùng chung 1 hệ thống sổ sách" — dùng chung một cơ chế toán học (blend skinning + pose-dependent blend shapes, giống hệt SMPL) cho toàn bộ cơ thể bao gồm cả tay và mặt, đảm bảo không có "đường nối" giữa các phần.

**Giới hạn của loại suy này**: "sáp nhập công ty" gợi ý rằng SMPL-X chỉ đơn giản là cộng gộp cơ học của 3 mô hình — thực tế việc học chung các blend shape đòi hỏi dữ liệu quét 3D toàn thân có ĐỒNG THỜI cả tay và mặt chi tiết (không phải ghép dữ liệu tay riêng + dữ liệu mặt riêng), một tập dữ liệu khó thu thập hơn nhiều so với dữ liệu cho từng phần riêng lẻ.

### Góc nhìn 2: Tăng độ phân giải của một bức ảnh (góc nhìn kỹ thuật số)

Nhìn theo góc "độ phân giải dữ liệu": SMPL giống một bức ảnh chân dung toàn thân chụp ở độ phân giải thấp — đủ thấy dáng người nhưng bàn tay chỉ là một vùng mờ không rõ ngón, mặt chỉ là một khối không biểu cảm. SMPL-X là "tăng độ phân giải" đúng 2 vùng quan trọng (tay: N tăng để đủ chi tiết ngón; mặt: thêm tham số expression) trong khi vẫn giữ nguyên độ phân giải phần thân đã đủ tốt — không phải tăng đều toàn bộ mesh.

**Giới hạn của loại suy này**: "tăng độ phân giải ảnh" ngụ ý chỉ là thêm chi tiết hình học thụ động — trong khi thực tế SMPL-X thêm cả **bậc tự do điều khiển được** (thêm khớp ngón tay, khớp hàm, khớp mắt vào K=54 khớp) chứ không chỉ thêm vertex hiển thị. Đây là khác biệt quan trọng: robot/animator có thể *điều khiển* các khớp mới này, không chỉ *nhìn* thấy chúng đẹp hơn.

## 📐 Định nghĩa chính xác

SMPL-X (*SMPL eXpressive*, Pavlakos, Choutas, Ghorbani, Bolkart, Osman, Tzionas, Black — CVPR 2019) là bản mở rộng của SMPL, dùng chung công thức:

```
M(β, θ, ψ) = W( T_P(β, θ, ψ), J(β), θ, 𝒲 )
```

Khác biệt với SMPL nằm ở kích thước và số nhóm tham số:

- **N = 10,475 vertex** (so với 6,890 của SMPL).
- **K = 54 khớp** (so với 23 của SMPL), bao gồm: khớp thân/tay/chân gốc như SMPL, cộng thêm **khớp cổ, khớp hàm (jaw), 2 khớp mắt (eyeballs)**, và các khớp ngón tay của cả hai bàn tay (dùng pose space của MANO — mỗi tay có PCA-reduced pose space thay vì axis-angle đầy đủ cho từng khớp ngón, để giảm chiều tham số).
- **Tổng 119 tham số mô hình**: 75 tham số cho **global body rotation + pose của thân/mắt/hàm**, 24 tham số cho **hand pose ở không gian PCA giảm chiều** (thay vì tham số hoá đầy đủ từng khớp ngón — MANO cung cấp sẵn "hand pose PCA space" học từ dữ liệu để nén số chiều), 10 tham số cho **shape β** (giống SMPL), và 10 tham số riêng cho **expression ψ** (biểu cảm khuôn mặt, dùng không gian của FLAME).
- **ψ (psi) — expression parameters**: tách biệt hoàn toàn khỏi β — cùng một β (một người với hình dáng cố định) có thể có nhiều ψ khác nhau (nhiều biểu cảm) mà không ảnh hưởng tới hình dáng cơ thể tổng thể.

SMPL-X dùng **pose space và pose corrective blend shapes của MANO** cho tay, và **expression space của FLAME** cho mặt — nghĩa là các thành phần chuyên biệt này không phải "phát minh lại" mà tái sử dụng trực tiếp cơ chế đã được kiểm chứng của 2 mô hình gốc, hợp nhất vào cùng một hàm khả vi duy nhất `M(β, θ, ψ)`.

## ⚙️ Cơ chế hoạt động — từng bước

```
                    β (shape, 10)   θ (pose, gồm cả   ψ (expression, 10)
                        │           thân+tay+mặt)         │
                        │               │                 │
                        ▼               ▼                 ▼
              ┌─────────────────────────────────────────────────┐
              │      Shape + Pose + Expression blend shapes      │
              │  B_S(β) + B_P(θ) + B_E(ψ)  (cộng dồn vào template)│
              └───────────────────────┬───────────────────────────┘
                                       │
                                       ▼
                          T̄ (N=10,475 vertex, K=54 khớp)
                                       │
                    ┌──────────────────┼───────────────────────┐
                    ▼                  ▼                       ▼
          ┌──────────────┐   ┌──────────────────┐   ┌──────────────────┐
          │ Thân + chân   │   │ Tay: pose space   │   │ Mặt: expression   │
          │ (như SMPL)    │   │ PCA của MANO      │   │ space của FLAME   │
          └──────┬────────┘   └────────┬──────────┘   └────────┬──────────┘
                 │                     │                        │
                 └─────────┬───────────┴────────────┬───────────┘
                           ▼                        ▼
                  Forward kinematics (θ toàn bộ 54 khớp)
                           │
                           ▼
                Linear Blend Skinning (LBS, trọng số 𝒲 mở rộng)
                           │
                           ▼
                  Mesh cuối M(β, θ, ψ) — 10,475 vertex
```

**Bước 1 — Hợp nhất template**: template mesh của SMPL-X được xây từ dữ liệu scan 3D người thật có ĐỦ cả tay và mặt chi tiết (không phải nối 3 mesh riêng của 3 model cũ) — nghĩa là các blend shape của SMPL-X (shape, pose, expression) được học lại từ đầu trên tập dữ liệu mới này, không đơn giản là "chuyển đổi" tham số từ SMPL/MANO/FLAME sang.

**Bước 2 — Tay dùng pose space PCA của MANO**: mỗi bàn tay có 15 khớp ngón (mỗi khớp 3 DOF axis-angle) — nếu tham số hoá đầy đủ sẽ tốn 45 chiều/tay. MANO học sẵn một không gian PCA giảm chiều (thường 6-24 chiều tuỳ cấu hình) bao phủ phần lớn các tư thế tay tự nhiên — SMPL-X kế thừa không gian này, giảm đáng kể số tham số cần tối ưu khi fit dữ liệu (quan trọng cho MoSh++ khi fit AMASS/OMOMO).

**Bước 3 — Mặt dùng expression space của FLAME**: tương tự, biểu cảm khuôn mặt được mã hoá bằng 10 tham số ψ học từ FLAME, tách biệt khỏi β (hình dáng đầu/mặt tĩnh) — cho phép cùng một khuôn mặt "cố định" biểu diễn nhiều biểu cảm khác nhau.

**Bước 4 — Skinning hợp nhất**: sau khi có mesh T_P với mọi hiệu chỉnh (shape+pose+expression), phép LBS chạy trên toàn bộ 10,475 vertex và 54 khớp cùng lúc — không có ranh giới "vùng thân" và "vùng tay/mặt" tách biệt trong bước này, đây chính là điểm khác với việc ghép cơ học 3 mesh riêng (tránh "đường nối" bị lộ).

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số tự chọn để dễ hình dung — không phải số liệu thật đo từ SMPL-X.)*

Giả sử ta cần biểu diễn một cử chỉ "chỉ tay" (pointing gesture) — ngón trỏ duỗi thẳng, các ngón khác co lại — bằng không gian PCA 6 chiều của MANO cho 1 bàn tay (giả định đơn giản hoá, MANO thật thường dùng nhiều chiều hơn).

Giả sử không gian PCA có 2 "hướng chính" quan trọng nhất cho ví dụ này (số minh hoạ):
- Hướng 1 `e₁`: co duỗi đồng loạt 4 ngón (trỏ, giữa, áp út, út), hệ số dương = co lại.
- Hướng 2 `e₂`: co duỗi riêng ngón trỏ, hệ số dương = duỗi thẳng.

Với hệ số PCA `c = (c₁, c₂, 0, 0, 0, 0) = (0.8, -0.9, 0, 0, 0, 0)` (4 chiều còn lại không dùng trong ví dụ này):

```
θ_hand = θ̄_hand + c₁·e₁ + c₂·e₂
       = θ̄_hand + 0.8·(co 4 ngón) + (-0.9)·(co ngón trỏ)
```

Vì `c₂ = -0.9` (âm, và hướng e₂ định nghĩa "dương = duỗi thẳng ngón trỏ" theo một quy ước ngược trong ví dụ này — số minh hoạ tự chọn), kết quả net là: 4 ngón co lại mạnh (do c₁=0.8 dương), riêng ngón trỏ được "kéo ngược" theo hướng duỗi thẳng một lượng lớn (do |c₂|=0.9) — tạo ra đúng hình dạng "chỉ tay". Đây minh hoạ nguyên lý: chỉ với **2 con số** (thay vì 15 góc khớp riêng biệt của 5 ngón × 3 khớp/ngón), không gian PCA đã tái tạo được một cử chỉ tay phức tạp — đây chính là lý do MoSh++ (dùng để fit AMASS) chọn tối ưu hoá trong không gian PCA thay vì không gian góc khớp đầy đủ: **ít tham số hơn → bài toán tối ưu hoá ổn định hơn, ít bị mắc kẹt ở cấu hình tay phi thực tế**.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | SMPL (gốc) | SMPL-H (trung gian) | SMPL-X |
|---|---|---|---|
| Số vertex | 6,890 | 6,890 (thân) | 10,475 |
| Số khớp | 23 | 23 + khớp tay (MANO) | 54 (thân + tay + cổ + hàm + 2 mắt) |
| Có tay chi tiết (ngón)? | Không | Có | Có |
| Có khuôn mặt biểu cảm? | Không | Không | Có (expression ψ, FLAME) |
| Số tham số mô hình | Ít hơn (β 10 + θ 72) | Trung bình | 119 (β 10 + θ 75 + hand-PCA 24 + ψ 10) |
| Dataset dùng trong dự án này | — (nền tảng lý thuyết) | **OMOMO** (chỉ cần tay chính xác, không cần biểu cảm mặt) | **AMASS** hỗ trợ cả 2 (SMPL-H và SMPL-X tuỳ bản tải) |
| Khi nào chọn | Khi chỉ cần dáng đi/tư thế toàn thân thô, không cần tay/mặt | Khi cần tay chính xác (cầm nắm) nhưng không quan tâm biểu cảm mặt — nhẹ hơn SMPL-X | Khi cần cả tay chính xác lẫn biểu cảm mặt (ví dụ VLA cần hiểu ý định qua cử chỉ + nét mặt) |

> Ghi chú: OMOMO công bố hỗ trợ cả SMPL-H và SMPL-X tuỳ phiên bản dữ liệu tải về (theo trang dự án và GitHub chính thức) — vì bài toán loco-manipulation chủ yếu cần độ chính xác bàn tay, không nhất thiết cần biểu cảm mặt.

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "SMPL-X chỉ là SMPL với nhiều vertex hơn để hình ảnh đẹp hơn."**
   Vì sao sai: việc tăng N (số vertex) chỉ là hệ quả — thay đổi cốt lõi là **thêm bậc tự do điều khiển được** (K từ 23 lên 54 khớp: thêm khớp ngón tay, hàm, mắt) và **thêm nhóm tham số mới tách biệt** (ψ cho expression). Đây là thay đổi về không gian điều khiển, không chỉ độ phân giải hiển thị.
   Hiểu đúng: SMPL-X mở rộng cả không gian hình học (vertex) LẪN không gian điều khiển (khớp, tham số biểu cảm).

2. **Hiểu nhầm: "Vì SMPL-X có nhiều tham số hơn (119 so với ~82 của SMPL), nó luôn tốt hơn, nên dùng SMPL-X cho mọi trường hợp."**
   Vì sao sai: với bài toán chỉ cần dáng đi/tư thế toàn thân (ví dụ theo dõi locomotion thuần tuý, không quan tâm ngón tay), thêm 119 tham số làm bài toán fit (MoSh++) và huấn luyện tốn tài nguyên hơn không cần thiết, dễ overfit vào chi tiết tay/mặt không liên quan tới mục tiêu.
   Hiểu đúng: chọn SMPL vs SMPL-H vs SMPL-X tuỳ theo bài toán cụ thể cần độ chi tiết ở đâu (xem bảng so sánh) — không mặc định "cái mới nhất luôn tốt nhất".

## 🏗️ Ví dụ minh hoạ trong dự án này

AMASS (xem bài giảng riêng) cung cấp dữ liệu ở cả định dạng SMPL-H và SMPL-X tuỳ subset; OMOMO dùng SMPL-H/SMPL-X vì bài toán loco-manipulation *là* tương tác tay-vật thể — nếu dùng SMPL gốc (bàn tay là khối cụt), sẽ không thể xác định chính xác điểm tiếp xúc giữa ngón tay và vật thể trong dữ liệu huấn luyện, làm hỏng chất lượng nhãn (label) cho bài toán "sinh chuyển động người từ chuyển động vật thể". Khi GMR hoặc SOMA-retargeter trong `02-motion-retargeting/` xử lý một file SMPL-X, chúng cần retarget riêng phần khớp tay (54−23=31 khớp bổ sung) sang bàn tay robot (nếu robot có tay khéo léo/dexterous hand) tách biệt với phần retarget thân — đây là lý do "vì sao SMPL-X" trực tiếp ảnh hưởng tới độ phức tạp của bước retargeting phía sau.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **ETCH-X: Robustify Expressive Body Fitting to Clothed Humans with Composable Datasets** (arXiv 2604.08548, 2026): công trình này giải quyết trực tiếp một hạn chế đã biết của SMPL-X — việc fit (khớp) mô hình vào ảnh/video người **mặc quần áo** (clothed humans) thường kém chính xác vì SMPL-X vốn được thiết kế fit vào hình dáng cơ thể trần, quần áo rộng làm méo ước lượng shape/pose. ETCH-X đề xuất bộ dữ liệu composable và phương pháp fitting bền vững hơn cho người mặc quần áo — cho thấy hướng nghiên cứu hiện tại tập trung vào việc mở rộng ĐỘ BỀN VỮNG của quy trình fit SMPL-X trong điều kiện thực tế (ảnh/video ngoài đời, có quần áo, occlusion), không phải thay đổi bản thân công thức toán học M(β,θ,ψ).

2. **Retargeting hiện đại vẫn dùng trực tiếp SMPL-X làm chuẩn đầu vào**: công cụ GMR (*General Motion Retargeting*, ICRA 2026, [github.com/YanjieZe/GMR](https://github.com/YanjieZe/GMR)) công bố hỗ trợ trực tiếp "SMPLX body models" làm một trong các định dạng input chính để retarget sang nhiều robot humanoid (Unitree G1, G1 with Hands, H1, H1-2) — xác nhận SMPL-X (không phải SMPL gốc) là định dạng thực tế được các pipeline retargeting mới nhất trong lĩnh vực humanoid ưu tiên hỗ trợ, đặc biệt khi robot có tay khéo léo (G1 with Hands) cần dữ liệu ngón tay từ SMPL-X.

3. Đây là kiến thức nền tảng (cơ chế toán học của SMPL-X) khá ổn định kể từ 2019 — phần "cải tiến" thực chất tập trung vào ỨNG DỤNG (fitting bền vững hơn với dữ liệu thực tế nhiễu — ETCH-X) và HỆ SINH THÁI CÔNG CỤ (GMR, SOMA hỗ trợ trực tiếp) hơn là thay đổi bản thân công thức blend skinning + PCA hand pose + expression space đã mô tả ở trên.

## ❓ Câu hỏi tự kiểm tra

1. SMPL-X hợp nhất những mô hình chuyên biệt nào, và mỗi mô hình đóng góp phần nào?
   <details><summary>Gợi ý đáp án</summary>MANO (bàn tay, pose space PCA) và FLAME (khuôn mặt, expression space) — hợp nhất cùng cơ chế body của SMPL thành 1 mô hình toàn thân duy nhất, học lại từ dữ liệu scan có đủ chi tiết tay+mặt.</details>

2. Vì sao ψ (expression) phải tách biệt hoàn toàn khỏi β (shape)?
   <details><summary>Gợi ý đáp án</summary>β mã hoá hình dáng CỐ ĐỊNH của một người (không đổi theo thời gian trong 1 phiên), còn ψ mã hoá biểu cảm THAY ĐỔI liên tục (cười, nhăn mặt...) độc lập với hình dáng đầu/mặt tĩnh — gộp chung sẽ khiến mô hình không thể biểu diễn "cùng 1 người, nhiều biểu cảm khác nhau" một cách nhất quán.</details>

3. Tại sao MANO dùng không gian PCA giảm chiều cho tay thay vì tham số hoá đầy đủ từng khớp ngón (axis-angle cho cả 15 khớp/tay)?
   <details><summary>Gợi ý đáp án</summary>Giảm số chiều tham số cần tối ưu (ví dụ 6-24 chiều thay vì 45 chiều/tay), giúp bài toán fit (MoSh++) ổn định hơn, ít rơi vào cấu hình tay phi thực tế — vì không gian PCA chỉ bao phủ các tư thế tay khả dĩ đã học từ dữ liệu thật.</details>

4. Trong dự án này, vì sao OMOMO cần SMPL-H/SMPL-X mà không dùng SMPL gốc?
   <details><summary>Gợi ý đáp án</summary>Vì bài toán loco-manipulation của OMOMO cần xác định chính xác điểm tiếp xúc tay-vật thể (cầm, nắm, đặt) — SMPL gốc không có ngón tay chi tiết nên không thể gán nhãn contact chính xác.</details>

5. Nếu một robot không có bàn tay khéo léo (chỉ có gripper đơn giản 1 DOF), việc dùng SMPL-X (thay vì SMPL) làm nguồn retargeting có còn mang lại lợi ích tương xứng với chi phí tính toán thêm không? Giải thích.
   <details><summary>Gợi ý đáp án</summary>Không nhiều lợi ích tương xứng — nếu robot chỉ có gripper 1 DOF, thông tin chi tiết 31 khớp ngón tay bổ sung của SMPL-X phần lớn bị "bỏ phí" vì không có nơi ánh xạ tới trên robot; SMPL/SMPL-H có thể đủ và nhẹ hơn cho pipeline retargeting trong trường hợp này.</details>

## 📝 Bài tập thực hành

1. **Đọc code thật**: trong repo [`vchoutas/smplx`](https://github.com/vchoutas/smplx), so sánh file định nghĩa class `SMPL`, `SMPLH`, và `SMPLX` (thường trong `smplx/body_models.py`) — liệt kê chính xác sự khác biệt về số lượng tham số đầu vào (`betas`, `body_pose`, `left_hand_pose`, `right_hand_pose`, `expression`, `jaw_pose`, `leye_pose`, `reye_pose`) giữa 3 class này, đối chiếu với số liệu 119 tham số đã nêu ở mục Định nghĩa.

2. **Tính tay biến thể khác**: lặp lại ví dụ PCA tay ở trên nhưng cho cử chỉ "nắm đấm" (fist) thay vì "chỉ tay" — giả sử hệ số PCA minh hoạ `c = (0.9, 0.85, 0, 0, 0, 0)` (cả 2 hướng đều dương, nghĩa là co tất cả các ngón kể cả trỏ) — mô tả bằng lời hình dạng bàn tay kết quả và giải thích vì sao khác với ví dụ "chỉ tay" trong bài.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

SMPL-X mở rộng SMPL bằng cách hợp nhất pose space của MANO (bàn tay) và expression space của FLAME (khuôn mặt) vào cùng một hàm khả vi duy nhất `M(β,θ,ψ)`, tăng từ 6,890 lên 10,475 vertex và từ 23 lên 54 khớp, tổng 119 tham số (10 shape + 75 pose thân/mắt/hàm + 24 hand-PCA + 10 expression) — mang lại độ chi tiết ngón tay và biểu cảm mặt mà SMPL gốc không có, đây chính là lý do AMASS và đặc biệt OMOMO (bài toán loco-manipulation cần vị trí tay chính xác) chọn SMPL-X/SMPL-H làm định dạng biểu diễn, và các công cụ retargeting hiện đại (GMR) hỗ trợ trực tiếp SMPL-X làm input chuẩn khi robot đích có bàn tay khéo léo.
