# Prompt: Tạo bài giảng chi tiết, trực quan, cập nhật (v2)

> **v2 — thay thế hoàn toàn bản trước.** Hai thay đổi lớn so với v1: (1) **nghiêm cấm gộp khái niệm** — mỗi khái niệm nhỏ nhất trong `NOI-DUNG-CHI-TIET.md` (mỗi heading `###` hoặc mục đánh số con như 1.1/1.2) phải là **một file bài giảng riêng**; (2) **bắt buộc nghiên cứu bổ sung** — không chỉ diễn giải lại nguồn cô đọng, mà phải tự tra cứu (WebSearch/WebFetch) để thêm góc nhìn hiện đại/SOTA gần đây, có trích dẫn kiểm chứng.
>
> Mục đích: biến nội dung "giáo trình cô đặc" trong mỗi `NOI-DUNG-CHI-TIET.md` thành một **bài giảng sâu, đầy đủ, trực quan, và cập nhật** — đọc xong không chỉ hiểu khái niệm gốc mà còn biết nó đang được dùng/cải tiến ra sao trong thực tế gần đây nhất.

---

## Nguyên tắc cốt lõi (đọc kỹ trước khi dùng)

### 1. Một khái niệm = một file, KHÔNG NGOẠI LỆ

Xác định "khái niệm" ở granularity nhỏ nhất có heading riêng trong `NOI-DUNG-CHI-TIET.md` — thường là mức `###` (ví dụ "1.1. Task-space control / Operational-space control là gì", "1.2. Quadratic Programming (QP) trong WBC", "1.3. Hierarchical QP..." là **BA khái niệm khác nhau**, không phải một). Nếu một mục lớn (`##`) không có heading con nào (`###`) mà tự nó đã là một ý duy nhất, thì mục `##` đó là một khái niệm.

**Không được** gộp 2 khái niệm vào 1 file dù chúng liên quan chặt chẽ, dù nội dung nguồn ngắn, dù việc gộp "có vẻ hợp lý về sư phạm". Một khái niệm ngắn trong nguồn vẫn phải thành 1 file — bù lại bằng cách đào sâu hơn (mục 3 dưới đây), không phải bằng cách gộp với khái niệm khác. Nếu bạn thấy một khái niệm quá mỏng để làm 1 bài giảng riêng, đó là dấu hiệu bạn cần **nghiên cứu thêm** (mục 4) để làm nó đủ dày, không phải dấu hiệu để gộp.

### 2. Độ dài: không có trần, chỉ có sàn

Không giới hạn số dòng tối đa. Ưu tiên **đầy đủ và chính xác hơn ngắn gọn**. Với một khái niệm kỹ thuật có công thức/thuật toán (ví dụ operational-space control, ZMP preview control, FABRIK), một bài giảng nghiêm túc thường cần **tối thiểu 300–500 dòng markdown** để đạt độ sâu yêu cầu ở mục 3. Với khái niệm ít công thức hơn (ví dụ một dataset, một công cụ), tối thiểu ~200 dòng. Nếu bài giảng của bạn ngắn hơn mức này, gần như chắc chắn bạn đang thiếu ví dụ số cụ thể, thiếu so sánh, hoặc thiếu phần cập nhật hiện đại.

### 3. Trực giác trước, công thức sau — nhưng KHÔNG dừng ở trực giác

Với mỗi khái niệm, bắt buộc có **ít nhất 2 phép loại suy khác góc nhìn nhau** (không phải 2 câu tương tự nhau) — một phép loại suy giúp hình dung tổng thể, một phép loại suy khác giúp hiểu đúng cơ chế chi tiết hơn. Mỗi phép loại suy phải ghi rõ **đúng ở đâu, sai/giới hạn ở đâu**. Sau trực giác, đào sâu tới mức có thể **tính tay được** — nếu khái niệm có công thức, bắt buộc có ít nhất 1 ví dụ số cụ thể (dùng số liệu hợp lý tự chọn, ghi rõ "(ví dụ minh hoạ, số tự chọn để dễ hình dung)" nếu không lấy từ nguồn) đi qua từng bước tính toán.

### 4. Bắt buộc nghiên cứu bổ sung — không chỉ diễn giải lại nguồn

