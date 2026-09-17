# Bài giảng: Ràng buộc vật lý — Joint limit clamping & Velocity limiting

*(Thuộc mảng: Motion Retargeting)*

## 🎯 Mục tiêu bài học

- Giải thích được vì sao góc khớp tính từ IK per-frame có thể vượt giới hạn cơ học của robot, và vì sao velocity limiting là một ràng buộc độc lập, không suy ra được từ joint limit tĩnh.
- Viết được công thức clamp góc khớp và công thức giới hạn tốc độ thay đổi góc giữa 2 frame liên tiếp.
- Tính tay được một ví dụ số cụ thể: phát hiện vi phạm joint limit và velocity limit, áp dụng clamp, và nhận biết hệ quả phụ (méo động tác) khi clamp quá tay.
- So sánh được cách xử lý "hậu kỳ" (post-processing clamp) với cách xử lý "tích hợp trong solver" (ràng buộc QP ngay trong IK).
- Nêu được ít nhất 2 hiểu nhầm phổ biến về việc clamp góc/tốc độ có "sửa" được hoàn toàn vấn đề retargeting hay không.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Sau khi Inverse Kinematics per-frame (Jacobian-based hoặc FABRIK) tính ra góc khớp thoả mãn vị trí đầu mút mục tiêu, kết quả **không được đảm bảo nằm trong khả năng vật lý của robot thật** — vì bài toán IK về bản chất chỉ tối ưu vị trí hình học, không "biết" động cơ (motor) của robot có xoay nổi tới góc đó hay đủ nhanh để theo kịp hay không. Joint limit clamping và velocity limiting là hai ràng buộc bắt buộc phải áp dụng (dù xử lý ở đâu — hậu kỳ hay tích hợp trong solver) trước khi chuyển động retarget có thể chạy trên robot thật, tiếp nối trực tiếp sau foot contact stabilization trong lớp "ràng buộc vật lý" của pipeline.

## 🧠 Trực giác

### Góc nhìn 1: Hàng rào và giới hạn tốc độ trên đường cao tốc

Joint limit giống như **hàng rào vật lý** ở hai bên đường — xe (khớp) không thể đi ra ngoài phạm vi đường dù người lái muốn. Velocity limit giống như **biển báo giới hạn tốc độ tối đa** — xe có thể đi trong phạm vi đường (không vi phạm hàng rào) nhưng vẫn có thể bị phạt nếu đi quá nhanh giữa hai điểm mốc trong thời gian quá ngắn. Hai ràng buộc này **độc lập nhau**: một chiếc xe hoàn toàn ở trong làn đường (không vi phạm hàng rào) vẫn có thể vi phạm tốc độ nếu tăng tốc đột ngột.

**Giới hạn của loại suy này:** hàng rào đường và biển tốc độ là ràng buộc tĩnh, cố định theo không gian vật lý; trong khi joint limit và velocity limit của robot là ràng buộc trên **một biến trừu tượng** (góc khớp) không trực tiếp là vị trí không gian — một vi phạm giới hạn góc không nhất thiết "nhìn thấy được" trực quan như một chiếc xe đi ra khỏi đường, cần tính toán mới phát hiện ra.

### Góc nhìn 2: Nhạc công chơi đàn quá nhanh so với khả năng ngón tay

Một phép loại suy khác: hình dung một bản nhạc yêu cầu ngón tay nhảy từ phím này sang phím khác trong 1/10 giây — về mặt "vị trí ngón tay cần đạt" điều này hoàn toàn khả thi (phím đàn nằm trong tầm tay), nhưng **tốc độ di chuyển** cần thiết vượt quá khả năng vận động của ngón tay người chơi thật. Velocity limiting giống như việc kiểm tra "liệu ngón tay có kịp di chuyển giữa hai nốt trong thời gian cho phép hay không", độc lập với việc "vị trí phím đàn có nằm trong tầm tay hay không" (đó là vai trò của joint limit).

**Giới hạn của loại suy này:** ngón tay người có thể "học" và cải thiện tốc độ qua luyện tập; motor robot có giới hạn tốc độ **vật lý cố định** theo thông số kỹ thuật (datasheet), không thể cải thiện bằng "luyện tập" — đây là một ràng buộc cứng, không phải một kỹ năng có thể tối ưu hoá thêm.

