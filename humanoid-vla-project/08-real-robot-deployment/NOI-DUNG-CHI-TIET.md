# Nội dung chi tiết — Teleoperation & Triển khai robot thật

> File này là phần "sách giáo trình" cho `README.md` cùng thư mục — giải thích đầy đủ 6 khái niệm ở mục A để đọc hiểu được mà không cần tự mở paper gốc/docs gốc. Các số liệu/kiến trúc mô tả dưới đây đều lấy trực tiếp từ nguồn đã fetch (arXiv full text + docs chính thức GR00T-WholeBodyControl), có ghi rõ chỗ nào paper không công bố chi tiết.

---

## 1. Vì sao cần teleoperation — giải thích qua case study ALOHA

### Vấn đề mà retargeting từ mocap không giải quyết được

Ở mảng `02-motion-retargeting/`, bạn đã học retargeting: lấy chuyển động con người (từ mocap hoặc video) rồi ánh xạ sang khớp robot. Cách này rất tốt cho chuyển động toàn thân thô (đi, với tay, cúi người) — vì mục tiêu chỉ là "robot bắt chước hình dáng chuyển động". Nhưng nó **không nắm bắt được lực và tiếp xúc**: khi con người thao tác tay khéo léo (cầm một cái nắp mỏng, xỏ dây, lắp pin vào khe hẹp), thứ quyết định thành công không phải là quỹ đạo hình học của bàn tay, mà là **lực tiếp xúc, độ trượt, phản hồi xúc giác theo thời gian thực** — những thứ mocap hoàn toàn không ghi lại được (mocap chỉ ghi vị trí marker, không ghi lực).

Teleoperation giải quyết vấn đề này bằng cách khác hẳn: thay vì ghi chuyển động người rồi *suy diễn* sang robot (offline, gián tiếp), người vận hành **điều khiển trực tiếp robot thật trong thời gian thực**, quan sát phản hồi hình ảnh (và đôi khi lực) ngay lập tức, tự điều chỉnh tay theo phản ứng thực tế của vật thể. Dữ liệu thu được (camera + trạng thái khớp + hành động khớp) vì vậy phản ánh đúng động lực học tiếp xúc thật — đây chính là dữ liệu "vàng" cho imitation learning.

### Case study: ACT / ALOHA (Zhao et al. 2023, arXiv:2304.13705)

Đây là ví dụ kinh điển nhất chứng minh luận điểm trên bằng số liệu cụ thể.

**Phần cứng — rẻ và dễ tái tạo:**
- Hệ thống bimanual (hai tay) gồm 2 cánh tay **ViperX 6-DOF** làm "follower" (robot thực thi) — giá khoảng **$5,600**, tải trọng 750g, sải tay 1.5m, độ chính xác 5–8mm.
- 2 cánh tay **WidowX** làm "leader" (tay cầm điều khiển mà người vận hành cầm trực tiếp để dẫn chuyển động, dùng cơ chế master-slave/leader-follower teleop, không qua VR) — giá khoảng **$3,300**.
- 4 camera Logitech C922x streaming ảnh RGB 480×640: 2 gắn ở cổ tay robot follower, 2 gắn ở phía trước và phía trên.
- Tổng chi phí toàn hệ thống: **nằm trong ngân sách 20,000 USD** — rẻ hơn rất nhiều so với các hệ thống teleop công nghiệp.

**Dữ liệu thu thập — rất ít:**
- **50 demo cho mỗi task**, riêng task "Thread Velcro" khó hơn nên thu **100 demo**.
- Tổng thời gian thu dữ liệu chỉ khoảng vài chục phút mỗi task (demo ngắn, vài giây đến một phút mỗi lần).
- Đây chính là minh chứng cụ thể cho luận điểm ở mục A.1 của README: **"vài chục demo teleop có thể huấn luyện được policy tay khéo léo"**.

**6 task thao tác tay tinh xảo:**

