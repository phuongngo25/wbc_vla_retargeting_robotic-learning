# Policy Evaluation cho Humanoid Robot — Nội dung chi tiết

*File này giải thích đầy đủ 5 khái niệm ở mục A của `README.md`, đủ chi tiết để tự thiết kế một bộ đánh giá (evaluation suite) cho policy whole-body control / VLA của bạn, không chỉ đọc link rồi đoán.*

---

## 1. Metric cho motion tracking — công thức cụ thể

Motion tracking là bài toán: robot phải bắt chước một chuyển động tham chiếu (reference motion — lấy từ mocap, hoặc từ retargeting, xem `02-motion-retargeting/`). Đánh giá "bắt chước tốt tới đâu" cần ít nhất 4 con số sau.

### 1.1. Tracking error (MPJPE — Mean Per Joint Position Error)

Đây là metric chuẩn mực nhất, mượn từ lĩnh vực 3D human pose estimation, được cộng đồng motion-tracking robot học (DeepMimic, AMP, SONIC...) dùng lại gần như nguyên vẹn.

**Công thức** (đã xác minh qua tài liệu pose estimation): tại mỗi frame, với N khớp (joint) đang theo dõi,

```
MPJPE = (1/N) × Σ_i || P_i − G_i ||₂
```

trong đó:
- `P_i` = vị trí 3D của khớp thứ i trên **robot** (predicted / policy tạo ra) tại frame đó, thường tính trong hệ toạ độ gắn với gốc thân (root-relative) để không bị nhiễu bởi lỗi định vị toàn cục.
- `G_i` = vị trí 3D của khớp thứ i trên **chuyển động tham chiếu** (ground truth) tại cùng frame.
- `||·||₂` = khoảng cách Euclid (căn bậc hai tổng bình phương hiệu 3 trục x,y,z).
- Kết quả trung bình theo N khớp, rồi thường trung bình tiếp theo toàn bộ các frame của episode → ra một con số duy nhất, đơn vị **mm** (đôi khi cm).

Biến thể hay gặp:
- **MPJPE-L** (local): tính trên hệ toạ độ cục bộ, loại bỏ sai số của root — đo "hình dạng tư thế" thuần tuý. SONIC (arXiv:2511.07820) dùng chính biến thể này khi báo cáo sim-to-real gap (xem mục 3).
- **PA-MPJPE** (Procrustes-Aligned): căn chỉnh (xoay/dịch/co giãn) trước khi so sánh, loại bỏ cả sai số vị trí lẫn hướng toàn cục — hữu ích khi bạn chỉ quan tâm "dáng đi có đúng không" chứ không quan tâm robot có đi lệch hướng vài độ.
- Với khớp xoay (không chỉ vị trí), một số paper còn báo thêm **sai số góc khớp** (joint angle error, đơn vị độ) song song với MPJPE vị trí.

> Lưu ý thực hành: luôn nói rõ bạn tính MPJPE **toàn thân** hay **theo từng bộ phận** (tay, chân, thân trên) — SONIC báo cáo tách riêng để lộ ra chỗ gap lớn nhất (xem mục 3), đây là cách làm nên học theo thay vì chỉ báo 1 con số trung bình toàn thân.

### 1.2. Success rate

Tỷ lệ phần trăm episode **hoàn thành thành công**, trên tổng số episode thử:

```
Success rate = (số episode thành công / tổng số episode thử) × 100%
```

Cái khó không phải công thức mà là **định nghĩa "thành công"** — phải khai báo rõ trước khi chạy thí nghiệm (không phải sau khi thấy kết quả đẹp mới chọn tiêu chí dễ đạt). Vài định nghĩa phổ biến cho humanoid:

- **Không ngã** trong suốt episode (thân trên không chạm sàn / không dưới một ngưỡng độ cao root).
- **Hoàn thành quãng đường/khoảng cách mục tiêu** (ví dụ đi hết 5 m mà không ngã).
- **Tracking error dưới ngưỡng** trong suốt episode (ví dụ MPJPE trung bình < X mm) — cách này khắt khe hơn "không ngã".
- Với task thao tác (manipulation): **vật thể được đặt đúng vị trí đích** trong dung sai cho phép.

### 1.3. Độ mượt chuyển động (jerk)

Chuyển động "giật cục" thường là dấu hiệu policy overfit vào reward hoặc không khả thi trên phần cứng thật (motor không theo kịp). Đo bằng **jerk** — đạo hàm bậc 3 của vị trí theo thời gian:

```
vị trí  p(t)
vận tốc      v(t) = dp/dt
gia tốc      a(t) = dv/dt
jerk         j(t) = da/dt      (đạo hàm bậc 3 của vị trí)
```

Trong thực hành (dữ liệu rời rạc theo từng timestep), jerk được tính bằng sai phân hữu hạn bậc 3, rồi báo cáo:
- **Mean squared jerk** hoặc **RMS jerk** trên toàn trajectory (đơn vị: đơn vị-vị-trí/s³) — càng thấp càng mượt.
- Có thể tính riêng cho vị trí khớp (joint jerk) hoặc cho root (root jerk).

Jerk thấp không tự động nghĩa là tốt tuyệt đối — một policy đứng yên có jerk = 0 nhưng vô dụng. Vì vậy jerk luôn phải đọc **cùng** với tracking error/success rate, không đọc riêng lẻ.

### 1.4. Root trajectory error

Đo riêng sai lệch của **gốc thân robot** (root — thường là pelvis hoặc torso) so với tham chiếu, tách biệt khỏi lỗi của các khớp còn lại vì root là "nền" mà mọi khớp khác tính tương đối theo:

- **Root position error**: khoảng cách Euclid giữa vị trí root robot và root tham chiếu (mm), tương tự công thức MPJPE nhưng chỉ cho 1 điểm.
- **Root orientation error**: sai lệch hướng (thường đo bằng góc geodesic giữa 2 quaternion, đơn vị độ) — quan trọng vì root nghiêng/lệch hướng dù vị trí đúng vẫn khiến toàn bộ chuyển động "trông sai".

Root error lớn trong khi joint-local MPJPE nhỏ thường là dấu hiệu robot đang "trôi" toàn cục (drift) dù dáng đi từng khớp đúng — một lỗi rất đặc trưng của các policy chỉ được huấn luyện bằng imitation reward cục bộ.

---

## 2. Metric cho VLA/loco-manipulation

Khi policy nhận lệnh ngôn ngữ (language-conditioned, như GR00T/SONIC-VLA — xem `06-vla-groot-sonic/`), success rate một con số duy nhất không đủ thông tin. Cần **phân nhóm**.

### 2.1. Success rate theo nhóm task ngôn ngữ

Cách phân nhóm chuẩn trong literature (theo mẫu GR00T N1, arXiv:2503.14734):

| Nhóm | Định nghĩa | Đo được gì |
|---|---|---|
| **Seen task** | Task (đối tượng + hành động + vị trí) xuất hiện trong tập huấn luyện | Khả năng học thuộc/khớp phân phối huấn luyện |
| **Unseen task** | Kết hợp đối tượng/hành động/ngôn ngữ **chưa từng xuất hiện** khi huấn luyện | **Generalization** thực sự — đây mới là con số quan trọng để tuyên bố "policy hiểu ngôn ngữ" chứ không phải học vẹt |

Trong "unseen", nên tách tiếp theo mức độ khác biệt:
- **Novel instruction, same object/scene**: đổi cách diễn đạt câu lệnh, giữ nguyên vật thể/bối cảnh.
- **Novel object**: vật thể mới nhưng cùng loại hành động đã học.
- **Novel object + novel instruction**: khó nhất, đo generalization compound.

Báo cáo success rate **riêng cho từng nhóm** (không gộp trung bình) — một policy có success rate cao ở "seen" nhưng thấp ở "unseen" là dấu hiệu overfit vào phân phối huấn luyện, điều mà một con số trung bình duy nhất sẽ che giấu.

### 2.2. Robustness testing (out-of-distribution evaluation)

Đây là một dạng đánh giá **out-of-distribution (OOD)**: cố tình đưa policy vào điều kiện lệch khỏi phân phối huấn luyện để đo độ bền, không phải đo khả năng cao nhất. Các trục nhiễu loạn thường dùng:

- **Ánh sáng**: đổi cường độ/màu/hướng sáng so với lúc thu dữ liệu huấn luyện.
- **Vị trí/hướng vật thể**: đặt vật thể ở vị trí/góc xoay không có trong tập huấn luyện, hoặc thêm vật cản.
- **Nhiễu môi trường**: thêm vật thể gây rối (distractor), đổi nền/texture, đổi camera viewpoint.
- **Nhiễu vật lý** (với sim hoặc robot thật): thay đổi ma sát, tải trọng, hoặc nhiễu cảm biến.

Cách báo cáo chuẩn: success rate trong điều kiện chuẩn (in-distribution) **so sánh cạnh nhau** với success rate dưới từng loại nhiễu loạn — độ sụt giảm (delta) chính là con số đo robustness, không phải con số tuyệt đối một mình.