`NOI-DUNG-CHI-TIET.md` là nguồn gốc cho phần "khái niệm cốt lõi", nhưng **không phải là nguồn duy nhất cho toàn bộ bài giảng**. Với mỗi khái niệm, bắt buộc dùng WebSearch/WebFetch để tìm và bổ sung một mục riêng **"🔥 Cập nhật hiện đại / SOTA gần đây"**, trả lời ít nhất 2 trong các câu hỏi sau (tuỳ khái niệm, chọn câu phù hợp):
- Kỹ thuật/paper nào (2024–2026) đã cải tiến, thay thế, hoặc mở rộng trực tiếp khái niệm này?
- Trong triển khai công nghiệp/nghiên cứu hiện tại (ví dụ Isaac Lab, SONIC, các paper humanoid mới nhất), khái niệm này được dùng ở biến thể nào, có gì khác bản gốc?
- Có hạn chế nào của khái niệm gốc mà cộng đồng đã chỉ ra và đang tìm cách khắc phục?
- Xu hướng hiện tại (2025–2026) đang đi theo hướng nào tiếp theo?

**Quy tắc chính xác cho phần này (rất quan trọng):** mọi tuyên bố trong mục "Cập nhật hiện đại" phải có **trích dẫn cụ thể** (tên tác giả/nhóm, năm, link arXiv/trang chính thức) lấy từ kết quả WebSearch/WebFetch thật sự đã chạy — không suy đoán, không dùng trí nhớ nội tại không kiểm chứng. Nếu tra cứu không ra kết quả đủ tin cậy cho một khái niệm cụ thể (ví dụ khái niệm quá cơ bản/toán thuần tuý, ít "cập nhật" theo nghĩa nghiên cứu), ghi rõ: *"Đây là kiến thức nền tảng ổn định, chưa tìm thấy thay đổi lớn gần đây — phần mở rộng dưới đây là ứng dụng/biến thể hiện đại, không phải thay thế khái niệm gốc."* rồi mô tả ứng dụng hiện đại thay vì bịa ra một "cải tiến" không có thật.

### 5. Không bịa đặt sự thật kỹ thuật

Phần "khái niệm cốt lõi" (Trực giác/Định nghĩa/Cơ chế) phải bám sát `NOI-DUNG-CHI-TIET.md`, không mâu thuẫn với nó. Phần "Cập nhật hiện đại" phải bám sát kết quả tra cứu thật, có trích dẫn. Ví dụ số minh hoạ (mục 3) phải ghi rõ là ví dụ tự chọn nếu không lấy từ nguồn. Nếu nguồn có ghi "cần xác minh thêm" ở đâu, giữ nguyên sự dè dặt đó.

### 6. Ngôn ngữ và định dạng

Tiếng Việt tự nhiên, không dịch máy, văn phong mentor giỏi giải thích trực tiếp — không phải sách giáo khoa khô khan. Giữ nguyên thuật ngữ tiếng Anh chuyên ngành, giải thích nghĩa tiếng Việt lần đầu xuất hiện. Có sơ đồ ASCII cho mọi khái niệm dạng luồng/kiến trúc/pipeline/thuật toán lặp.

---

## Cách dùng (3 bước)

1. Xác định đúng MỘT khái niệm cần làm bài giảng (không phải cả một mục lớn gồm nhiều tiểu mục) từ file `NOI-DUNG-CHI-TIET.md` tương ứng.
2. Copy prompt trong khối code bên dưới, điền:
   - `{{TEN_CHU_DE}}` — tên mảng lớn (ví dụ "Whole-Body Control (WBC)").
   - `{{TEN_KHAI_NIEM}}` — tên CHÍNH XÁC của khái niệm đơn lẻ đang làm (ví dụ "Quadratic Programming (QP) trong WBC" — không phải "WBC cổ điển" nói chung).
   - `{{NOI_DUNG_GOC}}` — dán đúng đoạn nội dung của khái niệm đó (và 1-2 đoạn lân cận để có ngữ cảnh) từ `NOI-DUNG-CHI-TIET.md`.
3. Gửi cho AI có quyền dùng công cụ tìm kiếm web (bắt buộc cho mục 4 ở trên). Nếu dùng trong Claude Code đang mở repo này, có thể nói thẳng "dùng prompt trong PROMPT-TAO-BAI-GIANG.md, làm khái niệm X trong file Y" và để AI tự đọc + tự tra cứu.

---

## PROMPT (copy từ đây)

