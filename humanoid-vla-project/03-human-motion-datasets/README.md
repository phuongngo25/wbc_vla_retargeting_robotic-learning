# 03 — Human Motion Datasets

> Toàn bộ pipeline (retargeting → WBC → VLA) phụ thuộc vào chất lượng và quy mô dữ liệu chuyển động người. Hiểu rõ 4 dataset mentor liệt kê — chúng khác nhau về **định dạng**, **loại chuyển động**, và **vai trò trong pipeline**.

---

## A. Bốn dataset chính

| Dataset | Định dạng | Nội dung | Quy mô | Vai trò trong pipeline | Link |
|---|---|---|---|---|---|
| **AMASS** | SMPL/SMPL-X (mesh + skeleton tham số hoá) | Hợp nhất ~15 bộ mocap optical marker khác nhau thành 1 định dạng thống nhất | >40 giờ, 11,000+ chuyển động, 480+ người | Nguồn chuyển động người **đa dạng** nhất, dùng làm input chính cho GMR/SOMA-retargeter | [amass.is.tue.mpg.de](https://amass.is.tue.mpg.de) (cần đăng ký) |
| **SMPL-X** | *(không phải dataset, mà là body model)* | Mô hình tham số hoá cơ thể người (dáng người + tư thế + bàn tay + mặt) mà AMASS/OMOMO dùng để biểu diễn chuyển động | — | Là "ngôn ngữ chung" để biểu diễn skeleton người trước khi retarget | [smpl-x.is.tue.mpg.de](https://smpl-x.is.tue.mpg.de) |
| **OMOMO** | SMPL-H/SMPL-X + object trajectory + 3D mesh vật thể | Mocap **tương tác người–vật thể** (Stanford, SIGGRAPH Asia 2023): 15 loại vật thể, ~10 giờ | ~10 giờ | Dữ liệu duy nhất trong 4 loại có **loco-manipulation** (vừa di chuyển vừa cầm/mang vật) — quan trọng cho VLA loco-manipulation của SONIC/GR00T | Tìm "OMOMO dataset Stanford SIGGRAPH Asia 2023" |
| **LAFAN1** | BVH (skeleton khớp góc, không mesh) | Mocap chất lượng sản xuất game (Ubisoft): đi, chạy, nhảy múa, đánh nhau, 5 diễn viên, 15 loại động tác | 496,672 frame @ 30Hz | Chuyển động **tự nhiên, mượt, chất lượng cao** — chuẩn benchmark phổ biến cho motion tracking RL (nhiều paper WBC dùng LAFAN1 làm test set) | Tìm "Ubisoft LAFAN1 dataset GitHub" |
| **BONES-SEED** | Riêng của NVIDIA, đã retarget sẵn sang G1 | 142,000+ chuyển động người (~288 giờ) **+ quỹ đạo G1 tương ứng** | ~288 giờ | Dataset dùng để huấn luyện SONIC ở quy mô lớn — minh hoạ vì sao retargeting phải làm *trước*, ở quy mô lớn, không phải thủ công từng file | Nhắc tới trong tài liệu `GR00T-WholeBodyControl` — chưa rõ đã public hoàn toàn hay một phần, **cần kiểm tra license/tình trạng phát hành khi bạn tới Phase 1** |

> 💡 **Cần research thêm** (mentor lưu ý): `humanoid-wbc-review` còn liệt kê **Motion-X++** (dataset chuyển động quy mô lớn khác, có text annotation) — đáng xem nếu bạn cần chuyển động có mô tả ngôn ngữ đi kèm (liên quan trực tiếp tới huấn luyện VLA ở `06-vla-groot-sonic/`). Ngoài ra còn **HumanML3D**, **KIT-ML** (dataset motion-language phổ biến trong literature review học thuật) — tìm qua GMR/awesome-humanoid-robot-learning để mở rộng danh sách.

---

## B. Tài liệu nền tảng để hiểu định dạng dữ liệu

| Tài liệu | Vì sao đọc |
|---|---|
| **SMPL** — Skinned Multi-Person Linear Model (paper gốc, SIGGRAPH Asia 2015) | Hiểu cách một vector tham số ngắn (pose + shape) sinh ra toàn bộ mesh cơ thể — nền của SMPL-X. |
| **SMPL-X** paper (CVPR 2019) | Mở rộng SMPL thêm bàn tay + khuôn mặt — định dạng AMASS/OMOMO dùng. |
| BVH format spec (bất kỳ tài liệu giới thiệu ngắn nào, ví dụ trên trang GMR) | BVH là định dạng "cây khớp + góc Euler theo frame" — đơn giản hơn SMPL nhưng phổ biến trong game/mocap công nghiệp (LAFAN1 dùng BVH). |

---

## C. Video hướng dẫn

| Video | Ngôn ngữ | Nội dung |
|---|---|---|
| Bất kỳ video giới thiệu "SMPL body model explained" trên YouTube | Tiếng Anh | Trực giác nhanh về body model trước khi đọc paper gốc — tìm với từ khoá "SMPL SMPL-X body model tutorial". |
| Blender AMASS visualization tutorials | Tiếng Anh | Nếu muốn tự tay xem chuyển động AMASS trong Blender bằng addon `SMPL-X Blender add-on`. |

> ⚠️ Chưa có tài liệu/video tiếng Việt cho mảng dataset chuyên biệt này. Nếu bạn cần học nền tảng thị giác máy tính/3D bằng tiếng Việt trước, có thể tham khảo các kênh phổ biến như *Deep Learning cơ bản* (Vũ Hữu Tiệp) hoặc *AI VIETNAM* — chỉ để lấy nền toán/3D, không có nội dung humanoid cụ thể.

---

## D. Lộ trình từng bước cho người mới bắt đầu

1. **(1–2 ngày)** Đọc README chính thức của AMASS và LAFAN1 để hiểu cấu trúc file (không cần đọc paper SMPL gốc ngay).
2. **(2–3 ngày)** Đăng ký AMASS, tải 1 subset nhỏ (ví dụ CMU Mocap con trong AMASS). Cài `body_visualizer`/`human_body_prior` (thư viện chính thức của SMPL-X) để render thử 1 sequence.
3. **(1–2 ngày)** Tải LAFAN1 (public), mở bằng một BVH viewer bất kỳ (Blender hoặc script Python đơn giản) để xem cấu trúc khớp.
4. **(2–3 ngày)** Đọc mô tả OMOMO, xác định: dữ liệu này khác AMASS ở đâu (có vật thể + trajectory vật thể) — đây là input cần thiết nếu sau này bạn làm loco-manipulation.
5. **(checkpoint)** Ghi vào literature map một bảng so sánh 4 dataset theo 4 tiêu chí: định dạng, quy mô, loại chuyển động, có phù hợp huấn luyện VLA loco-manipulation không.

---

## Liên kết chéo

- Output của bước này là input trực tiếp cho `02-motion-retargeting/`.
- Quy mô dữ liệu (BONES-SEED) là lý do SONIC scale được — liên hệ `01-whole-body-control/`.
- Nếu dataset có text annotation (Motion-X++, HumanML3D) → liên hệ trực tiếp `06-vla-groot-sonic/` (dữ liệu ngôn ngữ-chuyển động để huấn luyện VLA).
