# Bài giảng: Kiến trúc "decoupled WBC" của SONIC — ý tưởng tách 2 lớp

*(Thuộc mảng: Whole-Body Control (WBC))*

## 🎯 Mục tiêu bài học

- Mô tả chính xác 2 lớp trong kiến trúc decoupled của SONIC: low-level motion-tracking policy và high-level planner, kèm input/output cụ thể của mỗi lớp.
- Giải thích được vì sao tách 2 lớp cho phép DÙNG CHUNG một policy cấp thấp cho cả teleoperation lẫn VLA — không phải chỉ "chia code cho gọn".
- Tính tay được một ví dụ so sánh chi phí huấn luyện giữa kiến trúc end-to-end riêng biệt và kiến trúc decoupled dùng chung.
- Vẽ lại được sơ đồ luồng dữ liệu đầy đủ từ 2 nguồn cấp cao (người/VLA) tới robot thật.
- Liên hệ được kiến trúc này với chính paper SONIC thật (arXiv:2511.07820) bằng các con số cụ thể (tần số, tham số).
- Phân biệt được rõ vai trò của bài này (lớp thấp WBC) với vai trò của `06-vla-groot-sonic/` (lớp cao VLA).

## 🧭 Vì sao cần học cái này? (bối cảnh)

Ba bài trước (RL thay thế model-based, huấn luyện song song, domain randomization) đã xây xong "cách huấn luyện một policy WBC học sâu". Câu hỏi tiếp theo: nếu robot cần phục vụ **nhiều mục đích khác nhau** (điều khiển bởi người qua VR, hoặc điều khiển tự động bởi một mô hình VLA), có cần huấn luyện **một policy WBC riêng cho mỗi mục đích** không? SONIC trả lời: không cần — với điều kiện thiết kế đúng kiến trúc. Đây là bài giảng đầu tiên trong 2 bài về SONIC (bài sau: token space) — bài này tập trung vào **cấu trúc 2 lớp**, bài sau tập trung vào **cơ chế kỹ thuật cụ thể (FSQ)** cho phép 2 lớp "nói cùng ngôn ngữ".

## 🧠 Trực giác

### Góc nhìn 1: Một tài xế taxi chuyên nghiệp phục vụ được cả khách đặt qua app lẫn khách vẫy tay ngoài đường

Một tài xế taxi giỏi có đúng MỘT kỹ năng cốt lõi: **lái xe an toàn, mượt mà tới bất kỳ điểm đến nào được yêu cầu**. Kỹ năng này không cần biết yêu cầu tới từ đâu — có thể từ app đặt xe (tự động hoá, giống VLA) hoặc từ một khách vẫy tay ngoài đường rồi chỉ đường trực tiếp (giống người điều khiển trực tiếp qua VR teleop). Tài xế không cần học "lái xe theo app" và "lái xe theo khách vẫy tay" như hai kỹ năng riêng biệt — chỉ cần MỘT kỹ năng lái xe, cộng với khả năng nhận địa chỉ đích dưới bất kỳ hình thức nào. Policy cấp thấp của SONIC đóng đúng vai trò "kỹ năng lái xe" này: không cần biết lệnh chuyển động tới từ người hay từ VLA, chỉ cần biết theo dõi (track) nó.

**Giới hạn của loại suy này:** tài xế con người có thể **từ chối hoặc thương lượng** một yêu cầu không hợp lý (đường cấm, quá xa); policy cấp thấp của SONIC **không có quyền từ chối** lệnh chuyển động — nó cố gắng theo dõi bất kỳ lệnh nào được đưa vào, dù lệnh đó có hợp lý về mặt vật lý hay không (trách nhiệm đảm bảo lệnh "hợp lý" thuộc về lớp cao, không phải lớp thấp).

### Góc nhìn 2: Bộ điều khiển game console dùng chung được cho nhiều tựa game khác nhau

