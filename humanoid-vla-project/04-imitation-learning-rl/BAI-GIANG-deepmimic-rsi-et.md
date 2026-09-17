# Bài giảng: DeepMimic — Reference State Initialization & Early Termination

*(Thuộc mảng: Imitation Learning & Reinforcement Learning)*

## 🎯 Mục tiêu bài học

- Giải thích được vấn đề "khám phá khó" (hard exploration) mà DeepMimic gặp phải khi bắt chước động tác động học phức tạp (nhào lộn, đá, xoay người).
- Định nghĩa chính xác **Reference State Initialization (RSI)** và cơ chế nó tăng tốc học ra sao.
- Định nghĩa chính xác **Early Termination (ET)** và giải thích vì sao thiếu nó khiến policy học hành vi "chấp nhận ngã".
- Giải thích được cơ chế reward pha trộn imitation + task reward của DeepMimic.
- Tính tay được ví dụ minh hoạ về việc RSI làm giảm "quãng đường khám phá" cần thiết ra sao.
- Nêu được ít nhất 2 công trình 2024-2026 kế thừa trực tiếp RSI/ET cho humanoid thật.

## 🧭 Vì sao cần học cái này? (bối cảnh)

DeepMimic (2018) là bài giảng nền móng của toàn bộ chuỗi phát triển "PHC → OmniH2O → ASAP → SONIC" (xem bài giảng riêng). Trước khi hiểu AMP thay đổi gì so với DeepMimic, hay ASAP/SONIC kế thừa gì, bạn cần hiểu chính xác DeepMimic đã giải quyết vấn đề gì và bằng cơ chế nào. Bài này tập trung vào đúng 2 kỹ thuật huấn luyện cốt lõi của DeepMimic — RSI và ET — cùng với cách chúng phối hợp với motion-tracking reward (đã học ở bài giảng trước) để bắt chước thành công một clip mocap cụ thể.

## 🧠 Trực giác

### Góc nhìn 1: Học bơi bằng cách nhảy vào giữa hồ ở nhiều điểm khác nhau (RSI) thay vì luôn nhảy từ bờ (khởi tạo cố định)

Nếu bạn muốn học bơi 50m bướm hoàn chỉnh, và mỗi lần luyện tập đều bắt buộc phải bơi từ đầu bể tới cuối, bạn sẽ **luôn thất bại ở đoạn cuối** trước khi có cơ hội luyện riêng đoạn đó (vì phải bơi hết 40m đầu mới tới được đoạn 40-50m để luyện). RSI giống việc huấn luyện viên cho phép bạn **nhảy thẳng vào bất kỳ điểm nào trong bể** (10m, 25m, 45m...) ở mỗi buổi tập — bạn có thể luyện đoạn khó (45-50m) ngay từ đầu, song song với luyện đoạn dễ, thay vì phải "sống sót" qua toàn bộ hành trình trước.

*Đúng ở đâu:* nắm đúng vấn đề "học tuần tự từ đầu rất chậm với chuỗi dài" và cách RSI giải quyết bằng lấy mẫu ngẫu nhiên điểm khởi đầu.
*Giới hạn:* phép loại suy không phản ánh đúng rằng RSI không chỉ giúp *tốc độ* học mà còn giúp học được những kỹ năng **về nguyên tắc không thể học được nếu không có nó** — với backflip (nhào lộn ra sau), nếu luôn khởi tạo từ đầu clip, agent gần như không bao giờ tình cờ thực hiện đúng pha giữa không trung (mid-air) một cách ngẫu nhiên đủ để nhận reward dương ở đó — không phải vấn đề "chậm", mà là vấn đề "gần như không thể xảy ra".

### Góc nhìn 2: Ngắt trận đấu ngay khi phạm lỗi nghiêm trọng (ET) thay vì chơi hết hiệp dù đã "chết"

Hãy tưởng tượng một trò chơi điện tử huấn luyện AI, nơi nhân vật ngã gục nhưng trò chơi vẫn tiếp tục chạy đủ 60 giây còn lại của màn chơi (dù nhân vật nằm bất động). AI vẫn nhận một ít điểm rải rác trong 60 giây đó (ví dụ điểm sống sót), nên nó có thể học ra rằng "cứ ngã xong nằm im" vẫn tạo ra tổng điểm chấp nhận được — không có động lực đủ mạnh để tránh ngã. Early Termination giống việc **ngắt trò chơi ngay lập tức** khi phát hiện "game over" rõ ràng (chạm đất) — nhân vật không còn cơ hội tích luỹ thêm bất kỳ điểm nào từ trạng thái ngã, buộc AI phải học tránh ngã bằng mọi giá nếu muốn tối đa hoá điểm.

