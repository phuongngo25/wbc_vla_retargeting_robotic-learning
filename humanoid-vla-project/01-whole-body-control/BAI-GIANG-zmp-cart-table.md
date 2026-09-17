# Bài giảng: ZMP (Zero Moment Point) và mô hình cart-table

*(Thuộc mảng: Whole-Body Control (WBC))*

## 🎯 Mục tiêu bài học

- Định nghĩa chính xác được ZMP bằng ngôn ngữ mô-men lực, không chỉ bằng lời mô tả trực giác.
- Giải thích được vì sao "ZMP nằm trong đế chân" là điều kiện cần (không phải điều kiện đủ) cho thăng bằng động.
- Phân biệt được ZMP với capture point — hai khái niệm liên quan nhưng không đồng nhất.
- Suy ra được công thức cart-table `p = x_com − (z_c/g)·ẍ_com` từ mô hình vật lý đơn giản hóa.
- Tính tay được ZMP cho một quỹ đạo trọng tâm cụ thể, dùng công thức cart-table.
- Nêu được ít nhất 2 hạn chế đã biết của mô hình cart-table và một hướng khắc phục/mở rộng hiện đại.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Sau hai bài giảng về QP/HQP (giải quyết bài toán "làm sao thực thi nhiều tác vụ đồng thời"), bài này chuyển sang một câu hỏi khác, đặc thù cho robot hai chân: **làm sao biết robot có sắp ngã hay không, và tính rẻ đến mức chạy được real-time?** ZMP là câu trả lời kinh điển cho câu hỏi đó — một tiêu chí thăng bằng động dùng được trực tiếp làm RÀNG BUỘC trong QP-WBC (ví dụ "ZMP phải nằm trong đế chân" chính là một trong các ràng buộc bất đẳng thức đã nhắc ở bài giảng QP). Mô hình cart-table (Kajita 2003) là bước đơn giản hóa giúp TÍNH được ZMP mà không cần mô hình động lực học đầy đủ của một robot nhiều khớp — nền tảng trực tiếp cho ZMP preview control và MPC dáng đi (hai bài giảng tiếp theo).

## 🧠 Trực giác

### Góc nhìn 1: Cây gậy giữ thăng bằng trên đầu ngón tay

Hãy tưởng tượng bạn giữ thăng bằng một cây gậy dài trên đầu ngón tay. Nếu điểm tiếp xúc (ngón tay) luôn nằm ĐÚNG dưới "điểm cân bằng ảo" của cây gậy (điểm mà nếu đặt một trụ đỡ tưởng tượng ở đó, mọi mô-men lật sẽ triệt tiêu), gậy không đổ. ZMP chính là "điểm cân bằng ảo" đó cho toàn bộ cơ thể robot — chừng nào nó còn nằm trong vùng ngón tay có thể với tới (đế chân, support polygon), gậy (robot) còn đứng được.

**Giới hạn của loại suy này:** cây gậy là một vật cứng đơn giản (1 khối lượng phân bố dọc một trục); cơ thể robot có nhiều khớp, khối lượng phân tán phức tạp — "điểm cân bằng ảo" của robot phải tính từ TỔNG mô-men của MỌI khối lượng thành phần (thân, tay, chân), không đơn giản như một cây gậy đồng nhất.

### Góc nhìn 2: Bập bênh (see-saw) và điểm tựa "không xoay"

Ở góc độ mô-men lực: hãy nghĩ đến một cái bập bênh với điểm tựa có thể di chuyển. ZMP là vị trí đặt điểm tựa đó sao cho bập bênh KHÔNG XOAY quanh nó (tổng mô-men = 0) tại đúng thời điểm đang xét. Nếu bạn phải đặt điểm tựa ra NGOÀI PHẠM VI của tấm ván (đế chân) để bập bênh cân bằng, điều đó có nghĩa là không có cách nào giữ bập bênh khỏi xoay — nó SẼ lật.

**Giới hạn của loại suy này:** bập bênh là hệ 2D tĩnh; ZMP của robot là một điểm 2D (trên mặt sàn) nhưng phải tính từ động lực học 3D ĐANG CHUYỂN ĐỘNG (gia tốc trọng tâm, mô-men quán tính quay) — "không xoay" ở đây là điều kiện tức thời tại mỗi thời điểm, phải tính lại liên tục theo thời gian, không phải một cấu hình tĩnh một lần.

## 📐 Định nghĩa chính xác

