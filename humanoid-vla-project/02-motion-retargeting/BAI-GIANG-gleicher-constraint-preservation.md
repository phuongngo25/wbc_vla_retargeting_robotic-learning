# Bài giảng: Ý tưởng gốc Gleicher (1998) — constraint preservation

*(Thuộc mảng: Motion Retargeting)*

## 🎯 Mục tiêu bài học

- Giải thích được vì sao retargeting nên bảo toàn **constraint** thay vì sao chép góc khớp.
- Phân biệt được constraint, objective và motion displacement trong bài toán của Gleicher.
- Mô tả được vì sao **spacetime optimization** xét cả một đoạn thời gian tốt hơn IK độc lập từng frame.
- Viết được một bài toán tối ưu đồ chơi có equality constraint, inequality constraint và mục tiêu giữ gần chuyển động gốc.
- Tính tay được nghiệm tối thiểu thay đổi cho một ví dụ hai frame đơn giản.
- Liên hệ được tư tưởng năm 1998 với GMR, SOMA-retargeter và kinodynamic retargeting hiện đại.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Ở bài trước, ta đã thấy copy trực tiếp góc khớp người sang robot thất bại vì khác số bậc tự do, tỷ lệ cơ thể và giới hạn vật lý.

Nhưng biết cách làm sai chưa cho ta biết tiêu chí của một lời giải đúng.

Gleicher (1998) đưa ra một thay đổi góc nhìn quan trọng:

> Đừng cố giữ mọi con số của chuyển động nguồn; hãy chỉ rõ những tính chất nào làm nên ý nghĩa của chuyển động và buộc chuyển động đích giữ các tính chất ấy.

Các tính chất bắt buộc đó được biểu diễn thành **constraints** — ràng buộc.

Ví dụ:

- bàn chân phải ở trên sàn trong pha stance;
- bàn chân không trượt trong suốt footplant;
- hai tay chạm hộp tại thời điểm nhấc hộp;
- khoảng cách giữa hai tay phù hợp bề rộng hộp;
- khớp luôn nằm trong giới hạn cho phép.

Khái niệm này nằm ngay sau câu hỏi “vì sao không thể copy góc” và trước các bài riêng về skeleton mapping hay IK.

Nó trả lời câu hỏi thiết kế:

```text
Khi không thể giữ nguyên toàn bộ chuyển động nguồn,
ta phải ưu tiên giữ điều gì?
```

## 🧠 Trực giác

### Góc nhìn 1: Chuyển thể một bản nhạc sang nhạc cụ khác

Giả sử một bản nhạc được viết cho piano nhưng phải biểu diễn bằng guitar.

Ta không thể sao chép từng phím piano sang một dây guitar theo ánh xạ 1-1:

- số phím và dây khác nhau;
- cách tạo âm khác nhau;
- một số hợp âm không thể bấm y hệt;
- quãng âm khả dụng khác nhau.

Người chuyển soạn phải quyết định điều gì cần bảo toàn:

- giai điệu chính;
- nhịp;
- điểm nhấn;
- cảm giác nhanh/chậm;
- một số hoà âm quan trọng.

Sau đó họ tìm cách chơi mới phù hợp với guitar.

Trong retargeting:

| Chuyển soạn nhạc | Motion retargeting |
|---|---|
| Bản piano gốc | Chuyển động người gốc |
| Guitar | Robot có morphology khác |
| Giai điệu/nhịp cần giữ | Contact, vị trí end-effector, đặc tính thời gian |
| Cách bấm guitar mới | Góc khớp robot được tính lại |

Phép loại suy đúng ở chỗ nó tách **nội dung cần bảo toàn** khỏi **cách biểu diễn trên hệ nguồn**.

Giới hạn của loại suy:

- âm nhạc không chịu trọng lực hay ma sát;
- “nghe giống” là đánh giá cảm thụ, còn constraint robot thường là phương trình cụ thể;
- một chuyển động robot còn phải khả thi về động lực học, không chỉ tương tự về hình thức.

### Góc nhìn 2: Refactor phần mềm nhưng phải giữ test

Hãy tưởng tượng ta thay toàn bộ implementation của một thư viện.

Mã mới có thể:

- dùng cấu trúc dữ liệu khác;
- chia module khác;
- chạy trên phần cứng khác;
- không còn dòng code nào giống mã cũ.

