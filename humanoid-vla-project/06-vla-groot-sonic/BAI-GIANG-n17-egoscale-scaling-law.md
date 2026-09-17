# Bài giảng: N1.7 — EgoScale và scaling law cho dexterity

*(Thuộc mảng: Vision-Language-Action: GR00T N1 → N1.7 & SONIC)*

## 🎯 Mục tiêu bài học

- Giải thích được **EgoScale khác dữ liệu teleoperation robot ở đâu** — nguồn gốc, cách thu thập, và vì sao sự khác biệt này quan trọng về mặt kinh tế/khả năng mở rộng.
- Diễn giải đúng ý nghĩa con số **"1k → 20k giờ, task completion tăng hơn gấp đôi"** — biết nó là headline đơn giản hoá của một quan hệ toán học chính xác hơn (log-linear trong validation loss), không phải một quy luật tuyến tính đơn giản.
- Viết lại được công thức scaling law dạng log-linear và giải thích từng ký hiệu (L, D, hệ số góc).
- Phân biệt được **"scaling law cho dexterity"** với scaling law tổng quát của LLM (Kaplan/Chinchilla) — biết phạm vi áp dụng hẹp hơn nhiều (một loại kỹ năng: thao tác khéo léo, không phải mọi tác vụ robot).
- Nêu được đúng kiến trúc N1.7 ở mức "những gì đổi so với N1 gốc" (VLM backbone, số layer DiT) mà không cần nhắc lại toàn bộ cơ chế flow matching (đã có bài riêng).
- Nhận diện được ít nhất 2 hiểu lầm phổ biến khi đọc tin tức/blog tóm tắt về EgoScale.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Trong toàn bộ dòng GR00T (N1 → N1.5 → N1.6 → N1.7), nút thắt cổ chai lớn nhất luôn là **dữ liệu**: robot học kỹ năng thao tác khéo léo (dexterity — cầm, xoay, gấp, lắp ráp các vật thể nhỏ, tinh vi) cần rất nhiều demo, nhưng demo robot thật (teleoperation — người vận hành điều khiển robot từ xa để tạo dữ liệu) đắt, chậm, và bị giới hạn bởi số robot + số người vận hành có sẵn. N1 gốc và N1.6 đều vẫn phụ thuộc phần lớn vào loại dữ liệu này (N1.6 được mô tả là dùng "few thousand hours" dữ liệu teleoperation).

N1.7 là bước ngoặt vì nó lật ngược giả định đó: thay vì "muốn robot khéo léo hơn thì phải thu thập thêm demo robot", NVIDIA chứng minh (và công bố hẳn một paper riêng, không chỉ blog, xem mục SOTA) rằng **quay video người thật làm việc bằng camera đeo trên người (egocentric)** — một loại dữ liệu rẻ hơn, dễ mở rộng hơn nhiều lần — cũng cải thiện độ khéo léo của robot theo một quy luật có thể đo được và dự đoán được. Đây là lý do bài này xứng đáng có một bài giảng riêng: nó không chỉ là "một bản cập nhật kiến trúc" mà là một **phát hiện thực nghiệm về cách dữ liệu tạo ra năng lực**, tương tự tinh thần scaling law đã thay đổi cách người ta nghĩ về LLM một thập kỷ trước, nay lần đầu áp dụng cho robot học kỹ năng khéo léo.

## 🧠 Trực giác

### Góc nhìn 1: "Học nghề bằng cách xem video hướng dẫn, không cần thợ cả cầm tay chỉ việc"

Hãy tưởng tượng bạn muốn dạy một người học việc cách làm bánh, may vá, hay lắp ráp linh kiện tinh vi. Có hai cách: (a) một thợ cả đứng bên cạnh, cầm tay chỉ việc từng động tác cho người học (tương tự teleoperation — chuyên gia trực tiếp "lái" robot làm mẫu), rất chính xác nhưng tốn thời gian một-thợ-một-học-viên; hoặc (b) cho người học xem hàng chục nghìn giờ video người khác làm những việc tương tự ở nhà máy, cửa hàng, bệnh viện, nhà bếp — không ai cầm tay chỉ việc trực tiếp, nhưng khối lượng và sự đa dạng của những gì được thấy giúp người học nắm được các mẫu hình chuyển động chung.

