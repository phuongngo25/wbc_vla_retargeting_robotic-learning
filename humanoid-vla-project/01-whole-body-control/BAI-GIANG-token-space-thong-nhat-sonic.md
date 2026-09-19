# Bài giảng: Token space thống nhất của SONIC

*(Thuộc mảng: Whole-Body Control (WBC))*

## 🎯 Mục tiêu bài học

- Giải thích được ý tưởng gốc của Gato (Reed et al. 2022): token hoá mọi modality để một transformer duy nhất xử lý thống nhất.
- Mô tả chính xác 3 encoder chuyên biệt của SONIC và cách chúng ánh xạ vào cùng một latent space.
- Giải thích được cơ chế FSQ (Finite Scalar Quantization) khác VQ-VAE cổ điển ở điểm nào.
- Tính tay được một ví dụ đơn giản minh hoạ lượng tử hoá vô hướng từng chiều (per-dimension scalar quantization).
- Giải thích được chính xác vì sao "cùng hội tụ về một loại token" là cơ chế kỹ thuật cụ thể (không chỉ ý tưởng trừu tượng) cho phép một policy cấp thấp phục vụ nhiều nguồn lệnh.
- Nêu rõ ràng số liệu nào từ paper đã xác nhận được qua tra cứu và số liệu nào chưa xác nhận được đầy đủ.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Bài trước giải thích *tại sao* SONIC tách 2 lớp (low-level/high-level) và *lợi ích kinh tế* của việc đó. Nhưng có một câu hỏi kỹ thuật cụ thể chưa trả lời: nếu lớp cao có 3 nguồn khác nhau (người, kinematic planner, VLA) và mỗi nguồn tạo ra dữ liệu ở định dạng khác nhau (tư thế SMPL, chuyển động robot tham chiếu, lệnh hybrid), **bằng cơ chế kỹ thuật nào** chúng thực sự "nói cùng một ngôn ngữ" trước khi vào lớp thấp? Token space thống nhất (dùng FSQ) là câu trả lời kỹ thuật cụ thể — bài này đi sâu vào đúng cơ chế đó, tách biệt khỏi câu hỏi "tại sao tách lớp" đã trả lời ở bài trước.

## 🧠 Trực giác

### Góc nhìn 1: Cảng container hoá hàng hải — mọi loại hàng đều đóng vào container tiêu chuẩn trước khi vận chuyển

Trước khi có container hoá (containerization), mỗi loại hàng (gạo bao tải, máy móc, đồ nội thất) cần cách bốc dỡ/vận chuyển riêng, tàu chở hàng phải thiết kế khoang riêng cho từng loại. Container hoá giải quyết vấn đề này bằng cách: **bất kể hàng gì bên trong**, đóng vào một **container kích thước chuẩn** — cần cẩu, tàu, xe tải chỉ cần xử lý container chuẩn đó, không cần biết bên trong là gì. Token hoá (tokenization) trong Gato/SONIC làm đúng việc này với dữ liệu: bất kể dữ liệu gốc là ảnh, hành động, hay chuyển động, đóng gói vào "container" chuẩn (token rời rạc) trước khi đưa vào hệ thống xử lý chung.

**Giới hạn của loại suy này:** đóng hàng vào container không làm mất thông tin gì về hàng hoá (mở container ra vẫn còn nguyên hàng); token hoá **có mất mát thông tin** — lượng tử hoá (quantization) từ không gian liên tục sang rời rạc luôn làm tròn giá trị, mất đi độ chính xác vô hạn của số thực gốc (chi tiết ở mục Cơ chế/Ví dụ tính tay).

### Góc nhìn 2: Bảng chữ cái chung cho phép nhiều ngôn ngữ dùng chung một hệ thống gõ phím

Một số ngôn ngữ khác nhau (tiếng Việt, tiếng Anh, tiếng Pháp) đều có thể gõ được trên cùng một bàn phím QWERTY chuẩn, nhờ mỗi ngôn ngữ có quy ước riêng để **ánh xạ** âm/chữ của mình vào các phím có sẵn (dấu thanh tiếng Việt dùng tổ hợp phím, ký tự có dấu tiếng Pháp dùng phím chết — dead key). SONIC làm tương tự: mỗi "ngôn ngữ nguồn" (chuyển động người SMPL, chuyển động robot, lệnh hybrid) có một **encoder riêng** ánh xạ vào cùng một "bảng chữ cái chung" (universal token), nhưng bản thân bộ xử lý phía sau (policy cấp thấp) chỉ cần biết đọc bảng chữ cái chung đó.

