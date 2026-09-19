# Bài giảng: GMR — kiến trúc và pipeline

*(Thuộc mảng: Motion Retargeting)*

## 🎯 Mục tiêu bài học

- Mô tả được chính xác pipeline 4 bước của GMR: đọc pose người dạng dict → skeleton mapping/scale → giải multi-objective differential IK qua `mink`+MuJoCo → áp velocity limit.
- Giải thích được vì sao GMR chọn chạy trên CPU thay vì GPU, và đánh đổi gì khi làm vậy.
- Tính tay được: (a) một bước IK vi phân với velocity limit clamp cụ thể, (b) ngân sách thời gian một frame trong vòng điều khiển teleoperation, để thấy khi nào CPU laptop "theo kịp" và khi nào không.
- Phân biệt được GMR với SOMA-retargeter (CPU/streaming vs GPU/batch) và với UMR (sparse handcrafted correspondence vs learned dense correspondence).
- Nêu được ít nhất 2 hạn chế đã công bố của GMR và hướng khắc phục trong nghiên cứu 2025–2026.
- Định vị được đúng vai trò của GMR trong pipeline dự án: từ dữ liệu người (AMASS/LAFAN1/video) tới reference motion cho huấn luyện RL/teleoperation.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Các bài giảng trước đã xây xong nền tảng lý thuyết: vì sao không thể copy góc khớp, tư tưởng constraint preservation của Gleicher, cách skeleton mapping tạo target đã scale, và hai thuật toán IK per-frame (Jacobian/differential IK, FABRIK). GMR chính là nơi tất cả những lý thuyết đó được **lắp ráp thành một công cụ chạy được**, và là công cụ retargeting chính mà dự án này dùng (mentor chỉ định). Học bài này để hiểu: khi gõ một lệnh retarget một file AMASS sang Unitree G1, điều gì thực sự xảy ra bên trong, tại sao nó chạy được real-time trên CPU (không cần GPU), và những giới hạn nào của nó ảnh hưởng tới chất lượng dữ liệu huấn luyện ở các bước sau (`01-whole-body-control/`, `04-imitation-learning-rl/`).

## 🧠 Trực giác

### Góc nhìn 1: Một biên dịch viên đồng thời (simultaneous interpreter), không phải người dịch từng câu rồi mới nói

Một biên dịch viên đồng thời nghe một câu tiếng Anh và ngay lập tức nói ra tiếng Việt tương ứng, không đợi cả đoạn văn kết thúc, không "nhìn trước" câu tiếp theo, và không được sửa lại câu vừa nói khi đã nói ra. Đổi lại, họ phải nói với tốc độ theo kịp người nói — nếu chậm hơn, khán giả mất đồng bộ. GMR hoạt động đúng như vậy: xử lý **từng frame một cách độc lập, ngay khi frame đó tới**, không nhìn frame tương lai (khác với spacetime optimization của Gleicher, vốn nhìn cả đoạn), đổi lại tốc độ đủ nhanh để dùng trực tiếp trong streaming/teleoperation thời gian thực.

**Giới hạn của loại suy này:** biên dịch viên con người có khả năng "đoán trước" ngữ cảnh nhờ hiểu ngôn ngữ; GMR không có cơ chế dự đoán ngữ nghĩa nào — nó thuần tuý là một bộ giải toán tối ưu số học tại từng thời điểm. Ngoài ra biên dịch không có khái niệm "vận tốc tối đa của miệng"; GMR thì có — velocity limit là một ràng buộc cứng trong vòng lặp giải (xem mục Cơ chế).

### Góc nhìn 2: Một người thợ may đo ni tại chỗ, dùng thước dây có sẵn thay vì máy scan 3D

