# Bài giảng: Sim-to-real gap — case study SONIC

*(Thuộc mảng: Policy Evaluation)*

## 🎯 Mục tiêu bài học

- Trình bày lại chính xác thí nghiệm real-world evaluation của SONIC (arXiv:2511.07820): điều kiện, số liệu, và những gì paper đã xác nhận rõ vs. những gì cần xác minh thêm.
- Giải thích được vì sao "100% success rate trên 50 trajectory, zero-shot" là một tuyên bố mạnh, không phải một con số dễ đạt.
- Tính tay được sim-to-real gap từ bộ số liệu MPJPE-L định lượng của SONIC (sim ≈ 22.3mm vs real ≈ 25.7mm, tách theo bộ phận cơ thể).
- Phân biệt được 2 bộ số liệu khác nhau trong cùng paper (50 trajectory nhị phân vs. MPJPE-L liên tục trên tập rộng hơn) và tại sao không nên gộp chúng làm một.
- Áp dụng được cách đọc phản biện này cho bất kỳ case study sim-to-real nào khác bạn gặp trong tương lai.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Bài giảng `BAI-GIANG-sim-to-real-gap-cach-do.md` đã trình bày khung lý thuyết chung (công thức gap, 3 dạng báo cáo). Bài này áp dụng khung đó vào một case study cụ thể, đã được xác minh trực tiếp từ nội dung paper — SONIC, hệ thống trung tâm của mảng VLA trong dự án này (`06-vla-groot-sonic/`). Đây là ví dụ thực hành tốt nhất để luyện kỹ năng "đọc phản biện" một tuyên bố sim-to-real mạnh, thay vì chỉ nhớ con số "100%" mà không hiểu thí nghiệm đứng sau nó.

## 🧠 Trực giác

### Góc nhìn 1: "Một vận động viên thi đấu không có buổi tập làm quen sân"

Zero-shot deployment giống như một vận động viên được đưa thẳng vào thi đấu chính thức trên một sân vận động họ chưa từng bước chân vào, không có buổi tập làm quen nào — nếu họ vẫn thi đấu tốt, đó là bằng chứng mạnh về năng lực thích nghi tổng quát, không phải may mắn thuộc lòng sân đấu.

- **Đúng ở đâu:** nắm đúng lý do "không fine-tune" (zero-shot) làm tuyên bố mạnh hơn hẳn so với "được luyện tập trước".
- **Giới hạn:** loại suy vận động viên ngụ ý một "trận đấu" duy nhất — trong khi SONIC thử trên 50 trajectory khác nhau (đa dạng thể loại: dance, jump, loco-manipulation), tương đương thi đấu 50 trận khác nhau liên tiếp mà không thua trận nào — độ khó của tuyên bố này lớn hơn nhiều so với 1 trận đấu đơn lẻ mà loại suy gốc gợi ý.

### Góc nhìn 2: "Điểm thi tốt nghiệp (đỗ/trượt) và bảng điểm chi tiết từng môn"

"100% success rate trên 50 trajectory" giống như "đỗ tốt nghiệp 50/50 môn" — một tiêu chí nhị phân nghiêm ngặt. Nhưng con số MPJPE-L (22.3mm sim vs 25.7mm real) giống như bảng điểm chi tiết từng môn (điểm số liên tục) — một học sinh có thể "đỗ" mọi môn nhưng vẫn có sự khác biệt điểm số nhỏ giữa các lần thi thử (sim) và thi thật (real).

- **Đúng ở đâu:** làm rõ đây là **2 phép đo khác nhau về bản chất** (nhị phân vs liên tục) trong cùng 1 paper, không nên gộp làm 1 con số.
- **Giới hạn:** loại suy "cùng 1 học sinh, 2 kỳ thi" ngụ ý 2 bộ số liệu chắc chắn đo trên cùng một tập bài thi — nhưng bản thân nội dung paper SONIC (như trình bày ở mục Định nghĩa) chưa xác nhận rõ liệu "50 trajectory, 100%" và "tập rộng hơn cho MPJPE-L" có phải là cùng một thí nghiệm hay hai thí nghiệm riêng biệt — đây là điểm "cần xác minh thêm", không nên mặc định chúng khớp nhau.

## 📐 Định nghĩa chính xác

Toàn bộ số liệu dưới đây đã được xác minh trực tiếp qua nội dung phần Real-World Evaluation của paper SONIC (arXiv:2511.07820):

> *"We assess the real-world performance of SONIC by deploying it on 50 diverse motion trajectories, including dance, jumps, and loco-manipulation tasks... it succeeds on all sequences without a single failure (100% success rate)."*

