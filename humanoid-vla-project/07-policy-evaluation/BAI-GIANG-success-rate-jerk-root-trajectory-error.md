# Bài giảng: Metric motion tracking — success rate, jerk, root trajectory error

*(Thuộc mảng: Policy Evaluation)*

## 🎯 Mục tiêu bài học

- Định nghĩa được "thành công" cho một episode motion-tracking một cách cụ thể, kiểm chứng được (không chỉ nói chung chung "hoàn thành task").
- Viết được công thức jerk từ đạo hàm bậc 3 của vị trí, tính tay được jerk rời rạc từ một chuỗi vị trí giả định.
- Phân biệt được root trajectory error (position + orientation) với MPJPE toàn thân, giải thích được vì sao 2 con số này có thể mâu thuẫn nhau.
- Nhận ra được vì sao mỗi metric riêng lẻ (success rate, jerk, root error) đều có thể bị "lách luật" nếu đọc một mình, và cách đọc kết hợp cả bộ 4 (cùng MPJPE) để tránh bị đánh lừa.
- Áp dụng được cả 3 metric này để tự thiết kế bộ đánh giá cho policy của chính bạn.

## 🧭 Vì sao cần học cái này? (bối cảnh)

MPJPE (xem bài giảng riêng `BAI-GIANG-mpjpe-va-cac-bien-the.md`) đo "sai lệch trung bình" nhưng không tự nói cho bạn 3 điều quan trọng: (1) robot có thực sự hoàn thành nhiệm vụ hay không (một con số trung bình thấp không loại trừ khả năng robot ngã ở giữa episode), (2) chuyển động có mượt/khả thi trên phần cứng thật hay không (một policy có MPJPE thấp vẫn có thể "giật cục" tới mức động cơ thật không theo kịp), và (3) lỗi có tập trung ở "cái nền" — vị trí/hướng gốc thân — hay phân bố đều khắp cơ thể. Ba metric trong bài này (success rate, jerk, root trajectory error) lấp đầy 3 khoảng trống đó. Không có bộ 4 metric này đi cùng nhau, một con số MPJPE đẹp có thể là ảo giác.

## 🧠 Trực giác

### Góc nhìn 1: "Kỳ thi có điểm số và có đỗ/trượt"

MPJPE giống như điểm trung bình các câu hỏi trong một bài thi (một con số liên tục) — nhưng một trường học còn cần một ngưỡng "đỗ/trượt" riêng (success rate) vì đôi khi một học sinh đạt điểm trung bình cao nhưng bị đánh trượt vì phạm một lỗi nghiêm trọng ở một câu bắt buộc (ví dụ ngã giữa chừng). Success rate là con số nhị phân đó — độc lập với điểm trung bình.

- **Đúng ở đâu:** làm rõ success rate không phải là "phiên bản đã làm tròn" của MPJPE — nó đo một trục hoàn toàn khác (có/không hoàn thành), không phải mức độ chính xác.
- **Giới hạn:** phép loại suy "đỗ/trượt" ngụ ý tiêu chí cố định và khách quan (giống đề thi chuẩn hoá) — trong thực tế, tiêu chí "thành công" cho robot **do chính người thiết kế thí nghiệm chọn**, và đây chính là chỗ dễ bị lạm dụng (chọn ngưỡng dễ đạt sau khi thấy kết quả) — loại suy kỳ thi không cảnh báo bạn về rủi ro này.

### Góc nhìn 2: "Người lái xe êm tay vs người lái xe giật cục"

Hai tài xế có thể đưa xe tới đúng điểm đến (cùng "thành công", cùng vị trí cuối gần như giống hệt tham chiếu — MPJPE cuối cùng thấp), nhưng một người phanh/ga đều tay (jerk thấp), người kia liên tục đạp phanh gấp rồi tăng tốc đột ngột (jerk cao) — hành khách (và cơ cấu động cơ) chịu tải hoàn toàn khác nhau dù đích đến giống nhau.

