# Bài giảng: Kiến trúc dual-system của GR00T N1 (System 2 + System 1)

*(Thuộc mảng: Vision-Language-Action: GR00T N1 → N1.7 & SONIC)*

## 🎯 Mục tiêu bài học

- Giải thích được **vì sao** System 2 chạy ở 10Hz còn System 1 chạy ở 120Hz — không phải là một con số tuỳ chọn, mà là hệ quả trực tiếp của việc hai module giải hai bài toán khác bản chất (suy luận ngữ nghĩa vs. điều khiển motor thời gian thực).
- Tính được số tham số của System 1 từ hai con số đã công bố: tổng 2.2B và System 2 (VLM) 1.34B.
- Vẽ và đọc được sơ đồ luồng dữ liệu đầy đủ: camera + ngôn ngữ → Eagle-2 (System 2) → embedding $\phi_t$ → cross-attention → DiT flow-matching (System 1) → action chunk $A_t$ → robot.
- Tính được: (a) tỉ lệ tần số 120Hz/10Hz, (b) thời lượng thực tế mà một action chunk với $H=16$ bao phủ, (c) trong 1 giây, System 1 chạy bao nhiêu bước so với System 2.
- Phân biệt được "hai model độc lập ghép lại" (pipeline rời rạc, không đúng với GR00T N1) và "hai module tightly-coupled, jointly trained end-to-end" (đúng với GR00T N1).
- Giải thích được ẩn dụ "System 1 / System 2" mượn từ Kahneman **chỉ là tên gọi**, không hàm ý cùng cơ chế tính toán với não người.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Đây là khái niệm **trung tâm nhất của toàn bộ mảng 06**, và có thể nói là trung tâm của cả dự án học tập này, vì nó là nơi hai nửa bài toán humanoid robot gặp nhau: "hiểu phải làm gì" (nhận thức, ngôn ngữ, lập kế hoạch — thuộc lớp nhận thức cao) và "làm như thế nào cho mượt và ổn định" (điều khiển khớp/motor thời gian thực — thuộc lớp điều khiển thấp, chính là chủ đề của `01-whole-body-control/`).

Mọi bản GR00T sau này — N1.5, N1.6, N1.7 — đều **giữ nguyên khung dual-system này** và chỉ thay đổi các thành phần bên trong (backbone System 2 khác, dữ liệu huấn luyện khác, action head cải tiến). Nếu hiểu sai khung sườn ở bài này, mọi bài giảng tiếp theo trong mảng 06 (N1.5→N1.6, N1.7/EgoScale, flow matching, tích hợp SONIC) sẽ bị hiểu sai theo. Ngược lại, nắm chắc bài này thì các bài sau chỉ còn là "thay một mảnh ghép, giữ nguyên khung".

Bài này **chỉ** nói về kiến trúc dual-system (System 2 + System 1 tổ chức ra sao, tần số nào, tham số bao nhiêu, nối nhau bằng cơ chế gì). Các khái niệm liên quan sau đây có bài giảng riêng, không lặp lại chi tiết ở đây:
- Flow matching (cơ chế toán học cụ thể của System 1) → xem bài giảng riêng.
- N1.7 / EgoScale / scaling law cho dexterity → xem bài giảng riêng.
- SONIC / unified token space (cách System 1 xuất token cho lớp điều khiển whole-body) → xem bài giảng riêng.

## 🧠 Trực giác

### Góc nhìn 1: Não bộ và tuỷ sống (nhưng chỉ là ẩn dụ đặt tên, không phải cơ chế thật)

Khi bạn bắt một quả bóng đang bay, bạn **không** ngồi suy nghĩ "góc ném là bao nhiêu độ, vận tốc bao nhiêu, tay tôi cần di chuyển theo quỹ đạo nào". Có một tầng "hiểu tình huống" (đây là quả bóng, tôi cần bắt nó) chạy chậm và có ý thức, và một tầng phản xạ vận động chạy rất nhanh, gần như tự động, điều chỉnh liên tục vị trí bàn tay theo phản hồi thị giác/cảm nhận cơ thể.

Cái tên "System 1 / System 2" trong GR00T N1 **mượn trực tiếp** từ cách đặt tên này — cụ thể là từ cuốn sách kinh điển của nhà tâm lý học **Daniel Kahneman, "Thinking, Fast and Slow" (2011)**: System 1 là tư duy nhanh, trực giác, tự động, tốn ít công sức có ý thức; System 2 là tư duy chậm, phân tích, có ý thức, tốn nhiều công sức.

**Điều quan trọng cần nói rõ ngay:** đây **chỉ là một phép ẩn dụ đặt tên** cho hai module trong một mạng neural network nhân tạo — không phải tuyên bố khoa học rằng GR00T N1 tái tạo cơ chế tính toán của não bộ/tuỷ sống người. System 2 trong GR00T N1 là một Vision-Language Model (VLM) huấn luyện bằng gradient descent trên dữ liệu internet-scale; System 1 là một Diffusion Transformer huấn luyện bằng flow-matching loss. Cả hai đều là các hàm số vi phân được (differentiable functions) tối ưu bằng backpropagation — không có neuron sinh học, không có tuỷ sống, không có "phản xạ" theo nghĩa sinh lý học nào ở đây.

**Đúng ở đâu:** phép loại suy này giúp ghi nhớ *vai trò chức năng* — một tầng suy luận ngữ nghĩa chậm, một tầng điều khiển vận động nhanh — và giúp hiểu trực giác *tại sao* tách hai tần số lại hợp lý (mục dưới sẽ giải thích kỹ hơn).

**Giới hạn:** đừng suy diễn thêm rằng System 1 của GR00T "vô thức" theo nghĩa nào đó, hay System 2 "có ý thức" theo nghĩa nào đó. Cả hai là mạng neural nhân tạo hoàn toàn, không có gì tương đương với trải nghiệm chủ quan. Ẩn dụ này cũng không giải thích được *cơ chế cross-attention cụ thể* nối hai hệ thống — cơ chế đó là kỹ thuật thuần tuý (mục Cơ chế hoạt động bên dưới), không có tương đương sinh học rõ ràng.

### Góc nhìn 2: Kiến trúc planner–controller trong robotics cổ điển (nhưng "tightly coupled", không phải hai module tách rời)

