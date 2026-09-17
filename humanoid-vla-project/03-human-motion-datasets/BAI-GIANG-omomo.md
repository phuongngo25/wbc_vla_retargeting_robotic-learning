# Bài giảng: OMOMO — bài toán object motion guided human motion synthesis

*(Thuộc mảng: Human Motion Datasets)*

## 🎯 Mục tiêu bài học

- Phát biểu chính xác bài toán "object motion guided human motion synthesis" mà OMOMO phục vụ, và giải thích vì sao nó là bài toán ngược khó hơn "sinh chuyển động vật thể từ chuyển động người".
- Mô tả được kiến trúc 2 bước (hand position trước, full-body pose sau) mà OMOMO dùng để giải bài toán, và vì sao cách tiếp cận "diffusion 1 bước" thất bại.
- Nhớ đúng số liệu quy mô dataset OMOMO (~10 giờ, 15 loại vật thể) và cấu trúc dữ liệu (SMPL-H/SMPL-X + object trajectory + mesh vật thể).
- Giải thích được vì sao OMOMO là dữ liệu "duy nhất" trong nhóm 5 dataset của thư mục này có ràng buộc người-vật rõ ràng.
- So sánh được OMOMO với AMASS (thiếu tương tác vật thể) và với CORE4D (một dataset tương tác gần đây khác).
- Nêu được ít nhất 2 điểm cập nhật 2024-2026 về cách OMOMO được dùng trong huấn luyện robot thật (ví dụ OmniRetarget).

## 🧭 Vì sao cần học cái này? (bối cảnh)

Trong 5 khái niệm của thư mục này, OMOMO là dataset **duy nhất** ghi lại đồng thời chuyển động người VÀ trạng thái vật thể mà người đó tương tác — đây chính là dữ liệu cần thiết cho bài toán trung tâm của VLA loco-manipulation: robot vừa di chuyển vừa thao tác vật thể. AMASS (bài giảng trước) và LAFAN1 (bài giảng sau) chỉ ghi chuyển động người "trong chân không", không biết người đang cầm/đẩy/mang gì. Nếu mục tiêu cuối của dự án là huấn luyện một policy loco-manipulation (ví dụ cho GR00T/SONIC), OMOMO là một trong số ít nguồn dữ liệu công khai cung cấp đúng cặp nhãn (chuyển động người, quỹ đạo vật thể) cần thiết để học quan hệ nhân-quả giữa 2 thứ này.

## 🧠 Trực giác

### Góc nhìn 1: Diễn viên đóng thế theo kịch bản đạo cụ (góc nhìn sản xuất phim)

Hãy tưởng tượng một đạo diễn chỉ đưa cho diễn viên đóng thế một "kịch bản chuyển động của đạo cụ" — ví dụ "chiếc vali này sẽ bay từ điểm A lên điểm B theo quỹ đạo cong trong 3 giây" — và yêu cầu diễn viên **tự nghĩ ra cách cầm, cách di chuyển cơ thể** để tạo ra đúng quỹ đạo đó một cách tự nhiên. Đây chính là bài toán OMOMO: cho trước quỹ đạo vật thể (kịch bản đạo cụ), sinh ra chuyển động người hợp lý (diễn xuất của diễn viên) tạo ra quỹ đạo đó.

**Giới hạn của loại suy này**: một diễn viên thật có thể tuỳ ý sáng tạo nhiều cách cầm khác nhau miễn là "trông tự nhiên" theo đánh giá chủ quan của đạo diễn; trong khi OMOMO cần một tiêu chí khách quan hơn — **ràng buộc tiếp xúc vật lý chính xác** (điểm tay chạm đúng bề mặt vật thể, không xuyên qua vật, lực cầm hợp lý) mà một mô hình sinh (generative model) phải học được từ dữ liệu thật, không chỉ "trông giống thật" bằng mắt.

### Góc nhìn 2: Bài toán ngược trong xử lý tín hiệu (góc nhìn kỹ thuật)

