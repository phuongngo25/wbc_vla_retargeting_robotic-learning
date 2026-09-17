# Nội dung chi tiết — Motion Retargeting

> File này giải thích đầy đủ các khái niệm được liệt kê ở mục A của `README.md`. Mục tiêu: đọc xong file này, bạn hiểu được **cơ chế** (không chỉ tên gọi) mà không bắt buộc phải đọc paper gốc trước. Các citation dùng lại đúng những gì đã kiểm chứng ở README: Gleicher (1998), Aristidou & Lasenby (2011) — FABRIK, Harvey et al. (2020) — Robust Motion In-Betweening/LAFAN1, Villegas et al. (2018) — Neural Kinematic Networks, cùng hai công cụ GMR và SOMA-retargeter.

---

## 1. Định nghĩa bài toán retargeting

**Retargeting** là bài toán: cho một chuyển động nguồn được ghi lại trên một cấu trúc xương (skeleton) A — thường là người, ghi bằng motion capture hoặc ước lượng từ video — hãy tính ra một chuyển động tương đương trên một cấu trúc xương B — robot humanoid — sao cho **ý nghĩa chuyển động** (dáng đi, tư thế, các điểm tiếp xúc như bàn chân chạm đất, bàn tay chạm vật) được giữ lại, trong khi B có hình dạng, tỷ lệ và số bậc tự do khác A.

### Vì sao không thể copy trực tiếp góc khớp

Cách "ngây thơ" nhất — lấy góc khớp đo được ở người (ví dụ góc khuỷu tay, góc gối) và gán thẳng cho khớp tương ứng của robot — thất bại vì ba lý do cấu trúc:

1. **Số bậc tự do (DoF) chênh lệch rất lớn.** Mô hình cơ thể người dùng phổ biến trong nghiên cứu (SMPL/SMPL-X) mô tả khoảng 200+ bậc tự do khi tính cả các khớp nhỏ ở cột sống, vai, ngón tay, mặt. Một robot humanoid thực tế như Unitree G1/H1 chỉ có khoảng 23–43 khớp tuỳ cấu hình (một số bản có tay khéo léo/dexterous hand thì nhiều hơn, một số bản tối giản thì ít hơn). Nghĩa là **không tồn tại ánh xạ 1-1** giữa khớp người và khớp robot cho phần lớn các khớp — nhiều bậc tự do của người phải bị "gộp" hoặc bỏ qua khi ánh xạ sang robot.
2. **Tỷ lệ đoạn chi (limb length ratio) khác nhau.** Ngay cả với các khớp có tương ứng rõ ràng (vai–khuỷu tay–cổ tay), chiều dài xương cánh tay trên/dưới của người và của robot khác nhau, và **tỷ lệ giữa các đoạn** cũng khác (người có thể có tay dài hơn thân so với robot, hoặc ngược lại). Nếu chỉ copy góc khớp mà không xử lý tỷ lệ, đầu mút chi (bàn tay, bàn chân) của robot sẽ **không đến đúng vị trí không gian** mà chuyển động gốc mô tả — ví dụ bàn tay người chạm vào một vật ở độ cao X, nhưng bàn tay robot dừng ở độ cao khác hẳn vì cánh tay ngắn/dài hơn.
3. **Giới hạn vật lý khác nhau.** Góc khớp tối đa, tốc độ motor tối đa của robot thật thường hẹp hơn nhiều so với biên độ chuyển động của người. Một cú xoay hông hay giơ tay nhanh của vũ công hoàn toàn hợp lệ về mặt động học người, nhưng có thể vượt quá giới hạn góc hoặc giới hạn tốc độ motor của robot.

