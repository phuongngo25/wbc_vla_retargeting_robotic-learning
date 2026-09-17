# Nội dung chi tiết: VLA — GR00T N1 → N1.7 & SONIC

> File này là phần "sách giáo trình" đầy đủ cho 6 khái niệm ở mục A của `README.md`. Mục tiêu: đọc xong file này, không cần tự mở paper gốc vẫn hiểu được kiến trúc, vì sao nó được thiết kế như vậy, và các con số quan trọng — nhưng nên vẫn đọc paper gốc để có độ sâu toán học đầy đủ (paper luôn là nguồn thẩm quyền cuối cùng).
>
> Quy ước trong file: mọi con số/chi tiết kiến trúc đều được trích dẫn kèm nguồn. Chỗ nào không lấy được trực tiếp từ nguồn sơ cấp (paper/blog chính thức) mà chỉ có ở nguồn thứ cấp (bài phân tích, wiki cộng đồng...), sẽ được đánh dấu rõ **"cần xác minh thêm"**.

---

## 1. VLA (Vision-Language-Action) là gì, khác VLM thông thường ở đâu

### 1.1. Định nghĩa và khác biệt cốt lõi

**VLM (Vision-Language Model)** — ví dụ GPT-4V, LLaVA, Qwen-VL — nhận **ảnh + text** làm input, và sinh ra **text** làm output: mô tả ảnh, trả lời câu hỏi về ảnh, viết caption... Đầu ra vẫn nằm trong không gian ngôn ngữ (chuỗi token rời rạc lấy từ một từ điển hữu hạn), và VLM không "biết" gì về trạng thái vật lý hiện tại của một cơ thể robot.

**VLA (Vision-Language-Action Model)** nhận thêm một loại input thứ ba: **proprioception** — trạng thái khớp/tay hiện tại của robot (góc khớp, vận tốc, lực tiếp xúc...) — và thay vì sinh text, nó sinh ra **hành động liên tục theo thời gian**, thường gọi là **action chunk**: một chuỗi vector hành động cho H bước tiếp theo (ví dụ vị trí/vận tốc khớp mục tiêu, hoặc delta pose của end-effector). Đây là khác biệt bản chất, không chỉ là "thêm một loại input":

- VLM tối ưu để sinh phân phối xác suất trên **token rời rạc tiếp theo** (autoregressive, một token tại một thời điểm, từ một từ điển hữu hạn).
- VLA phải sinh ra **số thực liên tục**, đủ mượt và đủ nhanh để điều khiển motor thật — một chuỗi hành động giật cục hoặc trễ vài trăm mili-giây có thể khiến robot ngã hoặc làm rơi vật. Đây chính là lý do các kiến trúc VLA hiện đại (RT-2 là ngoại lệ một phần, xem dưới) không dùng thuần autoregressive token generation cho hành động mà bổ sung một **action head** chuyên biệt (diffusion hoặc flow matching — xem mục 2).

### 1.2. Dòng phát triển lịch sử: RT-1 → RT-2 → OpenVLA → π₀

Đây là bối cảnh bắt buộc phải hiểu trước khi đọc GR00T, vì GR00T N1/N1.7 kế thừa trực tiếp ý tưởng kiến trúc từ π₀ (flow-matching action head) và từ RT-2 (tận dụng VLM pretrain internet-scale).