Một tay cầm PlayStation không được thiết kế riêng cho từng tựa game — nó có một tập tín hiệu chuẩn (nút bấm, cần analog) mà **bất kỳ game nào** cũng có thể dùng, miễn là game đó "nói đúng ngôn ngữ" của tay cầm (đọc đúng tín hiệu chuẩn). Nhà phát triển game không cần biết bên trong tay cầm hoạt động ra sao (mạch điện, cảm biến) — họ chỉ cần biết giao diện tín hiệu chuẩn. Tương tự, "generative kinematic motion planner" và "VLA" trong SONIC không cần biết chi tiết bên trong policy cấp thấp (cách nó giữ thăng bằng, cách nó tính mô-men khớp) — chúng chỉ cần tạo ra đúng **định dạng lệnh chuyển động chuẩn** mà policy cấp thấp đã học cách hiểu.

**Giới hạn của loại suy này:** tay cầm game là phần cứng cố định, không "học" gì cả; policy cấp thấp của SONIC là một hệ thống đã được **huấn luyện** để hiểu đúng định dạng lệnh đó — nếu định dạng lệnh thay đổi đáng kể, cần huấn luyện lại (khác với tay cầm game, chuẩn giao tiếp thường ổn định qua nhiều thế hệ game).

## 📐 Định nghĩa chính xác

