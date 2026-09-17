# Bài giảng: Metric cho VLA/loco-manipulation — seen/unseen task, robustness

*(Thuộc mảng: Policy Evaluation)*

## 🎯 Mục tiêu bài học

- Giải thích được vì sao một con số success rate duy nhất không đủ để đánh giá một policy nhận lệnh ngôn ngữ (language-conditioned).
- Phân biệt được "seen task" và "unseen task", và phân tách tiếp unseen task thành 3 mức độ khó tăng dần.
- Tính tay được success rate cho từng nhóm task từ một bảng kết quả thô, phát hiện dấu hiệu overfit.
- Định nghĩa được robustness testing (OOD evaluation) và cách tính "delta" so với điều kiện chuẩn.
- Đọc được một bảng kết quả kiểu GR00T N1 và biết chỗ nào đang bị che giấu nếu chỉ nhìn trung bình chung.
- Tự thiết kế được một bộ đánh giá seen/unseen + robustness tối thiểu cho policy VLA của riêng bạn.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Khi policy chỉ làm motion tracking thuần tuý (không có ngôn ngữ), MPJPE/success rate/jerk (2 bài giảng trước) đã đủ. Nhưng khi policy nhận lệnh ngôn ngữ — như GR00T/SONIC-VLA ở `06-vla-groot-sonic/` — câu hỏi thay đổi hoàn toàn: không chỉ "robot có làm đúng động tác không" mà còn "robot có **hiểu** lệnh mới, vật thể mới, hoàn cảnh mới hay không, hay chỉ đang học thuộc lòng (memorize) đúng những gì đã thấy trong tập huấn luyện?". Đây là câu hỏi trung tâm của toàn bộ literature VLA hiện nay, và một con số success rate trung bình duy nhất — cách báo cáo phổ biến của các paper robot học cũ hơn — sẽ che giấu hoàn toàn sự khác biệt giữa "học thuộc" và "hiểu thật".

## 🧠 Trực giác

### Góc nhìn 1: "Kỳ thi có đề quen và đề lạ"

Tưởng tượng một học sinh được cho ôn đúng 50 đề mẫu trước kỳ thi. Nếu đề thi thật trùng khớp gần như y hệt 50 đề đó (seen), học sinh có thể đạt điểm cao chỉ nhờ học thuộc, không cần hiểu bản chất. Chỉ khi đề thi có câu hỏi **dạng mới, chưa từng gặp** (unseen) mà học sinh vẫn làm tốt, ta mới tin học sinh thực sự hiểu kiến thức.

- **Đúng ở đâu:** nắm đúng ý tưởng cốt lõi seen vs unseen — success rate cao ở seen không chứng minh được gì về khả năng tổng quát hoá (generalization).
- **Giới hạn:** phép loại suy "đề thi" ngụ ý ranh giới seen/unseen rõ ràng, nhị phân — thực tế trong robot learning ranh giới này mờ hơn nhiều (một vật thể "mới" có thể chỉ khác màu sắc so với vật đã thấy, hay khác hẳn hình dạng) — đây là lý do cần phân tách tiếp thành nhiều mức độ (novel instruction / novel object / cả hai), loại suy đơn giản "quen/lạ" không đủ độ phân giải.

### Góc nhìn 2: "Bài kiểm tra sức bền của cầu — không phải bài kiểm tra tải trọng tối đa"

Robustness testing giống như kiểm tra một cây cầu không phải bằng cách hỏi "cầu chịu được tải tối đa bao nhiêu trong điều kiện lý tưởng" mà bằng cách cố tình đưa vào điều kiện bất lợi (gió mạnh, nhiệt độ khắc nghiệt, rung động bất thường) rồi đo mức sụt giảm khả năng chịu tải so với điều kiện chuẩn.

- **Đúng ở đâu:** làm rõ robustness không đo "khả năng cao nhất" mà đo "độ bền khi bị nhiễu loạn" — nghĩa là **delta** (chênh lệch) quan trọng hơn con số tuyệt đối.
- **Giới hạn:** loại suy cây cầu ngụ ý các điều kiện nhiễu loạn là ngẫu nhiên/tự nhiên (thời tiết) — trong khi robustness test cho VLA thường là **nhiễu loạn được thiết kế có chủ đích** (đổi ánh sáng, đổi vị trí vật thể) để lộ ra điểm yếu cụ thể, không phải mô phỏng ngẫu nhiên môi trường thật.

## 📐 Định nghĩa chính xác

### Phân nhóm task theo GR00T N1 (arXiv:2503.14734)