Skeleton mapping của GMR giống một thợ may kinh nghiệm: có sẵn một bảng số đo tỷ lệ cơ thể khách quen (config JSON: `human_scale_table`, `ik_match_table`) và dùng thước dây (Jacobian/QP) đo — chỉnh trực tiếp theo từng frame. Thợ may này không cần máy quét 3D đắt tiền (GPU, mạng nơ-ron) — chỉ cần bảng số đo đúng và thước dây tốt là đủ nhanh, đủ chính xác cho hầu hết trường hợp.

**Giới hạn của loại suy này:** thợ may có thể "nhìn" khách hàng và tự điều chỉnh khi số đo không khớp thực tế; GMR chỉ tuân theo đúng config đã định nghĩa trước (`ik_match_table`, `human_scale_table`) — nếu bảng số đo cấu hình sai cho một robot mới, GMR không có cơ chế "tự nhận ra" sai số ngữ nghĩa, mà chỉ tối ưu hoá theo đúng target đã cho, đúng hay sai.

## 📐 Định nghĩa chính xác

**GMR (General Motion Retargeting)** là công cụ retargeting mã nguồn mở (Yanjie Ze và cộng sự, chấp nhận ICRA 2026) [GitHub](https://github.com/YanjieZe/GMR), hiện thực hoá retargeting như một bài toán **multi-objective differential Inverse Kinematics** giải **per-frame, tuần tự, trên CPU**. Với mỗi frame `t`, input là một dict:

```text
pose_human(t) : (tên xương người) → (vị trí toàn cục ∈ ℝ³, quaternion toàn cục ∈ SO(3))
```

Sau bước skeleton mapping + scale (đã học ở bài trước — tạo ra `p_target`, `R_target` cho từng frame robot cần khớp tới), GMR giải bài toán IK ở dạng **quy hoạch toàn phương (QP)** tại mỗi bước lặp Newton–Raphson, dùng thư viện `mink` (xây trên MuJoCo làm mô hình động học):

```text
minimize_{Δθ}   Σᵢ wᵢᵖ ‖Jᵢᵖ Δθ − eᵢᵖ‖² + wᵢʳ ‖Jᵢʳ Δθ − eᵢʳ‖²
subject to      θ_min ≤ θ + Δθ ≤ θ_max
                −v_max·Δt ≤ Δθ ≤ v_max·Δt
```

trong đó `eᵢᵖ = p_target,i − p_current,i` là sai số vị trí task `i` (một body/frame robot), `eᵢʳ` là sai số hướng biểu diễn dạng vector log của quaternion sai lệch, `wᵢᵖ, wᵢʳ` là trọng số task lấy từ `ik_match_table` (đã thấy ở bài Skeleton mapping), và `v_max` là **velocity limit** — theo tài liệu chính thức, mặc định khoảng `3π rad/s` cho các khớp. Điểm mấu chốt định nghĩa GMR (khác với IK "thuần" ở bài trước): ràng buộc vận tốc được đưa **vào ngay trong QP của mỗi bước giải**, không phải một bước hậu xử lý riêng — đây chính là lý do bài `joint-limit-clamping-velocity-limiting` mô tả velocity limiting như một khái niệm riêng, còn ở đây ta thấy nó được *tích hợp* thế nào vào solver cụ thể của GMR.

## ⚙️ Cơ chế hoạt động — từng bước

```text
┌──────────────────────────────────────────────────────────────┐
│ INPUT: 1 frame pose người                                     │
│  dict{tên xương → (vị trí 3D toàn cục, quaternion toàn cục)}  │
│  Nguồn: SMPL-X (AMASS/OMOMO) | BVH (LAFAN1/Nokov/Xsens)       │
│         FBX (OptiTrack) | video (qua GVHMR) | streaming UDP   │
└───────────────────────────┬────────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ BƯỚC 1 — Skeleton mapping + scale (bài giảng trước)           │
│  human_scale_table + ik_match_table (JSON theo cặp format→robot)│
│  → p_target,i , R_target,i  cho từng robot body/frame i        │
└───────────────────────────┬────────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ BƯỚC 2 — Multi-objective differential IK (vòng lặp Newton)     │
│  while chưa hội tụ:                                            │
│    tính Jacobian Jᵢᵖ, Jᵢʳ tại θ hiện tại (qua MuJoCo)           │
│    tính sai số eᵢᵖ, eᵢʳ so với target                          │
│    giải QP: Δθ tối ưu, ràng buộc θ_min/max + velocity_limit·Δt │
│    θ ← θ + Δθ                                                  │
└───────────────────────────┬────────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ BƯỚC 3 — Velocity limit đã nằm trong QP ở bước 2 (không phải   │
│  bước hậu xử lý riêng) — khác với joint limit clamping "cứng"  │
│  có thể áp thêm ở tầng ngoài cho một số pipeline khác          │
└───────────────────────────┬────────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ OUTPUT: root pose robot (vị trí+quaternion) + góc khớp actuated│
│  → dùng làm reference cho WBC / RL policy training /           │
│    hiển thị trực tiếp qua MuJoCo viewer / TWIST teleoperation  │
└──────────────────────────────────────────────────────────────┘
```

Ba điểm cần nhấn mạnh về *kiến trúc*, không chỉ thuật toán IK nói chung (đã học ở bài Jacobian/differential IK):

1. **Multi-objective, không phải single-chain độc lập.** GMR giải đồng thời tất cả các task (tay trái, tay phải, chân trái, chân phải, thân, đầu...) trong **một QP duy nhất mỗi bước lặp**, không giải từng chain riêng rồi ghép lại — nhờ vậy các chain chia sẻ đúng một nghiệm `θ` nhất quán toàn thân (tránh xung đột giữa các target).
2. **Non-uniform local scaling.** Theo paper *Retargeting Matters* (Araujo, Ze, Xu, Wu, Liu, 2025/ICRA 2026), GMR không dùng một hệ số scale toàn thân mà scale cục bộ theo từng body/chain (đã thấy cụ thể ở bảng `human_scale_table` trong bài Skeleton mapping) rồi chạy qua **hai giai đoạn tối ưu** để giảm các artefact (foot sliding, self-penetration, motion phi vật lý) mà cách scale ngây thơ tạo ra.
3. **Toàn bộ trên CPU, không cần GPU.** Vì `mink`/MuJoCo giải QP kích thước nhỏ (chỉ vài chục biến khớp) bằng phương pháp giải tích cổ điển, không cần song song hoá GPU — đây là điểm khác biệt kiến trúc cốt lõi so với SOMA-retargeter (bài giảng tiếp theo).

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số tự chọn để dễ hình dung, dựa trên thông số thật đã kiểm chứng: velocity limit mặc định ≈ 3π rad/s, FPS đo được trên CPU thật theo README GMR)*

