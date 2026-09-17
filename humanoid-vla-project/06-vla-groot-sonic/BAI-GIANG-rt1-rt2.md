# Bài giảng: Dòng phát triển RT-1 → RT-2

*(Thuộc mảng: Vision-Language-Action: GR00T N1 → N1.7 & SONIC)*

## 🎯 Mục tiêu bài học

- Nói được chính xác RT-1 huấn luyện trên loại dữ liệu gì, và giải thích được vì sao đây vừa là điểm mạnh vừa là "nỗi đau" của RT-1.
- Giải thích được ý tưởng đột phá của RT-2 bằng 1 câu: lấy VLM pretrain internet-scale rồi co-fine-tune cùng dữ liệu robot, biểu diễn hành động như token text.
- Tính tay được: cho một giá trị hành động liên tục cụ thể, rời rạc hoá nó thành bin nào trong 256 bin, và nói được bin đó trở thành token gì trong RT-2.
- Phân biệt được RT-1 và RT-2 trên ít nhất 4 tiêu chí (nguồn dữ liệu pretrain, cách sinh hành động, quy mô tham số, khả năng generalize).
- Chỉ ra được RT-2 kế thừa gì cho GR00T N1 (cụ thể: System 2 dùng VLM pretrain internet-scale) mà KHÔNG cần giải thích lại chi tiết GR00T N1 (đã có bài riêng).
- Nhận diện được ít nhất 2 hiểu nhầm phổ biến về RT-1/RT-2 và tự sửa được.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Trong toàn bộ dòng lịch sử VLA (Vision-Language-Action) dẫn tới GR00T N1/N1.7 mà dự án này theo dõi — RT-1 → RT-2 → OpenVLA → π₀ → GR00T N1 — RT-1 và RT-2 là hai mắt xích đầu tiên, và chúng đặt ra **câu hỏi thiết kế nền tảng** mà mọi VLA sau này đều phải trả lời: *hành động của robot nên được sinh ra từ đâu — chỉ từ dữ liệu robot thật (đắt, hiếm), hay có thể "mượn" tri thức đã học từ internet-scale data (rẻ, phong phú)?*

RT-1 (2022) trả lời bằng cách xây một kiến trúc transformer chuyên biệt, huấn luyện *toàn bộ* từ dữ liệu robot thật quy mô lớn — chứng minh được rằng kiến trúc "hấp thụ" tốt dữ liệu robot đa dạng, nhưng không giải quyết được vấn đề dữ liệu robot thật quá tốn kém để thu thập ở quy mô internet.

RT-2 (2023) trả lời khác hẳn: đừng huấn luyện từ đầu — hãy lấy một VLM khổng lồ đã pretrain sẵn trên ảnh+text từ web, rồi fine-tune thêm cùng lúc trên cả dữ liệu web gốc lẫn dữ liệu robot (co-fine-tune), biểu diễn hành động như một "ngôn ngữ" gồm token text. Đây chính là bước ngoặt mà **GR00T N1 kế thừa trực tiếp**: System 2 của GR00T N1 (mô-đun suy luận thị giác-ngôn ngữ) cũng là một VLM pretrain internet-scale (Eagle-2) — cùng triết lý "đừng học lại từ đầu, hãy tận dụng tri thức đã có sẵn" mà RT-2 đã chứng minh là hiệu quả.

Nếu không hiểu rõ RT-1 và RT-2 khác nhau ở đâu, bạn sẽ không hiểu vì sao GR00T N1 lại chọn dùng một VLM pretrain sẵn (Eagle-2) làm System 2 thay vì huấn luyện một encoder thị giác-ngôn ngữ từ đầu chỉ bằng dữ liệu robot — quyết định kiến trúc đó không phải ngẫu nhiên, nó là hệ quả trực tiếp của bài học RT-2 để lại.

Lưu ý phạm vi: bài này **chỉ** nói về RT-1 và RT-2. OpenVLA, π₀, và System 2 của GR00T N1 có bài giảng riêng (`BAI-GIANG-openvla-pi0.md`, `BAI-GIANG-groot-n1-dual-system.md`) — ở đây chỉ nhắc tên khi cần đặt bối cảnh, không giải thích chi tiết.

Định vị nhanh RT-1/RT-2 trên trục thời gian VLA (chi tiết từng mốc sau đọc ở các bài giảng riêng tương ứng):

- **2022 — RT-1:** VLA chuyên biệt, huấn luyện từ đầu, chỉ dùng dữ liệu robot thật.
- **2023 — RT-2:** VLM pretrain internet-scale + co-fine-tune, hành động = token text. *(bài giảng này)*
- **2023–2024 — Open X-Embodiment / RT-X:** mở rộng theo chiều đa dạng embodiment (nhiều loại robot), không phải mở rộng theo chiều tri thức pretrain (xem mục Cập nhật hiện đại).
- **2024 — OpenVLA:** mã nguồn mở, backbone Llama 2 + DINOv2/SigLIP, vẫn sinh hành động dạng token rời rạc như RT-2 nhưng công khai và nhỏ hơn nhiều lần.
- **2024 — π₀:** chuyển từ token rời rạc sang flow matching — sinh toàn bộ action chunk liên tục, không autoregressive từng token.
- **2025 — GR00T N1:** System 2 (Eagle-2, VLM pretrain) kế thừa triết lý RT-2; System 1 (flow-matching DiT) kế thừa hướng π₀.

## 🧠 Trực giác

### Góc nhìn 1: "Học nghề bằng cách chỉ xem thợ cả làm việc" so với "học nghề nhưng đã tốt nghiệp đại học tổng quát trước"

RT-1 giống như một người học nghề (thợ máy, thợ điện...) mà toàn bộ kiến thức chỉ đến từ việc quan sát và thực hành trực tiếp với nghề đó — không đọc sách, không học phổ thông trước. Người này có thể rất giỏi trong phạm vi những gì đã được dạy trực tiếp, và giỏi hơn nữa nếu được xem nhiều ca thực hành đa dạng hơn — nhưng gặp một tình huống chưa từng thấy (một loại vật liệu lạ, một dụng cụ mới) thì không có nền tảng kiến thức tổng quát để suy luận ra cách xử lý.