**Hệ quả nếu bỏ qua các khác biệt trên:** nếu chỉ copy góc khớp thô, kết quả trên robot có thể là:
- **"Trật khớp ảo"**: đầu mút chi (tay, chân) bị lệch khỏi vị trí không gian mà chuyển động gốc chủ định, hoặc khớp bị ép quay vượt giới hạn cơ học (trong mô phỏng thì "xuyên" qua giới hạn, trên robot thật thì motor bị stall/hỏng).
- **Mất cân bằng (loss of balance)**: vì trọng tâm (center of mass) của robot phụ thuộc vào tỷ lệ cơ thể của chính nó, một tư thế đứng cân bằng ở người (dựa trên tỷ lệ cơ thể người) không nhất thiết cân bằng khi áp lên tỷ lệ cơ thể robot — đặc biệt nghiêm trọng với chân, vì sai lệch vị trí bàn chân vài centimet có thể khiến robot ngã.
- **Vi phạm tiếp xúc (contact violation)**: bàn chân "trôi" trên sàn (foot sliding) hoặc lơ lửng không chạm đất đúng lúc, phá vỡ tính vật lý của bước đi.

### Ý tưởng gốc: Gleicher (1998) — constraint preservation

Bài toán này được định nghĩa chính xác lần đầu trong đồ hoạ máy tính bởi **Gleicher (1998)**, *"Retargetting Motion to New Characters"* (SIGGRAPH 98). Ý tưởng cốt lõi: thay vì cố giữ nguyên góc khớp tuyệt đối, hãy xác định một tập **constraint** (ràng buộc) cần được bảo toàn trong chuyển động — ví dụ "bàn chân trái chạm sàn tại các frame 10–25", "bàn tay chạm vào tay nắm cửa ở frame 40". Sau đó dùng một bộ giải **spacetime constraints** để tính lại chuyển động trên khung xương mới sao cho các constraint này được tái lập, đồng thời cố gắng giữ lại đặc tính tần số (frequency characteristics) — tức "cảm giác chuyển động" — của tín hiệu gốc càng nhiều càng tốt. Đây chính là lý do mọi công cụ retargeting hiện đại (bao gồm GMR, SOMA-retargeter) đều tách bài toán thành hai phần: (a) tái lập **vị trí không gian** của các điểm quan trọng (đầu mút chi, gốc thân) trên tỷ lệ cơ thể mới, và (b) giải ngược lại góc khớp thoả mãn các vị trí đó — tức là biến bài toán retargeting thành bài toán **Inverse Kinematics** (mục 3).

---

## 2. Skeleton mapping

Trước khi giải IK cho từng frame, cần một bước tiền xử lý gọi là **skeleton mapping** (hay retarget bone mapping): xác định tương ứng giữa cấu trúc xương nguồn (người) và cấu trúc xương đích (robot).

### Ánh xạ khớp nguồn → đích

Đây là một bảng cấu hình (thường ở dạng file YAML/JSON trong các công cụ như GMR) liệt kê, với mỗi khớp/xương quan trọng của robot, xương tương ứng bên phía người — ví dụ: `pelvis ↔ pelvis`, `left_shoulder ↔ left_shoulder`, `left_elbow ↔ left_elbow`, `left_ankle ↔ left_foot`. Không phải mọi khớp đều có tương ứng — chỉ các khớp "mốc" (thường là các khớp chính chịu trách nhiệm định hình dáng — vai, khuỷu, cổ tay, hông, gối, cổ chân, và gốc thân/pelvis) được ánh xạ trực tiếp; các khớp phụ (ngón tay, đốt sống nhỏ) thường bị bỏ qua vì robot không có DoF tương ứng.

### Retarget bone chain

Một **bone chain** là một chuỗi khớp liên tiếp nối từ một điểm gốc (root, ví dụ vai) tới một đầu mút (end-effector, ví dụ cổ tay). Retargeting thường xử lý theo từng chain độc lập: chain tay trái (vai→khuỷu→cổ tay), chain tay phải, chain chân trái (hông→gối→cổ chân), chain chân phải, và chain thân (pelvis→ngực→đầu). Việc chia thành chain giúp bài toán IK ở mục 3 trở thành nhiều bài toán con nhỏ, độc lập, giải song song được thay vì phải giải toàn bộ hệ khớp cùng lúc.

### Scale theo tỷ lệ xương

