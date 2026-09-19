# Bài giảng: Domain randomization (khái niệm)

*(Thuộc mảng: Whole-Body Control (WBC))*

## 🎯 Mục tiêu bài học

- Định nghĩa chính xác sim-to-real gap và giải thích vì sao nó luôn tồn tại dù mô phỏng có tinh vi tới đâu.
- Mô tả được cơ chế domain randomization: ngẫu nhiên hoá tham số nào, và vì sao "học trên một phân phối rộng" giúp robot thật hoạt động ổn định hơn.
- Tính tay được một ví dụ xác suất đơn giản minh hoạ vì sao huấn luyện trên một phân phối tham số rộng làm tăng khả năng "vật lý thật rơi vào phạm vi đã thấy".
- Giải thích được vì sao huấn luyện song song quy mô lớn (bài trước) là điều kiện cần cho domain randomization khả thi về mặt thời gian.
- Phân biệt được domain randomization thủ công (cố định phân phối) với Automatic Domain Randomization (ADR, cập nhật hiện đại).
- Nêu được một phương pháp thay thế/bổ sung domain randomization cổ điển: nhiễu loạn trong không gian mô-men khớp (joint torque space perturbation).

## 🧭 Vì sao cần học cái này? (bối cảnh)

Bài trước đã học cách huấn luyện policy nhanh (hàng nghìn instance song song). Nhưng tốc độ huấn luyện chỉ giải quyết được "học nhanh trong mô phỏng" — nó không tự động giải quyết vấn đề "policy học trong mô phỏng có hoạt động được trên robot thật không?". Đây chính là câu hỏi mà domain randomization trả lời, và nó **cần** hạ tầng huấn luyện song song quy mô lớn (bài trước) để khả thi — hai khái niệm này gắn bó chặt chẽ, đúng thứ tự xuất hiện trong `NOI-DUNG-CHI-TIET.md`. Học bài này để hiểu chính xác *cơ chế* domain randomization hoạt động (không chỉ nhớ tên gọi), vì đây là kỹ thuật xuất hiện lặp lại xuyên suốt gần như mọi paper RL cho robot thật (`04-imitation-learning-rl/`, SONIC, ASAP).

## 🧠 Trực giác

### Góc nhìn 1: Luyện tập chơi bóng bàn trên nhiều loại bàn/vợt/bóng khác nhau, không chỉ một bộ dụng cụ cố định

Nếu bạn chỉ luyện bóng bàn với đúng MỘT cây vợt, MỘT loại bóng, trên MỘT cái bàn cụ thể — kỹ năng bạn học được có thể "quá khớp" (overfit) với đúng bộ dụng cụ đó: đổi sang vợt khác (độ nảy khác), bóng khác (độ nặng khác), bạn có thể chơi tệ hẳn dù đã luyện tập rất nhiều. Nếu thay vào đó bạn cố tình luyện tập với **nhiều loại vợt, bóng, bàn khác nhau** trong lúc tập, kỹ năng bạn phát triển sẽ **bền vững hơn** với sự thay đổi — kể cả khi gặp một bộ dụng cụ hoàn toàn mới chưa từng dùng, khả năng cao nó vẫn "nằm trong phạm vi biến thể" mà bạn đã quen.

**Giới hạn của loại suy này:** một người chơi bóng bàn thực sự **hiểu** vì sao vợt nảy khác nhau (vật lý cao su, độ đàn hồi) và có thể **suy luận** để thích nghi nhanh với dụng cụ mới; một policy RL không "hiểu" gì cả — nó chỉ tối ưu hoá hành vi để hoạt động tốt trên **đúng phân phối tham số đã thấy khi huấn luyện**, không có khả năng suy luận trừu tượng để xử lý một tham số hoàn toàn nằm ngoài phân phối đó.

### Góc nhìn 2: Diễn tập chữa cháy với nhiều kịch bản giả lập khác nhau, không chỉ một đám cháy mẫu duy nhất

