# Bài giảng: Vì sao RL thay thế được model-based control

*(Thuộc mảng: Whole-Body Control (WBC))*

## 🎯 Mục tiêu bài học

- Liệt kê chính xác 3 hạn chế của WBC cổ điển (model-based) mà RL nhắm giải quyết.
- Giải thích được vì sao "reward = giống chuyển động tham chiếu" giải quyết vấn đề "viết tay mục tiêu" tốt hơn "reward thủ công kiểu ZMP".
- Phân biệt được rõ ràng "RL thay thế model-based" nghĩa là gì và KHÔNG có nghĩa là gì (không phải RL luôn tốt hơn tuyệt đối).
- Tính tay được một ví dụ minh hoạ chi phí kỹ sư (engineering cost) giữa việc viết tay reward cho N hành vi khác nhau so với dùng một pipeline motion-tracking chung.
- Nêu được bằng chứng thực nghiệm 2025 về đánh đổi cụ thể giữa hai cách tiếp cận (không chỉ nhận định định tính).
- Liên hệ được đây là "bước ngoặt" chuyển từ mục 1 (WBC cổ điển) sang mục 2 (WBC học sâu) của thư mục này.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Bốn bài trước (Task-space control, QP, HQP, ZMP+cart-table) và hai bài liền trước (ZMP preview control, MPC) đã xây dựng đầy đủ bộ công cụ WBC cổ điển — mỗi công cụ đều có **đảm bảo toán học rõ ràng**. Bài này là điểm bản lề của toàn bộ thư mục: nó không giới thiệu một công thức mới, mà giải thích **động lực (motivation)** khiến cả một lĩnh vực nghiên cứu chuyển hướng từ "thiết kế bộ điều khiển bằng tay" sang "huấn luyện policy bằng dữ liệu/thử nghiệm" — để hiểu đúng phần còn lại của dự án (04-imitation-learning-rl/, và toàn bộ pipeline SONIC/GR00T) không dùng công thức WBC cổ điển đã học, bạn cần hiểu chính xác *tại sao*, không chỉ chấp nhận đó là "xu hướng mới hơn".

## 🧠 Trực giác

### Góc nhìn 1: Viết luật giao thông chi tiết cho MỌI tình huống so với dạy một tài xế học lái xe

WBC cổ điển giống việc cố viết một cuốn sách luật giao thông đủ chi tiết để bao quát **mọi tình huống có thể xảy ra khi lái xe** — "nếu gặp tình huống A thì làm B, nếu gặp tình huống A' hơi khác thì làm B'..." — càng nhiều tình huống thực tế, cuốn sách luật càng phải dày, và luôn có tình huống mới chưa được viết tới. RL giống việc **dạy một người học lái xe qua thực hành** — họ không học một cuốn sách luật cho mọi tình huống, mà học một kỹ năng tổng quát ("cảm giác lái") có thể tự thích ứng với tình huống mới chưa từng gặp, miễn là đã luyện tập đủ đa dạng.

**Giới hạn của loại suy này:** một tài xế con người có khả năng suy luận trừu tượng, hiểu ý nghĩa biển báo mới chưa từng thấy; một policy RL chỉ tổng quát hoá tốt trong phạm vi **phân phối dữ liệu/tình huống đã từng gặp khi huấn luyện** — vượt ra ngoài phạm vi đó, policy RL có thể thất bại hoàn toàn không báo trước, khác với con người thường "nhận ra" mình gặp tình huống lạ và cẩn trọng hơn.

### Góc nhìn 2: Đầu bếp theo công thức nấu ăn viết sẵn so với đầu bếp học nấu bằng nếm-thử-sửa