**SONIC** (Luo et al. và 27 đồng tác giả, [arXiv:2511.07820](https://arxiv.org/abs/2511.07820), *"SONIC: Supersizing Motion Tracking for Natural Humanoid Whole-Body Control"*) tách kiến trúc điều khiển thành **hai lớp tách biệt**:

**Lớp thấp — Low-level motion-tracking policy (chạy ở 50Hz theo paper gốc):**
- **Input:** (1) tín hiệu bản thể (proprioception) — vị trí khớp, vận tốc khớp, vận tốc góc gốc thân (root angular velocity), vector trọng lực trong hệ quy chiếu gốc thân, hành động ở bước trước, biểu diễn trong hệ quy chiếu heading cục bộ của robot; (2) một **lệnh chuyển động** (motion command) — chuyển động robot tham chiếu, chuyển động người dạng SMPL, hoặc dạng lai (hybrid).
- **Output:** vị trí khớp mục tiêu (target joint positions), sau đó được các bộ điều khiển PD (proportional-derivative) ở từng khớp theo dõi để tạo mô-men thực thi.
- Huấn luyện bằng **PPO với domain randomization** (đúng 2 khái niệm đã học ở các bài trước).
- Nhiệm vụ DUY NHẤT: **biết cách di chuyển cơ thể một cách tự nhiên, cân bằng, mượt mà để "đuổi theo" (track) bất kỳ chuyển động tham chiếu nào được đưa vào** — không quan tâm chuyển động đó đến từ đâu hay "vì mục đích gì".

**Lớp cao — Planner/policy cấp cao (chạy ở 10Hz theo paper gốc):**
- Quyết định **"làm gì"**: đi đâu, cầm vật gì, tương tác thế nào — sinh ra chuỗi lệnh chuyển động để đưa xuống lớp thấp theo dõi.
- Ba nguồn khả dĩ:
  1. **Con người** — qua giao diện VR teleoperation (tư thế người thật, dạng SMPL).
  2. **Generative kinematic motion planner** — chạy autoregressive, liên tục tái sinh các đoạn chuyển động tương lai dài **0.8–2.4 giây**, dựa trên trạng thái robot hiện tại + lệnh người dùng mới nhất; theo paper, inference **<5ms trên laptop chuẩn, 12ms trên Jetson Orin GPU**.
  3. **Mô hình VLA** (GR00T N1.5/N1.x) — nhận ảnh + ngôn ngữ, xuất lệnh chuyển động dạng hybrid.

## ⚙️ Cơ chế hoạt động — từng bước

```text
┌──────────────────┐      ┌────────────────────────┐
│  Người (VR teleop) │      │  VLA (GR00T N1.5/N1.x)  │
│  → tư thế SMPL     │      │  → ảnh + ngôn ngữ       │
└─────────┬─────────┘      └───────────┬─────────────┘
          │  human motion              │  hybrid command
          │  (SMPL pose)               │  (upper-body pose + nav)
          ▼                            ▼
  ┌───────────────┐            ┌────────────────────────┐
  │ Human motion   │            │ Generative kinematic     │
  │ encoder        │            │ motion planner (10Hz,    │
  │                │            │ autoregressive, sinh      │
  │                │            │ đoạn 0.8-2.4s)             │
  └───────┬───────┘            └───────────┬─────────────┘
          │                                │  hybrid motion
          │                                ▼
          │                        ┌───────────────┐
          │                        │ Hybrid motion  │
          │                        │ encoder        │
          │                        └───────┬───────┘
          ▼                                ▼
       ┌────────────────────────────────────────┐
       │     Unified token space (FSQ quantizer)  │  ← bài giảng sau
       └───────────────────┬──────────────────────┘
                            │  universal motion token
                            ▼
             ┌────────────────────────────────┐
             │  Low-level motion-tracking       │
             │  policy (50Hz, PPO+domain rand.) │
             │  input thêm: proprioception       │
             │  output: target joint positions   │
             └───────────────┬──────────────────┘
                              ▼
                    PD controllers từng khớp
                              ▼
                       Robot thật / mô phỏng
```

**Chuỗi tần số triển khai thực tế** (theo paper gốc, phần "Deployment Stack"): policy 50Hz đưa target khớp tới bộ streamer lệnh cấp thấp chạy **500Hz**, kinematic planner ở **10Hz**, input người dùng thu ở **100Hz** — toàn bộ inference chạy **onboard trên Jetson Orin dùng TensorRT**. Đây là bằng chứng cụ thể rằng kiến trúc decoupled không chỉ là ý tưởng lý thuyết mà đã được triển khai thành một **hệ thống real-time nhiều tần số** hoạt động đồng thời trên phần cứng thật.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ chi phí huấn luyện, số tự chọn — để định lượng hoá lợi ích "dùng chung một policy cấp thấp" — không phải số liệu thật công bố trong paper SONIC)*

Giả sử huấn luyện một policy motion-tracking chất lượng cao (bền vững, domain-randomized, quy mô dữ liệu lớn) tốn trung bình **9.000 giờ GPU** (đúng con số thật của SONIC đã học ở bài "Huấn luyện song song quy mô lớn").

**Kịch bản A — kiến trúc end-to-end RIÊNG BIỆT cho từng use case (không decoupled):**

```text
Use case 1: policy cho teleoperation (huấn luyện riêng)   : 9.000 giờ GPU
Use case 2: policy cho VLA autonomous (huấn luyện riêng)  : 9.000 giờ GPU
Tổng: 18.000 giờ GPU — và HAI policy này KHÔNG chia sẻ được
  kinh nghiệm vận động cơ bản cho nhau
```

**Kịch bản B — kiến trúc decoupled (SONIC):**

```text
Lớp thấp (motion-tracking policy) — huấn luyện MỘT LẦN DUY NHẤT: 9.000 giờ GPU
Lớp cao cho teleoperation: chỉ cần encoder/interface nhỏ, KHÔNG cần huấn luyện
  lại kỹ năng vận động cơ bản — chi phí nhỏ hơn nhiều (huấn luyện encoder/
  token quantizer, quy mô nhỏ hơn nhiều bậc so với policy cấp thấp)
Lớp cao cho VLA: tương tự — VLA (GR00T) được huấn luyện SẴN cho mục đích khác
  (hiểu ảnh+ngôn ngữ), chỉ cần học SINH ĐÚNG ĐỊNH DẠNG lệnh chuyển động,
  không cần học lại "cách giữ thăng bằng khi đi"
Tổng ≈ 9.000 giờ GPU + chi phí nhỏ cho mỗi lớp cao (giả sử ~500 giờ GPU/lớp cao)
     ≈ 9.000 + 500 + 500 = 10.000 giờ GPU
```

**So sánh:**

```text
Tiết kiệm ≈ 18.000 − 10.000 = 8.000 giờ GPU (≈44% tổng chi phí trong ví dụ này)
```

**Ý nghĩa:** con số cụ thể ở trên là minh hoạ (chi phí lớp cao 500 giờ/lớp là số tự chọn), nhưng logic là thật và trực tiếp từ paper: "nếu huấn luyện MỘT policy end-to-end riêng cho teleoperation và MỘT policy khác riêng cho VLA, ta phải lặp lại toàn bộ công đoạn huấn luyện kỹ năng vận động cơ bản hai lần, và hai policy đó không chia sẻ được kinh nghiệm cho nhau" — kiến trúc decoupled biến phần **đắt nhất** (huấn luyện kỹ năng vận động cơ bản, cần domain randomization + dữ liệu lớn + nhiều giờ GPU) thành chi phí **một lần duy nhất, tái sử dụng vô hạn lần** cho bất kỳ lớp cao mới nào thêm vào sau này.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Kiến trúc end-to-end riêng biệt (mỗi use case 1 policy) | Kiến trúc decoupled (SONIC) |
|---|---|---|
| Số lần huấn luyện kỹ năng vận động cơ bản | N lần (N = số use case) | 1 lần duy nhất |
| Chia sẻ kinh nghiệm giữa các use case | Không | Có (qua token space chung, bài giảng sau) |
| Thêm use case mới (ví dụ nguồn điều khiển thứ 3) | Cần huấn luyện lại toàn bộ từ đầu | Chỉ cần thêm 1 encoder mới ánh xạ vào token space có sẵn |
| Độ phức tạp lớp cao | Gộp chung với lớp thấp, khó thay đổi độc lập | Tách biệt, có thể thay đổi/nâng cấp độc lập (đổi VLA mới không cần đổi policy thấp) |
| Rủi ro khi 1 use case cần thay đổi | Có thể ảnh hưởng policy chung nếu không tách biệt tốt | Thấp — thay đổi lớp cao không ảnh hưởng lớp thấp đã huấn luyện xong |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "decoupled WBC chỉ là chia code thành 2 module cho dễ quản lý, không có lợi ích huấn luyện gì đặc biệt".** Vì sao sai: lợi ích cốt lõi không nằm ở tổ chức code, mà ở việc **kỹ năng vận động cơ bản (khó huấn luyện nhất, cần domain randomization kỹ, dữ liệu lớn) chỉ cần học MỘT LẦN** và tái sử dụng cho mọi nguồn điều khiển cấp cao — như ví dụ tính tay cho thấy, đây là tiết kiệm chi phí GPU thực sự đáng kể, không chỉ là sự gọn gàng về kiến trúc phần mềm. **Hiểu đúng:** decoupling ở đây là một quyết định thiết kế có động lực kinh tế/kỹ thuật rõ ràng (tái sử dụng phần đắt nhất), không chỉ là "best practice" kỹ thuật phần mềm chung chung.
2. **Hiểu nhầm: "vì lớp thấp không quan tâm lệnh tới từ đâu, nó sẽ theo dõi bất kỳ lệnh nào một cách hoàn hảo, kể cả lệnh phi vật lý".** Vì sao sai: policy cấp thấp được huấn luyện để theo dõi chuyển động **trong phạm vi dữ liệu huấn luyện** (700 giờ mocap, đã retarget) và ràng buộc vật lý của chính robot — nếu lớp cao đưa ra một lệnh hoàn toàn phi vật lý (ví dụ yêu cầu robot dịch chuyển tức thời), policy cấp thấp sẽ cố gắng theo dõi tốt nhất có thể trong giới hạn vật lý, không phải "thực hiện hoàn hảo" lệnh phi lý đó. **Hiểu đúng:** trách nhiệm đảm bảo lệnh chuyển động "hợp lý" (khả thi về mặt vật lý, đúng ngữ cảnh) thuộc về lớp cao (kinematic planner hoặc VLA), không phải lớp thấp — đây là lý do kinematic planner của SONIC được thiết kế để sinh chuyển động khả thi (dựa trên trạng thái robot hiện tại), không sinh lệnh tuỳ tiện.

## 🏗️ Ví dụ minh hoạ trong dự án này

```text
Toàn bộ pipeline của dự án hội tụ về đúng điểm này:

02-motion-retargeting/ (GMR, SOMA-retargeter)
        │  retarget dữ liệu mocap → chuyển động robot tham chiếu
        ▼
03-human-motion-datasets/ (AMASS, LAFAN1, BONES-SEED — 700 giờ)
        │
        ▼
04-imitation-learning-rl/ + 05-simulation-mujoco-isaaclab/
        │  huấn luyện LỚP THẤP (motion-tracking policy, 9.000+ giờ GPU,
        │  domain randomization, PPO — đúng 3 bài trước đã học)
        ▼
      SONIC lớp thấp (50Hz) — ĐÃ HUẤN LUYỆN XONG, TÁI SỬ DỤNG
        │
        ├──▶ 08-real-robot-deployment/ (VR teleoperation → lớp cao = người)
        │
        └──▶ 06-vla-groot-sonic/ (GR00T VLA → lớp cao = mô hình VLA)
```

Bài giảng này chính là "cầu nối" — nội dung tập trung vào **lớp thấp (WBC)** và cơ chế cho phép hai lớp cao khác nhau (người/VLA) đều "cắm" được vào cùng một lớp thấp; phần "lớp cao = VLA" là chủ đề của `06-vla-groot-sonic/`.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **SONIC là chính công trình 2025 định nghĩa kiến trúc này — bản thân nó đã là "cập nhật hiện đại nhất" tại thời điểm viết.** *"SONIC: Supersizing Motion Tracking for Natural Humanoid Whole-Body Control"* (Luo et al. và 27 đồng tác giả, [arXiv:2511.07820](https://arxiv.org/abs/2511.07820)) giải quyết đúng vấn đề mà chính paper nêu: "mặc dù các mô hình nền tảng hàng tỷ tham số được huấn luyện trên hàng nghìn GPU đã xuất hiện, các thành tựu scale tương tự chưa được chứng minh cho điều khiển humanoid — các bộ điều khiển nơ-ron hiện tại cho humanoid vẫn khiêm tốn về kích thước, nhắm tới một tập hành vi hạn chế, và huấn luyện trên một số ít GPU". SONIC scale theo 3 trục: model size (1.2M→42M tham số), dataset (100M+ khung hình, 700 giờ mocap từ 170 người, retarget bằng **chính GMR** đã học ở `02-motion-retargeting/`), và compute (9.000+ giờ GPU, huấn luyện trên 128 GPU trong 3 ngày).
2. **Các công trình liên quan cùng thời điểm mở rộng hướng "cross-embodiment" của kiến trúc decoupled.** *"Scalable and General Whole-Body Control for Cross-Humanoid Locomotion"* ([arXiv:2602.05791](https://arxiv.org/pdf/2602.05791)) và *"Any2Any: Efficient Cross-Embodiment Transfer for Humanoid Whole-Body Tracking"* ([arXiv:2605.23733](https://arxiv.org/pdf/2605.23733)) mở rộng ý tưởng "một policy dùng chung" không chỉ qua nhiều nguồn lệnh điều khiển (như SONIC) mà còn qua **nhiều loại robot humanoid khác nhau** — một trục mở rộng tự nhiên tiếp theo của tư tưởng decoupling.
3. **Xu hướng chung:** kiến trúc "tách kỹ năng cơ bản khỏi ý định cấp cao" đang trở thành mẫu hình thiết kế chuẩn (design pattern) cho robot học hiện đại — tương tự cách các mô hình ngôn ngữ lớn (LLM) tách "khả năng ngôn ngữ tổng quát" (huấn luyện một lần, tốn kém) khỏi "fine-tuning cho tác vụ cụ thể" (rẻ hơn nhiều, tái sử dụng backbone chung) — một sự tương đồng kiến trúc đáng chú ý giữa hai lĩnh vực tưởng chừng khác biệt.

## ❓ Câu hỏi tự kiểm tra

1. Input và output cụ thể của lớp thấp (low-level motion-tracking policy) là gì, và nó chạy ở tần số nào theo paper gốc?
   <details><summary>Gợi ý đáp án</summary>Input: proprioception (vị trí/vận tốc khớp, vận tốc góc gốc thân, vector trọng lực, action bước trước) + motion command (từ lớp cao). Output: target joint positions, theo dõi bởi PD controllers. Chạy ở 50Hz.</details>
2. Vì sao "một policy cấp thấp duy nhất phục vụ được cả teleoperation lẫn VLA" là hệ quả trực tiếp của việc tách 2 lớp, không phải một tính năng thêm vào sau?
   <details><summary>Gợi ý đáp án</summary>Vì cả hai nguồn lệnh (người qua teleop, VLA) đều được quy về CÙNG một định dạng lệnh chuyển động (qua token space thống nhất, bài giảng sau) trước khi vào lớp thấp — bản thân lớp thấp không phân biệt được nguồn gốc lệnh, nên tự động phục vụ được mọi nguồn đã tuân theo đúng giao diện chung, không cần thiết kế thêm gì riêng cho từng nguồn.</details>
3. Trong ví dụ tính tay, khoản tiết kiệm chi phí GPU chủ yếu đến từ đâu?
   <details><summary>Gợi ý đáp án</summary>Từ việc phần đắt nhất (huấn luyện kỹ năng vận động cơ bản của lớp thấp, 9.000 giờ GPU) chỉ cần thực hiện MỘT LẦN duy nhất và tái sử dụng cho mọi lớp cao, thay vì lặp lại toàn bộ 9.000 giờ đó cho mỗi use case riêng biệt.</details>
4. Kinematic motion planner của SONIC hoạt động ở tần số nào, sinh ra đoạn chuyển động dài bao lâu, và tốc độ inference trên Jetson Orin là bao nhiêu?
   <details><summary>Gợi ý đáp án</summary>10Hz, sinh đoạn chuyển động dài 0.8-2.4 giây (autoregressive), inference 12ms trên Jetson Orin GPU (dưới 5ms trên laptop chuẩn).</details>
5. Vì sao trách nhiệm đảm bảo lệnh chuyển động "khả thi về vật lý" thuộc về lớp cao chứ không phải lớp thấp?
   <details><summary>Gợi ý đáp án</summary>Vì lớp thấp chỉ có nhiệm vụ theo dõi (track) tốt nhất có thể bất kỳ lệnh nào được đưa vào trong giới hạn vật lý của robot — nó không có cơ chế "từ chối" hay "kiểm tra tính hợp lý" của lệnh; kinematic planner/VLA (lớp cao) phải tự đảm bảo sinh ra lệnh khả thi dựa trên trạng thái robot hiện tại.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** với chi phí huấn luyện lớp thấp là 12.000 giờ GPU, chi phí mỗi lớp cao là 300 giờ GPU, và có 4 nguồn điều khiển cấp cao khác nhau cần hỗ trợ — tính tổng chi phí theo kiến trúc end-to-end riêng biệt (4× lớp thấp) so với kiến trúc decoupled (1× lớp thấp + 4× lớp cao), và tính % tiết kiệm.
2. **Đọc paper thật:** đọc phần "Deployment" hoặc "System" trong [arXiv:2511.07820](https://arxiv.org/html/2511.07820v1) — vẽ lại (trên giấy hoặc mô tả bằng lời) toàn bộ chuỗi tần số triển khai thực tế (50Hz policy → 500Hz command streamer → 10Hz planner → 100Hz input người dùng) và giải thích tại sao mỗi thành phần cần chạy ở tần số khác nhau thay vì tất cả cùng một tần số.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

SONIC tách kiến trúc điều khiển humanoid thành lớp thấp (motion-tracking policy, 50Hz, huấn luyện bằng PPO+domain randomization trên 700 giờ mocap đã retarget qua GMR) chỉ có nhiệm vụ theo dõi bất kỳ lệnh chuyển động nào, và lớp cao (người qua VR teleop, kinematic planner autoregressive 10Hz, hoặc VLA GR00T) quyết định "làm gì" — như ví dụ tính tay minh hoạ, cách tách này biến phần huấn luyện đắt nhất (kỹ năng vận động cơ bản, 9.000+ giờ GPU trong paper thật) thành chi phí một lần duy nhất, tái sử dụng cho mọi nguồn điều khiển cấp cao thay vì lặp lại cho từng use case. Toàn bộ hệ thống đã được triển khai thực tế với chuỗi tần số cụ thể (50Hz→500Hz cho lớp thấp, 10Hz cho planner, 100Hz cho input) chạy onboard trên Jetson Orin — một mẫu hình kiến trúc "tách kỹ năng cơ bản khỏi ý định cấp cao" đang trở thành chuẩn mực cho robot học hiện đại, tương đồng với cách các LLM tách backbone tổng quát khỏi fine-tuning tác vụ cụ thể.