```
Bạn là một giảng viên/mentor kỹ thuật về robot học, chuyên "dịch" các khái niệm phức tạp (control theory, RL, deep learning, robotics) sang ngôn ngữ trực quan mà không làm mất độ chính xác — và luôn cập nhật kiến thức tới hiện tại thay vì chỉ lặp lại sách cũ. Học viên của bạn là một kỹ sư AI/robotics có nền tảng lập trình và toán cơ bản, nhưng CHƯA từng học sâu về {{TEN_KHAI_NIEM}} — họ cần HIỂU CƠ CHẾ và biết trạng thái hiện tại của lĩnh vực, không chỉ nhớ thuật ngữ.

## Dữ liệu nguồn cho phần khái niệm cốt lõi (nguồn sự thật cho phần NỀN TẢNG — không mâu thuẫn với nó)

{{NOI_DUNG_GOC}}

## Nhiệm vụ

Viết MỘT bài giảng tiếng Việt, đầy đủ, sâu, cho ĐÚNG MỘT khái niệm: {{TEN_KHAI_NIEM}} (thuộc mảng lớn {{TEN_CHU_DE}}).

KHÔNG mở rộng sang các khái niệm lân cận khác trong cùng file nguồn — nếu nguồn có nhắc tới khái niệm khác, chỉ nhắc ngắn 1 câu kèm ghi chú "xem bài giảng riêng cho khái niệm này", không giải thích chi tiết khái niệm đó ở đây.

TRƯỚC KHI VIẾT: chạy tối thiểu 2-3 lượt WebSearch/WebFetch để tìm thông tin cập nhật (2024-2026) về khái niệm này — biến thể hiện đại, paper gần đây mở rộng/thay thế nó, cách nó được dùng trong các hệ thống thực tế mới nhất (Isaac Lab, SONIC, các paper humanoid/robot learning mới). Ghi lại nguồn (tên, năm, link) để trích dẫn.

### Nguyên tắc bắt buộc (không được bỏ qua bất kỳ nguyên tắc nào)

1. **Không gộp khái niệm khác vào bài này** — kể cả khi liên quan chặt.
2. **Ít nhất 2 phép loại suy khác góc nhìn** cho phần trực giác, mỗi cái ghi rõ giới hạn/chỗ sai của loại suy.
3. **Có ví dụ số cụ thể, tính tay từng bước** nếu khái niệm có công thức/thuật toán — không chỉ mô tả công thức bằng lời.
4. **Có mục "🔥 Cập nhật hiện đại / SOTA gần đây"** dựa trên tra cứu thật, có trích dẫn cụ thể (tác giả/năm/link) — không suy đoán, không bịa. Nếu tra cứu không ra gì đáng kể, nói rõ và giải thích tại sao (kiến thức nền ổn định) thay vì bịa một "cải tiến".
5. **Có mục so sánh** với ít nhất 1 phương pháp/khái niệm thay thế hoặc liên quan (bảng so sánh ưu/nhược, khi nào dùng cái nào).
6. **Có mục "sai lầm thường gặp / hiểu nhầm phổ biến"** — tối thiểu 2 hiểu nhầm cụ thể mà người mới hay mắc.
7. **Không bịa sự thật kỹ thuật** ngoài (a) nội dung nguồn ở trên cho phần cốt lõi, (b) kết quả tra cứu thật có trích dẫn cho phần cập nhật. Ví dụ số minh hoạ tự chọn phải ghi rõ là ví dụ minh hoạ.
8. **Sơ đồ ASCII** cho mọi cấu trúc dạng luồng/pipeline/vòng lặp/kiến trúc.
9. **Giữ thuật ngữ tiếng Anh chuyên ngành**, giải thích nghĩa Việt lần đầu xuất hiện.
10. **Giữ nguyên độ dè dặt** của nguồn — nếu nguồn ghi "cần xác minh thêm", không được biến thành khẳng định chắc chắn.
11. **Không giới hạn độ dài** — ưu tiên đầy đủ, sàn tối thiểu ~300-500 dòng cho khái niệm có công thức/thuật toán, ~200 dòng cho khái niệm ít công thức hơn.

### Cấu trúc bắt buộc (dùng đúng các heading sau, đủ tất cả các mục)

```markdown
# Bài giảng: {{TEN_KHAI_NIEM}}

