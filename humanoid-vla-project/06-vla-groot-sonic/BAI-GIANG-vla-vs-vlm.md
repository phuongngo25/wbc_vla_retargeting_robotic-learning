# Bài giảng: VLA là gì, khác VLM thông thường ở đâu

*(Thuộc mảng: Vision-Language-Action: GR00T N1 → N1.7 & SONIC)*

## 🎯 Mục tiêu bài học

- Định nghĩa chính xác được VLM và VLA, chỉ ra được input/output khác nhau ở đâu (không chỉ liệt kê "thêm một input").
- Giải thích được vì sao đầu ra của VLM (token rời rạc) về bản chất khác đầu ra của VLA (số thực liên tục), và vì sao khác biệt đó buộc VLA phải có kiến trúc riêng cho phần sinh hành động.
- Phân biệt được khái niệm **proprioception** (trạng thái khớp/lực nội tại của robot) với các loại input khác (ảnh, ngôn ngữ), và giải thích được vì sao không thể coi proprioception "chỉ là một loại ảnh khác".
- Giải thích được khái niệm **action chunk** ở mức định nghĩa (không đi sâu cách sinh ra nó — phần đó thuộc bài giảng riêng về flow matching/diffusion).
- Áp dụng được khung phân biệt VLM/VLA vào chính kiến trúc GR00T N1 (System 2 vs. toàn bộ hệ thống) mà không nhầm lẫn "một VLM bên trong" với "cả hệ thống là VLA".
- Nhận diện được ít nhất 2 hiểu nhầm phổ biến khi mới học VLA và biết cách sửa lại tư duy.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Đây là khái niệm nền móng của toàn bộ mảng "Vision-Language-Action: GR00T N1 → N1.7 & SONIC" — mọi bài giảng khác trong thư mục này (RT-1/RT-2, flow matching, kiến trúc dual-system của GR00T, unified token space của SONIC...) đều giả định bạn đã hiểu rõ: VLA không phải là "VLM cộng thêm một chút", mà là một lớp bài toán có ràng buộc vật lý hoàn toàn khác. Nếu hiểu sai ở tầng này, mọi thứ học sau — vì sao cần action head riêng, vì sao GR00T tách System 1/System 2 theo tần số, vì sao SONIC cần một unified token space — đều sẽ bị hiểu nông thành "chi tiết kỹ thuật tùy chọn" thay vì "hệ quả bắt buộc từ bản chất bài toán".

Trong pipeline của dự án, khái niệm này đứng ngay ở điểm nối giữa hai thế giới: thế giới của các mô hình nền tảng thị giác-ngôn ngữ (nơi GPT-4V, LLaVA, Eagle-2, Cosmos-Reason2 sống) và thế giới của điều khiển robot thời gian thực (nơi SONIC, whole-body control, và các bộ điều khiển tần số cao sống). GR00T N1/N1.7 chính là cây cầu nối hai thế giới đó, và hiểu "VLA khác VLM ở đâu" là điều kiện tiên quyết để hiểu tại sao cây cầu đó phải được xây theo đúng cách nó được xây (dual-system, hai tần số, cross-attention).

## 🧠 Trực giác

### Góc nhìn 1: Người bình luận thể thao vs. vận động viên đang thi đấu

Hãy tưởng tượng một trận bóng đá. Một **bình luận viên** (commentator) xem trận đấu qua camera, nghe câu hỏi của khán giả ("đội nào đang kiểm soát bóng tốt hơn?"), và trả lời bằng lời nói — mô tả, phân tích, dự đoán. Đó là VLM: nhận hình ảnh + ngôn ngữ, sinh ra ngôn ngữ. Bình luận viên có thể cực kỳ am hiểu chiến thuật, nhưng **không tự mình chạy ra sân sút bóng** — output của họ mãi mãi nằm trong không gian lời nói.

Ngược lại, một **cầu thủ đang thi đấu** cũng nhìn (thấy bóng, thấy đồng đội), cũng "hiểu ngôn ngữ" (nghe HLV hét chỉ đạo), nhưng còn có thêm một loại nhận thức thứ ba mà bình luận viên không có: **cảm giác cơ thể** — họ biết chân trái mình đang ở góc nào, trọng tâm đang nghiêng bên nào, cơ đùi đang căng ra sao (đây chính là ẩn dụ cho proprioception). Và quan trọng nhất: đầu ra của họ không phải là câu nói, mà là **một chuỗi chuyển động cơ liên tục, mượt, đúng thời điểm** — chậm nửa giây là mất bóng, ngắt quãng là ngã. Đó là VLA.

**Giới hạn của loại suy này:** bình luận viên và cầu thủ là hai con người tách biệt hoàn toàn; nhưng VLA thực tế không phải "hai model độc lập ghép lại" — như sẽ thấy ở phần cơ chế, kiến trúc VLA hiện đại (GR00T, π₀) có phần "hiểu" và phần "hành động" được **huấn luyện cùng nhau, có luồng thông tin hai chiều**, chứ không phải bình luận viên ném một bản báo cáo qua tường cho cầu thủ.

### Góc nhìn 2: Đầu ra rời rạc (menu có sẵn) vs. đầu ra liên tục (núm xoay vô cấp)