Lính cứu hoả không chỉ diễn tập với đúng MỘT kịch bản cháy cố định (một loại nhà, một loại vật liệu cháy, một hướng gió) — họ diễn tập với **nhiều biến thể**: nhà cao tầng khác nhà thấp tầng, gió mạnh khác gió yếu, vật liệu dễ cháy khác vật liệu khó cháy. Khi đám cháy thật xảy ra (chắc chắn khác với MỌI kịch bản diễn tập cụ thể, nhưng có thể "gần giống" một tổ hợp các biến thể đã diễn tập), đội cứu hoả phản ứng tốt hơn nhiều so với chỉ diễn tập một kịch bản duy nhất.

**Giới hạn của loại suy này:** lính cứu hoả điều chỉnh chiến thuật dựa trên **kinh nghiệm và huấn luyện đa dạng có chủ đích về loại tình huống** (không chỉ tham số số học); domain randomization robot chỉ ngẫu nhiên hoá các **tham số số học liên tục** (khối lượng, ma sát, độ trễ) trong một phạm vi đã định trước — không tạo ra được sự đa dạng về *loại* tình huống hoàn toàn mới mà con người có thể tưởng tượng ra khi thiết kế kịch bản diễn tập.

## 📐 Định nghĩa chính xác

**Sim-to-real gap:** một robot ảo huấn luyện hoàn hảo trong mô phỏng vẫn có thể thất bại khi đưa ra robot thật, vì mô phỏng không bao giờ khớp 100% với vật lý thật (khối lượng, ma sát, độ trễ động cơ, nhiễu cảm biến đều có sai lệch).

**Domain randomization** là kỹ thuật giảm khoảng cách đó bằng cách **cố tình ngẫu nhiên hoá các tham số mô phỏng** trong quá trình huấn luyện — ví dụ:

```text
- Khối lượng từng khớp/link:        m ~ Uniform(m₀×0.8, m₀×1.2)
- Hệ số ma sát sàn:                 μ ~ Uniform(0.4, 1.0)
- Độ trễ tín hiệu động cơ:          δ ~ Uniform(0ms, 20ms)
- Nhiễu cảm biến (quan sát):        n ~ Normal(0, σ²)
- Lực đẩy bất ngờ tác động robot:   F ~ Uniform(0N, 100N), thời điểm ngẫu nhiên
```

sao cho policy phải học cách **hoạt động tốt trên một PHÂN PHỐI RỘNG các điều kiện vật lý**, thay vì chỉ khớp với đúng một bộ tham số mô phỏng cố định. Khi đó, vật lý thật (dù không khớp chính xác với mô phỏng gốc) nhiều khả năng rơi vào "trong phạm vi" mà policy đã từng thấy khi huấn luyện, nên robot thật hoạt động ổn định hơn dù chưa từng được huấn luyện trực tiếp trên phần cứng thật.

**Điều kiện cần:** huấn luyện song song quy mô lớn (bài trước) là **điều kiện cần** để domain randomization khả thi về mặt thời gian — cần rất nhiều lượt thử với tham số khác nhau để policy học được sự bền vững đó (nếu chỉ huấn luyện tuần tự với 1 bộ tham số mỗi lần, việc "thấy đủ" các tổ hợp tham số ngẫu nhiên khác nhau sẽ mất thời gian không khả thi).

## ⚙️ Cơ chế hoạt động — từng bước

