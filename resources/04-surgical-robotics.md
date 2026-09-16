# 04 — Surgical / Medical Robotics (điểm giao đề xuất làm mũi nhọn PhD)

Đây là giao của **vision + robotics + y tế** — đúng sở trường của bạn và là hướng PhD rất hấp dẫn: image-guided intervention, surgical vision, autonomous surgical subtasks.

---

## A. Nền tảng & tổng quan

| Tài liệu | Nội dung | Link |
|---|---|---|
| **Yang et al. — "Medical robotics: Regulatory, ethical, and legal considerations for increasing levels of autonomy"** (*Science Robotics* 2017) | ⭐⭐ **Đọc bài này TRƯỚC mọi thứ khác trong file.** Định nghĩa **6 mức tự chủ (Level 0–5)** cho robot y tế — khung từ vựng mà mọi paper surgical autonomy đều dùng. Nó cho bạn cách **định vị nghiên cứu của mình** trong lĩnh vực. | [DOI](https://www.science.org/doi/10.1126/scirobotics.aam8638) |
| **Maier-Hein et al. — "Surgical Data Science"** (2017) và **"...from Concepts toward Clinical Translation"** (MedIA 2022) | ⭐⭐ Cặp bài khai sinh & cập nhật khái niệm **Surgical Data Science**. Bài 2022 liệt kê **rào cản thực tế khi đưa vào lâm sàng** — đây là nơi tốt nhất để tìm câu hỏi PhD. | [arXiv:1701.06482](https://arxiv.org/abs/1701.06482) · [arXiv:2011.02284](https://arxiv.org/abs/2011.02284) |
| **"Accelerating Surgical Robotics Research: 10 Years With the dVRK"** | ⭐ Bài review nền tảng — bức tranh, lịch sử, **và danh sách các viện đang dùng dVRK** (dùng cái này để lần ra lab mục tiêu). | [arXiv:2104.09869](https://arxiv.org/abs/2104.09869) |
| **JHU — Computer Integrated Surgery I** (601.455/655, Russell Taylor) | ⭐⭐ **Khoá học chính thống của lĩnh vực**, do người sáng lập ngành dạy. Slide và tài liệu lecture công khai — dùng như giáo trình tự học. | [CIIS Wiki](https://ciis.lcsr.jhu.edu/doku.php?id=courses:455-655) · [cs.jhu.edu/cista/455](https://www.cs.jhu.edu/cista/455/) |
| **Jiang et al. — "Robotic Ultrasound Imaging: State-of-the-Art and Future Perspectives"** | Review giao **robot ∩ ảnh y tế**. Rất sát nền của bạn (xử lý ảnh + thiết bị y tế) và **ít cạnh tranh hơn** nội soi. Đáng cân nhắc làm hướng chính. | [arXiv:2307.05545](https://arxiv.org/abs/2307.05545) |
| **Springer Handbook of Robotics — chương Medical Robotics** | Tổng quan hàn lâm về robot y tế. | [Springer](https://link.springer.com/referencework/10.1007/978-3-319-32552-1) |
| **Intuitive Foundation Research Wiki** | Tài liệu kỹ thuật cộng đồng dVRK. | [research.intusurg.com](https://research.intusurg.com) |

> 📖 **Toàn bộ danh sách paper của track này** (surgical scene understanding, 3D nội soi, autonomous subtasks — kèm arXiv ID đã kiểm chứng + kế hoạch đọc 12 tuần): **`08-core-reading-list.md` Track D**.

---

## B. Nền tảng nghiên cứu & repo (dùng ở Phase 2–3)

| Nguồn | Vai trò | Link |
|---|---|---|
| **da Vinci Research Kit (dVRK)** | Nền tảng phần cứng/phần mềm mở chuẩn cho NC robot phẫu thuật (>40 viện dùng). | [Intuitive Foundation](https://www.intuitive-foundation.org/dvrk/) |
| **dVRK software (JHU/WPI)** | Stack điều khiển mã nguồn mở. | [github.com/jhu-dvrk](https://github.com/jhu-dvrk) |
| **AMBF / Surgical Robotics simulators** | Mô phỏng phẫu thuật (nếu không có phần cứng). | [github.com/WPI-AIM/ambf](https://github.com/WPI-AIM/ambf) |
| **SurRoL** | ⭐ Môi trường RL cho dVRK (học tự động hoá tác vụ). **Chạy được trên laptop** — điểm khởi đầu thực tế nhất khi không có phần cứng. | [github.com/med-air/SurRoL](https://github.com/med-air/SurRoL) · [arXiv:2108.13035](https://arxiv.org/abs/2108.13035) |
| **ORBIT-Surgical** | ⭐ Framework surgical robot learning trên Isaac Sim (GPU, ICRA 2024) — hiện đại hơn SurRoL. So sánh hai cái rồi chọn theo phần cứng bạn có. | [github.com/orbit-surgical](https://github.com/orbit-surgical) · [arXiv:2404.16027](https://arxiv.org/abs/2404.16027) |

> 💡 **Không có dVRK thì làm gì?** Đây là câu hỏi thực tế nhất. Ba hướng khả thi hoàn toàn không cần phần cứng:
>
> 1. **Surgical vision trên video công khai** (Cholec80, CholecT50, EndoVis) — phase recognition, instrument segmentation, action triplet. **Lối vào dễ nhất cho paper đầu tiên.**
> 2. **3D/depth từ video nội soi** (EndoNeRF, Gaussian Splatting cho mô biến dạng) — chỉ cần dataset + GPU.
> 3. **Mô phỏng** (SurRoL / ORBIT-Surgical) cho autonomous subtask.
>
> Hướng (1) và (2) khớp trực tiếp với nền vision/imaging của bạn — **đừng để việc thiếu robot làm bạn trì hoãn**.

---

## C. Dataset & challenge

| Nguồn | Nội dung | Link |
|---|---|---|
| **Cholec80 / CholecT50** | ⭐⭐ **Bắt đầu ở đây.** Video cắt túi mật — nhận diện pha (Cholec80) & action triplet ⟨instrument, verb, target⟩ (CholecT50). Dataset được dùng nhiều nhất của ngành → có baseline sẵn để so sánh, có cộng đồng để hỏi. | [CAMMA, Strasbourg](https://camma.unistra.fr/datasets/) |
| **EndoVis** (MICCAI sub-challenges) | Instrument segmentation/tracking, scene understanding trong nội soi. Nhiều sub-challenge qua các năm — đọc báo cáo tổng kết để thấy câu hỏi mở. | [endovis.grand-challenge.org](https://endovis.grand-challenge.org) |
| **JIGSAWS** | Dataset cử chỉ & đánh giá kỹ năng phẫu thuật (dVRK) — có cả kinematics lẫn video. Kinh điển cho surgical skill assessment. | [JHU LCSR](https://cirl.lcsr.jhu.edu/research/hmm/datasets/jigsaws_release/) |
| **SAR-RARP50 / SurgToolLoc / HeiChole / PETRAW** | Các challenge nhận thức cảnh phẫu thuật gần đây. Tìm theo tên — mỗi cái là một bài toán đã có benchmark. | tìm trên [grand-challenge.org](https://grand-challenge.org) |
| **CAMMA — toàn bộ dataset** | Nhóm Strasbourg công khai nhiều dataset nhất trong ngành. Bookmark trang này. | [camma.unistra.fr](https://camma.unistra.fr) |

---

## D. Venue

- **MICCAI** (đặc biệt các track/workshop **CAI — Computer Assisted Intervention**, **AE-CAI**). **MICCAI 2026 tại Strasbourg.**
- **IPCAI** — Int. Conf. on Information Processing in Computer-Assisted Interventions (đúng trọng tâm surgical, thường gắn với MICCAI). [ipcai.org](https://www.ipcai.org)
- **ICRA / IROS** — track medical robotics.
- **Hamlyn Symposium on Medical Robotics** (Imperial College) — chuyên surgical robotics, không khí thân thiện với người mới. [hamlynsymposium.org](https://hamlynsymposium.org)
- Tạp chí: **IEEE TMI**, **IEEE T-RO / T-MRB (Transactions on Medical Robotics and Bionics)**, **IJCARS (Int. J. of Computer Assisted Radiology and Surgery)**, **Medical Image Analysis**.

> 💡 **Lối vào cho bài đầu tiên:** workshop MICCAI (CAI/AE-CAI) hoặc **IJCARS** — IJCARS dễ vào hơn TMI và là tạp chí "nhà" của cộng đồng IPCAI.

---

## E. Hướng nghiên cứu đang nóng (tìm câu hỏi PhD)

Xếp theo **độ khả thi cho bạn ngay bây giờ** (không cần phần cứng, nền vision sẵn có):

| Hướng | Khả thi ngay? | Ghi chú |
|---|---|---|
| **Surgical scene understanding** — segmentation/tracking dụng cụ, nhận diện pha, action triplet | ✅ Rất khả thi | Chỉ cần Cholec80/CholecT50 + GPU. **Lối vào tốt nhất cho paper đầu tiên.** Đã đông nhưng vẫn còn khoảng trống ở generalization giữa bệnh viện. |
| **Depth & 3D reconstruction trong nội soi** — mô mềm, biến dạng, ít texture | ✅ Rất khả thi | ⭐ Bài toán **khó và mở**, đang chuyển từ NeRF sang Gaussian Splatting. Khớp hoàn hảo nền vision của bạn. Xem EndoNeRF trong `08-core-reading-list.md`. |
| **Foundation model cho surgical vision** — SAM2/video, ít nhãn | ✅ Khả thi | Đang rất nóng (SurgVLP, EndoViT, SurgVLM). Cạnh tranh cao về scale dữ liệu → **đừng thi scale, hãy thi đánh giá/phân tích**. |
| **Image-guided intervention** — registration CT/MRI trước mổ với cảnh nội soi thời gian thực | ⚠️ Cần dữ liệu ghép cặp | Giá trị lâm sàng rất cao, ít người làm được. Cần cộng tác bệnh viện. Hướng PhD tuyệt vời nhưng khó bắt đầu một mình. |
| **Surgical skill assessment** & AI hỗ trợ đào tạo | ✅ Khả thi | JIGSAWS có sẵn. Dataset nhỏ → phải rất kỷ luật về thống kê (xem `07-experiments-and-rigor.md`). |
| **Autonomous surgical subtasks** — khâu, cắt, kéo mô | ⚠️ Cần mô phỏng | Làm được qua SurRoL/ORBIT-Surgical. Khoảng cách sim-to-real là câu hỏi mở lớn. |
| **Safety, uncertainty & human-in-the-loop autonomy** | ✅ Khả thi | ⭐ **Ít cạnh tranh, giá trị cao, khớp tư duy kỹ sư thiết bị y tế của bạn.** Dùng khung Level 0–5 của Yang et al. để định vị. |
| **Robotic ultrasound** | ✅ Khả thi | Giao robot ∩ imaging, ít đông hơn nội soi. Xem review arXiv:2307.05545. |

> 💡 **Điểm mạnh hồ sơ của bạn:** ít người có *đồng thời* nền vision/imaging + kỹ sư robot/thiết bị y tế. Hãy khai thác đúng giao điểm này khi viết SOP — và chọn hướng ở **hai dòng có ⭐** phía trên, nơi bạn có lợi thế thật, không chỉ hứng thú.

---

*Liên quan: `08-core-reading-list.md` Track D (paper cụ thể + kế hoạch 12 tuần) · `09-community-labs-funding.md` (lab surgical robotics & mentorship RISE-MICCAI) · `02-medical-imaging.md` (trục chính) · `03-robotics.md` (nền robotics cần thiết).*
