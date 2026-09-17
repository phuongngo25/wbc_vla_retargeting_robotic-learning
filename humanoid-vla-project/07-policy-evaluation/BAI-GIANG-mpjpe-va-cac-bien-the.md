# Bài giảng: Metric motion tracking — MPJPE và các biến thể

*(Thuộc mảng: Policy Evaluation)*

## 🎯 Mục tiêu bài học

- Viết được công thức MPJPE từ đầu, không cần nhìn tài liệu.
- Tính tay được MPJPE cho một vài khớp giả định, ra đúng đơn vị (mm).
- Phân biệt được MPJPE toàn cục (global) với MPJPE cục bộ (root-relative / MPJPE-L).
- Giải thích được PA-MPJPE khác MPJPE-L ở phép biến đổi nào, và tại sao chọn cái này thay cái kia làm thay đổi kết luận của một thí nghiệm.
- Nhận ra khi nào một con số MPJPE "đẹp" đang che giấu một vấn đề thật (ví dụ robot ngã nhưng vẫn có MPJPE thấp).
- Đọc được một bảng kết quả MPJPE trong paper (như SONIC) và biết hỏi đúng câu hỏi trước khi tin số liệu.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Trong toàn bộ pipeline của dự án — retargeting (`02-motion-retargeting/`) tạo ra reference motion, rồi policy được train bằng imitation/RL (`04-imitation-learning-rl/`) để bắt chước reference motion đó trong sim (`05-simulation-mujoco-isaaclab/`) — câu hỏi cuối cùng luôn là: **policy bắt chước tốt tới đâu?**. MPJPE là câu trả lời định lượng đầu tiên và phổ biến nhất cho câu hỏi đó. Nó là "thước đo cơ bản nhất" mà gần như mọi paper motion-tracking humanoid gần đây (DeepMimic, AMP, PHC, OmniH2O, SONIC) đều báo cáo, nên nếu bạn không hiểu chính xác nó đo cái gì và không đo cái gì, bạn sẽ đọc sai mọi bảng kết quả trong lĩnh vực này.

## 🧠 Trực giác

### Góc nhìn 1: "Chấm điểm tô màu theo mẫu" (stencil overlay)

Tưởng tượng bạn có một bản vẽ mẫu (reference motion) in trên giấy trong suốt, đặt chồng lên bản vẽ robot thực sự tạo ra. Với mỗi điểm đánh dấu (khớp), bạn đo khoảng cách giữa điểm trên bản mẫu và điểm tương ứng trên bản robot vẽ ra, rồi lấy trung bình các khoảng cách đó. Càng khớp, các điểm càng chồng khít, khoảng cách trung bình càng nhỏ.

- **Đúng ở đâu:** nắm bắt đúng ý tưởng cốt lõi — MPJPE là trung bình khoảng cách theo từng điểm, tại một thời điểm (một frame).
- **Giới hạn:** phép loại suy này không nói rõ "đặt chồng" theo cách nào — nếu bạn dịch chuyển cả tờ giấy robot sang một bên trước khi so, kết quả sẽ khác hẳn so với để nguyên vị trí gốc. Đây chính là chỗ phân biệt MPJPE toàn cục và MPJPE-L/PA-MPJPE ở phần dưới — loại suy "tô màu" không tự nói cho bạn biết dùng hệ toạ độ nào.

### Góc nhìn 2: "Đo khoảng cách giữa 2 đội hình nhảy đồng diễn"

Hai đội nhảy đồng diễn (đội tham chiếu và đội robot) đứng trên sân, mỗi người trong đội tương ứng với một khớp. Ở một khoảnh khắc, bạn đo khoảng cách giữa từng cặp người tương ứng (người số 3 đội A với người số 3 đội B, v.v.), rồi lấy trung bình. Nếu cả đội robot đứng lùi lại 2 mét so với đội tham chiếu nhưng đội hình bên trong vẫn y hệt, "MPJPE toàn cục" đo được sẽ rất lớn (vì tính cả độ lệch 2m của cả đội) — trong khi thực ra dáng đội hình *bên trong* là hoàn hảo.