**Giới hạn của loại suy này:** một người dùng bàn phím QWERTY vẫn "biết" mình đang gõ tiếng gì và có thể chuyển đổi ý thức; policy cấp thấp của SONIC **hoàn toàn không "biết"** và không cần biết token đến từ nguồn nào — đây là điểm mạnh hơn loại suy bàn phím, vì nó đạt được sự "trong suốt nguồn gốc" tuyệt đối, không chỉ là khả năng tương thích kỹ thuật.

## 📐 Định nghĩa chính xác

### Ý tưởng gốc: tokenization theo tinh thần Gato

**Reed et al. (DeepMind, 2022)** — Gato ([arXiv:2205.06175](https://arxiv.org/pdf/2205.06175)) — chứng minh: **bất kỳ modality nào** (ảnh, văn bản, hành động khớp robot, phần thưởng, quan sát cảm biến...) đều có thể được **rời rạc hoá thành một chuỗi token** theo một quy ước chung, để **MỘT kiến trúc transformer duy nhất** xử lý toàn bộ các modality đó như một bài toán "dự đoán token tiếp theo" (giống GPT dự đoán từ tiếp theo trong văn bản). Ảnh được chia patch rồi mã hoá thành token, hành động liên tục được rời rạc hoá thành token rời rạc, tất cả ghép vào chung MỘT chuỗi input cho transformer. Lợi ích: một khi mọi thứ đã là "token", kiến trúc mô hình **không cần biết** token đó đến từ ảnh, văn bản, hay tín hiệu điều khiển — cho phép **chia sẻ một backbone chung** giữa nhiều loại nhiệm vụ/input khác nhau về bản chất vật lý.

### Token space của SONIC — chuyên biệt hoá cho bài toán motion

SONIC áp dụng đúng tinh thần đó nhưng chuyên biệt hoá cho motion: chỉ cần token hoá **các loại lệnh chuyển động khác nhau** để chúng có chung một "ngôn ngữ" trước khi vào policy cấp thấp.

**Ba encoder chuyên biệt** (theo tra cứu trực tiếp bản HTML paper):

| Encoder | Input cụ thể |
|---|---|
| Robot encoder (`𝓔r`) | 10 khung hình TƯƠNG LAI của vị trí/vận tốc khớp robot, khoảng cách 0.1s (tức cửa sổ 1 giây) |
| Human encoder (`𝓔h`) | 10 khung hình TƯƠNG LAI của vị trí khớp SMPL 3D, khoảng cách 0.02s (tức cửa sổ 0.2 giây — độ phân giải thời gian cao hơn robot encoder) |
| Hybrid encoder (`𝓔m`) | Điểm mốc thưa (sparse keypoints) nửa thân trên (đầu, hai tay) ở khung hiện tại + 10 khung TƯƠNG LAI của chuyển động robot nửa thân dưới |

Cả ba encoder ánh xạ vào **cùng một không gian latent chung** qua MLP với kiến trúc `hidden=[2048,1024,512,512]` (theo tra cứu trực tiếp).

**FSQ (Finite Scalar Quantization):** không gian latent chung sau đó được **lượng tử hoá** thành token rời rạc — universal token là một vector `Dz` chiều với `Lz` mức lượng tử **mỗi chiều** (giá trị cụ thể `Dz, Lz` nằm trong bảng hyperparameter của paper, chưa trích xuất được số cụ thể qua bản HTML đã đọc — cần đối chiếu Table 3 của paper nếu cần con số chính xác tuyệt đối).

**Hai decoder:**
- Robot control decoder (`𝓓c`): chuyển token thành lệnh hành động khớp **29 chiều**.
- Robot motion decoder (`𝓓r`): tái tạo lại chuyển động robot tham chiếu, dùng làm giám sát phụ trợ (auxiliary supervision) trong lúc huấn luyện.

## ⚙️ Cơ chế hoạt động — từng bước

```text
┌────────────────────────────────────────────────────────────┐
│ Ba nguồn lệnh chuyển động khác nhau về ĐỊNH DẠNG:               │
│  - Robot reference motion (từ dữ liệu đã retarget, 02+03)      │
│  - Human motion SMPL (từ VR teleop)                             │
│  - Hybrid: sparse upper-body keypoints + robot lower-body       │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ ENCODER TƯƠNG ỨNG (𝓔r, 𝓔h, hoặc 𝓔m) — MLP [2048,1024,512,512]  │
│  → mỗi encoder ánh xạ định dạng RIÊNG của mình vào CÙNG MỘT     │
│    không gian latent liên tục chung (chia sẻ chiều/kích thước)  │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ FSQ QUANTIZER: latent liên tục → universal token RỜI RẠC        │
│  (mỗi chiều của vector latent được lượng tử hoá ĐỘC LẬP thành   │
│   một trong Lz mức rời rạc — không tra bảng codebook như VQ-VAE)│
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ UNIVERSAL MOTION TOKEN — TỪ ĐIỂM NÀY TRỞ ĐI, policy cấp thấp    │
│  KHÔNG PHÂN BIỆT được token đến từ encoder nào                  │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ Robot control decoder (𝓓c): token → action 29 chiều              │
│ (đồng thời: Robot motion decoder 𝓓r tái tạo motion tham chiếu    │
│  làm auxiliary supervision khi huấn luyện)                        │
└──────────────────────────────────────────────────────────────┘
```

**Vì sao FSQ, không phải VQ-VAE (Vector Quantization) cổ điển:** VQ-VAE học một **codebook** (bảng tra cứu) gồm N vector đại diện, rồi tìm vector gần nhất trong codebook cho mỗi latent — có nguy cơ "codebook collapse" (chỉ một số ít entry trong codebook được dùng, phần còn lại "chết"). FSQ thay vào đó lượng tử hoá **từng chiều vô hướng (scalar) một cách độc lập** vào một số mức cố định (`Lz` mức), không cần học codebook — đơn giản hơn, tránh được vấn đề collapse, và số lượng token khả dĩ được xác định trước bởi công thức `Lz^Dz` (số mức mũ số chiều) thay vì phụ thuộc vào việc học codebook có hội tụ tốt hay không.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ đơn giản hoá cực độ với Dz=2 chiều, Lz=5 mức mỗi chiều — số tự chọn để minh hoạ cơ chế FSQ, không phải giá trị Dz/Lz thật của SONIC vì paper không công bố rõ trong phần đã tra cứu)*

Giả sử không gian latent chỉ có **2 chiều** `(z1, z2)`, mỗi chiều được lượng tử hoá vào **5 mức** rời rạc, giả sử phạm vi mỗi chiều sau khi chuẩn hoá là `[-1, 1]`, chia đều thành 5 mức:

```text
Mức:        0      1      2      3      4
Giá trị:  -1.0   -0.5    0.0    0.5    1.0
```

Giả sử encoder tạo ra latent liên tục `(z1, z2) = (0.32, -0.71)`.

**Bước 1 — lượng tử hoá từng chiều độc lập (làm tròn về mức gần nhất):**

```text
z1 = 0.32 → gần nhất với mức 0.5 (mức 3) hơn mức 0.0 (mức 2)?
     |0.32 − 0.0| = 0.32
     |0.32 − 0.5| = 0.18
     → 0.18 < 0.32, chọn mức 3 (giá trị 0.5)

z2 = -0.71 → so sánh với mức -0.5 (mức 1) và mức -1.0 (mức 0):
     |-0.71 − (-0.5)| = 0.21
     |-0.71 − (-1.0)| = 0.29
     → 0.21 < 0.29, chọn mức 1 (giá trị -0.5)
```

**Bước 2 — universal token:**

```text
token = (mức_3, mức_1) = (3, 1)
  → có thể biểu diễn thành MỘT chỉ số nguyên duy nhất (nếu cần):
    index = mức_z1 × Lz + mức_z2 = 3×5 + 1 = 16
    (trong tổng số Lz^Dz = 5² = 25 token khả dĩ)
```

**Bước 3 — tính sai số lượng tử hoá (thông tin bị mất):**

```text
Δz1 = |0.32 − 0.5| = 0.18
Δz2 = |-0.71 − (-0.5)| = 0.21
```

**Ý nghĩa:** với `Dz=2, Lz=5`, toàn bộ không gian latent liên tục (vô hạn giá trị) bị nén thành đúng **25 token khả dĩ** (`5²`) — mỗi token mất một lượng thông tin nhất định (ở đây `Δz1=0.18, Δz2=0.21`). Nếu tăng `Lz` (ví dụ lên 20 mức thay vì 5), sai số lượng tử hoá trung bình giảm (mỗi mức "chi tiết" hơn) nhưng tổng số token tăng theo luỹ thừa (`20²=400` thay vì `25`) — đây chính là đánh đổi cốt lõi khi thiết kế `Dz, Lz` cho FSQ: độ chi tiết (giảm mất mát thông tin) đổi lấy kích thước không gian token (ảnh hưởng độ khó bài toán học "dự đoán token tiếp theo" của transformer phía sau).

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Tư thế người tường minh (explicit human pose) | Universal token (FSQ, SONIC) |
|---|---|---|
| Không gian hành động | Liên tục, nhiều chiều theo đúng cấu trúc SMPL/robot | Rời rạc, số chiều/mức cố định trước (Dz, Lz) |
| Cần biết nguồn gốc dữ liệu? | Có — policy phải xử lý riêng biệt tuỳ định dạng | Không — mọi nguồn đã quy về cùng 1 định dạng token |
| Dễ mở rộng thêm nguồn lệnh mới | Khó — cần thiết kế lại cách policy xử lý định dạng mới | Dễ hơn — chỉ cần thêm 1 encoder mới ánh xạ vào token space có sẵn |
| Mất mát thông tin | Không (giữ nguyên độ chính xác liên tục) | Có (do lượng tử hoá, như ví dụ tính tay) |
| Theo mô tả trong nguồn | — | Được báo cáo cho kết quả tốt hơn, kể cả tác vụ thao tác phức tạp *(xem lưu ý xác minh dưới)* |

**Lưu ý xác minh quan trọng:** `NOI-DUNG-CHI-TIET.md` (nguồn nội bộ) trích dẫn rằng paper báo cáo "biểu diễn hành động bằng token rời rạc cho kết quả tốt hơn hẳn so với dùng trực tiếp tư thế người tường minh làm không gian hành động". Khi tra cứu trực tiếp bản HTML đầy đủ của paper ([arXiv:2511.07820v1](https://arxiv.org/html/2511.07820v1)) trong lúc viết bài giảng này, phần nội dung đọc được **không chứa bảng số liệu ablation cụ thể** so sánh trực tiếp hai cách biểu diễn này — có thể do bảng số liệu đó nằm ở phần phụ lục/hình ảnh không được trích xuất đầy đủ qua công cụ tra cứu tự động. **Không nên coi đây là mâu thuẫn** (nguồn nội bộ vẫn được ưu tiên cho phần khái niệm cốt lõi), nhưng người đọc muốn trích dẫn con số ablation cụ thể nên tự kiểm tra trực tiếp Table/Figure trong bản PDF đầy đủ của paper.

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "FSQ giống hệt VQ-VAE, chỉ là tên gọi khác".** Vì sao sai: VQ-VAE tra cứu vector latent gần nhất trong một **codebook đã học** (có thể bị "codebook collapse" — nhiều entry không bao giờ được dùng); FSQ lượng tử hoá **từng chiều vô hướng độc lập** vào các mức cố định trước, không cần học codebook. Đây là khác biệt cơ chế cụ thể, không chỉ khác tên. **Hiểu đúng:** FSQ là "họ hàng gần" của VQ-VAE (cùng mục đích: nén latent liên tục thành token rời rạc) nhưng cơ chế lượng tử hoá khác hẳn — như bài trước đã lưu ý, đây là điểm cần phân biệt rõ khi đọc paper.
2. **Hiểu nhầm: "vì cả 3 encoder ánh xạ vào cùng không gian latent, chúng học được thông tin giống hệt nhau về chuyển động".** Vì sao sai: mỗi encoder nhận **input khác nhau về độ phân giải thời gian và nội dung** (robot encoder: 0.1s/khung, human encoder: 0.02s/khung — phân giải cao hơn 5 lần; hybrid encoder: chỉ có sparse keypoints nửa thân trên) — chúng học cách "dịch" các loại input khác nhau này về cùng MỘT không gian latent, nhưng bản thân dữ liệu đầu vào mỗi encoder xử lý là khác nhau về bản chất và độ chi tiết. **Hiểu đúng:** "cùng không gian latent" nghĩa là kết quả đầu ra (sau khi encode) có cùng cấu trúc chiều/định dạng để token hoá thống nhất, không có nghĩa là các encoder xử lý cùng loại thông tin đầu vào.

## 🏗️ Ví dụ minh hoạ trong dự án này

```text
Trong pipeline SONIC (bài trước), token space là "keo dán" kỹ thuật
cụ thể cho phép câu nói "một policy phục vụ nhiều nguồn" TRỞ THÀNH SỰ THẬT:

VR teleop → SMPL pose → Human encoder (𝓔h) ──┐
                                              ├──▶ FSQ ──▶ universal token
Kinematic planner → hybrid motion → Hybrid   │         (policy cấp thấp
  encoder (𝓔m) ────────────────────────────────┘          KHÔNG PHÂN BIỆT
                                                            nguồn gốc từ đây)
Dữ liệu retarget (02) → robot motion →
  Robot encoder (𝓔r) — dùng để HUẤN LUYỆN
  policy cấp thấp ban đầu (trước khi triển khai
  cho teleop/VLA)
```

Đây chính là lý do bài giảng trước gọi token space là "giao diện chung" — bài này cho thấy giao diện đó không phải một khái niệm trừu tượng, mà là một cơ chế toán học cụ thể (encoder MLP + FSQ) có thể cài đặt, huấn luyện, và kiểm tra được.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **Gato (2022) vẫn là nền tảng tư tưởng được trích dẫn rộng rãi cho hướng "tokenize mọi thứ".** Dù bản thân Gato không chuyên về motion/robot control, ảnh hưởng của nó lan rộng sang nhiều lĩnh vực robot học — SONIC là một ví dụ cụ thể hoá tư tưởng đó cho đúng bài toán motion tracking đa nguồn.
2. **FSQ là kỹ thuật tương đối mới trong hệ sinh thái quantization, được SONIC áp dụng cho miền chuyển động robot.** Việc chọn FSQ thay vì VQ-VAE cổ điển phản ánh xu hướng chung trong nghiên cứu generative model 2024-2026: ưu tiên các kỹ thuật lượng tử hoá đơn giản hơn, ổn định huấn luyện hơn (tránh codebook collapse) — dù bài giảng này chưa tìm được so sánh định lượng trực tiếp FSQ vs VQ-VAE trong đúng ngữ cảnh motion control của SONIC qua tra cứu hiện tại.
3. **Thận trọng cần thiết về số liệu chưa xác minh đầy đủ:** đúng tinh thần đã áp dụng nhất quán ở các bài giảng UMR và SONIC decoupled WBC — giá trị cụ thể `Dz` (số chiều token) và `Lz` (số mức lượng tử) của SONIC, cũng như bảng ablation so sánh token-based vs explicit-pose action space, **chưa được xác nhận đầy đủ** qua các lần tra cứu WebFetch đã thực hiện (nằm trong bảng hyperparameter/phụ lục chưa trích xuất được). Người đọc cần số liệu chính xác tuyệt đối nên tra cứu trực tiếp Table 3 và phần ablation của bản PDF đầy đủ.

## ❓ Câu hỏi tự kiểm tra

1. Ý tưởng cốt lõi của Gato mà SONIC kế thừa là gì?
   <details><summary>Gợi ý đáp án</summary>Bất kỳ modality nào (ảnh, hành động, chuyển động...) đều có thể rời rạc hoá thành chuỗi token theo một quy ước chung, để một kiến trúc xử lý thống nhất mà không cần biết token đến từ modality nào — cho phép chia sẻ backbone chung.</details>
2. Ba encoder của SONIC nhận input gì, và encoder nào có độ phân giải thời gian cao nhất?
   <details><summary>Gợi ý đáp án</summary>Robot encoder: 10 khung tương lai vị trí/vận tốc khớp robot (0.1s/khung). Human encoder: 10 khung tương lai vị trí khớp SMPL 3D (0.02s/khung — cao nhất, gấp 5 lần robot encoder). Hybrid encoder: sparse upper-body keypoints hiện tại + 10 khung tương lai robot lower-body.</details>
3. Trong ví dụ tính tay, vì sao tăng `Lz` từ 5 lên 20 (giữ `Dz=2`) làm giảm sai số lượng tử hoá nhưng tăng mạnh tổng số token khả dĩ?
   <details><summary>Gợi ý đáp án</summary>Sai số giảm vì mỗi mức "chi tiết" hơn (khoảng cách giữa 2 mức liên tiếp nhỏ hơn); tổng số token tăng theo luỹ thừa vì công thức là `Lz^Dz` — từ `5²=25` lên `20²=400`, tăng 16 lần dù Lz chỉ tăng 4 lần.</details>
4. FSQ khác VQ-VAE cổ điển ở cơ chế nào cụ thể?
   <details><summary>Gợi ý đáp án</summary>VQ-VAE học một codebook (bảng vector đại diện) rồi tra cứu vector gần nhất, có nguy cơ codebook collapse; FSQ lượng tử hoá từng chiều vô hướng độc lập vào các mức cố định trước, không cần học codebook.</details>
5. Vì sao bài giảng này không khẳng định chắc chắn con số Dz, Lz cụ thể của SONIC, dù đã tra cứu WebFetch trực tiếp paper?
   <details><summary>Gợi ý đáp án</summary>Vì các con số đó nằm trong bảng hyperparameter (Table 3)/phụ lục của paper mà công cụ tra cứu WebFetch đã dùng không trích xuất được đầy đủ trong lần đọc — đúng nguyên tắc "không suy đoán, nêu rõ mức độ chưa xác minh" thay vì bịa ra con số cụ thể không có căn cứ.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** với `Dz=3` chiều, `Lz=4` mức mỗi chiều (giá trị mức: -1.0, -0.33, 0.33, 1.0), latent liên tục `(z1,z2,z3)=(0.5, -0.9, 0.1)` — lượng tử hoá từng chiều về mức gần nhất, tính token kết quả và tổng số token khả dĩ (`Lz^Dz`).
2. **Đọc paper thật, đối chiếu cẩn trọng:** mở trực tiếp bản PDF đầy đủ [arXiv:2511.07820](https://arxiv.org/abs/2511.07820) (không chỉ bản HTML tóm tắt), tìm Table 3 (hoặc bảng hyperparameter tương ứng) — ghi lại chính xác giá trị `Dz`, `Lz` thật của SONIC, và tìm phần ablation so sánh token-based vs explicit human pose action space nếu có — xác nhận hoặc phủ định trực tiếp tuyên bố "token rời rạc tốt hơn hẳn" đã nêu trong nguồn nội bộ của dự án.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

Token space thống nhất của SONIC hiện thực hoá tư tưởng "tokenize mọi modality" của Gato (2022) chuyên biệt cho bài toán motion: ba encoder (robot, human-SMPL, hybrid) với input khác nhau về độ phân giải thời gian và nội dung đều ánh xạ vào cùng một không gian latent qua MLP chung, sau đó được lượng tử hoá bằng FSQ (lượng tử hoá từng chiều vô hướng độc lập, khác VQ-VAE ở việc không cần học codebook) thành universal token — như ví dụ tính tay minh hoạ, đây là cơ chế toán học cụ thể (không chỉ ý tưởng trừu tượng) khiến policy cấp thấp thực sự không phân biệt được token đến từ nguồn nào, đánh đổi một lượng thông tin bị mất do lượng tử hoá để đạt được khả năng dùng chung một policy cho nhiều nguồn lệnh. Một số chi tiết định lượng cụ thể (số chiều/mức token chính xác, bảng ablation so sánh với action space tường minh) chưa xác nhận đầy đủ được qua tra cứu trực tiếp bản HTML của paper trong lúc viết bài này — người cần trích dẫn chính xác tuyệt đối nên đối chiếu trực tiếp bản PDF đầy đủ.
