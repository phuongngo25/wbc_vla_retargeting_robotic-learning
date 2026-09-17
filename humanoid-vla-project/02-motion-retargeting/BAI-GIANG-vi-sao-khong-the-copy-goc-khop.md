# Bài giảng: Vì sao không thể copy trực tiếp góc khớp

*(Thuộc mảng: Motion Retargeting)*

## 🎯 Mục tiêu bài học

- Giải thích được chính xác **3 lý do cấu trúc** khiến việc gán thẳng góc khớp đo từ người sang robot thất bại.
- Tính được một ví dụ số cụ thể cho thấy sai lệch tỷ lệ chi (limb length ratio) làm bàn tay robot "trật" khỏi vị trí mục tiêu bao nhiêu centimet.
- Phân biệt được ba hệ quả cụ thể của việc copy góc khớp thô: "trật khớp ảo", mất cân bằng, vi phạm tiếp xúc.
- Giải thích được vì sao đây là bài toán **cấu trúc** (structural), không phải bài toán có thể "vá" bằng cách chỉnh tham số nhỏ.
- Nêu được vì sao bài toán này là tiền đề bắt buộc phải đọc trước khi học retargeting dựa trên IK (các bài giảng tiếp theo).

## 🧭 Vì sao cần học cái này? (bối cảnh)

Đây là khái niệm **mở đầu** của toàn bộ mảng Motion Retargeting — nó trả lời câu hỏi "tại sao chúng ta cần cả một lĩnh vực nghiên cứu (retargeting) thay vì chỉ đọc số từ file mocap và gán thẳng vào robot?". Nếu không hiểu rõ ba lý do cấu trúc ở đây, các bước tiếp theo trong pipeline — skeleton mapping, Inverse Kinematics per-frame, ràng buộc vật lý — sẽ chỉ là những "thao tác kỹ thuật" vô nghĩa, không hiểu được vì sao chúng bắt buộc phải tồn tại. Khái niệm này chính là "vấn đề" mà toàn bộ các khái niệm sau trong mảng này là "lời giải".

## 🧠 Trực giác

### Góc nhìn 1: Bản đồ giao thông của hai thành phố khác tỷ lệ

Hãy tưởng tượng bạn có một bản chỉ đường chi tiết ("rẽ trái ở góc phố, đi 200m, rẽ phải") được viết cho **thành phố A** (người) — thành phố có rất nhiều ngã tư nhỏ, đường hẹp, khoảng cách giữa các điểm mốc tính bằng vài chục mét. Bây giờ bạn đưa đúng những chỉ dẫn "rẽ trái, đi 200m, rẽ phải" đó cho người lái xe ở **thành phố B** (robot) — thành phố có ít ngã tư hơn nhiều, đường rộng hơn, khoảng cách giữa các điểm mốc có thể gấp đôi hoặc chỉ bằng một nửa. Kết quả: người lái xe ở B làm đúng theo chỉ dẫn về mặt "hành động" (rẽ trái, đi đúng 200m, rẽ phải) nhưng **không hề đến đúng địa điểm** mà chỉ dẫn ở thành phố A dự định đưa họ tới — vì tỷ lệ khoảng cách giữa hai thành phố khác nhau, và một số ngã tư ở A đơn giản không tồn tại ở B nên "rẽ trái ở ngã tư thứ 3" là vô nghĩa.

**Giới hạn của loại suy này:** bản đồ giao thông có ranh giới đường 2D rõ ràng và mỗi bước di chuyển độc lập; trong khi cơ thể là một hệ khớp nối tiếp (kinematic chain) 3D, sai số ở khớp gốc (ví dụ vai) sẽ **cộng dồn và khuếch đại** ở khớp xa hơn (cổ tay) — một sai lệch nhỏ ở vai có thể biến thành sai lệch lớn ở đầu ngón tay, điều mà loại suy bản đồ giao thông không thể hiện được vì các bước rẽ trong ví dụ đó không cộng dồn sai số theo kiểu nhân với chiều dài đòn bẩy.

### Góc nhìn 2: Mượn quần áo của người khác