- **Đúng ở đâu:** làm rõ vấn đề "lỗi định vị toàn cục" (global position drift) — nếu không trừ đi vị trí gốc chung, một robot có dáng đi hoàn hảo nhưng đi lệch hướng vẫn bị phạt nặng.
- **Giới hạn:** phép loại suy này giả định "người số 3" của 2 đội luôn khớp đúng nhau về mặt định danh (correspondence) — trong thực tế đây là giả định cần có sẵn (retargeting/skeleton mapping đã giải quyết từ trước, xem `02-motion-retargeting/BAI-GIANG-skeleton-mapping.md`), MPJPE không tự làm việc "khớp khớp nào với khớp nào".

## 📐 Định nghĩa chính xác

Tại mỗi frame, với N khớp (joint) đang theo dõi:

```
MPJPE = (1/N) × Σᵢ₌₁ᴺ ‖Pᵢ − Gᵢ‖₂
```

trong đó:
- `Pᵢ` = vị trí 3D của khớp thứ *i* trên **robot** (predicted, policy tạo ra) tại frame đó.
- `Gᵢ` = vị trí 3D của khớp thứ *i* trên **chuyển động tham chiếu** (ground truth) tại cùng frame.
- `‖·‖₂` = khoảng cách Euclid: `‖Pᵢ − Gᵢ‖₂ = √[(Pᵢₓ−Gᵢₓ)² + (Pᵢᵧ−Gᵢᵧ)² + (Pᵢᵤ−Gᵢᵤ)²]`.
- Kết quả trung bình theo N khớp tại 1 frame, sau đó thường trung bình tiếp theo mọi frame trong episode → một con số duy nhất, đơn vị **mm** (đôi khi cm hoặc m tuỳ paper — luôn kiểm tra đơn vị trước khi so sánh giữa các paper).

Ba biến thể quan trọng cần phân biệt rõ ràng — đây là phần hay bị nhầm lẫn nhất:

| Biến thể | Hệ toạ độ tính | Loại bỏ được sai số nào | Dùng khi nào |
|---|---|---|---|
| **MPJPE (global)** | Toạ độ thế giới (world frame) | Không loại bỏ gì | Muốn biết robot có đi đúng vị trí/hướng tuyệt đối không (ví dụ tracking một trajectory đi bộ cụ thể) |
| **MPJPE-L (local, root-relative)** | Toạ độ gắn với root (pelvis/torso), trừ đi vị trí+hướng root trước khi tính | Sai số vị trí/hướng của root (global drift) | Muốn đo "dáng tư thế" (pose shape) thuần tuý, tách biệt khỏi lỗi định vị toàn cục — SONIC dùng biến thể này khi báo cáo sim-to-real gap |
| **PA-MPJPE (Procrustes-Aligned)** | Sau khi tìm phép xoay/dịch/co giãn tối ưu (Procrustes analysis) để khớp 2 tập điểm gần nhau nhất, mới tính khoảng cách | Cả vị trí, hướng, VÀ tỉ lệ (scale) toàn cục | Chỉ quan tâm hình dạng tương đối giữa các khớp, không quan tâm cả vị trí lẫn hướng lẫn kích thước tổng thể |

Với khớp có bậc tự do xoay (không chỉ vị trí), một số paper báo thêm **sai số góc khớp** (joint angle error, đơn vị độ) song song với MPJPE vị trí — đây là một con số khác, không thay thế MPJPE.

## ⚙️ Cơ chế hoạt động — từng bước

