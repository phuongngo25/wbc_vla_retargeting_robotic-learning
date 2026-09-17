# Bài giảng: Kiến trúc teleoperation qua VR — Open-TeleVision

*(Thuộc mảng: Teleoperation & Triển khai robot thật)*

## 🎯 Mục tiêu bài học

Sau bài này, bạn sẽ:
- Vẽ lại và giải thích được từng khâu trong pipeline Open-TeleVision, từ camera stereo trên robot tới actuator robot.
- Phân biệt được IK real-time (dùng trong teleop) với IK per-frame offline (dùng trong retargeting mảng 02) — vì sao chúng khác nhau về ràng buộc thời gian.
- Giải thích được cơ chế "đồng bộ tỷ lệ" giữa cổ tay-đầu người và end-effector-đầu robot.
- Tính được độ trễ lý thuyết tối thiểu của một vòng lặp teleop chạy ở 60 Hz.
- So sánh được Open-TeleVision với ALOHA (leader-follower) và nêu được khi nào nên chọn kiến trúc nào.
- Nhận diện được 2 hiểu nhầm phổ biến về "egocentric view" và vai trò của CLIK/SLSQP.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Ở bài trước ("Vì sao cần teleoperation — case study ALOHA"), bạn đã thấy teleoperation kiểu leader-follower: 2 tay robot giống hệt nhau, ánh xạ khớp trực tiếp, không cần giải IK. Cách này đơn giản nhưng có giới hạn rõ ràng: nó chỉ hoạt động khi bạn có 2 robot cùng cấu trúc động học, và người vận hành phải đứng ngay cạnh, cầm trực tiếp tay leader — không mở rộng được cho humanoid toàn thân (cả tay, đầu, thân trên) hay cho trường hợp người vận hành và robot không có tay tay giống hệt nhau.

Open-TeleVision giải quyết đúng vấn đề đó: thay vì "tay điều khiển tay", nó dùng **kính VR** để tracking tư thế cơ thể người vận hành (không cần cầm tay robot cụ thể nào), rồi giải bài toán IK trong thời gian thực để ánh xạ sang bất kỳ robot nào — miễn có đủ bậc tự do tương ứng. Đây là bước phát triển tự nhiên tiếp theo trong mảng 08: từ cơ chế cơ khí đơn giản (leader-follower) sang cơ chế toán học tổng quát hơn (VR tracking + real-time IK), mở đường cho các framework tổng quát hơn nữa như XRoboToolkit (bài kế tiếp).

## 🧠 Trực giác

### Góc nhìn 1: "Đội chiếc đầu robot lên đầu mình" (embodiment transfer)

Hãy tưởng tượng bạn đội một chiếc mũ bảo hiểm đặc biệt: camera gắn trên đầu robot truyền thẳng hình ảnh 3D vào 2 mắt bạn, đúng góc nhìn, đúng độ sâu như thể bạn đang đứng ở vị trí đầu robot. Khi bạn quay đầu, camera robot quay theo (gimbal robot bắt chước cổ bạn); khi bạn đưa tay ra, tay robot đưa ra theo. Cảm giác là "tâm trí bạn được chuyển vào cơ thể robot" — đây chính là cách paper Open-TeleVision mô tả trải nghiệm này ("as if the operator's mind is transmitted to a robot embodiment").

**Giới hạn của loại suy này:** đây chỉ là ẩn dụ trải nghiệm (từ góc nhìn người dùng), không phải mô tả cơ chế kỹ thuật. Thực chất không có gì được "chuyển" cả — hệ thống chỉ đang chạy 2 luồng dữ liệu độc lập song song ở 60 Hz: (1) stream video từ robot tới kính, và (2) stream tracking từ kính tới robot qua IK. Nếu một trong hai luồng bị trễ hoặc đứt, ảo giác "nhập vai" biến mất ngay — cho thấy trải nghiệm này phụ thuộc hoàn toàn vào độ trễ hệ thống, không phải một cơ chế thần kỳ nào.

### Góc nhìn 2: "Người phiên dịch song song tức thời" (simultaneous interpreter)