Một đầu bếp mới học việc theo đúng công thức viết sẵn ("cho 200g bột, 2 quả trứng...") sẽ nấu đúng công thức đó, nhưng gặp nguyên liệu hơi khác (bột ẩm hơn bình thường) có thể ra kết quả tệ vì công thức không tính tới biến thể đó. Một đầu bếp học qua nếm-thử-sửa hàng nghìn lần với nguyên liệu đa dạng sẽ phát triển được **trực giác điều chỉnh** khi gặp biến thể — nhưng quá trình học tốn rất nhiều lần thử (và có thể có những lần thử thất bại tệ hại trong lúc học, mà robot thật thì đắt/nguy hiểm để "thử thất bại", nên phải học trong mô phỏng trước — đúng lý do domain randomization và huấn luyện song song quy mô lớn tồn tại, xem 2 bài tiếp theo).

**Giới hạn của loại suy này:** đầu bếp học qua thử nghiệm với thực phẩm thật không cần một "mô phỏng nấu ăn" trước khi thử — robot RL thì bắt buộc phải học trong mô phỏng trước (vì thử trực tiếp trên robot thật hàng triệu lần là bất khả thi/nguy hiểm) rồi mới chuyển sang thật (sim-to-real) — một bước trung gian mà loại suy nấu ăn không có tương đương trực tiếp.

## 📐 Định nghĩa chính xác

**Ba hạn chế của WBC cổ điển (model-based)** trong thực tế:

1. **Phải mô hình hoá tường minh** động lực học robot, tiếp xúc, ma sát — trong khi động lực học thật (đặc biệt khi tiếp xúc va chạm, địa hình phức tạp) rất khó mô hình chính xác.
2. **Reward/mục tiêu phải viết tay** (ZMP nằm trong đế chân, tay theo quỹ đạo X) — với các hành vi phức tạp, tự nhiên như người thật (nhảy, xoay người linh hoạt, phản ứng bất ngờ), viết tay công thức mục tiêu gần như bất khả thi.
3. **Khó tổng quát hoá:** bộ điều khiển thiết kế cho một tình huống cụ thể không tự động hoạt động tốt ở tình huống khác.

**WBC học sâu** thay thế cách tiếp cận "viết tay luật điều khiển" bằng cách **huấn luyện một policy** (thường là mạng nơ-ron, học bằng Reinforcement Learning) sao cho robot **tự học cách hành động** để đạt mục tiêu — mục tiêu phổ biến nhất hiện nay không phải là reward thủ công kiểu "giữ ZMP trong đế chân" mà là **reward = "giống với một chuyển động tham chiếu từ dữ liệu chuyển động người thật"** (motion tracking/motion imitation) — chi tiết thuật toán thuộc về `04-imitation-learning-rl/` (DeepMimic, AMP).

**Điểm mấu chốt cần hiểu đúng:** "RL thay thế model-based" không có nghĩa RL giải quyết bài toán "tốt hơn tuyệt đối" theo mọi tiêu chí — nó chuyển đổi **loại chi phí kỹ sư** cần bỏ ra: từ "chi phí viết công thức/mô hình tường minh cho từng hành vi" sang "chi phí thu thập dữ liệu tham chiếu + hạ tầng tính toán huấn luyện quy mô lớn" (2 bài tiếp theo).

## ⚙️ Cơ chế hoạt động — từng bước

Sơ đồ so sánh trực tiếp hai quy trình thiết kế bộ điều khiển:

```text
┌─────────────────────────────────────────────────────────────┐
│ QUY TRÌNH MODEL-BASED (mục 1 của thư mục này)                  │
│                                                                 │
│  1. Kỹ sư PHÂN TÍCH bài toán vật lý (động lực học, tiếp xúc)   │
│  2. Kỹ sư VIẾT TAY mô hình toán học (M·q̈+h=τ+J_c^T·F_c, ZMP...) │
│  3. Kỹ sư VIẾT TAY mục tiêu/ràng buộc (giữ ZMP trong đế chân)   │
│  4. Giải bài toán tối ưu (QP/HQP/MPC) MỖI BƯỚC ĐIỀU KHIỂN       │
│     → cần lặp lại bước 2-3 cho MỖI hành vi mới muốn thêm         │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ QUY TRÌNH LEARNING-BASED (mục 2 của thư mục này)                │
│                                                                 │
│  1. Thu thập DỮ LIỆU chuyển động tham chiếu (mocap người thật,   │
│     retarget qua GMR/SOMA-retargeter — 02-motion-retargeting/)  │
│  2. Định nghĩa reward TỔNG QUÁT: "giống chuyển động tham chiếu"  │
│     (KHÔNG cần viết tay công thức riêng cho từng hành vi)        │
│  3. HUẤN LUYỆN policy (mạng nơ-ron) qua thử-sai trong mô phỏng   │
│     quy mô lớn (bài tiếp theo: Rudin et al. 2022)                │
│  4. Policy đã huấn luyện CHẠY TRỰC TIẾP (forward pass mạng,      │
│     không giải tối ưu online) → nhanh hơn nhiều so với MPC       │
└─────────────────────────────────────────────────────────────┘
```

