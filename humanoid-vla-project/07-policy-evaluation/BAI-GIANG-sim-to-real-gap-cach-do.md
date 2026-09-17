# Bài giảng: Sim-to-real gap — cách đo và 3 dạng báo cáo

*(Thuộc mảng: Policy Evaluation)*

## 🎯 Mục tiêu bài học

- Viết được công thức sim-to-real gap và tính tay một ví dụ số cụ thể.
- Phân biệt được 3 dạng báo cáo gap: zero-shot transfer, fine-tuned transfer, và domain-randomization-coverage — biết câu hỏi cần hỏi khi đọc một paper báo cáo gap.
- Giải thích được vì sao so sánh gap giữa 2 paper dùng 2 dạng khác nhau là so sánh không công bằng.
- Nhận ra các nguồn gốc (nguyên nhân) chính gây ra sim-to-real gap trong humanoid robotics.
- Áp dụng được khung đo gap này để tự đánh giá policy của bạn khi có (hoặc chưa có) robot thật.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Toàn bộ pipeline huấn luyện của dự án (`04-imitation-learning-rl/`, `05-simulation-mujoco-isaaclab/`) diễn ra trong simulation — vì huấn luyện trực tiếp trên robot thật quá chậm, quá đắt, và quá rủi ro (một cú ngã có thể phá hỏng động cơ 50,000 USD). Nhưng mục tiêu cuối cùng luôn là robot thật (`08-real-robot-deployment/`). Sim-to-real gap là câu hỏi "khoảng cách giữa 2 thế giới đó lớn tới đâu" — và nếu bạn không biết chính xác gap được đo theo dạng nào, bạn sẽ dễ tin nhầm một tuyên bố "gap = 0" mạnh mẽ hơn (hoặc yếu hơn) thực tế của nó.

## 🧠 Trực giác

### Góc nhìn 1: "Học lái xe trong game vs lái xe thật ngoài đường"

Một người chơi game lái xe rất giỏi (điểm cao trong sim) không tự động lái được xe thật ngoài đường — vì game không mô phỏng đúng độ trễ phanh thật, ma sát lốp thật, phản ứng của các xe khác thật. Khoảng cách giữa "giỏi trong game" và "giỏi ngoài đường" chính là sim-to-real gap.

- **Đúng ở đâu:** nắm đúng ý tưởng — hiệu năng trong 2 môi trường có thể khác nhau dù cùng một "người lái" (chính sách/policy).
- **Giới hạn:** loại suy này ngụ ý gap chỉ có 1 con số duy nhất ("giỏi ngoài đường tới đâu") — thực tế robotics phân biệt rõ *cách đo* gap đó (có được "luyện thêm" ngoài đường trước khi đánh giá hay không, ví dụ vài buổi lái thử thật trước khi thi bằng lái) — đây chính là sự khác biệt giữa zero-shot và fine-tuned transfer ở phần dưới, loại suy đơn giản không phân biệt được 2 trường hợp này.

### Góc nhìn 2: "Bản đồ và lãnh thổ" (map vs territory)

Simulation là một tấm bản đồ — luôn là một mô hình xấp xỉ, không bao giờ là chính lãnh thổ thật. Bản đồ càng chi tiết (mô phỏng vật lý càng chính xác, domain randomization càng rộng), càng dễ đi lạc ít hơn khi ra thực địa — nhưng không có tấm bản đồ nào hoàn hảo tuyệt đối.

- **Đúng ở đâu:** nhấn mạnh gap là **cấu trúc**, không phải lỗi ngẫu nhiên — nó tồn tại vì bản chất simulation luôn là một xấp xỉ (động lực học tiếp xúc, độ trễ actuator, nhiễu cảm biến đều bị đơn giản hoá).
- **Giới hạn:** loại suy "bản đồ" gợi ý gap chỉ đến từ việc mô phỏng vật lý chưa đủ tốt — nhưng thực tế gap còn đến từ **sai lệch nhận thức (perception gap)**: camera/cảm biến thật có nhiễu, độ trễ, và điều kiện ánh sáng khác hẳn render trong sim — đây là một nguồn gap riêng biệt mà "bản đồ địa hình" không lột tả được.

