# Bài giảng: SOMA-retargeter — kiến trúc và pipeline

*(Thuộc mảng: Motion Retargeting)*

## 🎯 Mục tiêu bài học

- Mô tả chính xác 5 bước pipeline của SOMA-retargeter: đọc BVH → scale khớp → giải IK trên GPU (Newton + NVIDIA Warp) → ổn định tiếp xúc/giới hạn khớp → xuất CSV.
- Giải thích được vì sao xử lý *batch trên GPU* hợp lý cho bài toán sinh dữ liệu huấn luyện quy mô lớn, khác với lý do GMR chọn CPU/streaming.
- Tính tay được ví dụ so sánh thông lượng (throughput) giữa xử lý tuần tự kiểu GMR và xử lý song song hàng loạt kiểu SOMA-retargeter cho một tập dữ liệu lớn.
- Liệt kê được 5 robot có cấu hình sẵn và ý nghĩa của "chế độ headless".
- Nêu được vì sao output của SOMA-retargeter là "thuần kinematic" và hệ quả bắt buộc kiểm chứng trước khi lên robot thật.
- So sánh được SOMA-retargeter với GMR (đã học) theo đúng tiêu chí kiến trúc, không chỉ tên gọi.

## 🧭 Vì sao cần học cái này? (bối cảnh)

GMR (bài trước) được thiết kế để streaming real-time trên CPU — phù hợp teleoperation hoặc xử lý một file đơn lẻ. Nhưng dự án humanoid VLA hiện đại (GR00T-SONIC, các pipeline RL/imitation learning ở `04-imitation-learning-rl/`) cần **hàng nghìn đến hàng chục nghìn clip chuyển động đã retarget** làm dữ liệu huấn luyện — ví dụ tập SEED (Skeletal Everyday Embodiment Dataset) retarget sang G1 dùng chính công cụ này. Xử lý tuần tự từng frame một, dù nhanh tới đâu trên CPU, vẫn là **nút thắt cổ chai (bottleneck)** khi nhân với số lượng clip lớn. SOMA-retargeter (NVIDIA) giải quyết đúng bài toán quy mô này bằng cách đưa IK lên GPU và xử lý theo lô. Học bài này để hiểu công cụ thứ hai mà dự án dùng, và biết khi nào nên chọn GMR, khi nào nên chọn SOMA-retargeter.

## 🧠 Trực giác

### Góc nhìn 1: Nhà bếp gọi món lẻ (à la carte) so với dây chuyền sản xuất suất ăn công nghiệp

GMR giống một đầu bếp nấu từng món theo yêu cầu khách ngay tại chỗ (streaming, real-time) — nhanh cho một suất, linh hoạt, nhưng nếu phải nấu 10.000 suất giống hệt quy trình thì việc "nấu từng suất một, tuần tự" sẽ rất chậm dù đầu bếp có giỏi tới đâu. SOMA-retargeter giống một dây chuyền sản xuất suất ăn công nghiệp: chuẩn bị sẵn quy trình 5 bước cố định, chạy hàng trăm "trạm" xử lý song song (GPU cores) cùng lúc trên hàng loạt nguyên liệu đầu vào (các file BVH) — không tối ưu cho việc phục vụ một khách ngay lập tức, nhưng cực kỳ hiệu quả khi cần sản lượng lớn.

**Giới hạn của loại suy này:** một dây chuyền sản xuất suất ăn thường không "linh hoạt hoá" theo từng đơn hàng riêng lẻ; trong khi SOMA-retargeter vẫn xử lý *độc lập* từng file BVH (không có sự phụ thuộc chéo giữa các clip) — điểm song song hoá nằm ở việc xử lý *nhiều frame/nhiều clip cùng lúc trên GPU*, không phải ở việc "trộn lẫn" thông tin giữa các suất ăn khác nhau như một dây chuyền thật có thể ngụ ý.

### Góc nhìn 2: Biên dịch (compile) một lần cho cả dự án so với chạy REPL từng dòng lệnh