**ZMP** là điểm trên mặt phẳng tiếp xúc (giả sử mặt sàn phẳng, nằm ngang) tại đó **thành phần nằm ngang (song song mặt sàn) của tổng mô-men** do lực quán tính và trọng lực tác dụng lên robot, tính quanh điểm đó, **bằng không**:

```
Στ_x = 0,   Στ_y = 0     (tại điểm ZMP, quanh 2 trục nằm ngang)
```

**Điều kiện thăng bằng động (điều kiện CẦN, không phải điều kiện ĐỦ):**

```
ZMP ∈ support polygon (đa giác lồi bao quanh các điểm tiếp xúc bàn chân với sàn)
```

Nếu ZMP ra ngoài đa giác này, tồn tại một mô-men lật không triệt tiêu được → robot chắc chắn lật. Ngược lại, ZMP nằm trong đa giác KHÔNG đảm bảo robot sẽ mãi mãi ổn định (ví dụ robot có thể đang di chuyển với vận tốc lớn khiến nó sẽ rời khỏi support polygon trong tương lai gần dù ZMP hiện tại vẫn hợp lệ) — đây là lý do "capture point" (điểm mà robot cần đặt bước chân tiếp theo vào đó để dừng lại an toàn) là một khái niệm BỔ SUNG, không thay thế ZMP.

**Mô hình cart-table (Kajita et al., 2003):** đơn giản hóa toàn bộ cơ thể robot thành **một khối lượng điểm** di chuyển trên một "bàn" (table) không khối lượng, không ma sát, ở độ cao cố định `z_c` (xấp xỉ độ cao trọng tâm trung bình). Với giả định này, phương trình mô-men quanh điểm ZMP `p` cho ra quan hệ tuyến tính:

```
p = x_com − (z_c/g)·ẍ_com
```

trong đó `x_com` là vị trí ngang của trọng tâm, `ẍ_com` là gia tốc ngang của trọng tâm, `g` là gia tốc trọng trường (≈9.81 m/s²). Công thức này áp dụng độc lập cho từng trục ngang (x và y), nên mô hình 3D cart-table thực chất tách thành **hai mô hình 2D độc lập** (một cho trục x, một cho trục y).

## ⚙️ Cơ chế hoạt động — từng bước

```
┌──────────────────────────────────────────────────────────────┐
│ 1. Đơn giản hóa: coi toàn bộ khối lượng robot là 1 điểm ở       │
│    độ cao z_c (không đổi), đặt trên "bàn" không khối lượng      │
└────────────────────────────┬───────────────────────────────────┘
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ 2. Viết phương trình mô-men quanh điểm tiếp xúc bàn-sàn (ZMP)   │
│    → cân bằng mô-men trọng lực (m·g·x_com) và mô-men quán       │
│      tính (m·ẍ_com·z_c) quanh điểm đó                            │
└────────────────────────────┬───────────────────────────────────┘
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ 3. Giải ra quan hệ tuyến tính: p = x_com − (z_c/g)·ẍ_com          │
└────────────────────────────┬───────────────────────────────────┘
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ 4. Dùng công thức này THEO CHIỀU NGƯỢC: cho trước quỹ đạo p     │
│    (ZMP tham chiếu mong muốn), giải phương trình vi phân        │
│    tuyến tính để tìm quỹ đạo x_com cần thiết                     │
│    (đây chính là bài toán ZMP preview control — bài giảng riêng)│
└──────────────────────────────────────────────────────────────┘
```

Điểm mấu chốt về mặt sử dụng: mô hình cart-table không dùng để "đo" ZMP của một chuyển động đã cho theo chiều thuận (forward) là chính — nó thường được dùng theo chiều NGƯỢC (inverse): ta MUỐN ZMP đi theo một quỹ đạo tham chiếu an toàn (ví dụ luôn ở giữa đế chân đang chống), và cần giải ngược ra quỹ đạo trọng tâm `x_com(t)` tạo ra đúng quỹ đạo ZMP đó.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh họa, số tự chọn để dễ hình dung.)*

Robot có chiều cao trọng tâm cố định `z_c = 0.8 m`, `g = 9.8 m/s²`. Tại một thời điểm, trọng tâm đang ở vị trí `x_com = 0.05 m` (lệch nhẹ về phía trước so với gốc tọa độ đặt tại tâm bàn chân), với gia tốc `ẍ_com = -0.4 m/s²` (đang giảm tốc/hãm lại).

**Bước 1 — áp dụng công thức trực tiếp:**