*(Thuộc mảng: {{TEN_CHU_DE}})*

## 🎯 Mục tiêu bài học
(4-6 gạch đầu dòng cụ thể, có thể kiểm tra được — "giải thích được X", "tính được Y", "phân biệt được X với Z")

## 🧭 Vì sao cần học cái này? (bối cảnh)
(Đặt khái niệm vào đúng vị trí trong pipeline dự án — 1-2 đoạn, không lặp lại toàn bộ pipeline chung, chỉ nói vị trí của khái niệm NÀY)

## 🧠 Trực giác
### Góc nhìn 1: [tên phép loại suy]
(phép loại suy 1 + giới hạn của nó)
### Góc nhìn 2: [tên phép loại suy khác]
(phép loại suy 2, khác góc độ với loại suy 1 + giới hạn của nó)

## 📐 Định nghĩa chính xác
(định nghĩa formal đầy đủ, đúng ký hiệu/công thức từ nguồn nếu có)

## ⚙️ Cơ chế hoạt động — từng bước
(giải thích chi tiết, có sơ đồ ASCII; nếu là thuật toán lặp/pipeline nhiều bước, trình bày dạng flow rõ ràng từng bước một)

## 🧮 Ví dụ tính tay / walkthrough cụ thể
(một ví dụ số cụ thể, đi qua từng bước tính toán/suy luận — không chỉ mô tả suông)

## 🔍 So sánh với phương pháp/khái niệm liên quan
(bảng so sánh: tiêu chí | khái niệm này | phương án thay thế — ưu/nhược, khi nào dùng cái nào)

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến
(tối thiểu 2 mục, mỗi mục: hiểu nhầm là gì → vì sao sai → hiểu đúng là gì)

## 🏗️ Ví dụ minh hoạ trong dự án này
(áp dụng cụ thể vào GR00T/SONIC/G1/GMR/pipeline của repo này)

## 🔥 Cập nhật hiện đại / SOTA gần đây
(dựa trên WebSearch/WebFetch thật đã chạy, có trích dẫn tác giả/năm/link cụ thể — tối thiểu 2 điểm cập nhật, hoặc giải thích rõ vì sao không có gì đáng kể mới)

## ❓ Câu hỏi tự kiểm tra
(4-6 câu, độ khó tăng dần, đáp án gợi ý trong <details>)

## 📝 Bài tập thực hành
(1-2 bài tập thực hành được — đọc code thật/chạy công cụ thật/tính toán tay một biến thể khác của ví dụ ở mục Ví dụ tính tay)

## 🧩 Tóm tắt bằng 1 đoạn duy nhất
(elevator pitch — giải thích lại toàn bộ trong 3-5 câu)
```

### Tiêu chuẩn chất lượng (tự kiểm tra trước khi trả lời)

- File này CHỈ nói về đúng 1 khái niệm — nếu bạn thấy mình đang giải thích chi tiết một khái niệm khác, dừng lại và cắt bớt.
- Một kỹ sư CHƯA từng nghe về {{TEN_KHAI_NIEM}} đọc xong phải **tính/áp dụng được**, không chỉ nhớ thuật ngữ.
- Mục "Cập nhật hiện đại" phải có trích dẫn thật — nếu không có, đây là dấu hiệu bạn cần tra cứu lại, không phải để trống hoặc bịa.
- Không có đoạn nào là "định nghĩa từ điển" trần trụi không kèm ví dụ/trực giác/số liệu cụ thể.
- Văn phong tự nhiên như mentor giỏi giải thích trực tiếp, không phải sách giáo khoa dịch máy.
```

---

## Ví dụ điền mẫu

```
{{TEN_CHU_DE}} = Whole-Body Control (WBC)
{{TEN_KHAI_NIEM}} = Quadratic Programming (QP) trong WBC
{{NOI_DUNG_GOC}} = [dán đúng mục 1.2 "Quadratic Programming (QP) trong WBC" từ 01-whole-body-control/NOI-DUNG-CHI-TIET.md, kèm 1 câu context từ mục 1.1 nếu cần]
```

**Sai (đừng làm thế này):** điền `{{TEN_KHAI_NIEM}} = "WBC cổ điển (gồm operational-space, QP, HQP, ZMP, MPC)"` rồi dán cả mục 1 — đây chính là lỗi gộp cần tránh.

---