Sau khi có ánh xạ khớp, hệ thống tính **tỷ lệ chiều dài** giữa từng đoạn xương của robot và đoạn xương tương ứng của người (ví dụ: chiều dài cẳng tay robot / chiều dài cẳng tay người trong file chuyển động nguồn). Tỷ lệ này được dùng để scale lại **vị trí không gian mục tiêu** của từng đầu mút chi trước khi đưa vào IK — nói cách khác, thay vì hỏi "khớp khuỷu tay người đang ở góc bao nhiêu", hệ thống hỏi "nếu giữ đúng tỷ lệ cơ thể robot, thì bàn tay robot cần ở đâu trong không gian để tương ứng với việc bàn tay người đang ở vị trí này so với vai người". Đây chính là bước hiện thực hoá "constraint preservation" của Gleicher: mục tiêu retarget không phải là góc khớp, mà là **vị trí tương đối đã được scale** của đầu mút chi.

### Robot thiếu/thừa bậc tự do so với người

Vì số DoF của robot luôn ít hơn người rất nhiều (mục 1), skeleton mapping phải quyết định cách xử lý phần "thiếu":

- **Robot không có khớp cột sống linh hoạt như người**: người có nhiều đốt sống, cho phép uốn cong lưng mượt mà; hầu hết robot humanoid chỉ có 1–3 khớp ở thân (waist yaw/pitch/roll, hoặc thậm chí thân cứng hoàn toàn). Khi đó, chuyển động uốn lưng của người phải được "dồn" (nếu robot có khớp waist) hoặc **bỏ qua/xấp xỉ bằng chuyển động hông và chân** (nếu thân robot cứng) — nghĩa là một phần thông tin chuyển động gốc chắc chắn bị mất, và người thiết kế pipeline phải chấp nhận đánh đổi này có chủ đích chứ không phải lỗi.
- **Robot thừa bậc tự do so với ánh xạ trực tiếp** (hiếm hơn, ví dụ robot có bàn tay khéo léo nhiều ngón trong khi dữ liệu nguồn không có mocap ngón tay chi tiết): các DoF thừa này thường được giữ ở tư thế mặc định hoặc điều khiển bằng heuristic riêng (ví dụ nắm tay hờ), tách biệt khỏi pipeline retargeting thân/chi chính.

---

## 3. Inverse Kinematics (IK) per-frame

### IK là gì

**Inverse Kinematics (động học ngược)** là bài toán: cho một cấu trúc khớp nối tiếp (kinematic chain) và một **vị trí mục tiêu** trong không gian cho đầu mút của chain (ví dụ "bàn tay phải phải ở toạ độ (x, y, z)"), hãy tìm ra **góc của từng khớp** trong chain sao cho khi áp các góc đó, đầu mút thực sự đến đúng vị trí mục tiêu (hoặc gần nhất có thể, nếu mục tiêu nằm ngoài tầm với). Đây là bài toán ngược của **động học thuận (Forward Kinematics — FK)**: FK dễ giải (biết góc khớp → tính thẳng ra vị trí đầu mút bằng phép nhân ma trận biến đổi liên tiếp), còn IK khó hơn vì thường có vô số nghiệm (dư thừa bậc tự do — redundancy) hoặc vô nghiệm (mục tiêu ngoài tầm với), và không có công thức đóng (closed-form) tổng quát cho chain nhiều khớp.

Trong retargeting, ở **mỗi frame** của chuyển động nguồn, sau khi mục 2 đã tính ra vị trí mục tiêu (đã scale) cho từng đầu mút chi của robot, ta cần giải IK để ra góc khớp robot tại frame đó. Lặp lại cho toàn bộ chuỗi frame → có toàn bộ chuyển động robot. Đây là lý do phần này được gọi là "IK per-frame" (per-frame vì mỗi frame được giải độc lập, khác với các phương pháp tối ưu hoá toàn bộ chuỗi thời gian cùng lúc — trajectory optimization).

### Hai lớp thuật toán giải IK