Trong robotics cổ điển (trước học sâu), một hệ điều khiển robot thường được chia làm hai lớp rất quen thuộc:
- **Planner** (lớp lập kế hoạch): chạy chậm, ra quyết định cấp cao — ví dụ tính toán quỹ đạo tay/chân mong muốn, chọn hành vi tiếp theo.
- **Controller** (lớp điều khiển thấp): chạy rất nhanh (thường vài trăm Hz đến kHz), nhận setpoint từ planner và tính lệnh momen/vị trí khớp mỗi chu kỳ điều khiển, đảm bảo bám theo setpoint và giữ ổn định.

Đây gần giống với ý tưởng "tách hai tần số" của GR00T N1: System 2 giống planner (ra "ý định" cấp cao ở tần số thấp), System 1 giống controller (chạy vòng lặp nhanh sinh lệnh motor).

**Đúng ở đâu:** phép loại suy này nắm đúng lý do kỹ thuật — tách tần số theo *bản chất của bài toán con*, một mẫu thiết kế rất phổ biến và đã được kiểm chứng trong robotics cổ điển từ lâu trước deep learning.

**Giới hạn — đây là chỗ khác biệt quan trọng nhất so với kiến trúc planner–controller cổ điển:** trong robotics cổ điển, planner và controller thường là **hai module được thiết kế/huấn luyện tách biệt**, giao tiếp qua một giao diện đơn giản đã định nghĩa trước (ví dụ setpoint vị trí khớp). Ngược lại, paper GR00T N1 nêu rõ System 2 và System 1 **"are tightly coupled and jointly trained end-to-end"** — chúng được huấn luyện *cùng nhau*, gradient chảy qua cả hai, và giao tiếp qua cơ chế cross-attention học được (không phải một giao diện setpoint cứng do người thiết kế). Nếu chỉ dùng phép loại suy planner–controller mà không biết chi tiết này, người học rất dễ hiểu nhầm rằng System 2 và System 1 là hai model độc lập, huấn luyện riêng rồi ghép lại — đây chính là hiểu nhầm phổ biến số 1 sẽ nói ở mục "Sai lầm thường gặp".

## 📐 Định nghĩa chính xác