- **Đúng ở đâu:** nắm đúng ý tưởng cốt lõi của EgoScale — dữ liệu là **quan sát gián tiếp** (video người làm việc thật, không phải ai "dạy robot" trực tiếp), và số lượng/đa dạng lớn hơn nhiều so với cách (a).
- **Giới hạn quan trọng:** robot **không "xem" và "bắt chước" theo đúng nghĩa con người học kỹ năng** (không có cơ chế nhận thức, không có ý định, không tự suy luận "động tác này dùng để làm gì" như con người). Về mặt kỹ thuật, đây chỉ là **dữ liệu huấn luyện gián tiếp** cho một mạng neural — video egocentric cung cấp tín hiệu thị giác + (trong một số pipeline) nhãn hành động ước lượng từ chuyển động tay, được đưa vào quá trình tối ưu tham số bằng gradient descent, không phải một quá trình "học tập" giống con người. Nói "robot học bằng cách xem" là một cách nói tắt dễ hiểu, nhưng cơ chế thật sự là thống kê/tối ưu hoá trên dữ liệu lớn.

### Góc nhìn 2: "Scaling law giống đường cong loss-vs-compute của LLM (Kaplan/Chinchilla)"

Trong giới huấn luyện LLM, một trong những phát hiện quan trọng nhất thập kỷ qua là: khi tăng dữ liệu/tham số/compute theo đúng tỷ lệ, loss (hàm mất mát) giảm theo một đường cong **có thể dự đoán trước**, không ngẫu nhiên — đây là các "scaling law" nổi tiếng của Kaplan et al. (2020) và Hoffmann et al. (Chinchilla, 2022). EgoScale báo cáo một hiện tượng cùng *tinh thần*: tăng số giờ video egocentric người dùng để pretrain, validation loss của model giảm theo quan hệ **log-linear** (tuyến tính theo logarit của lượng dữ liệu) rất khớp với số liệu thực nghiệm (R² ≈ 0.9983 — gần như khớp hoàn hảo).

- **Đúng ở đâu:** cùng dạng toán học (log-linear trong loss theo log của lượng dữ liệu), cùng ý nghĩa thực tiễn (dự đoán được: biết thêm bao nhiêu dữ liệu sẽ cải thiện bao nhiêu, thay vì phải thử rồi mới biết).
- **Giới hạn quan trọng:** đây **không phải một quy luật tổng quát cho mọi loại tác vụ robot** như scaling law của LLM (vốn được quan sát trên rất nhiều loại benchmark ngôn ngữ khác nhau). Quan hệ log-linear này được đo cụ thể cho **một loại kỹ năng: thao tác khéo léo (dexterous manipulation) bằng tay robot nhiều bậc tự do (22 DoF)**, trên một tập task cụ thể trong một paper cụ thể. Không có bằng chứng (tại thời điểm soạn bài) rằng cùng quy luật áp dụng y hệt cho việc robot đi lại (locomotion), điều hướng (navigation), hay các tác vụ hoàn toàn khác. Đây chính là sai lầm phổ biến số 1 ở mục cảnh báo bên dưới.

## 📐 Định nghĩa chính xác

**EgoScale** (theo blog chính thức HuggingFace và paper arXiv riêng, xem nguồn ở mục SOTA):

> Tập dữ liệu pretraining gồm **20,854 giờ video egocentric của con người** (quay từ góc nhìn thứ nhất — camera đeo trên đầu/cổ tay, kèm hand-tracking), trải rộng **hơn 20 loại tác vụ** (manufacturing, retail, healthcare, home environments), được gán nhãn hành động (action-labeled) để có thể dùng làm tín hiệu huấn luyện giám sát.

Điểm mấu chốt cần khắc sâu: đây **không phải dữ liệu teleoperation robot** (người vận hành điều khiển robot từ xa, robot thật chuyển động) mà là **video người thật làm việc thật, không có robot nào tham gia lúc thu thập**. Quy mô 20.854 giờ được mô tả là "hơn 20 lần lớn hơn các nỗ lực trước đó" (more than 20× larger than prior efforts).

**"Scaling law cho dexterity"** trong ngữ cảnh này được định nghĩa bằng hai cách diễn đạt — cần phân biệt rõ, vì đây là chỗ các nguồn thứ cấp hay lẫn lộn:

1. **Cách diễn đạt của blog HuggingFace (headline, đơn giản hoá):** "more human egocentric data produces predictable, consistent improvements in dexterous manipulation capability — going from 1k to 20k hours more than doubles average task completion." Tức là mô tả bằng ngôn ngữ tự nhiên: X = giờ dữ liệu egocentric, Y = tỷ lệ hoàn thành tác vụ trung bình (average task completion rate), và quan hệ giữa X và Y là "dự đoán được, nhất quán" (predictable, consistent).
2. **Cách diễn đạt chính xác hơn của paper arXiv EgoScale** (arXiv:2602.16710, xem trích dẫn đầy đủ ở mục SOTA): quan hệ log-linear được đo trực tiếp là giữa **lượng dữ liệu pretraining (D, giờ)** và **validation loss (L)** của model, dạng:

```
L = 0.024 − 0.003 · ln(D)
```

với hệ số xác định R² ≈ 0.9983 (khớp gần như hoàn hảo với dữ liệu thực nghiệm). Paper báo cáo rằng validation loss này **tương quan mạnh (correlate) với hiệu năng downstream trên robot thật** — tức loss thấp hơn dự đoán được task completion cao hơn, chứ quan hệ log-linear không phải được đo trực tiếp trên trục "task completion" mà trên trục loss trước, rồi loss mới liên hệ với hiệu năng thực tế. Đây là lý do bài học này nói "scaling law đo trên loss, còn con số '>2x task completion' là một điểm dữ liệu cụ thể minh hoạ hệ quả của quy luật đó" — hai thứ liên quan nhưng không phải cùng một phép đo.

Con số cụ thể được paper báo cáo: **average task completion tăng đơn điệu (monotonically) từ 0.30 ở 1.000 giờ lên 0.71 ở 20.000 giờ** — đây chính là nguồn gốc câu "hơn gấp đôi" của blog (0.71 / 0.30 ≈ 2.37 lần, đúng là hơn gấp đôi).

## ⚙️ Cơ chế hoạt động — từng bước

Sơ đồ luồng tổng thể (dữ liệu → kiến trúc → khả năng):

```
┌──────────────────────────────────────────────────────────────────────┐
│  NGUỒN DỮ LIỆU                                                        │
│                                                                        │
│  Video egocentric người thật (ego camera / wrist camera / hand        │
│  tracking) khi làm việc: lắp ráp (manufacturing), xếp hàng (retail),  │
│  chăm sóc (healthcare), việc nhà (home)                                │
│                        │                                              │
│                        ▼                                              │
│  ═══════════ EgoScale dataset: 20,854 giờ, 20+ loại tác vụ ═══════   │
│                        │                                              │
└────────────────────────┼──────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────────────┐
│  STAGE I — HUMAN PRETRAINING (theo paper EgoScale)                    │
│  - Huấn luyện trên toàn bộ 20,854h video người                        │
│  - Batch size 8,192, learning rate 5×10⁻⁵                              │
│  - "Fully unfreezing every parameter" — mọi tham số đều được cập nhật │
│    (không đóng băng phần nào)                                          │
└────────────────────────┼──────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────────────┐
│  STAGE II — ALIGNED MID-TRAINING (căn chỉnh sang không gian robot)    │
│  - Fine-tune trên 54 giờ dữ liệu "human-robot play data" đã căn chỉnh  │
│  - Đóng băng (freeze) VLM backbone                                     │
│  - Chỉ cập nhật: vision encoder + DiT action expert                    │
│    → đây là bước "dịch" tri thức đã học từ người sang không gian      │
│      hành động thật của robot, với rất ít dữ liệu robot (54h, so với   │
│      20,854h dữ liệu người — chênh lệch ~386 lần)                      │
└────────────────────────┼──────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────────────┐
│  KIẾN TRÚC N1.7 (GR00T-N1.7-3B, 3 tỷ tham số)                          │
│                                                                        │
│  System 2 (VLM, "suy nghĩ chậm")                                       │
│    Backbone: Cosmos-Reason2-2B  (thay cho Eagle-2 của N1 gốc)         │
│    → xử lý ảnh + câu lệnh ngôn ngữ → sinh token hành động cấp cao      │
│                        │  (cross-attention, xem bài "dual-system")     │
│                        ▼                                              │
│  System 1 (DiT — Diffusion Transformer, "phản xạ nhanh")              │
│    32 layer (xác nhận trực tiếp từ config.json — xem Định nghĩa)      │
│    Huấn luyện theo mục tiêu flow matching (xem bài riêng)              │
│    + lightweight embodiment-conditioned MLP adapter ở input/output    │
│      (xác nhận qua paper EgoScale — giúp 1 action head dùng chung     │
│      được cho nhiều loại robot/embodiment khác nhau)                  │
│                        │                                              │
│                        ▼                                              │
│              Action chunk liên tục → khớp/tay robot 22 DoF             │
└──────────────────────────────────────────────────────────────────────┘
                         │
                         ▼
        Khả năng dexterity đo được tăng theo thang dữ liệu
        (0.30 completion @ 1k giờ  →  0.71 completion @ 20k giờ)
```