- **Đúng ở đâu:** nắm đúng ý tưởng "cùng kết quả cuối, khác chất lượng quá trình" — jerk đo *cách* đạt tới kết quả, không đo *có đạt được* kết quả hay không.
- **Giới hạn:** loại suy này dễ khiến người học nghĩ "jerk thấp luôn tốt hơn" — sai, vì một chiếc xe đứng yên hoàn toàn (không bao giờ di chuyển) cũng có jerk = 0 tuyệt đối nhưng vô dụng. Jerk chỉ có ý nghĩa khi đọc cùng với việc xe *có* di chuyển/hoàn thành nhiệm vụ.

## 📐 Định nghĩa chính xác

### Success rate

```
Success rate = (số episode thành công / tổng số episode thử) × 100%
```

Định nghĩa "thành công" phải khai báo **trước khi chạy thí nghiệm**, ví dụ (không đầy đủ, phải chọn và ghi rõ cho từng thí nghiệm cụ thể):
- Không ngã suốt episode (thân trên không chạm sàn / root không xuống dưới ngưỡng độ cao cho trước, ví dụ root height > 0.3× chiều cao chuẩn).
- Hoàn thành quãng đường/khoảng cách mục tiêu (ví dụ đi hết 5m mà không ngã).
- Tracking error dưới ngưỡng trong suốt episode (ví dụ MPJPE trung bình < X mm) — khắt khe hơn "không ngã".
- Với task thao tác: vật thể được đặt đúng vị trí đích trong dung sai cho phép.

### Jerk

```
vị trí   p(t)
vận tốc       v(t) = dp/dt
gia tốc       a(t) = dv/dt
jerk          j(t) = da/dt     (đạo hàm bậc 3 của vị trí theo thời gian)
```

Với dữ liệu rời rạc theo timestep Δt (trường hợp thực tế — log robot luôn là chuỗi rời rạc), dùng sai phân hữu hạn. Với chuỗi vị trí `p₀, p₁, p₂, p₃, ...` cách nhau Δt:

```
v_k = (p_{k+1} − p_k) / Δt                                (sai phân tiến, bậc 1)
a_k = (v_{k+1} − v_k) / Δt = (p_{k+2} − 2p_{k+1} + p_k) / Δt²
j_k = (a_{k+1} − a_k) / Δt = (p_{k+3} − 3p_{k+2} + 3p_{k+1} − p_k) / Δt³
```

Báo cáo dưới dạng **mean squared jerk** hoặc **RMS jerk** trên toàn trajectory:

```
RMS jerk = √[ (1/T) Σ_{k} j_k² ]
```

đơn vị: (đơn-vị-vị-trí)/s³ — ví dụ m/s³ nếu vị trí tính bằng mét.

### Root trajectory error

- **Root position error**: `‖P_root − G_root‖₂` — công thức giống MPJPE nhưng chỉ áp dụng cho 1 điểm duy nhất (root — thường là pelvis hoặc torso).
- **Root orientation error**: sai lệch hướng, đo bằng góc geodesic giữa 2 quaternion `q_robot` và `q_ref`:

```
θ_error = 2 · arccos(|⟨q_robot, q_ref⟩|)
```

trong đó `⟨q_robot, q_ref⟩` là tích vô hướng (dot product) của 2 quaternion (đã chuẩn hoá về đơn vị), kết quả `θ_error` tính bằng radian rồi đổi ra độ nếu muốn báo cáo dễ đọc hơn.

## ⚙️ Cơ chế hoạt động — từng bước