> Nguồn: **GR00T N1 — An Open Foundation Model for Generalist Humanoid Robots**, NVIDIA et al., [arXiv:2503.14734](https://arxiv.org/abs/2503.14734) (bản v1, tháng 3/2025).

GR00T N1 có **"a dual-system architecture"** gồm hai module:

| | **System 2** | **System 1** |
|---|---|---|
| Vai trò | *"interprets the environment through vision and language instructions"* | *"generates fluid motor actions in real time"* |
| Loại kiến trúc | Vision-Language Model (VLM) pretrained | Diffusion Transformer (DiT), mục tiêu huấn luyện là **flow matching** |
| Backbone cụ thể | **Eagle-2** (Li et al., 2025), pretrained trên dữ liệu internet-scale | DiT với alternating self-attention/cross-attention blocks |
| Tần số hoạt động | **10Hz** trên NVIDIA L40 GPU | **120Hz** |
| Đầu ra | embedding ngữ nghĩa $\phi_t$ (không trực tiếp sinh hành động) | action chunk $A_t = [a_t, a_{t+1}, \dots, a_{t+H-1}]$, với **H = 16** |
| Tham số | **1.34B** (thuộc tổng 2.2B của checkpoint GR00T-N1-2B) | phần còn lại ≈ **0.86B** (2.2B − 1.34B), không có con số công bố tách riêng |

Hai điểm định nghĩa quan trọng cần nhớ chính xác từng chữ:

1. **"Tightly coupled and jointly trained end-to-end"** — đây không phải hai model tách rời. Gradient của loss huấn luyện chảy qua cả System 1 lẫn System 2 trong cùng một vòng lặp huấn luyện.
2. **Cơ chế nối:** paper mô tả cụ thể — action chunk bị làm nhiễu (noised action chunk) được xử lý qua các khối **"alternating cross-attention and self-attention blocks, similar to Flamingo"**: self-attention giữa các action token noised với nhau, xen kẽ với cross-attention cho phép *"conditioning on the vision-language token embeddings $\phi_t$ output by VLM"*. Nói cách khác, ở mỗi bước sinh hành động, System 1 luôn "nhìn lại" biểu diễn ngữ nghĩa mà System 2 vừa tạo ra.
3. Trong quá trình rút trích $\phi_t$, paper còn nêu một chi tiết cụ thể: thay vì dùng biểu diễn ở lớp cuối của Eagle-2, nhóm tác giả **dùng biểu diễn từ lớp thứ 12 (12th layer)** của LLM bên trong VLM — vì lớp trung gian giữ nhiều thông tin ngữ nghĩa hữu ích cho điều khiển hơn lớp cuối (vốn được tối ưu cho sinh văn bản).

**Khung ẩn dụ đặt tên (nhắc lại, quan trọng):** *"System 1 / System 2"* mượn từ **Daniel Kahneman, "Thinking, Fast and Slow" (2011)** — hệ tư duy nhanh/trực giác/tự động so với hệ tư duy chậm/có ý thức/tốn công sức. Đây **chỉ là ẩn dụ đặt tên**, không phải tuyên bố GR00T N1 dùng cùng cơ chế tính toán với não người. Hai hệ trong GR00T N1 là hai mạng neural nhân tạo hoàn toàn (một VLM, một diffusion/flow-matching transformer), không có quan hệ cơ chế với tâm lý học nhận thức.

### Mục tiêu huấn luyện của System 1 (chỉ nhắc ngắn — chi tiết xem bài giảng riêng về flow matching)

Paper viết mục tiêu huấn luyện của DiT dưới dạng flow-matching loss (ký hiệu gần đúng, giữ nguyên cấu trúc công thức gốc):

$$\mathcal{L}_{fm}(\theta) = \mathbb{E}_{\tau}\Big[\big\|\, V_\theta(\phi_t, A_t^{\tau}, q_t) - (\epsilon - A_t) \,\big\|^2\Big]$$

Trong đó $V_\theta$ là mạng DiT cần học (tham số $\theta$), $\phi_t$ là embedding từ System 2, $A_t^{\tau}$ là action chunk ở bước nhiễu $\tau$, $q_t$ là proprioception, và $\epsilon$ là nhiễu ngẫu nhiên ban đầu. Trực giác ở mức tối thiểu cần biết cho bài này: mạng học cách dự đoán một "vector trường" (velocity field) đưa một action chunk toàn nhiễu dần dần biến đổi thành action chunk thật — đây là lý do vì sao System 1 được huấn luyện cần **K bước lặp** (K=4 lúc suy luận) để đi từ nhiễu tới hành động sạch. **Không đào sâu công thức này ở đây** — bài giảng riêng về flow matching sẽ giải thích đầy đủ vì sao mục tiêu này khác diffusion cổ điển và vì sao nó hội tụ nhanh hơn (ít bước hơn) trong suy luận.

### Dữ liệu và quy mô huấn luyện (bối cảnh giúp hiểu vì sao 2 hệ cần "tightly coupled")

Paper nêu GR00T N1 huấn luyện trên **"a heterogeneous mixture of real-robot trajectories, human videos, and synthetically generated datasets"**. Ba nguồn dữ liệu cụ thể được nhắc tới:

| Nguồn dữ liệu | Quy mô (paper) | Đặc điểm |
|---|---|---|
| Quỹ đạo robot thật (real-robot trajectories, GR00T N1 humanoid data) | **88 giờ** | Ít, đắt để thu thập, nhưng "sạch" — đúng embodiment thật. |
| Video con người / "neural trajectories" | **827 giờ** | Nhiều hơn hẳn, rẻ hơn để thu thập, nhưng cần suy luận hành động gián tiếp (không có nhãn action trực tiếp như robot thật). |
| Dữ liệu tổng hợp từ mô phỏng (qua DexMimicGen) | tương đương **6.500 giờ** (780.000 quỹ đạo) | Số lượng gần như không giới hạn, nhưng có domain gap giữa sim và thật. |

Vì cả ba nguồn dữ liệu này đều được đưa qua **cùng một pipeline huấn luyện end-to-end** (không huấn luyện System 2 trên tập này rồi System 1 trên tập khác một cách tách biệt), việc "tightly coupled and jointly trained" không chỉ là một câu mô tả kiến trúc suông — nó phản ánh đúng cách dữ liệu hỗn hợp này được dùng để tối ưu đồng thời cả hai module. Quy mô tính toán tương ứng: paper nêu huấn luyện dùng **tới 1024 GPU** cho một model, tiêu tốn khoảng **50.000 giờ H100 GPU** cho giai đoạn pretraining, với optimizer **AdamW, learning rate 1e-4**.

## ⚙️ Cơ chế hoạt động — từng bước

Sơ đồ luồng dữ liệu đầy đủ, một chu kỳ điều khiển:

```text
                         ┌─────────────────────────────────────┐
                         │            SYSTEM 2 (10Hz)           │
   camera frame  ──────▶ │   Eagle-2 VLM (vision-language)      │
   language cmd  ──────▶ │   → lấy representation ở layer 12    │
                         │   → sinh embedding ngữ nghĩa  φ_t     │
                         └──────────────────┬────────────────────┘
                                            │ φ_t (cập nhật mỗi 100ms)
                                            │  giữ nguyên trong ~12 bước System 1
                                            ▼
                         ┌─────────────────────────────────────┐
              q_t  ─────▶│            SYSTEM 1 (120Hz)          │
        (proprioception) │   DiT — flow-matching action head    │
   noised action chunk ─▶│                                       │
   A_t^τ  (τ = bước      │   [self-attn giữa action-token noised]│
    denoising, τ=1..K)   │        ⇅ xen kẽ ⇅                     │
                         │   [cross-attn: query = action token,  │
                         │    key/value = φ_t từ System 2]       │
                         │                                       │
                         │   lặp K=4 bước denoising (flow steps) │
                         └──────────────────┬────────────────────┘
                                            │
                                            ▼
                     action chunk  A_t = [a_t, a_{t+1}, ..., a_{t+15}]   (H=16)
                                            │
                                            ▼
                              robot thực thi từng a_i, mỗi bước 1/120s
```

Diễn giải từng bước:

1. **System 2 chạy 1 lần** (10Hz → chu kỳ 100ms): nhận ảnh camera hiện tại + câu lệnh ngôn ngữ, chạy qua Eagle-2 VLM, lấy biểu diễn ở layer 12, sinh ra embedding ngữ nghĩa $\phi_t$. $\phi_t$ đóng vai trò **điều kiện (condition)** — nó không phải hành động, chỉ là "bối cảnh + ý định" được mã hoá thành vector.
2. **$\phi_t$ được giữ cố định** trong khi System 1 chạy nhiều bước liên tiếp — vì System 2 chạy chậm hơn 12 lần, $\phi_t$ không đổi trong suốt khoảng thời gian đó (đây chính là điểm mấu chốt cho phép tách tần số: System 1 không cần chờ System 2 ở mỗi bước).
3. **System 1 chạy vòng lặp nhanh** (120Hz → chu kỳ ≈8.33ms): tại mỗi bước, nhận thêm proprioception hiện tại $q_t$ (trạng thái khớp), khởi tạo một action chunk bị nhiễu (noised action chunk, xuất phát từ nhiễu ngẫu nhiên theo công thức flow matching — chi tiết toán học xem bài giảng riêng về flow matching), rồi xử lý qua các khối attention xen kẽ:
   - **self-attention**: các action token trong chunk "nhìn nhau" để giữ tính mượt/liên tục theo thời gian.
   - **cross-attention**: mỗi action token "nhìn vào" $\phi_t$ để đảm bảo hành động sinh ra đúng với ý định ngữ nghĩa mà System 2 đã xác định.
4. Quá trình denoising (làm sạch nhiễu dần) này lặp **K=4 bước** (paper: *"we found K=4 inference steps to work well across all embodiments"*) để thu được action chunk cuối cùng $A_t = [a_t, \dots, a_{t+15}]$ với $H=16$.
5. Robot thực thi tuần tự từng hành động $a_i$ trong chunk, mỗi hành động ứng với 1 bước điều khiển ở tần số 120Hz — trong khi đó, hệ thống có thể tiếp tục sinh chunk mới theo kiểu closed-loop (chunk sau chồng lấp/thay thế chunk trước khi có quan sát mới), giữ vòng điều khiển luôn "closed-loop" thay vì mở hoàn toàn 133ms rồi mới nhìn lại.
6. Toàn bộ quá trình trên được huấn luyện **end-to-end** — nghĩa là loss (flow-matching loss, ký hiệu $\mathcal{L}_{fm}$) được backprop xuyên suốt cả DiT (System 1) lẫn Eagle-2 (System 2), không phải huấn luyện riêng rồi đóng băng một bên.

### Dòng thời gian trực quan: 1 chu kỳ System 2 = 12 chu kỳ System 1

Sơ đồ trên chỉ vẽ *một* lần System 2 chạy. Trên thực tế hai vòng lặp chạy **song song, không đồng bộ theo từng bước** — System 2 chạy nền ở tần số thấp trong khi System 1 liên tục lấy $\phi_t$ mới nhất có sẵn (không chờ). Trải theo trục thời gian thực (mỗi ô nhỏ ≈ 8.33ms):

```text
System 2 (10Hz):   [========== φ_t ==========][========= φ_t+1 =========]
                    │0ms                     100ms                   200ms

System 1 (120Hz):  [1][2][3][4][5][6][7][8][9][10][11][12][1][2][3]...
                    │   mỗi ô = 1 bước ≈8.33ms, dùng chung φ_t          │
                    └──────────── 12 bước dùng cùng 1 φ_t ─────────────┘
```

Đọc sơ đồ: trong suốt 100ms mà System 2 còn đang "nghĩ" ra $\phi_{t+1}$, System 1 **không đứng yên chờ** — nó vẫn chạy đủ 12 bước liên tục dựa trên $\phi_t$ (giá trị embedding cũ nhất vừa có). Khi $\phi_{t+1}$ sẵn sàng, System 1 chuyển sang dùng giá trị mới ngay ở bước tiếp theo. Đây chính là cơ chế cho phép vòng điều khiển robot **không bao giờ bị "đứng hình"** chờ VLM, dù VLM chạy chậm hơn nhiều — đánh đổi là System 1 luôn hành động dựa trên một $\phi_t$ có thể "hơi cũ" tối đa 100ms, điều này chấp nhận được vì bối cảnh ngữ nghĩa (ví dụ "cầm cái cốc màu đỏ") hiếm khi đổi trong 100ms.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

**(a) Tỉ lệ tần số System 1 / System 2:**

$$\frac{120\text{Hz}}{10\text{Hz}} = 12$$

Nghĩa là: cứ **mỗi lần System 2 cập nhật embedding ngữ nghĩa $\phi_t$**, System 1 chạy được **12 bước điều khiển motor** dựa trên cùng một $\phi_t$ đó (cho tới khi System 2 cập nhật $\phi_t$ mới).

**(b) Chu kỳ thời gian mỗi bước:**

- System 2 (10Hz): chu kỳ $= \dfrac{1}{10\text{Hz}} = 0.1\text{s} = 100\text{ms}$ cho mỗi lần suy luận ngữ nghĩa.
- System 1 (120Hz): chu kỳ $= \dfrac{1}{120\text{Hz}} \approx 0.00833\text{s} \approx 8.33\text{ms}$ cho mỗi bước sinh hành động.

Kiểm tra chéo: $12 \times 8.33\text{ms} = 99.96\text{ms} \approx 100\text{ms}$ — khớp với chu kỳ của System 2, xác nhận lại tỉ lệ 12 ở trên.

**(c) Một action chunk bao phủ bao nhiêu thời gian thực?**

Với $H = 16$ hành động, mỗi hành động cách nhau $1/120\text{s}$:

$$\frac{H}{120\text{Hz}} = \frac{16}{120}\text{s} \approx 0.1333\text{s} \approx 133\text{ms}$$

Nghĩa là mỗi lần System 1 "suy nghĩ" xong, nó đã lên kế hoạch động tác cho khoảng **133ms tiếp theo** — dài hơn một chút so với 1 chu kỳ System 2 (100ms), đủ để tạo độ trễ đệm (buffer) an toàn nếu System 2 cập nhật hơi chậm.

**(d) Phân rã tham số:**

Tổng tham số checkpoint công khai GR00T-N1-2B: **2.2B**. Trong đó System 2 (Eagle-2 VLM): **1.34B**.

$$\text{System 1} = 2.2\text{B} - 1.34\text{B} = 0.86\text{B}$$

Tỉ lệ phần trăm:

$$\frac{1.34}{2.2} \approx 60.9\% \quad\text{(System 2)} \qquad \frac{0.86}{2.2} \approx 39.1\% \quad\text{(System 1)}$$

Trực giác: gần **2/3 tổng tham số nằm ở System 2** (VLM hiểu ngôn ngữ/thị giác — bài toán "hiểu thế giới" vốn cần capacity lớn vì phải tổng quát hoá trên internet-scale data), còn **System 1 nhỏ hơn nhiều (~39%)** dù phải chạy nhanh gấp 12 lần — đây không phải trùng hợp mà là lựa chọn thiết kế bắt buộc (giải thích ở mục dưới).

**(e) Tốc độ suy luận thực đo (paper):** với cấu hình trên, GR00T N1 công bố tốc độ **63.9ms trên GPU L40 (bf16)** để sample xong 1 action chunk 16 hành động (bao gồm cả K=4 bước denoising) — nhỏ hơn 133ms (thời lượng chunk bao phủ), nghĩa là hệ thống về lý thuyết có thể sinh chunk kịp thời trước khi chunk cũ hết hiệu lực, giữ được vòng lặp thời gian thực.

**(f) Biên độ an toàn (safety margin) tính bằng số:**

$$133\text{ms (thời lượng chunk)} - 63.9\text{ms (thời gian sinh chunk mới)} = 69.1\text{ms}$$

Nói cách khác, hệ thống còn dư **69.1ms** — tương đương gần **8.3 bước System 1** ($69.1\text{ms} / 8.33\text{ms} \approx 8.3$) — làm vùng đệm trước khi chunk hiện tại dùng hết. Đây là lý do thiết kế $H=16$ (133ms) thay vì chọn $H$ nhỏ hơn: nếu $H$ quá nhỏ (ví dụ $H=8$, chỉ bao phủ 66.7ms), biên độ an toàn sẽ âm ($66.7 - 63.9 = 2.8\text{ms}$, quá mỏng), khiến hệ thống dễ "hết hành động" trước khi kịp tính xong chunk kế tiếp.

**(g) Quy mô tính toán huấn luyện, diễn giải lại từ số liệu paper:** với ngân sách "tới 1024 GPU" và "~50.000 giờ H100 GPU" cho pretraining, nếu dùng **toàn bộ 1024 GPU chạy song song liên tục**, thời gian thực tế (wall-clock) ước tính:

$$\frac{50{,}000 \text{ giờ-GPU}}{1024 \text{ GPU}} \approx 48.8 \text{ giờ} \approx 2 \text{ ngày}$$

(đây là phép tính suy ra từ hai con số paper đã công bố — bản thân paper không nói thẳng "2 ngày", con số 2 ngày là kết quả tính tay của người học, chỉ mang tính minh hoạ quy mô, thực tế còn phụ thuộc hiệu suất scaling thực khi dùng nhiều GPU song song).

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | **GR00T N1 (dual-system)** | **Single-system VLA (RT-2 / OpenVLA)** | **Helix (Figure AI, 2/2025)** |
|---|---|---|---|
| Số module sinh hành động | 2 (System 2 suy luận + System 1 sinh hành động), tách tần số | 1 (chính VLM sinh action token trực tiếp, autoregressive) | 2 — cùng triết lý dual-system với GR00T |
| Tần số suy luận ngữ nghĩa | System 2: **10Hz** | Cả pipeline chạy chung 1 tần số, bị giới hạn bởi tốc độ VLM | System 2 (7B param VLM): **7–9Hz** |
| Tần số sinh hành động | System 1: **120Hz** | Do autoregressive decode tuần tự từng token hành động, tốc độ thực đo: OpenVLA ≈ **0.33s/hành động (~3Hz)** trên A100 — early VLA nói chung bị giới hạn ở khoảng **2–5Hz** | System 1 (80M param): **200Hz**, điều khiển 35 bậc tự do (DoF) phần thân trên |
| Cơ chế sinh hành động | Diffusion Transformer + flow matching (liên tục) | Token hoá hành động rời rạc (discrete action tokens), decode tuần tự từng token | Visuomotor policy riêng biệt, tách khỏi VLM |
| Huấn luyện | Jointly trained end-to-end, tightly coupled | Một model duy nhất, huấn luyện chung | Hai model tách biệt về tần số/kiến trúc |
| Điểm mạnh | Vừa tận dụng VLM lớn cho hiểu ngữ nghĩa, vừa giữ vòng điều khiển đủ nhanh cho robot thật | Đơn giản, một pipeline duy nhất | Cực nhanh (200Hz) phù hợp thao tác tay tinh vi |
| Điểm yếu | Phức tạp hơn để huấn luyện/đồng bộ 2 module | Tần số thấp (~2–5Hz) không đủ cho điều khiển motor mượt trên robot thật → cần bù bằng action chunking hoặc chạy open-loop dài hơn | Cần đồng bộ 2 tiến trình chạy song song ở tần số khác xa nhau |

> Nguồn số liệu Helix: [Figure AI — "Helix: A Vision-Language-Action Model for Generalist Humanoid Control"](https://www.figure.ai/news/helix) (2/2025); Decrypt, ["Figure AI Is Supercharging Humanoid Robots"](https://decrypt.co/307058/figure-ai-supercharging-humanoid-robots).
> Nguồn số liệu OpenVLA/early VLA (~2–5Hz, ~3Hz cho OpenVLA autoregressive): tổng hợp từ khảo sát kỹ thuật gần đây về tối ưu tốc độ VLA, ví dụ ["Fine-Tuning Vision-Language-Action Models: Optimizing Speed and Success"](https://arxiv.org/html/2502.19645v1) (2025) và ["TurboVLA: Real-Time Vision-Language-Action Model"](https://arxiv.org/pdf/2607.27205) (2026, dùng làm đối chứng cho các con số VLA đời đầu). Đây là số liệu về các hệ single-system nói chung, không phải trích trực tiếp từ paper RT-2 gốc — nếu cần số chính xác từ chính paper RT-2/OpenVLA, cần xác minh thêm.

**Điểm mấu chốt của bảng so sánh:** cả GR00T N1 lẫn Helix đều đi đến cùng một kết luận kiến trúc — **tách VLM (chậm, hiểu ngữ nghĩa) khỏi action head (nhanh, điều khiển motor)** — dù backbone, tần số cụ thể, và tỉ lệ tham số khác nhau khá nhiều (GR00T: 1.34B/0.86B ở 10Hz/120Hz; Helix: 7B/80M ở 7-9Hz/200Hz). Đây là dấu hiệu cho thấy dual-system không phải lựa chọn riêng của NVIDIA mà đang trở thành **một mẫu thiết kế (design pattern) chung** cho VLA humanoid tần số cao.

**So sánh tỉ lệ tần số nhanh/chậm giữa các kiến trúc dual-system (để thấy 12 lần của GR00T không phải con số cố định của "mẫu thiết kế"):**

| Kiến trúc | Tần số System 2 (chậm) | Tần số System 1 (nhanh) | Tỉ lệ nhanh/chậm |
|---|---|---|---|
| GR00T N1 | 10Hz | 120Hz | 12× |
| Helix (Figure AI) | 7–9Hz | 200Hz | ≈22–28× |

Quan sát: cả hai đều chọn tỉ lệ khá lớn (System 1 nhanh hơn System 2 từ chục tới vài chục lần), nhưng **con số cụ thể khác nhau tuỳ ràng buộc phần cứng và bài toán điều khiển của từng robot** — không có một "tỉ lệ chuẩn" chung cho mọi dual-system VLA. Điều bất biến giữa các kiến trúc là *nguyên lý* (tách nhanh/chậm theo bản chất bài toán con), không phải con số Hz cụ thể.

**Vì sao so sánh với RT-2/OpenVLA quan trọng hơn là chỉ nói "chúng chậm hơn":** sự khác biệt cốt lõi không nằm ở việc RT-2/OpenVLA "kém" mà ở **chỗ đặt ranh giới kiến trúc**. RT-2/OpenVLA dùng đúng một mạng để vừa hiểu ngữ cảnh vừa quyết định từng thành phần hành động (vị trí, hướng, gripper) theo kiểu tuần tự token-by-token — nên độ trễ của toàn hệ thống bị trói chặt vào tốc độ của chính VLM đó. GR00T N1 (và Helix) "cắt" hệ thống làm hai, cho phép phần điều khiển tần số cao (System 1) không bao giờ phải chờ phần suy luận ngữ nghĩa nặng (System 2) chạy xong ở mỗi bước — đây là lý do kỹ thuật thật sự đứng sau con số Hz, không đơn thuần là "model nhỏ hơn thì nhanh hơn".

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

**Hiểu nhầm 1: "System 2 và System 1 là hai model huấn luyện độc lập, ghép lại lúc suy luận (inference)."**
→ Vì sao sai: paper nêu rõ hai module **"tightly coupled and jointly trained end-to-end"** — chúng được huấn luyện *cùng nhau* trong cùng một vòng lặp, có gradient chảy qua cả hai, không phải "train xong System 2 rồi đóng băng, sau đó train System 1 riêng dựa trên output của System 2". Cơ chế cross-attention nối chúng cũng là thành phần được học (learned), không phải một giao diện cố định do người thiết kế.
→ Hiểu đúng: đây là một mạng lớn duy nhất về mặt huấn luyện, chỉ được *tổ chức* thành hai module chức năng chạy ở hai tần số khác nhau lúc suy luận.

**Hiểu nhầm 2: "120Hz nghĩa là robot chuyển động nhanh hơn 12 lần so với chạy ở 10Hz."**
→ Vì sao sai: 120Hz là **tần số cập nhật lệnh điều khiển** (tần số System 1 tính toán và phát ra hành động mới), không phải tốc độ chuyển động vật lý của robot. Robot có thể di chuyển chậm rãi, nhẹ nhàng dù vòng điều khiển chạy ở 120Hz — tần số cao ở đây phục vụ mục đích **mượt mà và ổn định** (closed-loop feedback nhanh, phản ứng kịp thời với nhiễu/mất cân bằng), không phải "tốc độ hành động nhanh hơn".
→ Hiểu đúng: tần số 120Hz nghĩa là cứ 8.33ms, hệ thống lại tính lại/cập nhật hành động một lần dựa trên trạng thái mới nhất — giống như tần số làm mới của một vòng lặp điều khiển PID trong robot cổ điển, không liên quan trực tiếp đến biên độ/tốc độ chuyển động.

**Hiểu nhầm 3 (bổ sung, thường gặp khi mới đọc): "System 1 tự nó sinh ra ngữ nghĩa/hiểu ngôn ngữ, System 2 tự nó điều khiển khớp."**
→ Vì sao sai: đảo ngược vai trò. System 2 **không** trực tiếp sinh hành động (paper nói rõ System 2 chỉ tạo ra biểu diễn ngữ nghĩa $\phi_t$ làm điều kiện); System 1 **không** tự hiểu ngôn ngữ từ đầu, nó chỉ nhận $\phi_t$ đã được System 2 mã hoá sẵn qua cross-attention.
→ Hiểu đúng: System 2 = "hiểu và mã hoá ý định", System 1 = "dịch ý định đã mã hoá thành chuỗi hành động motor mượt".

**Hiểu nhầm 4: "Vì System 1 dùng cross-attention để 'nhìn' System 2 ở mỗi bước, nghĩa là System 2 cũng phải chạy lại (re-run) ở mỗi bước 120Hz đó — nên thực ra cả hệ vẫn bị giới hạn bởi tốc độ VLM."**
→ Vì sao sai: cross-attention của System 1 chỉ **đọc lại** giá trị $\phi_t$ đã được System 2 tính sẵn và giữ trong bộ nhớ (cache) — nó không kích hoạt một lượt forward-pass mới của Eagle-2. Trong 12 bước System 1 chạy giữa hai lần cập nhật của System 2, $\phi_t$ là *hằng số* được tái sử dụng, không phải giá trị tính lại mỗi lần. Đây chính là điều làm cho việc tách tần số có ý nghĩa thực tế — nếu mỗi bước cross-attention đều phải chạy lại toàn bộ VLM, tách tần số sẽ vô nghĩa vì tốc độ tổng thể vẫn bị trói vào 10Hz.
→ Hiểu đúng: cross-attention trong DiT chỉ là một phép nhân ma trận nhẹ (attention giữa action token và $\phi_t$ đã cache), chi phí tính toán của nó nhỏ hơn rất nhiều so với một lượt forward-pass đầy đủ qua Eagle-2 — đó là lý do System 1 vẫn giữ được 120Hz dù "phụ thuộc" vào $\phi_t$ của System 2.

## 🏗️ Ví dụ minh hoạ trong dự án này

Trong lộ trình học của mảng 06, khung dual-system này chính là "bộ xương" mà tất cả các phiên bản sau kế thừa và chỉ thay từng mảnh:

- **N1.5 → N1.6** (bài giảng riêng): giữ nguyên khung System 2 + System 1, cải thiện chủ yếu ở dữ liệu huấn luyện và tách phần whole-body control (WBC) thành nền chung với repo `GR00T-WholeBodyControl` — nhưng vẫn là "VLM chậm điều kiện hoá action head nhanh".
- **N1.7** (bài giảng riêng: EgoScale & scaling law): pretraining trên tập dữ liệu egocentric video quy mô lớn hơn nhiều (EgoScale, 20,854 giờ) — về bản chất là **mở rộng dữ liệu huấn luyện cho phần System 2 (hiểu ngữ nghĩa/hành vi từ video người)**, trong khi vẫn dùng flow-matching action transformer tương tự vai trò System 1. Nói cách khác, N1.7 không thay đổi khung dual-system, mà tăng chất lượng "đầu vào ngữ nghĩa" mà System 1 được điều kiện hoá theo.
- **Tích hợp với SONIC** (bài giảng riêng: unified token space): output của System 1 — action chunk $A_t$ — chính là thứ được đưa vào "không gian token thống nhất" mà SONIC dùng để điều khiển whole-body. Việc System 1 chạy tách biệt ở 120Hz, độc lập tần số với System 2, là lý do khiến việc "cắm" nó vào một vòng điều khiển whole-body tần số cao (vốn cũng cần chạy nhanh để giữ thăng bằng) trở nên khả thi về mặt kỹ thuật — nếu toàn bộ pipeline phải chờ VLM chạy ở 10Hz cho mọi hành động, ghép với WBC tần số cao sẽ không khả thi.

Nói ngắn gọn: hiểu đúng bài này giúp thấy rõ **tại sao GR00T N1 có thể "nói chuyện" được với lớp WBC/SONIC ở `01-whole-body-control/`** — chính nhờ System 1 đã tách sẵn ra một luồng tần số cao, tương thích về mặt tốc độ với vòng điều khiển robot thật.

Một cách hình dung khác, đối chiếu với chính khái niệm "decoupled WBC" của SONIC (đã học ở `01-whole-body-control/`, bài giảng riêng): SONIC tách WBC thành hai lớp — một lớp học kỹ năng vận động tổng quát (huấn luyện offline, tần số thấp hơn về mặt "ra quyết định chiến lược") và một lớp thực thi tần số cao bám theo lệnh. Đây là cùng một *nguyên lý tách tần số theo bản chất bài toán con* mà GR00T N1 áp dụng ở tầng VLA — cho thấy toàn bộ pipeline của dự án này, từ tầng nhận thức cao nhất (GR00T System 2) xuống tới tầng điều khiển thấp nhất (WBC/SONIC), đều lặp lại cùng một mẫu thiết kế: **"chậm mà sâu" ở trên, "nhanh mà hẹp" ở dưới**, nối nhau qua một biểu diễn trung gian gọn nhẹ (ở đây là $\phi_t$; ở SONIC là token/latent action).

## 🔥 Cập nhật hiện đại / SOTA gần đây

Dựa trên WebSearch/WebFetch thực hiện khi viết bài này (9/2026):

1. **Dual-system đang trở thành một mẫu thiết kế phổ biến cho VLA humanoid, không chỉ riêng GR00T.** Ngoài GR00T N1 và Helix (Figure AI, 2/2025 — System 2: VLM 7B tham số ở 7–9Hz; System 1: visuomotor policy 80M tham số ở 200Hz, điều khiển 35 DoF phần thân trên), một loạt paper 2025–2026 tiếp tục khai thác ý tưởng "tách tần số nhanh/chậm": **"Fast-in-Slow: A Dual-System Foundation Model Unifying Fast Manipulation within Slow Reasoning"** ([arXiv:2506.01953](https://arxiv.org/pdf/2506.01953), 2025) và **"Asynchronous Fast-Slow Vision-Language-Action Policies for Whole-Body Robotic Manipulation"** ([arXiv:2512.20188](https://arxiv.org/pdf/2512.20188), 12/2025) — bài này đặc biệt gần với bối cảnh humanoid whole-body của dự án này. Gần đây hơn có **"Libra-VLA: Achieving Learning Equilibrium via Asynchronous Coarse-to-Fine Dual-System"** ([arXiv:2604.24921](https://arxiv.org/pdf/2604.24921), 2026), tiếp tục tinh chỉnh cách cân bằng huấn luyện giữa nhánh coarse (chậm, giống System 2) và nhánh fine (nhanh, giống System 1).
2. **Xu hướng ngược lại cũng đang phát triển song song: cố "dồn" cả single-system chạy nhanh hơn thay vì tách hai hệ.** Ví dụ **TurboVLA** ([arXiv:2607.27205](https://arxiv.org/pdf/2607.27205), 2026) đạt độ trễ end-to-end 31.2ms (tương đương hơn 30 lần dự đoán action-chunk/giây, ~32Hz) cho một kiến trúc real-time, cho thấy cộng đồng đang tìm cách thu hẹp khoảng cách tần số giữa VLM và action head thay vì luôn tách rời chúng — một hướng thay thế khả dĩ cho dual-system nếu tối ưu đủ tốt phần suy luận VLM.
3. **Danh mục tổng hợp các robot foundation model 2025-2026** (repo cộng đồng ["Awesome-Robot-Foundation-Models-2025-2026"](https://github.com/jinruih2/Awesome-Robot-Foundation-Models-2025-2026)) ghi nhận: flow matching đã trở thành "paradigm thống trị" cho action generation trong các VLA hiện đại (GR00T, họ π₀, SmolVLA...) — củng cố lựa chọn thiết kế flow-matching cho System 1 mà GR00T N1 dùng ngay từ bản gốc 3/2025 không phải là lựa chọn tạm thời mà đã trở thành chuẩn mực chung của ngành.

**Về việc xác minh số liệu gốc:** khi fetch trực tiếp bản HTML của paper (arxiv.org/html/2503.14734v1), các con số 10Hz / 120Hz / H=16 / 2.2B / 1.34B đều **khớp chính xác** với những gì đã nêu ở nguồn cho bài này — không có sai lệch. Paper còn tiết lộ thêm một số chi tiết không có trong bản tóm tắt gốc nhưng đã xác minh được qua WebFetch (không phải suy diễn): số bước denoising **K=4**, việc dùng biểu diễn ở **layer 12** của LLM thay vì layer cuối, cơ chế attention **"alternating cross-attention and self-attention, similar to Flamingo"**, tốc độ suy luận thực đo **63.9ms trên L40 GPU (bf16)** cho một action chunk, quy mô huấn luyện **lên tới 1024 GPU, ~50.000 giờ H100** cho pretraining, optimizer **AdamW, learning rate 1e-4**, và dữ liệu huấn luyện gồm khoảng **88 giờ** quỹ đạo robot thật, **827 giờ** video/neural trajectories, và tương đương **6.500 giờ** dữ liệu mô phỏng (780.000 quỹ đạo qua DexMimicGen). Các con số này **không nằm trong yêu cầu cốt lõi phải trích dẫn của bài** nhưng được ghi lại ở đây vì đã xác minh trực tiếp qua paper gốc, giúp bức tranh đầy đủ hơn. Riêng số layer/attention-head cụ thể **bên trong** khối DiT của System 1 (ví dụ độ sâu mạng, số head) — paper **không công bố chi tiết này công khai**; nếu cần, đây là điểm **cần xác minh thêm** (có thể nằm trong code `NVIDIA/Isaac-GR00T` chứ không phải trong paper).

## ❓ Câu hỏi tự kiểm tra

1. System 2 và System 1 trong GR00T N1 chạy ở tần số bao nhiêu, tương ứng module nào chịu trách nhiệm gì?
<details><summary>Gợi ý đáp án</summary>System 2 (Eagle-2 VLM) chạy ở 10Hz, đọc camera + ngôn ngữ, sinh embedding ngữ nghĩa φ_t. System 1 (DiT flow-matching) chạy ở 120Hz, nhận φ_t qua cross-attention cùng proprioception q_t, sinh action chunk A_t.</details>

2. Nếu robot chạy đúng 3 giây liên tục, System 1 đã chạy bao nhiêu bước tính toán hành động (giả sử không gián đoạn)?
<details><summary>Gợi ý đáp án</summary>120Hz × 3s = 360 bước.</details>

3. Một action chunk với H=16 ở tần số 120Hz bao phủ bao nhiêu mili-giây thời gian thực? Nếu tốc độ suy luận thực đo là 63.9ms/chunk, hệ thống có kịp sinh chunk mới trước khi chunk cũ hết hạn không?
<details><summary>Gợi ý đáp án</summary>16/120 ≈ 0.1333s ≈ 133ms. Vì 63.9ms < 133ms, về lý thuyết hệ thống kịp sinh chunk mới trước khi chunk hiện tại dùng hết, còn dư khoảng 69ms biên độ an toàn.</details>

4. Tại sao không thể chỉ dùng một model VLM lớn duy nhất chạy ở 120Hz để vừa hiểu ngữ nghĩa vừa sinh hành động?
<details><summary>Gợi ý đáp án</summary>Ràng buộc tính toán: một VLM lớn (1.34B/2.2B tham số, tương đương ~61% tổng model) quá nặng để chạy ở 120Hz với phần cứng hiện tại — bằng chứng gián tiếp: các VLA single-system dùng VLM trực tiếp sinh hành động (như OpenVLA) chỉ đạt khoảng 2-5Hz do decode tuần tự token hành động. Action head nhỏ hơn (System 1, ~0.86B, 39% tổng model) có không gian đầu ra hẹp hơn nhiều nên chạy nhanh hơn được.</details>

5. Nếu checkpoint GR00T-N1-2B có tổng 2.2B tham số và System 2 chiếm 1.34B, System 1 chiếm bao nhiêu tham số và bao nhiêu phần trăm tổng model?
<details><summary>Gợi ý đáp án</summary>2.2B − 1.34B = 0.86B, tương đương ≈ 39.1% tổng tham số (System 2 ≈ 60.9%).</details>

6. Giải thích vì sao câu "System 2 và System 1 là hai model huấn luyện tách biệt" là sai, dẫn đúng cụm từ trong paper để chứng minh.
<details><summary>Gợi ý đáp án</summary>Sai vì paper nêu rõ hai hệ "are tightly coupled and jointly trained end-to-end" — nghĩa là huấn luyện chung, gradient chảy qua cả hai, cơ chế cross-attention nối chúng cũng được học chứ không phải giao diện cố định thiết kế sẵn.</details>

## 📝 Bài tập thực hành

1. **Tính lại với một tần số giả định khác (ví dụ minh hoạ, số tự chọn để luyện tập, không phải số thật của GR00T N1):** giả sử một phiên bản tương lai tăng tần số System 1 lên 240Hz nhưng giữ nguyên H=16. (a) Một action chunk lúc này bao phủ bao nhiêu mili-giây thực tế? (b) Nếu vẫn giữ System 2 ở 10Hz, tỉ lệ số bước System 1/System 2 mỗi chu kỳ System 2 là bao nhiêu? So sánh với tỉ lệ gốc (12) để thấy việc tăng tần số ảnh hưởng thế nào tới độ "mượt" của action chunk so với nhịp cập nhật ngữ nghĩa.

2. **Đọc code thật:** clone hoặc duyệt repo `NVIDIA/Isaac-GR00T` trên GitHub (https://github.com/NVIDIA/Isaac-GR00T), tìm class liên quan tới DiT/flow-matching action head (thường nằm trong thư mục liên quan tới `model`/`action_head`/`policy`). Ghi lại: (a) tên class chính, (b) có tìm thấy con số cụ thể về số layer/số attention head của DiT không — nếu có, đối chiếu xem có khớp với những gì paper KHÔNG công bố công khai (mục Cập nhật hiện đại đã ghi "cần xác minh thêm") hay không, và cập nhật lại hiểu biết của bạn nếu tìm ra số liệu chính xác.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

GR00T N1 tách bài toán "điều khiển humanoid từ ảnh + ngôn ngữ" thành hai module tightly-coupled, jointly-trained end-to-end: **System 2** — một VLM (Eagle-2) chạy chậm ở **10Hz**, đọc camera và câu lệnh để sinh embedding ngữ nghĩa $\phi_t$ (chiếm 1.34B trong tổng 2.2B tham số) — và **System 1** — một Diffusion Transformer huấn luyện bằng flow matching, chạy nhanh gấp 12 lần ở **120Hz** (phần tham số còn lại, ≈0.86B), nhận $\phi_t$ qua các khối cross-attention xen kẽ self-attention để sinh action chunk $A_t$ gồm **H=16** hành động, bao phủ khoảng **133ms** thời gian thực mỗi lần. Cái tên "System 1/System 2" chỉ là ẩn dụ mượn từ Kahneman để gọi tên vai trò chức năng (suy luận chậm vs. phản xạ nhanh), không phải tuyên bố về cơ chế sinh học; bản chất kỹ thuật là hai mạng neural nhân tạo được huấn luyện chung, không phải hai model độc lập ghép lại lúc suy luận. Toàn bộ khung này được N1.5, N1.6, N1.7 kế thừa nguyên vẹn và chỉ thay đổi các mảnh bên trong, đồng thời chính vì System 1 đã tách sẵn một luồng tần số cao mà GR00T N1 mới có thể tích hợp được với lớp whole-body control/SONIC vốn cũng cần chạy ở tần số cao để giữ thăng bằng cho robot thật.