Ở bài ALOHA, việc "dịch" chuyển động leader sang follower là copy 1-1 góc khớp — như dịch từng từ một, không cần hiểu ngữ cảnh. Ở Open-TeleVision, vì người vận hành và robot có cấu trúc cơ thể khác hẳn nhau (người có vai-khuỷu-cổ tay theo tỷ lệ người, robot H1/GR-1 có tỷ lệ khác, số bậc tự do khác), hệ thống phải hoạt động như một **người phiên dịch song song (simultaneous interpreter)**: liên tục nghe ý định của người nói (vị trí tay/đầu người) và ngay lập tức "nói lại" bằng ngôn ngữ robot (góc khớp robot) sao cho *ý nghĩa* (vị trí tương đối end-effector so với đầu) được giữ nguyên, dù "ngữ pháp" (cấu trúc động học) khác nhau — đây chính là việc bộ giải IK (CLIK, SLSQP) đảm nhiệm.

**Giới hạn của loại suy này:** người phiên dịch giỏi hiểu được ngữ cảnh và có thể diễn giải linh hoạt; bộ giải IK thì không "hiểu" gì cả, nó chỉ tối ưu hoá một hàm mục tiêu toán học (giảm sai số vị trí/hướng) trong giới hạn khớp cho phép. Khi vị trí mục tiêu vượt quá tầm với vật lý của robot (ví dụ người vận hành đưa tay ra xa hơn sải tay robot cho phép), bộ giải IK không thể "sáng tạo" ra giải pháp — nó chỉ trả về nghiệm gần đúng nhất có thể, có thể gây chuyển động bất thường gần điểm kỳ dị (singularity).

## 📐 Định nghĩa chính xác

