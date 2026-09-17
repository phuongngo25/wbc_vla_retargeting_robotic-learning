# Bài giảng: AMASS — cách xây dựng (MoSh/MoSh++)

*(Thuộc mảng: Human Motion Datasets)*

## 🎯 Mục tiêu bài học

- Giải thích được bài toán AMASS giải quyết: hợp nhất 15 bộ mocap khác nhau về cùng 1 định dạng.
- Mô tả được cơ chế MoSh/MoSh++ ở mức có thể tính tay (bài toán tối ưu hoá tìm β, θ khớp với marker quan sát).
- Nhớ và giải thích đúng số liệu chính thức của AMASS (>40 giờ, >11,000 motion, >300 subject), phân biệt với con số sai đã từng xuất hiện trong dự án ("480+ người").
- Liệt kê được ít nhất 3 giới hạn/thiên lệch dữ liệu của AMASS.
- So sánh được AMASS với ít nhất 1 dataset thay thế/bổ sung hiện đại (PHUMA, Motion-X++).
- Giải thích được phát hiện gần đây rằng "dùng ít hơn 3% AMASS có thể cho kết quả tốt hơn dùng toàn bộ" và ý nghĩa của nó với việc chọn dữ liệu huấn luyện.

## 🧭 Vì sao cần học cái này? (bối cảnh)

AMASS là nguồn dữ liệu chuyển động người **đa dạng nhất** trong 5 dataset của thư mục này, và là input chính cho hầu hết công cụ retargeting (GMR, SOMA-retargeter — xem `02-motion-retargeting/`) trước khi tới bước huấn luyện WBC/RL. Nhưng AMASS không phải là mocap gốc — nó là **kết quả của một quy trình xử lý** (MoSh/MoSh++) biến 15 bộ mocap rời rạc, không tương thích, thành một tham số hoá SMPL/SMPL-X thống nhất. Hiểu đúng quy trình này quan trọng vì: (1) nó giải thích tại sao chất lượng dữ liệu AMASS không đồng đều giữa các subset, (2) nó là ví dụ kinh điển của việc dùng chính SMPL (bài giảng trước) làm "đích" cho một bài toán tối ưu hoá ngược, và (3) các phát hiện gần đây về việc lọc/chọn lọc dữ liệu AMASS ảnh hưởng trực tiếp tới cách xây dựng tập huấn luyện cho SONIC/GMT/BeyondMimic.

## 🧠 Trực giác

### Góc nhìn 1: Phiên dịch viên hợp nhất nhiều phương ngữ (góc nhìn dữ liệu)

Hãy tưởng tượng 15 phòng thí nghiệm khác nhau trên thế giới, mỗi nơi ghi lại chuyển động người bằng một "phương ngữ" riêng — số lượng marker khác nhau (một số phòng dùng 41 marker, phòng khác dùng 53 marker), vị trí gắn marker khác nhau, định dạng file khác nhau. Muốn dạy một mạng neural học từ TẤT CẢ các nguồn này cùng lúc, ta cần một "phiên dịch viên chung" dịch mọi phương ngữ về cùng 1 ngôn ngữ chuẩn (SMPL). MoSh/MoSh++ chính là phiên dịch viên đó — với mỗi khung hình mocap (bất kể phương ngữ nào), nó tìm ra bộ tham số SMPL (β, θ) tương ứng.

**Giới hạn của loại suy này**: phiên dịch ngôn ngữ giữ nguyên ý nghĩa hoàn hảo (một câu tiếng Anh dịch sang tiếng Việt không mất thông tin cốt lõi), nhưng MoSh/MoSh++ là một phép **xấp xỉ tối ưu hoá** — luôn có sai số dư (residual error) giữa marker ảo trên mesh SMPL và marker thật đo được, đặc biệt ở các bộ mocap cũ có ít marker hơn (thông tin gốc không đủ để "dịch" chính xác 100%).

### Góc nhìn 2: Inverse Kinematics ở quy mô toàn thân (góc nhìn thuật toán)