Nhìn theo góc kỹ thuật: nếu coi "chuyển động người → chuyển động vật thể" là một phép biến đổi thuận (forward — vật lý xác định: tay di chuyển thế nào thì vật bị đẩy/kéo thế nấy, có thể mô phỏng bằng vật lý), thì OMOMO giải bài toán **ngược** (inverse) — từ output (quỹ đạo vật thể) suy ra input (chuyển động người) đã tạo ra nó. Bài toán ngược luôn khó hơn bài toán thuận vì **không có nghiệm duy nhất** (nhiều cách cầm/di chuyển khác nhau đều có thể tạo ra cùng 1 quỹ đạo vật thể) — đây là lý do OMOMO phải dùng mô hình sinh xác suất (diffusion) thay vì một hàm số xác định (deterministic).

**Giới hạn của loại suy này**: phép loại suy "xử lý tín hiệu ngược" ngụ ý bài toán chỉ có 1 biến đổi vật lý rõ ràng cần đảo ngược — thực tế OMOMO phải xử lý thêm ràng buộc **tiếp xúc rời rạc** (contact — tay chạm/rời vật tại các thời điểm cụ thể, không liên tục như tín hiệu thông thường), khiến bài toán không thuần tuý là "đảo ngược một hàm liên tục" mà còn phải suy luận đúng CẢ pha tiếp xúc lẫn pha không tiếp xúc.

## 📐 Định nghĩa chính xác

**OMOMO** (*Object MOtion guided human MOtion synthesis*, Li, Clegg, Mottaghi, Wu, Puig, Liu — ACM ToG/SIGGRAPH Asia 2023, arXiv:2309.16237) định nghĩa bài toán:

> Cho trước quỹ đạo 6-DOF (vị trí + hướng) theo thời gian của một vật thể `{T_obj(t)}_{t=1}^{N}`, sinh ra chuỗi chuyển động toàn thân người (bao gồm tư thế tay cầm nắm) `{(β, θ_t)}_{t=1}^{N}` sao cho: (a) hợp lý về mặt vật lý (không xuyên vật, tiếp xúc đúng bề mặt khi cần), và (b) tự nhiên như chuyển động người thật.

**Cách giải — 2 quá trình khử nhiễu (denoising) tách biệt** (theo kiến trúc diffusion của paper gốc): thay vì dùng 1 mô hình diffusion sinh trực tiếp toàn bộ pose từ object motion (cách này thất bại trong việc enforce chính xác ràng buộc tiếp xúc tay-vật), OMOMO tách thành:
1. **Denoising quá trình 1**: dự đoán **vị trí bàn tay** (hand positions) từ quỹ đạo vật thể.
2. **Denoising quá trình 2**: tổng hợp **pose toàn thân** dựa trên vị trí bàn tay đã dự đoán ở bước 1 (làm điều kiện/condition).

Cách tách 2 bước này cho phép enforce ràng buộc tiếp xúc chính xác hơn ở bước trung gian (vị trí tay), trước khi mở rộng ra toàn thân.

**Dataset OMOMO (thu thập kèm theo)**: mocap đồng thời người và vật thể khi diễn viên thao tác với **15 loại vật thể sinh hoạt hằng ngày** (máy hút bụi, cây lau nhà, đèn cây, giá treo quần áo, vali, thùng nhựa, ghế gỗ, ghế trắng, bàn lớn, bàn nhỏ, hộp lớn, hộp nhỏ, thùng rác, màn hình, ...), tổng **khoảng 10 giờ** dữ liệu. Mỗi mẫu dữ liệu gồm 3 thành phần: (1) chuyển động người dạng SMPL-H/SMPL-X, (2) quỹ đạo 6-DOF của vật thể theo thời gian, (3) mesh 3D quét sẵn của từng vật thể (để biết chính xác hình học lúc tiếp xúc).

## ⚙️ Cơ chế hoạt động — từng bước