```text
┌────────────────────────────────────────────────────────────┐
│ Định nghĩa phân phối tham số cho từng đại lượng vật lý cần     │
│  ngẫu nhiên hoá (khối lượng, ma sát, độ trễ, nhiễu, lực đẩy...) │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ MỖI episode huấn luyện (hoặc mỗi instance trong N instance     │
│  song song — đã học ở bài trước):                              │
│  1. LẤY MẪU một bộ tham số cụ thể từ các phân phối đã định      │
│     (ví dụ instance #37 nhận m=1.1×m₀, μ=0.6, δ=12ms...)        │
│  2. Robot ảo với BỘ THAM SỐ NÀY tương tác với môi trường,        │
│     policy hành động, nhận reward                                │
│  3. Gradient cập nhật policy dùng dữ liệu từ TẤT CẢ instance,    │
│     mỗi instance có bộ tham số KHÁC NHAU                          │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ Sau nhiều vòng huấn luyện: policy học được hành vi ỔN ĐỊNH      │
│  trên TOÀN BỘ phân phối tham số đã thấy — không "quá khớp"      │
│  với một bộ tham số cụ thể nào                                   │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ Triển khai lên ROBOT THẬT: tham số vật lý thật (chưa biết       │
│  chính xác) — NẾU nằm trong phạm vi phân phối đã huấn luyện,    │
│  policy hoạt động tốt (chưa từng thấy CHÍNH XÁC bộ tham số này  │
│  nhưng đã thấy đủ biến thể tương tự)                             │
└──────────────────────────────────────────────────────────────┘
```

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ xác suất đơn giản hoá, số tự chọn — để thấy rõ TẠI SAO phân phối rộng hơn tăng khả năng "phủ" được vật lý thật, không phải mô phỏng chính xác cơ chế học của policy)*

Giả sử hệ số ma sát sàn thật (chưa biết chính xác khi thiết kế mô phỏng) có thể là bất kỳ giá trị nào trong khoảng **[0.3, 0.9]** với xác suất đều (uniform) — đây là "sự thật" mà ta không biết trước khi huấn luyện.

**Kịch bản A — không domain randomization, huấn luyện với đúng 1 giá trị cố định `μ_sim = 0.7`:**

```text
Xác suất giá trị thật CHÍNH XÁC bằng 0.7 (một điểm duy nhất trong khoảng liên tục) ≈ 0
→ policy gần như CHẮC CHẮN gặp giá trị ma sát thật KHÁC với giá trị đã huấn luyện
```

**Kịch bản B — domain randomization, huấn luyện với `μ_sim ~ Uniform(0.5, 0.9)`:**

```text
Xác suất giá trị ma sát thật (Uniform[0.3,0.9]) rơi vào đúng khoảng
  đã huấn luyện [0.5, 0.9]:
  = độ dài phần giao / độ dài toàn khoảng thật
  = (0.9 − 0.5) / (0.9 − 0.3)
  = 0.4 / 0.6
  ≈ 0.667  (66.7%)
```

**Kịch bản C — domain randomization với phạm vi rộng hơn, `μ_sim ~ Uniform(0.2, 1.0)` (bao trọn khoảng thật):**

```text
Xác suất giá trị ma sát thật rơi vào khoảng đã huấn luyện [0.2, 1.0]:
  = (0.9 − 0.3) / (0.9 − 0.3)  (vì [0.3,0.9] ⊂ [0.2,1.0] hoàn toàn)
  = 1.0  (100%)
```