```
p = x_com − (z_c/g)·ẍ_com
  = 0.05 − (0.8/9.8)·(−0.4)
  = 0.05 − (0.0816)·(−0.4)
  = 0.05 + 0.0327
  = 0.0827 m
```

**Diễn giải:** dù trọng tâm chỉ lệch 0.05m so với gốc, ZMP thực tế lại ở 0.0827m — xa hơn một chút, vì gia tốc âm (đang hãm) tạo thêm một "đóng góp quán tính" dương vào vị trí ZMP. Nếu đế chân (support polygon) trải dài từ -0.06m đến +0.10m (kích thước bàn chân điển hình), ZMP = 0.0827m vẫn nằm TRONG vùng an toàn — robot không lật.

**Bước 2 — thử trường hợp gia tốc lớn hơn để thấy giới hạn:** nếu robot giảm tốc gấp hơn, `ẍ_com = -2.0 m/s²`:

```
p = 0.05 − (0.0816)·(−2.0) = 0.05 + 0.163 = 0.213 m
```

ZMP = 0.213m ĐÃ VƯỢT RA NGOÀI đế chân (giới hạn +0.10m) → theo tiêu chí ZMP, đây là dấu hiệu robot sẽ lật (ngã về phía trước) nếu tiếp tục gia tốc/giảm tốc ở mức này mà không điều chỉnh (ví dụ bước chân bù, hoặc giảm gia tốc). Đây chính xác là loại tính toán mà một bộ điều khiển ZMP-based (preview control, MPC) chạy liên tục ở mỗi bước thời gian để đảm bảo an toàn.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | ZMP (cart-table) | Capture Point | Zero Moment Line (multi-contact, mở rộng hiện đại) |
|---|---|---|---|
| Loại điều kiện | Cần (necessary) cho thăng bằng tại 1 thời điểm | Đủ để BIẾT nơi cần đặt bước chân để dừng an toàn | Cần, mở rộng cho tiếp xúc đa điểm 3D (không chỉ 2 chân trên sàn phẳng) |
| Giả định địa hình | Sàn phẳng, một mặt tiếp xúc | Thường cũng giả định sàn phẳng | Không giả định sàn phẳng — dùng cho tiếp xúc bất kỳ (tay, chân, nghiêng) |
| Độ phức tạp mô hình | Rất đơn giản (1 khối lượng điểm) | Trung bình (thêm động lực học con lắc ngược) | Cao hơn — cần biểu diễn ràng buộc 3D tổng quát |
| Dùng để làm gì | Kiểm tra/ràng buộc thăng bằng tức thời | Quyết định VỊ TRÍ bước chân tiếp theo | Kiểm tra thăng bằng khi robot chống nhiều điểm không đồng phẳng |
| Năm/nguồn | Vukobratović 1972 (khái niệm gốc), Kajita 2003 (cart-table) | Pratt et al. 2006 | PMC — "Zero Moment Line" (mở rộng 3D hiện đại) |

**Khi nào dùng cái nào:** ZMP/cart-table phù hợp cho địa hình phẳng, robot đi hai chân chuẩn; capture point bổ sung cho bài toán "phản ứng khi bị đẩy — bước chân ở đâu để không ngã"; Zero Moment Line/các mở rộng đa tiếp xúc cần thiết khi robot leo trèo, chống tay, hoặc đứng trên bề mặt nghiêng/không đồng phẳng — tình huống mà ZMP cổ điển (giả định 1 mặt sàn phẳng) không còn áp dụng trực tiếp được.

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "ZMP nằm trong đế chân nghĩa là robot chắc chắn không bao giờ ngã."** Sai — đây chỉ là điều kiện CẦN tại một thời điểm cụ thể, không phải điều kiện ĐỦ cho toàn bộ tương lai. Một robot có thể có ZMP hợp lệ ngay bây giờ nhưng đang di chuyển với động lượng lớn tới mức sẽ không kịp đặt bước chân tiếp theo đúng chỗ — đây chính xác là khoảng trống mà khái niệm capture point lấp đầy.

2. **Hiểu nhầm: "Mô hình cart-table áp dụng đúng cho MỌI chuyển động của robot, kể cả nhảy hay chạy."** Sai — cart-table giả định độ cao trọng tâm `z_c` CỐ ĐỊNH. Khi robot nhảy, chạy, hoặc gập gối sâu (độ cao trọng tâm thay đổi đáng kể), giả định này bị vi phạm và công thức `p = x_com − (z_c/g)·ẍ_com` không còn chính xác — cần các mô hình phức tạp hơn (ví dụ mô hình con lắc ngược có độ cao thay đổi, hoặc động lực học trọng tâm đầy đủ — centroidal dynamics).