*Đúng ở đâu:* nắm đúng cơ chế "loại bỏ động lực chấp nhận thất bại" của ET.
*Giới hạn:* phép loại suy không nói rõ **tác dụng phụ liên quan tới RSI**: vì ET cắt ngắn episode ngay khi ngã, kết hợp RSI (khởi tạo ngẫu nhiên ở giữa clip), một episode có thể chỉ kéo dài vài bước nếu điểm khởi tạo là một tư thế khó dẫn tới ngã gần như ngay lập tức — đây là điều cần thiết (loại bỏ nhanh các điểm khởi tạo/hành vi tệ) chứ không phải lỗi, nhưng người mới dễ nhầm là "huấn luyện không ổn định" khi thấy episode length dao động mạnh.

## 📐 Định nghĩa chính xác

DeepMimic (Peng, Abbeel, Levine, van de Panne, 2018, [arXiv:1804.02717](https://arxiv.org/abs/1804.02717)) huấn luyện policy π_θ bằng PPO (xem bài giảng "PPO vs SAC") với ba cơ chế cốt lõi:

**1. Reward pha trộn:**
```
r_t = w_I · r_imitation(t) + w_T · r_task(t)
```
- r_imitation(t): motion-tracking reward (xem bài giảng riêng) — khoảng cách pose/velocity/root so với frame tham chiếu tại thời điểm t trong clip mocap.
- r_task(t): reward bổ sung tùy bài toán cụ thể (ví dụ đá bóng trúng đích, đi theo hướng mong muốn) — cho phép policy thích nghi động tác thay vì sao chép y hệt.
- w_I, w_T: trọng số tương đối (hyperparameter).

**2. Reference State Initialization (RSI):** tại đầu mỗi episode huấn luyện, thay vì luôn khởi tạo state của robot mô phỏng tại frame đầu tiên (t=0) của clip mocap, RSI lấy mẫu một chỉ số thời gian ngẫu nhiên t₀ ~ Uniform(0, T_clip) và khởi tạo state mô phỏng **khớp với state tham chiếu tại t₀** (góc khớp, vận tốc, vị trí root đều được set trực tiếp bằng giá trị mocap tại t₀), rồi episode tiếp tục chạy từ đó.

**3. Early Termination (ET):** episode bị chấm dứt ngay lập tức (trước khi đạt độ dài tối đa quy định) tại thời điểm t nếu một điều kiện thất bại được kích hoạt — phổ biến nhất là **một bộ phận thân không được phép chạm đất (ví dụ torso, đầu) chạm sàn**, hoặc pose lệch quá xa reference tới mức không thể phục hồi. Khi ET kích hoạt, không có thêm reward nào được cộng dồn từ thời điểm đó trở đi trong episode này.

## ⚙️ Cơ chế hoạt động — từng bước

```
┌────────────────────────────────────────────────────────────────┐
│  BẮT ĐẦU MỘT EPISODE HUẤN LUYỆN (lặp lại hàng nghìn lần song   │
│  song trên nhiều môi trường mô phỏng)                            │
├────────────────────────────────────────────────────────────────┤
│ 1. RSI: lấy mẫu t₀ ~ Uniform(0, T_clip) ngẫu nhiên               │
│    Set state robot mô phỏng = state tham chiếu tại t₀            │
│    (copy trực tiếp góc khớp, vận tốc, root pose từ mocap)        │
├────────────────────────────────────────────────────────────────┤
│ 2. Với mỗi bước thời gian t = t₀, t₀+1, ...:                     │
│    a. Policy π_θ quan sát state hiện tại (kèm thông tin pha t   │
│       trong clip — "phase variable"), chọn hành động a_t         │
│    b. Mô phỏng vật lý cập nhật state robot                        │
│    c. Tính r_t = w_I·r_imitation(t) + w_T·r_task(t)               │
│    d. KIỂM TRA ĐIỀU KIỆN ET:                                      │
│       - torso/đầu chạm sàn? → CHẤM DỨT episode ngay tại đây      │
│       - lệch quá xa reference? → CHẤM DỨT episode ngay tại đây  │
│       - không vi phạm → tiếp tục bước tiếp theo                  │
├────────────────────────────────────────────────────────────────┤
│ 3. Episode kết thúc khi: hết clip mocap, HOẶC ET kích hoạt        │
├────────────────────────────────────────────────────────────────┤
│ 4. Toàn bộ trajectory (có thể rất ngắn nếu ET sớm, hoặc dài nếu  │
│    hoàn thành cả clip) đưa vào batch huấn luyện PPO               │
└────────────────────────────────────────────────────────────────┘
```

Lưu ý cơ chế phối hợp: RSI quyết định **"episode bắt đầu ở đâu trong không gian pha động tác"**, còn ET quyết định **"episode kết thúc khi nào nếu thất bại"** — hai cơ chế độc lập nhưng cộng hưởng: RSI đảm bảo agent được luyện tập đều ở mọi giai đoạn của động tác (kể cả đoạn khó ở giữa/cuối), ET đảm bảo agent không lãng phí thời gian huấn luyện ở các trạng thái vô nghĩa sau khi đã thất bại rõ ràng.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số liệu tự chọn để dễ hình dung.)*

Giả sử một clip mocap "backflip" (nhào lộn ra sau) dài T_clip = 90 frame (tại 30 FPS, tương đương 3 giây), trong đó pha khó nhất — giai đoạn giữa không trung khi cơ thể xoay úp — nằm ở frame 50-65 (khoảng giây thứ 1.67-2.17).

**Không có RSI** (luôn khởi tạo tại frame 0): để agent "chạm" tới được pha khó (frame 50), nó phải sống sót đúng qua 50 bước đầu tiên mà không bị ET kích hoạt sớm. Giả sử xác suất policy ngẫu nhiên ban đầu sống sót an toàn qua MỖI bước (không ngã) chỉ khoảng p = 0.9 (ước lượng minh hoạ cho giai đoạn đầu huấn luyện, policy còn kém). Xác suất một episode "chạm" được tới frame 50 mà chưa bị ET:

```
P(sống sót 50 bước) = p^50 = 0.9^50 ≈ 0.00515 (khoảng 0.5%)
```

Nghĩa là trung bình phải chạy khoảng 1/0.00515 ≈ 194 episode mới có MỘT episode chạm được tới đúng đoạn khó để agent có cơ hội học nó.

**Có RSI:** vì episode có thể khởi tạo trực tiếp tại t₀ ∈ [50, 65] với xác suất 15/90 ≈ 16.7% mỗi lần lấy mẫu (giả định uniform trên 90 frame), số episode trung bình cần để "chạm" đoạn khó chỉ là 1/0.167 ≈ 6 episode — nhanh hơn khoảng 194/6 ≈ 32 lần so với không có RSI trong ví dụ minh hoạ này.

**Diễn giải:** đây là lý do chính paper DeepMimic khẳng định RSI "thiết yếu" cho các kỹ năng có giai đoạn bay (flight phase) dài như backflip — không phải chỉ nhanh hơn một chút, mà là chênh lệch nhiều bậc độ lớn về số lượng thử nghiệm cần thiết để agent có cơ hội học đoạn khó.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | DeepMimic (RSI+ET, bám 1 clip) | AMP (discriminator, nhiều clip) |
|---|---|---|
| Reward | Khoảng cách pose chính xác từng frame so với 1 reference cụ thể | Điểm discriminator đo "độ tự nhiên nói chung" |
| Cần RSI/ET? | Có, thiết yếu cho kỹ năng động học phức tạp | Vẫn thường dùng ET tương tự, nhưng khái niệm "frame tham chiếu chính xác" ít quan trọng hơn |
| Khả năng tổng quát qua nhiều clip | Hạn chế — dễ overfit 1 clip | Tốt hơn — học khái niệm chung |
| Yêu cầu đồng bộ thời gian (phase alignment) giữa robot và reference | Bắt buộc (cần biết chính xác t để so khớp) | Không bắt buộc chặt như DeepMimic |

*(Xem bài giảng riêng "AMP — cơ chế discriminator" để so sánh chi tiết hơn.)*

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "RSI chỉ đơn giản là chia nhỏ episode ra huấn luyện riêng từng đoạn."** Sai — RSI không chia episode; mỗi episode huấn luyện với RSI vẫn có thể chạy dài (tới hết clip hoặc tới khi ET), chỉ có **điểm khởi tạo** là ngẫu nhiên. Một episode khởi tạo tại giữa clip vẫn phải tiếp tục mô phỏng và học cách hoàn thành phần còn lại của động tác, không dừng lại giữa chừng một cách nhân tạo.

2. **Hiểu nhầm: "Early Termination là một dạng phạt (reward âm) khi ngã."** Sai — ET không nhất thiết là thêm reward âm; bản chất của nó là **chấm dứt việc thu thập thêm reward** (kể cả reward dương nhỏ) từ thời điểm đó trở đi. Cơ chế hoạt động qua việc *cắt ngắn tổng reward tích luỹ có thể đạt được* (giảm return kỳ vọng của những trajectory dẫn tới ngã), chứ không phải qua một số hạng phạt cộng thêm vào công thức reward tại bước đó (dù một số triển khai có thể kết hợp cả hai).

3. **Hiểu nhầm: "RSI/ET là 2 kỹ thuật độc lập, không liên quan gì nhau."** Không hoàn toàn đúng — chúng cộng hưởng: RSI làm tăng đa dạng điểm khởi tạo (kể cả các điểm "khó", dễ ngã hơn), khiến ET càng quan trọng hơn để nhanh chóng loại bỏ các episode thất bại phát sinh từ những điểm khởi tạo khó đó, tránh lãng phí ngân sách huấn luyện vào các trajectory vô nghĩa.

## 🏗️ Ví dụ minh hoạ trong dự án này

Trong chuỗi phát triển PHC → OmniH2O → ASAP → SONIC (xem bài giảng riêng), RSI là kỹ thuật huấn luyện nền tảng vẫn tiếp tục được kế thừa: một policy motion-tracking tổng quát (theo dõi hàng trăm/nghìn clip khác nhau) cần khởi tạo ngẫu nhiên cả về **clip nào** lẫn **frame nào trong clip đó** để đảm bảo phủ đều không gian huấn luyện khổng lồ — đây chính là lý do các paper hiện đại hơn (KungfuBot, Visual Imitation..., xem mục Cập nhật hiện đại) vẫn trích dẫn và tái sử dụng nguyên lý RSI dù đã cách DeepMimic gần một thập kỷ. ET cũng là cơ chế bắt buộc trong huấn luyện song song quy mô lớn (`../05-simulation-mujoco-isaaclab/`) — với hàng nghìn môi trường ảo chạy đồng thời, việc tự động reset môi trường ngay khi robot ngã (thay vì để nó "nằm" hết episode) là điều kiện tiên quyết để huấn luyện hiệu quả trên GPU.

## 🔥 Cập nhật hiện đại / SOTA gần đây

- **KungfuBot** ([arXiv:2506.12851](https://arxiv.org/html/2506.12851v1), 2025, "Physics-Based Humanoid Whole-Body Control for Learning Highly-Dynamic Skills") sử dụng trực tiếp Reference State Initialization — khởi tạo state robot từ các pha thời gian được lấy mẫu ngẫu nhiên trong reference motion — để tạo điều kiện học song song các giai đoạn khác nhau của động tác và cải thiện đáng kể hiệu quả huấn luyện, áp dụng cho các kỹ năng highly-dynamic (võ thuật, động tác nhanh) trên humanoid thật — bằng chứng cho thấy RSI (2018) vẫn là thành phần thiết yếu trong pipeline motion-tracking humanoid hiện đại nhất năm 2025.
- **"Visual Imitation Enables Contextual Humanoid Control"** ([arXiv:2505.03729](https://arxiv.org/html/2505.03729v1), 2025) triển khai RSI kết hợp thêm kỹ thuật **motion load balancing** — một cải tiến bổ sung nhằm cân bằng tần suất lấy mẫu giữa các clip/pha khác nhau trong tập dữ liệu huấn luyện lớn, giải quyết vấn đề RSI gốc (2018) không tính tới: khi huấn luyện trên hàng trăm clip cùng lúc, lấy mẫu uniform thuần tuý có thể khiến các clip/pha "dễ" được luyện quá nhiều so với các clip/pha "khó".
- **RoboMirror** ([arXiv:2512.23649](https://arxiv.org/pdf/2512.23649), 2025) tiếp tục dùng khung RSI, lấy mẫu đều biến pha thời gian (uniform time-phase sampling) để ngẫu nhiên hoá điểm khởi đầu mà policy phải theo dõi — cho thấy RSI đã trở thành một "thành phần chuẩn" gần như mặc định trong bất kỳ pipeline motion-tracking humanoid nào công bố năm 2025, không còn được coi là một đóng góp riêng cần bàn cãi mà là hạ tầng nền của cả lĩnh vực.

## ❓ Câu hỏi tự kiểm tra

1. Tại sao DeepMimic đặc biệt nhấn mạnh RSI là "thiết yếu" cho các kỹ năng có giai đoạn bay (flight phase), thay vì áp dụng đều cho mọi loại động tác?
<details><summary>Gợi ý đáp án</summary>Vì các giai đoạn bay xảy ra ở giữa/cuối động tác và yêu cầu chuỗi hành động chính xác trước đó để "vào" đúng pha bay — nếu luôn học từ đầu, xác suất một policy ngẫu nhiên ban đầu tình cờ tới đúng pha bay mà chưa ngã là cực thấp (minh hoạ ở mục Ví dụ tính tay), khiến việc học gần như không khả thi nếu không có RSI.</details>

2. Nếu bỏ Early Termination nhưng vẫn giữ RSI, điều gì có khả năng xảy ra với hành vi học được?
<details><summary>Gợi ý đáp án</summary>Policy có thể học "chấp nhận ngã sớm rồi nằm im" nếu vẫn tích luỹ được một phần reward nhỏ trong phần còn lại episode (ví dụ reward root pose gần đúng dù đã ngã) — làm loãng tín hiệu huấn luyện với các trajectory vô nghĩa, như đã nêu trong định nghĩa ET.</details>

3. Trong ví dụ tính tay, nếu clip "backflip" dài gấp đôi (T_clip=180 frame) nhưng pha khó vẫn chỉ chiếm 15 frame, xác suất lấy trúng pha khó bằng RSI thay đổi ra sao, và điều đó có ý nghĩa gì?
<details><summary>Gợi ý đáp án</summary>Xác suất giảm còn 15/180 ≈ 8.3% (giảm một nửa so với 16.7% ban đầu) — vẫn tốt hơn nhiều so với việc phải "sống sót" tuần tự qua toàn bộ clip dài hơn nếu không có RSI, nhưng cho thấy hiệu quả của RSI giảm dần khi tỉ lệ đoạn khó/tổng clip nhỏ đi, ngụ ý có thể cần lấy mẫu KHÔNG đều (ưu tiên đoạn khó hơn) để tối ưu hơn — đúng như hướng cải tiến "motion load balancing" nêu ở mục Cập nhật hiện đại.</details>

4. Vì sao các paper 2025 (KungfuBot, Visual Imitation...) vẫn trích dẫn và dùng lại nguyên bản ý tưởng RSI từ 2018 thay vì thay thế bằng kỹ thuật hoàn toàn mới?
<details><summary>Gợi ý đáp án</summary>Vì vấn đề cốt lõi mà RSI giải quyết (khó khám phá tuần tự với chuỗi dài) không thay đổi theo thời gian hay quy mô dữ liệu — dù dữ liệu/mô hình lớn hơn nhiều so với 2018, bài toán "làm sao agent chạm được mọi pha của động tác trong huấn luyện" vẫn cần một cơ chế khởi tạo linh hoạt tương tự; các cải tiến (như motion load balancing) chỉ tinh chỉnh THÊM vào nguyên lý gốc, không thay thế nó.</details>

## 📝 Bài tập thực hành

1. Đọc mã nguồn (nếu có sẵn triển khai open-source của DeepMimic hoặc một môi trường motion-tracking tương tự trong Isaac Lab/MuJoCo Playground) và tìm đoạn code xử lý reset môi trường — xác định chính xác dòng nào lấy mẫu t₀ ngẫu nhiên (RSI) và dòng nào kiểm tra điều kiện ET (thường kiểm tra chiều cao/góc nghiêng torso).
2. Tính lại ví dụ ở mục "Ví dụ tính tay" với một giả định khác: xác suất sống sót mỗi bước của policy ở giai đoạn huấn luyện muộn hơn (đã học tốt hơn) là p=0.98 thay vì 0.9. Tính lại số episode trung bình cần thiết (không RSI) để chạm frame 50, và so sánh mức độ "lợi thế của RSI" có còn lớn như giai đoạn đầu huấn luyện hay không.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

DeepMimic (2018) đặt nền móng cho motion-tracking-as-RL bằng ba cơ chế: reward pha trộn imitation+task cho phép policy vừa bắt chước vừa thích nghi mục tiêu, Reference State Initialization (RSI) khởi tạo episode từ một frame ngẫu nhiên trong clip mocap để agent học song song mọi giai đoạn động tác thay vì phải "sống sót" tuần tự từ đầu, và Early Termination (ET) chấm dứt episode ngay khi phát hiện thất bại rõ ràng để tránh lãng phí huấn luyện vào trạng thái vô nghĩa — hai kỹ thuật này được chính paper gốc khẳng định là thiết yếu cho các kỹ năng động học phức tạp (backflip, kicks), và vẫn được kế thừa gần như nguyên bản trong các công trình humanoid 2025 như KungfuBot, cho thấy sức sống bền vững của ý tưởng này qua gần một thập kỷ.
