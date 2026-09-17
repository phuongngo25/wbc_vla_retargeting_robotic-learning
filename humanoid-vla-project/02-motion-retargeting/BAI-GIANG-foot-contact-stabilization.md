# Bài giảng: Ràng buộc vật lý — Foot contact stabilization

*(Thuộc mảng: Motion Retargeting)*

## 🎯 Mục tiêu bài học

- Giải thích được cơ chế phát sinh của hiện tượng foot sliding (trượt chân) và foot floating (lơ lửng) sau khi giải IK per-frame.
- Mô tả được thuật toán phát hiện contact dựa trên vận tốc bàn chân và cách "ghim" (clamp/lock) vị trí trong giai đoạn stance.
- Tính tay được một ví dụ số phát hiện contact bằng ngưỡng vận tốc và tính độ trôi tích luỹ nếu không có bước ổn định.
- So sánh được foot contact stabilization dựa trên heuristic vận tốc với các phương pháp contact-aware hiện đại hơn (2024-2026).
- Nêu được vì sao bước này vẫn cần thiết ngay cả khi dữ liệu nguồn (LAFAN1) đã có chất lượng contact tốt.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Sau khi Inverse Kinematics per-frame (Jacobian-based hoặc FABRIK — hai bài giảng riêng) đã cho ra góc khớp cho từng frame độc lập, kết quả **chưa đảm bảo khả thi vật lý** — vì IK giải mỗi frame tách biệt, không có ràng buộc "frame này phải nhất quán với frame trước" về mặt tiếp xúc. Foot contact stabilization là bước hậu xử lý (post-processing) đầu tiên trong lớp "ràng buộc vật lý" của pipeline retargeting, xử lý đúng vấn đề này cho bàn chân — bộ phận quan trọng nhất về mặt ổn định của robot humanoid.

## 🧠 Trực giác

### Góc nhìn 1: Ký tên trên giấy run tay qua nhiều lần

Hình dung bạn ký tên nhiều lần liên tiếp trên cùng một vị trí giấy, nhưng tay hơi run — mỗi lần ký, điểm đặt bút lệch đi vài milimet so với lần trước dù bạn *có ý định* đặt bút đúng một chỗ. Sau hàng chục lần, vết mực loang ra thành một vùng nhỏ thay vì một điểm — đây chính là "trôi" (drift) tích luỹ. Foot contact stabilization giống như việc bạn **ghim tay lại bằng một cái kẹp** ngay khi phát hiện ý định của bạn là "đặt bút đứng yên tại một điểm trong một khoảng thời gian" — loại bỏ hoàn toàn sự run tay trong khoảng đó.

**Giới hạn của loại suy này:** ký tên là hành động chủ động duy nhất của một tác nhân (bàn tay); trong khi bàn chân robot bị ảnh hưởng bởi **toàn bộ chain phía trên** (hông, gối, mắt cá) — sai số không chỉ đến từ chính khớp cổ chân mà lan truyền từ sai số IK của toàn bộ chuỗi động học, khiến "run tay" ở đây thực chất là hệ quả gián tiếp, không phải lỗi trực tiếp tại điểm chạm.

### Góc nhìn 2: Ảnh chụp nhanh liên tiếp của một vật đang đứng yên

Một phép loại suy khác: hình dung bạn chụp 100 tấm ảnh liên tiếp một chiếc cốc đang đứng yên trên bàn, nhưng máy ảnh mỗi lần đo khoảng cách có sai số nhỏ ngẫu nhiên (nhiễu đo — measurement noise). Nếu ghép 100 tấm ảnh này thành video, chiếc cốc sẽ trông như đang "rung" nhẹ dù thực tế nó đứng yên tuyệt đối. Foot contact stabilization giống việc phát hiện "đây là 100 khung hình của cùng một vật đứng yên" (dựa trên việc vận tốc đo được gần 0) rồi **thay tất cả 100 giá trị đo bằng đúng 1 giá trị trung bình/cố định** — loại bỏ rung do nhiễu đo trong khi vẫn giữ đúng "sự kiện" là vật đang đứng yên.

**Giới hạn của loại suy này:** nhiễu đo trong loại suy này là ngẫu nhiên và độc lập giữa các khung hình; trong khi sai số IK theo frame trong retargeting có thể **tương quan theo thời gian** (drift một chiều tích luỹ, không phải nhiễu trắng ngẫu nhiên đối xứng quanh 0) — nghĩa là trong nhiều trường hợp trôi chân là đơn điệu (monotonic) chứ không dao động qua lại như rung nhiễu đo thuần tuý.