```
     Input: quỹ đạo vật thể T_obj(t), t=1..N  (vị trí + hướng 6-DOF)
     + mesh 3D của vật thể (hình học chính xác, biết trước)
                              │
                              ▼
          ┌───────────────────────────────────────────┐
          │  Denoising process 1:                       │
          │  diffusion model dự đoán vị trí BÀN TAY      │
          │  (hand positions) theo thời gian, điều kiện   │
          │  hoá trên T_obj(t) và hình học vật thể         │
          └───────────────────┬───────────────────────────┘
                               │
                     Hand positions {h_left(t), h_right(t)}
                               │
                               ▼
          ┌───────────────────────────────────────────┐
          │  Denoising process 2:                       │
          │  diffusion model sinh POSE TOÀN THÂN (β,θ_t) │
          │  điều kiện hoá trên hand positions ở bước 1   │
          │  + vẫn tham chiếu T_obj(t) để nhất quán toàn  │
          │  cục (root trajectory hợp lý với vật thể)     │
          └───────────────────┬───────────────────────────┘
                               │
                               ▼
        Output: chuỗi chuyển động người đầy đủ (β, θ_t)
        khớp với quỹ đạo vật thể đầu vào, có tiếp xúc hợp lý
                               │
                               ▼
        (Ứng dụng hạ nguồn — không thuộc OMOMO gốc):
        retarget (β,θ_t) sang skeleton robot (GMR/SOMA)
        → huấn luyện policy loco-manipulation trên robot thật
```

**Bước 1 — Vì sao không sinh trực tiếp toàn thân từ object motion?**: paper gốc chỉ ra rằng áp dụng diffusion model một cách "ngây thơ" (naive) — sinh thẳng toàn bộ pose (β,θ) từ object motion trong 1 quá trình duy nhất — **thất bại trong việc enforce chính xác ràng buộc tiếp xúc** giữa tay và vật thể (tay có thể "trôi" khỏi bề mặt vật hoặc xuyên qua vật trong kết quả sinh ra). Nguyên nhân: không gian pose toàn thân có quá nhiều bậc tự do (tương tự SMPL θ 72 chiều hoặc SMPL-X 119 tham số — xem 2 bài giảng trước), khiến mô hình khó học chính xác ràng buộc cục bộ (tay-vật) khi phải đồng thời sinh cả cấu hình toàn thân.

**Bước 2 — Tách vị trí tay làm biến trung gian**: bằng cách dự đoán trước vị trí bàn tay (một không gian nhỏ hơn nhiều, chỉ 3 chiều × 2 tay theo thời gian, so với toàn bộ pose), mô hình có thể tập trung enforce ràng buộc tiếp xúc chính xác ở không gian nhỏ này trước.

**Bước 3 — Sinh toàn thân điều kiện hoá trên vị trí tay**: sau khi có vị trí tay đáng tin cậy, quá trình khử nhiễu thứ 2 chỉ cần giải bài toán "toàn thân nào tạo ra đúng vị trí tay này" — một dạng bài toán IK ngẫu nhiên (probabilistic IK), dễ hơn bài toán gốc vì đã có mục tiêu trung gian rõ ràng.

**Bước 4 — Thu thập dữ liệu song song người-vật**: khác với mocap người thuần tuý (chỉ cần theo dõi marker trên cơ thể), OMOMO cần đồng thời theo dõi marker/tracker trên VẬT THỂ (để có quỹ đạo 6-DOF chính xác) và quét mesh 3D của từng vật thể trước khi thu thập (để biết chính xác hình học bề mặt lúc tính điểm tiếp xúc) — đây là lý do quy mô OMOMO (~10 giờ) nhỏ hơn nhiều so với AMASS (>40 giờ): chi phí thu thập dữ liệu tương tác vật thể cao hơn nhiều so với mocap người đơn thuần.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số tự chọn để dễ hình dung — không phải số liệu thật từ OMOMO.)*

Giả sử một vật thể (hộp) di chuyển theo quỹ đạo đơn giản: tại thời điểm t=0, hộp ở vị trí `(0.40, 0.90, 0.00)` m (trên bàn), tại t=2s hộp ở vị trí `(0.40, 1.40, 0.00)` m (được nhấc lên cao 0.5m theo trục y, giả sử y là trục thẳng đứng trong hệ quy chiếu minh hoạ này).

**Bước 1 — Suy luận điểm tiếp xúc tay-vật (denoising process 1)**: mô hình cần dự đoán vị trí 2 tay tại mỗi thời điểm sao cho tay "bám" theo bề mặt hộp. Giả sử hộp có kích thước 0.30×0.20×0.20m, mô hình dự đoán 2 tay đặt ở 2 mặt bên đối diện của hộp:
```
t=0: h_left(0) = (0.25, 0.95, 0.00),  h_right(0) = (0.55, 0.95, 0.00)
t=2: h_left(2) = (0.25, 1.45, 0.00),  h_right(2) = (0.55, 1.45, 0.00)
```
Vị trí tay dịch chuyển đúng theo độ dịch của hộp (Δy = 0.5m ở cả 2 tay) — minh hoạ ràng buộc tiếp xúc "tay bám vật" được enforce.