```
┌──────────────── Vòng lặp đánh giá 1 episode ─────────────────┐
│                                                                 │
│  For mỗi episode e trong tổng M episode thử:                  │
│    1. Chạy policy từ đầu tới hết episode (hoặc tới khi bị      │
│       terminate sớm — ngã, hết thời gian, ...)                 │
│    2. Ghi log toàn bộ vị trí khớp + root pose theo từng frame   │
│    3. Kiểm tra tiêu chí thành công đã định nghĩa TRƯỚC          │
│       (ví dụ: root height > ngưỡng suốt episode?)               │
│         → success[e] = 1 nếu đạt, 0 nếu không                  │
│    4. Tính jerk theo chuỗi vị trí đã log (công thức sai phân)   │
│         → jerk[e] = RMS jerk của episode e                     │
│    5. Tính root position error + root orientation error mỗi    │
│       frame, trung bình theo episode → root_err[e]              │
│                                                                 │
│  Sau khi chạy hết M episode:                                   │
│    Success rate = (Σ success[e]) / M × 100%                    │
│    Jerk trung bình = mean(jerk[e] trên các episode — thường     │
│       chỉ tính trên episode THÀNH CÔNG, ghi rõ nếu khác)         │
│    Root error trung bình = mean(root_err[e])                    │
└─────────────────────────────────────────────────────────────┘
```

Điểm dễ bị bỏ sót: bước 4 (jerk) và bước 5 (root error) nên được tính **kể cả trên các episode thất bại** để chẩn đoán nguyên nhân thất bại (ví dụ root error tăng đột biến ngay trước khi ngã là dấu hiệu chẩn đoán rất hữu ích), nhưng khi **báo cáo tổng kết**, cần nói rõ có gộp cả episode thất bại vào trung bình jerk/root error hay không — gộp lẫn không rõ ràng sẽ làm số liệu khó so sánh.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số liệu tự chọn để dễ hình dung.)*

### Ví dụ 1 — Success rate

Giả sử bạn chạy 10 episode với 10 seed khác nhau, tiêu chí thành công = "root height luôn > 0.5m suốt episode 10 giây":

| Episode | Root height thấp nhất trong episode (m) | Thành công? |
|---|---|---|
| 1 | 0.62 | ✅ |
| 2 | 0.58 | ✅ |
| 3 | 0.31 | ❌ (ngã) |
| 4 | 0.55 | ✅ |
| 5 | 0.60 | ✅ |
| 6 | 0.49 | ❌ (dưới ngưỡng 0.5) |
| 7 | 0.57 | ✅ |
| 8 | 0.63 | ✅ |
| 9 | 0.20 | ❌ (ngã hẳn) |
| 10 | 0.59 | ✅ |

```
Success rate = 7/10 × 100% = 70%
```

### Ví dụ 2 — Jerk (tính tay từ sai phân)

Giả sử vị trí root theo trục x, lấy mẫu mỗi Δt = 0.1s, có 5 điểm liên tiếp (đơn vị mét): `p₀=0.00, p₁=0.01, p₂=0.03, p₃=0.04, p₄=0.045`.

**Bước 1 — vận tốc** `v_k = (p_{k+1}-p_k)/Δt`:
- v₀ = (0.01-0.00)/0.1 = 0.10 m/s
- v₁ = (0.03-0.01)/0.1 = 0.20 m/s
- v₂ = (0.04-0.03)/0.1 = 0.10 m/s
- v₃ = (0.045-0.04)/0.1 = 0.05 m/s

**Bước 2 — gia tốc** `a_k = (v_{k+1}-v_k)/Δt`:
- a₀ = (0.20-0.10)/0.1 = 1.00 m/s²
- a₁ = (0.10-0.20)/0.1 = -1.00 m/s²
- a₂ = (0.05-0.10)/0.1 = -0.50 m/s²

**Bước 3 — jerk** `j_k = (a_{k+1}-a_k)/Δt`:
- j₀ = (-1.00-1.00)/0.1 = -20.00 m/s³
- j₁ = (-0.50-(-1.00))/0.1 = 5.00 m/s³

**Bước 4 — RMS jerk** trên 2 mẫu:

```
RMS jerk = √[ ((-20.00)² + (5.00)²) / 2 ] = √[ (400 + 25)/2 ] = √212.5 ≈ 14.6 m/s³
```

So sánh: nếu chuyển động mượt hơn nhiều, ví dụ gia tốc gần như hằng số (a₀≈a₁≈a₂≈-0.5 m/s² đều nhau), jerk sẽ gần bằng 0 — con số 14.6 m/s³ ở trên phản ánh một cú "đổi gia tốc đột ngột" (từ +1.0 xuống -1.0 m/s² chỉ trong 0.1s) — đây chính xác là loại chuyển động "giật cục" mà motor thật khó theo kịp.