**(a) Giải tích/Jacobian-based (Newton–Raphson lặp trên không gian khớp):**
Ý tưởng: xây dựng **ma trận Jacobian** J, liên hệ vận tốc khớp (không gian khớp) với vận tốc đầu mút (không gian Cartesian): `Δx = J · Δθ`. Tại mỗi bước lặp, tính sai số giữa vị trí đầu mút hiện tại và vị trí mục tiêu, rồi dùng **giả nghịch đảo (pseudo-inverse)** của J để suy ra một bước cập nhật góc khớp: `Δθ = J⁺ · Δx`, cập nhật `θ ← θ + Δθ`, lặp lại (đây chính là một dạng Newton–Raphson áp dụng cho hệ phương trình động học phi tuyến) cho đến khi sai số đủ nhỏ. Biến thể hiện đại — dùng trong các công cụ như **mink** (thư viện IK mà GMR dựa trên, cùng MuJoCo) — hình thức hoá mỗi bước lặp thành một bài toán **quy hoạch toàn phương (quadratic program — QP)**: tìm vận tốc khớp tối ưu thoả mãn đồng thời nhiều mục tiêu task-space (nhiều đầu mút chi cùng lúc) **và** các ràng buộc (giới hạn góc khớp, giới hạn tốc độ) ngay trong vòng lặp giải, thay vì xử lý ràng buộc như bước hậu kỳ riêng. Đây được gọi là **differential IK**.

**(b) FABRIK (Forward And Backward Reaching Inverse Kinematics — Aristidou & Lasenby, 2011):**
FABRIK giải quyết đúng chain một cách hoàn toàn khác — **không dùng góc khớp hay ma trận xoay** trong quá trình lặp, mà coi mỗi khớp là một điểm trong không gian và mỗi đoạn xương là một đoạn có **chiều dài cố định**. Thuật toán gồm hai pha lặp lại cho tới khi hội tụ:
- **Forward reaching (từ đầu mút về gốc):** đặt đầu mút (end-effector) trực tiếp lên vị trí mục tiêu. Sau đó, đi ngược từng khớp về phía gốc: mỗi khớp được kéo về nằm trên đoạn thẳng nối nó với khớp "con" (đã cập nhật ở bước trước), tại khoảng cách đúng bằng chiều dài xương gốc — tức là **dịch chuyển từng điểm dọc theo đường thẳng** để bảo toàn độ dài đoạn, không xoay gì cả.
- **Backward reaching (từ gốc về đầu mút):** cố định lại điểm gốc (root) về đúng vị trí ban đầu của nó (vì sau pha forward, gốc đã bị "kéo" lệch đi). Sau đó đi xuôi từng khớp từ gốc tới đầu mút, lặp lại đúng thao tác "đặt điểm lên đoạn thẳng nối với khớp cha, cách đúng chiều dài xương" — nhưng theo chiều ngược lại.

Hai pha này được lặp lại (thường chỉ vài lần) cho đến khi khoảng cách giữa đầu mút và mục tiêu đủ nhỏ. Vì mỗi bước chỉ là phép chiếu điểm lên đoạn thẳng (không có nghịch đảo ma trận, không lượng giác phức tạp), **chi phí tính toán mỗi vòng lặp cực thấp** và số vòng lặp cần thiết để hội tụ cũng rất ít trong đa số trường hợp thực tế.

**Vì sao FABRIK phù hợp cho retargeting real-time:** retargeting real-time (ví dụ dùng trong teleoperation — điều khiển robot trực tiếp theo chuyển động người, xem `08-real-robot-deployment/`) cần giải lại IK cho hàng chục chain mỗi frame, ở tần số hàng chục Hz, trên phần cứng có thể chỉ là CPU. FABRIK cho lời giải "đủ tốt về mặt thị giác" (visually plausible) chỉ sau vài vòng lặp giá rẻ, không cần giải hệ phương trình tuyến tính hay nghịch đảo ma trận Jacobian ở mỗi bước — nên **rẻ hơn nhiều lần trên mỗi frame** so với các solver Newton–Raphson/Jacobian đầy đủ khi cần chạy hàng trăm lần mỗi giây cho nhiều chain cùng lúc. Đây là lý do FABRIK trở thành nền tảng phổ biến cho rất nhiều bộ giải IK thời gian thực trong game/animation, dù bản thân GMR (theo tài liệu chính thức) lại chọn hướng (a) — differential IK dựa trên Jacobian/QP qua thư viện `mink` — cho thấy cả hai lớp thuật toán đều có thể đạt real-time tuỳ cách hiện thực hoá và phần cứng mục tiêu.