Một góc nhìn thuần kỹ thuật hơn: hãy nghĩ VLM như một cái máy chỉ có thể trả lời bằng cách **chọn một món trong menu** — dù menu đó rất dài (hàng chục nghìn "món" là các token trong từ điển), tại mỗi bước nó vẫn đang **chọn 1 trong N lựa chọn rời rạc cố định sẵn**. Đây là lý do VLM luôn có thể được mô tả bằng một phân phối xác suất trên một tập hữu hạn.

VLA thì giống một cái **núm xoay vô cấp** (continuous dial) — không có "menu", chỉ có một dải giá trị liên tục (ví dụ góc khớp từ -180° đến 180°, với độ chính xác tùy ý). Việc "chọn" ở đây không phải là chọn 1 trong N lựa chọn có sẵn, mà là xác định một điểm trong không gian liên tục vô hạn chiều giá trị. Đây là lý do các công cụ toán học phù hợp cho hai bài toán này khác hẳn nhau: VLM dùng softmax + cross-entropy trên tập hữu hạn; VLA (thế hệ hiện đại) dùng các công cụ sinh phân phối liên tục như diffusion/flow matching.

**Giới hạn của loại suy này:** loại suy "núm xoay" gợi ý sai rằng VLA chỉ sinh **một** giá trị liên tục tại một thời điểm — thực tế VLA sinh ra cả một **action chunk**, tức nhiều "núm xoay" cho H bước thời gian tương lai cùng lúc, không phải xoay từng núm một cách tuần tự độc lập. Chi tiết cơ chế sinh action chunk (flow matching/diffusion) không nằm trong phạm vi bài này — xem bài giảng riêng.

### Góc nhìn 3: Google Maps đọc chỉ đường vs. cruise control tự lái giữ làn

Một góc nhìn thứ ba, gần với trải nghiệm hằng ngày hơn: hãy so sánh ứng dụng bản đồ đọc chỉ đường bằng giọng nói ("còn 200 mét nữa thì rẽ trái") với hệ thống **adaptive cruise control + lane-keeping assist** trên ô tô hiện đại. Ứng dụng bản đồ quan sát vị trí bạn (tương đương "vision" — dữ liệu GPS/bản đồ) và sinh ra một câu nói rời rạc, đúng lúc, nhưng **bạn** (người lái) mới là người biến câu nói đó thành động tác xoay vô-lăng cụ thể. Đây là VLM: quan sát rồi mô tả bằng ngôn ngữ, để một "tác nhân khác" (con người) thực thi.

Ngược lại, cruise control giữ làn không nói gì cả — nó liên tục đọc camera (vision) + tốc độ giới hạn từ biển báo (một dạng "ngôn ngữ" đã được nhận dạng), đồng thời **luôn biết góc vô-lăng hiện tại, tốc độ bánh xe hiện tại** (đây là proprioception của ô tô), rồi liên tục xuất ra **góc đánh lái, mức ga/phanh** — những con số liên tục, cập nhật hàng chục lần mỗi giây, đi thẳng xuống động cơ servo của vô-lăng. Không có bước "đọc thành lời rồi chờ ai đó thực thi" — đầu ra đã là hành động vật lý trực tiếp. Đây là VLA.

**Giới hạn của loại suy này:** cruise control thực tế là một hệ điều khiển cổ điển (control theory kinh điển, PID/MPC), không phải một mạng neural học từ dữ liệu quy mô lớn theo kiểu VLA hiện đại — loại suy này chỉ giúp làm rõ **khác biệt về loại đầu ra** (lời nói vs. tín hiệu điều khiển liên tục), không nên hiểu là "VLA về bản chất kỹ thuật giống hệt cruise control".

## 📐 Định nghĩa chính xác

**Vision-Language Model (VLM)** là một mô hình nhận **(ảnh, text)** làm input và sinh ra **text** làm output, trong đó output nằm trong không gian ngôn ngữ: một chuỗi token rời rạc lấy từ một từ điển hữu hạn kích thước cố định, được sinh theo kiểu autoregressive (dự đoán phân phối xác suất trên từ điển đó cho token tiếp theo, tại một thời điểm một token). Ví dụ: GPT-4V, LLaVA, Qwen-VL. VLM không có khái niệm về trạng thái vật lý hiện tại của một cơ thể (không có input proprioception), và output của nó không được thiết kế để điều khiển động cơ trực tiếp.

**Vision-Language-Action Model (VLA)** là một mô hình nhận **ba** loại input:

1. **Vision** — ảnh/video từ camera (giống VLM).
2. **Language** — chỉ thị ngôn ngữ, câu lệnh nhiệm vụ (giống VLM).
3. **Proprioception** — trạng thái nội tại hiện tại của robot: góc khớp, vận tốc khớp, lực/mô-men tiếp xúc đo được từ cảm biến trên chính robot. Đây là loại input **không tồn tại** trong VLM thông thường, vì VLM không gắn với một cơ thể vật lý cụ thể nào.

và sinh ra **hành động liên tục theo thời gian**, thường được gọi là **action chunk**: một chuỗi vector hành động $A_t = [a_t, a_{t+1}, \dots, a_{t+H-1}]$ cho H bước thời gian tiếp theo (ví dụ vị trí/vận tốc khớp mục tiêu, hoặc delta pose của end-effector), trong đó mỗi $a_i$ là một **vector số thực** trong không gian liên tục, không phải token rời rạc từ một từ điển hữu hạn.