**Ý nghĩa:** đây không phải xác suất chính xác "policy sẽ hoạt động tốt" (còn phụ thuộc nhiều yếu tố khác), nhưng minh hoạ đúng logic cốt lõi: **mở rộng phạm vi phân phối huấn luyện làm tăng khả năng giá trị thật (chưa biết) rơi vào vùng đã được policy "thấy" khi huấn luyện** — từ gần như 0% (một điểm cố định) lên 66.7% (phạm vi vừa phải) lên 100% (phạm vi bao trọn). *(Đánh đổi thực tế: phạm vi càng rộng, bài toán học của policy càng khó — huấn luyện policy hoạt động tốt trên `[0.2, 1.0]` khó hơn nhiều so với chỉ `[0.5, 0.9]`, nên không thể "mở rộng vô hạn" mà không trả giá về chất lượng/thời gian huấn luyện — đây chính là động lực cho Automatic Domain Randomization, xem mục Cập nhật hiện đại.)*

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Không domain randomization | Domain randomization thủ công (phạm vi cố định) | Automatic Domain Randomization (ADR, cập nhật hiện đại) |
|---|---|---|---|
| Phạm vi tham số | Một giá trị cố định | Phạm vi cố định, chọn thủ công bởi kỹ sư | Phạm vi tự động điều chỉnh theo hiệu năng policy |
| Rủi ro chọn phạm vi sai | N/A (không có phạm vi) | Cao — phạm vi quá hẹp không đủ bền vững, quá rộng làm bài toán học khó không cần thiết | Thấp hơn — tự động mở rộng khi policy đã "làm chủ" phạm vi hiện tại |
| Chi phí thiết kế | Thấp nhất | Cần chuyên gia tinh chỉnh phạm vi | Cần hạ tầng đánh giá tự động, nhưng ít cần tinh chỉnh thủ công |
| Yêu cầu huấn luyện song song quy mô lớn | Không bắt buộc | Bắt buộc (cần nhiều lượt thử tổ hợp tham số) | Bắt buộc, còn cần vòng lặp đánh giá liên tục |
| Kết quả sim-to-real thực tế | Kém, dễ thất bại trên robot thật | Tốt nếu phạm vi được chọn hợp lý | Báo cáo tốt hơn theo nghiên cứu 2025 (xem Cập nhật hiện đại) |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "phạm vi randomization càng rộng càng tốt, cứ mở rộng hết mức có thể".** Vì sao sai: như đã nêu ở ví dụ tính tay, mở rộng phạm vi làm **bài toán học của policy khó hơn** — nếu phạm vi quá rộng so với khả năng của kiến trúc mạng/thời gian huấn luyện, policy có thể học được một hành vi "an toàn nhưng tầm thường" (kém tối ưu trên MỌI giá trị) thay vì hành vi tốt trên phạm vi thực tế cần thiết. **Hiểu đúng:** phạm vi randomization cần đủ rộng để phủ được sự bất định thật, nhưng không rộng hơn mức cần thiết — đây chính là bài toán mà Automatic Domain Randomization (ADR) muốn tự động hoá thay vì chọn thủ công.
2. **Hiểu nhầm: "domain randomization giải quyết hoàn toàn sim-to-real gap, chỉ cần randomize đủ tham số là robot thật chắc chắn hoạt động tốt".** Vì sao sai: domain randomization chỉ tăng **khả năng** vật lý thật rơi vào phạm vi đã huấn luyện (đúng ví dụ xác suất ở trên) — nó không đảm bảo tuyệt đối, và không thể randomize những yếu tố mà kỹ sư **không lường trước được** (ví dụ một hiệu ứng vật lý thật hoàn toàn không có trong mô hình mô phỏng, không chỉ là "tham số sai" mà là "thiếu hẳn hiện tượng"). **Hiểu đúng:** domain randomization là một kỹ thuật **giảm thiểu** sim-to-real gap, không phải **loại bỏ hoàn toàn** — vẫn cần kiểm chứng thực tế (`07-policy-evaluation/`) trước khi tin tưởng hoàn toàn.

## 🏗️ Ví dụ minh hoạ trong dự án này

