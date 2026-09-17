# Bài giảng: So sánh công bằng (fair comparison) — 4 lỗi thường gặp

*(Thuộc mảng: Policy Evaluation)*

## 🎯 Mục tiêu bài học

- Kể tên và giải thích được 4 lỗi fair-comparison phổ biến nhất trong robot learning: cherry-picking, thiếu seed variance, test set rò rỉ, và so sánh không cùng compute budget.
- Tính tay được ví dụ minh hoạ mức độ sai lệch mà thiếu seed variance có thể gây ra (dùng thống kê cơ bản: mean, std, khoảng tin cậy).
- Nhận diện được từng lỗi này khi đọc một bảng kết quả trong paper hoặc trong báo cáo của chính mình.
- Thiết kế được một quy trình thí nghiệm "pre-registered" (khai báo trước) để tự phòng tránh cả 4 lỗi.
- Áp dụng được nguyên tắc fair comparison vào chính việc đánh giá policy trong dự án này.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Mọi metric đã học trong các bài trước (MPJPE, success rate, sim-to-real gap...) chỉ có giá trị khi được đo và so sánh **công bằng**. Robot learning là lĩnh vực đặc biệt dễ mắc lỗi so sánh không công bằng vì mỗi lần chạy robot thật tốn thời gian/rủi ro/chi phí, khiến người làm dễ "tiết kiệm" bằng cách cắt góc — chọn lần chạy đẹp nhất, chỉ chạy 1 seed, dùng lại dữ liệu train làm test, hoặc so sánh với baseline chưa được tune kỹ. Đây là nhóm lỗi phổ biến nhất khiến một kết quả "trông đẹp" nhưng không đáng tin, và là chỗ reviewer/mentor giỏi luôn kiểm tra đầu tiên.

## 🧠 Trực giác

### Góc nhìn 1: "Photographer chụp 100 tấm, chọn 1 tấm đẹp nhất để đăng"

Cherry-picking giống như một nhiếp ảnh gia chụp 100 tấm ảnh chân dung, sau đó chỉ chọn đúng 1 tấm đẹp nhất để đăng báo, khiến người xem tưởng nhầm "người này lúc nào cũng đẹp như vậy". Bản thân tấm ảnh không phải giả — chỉ là nó không đại diện cho phân phối thực sự của 100 tấm.

- **Đúng ở đâu:** nắm đúng cơ chế cherry-picking — không cần bịa số liệu, chỉ cần chọn lọc cái gì để trình bày.
- **Giới hạn:** loại suy "chụp ảnh" ngụ ý người xem dễ nhận ra đây chỉ là 1 khoảnh khắc — trong khi một video demo robot "thành công" trong báo cáo khoa học thường được trình bày như thể đại diện cho hiệu năng trung bình, khiến sự đánh lừa tinh vi hơn nhiều so với 1 tấm ảnh chân dung.

### Góc nhìn 2: "Thử thuốc trên 1 bệnh nhân duy nhất rồi kết luận thuốc hiệu quả"

Thiếu seed variance giống như một thử nghiệm y khoa chỉ thử thuốc trên đúng 1 bệnh nhân, rồi kết luận "thuốc này chữa khỏi bệnh" — trong khi bệnh nhân đó có thể tự khỏi vì lý do khác, hoặc là trường hợp đặc biệt không đại diện cho dân số chung.

- **Đúng ở đâu:** làm rõ tại sao "n=1" (1 seed) không đủ để kết luận về "phương pháp" nói chung — biến thiên tự nhiên giữa các cá thể (seed) có thể lớn hơn hẳn hiệu ứng thực sự của phương pháp.
- **Giới hạn:** loại suy y khoa ngụ ý biến thiên đến từ "cá thể" (mỗi bệnh nhân khác nhau) — trong RL, biến thiên giữa seed đến từ một nguồn khác: khởi tạo trọng số ngẫu nhiên và stochasticity trong quá trình training/rollout, không phải "cá thể" theo nghĩa sinh học — nhưng hệ quả thống kê (cần cỡ mẫu đủ lớn) là tương tự.

## 📐 Định nghĩa chính xác

Bốn lỗi thường gặp, áp dụng trực tiếp nguyên tắc chung ở `resources/07-experiments-and-rigor.md` vào bối cảnh robot policy:

| Lỗi | Biểu hiện cụ thể ở robot learning | Cách tránh |
|---|---|---|
| **1. Cherry-picking** | Quay video/báo cáo con số từ lần chạy đẹp nhất trong nhiều lần thử, không nói có bao nhiêu lần thử thất bại bị bỏ qua | Báo cáo **tất cả** các lần thử trong một khoảng thời gian/số lượng đã định trước (pre-registered), không chọn lọc sau khi thấy kết quả |
| **2. Thiếu seed variance** | Chỉ train 1 seed, báo cáo như thể đó là hiệu năng "của phương pháp" — trong khi RL/imitation learning dao động rất lớn giữa các seed | Chạy ≥3 seed (lý tưởng 5), báo cáo mean ± std |
| **3. Test set rò rỉ** | Dùng chính các trajectory/motion đã xuất hiện trong tập huấn luyện để làm "test" đo tracking error — con số đẹp giả tạo vì gần như đánh giá trên train set | Tách rõ tập motion huấn luyện và tập motion test/held-out ngay từ khâu chuẩn bị dữ liệu; với unseen-task evaluation, đảm bảo tổ hợp (vật thể, hành động, ngôn ngữ) thực sự chưa xuất hiện, không chỉ đổi câu chữ bề mặt |
| **4. So sánh baseline không cùng compute budget** | Method của bạn được tune hyperparameter kỹ, chạy nhiều bước huấn luyện hơn, trong khi baseline dùng cấu hình mặc định từ paper gốc | Cùng ngân sách tuning, cùng số bước huấn luyện/số lượng dữ liệu — hoặc nếu không thể, nói rõ sự khác biệt về ngân sách trong phần Limitations thay vì im lặng |

## ⚙️ Cơ chế hoạt động — từng bước

```
┌──────────── Quy trình thiết kế thí nghiệm phòng tránh cả 4 lỗi ────────────┐
│                                                                              │
│  TRƯỚC KHI CHẠY (pre-registration):                                         │
│    1. Viết ra: tiêu chí thành công là gì (chống lỗi 1 — cherry-picking)      │
│    2. Viết ra: số seed sẽ chạy (≥3, lý tưởng 5), CAM KẾT báo cáo TẤT CẢ      │
│       (chống lỗi 2 — thiếu seed variance)                                    │
│    3. Viết ra: tập test/held-out KHÔNG trùng tập train, kiểm tra chồng lấn   │
│       thủ công một lần trước khi chạy (chống lỗi 3 — test set rò rỉ)         │
│    4. Viết ra: ngân sách compute/tuning cho method của bạn VÀ cho baseline   │
│       — cùng số bước, cùng effort tuning nếu khả thi (chống lỗi 4)           │
│                                                                              │
│  TRONG KHI CHẠY:                                                            │
│    5. Log MỌI lần chạy (kể cả thất bại) — không xoá log sau khi thấy xấu     │
│                                                                              │
│  SAU KHI CHẠY (báo cáo):                                                    │
│    6. Báo cáo mean ± std trên toàn bộ seed đã cam kết ở bước 2               │
│    7. Nếu không thể cùng compute budget với baseline (bước 4), ghi rõ        │
│       trong phần Limitations — KHÔNG im lặng bỏ qua                          │
│    8. Nếu phát hiện chồng lấn test/train sau khi đã chạy, dừng lại, sửa      │
│       lại tập test, chạy lại từ đầu — không "vá" bằng cách loại bỏ vài       │
│       điểm dữ liệu xấu                                                       │
└──────────────────────────────────────────────────────────────────────────┘
```

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số liệu tự chọn để dễ hình dung — minh hoạ lỗi "thiếu seed variance".)*

Giả sử bạn train cùng một policy với 5 seed khác nhau, đo success rate mỗi seed trên 20 episode:

| Seed | Success rate |
|---|---|
| 1 | 85% |
| 2 | 60% |
| 3 | 90% |
| 4 | 55% |
| 5 | 75% |

**Nếu chỉ chạy seed 3 và báo cáo (lỗi thiếu seed variance):** "Policy đạt success rate 90%" — một con số ấn tượng nhưng gây hiểu lầm nghiêm trọng.

**Tính đúng — mean và std trên cả 5 seed:**

```
mean = (85+60+90+55+75)/5 = 365/5 = 73.0%
```

Độ lệch chuẩn (dùng công thức mẫu, chia cho n-1=4):