Khác biệt bản chất (không chỉ là "thêm một input") nằm ở **loại phân phối xác suất mà mô hình phải học sinh ra**:

- VLM tối ưu để mô hình hóa và sinh phân phối xác suất trên **token rời rạc tiếp theo** — một bài toán phân loại (classification) tại mỗi bước, trên một tập hữu hạn đã biết trước.
- VLA phải mô hình hóa và sinh phân phối xác suất trên **không gian hành động liên tục nhiều chiều** — một bài toán hồi quy sinh (generative regression), và quan trọng hơn: chuỗi hành động sinh ra phải đủ mượt và đủ nhanh để điều khiển motor thật. Một chuỗi hành động giật cục hoặc trễ vài trăm mili-giây có thể khiến robot ngã hoặc làm rơi vật — đây là một ràng buộc **thời gian thực vật lý** mà VLM hoàn toàn không có (không ai ngã nếu chatbot trả lời chậm nửa giây).

Chính vì ràng buộc này, các kiến trúc VLA hiện đại (RT-2 là ngoại lệ một phần — xem bài giảng riêng về RT-1/RT-2) không dùng thuần autoregressive token generation cho hành động, mà bổ sung một **action head** chuyên biệt (diffusion hoặc flow matching — xem bài giảng riêng về flow matching) được thiết kế riêng để sinh vector liên tục nhanh và mượt.

## ⚙️ Cơ chế hoạt động — từng bước

Phần này chỉ mô tả **luồng dữ liệu ở mức khối** (input nào đi qua khối nào, output cuối cùng thuộc loại gì) — không đi vào chi tiết toán học bên trong action head (flow matching/diffusion, xem bài giảng riêng) hay bên trong VLM backbone (kiến trúc transformer thị giác-ngôn ngữ cụ thể, không thuộc phạm vi bài này). Sơ đồ dưới đây đối chiếu luồng xử lý của VLM thuần túy và của một VLA hiện đại (kiểu GR00T/π₀ — VLM backbone + action head riêng):

```
┌─────────────────────────── VLM (ví dụ GPT-4V, LLaVA) ───────────────────────────┐
│                                                                                    │
│   [Ảnh]  ──┐                                                                      │
│            ├──► [Vision-Language Backbone] ──► [phân phối xác suất trên          │
│   [Text] ──┘         (transformer)                từ điển N token]              │
│                                                          │                        │
│                                                          ▼                        │
│                                              chọn 1 token, lặp lại (autoregressive)│
│                                                          │                        │
│                                                          ▼                        │
│                                              "Bức ảnh cho thấy một con robot..."  │
│                                                    (OUTPUT = TEXT)                │
└────────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────── VLA (ví dụ GR00T N1, π₀) ────────────────────────────┐
│                                                                                    │
│   [Ảnh]  ──┐                                                                      │
│            ├──► [VLM Backbone: System 2] ──► φ_t (vision-language token          │
│   [Text] ──┘      (Eagle-2 / Cosmos-Reason2)    embeddings — biểu diễn ngữ nghĩa) │
│                          (chạy TẦN SỐ THẤP, vd 10Hz)                              │
│                                                          │                        │
│                                                          │ cross-attention        │
│                                                          ▼ (điều kiện hóa)        │
│  [Proprioception] ──────────────────────►  [Action Head: System 1]               │
│  (góc khớp, vận tốc, lực tiếp xúc)             (DiT, flow matching/diffusion —    │
│                                                  xem bài giảng riêng)             │
│                                                  (chạy TẦN SỐ CAO, vd 120Hz)      │
│                                                          │                        │
│                                                          ▼                        │
│                                        A_t = [a_t, a_{t+1}, ..., a_{t+H-1}]       │
│                                        (OUTPUT = ACTION CHUNK, vector liên tục)   │
└────────────────────────────────────────────────────────────────────────────────┘
```

Đọc sơ đồ theo 4 bước:

1. **Input mở rộng:** VLA nhận thêm proprioception — một luồng dữ liệu hoàn toàn khác về bản chất so với ảnh/text (số liệu cảm biến nội tại theo thời gian thực, không phải "nội dung" cần được "hiểu" theo nghĩa ngữ nghĩa).
2. **Phần "hiểu" vẫn giống VLM:** phần xử lý ảnh + ngôn ngữ trong một VLA hiện đại thực chất **là** một VLM (hoặc rất gần với một VLM) — đây chính là điểm dễ gây hiểu nhầm, sẽ nói kỹ ở mục sai lầm thường gặp.
3. **Điểm rẽ nhánh quyết định:** thay vì đưa biểu diễn ngữ nghĩa đó qua một lớp softmax để chọn token tiếp theo (như VLM), VLA đưa nó làm **điều kiện (condition)** cho một mô-đun riêng (action head) — mô-đun này nhận thêm proprioception, rồi sinh ra vector hành động liên tục.
4. **Output khác loại hoàn toàn:** output cuối cùng không phải chuỗi ký tự đọc được, mà là một mảng số thực — sẵn sàng để (sau một bước ánh xạ sang lệnh motor cụ thể) đẩy thẳng xuống bộ điều khiển của robot.
5. **Vòng lặp đóng (closed-loop), không phải một lần sinh duy nhất:** vì hai khối chạy ở hai tần số khác nhau (System 2 chậm hơn System 1 nhiều lần), trong khi System 1 đang thực thi hết một action chunk H bước, System 2 có thể đã kịp cập nhật lại $\phi_t$ ít nhất một lần dựa trên ảnh/proprioception mới nhất. Nói cách khác, VLA không phải "nhìn một lần, hành động mãi mãi" mà liên tục nhìn lại và điều chỉnh — đây là lý do tách hai tần số (chi tiết lý do thiết kế: xem `NOI-DUNG-CHI-TIET.md` mục 2.4) quan trọng hơn nhiều so với một chi tiết kỹ thuật phụ.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số tự chọn — không lấy từ một paper cụ thể, chỉ để làm rõ khác biệt về loại đầu ra.)*