Tuy vậy, các **contract** quan trọng vẫn phải đúng:

- input hợp lệ cho output đúng;
- lỗi phải được xử lý theo quy ước;
- invariant dữ liệu không bị phá;
- test tích hợp vẫn vượt qua.

Các test đóng vai trò giống constraint.

Ta không bảo toàn implementation; ta bảo toàn hành vi quan trọng.

Trong motion retargeting, góc khớp nguồn giống implementation cũ.

Việc “tay chạm vật” hoặc “chân không trượt” giống test hành vi.

Phép loại suy đúng ở chỗ constraint mô tả **điều kết quả bắt buộc phải làm được**, còn cách đạt kết quả có thể thay đổi.

Giới hạn của loại suy:

- test phần mềm thường cho kết quả pass/fail rời rạc;
- constraint robot có thể là equality, inequality hoặc mục tiêu mềm liên tục;
- nhiều constraint robot có thể xung đột vì giới hạn hình học và vật lý.

## 📐 Định nghĩa chính xác

### 1. Chuyển động là một hàm theo thời gian

Gọi cấu hình nhân vật tại thời điểm `t` là:

```text
q(t) = [root position, root orientation, joint angles]ᵀ
```

Chuyển động nguồn là hàm:

```text
q₀(t)
```

Chuyển động sau retarget là:

```text
q(t)
```

Paper biểu diễn phần hiệu chỉnh bằng **motion displacement**:

```text
d(t) = q(t) − q₀(t)
```

hay tương đương:

```text
q(t) = q₀(t) + d(t)
```

Ta không muốn `d(t)` lớn hoặc chứa dao động không phù hợp, vì điều đó làm mất “chất” của chuyển động ban đầu.

### 2. Constraint là điều bắt buộc phải đúng

Một equality constraint có dạng khái quát:

```text
cᵢ(q(tᵢ)) = 0
```

Ví dụ, bàn tay phải phải ở vị trí vật thể `p_object` tại `t_pick`:

```text
FK_hand(q(t_pick)) − p_object = 0
```

Một inequality constraint có thể viết:

```text
gⱼ(q(tⱼ)) ≤ 0
```

Ví dụ một góc khớp phải nằm trong biên:

```text
q_min ≤ q(t) ≤ q_max
```

Paper gốc liệt kê những họ constraint thực dụng:

1. tham số nằm trong một khoảng, hữu ích cho joint limits;
2. một điểm trên nhân vật ở vị trí cụ thể, hữu ích cho footplant hoặc nắm vật;
3. một điểm nằm trong một vùng, chẳng hạn ở trên mặt sàn;
4. một điểm ở cùng vị trí tại hai thời điểm, hữu ích để chống foot skating;
5. một điểm đi theo đường của điểm khác;
6. hai điểm giữ khoảng cách quy định;
7. vector nối hai điểm giữ hướng quy định.

### 3. Objective chọn lời giải tốt trong nhiều lời giải hợp lệ

Constraints thường không xác định duy nhất một chuyển động.

Nhiều tư thế khác nhau có thể đưa tay tới cùng một vật.

Vì vậy cần **objective function** (hàm mục tiêu) để chọn nghiệm.

Một mục tiêu cơ bản là giữ chuyển động mới gần chuyển động gốc:

```text
minimize  E(q) = ∫ ||q(t) − q₀(t)||² dt
```

Nhưng chỉ giảm sai khác tham số theo từng thời điểm có thể tạo thay đổi đột ngột.

Gleicher nhấn mạnh phải tránh thêm **frequency content** không mong muốn.

Một cách hiểu khái quát là phạt cả độ lớn và độ gấp của displacement:

```text
minimize  E(d) = α ∫ ||d(t)||² dt + β ∫ ||ḋ(t)||² dt
```

Đây là biểu thức giảng giải để thấy vai trò hai thành phần; paper gốc dùng biểu diễn displacement và tiêu chí tần số cụ thể của nó.

### 4. Spacetime constraints

**Spacetime optimization** giải một bài toán lớn trên cả khoảng thời gian, không giải từng frame độc lập.

Dạng tổng quát:

```text
find      q(t), t ∈ [0, T]

minimize  E(q, q₀)

subject to
          cᵢ(q(t₁), q(t₂), ...) = 0
          gⱼ(q(t₁), q(t₂), ...) ≤ 0
```