### Ví dụ 3 — Root trajectory error

Root robot tại 1 frame: vị trí (0.12, 0.02, 0.58)m, quaternion `q_robot = (0.999, 0.02, 0.01, 0.0)` (đã chuẩn hoá gần đúng). Root tham chiếu: vị trí (0.10, 0.00, 0.60)m, quaternion `q_ref = (1.0, 0.0, 0.0, 0.0)` (không xoay).

- **Root position error**: `√[(0.12-0.10)² + (0.02-0.00)² + (0.58-0.60)²] = √[0.0004+0.0004+0.0004] = √0.0012 ≈ 0.0346m ≈ 34.6mm`.
- **Root orientation error**: dot product `⟨q_robot,q_ref⟩ ≈ 0.999×1.0 + 0.02×0 + 0.01×0 + 0×0 = 0.999`. `θ_error = 2·arccos(0.999) ≈ 2×2.56° ≈ 5.1°` (arccos(0.999) tính bằng radian rồi đổi độ: arccos(0.999) ≈ 0.0447 rad ≈ 2.56°, nhân 2 ≈ 5.11°).

Vậy root lệch khoảng 34.6mm vị trí và ~5.1° hướng — nếu MPJPE toàn thân đo được ở cùng frame chỉ là ~15mm, đây là dấu hiệu **root đang lệch nhiều hơn phần còn lại của cơ thể** — chính là hiện tượng "trôi toàn cục" (drift) đã nhắc ở bài MPJPE.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Success rate | Jerk | Root trajectory error | MPJPE (bài riêng) |
|---|---|---|---|---|
| Kiểu số liệu | Nhị phân → tỉ lệ % | Liên tục, đơn vị vật lý/s³ | Liên tục (m và độ) | Liên tục (mm) |
| Đo cái gì | Có hoàn thành hay không | Độ mượt của chuyển động | Sai lệch riêng của "nền" cơ thể | Sai lệch trung bình toàn bộ khớp |
| Rủi ro lớn nhất khi đọc một mình | Ẩn giấu mức độ chính xác (chỉ biết đạt/không đạt) | Có thể đánh lừa nếu policy đứng yên (jerk=0 nhưng vô dụng) | Không nói gì về dáng tư thế các khớp khác | Không nói gì về việc có ngã hay không |
| Khi nào là chỉ số quan trọng nhất | Task rời rạc rõ ràng (đi hết quãng đường, đặt đúng vật) | Chuẩn bị deploy lên robot thật (giới hạn động cơ) | Nghi ngờ robot "trôi" dù dáng đi đúng | So sánh tổng quát xuyên paper |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "jerk càng thấp càng chứng tỏ policy càng giỏi".**
   Vì sao sai: một policy học được cách đứng yên tuyệt đối (không tương tác gì) sẽ có jerk gần như 0 nhưng hoàn toàn không hoàn thành nhiệm vụ tracking. Jerk thấp chỉ có ý nghĩa tích cực khi đi kèm success rate cao và MPJPE thấp — nếu không, nó có thể là dấu hiệu policy đang "trốn" nhiệm vụ.
   Hiểu đúng: luôn đọc jerk trong bối cảnh "trên các episode đã thành công", so sánh jerk giữa các policy **cùng mức success rate/MPJPE tương đương**, không so sánh jerk một mình.

2. **Hiểu nhầm: định nghĩa "thành công" có thể linh hoạt điều chỉnh sau khi thấy kết quả để chọn ngưỡng có lợi nhất.**
   Vì sao sai: đây chính là một dạng cherry-picking ở cấp thiết kế metric (xem thêm bài `BAI-GIANG-fair-comparison.md`) — nếu bạn chạy thử, thấy 70% episode có root height thấp nhất là 0.48m, rồi mới hạ ngưỡng thành công xuống 0.45m để "vớt" thêm các episode đó, con số success rate không còn ý nghĩa khách quan.
   Hiểu đúng: viết ngưỡng + tiêu chí thành công thành văn bản **trước** khi chạy loạt thí nghiệm chính thức (pre-registration), chỉ thay đổi nếu phát hiện lỗi logic thực sự trong định nghĩa, và phải ghi chú rõ nếu có thay đổi.