**Phía VLM:** giả sử từ điển token có N = 32.000 token (kích thước từ điển tiêu biểu của một LLM cỡ vừa). Tại một bước sinh, VLM tính ra một vector logit 32.000 chiều, đưa qua softmax để có phân phối xác suất $p(\text{token}_i) $, $\sum_{i=1}^{32000} p(\text{token}_i) = 1$. Ví dụ token "robot" có xác suất 0.42, token "cánh tay" có xác suất 0.31, v.v. — mô hình lấy mẫu (hoặc lấy argmax) **một** token trong 32.000 lựa chọn rời rạc đó, rồi lặp lại cho token kế tiếp. Toàn bộ câu trả lời là một chuỗi các lựa chọn rời rạc nối tiếp nhau.

**Phía VLA:** giả sử robot có 7 bậc tự do ở một cánh tay (7 khớp), và action chunk có H = 16 bước thời gian (giống con số H=16 mà GR00T N1 dùng trong thực tế — xem `NOI-DUNG-CHI-TIET.md` mục 2.3). Tại một lần sinh, VLA không chọn 1-trong-N gì cả — nó phải xuất ra một **ma trận** kích thước 16 × 7 = 112 số thực liên tục:

```
A_t =
[ a_t      = (θ1, θ2, θ3, θ4, θ5, θ6, θ7)_t        ]   ví dụ: (12.3°, -45.1°, 0.02 rad/s, ...)
[ a_{t+1}  = (θ1, θ2, θ3, θ4, θ5, θ6, θ7)_{t+1}    ]   ví dụ: (12.5°, -44.8°, 0.03 rad/s, ...)
[ ...                                                ]
[ a_{t+15} = (θ1, θ2, θ3, θ4, θ5, θ6, θ7)_{t+15}   ]   ví dụ: (15.1°, -40.2°, 0.01 rad/s, ...)
```

Không có "menu 112 lựa chọn" nào cả — mỗi trong 112 giá trị này là một số thực trong một khoảng liên tục (ví dụ góc khớp trong [-180°, 180°]), và **112 giá trị này phải nhất quán với nhau** (khớp thứ t+1 phải là một chuyển động khả thi tiếp nối từ khớp thứ t, không phải 16 điểm độc lập ngẫu nhiên) — đây chính xác là bài toán mà một action head kiểu flow matching/diffusion được thiết kế để giải (chi tiết: xem bài giảng riêng về flow matching), khác hẳn bài toán "chọn 1 trong N" của VLM.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | VLM (Vision-Language Model) | VLA (Vision-Language-Action Model) |
|---|---|---|
| Input | Ảnh + text | Ảnh + text + **proprioception** |
| Output | Text (chuỗi token rời rạc) | **Action chunk** (chuỗi vector số thực liên tục, H bước) |
| Không gian output | Từ điển hữu hạn N token | Không gian liên tục nhiều chiều (vô hạn giá trị) |
| Bài toán học máy cốt lõi | Phân loại (classification) trên token tiếp theo | Hồi quy sinh (generative regression) trên vector hành động |
| Công cụ sinh phổ biến | Autoregressive + softmax + cross-entropy | Diffusion / flow matching (xem bài giảng riêng); một số model cũ như RT-2, OpenVLA vẫn dùng token hóa hành động + autoregressive |
| Ràng buộc thời gian thực | Không có (chậm vài trăm ms vẫn chấp nhận được với người dùng) | **Bắt buộc** — trễ/giật có thể làm robot ngã hoặc làm rơi vật |
| Gắn với cơ thể vật lý cụ thể? | Không — không "biết" trạng thái vật lý của bất kỳ ai | Có — được điều kiện hóa theo trạng thái khớp/lực hiện tại của một robot cụ thể |
| Ví dụ | GPT-4V, LLaVA, Qwen-VL | RT-1, RT-2, OpenVLA, π₀, GR00T N1/N1.7 |
| Quan hệ giữa hai loại | — | VLA hiện đại **chứa** một VLM (hoặc gần-VLM) bên trong làm phần "hiểu", cộng thêm action head sinh hành động |
| Tần số hoạt động điển hình | Không quan trọng lắm (giây tới vài giây/lượt vẫn chấp nhận được) | Thường tách hai tần số: phần "hiểu" chậm (ví dụ 10Hz), phần sinh hành động nhanh (ví dụ 120Hz) — xem ví dụ GR00T N1 bên dưới |
| Đơn vị đánh giá "đúng/sai" | Chất lượng ngôn ngữ (độ chính xác mô tả, mức độ hữu ích của câu trả lời) | Tỉ lệ hoàn thành tác vụ vật lý thật (task success rate), độ mượt/an toàn của chuyển động, không chỉ "hành động có hợp lý về mặt ngữ nghĩa không" |
| Có thể huấn luyện chỉ bằng dữ liệu internet (ảnh+text công khai)? | Có — đây chính là cách huấn luyện chuẩn của VLM | Không đủ — bắt buộc cần thêm dữ liệu gắn với robot/cơ thể vật lý (quỹ đạo robot thật, video egocentric con người, hoặc dữ liệu mô phỏng) để học phần proprioception → action, xem `NOI-DUNG-CHI-TIET.md` mục 2.5 |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **"VLA chỉ là VLM cộng thêm vài lớp ở cuối."** Đây là hiểu nhầm nguy hiểm nhất vì nó gần đúng ở cấp độ hời hợt (đúng là nhiều VLA hiện đại có một VLM y hệt bên trong, ví dụ GR00T N1 dùng Eagle-2) nhưng sai ở bản chất bài toán: phần "vài lớp ở cuối" đó không phải một MLP nhỏ chiếu logit sang không gian khác — nó là một **mô-đun sinh (generative module)** riêng biệt (diffusion/flow matching), phải giải một bài toán tối ưu hoàn toàn khác (sinh phân phối liên tục thay vì phân loại rời rạc), thường có tần số hoạt động khác, và trong các kiến trúc như GR00T N1 còn được **huấn luyện cùng lúc, có gradient chảy qua cả hai phần** (không phải "đóng băng VLM rồi gắn thêm đầu ra"). Coi nó là "vài lớp" khiến người học đánh giá thấp lý do vì sao lại cần cả một dòng nghiên cứu riêng (RT-2 → OpenVLA → π₀ → GR00T) chỉ để giải quyết phần sinh hành động.