Điểm quan trọng là một constraint có thể liên kết nhiều thời điểm.

Ví dụ chống trượt chân:

```text
FK_foot(q(t₁)) = FK_foot(q(t₂))
```

Vị trí chân không nhất thiết được ấn định trước; solver được phép chọn vị trí hợp lý, nhưng khi đã plant thì vị trí phải giống nhau giữa các thời điểm.

## ⚙️ Cơ chế hoạt động — từng bước

### Bước 1: Chọn motion source và target character

Input là một chuyển động đã tồn tại `q₀(t)`.

Target có tỷ lệ hoặc cấu trúc khác.

Paper gốc tập trung chủ yếu vào nhân vật có cấu trúc khớp giống nhau nhưng chiều dài đoạn khác, rồi bàn thêm trường hợp cấu trúc kém tương đồng hơn.

### Bước 2: Xác định feature mang ý nghĩa

Không phải mọi chi tiết đều quan trọng như nhau.

Trong ví dụ nhấc hộp:

- tay phải tới một phía hộp;
- tay trái tới phía còn lại;
- hai tay giữ đúng khoảng cách khi mang;
- chân plant không trượt;
- chân không xuyên sàn.

### Bước 3: Biến feature thành constraint

Mỗi feature được viết thành một quan hệ có thể kiểm tra.

```text
“tay chạm hộp”
      ↓
FK_hand(q(t_pick)) = p_handle

“chân không trượt trong stance”
      ↓
FK_foot(q(t_a)) = FK_foot(q(t_b))

“khớp không vượt biên”
      ↓
q_min ≤ q(t) ≤ q_max
```

### Bước 4: Chọn objective giữ chất chuyển động

Nếu chỉ thoả constraint, solver có thể tạo một chuyển động rất khác nguồn.

Objective yêu cầu:

- thay đổi vừa đủ;
- tránh displacement quá lớn;
- tránh thêm dao động cao tần không phù hợp;
- giữ đặc tính thời gian quan trọng của motion gốc.

### Bước 5: Chọn biểu diễn displacement theo thời gian

Thay vì cho phép mọi frame có một correction độc lập, displacement được biểu diễn bằng đường cong có số tham số ít hơn.

Trực giác:

```text
correction độc lập từng frame
  d₀ d₁ d₂ d₃ d₄ d₅ ...  → dễ giật

correction bằng curve trơn
  control points ── interpolation ──▶ d(t) trơn hơn
```

Biểu diễn này giới hạn loại thay đổi mà solver có thể tạo và giúp kiểm soát frequency content.

### Bước 6: Giải constrained optimization trên toàn đoạn

Solver xét đồng thời:

- yêu cầu ở frame hiện tại;
- constraint sẽ xuất hiện trong tương lai;
- constraint vừa xảy ra trong quá khứ;
- tổng mức thay đổi trên toàn motion.

Do đó nó có thể bắt đầu điều chỉnh trước thời điểm bàn tay chạm hộp, thay vì đợi đúng frame chạm mới “bẻ” tay tới mục tiêu.

### Bước 7: Kiểm tra constraint và chất lượng chuyển động

Sau khi tối ưu, cần kiểm tra hai lớp:

```text
Lớp bắt buộc: constraint có được thoả không?
Lớp chất lượng: motion có trơn và còn giống nguồn không?
```

Một lời giải có thể thoả tay chạm hộp nhưng làm chân trượt.

Một lời giải có thể giữ chân nhưng thêm một cú giật tay.

Retarget tốt phải quản lý cả hai lớp.

### Sơ đồ toàn bộ cơ chế

```text
┌──────────────────────────────────────┐
│ Chuyển động nguồn q₀(t)              │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│ Chọn feature quan trọng              │
│ footplant · hand-object · distance   │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│ Viết equality/inequality constraints │
└──────────────────┬───────────────────┘
                   │
                   ├──────────────┐
                   ▼              ▼
┌──────────────────────┐  ┌──────────────────────┐
│ Objective giữ gần q₀ │  │ Displacement curve   │
│ và giữ frequency     │  │ d(t), ít tham số     │
└──────────┬───────────┘  └──────────┬───────────┘
           └─────────────┬───────────┘
                         ▼
          ┌──────────────────────────┐
          │ Spacetime optimization   │
          │ xét toàn khoảng [0, T]   │
          └─────────────┬────────────┘
                        ▼
          ┌──────────────────────────┐
          │ q(t) = q₀(t) + d(t)      │
          │ constraint đúng, motion  │
          │ thay đổi vừa đủ          │
          └──────────────────────────┘
```

