# Nội dung chi tiết — Whole-Body Control

File này là phần "giải thích đầy đủ" cho 4 khái niệm liệt kê ở mục A của `README.md`. README chỉ nêu tên khái niệm và trỏ tới paper gốc; file này giải thích **cơ chế hoạt động, công thức, và lý do tồn tại** của từng khái niệm, để đọc xong là hiểu được bản chất mà không bắt buộc phải đọc paper gốc trước. Các citation dùng lại đúng nguồn đã kiểm chứng ở mục B của README — không có citation mới nào được thêm ở đây.

---

## 1. Whole-Body Control cổ điển (model-based)

### 1.1. Task-space control / Operational-space control là gì

Điều khiển robot có hai cách chọn "không gian" để mô tả mục tiêu:

- **Không gian khớp (joint space):** mục tiêu là "khớp 1 quay tới góc θ₁, khớp 2 quay tới góc θ₂, …". Đây là cách cổ điển nhất, nhưng rất khó dùng khi mục tiêu thật sự của robot là ở đầu công tác (ví dụ: "bàn tay phải đến toạ độ (x, y, z) trong không gian") — ta phải tự tính ngược ra từng góc khớp (inverse kinematics) rồi mới điều khiển.
- **Không gian tác vụ (task space / operational space):** mục tiêu được mô tả trực tiếp bằng đại lượng ta thực sự quan tâm — vị trí/hướng của bàn tay, vị trí trọng tâm cơ thể, hướng nhìn của đầu, v.v. — bất kể robot có bao nhiêu khớp hay cấu hình ra sao.

**Operational-space control**, do Khatib đề xuất năm 1987 (Khatib 1987, mục B), là công thức toán học cho phép **điều khiển trực tiếp trong không gian tác vụ** thay vì phải đi vòng qua không gian khớp. Ý tưởng cốt lõi:

- Gọi **x** là toạ độ tác vụ (ví dụ vị trí đầu công tác), **q** là toạ độ khớp. Quan hệ vi phân giữa chúng là **ẋ = J·q̇**, trong đó **J** là **Jacobian** của tác vụ — ma trận biến đổi vận tốc khớp thành vận tốc tác vụ.
- Vì J biến đổi *vận tốc*, J^T (chuyển vị của J) biến đổi ngược lại *lực*: một lực mong muốn **F** tác dụng ở đầu công tác được tạo ra bằng mô-men khớp **τ = J^T·F**. Đây là quan hệ công-ảo (virtual work) — nền tảng để "nghĩ" bằng lực tác vụ rồi quy đổi sang mô-men khớp thực thi.
- Để tính được **F** sao cho động lực học không gian tác vụ được tuyến tính hoá đẹp (giống như điều khiển một khối lượng đơn ở không gian x), Khatib định nghĩa **ma trận quán tính không gian tác vụ Λ(x) = (J·M⁻¹·J^T)⁻¹**, với **M** là ma trận quán tính khớp của robot. Từ đó phương trình động lực học trong không gian tác vụ có dạng gọn: **Λ·ẍ + (các số hạng Coriolis/trọng lực quy đổi) = F**. Kiểm soát được **F** nghĩa là kiểm soát được gia tốc tác vụ **ẍ** một cách tuyến tính và tách bạch — đây là lý do công thức này "thống nhất" điều khiển vị trí và lực (unified approach) như tên paper gốc.
- Với robot dư bậc tự do (số khớp nhiều hơn số chiều tác vụ — humanoid luôn ở trường hợp này), mô-men khớp tổng quát là **τ = J^T·F + (I − J^T·J^#)·τ₀**, trong đó **J^#** là nghịch đảo suy rộng theo trọng số quán tính, còn **(I − J^T·J^#)** là **phép chiếu vào null-space** của tác vụ chính — cho phép thực hiện thêm một tác vụ phụ **τ₀** (ví dụ giữ tư thế thoải mái) mà **không ảnh hưởng đến tác vụ chính**. Đây chính là hạt giống ý tưởng "ưu tiên tác vụ" sẽ được mở rộng thành Hierarchical QP ở phần 1.3.
- Khatib, Sentis, Park (2004) mở rộng công thức này từ một tay máy đơn sang **toàn thân humanoid** — nhiều tác vụ cùng lúc (thăng bằng, tay trái, tay phải, đầu) đều được biểu diễn bằng Jacobian riêng và kết hợp qua cơ chế ưu tiên/null-space nói trên. Đây là gốc rễ trực tiếp của cụm từ "whole-body control".

### 1.2. Quadratic Programming (QP) trong WBC

Trong thực tế, WBC hiện đại hiếm khi dùng công thức null-space thuần tuý ở trên vì nó khó xử lý **ràng buộc bất đẳng thức** (giới hạn khớp, giới hạn lực tiếp xúc chân với sàn, ma sát Coulomb — friction cone, giới hạn mô-men động cơ). Thay vào đó, bài toán được viết lại thành một **Quadratic Program**:

- Biến quyết định: thường là gia tốc khớp, lực tiếp xúc, và mô-men khớp (đôi khi giải đồng thời cả ba).
- Hàm mục tiêu (quadratic — bậc hai): tổng bình phương sai số giữa gia tốc tác vụ mong muốn và gia tốc tác vụ đạt được, cho tất cả các tác vụ (thăng bằng, tay, thân...), có trọng số.
- Ràng buộc (constraints): phương trình động lực học robot (M·q̈ + h = τ + J_c^T·F_c), giới hạn khớp, ràng buộc chân không trượt/không nhấc khỏi sàn (friction cone, ZMP nằm trong đế chân).

QP có lợi thế lớn: **giải được nhiều tác vụ cùng lúc trong một bài toán tối ưu duy nhất**, mỗi bước điều khiển (thường 200Hz–1kHz với solver hiện đại), và bộ giải QP (như qpOASES, OSQP, HPIPM) đã được tối ưu tốc độ cao để chạy real-time trên robot thật.

### 1.3. Hierarchical QP (HQP) — giải nhiều tác vụ theo thứ tự ưu tiên

Vấn đề: nếu gộp tất cả tác vụ vào MỘT hàm mục tiêu có trọng số (weighted sum), ta phải tự tay chỉnh trọng số — dễ vỡ khi robot gặp tình huống mới (ví dụ tác vụ thăng bằng "quan trọng hơn tác vụ tay" trong MỌI tình huống, nhưng trọng số cố định không đảm bảo điều đó tuyệt đối).

**Hierarchical QP** (Escande, Mansard, Wieber 2014, mục B) giải quyết bằng cách xếp các tác vụ theo **thứ tự ưu tiên cứng (strict priority)**, rồi giải **tuần tự từng cấp**:

1. Giải QP cho tác vụ ưu tiên cao nhất (ví dụ: thăng bằng — ZMP không rời khỏi đế chân) — thu được nghiệm tối ưu và độ "dư thừa" (slack) còn lại.
2. Giải QP cho tác vụ ưu tiên tiếp theo (ví dụ: tay phải với đồ vật), **nhưng ràng buộc rằng nghiệm không được làm xấu đi nghiệm tối ưu vừa tìm ở bước 1** — nói cách khác, tác vụ cấp 2 chỉ được "vùng vẫy" trong **null-space** của tác vụ cấp 1.
3. Lặp lại xuống các cấp thấp hơn (tay trái, tư thế đầu, tư thế thoải mái...), mỗi cấp bị giới hạn trong null-space của TẤT CẢ các cấp cao hơn nó.

Kết quả: tác vụ ưu tiên cao **không bao giờ bị hy sinh** để phục vụ tác vụ ưu tiên thấp — khác biệt căn bản so với cách tiếp cận trọng số. Đây là "công cụ toán học đứng sau hầu hết WBC cổ điển hiện đại" như README đã ghi. Về mặt kỹ thuật, HQP dùng một "generalized projector" để chiếu từng tác vụ vào đúng phần null-space cần thiết một cách hiệu quả về mặt tính toán (không cần nhân ma trận null-space tường minh ở mỗi bước), cho phép chạy online trên robot thật.

### 1.4. ZMP (Zero Moment Point) và mô hình cart-table

**ZMP** là điểm trên mặt phẳng tiếp xúc (thường là sàn) tại đó **tổng mô-men của các lực quán tính và trọng lực tác dụng lên robot, chiếu lên mặt sàn, bằng không**. Nói dễ hiểu hơn: đó là điểm mà nếu ta đặt một "trụ đỡ ảo", robot sẽ không bị lật quanh trụ đó.

Điều kiện thăng bằng động cơ bản của robot hai chân: **ZMP phải luôn nằm bên trong đa giác đỡ (support polygon)** — vùng lồi bao quanh các điểm tiếp xúc của bàn chân với sàn. Nếu ZMP ra ngoài vùng này, robot chắc chắn sẽ lật (đây là điều kiện cần, khác với "capture point" là điều kiện cho robot có thể dừng lại an toàn — hai khái niệm liên quan nhưng không đồng nhất).

**Mô hình cart-table (Kajita et al. 2003):** để tính ZMP mà không cần mô hình động lực học đầy đủ (rất phức tạp với robot nhiều khớp), Kajita đơn giản hoá cơ thể robot thành **một khối lượng điểm di chuyển trên một "bàn" không khối lượng, không ma sát**, ở độ cao cố định (xấp xỉ độ cao trọng tâm). Với mô hình này, quan hệ giữa vị trí trọng tâm (x_com) và ZMP (p) là tuyến tính bậc hai đơn giản: **p = x_com − (z_c/g)·ẍ_com** (z_c là chiều cao bàn, g là gia tốc trọng trường). Nhờ đơn giản hoá này, bài toán "tìm quỹ đạo trọng tâm sao cho ZMP đi theo quỹ đạo mong muốn" trở thành một bài toán điều khiển tuyến tính giải được nhanh.

**ZMP preview control:** vấn đề của mô hình cart-table là nó chỉ xấp xỉ — sai số giữa mô hình đơn giản và động lực học thật (nhiều khớp) gây lệch ZMP. Kajita giải quyết bằng lý thuyết **preview control**: bộ điều khiển không chỉ nhìn trạng thái hiện tại, mà **"nhìn trước" (preview) một cửa sổ quỹ đạo ZMP tham chiếu trong tương lai gần** (ví dụ 1–2 giây tới, đã biết trước vì dáng đi được lên kế hoạch trước), rồi dùng thông tin tương lai đó để tính bù ngay từ bây giờ — giống người đi bộ nhìn trước vài bước để điều chỉnh dáng đi thay vì chỉ phản ứng tại chỗ. Bộ điều khiển kết hợp ba thành phần: phản hồi trạng thái (state feedback), phản hồi tích luỹ sai số theo dõi (integral tracking error), và số hạng "hành động dự đoán" (preview action) dựa trên quỹ đạo ZMP tham chiếu tương lai. Đây là paper "mọi tài liệu về dáng đi humanoid đều trích dẫn" theo đúng mô tả ở README.

### 1.5. MPC cho dáng đi (mức khái niệm)

**Model Predictive Control (MPC)** là bước phát triển tiếp theo của tư duy "nhìn trước" ở preview control, nhưng tổng quát hơn: tại mỗi bước thời gian, MPC **giải lại một bài toán tối ưu trên toàn bộ cửa sổ tương lai** (ví dụ tối ưu vị trí chân đặt xuống, lực tiếp xúc, quỹ đạo trọng tâm trong 1 giây tới), chỉ áp dụng bước điều khiển đầu tiên của lời giải, rồi **lặp lại toàn bộ quá trình tối ưu ở bước kế tiếp** với thông tin cập nhật mới nhất (vị trí thật, nhiễu vừa đo được). Khác biệt so với preview control cổ điển: MPC có thể trực tiếp đưa vào các ràng buộc phi tuyến/bất đẳng thức (giới hạn lực ma sát, giới hạn động cơ, vị trí đặt chân rời rạc) trong cùng một bài toán tối ưu, thay vì chỉ có một mô hình tuyến tính cố định như cart-table. Đây là hướng được dùng rộng rãi trong WBC cổ điển hiện đại (ví dụ MIT Cheetah, ANYmal) trước khi RL trở nên phổ biến.

---

## 2. WBC học sâu (learning-based)

### 2.1. Vì sao RL thay thế được model-based control

WBC cổ điển (mục 1) có điểm mạnh là **có đảm bảo toán học rõ ràng** (ổn định, ràng buộc được tôn trọng), nhưng có ba hạn chế lớn trong thực tế:

1. **Phải mô hình hoá tường minh** động lực học robot, tiếp xúc, ma sát — trong khi động lực học thật (đặc biệt khi tiếp xúc va chạm, địa hình phức tạp) rất khó mô hình chính xác.
2. **Reward/mục tiêu phải viết tay** (ZMP nằm trong đế chân, tay theo quỹ đạo X) — với các hành vi phức tạp, tự nhiên như người thật (nhảy, xoay người linh hoạt, phản ứng bất ngờ), viết tay công thức mục tiêu gần như bất khả thi.
3. **Khó tổng quát hoá**: bộ điều khiển thiết kế cho một tình huống cụ thể không tự động hoạt động tốt ở tình huống khác.

**WBC học sâu** thay thế cách tiếp cận "viết tay luật điều khiển" bằng cách **huấn luyện một policy** (thường là mạng nơ-ron, học bằng Reinforcement Learning) sao cho robot **tự học cách hành động** để đạt mục tiêu — mục tiêu phổ biến nhất hiện nay không phải là reward thủ công kiểu "giữ ZMP trong đế chân" mà là **reward = "giống với một chuyển động tham chiếu từ dữ liệu chuyển động người thật"** (motion tracking / motion imitation) — chi tiết thuật toán thuộc về `04-imitation-learning-rl/` (DeepMimic, AMP).

### 2.2. Huấn luyện song song quy mô lớn (Rudin et al. 2022)

Trước 2021, huấn luyện RL cho robot chân thường mất **hàng giờ đến hàng ngày** vì mô phỏng chạy tuần tự trên CPU, mỗi lần chỉ một (hoặc một vài) robot ảo.

**Rudin, Hoeller, Reist, Hutter (2022)** (mục B, arXiv:2109.11978) chứng minh có thể **mô phỏng hàng nghìn robot ảo song song trên MỘT GPU** (nhờ simulator GPU-native như Isaac Gym), rút ngắn thời gian huấn luyện từ hàng giờ xuống **vài phút** cho địa hình phẳng (dưới 4 phút) và khoảng 20 phút cho địa hình gồ ghề — nhanh hơn các phương pháp trước đó nhiều bậc độ lớn. Đóng góp không chỉ là "chạy song song nhiều hơn" mà còn phân tích **các thành phần thuật toán nào thực sự quan trọng** khi huấn luyện ở chế độ song song quy mô lớn, và đề xuất một **curriculum lấy cảm hứng từ game** (độ khó địa hình tăng dần khi robot "qua màn" tốt) để huấn luyện hiệu quả hơn.

Đây là "bài chuyển giao quan trọng" đúng như README mô tả: kỹ thuật huấn luyện song song quy mô lớn này là nền tảng trực tiếp mà **Isaac Gym/Isaac Lab** (mục `05-simulation-mujoco-isaaclab/`) và **SONIC** kế thừa để scale huấn luyện lên quy mô còn lớn hơn nữa (một trong 3 trục scale mà SONIC nhấn mạnh: model/data/compute).

### 2.3. Domain randomization (khái niệm)

Một robot ảo huấn luyện hoàn hảo trong mô phỏng vẫn có thể thất bại khi đưa ra robot thật, vì mô phỏng không bao giờ khớp 100% với vật lý thật (khối lượng, ma sát, độ trễ động cơ, nhiễu cảm biến đều có sai lệch) — đây gọi là **sim-to-real gap**.

**Domain randomization** là kỹ thuật giảm khoảng cách đó bằng cách **cố tình ngẫu nhiên hoá các tham số mô phỏng** trong quá trình huấn luyện — ví dụ: ngẫu nhiên hoá khối lượng từng khớp, hệ số ma sát sàn, độ trễ tín hiệu động cơ, nhiễu cảm biến, lực đẩy bất ngờ tác động lên robot — sao cho policy phải học cách **hoạt động tốt trên một PHÂN PHỐI rộng các điều kiện vật lý**, thay vì chỉ khớp với đúng một bộ tham số mô phỏng cố định. Khi đó, vật lý thật (dù không khớp chính xác với mô phỏng gốc) nhiều khả năng rơi vào "trong phạm vi" mà policy đã từng thấy khi huấn luyện, nên robot thật hoạt động ổn định hơn dù chưa từng được huấn luyện trực tiếp trên phần cứng thật. Huấn luyện song song quy mô lớn (mục 2.2) là điều kiện cần để domain randomization khả thi về mặt thời gian — cần rất nhiều lượt thử với tham số khác nhau để policy học được sự bền vững đó.

---

## 3. Kiến trúc "decoupled WBC" của SONIC

### 3.1. Ý tưởng cốt lõi: tách policy cấp thấp khỏi bộ ra quyết định cấp cao

SONIC (arXiv:2511.07820, mục B) tách kiến trúc điều khiển thành **hai lớp tách biệt**:

**Lớp thấp — Low-level motion-tracking policy:**
- **Nhận input:** (1) tín hiệu bản thể (proprioception) — vị trí khớp, vận tốc khớp, vận tốc góc gốc thân (root angular velocity), vector trọng lực trong hệ quy chiếu gốc thân, hành động ở bước trước; và (2) một **lệnh chuyển động** (motion command) — có thể là chuyển động robot tham chiếu, chuyển động người ở định dạng SMPL, hoặc dạng lai (hybrid: điểm mốc nửa thân trên + chuyển động robot cho nửa thân dưới).
- **Xuất output:** vị trí khớp mục tiêu (target joint positions), sau đó được các bộ điều khiển PD (proportional-derivative) ở từng khớp theo dõi để tạo mô-men thực thi.
- Nói cách khác: policy này chỉ có MỘT nhiệm vụ — **biết cách di chuyển cơ thể một cách tự nhiên, cân bằng, mượt mà để "đuổi theo" (track) bất kỳ chuyển động tham chiếu nào được đưa vào** — không quan tâm chuyển động đó đến từ đâu hay "vì mục đích gì".

**Lớp cao — Planner/policy cấp cao:**
- Quyết định **"làm gì"**: đi đâu, cầm vật gì, tương tác thế nào — tức là **sinh ra chuỗi lệnh chuyển động** để đưa xuống cho lớp thấp theo dõi.
- Trong SONIC, lớp cao có thể là:
  - **Con người** — qua giao diện VR teleoperation (điều khiển từ xa bằng tư thế người thật);
  - **Một kinematic planner sinh tự động** — cụ thể SONIC dùng một **"generative kinematic motion planner"** chạy ở 10Hz, hoạt động theo kiểu **autoregressive** (liên tục tái sinh các đoạn chuyển động tương lai dài 0.8–2.4 giây, dựa trên trạng thái robot hiện tại và lệnh người dùng mới nhất) để làm cầu nối giữa ý định người dùng cấp cao (ví dụ "đi tới điểm X") và quỹ đạo tham chiếu chi tiết mà lớp thấp cần;
  - **Một mô hình VLA** (GR00T N1.5/N1.x) — nhận input ảnh + ngôn ngữ, xuất ra lệnh chuyển động dạng hybrid (tư thế nửa thân trên + lệnh di chuyển) để đưa xuống lớp thấp.

### 3.2. Vì sao tách hai lớp này lại cho phép DÙNG CHUNG một policy cấp thấp

Nếu huấn luyện MỘT policy end-to-end riêng cho teleoperation và MỘT policy khác riêng cho VLA, ta phải **lặp lại toàn bộ công đoạn huấn luyện kỹ năng vận động cơ bản (đi, giữ thăng bằng, tránh ngã) hai lần**, và hai policy đó không chia sẻ được kinh nghiệm cho nhau.

Kiến trúc decoupled giải quyết việc này bằng cách để **kỹ năng vận động cơ bản** (khó huấn luyện, cần dữ liệu lớn, cần domain randomization kỹ) nằm **hoàn toàn ở lớp thấp — chỉ huấn luyện MỘT LẦN**, còn phần "quyết định làm gì" (khác nhau giữa teleop và VLA) chỉ cần tạo ra **một lệnh chuyển động đúng định dạng** mà lớp thấp đã hiểu sẵn. Vì cả tín hiệu từ VR teleop lẫn tín hiệu từ VLA cuối cùng đều được quy về **cùng một loại lệnh chuyển động** trước khi vào lớp thấp, một policy cấp thấp duy nhất phục vụ được cả hai use case mà không cần huấn luyện lại.

### 3.3. Sơ đồ luồng dữ liệu (ASCII)

```
 ┌──────────────────┐      ┌────────────────────────┐
 │  Người (VR teleop) │      │  VLA (GR00T N1.5/N1.x)  │
 │  → tư thế SMPL     │      │  → ảnh + ngôn ngữ       │
 └─────────┬─────────┘      └───────────┬─────────────┘
           │  human motion              │  hybrid command
           │  (SMPL pose)               │  (upper-body pose + nav)
           ▼                            ▼
   ┌───────────────┐            ┌────────────────────────┐
   │ Human motion   │            │ Generative kinematic     │
   │ encoder        │            │ motion planner (10Hz)    │
   └───────┬───────┘            └───────────┬─────────────┘
           │                                │  hybrid motion
           │                                ▼
           │                        ┌───────────────┐
           │                        │ Hybrid motion  │
           │                        │ encoder        │
           │                        └───────┬───────┘
           ▼                                ▼
        ┌────────────────────────────────────────┐
        │     Unified token space (FSQ quantizer)  │  ← mục 4
        └───────────────────┬──────────────────────┘
                             │  universal motion token
                             ▼
              ┌────────────────────────────────┐
              │  Low-level motion-tracking       │
              │  policy (50Hz)                   │
              │  input thêm: proprioception       │
              │  output: target joint positions   │
              └───────────────┬──────────────────┘
                               ▼
                     PD controllers từng khớp
                               ▼
                        Robot thật / mô phỏng
```

Đây là lý do README gọi đây là "cầu nối sang `06-vla-groot-sonic/`": phần "lớp cao = VLA" chính là chủ đề của thư mục đó, còn nội dung ở đây tập trung vào lớp thấp (WBC) và cơ chế cho phép hai lớp cao khác nhau (người/VLA) đều "cắm" được vào cùng một lớp thấp.

---

## 4. Token space thống nhất

### 4.1. Ý tưởng "token hoá" theo tinh thần Gato

**Reed et al. (DeepMind, 2022)** — Gato (mục B, arXiv:2205.06175) — chứng minh một ý tưởng có ảnh hưởng lớn: **bất kỳ modality nào** (ảnh, văn bản, hành động khớp robot, phần thưởng, quan sát cảm biến...) đều có thể được **rời rạc hoá thành một chuỗi token** theo một quy ước chung, để **MỘT kiến trúc transformer duy nhất** xử lý toàn bộ các modality đó như một bài toán "dự đoán token tiếp theo" (giống hệt cách GPT dự đoán từ tiếp theo trong văn bản). Ảnh được chia patch rồi mã hoá thành token, hành động liên tục được rời rạc hoá (discretize) thành token rời rạc, và tất cả được ghép vào chung MỘT chuỗi input cho transformer.

Lợi ích cốt lõi của cách làm này: một khi mọi thứ đã là "token", kiến trúc mô hình **không cần biết** token đó đến từ ảnh, văn bản, hay tín hiệu điều khiển — nó chỉ xử lý chuỗi token theo đúng một cơ chế attention duy nhất. Điều này cho phép **chia sẻ một backbone chung** giữa nhiều loại nhiệm vụ/nhiều loại input hoàn toàn khác nhau về bản chất vật lý.

### 4.2. Token space của SONIC hoạt động cụ thể ra sao

SONIC áp dụng đúng tinh thần đó nhưng chuyên biệt hoá cho bài toán motion: thay vì token hoá "mọi thứ trong đời sống" như Gato, SONIC chỉ cần token hoá **các loại lệnh chuyển động khác nhau** để chúng có chung một "ngôn ngữ" trước khi vào policy cấp thấp:

- Ba bộ **encoder chuyên biệt** (robot-motion encoder, human-motion/SMPL encoder, hybrid encoder) trước hết chuyển từng loại lệnh chuyển động (đến từ nguồn khác nhau — robot tham chiếu, người thật qua SMPL, hoặc dạng lai upper-body+navigation) vào một **không gian latent chung**.
- Không gian latent chung đó sau đó được **lượng tử hoá (quantize)** thành một **token rời rạc thống nhất (universal token)**, bằng một kỹ thuật gọi là **FSQ — Finite Scalar Quantization** (một dạng vector quantizer, họ hàng gần với VQ-VAE nhưng lượng tử hoá từng chiều vô hướng độc lập thay vì tra bảng codebook vector — *chi tiết số chiều/số mức lượng tử chính xác cần xác minh thêm từ bản đầy đủ của paper, phần phụ lục*).
- Universal token này sau đó được một **bộ giải mã (decoder)** chuyển thành lệnh điều khiển cụ thể mà policy cấp thấp (mục 3.1) hiểu và theo dõi được.
- Theo mô tả kiến trúc, hệ thống chạy policy cấp thấp ở **50Hz**, dùng token chuyển động có độ dài **64 chiều** ở latent — đây là các con số kiến trúc cụ thể được công bố trong tài liệu chính thức/trang dự án SONIC (nên đối chiếu thêm với bản PDF đầy đủ của paper khi cần trích dẫn chính xác tuyệt đối).

### 4.3. Vì sao điều này cho phép VR teleop và VLA "nói cùng ngôn ngữ" với policy WBC

Vì cả nhánh **người (VR teleop → SMPL → human-motion encoder)** lẫn nhánh **VLA (ảnh+ngôn ngữ → kinematic planner → hybrid encoder)** đều **hội tụ về cùng một loại token rời rạc** trước khi vào policy cấp thấp, bản thân policy cấp thấp **không cần biết và không cần phân biệt** token đó bắt nguồn từ đâu — nó chỉ học một kỹ năng duy nhất: "theo dõi tốt bất kỳ token chuyển động nào được đưa vào". Đây chính là cơ chế kỹ thuật cụ thể đứng sau phát biểu ở mục 3.2 ("một policy WBC duy nhất phục vụ được cả teleoperation lẫn VLA") — token space thống nhất là "giao diện chung" (common interface) giữa hai lớp cao khác nhau (người/VLA) với một lớp thấp duy nhất. Kết quả thực nghiệm được báo cáo trong tài liệu: biểu diễn hành động bằng **token rời rạc** cho kết quả tốt hơn hẳn so với dùng trực tiếp **tư thế người tường minh (explicit human poses)** làm không gian hành động, kể cả trên các tác vụ thao tác phức tạp (ví dụ cầm lon nước).

---

## Bảng thuật ngữ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| **Task space / Operational space** | Không gian mô tả mục tiêu điều khiển bằng đại lượng vật lý thực sự quan tâm (vị trí tay, trọng tâm...) thay vì góc khớp. |
| **Jacobian (J)** | Ma trận biến đổi vận tốc khớp thành vận tốc tác vụ (ẋ = J·q̇); chuyển vị J^T dùng để quy đổi lực tác vụ sang mô-men khớp. |
| **Null-space** | Không gian các chuyển động khớp không gây ảnh hưởng đến một tác vụ đã cho — nơi các tác vụ ưu tiên thấp hơn được thực hiện mà không phá tác vụ ưu tiên cao. |
| **Quadratic Programming (QP)** | Bài toán tối ưu hàm mục tiêu bậc hai với ràng buộc tuyến tính/bất đẳng thức; dùng để giải đồng thời nhiều tác vụ điều khiển kèm ràng buộc vật lý. |
| **Hierarchical QP (HQP)** | Giải nhiều QP theo thứ tự ưu tiên cứng — mỗi cấp thấp bị giới hạn trong null-space của mọi cấp cao hơn. |
| **ZMP (Zero Moment Point)** | Điểm trên mặt sàn nơi tổng mô-men của lực quán tính + trọng lực chiếu xuống bằng không; phải nằm trong đế chân để robot không lật. |
| **Cart-table model** | Mô hình đơn giản hoá cơ thể robot thành một khối lượng điểm trên bàn không khối lượng, dùng để tính ZMP nhanh (Kajita 2003). |
| **ZMP preview control** | Bộ điều khiển "nhìn trước" một cửa sổ quỹ đạo ZMP tương lai đã biết trước, để tính bù ngay từ hiện tại. |
| **MPC (Model Predictive Control)** | Tại mỗi bước, giải lại bài toán tối ưu trên cửa sổ thời gian tương lai, chỉ áp dụng bước đầu tiên, rồi lặp lại. |
| **Domain randomization** | Ngẫu nhiên hoá tham số mô phỏng (khối lượng, ma sát, nhiễu...) khi huấn luyện để policy bền vững hơn khi chuyển sang robot thật (giảm sim-to-real gap). |
| **Motion tracking / motion imitation** | Huấn luyện policy để "bắt chước" theo một chuyển động tham chiếu (thường từ dữ liệu người) thay vì dùng reward thủ công. |
| **Decoupled WBC** | Kiến trúc tách policy vận động cấp thấp (motion-tracking) khỏi bộ ra quyết định cấp cao (planner/VLA/người). |
| **Kinematic motion planner** | Bộ sinh quỹ đạo chuyển động tham chiếu (không tính lực/động lực học) để làm cầu nối giữa ý định cấp cao và policy cấp thấp. |
| **Token space / tokenization** | Rời rạc hoá dữ liệu (ảnh, hành động, chuyển động...) thành chuỗi token, để một kiến trúc (thường transformer) xử lý thống nhất nhiều modality. |
| **FSQ (Finite Scalar Quantization)** | Kỹ thuật lượng tử hoá vector latent thành token rời rạc, họ hàng với VQ-VAE, dùng trong token space của SONIC. |