**Điều kiện thí nghiệm (đã xác minh):**

| Thành phần | Giá trị |
|---|---|
| Robot thật | Unitree G1 |
| Số trajectory | 50, đa dạng thể loại (dance, jump, loco-manipulation) |
| Chế độ | Zero-shot — không fine-tune trên dữ liệu thật |
| Kết quả | 100% success rate (0 thất bại trên 50 trajectory) |
| Số lần thử/trajectory | Không nêu rõ số chính xác — **cần xác minh thêm** (gợi ý từ cách trình bày: có thể 1 lần/trajectory — single trial) |
| Baseline so sánh trong điều kiện thật | Theo nội dung đã xác minh, phần so sánh với Any2Track/BeyondMimic/GMT diễn ra **trong simulation**, không phải trực tiếp trên robot thật cùng 50 trajectory này — **cần xác minh thêm** nếu có baseline nào khác cũng zero-shot thật |

**Số liệu định lượng riêng biệt (MPJPE-L, xem bài giảng `BAI-GIANG-mpjpe-va-cac-bien-the.md` để hiểu định nghĩa MPJPE-L):**

| Bộ phận | Sim (mm) | Real (mm) | Gap (điểm mm) |
|---|---|---|---|
| Toàn thân (broader tập motion) | ≈ 22.3 | ≈ 25.7 | ≈ 3.4 |
| Upper body (thân trên) | 21.8 | 22.2 | 0.4 |
| Feet (bàn chân) | 29.0 | 53.7 | 24.7 |

> **Cần xác minh thêm:** liệu bộ số liệu MPJPE-L này (124 motion sequences, 99.2% = 123/124 theo một cách đọc khác của cùng phần này) có phải cùng một thí nghiệm với tập "50 trajectory, 100%" hay là một tập đánh giá riêng biệt — hai bộ số liệu xuất hiện ở các phần khác nhau của paper, đo hai thứ khác nhau về bản chất (nhị phân thành/bại theo trajectory vs. sai số liên tục theo từng phần cơ thể). Trước khi trích dẫn học thuật, nên đối chiếu lại bản PDF/HTML gốc.

## ⚙️ Cơ chế hoạt động — từng bước

```
┌────────── Cách đọc phản biện 1 tuyên bố sim-to-real mạnh ──────────┐
│                                                                       │
│  BƯỚC 1 — Xác định dạng transfer: zero-shot hay fine-tuned?          │
│    SONIC → zero-shot (không fine-tune trên dữ liệu thật)             │
│                                                                       │
│  BƯỚC 2 — Xác định metric: nhị phân (success/fail) hay liên tục      │
│    (MPJPE)? Có mấy bộ số liệu khác nhau trong cùng paper?            │
│    SONIC → CÓ 2 bộ: (a) 50 trajectory, nhị phân, 100%                │
│                      (b) MPJPE-L, liên tục, theo bộ phận cơ thể       │
│                                                                       │
│  BƯỚC 3 — Với mỗi bộ số liệu, xác định độ đa dạng của test set:      │
│    SONIC → 50 trajectory đa dạng thể loại (dance/jump/loco-manip),   │
│    KHÔNG chỉ đi bộ thẳng đơn giản (dance/jump là chuyển động động    │
│    lực học cao — dễ lộ sim-to-real gap hơn đi bộ thường)             │
│                                                                       │
│  BƯỚC 4 — Kiểm tra baseline so sánh có cùng điều kiện thật không     │
│    SONIC → so sánh Any2Track/BeyondMimic/GMT diễn ra TRONG SIM,      │
│    không phải cùng 50 trajectory thật này → không so sánh trực tiếp  │
│    được "100%" với baseline nào khác trên CÙNG điều kiện thật        │
│                                                                       │
│  BƯỚC 5 — Liệt kê rõ các điểm "cần xác minh thêm" thay vì mặc định   │
│    → số lần thử/trajectory, mối liên hệ giữa 2 bộ số liệu,           │
│      điều kiện môi trường thử (sàn cứng/mềm), định nghĩa "thất bại"  │
│                                                                       │
│  BƯỚC 6 — Kết luận có điều kiện: "100% zero-shot trên 50 trajectory  │
│    đa dạng là một tuyên bố mạnh VỀ..." (nêu rõ về cái gì, dựa trên   │
│    bước 1-4), không phải "SONIC hoàn hảo, gap = 0" một cách vô điều  │
│    kiện                                                              │
└──────────────────────────────────────────────────────────────────┘
```