2. **"Proprioception giống như một loại ảnh input khác, chỉ là thêm một 'view' nữa."** Sai vì hai lý do: (a) proprioception không mang thông tin không gian 2D/3D theo kiểu pixel — nó là một vector số liệu cảm biến (góc, vận tốc, lực) thay đổi rất nhanh (thường phải đọc ở tần số cao hơn nhiều so với tần số cập nhật ảnh camera); (b) quan trọng hơn, proprioception là thứ **buộc mô hình phải gắn với một cơ thể vật lý cụ thể tại đúng thời điểm hiện tại** — hai lần chạy VLA với cùng ảnh + cùng câu lệnh nhưng proprioception khác nhau (ví dụ tay robot đang ở góc khác) phải cho ra action chunk khác nhau, trong khi VLM xử lý hai ảnh giống hệt nhau sẽ luôn cho cùng một phân phối token. VLM hoàn toàn không có khái niệm "trạng thái hiện tại của một cơ thể" — nó không cần vì nó không điều khiển gì cả.

3. **(Bổ sung, thường gặp khi mới đọc GR00T)** *"GR00T N1 dùng một VLM (Eagle-2) nên bản thân Eagle-2 là một VLA."* Không đúng — Eagle-2 (System 2) chỉ là phần "hiểu", vẫn nhận (ảnh, text) và sinh ra biểu diễn ngữ nghĩa (không phải action chunk). Toàn bộ hệ thống GR00T N1 (System 2 + System 1 kết hợp) mới là VLA, vì chỉ output cuối cùng của cả hệ thống mới là action chunk. Xem mục kế tiếp để làm rõ ranh giới này.

4. **"VLA nào cũng phải dùng diffusion/flow matching, vì đó mới là 'cách đúng' để sinh hành động liên tục."** Cũng là hiểu nhầm — như bảng so sánh ở trên và như dòng lịch sử RT-2 → OpenVLA (xem bài giảng riêng) cho thấy, một số VLA có ảnh hưởng lớn (RT-2, OpenVLA) vẫn **token hóa hành động thành các token rời rạc** rồi sinh autoregressive giống VLM, chỉ khác là các token đó được ánh xạ ngược, có kiểm soát, sang giá trị điều khiển liên tục ở bước cuối. Diffusion/flow matching (π₀, GR00T N1/N1.7) là một hướng thiết kế **tốt hơn về mặt chất lượng chuyển động mượt và tốc độ inference** cho hành động nhiều chiều, liên tục, chứ không phải điều kiện bắt buộc để một mô hình được gọi là VLA. Định nghĩa VLA nằm ở loại input/output (mục Định nghĩa chính xác), không nằm ở việc chọn công cụ toán học nào để sinh ra output đó.

## 🏗️ Ví dụ minh hoạ trong dự án này

GR00T N1 là ví dụ trực tiếp và rõ ràng nhất để "neo" định nghĩa VLA/VLM vào một kiến trúc thật (chi tiết đầy đủ: `NOI-DUNG-CHI-TIET.md` mục 2):