Một phép loại suy khác, gần cơ thể hơn: hãy nghĩ góc khớp như là "công thức may đo" — số đo góc gập khuỷu tay, góc gập gối tại từng thời điểm. Nếu bạn lấy đúng công thức may đo của một người cao 1m90 tay dài để may quần áo cho một người cao 1m50 tay ngắn, bộ quần áo "đúng số đo góc/hình dạng tương đối" đó sẽ **không vừa** — tay áo quá dài, thân áo lệch. Cơ thể càng lệch tỷ lệ, "quần áo" càng không vừa, dù bạn có sao chép chính xác từng con số góc trong công thức gốc.

**Giới hạn của loại suy này:** quần áo là vật thể tĩnh (may một lần, mặc nhiều lần), trong khi retargeting là vấn đề **động** — mỗi frame chuyển động là một "bộ quần áo" khác nhau cần "may lại" liên tục theo thời gian, và ngoài vấn đề tỷ lệ còn có vấn đề *số lượng bộ phận* khác nhau (robot thiếu "ngón tay", thiếu "đốt sống") mà loại suy may đo không đề cập tới rõ ràng.

## 📐 Định nghĩa chính xác

**Retargeting** là bài toán: cho một chuyển động nguồn ghi trên skeleton A (người, thường có N_A bậc tự do — degrees of freedom, DoF), tính ra một chuyển động tương đương trên skeleton B (robot, N_B bậc tự do, N_B ≪ N_A) sao cho **ý nghĩa chuyển động** (dáng đi, tư thế, các điểm tiếp xúc) được giữ lại.

Việc "copy trực tiếp góc khớp" — tức định nghĩa một ánh xạ đồng nhất θ_B(t) := θ_A(t) cho các khớp có tên tương ứng — thất bại về mặt cấu trúc vì ba lý do độc lập, mỗi lý do đủ để làm sai kết quả dù hai lý do kia không xảy ra:

**(1) Chênh lệch bậc tự do (DoF mismatch).** Mô hình cơ thể người phổ biến trong nghiên cứu — SMPL/SMPL-X (xem `03-human-motion-datasets/`) — mô tả khoảng **200+ bậc tự do** khi tính cả cột sống, vai, ngón tay, mặt. Một robot humanoid thực tế như Unitree G1/H1 chỉ có khoảng **23–43 khớp** tuỳ cấu hình. Không tồn tại ánh xạ 1-1 giữa phần lớn khớp người và khớp robot.

**(2) Chênh lệch tỷ lệ đoạn chi (limb length ratio mismatch).** Ngay cả các khớp có tương ứng rõ ràng (vai–khuỷu–cổ tay), chiều dài từng đoạn xương và **tỷ lệ giữa các đoạn** khác nhau giữa người và robot. Copy góc khớp mà không xử lý tỷ lệ khiến đầu mút chi (end-effector — bàn tay, bàn chân) không đến đúng vị trí không gian mà chuyển động gốc mô tả.

**(3) Chênh lệch giới hạn vật lý (physical limit mismatch).** Góc khớp tối đa (joint limit) và tốc độ motor tối đa (velocity limit) của robot thật hẹp hơn nhiều so với biên độ chuyển động của người.

**Bảng minh hoạ độ chênh lệch DoF theo từng vùng cơ thể** *(số liệu tổng hợp, dùng để minh hoạ quy mô chênh lệch — số chính xác thay đổi theo phiên bản robot/mô hình body model cụ thể)*:

| Vùng cơ thể | SMPL/SMPL-X (người) | Unitree G1 tiêu chuẩn (robot) | Tỷ lệ chênh lệch |
|---|---|---|---|
| Cột sống/thân (spine) | ~3–24 khớp (tuỳ số đốt sống mô hình hoá) | 1–3 khớp (waist yaw/pitch/roll) | ~5–10× |
| Cánh tay (mỗi bên) | ~3 khớp lớn (vai/khuỷu/cổ tay) × nhiều DoF xoay mỗi khớp | 5–7 khớp actuated | ~1–2× |
| Bàn tay/ngón tay (mỗi bên) | ~15 khớp ngón × 2–3 DoF | 0 (tay đơn giản) hoặc dexterous hand riêng | 10×+ hoặc không tương ứng |
| Chân (mỗi bên) | ~3 khớp lớn (hông/gối/cổ chân) × nhiều DoF | 6 khớp actuated | tương đối gần |
| Mặt/biểu cảm | Hàng chục blendshape (SMPL-X) | 0 | Không tương ứng |