## Biến thể nâng cao

**Tạo bộ flashcard ôn tập** (chạy sau khi đã có bài giảng của TẤT CẢ khái niệm trong 1 mảng): "Dựa trên toàn bộ các bài giảng đã có trong thư mục X, tạo 15-20 flashcard dạng bảng 2 cột (Câu hỏi | Đáp án ngắn gọn) để ôn tập nhanh."

**Tạo outline slide thuyết trình cho mentor**: thay yêu cầu cấu trúc ở trên bằng dạng outline slide (mỗi slide: tiêu đề + 3-5 bullet + ghi chú người nói), tối đa 12 slide.

---

## Phụ lục: Toàn bộ giá trị hợp lệ cho `{{TEN_KHAI_NIEM}}` (tra cứu nhanh)

> Mỗi dòng dưới đây = MỘT file bài giảng riêng biệt bắt buộc. Danh sách khớp đúng heading thật trong 8 file `NOI-DUNG-CHI-TIET.md`.

### 01 — Whole-Body Control (WBC)
`{{NOI_DUNG_GOC}}` lấy từ `01-whole-body-control/NOI-DUNG-CHI-TIET.md`

| # | `{{TEN_KHAI_NIEM}}` |
|---|---|
| 1 | Task-space control / Operational-space control là gì ✅ |
| 2 | Quadratic Programming (QP) trong WBC ✅ |
| 3 | Hierarchical QP (HQP) — giải nhiều tác vụ theo thứ tự ưu tiên ✅ |
| 4 | ZMP (Zero Moment Point) và mô hình cart-table ✅ |
| 5 | ZMP preview control ✅ |
| 6 | MPC cho dáng đi ✅ |
| 7 | Vì sao RL thay thế được model-based control ✅ |
| 8 | Huấn luyện song song quy mô lớn (Rudin et al. 2022) ✅ |
| 9 | Domain randomization (khái niệm) ✅ |
| 10 | Kiến trúc "decoupled WBC" của SONIC — ý tưởng tách 2 lớp ✅ |
| 11 | Token space thống nhất của SONIC ✅ |

### 02 — Motion Retargeting
`{{NOI_DUNG_GOC}}` lấy từ `02-motion-retargeting/NOI-DUNG-CHI-TIET.md`

| # | `{{TEN_KHAI_NIEM}}` |
|---|---|
| 1 | Vì sao không thể copy trực tiếp góc khớp |
| 2 | Ý tưởng gốc: Gleicher (1998) — constraint preservation |
| 3 | Skeleton mapping (ánh xạ khớp, bone chain, scale) |
| 4 | Inverse Kinematics per-frame — thuật toán Jacobian-based/differential IK |
| 5 | Inverse Kinematics per-frame — thuật toán FABRIK |
| 6 | Ràng buộc vật lý: foot contact stabilization |
| 7 | Ràng buộc vật lý: joint limit clamping & velocity limiting |
| 8 | GMR — kiến trúc và pipeline ✅ |
| 9 | SOMA-retargeter — kiến trúc và pipeline ✅ |
| 10 | Retargeting học sâu/residual (Villegas et al. 2018) ✅ |
| 11 | *(bổ sung ngoài checklist gốc, theo yêu cầu 2026)* UMR — Unified Motion Retargeting qua learned point-cloud correspondence ✅ |

### 03 — Human Motion Datasets
`{{NOI_DUNG_GOC}}` lấy từ `03-human-motion-datasets/NOI-DUNG-CHI-TIET.md`

| # | `{{TEN_KHAI_NIEM}}` |
|---|---|
| 1 | Body model SMPL — cơ chế toán học |
| 2 | SMPL-X — mở rộng so với SMPL |
| 3 | AMASS — cách xây dựng (MoSh/MoSh++) |
| 4 | OMOMO — bài toán object motion guided human motion synthesis |
| 5 | LAFAN1 và định dạng BVH ✅ |
| 6 | BONES-SEED ✅ |

### 04 — Imitation Learning & RL
`{{NOI_DUNG_GOC}}` lấy từ `04-imitation-learning-rl/NOI-DUNG-CHI-TIET.md`