## 🧮 Ví dụ tính tay / walkthrough cụ thể

Dùng đúng số liệu MPJPE-L đã xác minh ở trên, áp dụng công thức gap từ bài giảng `BAI-GIANG-sim-to-real-gap-cach-do.md`: `G = ψ_sim − ψ_real` (ở đây ψ là MPJPE-L, đơn vị mm — lưu ý: với MPJPE, số càng nhỏ càng tốt, nên "gap dương" ở đây nghĩa là sai số tăng lên khi chuyển sang thật, ngược chiều so với success rate nơi số lớn hơn là tốt hơn — cẩn thận dấu khi diễn giải):

**Toàn thân:**
```
G_toàn_thân = 25.7 − 22.3 = 3.4 mm
```
Gap tương đối nhỏ so với độ lớn tuyệt đối (~15% của giá trị sim: 3.4/22.3 ≈ 15.2%).

**Upper body (thân trên):**
```
G_upper = 22.2 − 21.8 = 0.4 mm
```
Gap gần như không đáng kể (0.4/21.8 ≈ 1.8%) — thân trên gần như "chuyển giao hoàn hảo" từ sim sang thật.

**Feet (bàn chân):**
```
G_feet = 53.7 − 29.0 = 24.7 mm
```
Gap rất lớn so với giá trị sim (24.7/29.0 ≈ 85.2%) — sai số bàn chân trên robot thật gần **gấp đôi** so với trong sim.

**So sánh 3 con số gap:**

| Bộ phận | Gap tuyệt đối (mm) | Gap tương đối (%) |
|---|---|---|
| Upper body | 0.4 | 1.8% |
| Toàn thân | 3.4 | 15.2% |
| Feet | 24.7 | 85.2% |

**Kết luận rút ra từ phép tính này (không phải suy đoán, mà là hệ quả trực tiếp của số liệu đã xác minh):** gap sim-to-real của SONIC **không đồng đều trên cơ thể** — tập trung mạnh nhất ở bàn chân (nơi tiếp xúc trực tiếp với mặt đất, chịu ảnh hưởng lớn nhất từ sai lệch mô hình ma sát/tiếp xúc giữa sim và thật), trong khi thân trên gần như không có gap đáng kể. Đây chính xác là lý do tại sao báo cáo MPJPE tách theo từng bộ phận cơ thể (thay vì chỉ 1 con số toàn thân) hữu ích hơn — nếu SONIC chỉ báo "gap toàn thân = 3.4mm", con số này che giấu hoàn toàn vấn đề nghiêm trọng ở bàn chân.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | SONIC (100% zero-shot, 50 trajectory) | Fine-tuned transfer điển hình (mô tả chung, không phải SONIC) | Domain-randomization-coverage report |
|---|---|---|---|
| Cần dữ liệu thật để "sửa" trước khi đo? | Không | Có | Không đo trực tiếp |
| Độ đa dạng test set | Cao (dance/jump/loco-manip — nhiều loại động lực học) | Thường thấp hơn nếu chỉ test task đã fine-tune | Không áp dụng (không phải test set) |
| Mức độ thuyết phục về domain randomization | Cao nhất trong 3 loại | Thấp | Trung bình (gián tiếp) |
| Rủi ro đọc sai nếu thiếu ngữ cảnh | Tưởng "0 thất bại" nghĩa là hoàn hảo tuyệt đối trên MỌI mặt (bỏ qua gap MPJPE-L theo bộ phận) | — | — |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "SONIC đạt 100% success rate zero-shot nghĩa là sim-to-real gap của SONIC bằng 0 hoàn toàn".**
   Vì sao sai: 100% success rate chỉ đo tiêu chí nhị phân "trajectory có hoàn thành hay không" — một tiêu chí tương đối "khoan dung" (một robot có thể hoàn thành trajectory dù dáng đi hơi lệch chút). Bộ số liệu MPJPE-L riêng biệt (đã tính ở trên) cho thấy vẫn còn gap thực sự, đặc biệt ở bàn chân (gap ~85% tương đối) — hai bộ số liệu đo hai khía cạnh khác nhau, không thể dùng "100% success" để suy ra "0 gap ở mọi mức độ chi tiết".
   Hiểu đúng: đọc cả 2 bộ số liệu song song — success rate nhị phân trả lời "có hoàn thành nhiệm vụ không", MPJPE-L liên tục trả lời "chính xác tới đâu, ở đâu còn yếu" — chúng bổ sung chứ không thay thế nhau.