Bảng này cho thấy rõ: độ chênh lệch DoF **không đồng đều** giữa các vùng cơ thể — cột sống và bàn tay là nơi mất thông tin nhiều nhất, trong khi chân là nơi tỷ lệ khớp tương đối gần nhau nhất (nhưng lại là nơi hậu quả sai số nghiêm trọng nhất, vì liên quan trực tiếp tới cân bằng — xem mục Cơ chế).

## ⚙️ Cơ chế hoạt động — từng bước

Sơ đồ dưới đây minh hoạ vì sao "copy thô" thất bại, và ba lý do can thiệp ở đâu trong luồng dữ liệu:

```
Chuyển động người (SMPL/SMPL-X, ~200+ DoF)
        │
        │  copy-thẳng θ_robot(t) := θ_người(t) cho khớp "cùng tên"
        ▼
┌───────────────────────────────────────────────────────────┐
│ (1) DoF mismatch:  200+ DoF người → 23–43 DoF robot          │
│     → phần lớn khớp KHÔNG có khớp robot tương ứng            │
│     → thông tin bị bỏ/gộp một cách không kiểm soát           │
├───────────────────────────────────────────────────────────┤
│ (2) Limb ratio mismatch: L_người ≠ L_robot, tỷ_lệ_người ≠     │
│     tỷ_lệ_robot                                               │
│     → dù copy đúng góc, vị trí đầu mút chi (x,y,z) SAI        │
├───────────────────────────────────────────────────────────┤
│ (3) Physical limit mismatch: θ_max_robot < biên độ người,     │
│     ω_max_motor < tốc độ xoay người                           │
│     → góc/​tốc độ yêu cầu VƯỢT giới hạn cơ học robot           │
└───────────────────────────────────────────────────────────┘
        │
        ▼
Kết quả trên robot: trật khớp ảo | mất cân bằng | vi phạm tiếp xúc
```

Mỗi lý do gây ra một loại lỗi quan sát được khác nhau:

1. **"Trật khớp ảo"**: đầu mút chi lệch khỏi vị trí không gian chủ định (do lý do 1, 2), hoặc khớp bị ép quay vượt giới hạn cơ học — trong mô phỏng thì "xuyên" qua giới hạn, trên robot thật motor bị stall/hỏng (do lý do 3).
2. **Mất cân bằng (loss of balance)**: trọng tâm (center of mass — CoM) của robot phụ thuộc vào tỷ lệ cơ thể của chính nó; một tư thế cân bằng ở người không nhất thiết cân bằng khi áp lên tỷ lệ robot — sai lệch vị trí bàn chân vài centimet có thể khiến robot ngã (chủ yếu do lý do 2, cộng hưởng với lý do 1 khi cột sống/hông bị bỏ qua).
3. **Vi phạm tiếp xúc (contact violation)**: bàn chân "trôi" trên sàn (foot sliding) hoặc không chạm đất đúng lúc (do cả 3 lý do cộng hưởng).

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số tự chọn để dễ hình dung — không lấy từ nguồn cụ thể nào)*

Giả sử ta chỉ xét chain tay phải: vai → khuỷu → cổ tay, đơn giản hoá về mặt phẳng 2D để tính tay dễ dàng.

- Người: chiều dài cánh tay trên (vai→khuỷu) L1_người = 30 cm, cẳng tay (khuỷu→cổ tay) L2_người = 25 cm.
- Robot: L1_robot = 22 cm, L2_robot = 18 cm (robot nhỏ hơn, và **tỷ lệ** L2/L1 cũng khác: người 25/30 ≈ 0.833, robot 18/22 ≈ 0.818 — gần nhau trong ví dụ này để minh hoạ ngay cả khi tỷ lệ "gần giống" sai số vẫn xảy ra).