### Phần A — Velocity limit clamp trong một bước IK

Giả sử tại frame hiện tại, khớp khuỷu tay robot đang ở góc `θ = 0.20 rad`. Bộ giải IK (không tính ràng buộc vận tốc) đề xuất một bước cập nhật `Δθ_đề_xuất = 0.45 rad` để khớp mọi target vị trí/hướng ngay lập tức (ví dụ do dữ liệu nguồn có một chuyển động rất nhanh ở frame này, hoặc do nghiệm IK "nhảy" giữa hai cấu hình).

Velocity limit: `v_max = 3π rad/s ≈ 9.4248 rad/s`. Nếu retarget chạy ở `60 FPS` → `Δt = 1/60 s ≈ 0.016667 s`.

Giới hạn thay đổi góc tối đa cho phép trong một frame:

```text
Δθ_max = v_max · Δt = 9.4248 × 0.016667 ≈ 0.15708 rad  (≈ 9.0°)
```

Vì `Δθ_đề_xuất = 0.45 rad > Δθ_max = 0.157 rad`, QP sẽ **clamp** bước cập nhật về đúng biên:

```text
Δθ_thực_tế = 0.15708 rad
θ_mới = θ + Δθ_thực_tế = 0.20 + 0.15708 = 0.35708 rad
```