## 📐 Định nghĩa chính xác

**Joint limit clamping:** cho vector góc khớp θ(t) tính từ IK tại frame t, và giới hạn cơ học `[θ_min, θ_max]` theo cấu hình từng khớp của robot cụ thể (G1, H1, Booster...), phép clamp áp dụng theo từng thành phần (element-wise):

```
θ_clamped_i(t) = max(θ_min_i, min(θ_max_i, θ_i(t)))    với mọi khớp i, mọi frame t
```

**Velocity limiting:** cho hai frame liên tiếp t−1, t cách nhau Δt (nghịch đảo tần số dữ liệu, ví dụ Δt=1/30s ở 30fps), và giới hạn tốc độ tối đa `ω_max_i` của khớp i (theo datasheet motor), tốc độ thay đổi góc tức thời ước lượng bằng sai phân:

```
ω_i(t) = (θ_i(t) − θ_i(t−1)) / Δt
```

Nếu `|ω_i(t)| > ω_max_i`, góc tại frame t được **giới hạn lại** sao cho tốc độ không vượt ngưỡng:

```
θ_i(t)_giới_hạn = θ_i(t−1) + sign(ω_i(t)) · ω_max_i · Δt
```

Theo tài liệu GMR, ngưỡng velocity limit mặc định cho các khớp là khoảng **3π rad/s** (≈ 540°/s) — một ngưỡng khá cao, phản ánh việc GMR ưu tiên giữ độ trung thực chuyển động và chỉ can thiệp khi thực sự cần thiết cho khả thi phần cứng.

**Điểm quan trọng cần phân biệt:** joint limit là ràng buộc **tĩnh** (áp dụng độc lập tại từng frame, không cần biết frame trước), trong khi velocity limit là ràng buộc **động** (liên-frame — cần biết cả θ(t) lẫn θ(t−1) để đánh giá). Một chuỗi góc khớp có thể thoả mãn joint limit ở mọi frame riêng lẻ nhưng vẫn vi phạm velocity limit nếu thay đổi quá nhanh giữa hai frame liên tiếp — đây chính là lý do cả hai ràng buộc đều cần thiết, không thể suy ràng buộc này từ ràng buộc kia.

## ⚙️ Cơ chế hoạt động — từng bước

```
Input: θ(t) từ IK per-frame, t = 1..T
       Cấu hình robot: θ_min_i, θ_max_i, ω_max_i cho mỗi khớp i

┌──────────────────────────────────────────────────────────────┐
│  BƯỚC 1 — Joint limit clamping (áp dụng độc lập từng frame)      │
│    for t = 1 to T:                                                │
│        for mỗi khớp i:                                            │
│            θ_i(t) ← clamp(θ_i(t), θ_min_i, θ_max_i)               │
│                                                                    │
│  BƯỚC 2 — Velocity limiting (áp dụng tuần tự, cần θ(t−1))          │
│    for t = 2 to T:                                                 │
│        for mỗi khớp i:                                            │
│            ω_i = (θ_i(t) − θ_i(t−1)) / Δt                          │
│            if |ω_i| > ω_max_i:                                    │
│                θ_i(t) ← θ_i(t−1) + sign(ω_i)·ω_max_i·Δt            │
│                                                                    │
│  BƯỚC 3 — (nếu velocity limiting làm θ(t) thay đổi so với BƯỚC 1)  │
│           kiểm tra lại joint limit cho θ(t) mới (thứ tự áp dụng    │
│           có thể lặp lại 1-2 vòng nếu 2 ràng buộc "xung đột")      │
└──────────────────────────────────────────────────────────────┘
                    │
                    ▼
        θ_final(t): vừa nằm trong [θ_min, θ_max], vừa có
        tốc độ thay đổi ≤ ω_max giữa mọi cặp frame liên tiếp

── SO SÁNH: cách tích hợp trong solver (thay vì hậu kỳ) ──
Thay vì BƯỚC 1-3 ở trên (áp dụng SAU khi IK đã giải xong),
"differential IK dạng QP" (bài giảng IK Jacobian-based) đưa
θ_min/θ_max và ω_max làm RÀNG BUỘC BÊN TRONG bài toán QP mỗi
bước lặp — nghĩa là nghiệm IK trả về NGAY TỪ ĐẦU đã tôn trọng
2 giới hạn này, không cần bước clamp hậu kỳ riêng.
```

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số tự chọn để dễ hình dung)*