| Nhóm | Định nghĩa | Đo được gì |
|---|---|---|
| **Seen task** | Task (đối tượng + hành động + vị trí) xuất hiện trong tập huấn luyện | Khả năng học thuộc/khớp phân phối huấn luyện |
| **Unseen task** | Kết hợp đối tượng/hành động/ngôn ngữ **chưa từng xuất hiện** khi huấn luyện | Generalization thực sự |

Trong "unseen", phân tách tiếp theo mức độ khác biệt (độ khó tăng dần):
1. **Novel instruction, same object/scene**: đổi cách diễn đạt câu lệnh, giữ nguyên vật thể/bối cảnh.
2. **Novel object**: vật thể mới nhưng cùng loại hành động đã học.
3. **Novel object + novel instruction**: khó nhất — generalization dạng "compound" (kết hợp cả 2 trục lạ cùng lúc).

Quy tắc báo cáo bắt buộc: success rate **riêng cho từng nhóm**, không gộp trung bình — một trung bình duy nhất che giấu chênh lệch giữa seen và unseen.

### Robustness testing (OOD evaluation)

Định nghĩa: cố tình đưa policy vào điều kiện lệch khỏi phân phối huấn luyện (out-of-distribution) để đo độ bền, không phải đo khả năng cao nhất. Các trục nhiễu loạn thường dùng:

- **Ánh sáng**: đổi cường độ/màu/hướng sáng so với lúc thu dữ liệu huấn luyện.
- **Vị trí/hướng vật thể**: đặt vật thể ở vị trí/góc xoay không có trong tập huấn luyện, hoặc thêm vật cản.
- **Nhiễu môi trường**: thêm vật thể gây rối (distractor), đổi nền/texture, đổi camera viewpoint.
- **Nhiễu vật lý**: thay đổi ma sát, tải trọng, nhiễu cảm biến.

Công thức báo cáo chuẩn:

```
Δ_robustness = SuccessRate(điều kiện chuẩn) − SuccessRate(dưới nhiễu loạn)
```

Δ nhỏ = robust (bền), Δ lớn = dễ vỡ (fragile) dưới nhiễu loạn đó. Δ phải được báo cáo **riêng cho từng trục nhiễu loạn** — gộp chung "robustness score" duy nhất che giấu việc policy có thể bền với ánh sáng nhưng rất yếu với vị trí vật thể mới, ví dụ.

## ⚙️ Cơ chế hoạt động — từng bước

```
┌─────────────── Thiết kế bộ đánh giá VLA hoàn chỉnh ────────────────┐
│                                                                      │
│  BƯỚC 1 — Phân loại tập test TRƯỚC khi chạy:                        │
│    - Liệt kê mọi (object, action, instruction) đã dùng lúc train    │
│    - Với mỗi test case, gán nhãn: seen / unseen-instr / unseen-obj /│
│      unseen-cả-hai                                                  │
│                                                                      │
│  BƯỚC 2 — Chạy policy trên từng nhóm, N lần thử mỗi test case:      │
│    seen: (n_success_seen / n_total_seen)                            │
│    unseen-instr: (n_success_ui / n_total_ui)                        │
│    unseen-obj: (n_success_uo / n_total_uo)                          │
│    unseen-cả-hai: (n_success_both / n_total_both)                   │
│    → 4 con số success rate RIÊNG BIỆT, không gộp                    │
│                                                                      │
│  BƯỚC 3 — Robustness test (OOD), lặp lại trên tập seen (hoặc một    │
│  tập con cố định) NHƯNG thay đổi 1 trục nhiễu loạn mỗi lần:         │
│    baseline_success = success rate điều kiện chuẩn                  │
│    lighting_success = success rate khi đổi ánh sáng                 │
│    position_success  = success rate khi đổi vị trí vật thể          │
│    → Δ_lighting = baseline − lighting_success                       │
│    → Δ_position = baseline − position_success                       │
│                                                                      │
│  BƯỚC 4 — Báo cáo dạng bảng đầy đủ (không rút gọn thành 1 số):      │
│    | Nhóm | Success rate |                                          │
│    | Seen | X% |                                                    │
│    | Unseen-instr | Y% |                                            │
│    | Unseen-obj | Z% |                                              │
│    | Unseen-cả-hai | W% |                                           │
│    | Robustness Δ (lighting) | Δ1 |                                 │
│    | Robustness Δ (position) | Δ2 |                                 │
└──────────────────────────────────────────────────────────────────┘
```

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số liệu tự chọn để dễ hình dung.)*

Giả sử bạn đánh giá 1 policy VLA trên 4 nhóm, mỗi nhóm 20 lần thử:

| Nhóm | Số lần thành công | Tổng số lần thử |
|---|---|---|
| Seen | 18 | 20 |
| Unseen — novel instruction | 14 | 20 |
| Unseen — novel object | 9 | 20 |
| Unseen — cả hai | 4 | 20 |

**Tính success rate từng nhóm:**

```
Seen:              18/20 = 90.0%
Unseen-instr:       14/20 = 70.0%
Unseen-obj:          9/20 = 45.0%
Unseen-cả-hai:       4/20 = 20.0%
```

**Nếu gộp trung bình (cách làm SAI — chỉ để minh hoạ tác hại):**

```
Trung bình gộp (không trọng số nhóm) = (90+70+45+20)/4 = 225/4 = 56.25%
```

Một con số "56.25%" duy nhất khiến người đọc không biết được rằng policy **rất tốt trên seen (90%) nhưng gần như thất bại trên trường hợp khó nhất (20%)** — đây chính xác là dấu hiệu overfit vào phân phối huấn luyện mà báo cáo mục 2.1 của `NOI-DUNG-CHI-TIET.md` cảnh báo.

**Bây giờ tính robustness:** giả sử baseline (điều kiện ánh sáng chuẩn, trên tập seen) = 90%. Khi đổi ánh sáng (giữ nguyên object/instruction), success rate còn 72%; khi đổi vị trí vật thể, còn 55%.

```
Δ_lighting = 90% − 72% = 18 điểm phần trăm
Δ_position = 90% − 55% = 35 điểm phần trăm
```

Kết luận: policy này **nhạy với thay đổi vị trí vật thể hơn nhiều so với thay đổi ánh sáng** (Δ_position gấp gần 2 lần Δ_lighting) — đây là thông tin chẩn đoán cụ thể, hữu ích hơn hẳn một "robustness score" gộp chung.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Success rate theo seen/unseen | Robustness Δ (OOD) | Success rate tổng (1 con số) |
|---|---|---|---|
| Mục đích chính | Đo generalization theo trục ngữ nghĩa (object/action/instruction mới) | Đo độ bền dưới nhiễu loạn vật lý/cảm quan | Đo hiệu năng trung bình chung |
| Cách phát hiện overfit | So sánh trực tiếp seen vs unseen — gap lớn = overfit | So sánh baseline vs OOD — Δ lớn = fragile | Không phát hiện được — bị che giấu hoàn toàn |
| Rủi ro khi dùng một mình | Không nói gì về độ bền vật lý/cảm quan | Không nói gì về khả năng hiểu ngôn ngữ/khái niệm mới | Che giấu mọi phân tích nguyên nhân |
| Khi nào ưu tiên | Đánh giá "policy có hiểu ngôn ngữ thật không" | Chuẩn bị deploy trong môi trường thực tế biến động | Chỉ dùng làm con số tóm tắt phụ, không thay thế bảng chi tiết |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: chỉ cần báo cáo 1 con số "success rate trung bình" là đủ thể hiện chất lượng policy VLA.**
   Vì sao sai: như ví dụ tính tay ở trên, 56.25% trung bình gộp có thể đến từ (90%, 70%, 45%, 20%) — một policy cực kỳ lệch giữa dễ và khó — hoặc từ một kịch bản khác (56%, 57%, 56%, 55%) — một policy đồng đều nhưng tầm thường ở mọi mức. Hai policy này có chất lượng rất khác nhau nhưng cùng một trung bình.
   Hiểu đúng: luôn báo cáo bảng đầy đủ theo từng nhóm (seen, 3 mức unseen), không rút gọn thành 1 con số duy nhất trong phần kết quả chính — nếu cần 1 con số tóm tắt cho tiêu đề/abstract, phải kèm bảng chi tiết ngay bên cạnh.

2. **Hiểu nhầm: "unseen object" và "novel instruction" là cùng một mức độ khó, có thể gộp chung thành "unseen".**
   Vì sao sai: đổi cách diễn đạt câu lệnh (giữ nguyên vật thể/hành động) thường dễ hơn nhiều so với đưa vào một vật thể hoàn toàn mới, vì mô hình ngôn ngữ (language backbone) của VLA thường đã robust với paraphrase từ pretraining, trong khi nhận diện + thao tác với hình dạng vật thể mới đòi hỏi generalization ở tầng thị giác-hành động sâu hơn. Ví dụ tính tay ở trên cho thấy chênh lệch rõ: 70% (unseen-instr) vs 45% (unseen-obj) — gộp chung sẽ làm mất thông tin chẩn đoán quan trọng này.
   Hiểu đúng: giữ nguyên 3 mức phân tách (novel instruction / novel object / cả hai) khi thiết kế test set và khi báo cáo, đúng như cấu trúc đã nêu ở mục Định nghĩa.