Sai số còn lại chưa khớp target: `0.45 − 0.15708 = 0.29292 rad` sẽ được giải tiếp ở (các) frame kế tiếp — nghĩa là khớp "đuổi theo" target trong nhiều frame liên tiếp thay vì nhảy tức thời. Số frame tối thiểu cần để bắt kịp hoàn toàn (giả sử mỗi frame vẫn cần đúng `Δθ_max` và target không đổi thêm):

```text
n ≈ ⌈0.45 / 0.15708⌉ = ⌈2.865⌉ = 3 frame
```

**Ý nghĩa:** velocity limit không làm "mất" chuyển động, mà **trải nó ra theo thời gian thực tế motor có thể theo kịp** — đây chính là cơ chế khiến GMR không bao giờ yêu cầu robot xoay khớp nhanh hơn giới hạn vật lý, đổi lại chuyển động rất nhanh trong dữ liệu gốc có thể bị "làm chậm/mượt" cục bộ so với bản gốc.

### Phần B — Ngân sách thời gian: laptop CPU có theo kịp vòng điều khiển teleoperation không?

Theo README chính thức của GMR, tốc độ đo được:

```text
AMD Threadripper 7960X (24 lõi): 60–70 FPS
Intel i9-13900K (24 lõi):        35–45 FPS
```

Giả sử một hệ teleoperation (xem `08-real-robot-deployment/`) cần vòng điều khiển chạy ở `50 Hz` → ngân sách thời gian tối đa mỗi frame:

```text
budget = 1/50 = 0.020 s = 20 ms
```

Thời gian xử lý mỗi frame trên từng CPU (lấy giá trị trung bình dải FPS):

```text
Threadripper: t ≈ 1/65 ≈ 0.01538 s = 15.4 ms   → 15.4 ms < 20 ms  → THEO KỊP
i9-13900K:    t ≈ 1/40 ≈ 0.02500 s = 25.0 ms   → 25.0 ms > 20 ms  → KHÔNG THEO KỊP
```

**Kết luận số học:** với đúng cùng một thuật toán GMR, việc có "theo kịp" vòng điều khiển 50 Hz cho teleoperation real-time hay không phụ thuộc trực tiếp vào phần cứng CPU — trên i9-13900K, hệ thống sẽ phải hoặc (a) hạ tần số vòng điều khiển xuống dưới `1/0.025 = 40 Hz`, hoặc (b) chấp nhận drop frame, hoặc (c) dùng phần cứng mạnh hơn (Threadripper trở lên). Đây là lý do tài liệu GMR công bố rõ cả hai mức FPS trên hai loại CPU khác nhau thay vì chỉ một con số duy nhất — người triển khai cần tự tính ngân sách này cho ứng dụng cụ thể của mình.

### Hai giai đoạn tối ưu (two-stage optimization) — vì sao cần tách làm hai?

Nếu chỉ scale rồi giải IK một lần duy nhất, các artefact hình học (đặc biệt là ground penetration/self-intersection khi tỷ lệ robot khác xa tỷ lệ người) rất dễ xuất hiện, vì solver không có cơ hội "sửa" một quyết định scale sai từ đầu. Theo tinh thần công bố trong *Retargeting Matters*, GMR tách quy trình thành:

```text
Giai đoạn 1 — Thô (coarse):
  scale cục bộ theo human_scale_table
  giải IK sơ bộ để có q(t) khả dĩ cho toàn bộ clip
              │
              ▼
Giai đoạn 2 — Tinh chỉnh (refinement):
  rà lại các frame/khớp có artefact (foot penetration, self-intersection,
  jump đột ngột giữa các frame liền kề)
  điều chỉnh lại scale cục bộ hoặc nghiệm IK tại các vùng có vấn đề
              │
              ▼
  q(t) cuối cùng: ít artefact hơn một lần giải duy nhất
```

