# Bài giảng: Hierarchical QP (HQP) — giải nhiều tác vụ theo thứ tự ưu tiên

*(Thuộc mảng: Whole-Body Control (WBC))*

## 🎯 Mục tiêu bài học

- Giải thích được vì sao QP một tầng (weighted-sum) không đảm bảo ưu tiên tuyệt đối, và HQP giải quyết vấn đề đó bằng cơ chế nào.
- Trình bày được thuật toán HQP: giải tuần tự từng cấp, mỗi cấp bị ràng buộc trong null-space của mọi cấp cao hơn.
- Tính tay được một ví dụ HQP 2 cấp đơn giản (2 biến, 2 tác vụ ưu tiên khác nhau).
- Giải thích được khái niệm "generalized projector" và vì sao nó giúp HQP chạy online mà không cần nhân null-space tường minh mỗi bước.
- Phân biệt được HQP với QP một tầng và với null-space projection kiểu operational-space cổ điển.
- Nêu được ít nhất một hạn chế đã biết của HQP (ví dụ tính khả vi, chi phí tính toán) và một hướng khắc phục hiện đại.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Bài giảng "QP trong WBC" đã chỉ ra một hạn chế cụ thể: dùng trọng số `wᵢ` để cân bằng nhiều tác vụ trong MỘT hàm mục tiêu không bao giờ đảm bảo tuyệt đối 100% rằng tác vụ ưu tiên cao "không bao giờ" bị hy sinh — nó chỉ là một sự thỏa hiệp liên tục theo tỷ lệ trọng số. Với humanoid, đây là vấn đề nghiêm trọng: tác vụ **thăng bằng** (không ngã) phải luôn thắng tuyệt đối so với tác vụ **tay với đồ vật**, trong MỌI tình huống — kể cả tình huống bất ngờ chưa từng tinh chỉnh trọng số cho. Hierarchical QP (Escande, Mansard, Wieber 2014) là câu trả lời: thay vì một QP với trọng số, giải MỘT CHUỖI QP theo thứ tự ưu tiên cứng. Đây là "công cụ toán học đứng sau hầu hết WBC cổ điển hiện đại" theo đúng mô tả trong README của thư mục này.

## 🧠 Trực giác

### Góc nhìn 1: Quân đội tuân lệnh theo cấp bậc — cấp dưới chỉ được "tùy nghi" trong phạm vi không phá lệnh cấp trên

Hãy tưởng tượng một chuỗi mệnh lệnh quân sự nghiêm ngặt: tướng ra lệnh ưu tiên 1 (ví dụ "giữ vững phòng tuyến"), đại tá thực hiện lệnh ưu tiên 2 (ví dụ "tấn công mục tiêu phụ") nhưng CHỈ ĐƯỢC làm vậy nếu không vi phạm lệnh của tướng — nếu tấn công mục tiêu phụ có nguy cơ làm vỡ phòng tuyến, đại tá phải giảm bớt/từ bỏ hành động đó. HQP hoạt động y hệt: giải QP cấp 1 trước để có nghiệm tối ưu tuyệt đối cho tác vụ ưu tiên cao nhất, rồi giải QP cấp 2 nhưng bị RÀNG BUỘC không được làm xấu đi nghiệm cấp 1 dù chỉ một chút.

**Giới hạn của loại suy này:** trong quân đội, "không vi phạm lệnh cấp trên" thường là một ràng buộc định tính, mờ; trong HQP, đây là một ràng buộc TOÁN HỌC CHÍNH XÁC (nghiệm cấp dưới phải nằm trong tập nghiệm tối ưu — hoặc gần tối ưu trong một dung sai ε — của mọi cấp cao hơn), không có vùng xám.

### Góc nhìn 2: Lọc nước qua nhiều tầng màng — mỗi tầng chỉ "lọt qua" phần không ảnh hưởng tầng trước

Ở góc độ không gian toán học: hãy nghĩ null-space của tác vụ cấp 1 như một "tầng màng lọc" — chỉ cho phép các chuyển động khớp không ảnh hưởng tác vụ cấp 1 "lọt qua". Tác vụ cấp 2 chỉ được phép hoạt động trong phần "lọt qua" đó. Null-space của (cấp 1 + cấp 2) lại là một tầng màng lọc mịn hơn nữa, và tác vụ cấp 3 chỉ được hoạt động trong phần còn sót lại đó. Càng xuống cấp thấp, "không gian tự do" càng bị thu hẹp dần.