---

## 4. Ràng buộc vật lý

Giải IK per-frame xong chưa đảm bảo chuyển động **khả thi trên robot thật** — cần thêm một lớp ràng buộc vật lý áp lên kết quả IK.

### Foot contact stabilization (ổn định tiếp xúc chân)

Khi retarget tư thế đứng/đi từ người sang robot có tỷ lệ chân khác, một sai số nhỏ trong lời giải IK ở mỗi frame (do hội tụ chưa hoàn hảo, hoặc do sai số làm tròn khi scale) có thể khiến vị trí bàn chân "trôi" nhẹ qua từng frame — ngay cả khi trong chuyển động gốc, bàn chân đang đứng yên hoàn toàn trên sàn (giai đoạn "stance"). Kết quả là hiện tượng **trượt chân (foot sliding)**: nhìn animation thấy bàn chân robot lướt trên sàn thay vì đứng cố định — vừa sai về mặt vật lý, vừa gây mất ổn định nếu robot thật cố bám theo (vì bộ điều khiển thân dưới giả định chân đang tiếp xúc chắc chắn). Ngược lại, cũng có thể xảy ra hiện tượng **lơ lửng (foot floating)**: bàn chân dừng cách sàn một khoảng nhỏ đáng lẽ phải chạm. Bước **foot contact stabilization** phát hiện các frame mà chuyển động gốc coi là "chân chạm đất" (thường dựa trên vận tốc bàn chân trong dữ liệu gốc — nếu vận tốc gần 0 thì coi là contact), rồi ghim (clamp/lock) vị trí bàn chân robot đứng yên tuyệt đối trong suốt giai đoạn đó, loại bỏ trôi dạt tích luỹ qua các frame.

### Joint limit clamping

Sau IK, góc khớp tính ra có thể vượt giới hạn cơ học thật của robot (do mục tiêu vị trí đôi khi chỉ khả thi nếu khớp xoay quá góc cho phép). Bước này **cắt (clamp)** mọi góc khớp về trong khoảng [góc_min, góc_max] được cấu hình sẵn theo thông số kỹ thuật của từng loại robot (G1, H1, Booster, v.v.), đảm bảo chuyển động xuất ra không bao giờ yêu cầu robot vượt giới hạn vật lý của chính nó.

### Velocity limiting cho motor thật

Ngoài giới hạn góc tĩnh, motor thật còn có **giới hạn tốc độ quay tối đa**. Nếu hai frame liên tiếp có góc khớp chênh lệch quá lớn so với thời gian giữa hai frame (ví dụ dữ liệu nguồn quay rất nhanh, hoặc IK hội tụ về hai nghiệm cách xa nhau ở hai frame gần nhau — hiện tượng "jump" giữa các nghiệm IK không duy nhất), motor thật sẽ không theo kịp. Bước velocity limiting giới hạn tốc độ thay đổi góc khớp giữa các frame liên tiếp về dưới một ngưỡng cấu hình được (ví dụ GMR mặc định dùng ngưỡng khoảng 3π rad/s cho velocity limit của các khớp).

### Liên hệ tới Harvey et al. (2020) — Robust Motion In-Betweening / LAFAN1

Việc coi trọng chất lượng tiếp xúc chân và độ mượt của chuyển động không phải là đặc thù riêng của bước retargeting — nó bắt nguồn từ ngay khâu **sinh/thu thập dữ liệu chuyển động nguồn**. **Harvey, Yurick, Nowrouzezahrai, Pal (2020)**, *"Robust Motion In-Betweening"* (ACM ToG/SIGGRAPH 2020) là paper tạo ra bộ dữ liệu **LAFAN1** (xem `03-human-motion-datasets/`) — họ xây dựng mô hình sinh chuyển động "lấp khoảng trống" (in-betweening) giữa các keyframe sao cho chuyển động sinh ra vẫn giữ được tính vật lý hợp lý, đặc biệt là tiếp xúc chân chính xác và chuyển tiếp mượt. Việc dữ liệu nguồn (LAFAN1) đã được đảm bảo chất lượng tiếp xúc chân tốt từ khâu sinh dữ liệu giúp bước retargeting phía sau (mục 5) có input "sạch" hơn, giảm gánh nặng phải tự "sửa" trôi dạt tiếp xúc — nhưng bước foot contact stabilization ở retargeting vẫn cần thiết độc lập, vì ngay cả input hoàn hảo cũng có thể phát sinh trôi dạt do chính quá trình scale + IK giữa hai skeleton khác tỷ lệ.