```
Bước 1 — độ lệch từng seed so với mean:
  85-73 = 12    → 12² = 144
  60-73 = -13   → (-13)² = 169
  90-73 = 17    → 17² = 289
  55-73 = -18   → (-18)² = 324
  75-73 = 2     → 2² = 4

Bước 2 — tổng bình phương độ lệch:
  144+169+289+324+4 = 930

Bước 3 — phương sai mẫu (chia n-1 = 4):
  930/4 = 232.5

Bước 4 — độ lệch chuẩn:
  std = √232.5 ≈ 15.25%
```

Vậy con số đúng để báo cáo là **73.0% ± 15.25%** — khác rất xa so với "90%" nếu chỉ chọn seed đẹp nhất. Độ lệch chuẩn 15.25 điểm phần trăm cũng cho thấy đây là một phương pháp có **variance rất lớn giữa các seed** — bản thân thông tin này (độ ổn định thấp) cũng là một phát hiện quan trọng, không nên bị ẩn đi.

**Minh hoạ tại sao n=1 nguy hiểm gấp đôi:** nếu ai đó chỉ chạy 1 seed và ngẫu nhiên trúng seed 4 (55%), họ có thể kết luận sai theo hướng ngược lại — "phương pháp này tệ" — trong khi thực ra hiệu năng trung bình thực sự (73%) không tệ như vậy. Cả hai hướng sai lệch (quá lạc quan lẫn quá bi quan) đều là hệ quả trực tiếp của cỡ mẫu quá nhỏ.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Báo cáo "best of N" (sai) | Báo cáo mean ± std trên N seed cố định (đúng) | Báo cáo median (thường ít dùng hơn trong RL) |
|---|---|---|---|
| Phản ánh đúng hiệu năng kỳ vọng? | Không — luôn lạc quan hơn thực tế | Có | Có, nhưng ít nhạy với outlier hơn mean |
| Cho biết độ ổn định (variance)? | Không | Có (qua std) | Không trực tiếp (cần thêm IQR) |
| Rủi ro bị lạm dụng | Rất cao — dễ "chọn N cho tới khi thấy 1 kết quả đẹp" | Thấp nếu N cam kết trước | Thấp |
| Chuẩn cộng đồng RL hiện nay | Không chấp nhận trong venue nghiêm túc | Chuẩn khuyến nghị | Ít phổ biến hơn nhưng vẫn hợp lệ nếu nêu rõ |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "chạy nhiều seed rồi báo cáo seed tốt nhất" cũng coi như đã làm đúng vì có nhiều dữ liệu."**
   Vì sao sai: đây thực chất là cherry-picking trá hình — bạn có nhiều dữ liệu (nhiều seed) nhưng chỉ chọn trình bày 1 kết quả, y hệt lỗi 1 đã nêu. Việc "có chạy nhiều seed" chỉ có giá trị nếu bạn **báo cáo thống kê trên toàn bộ**, không phải chọn lọc sau khi thấy kết quả.
   Hiểu đúng: mọi seed đã chạy phải được đưa vào tính mean ± std, không có ngoại lệ "bỏ seed xấu vì có vẻ là do lỗi ngẫu nhiên" trừ khi có lý do kỹ thuật xác đáng và ghi chú rõ (ví dụ seed đó bị crash do lỗi phần cứng, không phải do hiệu năng kém).

2. **Hiểu nhầm: "test set rò rỉ" chỉ xảy ra khi dùng đúng y hệt file dữ liệu train làm test, còn nếu đổi chút ít (paraphrase câu lệnh, đổi góc quay nhẹ) thì không tính là rò rỉ."**
   Vì sao sai: như đã nhấn mạnh ở bài `BAI-GIANG-metric-vla-loco-manipulation.md`, "novel instruction, same object" là một dạng unseen NHẸ — nếu bạn chỉ đổi câu chữ bề mặt của lệnh ngôn ngữ nhưng tổ hợp (đối tượng, hành động, bối cảnh) về bản chất giống hệt tập train, đây vẫn là một dạng rò rỉ thông tin (leakage) ở mức độ nhẹ hơn, khiến con số "unseen" bị thổi phồng.
   Hiểu đúng: kiểm tra rò rỉ ở cấp độ tổ hợp ngữ nghĩa (đối tượng + hành động + bối cảnh), không chỉ ở cấp độ chuỗi ký tự của câu lệnh hay tên file dữ liệu.