**Giới hạn của loại suy này:** màng lọc vật lý là cố định; các "tầng lọc" null-space trong HQP thay đổi động theo cấu hình robot ở MỌI bước điều khiển (giống Jacobian thay đổi theo q) — không phải một cấu trúc tĩnh được thiết lập một lần.

## 📐 Định nghĩa chính xác

Cho `p` cấp ưu tiên, mỗi cấp `k` có tác vụ với ma trận `Jₖ`, sai số mục tiêu `ẍₖ*`. HQP giải tuần tự:

**Cấp 1** (ưu tiên cao nhất):

```
minimize (over z₁)   ‖J₁z₁ − ẍ₁*‖²
subject to             A_ineq·z₁ ≤ b_ineq
```

Thu được nghiệm tối ưu `z₁*` và giá trị mục tiêu tối ưu `w₁* = ‖J₁z₁* − ẍ₁*‖²` (residual nhỏ nhất có thể đạt).

**Cấp 2:**

```
minimize (over z₂)   ‖J₂z₂ − ẍ₂*‖²
subject to             ‖J₁z₂ − ẍ₁*‖² ≤ w₁* + ε    (không làm xấu nghiệm cấp 1, ε là dung sai số học nhỏ)
                       A_ineq·z₂ ≤ b_ineq
```

**Cấp k bất kỳ** (tổng quát hóa):

```
minimize (over zₖ)   ‖Jₖzₖ − ẍₖ*‖²
subject to             ‖Jⱼzₖ − ẍⱼ*‖² ≤ wⱼ* + ε    với mọi j < k
                       A_ineq·zₖ ≤ b_ineq
```

Nghiệm cuối cùng dùng để điều khiển robot là `z_p*` (nghiệm của cấp thấp nhất) — vì mỗi ràng buộc ở cấp k đã "khóa" đúng phần đạt được ở mọi cấp cao hơn, `z_p*` đồng thời tối ưu cho cấp p VÀ giữ nguyên chất lượng tối ưu của mọi cấp cao hơn nó.

**Dạng tương đương bằng null-space projection** (cách Escande-Mansard-Wieber triển khai hiệu quả về mặt tính toán): thay vì thêm ràng buộc bất đẳng thức "không làm xấu" một cách tường minh (tốn kém), người ta dùng một **generalized projector** `P_k` (ma trận chiếu vào null-space của TẤT CẢ các cấp cao hơn k, tính đệ quy) sao cho nghiệm ở cấp k được tìm dưới dạng `z = z_{k-1}* + P_k·Δz`, với `Δz` là biến quyết định mới CHỈ ảnh hưởng trong null-space đó. Cách này tránh phải nhân tường minh ma trận null-space kích thước lớn ở mỗi bước, cho phép chạy online tốc độ cao trên robot thật.

## ⚙️ Cơ chế hoạt động — từng bước

```
┌────────────────────────────────────────────────────────────┐
│ 1. Đo trạng thái q, q̇; tính Jₖ, ẍₖ* cho mọi cấp k = 1..p        │
└──────────────────────────┬─────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ 2. Giải QP cấp 1: chỉ có ràng buộc bất đẳng thức vật lý        │
│    → z₁*, w₁* (residual tối ưu cấp 1)                          │
└──────────────────────────┬─────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ 3. Tính ma trận chiếu null-space P₂ = I − J₁ᵀ·J₁#               │
│    (hoặc dùng generalized projector đệ quy nếu p>2)             │
└──────────────────────────┬─────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ 4. Giải QP cấp 2 trong không gian con P₂ (biến Δz)              │
│    → z₂* = z₁* + P₂·Δz*                                        │
└──────────────────────────┬─────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ 5. Lặp lại bước 3-4 cho cấp 3, 4, ..., p                        │
│    (mỗi cấp chiếu vào null-space TÍCH LŨY của mọi cấp trên)     │
└──────────────────────────┬─────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ 6. Dùng nghiệm cấp cuối z_p* để tính τ, gửi xuống động cơ       │
└──────────────────────────────────────────────────────────────┘
                            │
                            └──── lặp lại từ bước 1 mỗi chu kỳ điều khiển
```

Chi phí tính toán: mỗi bước cần giải `p` QP nhỏ liên tiếp (thay vì 1 QP lớn) — chậm hơn về số lần gọi bộ giải, nhưng mỗi QP con thường có kích thước nhỏ hơn nhiều (chỉ còn biến tự do trong null-space đã thu hẹp), nên tổng chi phí vẫn khả thi cho tần số 100Hz-1kHz với cài đặt tốt.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh họa, số tự chọn để dễ hình dung — dùng đúng bối cảnh 1-biến như bài giảng QP để dễ so sánh trực tiếp.)*