```
┌─────────────────────────────────────────────────────────────┐
│  Với mỗi frame t trong episode:                              │
│                                                                │
│  1. Lấy vị trí 3D của N khớp trên robot: {P₁,...,Pₙ}(t)       │
│  2. Lấy vị trí 3D của N khớp trên reference motion: {G₁,...,Gₙ}(t)│
│  3. (Nếu tính MPJPE-L) Trừ vị trí root + xoay ngược theo      │
│     hướng root → đưa cả P và G về hệ toạ độ cục bộ            │
│  4. (Nếu tính PA-MPJPE) Tìm phép Procrustes tối ưu (R,t,s)    │
│     sao cho ‖s·R·P + t − G‖ nhỏ nhất, áp phép này lên P       │
│  5. Với mỗi khớp i: tính dᵢ(t) = ‖Pᵢ(t) − Gᵢ(t)‖₂             │
│  6. MPJPE(t) = (1/N) Σᵢ dᵢ(t)   ← một số cho frame t          │
│                                                                │
│  Sau khi duyệt hết T frame:                                   │
│  7. MPJPE_episode = (1/T) Σₜ MPJPE(t)  ← một số cho cả episode│
└─────────────────────────────────────────────────────────────┘
```

Điểm quan trọng thường bị bỏ sót ở bước 3–4: MPJPE-L và PA-MPJPE **không phải là hậu xử lý sau khi đã có MPJPE global** — chúng là 2 pipeline tính riêng biệt ngay từ bước biến đổi toạ độ, cho ra 3 con số khác nhau trên cùng một cặp (P, G).

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số liệu tự chọn để dễ hình dung — không lấy từ một paper cụ thể nào.)*

Giả sử tại 1 frame, ta theo dõi N = 3 khớp: **hông (hip)**, **đầu gối (knee)**, **cổ chân (ankle)** của chân phải, đơn vị mét (m), toạ độ thế giới (x, y, z):

| Khớp | Vị trí robot P (m) | Vị trí tham chiếu G (m) |
|---|---|---|
| Hip | (0.10, 0.00, 0.90) | (0.10, 0.00, 0.90) |
| Knee | (0.12, 0.00, 0.50) | (0.10, 0.00, 0.48) |
| Ankle | (0.15, 0.00, 0.05) | (0.10, 0.00, 0.02) |

**Bước 1 — tính khoảng cách Euclid từng khớp:**

- Hip: `d = √[(0.10−0.10)² + (0−0)² + (0.90−0.90)²] = √0 = 0` m
- Knee: `d = √[(0.12−0.10)² + 0² + (0.50−0.48)²] = √[0.0004 + 0.0004] = √0.0008 ≈ 0.0283` m
- Ankle: `d = √[(0.15−0.10)² + 0² + (0.05−0.02)²] = √[0.0025 + 0.0009] = √0.0034 ≈ 0.0583` m

**Bước 2 — trung bình theo N=3 khớp:**

```
MPJPE(t) = (0 + 0.0283 + 0.0583) / 3 = 0.0866 / 3 ≈ 0.0289 m = 28.9 mm
```

Vậy tại frame này, MPJPE ≈ **28.9 mm** — nằm trong khoảng hợp lý so với các con số thực tế đã dẫn ở mục Cập nhật hiện đại bên dưới (nhiều hệ thống báo cáo 20–80 mm tuỳ độ khó task).

**Giờ thử tình huống "robot bị lệch toàn cục":** nếu robot bị trôi cả người sang phải 5 cm (root dịch chuyển, dáng tư thế bên trong y hệt), mọi khớp P đều cộng thêm (0.05, 0, 0):

- Hip: `d = √[(0.15−0.10)² + 0 + 0] = 0.05` m = 50mm
- Knee: `d = √[(0.17−0.10)² + (0.50−0.48)²] = √[0.0049+0.0004] ≈ 0.0728` m
- Ankle: `d = √[(0.20−0.10)² + (0.05−0.02)²] = √[0.01+0.0009] ≈ 0.1044` m

```
MPJPE_global(t) = (0.05 + 0.0728 + 0.1044)/3 ≈ 0.0757 m = 75.7 mm
```

