# 03 — Human Motion Datasets

> Toàn bộ pipeline (retargeting → WBC → VLA) phụ thuộc vào chất lượng và quy mô dữ liệu chuyển động người. Hiểu rõ 4 dataset mentor liệt kê — chúng khác nhau về **định dạng**, **loại chuyển động**, và **vai trò trong pipeline**.
>
> 📖 **Giải thích chi tiết đầy đủ** (cơ chế body model, cấu trúc dữ liệu, cách xây dựng từng dataset): xem `NOI-DUNG-CHI-TIET.md`.
>
> 🎓 **Bài giảng đã có**: [`BAI-GIANG-smpl-body-model.md`](BAI-GIANG-smpl-body-model.md) — cơ chế toán học của body model SMPL (shape/pose blend shapes, Linear Blend Skinning).
>
> 🎓 **Bài giảng đã có**: [`BAI-GIANG-smpl-x-mo-rong.md`](BAI-GIANG-smpl-x-mo-rong.md) — SMPL-X mở rộng gì so với SMPL (bàn tay MANO, khuôn mặt FLAME, 119 tham số).
>
> 🎓 **Bài giảng đã có**: [`BAI-GIANG-amass-mosh.md`](BAI-GIANG-amass-mosh.md) — cách AMASS hợp nhất 15 bộ mocap bằng MoSh/MoSh++, số liệu chính thức và giới hạn dữ liệu.
>
> 🎓 **Bài giảng đã có**: [`BAI-GIANG-omomo.md`](BAI-GIANG-omomo.md) — bài toán object motion guided human motion synthesis và kiến trúc diffusion 2 bước của OMOMO.
>
> 🎓 **Bài giảng đã có**: [`BAI-GIANG-lafan1-dinh-dang-bvh.md`](BAI-GIANG-lafan1-dinh-dang-bvh.md) — cấu trúc file BVH (HIERARCHY + MOTION), 3 khác biệt với SMPL/SMPL-X, quy trình mocap LAFAN1 (Ubisoft La Forge).
>
> 🎓 **Bài giảng đã có**: [`BAI-GIANG-bones-seed.md`](BAI-GIANG-bones-seed.md) — quy mô 142.220 chuyển động/288 giờ, vai trò công cụ của NVIDIA (SOMA-retargeter + Kimodo), bản retarget G1 sẵn có.

---

## A. Bốn dataset chính