## 📐 Định nghĩa chính xác

**Foot contact stabilization** là bước hậu xử lý áp dụng sau khi giải IK per-frame, gồm hai phần:

**(1) Phát hiện contact (contact detection):** với mỗi frame t và mỗi bàn chân, xác định nhãn nhị phân `contact(t) ∈ {0, 1}` — thường dựa trên **vận tốc bàn chân trong dữ liệu chuyển động nguồn**:

```
contact(t) = 1   nếu  ‖v_foot(t)‖ < v_threshold   (bàn chân gần như đứng yên → đang chạm đất)
contact(t) = 0   nếu  ‖v_foot(t)‖ ≥ v_threshold   (bàn chân đang di chuyển → không chạm đất / swing phase)
```

trong đó `v_foot(t) = (pos_foot(t) − pos_foot(t−1)) / Δt` là vận tốc ước lượng bằng sai phân hữu hạn, và `v_threshold` là một ngưỡng cấu hình được (ví dụ vài cm/s).

**(2) Ghim vị trí (position locking/clamping):** với mỗi đoạn frame liên tiếp có `contact(t) = 1` (một "giai đoạn stance"), thay vì giữ nguyên vị trí bàn chân đã tính từ IK (có thể trôi nhẹ qua từng frame), hệ thống **ghim** (lock) vị trí bàn chân về một giá trị cố định duy nhất trong suốt giai đoạn đó — thường là vị trí tại frame đầu tiên của giai đoạn stance, hoặc trung bình của cả giai đoạn:

```
pos_foot_ổn_định(t) = pos_foot(t_bắt_đầu_stance)   với mọi t trong cùng giai đoạn stance
```

Việc ghim này sau đó được lan truyền ngược lại thành **ràng buộc vị trí mục tiêu mới** cho IK (giải lại IK với mục tiêu bàn chân = vị trí đã ghim), hoặc — trong các hệ thống đơn giản hơn — áp trực tiếp lên góc khớp chân bằng một bước IK cục bộ bổ sung chỉ cho chain chân.

## ⚙️ Cơ chế hoạt động — từng bước

```
Input: chuỗi vị trí bàn chân sau IK per-frame: pos_foot(1), pos_foot(2), ..., pos_foot(T)
       chuỗi vận tốc bàn chân từ DỮ LIỆU NGUỒN (người): v_foot_nguồn(1..T)

┌──────────────────────────────────────────────────────────────┐
│  BƯỚC 1 — Phát hiện contact cho toàn bộ chuỗi                    │
│    for t = 1 to T:                                                │
│        contact(t) = (‖v_foot_nguồn(t)‖ < v_threshold) ? 1 : 0     │
│    → chuỗi nhãn: [0,0,1,1,1,1,1,0,0,0,1,1,1,...]                  │
│                                                                    │
│  BƯỚC 2 — Nhóm thành các "giai đoạn stance" liên tiếp              │
│    stance_1 = [t=3..7], stance_2 = [t=11..15], ...                │
│                                                                    │
│  BƯỚC 3 — Với mỗi giai đoạn stance, ghim vị trí                    │
│    p_ghim = pos_foot(t_bắt_đầu_stance)   (hoặc trung bình)         │
│    for t in stance: pos_foot_ổn_định(t) = p_ghim                  │
│                                                                    │
│  BƯỚC 4 — Với các frame KHÔNG ở stance (swing phase)               │
│    giữ nguyên pos_foot(t) từ IK (chân đang di chuyển trên không,   │
│    không cần ổn định vị trí tuyệt đối)                             │
│                                                                    │
│  BƯỚC 5 — (tuỳ hệ thống) giải lại IK cục bộ cho chain chân          │
│    với mục tiêu = pos_foot_ổn_định(t) đã ghim ở BƯỚC 3              │
└──────────────────────────────────────────────────────────────┘
                    │
                    ▼
        Chuỗi chuyển động cuối: bàn chân đứng yên tuyệt đối
        trong giai đoạn stance, di chuyển bình thường ở swing
```

Điểm quan trọng: **nhãn contact được lấy từ dữ liệu nguồn (người)**, không phải từ kết quả IK của robot — vì mục đích là bảo toàn "ý định chuyển động" (chân người đang đứng yên) chứ không phải chữa cháy theo kết quả robot (nếu dùng vận tốc bàn chân robot sau IK để phát hiện contact, ta sẽ bỏ lỡ chính xác các frame mà bàn chân *lẽ ra* phải đứng yên nhưng IK đã làm nó trôi).

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số tự chọn để dễ hình dung)*