Xét khớp gối robot với giới hạn `θ_min = 0°`, `θ_max = 150°` (gối chỉ gập một chiều, giống gối người), và `ω_max = 3π rad/s ≈ 540°/s` (theo mặc định GMR đã nêu). Dữ liệu ở 30fps, Δt ≈ 0.0333s → trong 1 frame, góc tối đa có thể thay đổi mà không vi phạm velocity limit là `ω_max × Δt = 540 × 0.0333 ≈ 18°/frame`.

**Chuỗi góc gối tính từ IK per-frame** (chưa qua clamp/limit) qua 5 frame:

| Frame t | θ_IK(t) (độ) | Vi phạm joint limit? | Δθ so với t−1 | Vi phạm velocity limit (>18°/frame)? |
|---|---|---|---|---|
| 1 | 20° | Không | — | — |
| 2 | 45° | Không | +25° | **Có** (25° > 18°) |
| 3 | 160° | **Có** (>150°) | +115° | Có |
| 4 | 155° | **Có** (>150°) | −5° | Không |
| 5 | 90° | Không | −65° | Có |

**Bước 1 — Joint limit clamping** (áp dụng độc lập từng frame):
- θ(3): 160° → clamp về 150° (θ_max)
- θ(4): 155° → clamp về 150° (θ_max)
- Các frame khác giữ nguyên.

Chuỗi sau Bước 1: 20°, 45°, 150°, 150°, 90°.

**Bước 2 — Velocity limiting** (áp dụng tuần tự, dùng chuỗi đã clamp ở Bước 1):
- t=2: Δθ = 45−20 = 25° > 18° → giới hạn: θ(2) = θ(1) + 18° = 20+18 = 38°
- t=3: Δθ = 150−38 = 112° > 18° → giới hạn: θ(3) = 38+18 = 56°
- t=4: Δθ = 150−56 = 94° > 18° → giới hạn: θ(4) = 56+18 = 74°
- t=5: Δθ = 90−74 = 16° ≤ 18° → giữ nguyên: θ(5) = 90°