GMR giống một REPL (Read-Eval-Print Loop) — gõ một lệnh (một frame), nhận kết quả ngay, lặp lại liên tục, tương tác được. SOMA-retargeter giống việc biên dịch (compile) toàn bộ một dự án phần mềm lớn: bạn không cần thấy kết quả ngay lập tức cho từng dòng, mà chấp nhận đợi một khoảng thời gian để trình biên dịch xử lý *toàn bộ* các file cùng lúc, tận dụng tối đa các lõi xử lý song song, rồi nhận về toàn bộ output đã build xong.

**Giới hạn của loại suy này:** biên dịch phần mềm có các bước phụ thuộc lẫn nhau nghiêm ngặt (module A phải build trước module B nếu B import A); trong khi các clip chuyển động trong batch của SOMA-retargeter **độc lập hoàn toàn** với nhau — không có thứ tự phụ thuộc nào giữa việc retarget clip đi bộ và clip nhảy múa, nên mức độ song song hoá lý thuyết của SOMA-retargeter còn cao hơn build hệ thống phần mềm điển hình.

## 📐 Định nghĩa chính xác

**SOMA-retargeter** là thư viện mã nguồn mở của NVIDIA [GitHub](https://github.com/NVIDIA/soma-retargeter), giấy phép Apache-2.0, chuyển đổi animation BVH theo định dạng **SOMA-skeleton** (một chuẩn hoá tỷ lệ cơ thể thống nhất, xem thêm dataset SEED liên quan) thành chuyển động khớp robot, thực hiện trên **GPU** thông qua hai thư viện của NVIDIA:

- **Newton** — thư viện vật lý/robotics dùng để biểu diễn và giải các bài toán ràng buộc động học/tối ưu.
- **NVIDIA Warp** — framework tính toán song song hiệu năng cao, cho phép viết kernel bằng Python nhưng biên dịch (JIT) để chạy trực tiếp trên GPU.

Về mặt hình thức toán học, bước giải IK của SOMA-retargeter cùng thuộc lớp bài toán tối ưu hình học per-frame như GMR (tìm cấu hình khớp `θ` khớp với target vị trí/hướng đã scale), nhưng khác biệt kiến trúc mấu chốt là: thay vì giải **tuần tự từng frame trên một luồng CPU**, SOMA-retargeter **rải bài toán của hàng trăm/hàng nghìn frame (từ nhiều clip khác nhau) thành các kernel Warp chạy song song trên hàng nghìn lõi GPU cùng lúc** — biến bài toán "giải N lần tuần tự" thành "giải N lần song song", giảm tổng thời gian tường (wall-clock time) cho khối lượng lớn dù thời gian giải *một* frame đơn lẻ trên GPU có thể không nhanh hơn CPU.

## ⚙️ Cơ chế hoạt động — từng bước

Theo README chính thức, pipeline gồm đúng 5 bước:

```text
┌────────────────────────────────────────────────────────────┐
│ BƯỚC 1 — Đọc BVH                                            │
│  Nạp file animation định dạng SOMA-skeleton                 │
│  (tỷ lệ cơ thể đã chuẩn hoá theo chuẩn SOMA)                 │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ BƯỚC 2 — Scale khớp                                         │
│  Retarget khớp theo tỷ lệ người → tỷ lệ robot đích           │
│  (cùng nguyên lý skeleton mapping + scale đã học,            │
│   nhưng cấu hình riêng theo từng robot đích)                 │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ BƯỚC 3 — Giải IK từng frame (GPU)                            │
│  Newton (bài toán ràng buộc/động học)                        │
│  + NVIDIA Warp (kernel song song hoá trên GPU)               │
│  → hàng loạt frame được giải ĐỒNG THỜI, không tuần tự        │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ BƯỚC 4 — Ổn định tiếp xúc + giới hạn khớp                   │
│  Foot contact stabilization + joint limit clamping           │
│  (áp lên kết quả IK, cùng khái niệm đã học ở 2 bài trước)    │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ BƯỚC 5 — Xuất CSV                                            │
│  root pose (vị trí + góc xoay gốc robot)                     │
│  + giá trị các khớp actuated                                 │
└──────────────────────────────────────────────────────────────┘
```

Hai chế độ vận hành:

- **Interactive viewer** — GUI hiển thị đồng thời chuyển động nguồn SOMA và robot đã retarget trong viewport 3D, dùng để nạp/kiểm tra/lưu từng file riêng lẻ — phù hợp debug hoặc kiểm tra chất lượng một clip cụ thể.
- **Headless mode** — chuyển đổi hàng loạt cả một thư mục chuyển động mà không cần giao diện đồ hoạ — đây chính là chế độ dùng cho việc sinh dữ liệu huấn luyện quy mô lớn (batch), là lý do chính khiến công cụ này tồn tại song song với GMR trong dự án.

Robot đã có cấu hình sẵn (theo README chính thức): `unitree_g1` (mặc định), `unitree_h2`, `booster_t1`, `agibot_x2ultra`, `agibot-a3t3`.

**Lưu ý bắt buộc từ tài liệu chính thức:** SOMA-retargeter hiện đang ở giai đoạn *active beta*, và chuyển động sinh ra là **thuần kinematic** (chỉ đảm bảo đúng động học — vị trí/góc khớp — chứ không đảm bảo tính khả thi động lực học như lực, mô-men, cân bằng) — cần kiểm chứng trong mô phỏng (`05-simulation-mujoco-isaaclab/`) trước khi dùng cho huấn luyện policy hoặc đưa lên robot thật.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số tự chọn để dễ hình dung quy mô lợi ích song song hoá — không phải benchmark thật đã công bố của SOMA-retargeter, vì tài liệu chính thức chưa công bố số FPS/throughput cụ thể tại thời điểm tra cứu)*