| Task | Mô tả ngắn | Tỷ lệ thành công |
|---|---|---|
| Slide Ziploc | Trượt khoá túi zip | 86% |
| Slot Battery | Lắp pin vào khe | 93% |
| Open Cup | Mở nắp cốc (nhựa trong, mỏng) | 84% |
| Thread Velcro | Xâu dây velcro | 20% (khó nhất) |
| Prep Tape | Chuẩn bị băng dính | 64% |
| Put On Shoe | Xỏ giày | 92% |

**Thuật toán — ACT (Action Chunking with Transformers):** học một mô hình sinh (generative model) trên *chuỗi hành động* (action sequence) thay vì hành động đơn lẻ từng bước — mục đích là giảm lỗi tích luỹ (compounding error, vấn đề kinh điển của imitation learning: sai số nhỏ ở bước đầu dồn tích khiến robot lệch quỹ đạo dần) và xử lý được tính không ổn định của demo con người (con người demo không hoàn toàn nhất quán giữa các lần).

**Kết luận rút ra:** chỉ với 50–100 demo teleop, thu trên phần cứng giá rẻ (~20k USD), không cần simulation, không cần motion-capture phòng thí nghiệm đắt tiền, ALOHA đạt 80–93% thành công trên hầu hết task thao tác tay khéo léo. Đây là lý do ngành robot học hiện nay coi **teleoperation + imitation learning** là con đường thực tế nhất để dạy robot kỹ năng tay tinh xảo — thứ mà retargeting từ mocap/video người không làm được.

---

## 2. Kiến trúc teleoperation qua VR — giải thích chi tiết theo Open-TeleVision