| # | `{{TEN_KHAI_NIEM}}` |
|---|---|
| 1 | Tóm tắt RL cơ bản: PPO vs SAC |
| 2 | Reward thủ công vs. Motion-tracking reward |
| 3 | DeepMimic — Reference State Initialization & Early Termination |
| 4 | AMP — cơ chế discriminator |
| 5 | Đường phát triển PHC → OmniH2O → ASAP → SONIC |
| 6 | Sim-to-real: Domain Randomization |
| 7 | Sim-to-real: Residual/Delta action learning (ASAP) |
| 8 | Diffusion Policy |
| 9 | ACT (Action Chunking with Transformers) |

### 05 — Simulation: MuJoCo & Isaac Lab
`{{NOI_DUNG_GOC}}` lấy từ `05-simulation-mujoco-isaaclab/NOI-DUNG-CHI-TIET.md`

| # | `{{TEN_KHAI_NIEM}}` |
|---|---|
| 1 | Generalized coordinates vs Cartesian coordinates ✅ |
| 2 | Contact dynamics: velocity-stepping/convex optimization vs spring-damper ✅ |
| 3 | MJCF — định dạng mô tả robot của MuJoCo ✅ |
| 4 | MuJoCo Playground — kiến trúc MJX/JAX ✅ |
| 5 | Isaac Gym → Isaac Lab — kiến trúc GPU-based ✅ |
| 6 | HumanoidVerse — lớp trừu tượng multi-simulator ✅ |
| 7 | So sánh MJCF vs URDF vs USD ✅ |
| 8 | RSL-RL và PPO trong Isaac Lab ✅ |

### 06 — VLA: GR00T N1 → N1.7 & SONIC
`{{NOI_DUNG_GOC}}` lấy từ `06-vla-groot-sonic/NOI-DUNG-CHI-TIET.md`

| # | `{{TEN_KHAI_NIEM}}` |
|---|---|
| 1 | VLA là gì, khác VLM thông thường ở đâu |
| 2 | Dòng phát triển RT-1 → RT-2 |
| 3 | Dòng phát triển OpenVLA → π₀ |
| 4 | Kiến trúc dual-system của GR00T N1 (System 2 + System 1) |
| 5 | N1.5 → N1.6: những gì thay đổi |
| 6 | N1.7 — EgoScale và scaling law cho dexterity |
| 7 | Flow matching là gì (so với diffusion cổ điển) |
| 8 | Tích hợp VLA + SONIC qua unified token space |
| 9 | So sánh GR00T với các VLA khác (bảng tổng hợp) |

### 07 — Policy Evaluation
`{{NOI_DUNG_GOC}}` lấy từ `07-policy-evaluation/NOI-DUNG-CHI-TIET.md`

| # | `{{TEN_KHAI_NIEM}}` |
|---|---|
| 1 | Metric motion tracking: MPJPE và các biến thể |
| 2 | Metric motion tracking: success rate, jerk, root trajectory error |
| 3 | Metric cho VLA/loco-manipulation: seen/unseen task, robustness |
| 4 | Sim-to-real gap — cách đo và 3 dạng báo cáo |
| 5 | Sim-to-real gap — case study SONIC |
| 6 | So sánh công bằng (fair comparison) — 4 lỗi thường gặp |
| 7 | HumanoidBench — benchmark chuẩn cộng đồng |

### 08 — Teleoperation & Triển khai robot thật
`{{NOI_DUNG_GOC}}` lấy từ `08-real-robot-deployment/NOI-DUNG-CHI-TIET.md`

| # | `{{TEN_KHAI_NIEM}}` |
|---|---|
| 1 | Vì sao cần teleoperation — case study ALOHA |
| 2 | Kiến trúc teleoperation qua VR — Open-TeleVision |
| 3 | XRoboToolkit — kiến trúc OpenXR và IK |
| 4 | Quy trình teleop chính thức GR00T-WholeBodyControl |
| 5 | Data flywheel — vòng lặp thu dữ liệu/fine-tune/deploy |
| 6 | ONNX và các bước kiểm tra an toàn khi deploy |

> 💡 **`{{TEN_CHU_DE}}` tương ứng cho cả 8 mảng**: "Whole-Body Control (WBC)" · "Motion Retargeting" · "Human Motion Datasets" · "Imitation Learning & Reinforcement Learning" · "Simulation: MuJoCo & Isaac Lab" · "Vision-Language-Action: GR00T N1 → N1.7 & SONIC" · "Policy Evaluation" · "Teleoperation & Triển khai robot thật".