- **System 2 của GR00T N1** — backbone **Eagle-2** (ở N1 gốc) hoặc **Cosmos-Reason2-2B** (ở N1.7) — bản thân nó **là một VLM đúng nghĩa**: nhận ảnh camera + chỉ thị ngôn ngữ, chạy ở tần số thấp (10Hz), và sinh ra token embedding ngữ nghĩa $\phi_t$. Nếu tách riêng System 2 ra và chỉ hỏi nó "mô tả bức ảnh này", nó hoạt động đúng như một VLM thông thường.
- **System 1 của GR00T N1** — một Diffusion Transformer huấn luyện theo mục tiêu flow matching, chạy ở tần số cao (120Hz), nhận proprioception + $\phi_t$ (từ System 2, qua cross-attention) làm điều kiện, sinh ra action chunk $A_t$ với H = 16 bước.
- **Toàn bộ GR00T N1 (System 2 + System 1)** mới là VLA đúng nghĩa theo định nghĩa ở mục trên: input có đủ ba loại (ảnh, ngôn ngữ, proprioception), output là action chunk liên tục. Đây chính là lý do bài giảng này nhấn mạnh ranh giới "một VLM nằm bên trong" khác với "cả hệ thống là một VLA" — nhầm hai điều này là nguồn gốc của sai lầm thường gặp số 3 ở trên.
- Việc paper GR00T N1 mô tả hai hệ thống này *"tightly coupled and jointly trained end-to-end"* (không phải hai model huấn luyện tách rời rồi ghép) chính là minh chứng cho ý ở mục sai lầm thường gặp số 1: action head không phải "vài lớp gắn thêm", mà là một phần của cùng một quá trình huấn luyện, có gradient chảy ngược qua cả VLM.
- Một con số cụ thể càng làm rõ tỉ lệ "phần VLM" so với "phần action head" trong một VLA thật: checkpoint công khai GR00T-N1-2B có tổng cộng 2.2B tham số, trong đó **1.34B tham số thuộc về VLM (System 2)** — tức là phần action head (System 1) chỉ chiếm khoảng 0.86B, nhỏ hơn hẳn. Điều này hợp lý vì không gian output của System 1 (vector hành động chiều thấp) đơn giản hơn nhiều so với không gian output "ngôn ngữ mở, ngữ nghĩa rộng" mà System 2 phải xử lý — nhưng đừng để tỉ lệ tham số nhỏ hơn đánh lừa: phần nhỏ hơn đó vẫn là phần quyết định liệu cả hệ thống có phải là VLA hay không (một VLM 1.34B tham số dù lớn tới đâu, thiếu System 1, vẫn chỉ là VLM).
- SONIC (whole-body control policy, xem `../01-whole-body-control/`) đứng ở tầng dưới System 1 — nó nhận action chunk / token hành động cấp cao từ GR00T N1.x qua một unified token space và biến thành lệnh motor cấp thấp. Việc này minh hoạ thêm một lớp phân tách nữa: ngay cả action chunk do VLA sinh ra cũng chưa phải là lệnh motor cuối cùng — đó là chủ đề của bài giảng riêng về SONIC/unified token space, không thuộc phạm vi bài này.

## 🔥 Cập nhật hiện đại / SOTA gần đây

Ba điểm dưới đây dựa trên tài liệu tìm được qua WebSearch/WebFetch ngày 17/09/2026 (đã kiểm tra trực tiếp abstract/nội dung, không suy đoán từ trí nhớ nội tại):

1. **Cách giới nghiên cứu 2025 đóng khung lại định nghĩa VLA: từ "sinh chuỗi thụ động" sang "tác nhân chủ động".** Một survey gần đây — *"Pure Vision Language Action (VLA) Models: A Comprehensive Survey"* (arXiv:2509.19012, 09/2025) — mô tả VLA như một "paradigm shift từ điều khiển dựa trên policy truyền thống sang robotics tổng quát hoá, tái định khung VLM từ những cỗ máy sinh chuỗi thụ động (passive sequence generators) thành các tác nhân chủ động (active agents) cho thao tác và ra quyết định trong môi trường phức tạp, động." Đây đúng là cách diễn đạt lại — bằng ngôn ngữ mới hơn — chính khác biệt cốt lõi mà bài giảng này trình bày: VLM "thụ động" sinh mô tả, VLA "chủ động" tạo ra hành động ảnh hưởng ngược lại thế giới vật lý. Survey này cũng phân loại các cách tiếp cận VLA hiện tại thành 5 nhóm paradigm (autoregression-based, diffusion-based, reinforcement-based, hybrid, specialized) — cho thấy "diffusion/flow-matching action head" (như GR00T dùng) chỉ là một trong vài hướng đang tồn tại song song, không phải hướng duy nhất.

2. **Vấn đề độ trễ ở ranh giới action chunk vẫn là bài toán mở đang được giải năm 2025 — xác nhận thêm rằng "ràng buộc thời gian thực" không phải chi tiết phụ.** Black, Galliker, và Levine (Physical Intelligence / UC Berkeley), trong *"Real-Time Execution of Action Chunking Flow Policies"* (arXiv:2506.07339, 2025), chỉ ra rằng dù action chunking đã giúp các VLA hiện đại (như π₀) đạt được "temporal consistency" trong các tác vụ điều khiển tần số cao, nó "không giải quyết trọn vẹn vấn đề độ trễ" — dẫn tới hiện tượng dừng khựng hoặc "chuyển động giật cục lệch phân phối huấn luyện tại ranh giới giữa các chunk" (jerky movements at chunk boundaries). Nhóm tác giả đề xuất thuật toán suy luận thời gian thực (Real-Time Chunking, RTC) áp dụng được cho "bất kỳ VLA nào dựa trên diffusion hoặc flow, không cần huấn luyện lại." Điểm đáng chú ý cho bài giảng này: đây là bằng chứng trực tiếp, từ một paper 2025, rằng bài toán "sinh hành động liên tục đủ mượt, đủ nhanh" — chính là điểm phân biệt VLA với VLM mà mục Định nghĩa nêu ra — **vẫn là một mặt trận nghiên cứu đang mở**, không phải vấn đề đã giải xong từ 2022-2023.