Giả sử ngưỡng vận tốc `v_threshold = 2 cm/s`, tần số dữ liệu 30 fps (Δt ≈ 0.0333s). Xét chuỗi vị trí bàn chân trái theo trục x (đơn giản hoá 1D) tính được từ IK per-frame qua 6 frame liên tiếp:

| Frame t | pos_foot_IK(t) (cm) | v_foot_nguồn(t) = Δpos_nguồn/Δt (cm/s) | contact(t)? |
|---|---|---|---|
| 1 | 10.00 | — (frame đầu) | — |
| 2 | 10.08 | 1.5 | 1 (đứng yên) |
| 3 | 10.15 | 1.2 | 1 |
| 4 | 10.21 | 1.4 | 1 |
| 5 | 12.60 | 45.0 | 0 (đang bước) |
| 6 | 15.90 | 52.0 | 0 |

Ở frame 2–4, `v_foot_nguồn < 2 cm/s` → contact=1 → đây là giai đoạn stance dù kết quả IK cho thấy vị trí bàn chân **vẫn dịch chuyển nhẹ mỗi frame** (10.00→10.08→10.15→10.21 cm — trôi tích luỹ 0.21 cm qua 3 frame do sai số IK/scale). Nếu không có foot contact stabilization, đến frame 4 bàn chân đã "trôi" 0.21 cm so với vị trí ban đầu — nhỏ nhưng **tích luỹ tuyến tính theo số frame trong giai đoạn stance**: với một giai đoạn stance dài 30 frame (1 giây ở 30fps) và tốc độ trôi trung bình tương tự (~0.07 cm/frame), tổng độ trôi có thể lên tới `30 × 0.07 ≈ 2.1 cm` — đủ lớn để quan sát được bằng mắt thường trên video, và đủ lớn để gây bất ổn nếu robot thật cố "tin" vào các vị trí bàn chân này khi bộ điều khiển thân dưới giả định chân đang tiếp xúc chắc chắn.

**Sau khi áp foot contact stabilization:** với p_ghim = pos_foot_IK(2) = 10.08 cm (vị trí tại frame đầu tiên của giai đoạn stance), toàn bộ frame 2, 3, 4 được gán lại: `pos_foot_ổn_định(2)=pos_foot_ổn_định(3)=pos_foot_ổn_định(4) = 10.08 cm`. Độ trôi trong giai đoạn stance giảm từ 0.21 cm xuống **đúng 0 cm** — bàn chân đứng yên tuyệt đối như dữ liệu nguồn dự định.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Foot contact stabilization (heuristic vận tốc + ghim) | Contact-aware optimization/RL hiện đại (2024-2026) |
|---|---|---|
| Cơ chế phát hiện contact | Ngưỡng vận tốc đơn giản, tính trực tiếp từ dữ liệu nguồn | Học/tối ưu hoá đồng thời với toàn bộ động lực học (dynamics), có thể dùng mô hình tiếp xúc vật lý đầy đủ |
| Chi phí tính toán | Rất thấp — hậu xử lý đơn giản sau IK | Cao hơn nhiều — cần mô phỏng vật lý hoặc huấn luyện RL |
| Xử lý tương tác với các ràng buộc khác (joint limit, cân bằng CoM) | Tách biệt — xử lý riêng từng bước | Tích hợp đồng thời trong một bài toán tối ưu chung |
| Độ chính xác vật lý (đảm bảo động lực học khả thi) | Không đảm bảo — chỉ đúng về mặt hình học/kinematic | Có thể đảm bảo tốt hơn vì tính tới lực, mô-men, ma sát |
| Sử dụng trong dự án | GMR, SOMA-retargeter (cả hai bài giảng riêng) đều dùng heuristic vận tốc dạng này | Hướng nghiên cứu mở rộng (CoRe, ReActor — xem mục Cập nhật hiện đại) |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "chỉ cần phát hiện contact dựa trên kết quả IK của robot (không cần nhìn dữ liệu nguồn) là đủ".** Vì sao sai: nếu IK đã làm bàn chân trôi nhiều, vận tốc **của chính kết quả IK** cũng sẽ bị nhiễu bởi chính lỗi cần sửa — dùng dữ liệu đã lỗi để phát hiện lỗi là vòng lặp tự tham chiếu không đáng tin. **Hiểu đúng:** nhãn contact phải lấy từ **ý định chuyển động gốc** (vận tốc bàn chân trong dữ liệu người), phản ánh đúng "chân có nên đứng yên hay không" độc lập với sai số retargeting.
2. **Hiểu nhầm: "ghim chặt bàn chân trong mọi giai đoạn stance sẽ luôn cho kết quả tốt hơn, càng ghim chặt càng an toàn".** Vì sao sai: nếu ngưỡng `v_threshold` đặt quá cao, các frame chuyển tiếp (gần bắt đầu/kết thúc bước chân, vận tốc nhỏ nhưng khác 0 có chủ đích) có thể bị nhãn nhầm thành "stance" và bị ghim cứng, tạo ra chuyển động "giật cục" khi chuyển từ ghim sang tự do (discontinuity ở biên giai đoạn). **Hiểu đúng:** ngưỡng cần hiệu chỉnh theo tốc độ chuyển động cụ thể (đi bộ chậm khác chạy nhanh), và nên có làm mượt (blending) ở biên giai đoạn stance/swing thay vì chuyển đổi cứng nhắc.