## 🏗️ Ví dụ minh hoạ trong dự án này

Khi bạn tự huấn luyện policy tracking chuyển động ở `04-imitation-learning-rl/` (ví dụ theo pipeline DeepMimic/AMP), một bộ đánh giá tối thiểu nên gồm:

- **Success rate** trên ≥3 seed, với tiêu chí cố định trước (ví dụ "root height > 0.4× chiều cao chuẩn suốt episode" cho robot G1/H1).
- **RMS jerk** của root và của các khớp chân — đặc biệt quan trọng nếu bạn có kế hoạch chuyển tiếp sang `08-real-robot-deployment/`, vì động cơ thật (đặc biệt actuator của Unitree G1) có giới hạn tốc độ đáp ứng, chuyển động giật cục trong sim có thể không thực thi được trên phần cứng thật hoặc gây hỏng động cơ.
- **Root trajectory error** tách riêng khỏi MPJPE toàn thân — nếu bạn thấy root error tăng mạnh trong khi MPJPE-L (cục bộ) vẫn thấp, đây là dấu hiệu cụ thể để quay lại kiểm tra reward liên quan tới ổn định gốc thân (root stability reward) trong công thức huấn luyện.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **Success rate + jerk tiếp tục là cặp metric chuẩn trong các paper loco-manipulation 2025.** Theo kết quả tra cứu (WebSearch tháng 9/2026), một nghiên cứu 2025 về "Robust Visuomotor Control for Humanoid Loco-Manipulation Using Hybrid Reinforcement Learning" (MDPI, 2025, DOI liên quan PMC12292580) báo cáo success rate tổng thể 83% cho các task mang vác (load carrying) và mở cửa (door opening), với success rate cho các task loco-manipulation khó hơn đạt khoảng 85% — cho thấy success rate vẫn là con số "đầu tiên" được nêu trong abstract của các paper ứng dụng, đúng vai trò "chỉ số nhị phân dễ hiểu nhất" đã mô tả ở trên.
2. **Jerk trong bối cảnh so sánh policy học được (learned) với teleoperation** có một phát hiện thú vị: theo một benchmark robot manipulation 2025 (arXiv:2507.00435 — *"Where Robotic Manipulation Meets Structured and Scalable Evaluation"*), các policy học được có Cartesian jerk thấp hơn ở 26/48 so sánh và joint jerk thấp hơn ở 21/48 so sánh so với teleoperation của con người — nhưng nhóm tác giả lưu ý đây phản ánh việc chuyển động được "làm mượt" bởi cơ chế sinh hành động đã học, chứ không hẳn phản ánh chất lượng điều khiển tốt hơn về bản chất (teleoperation có nhiều điều chỉnh rời rạc do con người can thiệp trực tiếp, làm tăng jerk một cách "tự nhiên" chứ không phải lỗi). Đây là một lời nhắc quan trọng: jerk thấp hơn không tự động nghĩa là "giỏi hơn" — phải xét ngữ cảnh tạo ra chuyển động đó.
3. **Root trajectory error / lỗi "trôi" (drift) vẫn là kiến thức nền tảng ổn định** — chưa tìm thấy biến thể mới thay thế đáng kể trong 2024-2026 qua các lượt tra cứu đã chạy; phần mở rộng gần đây chủ yếu là các paper báo cáo root error tách riêng theo trục (x/y ngang so với y/z chiều cao) khi phân tích thất bại cụ thể của policy, chứ không phải một công thức đo mới.

## ❓ Câu hỏi tự kiểm tra

1. Cho 8 episode, 5 episode thành công theo tiêu chí "không ngã". Tính success rate.
<details><summary>Gợi ý đáp án</summary>5/8 × 100% = 62.5%.</details>