---

## 3. Sim-to-real gap — cách đo và ý nghĩa

### 3.1. Công thức khái niệm

```
sim-to-real gap = performance(sim) − performance(real)
```

tính trên **cùng một tập task**, cùng metric (success rate, hoặc MPJPE, v.v.), cùng điều kiện thử càng giống nhau càng tốt. Gap dương lớn nghĩa là policy "ảo tưởng" trong sim nhưng không chuyển giao được ra thật — dấu hiệu domain randomization/reward shaping chưa đủ.

Theo tổng hợp của Zhao, Queralta & Westerlund (2020, arXiv:2009.13303) — survey về sim-to-real transfer trong deep RL cho robotics — cộng đồng thường báo cáo gap theo 3 dạng chính:

1. **Zero-shot transfer**: policy huấn luyện hoàn toàn trong sim, deploy thẳng lên robot thật **không hề có bước fine-tune nào**. Đây là bằng chứng mạnh nhất cho domain randomization/system identification tốt — vì không có cơ hội "vá lỗi" bằng dữ liệu thật.
2. **Fine-tuned transfer**: sau khi deploy, cho phép một lượng nhỏ dữ liệu thật để fine-tune policy trước khi đo performance cuối — gap đo được ở đây thường nhỏ hơn nhưng **không phản ánh** khả năng của riêng simulation training.
3. **Domain randomization coverage**: thay vì đo gap trực tiếp, một số nghiên cứu báo cáo *độ rộng* của phân phối tham số được randomize khi huấn luyện (khối lượng, ma sát, độ trễ actuator, nhiễu cảm biến...) như một proxy cho "khả năng chịu được sim-to-real gap trong tương lai" — cách này gián tiếp hơn nhưng hữu ích khi chưa có robot thật để đo trực tiếp.

Khi đọc một paper báo cáo sim-to-real gap, câu hỏi bắt buộc phải trả lời được là: **gap được đo theo dạng nào trong 3 dạng trên?** — vì so sánh gap "zero-shot" của paper A với gap "fine-tuned" của paper B là so sánh không công bằng.

### 3.2. Trường hợp cụ thể: SONIC (arXiv:2511.07820)

Đã xác minh trực tiếp qua nội dung phần Real-World Evaluation của paper:

> *"We assess the real-world performance of SONIC by deploying it on 50 diverse motion trajectories, including dance, jumps, and loco-manipulation tasks... it succeeds on all sequences without a single failure (100% success rate)."*

Chi tiết điều kiện thí nghiệm (đã xác minh):
- **Robot thật**: Unitree G1.
- **Số trajectory**: 50, đa dạng thể loại (dance, jump, loco-manipulation).
- **Chế độ**: **zero-shot** — không fine-tune trên dữ liệu thật, đúng dạng (1) ở mục 3.1, nên "100% success" ở đây là một tuyên bố mạnh về domain randomization/hệ thống transfer của họ, không phải một con số "dễ đạt".
- **Số lần thử mỗi trajectory**: theo cách phần này được trình bày, mỗi trajectory được thử — **cần xác minh thêm** con số chính xác nếu bạn cần trích dẫn học thuật (ví dụ 1 lần/trajectory hay nhiều lần rồi lấy tốt nhất); bản thân "không một lần thất bại nào" gợi ý mỗi trajectory chỉ được tính 1 lần thử (single trial), nhưng nên đối chiếu lại bản PDF gốc trước khi trích dẫn số liệu này trong báo cáo chính thức.
- **Baseline so sánh trong điều kiện thật**: theo nội dung đã xác minh, phần so sánh với baseline (Any2Track, BeyondMimic, GMT) diễn ra **trong simulation**, không phải trực tiếp trên robot thật cùng 50 trajectory này — **cần xác minh thêm** nếu bạn cần biết liệu có baseline nào khác cũng được thử zero-shot thật để so sánh trực tiếp con số 100%.
- SONIC còn báo cáo một phép đo gap **định lượng** (không chỉ nhị phân thành/bại) bằng MPJPE-L trên một tập broader các motion: sim ≈ 22.3 mm vs real ≈ 25.7 mm; gap nhỏ nhất ở phần thân trên (upper body: 21.8 mm sim vs 22.2 mm real), gap lớn nhất ở bàn chân (feet: 29.0 mm sim vs 53.7 mm real) — **cần xác minh thêm** để đối chiếu chính xác con số này (124 motion sequences, 99.2% = 123/124) có phải cùng một thí nghiệm với tập "50 trajectory, 100%" hay là một tập đánh giá riêng biệt trong cùng paper — hai bộ số liệu này xuất hiện ở các phần khác nhau của paper và có thể đo hai thứ khác nhau (nhị phân thành/bại theo trajectory vs. sai số liên tục theo từng phần cơ thể).