Giả sử ta cần retarget một tập dữ liệu gồm **N = 10.000 clip**, mỗi clip trung bình dài **T = 5 giây** ở **30 FPS nguồn** → mỗi clip có `5 × 30 = 150` frame. Tổng số frame cần xử lý:

```text
Tổng frame = N × 150 = 10.000 × 150 = 1.500.000 frame
```

**Kịch bản A — xử lý tuần tự kiểu GMR** (giả sử retarget được 50 FPS trung bình, một con số trong dải 35–70 FPS đã học ở bài GMR, xử lý *tuần tự từng frame*):

```text
thời gian = 1.500.000 / 50 = 30.000 giây = 500 phút ≈ 8 giờ 20 phút
```

**Kịch bản B — xử lý song song theo batch kiểu SOMA-retargeter** (giả sử GPU xử lý đồng thời `B = 512` frame một lượt — một con số minh hoạ hợp lý cho một GPU workstation hiện đại chạy Warp kernel — và mỗi lượt batch mất `Δt_batch = 0.05 giây`, tức tương đương hiệu năng `512/0.05 = 10.240` frame/giây):

```text
số lượt batch cần = ⌈1.500.000 / 512⌉ = ⌈2929.7⌉ = 2.930 lượt
thời gian = 2.930 × 0.05 = 146.5 giây ≈ 2.44 phút
```

**So sánh tốc độ:**

```text
tăng tốc ≈ 30.000 / 146.5 ≈ 204.8 lần
```