Tại một frame, người có góc vai α = 90° (cánh tay đưa ngang ra trước) và góc khuỷu β = 45° (gập nhẹ). Nếu ta **copy thẳng** α = 90°, β = 45° cho robot mà không scale gì:

Vị trí cổ tay theo công thức động học thuận (Forward Kinematics — FK) 2D đơn giản, gốc toạ độ tại vai, trục x ngang:

```
x_khuỷu = L1 · cos(α)
y_khuỷu = L1 · sin(α)
x_cổ_tay = x_khuỷu + L2 · cos(α + β)
y_cổ_tay = y_khuỷu + L2 · sin(α + β)
```

**Người** (L1=30, L2=25, α=90°, β=45°):
- x_khuỷu = 30·cos(90°) = 0 cm, y_khuỷu = 30·sin(90°) = 30 cm
- α+β = 135°: x_cổ_tay = 0 + 25·cos(135°) = 0 + 25×(−0.707) = −17.68 cm
- y_cổ_tay = 30 + 25·sin(135°) = 30 + 25×0.707 = 47.68 cm
- → Cổ tay người ở (−17.68, 47.68) cm so với vai.

**Robot copy góc thô** (L1=22, L2=18, cùng α=90°, β=45°):
- x_khuỷu = 22·cos(90°) = 0, y_khuỷu = 22·sin(90°) = 22 cm
- x_cổ_tay = 0 + 18·cos(135°) = −12.73 cm
- y_cổ_tay = 22 + 18·sin(135°) = 22 + 12.73 = 34.73 cm
- → Cổ tay robot ở (−12.73, 34.73) cm so với vai robot.

**So sánh theo tỷ lệ cơ thể:** nếu ta *quy đổi đúng cách* (scale theo tỷ lệ cơ thể, cách làm ở bài giảng "Skeleton mapping"), vị trí mục tiêu đúng cho cổ tay robot phải là vị trí người scale theo tỷ lệ chiều dài tổng cánh tay robot/người = (22+18)/(30+25) = 40/55 ≈ 0.727:
- x_mục_tiêu_đúng = −17.68 × 0.727 ≈ −12.86 cm
- y_mục_tiêu_đúng = 47.68 × 0.727 ≈ 34.67 cm

So với kết quả "copy góc thô" ở trên (−12.73, 34.73) cm — trong ví dụ số này hai kết quả *tình cờ* khá gần nhau vì tỷ lệ L2/L1 giữa người và robot đã được chọn gần giống nhau. **Đây chính là điểm mấu chốt cần hiểu:** copy góc thô chỉ "vô tình đúng" khi tỷ lệ đoạn chi giống nhau; hãy thử lại với robot có cẳng tay ngắn bất thường, ví dụ L2_robot = 10 cm (tỷ lệ L2/L1 = 10/22 ≈ 0.455, khác hẳn 0.833 của người):
- x_khuỷu = 22·cos(90°) = 0, y_khuỷu = 22
- x_cổ_tay = 0 + 10·cos(135°) = −7.07 cm
- y_cổ_tay = 22 + 10·sin(135°) = 22 + 7.07 = 29.07 cm
- → Cổ tay robot ở (−7.07, 29.07) cm — lệch xa so với vị trí mục tiêu đúng (nếu scale đúng theo tổng chiều dài (22+10)/55≈0.582: x≈−10.29, y≈27.75) **và lệch càng xa** so với "ý định chuyển động" ban đầu (với ai đó theo dõi chuyển động, bàn tay robot sẽ không vươn đủ xa để chạm vào vị trí mà bàn tay người đang chạm).