Điểm khác biệt về **nơi đặt "trí tuệ"**: trong model-based, trí tuệ nằm trong **công thức do kỹ sư viết** (giải thích được, kiểm chứng được từng bước); trong learning-based, trí tuệ nằm trong **trọng số mạng nơ-ron đã học** (khó giải thích từng bước, nhưng có thể học được hành vi phức tạp mà không ai biết cách viết công thức tường minh).

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số tự chọn để định lượng hoá "chi phí kỹ sư" — không phải số liệu benchmark thật của bất kỳ dự án cụ thể nào)*

Giả sử một nhóm nghiên cứu muốn robot thực hiện được **N = 20 loại hành vi khác nhau** (đi bộ, chạy, nhảy, xoay người, leo cầu thang, vẫy tay, cúi nhặt đồ...).

**Chi phí theo hướng model-based (viết tay reward/công thức riêng cho từng hành vi):**

```text
Giả sử mỗi hành vi cần trung bình 3 tuần kỹ sư (phân tích động lực học riêng,
  thiết kế ràng buộc, tune tham số QP/MPC cho đúng hành vi đó)
Tổng chi phí = 20 hành vi × 3 tuần = 60 tuần-kỹ-sư
```

**Chi phí theo hướng motion-tracking (RL với reward tổng quát "giống chuyển động tham chiếu"):**

```text
Chi phí CỐ ĐỊNH (không đổi theo N):
  - Xây pipeline retargeting (GMR/SOMA-retargeter, đã học 02-motion-retargeting/): ~4 tuần (một lần)
  - Xây hạ tầng huấn luyện RL song song (05-simulation-mujoco-isaaclab/): ~4 tuần (một lần)
  - Định nghĩa reward tổng quát "motion tracking": ~1 tuần (một lần, không đổi theo hành vi)
  Tổng chi phí cố định = 9 tuần

Chi phí BIÊN cho mỗi hành vi mới (chỉ cần THÊM dữ liệu tham chiếu, không viết công thức mới):
  - Tìm/retarget 1 clip mocap cho hành vi đó: ~0.5 tuần/hành vi
  Tổng chi phí biên = 20 × 0.5 = 10 tuần

Tổng chi phí = 9 + 10 = 19 tuần-kỹ-sư
```

**So sánh:**

```text
Model-based: 60 tuần-kỹ-sư (tuyến tính theo N, không có phần "cố định" tách biệt)
Motion-tracking RL: 19 tuần-kỹ-sư (9 tuần cố định + 0.5×N tuần biên)
```

**Điểm hoà vốn (break-even point):** với `N` hành vi, chi phí model-based là `3N`, chi phí RL là `9 + 0.5N`. Giải `3N = 9 + 0.5N`:

```text
2.5N = 9
N ≈ 3.6
```