## 📐 Định nghĩa chính xác

```
sim-to-real gap = performance(sim) − performance(real)
```

tính trên **cùng một tập task**, cùng metric (success rate, MPJPE, v.v.), cùng điều kiện thử càng giống nhau càng tốt. Theo cách hình thức hoá gần đây hơn (Zhao, Queralta & Westerlund, 2020, arXiv:2009.13303, và được các survey 2025 kế thừa): với policy π, gọi `ψ_sim(π)` và `ψ_real(π)` là hiệu năng đo được của cùng policy π trong sim và trong thực tế:

```
G(π) = ψ_sim(π) − ψ_real(π)
```

Gap dương lớn = policy "ảo tưởng" trong sim nhưng không chuyển giao được ra thật — dấu hiệu domain randomization/reward shaping chưa đủ.

### Ba dạng báo cáo gap (bắt buộc phải phân biệt khi đọc một paper)

| Dạng | Định nghĩa | Ý nghĩa khi gap nhỏ |
|---|---|---|
| **1. Zero-shot transfer** | Policy huấn luyện hoàn toàn trong sim, deploy thẳng lên robot thật **không hề có bước fine-tune nào** | Bằng chứng mạnh nhất cho domain randomization/system identification tốt — không có cơ hội "vá lỗi" bằng dữ liệu thật |
| **2. Fine-tuned transfer** | Sau khi deploy, cho phép một lượng nhỏ dữ liệu thật để fine-tune policy trước khi đo performance cuối | Gap đo được thường nhỏ hơn nhưng **không phản ánh** khả năng của riêng simulation training |
| **3. Domain randomization coverage** | Thay vì đo gap trực tiếp, báo cáo *độ rộng* phân phối tham số được randomize (khối lượng, ma sát, độ trễ actuator, nhiễu cảm biến...) khi huấn luyện | Proxy gián tiếp cho "khả năng chịu được sim-to-real gap trong tương lai" — hữu ích khi chưa có robot thật để đo trực tiếp |

**Câu hỏi bắt buộc khi đọc một paper báo cáo sim-to-real gap:** gap được đo theo dạng nào trong 3 dạng trên? — so sánh gap "zero-shot" của paper A với gap "fine-tuned" của paper B là so sánh không công bằng, vì fine-tuned transfer gần như luôn cho gap nhỏ hơn một cách "không công bằng" (đã dùng dữ liệu thật để sửa lỗi).

## ⚙️ Cơ chế hoạt động — từng bước

```
┌────────────── Quy trình đo sim-to-real gap đúng cách ──────────────┐
│                                                                       │
│  BƯỚC 1 — Chọn 1 tập task cố định T = {t1, t2, ..., tn}              │
│           (ví dụ: 20 trajectory đi bộ + 10 trajectory thao tác)      │
│                                                                       │
│  BƯỚC 2 — Chọn 1 metric cố định M (success rate, hoặc MPJPE-L, ...)  │
│           dùng công thức GIỐNG HỆT cho cả 2 môi trường               │
│                                                                       │
│  BƯỚC 3 — Chạy policy π (KHÔNG thay đổi trọng số) trong sim trên T:  │
│           ψ_sim(π) = M(π chạy trên T, trong simulation)              │
│                                                                       │
│  BƯỚC 4 — Quyết định dạng transfer TRƯỚC khi chạy robot thật:        │
│    (a) Zero-shot: deploy π y hệt, không sửa gì                       │
│    (b) Fine-tuned: cho phép fine-tune trên dữ liệu thật trước         │
│    → Đây là quyết định phải NÊU RÕ khi báo cáo, không được giấu       │
│                                                                       │
│  BƯỚC 5 — Chạy π (đã chọn dạng ở bước 4) trên robot thật, cùng T:    │
│           ψ_real(π) = M(π chạy trên T, trên robot thật)               │
│                                                                       │
│  BƯỚC 6 — Tính gap:                                                  │
│           G(π) = ψ_sim(π) − ψ_real(π)                                │
│           → BÁO CÁO KÈM dạng transfer đã chọn ở bước 4                │
└──────────────────────────────────────────────────────────────────┘
```

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số liệu tự chọn để dễ hình dung — xem bài giảng riêng "sim-to-real-gap-case-study-sonic" cho số liệu thật đã xác minh từ paper SONIC.)*

