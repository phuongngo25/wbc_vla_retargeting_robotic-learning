# 06 — Vision-Language-Action (VLA): GR00T N1 → N1.7 & SONIC

> Đây là lớp "não cấp cao": nhận ảnh + ngôn ngữ, quyết định *làm gì*, rồi giao lại cho SONIC/WBC (`01-whole-body-control/`) thực thi *làm như thế nào* ở mức khớp/motor. Đây cũng là mảnh ghép cuối cùng nối tất cả các thư mục trước lại với nhau.

---

## A. Khái niệm cần nắm, theo thứ tự

1. **VLA là gì:** một model nhận multimodal input (ảnh camera + câu lệnh ngôn ngữ + proprioception — trạng thái khớp hiện tại) và sinh ra hành động (action chunk), thay vì chỉ phân loại/mô tả ảnh như VLM thông thường.
2. **Dòng phát triển VLA nói chung** (đã có trong `../../resources/08-core-reading-list.md` Track C2 — đọc trước nếu chưa quen): RT-2 → OpenVLA → π₀. GR00T là một nhánh **chuyên biệt cho humanoid whole-body**, khác các VLA này ở chỗ output không chỉ điều khiển 1 tay máy cố định mà cả cơ thể (đi lại + thao tác).
3. **GR00T N1 (bản gốc, 3/2025):** foundation model đầu tiên cho generalist humanoid — kiến trúc dual-system (system 2 "suy nghĩ chậm" bằng VLM + system 1 "phản xạ nhanh" bằng diffusion action head).
4. **N1.5 → N1.6:** các bản cải tiến trung gian — cải thiện dữ liệu huấn luyện, kiến trúc decoupled WBC được tách ra thành nền chung (đây là lý do WBC và VLA đi chung 1 repo `GR00T-WholeBodyControl`).
5. **N1.7 (bản mới nhất):** pretraining trên **EgoScale** — 20,854 giờ video egocentric (góc nhìn thứ nhất) trải rộng 20+ loại tác vụ (sản xuất, bán lẻ, y tế, gia đình...). Phát hiện quan trọng: **scaling law đầu tiên cho độ khéo léo (dexterity) của robot** — càng nhiều dữ liệu video người thật (không cần dữ liệu robot!), robot càng khéo léo hơn theo quy luật có thể dự đoán. Kiến trúc dùng **flow-matching action transformer** để sinh action chunk từ ảnh + ngôn ngữ + proprioception.
6. **Tích hợp VLA + SONIC (unified token space):** SONIC không chỉ nhận lệnh từ VR teleop mà còn nhận lệnh trực tiếp từ GR00T N1.x qua cùng một không gian token — cho phép "autonomous VLA-driven whole-body loco-manipulation" (robot tự chủ vừa di chuyển vừa thao tác, không cần người điều khiển).

---

## B. Tài liệu chính