**Vì sao "100% trên 50 trajectory, zero-shot" là một tuyên bố mạnh**: vì (a) không có bước sửa lỗi bằng dữ liệu thật (zero-shot), (b) 50 trajectory đủ đa dạng thể loại (không chỉ đi bộ thẳng — có dance, jump là các chuyển động động lực học cao, dễ lộ sim-to-real gap), và (c) tiêu chí "0 thất bại" nghiêm ngặt hơn nhiều so với báo cáo "success rate trung bình X%" vì chỉ cần 1 lần ngã là phá vỡ tuyên bố. Người đọc nên tự hỏi thêm: môi trường thử (sàn cứng/mềm, không gian mở) và định nghĩa "thất bại" (ngã hoàn toàn hay chỉ lệch quỹ đạo) được kiểm soát ra sao — đây là các chi tiết cần đọc bản đầy đủ PDF/HTML của paper để nắm chính xác trước khi trích dẫn.

---

## 4. So sánh công bằng (fair comparison)

Áp dụng trực tiếp các nguyên tắc chung ở `resources/07-experiments-and-rigor.md` (mục B, C) vào bối cảnh robot policy — nơi các lỗi này cực kỳ phổ biến vì mỗi lần chạy robot thật tốn thời gian/rủi ro nên người ta dễ "tiết kiệm" bằng cách cắt góc:

| Lỗi thường gặp | Biểu hiện cụ thể ở robot learning | Cách tránh |
|---|---|---|
| **Cherry-picking** | Quay video/báo cáo con số từ lần chạy đẹp nhất trong nhiều lần thử, không nói có bao nhiêu lần thử thất bại bị bỏ qua. | Báo cáo **tất cả** các lần thử trong một khoảng thời gian/số lượng đã định trước (pre-registered), không chọn lọc sau khi thấy kết quả. |
| **Thiếu seed variance** | Chỉ train 1 seed, báo cáo như thể đó là hiệu năng "của phương pháp" — trong khi RL/imitation learning dao động rất lớn giữa các seed. | Chạy ≥3 seed (lý tưởng 5), báo cáo mean ± std, giống nguyên tắc ở mục C của `07-experiments-and-rigor.md`. |
| **Test set rò rỉ** | Dùng chính các trajectory/motion đã xuất hiện trong tập huấn luyện để làm "test" đo tracking error — con số sẽ đẹp giả tạo vì đó gần như là đánh giá trên train set. | Tách rõ **tập motion huấn luyện** và **tập motion test/held-out** ngay từ khâu chuẩn bị dữ liệu (xem `03-human-motion-datasets/`); với unseen-task evaluation ở mục 2.1, đảm bảo tổ hợp (vật thể, hành động, ngôn ngữ) thực sự chưa xuất hiện, không chỉ đổi câu chữ bề mặt. |
| **So sánh baseline không cùng compute budget** | Method của bạn được tune hyperparameter kỹ, chạy nhiều bước huấn luyện hơn, trong khi baseline dùng cấu hình mặc định từ paper gốc. | Cùng ngân sách tuning, cùng số bước huấn luyện/số lượng dữ liệu — hoặc nếu không thể, nói rõ sự khác biệt về ngân sách trong phần Limitations thay vì im lặng. |

---

## 5. Benchmark chuẩn cộng đồng: HumanoidBench

Đã xác minh qua trang chính thức (humanoid-bench.github.io) và tóm tắt paper (Sferrazza, Huang et al., RSS 2024):

### 5.1. Tổng quan

- **27 task whole-body** được chia thành 3 nhóm:
  - **Locomotion tasks**: các task di chuyển (ví dụ đi bộ — walk, chạy — run, leo cầu thang — stair, bò — crawl, và các biến thể địa hình/tốc độ khác).
  - **"Static" manipulation tasks**: thao tác vật thể trong khi phần chân giữ tương đối ổn định.
  - **"Dynamic" manipulation tasks**: thao tác vật thể đòi hỏi phối hợp toàn thân trong lúc di chuyển (ví dụ mang vác vật khi đi — package carry, đẩy vật — push) — đây là nhóm khó nhất vì đòi hỏi cả whole-body control lẫn manipulation cùng lúc.

  > Trang chủ chính thức liệt kê tổng 27 task theo 3 nhóm trên nhưng không nêu đầy đủ tên từng task riêng lẻ trong phần tóm tắt công khai — nếu bạn cần danh sách đầy đủ tên 27 task để trích dẫn chính xác trong báo cáo, nên đối chiếu trực tiếp bảng phụ lục (appendix) của bản PDF/RSS proceedings.