2. Vì sao một policy có jerk = 0 tuyệt đối không chắc là một policy tốt?
<details><summary>Gợi ý đáp án</summary>Vì đứng yên hoàn toàn (không di chuyển gì) cũng cho jerk=0 — jerk chỉ có ý nghĩa khi đọc cùng success rate/MPJPE để xác nhận policy có thực sự hoàn thành nhiệm vụ.</details>

3. Root position error lớn trong khi MPJPE-L (cục bộ) nhỏ gợi ý điều gì về hành vi của robot?
<details><summary>Gợi ý đáp án</summary>Robot đang "trôi" (drift) toàn cục — dáng tư thế các khớp tương đối với root là đúng, nhưng bản thân root lệch khỏi vị trí/hướng tham chiếu.</details>

4. Tại sao không nên định nghĩa lại tiêu chí thành công sau khi đã thấy kết quả thí nghiệm?
<details><summary>Gợi ý đáp án</summary>Vì đó là một dạng cherry-picking ở cấp thiết kế metric — chọn ngưỡng có lợi cho kết quả đã thấy làm mất tính khách quan, khiến số liệu không còn đáng tin và không thể so sánh công bằng với công trình khác.</details>

5. Từ chuỗi vị trí `p₀=0, p₁=0.02, p₂=0.05, p₃=0.09` (Δt=0.1s), tính vận tốc `v₀, v₁, v₂` và gia tốc `a₀, a₁`.
<details><summary>Gợi ý đáp án</summary>v₀=(0.02-0)/0.1=0.2, v₁=(0.05-0.02)/0.1=0.3, v₂=(0.09-0.05)/0.1=0.4 (m/s). a₀=(0.3-0.2)/0.1=1.0, a₁=(0.4-0.3)/0.1=1.0 (m/s²) — gia tốc hằng số nên jerk ở đoạn này = 0.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** dùng chuỗi vị trí ở câu hỏi tự kiểm tra số 5 (`p₀=0, p₁=0.02, p₂=0.05, p₃=0.09`, Δt=0.1s), tính tiếp jerk `j₀` từ 2 gia tốc `a₀, a₁` vừa tính. So sánh với ví dụ tính tay trong bài (jerk₀ ≈ -20 m/s³) — giải thích tại sao chuỗi này cho jerk khác hẳn (gợi ý: nhìn vào việc gia tốc ở đây có đổi dấu đột ngột hay không).
2. **Đọc log thật:** nếu bạn có một checkpoint policy đã huấn luyện ở `04-imitation-learning-rl/`, xuất log vị trí root theo 20-30 frame liên tiếp của 1 episode, tự viết một đoạn script ngắn (numpy `np.diff` 3 lần liên tiếp chia cho Δt³) để tính RMS jerk thực tế, rồi so sánh giữa episode thành công và episode thất bại (nếu có) của cùng policy đó.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

Success rate, jerk, và root trajectory error là 3 metric bổ sung cho MPJPE, mỗi cái lấp một khoảng trống riêng: success rate trả lời câu hỏi nhị phân "có hoàn thành hay không" dựa trên một tiêu chí phải khai báo trước khi thí nghiệm; jerk (đạo hàm bậc 3 của vị trí, tính bằng sai phân hữu hạn bậc 3 trong thực hành) đo độ mượt của chuyển động — quan trọng cho khả năng triển khai lên phần cứng thật nhưng chỉ có ý nghĩa khi đọc cùng success rate (một policy đứng yên cũng có jerk=0 mà vô dụng); và root trajectory error (vị trí + hướng của root, tách biệt khỏi các khớp khác) phát hiện hiện tượng "trôi toàn cục" mà MPJPE cục bộ không lộ ra được. Không metric nào trong 4 con số (bao gồm cả MPJPE) nên được đọc một mình — chúng bổ sung cho nhau và chỉ khi kết hợp cả bộ mới cho một bức tranh đáng tin cậy về chất lượng policy.