**Ý nghĩa:** đây là ví dụ minh hoạ *tại sao* GPU-batch đáng giá cho bài toán sinh dữ liệu quy mô lớn — dù giả định về batch size/thời gian mỗi batch chỉ là số tự chọn hợp lý, cấu trúc phép tính cho thấy đúng cơ chế thật: lợi ích song song hoá **tăng tuyến tính theo batch size** cho tới khi phần cứng GPU bão hoà, trong khi xử lý tuần tự luôn bị giới hạn bởi FPS của một luồng xử lý duy nhất, bất kể có bao nhiêu clip cần xử lý.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | SOMA-retargeter | GMR (đã học) |
|---|---|---|
| Phần cứng | GPU (Newton + NVIDIA Warp) | CPU |
| Chế độ xử lý | Batch hàng loạt, có headless mode | Streaming, per-frame, real-time |
| Định dạng input | BVH chuẩn SOMA-skeleton | SMPL-X, BVH, FBX, video, streaming trực tiếp |
| Correspondence | Sparse, cấu hình theo từng robot (5 robot có sẵn) | Sparse, cấu hình theo `ik_match_table` JSON |
| Dùng cho teleoperation trực tiếp? | Không phù hợp (không phải streaming) | Có (dùng cho TWIST) |
| Dùng cho sinh dữ liệu huấn luyện quy mô lớn? | Rất phù hợp (batch GPU, headless) | Khả thi nhưng chậm hơn nhiều ở quy mô lớn (tuần tự) |
| Output | CSV: root pose + joint values | Root pose + joint values (trực tiếp trong pipeline Python/MuJoCo) |
| Trạng thái phát triển | Active beta (theo README chính thức) | Đã công bố paper, chấp nhận ICRA 2026 |

**Khi nào dùng cái nào:** nếu mục tiêu là **một** hoặc vài clip cần xem/dùng ngay (demo, teleoperation, kiểm tra nhanh một ý tưởng), GMR là lựa chọn tự nhiên vì không cần hạ tầng GPU và cho phản hồi tức thời. Nếu mục tiêu là **sinh nhãn huấn luyện quy mô lớn** (hàng nghìn clip AMASS/LAFAN1/SEED cần retarget một lần để dùng lặp lại cho nhiều lần huấn luyện RL sau này), SOMA-retargeter tận dụng GPU tốt hơn nhiều — đúng như cách tập SEED được retarget sang G1 bằng chính công cụ này. Hai công cụ vì thế không cạnh tranh trực tiếp mà **bổ sung cho nhau theo hai chế độ sử dụng khác nhau** trong cùng một dự án.

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "SOMA-retargeter chính xác hơn GMR vì dùng GPU/vật lý (Newton)".** Vì sao sai: tên thư viện "Newton" gợi ý tới động lực học Newton, nhưng theo tài liệu chính thức, output của SOMA-retargeter vẫn là **thuần kinematic** — Newton ở đây được dùng như một khung giải bài toán ràng buộc/tối ưu hoá (constraint-solving framework), không có nghĩa là pipeline đã mô phỏng đầy đủ lực/mô-men/cân bằng động lực học. Việc dùng GPU quyết định *tốc độ xử lý hàng loạt*, không tự động quyết định *độ chính xác vật lý* của kết quả. **Hiểu đúng:** cả GMR và SOMA-retargeter đều cần bước kiểm chứng vật lý riêng (mô phỏng ở `05-simulation-mujoco-isaaclab/`) trước khi tin tưởng chuyển động khả thi trên robot thật.
2. **Hiểu nhầm: "Batch GPU luôn nhanh hơn CPU streaming trong mọi trường hợp".** Vì sao sai: như ví dụ tính tay ở trên cho thấy, lợi ích song song hoá chỉ thể hiện rõ khi **số lượng frame/clip đủ lớn** để lấp đầy batch GPU; với một clip đơn lẻ, chi phí khởi tạo GPU/copy dữ liệu có thể khiến SOMA-retargeter chậm hơn GMR chạy trực tiếp trên CPU cho đúng clip đó. **Hiểu đúng:** lựa chọn công cụ phải dựa vào *quy mô bài toán thực tế* (một clip vs. hàng nghìn clip), không phải giả định "GPU luôn thắng".

## 🏗️ Ví dụ minh hoạ trong dự án này

Trong pipeline dữ liệu của dự án, SOMA-retargeter đóng vai trò xử lý hàng loạt cho các nguồn dữ liệu lớn trước khi đưa vào huấn luyện:

```text
LAFAN1 / SEED / các thư mục BVH chuẩn SOMA-skeleton lớn
                    │
                    ▼
     SOMA-retargeter — headless mode (GPU batch)
                    │
        (5 robot có sẵn: G1, H2, Booster T1,
         AGIBot X2Ultra/A3T3)
                    │
                    ▼
       CSV hàng loạt: root pose + joint values
                    │
                    ▼
  Kiểm chứng khả thi trong mô phỏng (05-simulation)
                    │
                    ▼
  Dữ liệu tham chiếu cho huấn luyện RL/imitation (04, 01)
```

Đây chính là lý do README của thư mục `02-motion-retargeting/` liệt kê SOMA-retargeter là "công cụ thứ hai, mentor chỉ định" — không thay thế GMR, mà đảm nhiệm phần việc mà GMR (thiết kế cho streaming) không tối ưu: chuẩn bị dữ liệu huấn luyện ở quy mô lớn trước khi vòng lặp RL/imitation learning bắt đầu.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **SOMA-retargeter vẫn đang trong giai đoạn active beta, phát triển liên tục.** Theo lịch sử phát triển công khai trên GitHub (ví dụ pull request "Soma Retargeter Release v0.2.0"), công cụ này đang tiếp tục bổ sung tính năng (robot mới, cải thiện pipeline) — khác với GMR đã có paper học thuật hoàn chỉnh công bố tại ICRA 2026, SOMA-retargeter hiện chủ yếu được tài liệu hoá qua README/code trực tiếp, chưa có paper riêng mô tả đầy đủ thuật toán IK-trên-GPU cụ thể tại thời điểm tra cứu (09/2026). [GitHub](https://github.com/NVIDIA/soma-retargeter) · [PR #21](https://github.com/NVIDIA/soma-retargeter/pull/21)
2. **Tập dữ liệu SEED xác nhận vai trò thực tế của SOMA-retargeter trong việc sinh dữ liệu huấn luyện quy mô lớn.** SEED (Skeletal Everyday Embodiment Dataset) là bộ sưu tập chuyển động người quy mô lớn trên skeleton tỷ lệ đồng nhất SOMA, và theo tài liệu liên quan, dữ liệu chuyển động G1 trong SEED được retarget bằng chính SOMA-retargeter — một bằng chứng thực tế cho thấy công cụ này đã được dùng để sinh nhãn cho một dataset công khai, không chỉ là một pipeline nội bộ lý thuyết. [arXiv:2603.16858 — SOMA: Unifying Parametric Human Body Models](https://arxiv.org/pdf/2603.16858)
3. **Xu hướng chung — GPU-batch retargeting đang trở thành hạ tầng chuẩn cho data engine của humanoid RL.** OmniRetarget (arXiv:2509.26633) và UMR (arXiv:2609.02134, xem bài giảng riêng) đều nhắm vào bối cảnh tương tự: sinh một lượng lớn dữ liệu tham chiếu chất lượng cao để huấn luyện policy, cho thấy nhu cầu về các công cụ retargeting quy mô batch (như SOMA-retargeter) đang tăng song song với nhu cầu retargeting streaming (như GMR) — hai nhu cầu khác nhau của cùng một lĩnh vực đang phát triển độc lập chứ không hội tụ về một công cụ duy nhất.

## ❓ Câu hỏi tự kiểm tra

1. Liệt kê đúng 5 bước pipeline của SOMA-retargeter theo đúng thứ tự.
   <details><summary>Gợi ý đáp án</summary>(1) Đọc BVH, (2) Scale khớp, (3) Giải IK từng frame trên GPU (Newton + Warp), (4) Ổn định tiếp xúc + giới hạn khớp, (5) Xuất CSV.</details>
2. Vì sao "chế độ headless" quan trọng cho việc sinh dữ liệu huấn luyện quy mô lớn, còn "interactive viewer" thì không phù hợp cho việc đó?
   <details><summary>Gợi ý đáp án</summary>Headless bỏ qua giao diện đồ hoạ, cho phép xử lý tự động hàng loạt thư mục mà không cần người theo dõi từng file; interactive viewer yêu cầu tương tác thủ công từng file, không khả thi khi xử lý hàng nghìn clip.</details>
3. Trong ví dụ tính tay, nếu batch size tăng từ 512 lên 1024 (giả sử thời gian mỗi lượt batch không đổi `0.05s` — giả định lý tưởng phần cứng chưa bão hoà), tổng thời gian xử lý 1.500.000 frame thay đổi thế nào?
   <details><summary>Gợi ý đáp án</summary>Số lượt batch giảm còn `⌈1.500.000/1024⌉ = 1465` lượt → thời gian `1465 × 0.05 = 73.25s`, giảm gần một nửa so với `146.5s` — minh hoạ đúng cơ chế thời gian tỷ lệ nghịch với batch size khi phần cứng chưa bão hoà.</details>
4. Vì sao dùng tên thư viện "Newton" không có nghĩa là output của SOMA-retargeter đã đảm bảo tính khả thi động lực học?
   <details><summary>Gợi ý đáp án</summary>Vì Newton ở đây được dùng làm khung giải bài toán ràng buộc/tối ưu hình học (constraint solving), còn output chính thức được tài liệu mô tả là "thuần kinematic" — chưa xét lực, mô-men hay cân bằng động lực học thật, cần kiểm chứng riêng trong mô phỏng.</details>
5. Vì sao SOMA-retargeter và GMR được coi là "bổ sung cho nhau" thay vì "cạnh tranh trực tiếp" trong cùng một dự án?
   <details><summary>Gợi ý đáp án</summary>Vì chúng tối ưu cho hai chế độ sử dụng khác nhau: GMR cho streaming/real-time/một clip đơn lẻ (kể cả teleoperation), SOMA-retargeter cho batch GPU/quy mô lớn/headless (sinh dữ liệu huấn luyện) — không có tình huống nào cả hai cùng là lựa chọn tối ưu, nên dự án dùng cả hai tuỳ ngữ cảnh.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** với `N = 50.000` clip, mỗi clip `T = 3` giây ở `24 FPS`, batch size `B = 256`, mỗi lượt batch mất `0.03s`. Tính tổng số frame, số lượt batch cần, tổng thời gian xử lý, và so sánh với thời gian xử lý tuần tự giả định `55 FPS`.
2. **Đọc code/README thật:** clone `github.com/NVIDIA/soma-retargeter`, đọc kỹ mục cấu hình cho một trong 5 robot có sẵn (ví dụ `unitree_g1`), liệt kê các tham số scale/joint-limit cụ thể được cấu hình — so sánh cấu trúc file cấu hình này với `ik_match_table`/`human_scale_table` của GMR đã học ở bài Skeleton mapping, chỉ ra điểm giống và khác về cách hai công cụ biểu diễn correspondence.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

SOMA-retargeter là công cụ retargeting thứ hai của dự án, giải cùng loại bài toán hình học như GMR (scale + IK per-frame + ổn định tiếp xúc/giới hạn khớp) nhưng chọn kiến trúc hoàn toàn khác: xử lý theo lô (batch) trên GPU qua Newton và NVIDIA Warp, với chế độ headless cho phép convert hàng loạt thư mục BVH chuẩn SOMA-skeleton mà không cần giao diện — phù hợp sinh dữ liệu huấn luyện quy mô lớn (như tập SEED) thay vì streaming thời gian thực. Như ví dụ tính tay minh hoạ, lợi ích song song hoá GPU có thể mang lại tăng tốc hàng trăm lần khi số lượng frame đủ lớn, nhưng output vẫn thuần kinematic và cần kiểm chứng vật lý riêng trước khi dùng cho robot thật — đúng ràng buộc chung mà cả GMR lẫn các phương pháp GPU-batch hiện đại hơn (OmniRetarget, UMR) đều phải tuân theo.