RT-2 giống như một người đã học xong đại học tổng quát (đọc rất nhiều sách, biết rất nhiều khái niệm về thế giới — tương đương VLM pretrain trên internet-scale data), sau đó mới học nghề cụ thể (co-fine-tune trên dữ liệu robot). Người này khi gặp tình huống lạ có thể suy luận dựa trên kiến thức nền rộng đã có (ví dụ: "đây là một quả bơ, tuy chưa từng cầm quả bơ nhưng biết nó là trái cây mềm nên cầm nhẹ tay").

- **Đúng ở đâu:** nắm đúng bản chất khác biệt cốt lõi — nguồn tri thức nền (chỉ dữ liệu nghề vs. tri thức tổng quát + nghề).
- **Giới hạn:** phép loại suy "con người học nghề" ngầm giả định người học có khả năng suy luận trừu tượng tự nhiên — trong khi RT-2 thực chất "suy luận" được là vì các trọng số của VLM pretrain đã mã hoá sẵn các mối liên hệ ngữ nghĩa thống kê từ dữ liệu web, không phải một quá trình tư duy có ý thức. Đừng suy diễn quá xa rằng RT-2 "hiểu" theo nghĩa con người hiểu.

### Góc nhìn 2: "Viết nhật ký bằng ký hiệu riêng" so với "viết nhật ký bằng chính ngôn ngữ mẹ đẻ đã biết"

Hãy tưởng tượng bạn cần ghi lại một chuỗi động tác (ví dụ công thức nấu ăn từng bước) để người khác làm theo. RT-1 giống như phát minh ra một *bộ ký hiệu số học hoàn toàn mới* riêng cho công thức đó — mỗi ký hiệu có ý nghĩa cụ thể trong hệ thống riêng, nhưng ký hiệu này không liên quan gì tới ngôn ngữ mà bạn (hay VLM) đã biết trước đó; đầu ra là một "đầu phân loại" (classification head) chuyên biệt cho từng chiều hành động, tách rời khỏi từ vựng ngôn ngữ.

RT-2 thì khác: nó viết công thức bằng cách *mượn luôn các từ/ký tự đã có sẵn trong cuốn từ điển tiếng mẹ đẻ* (từ vựng token của VLM) — chỉ định nghĩa lại ý nghĩa của một số từ ít dùng để chúng đại diện cho các bước động tác. Nhờ vậy bước sinh ra công thức động tác dùng chung một cơ chế với việc viết văn bản bình thường — không cần xây một "đầu ra" hoàn toàn mới.

- **Đúng ở đâu:** làm rõ điểm mấu chốt kỹ thuật — RT-2 tái sử dụng chính cơ chế sinh token ngôn ngữ (autoregressive next-token) cho hành động, thay vì có một action head tách biệt.
- **Giới hạn:** phép loại suy "mượn từ trong từ điển" dễ khiến người học tưởng nhầm rằng bản thân các token đó *có ý nghĩa ngôn ngữ* (ví dụ token số 172 "có nghĩa là gì đó" trong tiếng Anh) — thực ra không, các token bị ghi đè hoàn toàn tuỳ tiện (chọn các token *ít dùng nhất* trong vocab để giảm xáo trộn khả năng ngôn ngữ gốc), ý nghĩa duy nhất của chúng là vị trí bin trong lược đồ rời rạc hoá hành động, không mang ngữ nghĩa ngôn ngữ tự nhiên nào.

## 📐 Định nghĩa chính xác