---

## 5. Ba trường phái retargeting

### (a) GMR — real-time CPU, per-frame IK

GMR (General Motion Retargeting) là công cụ chính được mentor chỉ định trong dự án này. Theo tài liệu chính thức của GMR, pipeline hoạt động như sau: mỗi frame chuyển động người được biểu diễn dưới dạng một dict ánh xạ `(tên xương người) → (toạ độ toàn cục 3D + góc xoay toàn cục dạng quaternion)`. Sau khi đi qua skeleton mapping + scale (mục 2), GMR dùng một bộ giải IK vi phân (differential IK) xây dựng trên thư viện **`mink`** (kết hợp với **MuJoCo** làm mô hình động học/vật lý) để giải ra `(vị trí + góc xoay gốc robot, góc các khớp actuated)` — thuộc lớp thuật toán giải tích/Jacobian-QP mô tả ở mục 3(a). GMR áp dụng thêm giới hạn tốc độ khớp (velocity limit, mặc định 3π rad/s) như một ràng buộc trong chính vòng lặp giải QP. Nhờ hoàn toàn chạy trên CPU và không cần GPU, GMR đạt tốc độ **60–70 FPS** trên CPU máy trạm mạnh (AMD Threadripper) và **35–45 FPS** trên CPU laptop/desktop phổ thông (Intel i9), đủ nhanh cho ứng dụng **real-time**, kể cả streaming trực tiếp từ thiết bị mocap (Xsens, OptiTrack) hoặc dùng trong teleoperation.

### (b) SOMA-retargeter — GPU batch, Newton + NVIDIA Warp

SOMA-retargeter (NVIDIA) nhắm tới xử lý **hàng loạt (batch)** các file chuyển động BVH (định dạng SOMA-skeleton) thay vì streaming từng frame, tận dụng GPU để tăng thông lượng khi phải retarget một lượng lớn dữ liệu (ví dụ chuẩn bị nhãn huấn luyện cho RL/imitation learning ở quy mô lớn). Theo README chính thức, pipeline gồm đúng 5 bước:

1. **Đọc BVH** — nạp file animation định dạng SOMA-skeleton.
2. **Scale khớp** — retarget các khớp theo tỷ lệ cơ thể người sang tỷ lệ của robot đích (đúng nguyên lý skeleton mapping + scale ở mục 2).
3. **Giải IK từng frame** — chạy trên GPU thông qua **Newton** (thư viện vật lý/robotics của NVIDIA, dùng cho giải các bài toán ràng buộc/động học) kết hợp **NVIDIA Warp** (framework tính toán song song hiệu năng cao trên GPU, cho phép viết kernel Python biên dịch chạy trực tiếp trên GPU) — đây chính là điểm khác biệt cốt lõi so với GMR: IK được giải **song song hàng loạt trên GPU** thay vì tuần tự trên CPU.
4. **Ổn định tiếp xúc + giới hạn khớp** — áp foot contact stabilization và joint limit clamping (mục 4) lên kết quả IK.
5. **Xuất CSV** — ghi ra root pose (vị trí + góc xoay gốc robot) và giá trị các khớp actuated dưới dạng file CSV.

SOMA-retargeter hỗ trợ sẵn cấu hình cho G1, H2, Booster T1, AGIBot X2Ultra/A3T3, và có chế độ **headless** để convert hàng loạt thư mục chuyển động mà không cần viewer tương tác — phù hợp việc xử lý theo lô (batch) quy mô lớn. Tài liệu chính thức cũng lưu ý rõ: chuyển động sinh ra là **thuần kinematic** — cần kiểm chứng tính khả thi/an toàn/tương thích bộ điều khiển trong mô phỏng trước khi đưa lên robot thật (đây chính là lý do các bước 04-imitation-learning-rl/05-simulation-mujoco-isaaclab tồn tại như bước tiếp theo trong pipeline).