Nhìn theo góc thuật toán: MoSh/MoSh++ về bản chất là một bài toán **Inverse Kinematics (IK)** tương tự IK per-frame trong retargeting (xem `02-motion-retargeting/`), nhưng thay vì tìm góc khớp robot khớp với vị trí bàn tay/bàn chân mục tiêu, nó tìm (β, θ) của SMPL sao cho vị trí các "marker ảo" gắn sẵn trên mesh SMPL khớp với vị trí marker thật đo được từ camera hồng ngoại — một bài toán tối thiểu hoá sai số bình phương (least-squares) giải bằng gradient descent, tận dụng tính khả vi của SMPL (đã học ở bài giảng trước).

**Giới hạn của loại suy này**: IK per-frame trong retargeting robot thường có RẤT ít điểm mục tiêu (4-6 điểm end-effector: 2 tay, 2 chân, đầu, hông), trong khi MoSh/MoSh++ xử lý HÀNG CHỤC điểm marker đồng thời và phải tách riêng ước lượng β (không đổi suốt phiên) khỏi θ (đổi mỗi khung) — đây là một bài toán tối ưu hoá phức tạp hơn nhiều, không chỉ là "IK nhiều điểm hơn".

## 📐 Định nghĩa chính xác

AMASS (*Archive of Motion Capture as Surface Shapes*, Mahmood, Ghorbani, Troje, Pons-Moll, Black — ICCV 2019) là một dataset hợp nhất **15 bộ mocap optical-marker khác nhau** (CMU Mocap, HumanEva, KIT, TotalCapture, BioMotionLab, và các bộ khác) thành một tham số hoá SMPL/SMPL-X/DMPL thống nhất, thông qua kỹ thuật **MoSh++** (bản cải tiến của **MoSh** — *Motion and Shape capture*, kỹ thuật gốc do Loper et al. 2014 công bố trước AMASS).

**Bài toán tối ưu hoá của MoSh/MoSh++**: cho một khung hình mocap gồm N điểm marker 3D quan sát được `{p_i}`, và một tập "marker ảo" tương ứng gắn cố định trên bề mặt mesh SMPL tại các vị trí đã biết trước (calibration), tìm (β, θ) sao cho:

```
(β*, θ*) = argmin_{β,θ}  Σ_i || p_i  −  M_i(β, θ) ||²  +  regularization
```

trong đó `M_i(β,θ)` là vị trí marker ảo thứ i tính từ mesh SMPL sinh bởi (β, θ), và `regularization` là các số hạng phạt (penalty) giữ θ trong vùng tư thế hợp lý (tránh khớp xoay phi thực tế).

**Điểm khác biệt MoSh++ so với MoSh gốc**: MoSh++ ước lượng **β một lần cho toàn bộ phiên mocap của một người** (vì hình dáng không đổi theo thời gian), sau đó tối ưu **θ riêng cho từng khung hình**, và bổ sung thêm **DMPL** (hệ số dao động mô mềm/soft-tissue, mô phỏng rung động da-mỡ khi chuyển động nhanh) — giúp chuyển động sinh ra tự nhiên hơn ở tốc độ cao (chạy, nhảy) so với MoSh gốc chỉ tối ưu θ mà không có thành phần dao động mô mềm.