**RT-1** — Brohan et al., Google, 2022, *"RT-1: Robotics Transformer for Real-World Control at Scale"*, [arXiv:2212.06817](https://arxiv.org/abs/2212.06817).
Đây là VLA robot tay máy đầu tiên có ảnh hưởng lớn. Theo abstract của paper: RT-1 là "a high-capacity architecture that can absorb diverse robotic data", được huấn luyện qua "large-scale data collection on real robots performing real-world tasks" — tức là **dữ liệu huấn luyện chính là dữ liệu robot thật**, không phải dữ liệu internet-scale ảnh+text. Điểm mạnh: kiến trúc transformer "hấp thụ" được dữ liệu đa dạng, thể hiện "promising scalable model properties" (hiệu năng tăng theo lượng/độ đa dạng dữ liệu). Hạn chế ngầm định (paper nhấn mạnh): việc thu thập dữ liệu robot thật rất tốn kém — đây chính là "nỗi đau" mà bước tiếp theo (RT-2) giải quyết.

**RT-2** — [arXiv:2307.15818](https://arxiv.org/abs/2307.15818).
Ý tưởng đột phá: thay vì huấn luyện một model từ đầu chỉ trên dữ liệu robot, RT-2 lấy một **VLM lớn đã pretrain trên dữ liệu internet-scale** (ảnh+text từ web) rồi **co-fine-tune** (đồng thời fine-tune) trên cả dữ liệu web gốc lẫn dữ liệu robot. Hành động được biểu diễn như **token text** — thêm vào từ vựng của VLM như một "ngôn ngữ hành động". Nhờ vậy RT-2 thừa hưởng tri thức ngữ nghĩa/thường thức (common sense) đã học từ internet — ví dụ suy luận được các khái niệm chưa từng xuất hiện trong demo robot. Đây là bước chuyển quan trọng: từ "chỉ học từ demo robot" sang "tận dụng cả tri thức internet-scale" — chính hướng đi mà GR00T N1 kế thừa (System 2 của GR00T N1 cũng là một VLM pretrain internet-scale).

**OpenVLA** — [arXiv:2406.09246](https://arxiv.org/abs/2406.09246).
Phiên bản mã nguồn mở với kiến trúc công khai rõ ràng: backbone kết hợp **Llama 2** (language model) với **visual encoder fuse đặc trưng từ DINOv2 và SigLIP** — hai vision encoder pretrain khác nhau bổ sung cho nhau (DINOv2 mạnh về đặc trưng không gian/hình học, SigLIP mạnh về đặc trưng ngữ nghĩa gắn với ngôn ngữ). Huấn luyện trên **970k demo robot thật** đa dạng nhiều robot/embodiment. Kết quả đáng chú ý: dù chỉ có 7B tham số, OpenVLA "outperforming closed models such as RT-2-X (55B) by 16.5% in absolute task success rate across 29 tasks and multiple robot embodiments, with 7x fewer parameters" — tức là **kiến trúc + chất lượng dữ liệu tốt có thể thắng việc scale thuần tham số**. OpenVLA vẫn dùng cách sinh hành động kiểu autoregressive-token (giống RT-2, hành động được rời rạc hóa thành token).

**π₀** — [arXiv:2410.24164](https://arxiv.org/abs/2410.24164), tên đầy đủ *"π₀: A Vision-Language-Action **Flow** Model for General Robot Control"* (Physical Intelligence).
Đây là bước chuyển kiến trúc quan trọng nhất đối với GR00T: thay vì sinh hành động autoregressive từng token rời rạc một (chậm, giới hạn độ mượt, không tự nhiên cho tín hiệu điều khiển liên tục), π₀ dùng **flow matching** — xây trên một VLM pretrain sẵn để thừa hưởng tri thức internet-scale, nhưng phần sinh hành động là một mô-đun flow-matching riêng, sinh ra **toàn bộ action chunk liên tục** thay vì từng token. Đây chính xác là hướng mà GR00T N1 (System 1 là flow-matching action transformer, xem mục 2) và N1.7 tiếp tục đi theo — không phải trùng hợp, mà là cùng một trào lưu kiến trúc VLA thế hệ mới: **VLM (tri thức internet-scale) + action head chuyên biệt sinh hành động liên tục (flow matching/diffusion)** thay vì một transformer autoregressive duy nhất làm tất cả.

---

## 2. Kiến trúc GR00T N1 chi tiết

Nguồn chính: **GR00T N1 — An Open Foundation Model for Generalist Humanoid Robots**, [arXiv:2503.14734](https://arxiv.org/abs/2503.14734) (đọc trực tiếp bản HTML đầy đủ, không chỉ abstract).

### 2.1. Tổng quan: kiến trúc dual-system

GR00T N1 có **"a dual-system architecture"** gồm hai module:

- **System 2** — module suy luận thị giác-ngôn ngữ ("interprets the environment through vision and language instructions").
- **System 1** — module diffusion transformer ("generates fluid motor actions in real time").

Nguyên văn paper: hai hệ thống này **"are tightly coupled and jointly trained end-to-end"** — nghĩa là không phải hai model huấn luyện riêng rồi ghép lại (pipeline rời rạc), mà được huấn luyện cùng nhau, có gradient chảy qua cả hai, và có cơ chế cross-attention nối chúng (xem 2.3).

**Nguồn gốc cái tên "System 1 / System 2":** thuật ngữ này mượn trực tiếp từ tâm lý học nhận thức — **Daniel Kahneman, "Thinking, Fast and Slow" (2011)** — trong đó Kahneman mô tả hai "hệ thống" tư duy của con người: System 1 là tư duy nhanh, trực giác, tự động (ví dụ nhận ra khuôn mặt quen); System 2 là tư duy chậm, có ý thức, tốn công sức (ví dụ giải một phép tính khó). **Đây chỉ là một ẩn dụ đặt tên**, không phải tuyên bố rằng GR00T N1 dùng cùng cơ chế tính toán với não người — System 1/System 2 trong GR00T N1 là hai mạng neural network hoàn toàn nhân tạo (một VLM và một diffusion/flow-matching transformer), không có quan hệ cơ chế nào với tâm lý học nhận thức. Việc mượn tên chỉ nhằm truyền đạt trực quan ý tưởng "một phần suy nghĩ chậm/ngữ nghĩa, một phần phản xạ nhanh/vận động" cho người đọc.

### 2.2. System 2: VLM suy luận ngữ nghĩa, tần số thấp

- **Backbone:** paper nêu rõ — *"For encoding vision and language inputs, GR00T N1 uses the **Eagle-2** (Li et al., 2025) vision-language model (VLM) pretrained on Internet-scale data."* Tức là System 2 không phải một VLM huấn luyện từ đầu riêng cho robot, mà là một VLM đã pretrain sẵn trên dữ liệu internet-scale (ảnh+text từ web) — cùng triết lý với RT-2 (mục 1.2): tận dụng tri thức ngữ nghĩa rộng đã có sẵn, không phải học lại từ đầu chỉ bằng dữ liệu robot (vốn khan hiếm và đắt).
- **Tần số hoạt động:** *"The System 2 reasoning module is a pre-trained Vision-Language Model (VLM) that runs at **10Hz** on an NVIDIA L40 GPU."*
- **Vai trò:** đọc ảnh camera + câu lệnh ngôn ngữ, sinh ra biểu diễn ngữ nghĩa (token embedding) làm điều kiện (condition) cho System 1 — tức System 2 không trực tiếp sinh hành động, nó "hiểu tình huống và ý định" rồi truyền thông tin đó xuống cho System 1.

### 2.3. System 1: flow-matching action transformer, tần số cao, real-time

- **Loại kiến trúc:** một Diffusion Transformer (DiT) — nhưng **mục tiêu huấn luyện cụ thể là flow matching**, không phải denoising diffusion cổ điển. Đây là điểm dễ nhầm lẫn: tên "diffusion transformer"/DiT bắt nguồn từ dòng kiến trúc ban đầu áp dụng trong các mô hình sinh ảnh bằng diffusion, nhưng GR00T N1 dùng kiến trúc DiT đó để học theo mục tiêu flow matching (xem giải thích flow matching ở mục 3.3) — nhanh hơn diffusion nhiều bước cổ điển vì không cần hàng trăm bước khử nhiễu tuần tự.
- **Tần số hoạt động:** *"It generates closed-loop motor actions at a higher frequency (**120Hz**)."* — nhanh hơn System 2 gấp 12 lần.
- **Action chunk:** paper định nghĩa hành động tại thời điểm t là một chuỗi $A_t = [a_t, a_{t+1}, \dots, a_{t+H-1}]$ gồm các vector hành động từ timestep t tới t+H-1, với **H = 16** trong implementation công bố ("We set H=16 in our implementation"). Tức là mỗi lần chạy, System 1 không sinh một hành động đơn lẻ mà sinh cả một **đoạn (chunk) 16 bước hành động liên tục cùng lúc** — giúp hành động mượt hơn và giảm số lần phải gọi lại toàn bộ pipeline.
- **Cơ chế nối với System 2 (cross-attention):** paper mô tả — action chunk bị làm nhiễu (noised action chunk) được xử lý qua các khối self-attention (giữa các action token noised với nhau) xen kẽ với các khối cross-attention cho phép "conditioning on the vision-language token embeddings $\phi_t$ output by VLM" — tức là quá trình sinh hành động của System 1 liên tục "nhìn lại" (cross-attend) biểu diễn ngữ nghĩa do System 2 tạo ra, giữ cho hành động luôn bám sát ý định ngữ nghĩa hiện tại thay vì trôi tự do.
- **Tham số:** checkpoint công khai **GR00T-N1-2B có 2.2B tham số tổng cộng, trong đó 1.34B thuộc VLM (System 2)** — nghĩa là phần System 1 (action head) chỉ chiếm khoảng 0.86B tham số, nhỏ hơn nhiều so với VLM — hợp lý vì nhiệm vụ của nó (sinh vector hành động liên tục, chiều thấp) đơn giản hơn về mặt không gian đầu ra so với việc hiểu ảnh + ngôn ngữ mở.

### 2.4. Vì sao tách hai tần số lại hợp lý cho robot thật

Đây là lý do thiết kế cốt lõi, không chỉ là chi tiết kỹ thuật:

1. **Suy luận ngữ nghĩa biến đổi chậm.** Câu hỏi "tôi đang làm nhiệm vụ gì, vật thể mục tiêu ở đâu, bước tiếp theo là gì" thường không đổi trong khoảng thời gian ngắn (vài trăm mili-giây tới vài giây) — không cần cập nhật lại mỗi mili-giây.
2. **Điều khiển motor/giữ thăng bằng cần cập nhật rất nhanh.** Với robot hình người (humanoid), giữ thăng bằng và thực hiện chuyển động tiếp xúc (contact-rich) đòi hỏi vòng lặp điều khiển tần số cao (hàng chục tới hàng trăm Hz) — chạy chậm hơn dễ gây giật cục, mất ổn định, hoặc robot ngã.
3. **Ràng buộc tính toán.** Một VLM lớn (System 2, chứa phần lớn tham số — 1.34B/2.2B) không thể chạy ở 120Hz với phần cứng hiện tại; ngược lại một action head nhỏ, chuyên biệt (System 1) có thể chạy nhanh vì không gian đầu ra của nó hẹp hơn nhiều (chỉ là vector hành động chiều thấp, không phải sinh ngôn ngữ tự do).

Vì vậy việc tách "suy nghĩ cái gì" (chậm, ít tài nguyên tính toán hơn tương đối) khỏi "làm như thế nào ở mức motor" (nhanh, real-time) không phải là một lựa chọn tùy tiện mà phản ánh đúng đặc tính vật lý khác nhau của hai loại quyết định — đây cũng chính là lý do ẩn dụ Kahneman "nghe hợp lý" dù chỉ là tên gọi mượn, không phải cùng cơ chế.

### 2.5. Dữ liệu huấn luyện

Paper nêu: GR00T N1 được huấn luyện với **"a heterogeneous mixture of real-robot trajectories, human videos, and synthetically generated datasets"** — ba nguồn dữ liệu khác loại: quỹ đạo robot thật (ít, đắt), video con người (nhiều, rẻ hơn nhưng phải suy luận gián tiếp sang robot), và dữ liệu tổng hợp (sinh bằng simulation, không giới hạn số lượng nhưng có domain gap với thực tế). Việc phối trộn ba nguồn này là chủ đề mà N1.7 đẩy mạnh hơn nữa qua EgoScale (mục 3).

---

## 3. N1.5 → N1.6 → N1.7: những gì thay đổi

### 3.1. N1.5 và N1.6

Tại thời điểm viết tài liệu này, **N1.5 và N1.6 chưa có paper kỹ thuật riêng** — thông tin chính thức duy nhất là changelog/model card trên HuggingFace (`nvidia/GR00T-N1.5-*`, `nvidia/GR00T-N1.6-*`) và tài liệu repo `GR00T-WholeBodyControl`. Từ các nguồn thứ cấp tìm được khi soạn tài liệu này: N1.5 giữ cùng công thức kiến trúc với N1 gốc — VLM (Eagle-2) trích đặc trưng thị giác-ngôn ngữ làm điều kiện, DiT làm action head huấn luyện theo flow matching — với các cải tiến chủ yếu về dữ liệu/chất lượng huấn luyện hơn là thay đổi kiến trúc nền tảng. N1.6 được mô tả (nguồn thứ cấp) là vẫn dựa trên "few thousand hours of robot teleoperation data" — tức là trước N1.7, dữ liệu huấn luyện quy mô lớn vẫn chủ yếu là teleoperation robot thật, chưa tận dụng video con người ở quy mô lớn.

> ⚠️ **Cần xác minh thêm:** các chi tiết cụ thể về N1.5/N1.6 (con số dữ liệu chính xác, thay đổi kiến trúc nếu có) ở trên dựa trên mô tả tổng quan từ nguồn thứ cấp (bài viết cộng đồng, blog N1.7 nhắc lại), chưa được kiểm tra trực tiếp trong changelog/model card chính thức. Trước khi trích dẫn số liệu N1.5/N1.6 cụ thể ở nơi khác, nên đọc trực tiếp model card HuggingFace tương ứng.

### 3.2. N1.7: EgoScale và scaling law cho dexterity

Nguồn: [HF blog N1.7](https://huggingface.co/blog/nvidia/gr00t-n1-7) (nguồn chính thức, không có paper riêng tại thời điểm soạn tài liệu).

**EgoScale** là tên tập dữ liệu pretraining mới của N1.7: **"20,854 hours of human egocentric video spanning 20+ task categories, from manufacturing and retail to healthcare and home environments."** Điểm mấu chốt: đây là **video quay từ góc nhìn thứ nhất của con người** (ego camera/wrist camera/hand tracking) khi họ thực hiện các tác vụ thật ngoài đời (sản xuất, bán lẻ, y tế, gia đình...), **không phải dữ liệu teleoperation robot**. So với N1.6 (chỉ "few thousand hours" dữ liệu teleoperation robot), đây là bước nhảy vọt về quy mô (20,854 giờ) và về loại nguồn dữ liệu (video người, không phải robot).

**"Scaling law đầu tiên cho dexterity" nghĩa là gì cụ thể?** Theo blog: *"we discovered the first-ever scaling law for robot dexterity. More human egocentric data produces predictable, consistent improvements in dexterous manipulation capability — going from 1k to 20k hours more than doubles average task completion."* Diễn giải: nhóm nghiên cứu quan sát được một **quan hệ có thể dự đoán (predictable)** giữa lượng video egocentric con người dùng để pretrain và mức độ khéo léo (dexterity) của robot khi thực thi tác vụ thao tác tinh vi — cụ thể, tăng dữ liệu từ 1.000 lên 20.000 giờ giúp **tỷ lệ hoàn thành tác vụ trung bình tăng hơn gấp đôi**. Điểm quan trọng nhất: cải thiện này đến từ **dữ liệu video người, không cần thêm dữ liệu robot thật** — nghĩa là có thể tiếp tục cải thiện độ khéo léo của robot bằng cách quay thêm video người làm việc (rẻ, dễ mở rộng), thay vì phải thu thập thêm demo teleoperation robot (đắt, chậm, giới hạn về số robot/nhân lực). Đây là lý do phát hiện này được gọi là "scaling law" — tương tự tinh thần các scaling law đã biết trong LLM (loss giảm có quy luật theo lượng dữ liệu/tham số/compute), nhưng lần đầu áp dụng và quan sát được cho độ khéo léo tay robot (dexterity), dùng nguồn dữ liệu ngoài robot.

**Kiến trúc N1.7 — "flow-matching action transformer":**

- **VLM backbone đổi:** từ Eagle-2 (N1 gốc) sang **Cosmos-Reason2-2B** — theo nguồn thứ cấp: *"System 2 (Reasoning): Cosmos-Reason2-2B backbone processes image tokens and language instructions to produce high-level action tokens."*
- **Action head vẫn là kiến trúc DiT huấn luyện theo flow matching**, kế thừa trực tiếp công thức của N1 gốc (mục 2.3) — VLM trích đặc trưng thị giác-ngôn ngữ làm điều kiện, DiT nhận điều kiện đó cộng với trạng thái robot hiện tại (proprioception) rồi sinh action chunk liên tục qua quá trình tích phân trường vận tốc (xem giải thích flow matching bên dưới), thay vì sinh token hành động rời rạc từng bước.
- **Số layer của DiT:** các nguồn thứ cấp khác nhau đưa ra số khác nhau (32-layer hoặc giảm còn 16-layer ở N1.7). **Cần xác minh thêm** — chưa kiểm tra được trực tiếp từ blog/model card chính thức nên không khẳng định con số cụ thể ở đây; khi cần con số chính xác, nên tra trực tiếp `config.json` của checkpoint trên HuggingFace (`nvidia/GR00T-N1.7-3B` hoặc `nvidia/GR00T-H-N1.7`).
- **Embodiment-conditioned MLP adapter:** một số nguồn thứ cấp (ví dụ DeepWiki cộng đồng) mô tả N1.7 bổ sung "lightweight embodiment-conditioned MLP adapters at both the input and output interfaces of the DiT action module" để hỗ trợ nhiều loại robot (embodiment) khác nhau dùng chung một action head. **Cần xác minh thêm** — đây là nguồn thứ cấp, chưa được xác nhận trực tiếp từ blog chính thức NVIDIA/HuggingFace hoặc từ code repo.

### 3.3. Flow matching là gì (giải thích khái niệm)

Nguồn gốc toán học: **Lipman et al. (2023), "Flow Matching for Generative Modeling"**, [arXiv:2210.02747](https://arxiv.org/abs/2210.02747). Theo abstract của paper: flow matching là *"a simulation-free approach for training Continuous Normalizing Flows (CNFs) based on regressing vector fields of fixed conditional probability paths."*

Diễn giải ở mức khái niệm (không cần hiểu công thức toán để nắm ý chính):

- Hãy tưởng tượng bạn muốn biến một đám mây điểm phân bố ngẫu nhiên hoàn toàn (nhiễu — noise) thành một đám mây điểm phân bố theo đúng "hình dạng" của dữ liệu thật (ở đây là phân phối các hành động hợp lý, có điều kiện theo ảnh + ngôn ngữ + trạng thái robot).
- Flow matching học một **trường vận tốc (vector field)** — tại mỗi điểm trong không gian và mỗi "thời điểm" trung gian giữa nhiễu và dữ liệu thật, mạng neural dự đoán "điểm này nên di chuyển theo hướng nào, tốc độ bao nhiêu" để dần dần biến đổi từ nhiễu thành mẫu hợp lệ.
- Khi sinh hành động thực tế (inference), ta bắt đầu từ một điểm nhiễu ngẫu nhiên, rồi **tích phân trường vận tốc đó theo một phương trình vi phân thường (ODE)** — dùng một bộ giải ODE số học có sẵn ("off-the-shelf numerical ODE solvers", theo paper) — để "trôi" dần điểm đó tới một mẫu hành động hợp lệ.
- **Khác biệt với diffusion cổ điển (denoising diffusion):** diffusion cổ điển cũng biến nhiễu thành dữ liệu, nhưng thường học quá trình đó qua rất nhiều bước khử nhiễu nhỏ, tuần tự, mô phỏng một quá trình ngẫu nhiên (stochastic). Flow matching dùng đường đi (path) hiệu quả hơn — paper nêu: dùng "Optimal Transport displacement interpolation" cho ra các đường "more efficient than diffusion paths, provide faster training and sampling" — về bản chất là chọn được đường đi "thẳng" hơn từ nhiễu tới dữ liệu, nên bộ giải ODE cần **ít bước hơn** để đạt cùng chất lượng, tức là **sinh mẫu (ở đây là sinh action chunk) nhanh hơn**. Đây chính xác là lý do các VLA thế hệ mới (π₀, GR00T N1/N1.7) chọn flow matching thay vì autoregressive token hay diffusion nhiều bước cổ điển: cần tốc độ inference đủ nhanh để chạy real-time trên robot thật (120Hz như System 1 của GR00T N1, mục 2.3).

---

## 4. Tích hợp VLA + SONIC qua unified token space

Nguồn: **SONIC**, [arXiv:2511.07820](https://arxiv.org/abs/2511.07820).

### 4.1. SONIC là gì (tóm tắt quy mô, để có bối cảnh)

Theo abstract: SONIC là một foundation model cho whole-body control của humanoid, được scale theo ba chiều — kích thước mạng (từ 1.2M tới 42M tham số), khối lượng dữ liệu (100M+ frames, từ 700 giờ motion capture), và compute (21k GPU-hours). Đây là chủ đề chính của `../01-whole-body-control/` — ở đây chỉ nhắc lại phần liên quan trực tiếp tới VLA.

### 4.2. Unified token space — SONIC nhận lệnh từ VR teleop và từ VLA theo cùng một cách

Điểm quan trọng nhất cho mục này, theo abstract của SONIC: paper xây dựng *"một unified token space hỗ trợ VR teleoperation và vision-language-action (VLA) models với một single policy duy nhất."* Diễn giải ý nghĩa:

- Thông thường, một hệ thống điều khiển robot có thể cần **hai policy khác nhau**: một policy nhận lệnh từ người vận hành qua VR teleoperation (ví dụ: vị trí tay/đầu người vận hành đọc từ headset VR, ánh xạ sang chuyển động robot), và một policy khác nhận lệnh "cấp cao" từ một model AI tự động.
- SONIC gộp cả hai vào **cùng một không gian token** — nghĩa là dù lệnh đến từ đâu (bàn tay người vận hành qua VR, hay token hành động do GR00T N1.x sinh ra), SONIC nhận vào **cùng một định dạng token**, xử lý bằng **cùng một policy** (single policy) để sinh ra góc khớp/lệnh motor cấp thấp.
- Kết quả trực tiếp: N1.x (VLA) có thể "nói chuyện" với SONIC (WBC) mà không cần một tầng chuyển đổi/thích ứng (adapter) riêng biệt giữa hai hệ thống — token hành động cấp cao do System 1 của GR00T N1.x sinh ra được SONIC diễn giải **giống hệt cách nó diễn giải lệnh từ VR teleop**.
- Điều này cho phép chế độ vận hành gọi là **"autonomous VLA-driven whole-body loco-manipulation"** — paper nêu rõ đây là ứng dụng "yêu cầu coordinated hand and foot placement" (phối hợp đặt tay và đặt chân) — tức là robot tự chủ hoàn toàn (không có người điều khiển VR) vừa di chuyển (locomotion) vừa thao tác (manipulation) cùng lúc, dưới sự điều phối của N1.x làm "não cấp cao" và SONIC làm "tủy sống/phản xạ vận động cấp thấp".

> ⚠️ **Cần xác minh thêm:** cơ chế token hóa cụ thể (định dạng token chính xác, cách N1.x mã hóa hành động cấp cao thành token tương thích với SONIC, kiến trúc phần giao tiếp giữa hai model) chỉ được nắm ở mức abstract trong lần đọc này — chưa đọc phần methodology/implementation chi tiết của paper SONIC. Khi cần giải thích chính xác cơ chế token hóa (ví dụ để tự cài đặt lại), bắt buộc đọc phần method đầy đủ của paper, không chỉ abstract.

### 4.3. Liên hệ với Gato: ý tưởng tổng quát hơn về tokenize đa phương thức

**Gato** — Reed et al. (DeepMind, 2022), [arXiv:2205.06175](https://arxiv.org/abs/2205.06175) — là nền tảng ý tưởng tổng quát hơn cho cách tiếp cận "mọi thứ đều là token" mà SONIC áp dụng ở quy mô hẹp hơn (giữa VR teleop và VLA). Gato là một "generalist agent": một transformer duy nhất, tokenize **mọi loại dữ liệu** (ảnh dưới dạng patch, văn bản, và cả hành động/trạng thái rời rạc hóa) thành **một chuỗi token chung**, rồi dùng cùng một tập trọng số để giải hàng trăm tác vụ khác nhau (chơi Atari, chú thích ảnh, điều khiển tay máy thật xếp khối...). Ý tưởng cốt lõi mà SONIC kế thừa ở quy mô hẹp hơn: **khi các nguồn tín hiệu điều khiển khác nhau (người vận hành qua VR, model AI tự động) đều được biểu diễn trong cùng một không gian token, một policy duy nhất có thể xử lý tất cả** — không cần thiết kế riêng một pipeline cho mỗi loại nguồn lệnh.

---

## 5. So sánh GR00T với các VLA khác

| | RT-2 | OpenVLA | π₀ | GR00T N1 / N1.7 |
|---|---|---|---|---|
| arXiv | [2307.15818](https://arxiv.org/abs/2307.15818) | [2406.09246](https://arxiv.org/abs/2406.09246) | [2410.24164](https://arxiv.org/abs/2410.24164) | [2503.14734](https://arxiv.org/abs/2503.14734) (N1); [HF blog](https://huggingface.co/blog/nvidia/gr00t-n1-7) (N1.7) |
| Đối tượng điều khiển | Tay máy cố định (fixed-base manipulator) | Tay máy cố định, nhiều embodiment | Tay máy cố định, nhiều embodiment (7 nền tảng robot) | **Humanoid whole-body** — cả di chuyển (locomotion) lẫn thao tác (manipulation), toàn thân |
| Cách sinh hành động | Token hành động rời rạc, autoregressive (hành động = một "ngôn ngữ" thêm vào VLM) | Token hành động rời rạc, autoregressive | **Flow matching** — sinh action chunk liên tục | **Flow matching** (DiT action head) — sinh action chunk liên tục (H=16 ở N1) |
| Tận dụng tri thức internet-scale | Có (co-fine-tune VLM lớn) | Có (Llama 2 + DINOv2/SigLIP pretrained) | Có (VLM pretrained làm nền) | Có (System 2 = Eagle-2 ở N1, Cosmos-Reason2-2B ở N1.7 — đều pretrain internet-scale) |
| Tách suy luận chậm / điều khiển nhanh (dual-frequency) | Không tách rõ (một transformer autoregressive) | Không tách rõ | Có phần tách (VLM + flow-matching head), nhưng paper không nhấn mạnh khung "dual-system" hai tần số như GR00T | **Có, rõ ràng** — System 2 (10Hz) / System 1 (120Hz), cross-attention nối hai hệ |
| Nguồn dữ liệu pretraining đặc thù | Dữ liệu web + robot | 970k demo robot thật | Dữ liệu nhiều nền tảng robot (7 platform, 68 task) | N1: real-robot + human video + synthetic; **N1.7: EgoScale — 20,854 giờ video egocentric người**, scaling law cho dexterity |
| Khác biệt cốt lõi so với GR00T | VLA cho tay máy cố định, không có whole-body/locomotion | Mã nguồn mở, kiến trúc đơn giản hơn để học nền tảng trước khi vào GR00T | Cùng dùng flow matching cho action head — kiến trúc gần GR00T nhất về mặt "cách sinh hành động", nhưng không có khung whole-body humanoid và không tích hợp trực tiếp với một hệ WBC như SONIC | Duy nhất trong bảng có: (1) whole-body humanoid, (2) khung dual-system hai tần số tường minh, (3) tích hợp trực tiếp với một hệ điều khiển toàn thân riêng biệt (SONIC) qua unified token space |

**Điểm khác biệt cốt lõi, tóm gọn một câu:** các VLA phi-humanoid (RT-2, OpenVLA, π₀) đều nhắm tới điều khiển **một tay máy cố định** (đế robot không di chuyển, chỉ tay thao tác) — bài toán điều khiển vì vậy chỉ cần sinh hành động cho vài bậc tự do (DoF) của tay; GR00T N1/N1.7 phải giải bài toán khó hơn hẳn: sinh hành động phối hợp cho **toàn bộ cơ thể humanoid** (chân để đi lại/giữ thăng bằng + tay để thao tác, hàng chục DoF cùng lúc), và vì lý do đó cần một hệ điều khiển toàn thân chuyên biệt (SONIC) ở tầng dưới để chuyển "ý định cấp cao" (token hành động từ System 1) thành chuyển động khớp ổn định, thay vì tự VLA sinh trực tiếp góc khớp cho toàn thân.

---

## Bảng thuật ngữ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| **VLM (Vision-Language Model)** | Model nhận ảnh+text, sinh ra text (mô tả, trả lời câu hỏi). Không sinh hành động vật lý. |
| **VLA (Vision-Language-Action Model)** | Model nhận ảnh+text+proprioception, sinh ra hành động liên tục (action chunk) để điều khiển robot. |
| **Proprioception** | Trạng thái nội tại của robot tại thời điểm hiện tại — góc khớp, vận tốc khớp, lực tiếp xúc... — tương tự "cảm giác cơ thể" ở người. |
| **Action chunk** | Một đoạn nhiều bước hành động liên tục được sinh ra cùng lúc (ví dụ H=16 bước ở GR00T N1), thay vì sinh từng hành động đơn lẻ một. |
| **Dual-system architecture** | Kiến trúc tách hai module: một module suy luận ngữ nghĩa tần số thấp (System 2/VLM) và một module sinh hành động tần số cao (System 1/action head), có liên kết (cross-attention) với nhau. |
| **System 1 / System 2** | Tên mượn từ Kahneman, "Thinking, Fast and Slow" (2011) — chỉ là ẩn dụ đặt tên cho hai module có tần số/vai trò khác nhau, không phải tuyên bố cùng cơ chế với nhận thức con người. |
| **DiT (Diffusion Transformer)** | Kiến trúc transformer vốn dùng phổ biến trong các mô hình sinh ảnh bằng diffusion; ở GR00T được dùng làm action head, huấn luyện theo mục tiêu flow matching. |
| **Diffusion (denoising diffusion)** | Lớp phương pháp sinh dữ liệu bằng cách học đảo ngược một quá trình thêm nhiễu dần, qua nhiều bước khử nhiễu tuần tự. |
| **Flow matching** | Phương pháp sinh dữ liệu bằng cách học một trường vận tốc (vector field) biến đổi liên tục từ phân phối nhiễu sang phân phối dữ liệu mục tiêu, rồi tích phân bằng bộ giải ODE — thường cần ít bước hơn diffusion cổ điển nên sinh mẫu nhanh hơn. Nguồn: Lipman et al. (2023), [arXiv:2210.02747](https://arxiv.org/abs/2210.02747). |
| **Vector field / trường vận tốc** | Hàm gán cho mỗi điểm (và mỗi thời điểm trung gian) một hướng+tốc độ di chuyển — là đối tượng mà flow matching học để "dẫn đường" từ nhiễu tới mẫu hợp lệ. |
| **ODE solver (bộ giải phương trình vi phân thường)** | Thuật toán số học dùng để tích phân trường vận tốc đã học, từ điểm nhiễu ban đầu ra mẫu cuối cùng, trong quá trình inference của flow matching. |
| **EgoScale** | Tập dữ liệu pretraining của GR00T N1.7 — 20.854 giờ video egocentric (góc nhìn thứ nhất) của con người, trải hơn 20 loại tác vụ. |
| **Egocentric video** | Video quay từ góc nhìn thứ nhất (ví dụ camera gắn trên đầu/cổ tay người), khác với video quan sát từ bên ngoài (third-person). |
| **Scaling law** | Quan hệ có thể dự đoán giữa một đại lượng đầu vào (ở đây: lượng dữ liệu video người) và một chỉ số hiệu năng đầu ra (ở đây: tỷ lệ hoàn thành tác vụ đòi hỏi độ khéo léo). |
| **Dexterity (độ khéo léo)** | Khả năng thực hiện các thao tác tinh vi, đòi hỏi phối hợp nhiều bậc tự do của tay (contact-rich manipulation). |
| **Unified token space** | Một không gian biểu diễn token chung, cho phép một policy duy nhất (ở đây: SONIC) xử lý lệnh đến từ nhiều nguồn khác nhau (VR teleoperation, VLA) theo cùng một cách. |
| **Loco-manipulation** | Thực hiện đồng thời di chuyển toàn thân (locomotion) và thao tác vật thể (manipulation) — bài toán đặc thù của whole-body humanoid, không xuất hiện ở VLA tay máy cố định. |
| **Embodiment** | Loại "cơ thể" vật lý cụ thể của robot (số khớp, hình dạng, loại tay/chân...) — một model "đa embodiment" có thể điều khiển nhiều loại robot khác nhau. |
| **Co-fine-tune** | Kỹ thuật fine-tune một model đồng thời trên nhiều nguồn dữ liệu khác loại (ví dụ dữ liệu web gốc + dữ liệu robot) thay vì chỉ fine-tune trên dữ liệu mới, nhằm giữ lại tri thức đã học từ nguồn pretrain gốc. |

---

**Xem thêm:** `README.md` trong cùng thư mục cho bảng tài liệu gốc, video, và lộ trình học theo từng bước.