Cách hiểu trực giác: giai đoạn 1 giống một bản nháp nhanh, giai đoạn 2 giống một lượt "biên tập" chỉ sửa những chỗ nháp bị lỗi rõ ràng — thay vì cố làm hoàn hảo ngay từ đầu (điều khó vì scale tối ưu cho khớp này có thể làm khớp khác tệ hơn).

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | GMR | SOMA-retargeter (bài sau) | UMR (bài sau) |
|---|---|---|---|
| Phần cứng | CPU | GPU (Newton + NVIDIA Warp) | GPU (học + inference correspondence) |
| Chế độ | Streaming, per-frame, real-time | Batch, có chế độ headless | Batch (tiền xử lý correspondence) + retarget |
| Correspondence | Sparse, handcrafted (`ik_match_table` JSON) | Sparse, handcrafted (config theo robot) | Dense, học từ dữ liệu (point cloud) |
| IK core | Multi-objective differential IK (mink/MuJoCo, QP) | IK per-frame trên GPU (Newton) | Constrained point-cloud matching optimization |
| Input hỗ trợ | SMPL-X, BVH, FBX, video, streaming trực tiếp | BVH định dạng SOMA-skeleton | Nguồn hình học đa dạng qua bề mặt point cloud |
| Cần dữ liệu huấn luyện? | Không (thuần hình học) | Không (thuần hình học) | Có (học correspondence) |
| Ưu điểm chính | Real-time, không cần GPU, input đa dạng, dùng được cho teleoperation trực tiếp | Song song hoá lớn, phù hợp sinh dữ liệu RL quy mô lớn | Giảm phụ thuộc thiết kế tay, khái quát hoá tốt qua nhiều robot/nguồn |
| Nhược điểm chính | Vẫn phụ thuộc config sparse thủ công theo từng robot mới | Không phải real-time streaming; cần hạ tầng GPU | Cần mesh nguồn + canonical template, chưa xử lý bàn tay khéo léo/đa agent |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "GMR chạy trên CPU nghĩa là nó chậm/kém hơn phương pháp GPU".** Vì sao sai: kích thước bài toán IK mỗi frame của GMR rất nhỏ (vài chục biến khớp, vài task), giải bằng QP giải tích hội tụ trong vài vòng lặp — với bài toán nhỏ như vậy, overhead khởi tạo/di chuyển dữ liệu lên GPU thường **đắt hơn** lợi ích song song hoá. GPU chỉ có lợi khi xử lý *hàng loạt* frame/file cùng lúc (đúng use case của SOMA-retargeter), không phải khi xử lý tuần tự từng frame streaming. **Hiểu đúng:** CPU vs GPU ở đây là lựa chọn kiến trúc phù hợp với chế độ sử dụng (streaming vs batch), không phải một thang đo "nhanh/chậm" tuyệt đối.
2. **Hiểu nhầm: "non-uniform local scaling nghĩa là GMR tự động học tỷ lệ đúng cho mọi robot mới, không cần cấu hình gì thêm".** Vì sao sai: "non-uniform" chỉ có nghĩa là **hệ số scale khác nhau giữa các body/chain** (tay khác chân khác thân — đã thấy ở bài Skeleton mapping với các giá trị 0.75/0.9 khác nhau trong config), chứ không phải một cơ chế học tự động — các hệ số này vẫn nằm trong file JSON cấu hình do người thiết kế đặt cho từng cặp (định dạng nguồn, robot đích) cụ thể. Thêm một robot mới vẫn cần tạo/tinh chỉnh config này bằng tay. **Hiểu đúng:** "non-uniform" mô tả *độ chi tiết* của scale (theo từng phần cơ thể) chứ không mô tả *cách* các hệ số đó được xác định (vẫn là thủ công, giống hạn chế mà UMR nhắm tới khắc phục).
3. **Hiểu nhầm: "velocity limit là bước lọc (filter) áp sau khi đã giải xong IK".** Vì sao sai: như đã thấy ở mục Định nghĩa/Cơ chế, GMR đưa `−v_max·Δt ≤ Δθ ≤ v_max·Δt` **ngay vào ràng buộc của QP mỗi bước giải**, nghĩa là nghiệm IK được tìm *có tính tới* giới hạn vận tốc ngay từ đầu, không phải giải xong rồi cắt bớt kết quả (cách cắt bớt sau có thể phá vỡ vị trí/hướng đã đạt được của các task khác). **Hiểu đúng:** đây là ràng buộc trong vòng lặp (in-the-loop constraint), không phải hậu xử lý (post-processing filter) — khác biệt này ảnh hưởng trực tiếp tới chất lượng nghiệm khi nhiều task xung đột nhau.