Giả sử bạn đo trên 30 trajectory đi bộ, metric = success rate (không ngã suốt trajectory):

- Trong sim: 29/30 trajectory thành công → `ψ_sim = 29/30 × 100% ≈ 96.7%`
- Trên robot thật (zero-shot, không fine-tune): 24/30 trajectory thành công → `ψ_real = 24/30 × 100% = 80.0%`

```
G(π) = 96.7% − 80.0% = 16.7 điểm phần trăm
```

Đây là gap dạng **zero-shot** — 16.7 điểm là một gap tương đối lớn, gợi ý domain randomization chưa phủ đủ (ví dụ chưa randomize đủ rộng ma sát sàn hoặc độ trễ actuator thật).

**Bây giờ giả sử bạn cho phép fine-tune với 100 episode dữ liệu thật** trước khi đo lại: success rate trên robot thật tăng lên 28/30 = 93.3%.

```
G_fine-tuned(π) = 96.7% − 93.3% = 3.4 điểm phần trăm
```

Gap giảm từ 16.7 xuống 3.4 điểm phần trăm — nhưng **đây không phải là bằng chứng domain randomization tốt hơn**, mà chỉ là bằng chứng "fine-tune trên dữ liệu thật giúp ích" — một sự thật gần như luôn đúng và không nói lên gì về chất lượng sim training gốc. Nếu một người chỉ đọc con số "gap = 3.4%" mà không biết đây là dạng fine-tuned, họ sẽ đánh giá quá cao chất lượng domain randomization của hệ thống này so với thực tế.

**Domain randomization coverage (dạng 3) là một cách báo cáo khác hẳn:** thay vì con số gap, bạn báo cáo ví dụ "khối lượng thân robot được randomize ±15%, hệ số ma sát sàn randomize trong [0.4, 1.2], độ trễ actuator randomize [0, 20]ms" — đây không phải một con số gap, mà là một mô tả về "độ rộng bảo hiểm" cho tương lai, dùng khi bạn chưa có robot thật để đo trực tiếp ψ_real.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Zero-shot gap | Fine-tuned gap | Domain-randomization coverage |
|---|---|---|---|
| Cần robot thật để đo? | Có | Có | Không bắt buộc (proxy gián tiếp) |
| Độ tin cậy như bằng chứng cho sim training | Cao nhất | Thấp — bị "làm đẹp" bởi dữ liệu thật | Trung bình — chỉ là proxy, chưa xác nhận thực tế |
| Rủi ro khi so sánh xuyên paper | Có thể so sánh nếu cùng dạng | Không thể so sánh với zero-shot | Không so sánh trực tiếp bằng số được — chỉ so sánh định tính |
| Khi nào dùng | Khi có robot thật và muốn tuyên bố mạnh | Khi mục tiêu là hiệu năng cuối cùng, không chứng minh sim tốt | Khi CHƯA có robot thật, cần báo cáo tạm |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "gap càng nhỏ luôn chứng tỏ hệ thống tốt hơn", so sánh trực tiếp con số gap giữa các paper.**
   Vì sao sai: như ví dụ tính tay ở trên, gap 3.4% (fine-tuned) không "tốt hơn" gap 16.7% (zero-shot) của cùng một hệ thống — chúng đo hai thứ khác nhau. Một paper báo cáo "gap chỉ 2%" bằng cách fine-tune trên hàng trăm episode dữ liệu thật không chứng minh được domain randomization tốt hơn một paper báo cáo "gap 15% zero-shot".
   Hiểu đúng: luôn kiểm tra dạng báo cáo (zero-shot/fine-tuned/coverage) trước khi so sánh bất kỳ con số gap nào giữa 2 nguồn khác nhau, đúng nguyên tắc đã nêu ở mục Định nghĩa.