| Dataset | Định dạng | Nội dung | Quy mô | Vai trò trong pipeline | Link |
|---|---|---|---|---|---|
| **AMASS** | SMPL/SMPL-X (mesh + skeleton tham số hoá) | Hợp nhất 15 bộ mocap optical marker khác nhau thành 1 định dạng thống nhất | >40 giờ, 11,000+ chuyển động, 300+ chủ thể *(đã sửa từ "480+" — số liệu cũ sai; nguyên văn abstract paper gốc: "over 300 subjects")* | Nguồn chuyển động người **đa dạng** nhất, dùng làm input chính cho GMR/SOMA-retargeter | [amass.is.tue.mpg.de](https://amass.is.tue.mpg.de) (cần đăng ký) |
| **SMPL-X** | *(không phải dataset, mà là body model)* | Mô hình tham số hoá cơ thể người (dáng người + tư thế + bàn tay + mặt) mà AMASS/OMOMO dùng để biểu diễn chuyển động | — | Là "ngôn ngữ chung" để biểu diễn skeleton người trước khi retarget | [smpl-x.is.tue.mpg.de](https://smpl-x.is.tue.mpg.de) |
| **OMOMO** | SMPL-H/SMPL-X + object trajectory + 3D mesh vật thể | Mocap **tương tác người–vật thể** (Stanford, SIGGRAPH Asia 2023): 15 loại vật thể, ~10 giờ | ~10 giờ | Dữ liệu duy nhất trong 4 loại có **loco-manipulation** (vừa di chuyển vừa cầm/mang vật) — quan trọng cho VLA loco-manipulation của SONIC/GR00T | [Trang dự án](https://lijiaman.github.io/projects/omomo/) · [GitHub](https://github.com/lijiaman/omomo_release) |
| **LAFAN1** | BVH (skeleton khớp góc, không mesh) | Mocap chất lượng sản xuất game (Ubisoft): đi, chạy, nhảy múa, đánh nhau, 5 diễn viên, 15 loại động tác | 496,672 frame @ 30Hz | Chuyển động **tự nhiên, mượt, chất lượng cao** — chuẩn benchmark phổ biến cho motion tracking RL (nhiều paper WBC dùng LAFAN1 làm test set) | [GitHub Ubisoft](https://github.com/ubisoft/ubisoft-laforge-animation-dataset) |
| **BONES-SEED** | Công bố bởi **Bones Studio** (không phải nội bộ NVIDIA — đã sửa; NVIDIA chỉ đóng góp công cụ SOMA-retargeter + temporal segmentation cho dự án Kimodo), gated trên Hugging Face (`bones-studio/seed`) | 142,220 chuyển động (71,132 gốc + 71,088 mirror), 522 diễn viên, có sẵn bản **đã retarget sang G1** (định dạng CSV, tương thích MuJoCo) + annotation ngôn ngữ tự nhiên | ~288 giờ (tính ở 120fps) | Quy mô lớn nhất trong nhóm, dùng để huấn luyện SONIC ở quy mô lớn — minh hoạ vì sao retargeting phải làm *trước*, ở quy mô lớn, không phải thủ công từng file | Công bố gần đây (2026), cần chấp nhận điều khoản license (`bones-seed-license`, liên hệ `licensing@bones.studio`) — **cần kiểm tra license/tình trạng phát hành khi bạn tới Phase 1** |

> 💡 **Cần research thêm** (mentor lưu ý): `humanoid-wbc-review` còn liệt kê **Motion-X++** (dataset chuyển động quy mô lớn khác, có text annotation) — đáng xem nếu bạn cần chuyển động có mô tả ngôn ngữ đi kèm (liên quan trực tiếp tới huấn luyện VLA ở `06-vla-groot-sonic/`). Ngoài ra còn **HumanML3D**, **KIT-ML** (dataset motion-language phổ biến trong literature review học thuật) — tìm qua GMR/awesome-humanoid-robot-learning để mở rộng danh sách.

---

## B. Paper gốc của từng dataset/định dạng — đọc để hiểu chính xác cách dữ liệu được xây dựng

| Tài liệu | Tầng | Vì sao đọc | Link |
|---|---|---|---|
| **Loper, Mahmood, Romero, Pons-Moll, Black (2015)** — *"SMPL: A Skinned Multi-Person Linear Model"*, ACM ToG (SIGGRAPH Asia) 34(6) | 🔴 | Paper gốc của body model SMPL — hiểu cách một vector tham số ngắn (pose + shape) sinh ra toàn bộ mesh cơ thể. Nền của SMPL-X, và gián tiếp là nền của AMASS/OMOMO. | [smpl.is.tue.mpg.de](https://smpl.is.tue.mpg.de) |
| **Pavlakos, Choutas, Ghorbani, Bolkart, Osman, Tzionas, Black (2019)** — *"Expressive Body Capture: 3D Hands, Face, and Body from a Single Image"*, CVPR 2019 | 🔴 | Paper gốc của **SMPL-X** — mở rộng SMPL thêm bàn tay + khuôn mặt. Đây là định dạng chính AMASS/OMOMO dùng, phải đọc để hiểu chính xác từng tham số trước khi retarget. | [CVF Open Access](https://openaccess.thecvf.com/content_CVPR_2019/html/Pavlakos_Expressive_Body_Capture_3D_Hands_Face_and_Body_From_a_CVPR_2019_paper.html) |
| **Mahmood, Ghorbani, Troje, Pons-Moll, Black (2019)** — *"AMASS: Archive of Motion Capture as Surface Shapes"*, ICCV 2019 | 🔴 | Paper gốc của dataset AMASS — cách 15 bộ mocap khác nhau được hợp nhất vào 1 tham số hoá SMPL chung. Đọc để hiểu rõ giới hạn/thiên lệch dữ liệu (subject nào, loại chuyển động nào chiếm đa số) trước khi dùng làm nguồn huấn luyện chính. | [CVF Open Access](https://openaccess.thecvf.com/content_ICCV_2019/papers/Mahmood_AMASS_Archive_of_Motion_Capture_As_Surface_Shapes_ICCV_2019_paper.pdf) |
| **Li, Clegg, Mottaghi, Wu, Puig, Liu (2023)** — *"Object Motion Guided Human Motion Synthesis"* (OMOMO), ACM ToG (SIGGRAPH Asia 2023) | 🔴 | Paper gốc OMOMO — mô tả chính xác cách thu thập chuyển động người tương tác vật thể, và bài toán "sinh chuyển động người từ chuyển động vật thể" mà dataset này phục vụ. | [Trang dự án](https://lijiaman.github.io/projects/omomo/) |
| **Harvey, Yurick, Nowrouzezahrai, Pal (2020)** — *"Robust Motion In-Betweening"*, ACM ToG (SIGGRAPH) 39(4) | 🔴 | Paper gốc tạo ra dataset **LAFAN1** — đọc để hiểu tiêu chuẩn chất lượng mocap (5 diễn viên, motion-capture chất lượng sản xuất game) mà dataset này đại diện. | [Ubisoft La Forge](https://www.ubisoft.com/en-us/studio/laforge/news/2NBPwJzPl3DwCAzGTav7Tg/robust-motion-inbetweening) |
| BVH format spec (tài liệu giới thiệu ngắn, ví dụ trên trang GMR) | 🟡 | BVH là định dạng "cây khớp + góc Euler theo frame" — đơn giản hơn SMPL nhưng phổ biến trong game/mocap công nghiệp (LAFAN1 dùng BVH). | [GitHub GMR](https://github.com/YanjieZe/GMR) |

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