MPJPE global tăng vọt từ 28.9mm lên 75.7mm dù dáng tư thế *bên trong* không đổi chút nào — đây chính xác là lý do cần MPJPE-L: nếu tính lại sau khi trừ vị trí root (đưa hip robot về lại (0.10,0,0.90) làm gốc, các khớp khác trừ theo), kết quả MPJPE-L sẽ quay lại đúng 28.9mm như ban đầu, vì trục lệch toàn cục đã bị loại bỏ.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | MPJPE (global/local) | Success rate | Joint angle error |
|---|---|---|---|
| Đơn vị | mm (khoảng cách) | % | độ (góc) |
| Đo liên tục hay nhị phân | Liên tục (continuous) — càng nhỏ càng tốt, không có ngưỡng "đạt/không đạt" | Nhị phân (binary) trên mỗi episode | Liên tục |
| Nhạy với lỗi nhỏ tích luỹ | Rất nhạy — 1 khớp lệch nhẹ mọi frame vẫn kéo trung bình lên | Không nhạy — chỉ quan tâm ngưỡng cuối cùng đạt/không | Nhạy tương tự MPJPE nhưng đo không gian góc khớp thay vì không gian Descartes |
| Dễ "đánh lừa" bằng cách nào | Robot đứng gần đúng vị trí trung bình suốt episode có thể có MPJPE thấp giả tạo dù chưa từng thực sự theo kịp chuyển động động | Định nghĩa "thành công" quá dễ (xem bài `success-rate-jerk-root-trajectory-error`) | Hai tư thế có vị trí Descartes giống nhau vẫn có thể có góc khớp khác nhau (dư thừa động học/redundancy) — hoặc ngược lại |
| Khi nào ưu tiên dùng | Cần một con số tổng quát, so sánh được xuyên paper (chuẩn cộng đồng) | Cần biết "task có xong hay không" theo nghĩa thực dụng | Khi quan tâm chính xác không gian điều khiển (control space), không phải không gian Descartes |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "MPJPE thấp = policy tốt", chấm hết.**
   Vì sao sai: MPJPE thấp có thể đạt được bởi một policy "lười" — ví dụ đứng yên gần vị trí trung bình của chuyển động tham chiếu thay vì thực sự theo kịp động lực học. Một policy ngã giữa chừng nhưng episode bị cắt sớm (early termination) đôi khi vẫn có MPJPE trung bình thấp trên phần đã đi qua, dù thực chất episode đó là thất bại.
   Hiểu đúng: MPJPE luôn phải đọc **cùng** với success rate — MPJPE chỉ có ý nghĩa trên các episode được coi là "thành công", hoặc phải báo cáo song song với tỉ lệ ngã/thất bại.

2. **Hiểu nhầm: so sánh trực tiếp con số MPJPE giữa 2 paper khác nhau mà không kiểm tra biến thể.**
   Vì sao sai: một paper báo cáo MPJPE global (bao gồm cả lỗi drift toàn cục), một paper khác báo cáo MPJPE-L (đã loại bỏ drift) — hai con số này đo hai thứ khác nhau về bản chất, dù cùng tên gọi "MPJPE". Ví dụ đã thấy ở phần tính tay: cùng một dáng tư thế, MPJPE global có thể gấp 2.5 lần MPJPE-L.
   Hiểu đúng: luôn kiểm tra paper định nghĩa MPJPE theo biến thể nào (đọc phần Method/Evaluation Metrics, không đoán từ tên gọi), và chỉ so sánh cùng biến thể, cùng đơn vị, cùng bộ khớp N được tính.

## 🏗️ Ví dụ minh hoạ trong dự án này

Trong pipeline của dự án (retargeting ở `02-motion-retargeting/` → train policy ở `04-imitation-learning-rl/` → sim ở `05-simulation-mujoco-isaaclab/`), khi bạn tự huấn luyện một policy tracking một điệu nhảy lấy từ AMASS/LAFAN1 (`03-human-motion-datasets/`), MPJPE là con số đầu tiên bạn nên log mỗi lần đánh giá checkpoint:

- Dùng **MPJPE-L toàn thân** làm chỉ số theo dõi chính trong lúc training (vì robot G1/H1 chưa chắc đi đúng vị trí tuyệt đối ngay từ đầu, MPJPE global sẽ nhiễu và khó dùng để so sánh giữa các checkpoint).
- Khi đánh giá cuối cùng để báo cáo, tách MPJPE theo từng bộ phận (tay/chân/thân trên) như SONIC làm — xem bài giảng riêng "sim-to-real-gap-case-study-sonic" để thấy ví dụ cụ thể tay vs chân có gap rất khác nhau (upper body ~22mm, feet ~30-54mm) — cùng logic áp dụng được khi so sánh MPJPE giữa các checkpoint sim của chính bạn.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **MPJPE đã trở thành metric mặc định trong toàn bộ dòng nghiên cứu motion-tracking humanoid 2024-2025**, kế thừa trực tiếp từ PHC và OmniH2O. Theo tổng hợp tìm được (qua nhiều paper 2025 như *"RL from Physical Feedback"*, arXiv:2506.12769, và *"Humanoid-VLA"*, arXiv:2502.14795), các paper hiện dùng bộ 3 metric chuẩn "Success Rate + MPJPE + MPKPE" (MPKPE = Mean Per Keypoint Position Error, một biến thể mở rộng theo keypoint thay vì chỉ khớp xương) để báo cáo motion tracking — cho thấy cộng đồng đã hội tụ về việc luôn báo MPJPE **kèm** success rate, đúng như cảnh báo ở mục "Sai lầm thường gặp" trên.
2. **Khoảng giá trị MPJPE thực tế trên robot thật đã được công khai nhiều hơn** — ví dụ một hệ thống triển khai thực tế báo cáo MPJPE trung bình 82.45mm trên robot thật, trong khi một số pipeline khác đạt dưới 40mm một cách nhất quán (theo kết quả tổng hợp WebSearch tháng 9/2026, không có một nguồn duy nhất gộp toàn bộ số liệu này — cần đọc trực tiếp từng paper nếu trích dẫn học thuật). Điều này cho thấy khoảng "MPJPE tốt" rất phụ thuộc vào robot, độ khó chuyển động, và biến thể (global/local) — không có một ngưỡng chung "dưới X mm là tốt" áp dụng cho mọi trường hợp.
3. **PA-MPJPE tiếp tục là chuẩn khi so sánh hình dạng chuyển động thuần tuý**, đặc biệt khi retargeting giữa các bộ khung xương có kích thước/tỷ lệ khác nhau (người → robot) — nguồn gốc kỹ thuật này từ lĩnh vực 3D human pose estimation vẫn được giữ nguyên khi mượn sang robot learning, chưa thấy biến thể thay thế PA-MPJPE nổi bật hơn trong các paper 2024-2026 đã tra cứu được.

## ❓ Câu hỏi tự kiểm tra

1. Viết công thức MPJPE cho N=4 khớp, giải thích từng ký hiệu.
<details><summary>Gợi ý đáp án</summary>MPJPE = (1/4)Σᵢ₌₁⁴‖Pᵢ−Gᵢ‖₂, với Pᵢ là vị trí khớp i trên robot, Gᵢ trên reference, ‖·‖₂ là khoảng cách Euclid 3D.</details>

2. Tại sao MPJPE-L luôn ≤ MPJPE global cho cùng một cặp trajectory (gợi ý: nghĩ về việc trừ đi vị trí root ảnh hưởng gì tới các số hạng)?
<details><summary>Gợi ý đáp án</summary>Không hẳn luôn nhỏ hơn tuyệt đối trong mọi trường hợp lý thuyết, nhưng trong thực tế gần như luôn nhỏ hơn hoặc bằng vì trừ đi vị trí root loại bỏ hẳn một thành phần lỗi hệ thống (lỗi drift chung của toàn bộ trajectory) khỏi mọi khớp cùng lúc — trong khi các thành phần lỗi cục bộ (dáng tư thế) vẫn giữ nguyên.</details>