| Tài liệu | Tầng | Vì sao đọc | Link |
|---|---|---|---|
| **GR00T N1** — An Open Foundation Model for Generalist Humanoid Robots | 🔴🔴 | Paper gốc, bắt buộc đọc kỹ trước mọi bản N1.x sau này. | [arXiv:2503.14734](https://arxiv.org/abs/2503.14734) |
| **GR00T N1.7 — HuggingFace blog** | 🔴 | Thông tin chi tiết nhất hiện có về N1.7 (EgoScale, scaling law, flow-matching action transformer) — chưa có paper riêng, dùng blog này làm nguồn chính. | [HF blog](https://huggingface.co/blog/nvidia/gr00t-n1-7) |
| **`nvidia/GR00T-N1.7-3B`** (checkpoint) | 🔴 | Model thật để chạy inference — không chỉ đọc lý thuyết, phải tự chạy. | [HuggingFace](https://huggingface.co/nvidia/GR00T-N1.7-3B) |
| **`NVIDIA/Isaac-GR00T`** (GitHub — code inference/finetune chính thức) | 🔴 | Repo code để chạy và fine-tune N1.x, tách biệt với repo `GR00T-WholeBodyControl` (repo kia tập trung WBC/SONIC, repo này tập trung VLA). | [GitHub](https://github.com/NVIDIA/Isaac-GR00T) |
| **SONIC** (đọc lại lần 3, tập trung phần "unified token space" và kết quả VLA-driven loco-manipulation) | 🔴 | Đây là nơi WBC và VLA thực sự gặp nhau. | [arXiv:2511.07820](https://arxiv.org/abs/2511.07820) |
| **GR00T-WholeBodyControl — tutorials/VLA workflows** | 🔴 | Hướng dẫn thực hành chính thức cách N1.x gọi SONIC. | [Docs](https://nvlabs.github.io/GR00T-WholeBodyControl/) |

**Đối chiếu với VLA phi-humanoid (đã có ở Track C2, đọc lại để so sánh):**

| Paper | Khác biệt so với GR00T |
|---|---|
| RT-2 ([arXiv:2307.15818](https://arxiv.org/abs/2307.15818)) | VLA cho tay máy cố định, không có whole-body/locomotion. |
| OpenVLA ([arXiv:2406.09246](https://arxiv.org/abs/2406.09246)) | Mã nguồn mở, dùng để hiểu kiến trúc VLA cơ bản trước khi vào GR00T (phức tạp hơn vì có thêm WBC). |
| π₀ ([arXiv:2410.24164](https://arxiv.org/abs/2410.24164)) | Cùng dùng flow-matching cho action head — so sánh trực tiếp với cách N1.7 dùng flow-matching action transformer. |

---

## C. Video hướng dẫn

| Video | Ngôn ngữ | Nội dung |
|---|---|---|
| NVIDIA GTC talks về GR00T (tìm "NVIDIA GR00T GTC keynote") | Tiếng Anh | Demo trực quan robot G1 thực hiện task ngôn ngữ→hành động — xem trước khi đọc paper để có hình dung thực tế. |
| HuggingFace blog N1.7 kèm video demo nhúng | Tiếng Anh | Trực tiếp trên trang blog, có video minh hoạ EgoScale pretraining và kết quả dexterity. |

> ⚠️ Chưa có video/tài liệu tiếng Việt về GR00T/SONIC cụ thể — đây là công nghệ rất mới (N1.7 và SONIC đều công bố trong năm 2025). Ưu tiên đọc tài liệu gốc tiếng Anh; dùng công cụ dịch/AI hỗ trợ đọc nếu cần, nhưng đừng chỉ đọc bản tóm tắt — paper gốc có nhiều chi tiết kiến trúc quan trọng bị mất khi tóm tắt.

---

## D. Lộ trình từng bước cho người mới bắt đầu

1. **(2–3 ngày)** Xem 1–2 video demo GR00T (GTC talk hoặc blog N1.7) để có hình dung trực quan trước khi đọc paper.
2. **(1 tuần)** Đọc GR00T N1 pass-3 đầy đủ — đây là paper nền, hiểu kỹ kiến trúc dual-system trước khi đọc các bản sau.
3. **(3–5 ngày)** Đọc blog N1.7 kỹ — so sánh với N1 gốc: thay đổi gì về dữ liệu (EgoScale), kiến trúc (flow-matching action transformer), và phát hiện mới (scaling law cho dexterity).
4. **(2–3 ngày)** Tải checkpoint `GR00T-N1.7-3B`, chạy inference theo hướng dẫn trong `Isaac-GR00T` repo trên 1 task mẫu có sẵn (không cần robot thật — có thể chạy trên ảnh/video mẫu hoặc trong simulation).
5. **(1 tuần)** Đọc lại SONIC lần 3, tập trung đúng phần "token space thống nhất" và thí nghiệm VLA-driven loco-manipulation — đây là phần trả lời câu hỏi "N1.7 và SONIC nói chuyện với nhau như thế nào".
6. **(checkpoint)** Vẽ sơ đồ (tay hoặc draw.io) luồng dữ liệu đầy đủ: camera+ngôn ngữ → N1.7 → token hành động → SONIC → góc khớp motor → robot. Giải thích cho người khác (hoặc mentor) sơ đồ này bằng lời — đây là bài kiểm tra hiểu bài tốt nhất.

---

## Liên kết chéo

- Nhận input hành vi cấp thấp từ `01-whole-body-control/` (SONIC).
- Dữ liệu huấn luyện/fine-tune có thể tới từ teleoperation — `08-real-robot-deployment/`.
- Đánh giá chất lượng model — `07-policy-evaluation/`.