Nhắc lại bối cảnh: biến điều khiển `u` (vô hướng), tác vụ 1 (ưu tiên CAO NHẤT, không còn trọng số nữa) muốn `u = 2`, tác vụ 2 (ưu tiên THẤP hơn) muốn `u = 5`, ràng buộc vật lý `u ≤ 3`.

**Cấp 1 — giải QP chỉ cho tác vụ ưu tiên cao nhất:**

```
minimize (u−2)²   subject to u ≤ 3
```

Cực tiểu không ràng buộc tại u=2, và 2 ≤ 3 nên ràng buộc không active. Nghiệm cấp 1: `u₁* = 2`, residual tối ưu `w₁* = (2−2)² = 0`.

**Cấp 2 — giải QP cho tác vụ ưu tiên thấp hơn, với ràng buộc "không được làm xấu nghiệm cấp 1":**

```
minimize (u−5)²   subject to (u−2)² ≤ 0 + ε,  u ≤ 3
```

Vì `w₁*=0` và ε rất nhỏ (gần 0), ràng buộc `(u−2)² ≤ ε` ép `u` phải gần bằng đúng 2 (sai số cực nhỏ cho phép do số học). Nghiệm cấp 2 do đó gần như bị "khóa cứng" ở `u₂* ≈ 2`, BẤT KỂ tác vụ 2 muốn u=5 mạnh đến đâu.

**So sánh trực tiếp với QP một tầng (từ bài giảng trước):** với trọng số w₁=1, w₂=4 (tác vụ 2 được ưu tiên trọng số CAO hơn tác vụ 1 trong công thức weighted-sum), nghiệm QP một tầng là u*=2.6 — bị "kéo" đáng kể về phía 5 dù mục đích ban đầu là muốn tác vụ 1 (u=2) quan trọng hơn. Với HQP, chỉ cần khai báo ĐÚNG THỨ TỰ ưu tiên (tác vụ "u=2" là cấp 1) mà KHÔNG CẦN chỉnh trọng số gì cả, nghiệm luôn giữ chặt u≈2 — đây chính là ưu điểm "đảm bảo cứng" mà QP một tầng không có.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Hierarchical QP (HQP) | QP một tầng (weighted-sum) | Null-space projection giải tích (Khatib) |
|---|---|---|---|
| Đảm bảo ưu tiên tuyệt đối | Có — cấp cao không bao giờ bị hy sinh | Không — chỉ là thỏa hiệp theo trọng số | Có, nhưng không xử lý tự nhiên ràng buộc bất đẳng thức |
| Cần tay chỉnh trọng số | Không (chỉ cần xếp thứ tự) | Có, và dễ vỡ khi tình huống mới | Không |
| Xử lý ràng buộc bất đẳng thức | Có, ở từng cấp | Có, trong một QP | Không tự nhiên |
| Số lần giải mỗi bước | p QP nhỏ (p = số cấp) | 1 QP | 0 (đóng, không cần bộ giải số) |
| Tính khả vi (differentiability) qua các cấp | Không khả vi tại biên chuyển cấp (theo nghiên cứu 2025) | Khả vi (một hàm mục tiêu trơn) | Khả vi |
| Phù hợp khi | Có ràng buộc ưu tiên cứng rõ ràng (cân bằng > tay) | Ít tác vụ, trọng số dễ tinh chỉnh | Tay máy đơn, ít ràng buộc bất đẳng thức |

**Khi nào dùng cái nào:** HQP là lựa chọn mặc định cho WBC humanoid hiện đại khi có một chuỗi ưu tiên rõ ràng cần đảm bảo tuyệt đối (an toàn > cân bằng > tác vụ chính > tác vụ phụ); QP một tầng đơn giản hơn, phù hợp hệ thống ít tác vụ và không đòi hỏi đảm bảo tuyệt đối.

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "HQP luôn tốt hơn QP một tầng trong mọi trường hợp."** Không hẳn — HQP tốn nhiều lần giải QP hơn (chi phí tính toán cao hơn), và nếu số cấp ưu tiên quá nhiều/quá mịn, hệ thống có thể trở nên "cứng nhắc" quá mức: một tác vụ ở cấp rất thấp gần như không bao giờ có không gian để hoạt động (null-space bị thu hẹp gần hết bởi các cấp trên), dẫn tới hành vi robot "phớt lờ" hoàn toàn các mục tiêu phụ.