### (c) Học sâu/residual — theo tinh thần Villegas et al. (2018)

Khác với hai trường phái trên (đều dựa trên giải trực tiếp bài toán IK theo hình học/vật lý mỗi frame), trường phái thứ ba dùng **mạng nơ-ron để học retargeting**. Đại diện kinh điển là **Villegas, Yang, Ceylan, Lee (2018)**, *"Neural Kinematic Networks for Unsupervised Motion Retargetting"* (CVPR 2018). Ý tưởng chính:

- Một mạng nơ-ron hồi quy (recurrent) nhận chuyển động nguồn và dự đoán ra một chuyển động trên khung xương đích.
- Mạng có một **lớp Forward-Kinematics (FK layer)** được nhúng trực tiếp vào kiến trúc: thay vì mạng phải tự học quan hệ hình học giữa góc khớp và vị trí không gian đầu mút (vốn đã biết chính xác bằng công thức FK), lớp này áp thẳng công thức động học thuận đã biết trước — giúp mạng chỉ cần học phần "khó" (nghiệm IK) chứ không phải học lại cả hình học đã biết.
- Việc học được thực hiện **không giám sát (unsupervised)** — tức không cần cặp dữ liệu (chuyển động nguồn, chuyển động đích "đúng") làm nhãn, vì cặp nhãn như vậy rất khó thu thập ở quy mô lớn. Thay vào đó, mạng dùng một mục tiêu huấn luyện dạng **cycle-consistency**: retarget từ khung xương A sang B rồi retarget ngược lại từ B về A, kết quả phải khớp lại với chuyển động A ban đầu — nếu chu trình này nhất quán, có cơ sở để tin rằng phép retarget trung gian đã giữ đúng "nội dung" chuyển động dù không có nhãn giám sát trực tiếp.

Theo tinh thần này, trong bối cảnh humanoid VLA hiện đại, hướng "học sâu/residual" thường không thay thế hoàn toàn IK hình học, mà **học một phần hiệu chỉnh (residual)** bổ sung sau một bước retarget hình học thô (GMR/SOMA-retargeter) — ví dụ học sửa lỗi cân bằng, hoặc học trực tiếp một policy tracking chuyển động đã retarget (đây chính là ranh giới nối sang `01-whole-body-control/` và `04-imitation-learning-rl/`, nơi retargeting không còn là bước tiền xử lý độc lập mà hoà vào vòng lặp huấn luyện RL/imitation).

### So sánh ba trường phái

| Tiêu chí | (a) GMR — IK giải tích, CPU | (b) SOMA-retargeter — GPU batch | (c) Học sâu/residual (theo Villegas 2018) |
|---|---|---|---|
| Phần cứng | CPU | GPU (Newton + NVIDIA Warp) | GPU (huấn luyện mạng) |
| Chế độ xử lý | Từng frame, streaming real-time | Batch hàng loạt, có chế độ headless | Batch (huấn luyện offline), suy luận có thể online |
| Cơ chế lõi | Differential IK (Jacobian/QP, thư viện `mink`+MuJoCo) | IK từng frame trên GPU + ổn định tiếp xúc/giới hạn khớp | Mạng nơ-ron + FK layer + cycle-consistency (không giám sát) |
| Cần dữ liệu huấn luyện? | Không (thuật toán hình học thuần, không học) | Không (thuật toán hình học thuần, không học) | Có — cần tập chuyển động đa dạng để mạng học tổng quát hoá |
| Ưu điểm chính | Nhanh, real-time, không cần GPU, dễ dùng cho teleoperation | Tận dụng song song GPU cho khối lượng dữ liệu lớn; pipeline rõ ràng 5 bước | Có thể học các quy luật retargeting phức tạp khó mã hoá tường minh (ví dụ tương tác tay–vật thể) |
| Nhược điểm chính | Mỗi frame độc lập → dễ tích luỹ sai số nhỏ (trôi tiếp xúc) nếu không có bước ổn định riêng | Cần hạ tầng GPU; không phải real-time theo nghĩa streaming thấp độ trễ như GMR | Cần dữ liệu + huấn luyện; khó đảm bảo tường minh ràng buộc vật lý cứng (giới hạn khớp/tốc độ) như cách IK hình học làm trực tiếp |
| Vai trò trong dự án này | 🔴 Công cụ chính (mentor chỉ định) | 🔴 Công cụ thứ hai (mentor chỉ định) | 🟡 Hướng nghiên cứu mở rộng, cần tìm thêm citation hiện đại hơn (xem README mục B, ghi chú "cần research thêm") |