3. **Xu hướng tái sử dụng nguyên khối VLM lớn làm backbone VLA, kèm bộ giải mã hành động tách rời — Gemini Robotics.** Theo tài liệu *"Gemini Robotics: Bringing AI into the Physical World"* (Google DeepMind, arXiv:2503.20020, 2025) và trang Wikipedia tổng hợp tương ứng, Gemini Robotics gồm hai phần: một **VLA backbone** chạy trên cloud (bản chưng cất — distilled — từ Gemini Robotics-ER, vốn dựa trên Gemini 2.0) và một **bộ giải mã hành động cục bộ** (Gemini Robotics decoder) chạy ngay trên máy tính onboard của robot, với độ trễ truy vấn được tối ưu "từ vài giây xuống dưới 160ms". Đây là cùng một triết lý dual-system mà GR00T N1 áp dụng (VLM tần số thấp cho "hiểu", mô-đun riêng tần số cao cho "hành động"), nhưng ở quy mô một VLM nền tảng cực lớn (Gemini) — cho thấy hướng "tách biệt phần hiểu (chạy xa, chậm hơn) khỏi phần quyết định hành động (chạy gần robot, nhanh hơn)" không phải là lựa chọn riêng của NVIDIA mà là một mẫu hình kiến trúc đang được nhiều nhóm nghiên cứu lớn (Google DeepMind, Physical Intelligence, NVIDIA) hội tụ về cùng lúc trong giai đoạn 2024-2026.

> ⚠️ Có tìm thấy các mô tả về Figure AI's "Helix" (VLA điều khiển toàn bộ phần thân trên robot hình người, cũng theo kiến trúc dual-system) qua kết quả tìm kiếm, nhưng nguồn chủ yếu là blog/báo công nghệ thứ cấp, không phải paper/tài liệu kỹ thuật chính thức có thể trích dẫn số liệu cụ thể — nên **không đưa số liệu Helix vào đây**, chỉ ghi nhận xu hướng dual-system đang phổ biến để tránh bịa chi tiết chưa xác minh.

## ❓ Câu hỏi tự kiểm tra

1. Nếu một mô hình nhận (ảnh + text) và sinh ra text mô tả "robot nên cầm cốc bằng tay phải", mô hình đó là VLM hay VLA? Vì sao?

<details>
<summary>Đáp án</summary>

Đây vẫn là VLM, dù nội dung câu trả lời "nói về" hành động. Lý do: output vẫn nằm trong không gian ngôn ngữ (chuỗi token rời rạc, con người phải tự diễn giải và tự thực thi), không phải một action chunk (vector số thực liên tục sẵn sàng đẩy xuống bộ điều khiển motor). Ranh giới VLM/VLA nằm ở **loại output**, không nằm ở việc nội dung có "liên quan đến hành động vật lý" hay không.
</details>

2. Vì sao không thể coi proprioception là "một view camera nữa" và đưa vào VLM qua đúng cơ chế xử lý ảnh hiện có?

<details>
<summary>Đáp án</summary>

Vì proprioception không mang cấu trúc không gian kiểu pixel (không có "hàng xóm không gian" theo nghĩa ảnh 2D), có tần số cập nhật và bản chất vật lý khác (số liệu cảm biến góc/vận tốc/lực, thay đổi rất nhanh), và quan trọng nhất — nó gắn liền với **trạng thái hiện tại của một cơ thể robot cụ thể tại đúng thời điểm ra quyết định**, khiến hai lần forward-pass với cùng ảnh/text nhưng proprioception khác nhau phải cho ra hành động khác nhau. Đây là một ràng buộc mà kiến trúc xử lý ảnh thuần túy của VLM không được thiết kế để mang.
</details>

3. GR00T N1 có Eagle-2 (System 2) bên trong. Vậy Eagle-2 tự nó là VLA hay VLM? Còn GR00T N1 nói chung?

<details>
<summary>Đáp án</summary>

Eagle-2 tự nó là **VLM** — nhận (ảnh, text), sinh ra biểu diễn/token ngữ nghĩa, không tự sinh action chunk. Toàn bộ **GR00T N1** (System 2 + System 1, huấn luyện end-to-end cùng nhau) mới là **VLA** — vì chỉ ở mức toàn hệ thống, output cuối cùng mới là action chunk (từ System 1, được điều kiện hóa bởi output của System 2 và bởi proprioception).
</details>

4. Vì sao "trễ hoặc giật cục vài trăm mili-giây" là vấn đề nghiêm trọng với VLA nhưng gần như vô hại với VLM?

<details>
<summary>Đáp án</summary>

Vì output của VLA điều khiển trực tiếp một hệ vật lý thật đang chuyển động (motor, khớp, tiếp xúc) — một khoảng trễ hoặc bước nhảy giá trị đột ngột có thể tương ứng với một cú giật cơ học thật, đủ để làm robot mất thăng bằng hoặc làm rơi vật đang cầm. VLM chỉ sinh văn bản để con người đọc — một câu trả lời chậm nửa giây hoặc "giật cục" về mặt logic không tạo ra hậu quả vật lý tức thời nào.
</details>