**Bước 2 — Suy luận pose toàn thân (denoising process 2)**: với vị trí tay đã biết ở bước 1 làm điều kiện, mô hình sinh pose toàn thân — ví dụ suy luận rằng để đạt `h_left, h_right` ở độ cao 1.45m (cao hơn ban đầu 0.5m), người phải **gập gối và duỗi thẳng dần** trong 2 giây (chuyển động "nhấc vật lên"), kèm góc khuỷu tay/vai điều chỉnh tương ứng — đây là bước sinh θ_t đầy đủ (72+ chiều tuỳ SMPL/SMPL-X) mà bài toán gốc (nếu sinh trực tiếp từ object motion) sẽ khó đảm bảo nhất quán tay-vật như khi đã có bước trung gian này.

**Ý nghĩa của ví dụ**: minh hoạ đúng lý do kiến trúc 2 bước hiệu quả hơn — bước 1 giải quyết đúng phần khó nhất (ràng buộc tiếp xúc cục bộ) trong không gian nhỏ, bước 2 chỉ cần "điền vào" phần còn lại (toàn thân) với một mục tiêu trung gian đã rõ ràng.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | OMOMO (2023) | AMASS (2019) | CORE4D (2024) |
|---|---|---|---|
| Có tương tác vật thể? | Có — trọng tâm của dataset | Không | Có — tương tác người-người-vật (collaborative) |
| Quy mô | ~10 giờ, 15 loại vật thể | >40 giờ, >11,000 motion | Khác biệt về phạm vi: tập trung tái sắp xếp đồ vật hợp tác giữa nhiều người |
| Định dạng | SMPL-H/SMPL-X + object trajectory + mesh vật thể | SMPL/SMPL-X (không object) | Nhiều người + vật thể (4D human-object-human) |
| Bài toán trung tâm | Sinh chuyển động người từ chuyển động vật thể (object→human) | Không có bài toán sinh có điều kiện — là kho lưu trữ chuyển động thuần | Tái sắp xếp vật thể hợp tác (collaborative object rearrangement) giữa 2+ người |
| Vai trò trong dự án này | Nguồn dữ liệu duy nhất có ràng buộc loco-manipulation rõ ràng cho robot đơn | Nguồn chuyển động đa dạng nhất nhưng không có tương tác vật | Ít liên quan trực tiếp (robot đơn, không phải đa tác nhân hợp tác người-người) |
| Khi nào dùng | Khi cần huấn luyện robot học cách cầm/mang/đẩy vật thể cụ thể | Khi cần tập chuyển động nền tảng đa dạng (đi, chạy, các bài kiểm tra vận động) | Khi nghiên cứu tương tác đa tác nhân (không phải trọng tâm của dự án hiện tại) |

> Nguồn CORE4D: *"CORE4D: A 4D Human-Object-Human Interaction Dataset for Collaborative Object REarrangement"*, arXiv:2406.19353 (2024) — nêu ra để đối chiếu phạm vi, không phải nguồn dữ liệu chính của dự án này.

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "OMOMO là dataset ghi hình người cầm vật, giống hệt AMASS nhưng thêm vật thể."**
   Vì sao sai: OMOMO không chỉ là "AMASS + object" — nó gắn liền với một **bài toán sinh có điều kiện cụ thể** (object motion → human motion) và một kiến trúc giải quyết 2 bước (hand position trước, full-body sau). Bản thân dataset được thu thập RIÊNG cho bài toán này (có mesh 3D vật thể quét sẵn để tính contact chính xác — thứ AMASS không cần vì AMASS không quan tâm vật thể).
   Hiểu đúng: OMOMO vừa là dataset vừa gắn với một phương pháp (diffusion 2 bước) được thiết kế cụ thể cho bài toán loco-manipulation, khác về bản chất với vai trò "kho lưu trữ chuyển động thuần" của AMASS.