**Kết luận số học:** copy góc thô chỉ tình cờ "gần đúng" khi tỷ lệ đoạn chi giữa hai skeleton gần giống nhau — điều **không đảm bảo** trong thực tế (người và robot hiếm khi có cùng tỷ lệ cẳng tay/cánh tay trên, tỷ lệ đùi/ống chân...). Đây chính là lý do lý do (2) trong định nghĩa ở trên là một lỗi cấu trúc, không phải lỗi ngẫu nhiên có thể bỏ qua.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Copy góc khớp thô (naive joint copy) | Retargeting đúng cách (scale + IK — xem các bài giảng sau) |
|---|---|---|
| Vị trí đầu mút chi | Sai lệch, tỷ lệ với chênh lệch tỷ lệ chi | Đúng theo vị trí đã scale theo tỷ lệ cơ thể đích |
| Xử lý DoF thừa của người | Không xử lý — gán tuỳ tiện hoặc bỏ khớp không tên tương ứng | Skeleton mapping quyết định có chủ đích khớp nào giữ/gộp/bỏ |
| Giới hạn vật lý robot | Không kiểm tra — có thể yêu cầu góc/tốc độ vượt giới hạn | Joint limit clamping + velocity limiting áp dụng tường minh |
| Chi phí tính toán | Rẻ nhất (không cần giải gì) | Đắt hơn (giải IK mỗi frame) nhưng khả thi trên robot thật |
| Khi nào "tạm chấp nhận được" | Chỉ khi hai skeleton gần như giống hệt tỷ lệ (ví dụ hai robot cùng dòng, khác màu vỏ) | Luôn cần khi hai skeleton khác loại (người → robot) |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "chỉ cần chỉnh lại vài khớp lệch nhiều nhất là xong".** Vì sao sai: ba lý do (DoF, tỷ lệ, giới hạn vật lý) là lỗi **hệ thống**, ảnh hưởng tích luỹ dọc theo toàn bộ kinematic chain — sai số ở khớp gốc (vai/hông) khuếch đại ở khớp xa (cổ tay/bàn chân) theo đúng công thức FK ở trên (thể hiện qua sin/cos của tổng các góc). Sửa cục bộ vài khớp không giải quyết được lỗi lan truyền qua chain. **Hiểu đúng:** cần một quy trình có chủ đích (skeleton mapping → scale → giải IK theo vị trí mục tiêu) xử lý toàn bộ chain cùng lúc, không phải sửa từng khớp riêng lẻ.
2. **Hiểu nhầm: "nếu robot và người có cùng số khớp ở một chi cụ thể (ví dụ cùng 3 khớp: vai, khuỷu, cổ tay) thì copy góc là an toàn".** Vì sao sai: số khớp giống nhau không đảm bảo **tỷ lệ chiều dài đoạn xương** giống nhau — như ví dụ tính tay ở trên cho thấy, chỉ cần tỷ lệ L2/L1 khác đi, vị trí đầu mút đã lệch đáng kể dù số khớp và góc giống hệt. **Hiểu đúng:** số khớp tương ứng chỉ là điều kiện cần cho bước skeleton mapping (bài giảng riêng), không phải điều kiện đủ để copy góc trực tiếp.

## 🏗️ Ví dụ minh hoạ trong dự án này

Trong pipeline của dự án (GMR/SOMA-retargeter → Unitree G1/H1), đầu vào luôn là dữ liệu chuyển động người dạng SMPL-X (từ AMASS/OMOMO) hoặc BVH (từ LAFAN1), với hàng chục đến hàng trăm khớp/tham số. G1 chỉ có khoảng 23–43 khớp actuated tuỳ phiên bản (có/không có tay khéo léo). Nếu ai đó thử viết một script "đọc góc khớp SMPL, gán thẳng cho khớp G1 cùng tên" — đúng như mô tả ở bài này — kết quả quan sát được sẽ là: robot G1 trong mô phỏng MuJoCo bị "xuyên sàn" hoặc ngã ngay khi chạy động tác đi bộ, vì độ dài chân G1 khác tỷ lệ chân trong dữ liệu AMASS, khiến vị trí bàn chân tính theo góc thô không khớp với mặt sàn. Đây chính là động lực trực tiếp khiến cả GMR và SOMA-retargeter đều bắt buộc có bước skeleton mapping + scale + giải IK (xem các bài giảng tiếp theo) thay vì bất kỳ hình thức copy góc trực tiếp nào.