**Open-TeleVision** — Cheng, Li, Yang, Yang, Wang (2024), *"Open-TeleVision: Teleoperation with Immersive Active Visual Feedback"*, CoRL 2024, [arXiv:2407.01512](https://arxiv.org/abs/2407.01512) — là một kiến trúc teleoperation humanoid dựa trên kính VR, gồm hai luồng dữ liệu song song chạy ở 60 Hz:

1. **Luồng phản hồi hình ảnh (visual feedback)**: camera stereo (ZED Mini) gắn trên gimbal 2-3 DOF ở đầu robot → stream video 3D qua web server (framework Vuer) → hiển thị egocentric (góc nhìn thứ nhất) trên kính VR (Apple Vision Pro).
2. **Luồng điều khiển (control)**: tracking tư thế đầu/tay/cổ tay người vận hành trong không gian SE(3) → giải IK real-time (CLIK cho cánh tay, SLSQP/dex-retargeting cho ngón tay) → lệnh khớp cho actuator robot (tay, bàn tay khéo léo, gimbal đầu).

Điểm mấu chốt về mặt định nghĩa: đây là **teleoperation gián tiếp qua ánh xạ hình học thời gian thực (real-time geometric retargeting)**, khác với ALOHA (ánh xạ khớp trực tiếp 1-1) và khác với retargeting offline ở mảng 02 (xử lý toàn bộ file chuyển động đã ghi sẵn, không có ràng buộc tức thời).

## ⚙️ Cơ chế hoạt động — từng bước

```
┌──────────────────────────────────────────────────────────────────────┐
│                  Kiến trúc Open-TeleVision (2 luồng song song)        │
│                                                                        │
│  LUỒNG 1: PHẢN HỒI HÌNH ẢNH (robot → người)                          │
│                                                                        │
│  [Camera stereo ZED Mini trên gimbal đầu robot]                       │
│   (480×640/mắt; H1: gimbal 2-DOF yaw+pitch; GR-1: cổ 3-DOF)           │
│         │                                                              │
│         ▼  60 Hz, qua web server (framework Vuer)                     │
│  [Stream video stereo 3D]                                              │
│         │                                                              │
│         ▼                                                              │
│  [Kính VR Apple Vision Pro hiển thị egocentric 3D]                    │
│         │  (người vận hành "thấy" đúng góc nhìn robot)                │
│         ▼                                                              │
│  ┌───────────────────────────────────────────────────┐                │
│  │        NGƯỜI VẬN HÀNH quan sát & phản ứng          │                │
│  └───────────────────────────────────────────────────┘                │
│         │                                                              │
│         ▼                                                              │
│  LUỒNG 2: ĐIỀU KHIỂN (người → robot)                                  │
│                                                                        │
│  [Kính VR tracking đầu + tay + cổ tay, SE(3), 60 Hz]                  │
│         │                                                              │
│         ▼                                                              │
│  [IK real-time — GIẢI MỖI FRAME, không offline]                       │
│    ├─ Cánh tay: CLIK (Closed-loop IK) trên nền Pinocchio               │
│    └─ Ngón tay: SLSQP (dex-retargeting library)                       │
│         │                                                              │
│         ▼                                                              │
│  [Ràng buộc tỷ lệ: Δ(end-effector, đầu robot) ≈ Δ(cổ tay, đầu người)] │
│         │                                                              │
│         ▼                                                              │
│  [Actuator: cánh tay + bàn tay khéo léo + gimbal đầu robot di chuyển] │
│         │                                                              │
│         └──────────► (vòng lặp quay lại LUỒNG 1, camera thấy kết quả)│
└──────────────────────────────────────────────────────────────────────┘
```

**Diễn giải từng khâu:**

1. **Camera stereo + gimbal**: ZED Mini là camera 2 ống kính, cho depth 3D thật (khác depth ước lượng AI từ 1 ảnh). Gimbal mô phỏng cổ người: H1 dùng gimbal tuỳ chỉnh 2-DOF (yaw + pitch, động cơ Dynamixel XL330-M288-T); GR-1 dùng cổ sẵn có 3-DOF (yaw + roll + pitch) — GR-1 có thêm bậc tự do roll (nghiêng đầu) mà H1 không có.

2. **Stream 60 Hz qua Vuer**: toàn bộ vòng camera → mạng → kính chạy ở tần số cố định 60 Hz — nghĩa là chu kỳ khung hình lý thuyết là 1/60 ≈ 16.7ms. Vuer là framework web server chuyên dụng cho việc này, xử lý mã hoá/giải mã video 3D hiệu quả qua mạng.

3. **Tracking SE(3)**: Apple Vision Pro tracking đầu, tay, cổ tay người vận hành trong không gian 6 bậc tự do (3 vị trí + 3 hướng xoay = SE(3)) — hệ thống được thiết kế "agnostic" (không phụ thuộc kính cụ thể), nghĩa là về nguyên tắc có thể thay bằng kính khác miễn hỗ trợ tracking tương tự.

4. **IK real-time — khác biệt cốt lõi với retargeting offline (mảng 02)**: retargeting offline có thể xử lý toàn bộ chuỗi chuyển động cùng lúc, tối ưu hoá qua nhiều vòng lặp (ví dụ làm mượt quỹ đạo, sửa lỗi tương lai dựa trên khung hình sau). Ở đây, hệ thống CHỈ có dữ liệu của khung hình hiện tại, phải trả lời "góc khớp là bao nhiêu NGAY BÂY GIỜ" trong vài mili-giây, không được nhìn trước tương lai — đây là ràng buộc "online/causal" điển hình của mọi hệ điều khiển thời gian thực.
   - **CLIK (Closed-loop Inverse Kinematics)**: thuật toán IK dạng vòng lặp đóng — liên tục tính sai số giữa vị trí hiện tại và vị trí mục tiêu, cập nhật góc khớp theo hướng giảm sai số (thường dùng đạo hàm Jacobian ngược hoặc giả nghịch đảo) — phù hợp chạy real-time vì mỗi bước tính rẻ, không cần giải tối ưu toàn cục.
   - **SLSQP (Sequential Least-Squares Quadratic Programming)** cho ngón tay qua thư viện dex-retargeting: bài toán retargeting bàn tay phức tạp hơn cánh tay (nhiều khớp hơn, nhiều ràng buộc tiếp xúc ngón tay hơn), nên cần một bộ giải tối ưu có ràng buộc mạnh hơn CLIK đơn thuần.

5. **Ràng buộc tỷ lệ end-effector↔đầu**: thay vì ánh xạ tuyệt đối (vị trí tay người trong phòng = vị trí tay robot trong phòng — vô nghĩa vì 2 không gian khác nhau), hệ thống ánh xạ **tương đối**: khoảng cách từ tay tới đầu của người vận hành được scale/copy thành khoảng cách từ end-effector tới đầu robot. Đây là lý do người vận hành "cảm thấy tự nhiên" dù cơ thể robot khác tỷ lệ người.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

**Bài toán 1 — Độ trễ lý thuyết tối thiểu của vòng lặp 60 Hz:**

Tần số 60 Hz nghĩa là hệ thống xử lý 60 khung hình mỗi giây. Chu kỳ một khung hình:
$$T_{frame} = \frac{1}{60} \approx 0.01667\ s = 16.67\ ms$$

Đây là độ trễ **tối thiểu về mặt lý thuyết cho MỘT khung hình** của luồng camera hoặc luồng tracking (nếu không có xử lý gì thêm). Trong thực tế, độ trễ tổng end-to-end (từ lúc camera robot chụp ảnh tới lúc mắt người nhìn thấy, CỘNG THÊM từ lúc tay người cử động tới lúc robot phản ứng) sẽ lớn hơn nhiều vì phải cộng thêm: độ trễ mã hoá/giải mã video, độ trễ truyền mạng (network latency), độ trễ tính IK. Bài giảng "XRoboToolkit" (bài kế tiếp) có số liệu đo thực tế: XRoboToolkit đo được độ trễ end-to-end trung bình cho baseline Open-TeleVision là **121.50 ms** — gấp khoảng 7.3 lần chu kỳ khung hình lý thuyết 16.67ms, cho thấy phần lớn độ trễ thực tế đến từ mạng + xử lý, không phải từ tần số camera.

**Bài toán 2 — Ước lượng tỷ lệ ánh xạ end-effector↔đầu (ví dụ minh hoạ, số tự chọn để dễ hình dung, không lấy từ paper):**

Giả sử người vận hành đưa tay ra phía trước một khoảng cách 40cm tính từ đầu (tức khoảng cách cổ tay-đầu người tăng từ 20cm lên 60cm, Δ = 40cm). Giả sử tỷ lệ cơ thể robot so với người là 0.9 (robot có sải tay ngắn hơn người 10%, ví dụ minh hoạ). Khi đó:

$$\Delta_{robot} = \Delta_{human} \times k = 40\ cm \times 0.9 = 36\ cm$$

→ End-effector robot sẽ di chuyển ra xa đầu robot thêm 36cm (thay vì đúng 40cm như người vận hành), theo đúng tỷ lệ cơ thể robot. Đây là lý do robot với sải tay ngắn hơn người **không thể** với xa bằng người vận hành thật — một giới hạn vật lý mà bộ giải IK phải tôn trọng (không thể "vượt" quá tầm với thật của robot dù người vận hành cố đưa tay xa hơn).

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Open-TeleVision (VR + real-time IK) | ALOHA (leader-follower, không VR) |
|---|---|---|
| Giao diện điều khiển | Kính VR (Apple Vision Pro), không cầm gì | Tay cầm leader vật lý (WidowX), cầm trực tiếp |
| Cần giải IK? | Có, bắt buộc mỗi khung hình (CLIK + SLSQP) | Không, chỉ copy góc khớp trực tiếp |
| Yêu cầu về cấu trúc robot | Linh hoạt — bất kỳ robot nào đủ bậc tự do tương ứng | Robot phải cùng cấu trúc động học với leader |
| Phản hồi cho người vận hành | Hình ảnh stereo 3D egocentric qua kính | Nhìn màn hình camera thường (không 3D, không đội kính) |
| Độ phức tạp triển khai | Cao hơn (cần thư viện IK, tracking VR, mạng ổn định) | Thấp hơn (chủ yếu là phần cứng cơ khí + mapping đơn giản) |
| Phạm vi áp dụng | Toàn thân (tay, ngón tay, đầu) trên humanoid | Chủ yếu cánh tay/gripper (không có phần đầu/toàn thân trong ALOHA gốc) |
| Độ trễ điển hình | Có nhưng đo được cụ thể (~121.5ms baseline theo XRoboToolkit) | Thấp hơn về lý thuyết (không qua bước giải IK phức tạp) nhưng paper gốc không công bố số đo cụ thể |

**Khi nào dùng cái nào:** dùng ALOHA khi bạn có sẵn cặp tay robot giống hệt nhau và chỉ cần thao tác tay/gripper đơn giản, ưu tiên triển khai nhanh, chi phí thấp. Dùng Open-TeleVision (hoặc XRoboToolkit) khi cần điều khiển toàn thân humanoid (đầu, tay, ngón tay khéo léo) với robot có cấu trúc khác người vận hành, và cần trải nghiệm immersive để thao tác chính xác các task dài hơi, nhiều bước.

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "Egocentric view" nghĩa là chỉ cần 1 camera thường gắn ở đầu robot** — nhiều người nhầm "góc nhìn thứ nhất" chỉ đơn giản là đặt camera ở vị trí đầu. *Vì sao sai:* nếu chỉ dùng 1 camera thường (monocular), người vận hành sẽ không có cảm nhận độ sâu (depth) — rất khó đánh giá khoảng cách khi thao tác cầm/nắm vật. *Hiểu đúng:* Open-TeleVision dùng camera **stereo** (2 ống kính, như ZED Mini) kết hợp kính VR có 2 màn hình riêng cho 2 mắt — đây là điều kiện cần để "egocentric" thực sự mang lại cảm nhận 3D, không chỉ là góc nhìn 2D từ vị trí đầu.

2. **Hiểu nhầm: "CLIK và SLSQP là cùng một thuật toán, chỉ khác tên gọi"** — vì cả hai đều là "bộ giải IK" nên dễ nhầm là tương đương. *Vì sao sai:* CLIK là một họ thuật toán IK dạng vòng lặp đóng dựa trên Jacobian (đơn giản, nhanh, phù hợp bậc tự do trung bình như cánh tay); SLSQP là một thuật toán tối ưu hoá tổng quát hơn (Sequential Least-Squares Quadratic Programming), xử lý được nhiều ràng buộc phức tạp hơn — cần thiết cho bàn tay khéo léo (nhiều khớp, nhiều ràng buộc tiếp xúc ngón tay cùng lúc). *Hiểu đúng:* việc paper chọn 2 thuật toán khác nhau cho tay và ngón tay phản ánh đúng sự khác biệt về độ phức tạp bài toán IK giữa hai bộ phận này, không phải ngẫu nhiên hay dư thừa.

3. **Hiểu nhầm: "Robot bắt chước y hệt 100% chuyển động người"** — vì thấy robot "theo" chuyển động người mượt mà, dễ nghĩ là sao chép tuyệt đối. *Vì sao sai:* như ví dụ tính tay ở trên, ánh xạ là theo **tỷ lệ tương đối** (end-effector↔đầu robot so với cổ tay↔đầu người), và luôn bị giới hạn bởi tầm với vật lý, giới hạn khớp của robot cụ thể. *Hiểu đúng:* đây là một phép retargeting hình học thời gian thực có ràng buộc, không phải sao chép tuyệt đối — càng gần giới hạn vật lý của robot, sai lệch so với ý định người vận hành càng lớn.

## 🏗️ Ví dụ minh hoạ trong dự án này

Trong pipeline của dự án, kiến trúc kiểu Open-TeleVision chính là nền tảng khái niệm cho quy trình teleop chính thức của GR00T-WholeBodyControl (bài giảng riêng, mục 4 của mảng 08) — cả hai đều dùng kính VR tracking người vận hành và ánh xạ real-time sang robot, dù GR00T-WholeBodyControl dùng framework XRoboToolkit/PICO thay vì Vuer/Apple Vision Pro. Hiểu Open-TeleVision trước sẽ giúp bạn hiểu ngay tại sao quy trình GR00T yêu cầu "Wi-Fi tốc độ cao, độ trễ thấp" — vì bất kỳ hệ VR teleop nào cũng nhạy cảm với độ trễ mạng theo đúng cơ chế đã phân tích ở mục Ví dụ tính tay.

Nếu robot của bạn (theo README mục D) là G1 hoặc H1/GR-1-like, dữ liệu thu qua kiến trúc này (ảnh + tracking + góc khớp mỗi frame) có thể dùng trực tiếp để fine-tune GR00T N1.x (mảng 06) — cùng cấu trúc dữ liệu (quan sát, hành động) như đã nói ở bài ALOHA, chỉ khác nguồn tạo ra hành động (IK real-time thay vì copy khớp trực tiếp).

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **Isaac Lab 2.3 — Enhanced Teleoperation (NVIDIA, cuối 2025)** — theo blog kỹ thuật chính thức [NVIDIA Developer Blog](https://developer.nvidia.com/blog/streamline-robot-learning-with-whole-body-control-and-enhanced-teleoperation-in-nvidia-isaac-lab-2-3), Isaac Lab bản 2.3 đã tích hợp trực tiếp tinh thần của Open-TeleVision vào framework simulation công nghiệp: hỗ trợ teleop qua **Meta Quest VR và găng tay Manus (Manus gloves)**, cùng "dexterous retargeting" cho robot Unitree G1 — dịch cấu hình bàn tay người sang góc khớp bàn tay robot để cải thiện hiệu năng cho task contact-rich. Đây là bằng chứng rõ ràng rằng kiến trúc "VR tracking + real-time retargeting" từ Open-TeleVision (2024) đã trở thành tính năng chuẩn trong công cụ mô phỏng robot học chính thống chỉ hơn 1 năm sau khi paper công bố.

2. **Mở rộng lên hệ đa camera/multi-view (2025-2026)** — một hướng nghiên cứu tiếp nối trực tiếp là *"A Multi-View 3D Telepresence System for XR Robot Teleoperation"* ([arXiv:2604.03730](https://arxiv.org/pdf/2604.03730)), mở rộng ý tưởng stereo 2 camera của Open-TeleVision lên **nhiều camera/nhiều góc nhìn** để tăng vùng quan sát và giảm điểm mù (blind spot) — một hạn chế thực tế của thiết kế gimbal 2-3 DOF gốc (chỉ xoay được trong phạm vi giới hạn, không bao phủ 360°).

3. **Bổ sung phản hồi lực (force feedback) — Prometheus (2025)** — *"Prometheus: Universal, Open-Source Mocap-Based Teleoperation System with Force Feedback for Dataset Collection in Robot Learning"* ([arXiv:2510.01023](https://arxiv.org/pdf/2510.01023)). Đây là câu trả lời trực tiếp cho một giới hạn đã nêu ở phần Trực giác của bài này: Open-TeleVision (và ALOHA) đều chỉ có phản hồi **hình ảnh**, không có phản hồi **lực** cho người vận hành — nghĩa là người vận hành phải suy luận lực tiếp xúc hoàn toàn qua thị giác. Các hệ thống như Prometheus đang bổ sung thêm kênh phản hồi lực (haptic) thật sự, hướng tới giải quyết đúng khoảng trống này trong tương lai gần.

4. **Không có gì thay thế "camera stereo + VR tracking" như nguyên lý cốt lõi** — điểm đáng chú ý là dù các hệ thống 2024-2026 (Isaac Lab 2.3, XRoboToolkit, GR00T-WholeBodyControl) đều dùng phần cứng/framework khác Open-TeleVision gốc (PICO thay Apple Vision Pro, ZED Mini vẫn phổ biến), nguyên lý cốt lõi — stereo camera cho depth thật + tracking SE(3) + IK real-time — vẫn được giữ nguyên, không có kiến trúc thay thế hoàn toàn khác nổi lên tính đến thời điểm tra cứu. Đây là dấu hiệu cho thấy đây là kiến trúc nền tảng ổn định của lĩnh vực, các cải tiến chủ yếu là mở rộng phần cứng hỗ trợ và giảm độ trễ, không phải thay đổi nguyên lý.

## 📎 Thuật ngữ nhanh dùng trong bài

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Stereo camera | Camera 2 ống kính, tạo được chiều sâu (depth) 3D thật từ chênh lệch góc nhìn hai mắt — khác với ước lượng độ sâu bằng AI từ 1 ảnh. |
| Egocentric view | Góc nhìn thứ nhất — người xem thấy đúng như thể họ đang ở vị trí/góc nhìn của robot. |
| Gimbal | Cơ cấu xoay có động cơ riêng, dùng để mô phỏng chuyển động cổ (yaw/pitch/roll) cho camera đầu robot. |
| SE(3) | Không gian biểu diễn vị trí + hướng xoay 3D đầy đủ của một vật thể cứng (rigid body). |
| CLIK | Closed-loop Inverse Kinematics — thuật toán IK vòng lặp đóng, tính lại liên tục theo sai số hiện tại, phù hợp real-time. |
| SLSQP | Sequential Least-Squares Quadratic Programming — thuật toán tối ưu hoá có ràng buộc, dùng cho retargeting bàn tay. |
| Singularity (điểm kỳ dị) | Cấu hình khớp mà robot mất tạm thời một hoặc nhiều bậc tự do điều khiển — IK thường kém ổn định gần đây. |
| Pinocchio | Thư viện tính toán động lực học rigid-body mã nguồn mở, phổ biến làm nền cho các bộ giải IK trong robot học. |

## ⏱️ Vì sao tần số và độ trễ được đo tách biệt

Một điểm dễ gây nhầm lẫn: "tần số 60 Hz" và "độ trễ 121.5ms" không mâu thuẫn nhau, vì chúng đo hai đại lượng khác nhau:

- **Tần số (frequency)** đo *tốc độ lặp lại* của vòng lặp — bao nhiêu khung hình/lệnh điều khiển được xử lý mỗi giây. 60 Hz nghĩa là cứ 16.67ms lại có MỘT khung hình mới được xử lý.
- **Độ trễ (latency)** đo *thời gian trễ* giữa lúc một sự kiện xảy ra (camera robot chụp ảnh, hoặc tay người di chuyển) và lúc kết quả của nó được nhận biết ở đầu bên kia (mắt người nhìn thấy, hoặc robot phản ứng).

Một hệ thống có thể chạy ở tần số rất cao (60 Hz, xử lý khung hình rất nhanh) NHƯNG vẫn có độ trễ lớn (121.5ms) nếu mỗi khung hình phải đi qua nhiều bước xử lý tuần tự (mã hoá video → truyền mạng → giải mã → hiển thị). Ẩn dụ dễ hình dung: một dây chuyền sản xuất có thể ra 60 sản phẩm/giây (thông lượng — throughput cao), nhưng một sản phẩm cụ thể vẫn mất 2 giây để đi hết dây chuyền từ đầu tới cuối (độ trễ — latency riêng của nó vẫn cao). Hai chỉ số này cần được tối ưu hoá độc lập.

## ❓ Câu hỏi tự kiểm tra

1. Vì sao Open-TeleVision cần giải IK real-time trong khi ALOHA thì không?
<details><summary>Gợi ý đáp án</summary>Vì ALOHA dùng leader và follower cùng cấu trúc động học, nên chỉ cần copy góc khớp trực tiếp. Open-TeleVision dùng người vận hành (tracking SE(3) qua VR) điều khiển robot có cấu trúc cơ thể khác hẳn, nên cần bộ giải IK để chuyển đổi vị trí/hướng mục tiêu sang góc khớp phù hợp với robot cụ thể.</details>

2. Phân biệt "IK real-time" ở bài này với "IK per-frame" đã học ở mảng 02 (retargeting offline) — điểm khác biệt cốt lõi là gì?
<details><summary>Gợi ý đáp án</summary>Cả hai đều giải IK cho từng frame, nhưng retargeting offline có thể xử lý toàn bộ chuỗi frame cùng lúc (nhìn được frame tương lai, tối ưu hoá làm mượt quỹ đạo qua nhiều lần lặp). IK real-time trong teleop chỉ có dữ liệu frame hiện tại, phải trả lời tức thời trong vài ms, không được nhìn trước.</details>

3. Tại sao hệ thống ánh xạ theo "tỷ lệ tương đối" (end-effector↔đầu robot so với cổ tay↔đầu người) thay vì ánh xạ tuyệt đối?
<details><summary>Gợi ý đáp án</summary>Vì không gian toạ độ của người vận hành (trong phòng) và không gian robot là 2 hệ khác nhau — ánh xạ tuyệt đối không có ý nghĩa. Ánh xạ tương đối giữ được "ý định chuyển động" (đưa tay ra xa/gần đầu bao nhiêu) bất kể tỷ lệ cơ thể khác nhau giữa người và robot.</details>

4. Trong ví dụ tính tay, nếu tỷ lệ cơ thể robot k = 1.2 (robot có sải tay DÀI hơn người 20%) và người vận hành đưa tay ra thêm 30cm, end-effector robot sẽ di chuyển thêm bao nhiêu?
<details><summary>Gợi ý đáp án</summary>Δ_robot = 30cm × 1.2 = 36cm — robot với sải tay dài hơn sẽ "phóng đại" chuyển động của người vận hành theo đúng tỷ lệ.</details>

5. Vì sao CLIK phù hợp cho cánh tay nhưng bàn tay khéo léo lại cần SLSQP thay vì cũng dùng CLIK?
<details><summary>Gợi ý đáp án</summary>Bàn tay khéo léo có nhiều khớp hơn và nhiều ràng buộc tiếp xúc/hình học phức tạp hơn cánh tay (nhiều ngón, mỗi ngón nhiều khớp, cần giữ hình dạng nắm hợp lý) — SLSQP là một bộ giải tối ưu có ràng buộc mạnh hơn, xử lý được các ràng buộc phức tạp này tốt hơn CLIK đơn giản.</details>

6. Dựa trên xu hướng Isaac Lab 2.3 và Prometheus, hai hướng phát triển chính tiếp theo của kiến trúc kiểu Open-TeleVision là gì?
<details><summary>Gợi ý đáp án</summary>(1) Tích hợp vào các framework công nghiệp/simulation chuẩn (Isaac Lab) với hỗ trợ nhiều thiết bị VR/glove hơn; (2) bổ sung kênh phản hồi lực (force/haptic feedback) để giải quyết giới hạn "chỉ có phản hồi thị giác" của kiến trúc gốc.</details>

## 📝 Bài tập thực hành

1. **Đọc phần "System Overview" của paper gốc** ([arXiv:2407.01512](https://arxiv.org/abs/2407.01512)), xác định chính xác Jacobian được dùng trong CLIK là Jacobian của khớp nào tới khớp nào (toàn bộ chuỗi động học tay, hay chỉ từ vai tới cổ tay) — so sánh với mô tả CLIK/Jacobian-based IK trong bài giảng "Inverse Kinematics per-frame — thuật toán Jacobian-based/differential IK" ở mảng 02, ghi lại 2 điểm giống và 2 điểm khác giữa 2 ngữ cảnh sử dụng.

2. **Tính lại biến thể ví dụ tính tay**: giả sử độ trễ end-to-end đo được của một hệ VR teleop khác là 95ms (thay vì 121.5ms baseline Open-TeleVision). Tính xem trong khoảng thời gian trễ này, nếu người vận hành đang di chuyển tay với vận tốc 0.5 m/s, tay đã di chuyển thêm bao nhiêu cm trước khi robot "bắt kịp" được vị trí đó — đây chính là độ trễ vị trí (position lag) mà người vận hành cảm nhận được trong thực tế.

3. **Xem repo GitHub chính thức** ([OpenTeleVision/TeleVision](https://github.com/OpenTeleVision/TeleVision)), tìm file cấu hình gimbal/camera cho Unitree H1 và Fourier GR-1 — so sánh 2-DOF (H1) với 3-DOF (GR-1) và thử liệt kê 1 loại chuyển động đầu mà GR-1 làm được nhưng H1 không làm được (gợi ý: liên quan tới bậc tự do roll).

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

Open-TeleVision (Cheng et al., CoRL 2024) giải quyết giới hạn của teleop kiểu leader-follower (ALOHA) bằng cách dùng kính VR tracking tư thế toàn thân người vận hành và giải bài toán IK trong thời gian thực (CLIK cho tay, SLSQP cho ngón tay) để ánh xạ sang bất kỳ robot humanoid nào, kết hợp phản hồi hình ảnh stereo 3D egocentric ở 60 Hz để người vận hành cảm nhận như đang "nhập vai" vào cơ thể robot. Kiến trúc này khác biệt cốt lõi so với retargeting offline ở chỗ phải giải IK tức thời từng khung hình, không được nhìn trước tương lai, và ánh xạ theo tỷ lệ tương đối (end-effector↔đầu) chứ không sao chép tuyệt đối. Chỉ hơn một năm sau khi công bố, nguyên lý này đã được tích hợp vào các framework công nghiệp như Isaac Lab 2.3 (hỗ trợ Meta Quest, Manus gloves, Unitree G1), trong khi các hướng nghiên cứu tiếp theo (đa camera, phản hồi lực như Prometheus) đang mở rộng thêm những khoảng trống mà bản gốc còn để lại — cho thấy đây là kiến trúc nền tảng ổn định, đang được công nghiệp hoá nhanh chóng.