## 🧮 Ví dụ tính tay / walkthrough cụ thể

Đây là ví dụ minh hoạ tự chọn để hiểu constrained optimization, không phải số liệu thực nghiệm từ paper.

### Bài toán

Ta chỉ xét một toạ độ ngang `x` của bàn chân ở hai frame liên tiếp trong pha stance.

Chuyển động nguồn sau khi áp sang nhân vật mới cho:

```text
x₁⁰ = 0.02 m
x₂⁰ = 0.08 m
```

Nếu giữ nguyên, chân trượt `0.06 m` giữa hai frame.

Constraint preservation yêu cầu chân plant không trượt:

```text
x₁ = x₂
```

Ta muốn thay đổi ít nhất so với nguồn:

```text
minimize E = (x₁ − 0.02)² + (x₂ − 0.08)²
subject to x₁ − x₂ = 0
```

### Cách 1: Thay constraint trực tiếp

Vì `x₁ = x₂`, đặt cả hai bằng `x`:

```text
E(x) = (x − 0.02)² + (x − 0.08)²
```

Khai triển:

```text
E(x) = x² − 0.04x + 0.0004
     + x² − 0.16x + 0.0064

     = 2x² − 0.20x + 0.0068
```

Lấy đạo hàm:

```text
dE/dx = 4x − 0.20
```

Cho đạo hàm bằng 0:

```text
4x − 0.20 = 0
x = 0.05 m
```

Nghiệm là:

```text
x₁ = x₂ = 0.05 m
```

### Kiểm tra mức thay đổi

Frame 1 dịch:

```text
d₁ = 0.05 − 0.02 = +0.03 m
```

Frame 2 dịch:

```text
d₂ = 0.05 − 0.08 = −0.03 m
```

Objective:

```text
E = 0.03² + (−0.03)²
  = 0.0009 + 0.0009
  = 0.0018 m²
```

Constraint:

```text
x₁ − x₂ = 0.05 − 0.05 = 0
```

Chân không còn trượt và correction được chia đều cho hai frame.

### Cách 2: Dùng Lagrange multiplier

Lagrangian:

```text
L(x₁, x₂, λ)
= (x₁ − 0.02)² + (x₂ − 0.08)² + λ(x₁ − x₂)
```

Điều kiện dừng:

```text
∂L/∂x₁ = 2(x₁ − 0.02) + λ = 0
∂L/∂x₂ = 2(x₂ − 0.08) − λ = 0
∂L/∂λ  = x₁ − x₂ = 0
```

Cộng hai phương trình đầu:

```text
2x₁ + 2x₂ − 0.20 = 0
```

Vì `x₁ = x₂ = x`:

```text
4x = 0.20
x = 0.05 m
```

Ta thu được cùng nghiệm.

### Vì sao ví dụ này thể hiện đúng tư tưởng Gleicher?

- Ta không bắt góc khớp hai frame giống nguồn.
- Ta chỉ bắt tính chất quan trọng: chân không di chuyển khi plant.
- Trong số vô hạn vị trí plant khả dĩ, objective chọn vị trí làm tổng thay đổi nhỏ nhất.
- Hai thời điểm được giải chung trong một bài toán.

### Mở rộng thêm một inequality constraint

Giả sử mặt sàn cho phép vùng đặt chân:

```text
0.00 ≤ x ≤ 0.04 m
```

Nghiệm không ràng buộc vùng là `0.05 m`, vượt biên trên.

Nghiệm khả thi gần nhất trở thành:

```text
x₁ = x₂ = 0.04 m
```

Objective mới:

```text
E = (0.04 − 0.02)² + (0.04 − 0.08)²
  = 0.02² + (−0.04)²
  = 0.0004 + 0.0016
  = 0.0020 m²
```

Chi phí tăng từ `0.0018` lên `0.0020 m²` vì solver phải tôn trọng thêm constraint.

Đây là trade-off cơ bản:

```text
nhiều constraint hơn
→ miền nghiệm nhỏ hơn
→ có thể phải thay đổi motion nhiều hơn
```

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Spacetime constraint preservation | IK độc lập từng frame | Low-pass sau IK |
|---|---|---|---|
| Phạm vi thời gian | Cả đoạn motion | Một frame | Lọc chuỗi sau khi đã giải |
| Biết constraint tương lai | Có | Không | Không khi giải IK |
| Liên kết hai thời điểm | Trực tiếp bằng constraint | Không | Chỉ gián tiếp qua filter |
| Giữ contact chính xác | Có thể áp constraint cứng | Có thể đúng từng frame | Có thể bị filter làm sai |
| Kiểm soát độ mượt | Trong objective/biểu diễn | Không tự nhiên có | Có, nhưng hậu xử lý |
| Nguy cơ snap tại lúc contact bắt đầu | Thấp hơn nếu tối ưu cả đoạn | Cao hơn | Giảm snap nhưng dễ vi phạm contact |
| Chi phí tính toán | Bài toán lớn hơn | Nhỏ, dễ real-time | Thêm một bước rẻ |
| Khi phù hợp | Offline, cần nhất quán thời gian | Streaming hoặc bài toán cục bộ | Làm sạch đơn giản khi sai số contact chấp nhận được |

Không nên hiểu bảng này là “spacetime luôn tốt hơn”.

Gleicher lựa chọn bài toán toàn đoạn để bảo toàn đặc tính thời gian.

Các pipeline robot hiện đại còn phải cân bằng:

- latency;
- khối lượng dataset;
- constraint động lực học;
- khả năng chạy online;
- khả năng song song hoá.

Skeleton mapping và IK sẽ có bài giảng riêng; ở đây chúng chỉ là những cách tạo hoặc giải các quan hệ hình học nằm dưới constraint.

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

### 1. “Constraint preservation nghĩa là giữ nguyên mọi thứ”

Vì sao sai:

Nếu giữ nguyên mọi góc, vị trí và quỹ đạo, ta quay lại copy motion nguồn — điều không khả thi trên morphology mới.

Hiểu đúng:

Chỉ những feature được chọn mới trở thành constraint.

Phần còn lại được phép thay đổi theo objective.

### 2. “Constraint và objective là một”

Vì sao sai:

Constraint định nghĩa miền nghiệm hợp lệ.

Objective xếp hạng các nghiệm hợp lệ.

Hiểu đúng:

“Chân phải ở yên” có thể là constraint; “thay đổi ít nhất so với nguồn” là objective.

### 3. “Giải IK ở mỗi frame rồi lọc là tương đương spacetime optimization”

Vì sao sai:

Low-pass filter có thể kéo end-effector rời vị trí constraint vừa đạt được.

Hiểu đúng:

Spacetime formulation tối ưu độ mượt và constraint đồng thời.

### 4. “Constraint càng nhiều càng tốt”

Vì sao sai:

Constraint có thể xung đột hoặc làm bài toán vô nghiệm.

Constraint quá đặc thù còn làm motion khó tái sử dụng.

Hiểu đúng:

Chọn tập constraint nhỏ nhưng phản ánh đúng semantic quan trọng.

### 5. “Paper Gleicher đã bảo đảm động lực học robot thật”

Vì sao sai:

Paper tự nêu sự đơn giản hoá và chỉ ra ví dụ mất cân bằng vì hệ thống chưa xét gravity hoặc posture đầy đủ.

Hiểu đúng:

Đây là nền tảng constraint-based motion adaptation, không phải một pipeline robot-ready hoàn chỉnh.

### 6. “Frequency preservation nghĩa là giữ nguyên tốc độ phát motion”

Vì sao sai:

Frequency content nói về cấu trúc nhanh/chậm của tín hiệu chuyển động và các thay đổi được thêm vào, không chỉ frame rate playback.

Hiểu đúng:

Một correction đột ngột thêm thành phần cao tần và tạo jerk; correction trải quá rộng cũng có thể làm sai đặc tính motion.

## 🏗️ Ví dụ minh hoạ trong dự án này

### Ví dụ 1: Retarget một bước đi sang Unitree G1

Các feature nên bảo toàn:

- chân stance ở trên sàn;
- chân stance không trượt;
- thời điểm chuyển contact hợp lý;
- root không tạo discontinuity lớn.

Các đại lượng không cần giữ tuyệt đối:

- góc gối đúng bằng người;
- độ dài sải chân theo đơn vị mét của người;
- vị trí pelvis tuyệt đối của skeleton nguồn.

### Ví dụ 2: Retarget động tác nhấc hộp

Constraint semantic:

```text
t = t_pick:
  hand_left  = left_handle
  hand_right = right_handle

t ∈ [t_pick, t_release]:
  distance(hand_left, hand_right) = box_width
```

Nếu G1 có tay ngắn hơn người, solver có thể:

- điều chỉnh bước chân;
- đổi vị trí pelvis;
- thay góc vai và khuỷu;
- vẫn giữ hai tay ở đúng quan hệ với hộp.

### Liên hệ pipeline repo

```text
AMASS / LAFAN1 / motion nguồn
             │
             ▼
   Chọn correspondence + constraint
             │
             ▼
  GMR hoặc SOMA-retargeter giải hình học
             │
             ▼
 contact stabilization + joint/velocity limits
             │
             ▼
 reference motion cho WBC / imitation learning
```

Ý tưởng constraint preservation xuất hiện xuyên suốt pipeline, dù solver hiện đại không nhất thiết dùng đúng implementation spacetime năm 1998.

## 🔥 Cập nhật hiện đại / SOTA gần đây

### 1. GMR 2025: chất lượng constraint không chỉ ảnh hưởng hình ảnh

Araujo, Ze, Xu, Wu và Liu (2025), *Retargeting Matters: General Motion Retargeting for Humanoid Motion Tracking*, đánh giá có hệ thống ảnh hưởng của retargeting lên policy tracking.

Paper báo cáo các artifact như ground penetration, self-intersection và jump ở waist có thể làm giảm robustness của policy downstream.

Điều này mở rộng ý tưởng Gleicher:

```text
1998: constraint sai → animation nhìn sai
2025: constraint/artifact sai → dữ liệu reference khó track
                         → policy học kém bền vững hơn
```

GMR dùng non-uniform local scaling và two-stage optimization để xử lý các vấn đề scaling mà các baseline tạo ra.

Nguồn:

- João Pedro Araujo et al. (2025), [arXiv:2510.02252](https://arxiv.org/abs/2510.02252).
- [PDF chính thức của nhóm tác giả](https://jiajunwu.com/papers/gmr_icra.pdf).

### 2. SOMA-retargeter: constraint preservation được đóng gói thành pipeline công cụ

README chính thức hiện tại của NVIDIA SOMA-retargeter mô tả pipeline:

1. đọc SOMA BVH;
2. scale theo tỷ lệ robot;
3. giải IK per-frame;
4. stabilize contact và clamp joint limits;
5. xuất root pose cùng actuated joint values.

Đây là phiên bản thực dụng của cùng tư tưởng:

- constraint hình học được xử lý trong scaling/IK;
- constraint contact và joint range được xử lý tường minh;
- output vẫn được cảnh báo là kinematic và phải kiểm chứng trước hardware.

Nguồn:

- NVIDIA (đang phát triển), [SOMA-retargeter README](https://github.com/NVIDIA/soma-retargeter/blob/main/README.md).

### 3. KDMR 2026: từ kinematic constraint sang dynamics + contact complementarity

Zhang, Haener, Madabushi và Tucker (2026) đề xuất **Kinodynamic Motion Retargeting (KDMR)**.

Thay vì chỉ giữ constraint không gian, KDMR viết retargeting thành multi-contact whole-body trajectory optimization và thêm:

- rigid-body dynamics;
- contact complementarity constraints;
- ground reaction force (GRF) từ dữ liệu;
- phát hiện heel-toe contact.

Đây là một bước phát triển trực tiếp từ hạn chế Gleicher đã thừa nhận: constraint hình học chưa đủ để bảo đảm cân bằng và khả thi động lực học.

Nguồn:

- Xiaoyu Zhang et al. (2026), [arXiv:2603.09956](https://arxiv.org/abs/2603.09956), bản v2 ngày 30/07/2026.

### 4. Xu hướng chung: constraint vẫn sống, nhưng ngày càng giàu vật lý

Từ các nguồn trên có thể rút ra một nhận xét có căn cứ:

```text
Gleicher 1998
  feature constraints + temporal consistency
              │
              ▼
GMR / SOMA
  morphology scaling + IK + contact/joint constraints
              │
              ▼
KDMR 2026
  whole-body dynamics + complementarity + GRF/contact sequence
```

Ý tưởng “chọn điều cần bảo toàn rồi tối ưu phần còn lại” không bị thay thế.

Phần đang thay đổi là:

- constraint được mô tả giàu hơn;
- physics được đưa vào sâu hơn;
- chất lượng được đánh giá bằng khả năng train/track downstream;
- pipeline được tối ưu cho GPU và dataset lớn.

## ❓ Câu hỏi tự kiểm tra

1. Trong constraint preservation, vì sao không bắt mọi góc khớp đích bằng góc nguồn?

   <details><summary>Gợi ý đáp án</summary>Vì morphology, số DoF và giới hạn khác nhau. Góc nguồn là cách thực hiện trên embodiment nguồn; constraint nên giữ semantic như contact hoặc vị trí tương tác.</details>

2. Constraint và objective khác nhau thế nào?

   <details><summary>Gợi ý đáp án</summary>Constraint xác định lời giải nào hợp lệ; objective chọn lời giải tốt nhất trong tập hợp hợp lệ, chẳng hạn lời giải thay đổi ít nhất so với motion nguồn.</details>

3. Vì sao IK độc lập từng frame có thể tạo snap khi footplant bắt đầu?

   <details><summary>Gợi ý đáp án</summary>Frame trước không biết constraint sắp xuất hiện nên không chuẩn bị dần; tới frame bắt đầu contact, solver phải sửa đột ngột để đạt vị trí.</details>

4. Trong ví dụ số, tại sao nghiệm footplant là `0.05 m`?

   <details><summary>Gợi ý đáp án</summary>Constraint bắt hai frame có cùng vị trí. Trung bình `0.05 m` chia đều correction giữa `0.02` và `0.08`, làm tổng bình phương thay đổi nhỏ nhất.</details>

5. Vì sao low-pass kết quả IK có thể phá constraint?

   <details><summary>Gợi ý đáp án</summary>Filter trộn giá trị giữa các thời điểm để làm trơn; giá trị sau lọc không còn nhất thiết đúng vị trí end-effector mà IK đã đạt chính xác.</details>

6. KDMR 2026 bổ sung lớp constraint nào so với retargeting thuần kinematic?

   <details><summary>Gợi ý đáp án</summary>Rigid-body dynamics, contact complementarity và thông tin ground reaction force/contact heel-toe để tăng tính khả thi động lực học.</details>

## 📝 Bài tập thực hành

1. **Tính tay:** đổi dữ liệu ví dụ thành `x₁⁰ = −0.03 m`, `x₂⁰ = 0.09 m`, constraint `x₁=x₂`, và vùng hợp lệ `−0.02≤x≤0.02`. Tìm nghiệm trước và sau khi thêm inequality; tính objective ở cả hai trường hợp.
2. **Đọc paper/code:** đọc mục 3.5 “Sources of Constraints” trong [paper Gleicher](https://graphics.cs.wisc.edu/Papers/1998/Gle98/retarget-preprint.pdf), rồi mở cấu hình G1 của [SOMA-retargeter](https://github.com/NVIDIA/soma-retargeter). Lập bảng ánh xạ ít nhất bốn constraint năm 1998 với tham số/objective hiện đại có ý nghĩa tương ứng.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

Constraint preservation của Gleicher thay mục tiêu “copy góc khớp” bằng “giữ những feature làm nên ý nghĩa chuyển động”, chẳng hạn footplant, vị trí tay–vật và khoảng cách giữa hai điểm; các constraint xác định miền nghiệm, còn objective chọn chuyển động thay đổi ít và không thêm đặc tính tần số không mong muốn. Spacetime optimization xét cả một khoảng thời gian nên có thể chuẩn bị trước cho constraint tương lai và tránh snap mà IK độc lập từng frame dễ tạo ra. GMR và SOMA-retargeter hiện thực hoá tinh thần này bằng scaling, IK, contact và joint constraints, trong khi KDMR 2026 mở rộng sang rigid-body dynamics, contact complementarity và GRF. Ý tưởng cốt lõi từ năm 1998 vì thế vẫn còn nguyên giá trị: bảo toàn semantic bắt buộc, cho phép representation cụ thể thay đổi theo embodiment đích.