## 🏗️ Ví dụ minh hoạ trong dự án này

Khi đánh giá một policy VLA tích hợp SONIC (`06-vla-groot-sonic/`) cho task loco-manipulation (ví dụ "nhặt vật X đặt vào giỏ Y" khi robot đang di chuyển), bộ test tối thiểu nên gồm:

- **Seen**: đúng vật thể + đúng câu lệnh + đúng vị trí đã dùng lúc thu dữ liệu huấn luyện.
- **Unseen-instr**: đổi cách nói ("nhặt cái ly đỏ" → "lấy chiếc cốc màu đỏ kia"), giữ nguyên vật thể/vị trí.
- **Unseen-obj**: vật thể mới cùng loại hành động cầm-nắm (ví dụ đổi ly thành chai nước, hình dạng khác).
- **Robustness OOD**: chạy lại đúng bộ "seen" nhưng đổi cường độ ánh sáng phòng thí nghiệm, và một lần khác đổi vị trí đặt vật thể ra ngoài vùng đã từng xuất hiện trong dữ liệu huấn luyện.

Việc phân nhóm này giúp bạn biết chính xác: nếu policy tụt mạnh ở unseen-obj nhưng vẫn ổn ở unseen-instr, vấn đề nằm ở phần perception/vision-object-grounding, không phải ở phần hiểu ngôn ngữ — hướng debug hoàn toàn khác nhau.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **Cộng đồng VLA 2025 đã phát triển các benchmark chuyên biệt để đo robustness một cách hệ thống hơn**, thay vì chỉ thêm vài test case OOD thủ công. Theo tra cứu (WebSearch tháng 9/2026): **LIBERO-Plus** (arXiv:2510.13626) giới thiệu các biến thể nhiễu loạn có hệ thống trên các episode đánh giá, gồm 3 nhóm trục: visual (ánh sáng, nền, góc camera), physical (vị trí vật thể, trạng thái robot), và semantic (viết lại câu lệnh — paraphrase) — đúng khớp với phân loại "trục nhiễu loạn" đã nêu ở mục Định nghĩa, cho thấy literature đang chuẩn hoá chính xác các trục này thành benchmark tái sử dụng được, thay vì mỗi paper tự chế một bộ test riêng.
2. **AGNOSTOS** (theo kết quả tra cứu về "cross-task generalization" 2025) là một benchmark riêng cho **cross-task zero-shot generalization**, gồm 23 task thao tác chưa từng thấy, chia theo "hai mức độ khó generalization" — cùng tinh thần phân tầng độ khó unseen đã mô tả trong bài này, nhưng áp dụng ở cấp độ toàn bộ *task* (không chỉ object/instruction) thay vì chỉ đổi từng biến số riêng lẻ. Phát hiện chung của các paper này: "các mô hình VLA hiện tại, dù xuất sắc trên benchmark chuẩn (như LIBERO gốc), vẫn thiếu robustness/generalization thực sự khi đối mặt các task chưa từng thấy" — xác nhận đúng lo ngại cốt lõi mà phần seen/unseen của bài giảng này đặt ra.
3. **LIBERO-Para** (một benchmark 2025 khác tìm được qua tra cứu) tập trung riêng vào "paraphrase robustness" — chỉ đổi cách diễn đạt câu lệnh, giữ nguyên mọi thứ khác — cho thấy cộng đồng đang tách metric "hiểu ngôn ngữ" (paraphrase) ra khỏi metric "generalization vật lý" (object/scene mới) thành 2 trục đo lường độc lập, thay vì đo gộp như trước — đúng xu hướng phân tách càng lúc càng chi tiết hơn mà mục Định nghĩa ở trên đã trình bày cho dự án này.

## ❓ Câu hỏi tự kiểm tra

1. Vì sao "seen task success rate cao, unseen task success rate thấp" là dấu hiệu overfit, không phải dấu hiệu "policy giỏi nhưng benchmark khó"?
<details><summary>Gợi ý đáp án</summary>Vì unseen task được định nghĩa là những gì đã KHÔNG xuất hiện trong tập huấn luyện — nếu policy thực sự học được cơ chế chung (hiểu ngôn ngữ, nhận diện đối tượng tổng quát) thay vì học thuộc phân phối huấn luyện, nó phải generalize được ít nhiều sang unseen. Gap lớn giữa 2 nhóm cho thấy policy phụ thuộc quá nhiều vào việc "đã từng thấy" thay vì hiểu bản chất.</details>