2. **Hiểu nhầm: sim-to-real gap chỉ đến từ "vật lý mô phỏng chưa đủ chính xác" (contact dynamics, ma sát...).**
   Vì sao sai: theo các survey gần đây (xem mục Cập nhật hiện đại), gap còn đến từ nhiều nguồn khác: perception gap (cảm biến/camera thật có nhiễu và độ trễ khác hẳn simulation render), actuator gap (động cơ thật có độ trễ, độ rơ, giới hạn mô-men khác model lý tưởng trong sim), và cả gap về latency tính toán (thời gian suy luận policy thật chậm hơn trong vòng lặp điều khiển thực).
   Hiểu đúng: khi phân tích nguyên nhân gap, cần xét đủ 3 nhóm nguồn: dynamics gap (động lực học), perception gap (cảm nhận), và actuation/latency gap (thực thi) — không chỉ đổ lỗi cho vật lý tiếp xúc.

## 🏗️ Ví dụ minh hoạ trong dự án này

Khi bạn huấn luyện policy ở `04-imitation-learning-rl/` với domain randomization (xem `BAI-GIANG-domain-randomization` ở thư mục 01/04), và có kế hoạch deploy lên robot G1 thật ở `08-real-robot-deployment/`, hãy tự hỏi trước khi báo cáo bất kỳ con số gap nào:

- Đây có phải zero-shot không, hay đã cho phép một vòng fine-tune nhỏ trên dữ liệu robot thật?
- Tập task dùng để đo trong sim và trên robot thật có thực sự giống hệt nhau (cùng trajectory, cùng điều kiện môi trường mô phỏng được tới đâu — ví dụ cùng loại sàn cứng/mềm)?
- Nếu chưa có robot G1 thật để thử, báo cáo theo dạng (3) — mô tả cụ thể độ rộng domain randomization đã dùng (khối lượng, ma sát, độ trễ...) thay vì bịa ra một con số gap không đo được.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **Một survey lớn 2025 hình thức hoá lại sim-to-real gap bằng khung POMDP.** Theo tra cứu (WebSearch tháng 9/2026), bài *"Reality Gap in Robotics: Analysis & Solutions"* (tóm tắt trên EmergentMind, tham chiếu arXiv:2510.20808) trình bày một khung phân tích chặt chẽ hơn: hình thức hoá reality gap bằng POMDP (Partially Observable Markov Decision Process), phân biệt rõ discrepancy trong dynamics, perception, và actuation giữa sim và real — đúng khớp với 3 nhóm nguồn gap đã nêu ở mục "Sai lầm thường gặp" số 2, cho thấy literature đang đồng thuận về việc gap không chỉ là 1 nguồn duy nhất.
2. **Một survey khác 2025 tổng hợp hơn 250 paper riêng cho legged robot / bipedal (bao gồm humanoid)** về kỹ thuật giảm sim-to-real gap, với phát hiện chính: domain randomization vẫn là kỹ thuật thống trị (xuất hiện trong hơn 80 hệ thống được khảo sát), nhưng chuyển giao thành công thường đòi hỏi **kết hợp nhiều kỹ thuật cùng lúc** (domain randomization + system identification + residual learning...) thay vì chỉ dựa vào một kỹ thuật đơn lẻ — kết luận này củng cố lý do vì sao dạng báo cáo (3) "domain randomization coverage" một mình không đủ để dự đoán chắc chắn gap thực tế sẽ nhỏ.
3. **Một hướng đo lường thực dụng hơn xuất hiện gần đây**: thay vì đo gap trên toàn bộ task phức tạp, một số nghiên cứu đưa ra **protocol đơn giản hoá** để cô lập "embodiment gap" — ví dụ cố định lệnh vận tốc tiến (forward velocity command) và đo sai số bám vận tốc trung bình (velocity tracking error) ở cả sim và thật, tạo ra một con số gap "tối giản" dễ tái lập giữa các nhóm nghiên cứu hơn là so sánh trên cả một bộ task phức tạp khó đồng nhất điều kiện — đây là một xu hướng đáng chú ý hướng tới các phép đo gap dễ chuẩn hoá, dễ tái lập hơn.

## ❓ Câu hỏi tự kiểm tra