2. **Hiểu nhầm: coi "so sánh với Any2Track/BeyondMimic/GMT" là bằng chứng SONIC vượt trội hơn các baseline này trên robot thật.**
   Vì sao sai: theo nội dung đã xác minh, phần so sánh với các baseline này diễn ra **trong simulation**, không phải trên cùng 50 trajectory thật. Do đó "SONIC đạt 100% trên robot thật" và "SONIC tốt hơn baseline X" là hai tuyên bố tách biệt — không thể ghép chúng lại thành "SONIC tốt hơn baseline X trên robot thật" nếu baseline X chưa từng được thử zero-shot thật cùng điều kiện.
   Hiểu đúng: chỉ so sánh trực tiếp các hệ thống khi chúng được đánh giá trên **cùng môi trường** (cùng là sim, hoặc cùng là robot thật) — đây cũng chính là nguyên tắc fair comparison ở bài giảng riêng `BAI-GIANG-fair-comparison.md`.

## 🏗️ Ví dụ minh hoạ trong dự án này

Khi bạn báo cáo kết quả tích hợp SONIC vào pipeline VLA của dự án (`06-vla-groot-sonic/`), nếu trích dẫn con số "100% success rate" của SONIC làm điểm tham chiếu, hãy luôn kèm theo:

- Rõ đây là kết quả **zero-shot** trên 50 trajectory đa dạng — không phải một con số "sim-to-real gap = 0" tổng quát.
- Nếu bạn tự đo lại (hoặc thiết kế thí nghiệm tương tự) trên robot G1 của mình, hãy tách MPJPE theo bộ phận cơ thể như SONIC làm — ví dụ rất có thể bạn cũng sẽ thấy gap tập trung ở bàn chân (vì đây là nơi tiếp xúc, chịu ảnh hưởng mạnh nhất từ khác biệt mô hình ma sát sim-thật) — đây là một giả thuyết hợp lý để kiểm chứng trên hệ thống riêng của bạn, không phải một quy luật chắc chắn đúng cho mọi robot.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **SONIC (arXiv:2511.07820) tự thân đã là một trong những cập nhật SOTA gần nhất (2025-2026) cho lĩnh vực motion tracking humanoid** — theo tra cứu, SONIC được mô tả là hệ thống "Supersizing Motion Tracking for Natural Humanoid Whole-Body Control", với tác giả gồm Zhengyi Luo, Ye Yuan, Tingwu Wang, Chenran Li, Fernando Castañeda và cộng sự — giải quyết một hạn chế cụ thể của các phương pháp trước (như AMP-based, xem `04-imitation-learning-rl/`): "generative imitation methods có discriminator-based training signal dễ bị mode collapse khi motion dataset lớn và đa dạng lên" — đây chính là bối cảnh kỹ thuật khiến việc mở rộng quy mô (supersizing) huấn luyện trở nên khó khăn trước SONIC.
2. **SONIC tích hợp trực tiếp với VLA (GR00T)** — theo tra cứu, kiến trúc gồm một "high-level model xử lý thông tin thị giác và ngôn ngữ để quyết định bước di chuyển tiếp theo của robot", tức SONIC đóng vai trò low-level whole-body control layer nhận lệnh từ VLA cấp cao — đây là điểm liên kết trực tiếp với `06-vla-groot-sonic/` của dự án này, và là lý do case study sim-to-real gap của SONIC quan trọng cho toàn bộ pipeline VLA, không chỉ riêng phần motion tracking.
3. **Chưa tìm thấy một case study sim-to-real khác công bố công khai với độ chi tiết tương đương** (tách MPJPE theo bộ phận cơ thể, kèm 50-trajectory zero-shot binary test) trong khoảng 2024-2026 qua các lượt tra cứu đã chạy cho bài này — đây có thể là dấu hiệu SONIC đang thiết lập một chuẩn mực báo cáo mới (báo cáo cả nhị phân lẫn liên tục, cả toàn thân lẫn theo bộ phận) mà các paper tương lai có thể sẽ noi theo, nhưng nhận định này cần được xác minh lại khi có thêm paper mới xuất bản.

## ❓ Câu hỏi tự kiểm tra

1. "100% success rate trên 50 trajectory, zero-shot" của SONIC nói lên điều gì, và KHÔNG nói lên điều gì?
<details><summary>Gợi ý đáp án</summary>Nói lên: policy hoàn thành mọi trajectory thử (tiêu chí nhị phân) mà không cần bất kỳ dữ liệu thật nào để sửa lỗi trước. KHÔNG nói lên: sai số chi tiết ở từng bộ phận cơ thể là bằng 0 — MPJPE-L vẫn cho thấy gap thực sự, đặc biệt ở bàn chân.</details>