## 🔥 Cập nhật hiện đại / SOTA gần đây

Tra cứu WebSearch (tháng 9/2026) cho các hướng nghiên cứu 2024–2026 liên quan trực tiếp tới vấn đề "embodiment gap" mà bài này mô tả:

1. **"Embodiment gap" được xác nhận là thuật ngữ chuẩn trong các paper 2025–2026.** Ví dụ bài *"Human2Humanoid: Physics-Aware Cross-Morphology Motion Retargeting for Humanoid Robots"* (arXiv:2606.03476) mô tả rõ: "Motion retargeting for humanoid robots faces substantial embodiment gaps arising from morphological discrepancies in skeletal topology, limb proportions and degrees of freedom (DoFs)" — đúng ba lý do (1)(2)(3) đã nêu ở bài này, dùng ngôn ngữ hiện đại hơn ("embodiment gap") nhưng bản chất vấn đề không đổi so với những gì Gleicher đã mô tả từ 1998. [arXiv:2606.03476](https://arxiv.org/html/2606.03476)
2. **Paper GMR chính thức** — Araujo, Ze, Xu, Wu, Liu, *"Retargeting Matters: General Motion Retargeting for Humanoid Motion Tracking"* (arXiv:2510.02252, ICRA 2026) — định lượng lại đúng hệ quả đã nêu ở mục "Cơ chế": "directly mapping joint values from human data to humanoid robots often leads to non-physical artifacts such as base floating, foot penetration, foot sliding, and non-smooth motions", và chứng minh bằng thực nghiệm rằng **chất lượng retargeting ảnh hưởng trực tiếp tới độ vững (robustness) của policy RL học từ dữ liệu đó** — tức là lỗi copy góc thô không chỉ gây lỗi hình ảnh mà còn lan tới tận chất lượng huấn luyện robot thật. [arXiv:2510.02252](https://arxiv.org/abs/2510.02252)
3. **Xu hướng 2025–2026 đang đi theo hướng giảm phụ thuộc vào ánh xạ khớp thủ công.** Nhiều paper gần đây (ví dụ *"Unified Motion Retargeting for Humanoids with Learned Point Cloud Correspondence"*, arXiv:2609.02134) chỉ ra: "Existing retargeting methods typically address these differences by defining human-robot correspondence through hand-crafted sparse keypoints or body-part pairs" — tức là ngay cả các giải pháp "đúng cách" (skeleton mapping + IK, xem bài giảng sau) hiện tại vẫn phụ thuộc nhiều vào thiết kế thủ công, và nghiên cứu mới đang thử học tương ứng (correspondence) tự động từ dữ liệu thay vì định nghĩa cứng — một bước tiến xa hơn so với vấn đề "copy góc khớp" ở bài này, nhưng cho thấy cả vấn đề gốc lẫn lời giải "kinh điển" đều vẫn đang tiến hoá. [arXiv:2609.02134](https://arxiv.org/abs/2609.02134)

## ❓ Câu hỏi tự kiểm tra

1. Nêu 3 lý do cấu trúc khiến copy góc khớp thô thất bại. Với mỗi lý do, nêu một hệ quả quan sát được trên robot thật.
   <details><summary>Gợi ý đáp án</summary>(1) DoF mismatch — dữ liệu bị mất/gộp không kiểm soát; (2) limb ratio mismatch — đầu mút chi lệch vị trí không gian; (3) physical limit mismatch — góc/tốc độ vượt giới hạn cơ học, motor stall.</details>
2. Trong ví dụ tính tay ở trên, vì sao kết quả "copy góc thô" với L2_robot=18cm lại gần với kết quả "scale đúng", nhưng với L2_robot=10cm thì lại lệch xa?
   <details><summary>Gợi ý đáp án</summary>Vì tỷ lệ L2/L1 của trường hợp L2=18cm (0.818) gần với tỷ lệ của người (0.833), còn tỷ lệ của trường hợp L2=10cm (0.455) lệch xa tỷ lệ người — copy góc thô chỉ "tình cờ đúng" khi tỷ lệ đoạn chi giống nhau.</details>
3. Vì sao mất cân bằng (loss of balance) là hệ quả nghiêm trọng hơn với chân so với tay, dù cả hai đều chịu ảnh hưởng của cùng 3 lý do?
   <details><summary>Gợi ý đáp án</summary>Vì vị trí bàn chân quyết định trực tiếp vùng chống đỡ (support polygon) và mối quan hệ với trọng tâm (CoM) — sai lệch vài centimet ở bàn chân có thể đẩy hình chiếu CoM ra ngoài vùng chống đỡ, gây ngã; sai lệch tương đương ở tay thường chỉ gây "nhìn không tự nhiên" chứ không trực tiếp gây mất cân bằng.</details>
4. Tại sao nói ba lý do này là lỗi "cấu trúc" chứ không phải lỗi "tham số" có thể sửa bằng cách chỉnh một hằng số?
   <details><summary>Gợi ý đáp án</summary>Vì chúng xuất phát từ sự khác biệt bản chất về hình học/topology (số khớp khác nhau, tỷ lệ khác nhau tại mọi cặp khớp, giới hạn cơ khí khác nhau ở mọi khớp) — không tồn tại một hệ số hiệu chỉnh (offset/scale) đơn lẻ áp dụng chung cho mọi khớp, mọi frame, mọi chuyển động để sửa hết cả 3 vấn đề cùng lúc.</details>
5. Nếu robot có tay dài hơn người theo đúng tỷ lệ toàn thân (không có sự khác biệt tỷ lệ đoạn xương), điều đó có loại bỏ hoàn toàn nhu cầu retargeting không? Vì sao?
   <details><summary>Gợi ý đáp án</summary>Không — lý do (1) DoF mismatch và (3) physical limit mismatch vẫn tồn tại độc lập với tỷ lệ chi; retargeting (ít nhất là bước skeleton mapping và giới hạn vật lý) vẫn cần thiết.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** lặp lại ví dụ tính tay ở trên nhưng với chain chân (hông→gối→cổ chân), tự chọn L1_người=45cm, L2_người=40cm, L1_robot=35cm, L2_robot=30cm, góc hông α=100°, góc gối β=−60° (gối gập ngược chiều). Tính vị trí cổ chân cho cả trường hợp "copy góc thô" và "scale đúng theo tổng chiều dài", so sánh sai lệch (cm).
2. **Đọc code thật:** clone repo GMR (`github.com/YanjieZe/GMR`), tìm trong code phần đọc dữ liệu SMPL-X đầu vào — đếm số lượng khớp/tham số thực tế trong một file AMASS mẫu, so sánh với số khớp actuated của cấu hình robot G1 mà GMR hỗ trợ (xem file cấu hình robot trong repo) — xác nhận lại bằng số thật độ chênh lệch DoF đã nêu ở mục Định nghĩa (200+ vs 23–43).

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

Copy trực tiếp góc khớp từ người sang robot thất bại không phải vì một lỗi kỹ thuật nhỏ có thể vá, mà vì ba khác biệt cấu trúc độc lập giữa hai skeleton — số bậc tự do chênh lệch hàng chục lần, tỷ lệ đoạn chi khác nhau khiến vị trí đầu mút chi bị tính sai (như ví dụ tính tay ở trên cho thấy rõ bằng số), và giới hạn góc/tốc độ vật lý của robot hẹp hơn biên độ chuyển động người — mỗi lý do đủ sức tự nó phá vỡ kết quả dù hai lý do còn lại không xảy ra; đây chính là "vấn đề gốc" mà toàn bộ các kỹ thuật retargeting hiện đại (skeleton mapping, Inverse Kinematics per-frame, ràng buộc vật lý, hay các phương pháp học sâu gần đây như học tương ứng point-cloud) đều được thiết kế để giải quyết, và các paper 2025–2026 (GMR, Human2Humanoid) xác nhận rằng chất lượng giải quyết vấn đề này ảnh hưởng trực tiếp tới chất lượng policy robot học được ở các bước sau trong pipeline.