Lưu ý quan trọng: cơ chế **flow matching** (làm sao DiT sinh action chunk liên tục từ nhiễu) và cơ chế **dual-system** (System 1/System 2 phối hợp qua cross-attention) đã có bài giảng riêng — `BAI-GIANG-groot-n1-dual-system.md` và `BAI-GIANG-flow-matching.md`. Bài này chỉ nhắc tên để định vị N1.7 trong bức tranh lớn, không giảng lại chi tiết cơ chế đó.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Các phép tính dưới đây dùng số liệu đã trích dẫn ở trên; phần suy luận thêm ngoài số liệu gốc được ghi rõ là minh hoạ tự làm, không phải số chính thức từ paper/blog.)*

**(a) Tỷ lệ tăng dữ liệu vs. tỷ lệ tăng task completion**

- Dữ liệu: tăng từ 1.000 giờ lên 20.000 giờ → hệ số tăng = 20.000 / 1.000 = **20 lần**.
- Task completion: tăng từ 0.30 lên 0.71 → hệ số tăng = 0.71 / 0.30 ≈ **2.37 lần** ("hơn gấp đôi" — khớp với câu blog).

Vậy: dữ liệu tăng **20 lần**, nhưng năng lực chỉ tăng **~2.37 lần** — đây chính xác là bản chất của quan hệ **log-linear** (không phải tuyến tính 1:1): tăng dữ liệu theo cấp số nhân chỉ tạo ra tăng năng lực theo cấp số cộng (hoặc chậm hơn tuyến tính). Nếu quan hệ là tuyến tính 1:1, tăng 20 lần dữ liệu sẽ phải cho ra tăng completion 20 lần (tức completion phải đạt ~6.0 — vô lý vì completion tối đa là 1.0). Việc quan sát "chỉ" tăng 2.37 lần dù dữ liệu tăng 20 lần thực ra **xác nhận** dạng log-linear, không phải bằng chứng cho thấy dữ liệu "kém hiệu quả" — đây là bản chất toán học của log.

**(b) Thử suy luận log-linear đơn giản (minh hoạ tự làm, không phải công thức chính thức của paper cho trục task completion)**

Lấy 2 điểm dữ liệu đã biết: D₁ = 1.000h → completion 0.30; D₂ = 20.000h → completion 0.71.

```
ln(D₂/D₁) = ln(20.000/1.000) = ln(20) ≈ 3.00
Δ completion = 0.71 − 0.30 = 0.41
```

Nếu coi gần đúng quan hệ completion-vs-ln(D) cũng xấp xỉ tuyến tính trong khoảng này (giả định đơn giản hoá, KHÔNG phải công thức được paper xác nhận — paper chỉ xác nhận log-linear cho trục loss, không phải trực tiếp cho trục completion):

```
độ dốc ≈ Δcompletion / Δln(D) = 0.41 / 3.00 ≈ 0.137 (điểm % completion / đơn vị ln)
```

Suy ra: mỗi lần dữ liệu tăng **10 lần** (ln(10) ≈ 2.303), completion tăng thêm xấp xỉ `0.137 × 2.303 ≈ 0.315`, tức khoảng **31–32 điểm phần trăm** mỗi 10x dữ liệu — đây là con số **ước lượng minh hoạ tự suy ra** để cảm nhận độ dốc, không phải trích dẫn trực tiếp từ paper, và chỉ hợp lý gần đúng trong khoảng 1k–20k giờ đã đo (ngoại suy ra xa hơn — ví dụ 200k giờ — có thể sai lệch lớn vì đường cong thật có thể bão hoà/saturate ở gần completion = 1.0, không thể tăng vô hạn).

**(c) So sánh quy mô EgoScale với N1.6**

N1.6 được mô tả (nguồn thứ cấp, không có số chính xác) là dùng "few thousand hours" dữ liệu teleoperation. Nếu giả định minh hoạ N1.6 ≈ 3.000 giờ (con số ví dụ, KHÔNG phải số chính thức — "few thousand" có thể là 2.000–5.000 hoặc hơn):

```
20.854 / 3.000 ≈ 6.95 ≈ ~7 lần
```