**Kết quả cuối cùng:** 20°, 38°, 56°, 74°, 90° — một chuỗi **tăng đều đặn** thay vì chuỗi gốc dao động mạnh (20→45→160→155→90). Đây chính là hệ quả quan trọng cần nhận biết: velocity limiting không chỉ "cắt bớt" các bước nhảy quá nhanh, mà **làm trễ pha (lag)** toàn bộ chuyển động so với ý định gốc — góc gối tại frame 3 lẽ ra phải gần 150° (theo joint-limit-clamp) nhưng vì velocity limit, nó chỉ đạt 56°, và phải mất thêm nhiều frame sau đó mới "đuổi kịp" giá trị mục tiêu ban đầu (nếu chuỗi dữ liệu tiếp tục đủ dài). Đây là điểm mấu chốt cần hiểu ở mục Sai lầm thường gặp bên dưới: velocity limit quá chặt có thể làm méo động tác nhanh (ví dụ cú đá, nhảy) đáng kể.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Joint limit clamping (hậu kỳ) | Velocity limiting (hậu kỳ) | Tích hợp ràng buộc trong QP (differential IK) |
|---|---|---|---|
| Loại ràng buộc | Tĩnh, độc lập từng frame | Động, phụ thuộc frame trước | Cả hai, đồng thời với chính bước giải IK |
| Chi phí tính toán | Rất thấp (so sánh + min/max) | Thấp (1 phép trừ + so sánh mỗi khớp) | Cao hơn (ràng buộc thêm vào bài toán QP) |
| Rủi ro tác dụng phụ | Có thể làm vị trí đầu mút chi lệch khỏi mục tiêu IK ban đầu (vì clamp góc không tính lại vị trí Cartesian tương ứng) | Có thể gây trễ pha/lag chuyển động (như ví dụ tính tay ở trên) | Nghiệm trả về đã tối ưu đồng thời với ràng buộc — ít "xung đột ẩn" hơn |
| Thời điểm phát hiện vi phạm | Sau khi IK đã giải xong (có thể đã "lãng phí" một nghiệm không khả thi) | Sau khi IK đã giải xong, cần dữ liệu 2 frame | Ngay trong quá trình giải, tránh nghiệm không khả thi từ đầu |
| Công cụ tiêu biểu | SOMA-retargeter bước 4 (áp dụng hậu kỳ) | GMR (áp dụng như ràng buộc mặc định ~3π rad/s) | `mink`/QP (bài giảng IK Jacobian-based) |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "clamp góc khớp về đúng giới hạn là xong, không ảnh hưởng gì tới các phần khác của chuyển động".** Vì sao sai: khi một khớp bị clamp (ví dụ từ 160° về 150° trong ví dụ tính tay), **vị trí đầu mút chi (Cartesian) thực tế sẽ khác** so với vị trí mục tiêu mà IK ban đầu tính ra — vì FK(θ_clamped) ≠ FK(θ_IK) nói chung. Điều này có thể phá vỡ chính ràng buộc mà IK vừa cố gắng thoả mãn (ví dụ bàn chân không còn đúng vị trí đã ổn định ở bước foot contact stabilization). **Hiểu đúng:** sau khi clamp, lý tưởng nên giải lại IK cục bộ (hoặc ít nhất kiểm tra lại sai số vị trí đầu mút) thay vì coi clamp là bước "kết thúc sạch sẽ" không có tác dụng phụ.
2. **Hiểu nhầm: "velocity limit càng chặt (ω_max nhỏ) thì chuyển động robot càng an toàn, nên đặt càng thấp càng tốt".** Vì sao sai: như ví dụ tính tay ở trên cho thấy, velocity limit quá chặt làm chuyển động bị **trễ pha** (lag) tích luỹ — với các động tác nhanh (đá, nhảy, xoay người vũ đạo), việc giới hạn quá mức khiến robot "theo không kịp" chuyển động gốc, biến một cú đá nhanh dứt khoát thành một chuyển động chậm chạp, méo mó về mặt thị giác và có thể còn nguy hiểm hơn (robot phản ứng trễ so với dự định). **Hiểu đúng:** ω_max phải đặt đúng bằng (hoặc sát) giới hạn tốc độ thật của motor theo datasheet — không phải một giá trị an toàn tuỳ ý thấp hơn, vì vừa không cần thiết vừa gây méo động tác không đáng có.
3. **Hiểu nhầm: "joint limit clamping và velocity limiting là 2 cách gọi khác nhau của cùng một ràng buộc".** Vì sao sai: một chuỗi góc có thể **hoàn toàn nằm trong [θ_min, θ_max] ở mọi frame** (không vi phạm joint limit) nhưng vẫn thay đổi quá nhanh giữa 2 frame liên tiếp (vi phạm velocity limit) — như ví dụ tính tay ở trên, sau Bước 1 (chỉ joint limit clamping), chuỗi 20°,45°,150°,150°,90° vẫn hoàn toàn hợp lệ về joint limit nhưng bước nhảy 45°→150° (105°/frame) vẫn vi phạm velocity limit nghiêm trọng. **Hiểu đúng:** đây là hai ràng buộc độc lập, cả hai đều bắt buộc kiểm tra, không thể suy ràng buộc này từ ràng buộc kia.

## 🏗️ Ví dụ minh hoạ trong dự án này