3. Cho ví dụ một tình huống MPJPE-L thấp nhưng success rate = 0%.
<details><summary>Gợi ý đáp án</summary>Robot theo đúng dáng tư thế tương đối rất tốt (MPJPE-L thấp) nhưng bị ngã hoàn toàn ở frame cuối do mất cân bằng root — nếu tiêu chí thành công là "không ngã suốt episode", success rate vẫn = 0% dù phần lớn episode có MPJPE-L đẹp.</details>

4. PA-MPJPE khác MPJPE-L ở phép biến đổi nào cụ thể? Nếu robot có kích thước (chiều dài chân) khác hoàn toàn so với người trong reference motion, biến thể nào phù hợp hơn để đánh giá "dáng đi có giống không"?
<details><summary>Gợi ý đáp án</summary>PA-MPJPE thêm phép co giãn (scale) và tìm phép xoay/dịch tối ưu (Procrustes) thay vì chỉ trừ vị trí+hướng root cố định như MPJPE-L. Khi kích thước khác biệt lớn (người vs robot), PA-MPJPE phù hợp hơn vì nó loại bỏ luôn sai số do tỉ lệ cơ thể khác nhau, chỉ còn lại sai số hình dạng thuần tuý.</details>

5. Một paper báo cáo "MPJPE = 25mm" nhưng không nói biến thể nào. Bạn cần hỏi thêm những gì trước khi so sánh với kết quả của mình?
<details><summary>Gợi ý đáp án</summary>Hỏi: (a) global hay local (root-relative) hay Procrustes-aligned? (b) tính trên N khớp nào, có gồm ngón tay/đầu không? (c) trung bình theo toàn episode hay chỉ trên các episode "thành công"? (d) đơn vị chính xác (mm/cm/m)?</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** dùng lại bảng số liệu ở phần "Ví dụ tính tay" (3 khớp hip/knee/ankle, trường hợp robot bị lệch 5cm toàn cục), nhưng lần này áp dụng PA-MPJPE: tìm phép dịch chuyển tối ưu duy nhất (không xoay, không scale, giả sử bài toán 1D chỉ theo trục x) sao cho tổng bình phương sai số nhỏ nhất, rồi tính lại MPJPE sau khi dịch. So sánh với kết quả MPJPE-L (chỉ trừ theo root/hip) đã tính ở trên — chúng có bằng nhau không? Vì sao?
2. **Đọc code thật:** nếu bạn có quyền truy cập một pipeline eval trong `HumanoidVerse` hoặc code eval của DeepMimic/AMP (đã dẫn trong `NOI-DUNG-CHI-TIET.md`), tìm đúng hàm tính MPJPE, xác nhận nó tính theo biến thể nào (global/local/PA) bằng cách đọc xem có bước trừ root hoặc Procrustes align hay không trước khi log kết quả này vào sổ tay cá nhân của bạn cho lần dùng lại.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

MPJPE (Mean Per Joint Position Error) là khoảng cách Euclid trung bình giữa vị trí các khớp robot và vị trí khớp tham chiếu, tính theo mm và trung bình theo cả số khớp lẫn số frame — đây là thước đo cơ bản nhất cho "bắt chước chuyển động tốt tới đâu" trong toàn bộ literature motion-tracking humanoid. Ba biến thể — global (không xử lý gì), local/MPJPE-L (trừ vị trí+hướng root để tách lỗi drift toàn cục khỏi lỗi dáng tư thế), và PA-MPJPE (thêm cả phép co giãn qua Procrustes analysis) — đo ba thứ khác nhau về bản chất và cho ra những con số rất khác nhau trên cùng một trajectory, nên không bao giờ được so sánh chéo giữa các biến thể mà không kiểm tra kỹ định nghĩa. MPJPE là một con số liên tục, không tự nói cho bạn biết episode có "thành công" hay không — vì vậy luôn phải đọc song song với success rate, nếu không một policy lười biếng hoặc một episode bị cắt sớm có thể tạo ra ảo giác MPJPE thấp mà thực chất là thất bại.