5. Paper "Real-Time Execution of Action Chunking Flow Policies" (2025) chỉ ra action chunking giải quyết được vấn đề gì và **chưa** giải quyết được vấn đề gì? Điều này nói lên gì về việc coi "VLA = VLM + action head" là một bài toán đã đóng?

<details>
<summary>Đáp án</summary>

Action chunking giải quyết được tính nhất quán theo thời gian (temporal consistency) cho các tác vụ điều khiển tần số cao, nhưng **chưa** giải quyết trọn vẹn vấn đề độ trễ — vẫn có thể xảy ra dừng khựng hoặc chuyển động giật cục ở ranh giới giữa các chunk liên tiếp. Điều này cho thấy phần "sinh hành động liên tục, mượt, đúng thời gian thực" — chính là phần khiến VLA khác VLM về bản chất — vẫn là một mặt trận nghiên cứu đang mở năm 2025, không phải một chi tiết kỹ thuật đã được giải quyết xong và có thể xem nhẹ.
</details>

6. Một VLA giả định nhận (ảnh, text, proprioception) nhưng vẫn sinh ra output là token rời rạc mô tả hành động bằng ngôn ngữ (ví dụ "di chuyển tay phải lên trên") thay vì vector số thực liên tục — mô hình này có thoả mãn định nghĩa VLA đầy đủ trong bài này không? Vì sao cần cẩn trọng ở câu trả lời?

<details>
<summary>Đáp án</summary>

Đây là vùng xám cần cẩn trọng: theo định nghĩa nghiêm ngặt trong bài (output phải là action chunk — vector số thực liên tục), một mô hình chỉ sinh mô tả ngôn ngữ về hành động chưa thoả mãn đầy đủ, dù có nhận proprioception làm input. Tuy nhiên trong thực tế lịch sử (RT-2, OpenVLA — xem bài giảng riêng), hành động **được tokenize thành các token rời rạc** rồi vẫn được coi là VLA, vì các token đó được ánh xạ trực tiếp, xác định (không qua diễn giải ngôn ngữ tự nhiên) sang giá trị điều khiển liên tục ở bước giải mã cuối. Ranh giới then chốt không phải "có token hay không", mà là: token/giá trị đó có được thiết kế để **ánh xạ trực tiếp, không mơ hồ, sang lệnh motor** hay không — khác với token ngôn ngữ tự nhiên vốn cần con người/một hệ thống khác diễn giải lại.
</details>

## 📝 Bài tập thực hành

1. **Tự vẽ lại sơ đồ input/output** cho ba hệ thống bạn tự chọn (ví dụ: một chatbot ảnh bạn từng dùng, một robot hút bụi tự hành, và GR00T N1) theo đúng mẫu ASCII ở mục Cơ chế hoạt động. Với mỗi hệ thống, ghi rõ: input có đủ 3 loại (vision/language/proprioception) không, và output có phải action chunk liên tục hay không. Từ đó tự phân loại: hệ thống nào là VLM, hệ thống nào là VLA, hệ thống nào không thuộc cả hai loại.

2. **Đọc lại mục 2.2-2.3 của `NOI-DUNG-CHI-TIET.md`** (System 2 và System 1 của GR00T N1) và tự viết một đoạn 5-7 câu (không xem lại bài giảng này) giải thích cho một người mới học: vì sao Eagle-2 một mình không đủ để gọi là VLA, nhưng khi ghép với System 1 thì toàn bộ hệ thống lại là VLA. Sau đó so sánh với mục "Ví dụ minh hoạ trong dự án này" ở trên để tự chấm điểm mức độ chính xác của lời giải thích.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

VLM (Vision-Language Model) nhận ảnh và văn bản rồi sinh ra văn bản — output của nó mãi mãi là một chuỗi token rời rạc lấy từ một từ điển hữu hạn, và nó không biết gì về trạng thái vật lý hiện tại của bất kỳ cơ thể nào. VLA (Vision-Language-Action Model) nhận thêm proprioception — trạng thái khớp/lực hiện tại của robot — và thay vì sinh văn bản, nó sinh ra một action chunk: một chuỗi vector số thực liên tục cho nhiều bước thời gian tới, đủ mượt và đủ nhanh để điều khiển motor thật mà không làm robot giật cục hay ngã. Khác biệt này không phải "thêm một input, thêm vài lớp ở cuối" mà là khác biệt về loại bài toán học máy cốt lõi — phân loại rời rạc (VLM) so với sinh liên tục có ràng buộc thời gian thực (VLA) — và chính khác biệt đó là lý do các kiến trúc VLA hiện đại (GR00T N1/N1.7, π₀, Gemini Robotics) đều tách riêng một action head chuyên biệt (diffusion/flow matching) thay vì dùng thuần autoregressive token generation. Trong GR00T N1, Eagle-2/Cosmos-Reason2 (System 2) chỉ là phần VLM bên trong; chỉ toàn bộ hệ thống (System 2 + System 1, huấn luyện cùng nhau end-to-end) mới thoả mãn định nghĩa VLA đầy đủ.