```text
Huấn luyện song song quy mô lớn (bài trước, N=4096+ instance)
        │
        ▼
MỖI instance nhận một bộ tham số ngẫu nhiên khác nhau:
  - Khối lượng G1 (± dao động so với thông số thiết kế)
  - Hệ số ma sát sàn khác nhau giữa các instance
  - Độ trễ động cơ, nhiễu cảm biến IMU/encoder khớp
  - Lực đẩy bất ngờ (mô phỏng va chạm/gió/địa hình bất thường)
        │
        ▼
Policy motion-tracking (04-imitation-learning-rl/) học cách
  theo dõi chuyển động tham chiếu (đã retarget từ 02) MÀ VẪN
  giữ thăng bằng dù tham số vật lý thay đổi
        │
        ▼
Triển khai lên Unitree G1 THẬT (08-real-robot-deployment/)
  — tham số vật lý thật (chưa biết chính xác 100% dù có thông số
  kỹ thuật) nhiều khả năng rơi vào phạm vi đã huấn luyện
        │
        ▼
Đây chính là điều kiện TIÊN QUYẾT để một policy huấn luyện
  THUẦN TRONG MÔ PHỎNG có cơ hội hoạt động được ngay trên robot
  thật — không có domain randomization, việc chuyển từ 05
  (mô phỏng) sang 08 (robot thật) gần như chắc chắn thất bại
```

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **Automatic Domain Randomization (ADR) — thay tinh chỉnh thủ công bằng thuật toán thích ứng.** ADR cung cấp "một phương án thay thế có nguyên tắc, dựa trên thuật toán, cho việc tinh chỉnh mô phỏng thủ công" — bằng cách **điều chỉnh dần curriculum trên cả tác vụ lẫn biến thể môi trường một cách thích ứng**, cải thiện nhất quán hiệu quả mẫu (sample efficiency), khả năng tổng quát hoá, và độ bền vững khi chuyển giao (transfer robustness) qua các bối cảnh RL, học có giám sát, và hybrid. Nguyên tắc hoạt động: bắt đầu với phạm vi hẹp, **tự động mở rộng phạm vi randomization** khi policy đã đạt hiệu năng tốt trên phạm vi hiện tại — giải quyết trực tiếp đánh đổi đã nêu ở mục Sai lầm thường gặp #1 (phạm vi rộng làm bài toán khó hơn, nhưng phạm vi hẹp không đủ bền vững).
2. **Vượt ra ngoài randomize tham số cố định: nhiễu loạn không gian mô-men khớp (joint torque space perturbation).** Một hướng thay thế 2025 cho sim-to-real của policy locomotion humanoid thêm **nhiễu loạn phụ thuộc trạng thái (state-dependent perturbations) vào mô-men khớp đầu vào** trong lúc huấn luyện, thay vì chỉ ngẫu nhiên hoá một tập hữu hạn tham số mô phỏng cố định — cách này **mô phỏng được phạm vi rộng hơn các khoảng cách với thực tế** (reality gap) so với randomize tham số cố định, giúp policy locomotion humanoid bền vững hơn trước các khoảng cách thực tế phức tạp/chưa từng thấy. Đây là một cách "tấn công" sim-to-real gap ở một tầng khác (nhiễu loạn trực tiếp lên tín hiệu điều khiển) thay vì chỉ ở tầng tham số mô phỏng vật lý.
3. **Kết quả zero-shot sim-to-real được báo cáo cải thiện đáng kể.** Các pipeline humanoid 2025 báo cáo tỷ lệ chuyển giao zero-shot sim-to-real (không cần fine-tune trên robot thật) đạt khoảng **84-93%** trên các tác vụ locomotion — một con số cụ thể cho thấy domain randomization (kết hợp các kỹ thuật bổ sung) đã trưởng thành từ một "kỹ thuật cần thiết nhưng chưa chắc chắn" (giai đoạn đầu) thành một thành phần hạ tầng đáng tin cậy ở mức độ cao trong pipeline production hiện đại.

## ❓ Câu hỏi tự kiểm tra

1. Vì sao sim-to-real gap luôn tồn tại dù mô phỏng có tinh vi tới đâu?
   <details><summary>Gợi ý đáp án</summary>Vì mô phỏng không bao giờ khớp 100% với vật lý thật — khối lượng, ma sát, độ trễ động cơ, nhiễu cảm biến của robot thật luôn có sai lệch so với thông số/mô hình dùng trong mô phỏng, dù đã cố gắng đo đạc/mô hình hoá chính xác tới đâu.</details>
2. Trong ví dụ tính tay, vì sao huấn luyện với phạm vi `[0.5, 0.9]` cho xác suất "phủ" giá trị thật là 66.7%, còn phạm vi `[0.2, 1.0]` cho 100%?
   <details><summary>Gợi ý đáp án</summary>Vì giá trị ma sát thật nằm trong `[0.3, 0.9]`; phần giao giữa `[0.3,0.9]` và `[0.5,0.9]` là `[0.5,0.9]` có độ dài 0.4/0.6≈66.7% so với toàn khoảng thật; còn `[0.2,1.0]` bao trọn hoàn toàn `[0.3,0.9]` nên xác suất phủ là 100%.</details>