**Ý nghĩa:** với ví dụ minh hoạ này, ngay khi cần **hơn khoảng 4 hành vi khác nhau**, hướng motion-tracking RL đã rẻ hơn về tổng chi phí kỹ sư — và khoảng cách càng lớn khi N càng tăng (chi phí biên 0.5 tuần/hành vi so với 3 tuần/hành vi). Đây chính là cách định lượng hoá cụ thể cho nhận định "với các hành vi phức tạp, tự nhiên như người thật, viết tay công thức mục tiêu gần như bất khả thi" — không phải vì *không thể* viết công thức cho một hành vi cụ thể, mà vì **chi phí nhân lên tuyến tính theo số hành vi** trở nên không khả thi khi cần hàng chục/hàng trăm hành vi (đúng bối cảnh SONIC với "generalist humanoid controller", `06-vla-groot-sonic/`).

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | WBC cổ điển (model-based, mục 1) | WBC học sâu (learning-based, mục 2) |
|---|---|---|
| Cần mô hình động lực học chính xác | Bắt buộc | Không bắt buộc (học từ tương tác mô phỏng) |
| Cách định nghĩa mục tiêu | Viết tay công thức/ràng buộc riêng từng hành vi | Reward tổng quát (motion tracking), tái sử dụng cho mọi hành vi |
| Chi phí biên khi thêm 1 hành vi mới | Cao (thiết kế lại từ đầu) | Thấp (chỉ cần thêm dữ liệu tham chiếu) |
| Chi phí tính toán runtime (trên robot thật) | Có thể cao (giải QP/MPC mỗi bước) | Thấp (chỉ forward pass mạng đã huấn luyện) |
| Đảm bảo toán học tường minh | Có | Không (chỉ "khả năng cao" nếu huấn luyện tốt) |
| Đòi hỏi hạ tầng gì | Kiến thức điều khiển học sâu, solver QP/MPC | Hạ tầng mô phỏng GPU-parallel, dữ liệu mocap, domain randomization |
| Khi nào ưu thế | Cần đảm bảo an toàn tường minh, ít hành vi, có mô hình chính xác | Cần nhiều hành vi đa dạng, mô hình khó tường minh, có sẵn dữ liệu tham chiếu |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "RL thay thế model-based nghĩa là RL luôn tốt hơn model-based về mọi mặt".** Vì sao sai: như bảng so sánh cho thấy, model-based vẫn giữ ưu thế tuyệt đối ở khả năng **đảm bảo toán học tường minh** (constraint satisfaction) — một tính chất quan trọng cho các ứng dụng cần an toàn cao (ví dụ robot công nghiệp trong không gian hẹp với con người). RL không "thắng tuyệt đối", nó thắng cụ thể ở **bài toán tổng quát hoá qua nhiều hành vi phức tạp** — đúng bối cảnh mà dự án này (SONIC, humanoid VLA) đang giải quyết. **Hiểu đúng:** lựa chọn giữa hai hướng phụ thuộc vào đặc thù bài toán, không có "người thắng tuyệt đối" — thực tế 2025-2026 (đã học ở bài MPC) còn cho thấy xu hướng kết hợp cả hai.
2. **Hiểu nhầm: "reward 'motion tracking' loại bỏ hoàn toàn nhu cầu viết tay công thức".** Vì sao sai: bản thân hàm reward "so sánh với chuyển động tham chiếu" (chi tiết ở `04-imitation-learning-rl/`) vẫn cần thiết kế cẩn thận (trọng số so sánh vị trí khớp/vận tốc/contact...) — chỉ là công thức đó **tổng quát và tái sử dụng được cho mọi hành vi**, thay vì phải viết một công thức mục tiêu hoàn toàn khác cho từng hành vi cụ thể (như "giữ ZMP trong đế chân" chỉ áp dụng cho đi/đứng, không áp dụng trực tiếp cho vẫy tay). **Hiểu đúng:** đây là chuyển từ "N công thức khác nhau" sang "1 công thức tổng quát + N tập dữ liệu tham chiếu", không phải "0 công thức nào cả".

## 🏗️ Ví dụ minh hoạ trong dự án này