- **Robot & end-effector hỗ trợ**:
  - Thân robot: **Unitree H1**, **Agility Robotics Digit**.
  - Tay/gripper: **Shadow Hand** (bàn tay khéo léo nhiều bậc tự do — dexterous hand) và **Robotiq 2F-85** (gripper 2 ngón đơn giản hơn).
  - Cấu hình thực nghiệm chính trong paper: Unitree H1 gắn 2 Shadow Hand.
- **Engine mô phỏng**: **MuJoCo**.

### 5.2. Phát hiện chính của paper

- **Các thuật toán RL SOTA (state-of-the-art) gặp khó khăn với phần lớn task** — đặc biệt các phương pháp end-to-end (huấn luyện một policy duy nhất từ input tới action) chật vật với động lực học phức tạp và horizon dài của humanoid.
- **Baseline phân cấp (hierarchical) vượt trội** khi có low-level policy đủ mạnh: kiến trúc gồm một **high-level planning policy** phát ra setpoint, truyền xuống cho các **low-level policy** chuyên biệt (ví dụ policy đi bộ, policy với tay) đã cho kết quả tốt hơn end-to-end — nhưng chỉ khi các low-level policy này (walking, reaching...) đã đủ "vững" từ trước.
- Kết luận chung của nhóm tác giả: **nhiều task trong 27 task vẫn chưa được giải quyết** (unsolved) — HumanoidBench được thiết kế cố tình khó, để còn chỗ cho các phương pháp tương lai cải thiện, không phải benchmark đã "gần bão hoà".

### 5.3. Vì sao nên dùng benchmark có sẵn thay vì tự chế

- So sánh được trực tiếp với số liệu đã công bố của paper khác (cùng task, cùng robot, cùng engine) — tránh việc "phát minh lại" một benchmark riêng khiến không ai so sánh được với bạn.
- Test set và success criterion đã được cộng đồng kiểm chứng, giảm rủi ro bạn vô tình định nghĩa "thành công" theo hướng dễ đạt cho riêng policy của mình (một dạng cherry-picking ở cấp thiết kế benchmark, xem mục 4).

---

## Checklist thực hành

Dùng khi tự thiết kế bộ đánh giá cho policy của bạn trong dự án này (whole-body control ở `04-imitation-learning-rl/`, VLA ở `06-vla-groot-sonic/`):

- [ ] Đã chọn tối thiểu 3–4 metric bổ sung nhau cho motion tracking: MPJPE (hoặc MPJPE-L), success rate, jerk, root trajectory error — không chỉ báo 1 con số duy nhất.
- [ ] Đã định nghĩa rõ "thành công" **trước khi** chạy thí nghiệm (ngưỡng ngã, ngưỡng tracking error...), viết ra thành văn bản, không đổi sau khi thấy kết quả.
- [ ] Nếu có task ngôn ngữ (VLA): đã tách riêng success rate cho seen task và unseen task, không gộp trung bình.
- [ ] Đã chạy robustness test dưới ít nhất 1 trục nhiễu loạn OOD (ánh sáng/vị trí vật thể/môi trường) và báo cáo delta so với điều kiện chuẩn.
- [ ] Nếu tuyên bố sim-to-real: đã nói rõ đây là gap dạng nào (zero-shot / fine-tuned / domain-randomization-coverage) và cùng tập task, cùng điều kiện giữa sim và real.
- [ ] Đã chạy ≥3 seed và báo cáo mean ± std, không phải "best of N" hay 1 lần chạy đẹp.
- [ ] Test set (motion/trajectory dùng để đánh giá) không trùng với tập đã dùng huấn luyện.
- [ ] Nếu so sánh với benchmark cộng đồng (HumanoidBench hoặc tương đương): dùng đúng task/robot/success criterion đã công bố, không sửa đổi mà không ghi chú rõ.

---

*Liên quan: `README.md` (mục A–D) · `../../resources/07-experiments-and-rigor.md` (nguyên tắc thiết kế thí nghiệm tổng quát) · `../04-imitation-learning-rl/`, `../05-simulation-mujoco-isaaclab/` (nơi policy được huấn luyện trước khi đánh giá) · `../08-real-robot-deployment/` (quyết định deploy dựa trên kết quả đánh giá sim-to-real).*