3. **Hiểu nhầm: "ZMP là một đại lượng đo được trực tiếp bằng cảm biến, giống như đo vị trí GPS."** Không chính xác hoàn toàn — trên robot thật, ZMP thường được TÍNH GIÁN TIẾP từ lực/mô-men đo được ở cảm biến lực 6 trục (F/T sensor) gắn ở bàn chân, kết hợp với mô hình động lực học, KHÔNG phải một đại lượng đo trực tiếp một-bước như vị trí GPS — sai số cảm biến lực và độ trễ tính toán đều ảnh hưởng tới độ chính xác ước lượng ZMP thực tế.

## 🏗️ Ví dụ minh họa trong dự án này

Theo `NOI-DUNG-CHI-TIET.md` mục 1.4-1.5, ZMP/cart-table là nền tảng của WBC model-based cổ điển — nhóm kỹ thuật mà dự án này (thiên về learning-based, SONIC) không triển khai trực tiếp. Tuy nhiên, hiểu ZMP rất hữu ích khi đọc reward function của các policy RL hiện đại cho locomotion (`04-imitation-learning-rl/`): nhiều công trình (ví dụ nghiên cứu "Humanoid Whole-Body Locomotion on Narrow Terrain" nhắc ở mục Cập nhật hiện đại dưới đây) vẫn dùng ZMP hoặc biến thể của nó làm một THÀNH PHẦN REWARD (ZMP-driven reward) để "hướng dẫn" policy học cách giữ thăng bằng, dù bản thân policy không giải trực tiếp phương trình cart-table — ZMP trở thành một tín hiệu huấn luyện (training signal) thay vì một ràng buộc điều khiển tường minh.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **ZMP làm reward cho RL trên địa hình khó (2025):** "Humanoid Whole-Body Locomotion on Narrow Terrain via Dynamic Balance and Reinforcement Learning" (arXiv:2502.17219, 2025) dùng "extended ZMP-driven rewards" trong một khung actor-critic toàn thân để huấn luyện policy đi trên địa hình hẹp/khó — cho thấy khái niệm ZMP cổ điển vẫn được TÁI SỬ DỤNG trực tiếp như một tín hiệu thiết kế reward trong các hệ thống learning-based hiện đại, thay vì bị thay thế hoàn toàn. ([arXiv:2502.17219](https://arxiv.org/pdf/2502.17219))

2. **Mở rộng ZMP cho tiếp xúc đa điểm 3D — Zero Moment Line:** nghiên cứu công bố trên PMC "Zero Moment Line—Universal Stability Parameter for Multi-Contact Systems in Three Dimensions" mở rộng khái niệm ZMP (vốn giả định một mặt sàn phẳng, tiếp xúc 2 chân) thành một tham số ổn định tổng quát hơn cho các hệ tiếp xúc đa điểm không đồng phẳng (ví dụ robot leo trèo, chống tay) — giải quyết đúng hạn chế "ZMP không dùng được khi robot chống ở độ cao khác nhau hoặc bị đẩy/kéo bằng tay" đã biết từ lâu. ([PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC9371053/))

3. **Hạn chế lý thuyết đã biết — hiệu ứng "waterbed" và pha không cực tiểu:** các phân tích hệ thống điều khiển hiện đại chỉ ra mô hình con lắc ngược một khối lượng (nền tảng của cart-table) tạo ra một hệ **non-minimum phase** (pha không cực tiểu) khi dùng cho thiết kế bộ điều khiển — có hiện tượng "waterbed effect" trong miền tần số và "undershoot" không tránh khỏi trong miền thời gian, nghĩa là có giới hạn cơ bản (không chỉ do thiết kế kém) về việc bám chính xác một quỹ đạo ZMP tham chiếu bất kỳ — đây là động lực trực tiếp cho preview control (bài giảng tiếp theo) và các phương pháp MPC hiện đại hơn.

4. **MPC pha-phân đoạn thay thế preview control cổ điển (2025):** "Phase-based Nonlinear Model Predictive Control for Humanoid Walking Stabilization with Single and Double Support Time Adjustments" (arXiv:2506.03856, 2025) và "Flexible Model Predictive Control for Bounded Gait Generation in Humanoid Robots" (PMC, 2025) tiếp tục phát triển theo hướng thay preview control tuyến tính cổ điển bằng MPC phi tuyến linh hoạt hơn, có thể điều chỉnh cả thời gian pha chống một chân/hai chân — vượt ra ngoài giả định "quỹ đạo ZMP tham chiếu cố định trước" của preview control gốc.

## ❓ Câu hỏi tự kiểm tra

1. Vì sao "ZMP nằm trong đế chân" chỉ là điều kiện CẦN chứ không phải điều kiện ĐỦ cho thăng bằng?
<details><summary>Gợi ý đáp án</summary>Vì nó chỉ đảm bảo không có mô-men lật tại thời điểm hiện tại — không nói gì về động lượng/vận tốc hiện tại của robot có thể khiến nó rời khỏi vùng ổn định trong tương lai gần; cần thêm khái niệm như capture point để đánh giá điều đó.</details>

2. Trong công thức `p = x_com − (z_c/g)·ẍ_com`, nếu `ẍ_com = 0` (trọng tâm chuyển động đều, không gia tốc), ZMP bằng gì?
<details><summary>Gợi ý đáp án</summary>ZMP = x_com — khi không có gia tốc, ZMP trùng với hình chiếu của trọng tâm xuống mặt sàn, đúng như trực giác tĩnh học cơ bản.</details>

3. Tại sao mô hình cart-table không áp dụng được khi robot đang nhảy (độ cao trọng tâm thay đổi mạnh)?
<details><summary>Gợi ý đáp án</summary>Vì công thức cart-table giả định z_c (độ cao trọng tâm) không đổi trong suốt quá trình dẫn xuất phương trình mô-men; khi z_c thay đổi mạnh (nhảy, gập gối sâu), giả định này bị vi phạm và quan hệ tuyến tính đơn giản không còn đúng.</details>

4. Trong ví dụ tính tay, nếu đế chân trải dài từ -0.06m đến +0.10m và ZMP tính được là 0.213m, cụ thể điều gì có khả năng xảy ra với robot nếu không có hành động điều chỉnh?
<details><summary>Gợi ý đáp án</summary>Robot có nguy cơ lật/ngã về phía trước (theo hướng ZMP vượt ra ngoài, tức hướng dương) — vì tồn tại mô-men lật không thể triệt tiêu bởi bất kỳ phân bố lực tiếp xúc nào trong phạm vi đế chân hiện có.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể:** với `z_c = 0.9m` (robot cao hơn), `g=9.8`, `x_com=0.02m`, hãy tính ZMP cho hai trường hợp gia tốc: `ẍ_com = 1.0 m/s²` và `ẍ_com = -1.0 m/s²`. So sánh kết quả và giải thích trực giác vì sao dấu của gia tốc lại đẩy ZMP về hai hướng ngược nhau.
2. **Đọc/tra cứu thêm:** đọc phần "Ch. 5 - Highly-articulated Legged Robots" trong Underactuated Robotics (Tedrake, đã có trong `../../resources/03-robotics.md`) — tìm đoạn giải thích quan hệ giữa ZMP và support polygon bằng hình vẽ, đối chiếu với ví dụ tính tay ở trên để kiểm tra hiểu đúng.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

ZMP là điểm trên mặt sàn nơi tổng mô-men ngang của lực quán tính và trọng lực triệt tiêu — nó phải nằm trong đế chân (support polygon) như một điều kiện CẦN (không phải đủ) để robot hai chân không bị lật tại một thời điểm cho trước. Mô hình cart-table (Kajita 2003) đơn giản hóa cơ thể robot thành một khối lượng điểm ở độ cao cố định trên một "bàn" không khối lượng, cho ra quan hệ tuyến tính gọn `p = x_com − (z_c/g)·ẍ_com` giữa ZMP và động lực học trọng tâm — nền tảng tính toán rẻ giúp các bộ điều khiển dáng đi cổ điển (preview control, MPC) chạy được real-time. Mô hình này có giới hạn rõ (giả định độ cao trọng tâm cố định, chỉ áp dụng khi tiếp xúc trên một mặt sàn phẳng), và các hướng mở rộng hiện đại (Zero Moment Line cho đa tiếp xúc 3D, MPC phi tuyến pha-phân đoạn, hoặc dùng ZMP làm tín hiệu reward cho RL) tiếp tục phát triển từ chính nền tảng khái niệm này.