## 🏗️ Ví dụ minh hoạ trong dự án này

Khi bạn báo cáo kết quả policy của chính mình huấn luyện ở `04-imitation-learning-rl/`/`05-simulation-mujoco-isaaclab/`:

- Trước khi huấn luyện, viết ra văn bản: "sẽ chạy 5 seed, báo cáo mean±std success rate trên 20 episode/seed, tập test là các trajectory X,Y,Z KHÔNG có trong tập train" — đúng tinh thần pre-registration.
- Nếu so sánh với một baseline có sẵn (ví dụ so sánh AMP-based policy của bạn với DeepMimic gốc), kiểm tra baseline đó dùng bao nhiêu bước huấn luyện/bao nhiêu dữ liệu — nếu bạn train policy của mình nhiều hơn baseline gốc 10 lần số bước, phải ghi rõ điều này trong báo cáo thay vì để người đọc ngộ nhận đây là so sánh công bằng "phương pháp X tốt hơn phương pháp Y".
- Khi quay video demo cho `06-vla-groot-sonic/` hoặc `08-real-robot-deployment/`, ghi rõ video này là "1 trong N lần thử, tỉ lệ thành công tổng thể là M%" thay vì trình bày như một minh chứng đại diện duy nhất.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **"Khủng hoảng tái lập" (reproducibility) trong RL đã được định lượng cụ thể qua các nghiên cứu thống kê gần đây.** Theo tra cứu, công trình kinh điển của Henderson et al. (2018) — vẫn được các bài viết 2025 trích dẫn lại như nền tảng — cho thấy khi train 5 thuật toán policy gradient hàng đầu (PPO, TRPO, DDPG, TD3, SAC) trên 6 môi trường MuJoCo chuẩn, chỉ thay đổi random seed (giữ nguyên mọi hyperparameter khác), các đường cong học (learning curve) giữa các seed **không hề chồng lấn lên nhau** — một seed tốt của thuật toán A có thể cho kết quả tương đương seed tốt nhất của thuật toán B, khiến việc so sánh thuật toán trở nên không đáng tin nếu chỉ dựa vào vài seed.
2. **Một phân tích thống kê gần đây (theo tra cứu tổng hợp về reproducibility 2025) chỉ ra hệ quả cụ thể của cỡ mẫu nhỏ**: so sánh điểm ước lượng (point estimate) từ 5 lần chạy/task trên benchmark Atari 100k cho tỉ lệ lỗi loại I (Type I error — kết luận "có khác biệt" khi thực ra không có) **vượt quá 50%** — nghĩa là việc thêm nhiễu ngẫu nhiên (không có tác dụng thực sự gì) vẫn có thể "trông như cải thiện" trong hơn một nửa số so sánh nếu chỉ dựa vào rất ít lần chạy. Đây là bằng chứng định lượng mạnh cho lý do tại sao ≥3-5 seed là mức tối thiểu, không phải một quy ước tuỳ tiện.
3. **Khuyến nghị chuẩn hoá hiện nay** (theo tổng hợp WebSearch về reproducibility trong ML/RL 2025): các mô hình ML/RL "nên được benchmark và đánh giá với nhiều random seed, sao cho variance có thể được báo cáo và phản ánh đúng hiệu năng thật của mô hình" — đây chính là khuyến nghị cộng đồng học thuật đang dần chuẩn hoá thành quy tắc bắt buộc ở nhiều venue công bố (workshop reproducibility checklist, yêu cầu báo cáo std/confidence interval), không còn là "gợi ý tốt nên có" mà đang tiến gần thành yêu cầu bắt buộc.

## ❓ Câu hỏi tự kiểm tra

1. Kể tên 4 lỗi fair-comparison đã học, mỗi lỗi 1 câu mô tả ngắn.
<details><summary>Gợi ý đáp án</summary>(1) Cherry-picking: chọn lần chạy đẹp nhất để báo cáo. (2) Thiếu seed variance: chỉ chạy 1 seed rồi coi là đại diện. (3) Test set rò rỉ: dùng lại dữ liệu train làm test. (4) So sánh không cùng compute budget: method của mình được tune kỹ hơn baseline.</details>