## 🏗️ Ví dụ minh hoạ trong dự án này

Trong pipeline của dự án, GMR là bước cầu nối trực tiếp giữa `03-human-motion-datasets/` (AMASS, LAFAN1, OMOMO) và cả `01-whole-body-control/`/`04-imitation-learning-rl/` lẫn `08-real-robot-deployment/`:

```text
AMASS (SMPL-X) / LAFAN1 (BVH) / video (qua GVHMR)
                    │
                    ▼
        GMR (CPU, real-time, per-frame)
                    │
        ┌───────────┴────────────┐
        ▼                        ▼
reference motion offline    streaming trực tiếp
(dùng train RL policy       (TWIST — teleoperation
 tracking ở 04/01)           whole-body qua VR, 08)
```

Vì GMR đủ nhanh để chạy streaming (không chỉ offline batch), nó được chính tác giả dùng làm "Retargeter for TWIST" — hệ thống teleoperation whole-body thời gian thực (xem thêm `08-real-robot-deployment/`) — một bằng chứng thực tế cho thấy đặc tính CPU/real-time không chỉ là lựa chọn kỹ thuật trừu tượng mà quyết định trực tiếp GMR có thể dùng cho ứng dụng nào (teleoperation trực tiếp) mà SOMA-retargeter (GPU/batch) không phù hợp bằng.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **Paper chính thức của GMR đã định lượng ảnh hưởng của chất lượng retargeting lên RL policy.** Araujo, Ze, Xu, Wu, Liu, *"Retargeting Matters: General Motion Retargeting for Humanoid Motion Tracking"* (arXiv:2510.02252, ICRA 2026) không chỉ mô tả kiến trúc GMR mà còn đánh giá GMR so với hai baseline mã nguồn mở là **PHC** và **ProtoMotions**, cộng thêm một baseline dữ liệu closed-source của Unitree — dùng success rate nghiêm ngặt (policy phải hoàn thành *toàn bộ* motion tham chiếu) và huấn luyện qua **BeyondMimic** trên tập LAFAN1. Kết quả công bố: GMR đạt "perceptual fidelity và success rate gần với baseline closed-source" — tức là retargeting hình học thuần (không học) vẫn có thể cạnh tranh với dữ liệu được xử lý thủ công kỹ lưỡng bởi một công ty robot thương mại. *(Lưu ý: bảng số liệu MPJPE/success-rate chi tiết chưa trích xuất được đầy đủ từ bản PDF trong lần tra cứu này — nếu cần số chính xác, nên đọc trực tiếp Table trong paper.)* [arXiv:2510.02252](https://arxiv.org/abs/2510.02252)
2. **Hai hướng nghiên cứu 2025–2026 trực tiếp thách thức giả định "correspondence sparse, thủ công" của GMR.** UMR — *"Unified Motion Retargeting for Humanoids with Learned Point Cloud Correspondence"* (Cao et al., arXiv:2609.02134, 09/2026) chỉ ra: cách làm của GMR (và SOMA-retargeter) dựa trên "hand-crafted sparse keypoints or body-part pairs", giới hạn khả năng khái quát hoá qua nhiều nguồn dữ liệu/robot khác nhau — và đề xuất học dense point-cloud correspondence thay thế (xem bài giảng UMR riêng). Trong khi đó, **OmniRetarget** — *"Interaction-Preserving Data Generation for Humanoid Whole-Body Loco-Manipulation and Scene Interaction"* (arXiv:2509.26633, 2026) tấn công một hạn chế khác: GMR (như hầu hết retargeter geometric per-frame) tập trung vào tư thế cơ thể, chưa mô hình hoá tường minh quan hệ tương tác với vật thể/địa hình (contact với object, terrain) — OmniRetarget dùng **interaction mesh** để giữ các quan hệ không gian/tiếp xúc này khi retarget các kỹ năng loco-manipulation phức tạp (leo trèo, mang vật, tương tác cảnh vật) mà một retargeter "per-body-part" thuần tuý dễ làm sai. [arXiv:2609.02134](https://arxiv.org/abs/2609.02134) · [arXiv:2509.26633](https://arxiv.org/abs/2509.26633)
3. **Xu hướng chung 2025–2026:** GMR vẫn là baseline thực dụng, minh bạch, dễ debug (đúng như nhận định ở bài Skeleton mapping) — nhưng cả ba hướng mở rộng đang phát triển song song là (a) học correspondence thay vì thiết kế tay (UMR), (b) mô hình hoá tương tác vật thể/địa hình tường minh hơn (OmniRetarget), và (c) đưa thêm ràng buộc động lực học/tiếp xúc sâu hơn (KDMR — xem bài Gleicher). Không có dấu hiệu nào cho thấy các hướng này "thay thế" GMR trong ngắn hạn — chúng thường được dùng như một lớp bổ sung hoặc so sánh baseline, đúng như UMR tự báo cáo so sánh trực tiếp với GMR trên Unitree G1.

## ❓ Câu hỏi tự kiểm tra

1. Vì sao GMR giải đồng thời tất cả các task (tay, chân, thân) trong một QP duy nhất thay vì giải từng chain riêng biệt rồi ghép kết quả?
   <details><summary>Gợi ý đáp án</summary>Để đảm bảo nghiệm `θ` nhất quán toàn thân — nếu giải từng chain độc lập, các chain chia sẻ chung gốc (pelvis/root) có thể cho ra yêu cầu xung đột nhau về vị trí/hướng của root.</details>
2. Trong ví dụ velocity limit ở trên, nếu retarget chạy ở 30 FPS thay vì 60 FPS, `Δθ_max` mỗi frame thay đổi thế nào, và cần bao nhiêu frame để bắt kịp `Δθ_đề_xuất = 0.45 rad`?
   <details><summary>Gợi ý đáp án</summary>`Δt = 1/30 ≈ 0.0333s` → `Δθ_max = 9.4248 × 0.0333 ≈ 0.3142 rad` (gấp đôi so với 60 FPS, vì Δt gấp đôi) → cần `⌈0.45/0.3142⌉ = 2` frame, ít hơn so với 3 frame ở 60 FPS — vì mỗi frame ở tần số thấp hơn "được phép" đổi góc nhiều hơn tính theo rad tuyệt đối, dù tính theo thời gian thực tế cả hai đều giới hạn đúng cùng vận tốc góc tối đa.</details>
3. Vì sao velocity limit được đưa vào ràng buộc của QP thay vì áp dụng như một bộ lọc (low-pass filter) sau khi giải xong IK?
   <details><summary>Gợi ý đáp án</summary>Vì lọc sau khi giải có thể kéo nghiệm ra khỏi các target vị trí/hướng khác đã đạt được trong cùng bước giải — ràng buộc trong vòng lặp đảm bảo nghiệm cuối cùng vừa thoả velocity limit vừa vẫn tối ưu đồng thời cho mọi task khác, đúng tinh thần đã học ở bài Gleicher (constraint và objective giải cùng một bài toán, không tách rời).</details>
4. Một laptop có FPS đo được là 38 FPS. Vòng điều khiển teleoperation cần 50 Hz. Ngân sách mỗi frame là bao nhiêu ms, thời gian xử lý thực tế là bao nhiêu ms, và kết luận có theo kịp không?
   <details><summary>Gợi ý đáp án</summary>Ngân sách: `1/50 = 20ms`. Thời gian xử lý: `1/38 ≈ 26.3ms`. Vì `26.3ms > 20ms` → KHÔNG theo kịp vòng điều khiển 50Hz, cần hạ tần số điều khiển xuống dưới `1/0.0263 ≈ 38Hz` hoặc nâng cấp phần cứng.</details>
5. Vì sao nói GMR và SOMA-retargeter "cùng dùng correspondence sparse thủ công" dù một cái chạy CPU/streaming còn cái kia chạy GPU/batch?
   <details><summary>Gợi ý đáp án</summary>Vì cả hai đều dựa vào một bảng cấu hình (JSON/config) ánh xạ tường minh từng khớp/body người sang khớp/body robot do con người thiết kế trước (`ik_match_table` của GMR, config theo robot của SOMA) — khác biệt CPU/GPU và streaming/batch là ở tầng *giải* IK và *chế độ xử lý*, không phải ở tầng *correspondence*; đây chính là điểm UMR nhắm vào để cải tiến, độc lập với việc cả hai công cụ chạy trên phần cứng nào.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** giả sử `Δθ_đề_xuất = 0.60 rad` tại một khớp đang ở `θ = 0.10 rad`, chạy ở `45 FPS`, `v_max` không đổi (`3π rad/s`). Tính `Δθ_max` mỗi frame, số frame tối thiểu để khớp bắt kịp hoàn toàn (giả sử target không đổi thêm trong lúc đó), và giá trị `θ` sau frame đầu tiên.
2. **Đọc code/benchmark thật:** clone GMR (`github.com/YanjieZe/GMR`), chạy một demo có sẵn, tự đo FPS thực tế trên máy của bạn bằng cách bấm giờ xử lý N frame (ví dụ dùng `time.time()` quanh vòng lặp retarget). So sánh với hai con số 60–70 FPS (Threadripper) và 35–45 FPS (i9) đã nêu — máy bạn gần với mức nào, và tính thử ngân sách vòng điều khiển tối đa mà máy bạn theo kịp được (giống Phần B của ví dụ tính tay).

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

GMR là công cụ retargeting chính của dự án, hiện thực hoá toàn bộ lý thuyết đã học (constraint preservation, skeleton mapping + scale, differential IK) thành một pipeline 4 bước chạy hoàn toàn trên CPU: đọc pose người dạng dict, scale theo bảng cấu hình sparse thủ công, giải multi-objective differential IK dạng QP qua `mink`+MuJoCo với velocity limit (mặc định ≈3π rad/s) nằm ngay trong ràng buộc giải chứ không phải hậu xử lý, và xuất root pose + góc khớp actuated — đạt 60–70 FPS trên CPU workstation mạnh và 35–45 FPS trên CPU phổ thông, đủ nhanh cho cả xử lý offline lẫn streaming trực tiếp trong teleoperation (TWIST). So với SOMA-retargeter (GPU/batch) và UMR (dense correspondence học được), GMR đánh đổi khả năng khái quát hoá tự động để lấy sự đơn giản, minh bạch và tốc độ real-time trên CPU — một lựa chọn kiến trúc phù hợp cho streaming nhưng vẫn phụ thuộc cấu hình correspondence thủ công theo từng robot mới, đúng hạn chế mà cả UMR lẫn OmniRetarget đang tìm cách khắc phục.