Vậy EgoScale lớn hơn khoảng **~7 lần** so với ước tính N1.6 (nếu N1.6 thực sự khoảng 3.000 giờ) — nhưng vì "few thousand hours" không phải số chính xác được công bố, con số "~7 lần" chỉ là ước tính minh hoạ, không nên trích dẫn như một sự thật đã xác nhận.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | N1.6 (trước EgoScale) | N1.7 (EgoScale) |
|---|---|---|
| Nguồn dữ liệu chính | Teleoperation robot thật (người vận hành điều khiển robot trực tiếp) | Video egocentric người thật, không có robot khi thu thập |
| Quy mô dữ liệu | "Few thousand hours" (số chính xác chưa công bố) | 20.854 giờ (số chính xác, đã công bố) |
| VLM backbone (System 2) | Eagle-2 (kế thừa từ N1 gốc, theo mô tả nguồn thứ cấp) | Cosmos-Reason2-2B |
| Action head (System 1) | DiT + flow matching (không đổi công thức nền tảng) | DiT 32-layer + flow matching, có thêm embodiment-conditioned MLP adapter |
| Phát hiện khoa học mới | Không có scaling law được công bố riêng | Scaling law log-linear đầu tiên cho dexterity, có paper arXiv riêng xác nhận |
| Có paper kỹ thuật riêng? | Không (chỉ changelog/model card) | Có — arXiv:2602.16710 (paper EgoScale, 18/2/2026), tách biệt với blog giới thiệu N1.7 |

**So với các hướng tiếp cận cùng thời (2025–2026) khác cũng dùng dữ liệu người/internet-scale cho robot:**

| Hệ thống | Triết lý dữ liệu | Khác EgoScale/N1.7 ở đâu |
|---|---|---|
| **Physical Intelligence π0.5** | Dữ liệu robot thật thu thập trực tiếp từ nhiều nền tảng robot khác nhau (đa robot, không dùng video người làm nguồn chính). Đạt 42.3% average task progress trong đánh giá zero-shot. | Triết lý ngược lại: cho rằng lực vật lý và chuyển động thật không thể thay thế bằng dữ liệu tổng hợp/internet — ưu tiên dữ liệu robot hơn là video người. |
| **Google Gemini Robotics 1.5** | Xây trên nền VLM Gemini đã pretrain internet-scale (ảnh+text+video web nói chung, không phải video egocentric chuyên biệt cho thao tác), bổ sung dữ liệu robot đa nền tảng. | Giả định tri thức ngôn ngữ-thị giác internet-scale đã ngầm chứa hiểu biết vật lý, khác với việc EgoScale chủ động thu thập một tập video egocentric chuyên biệt cho thao tác tay. |
| **Figure Helix / Project Go-Big** | Huấn luyện Helix **hoàn toàn (100%)** trên video egocentric người, thu thập tại các căn hộ thật (hợp tác với Brookfield, đối tác sở hữu hơn 100.000 căn hộ), **hoàn toàn không có demo robot nào** cho khả năng mới (điều hướng theo lệnh ngôn ngữ như "đi tới tủ lạnh"). | Cùng tinh thần "học từ video người" như EgoScale, nhưng Figure áp dụng cho bài toán **điều hướng (navigation)**, không phải thao tác khéo léo tay (dexterous manipulation) — và bỏ hẳn giai đoạn dữ liệu robot, khác với N1.7 vẫn có Stage II mid-training 54 giờ trên dữ liệu robot đã căn chỉnh. |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "scaling law cho dexterity" là quy luật áp dụng cho MỌI loại tác vụ robot (đi lại, điều hướng, thao tác thô...).**
   Vì sao sai: quan hệ log-linear được đo và xác nhận (R² ≈ 0.9983) cụ thể cho **thao tác khéo léo bằng tay robot 22 bậc tự do**, trên một tập nhiệm vụ cụ thể trong paper EgoScale. Không có bằng chứng công bố rằng cùng hệ số góc, hoặc thậm chí cùng dạng log-linear, áp dụng cho locomotion (đi lại) hay navigation (điều hướng, như Figure Helix đang làm theo hướng khác).
   Hiểu đúng: đây là một scaling law **hẹp, cho một năng lực cụ thể** (dexterity), tương tự tinh thần nhưng không đồng nhất với scaling law tổng quát của LLM.

2. **Hiểu nhầm: EgoScale là dữ liệu teleoperation robot "trá hình" (tức vẫn có robot tham gia, chỉ gọi tên khác).**
   Vì sao sai: theo định nghĩa chính thức, EgoScale là **video người thật làm việc thật, không có robot nào tham gia lúc thu thập** (ego camera/wrist camera/hand tracking gắn trên người, không gắn trên robot). Đây là điểm phân biệt cốt lõi với mọi thế hệ N1.x trước, vốn phụ thuộc phần lớn vào teleoperation.
   Hiểu đúng: có một giai đoạn huấn luyện thứ hai (Stage II, mid-training) dùng dữ liệu "human-robot play data" đã căn chỉnh (chỉ 54 giờ) để dịch tri thức đã học từ video người sang không gian hành động robot thật — nhưng đây là bước rất nhỏ so với 20.854 giờ pretraining chính, không phải nguồn dữ liệu chính.