**RT-1** (Brohan et al., Google, 2022 — *"RT-1: Robotics Transformer for Real-World Control at Scale"*, [arXiv:2212.06817](https://arxiv.org/abs/2212.06817)):

Một kiến trúc transformer "high-capacity" được thiết kế để **hấp thụ (absorb) dữ liệu robot đa dạng**, huấn luyện end-to-end từ đầu (không dùng VLM pretrain internet-scale) trên **dữ liệu thu thập trực tiếp từ robot thật thực hiện tác vụ thật** ("large-scale data collection on real robots performing real-world tasks"). Đầu vào là chuỗi ảnh camera + câu lệnh ngôn ngữ; đầu ra là hành động rời rạc hoá, sinh qua một tập các **đầu phân loại (classification head) theo từng chiều hành động** — không phải token ngôn ngữ tự nhiên.

**RT-2** ([arXiv:2307.15818](https://arxiv.org/abs/2307.15818)):

Một họ VLA lấy một **VLM lớn đã pretrain trên dữ liệu internet-scale** (ảnh+text từ web — cụ thể RT-2 dùng backbone **PaLI-X** hoặc **PaLM-E**), sau đó **co-fine-tune** (đồng thời fine-tune) trên cả dữ liệu web gốc lẫn dữ liệu robot thật. Hành động được **biểu diễn như token text**, chèn thêm vào từ vựng (vocabulary) sẵn có của VLM như một "ngôn ngữ hành động" — VLM sinh ra chuỗi token hành động này bằng đúng cơ chế autoregressive next-token prediction vốn đã dùng để sinh văn bản. Nhờ kế thừa trọng số pretrain trên internet-scale, RT-2 mang theo tri thức ngữ nghĩa/thường thức (common sense) mà dữ liệu robot đơn thuần không thể cung cấp.

Về mặt rời rạc hoá hành động (cơ chế chung mà cả RT-1 và RT-2 đều dùng): mỗi chiều hành động liên tục được **rời rạc hoá đều (uniformly discretized) thành 256 bin** trong khoảng giá trị giới hạn của chiều đó. Với RT-1, chỉ số bin này được một đầu phân loại chuyên biệt dự đoán trực tiếp. Với RT-2, chỉ số bin này được **ánh xạ thành một token trong từ vựng VLM** — với PaLI-X (vốn có token số nguyên riêng cho các số tới 1000), bin số *n* dùng luôn token số *n*; với PaLM-E (không có sẵn cơ chế biểu diễn số thuận tiện như vậy), RT-2 **ghi đè 256 token ít được dùng nhất trong từ vựng** để chúng đại diện cho 256 bin hành động.

Viết dưới dạng công thức cho một chiều hành động liên tục `a ∈ [a_min, a_max]`, rời rạc hoá đều thành `K = 256` bin:

```
bin_width  = (a_max − a_min) / K
bin_index  = clip( floor( (a − a_min) / bin_width ), 0, K−1 )
```

- RT-1: `bin_index` là nhãn huấn luyện cho đầu phân loại `softmax` K=256 lớp của chiều hành động đó (tối ưu bằng cross-entropy).
- RT-2: `bin_index` được tra vào bảng ánh xạ `token_id = f(bin_index)` — với `f` là "token số nguyên tương ứng" (PaLI-X) hoặc "token thứ *bin_index* trong danh sách 256 token ít dùng nhất bị ghi đè" (PaLM-E) — rồi token đó được nối vào chuỗi output cùng các chiều khác.

## ⚙️ Cơ chế hoạt động — từng bước

**Pipeline RT-1** — huấn luyện từ đầu, chỉ dùng dữ liệu robot thật:

```
┌──────────────────────────────────────────────────────────────────┐
│                          RT-1 PIPELINE                             │
│                                                                      │
│  [Ảnh camera (chuỗi 6 frame)] ──► FiLM-EfficientNet-B3 (16M tham số)│
│           │                              │                          │
│  [Câu lệnh ngôn ngữ] ──► embedding ──► FiLM conditioning ────┐      │
│                                                                │      │
│                                          ▼                    │      │
│                                   81 visual token/ảnh          │      │
│                                          │                     │      │
│                                   TokenLearner (nén 81→8 token)│      │
│                                          │                     │      │
│                                          ▼                     │      │
│                          Transformer decoder-only               │      │
│                          (8 lớp self-attention, 19M tham số)     │      │
│                                          │                     │      │
│                                          ▼                     │      │
│           Đầu phân loại riêng cho từng chiều hành động          │      │
│      (11 chiều: 7 tay máy + 3 di chuyển đế + 1 chọn mode)       │      │
│           mỗi chiều → 256 bin rời rạc (uniform)                 │      │
│                                                                      │
│  Toàn bộ pipeline: 35M tham số, huấn luyện 100% từ dữ liệu robot   │
│  thật (130k demo, 13 robot, 744 tác vụ), chạy ở 3Hz.               │
└──────────────────────────────────────────────────────────────────┘
```

**Pipeline RT-2** — VLM pretrain internet-scale + co-fine-tune, hành động = token text:

```
┌──────────────────────────────────────────────────────────────────┐
│                          RT-2 PIPELINE                             │
│                                                                      │
│  BƯỚC 1 — Pretrain (đã có sẵn, không làm lại):                     │
│  VLM lớn (PaLI-X 55B hoặc PaLM-E 12B) được pretrain trên            │
│  dữ liệu internet-scale (ảnh+text từ web: caption, VQA...)          │
│                                                                      │
│  BƯỚC 2 — Chuẩn bị "ngôn ngữ hành động":                            │
│  Rời rạc hoá mỗi chiều hành động liên tục → 256 bin                 │
│  Ánh xạ bin → token trong từ vựng VLM:                              │
│    - PaLI-X: dùng token số nguyên có sẵn (0..1000)                  │
│    - PaLM-E: ghi đè 256 token ít dùng nhất trong vocab              │
│                                                                      │
│  BƯỚC 3 — Co-fine-tune (đồng thời, không tuần tự):                 │
│  ┌─────────────────────┐        ┌─────────────────────────┐        │
│  │ Dữ liệu web gốc      │        │ Dữ liệu robot thật        │        │
│  │ (caption, VQA...)    │  cùng  │ (ảnh + lệnh → chuỗi       │        │
│  │                      │  lúc   │ token hành động)          │        │
│  └─────────────────────┘        └─────────────────────────┘        │
│              │                              │                       │
│              └──────────────┬───────────────┘                       │
│                              ▼                                      │
│          VLM (đã pretrain) tiếp tục học, KHÔNG quên tri thức web    │
│                              │                                      │
│                              ▼                                      │
│  [Ảnh camera + câu lệnh] ──► VLM ──► sinh chuỗi token AUTOREGRESSIVE│
│                                       ví dụ: "1 128 91 241 5 101..." │
│                                       (mỗi số = 1 bin/1 chiều hành động)│
│                              │                                      │
│                              ▼                                      │
│               Giải mã chuỗi token → vector hành động liên tục       │
└──────────────────────────────────────────────────────────────────┘
```

Khác biệt cốt lõi nằm ở **bước xuất phát**: RT-1 bắt đầu từ số 0 (random init) và học mọi thứ — cả cách "nhìn" lẫn cách "hành động" — chỉ từ dữ liệu robot. RT-2 bắt đầu từ một VLM đã "biết nhìn và biết ngôn ngữ" rất tốt (nhờ pretrain trên dữ liệu lớn hơn dữ liệu robot nhiều bậc), và chỉ cần dạy thêm nó "nói" bằng một phương ngữ mới (token hành động) song song với việc không quên ngôn ngữ gốc.

**Vì sao "co-fine-tune" chứ không phải "fine-tune" đơn thuần — và tỉ lệ trộn dữ liệu quan trọng thế nào:**

Nếu chỉ fine-tune thuần tuý trên dữ liệu robot (bỏ hẳn dữ liệu web trong giai đoạn này), VLM sẽ dần "quên" tri thức ngữ nghĩa đã pretrain — hiện tượng gọi là **catastrophic forgetting** (quên thảm hoạ: mô hình đánh mất năng lực cũ khi học năng lực mới trên một phân phối dữ liệu hẹp hơn). Đây chính là lý do RT-2 phải **co-fine-tune**: trộn dữ liệu web gốc (VQA, caption...) cùng dữ liệu robot trong **cùng một batch huấn luyện**, không huấn luyện tuần tự riêng rẽ. Theo phân tích từ cộng đồng nghiên cứu VLA (tổng hợp qua WebSearch, dựa trên cách RT-2 công bố), dữ liệu robot thường được **lấy mẫu nhiều hơn (up-sampled) để chiếm khoảng 50–66% mỗi minibatch**, phần còn lại (34–50%) là dữ liệu web — nhằm vừa đủ để mô hình học tốt điều khiển robot vừa không xoá mất năng lực suy luận ngữ nghĩa đã có. Tỉ lệ trộn này là một siêu tham số (hyperparameter) không tầm thường, đòi hỏi nhiều lượt huấn luyện và đánh giá thực tế mới tìm ra được mức cân bằng phù hợp. **Cần xác minh thêm** con số tỉ lệ chính xác nếu trích dẫn học thuật trực tiếp — bài giảng này tổng hợp từ phân tích thứ cấp, không trích trực tiếp từ một bảng số liệu cụ thể trong paper gốc.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ tự chọn số liệu để dễ hình dung, dựa đúng lược đồ "rời rạc hoá đều thành 256 bin" mà cả RT-1 và RT-2 dùng theo paper — không lấy từ một trial cụ thể nào trong paper.)*

Giả sử ta đang rời rạc hoá **1 chiều hành động**: độ mở của gripper (tay kẹp), với khoảng giá trị được chuẩn hoá về `[-1, 1]` (−1 = đóng hoàn toàn, +1 = mở hoàn toàn), dùng **256 bin đều nhau (uniform)**.

**Bước 1 — tính độ rộng mỗi bin:**

```
bin_width = (max − min) / số_bin = (1 − (−1)) / 256 = 2 / 256 = 0.0078125
```

**Bước 2 — giả sử policy cho ra giá trị hành động liên tục cụ thể:** `a = 0.35` (gripper mở khoảng 67.5% — vì (0.35-(-1))/2 = 0.675).

**Bước 3 — tính chỉ số bin:**

```
bin_index = floor((a − min) / bin_width)
          = floor((0.35 − (−1)) / 0.0078125)
          = floor(1.35 / 0.0078125)
          = floor(172.8)
          = 172
```

Vậy giá trị hành động liên tục `a = 0.35` rơi vào **bin số 172** (trong khoảng bin 0..255).

**Bước 4 — với RT-1:** bin 172 là nhãn (label) mà đầu phân loại riêng cho chiều "gripper" cần dự đoán đúng — đây thuần tuý là bài toán phân loại 256 lớp (cross-entropy), không liên quan gì tới từ vựng ngôn ngữ.

**Bước 5 — với RT-2:** bin 172 phải được ánh xạ thành **1 token trong từ vựng của VLM**:
- Nếu backbone là **PaLI-X** (có token số nguyên riêng cho các số tới 1000): bin 172 dùng luôn **token biểu diễn số "172"**.
- Nếu backbone là **PaLM-E** (không có cơ chế số thuận tiện): bin 172 tương ứng với **1 trong 256 token đã bị ghi đè** khỏi nhóm token ít dùng nhất trong vocab gốc — ví dụ (minh hoạ) token gốc thứ 172 trong danh sách 256 token bị ghi đè đó, không mang ý nghĩa ngôn ngữ gốc của nó nữa.

Nếu hành động đầy đủ có nhiều chiều (ví dụ 8 chiều: vị trí x,y,z + xoay roll,pitch,yaw + gripper + mode), mỗi chiều được rời rạc hoá và ánh xạ token riêng, rồi **nối chuỗi lại bằng ký tự khoảng trắng** thành một câu "hành động" — đúng như ví dụ trên trang chính thức của RT-2: chuỗi token dạng `"1 128 91 241 5 101 127 217"` — mỗi số là chỉ số bin/token của 1 chiều hành động.

**Điểm dễ bỏ sót: mỗi chiều hành động có khoảng giá trị (và do đó `bin_width`) riêng, không dùng chung `[-1, 1]`.** Ví dụ chiều "di chuyển theo trục z của end-effector" có thể có khoảng giá trị khác, chẳng hạn `[0, 0.5]` mét (chỉ di chuyển lên, không âm):

```
bin_width_z = (0.5 − 0) / 256 = 0.001953125
```

Giả sử giá trị hành động liên tục theo z là `a_z = 0.28` mét:

```
bin_index_z = floor((0.28 − 0) / 0.001953125) = floor(143.36) = 143
```

Vậy cùng một lược đồ "256 bin" nhưng chiều gripper (khoảng `[-1,1]`) cho ra bin 172 với giá trị 0.35, còn chiều z (khoảng `[0, 0.5]`) cho ra bin 143 với giá trị 0.28 — hai con số bin gần nhau về độ lớn nhưng biểu diễn hai đại lượng vật lý hoàn toàn khác nhau, vì `bin_width` của mỗi chiều được tính riêng theo khoảng giá trị riêng của chiều đó. Đây là lý do khi đọc một chuỗi token hành động dạng `"1 128 91 241..."`, con số tự nó không có ý nghĩa gì nếu tách rời khỏi biết trước "vị trí thứ mấy trong chuỗi tương ứng với chiều nào và khoảng giá trị nào" — thông tin ánh xạ đó phải được định nghĩa cố định trước khi huấn luyện, không suy ra được từ bản thân con số token.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | RT-1 | RT-2 |
|---|---|---|
| Nguồn pretrain | Không có — huấn luyện từ đầu (from scratch) 100% trên dữ liệu robot thật | VLM lớn (PaLI-X hoặc PaLM-E) đã pretrain sẵn trên dữ liệu internet-scale (ảnh+text web) |
| Cách sinh hành động | Đầu phân loại (classification head) riêng cho từng chiều hành động, không qua từ vựng ngôn ngữ | Token text — ánh xạ bin hành động vào từ vựng VLM, sinh bằng cơ chế autoregressive next-token giống sinh văn bản |
| Quy mô tham số | 35M tham số (nhỏ) | 55B (PaLI-X) hoặc 12B (PaLM-E) — lớn hơn RT-1 hàng nghìn lần |
| Dữ liệu huấn luyện | 130.000 demo robot thật, 13 robot, 744 tác vụ (theo paper RT-1) | Co-fine-tune đồng thời: dữ liệu web gốc (VQA, caption...) + dữ liệu robot thật (kế thừa dữ liệu kiểu RT-1) |
| Tần số điều khiển | 3Hz (báo cáo trong paper RT-1) | Chậm hơn đáng kể do backbone rất lớn — thường cần chạy trên cụm máy chủ từ xa (cloud), không phải on-board; **cần xác minh thêm** con số Hz cụ thể nếu trích dẫn học thuật |
| Khả năng generalize tới đối tượng/khung cảnh chưa từng thấy | Hạn chế hơn — theo khảo sát cộng đồng, RT-2 "so với RT-1 thì tương đương ở tác vụ đã thấy (seen), nhưng vượt trội rõ ở đối tượng/khung cảnh/môi trường chưa từng thấy" | Vượt trội ở generalization nhờ tri thức internet-scale — theo trang chính thức RT-2, cải thiện ~2 lần (2x) ở các trục generalization và ~3 lần (3x) ở các kỹ năng "emergent" (biểu tượng, suy luận, nhận diện người) so với baseline RT-1/VC-1 |
| Suy luận ngữ nghĩa/common sense (ví dụ hiểu "đá có thể dùng như búa") | Không có cơ chế này — chỉ học từ demo cụ thể | Có, đặc biệt ở biến thể PaLM-E dùng chain-of-thought — vì thừa hưởng tri thức pretrain internet-scale |
| "Nỗi đau" cần giải quyết | Thu thập dữ liệu robot thật cực kỳ tốn kém, khó scale lên internet-scale | Ngược lại: cần cẩn trọng co-fine-tune đúng tỉ lệ để không "quên" tri thức web gốc (catastrophic forgetting), và chi phí tính toán/độ trễ khi backbone rất lớn |

Nhìn vào bảng trên, có một điểm đáng nói thêm về hàng "Nỗi đau cần giải quyết": RT-1 và RT-2 không nằm trên một trục "cái nào tốt hơn tuyệt đối", mà mỗi bên đánh đổi một loại chi phí lấy một loại năng lực khác. RT-1 trả giá bằng **chi phí thu thập dữ liệu** (mỗi demo robot thật tốn công sức con người vận hành/teleoperate) để đổi lấy một mô hình nhỏ, chạy nhanh (3Hz, 35M tham số), không phụ thuộc hạ tầng tính toán lớn. RT-2 trả giá bằng **chi phí hạ tầng tính toán** (backbone hàng chục tỷ tham số, khó chạy on-board thời gian thực) để đổi lấy khả năng khái quát hoá vượt trội mà không cần tăng tương ứng lượng dữ liệu robot. Đây chính là lý do các thế hệ VLA sau (OpenVLA, π₀, GR00T N1 — xem các bài giảng riêng) đều tìm cách "lấy cả hai": giữ lại lợi ích VLM pretrain internet-scale của RT-2, nhưng thiết kế lại phần sinh hành động (action head) để chạy đủ nhanh cho điều khiển robot thời gian thực — thay vì token text autoregressive chậm của RT-2.

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "RT-2 chỉ là RT-1 cộng thêm nhiều dữ liệu hơn."**
   Vì sao sai: đây không phải khác biệt về *lượng* dữ liệu, mà là khác biệt về *loại* tri thức nền và *cơ chế sinh hành động*. RT-2 không chỉ thêm dữ liệu robot — nó thay đổi hẳn điểm xuất phát (một VLM pretrain internet-scale khổng lồ, không liên quan gì tới robot) và thay đổi hẳn cách hành động được biểu diễn (token text thay vì đầu phân loại chuyên biệt). Nếu chỉ thêm dữ liệu robot vào kiến trúc kiểu RT-1 mà không đổi backbone/cơ chế, sẽ không có được các khả năng "emergent" (suy luận trên đối tượng/khái niệm chưa từng có trong demo robot) mà RT-2 thể hiện.
   Hiểu đúng: khác biệt là kiến trúc + nguồn tri thức nền, không phải quy mô dữ liệu robot.

2. **Hiểu nhầm: "RT-1 cũng dùng VLM pretrain internet-scale, chỉ là nhỏ hơn RT-2."**
   Vì sao sai: RT-1 được huấn luyện hoàn toàn từ đầu (from scratch) trên dữ liệu robot thật — backbone thị giác của nó (FiLM-EfficientNet-B3) và Transformer decoder không hề pretrain trên dữ liệu internet-scale ảnh+text kiểu VLM. Đây chính xác là điểm RT-2 khắc phục — nếu nói RT-1 "cũng có VLM pretrain, chỉ nhỏ hơn" là xoá bỏ chính ý tưởng đột phá làm nên RT-2.
   Hiểu đúng: RT-1 không dùng VLM pretrain internet-scale; đây là khác biệt về *loại kiến trúc*, không phải khác biệt về *quy mô* của cùng một loại kiến trúc.

3. **Hiểu nhầm: "Token hành động của RT-2 mang ý nghĩa ngôn ngữ tự nhiên nào đó."**
   Vì sao sai: như đã nói ở phần Trực giác góc nhìn 2, các token bị ghi đè (đặc biệt với PaLM-E) được chọn từ nhóm *ít dùng nhất* trong từ vựng gốc chỉ để giảm xáo trộn khả năng ngôn ngữ hiện có — bản thân token đó không còn giữ ý nghĩa ngôn ngữ ban đầu, ý nghĩa duy nhất của nó bây giờ là chỉ số bin hành động.
   Hiểu đúng: đây là một phép "mượn hạ tầng sinh token" (kỹ thuật), không phải một tuyên bố ngữ nghĩa học.

4. **Hiểu nhầm: "RT-2 sinh nguyên cả action chunk (nhiều timestep) cùng lúc, giống flow-matching action head của các VLA hiện đại (π₀/GR00T N1)."**
   Vì sao sai: RT-2 (và RT-1) sinh hành động theo cơ chế autoregressive — từng token một, tuần tự, cho một timestep hành động tại một thời điểm (rồi lặp lại toàn bộ pipeline cho timestep tiếp theo). Đây khác hẳn với flow-matching/diffusion action head (π₀, GR00T N1 System 1) vốn sinh ra **toàn bộ một action chunk H bước liên tục cùng lúc** trong một lượt forward pass. Nhầm hai cơ chế này với nhau sẽ khiến bạn hiểu sai lý do vì sao RT-2 chậm hơn nhiều so với các VLA thế hệ sau khi triển khai điều khiển thời gian thực.
   Hiểu đúng: RT-2 sinh từng token/từng timestep tuần tự (autoregressive, chậm); flow-matching action head sinh cả chunk hành động liên tục cùng lúc (nhanh hơn nhiều cho điều khiển real-time) — đây là đúng lý do các VLA sau RT-2 chuyển hướng, như đã nhắc ở mục Cập nhật hiện đại bên dưới.

## 🏗️ Ví dụ minh hoạ trong dự án này

Trong dự án này, khái niệm RT-1 → RT-2 được dùng làm **bối cảnh bắt buộc** để hiểu quyết định kiến trúc của GR00T N1 (xem `NOI-DUNG-CHI-TIET.md` mục 2, và bài giảng riêng về dual-system của GR00T N1):

- **System 2 của GR00T N1** — mô-đun suy luận thị giác-ngôn ngữ, dùng backbone **Eagle-2**, một "pre-trained Vision-Language Model (VLM) pretrained on Internet-scale data" — đây **chính xác là cùng triết lý kiến trúc mà RT-2 đã chứng minh hiệu quả**: đừng học lại tri thức ngữ nghĩa/thị giác từ đầu chỉ bằng dữ liệu robot khan hiếm, hãy tận dụng một VLM đã pretrain sẵn.
- Điểm khác biệt quan trọng mà GR00T N1 không đi theo RT-2 y hệt: GR00T N1 **không** biểu diễn hành động như token text sinh autoregressive (cách RT-2 làm) — thay vào đó System 1 của GR00T N1 dùng một action head flow-matching riêng biệt (kế thừa từ hướng đi của π₀, xem bài giảng riêng `BAI-GIANG-openvla-pi0.md`), sinh trực tiếp action chunk liên tục thay vì từng token rời rạc một. Đây là bài học tiếp theo mà cộng đồng VLA rút ra sau RT-2: sinh token rời rạc tuần tự (autoregressive) quá chậm và giới hạn độ mượt cho điều khiển motor thời gian thực — nhưng đó là câu chuyện của bài giảng khác, không phải bài này.

Tóm lại: nếu không hiểu RT-2 "mượn VLM pretrain internet-scale" để giải quyết vấn đề khan hiếm dữ liệu robot của RT-1, bạn sẽ không hiểu vì sao GR00T N1 lại chọn Eagle-2 làm System 2 thay vì tự huấn luyện một mô-đun thị giác-ngôn ngữ mới từ đầu.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **Không có "RT-3" chính thức từ Google DeepMind.** Qua các lượt tìm kiếm (tháng 9/2026), không tìm thấy paper hoặc công bố chính thức nào tên "RT-3". Thay vào đó, hướng mở rộng chính thức tiếp theo của dòng RT là **Open X-Embodiment và các mô hình RT-X** — [arXiv:2310.08864](https://arxiv.org/abs/2310.08864), *"Open X-Embodiment: Robotic Learning Datasets and RT-X Models"* (công bố 2023, trình bày tại ICRA 2024): một bộ dữ liệu tổng hợp từ **22 embodiment robot khác nhau, do 21 tổ chức nghiên cứu đóng góp, bao phủ 527 kỹ năng (skills)**. Từ bộ dữ liệu này, các mô hình **RT-1-X** và **RT-2-X** được huấn luyện để kiểm chứng khả năng khái quát hoá xuyên nhiều loại robot (cross-embodiment) — đây là hướng "mở rộng theo chiều đa dạng embodiment", khác với việc ra một phiên bản "RT-3" kế tiếp thuần tăng quy mô.

2. **Con số cụ thể của RT-1 (theo paper gốc, đã xác minh qua bản HTML đầy đủ):** huấn luyện trên **130.000 demonstration**, thu thập từ **13 robot** trong 17 tháng, bao phủ **744 câu lệnh tác vụ** riêng biệt; đạt **97% success rate trên tác vụ đã thấy (seen)** và **76% trên tác vụ chưa thấy/tổ hợp mới (unseen)**; robustness với vật cản gây nhiễu (distractor) đạt 83%, với khung cảnh mới (background) đạt 59%. Kiến trúc: FiLM-conditioned EfficientNet-B3 (16M tham số) + TokenLearner (nén 81 token ảnh còn 8 token) + Transformer decoder-only 8 lớp (19M tham số) — tổng **35M tham số**, chạy ở **3Hz**. Nguồn: [arXiv:2212.06817](https://arxiv.org/abs/2212.06817).

3. **Con số cụ thể của RT-2 (theo paper gốc và trang chính thức):** hai biến thể **RT-2 (PaLI-X, 55 tỷ tham số)** và **RT-2 (PaLM-E, 12 tỷ tham số)**. Đánh giá qua khoảng **6.000 lượt thử (evaluation trials)**. Trên benchmark mô phỏng Language-Table, RT-2 đạt **90% success rate so với 77% của SOTA trước đó**. So với baseline (RT-1 và VC-1), RT-2 cải thiện khoảng **gấp 2 lần (2x)** ở các trục khái quát hoá (đối tượng/khung cảnh/môi trường mới) và khoảng **gấp 3 lần (3x)** ở các kỹ năng "nổi sinh" (emergent) như hiểu biểu tượng, suy luận, nhận diện người. Nguồn: [arXiv:2307.15818](https://arxiv.org/abs/2307.15818), trang chính thức [robotics-transformer2.github.io](https://robotics-transformer2.github.io/).

4. **RT-1/RT-2 vẫn được trích dẫn làm mốc lịch sử nền tảng trong các khảo sát VLA 2025-2026**, ví dụ *"Vision-Language-Action Models: Concepts, Progress, Applications and Challenges"* ([arXiv:2505.04769](https://arxiv.org/abs/2505.04769)) và *"A Survey on Vision-Language-Action Models: An Action Tokenization Perspective"* ([arXiv:2507.01925](https://arxiv.org/abs/2507.01925)) — cả hai đều dùng RT-1 làm ví dụ đầu tiên của "VLA transformer huấn luyện từ đầu trên dữ liệu robot" và RT-2 làm ví dụ đầu tiên của "hợp nhất VLM pretrain với sinh hành động dạng token", đúng khung phân loại mà bài giảng này trình bày.

5. **Xu hướng bỏ hẳn "token hành động rời rạc kiểu RT-2"** đã xuất hiện rõ ở các công trình gần hơn: [FAST (Physical Intelligence)](https://www.pi.website/download/fast.pdf) chỉ ra rằng lược đồ rời rạc hoá theo từng chiều/timestep độc lập kiểu RT-1/RT-2 kém hiệu quả khi sinh action chunk dài, và đề xuất một scheme tokenization hiệu quả hơn (dùng biến đổi tần số — discrete cosine transform); còn dòng π₀/GR00T N1 (đã nhắc ở mục Ví dụ dự án) bỏ hẳn token rời rạc, chuyển sang flow matching sinh hành động liên tục trực tiếp. Điều này cho thấy RT-2 tuy là bước ngoặt quan trọng (VLM pretrain + hành động dạng token) nhưng **chính cách biểu diễn hành động dạng token rời rạc của nó lại là điểm mà thế hệ VLA sau (2024-2026) tìm cách thay thế**, trong khi vẫn giữ lại ý tưởng "tận dụng VLM pretrain internet-scale" mà RT-2 mở đường.

6. **Bối cảnh SOTA 2026 (mức tổng quan, không đi sâu vì ngoài phạm vi bài này):** theo tổng hợp tìm kiếm, tính tới 2026 các hệ thống được xem là SOTA tiếp nối đúng triết lý "VLM pretrain + action generation" mà RT-2 khởi xướng bao gồm **π₀.6 (Physical Intelligence)** và **Gemini Robotics (Google DeepMind)** — cả hai đều là VLA thế hệ mới xây trên nền tảng ý tưởng RT-2 để lại, nhưng với action head và quy mô dữ liệu khác biệt đáng kể; **cần xác minh thêm** các con số cụ thể của hai hệ thống này nếu trích dẫn học thuật, vì các con số trong bài này chỉ tổng hợp từ tìm kiếm web, chưa đọc trực tiếp paper/technical report tương ứng.

7. **Vấn đề catastrophic forgetting của RT-2 vẫn là đề tài nghiên cứu tiếp diễn năm 2025**, ví dụ *"Actions as Language: Fine-Tuning VLMs into VLAs Without Catastrophic Forgetting"* ([arXiv:2509.22195](https://arxiv.org/abs/2509.22195)) — tên paper nêu thẳng vấn đề mà mục Cơ chế hoạt động ở trên đã giải thích: co-fine-tune kiểu RT-2 (trộn dữ liệu web + robot theo một tỉ lệ cố định trong mỗi batch) là một giải pháp hiệu quả nhưng không triệt để, và các công trình sau vẫn tiếp tục tìm cách fine-tune VLM thành VLA mà không đánh đổi năng lực ngôn ngữ/thị giác gốc. Điều này củng cố thêm luận điểm: RT-2 mở ra hướng đi đúng (tận dụng VLM pretrain) nhưng cách hiện thực hoá cụ thể của nó (co-fine-tune tỉ lệ cố định, hành động dạng token rời rạc) vẫn còn nhiều dư địa cải tiến mà cộng đồng VLA tiếp tục khai thác nhiều năm sau khi RT-2 công bố.

## ❓ Câu hỏi tự kiểm tra

1. RT-1 huấn luyện từ loại dữ liệu nào? Vì sao đây vừa là điểm mạnh vừa là hạn chế?
<details><summary>Gợi ý đáp án</summary>Dữ liệu robot thật thu thập trực tiếp (130k demo, 13 robot, 744 tác vụ). Điểm mạnh: kiến trúc "hấp thụ" tốt dữ liệu đa dạng, hiệu năng tăng theo lượng/độ đa dạng dữ liệu (scalable). Hạn chế: dữ liệu robot thật rất tốn kém để thu thập ở quy mô lớn, không thể dễ dàng mở rộng lên quy mô internet-scale như dữ liệu ảnh+text.</details>

2. RT-2 giải quyết "nỗi đau" của RT-1 bằng cách nào, cụ thể là gì?
<details><summary>Gợi ý đáp án</summary>Lấy một VLM lớn đã pretrain sẵn trên dữ liệu internet-scale (PaLI-X hoặc PaLM-E), rồi co-fine-tune (đồng thời fine-tune) trên cả dữ liệu web gốc lẫn dữ liệu robot — thay vì phải huấn luyện lại toàn bộ tri thức chỉ từ dữ liệu robot khan hiếm.</details>

3. Hành động trong RT-2 được biểu diễn như thế nào? Nêu khác biệt giữa cách ánh xạ bin-token của PaLI-X và PaLM-E.
<details><summary>Gợi ý đáp án</summary>Biểu diễn như token text, thêm vào từ vựng VLM như một "ngôn ngữ hành động", sinh qua cơ chế autoregressive next-token. PaLI-X: dùng token số nguyên có sẵn (0-1000), bin n dùng token số n. PaLM-E: không có token số thuận tiện, nên ghi đè 256 token ít dùng nhất trong vocab để đại diện 256 bin.</details>

4. Cho giá trị hành động liên tục `a = -0.6` trong khoảng `[-1, 1]`, rời rạc hoá đều thành 256 bin. Tính chỉ số bin.
<details><summary>Gợi ý đáp án</summary>bin_width = 2/256 = 0.0078125. bin_index = floor((-0.6 - (-1))/0.0078125) = floor(0.4/0.0078125) = floor(51.2) = 51.</details>

5. Vì sao nói "RT-2 chỉ là RT-1 cộng thêm dữ liệu" là một hiểu nhầm? Chỉ ra đúng 2 điểm khác biệt bản chất.
<details><summary>Gợi ý đáp án</summary>Khác biệt bản chất là: (1) nguồn tri thức nền — RT-1 học từ đầu chỉ từ dữ liệu robot, RT-2 bắt đầu từ VLM pretrain internet-scale; (2) cơ chế sinh hành động — RT-1 dùng đầu phân loại riêng theo từng chiều, RT-2 dùng token text sinh autoregressive qua chính cơ chế ngôn ngữ của VLM. Đây là khác biệt kiến trúc/loại tri thức, không phải khác biệt số lượng dữ liệu.</details>

6. GR00T N1 kế thừa ý tưởng gì từ RT-2, và khác RT-2 ở điểm nào trong cách sinh hành động?
<details><summary>Gợi ý đáp án</summary>Kế thừa: dùng một VLM pretrain internet-scale (Eagle-2) làm mô-đun suy luận ngữ nghĩa (System 2), cùng triết lý "tận dụng tri thức pretrain thay vì học lại từ đầu" mà RT-2 mở đường. Khác: GR00T N1 không sinh hành động dạng token text autoregressive như RT-2 — System 1 dùng action head flow-matching riêng, sinh action chunk liên tục trực tiếp (kế thừa hướng π₀, không phải hướng token rời rạc của RT-2).</details>

7. Vì sao RT-2 cần "co-fine-tune" thay vì chỉ fine-tune thuần tuý trên dữ liệu robot? Tỉ lệ trộn dữ liệu (theo tổng hợp thứ cấp) nằm trong khoảng nào?
<details><summary>Gợi ý đáp án</summary>Vì fine-tune thuần tuý trên dữ liệu robot (bỏ dữ liệu web) sẽ gây catastrophic forgetting — VLM quên dần tri thức ngữ nghĩa đã pretrain. Co-fine-tune trộn cả 2 loại dữ liệu trong cùng batch để giữ lại năng lực ngôn ngữ/thị giác gốc. Theo tổng hợp thứ cấp, dữ liệu robot thường chiếm khoảng 50-66% mỗi minibatch, phần còn lại là dữ liệu web (34-50%) — cần xác minh thêm nếu trích dẫn học thuật.</details>

## 📝 Bài tập thực hành

1. **Tính tay đầy đủ 1 hành động nhiều chiều:** cho một hành động giả định 3 chiều (x, y, gripper), mỗi chiều trong khoảng `[-1, 1]`, giá trị lần lượt là `x = 0.10`, `y = -0.85`, `gripper = 0.99`. Rời rạc hoá đều mỗi chiều thành 256 bin (dùng công thức đã học ở phần Ví dụ tính tay), tính 3 chỉ số bin, rồi viết ra chuỗi "token hành động" dạng RT-2 (nối 3 số bằng khoảng trắng, ví dụ tương tự `"1 128 91..."` trên trang chính thức RT-2).

   Gợi ý các bước làm:
   - Bước 1: tính `bin_width = 2/256` cho cả 3 chiều (vì cả 3 đều dùng khoảng `[-1,1]`).
   - Bước 2: áp công thức `bin_index = floor((a − min)/bin_width)` cho từng giá trị `x`, `y`, `gripper`.
   - Bước 3: chú ý trường hợp `gripper = 0.99` rất gần biên trên (`max = 1`) — kiểm tra xem `bin_index` tính ra có vượt quá 255 không, và nếu có, nhớ áp bước `clip` đã nêu ở phần Cơ chế hoạt động.
   - Bước 4: viết 3 chỉ số bin cạnh nhau, cách nhau bằng khoảng trắng, theo đúng định dạng chuỗi token hành động của RT-2.

2. **Đọc thêm và đối chiếu:** mở phần "3.2. N1.7: EgoScale và scaling law cho dexterity" trong `NOI-DUNG-CHI-TIET.md` (cùng thư mục `06-vla-groot-sonic/`), và viết 3-5 câu (bằng lời của bạn, không copy) giải thích: N1.7 dùng video egocentric con người thay vì token hành động dạng RT-2 để mở rộng dữ liệu — vì sao đây là một hướng "mở rộng dữ liệu" khác hẳn với hướng Open X-Embodiment/RT-X (mở rộng theo số embodiment robot) đã nhắc ở mục Cập nhật hiện đại của bài này.

3. **Tự kiểm chứng nguồn (source-checking):** mở trực tiếp trang chính thức RT-2 ([robotics-transformer2.github.io](https://robotics-transformer2.github.io/)) hoặc bản HTML đầy đủ của paper RT-1 ([arxiv.org/abs/2212.06817](https://arxiv.org/abs/2212.06817)), tìm đúng đoạn văn xác nhận 2 con số đã dùng trong bài giảng này: (a) success rate 97%/76% của RT-1 trên tác vụ seen/unseen, (b) số lượng 6.000 lượt thử (evaluation trials) của RT-2. Ghi lại chính xác câu trích dẫn (quote) và vị trí (mục nào trong paper) — đây là kỹ năng quan trọng hơn bản thân con số: luôn tự xác minh lại số liệu trước khi dùng nó trong bài viết/báo cáo của riêng bạn, đúng tinh thần "không bịa số liệu" mà toàn bộ tài liệu dự án này tuân theo.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

RT-1 (2022) là VLA robot tay máy có ảnh hưởng lớn đầu tiên, huấn luyện hoàn toàn từ đầu trên dữ liệu robot thật quy mô lớn (130k demo, 13 robot, 744 tác vụ, đạt 97%/76% success rate trên tác vụ đã thấy/chưa thấy) — chứng minh kiến trúc transformer hấp thụ tốt dữ liệu đa dạng, nhưng để lại "nỗi đau" là chi phí thu thập dữ liệu robot thật quá lớn để mở rộng lên quy mô internet. RT-2 (2023) giải quyết nỗi đau đó bằng cách lấy một VLM khổng lồ đã pretrain trên dữ liệu internet-scale (PaLI-X 55B hoặc PaLM-E 12B), co-fine-tune đồng thời trên cả dữ liệu web gốc lẫn dữ liệu robot, và biểu diễn hành động như token text — mỗi chiều hành động liên tục rời rạc hoá thành 256 bin rồi ánh xạ vào từ vựng VLM, sinh ra bằng chính cơ chế autoregressive next-token vốn dùng để sinh văn bản. Nhờ vậy RT-2 đạt được các khả năng "nổi sinh" (emergent) như suy luận trên đối tượng/khái niệm chưa từng có trong demo robot — điều RT-1 không làm được. Bước ngoặt triết lý này — "tận dụng VLM pretrain internet-scale thay vì học lại từ đầu chỉ bằng dữ liệu robot" — chính là điều GR00T N1 kế thừa trực tiếp cho System 2 (dùng Eagle-2), dù GR00T N1 không đi theo cách sinh hành động dạng token rời rạc của RT-2 mà chuyển sang flow matching liên tục (hướng π₀). Vì vậy, hiểu đúng RT-1 → RT-2 là điều kiện tiên quyết để hiểu tại sao các VLA thế hệ sau, bao gồm GR00T N1, được thiết kế đúng như hiện tại.