2. **Hiểu nhầm: "Ràng buộc `không làm xấu nghiệm cấp trên` nghĩa là nghiệm cấp dưới phải giữ NGUYÊN GIÁ TRỊ cấp trên."** Không chính xác — ràng buộc là `‖Jⱼz−ẍⱼ*‖² ≤ wⱼ*+ε`, tức là **residual (sai số) không được XẤU HƠN**, chứ không bắt buộc z phải là đúng z_j* — có thể có nhiều z khác nhau cùng đạt residual tối ưu wⱼ*, và cấp dưới được tự do chọn TRONG SỐ các z đó (đây chính là không gian null-space thật sự).

3. **Hiểu nhầm: "HQP là một thuật toán RL hoặc học máy."** Sai — HQP là một phương pháp tối ưu số cổ điển thuần túy (không có thành phần học), thuộc nhóm WBC model-based. Nó thường bị nhầm lẫn vì các bài báo hiện đại đôi khi kết hợp HQP với các thành phần học (ví dụ dùng HQP làm "safety filter" phía sau một policy RL), nhưng bản thân thuật toán HQP không "học" gì cả.

## 🏗️ Ví dụ minh họa trong dự án này

Theo `NOI-DUNG-CHI-TIET.md` mục 1.3, HQP đại diện cho đỉnh cao của WBC model-based trong dự án này — nhưng dự án chọn hướng learning-based (SONIC, policy RL) thay vì triển khai HQP trực tiếp. Tuy vậy, tư duy "ưu tiên cứng" của HQP vẫn xuất hiện gián tiếp: khi thiết kế reward function cho policy RL (`04-imitation-learning-rl/`), người ta thường phải cân nhắc một cấu trúc reward tương tự tinh thần phân tầng (ví dụ phạt ngã nặng hơn rất nhiều so với phạt lệch quỹ đạo tay) để mô phỏng ý tưởng "an toàn/cân bằng phải thắng tuyệt đối" mà HQP đảm bảo bằng toán học tường minh, còn RL phải "học" ra hành vi tương tự thông qua thiết kế reward cẩn thận.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **Hạn chế về tính khả vi được chỉ ra rõ ràng (2025):** nghiên cứu "Dual Hierarchical Least-Squares Programming with Equality Constraints" (arXiv:2505.21071, 2025) chỉ ra rằng HQP tuần tự cổ điển **không khả vi tại ranh giới chuyển giữa các cấp ưu tiên**, giới hạn khả năng dùng trực tiếp trong các pipeline học gradient-based (ví dụ không thể lan truyền ngược gradient qua một bộ giải HQP tuần tự một cách trơn tru) — trừ khi dùng các biến thể dual/ADMM được thiết kế lại để khả vi. ([arXiv:2505.21071](https://arxiv.org/pdf/2505.21071))

2. **Kết hợp HQP với Control Barrier Functions cho tương tác người-robot an toàn:** "Control Barrier Functions Solved with Hierarchical Quadratic Programming for Safe Physical Human-Robot Interaction" (arXiv:2604.23039) đặt các ràng buộc an toàn dạng CBF vào đúng cấp ưu tiên cao nhất trong khung HQP, đảm bảo an toàn vật lý luôn thắng tuyệt đối các tác vụ khác — một ứng dụng trực tiếp của tinh thần "ưu tiên cứng" cho bài toán an toàn hiện đại. ([arXiv:2604.23039](https://arxiv.org/pdf/2604.23039))

3. **HQP làm tầng thực thi bên dưới policy học (2025):** hệ thống "HWC-Loco: A Hierarchical Whole-Body Control Approach to Robust Humanoid Locomotion" (arXiv:2503.00923) và một nghiên cứu khác về robot chi nặng ("Whole-Body Control Framework for Humanoid Robots with Heavy Limbs", arXiv:2506.14278) dùng HQP để giải quyết mâu thuẫn giữa nhiều mục tiêu theo dõi tham chiếu (từ một planner/policy cấp cao) và ràng buộc tuân thủ vật lý (compliance) — cho thấy HQP vẫn là công cụ được lựa chọn khi cần đảm bảo cứng, ngay cả trong các kiến trúc lai kết hợp học máy.

4. **Cải tiến hiệu năng tính toán bằng recursive projector:** ý tưởng "Recursive Hierarchical Projection" (RHP, gốc từ 2021, arXiv:2109.07236) — tính ma trận chiếu null-space một cách đệ quy để hỗ trợ chuyển đổi mượt giữa các mức ưu tiên mà không tăng chi phí tính toán — tiếp tục là nền tảng cho các cải tiến hiệu năng HQP những năm gần đây, đặc biệt khi số cấp ưu tiên lớn.

## ❓ Câu hỏi tự kiểm tra

1. Vì sao cấp 2 trong HQP không thể "vô tình" làm xấu nghiệm cấp 1, ngay cả khi tác vụ cấp 2 rất mạnh?
<details><summary>Gợi ý đáp án</summary>Vì ràng buộc `‖J₁z₂−ẍ₁*‖² ≤ w₁*+ε` được áp CỨNG vào bài toán QP cấp 2 — bất kể hàm mục tiêu cấp 2 "muốn" gì, nghiệm phải nằm trong tập khả thi đó, nên residual cấp 1 không thể vượt quá w₁*+ε.</details>

2. Trong ví dụ tính tay, nếu tác vụ cấp 1 có ràng buộc vật lý khiến w₁* > 0 (ví dụ ràng buộc buộc u≤1, trong khi tác vụ cấp 1 muốn u=2), điều gì xảy ra ở cấp 2?
<details><summary>Gợi ý đáp án</summary>Cấp 2 sẽ được ràng buộc bởi residual w₁* > 0 (không phải bằng 0 nữa) — nghĩa là cấp 2 có MỘT CHÚT không gian tự do quanh nghiệm cấp 1 (không bị khóa cứng tuyệt đối vào đúng 1 điểm), vì tập nghiệm "đạt residual ≤ w₁*+ε" khi w₁*>0 thường rộng hơn tập "đạt residual ≤ ε" khi w₁*=0.</details>

3. Vì sao HQP dùng generalized projector thay vì thêm ràng buộc bất đẳng thức tường minh "không làm xấu nghiệm" ở mỗi cấp?
<details><summary>Gợi ý đáp án</summary>Vì thêm ràng buộc bất đẳng thức dạng toàn phương (`‖Jz−ẍ*‖²≤w*+ε`) ở mỗi cấp làm bài toán trở thành QCQP (không còn thuần QP, tốn kém hơn); dùng phép chiếu null-space cho phép biến bài toán cấp dưới thành một QP thuần túy nhỏ hơn trong không gian con đã bị thu hẹp, tính toán hiệu quả hơn nhiều.</details>

4. Tại sao tính không khả vi của HQP tuần tự là một vấn đề với các pipeline học máy hiện đại?
<details><summary>Gợi ý đáp án</summary>Vì nhiều pipeline hiện đại muốn huấn luyện end-to-end bằng gradient descent qua cả bộ điều khiển (ví dụ học tham số của HQP hoặc dùng HQP như một lớp trong mạng nơ-ron khả vi) — nếu HQP không khả vi tại ranh giới chuyển cấp, gradient không lan truyền được mượt mà qua đó, cần các biến thể dual/ADMM được thiết kế lại để khắc phục.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể 3 cấp:** mở rộng ví dụ 1-biến ở trên thành 3 cấp ưu tiên: cấp 1 muốn u=1 (ràng buộc u≤3), cấp 2 muốn u=2, cấp 3 muốn u=5. Giải tuần tự từng cấp theo đúng thuật toán HQP, chỉ ra nghiệm cuối cùng và giải thích vì sao cấp 3 gần như "vô dụng" trong ví dụ này.
2. **Đọc tài liệu/code thật:** tìm một thư viện mã nguồn mở có cài đặt HQP cho robot (ví dụ `tsid` — Task Space Inverse Dynamics, hoặc `eiquadprog`/`HiQP`) — xác định đoạn code triển khai bước "tính ma trận chiếu null-space" (generalized projector), so sánh với sơ đồ 6 bước ở mục Cơ chế hoạt động.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

Hierarchical QP (HQP) giải quyết hạn chế lớn nhất của QP một tầng — việc phải tay chỉnh trọng số mà không đảm bảo ưu tiên tuyệt đối — bằng cách xếp các tác vụ theo thứ tự ưu tiên cứng và giải TUẦN TỰ từng cấp QP, mỗi cấp bị ràng buộc không được làm xấu residual tối ưu của mọi cấp cao hơn (thực hiện hiệu quả qua phép chiếu null-space đệ quy — generalized projector — thay vì thêm ràng buộc bất đẳng thức tường minh tốn kém). Kết quả là tác vụ ưu tiên cao (ví dụ thăng bằng) không bao giờ bị hy sinh cho tác vụ ưu tiên thấp (ví dụ tay với đồ vật), khác biệt căn bản so với cách tiếp cận trọng số. HQP vẫn là công cụ chuẩn cho WBC model-based hiện đại và các lớp "safety filter" phía sau policy học, dù có hạn chế đã biết về chi phí tính toán (nhiều QP nhỏ mỗi bước) và tính không khả vi tại ranh giới chuyển cấp — điều đang được các biến thể dual/ADMM gần đây tìm cách khắc phục.