2. Tính gap tuyệt đối và tương đối cho bộ phận "upper body" từ số liệu sim=21.8mm, real=22.2mm.
<details><summary>Gợi ý đáp án</summary>Gap tuyệt đối = 22.2−21.8 = 0.4mm. Gap tương đối = 0.4/21.8 ≈ 1.8%.</details>

3. Vì sao không nên kết luận "SONIC tốt hơn BeyondMimic trên robot thật" chỉ dựa vào con số 100% success rate?
<details><summary>Gợi ý đáp án</summary>Vì theo nội dung đã xác minh, so sánh với BeyondMimic (và Any2Track, GMT) diễn ra trong simulation, không phải trên cùng 50 trajectory thật — không có bằng chứng BeyondMimic đã được thử zero-shot thật cùng điều kiện để so sánh trực tiếp.</details>

4. Tại sao gap ở bàn chân (24.7mm, ~85% tương đối) lớn hơn nhiều so với thân trên (0.4mm, ~1.8%)? Đưa ra 1 giả thuyết hợp lý.
<details><summary>Gợi ý đáp án</summary>Giả thuyết hợp lý: bàn chân là nơi tiếp xúc trực tiếp với mặt đất, chịu ảnh hưởng mạnh nhất từ sai lệch giữa mô hình ma sát/tiếp xúc trong simulation và ma sát/độ đàn hồi thực tế của sàn — trong khi thân trên không tiếp xúc với môi trường nên ít bị ảnh hưởng bởi loại sai lệch này. (Đây là giả thuyết hợp lý dựa trên hiểu biết chung về contact dynamics, không phải khẳng định trực tiếp từ paper.)</details>

5. Hai bộ số liệu (50-trajectory nhị phân và MPJPE-L theo bộ phận) trong paper SONIC có chắc chắn đo trên cùng một tập test không?
<details><summary>Gợi ý đáp án</summary>Không chắc chắn — đây là điểm "cần xác minh thêm" theo nội dung nguồn: hai bộ số liệu xuất hiện ở các phần khác nhau của paper và có thể là hai thí nghiệm riêng biệt, đo hai thứ khác nhau (nhị phân vs liên tục).</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** giả sử một hệ thống khác báo cáo MPJPE-L theo 2 bộ phận: lower body (chân, không tính bàn chân riêng) sim=25mm/real=31mm, và feet sim=30mm/real=48mm. Tính gap tuyệt đối và tương đối cho từng bộ phận, so sánh với con số feet của SONIC (29.0→53.7mm, gap tương đối 85.2%) — hệ thống giả định này có gap tương đối ở bàn chân lớn hơn hay nhỏ hơn SONIC?
2. **Đọc paper thật:** tìm và đọc trực tiếp phần Real-World Evaluation của SONIC (arXiv:2511.07820, qua trang chính thức nvlabs.github.io/SONIC hoặc bản PDF trên arXiv), xác minh lại chính xác số lần thử mỗi trajectory và làm rõ mối quan hệ giữa 2 bộ số liệu (50-trajectory và MPJPE-L 124-motion) — đây là 2 điểm "cần xác minh thêm" đã nêu trong bài, cập nhật lại phát hiện của bạn vào ghi chú cá nhân.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

SONIC báo cáo "100% success rate trên 50 trajectory thật, zero-shot" — một tuyên bố mạnh vì không có bước fine-tune, test set đa dạng (dance/jump/loco-manipulation, không chỉ đi bộ đơn giản), và tiêu chí "0 thất bại" nghiêm ngặt hơn nhiều so với báo cáo success rate trung bình; nhưng đây chỉ là một bộ số liệu nhị phân, tách biệt với bộ số liệu MPJPE-L liên tục (sim≈22.3mm vs real≈25.7mm toàn thân) cho thấy gap thực sự vẫn tồn tại và phân bố rất không đồng đều trên cơ thể — gần như không đáng kể ở thân trên (0.4mm, ~1.8%) nhưng rất lớn ở bàn chân (24.7mm, ~85% tương đối). Bài học tổng quát từ case study này: một tuyên bố sim-to-real mạnh cần được đọc cùng với chi tiết thiết kế thí nghiệm đứng sau nó (dạng transfer, độ đa dạng test set, baseline có cùng điều kiện hay không), và các bộ số liệu khác nhau trong cùng một paper (nhị phân vs liên tục) đo những khía cạnh khác nhau, không thể gộp lại thành một kết luận duy nhất "gap bằng 0".