## 🏗️ Ví dụ minh hoạ trong dự án này

Cả hai công cụ retargeting chính của dự án — **GMR** và **SOMA-retargeter** — đều tích hợp foot contact stabilization như một bước bắt buộc trong pipeline: GMR áp dụng nó cùng với ràng buộc velocity limit ngay trong vòng lặp giải QP (xem bài giảng "GMR — kiến trúc và pipeline"), còn SOMA-retargeter thực hiện nó như bước 4 riêng biệt sau khi giải IK trên GPU (xem bài giảng "SOMA-retargeter — kiến trúc và pipeline"), trước khi xuất CSV. Ý nghĩa thực tiễn: khi retarget một chuỗi đi bộ từ AMASS sang Unitree G1, nếu tắt bước này (để kiểm chứng), video kết quả trong MuJoCo viewer sẽ cho thấy rõ bàn chân G1 "lướt nhẹ" trên sàn trong lúc lẽ ra phải đứng yên hoàn toàn — một lỗi thị giác dễ nhận biết, và là dấu hiệu đầu tiên các mentor trong dự án dùng để đánh giá chất lượng một lần chạy retargeting.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **Retargeting artifacts (bao gồm foot sliding) được xác nhận ảnh hưởng trực tiếp tới chất lượng huấn luyện RL downstream.** Paper GMR chính thức (Araujo et al., arXiv:2510.02252, ICRA 2026) ghi nhận: "artifacts in retargeted data significantly reduce policy robustness, particularly for dynamic or long sequences" — cho thấy foot contact stabilization không chỉ là vấn đề thẩm mỹ mà ảnh hưởng trực tiếp tới bước RL sau này (`04-imitation-learning-rl/`). [arXiv:2510.02252](https://arxiv.org/abs/2510.02252)
2. **CoRe (2024-2025) tích hợp contact-aware refinement như một pha riêng trong pipeline tự động end-to-end.** *"CoRe: A Hybrid Approach of Contact-Aware Optimization and Learning for Humanoid Robot Motions"* mô tả một pipeline gồm sinh chuyển động từ text → retargeting → **optimization-based motion refinement** (đúng vai trò của foot contact stabilization nhưng mở rộng, xử lý cả self-penetration và joint acceleration quá lớn) → RL với "contact-aware rewards" — cho thấy xu hướng 2024-2025 là gộp bước ổn định tiếp xúc heuristic (như trong bài này) với các ràng buộc vật lý khác thành một bước tối ưu hoá thống nhất, thay vì xử lý tách biệt như GMR/SOMA-retargeter hiện tại. [Illinois Experts](https://experts.illinois.edu/en/publications/core-a-hybrid-approach-of-contact-aware-optimization-and-learning/)
3. **Nhiều nghiên cứu 2024-2026 để lại foot sliding cho tầng RL sửa thay vì sửa hết ở tầng retargeting.** Theo tổng hợp trong *"Human2Humanoid: Physics-Aware Cross-Morphology Motion Retargeting"* và các paper liên quan, nhiều pipeline hiện tại "leave artifacts such as foot sliding in the reference trajectories for the RL policy to correct" — tức là coi foot contact stabilization ở tầng retargeting chỉ là bước giảm nhẹ (không cần hoàn hảo), vì tầng RL tracking phía sau (với contact reward) có thể tự sửa phần còn sót — đây là một triết lý thiết kế khác với cách GMR/SOMA-retargeter làm (cố gắng sửa triệt để ở tầng retargeting). [arXiv:2606.03476](https://arxiv.org/html/2606.03476)

## ❓ Câu hỏi tự kiểm tra

1. Vì sao nhãn contact(t) nên được tính từ vận tốc bàn chân trong dữ liệu nguồn (người) thay vì từ kết quả IK của robot?
   <details><summary>Gợi ý đáp án</summary>Vì kết quả IK có thể đã bị trôi (chính là lỗi cần phát hiện) — dùng nó để phát hiện lỗi tạo vòng lặp tự tham chiếu không đáng tin; dữ liệu nguồn phản ánh đúng ý định chuyển động độc lập với sai số retargeting.</details>
2. Trong ví dụ tính tay, vì sao độ trôi tích luỹ có thể lên tới ~2cm dù mỗi frame chỉ trôi rất nhỏ?
   <details><summary>Gợi ý đáp án</summary>Vì độ trôi mỗi frame (dù nhỏ, ví dụ ~0.07cm) được cộng dồn qua nhiều frame liên tiếp trong cùng một giai đoạn stance (ví dụ 30 frame ≈ 1 giây) — sai số tích luỹ tuyến tính theo số frame.</details>
3. Hậu quả gì xảy ra nếu ngưỡng v_threshold đặt quá thấp (gần 0)?
   <details><summary>Gợi ý đáp án</summary>Rất ít frame được nhận là contact=1 (vì hầu như luôn có chút vận tốc khác 0 do nhiễu đo), dẫn tới bỏ sót nhiều giai đoạn lẽ ra cần ghim, foot sliding không được sửa.</details>
4. Foot contact stabilization giải quyết vấn đề gì mà chỉ riêng IK per-frame (dù chính xác tuyệt đối tại mỗi frame) không thể giải quyết được?
   <details><summary>Gợi ý đáp án</summary>IK per-frame giải độc lập từng frame, không có ràng buộc nhất quán liên-frame ("frame này phải giống hệt frame trước nếu chân đang đứng yên") — đây là vấn đề thời gian (temporal), không phải vấn đề hình học tại một thời điểm mà IK per-frame giải quyết.</details>
5. Vì sao một số hệ thống hiện đại (2024-2026) chọn "để lại" foot sliding cho tầng RL sửa thay vì cố sửa hết ở tầng retargeting?
   <details><summary>Gợi ý đáp án</summary>Vì tầng RL tracking phía sau có thể có contact-aware reward và tương tác trực tiếp với mô phỏng vật lý đầy đủ (lực, ma sát), sửa lỗi triệt để hơn về mặt động lực học so với heuristic hình học ở tầng retargeting — đây là một đánh đổi thiết kế giữa xử lý sớm (retargeting) và xử lý muộn (RL).</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể:** với ngưỡng v_threshold=3cm/s, và chuỗi vận tốc bàn chân nguồn tự chọn khác (7 frame, tự đặt số liệu mô phỏng một giai đoạn stance dài hơn xen giữa 2 giai đoạn swing), xác định các giai đoạn stance, tính độ trôi tích luỹ trước và sau khi ghim.
2. **Đọc code thật:** trong repo GMR hoặc SOMA-retargeter, tìm đoạn code xử lý foot contact detection (thường có tên như `detect_contact`, `foot_contact`, hoặc tương tự) — xác định ngưỡng vận tốc thực tế được dùng và so sánh với con số minh hoạ (2cm/s) trong bài này.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

Foot contact stabilization là bước hậu xử lý sau IK per-frame, phát hiện các frame mà dữ liệu chuyển động nguồn coi là "chân chạm đất" (dựa trên ngưỡng vận tốc bàn chân trong dữ liệu người) rồi ghim vị trí bàn chân robot đứng yên tuyệt đối trong suốt giai đoạn đó, loại bỏ hiện tượng trôi/trượt (foot sliding) tích luỹ qua các frame — như ví dụ tính tay ở trên cho thấy, một độ trôi nhỏ mỗi frame có thể cộng dồn thành vài centimet quan sát được nếu không xử lý; cả GMR lẫn SOMA-retargeter đều tích hợp bước này như một phần bắt buộc của pipeline, và các nghiên cứu 2024-2026 (GMR paper, CoRe) xác nhận chất lượng xử lý tiếp xúc ảnh hưởng trực tiếp tới độ vững của policy RL học từ dữ liệu retarget, dù một số hướng nghiên cứu mới hơn chọn để lại một phần lỗi này cho tầng RL sửa bằng contact-aware reward thay vì cố sửa triệt để ngay ở tầng retargeting.