2. **Hiểu nhầm: "Vì OMOMO có sẵn 15 loại vật thể, robot huấn luyện trên OMOMO sẽ generalize tốt sang mọi vật thể khác."**
   Vì sao sai: 15 loại vật thể là một tập rất nhỏ so với sự đa dạng vật thể trong đời sống thực (hình dạng, kích thước, trọng lượng, chất liệu khác nhau) — đây chính là giới hạn được ghi rõ trong `NOI-DUNG-CHI-TIET.md`: "quy mô nhỏ, chỉ 15 vật thể, chưa đa dạng đủ cho generalization rộng."
   Hiểu đúng: OMOMO phù hợp làm dữ liệu khởi động (seed data) hoặc benchmark hẹp, không nên coi là đủ để đảm bảo generalization sang vật thể ngoài phân bố huấn luyện mà không có thêm dữ liệu/kỹ thuật domain randomization bổ sung.

## 🏗️ Ví dụ minh hoạ trong dự án này

Theo `README.md` của thư mục này, OMOMO là dataset **duy nhất trong 4 (nay 5 với BONES-SEED) loại có loco-manipulation** — điều này có ý nghĩa trực tiếp: nếu dự án hướng tới huấn luyện policy VLA có khả năng vừa di chuyển vừa thao tác vật thể (ví dụ GR00T/SONIC loco-manipulation), OMOMO là nguồn dữ liệu gần nhất với bài toán mục tiêu, còn AMASS/LAFAN1 chỉ cung cấp nền tảng locomotion thuần tuý. Khi retarget dữ liệu OMOMO sang Unitree G1 (qua GMR hoặc SOMA-retargeter, thư mục `02-motion-retargeting/`), bước retargeting cần xử lý thêm ràng buộc tiếp xúc tay-vật (không chỉ vị trí khớp) để giữ đúng ngữ nghĩa "cầm/mang vật" khi chuyển từ skeleton người sang skeleton robot — đây là một ràng buộc bổ sung mà retargeting từ AMASS/LAFAN1 (không có vật thể) không cần xử lý.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **OmniRetarget — dùng trực tiếp OMOMO để huấn luyện loco-manipulation trên robot thật** (arXiv:2509.26633, 2025): công trình này retarget dữ liệu từ OMOMO (kết hợp với LAFAN1 và mocap nội bộ) thành hơn **8 giờ quỹ đạo** chất lượng cao (giữ ràng buộc tiếp xúc và kinematic tốt hơn baseline), cho phép huấn luyện policy RL proprioceptive thực thi các kỹ năng **loco-manipulation dài hạn (tới 30 giây)** trên Unitree G1 — bao gồm minh hoạ cụ thể: robot mang một chiếc ghế nặng 4.6kg tới một bục, dùng ghế làm bậc đá để leo lên rồi nhảy xuống theo phong cách parkour. Đây là bằng chứng trực tiếp cho thấy OMOMO không chỉ là dataset học thuật mà đã được dùng để tạo dữ liệu huấn luyện thành công trên robot G1 thật trong nghiên cứu gần đây.

2. **CORE4D (arXiv:2406.19353, 2024)** mở rộng hướng "tương tác người-vật" sang bối cảnh **đa tác nhân** (2+ người cùng di chuyển 1 vật thể) — cho thấy xu hướng nghiên cứu 2024-2026 đang mở rộng bài toán OMOMO (1 người - 1 vật) sang các kịch bản phức tạp hơn (nhiều người, nhiều vật, tương tác hợp tác) — dù CORE4D chưa trực tiếp liên quan tới pipeline robot đơn của dự án này.

3. Bài toán gốc của OMOMO (object motion → human motion qua 2 bước diffusion) vẫn là kiến trúc tham chiếu — tra cứu không tìm thấy công trình nào công bố thay thế trực tiếp kiến trúc 2-bước-diffusion của OMOMO; các cải tiến gần đây (OmniRetarget) tập trung vào **cách khai thác dữ liệu OMOMO cho retargeting/huấn luyện robot** hơn là thay đổi bản thân phương pháp sinh chuyển động của OMOMO.

## ❓ Câu hỏi tự kiểm tra

1. Vì sao bài toán "sinh chuyển động người từ chuyển động vật thể" khó hơn bài toán ngược lại ("sinh chuyển động vật thể từ chuyển động người")?
   <details><summary>Gợi ý đáp án</summary>Vì đây là bài toán ngược (inverse problem) không có nghiệm duy nhất — nhiều cách cầm/di chuyển cơ thể khác nhau đều có thể tạo ra cùng một quỹ đạo vật thể, trong khi chiều thuận (người tác động lên vật) được xác định rõ hơn bởi vật lý.</details>