**Số liệu chính thức** (theo abstract paper gốc và trang chủ [amass.is.tue.mpg.de](https://amass.is.tue.mpg.de)): *"more than 40 hours of motion data, spanning over 300 subjects, more than 11000 motions"* — tức **>40 giờ, >11,000 chuyển động, >300 chủ thể**, hợp nhất từ **15 bộ mocap optical marker-based**.

> ⚠️ Một số tài liệu thứ cấp trích con số cụ thể hơn "11,265 motions / 344 subjects" — chưa xác minh trực tiếp trong văn bản chính thức, cần coi là **cần xác minh thêm** nếu cần độ chính xác tuyệt đối (giữ nguyên độ dè dặt từ `NOI-DUNG-CHI-TIET.md`).

## ⚙️ Cơ chế hoạt động — từng bước

```
  15 bộ mocap gốc (CMU, HumanEva, KIT, TotalCapture, BioMotionLab, ...)
  mỗi bộ: số marker khác nhau, vị trí gắn khác nhau, định dạng file khác nhau
                              │
                              ▼
        ┌─────────────────────────────────────────────┐
        │  Bước 1: Calibration — xác định marker layout │
        │  của TỪNG bộ mocap, gắn "marker ảo" tương ứng │
        │  lên đúng vị trí bề mặt mesh SMPL              │
        └───────────────────────┬───────────────────────┘
                                 │
                                 ▼
        ┌─────────────────────────────────────────────┐
        │  Bước 2: Ước lượng β (shape) MỘT LẦN cho      │
        │  toàn bộ phiên mocap của 1 chủ thể — dùng      │
        │  nhiều khung hình cùng lúc (β không đổi theo   │
        │  thời gian trong 1 phiên)                       │
        └───────────────────────┬───────────────────────┘
                                 │
                                 ▼
        ┌─────────────────────────────────────────────┐
        │  Bước 3: Với β cố định, tối ưu θ RIÊNG cho     │
        │  TỪNG khung hình bằng gradient descent          │
        │  (least-squares giữa marker ảo và marker thật)  │
        └───────────────────────┬───────────────────────┘
                                 │
                                 ▼
        ┌─────────────────────────────────────────────┐
        │  Bước 4: Ước lượng thêm DMPL (soft-tissue      │
        │  coefficients) để mô phỏng dao động mô mềm     │
        │  ở chuyển động nhanh (chạy, nhảy)               │
        └───────────────────────┬───────────────────────┘
                                 │
                                 ▼
       Chuỗi (β, θ_t, DMPL_t) theo thời gian — 1 "motion"
       trong AMASS, cùng định dạng bất kể nguồn mocap gốc
                                 │
                                 ▼
       Hợp nhất >11,000 motion từ 15 nguồn → AMASS
       (>40 giờ, >300 chủ thể)
```

**Bước 1 — Calibration marker layout**: mỗi bộ mocap gốc có một sơ đồ gắn marker riêng (ví dụ 41 marker ở đâu trên cơ thể) — bước đầu tiên là xác định chính xác các marker này tương ứng với vị trí nào trên bề mặt mesh SMPL, để tạo "marker ảo" — đây là bước thủ công/bán tự động tốn nhiều công sức nhất khi tích hợp một bộ mocap mới.

**Bước 2 — Ước lượng β dùng nhiều khung**: vì một người có hình dáng cố định trong suốt phiên quay, MoSh++ gộp thông tin từ nhiều (thường là toàn bộ) khung hình của phiên đó để ước lượng β chính xác hơn là chỉ dùng 1 khung — giảm nhiễu, tránh trường hợp 1 khung hình có tư thế che khuất (occlusion) marker làm méo ước lượng hình dáng.

**Bước 3 — Tối ưu θ theo từng khung**: với β đã cố định, bài toán còn lại ở mỗi khung hình chỉ là tìm θ (72 chiều với SMPL, nhiều hơn với SMPL-X) sao cho marker ảo khớp marker thật — bài toán nhỏ hơn nhiều so với tối ưu đồng thời (β, θ), hội tụ nhanh và ổn định hơn.

**Bước 4 — DMPL cho dao động mô mềm**: ở chuyển động nhanh (chạy, nhảy, va chạm), da/mỡ dao động theo quán tính mà bản thân khung xương+skinning tuyến tính không nắm bắt được — DMPL thêm một lớp hiệu chỉnh bổ sung dựa trên gia tốc chuyển động, làm chuyển động AMASS trông tự nhiên hơn ở tốc độ cao.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số tự chọn đơn giản hoá để dễ hình dung — không phải số liệu thật.)*

Giả sử một khung hình mocap chỉ có **2 marker** đơn giản hoá: marker A gắn ở cổ tay, marker B gắn ở khuỷu tay, đo được vị trí thật:
```
p_A = (0.50, 1.20, 0.10) m
p_B = (0.35, 1.15, 0.08) m
```

Với β đã ước lượng sẵn (cố định), giả sử ở một giá trị θ thử nghiệm ban đầu `θ⁽⁰⁾`, mesh SMPL sinh ra marker ảo:
```
M_A(β, θ⁽⁰⁾) = (0.48, 1.22, 0.11) m
M_B(β, θ⁽⁰⁾) = (0.36, 1.18, 0.09) m
```

**Bước 1 — Tính sai số bình phương ở θ⁽⁰⁾**:
```
e_A² = (0.50-0.48)² + (1.20-1.22)² + (0.10-0.11)²
     = 0.0004 + 0.0004 + 0.0001 = 0.0009
e_B² = (0.35-0.36)² + (1.15-1.18)² + (0.08-0.09)²
     = 0.0001 + 0.0009 + 0.0001 = 0.0011

Tổng sai số L(θ⁽⁰⁾) = e_A² + e_B² = 0.0020
```

**Bước 2 — Một bước gradient descent (minh hoạ, không tính đạo hàm thật của SMPL vì quá phức tạp cho tay)**: giả sử sau khi lan truyền gradient của L theo θ và cập nhật `θ⁽¹⁾ = θ⁽⁰⁾ − η·∇L`, mesh mới sinh ra:
```
M_A(β, θ⁽¹⁾) = (0.495, 1.205, 0.098) m
M_B(β, θ⁽¹⁾) = (0.352, 1.155, 0.082) m
```

**Bước 3 — Tính lại sai số ở θ⁽¹⁾**:
```
e_A² = (0.005)² + (-0.005)² + (0.002)² ≈ 0.000054
e_B² = (-0.002)² + (-0.005)² + (-0.002)² ≈ 0.000033
Tổng sai số L(θ⁽¹⁾) ≈ 0.000087
```

Sai số giảm từ **0.0020 xuống 0.000087** — minh hoạ đúng cơ chế: mỗi bước gradient descent đưa (β,θ) tới gần hơn giá trị khiến marker ảo khớp marker thật. Với dữ liệu thật, quá trình này lặp lại hàng chục/hàng trăm bước cho tới khi sai số đủ nhỏ (hội tụ), và lặp lại cho **mỗi khung hình** trong toàn bộ phiên mocap (có thể hàng nghìn khung hình cho 1 sequence dài vài phút).

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | AMASS (MoSh++, 2019) | PHUMA (2025) | Motion-X++ (2025) |
|---|---|---|---|
| Nguồn dữ liệu gốc | 15 bộ mocap optical-marker phòng lab | Kết hợp mocap + video internet, xử lý vật lý bổ sung | Video RGB internet + annotation tự động |
| Quy mô | >40 giờ, >11,000 motion, >300 chủ thể | ~73 giờ | 120.5K sequences, 19.5M frame-level pose annotation |
| Độ tin cậy vật lý (physical plausibility) | Phụ thuộc chất lượng marker gốc, một số fit lỗi ở tay/chân | Thiết kế riêng để đảm bảo "physically reliable" — sửa lỗi nổi/trượt chân | Không tối ưu riêng cho tính vật lý — tập trung vào annotation ngôn ngữ |
| Có text annotation? | Không | Không (tập trung vào locomotion sạch) | Có — 120.5K nhãn ngôn ngữ mô tả chuyển động |
| Đa dạng loại chuyển động | Lệch về động tác phòng lab (đi, chạy, bài kiểm tra vận động) | Tập trung locomotion (theo tên "Humanoid Locomotion Dataset") | Đa dạng scene/hoạt động hơn (video internet) |
| Vai trò trong dự án này | Nguồn chính cho retargeting GMR/SOMA | Ứng viên bổ sung nếu cần dữ liệu locomotion "sạch vật lý" hơn AMASS | Ứng viên nếu cần huấn luyện VLA có mô tả ngôn ngữ đi kèm chuyển động |

> Nguồn PHUMA: Lee, Kim, Lee, Park, Kim, Hwang, Kim, Lee, Choo (2025), *"PHUMA: Physically Reliable Humanoid Locomotion Dataset"*, arXiv:2510.26236.
> Nguồn Motion-X++: *"Motion-X++: A Large-Scale Multimodal 3D Whole-body Human Motion Dataset"*, arXiv:2501.05098 (2025).

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "AMASS là mocap gốc, chất lượng đồng đều ở mọi subset."**
   Vì sao sai: AMASS là kết quả FIT (xấp xỉ tối ưu hoá) từ 15 nguồn khác nhau, có nguồn dùng ít marker hơn (dẫn tới θ ước lượng kém chính xác hơn, đặc biệt ở tay/ngón chân) — chất lượng không đồng đều giữa các subset con.
   Hiểu đúng: khi dùng AMASS cho huấn luyện nghiêm túc, cần kiểm tra/lọc theo từng subset gốc thay vì coi toàn bộ AMASS là một khối chất lượng đồng nhất — đây cũng là lý do các paper như PHC lọc AMASS trước khi dùng.

2. **Hiểu nhầm: "Càng dùng nhiều dữ liệu AMASS càng tốt cho huấn luyện motion tracking."**
   Vì sao sai: nghiên cứu gần đây (LIMMT — "Less Is More for Motion Tracking", arXiv:2606.06953, 2026) cho thấy huấn luyện với **dưới 3% AMASS** được chọn lọc kỹ có thể cho kết quả tracking tốt hơn dùng toàn bộ dataset — vì phần lớn dữ liệu dư thừa/trùng lặp hoặc chứa các motion có vấn đề vật lý làm nhiễu quá trình huấn luyện.
   Hiểu đúng: chất lượng và tính đại diện (diversity) của tập con được chọn quan trọng hơn tổng khối lượng dữ liệu thô — đây là nguyên tắc chung khi thiết kế tập huấn luyện cho motion tracking RL.

## 🏗️ Ví dụ minh hoạ trong dự án này

AMASS đóng vai trò "nguồn chuyển động đa dạng nhất" cho pipeline retargeting của dự án: khi GMR hoặc SOMA-retargeter cần một tập lớn chuyển động người đa dạng (đi, chạy, các bài kiểm tra vận động cơ bản) để retarget sang Unitree G1 làm dữ liệu huấn luyện RL/imitation learning, AMASS là lựa chọn mặc định vì tính công khai và quy mô — nhưng README của dự án (mục A) đã ghi rõ rằng AMASS **thiếu tương tác vật thể** và **thiếu annotation ngôn ngữ**, đây chính là lý do OMOMO (tương tác vật thể) và các dataset có text annotation (Motion-X++, HumanML3D — được `README.md` liệt kê ở mục "Cần research thêm") cần được bổ sung song song khi mục tiêu là huấn luyện VLA loco-manipulation đầy đủ.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **PHUMA — phê phán trực tiếp hạn chế vật lý của AMASS** (Lee et al. 2025, arXiv:2510.26236): paper chỉ ra AMASS "đắt tiền và hạn chế về quy mô" so với nguồn video internet, đồng thời nguồn thay thế thuần video (như Humanoid-X) lại gặp lỗi vật lý (nhân vật nổi, xuyên chân đất, chân trượt — foot-skating) mà AMASS ít gặp hơn nhờ mocap marker trực tiếp. PHUMA đề xuất kết hợp cả 2 nguồn qua "physics-aware curation và physics-constrained retargeting", báo cáo hiệu suất tracking cao hơn cả AMASS và Humanoid-X trên benchmark của họ với quy mô ~73 giờ.

2. **LIMMT — dữ liệu ít hơn nhưng chọn lọc tốt hơn thắng dữ liệu nhiều nhưng thô** ("Less is More for Motion Tracking", arXiv:2606.06953, 2026): công trình này huấn luyện motion tracking policy và phát hiện rằng dùng **dưới 3% AMASS** (được chọn lọc) cho kết quả tracking tốt hơn dùng toàn bộ AMASS — một phát hiện quan trọng cho việc thiết kế pipeline dữ liệu: không phải "càng nhiều càng tốt" mà "càng đại diện và sạch càng tốt".

3. **GMT (General Motion Tracking, arXiv:2506.14770, 2025)** báo cáo hiệu suất trên "cả AMASS test set và toàn bộ LAFAN1 dataset" — xác nhận AMASS vẫn là benchmark chuẩn được dùng song song với LAFAN1 trong các paper motion tracking humanoid mới nhất (2025-2026), dù có các nguồn thay thế/bổ sung (PHUMA, Motion-X++) đang nổi lên để giải quyết các hạn chế cụ thể của AMASS (tính vật lý, thiếu annotation ngôn ngữ).

## ❓ Câu hỏi tự kiểm tra

1. Vì sao MoSh++ tách riêng việc ước lượng β (một lần cho cả phiên) khỏi việc tối ưu θ (mỗi khung hình)?
   <details><summary>Gợi ý đáp án</summary>Vì β (hình dáng) không đổi trong 1 phiên mocap của 1 người, gộp thông tin nhiều khung hình giúp ước lượng β chính xác và ổn định hơn; sau đó bài toán mỗi khung chỉ còn tối ưu θ (số chiều nhỏ hơn nhiều so với tối ưu đồng thời β,θ), hội tụ nhanh và ổn định hơn.</details>

2. Con số chính thức về quy mô AMASS là gì, và vì sao con số "480+ người" từng xuất hiện trong dự án là sai?
   <details><summary>Gợi ý đáp án</summary>Chính thức: >40 giờ, >11,000 motion, >300 chủ thể (theo trang chủ amass.is.tue.mpg.de và abstract paper gốc). "480+ người" không khớp với "over 300 subjects" ghi rõ trong nguồn chính thức — đây là lỗi số liệu đã được phát hiện và sửa trong README.md của dự án.</details>

3. Phát hiện của LIMMT (dùng <3% AMASS tốt hơn dùng toàn bộ) có ý nghĩa gì cho việc thiết kế pipeline dữ liệu huấn luyện?
   <details><summary>Gợi ý đáp án</summary>Chất lượng/tính đại diện của tập con dữ liệu quan trọng hơn tổng khối lượng — nên đầu tư vào việc lọc/chọn lọc dữ liệu (loại motion nhiễu, trùng lặp, phi vật lý) thay vì chỉ tăng số lượng dữ liệu thô đưa vào huấn luyện.</details>

4. Vì sao DMPL (soft-tissue coefficients) cần thiết trong MoSh++ nhưng không phải là thành phần bắt buộc của bản thân model SMPL?
   <details><summary>Gợi ý đáp án</summary>DMPL mô phỏng dao động mô mềm (da/mỡ) do quán tính khi chuyển động nhanh — đây là hiệu ứng ĐỘNG (phụ thuộc gia tốc theo thời gian) mà SMPL (một hàm tĩnh của β,θ tại một khung hình) không tự nhiên nắm bắt được; MoSh++ thêm DMPL như một lớp hiệu chỉnh bổ sung riêng cho bài toán khớp dữ liệu mocap động, không phải một phần cố định của định nghĩa SMPL.</details>

5. AMASS thiếu gì mà OMOMO bổ sung, và vì sao thiếu sót đó lại quan trọng cho VLA loco-manipulation?
   <details><summary>Gợi ý đáp án</summary>AMASS chỉ có chuyển động người thuần tuý, không có object trajectory/tương tác vật thể. OMOMO bổ sung đúng dữ liệu tương tác người-vật (nâng, kéo, mang) cần thiết để học quan hệ nhân-quả giữa chuyển động tay và trạng thái vật thể — thứ VLA loco-manipulation cần học.</details>

## 📝 Bài tập thực hành

1. **Đọc/khám phá dữ liệu thật**: nếu có quyền truy cập AMASS (đăng ký tại amass.is.tue.mpg.de), tải 1 subset nhỏ (ví dụ ACCAD hoặc CMU con trong AMASS), dùng thư viện `human_body_prior` chính thức để load 1 file `.npz`, in ra số khung hình, kiểm tra xem β có cố định qua toàn bộ chuỗi hay không (đúng như MoSh++ mô tả).

2. **Tính tay biến thể khác**: lặp lại ví dụ tính sai số bình phương ở trên nhưng với **3 marker** (thêm marker C ở vai, `p_C=(0.20,1.40,0.05)`, marker ảo ban đầu `M_C(θ⁽⁰⁾)=(0.22,1.38,0.06)`) — tính tổng sai số bình phương `L(θ⁽⁰⁾)` mới bao gồm cả 3 marker, so sánh với kết quả chỉ 2 marker trong bài.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

AMASS hợp nhất 15 bộ mocap optical-marker rời rạc, không tương thích định dạng, thành một tham số hoá SMPL/SMPL-X thống nhất bằng kỹ thuật MoSh++ — một bài toán tối ưu hoá tương tự Inverse Kinematics tìm (β, θ) sao cho marker ảo trên mesh SMPL khớp với marker thật đo được, ước lượng β một lần cho cả phiên và θ riêng cho từng khung hình, cộng thêm DMPL cho dao động mô mềm — cho ra kết quả chính thức >40 giờ, >11,000 chuyển động, >300 chủ thể (không phải "480+ người" như số liệu sai từng có trong dự án), nhưng thiên lệch về động tác phòng lab và thiếu tương tác vật thể — hai giới hạn mà PHUMA (tính vật lý) và OMOMO (tương tác vật thể) lần lượt bổ sung, trong khi phát hiện gần đây (LIMMT) cho thấy chọn lọc kỹ một tập con nhỏ của AMASS có thể hiệu quả hơn dùng toàn bộ.