3. Vì sao huấn luyện song song quy mô lớn (bài trước) là "điều kiện cần" cho domain randomization?
   <details><summary>Gợi ý đáp án</summary>Vì cần rất nhiều lượt thử với các tổ hợp tham số ngẫu nhiên khác nhau để policy học được sự bền vững trên toàn bộ phân phối — nếu chỉ huấn luyện tuần tự (1 bộ tham số mỗi lần), việc "thấy đủ" các tổ hợp tham số khác nhau sẽ tốn thời gian không khả thi.</details>
4. Automatic Domain Randomization (ADR) giải quyết đánh đổi nào mà domain randomization thủ công gặp phải?
   <details><summary>Gợi ý đáp án</summary>Đánh đổi giữa phạm vi hẹp (không đủ bền vững sim-to-real) và phạm vi rộng (bài toán học khó hơn, tốn thời gian huấn luyện hơn) — ADR tự động mở rộng phạm vi dần dần khi policy đã "làm chủ" phạm vi hiện tại, thay vì kỹ sư phải chọn thủ công một phạm vi cố định ngay từ đầu.</details>
5. Nhiễu loạn không gian mô-men khớp (joint torque space perturbation) khác gì so với domain randomization tham số cổ điển?
   <details><summary>Gợi ý đáp án</summary>Thay vì ngẫu nhiên hoá một tập hữu hạn tham số mô phỏng cố định (khối lượng, ma sát...), nó thêm nhiễu loạn phụ thuộc trạng thái trực tiếp vào tín hiệu mô-men khớp đầu vào trong lúc huấn luyện — mô phỏng được phạm vi rộng hơn các loại khoảng cách với thực tế mà việc chỉ randomize tham số cố định không bao quát hết.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** giả sử độ trễ động cơ thật nằm trong `[5ms, 25ms]` (uniform). Tính xác suất "phủ" nếu huấn luyện với phạm vi (a) `[10ms, 20ms]`, (b) `[0ms, 30ms]` — theo đúng công thức tỷ lệ phần giao/toàn khoảng đã dùng trong bài.
2. **Đọc paper/tài liệu thật:** tìm và đọc tóm tắt paper gốc về Automatic Domain Randomization (OpenAI, 2019, về bài toán Rubik's Cube robot tay) — mô tả bằng lời (3-5 câu) cơ chế cụ thể ADR dùng để quyết định "khi nào nên mở rộng phạm vi randomization thêm", và so sánh với cách hiểu ở mục Cập nhật hiện đại của bài này.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

Domain randomization giảm sim-to-real gap bằng cách cố tình ngẫu nhiên hoá các tham số mô phỏng (khối lượng, ma sát, độ trễ, nhiễu, lực đẩy) trong quá trình huấn luyện, buộc policy học hành vi bền vững trên một phân phối rộng thay vì quá khớp với một bộ tham số cố định — như ví dụ tính tay minh hoạ bằng xác suất, mở rộng phạm vi huấn luyện làm tăng khả năng vật lý thật (chưa biết chính xác) rơi vào vùng đã "thấy" khi huấn luyện, đánh đổi với việc bài toán học trở nên khó hơn. Kỹ thuật này cần huấn luyện song song quy mô lớn (bài trước) làm điều kiện khả thi về thời gian; hướng phát triển hiện đại gồm Automatic Domain Randomization (tự động mở rộng phạm vi theo hiệu năng, giải quyết đánh đổi hẹp/rộng) và nhiễu loạn không gian mô-men khớp (một cách "tấn công" sim-to-real gap ở tầng tín hiệu điều khiển thay vì chỉ tham số vật lý) — các pipeline humanoid 2025 hiện báo cáo tỷ lệ chuyển giao zero-shot sim-to-real khoảng 84-93%, cho thấy domain randomization đã trưởng thành thành một thành phần hạ tầng đáng tin cậy trong pipeline production.