2. Vì sao OMOMO tách thành 2 quá trình denoising (hand position trước, full-body sau) thay vì sinh trực tiếp toàn thân?
   <details><summary>Gợi ý đáp án</summary>Vì sinh trực tiếp toàn thân từ object motion trong 1 bước thất bại trong việc enforce chính xác ràng buộc tiếp xúc tay-vật (do không gian pose toàn thân quá nhiều bậc tự do) — tách vị trí tay làm bước trung gian giúp enforce contact chính xác trước, sau đó bài toán sinh toàn thân trở thành dễ hơn (có điều kiện rõ ràng).</details>

3. Tại sao quy mô OMOMO (~10 giờ) nhỏ hơn nhiều so với AMASS (>40 giờ)?
   <details><summary>Gợi ý đáp án</summary>Vì thu thập dữ liệu OMOMO cần đồng thời theo dõi vật thể (quỹ đạo 6-DOF chính xác) và quét mesh 3D từng vật thể trước khi thu thập, chi phí cao hơn nhiều so với mocap người đơn thuần như các nguồn hợp thành AMASS.</details>

4. OmniRetarget dùng OMOMO như thế nào để huấn luyện robot G1 thực hiện parkour/loco-manipulation?
   <details><summary>Gợi ý đáp án</summary>Retarget dữ liệu OMOMO (kết hợp LAFAN1 + mocap nội bộ) thành >8 giờ quỹ đạo chất lượng cao giữ ràng buộc tiếp xúc/kinematic, dùng làm dữ liệu huấn luyện policy RL proprioceptive thực thi kỹ năng dài hạn (tới 30 giây) như mang ghế, leo bục, nhảy kiểu parkour trên G1.</details>

5. Vì sao "robot huấn luyện trên OMOMO sẽ generalize tốt sang mọi vật thể" là một hiểu nhầm?
   <details><summary>Gợi ý đáp án</summary>Vì OMOMO chỉ có 15 loại vật thể — một tập rất nhỏ so với đa dạng vật thể thực tế (hình dạng, kích thước, chất liệu khác nhau); cần domain randomization hoặc dữ liệu bổ sung để đạt generalization rộng hơn.</details>

## 📝 Bài tập thực hành

1. **Đọc code thật**: clone repo chính thức [`lijiaman/omomo_release`](https://github.com/lijiaman/omomo_release), tìm phần định nghĩa 2 mô hình diffusion (thường có 2 file/class riêng cho "hand prediction" và "full-body synthesis"). Đối chiếu input/output của từng model với 2 bước mô tả ở mục "Cơ chế hoạt động" trên.

2. **Tính tay biến thể khác**: lặp lại ví dụ tính tay ở trên nhưng cho trường hợp vật thể **xoay** (không chỉ tịnh tiến) — giả sử hộp xoay 45° quanh trục thẳng đứng từ t=0 đến t=2s trong khi vẫn tịnh tiến như ví dụ gốc. Mô tả bằng lời cách vị trí 2 tay (h_left, h_right) cần thay đổi để "bám" theo cả chuyển động tịnh tiến LẪN xoay của vật thể, so với ví dụ gốc chỉ có tịnh tiến.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

OMOMO giải bài toán ngược "cho quỹ đạo vật thể, sinh chuyển động người hợp lý" bằng kiến trúc diffusion 2 bước (dự đoán vị trí tay trước, tổng hợp toàn thân sau) để enforce chính xác ràng buộc tiếp xúc tay-vật, đi kèm một dataset ~10 giờ mocap đồng thời người-vật với 15 loại vật thể sinh hoạt (SMPL-H/SMPL-X + object trajectory + mesh 3D vật thể) — đây là nguồn dữ liệu duy nhất trong nhóm 5 dataset của thư mục này có ràng buộc loco-manipulation rõ ràng, đã được chứng minh hữu ích trong thực tế khi OmniRetarget (2025) dùng nó để huấn luyện robot Unitree G1 thực hiện các kỹ năng mang vật và parkour dài hạn, dù quy mô 15 loại vật thể vẫn còn hạn chế cho bài toán generalization rộng.