Theo tài liệu chính thức, **GMR** áp giới hạn tốc độ khớp (velocity limit, mặc định 3π rad/s) trực tiếp như một ràng buộc trong chính vòng lặp giải QP (differential IK qua `mink`) — đúng theo cách "tích hợp trong solver" ở cột cuối bảng so sánh trên, giúp tránh vấn đề "clamp hậu kỳ làm lệch vị trí đầu mút" đã nêu ở mục Sai lầm thường gặp #1. Ngược lại, **SOMA-retargeter** thực hiện "ổn định tiếp xúc + giới hạn khớp" như bước 4 riêng biệt, SAU khi đã giải IK trên GPU ở bước 3 — tức theo cách "hậu kỳ". Đây là một khác biệt kiến trúc cụ thể giữa hai công cụ retargeting chính của dự án (xem hai bài giảng riêng "GMR — kiến trúc và pipeline" và "SOMA-retargeter — kiến trúc và pipeline"), phản ánh đúng đánh đổi giữa độ chính xác (tích hợp trong solver, tốn thêm tính toán mỗi bước) và tính đơn giản/module hoá của pipeline (hậu kỳ, dễ implement và debug độc lập từng bước).

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **Xử lý ràng buộc trong chính bài toán tối ưu hoá quỹ đạo là xu hướng chính 2024-2025, thay vì clamp hậu kỳ.** *"Review on Motion Planning of Robotic Manipulator in Dynamic Environments"* (2024) tổng hợp: các phương pháp motion planning hiện đại tích hợp joint limit như bất đẳng thức ràng buộc trực tiếp trong bài toán tối ưu, kèm số hạng làm mượt (smoothness) để giảm thiểu đạo hàm bậc cao của trạng thái robot (vận tốc, gia tốc) — mở rộng ý tưởng velocity limiting ở bài này sang cả gia tốc và giật (jerk). [Wiley 2024](https://onlinelibrary.wiley.com/doi/10.1155/2024/5969512)
2. **Ràng buộc an toàn (joint/velocity/torque limit) được chuẩn hoá theo tiêu chuẩn ISO cho robot cộng tác.** Nghiên cứu *"Near Time-Optimal Trajectories with ISO Standard Constraints for Human–Robot Collaboration"* (2024) cho thấy trong bối cảnh robot làm việc gần người, các giới hạn góc/tốc độ/mô-men không chỉ là thông số kỹ thuật nội bộ mà còn phải tuân theo tiêu chuẩn an toàn quốc tế (ISO) — một góc nhìn mở rộng so với việc chỉ coi joint/velocity limit là ràng buộc "để robot không hỏng" (góc nhìn của retargeting) sang "để robot không gây nguy hiểm cho người xung quanh". [MDPI Robotics 2024](https://doi.org/10.3390/robotics14020010)
3. **Xu hướng xử lý mượt tại biên giới hạn (bound-aware smoothing).** Một số phương pháp hiện đại "respect bounds for joint position, velocity, acceleration, and jerk, with constraints that reduce joint speed to zero when position bounds are approached" — tức thay vì clamp cứng nhắc khi chạm giới hạn (như ví dụ tính tay ở bài này), các hệ thống mới giảm dần tốc độ khi tiệm cận giới hạn góc, tránh việc dừng đột ngột hoặc "dính chặt" ở biên gây giật cục — một cải tiến trực tiếp giải quyết vấn đề nêu ở mục Sai lầm thường gặp #1 (clamp hậu kỳ gây lệch vị trí đầu mút đột ngột). *Đây là mô tả tổng hợp xu hướng chung từ nhiều nguồn tìm được, không gắn với một paper/nhóm cụ thể duy nhất — ghi nhận thận trọng theo đúng tinh thần "không bịa trích dẫn" của bài giảng này.*

## ❓ Câu hỏi tự kiểm tra

1. Vì sao joint limit clamping và velocity limiting là hai ràng buộc độc lập, không thể suy ràng buộc này từ ràng buộc kia?
   <details><summary>Gợi ý đáp án</summary>Joint limit là ràng buộc tĩnh trên giá trị góc tại một frame; velocity limit là ràng buộc động trên tốc độ thay đổi giữa 2 frame — một chuỗi có thể thoả mãn joint limit ở mọi frame riêng lẻ nhưng vẫn thay đổi quá nhanh giữa các frame (vi phạm velocity limit), như minh hoạ trong ví dụ tính tay.</details>
2. Trong ví dụ tính tay, vì sao kết quả cuối cùng (20°,38°,56°,74°,90°) lại "trễ pha" so với chuỗi đã clamp joint limit (20°,45°,150°,150°,90°)?
   <details><summary>Gợi ý đáp án</summary>Vì velocity limiting giới hạn mỗi bước thay đổi tối đa 18°/frame, nên khi chuỗi gốc đòi hỏi bước nhảy lớn hơn (ví dụ 112° ở frame 3), giá trị thực tế chỉ tăng dần 18°/frame mỗi lần, mất nhiều frame hơn mới tiệm cận giá trị mục tiêu ban đầu.</details>
3. Nêu một tác dụng phụ của việc clamp góc khớp về giới hạn mà không giải lại IK.
   <details><summary>Gợi ý đáp án</summary>Vị trí đầu mút chi (Cartesian) tính từ góc đã clamp sẽ khác vị trí mục tiêu ban đầu mà IK nhắm tới (vì FK(θ_clamped) ≠ FK(θ_IK)), có thể phá vỡ các ràng buộc khác đã đạt được trước đó (ví dụ vị trí bàn chân đã ổn định).</details>
4. Vì sao đặt ω_max thấp hơn nhiều so với giới hạn thật của motor không phải là lựa chọn "an toàn hơn" một cách miễn phí?
   <details><summary>Gợi ý đáp án</summary>Vì nó gây trễ pha/lag không cần thiết, làm méo các động tác nhanh (đá, nhảy) một cách không tương xứng với lợi ích an toàn thực sự đạt được, trong khi motor vẫn có khả năng đáp ứng tốc độ cao hơn theo đúng datasheet.</details>
5. Nêu điểm khác biệt kiến trúc giữa cách GMR và SOMA-retargeter xử lý joint/velocity limit.
   <details><summary>Gợi ý đáp án</summary>GMR tích hợp velocity limit trực tiếp như ràng buộc trong bài toán QP của differential IK (trong chính vòng lặp giải); SOMA-retargeter xử lý joint limit clamping như bước hậu kỳ riêng (bước 4) sau khi đã giải IK trên GPU ở bước 3.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể:** với cùng θ_min=0°, θ_max=150°, ω_max=540°/s, Δt=1/30s (giới hạn 18°/frame), áp dụng cả 2 bước (joint limit clamping rồi velocity limiting) cho chuỗi góc IK mới tự chọn: 100°, 170°, 20°, 160°, 80° (5 frame) — trình bày đầy đủ bảng trung gian như ví dụ trong bài.
2. **Đọc code thật:** trong repo GMR, tìm phần cấu hình velocity limit (thường trong file cấu hình robot hoặc solver, tìm từ khoá liên quan tới "velocity limit" hoặc giá trị số gần `3*pi`) — xác nhận lại giá trị mặc định thực tế và xem nó có thay đổi theo từng loại robot (G1 vs H1 vs Booster) hay dùng chung một giá trị.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

Joint limit clamping và velocity limiting là hai ràng buộc vật lý độc lập bắt buộc áp dụng sau (hoặc trong) bước IK per-frame: joint limit clamping cắt góc khớp về đúng phạm vi cơ học `[θ_min, θ_max]` tại từng frame riêng lẻ, còn velocity limiting giới hạn tốc độ thay đổi góc giữa hai frame liên tiếp về dưới ngưỡng motor thật cho phép (mặc định ~3π rad/s trong GMR) — như ví dụ tính tay ở trên cho thấy, hai ràng buộc này có thể vi phạm độc lập nhau và việc áp dụng chúng (đặc biệt velocity limiting) có thể gây tác dụng phụ đáng kể như trễ pha chuyển động nếu áp dụng quá chặt hoặc gây lệch vị trí đầu mút chi nếu clamp mà không giải lại IK; GMR chọn tích hợp velocity limit trực tiếp trong bài toán QP của differential IK để tránh các tác dụng phụ này, trong khi SOMA-retargeter xử lý cả hai như bước hậu kỳ riêng biệt — một đánh đổi kiến trúc cụ thể giữa độ chính xác và tính module hoá mà các nghiên cứu 2024-2025 (motion planning với ràng buộc tích hợp, chuẩn ISO cho an toàn) đang tiếp tục khai thác theo hướng làm mượt tại biên giới hạn thay vì clamp cứng nhắc.