2. Cho success rate 4 nhóm là (95%, 90%, 88%, 85%) — so với ví dụ (90%, 70%, 45%, 20%) trong bài — policy nào "generalize" tốt hơn dù có thể có seen thấp hơn?
<details><summary>Gợi ý đáp án</summary>Policy với (95,90,88,85) generalize tốt hơn nhiều dù seen (95%) chỉ nhỉnh hơn chút so với (90%) của policy kia — vì gap giữa seen và unseen-khó-nhất chỉ 10 điểm % (95→85) so với 70 điểm % (90→20) của policy còn lại.</details>

3. Δ_robustness được tính như thế nào, và tại sao phải báo cáo riêng cho từng trục nhiễu loạn thay vì gộp thành 1 "robustness score"?
<details><summary>Gợi ý đáp án</summary>Δ = success rate điều kiện chuẩn − success rate dưới nhiễu loạn. Phải tách riêng vì các trục nhiễu loạn khác nhau kiểm tra các thành phần khác nhau của policy (ví dụ ánh sáng kiểm tra robustness thị giác, vị trí vật thể kiểm tra robustness không gian) — gộp chung che giấu policy có thể rất yếu ở 1 trục cụ thể trong khi mạnh ở các trục khác.</details>

4. Vì sao "novel instruction, same object" thường dễ hơn "novel object"?
<details><summary>Gợi ý đáp án</summary>Vì language backbone của VLA thường đã được pretrain rộng và có khả năng xử lý paraphrase tốt sẵn (hiểu nhiều cách diễn đạt cùng ý), trong khi nhận diện và thao tác chính xác với một hình dạng/kích thước vật thể hoàn toàn mới đòi hỏi generalization ở tầng thị giác-hành động — tầng này thường khó tổng quát hoá hơn tầng ngôn ngữ thuần tuý.</details>

5. Một báo cáo chỉ ghi "robustness score = 0.82" mà không nói trục nhiễu loạn nào. Bạn cần hỏi thêm gì?
<details><summary>Gợi ý đáp án</summary>Hỏi: (a) 0.82 là tỉ lệ gì — success rate dưới nhiễu hay tỉ lệ 1 trừ Δ? (b) đo trên trục nhiễu loạn nào (ánh sáng/vị trí/nền/vật cản)? (c) so với baseline bao nhiêu %, để tính ra Δ thực sự? (d) có báo cáo riêng từng trục hay đã gộp trung bình nhiều trục?</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** giả sử một policy khác có bảng kết quả (seen=20/20, unseen-instr=19/20, unseen-obj=6/20, unseen-cả-hai=3/20). Tính 4 success rate riêng biệt, tính trung bình gộp không trọng số, và so sánh với ví dụ trong bài (90/70/45/20, trung bình 56.25%) — hai policy này có "trung bình gộp" gần nhau không? Chúng có thực sự "giỏi như nhau" không? Giải thích bằng số liệu cụ thể.
2. **Đọc paper thật:** mở phần Experiments của GR00T N1 (arXiv:2503.14734) hoặc một trong các benchmark vừa nêu ở mục Cập nhật hiện đại (LIBERO-Plus, AGNOSTOS), tìm đúng bảng phân nhóm seen/unseen (hoặc tương đương), ghi lại chính xác cách họ định nghĩa từng nhóm và so sánh cách phân nhóm đó với 4 nhóm đã trình bày trong bài giảng này — có điểm gì giống/khác?

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

Khi policy nhận lệnh ngôn ngữ (VLA), một con số success rate trung bình duy nhất che giấu sự khác biệt sống còn giữa "học thuộc phân phối huấn luyện" và "hiểu và tổng quát hoá thật sự" — vì vậy success rate phải được báo cáo tách riêng theo 4 nhóm: seen, unseen-novel-instruction, unseen-novel-object, và unseen-cả-hai (độ khó tăng dần), không gộp trung bình. Song song đó, robustness testing (đưa policy vào điều kiện out-of-distribution có chủ đích như đổi ánh sáng, vị trí vật thể, môi trường) đo độ bền bằng con số delta (Δ = success rate chuẩn − success rate dưới nhiễu) tính riêng cho từng trục nhiễu loạn, vì mỗi trục kiểm tra một thành phần khác nhau của policy (thị giác, không gian, ngôn ngữ). Bỏ qua 2 nguyên tắc phân tách này — dùng 1 con số tổng thay vì bảng chi tiết — là sai lầm phổ biến nhất khi báo cáo kết quả VLA, và chính là điều literature 2025 (LIBERO-Plus, LIBERO-Para, AGNOSTOS) đang chuẩn hoá thành các benchmark tái sử dụng được để cả cộng đồng đo đúng cùng một cách.