---

## Bảng thuật ngữ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| **Retargeting** | Chuyển đổi một chuyển động từ khung xương nguồn (người) sang khung xương đích (robot) khác tỷ lệ/số bậc tự do, giữ lại ý nghĩa chuyển động. |
| **DoF (Degrees of Freedom)** | Bậc tự do — số biến độc lập (thường là góc khớp) cần thiết để mô tả đầy đủ trạng thái của một hệ khớp nối. |
| **Skeleton mapping** | Bước ánh xạ tương ứng giữa các khớp/xương của skeleton nguồn và skeleton đích. |
| **Bone chain (retarget bone chain)** | Một chuỗi khớp liên tiếp từ điểm gốc (root) tới đầu mút (end-effector), được xử lý như một đơn vị trong retargeting/IK. |
| **End-effector** | Đầu mút của một kinematic chain — ví dụ bàn tay, bàn chân — nơi vị trí mục tiêu được đặt ra cho bài toán IK. |
| **Forward Kinematics (FK)** | Tính vị trí đầu mút chi từ góc khớp đã biết — bài toán thuận, có công thức đóng, dễ giải. |
| **Inverse Kinematics (IK)** | Tính góc khớp cần thiết để đầu mút chi đạt một vị trí mục tiêu cho trước — bài toán ngược, thường không có nghiệm duy nhất hoặc không có công thức đóng. |
| **Jacobian (ma trận Jacobian)** | Ma trận liên hệ vận tốc không gian khớp và vận tốc không gian Cartesian của đầu mút; dùng trong các bộ giải IK giải tích/vi phân. |
| **Pseudo-inverse (giả nghịch đảo)** | Suy rộng của nghịch đảo ma trận cho ma trận không vuông/suy biến, dùng để giải xấp xỉ `Δθ = J⁺Δx` trong IK Jacobian-based. |
| **FABRIK** | Forward And Backward Reaching Inverse Kinematics — thuật toán IK lặp hai pha (forward/backward), không dùng góc/ma trận xoay, chỉ dịch chuyển điểm dọc đoạn thẳng bảo toàn chiều dài xương. |
| **Differential IK** | Lớp phương pháp IK giải mỗi bước lặp như một bài toán tối ưu vận tốc khớp cục bộ (thường là QP), có thể tích hợp trực tiếp nhiều mục tiêu và ràng buộc. |
| **Foot contact stabilization** | Ghim vị trí bàn chân đứng yên trong các giai đoạn chân chạm đất, loại bỏ hiện tượng trượt/lơ lửng phát sinh từ sai số IK/scale. |
| **Joint limit clamping** | Cắt giá trị góc khớp về trong khoảng vật lý cho phép của robot sau khi giải IK. |
| **Velocity limiting** | Giới hạn tốc độ thay đổi góc khớp giữa các frame liên tiếp để phù hợp tốc độ quay tối đa của motor thật. |
| **Cycle-consistency** | Mục tiêu huấn luyện không giám sát: retarget A→B rồi B→A phải cho lại kết quả gần với A ban đầu — dùng khi không có cặp nhãn (nguồn, đích) trực tiếp. |

---

## Liên kết chéo

- Quay lại tổng quan và bảng nguồn tham khảo: `README.md`.
- Dữ liệu nguồn cho retargeting: `../03-human-motion-datasets/`.
- Kết quả retarget dùng làm dữ liệu huấn luyện cho: `../01-whole-body-control/`, `../04-imitation-learning-rl/`.
- Retargeting real-time dùng trực tiếp trong teleoperation: `../08-real-robot-deployment/`.