1. Viết công thức sim-to-real gap và giải thích ý nghĩa của gap dương lớn.
<details><summary>Gợi ý đáp án</summary>G(π) = ψ_sim(π) − ψ_real(π). Gap dương lớn nghĩa là policy hoạt động tốt trong sim nhưng kém hẳn khi chuyển sang thực tế — dấu hiệu domain randomization/mô phỏng chưa đủ sát thực tế.</details>

2. Tại sao không nên so sánh trực tiếp gap zero-shot của một paper với gap fine-tuned của paper khác?
<details><summary>Gợi ý đáp án</summary>Vì fine-tuned transfer cho phép sửa lỗi bằng dữ liệu thật trước khi đo, gần như luôn cho gap nhỏ hơn một cách "không công bằng" — nó không phản ánh chất lượng riêng của quá trình huấn luyện trong simulation, khác hẳn ý nghĩa của gap zero-shot.</details>

3. Kể tên 3 nhóm nguồn gây ra sim-to-real gap ngoài "vật lý tiếp xúc chưa chính xác".
<details><summary>Gợi ý đáp án</summary>Dynamics gap (mô hình vật lý/tiếp xúc), perception gap (cảm biến/camera thật có nhiễu, độ trễ khác sim), và actuation/latency gap (động cơ thật có độ trễ/độ rơ, và độ trễ tính toán trong vòng lặp điều khiển thực).</details>

4. Với ψ_sim = 88%, ψ_real (zero-shot) = 61%, tính gap và nhận xét mức độ nghiêm trọng.
<details><summary>Gợi ý đáp án</summary>G = 88% − 61% = 27 điểm phần trăm — đây là gap khá lớn, gợi ý domain randomization/system identification cần cải thiện đáng kể trước khi tin tưởng deploy rộng rãi.</details>

5. Domain-randomization-coverage (dạng 3) khác 2 dạng còn lại ở điểm cốt lõi nào, và khi nào bạn buộc phải dùng dạng này?
<details><summary>Gợi ý đáp án</summary>Nó không đo trực tiếp ψ_real (không cần robot thật) mà chỉ báo cáo độ rộng phân phối tham số đã randomize khi huấn luyện — một proxy gián tiếp. Buộc phải dùng khi chưa có robot thật để thử nghiệm trực tiếp.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** giả sử ψ_sim = 95% (40/42 trajectory), zero-shot ψ_real = 71% (30/42), sau đó fine-tune trên 50 episode thật ψ_real tăng lên 90% (38/42). Tính G(π) cho cả 2 dạng (zero-shot và fine-tuned), và viết 1 câu giải thích tại sao không nên báo cáo "gap chỉ 5%" mà bỏ qua con số 24% zero-shot.
2. **Đọc paper thật:** mở phần Experiments của một trong các survey đã dẫn ở mục Cập nhật hiện đại (arXiv:2510.20808, hoặc arXiv:2009.13303 đã dẫn trong `NOI-DUNG-CHI-TIET.md`), tìm 1 ví dụ cụ thể paper đó trích dẫn có báo cáo gap theo dạng nào (zero-shot/fine-tuned/coverage), ghi lại chính xác con số và điều kiện thí nghiệm.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

Sim-to-real gap là hiệu số giữa hiệu năng của cùng một policy đo trong simulation và trên robot thật (G(π) = ψ_sim − ψ_real), nhưng con số này chỉ có ý nghĩa khi biết rõ nó thuộc dạng báo cáo nào trong 3 dạng: zero-shot transfer (không fine-tune, bằng chứng mạnh nhất cho chất lượng sim training), fine-tuned transfer (cho phép sửa bằng dữ liệu thật, gap nhỏ hơn nhưng không chứng minh gì về sim training), hoặc domain-randomization-coverage (báo cáo độ rộng randomization như một proxy gián tiếp khi chưa có robot thật để đo trực tiếp). So sánh gap giữa 2 hệ thống mà không kiểm tra chúng cùng dạng báo cáo là một lỗi phổ biến và nghiêm trọng, và nguồn gốc của gap không chỉ đến từ vật lý tiếp xúc mà còn từ perception gap (cảm biến) và actuation/latency gap (thực thi) — ba nhóm nguồn này đều cần được xem xét khi phân tích và cố gắng thu hẹp gap.