2. Từ 5 số liệu success rate (85,60,90,55,75)%, tính lại mean nếu bỏ đi seed 3 (90%, seed "đẹp nhất") một cách vô căn cứ — kết quả thay đổi bao nhiêu so với mean đúng (73%)?
<details><summary>Gợi ý đáp án</summary>Mean của 4 seed còn lại (85,60,55,75) = 275/4 = 68.75% — khác 4.25 điểm % so với mean đúng (73%) trên cả 5 seed. Việc bỏ 1 seed bất kỳ mà không có lý do kỹ thuật xác đáng đã làm thay đổi kết luận, minh hoạ tại sao không được tự ý loại bỏ dữ liệu.</details>

3. Vì sao "đổi cách diễn đạt câu lệnh nhẹ" vẫn có thể coi là một dạng rò rỉ test set, dù không dùng đúng y hệt câu lệnh gốc?
<details><summary>Gợi ý đáp án</summary>Vì rò rỉ được xét ở cấp độ tổ hợp ngữ nghĩa (đối tượng + hành động + bối cảnh), không chỉ ở chuỗi ký tự — nếu tổ hợp ngữ nghĩa về bản chất giống hệt tập train, chỉ đổi vỏ ngôn ngữ, mô hình có thể vẫn đang "nhận diện" tổ hợp đã học chứ chưa thực sự tổng quát hoá.</details>

4. Tỉ lệ lỗi loại I vượt 50% khi so sánh 5 lần chạy/task (theo phát hiện đã dẫn) có ý nghĩa gì cho việc thiết kế thí nghiệm của bạn?
<details><summary>Gợi ý đáp án</summary>Nghĩa là với cỡ mẫu quá nhỏ, một sự khác biệt "có vẻ cải thiện" hoàn toàn có thể chỉ là nhiễu ngẫu nhiên trong hơn một nửa trường hợp — cần cỡ mẫu (số seed) đủ lớn và kiểm định thống kê phù hợp trước khi kết luận một phương pháp "tốt hơn" phương pháp khác.</details>

5. Nếu bạn không có đủ compute để chạy baseline với cùng ngân sách tuning như method của mình, bạn nên làm gì?
<details><summary>Gợi ý đáp án</summary>Ghi rõ sự khác biệt về ngân sách compute/tuning trong phần Limitations của báo cáo, thay vì im lặng trình bày như thể đây là một so sánh công bằng ngang hàng.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** cho 4 seed với success rate (70%, 45%, 80%, 65%). Tính mean và std (dùng công thức chia n-1=3 như ví dụ trong bài). So sánh độ lệch chuẩn này với ví dụ 5-seed trong bài (std≈15.25%) — biến thiên giữa các seed ở đây lớn hơn hay nhỏ hơn?
2. **Đọc kết quả thật:** nếu bạn có sẵn ≥2 checkpoint (2 seed) đã huấn luyện ở `04-imitation-learning-rl/`, chạy đánh giá độc lập từng checkpoint trên cùng 1 tập test cố định, so sánh xem success rate giữa 2 seed lệch nhau bao nhiêu — nếu chỉ có 1 checkpoint, huấn luyện thêm 1 seed thứ 2 với cùng cấu hình (chỉ đổi random seed) để tự trải nghiệm mức độ biến thiên giữa các seed trên chính hệ thống của bạn.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

Bốn lỗi fair-comparison phổ biến nhất trong robot learning — cherry-picking (chọn lần chạy đẹp nhất), thiếu seed variance (chỉ chạy 1 seed rồi coi là đại diện), test set rò rỉ (đánh giá trên dữ liệu đã dùng để train), và so sánh không cùng compute budget (baseline bị "xử tệ" hơn method của mình) — đều xuất phát từ cùng một động cơ: mỗi lần chạy robot/thí nghiệm tốn kém, khiến người làm dễ cắt góc để có kết quả "đẹp" nhanh hơn. Cách phòng tránh chung cho cả 4 lỗi là pre-registration: viết ra trước khi chạy thí nghiệm chính thức — tiêu chí thành công, số seed sẽ chạy và cam kết báo cáo hết, ranh giới rõ ràng giữa tập train/test, và ngân sách compute cho cả method lẫn baseline — rồi tuân thủ đúng những gì đã viết ra dù kết quả có đẹp hay không. Bằng chứng định lượng gần đây (Henderson et al. 2018 và các phân tích kế thừa) cho thấy hậu quả của việc bỏ qua các nguyên tắc này là nghiêm trọng: so sánh dựa trên quá ít lần chạy có thể cho tỉ lệ kết luận sai (Type I error) vượt quá 50%.