```text
Yêu cầu: robot G1 cần thực hiện được nhiều hành vi tự nhiên khác nhau
  (đi, chạy, tương tác vật thể, phản ứng linh hoạt theo ngữ cảnh)
        │
        ▼
NẾU đi theo hướng model-based (mục 1 của thư mục này):
  → cần thiết kế lại ràng buộc/mục tiêu QP-MPC cho MỖI hành vi
  → không khả thi ở quy mô "generalist controller" mà SONIC nhắm tới
        │
        ▼
THAY VÀO ĐÓ, dự án đi theo hướng WBC học sâu:
  AMASS/LAFAN1/BONES-SEED (03-human-motion-datasets/, hàng trăm giờ dữ liệu)
        │
        ▼
  GMR/SOMA-retargeter (02-motion-retargeting/) — retarget MỘT LẦN,
    quy mô lớn, không cần thiết kế công thức riêng từng clip
        │
        ▼
  Huấn luyện policy motion-tracking (04-imitation-learning-rl/,
    Isaac Lab/MuJoCo Playground, 05-simulation-mujoco-isaaclab/)
        │
        ▼
  MỘT policy (hoặc kiến trúc decoupled như SONIC, bài giảng sau)
    xử lý được TOÀN BỘ phạm vi hành vi trong dữ liệu huấn luyện
```

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **Benchmark trực tiếp xác nhận đánh đổi (không chỉ nhận định định tính).** Như đã trích ở bài "MPC cho dáng đi", *"Benchmarking Model Predictive Control and Reinforcement Learning Based Control for Legged Robot Locomotion in MuJoCo Simulation"* ([arXiv:2501.16590](https://arxiv.org/abs/2501.16590), 2025) xác nhận bằng thực nghiệm: RL vượt trội xử lý nhiễu/hiệu quả năng lượng, nhưng "gặp khó khăn tổng quát hoá sang địa hình mới do phụ thuộc vào policy đã học tailored cho môi trường cụ thể" — một bằng chứng thực nghiệm cụ thể cho hạn chế thứ 3 (khó tổng quát hoá) mà chính hướng learning-based cũng gặp phải, không chỉ model-based.
2. **SONIC (arXiv:2511.07820, xem bài "Kiến trúc decoupled WBC của SONIC") là minh chứng quy mô lớn nhất cho luận điểm "reward tổng quát + dữ liệu lớn thắng thế viết tay công thức riêng".** SONIC huấn luyện một "generalist humanoid controller" từ hơn 100 triệu khung hình / 700 giờ dữ liệu mocap, dữ liệu được retarget bằng chính GMR đã học ở `02-motion-retargeting/` — quy mô dữ liệu và compute (9.000+ giờ GPU) là bằng chứng thực tế cho xu hướng "đầu tư vào dữ liệu + hạ tầng huấn luyện" thay vì "đầu tư vào thiết kế công thức riêng từng hành vi" đã phân tích ở ví dụ tính tay.
3. **Xu hướng chung 2025-2026: không phải "RL thắng, MPC thua" mà là "ranh giới giữa hai cách tiếp cận đang hội tụ".** Các kiến trúc hybrid (RL-compensated MPC, MPC với residual học được — đã học ở bài MPC) cho thấy câu hỏi thực tế không còn là "chọn RL hay model-based" mà là "kết hợp thế mạnh của cả hai ở đâu trong kiến trúc điều khiển tổng thể" — một sự tinh tế hơn nhiều so với khung "RL thay thế model-based" đơn giản hoá.

## ❓ Câu hỏi tự kiểm tra

1. Liệt kê 3 hạn chế của WBC cổ điển mà WBC học sâu nhắm giải quyết.
   <details><summary>Gợi ý đáp án</summary>(1) Phải mô hình hoá tường minh động lực học/tiếp xúc/ma sát; (2) reward/mục tiêu phải viết tay, bất khả thi cho hành vi phức tạp tự nhiên; (3) khó tổng quát hoá sang tình huống khác.</details>
2. Trong ví dụ tính tay, điểm hoà vốn N≈3.6 có ý nghĩa gì, và tại sao con số cụ thể không quan trọng bằng xu hướng nó thể hiện?
   <details><summary>Gợi ý đáp án</summary>Nó cho thấy ngay khi số hành vi cần vượt một ngưỡng nhỏ, chi phí kỹ sư của hướng motion-tracking RL đã thấp hơn model-based nhờ chi phí biên nhỏ hơn nhiều (0.5 tuần/hành vi so với 3 tuần/hành vi) — con số 3.6 chỉ là minh hoạ với giả định tự chọn, điều quan trọng là XU HƯỚNG chi phí biên thấp hơn nhiều, càng rõ rệt khi N lớn (như trường hợp SONIC với hàng trăm/nghìn giờ dữ liệu).</details>
3. Vì sao nói "RL thay thế model-based" không có nghĩa RL luôn tốt hơn tuyệt đối?
   <details><summary>Gợi ý đáp án</summary>Vì model-based vẫn có ưu thế tuyệt đối ở khả năng đảm bảo toán học tường minh (constraint satisfaction) — quan trọng cho ứng dụng cần an toàn cao; benchmark 2025 còn cho thấy RL cũng có hạn chế tổng quát hoá riêng (kém hơn khi gặp địa hình/tình huống chưa từng huấn luyện).</details>
4. Reward "motion tracking" (giống chuyển động tham chiếu) giải quyết vấn đề "viết tay mục tiêu" như thế nào — có phải là loại bỏ hoàn toàn việc thiết kế công thức không?
   <details><summary>Gợi ý đáp án</summary>Không loại bỏ hoàn toàn — vẫn cần thiết kế công thức so sánh (trọng số vị trí/vận tốc/contact...) nhưng công thức đó TỔNG QUÁT, dùng lại được cho MỌI hành vi, thay vì phải viết một công thức mục tiêu hoàn toàn riêng biệt cho từng hành vi cụ thể như hướng model-based.</details>
5. SONIC minh hoạ luận điểm nào của bài này bằng quy mô thực tế?
   <details><summary>Gợi ý đáp án</summary>Minh hoạ luận điểm "đầu tư vào dữ liệu + hạ tầng huấn luyện thắng thế đầu tư vào thiết kế công thức riêng từng hành vi" — SONIC dùng hơn 100 triệu khung hình/700 giờ mocap (retarget bằng GMR) và 9.000+ giờ GPU để huấn luyện MỘT policy generalist, thay vì thiết kế công thức WBC riêng cho từng hành vi.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** với chi phí model-based là 4 tuần/hành vi, chi phí cố định motion-tracking là 12 tuần, chi phí biên là 0.3 tuần/hành vi — tính điểm hoà vốn N, và tổng chi phí mỗi hướng khi N=50.
2. **Đọc paper thật:** đọc phần Introduction/Abstract của SONIC ([arXiv:2511.07820](https://arxiv.org/abs/2511.07820)) — tìm câu văn cụ thể mô tả vấn đề "current neural controllers for humanoids remain modest in size, targeting a limited set of behaviors" — giải thích bằng lời (2-3 câu) cách câu này liên hệ trực tiếp với hạn chế thứ 2 và thứ 3 đã học ở bài này (viết tay mục tiêu, khó tổng quát hoá).

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

WBC học sâu thay thế WBC cổ điển không phải vì "tốt hơn tuyệt đối", mà vì giải quyết ba hạn chế cụ thể — mô hình hoá tường minh khó chính xác, mục tiêu phải viết tay bất khả thi cho hành vi phức tạp, khó tổng quát hoá — bằng cách chuyển từ "viết công thức riêng cho từng hành vi" sang "một reward tổng quát (motion tracking) + huấn luyện trên dữ liệu chuyển động người quy mô lớn"; như ví dụ tính tay minh hoạ, chi phí kỹ sư của hướng này có chi phí cố định ban đầu cao hơn nhưng chi phí biên mỗi hành vi mới thấp hơn nhiều, khiến nó áp đảo khi cần hàng chục/hàng trăm hành vi (đúng quy mô SONIC — 100M+ khung hình, dữ liệu retarget bằng GMR). Benchmark thực nghiệm 2025 (arXiv:2501.16590) xác nhận đánh đổi thực tế: RL xử lý nhiễu tốt hơn nhưng tổng quát hoá kém hơn tới địa hình mới — nên xu hướng 2025-2026 không phải "RL thắng tuyệt đối" mà là các kiến trúc hybrid kết hợp thế mạnh của cả model-based và learning-based.