3. **Hiểu nhầm: con số "hơn gấp đôi task completion" và công thức log-linear là cùng một phép đo.**
   Vì sao sai: như đã phân tích ở mục Định nghĩa, quan hệ log-linear (L = 0.024 − 0.003·ln(D), R²=0.9983) được đo trực tiếp trên **validation loss**, không phải trực tiếp trên task completion. Con số 0.30 → 0.71 là điểm dữ liệu thực nghiệm minh hoạ hệ quả, không phải chính công thức log-linear áp cho completion.
   Hiểu đúng: cần đọc kỹ xem một con số cụ thể được trích từ trục nào (loss hay completion) trước khi dùng nó để so sánh hoặc trích dẫn ở nơi khác.

## 🏗️ Ví dụ minh hoạ trong dự án này

Trong pipeline tổng thể của dự án, câu hỏi "dữ liệu egocentric video người lấy từ đâu, đi vào hệ thống thế nào" liên quan trực tiếp tới hai mảng khác:

- **`03-human-motion-datasets/`** — nơi dự án đã tìm hiểu các bộ dữ liệu chuyển động người (AMASS, LAFAN1...) dùng để retarget sang robot. EgoScale khác về bản chất: các bộ dữ liệu ở `03-human-motion-datasets/` chủ yếu là **motion capture 3D toàn thân** (dùng cho retargeting kỹ năng đi lại/toàn thân, xem `02-motion-retargeting/`), trong khi EgoScale là **video RGB góc nhìn thứ nhất tập trung vào tay/thao tác cận cảnh** (dùng để pretrain dexterity, không phải để retarget toàn thân theo nghĩa cổ điển). Đây là hai loại "dữ liệu người" phục vụ hai mục đích khác nhau trong cùng một dự án tổng thể — dễ nhầm nếu chỉ nghe "dữ liệu con người" mà không phân biệt loại.
- **`08-real-robot-deployment/`** — nơi bàn về vòng lặp data flywheel (thu thập dữ liệu thực tế → cải thiện model → triển khai → thu thập thêm). Cơ chế Stage II "aligned mid-training" của EgoScale (chỉ 54 giờ dữ liệu human-robot play data để "dịch" tri thức đã pretrain sang robot thật) chính là một ví dụ cụ thể, thực tế của tư duy flywheel: phần lớn năng lực đến từ dữ liệu rẻ/dễ mở rộng (video người), chỉ cần một lượng rất nhỏ dữ liệu đắt (robot thật, 54h) để "chốt" lại năng lực đó vào đúng không gian hành động vật lý của robot.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **Đã có paper arXiv chính thức riêng cho EgoScale, công bố sau blog HuggingFace** — đây là câu trả lời trực tiếp cho câu hỏi "tính đến 2026 đã có paper riêng cho N1.7 chưa": có, dưới dạng paper tập trung vào chính phương pháp EgoScale (không phải một bài "N1.7 technical report" tổng quát). **"EgoScale: Scaling Dexterous Manipulation with Diverse Egocentric Human Data"**, tác giả Ruijie Zheng, Dantong Niu, Yuqi Xie, Jing Wang, Mengda Xu, Yunfan Jiang, Fernando Castañeda, Fengyuan Hu, You Liang Tan, Letian Fu, Trevor Darrell, Furong Huang, Yuke Zhu, Danfei Xu, Linxi Fan — [arXiv:2602.16710](https://arxiv.org/abs/2602.16710), công bố 18/2/2026 (tức là sau ngày blog N1.7 lên HuggingFace 17/4/2026 theo trình tự công bố... lưu ý: ngày arXiv 2602.16710 tương ứng số hiệu tháng 2/2026, sớm hơn blog HF tháng 4/2026 — có thể paper được nộp trước rồi blog giới thiệu sản phẩm ra sau, hoặc đây là cùng một công bố phối hợp; thứ tự chính xác giữa hai mốc này nên được xác minh lại nếu cần trích dẫn học thuật chính xác về "cái nào công bố trước"). Paper này chính là nguồn của công thức log-linear `L = 0.024 − 0.003·ln(D)` (R²≈0.9983) và kết quả **cải thiện trên 54% average success rate** so với baseline không pretrain, đo trên tay robot 22 bậc tự do (22 DoF) qua 5 tác vụ thao tác, cùng kết quả one-shot transfer gấp quần áo (shirt folding) đạt success rate 0.88.

2. **Số layer DiT đã được xác minh trực tiếp: 32 layer** — kiểm tra trực tiếp file `config.json` của checkpoint `nvidia/GR00T-N1.7-3B` trên HuggingFace (`https://huggingface.co/nvidia/GR00T-N1.7-3B/raw/main/config.json`), trường `diffusion_model_cfg.num_layers = 32`. Điều này giải quyết dứt khoát mâu thuẫn "32-layer hay 16-layer" từng thấy ở các nguồn thứ cấp trong tài liệu gốc của dự án: **32 là đúng**. Đáng chú ý, config này còn có trường `select_layer: 16` (chỉ định trích đặc trưng từ layer thứ 16 của backbone thị giác) — nhiều khả năng đây chính là nguồn gốc của con số "16-layer" bị lan truyền sai trong một số bài phân tích cộng đồng (nhầm lẫn giữa "layer được chọn để trích đặc trưng" và "tổng số layer của DiT"), nhưng đây là suy đoán của người soạn bài dựa trên cấu trúc config quan sát được, chưa xác nhận trực tiếp từ nguồn nào nói rõ đây là lý do gây nhầm lẫn.

3. **Embodiment-conditioned MLP adapter đã được xác nhận qua paper chính thức (arXiv:2602.16710), không còn là "cần xác minh thêm" như trong tài liệu nền của dự án.** Paper mô tả kiến trúc dùng "lightweight embodiment-conditioned MLP adapters at the input and output interfaces" của DiT action expert — xác nhận độc lập với mô tả từng thấy ở nguồn thứ cấp (DeepWiki cộng đồng), nay có nguồn sơ cấp hỗ trợ.

4. **Phản ứng cộng đồng và so sánh cùng thời điểm:** giới quan sát ngành robot học (ví dụ bài phân tích "Physical AI has Scaling Laws now" và các bài tổng hợp so sánh kiến trúc "Three Teams, Three Robot Brains" đối chiếu GR00T/EgoScale với Gemini Robotics và dòng π của Physical Intelligence) đều nhấn mạnh cùng một điểm: 2025–2026 là giai đoạn các phòng lab lớn (NVIDIA, Physical Intelligence, Google DeepMind, Figure) đồng loạt tìm cách giải quyết bài toán khan hiếm dữ liệu robot bằng các chiến lược khác nhau — dùng video người (NVIDIA EgoScale, Figure Helix/Project Go-Big), dùng dữ liệu robot đa nền tảng (Physical Intelligence π0.5), hoặc dựa vào tri thức internet-scale sẵn có của VLM nền (Google Gemini Robotics). Không có sự đồng thuận về cách tiếp cận nào "thắng" — mỗi hướng có đánh đổi khác nhau về chi phí, độ chính xác vật lý, và khả năng mở rộng.

## ❓ Câu hỏi tự kiểm tra

1. EgoScale khác dữ liệu teleoperation robot ở điểm nào quan trọng nhất?
<details><summary>Gợi ý đáp án</summary>EgoScale là video người thật làm việc thật, quay bằng camera đeo trên người (ego/wrist camera, hand tracking) — không có robot nào tham gia lúc thu thập. Teleoperation là người vận hành trực tiếp điều khiển robot thật để tạo demo, robot có mặt và chuyển động thật ngay từ đầu.</details>

2. Viết lại công thức log-linear scaling law của EgoScale, giải thích từng ký hiệu.
<details><summary>Gợi ý đáp án</summary>L = 0.024 − 0.003·ln(D), trong đó L là validation loss, D là số giờ dữ liệu pretraining egocentric, ln là logarit tự nhiên. Hệ số góc âm (−0.003) nghĩa là loss giảm khi D tăng; R²≈0.9983 nghĩa là công thức khớp gần như hoàn hảo với dữ liệu thực nghiệm đo được.</details>

3. Nếu dữ liệu tăng từ 1.000 lên 20.000 giờ (tăng 20 lần), vì sao task completion "chỉ" tăng ~2.37 lần chứ không phải 20 lần — điều này có phải dấu hiệu dữ liệu egocentric kém hiệu quả không?
<details><summary>Gợi ý đáp án</summary>Không phải dấu hiệu kém hiệu quả — đây chính xác là bản chất của quan hệ log-linear: tăng dữ liệu theo cấp số nhân (×20) chỉ tạo ra tăng năng lực theo một lượng nhỏ hơn nhiều (log của 20), đúng như mọi scaling law dạng log-linear đã biết (kể cả ở LLM). Nếu quan hệ là tuyến tính 1:1 thật sự, con số sẽ vô lý (completion vượt quá 1.0).</details>

4. "Scaling law cho dexterity" của N1.7/EgoScale có áp dụng được cho bài toán locomotion (đi lại) của humanoid không? Vì sao?
<details><summary>Gợi ý đáp án</summary>Không có bằng chứng công bố cho việc đó. Quan hệ log-linear được đo và xác nhận cụ thể cho thao tác khéo léo (dexterous manipulation) bằng tay robot 22 DoF, trên tập nhiệm vụ cụ thể của paper EgoScale — chưa được kiểm chứng cho locomotion hay các năng lực khác.</details>

5. VLM backbone của System 2 trong N1.7 là gì, khác N1 gốc ở đâu? Số layer của DiT (System 1) là bao nhiêu, xác minh từ nguồn nào?
<details><summary>Gợi ý đáp án</summary>N1.7 dùng Cosmos-Reason2-2B (thay cho Eagle-2 của N1 gốc). DiT có 32 layer, xác minh trực tiếp từ trường `diffusion_model_cfg.num_layers = 32` trong file config.json của checkpoint `nvidia/GR00T-N1.7-3B` trên HuggingFace.</details>

6. Nếu N1.6 có khoảng 3.000 giờ dữ liệu teleoperation (giả định minh hoạ, không phải số chính thức), EgoScale (20.854 giờ) lớn hơn khoảng bao nhiêu lần?
<details><summary>Gợi ý đáp án</summary>20.854 / 3.000 ≈ 6.95, tức khoảng ~7 lần — nhưng đây chỉ là ước tính minh hoạ vì "few thousand hours" của N1.6 không phải con số chính xác đã công bố.</details>

## 📝 Bài tập thực hành

1. **Đọc trực tiếp blog HF N1.7** (`https://huggingface.co/blog/nvidia/gr00t-n1-7`) và tự tìm thêm **ít nhất 1 con số benchmark khác** chưa xuất hiện trong bài này (ví dụ: một task cụ thể trong 20+ loại tác vụ, hoặc một chi tiết về các embodiment được hỗ trợ). Ghi lại nguyên văn câu trích dẫn và đường link, theo đúng quy ước "trích dẫn kèm nguồn" của `NOI-DUNG-CHI-TIET.md`.
2. **Đọc trực tiếp paper arXiv:2602.16710 (bản HTML đầy đủ, không chỉ abstract)** và tìm phần mô tả chi tiết "Stage II — aligned mid-training": xác nhận lại xem 54 giờ dữ liệu "human-robot play data" được thu thập bằng phương pháp nào (teleoperation, kinesthetic teaching, hay cách khác), và liệu paper có nêu rõ số lượng robot/embodiment cụ thể dùng ở giai đoạn này hay không. So sánh với những gì bài giảng này đã ghi để phát hiện nếu có chi tiết bị bỏ sót hoặc diễn giải chưa chính xác.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

N1.7 đánh dấu bước ngoặt dữ liệu quan trọng nhất trong dòng GR00T: thay vì tiếp tục phụ thuộc vào dữ liệu teleoperation robot đắt đỏ như N1.6, nó pretrain trên **EgoScale** — 20.854 giờ video egocentric của con người làm việc thật (sản xuất, bán lẻ, y tế, gia đình), lớn hơn hơn 20 lần các nỗ lực trước đó, và không cần bất kỳ robot nào lúc thu thập dữ liệu. Từ đó, nhóm nghiên cứu phát hiện và công bố (qua cả blog và một paper arXiv riêng, 2602.16710) **scaling law log-linear đầu tiên cho dexterity**: validation loss giảm theo `L = 0.024 − 0.003·ln(D)` khi lượng dữ liệu D tăng, tương quan với việc average task completion tăng từ 0.30 (ở 1k giờ) lên 0.71 (ở 20k giờ) — hơn gấp đôi, dù dữ liệu chỉ tăng 20 lần, đúng bản chất log chứ không phải tuyến tính. Về kiến trúc, N1.7 (3B tham số, checkpoint `nvidia/GR00T-N1.7-3B`) đổi VLM backbone sang Cosmos-Reason2-2B và dùng DiT 32-layer (đã xác minh trực tiếp từ config.json) làm action head theo flow matching — nhưng cơ chế flow matching/dual-system tự nó không phải trọng tâm của bài này, đã có bài giảng riêng. Điểm cần nhớ nhất: đây là một quy luật **hẹp, cho riêng năng lực thao tác khéo léo**, không phải quy luật tổng quát cho mọi loại tác vụ robot, và nó đặt N1.7 vào cùng một trào lưu 2025–2026 mà các phòng lab khác (Physical Intelligence với π0.5, Figure với Helix/Project Go-Big, Google với Gemini Robotics) cũng đang theo đuổi bằng những chiến lược dữ liệu khác nhau để giải bài toán khan hiếm dữ liệu robot.