Nguồn: Cheng, Li, Yang, Yang, Wang (2024), *"Open-TeleVision: Teleoperation with Immersive Active Visual Feedback"*, CoRL 2024, [arXiv:2407.01512](https://arxiv.org/abs/2407.01512). Chi tiết dưới đây lấy từ toàn văn paper (bản HTML ar5iv), không phải suy diễn.

### Sơ đồ luồng ASCII

```
[Robot: camera stereo trên đầu]
        │  (ZED Mini, 480×640/mắt, gắn trên gimbal 2–3 DOF)
        ▼
[Stream video stereo 3D — 60 Hz, qua web server nền Vuer]
        ▼
[VR headset người vận hành]
        │  (Apple Vision Pro — hệ thống "agnostic", tức không
        │   phụ thuộc hãng kính cụ thể, có thể thay bằng kính khác)
        ▼
[Người vận hành nhìn thấy hình ảnh 3D egocentric đúng góc nhìn robot]
        │
        ▼
[Tracking tay + đầu + cổ tay người vận hành, SE(3), 60 Hz]
        │
        ▼
[IK real-time: CLIK (Closed-loop Inverse Kinematics, nền Pinocchio)
 cho tay/cánh tay + SLSQP (dex-retargeting) cho các ngón tay]
        │
        ▼
[Robot actuator: cánh tay + bàn tay khéo léo + gimbal đầu (đồng bộ
 vị trí tương đối end-effector ↔ đầu, khớp với tương quan cổ tay ↔ đầu người)]
```

### Giải thích từng khâu

**Camera stereo trên đầu robot:** dùng camera **ZED Mini** — một camera stereo (2 ống kính, tạo chiều sâu 3D thật, không phải ước lượng độ sâu từ 1 ảnh). Độ phân giải 480×640 cho mỗi mắt. Camera được gắn trên cơ cấu xoay (gimbal) có actuation riêng để mô phỏng chuyển động cổ người:
- Trên **Unitree H1**: gimbal tuỳ chỉnh **2-DOF** (yaw + pitch), dùng động cơ Dynamixel XL330-M288-T.
- Trên **Fourier GR-1**: cổ có sẵn của nhà sản xuất, **3-DOF** (yaw + roll + pitch).

**Stream hình ảnh 3D egocentric về kính VR:** toàn bộ vòng lặp (camera robot → mạng → kính VR) chạy ở tần số **60 Hz**, dùng một web server dựng trên framework **Vuer**. "Egocentric" nghĩa là người vận hành nhìn thấy đúng góc nhìn thứ nhất của robot — như thể họ đang "đội" đầu robot — chứ không phải góc nhìn thứ ba qua camera cố định.

**Tracking tay/đầu người vận hành → ánh xạ ngược sang robot (real-time IK):** đây là điểm khác biệt cốt lõi so với retargeting offline ở mảng 02. Retargeting offline xử lý *toàn bộ file* chuyển động đã ghi sẵn, có thể tối ưu hoá qua nhiều vòng lặp, không có ràng buộc thời gian thực. Ở đây, kính VR (Apple Vision Pro) tracking tư thế tay, đầu, cổ tay người vận hành trong không gian SE(3) (vị trí + hướng 3D), và hệ thống phải giải bài toán IK **trong từng khung hình (frame), tức thời**, để robot bắt kịp chuyển động ngay lập tức:
- Với cánh tay: dùng thuật toán **CLIK (Closed-loop Inverse Kinematics)** xây trên thư viện **Pinocchio** (thư viện tính toán động lực học rigid-body phổ biến trong robot học).
- Với các ngón tay (retargeting bàn tay khéo léo): dùng thư viện **dex-retargeting**, giải bằng **SLSQP (Sequential Least-Squares Quadratic Programming)** — một phương pháp tối ưu hoá có ràng buộc.
- Cánh tay robot được điều khiển sao cho **vị trí tương đối giữa end-effector (bàn tay) và đầu robot** khớp với tương quan giữa cổ tay và đầu của người vận hành thật — nghĩa là nếu người vận hành đưa tay ra xa mặt mình một khoảng, robot cũng đưa tay ra xa "mặt" nó một khoảng tương ứng theo tỷ lệ.

**Robot đã thử nghiệm:** Unitree H1 (tay khéo léo 6-DOF) và Fourier GR-1 (kẹp song song/parallel-jaw gripper) — 2 loại humanoid khác hẳn nhau về cơ cấu tay, chứng minh kiến trúc tổng quát hoá được.

**4 task đánh giá trong paper:** Can Sorting (phân loại lon), Can Insertion (lắp lon vào khe), Folding (gấp vải), Unloading (dỡ hàng) — các task đòi hỏi thao tác chính xác, dài hơi (long-horizon).

---

## 3. XRoboToolkit — kiến trúc chi tiết

Nguồn: paper chính thức [arXiv:2508.00097](https://arxiv.org/abs/2508.00097), toàn văn.

### OpenXR là gì và vì sao dùng

**OpenXR** là một chuẩn API mở (open standard), do tổ chức Khronos Group phát triển, cho phép phần mềm VR/AR giao tiếp với phần cứng (kính, controller, tracker) **không phụ thuộc vào hãng sản xuất cụ thể**. Trước khi có các framework như XRoboToolkit, mỗi hãng kính VR (Meta Quest, PICO, Apple Vision Pro, HTC Vive...) có SDK riêng, định dạng dữ liệu tracking riêng — nghĩa là code viết cho kính này không chạy được trên kính khác, và mỗi khi tích hợp robot mới hoặc kính mới cần viết lại phần lớn.

Paper mô tả rõ vấn đề này: **"thiếu định dạng dữ liệu chuẩn hoá giữa thiết bị XR và bộ điều khiển robot, buộc phải làm lại tích hợp đáng kể cho mỗi thiết bị XR hoặc nền tảng robot mới"**. XRoboToolkit giải quyết bằng cách xây một **lớp giao diện tổng quát hoá (generalized interface layer)** trên nền OpenXR — viết code một lần, chạy được trên nhiều kính VR khác nhau miễn kính đó hỗ trợ OpenXR.

### IK tối ưu hoá dùng trong hệ thống

XRoboToolkit dùng bộ giải IK dựa trên **quy hoạch toàn phương (QP — Quadratic Programming)**, cụ thể là thư viện **PlaCo**, xây trên nền **Pinocchio** (cùng thư viện động lực học mà Open-TeleVision dùng ở CLIK). Bộ giải này tối thiểu hoá **residual có trọng số (weighted task residuals)** của các ràng buộc nhiệm vụ, tuân theo giới hạn khớp và các ràng buộc khác. Điểm đáng chú ý: hệ thống có thêm **số hạng chính quy hoá khả năng thao tác (manipulability regularization term)** — mục đích là giữ robot ổn định khi tay/khớp tiến gần điểm kỳ dị (singularity, vị trí mà robot mất bậc tự do điều khiển tức thời).

### Các loại tracking hỗ trợ

| Loại tracking | Chi tiết |
|---|---|
| **Đầu (head)** | Vị trí/hướng kính VR, kèm trạng thái độ tin cậy tracking |
| **Controller** | Vị trí/hướng tay cầm trái-phải, trục joystick, độ nhấn trigger/grip, trạng thái nút |
| **Cử chỉ tay (hand gestures)** | Mô hình bàn tay 26 khớp (4 khớp ngón cái, 5 khớp mỗi ngón còn lại, cộng khớp lòng bàn tay/cổ tay), tần số **60 Hz** |
| **Toàn thân (whole-body)** | Mô hình 24 khớp theo dõi các khớp chính của cơ thể người, gồm cả vị trí/vận tốc/gia tốc |
| **Tracker phụ (auxiliary motion trackers)** | Vị trí/vận tốc/gia tốc, dùng với PICO 4 Ultra, để ràng buộc thêm cho khuỷu tay/thân người |

### Nền tảng robot hỗ trợ

Tay máy chính xác (UR5, ARX R5), robot di động thao tác (mobile manipulator Galaxea R1-Lite), bàn tay khéo léo (Shadow Hand), và môi trường mô phỏng MuJoCo.

### Độ trễ end-to-end (số liệu paper công bố)

| Cấu hình | Độ trễ trung bình | Độ lệch chuẩn |
|---|---|---|
| ZED Mini → PICO 4 Ultra | **82.00 ms** | 6.32 ms |
| PICO 4 Ultra → PICO 4 Ultra | 100.50 ms | 3.12 ms |
| Open-TeleVision (dùng làm baseline so sánh) | 121.50 ms | 6.01 ms |

Nhận xét: XRoboToolkit (82ms với ZED Mini) có độ trễ thấp hơn rõ rệt so với baseline Open-TeleVision (121.5ms) — đây là một trong những đóng góp chính paper nhấn mạnh: giảm độ trễ vòng lặp hình ảnh, giúp người vận hành phản ứng nhanh hơn với phản hồi thị giác.

---

## 4. Quy trình teleop chính thức GR00T-WholeBodyControl

Nguồn: 2 trang docs chính thức — [VR Teleop Setup](https://nvlabs.github.io/GR00T-WholeBodyControl/getting_started/vr_teleop_setup.html) và [VR Whole-Body Teleop tutorial](https://nvlabs.github.io/GR00T-WholeBodyControl/tutorials/vr_wholebody_teleop.html).

### Phần cứng cần có

- Kính VR **PICO 4** hoặc **PICO 4 Pro**.
- 2 tay cầm điều khiển PICO (controller).
- 2 tracker chuyển động PICO, gắn ở **mắt cá chân** (ankle-mounted) — dùng để tracking chân, phục vụ điều khiển toàn thân (whole-body), không chỉ tay.
- Mạng Wi-Fi tốc độ cao, độ trễ thấp (bắt buộc, vì toàn bộ luồng video + tracking đi qua mạng này).

### Các bước cấu hình (setup)

1. **Cài XRoboToolkit PC Service** trên workstation (máy tính chạy policy), trước khi kết nối kính PICO — có gói cài riêng cho Ubuntu 22.04, Ubuntu 24.04, và cho Jetson (`roboticsservice_*.deb`) — nghĩa là quy trình này hỗ trợ cả chạy trên máy tính biên gắn trực tiếp trên robot.
2. **Cài app XRoboToolkit trên kính PICO**: bật Developer Mode trên kính, tải file APK qua trình duyệt PICO, cài từ mục "Unknown" trong thư viện app.
3. **Cấu hình tracker chuyển động**: đeo tracker vào hai mắt cá chân (đèn báo hướng lên trên), tắt "Safeguard" trong Developer settings, hủy ghép nối (unpair) tracker cũ nếu có, giữ nút tracker 6 giây để vào chế độ ghép nối mới.
4. **Hiệu chỉnh (calibration)** — 2 bước trong khi đội kính: (a) đứng thẳng, hai tay cầm controller thả dọc thân người; (b) nhìn xuống để camera kính nhận diện được tracker ở chân, sau đó chỉnh lại vị trí kính quanh trán để thao tác thoải mái.
5. **Cài môi trường teleop trên workstation**: chạy `bash install_scripts/install_pico.sh` từ gốc repo — tạo môi trường ảo Python 3.10 (`.venv_teleop`) gồm ZMQ, Pinocchio, MuJoCo, và SDK của XRoboToolkit.
6. **Cấu hình mạng**: kết nối cả workstation và kính PICO vào cùng mạng Wi-Fi, ghi lại địa chỉ IPv4 của workstation, nhập địa chỉ đó vào app XRoboToolkit trên kính, xác nhận trạng thái hiển thị "WORKING", rồi bật các mục: Head tracking, Controller tracking, và truyền dữ liệu Full body motion tracker.

### Vận hành thực tế — cách policy WBC nhận lệnh

Hệ thống hỗ trợ nhiều chế độ điều khiển, khác nhau ở cách "token" lệnh được sinh ra và đưa vào policy:

- **POSE mode**: chế độ whole-body teleop đầy đủ — stream trực tiếp **tư thế SMPL** (SMPL là một mô hình tham số hoá hình dạng/tư thế cơ thể người chuẩn trong nghiên cứu, đã gặp ở mảng 03 khi nói về format dữ liệu mocap) từ kính PICO sang phía triển khai C++ trên robot — tức toàn bộ tư thế người vận hành được ánh xạ trực tiếp, tức thời sang không gian hành động (token space) của policy WBC.
- **PLANNER mode**: điều khiển bằng joystick cho phần di chuyển (locomotion), trong khi phần thân trên (upper body) do AI tự sinh chuyển động — hỗ trợ tới **20 chế độ di chuyển** khác nhau: Idle, Walk, Run, Squat, Kneel, Crawling, các biến thể Boxing, Jump, Stealth Walk, Injured Walk.
- **VR_3PT mode**: kết hợp tracking đầu + hai tay với điều khiển di chuyển kiểu planner.

**Quy trình vận hành theo bước:**
1. Vào tư thế hiệu chỉnh: đứng thẳng, hai chân chụm, cẳng tay gập 90° hướng về phía trước.
2. Nhấn tổ hợp phím **A+B+X+Y** để kích hoạt policy và chạy full calibration.
3. Nhấn **A+X** để vào POSE mode — điều khiển toàn thân đầy đủ.
4. Di chuyển tay/chân — robot bám theo tức thời.
5. Quay lại planner mode hoặc dừng bằng tổ hợp phím tương ứng.

**Lưu ý an toàn được docs nhấn mạnh rõ:** *"Whole-body teleoperation involves fast, agile motions. Always maintain a clear safety zone and keep a safety operator at the keyboard ready to trigger an emergency stop."* — luôn có một người vận hành an toàn (không phải người đang đội kính) đứng sẵn cạnh bàn phím, sẵn sàng bấm dừng khẩn cấp; ngoài ra người đội kính "phải mặc quần bó/legging để đảm bảo tầm nhìn của tracker chân" (yêu cầu kỹ thuật cho việc camera kính nhận diện đúng tracker).

---

## 5. Từ dữ liệu teleop tới fine-tuning — "data flywheel"

"Data flywheel" (bánh đà dữ liệu) là mô hình vận hành vòng lặp khép kín, phổ biến trong ngành robot học hiện đại (đây là mô tả **chung của ngành**, không phải khái niệm riêng của GR00T):

```
 ┌─────────────────────────────────────────────────────┐
 │                                                       │
 ▼                                                       │
[1. Thu dữ liệu teleop]                                  │
   (camera + trạng thái khớp + hành động,                │
    qua VR như mục 2–4 ở trên)                            │
 │                                                       │
 ▼                                                       │
[2. Huấn luyện / fine-tune policy]                       │
   (fine-tune VLA — ví dụ GR00T N1.x — hoặc              │
    bổ sung dữ liệu/reward cho WBC)                      │
 │                                                       │
 ▼                                                       │
[3. Triển khai (deploy) lên robot thật]                  │
 │                                                       │
 ▼                                                       │
[4. Phát hiện thất bại]                                  │
   (task nào robot làm sai, làm rớt, không hoàn thành)    │
 │                                                       │
 ▼                                                       │
[5. Thu thêm dữ liệu CÓ MỤC TIÊU — targeted collection]  │
   (tập trung teleop đúng vào những tình huống robot      │
    đang thất bại, không thu dàn trải ngẫu nhiên)          │
 │                                                       │
 └──────────────── quay lại bước 2 ─────────────────────┘
```

Điểm mấu chốt của "flywheel" không phải là thu càng nhiều dữ liệu càng tốt một cách dàn trải, mà là **targeted data collection**: sau khi deploy, xác định chính xác robot đang thất bại ở đâu (loại vật thể nào, tư thế nào, điều kiện ánh sáng nào), rồi dùng teleop để thu thêm dữ liệu **đúng vào lỗ hổng đó**. Vòng lặp càng chạy nhiều lần, policy càng bao phủ được nhiều tình huống thực tế (edge case) — đây là lý do các công ty robot học thương mại vận hành đội ngũ "teleop operator" liên tục, không chỉ thu dữ liệu một lần rồi dừng.

**Ví dụ ngành (thông tin chung, không phải riêng GR00T):** các công ty như Tesla (Optimus), Figure, và 1X Technologies được biết đến là vận hành vòng lặp tương tự — dùng đội ngũ vận hành viên teleop thu dữ liệu thao tác thật trên robot thật, fine-tune model, deploy, rồi lặp lại dựa trên thất bại quan sát được. Đây là mô hình chung của ngành, không phải chi tiết kỹ thuật riêng của hệ sinh thái GR00T.

Trong bối cảnh GR00T cụ thể: dữ liệu teleop thu theo quy trình mục 4 có thể dùng để **fine-tune GR00T N1.x** (xem `06-vla-groot-sonic/`, code fine-tune thực tế nằm trong repo `NVIDIA/Isaac-GR00T`), hoặc bổ sung trực tiếp vào dữ liệu huấn luyện/reward cho policy WBC.

---

## 6. Triển khai (deployment): ONNX và an toàn

### ONNX là gì, cụ thể

**ONNX (Open Neural Network Exchange)** là một **định dạng file trung gian, mở**, dùng để biểu diễn mô hình mạng neural theo cách không phụ thuộc framework. Vấn đề nó giải quyết: một model được huấn luyện bằng PyTorch (framework phổ biến nhất cho nghiên cứu robot học/VLA hiện nay) mang theo toàn bộ dependency nặng của PyTorch (autograd, CUDA driver đầy đủ, Python interpreter...) — những thứ không cần thiết và tốn tài nguyên khi chỉ cần *chạy inference* trên phần cứng biên (edge) như **Jetson** gắn trên robot.

Quy trình: **export** model từ PyTorch (hoặc TensorFlow) sang file `.onnx` — file này mô tả kiến trúc mạng (các lớp, phép toán, trọng số đã huấn luyện) dưới dạng đồ thị tính toán (computation graph) chuẩn hoá. Sau đó, file `.onnx` được chạy bằng **ONNX Runtime** — một runtime nhẹ, được tối ưu riêng cho việc *thực thi* (không cần huấn luyện), hỗ trợ tăng tốc phần cứng (GPU, các bộ tăng tốc chuyên dụng trên Jetson như TensorRT) mà không cần cài cả framework training đầy đủ. Kết quả: chạy nhanh hơn, tốn ít bộ nhớ hơn, dễ deploy trên phần cứng giới hạn tài nguyên gắn trực tiếp trên robot.

### Các bước kiểm tra an toàn trước khi chạy trên robot thật

Đây là các nguyên tắc an toàn chuẩn trong triển khai robot học (không riêng GR00T), cần thực hiện tuần tự trước khi để policy điều khiển robot thật chạy tự do:

1. **Giới hạn tốc độ và mô-men động cơ (velocity/torque limits)** — đặt giới hạn cứng ở tầng điều khiển thấp (low-level controller), độc lập với policy, để dù policy có ra lệnh sai/bất thường, động cơ vẫn không thể vượt quá tốc độ/lực an toàn đã định trước.
2. **Watchdog timeout** — cơ chế giám sát: nếu tín hiệu điều khiển từ policy/máy tính không đến đúng chu kỳ dự kiến (ví dụ mất kết nối, policy treo, độ trễ mạng tăng đột biến), watchdog tự động ngắt và đưa robot về trạng thái an toàn (dừng hoặc giữ tư thế hiện tại) thay vì để robot đứng yên chờ vô thời hạn với lệnh cũ hoặc rơi vào trạng thái không kiểm soát.
3. **Emergency stop (E-stop) vật lý** — nút dừng khẩn cấp phần cứng, luôn có người đứng cạnh sẵn sàng bấm (đúng như lưu ý trong docs GR00T-WholeBodyControl ở mục 4: cần một "safety operator" đứng cạnh bàn phím).
4. **Emergency stop phần mềm** — lệnh dừng khẩn cấp có thể kích hoạt từ xa (qua bàn phím/giao diện điều khiển), tách biệt khỏi luồng điều khiển chính của policy, để không bị phụ thuộc vào việc policy/luồng chính vẫn đang phản hồi bình thường.
5. **Kiểm tra độ trễ thực thi (inference latency)** trước khi chạy trên phần cứng thật — đo thời gian từ lúc nhận input (ảnh/trạng thái khớp) tới lúc có output hành động, đảm bảo đủ nhanh so với tần số điều khiển yêu cầu của robot (ví dụ nếu robot cần lệnh mỗi 20ms mà inference mất 50ms, robot sẽ mất ổn định).

Về nội dung áp dụng cụ thể của ONNX vào humanoid trong hệ sinh thái GR00T-WholeBodyControl (mục "Reference: ONNX models & deployment" dẫn ở README mục B): trang docs tại thời điểm truy cập không có nội dung chi tiết riêng cho phần này ngoài mô tả chung — **cần xác minh thêm khi có phiên bản docs đầy đủ hơn**, không suy diễn thêm ở đây.

---

## Bảng thuật ngữ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| **Teleoperation** | Điều khiển robot thật từ xa, theo thời gian thực, bởi người vận hành (qua VR, tay cầm, leader-follower arm...). |
| **Imitation learning** | Học chính sách (policy) bằng cách bắt chước hành vi từ dữ liệu demo do người thực hiện, không cần reward function. |
| **ACT (Action Chunking with Transformers)** | Thuật toán imitation learning, học mô hình sinh trên *chuỗi* hành động thay vì từng hành động đơn lẻ, giảm lỗi tích luỹ. |
| **Compounding error** | Lỗi tích luỹ: sai số nhỏ ở bước điều khiển đầu dồn tích qua thời gian khiến robot lệch xa quỹ đạo mong muốn. |
| **Leader-follower teleop** | Kiểu teleop dùng 2 tay máy giống hệt nhau — người vận hành cầm tay "leader", tay "follower" bắt chước chuyển động (như ALOHA), không cần VR. |
| **Stereo camera** | Camera 2 ống kính, tạo được chiều sâu (depth) 3D thật từ chênh lệch góc nhìn hai mắt. |
| **Egocentric view** | Góc nhìn thứ nhất — người xem thấy đúng như thể họ đang ở vị trí/góc nhìn của robot. |
| **SE(3)** | Không gian biểu diễn vị trí + hướng xoay 3D đầy đủ của một vật thể cứng (rigid body) trong không gian 3 chiều. |
| **Inverse Kinematics (IK)** | Bài toán tính góc khớp cần thiết để end-effector (bàn tay/chân) đạt được vị trí/hướng mục tiêu. |
| **CLIK (Closed-loop Inverse Kinematics)** | Thuật toán IK dạng vòng lặp đóng, tính lại liên tục theo sai số hiện tại, phù hợp điều khiển thời gian thực. |
| **QP (Quadratic Programming)** | Bài toán tối ưu hoá với hàm mục tiêu toàn phương và ràng buộc tuyến tính — dùng làm nền cho nhiều bộ giải IK. |
| **SLSQP** | Sequential Least-Squares Quadratic Programming — thuật toán tối ưu hoá có ràng buộc, dùng trong retargeting bàn tay. |
| **Manipulability regularization** | Số hạng thêm vào bài toán IK để giữ robot tránh xa điểm kỳ dị (singularity), duy trì ổn định điều khiển. |
| **Singularity (điểm kỳ dị)** | Cấu hình khớp mà robot mất tạm thời một hoặc nhiều bậc tự do điều khiển. |
| **OpenXR** | Chuẩn API mở (Khronos Group) cho phần mềm VR/AR giao tiếp với phần cứng không phụ thuộc hãng sản xuất. |
| **SMPL** | Mô hình tham số hoá hình dạng/tư thế cơ thể người, dùng phổ biến làm định dạng trung gian cho dữ liệu mocap/pose. |
| **Token space** | Không gian biểu diễn lệnh điều khiển dưới dạng token — cách policy WBC/VLA nhận và xử lý input hành động. |
| **Data flywheel** | Vòng lặp: thu dữ liệu → huấn luyện → deploy → phát hiện lỗi → thu dữ liệu có mục tiêu → lặp lại — mô hình vận hành robot học thương mại hiện đại. |
| **Targeted data collection** | Thu dữ liệu tập trung đúng vào tình huống robot đang thất bại, thay vì thu ngẫu nhiên dàn trải. |
| **ONNX (Open Neural Network Exchange)** | Định dạng file trung gian, mở, biểu diễn model neural không phụ thuộc framework — dùng để export sang runtime nhẹ. |
| **ONNX Runtime** | Runtime nhẹ chuyên chạy inference cho model ONNX, không cần cài framework huấn luyện đầy đủ. |
| **Edge device / phần cứng biên** | Máy tính nhỏ, gọn, tiêu thụ ít năng lượng, gắn trực tiếp trên robot (ví dụ NVIDIA Jetson) — dùng để chạy inference tại chỗ. |
| **Watchdog timeout** | Cơ chế giám sát tự động ngắt hệ thống về trạng thái an toàn khi không nhận được tín hiệu điều khiển đúng hạn. |
| **E-stop (Emergency stop)** | Nút/lệnh dừng khẩn cấp, vật lý hoặc phần mềm, cắt ngay hoạt động của robot khi có sự cố. |
