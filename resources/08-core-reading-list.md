# 08 — Danh sách paper nền tảng (Core Reading List)

*Cập nhật: 2026-07-29. **Mọi arXiv ID trong file này đã được kiểm chứng tự động** — link trỏ đúng paper ghi trong bảng.*

`resources/00`–`05` cho bạn **sách và khoá học**. File này cho bạn thứ mà ROADMAP thực sự yêu cầu ở Phase 1: **paper cụ thể để đọc**, xếp theo thứ tự, có lý do.

---

## Cách dùng file này

**Ba tầng ưu tiên:**

| Tầng | Ý nghĩa | Cách đọc |
|---|---|---|
| 🔴 **T1** | Phải đọc kỹ (pass-3). Không đọc thì không hiểu được các paper khác trong ngành. | Note đầy đủ trong `templates/paper-reading-note.md` |
| 🟡 **T2** | Nên đọc pass-2. Biết ý tưởng chính, biết khi nào cần quay lại. | Note ngắn 5–10 dòng |
| ⚪ **T3** | Pass-1 là đủ. Biết nó tồn tại và giải bài toán gì. | Một dòng trong literature map |

**Quy tắc:** đừng đọc tuyến tính từ trên xuống. Đọc **hết 🔴 T1 của Track A + Track 0**, rồi nhảy sang track chuyên môn của bạn. Mỗi tuần: 1 paper pass-3 + 3–5 paper pass-1/2 (đúng như ROADMAP Phase 1).

**Mục tiêu Phase 1 (Tháng 1–3):** đọc kỹ ≥10 paper 🔴, quét ≥30 paper. Ghi vào `templates/literature-map.md`.

---

## Track 0 — Craft of research (đọc TRƯỚC mọi paper kỹ thuật)

| # | Tài liệu | Tầng | Vì sao |
|---|---|---|---|
| 0.1 | **Hamming — "You and Your Research"** — [link](https://www.cs.virginia.edu/~robins/YouAndYourResearch.html) | 🔴 | Chọn bài toán quan trọng. Đọc lại mỗi 6 tháng. |
| 0.2 | **Keshav — "How to Read a Paper"** — [PDF](http://ccr.sigcomm.org/online/files/p83-keshavA.pdf) | 🔴 | Three-pass method. Dùng ngay cho mọi paper dưới đây. |
| 0.3 | **Lipton & Steinhardt — "Troubling Trends in ML Scholarship"** — [arXiv:1807.03341](https://arxiv.org/abs/1807.03341) | 🔴 | Bốn bệnh của paper ML. Sau khi đọc, bạn sẽ đọc mọi paper khác bằng mắt khác. |
| 0.4 | **Kapoor & Narayanan — "Leakage and the Reproducibility Crisis"** — [arXiv:2207.07048](https://arxiv.org/abs/2207.07048) | 🔴 | 8 loại rò rỉ dữ liệu. Cực kỳ liên quan tới y tế. |
| 0.5 | **Musgrave et al. — "A Metric Learning Reality Check"** — [arXiv:2003.08505](https://arxiv.org/abs/2003.08505) | 🟡 | Bài học: so sánh không công bằng làm bốc hơi cả một thập kỷ "tiến bộ". |
| 0.6 | **Picard — "torch.manual_seed(3407) is all you need"** — [arXiv:2109.08203](https://arxiv.org/abs/2109.08203) | 🟡 | Vì sao một seed là vô nghĩa. |
| 0.7 | **Sutton — "The Bitter Lesson"** (blog, ~1 trang) | 🟡 | Tìm "Rich Sutton The Bitter Lesson". Quan điểm gây tranh cãi nhưng định hình cả ngành — cần biết để có ý kiến riêng. |

---

## Track A — Nền tảng Deep Learning & Vision (bắt buộc cho mọi hướng)

### A1. Kiến trúc & huấn luyện — nền không thể thiếu

| # | Paper | Tầng | Vì sao đọc |
|---|---|---|---|
| A1.1 | **ResNet** — Deep Residual Learning — [arXiv:1512.03385](https://arxiv.org/abs/1512.03385) | 🔴 | Skip connection. Nền của gần như mọi backbone. Vẫn là baseline sống. |
| A1.2 | **U-Net** — Convolutional Networks for Biomedical Image Segmentation — [arXiv:1505.04597](https://arxiv.org/abs/1505.04597) | 🔴 | **Paper quan trọng nhất của ảnh y tế.** Encoder–decoder + skip. Đọc *rất* kỹ — mọi thứ sau này là biến thể. |
| A1.3 | **Attention Is All You Need** — [arXiv:1706.03762](https://arxiv.org/abs/1706.03762) | 🔴 | Transformer. Nền của ViT, VLM, foundation model. |
| A1.4 | **ViT** — An Image is Worth 16x16 Words — [arXiv:2010.11929](https://arxiv.org/abs/2010.11929) | 🔴 | Transformer cho ảnh. Cần cho UNETR/Swin UNETR/SAM. |
| A1.5 | **ConvNeXt** — A ConvNet for the 2020s — [arXiv:2201.03545](https://arxiv.org/abs/2201.03545) | 🟡 | Phản biện lành mạnh: CNN thiết kế đúng vẫn ngang ViT. Bài học về **so sánh công bằng**. |
| A1.6 | **DETR** — End-to-End Object Detection with Transformers — [arXiv:2005.12872](https://arxiv.org/abs/2005.12872) | 🟡 | Set prediction, bỏ NMS/anchor. Ý tưởng lan sang detection dụng cụ phẫu thuật. |

### A2. Học biểu diễn & foundation model

| # | Paper | Tầng | Vì sao đọc |
|---|---|---|---|
| A2.1 | **MAE** — Masked Autoencoders Are Scalable Vision Learners — [arXiv:2111.06377](https://arxiv.org/abs/2111.06377) | 🔴 | Self-supervised pretraining. **Cực quan trọng cho y tế** — nơi nhãn đắt. Nền của EndoViT và nhiều surgical FM. |
| A2.2 | **CLIP** — Learning Transferable Visual Models From NL Supervision — [arXiv:2103.00020](https://arxiv.org/abs/2103.00020) | 🔴 | Ảnh ↔ text. Nền của mọi VLM y tế/phẫu thuật (SurgVLP, CONCH…). |
| A2.3 | **DINOv2** — Robust Visual Features without Supervision — [arXiv:2304.07193](https://arxiv.org/abs/2304.07193) | 🟡 | SSL feature dùng được ngay không cần fine-tune. Baseline mạnh, hay bị bỏ qua. |
| A2.4 | **SAM** — Segment Anything — [arXiv:2304.02643](https://arxiv.org/abs/2304.02643) | 🔴 | Promptable segmentation. Đọc kỹ *cả phần data engine* — bài học về xây dataset. |
| A2.5 | **SAM 2** — Segment Anything in Images and Videos — [arXiv:2408.00714](https://arxiv.org/abs/2408.00714) | 🔴 | Thêm **memory cho video** → nền cho segmentation video nội soi. Trực tiếp liên quan surgical vision. |

### A3. Sinh ảnh (diffusion) — mạnh trong y tế: tăng cường dữ liệu, tái tạo, chuyển modality

| # | Paper | Tầng | Vì sao đọc |
|---|---|---|---|
| A3.1 | **DDPM** — Denoising Diffusion Probabilistic Models — [arXiv:2006.11239](https://arxiv.org/abs/2006.11239) | 🟡 | Nền toán của diffusion. |
| A3.2 | **Latent Diffusion (Stable Diffusion)** — [arXiv:2112.10752](https://arxiv.org/abs/2112.10752) | 🟡 | Diffusion trong không gian latent → khả thi cho ảnh 3D độ phân giải cao. |

---

## Track B — Medical Imaging (trục chính đề xuất)

### B1. Segmentation — xương sống của ngành

| # | Paper | Tầng | Vì sao đọc |
|---|---|---|---|
| B1.1 | **V-Net** — [arXiv:1606.04797](https://arxiv.org/abs/1606.04797) | 🔴 | U-Net cho **3D** + **Dice loss**. Bạn sẽ dùng Dice loss suốt đời nghiên cứu. |
| B1.2 | **nnU-Net** — [arXiv:1809.10486](https://arxiv.org/abs/1809.10486) (bản gốc) và **Nature Methods 2021** — Isensee et al., *"nnU-Net: a self-configuring method for deep learning-based biomedical image segmentation"* | 🔴🔴 | **Paper phải đọc kỹ nhất của Track B.** Thông điệp: pipeline được cấu hình tốt thắng kiến trúc "mới". Là **baseline bắt buộc** trong mọi paper segmentation y tế của bạn. |
| B1.3 | **Attention U-Net** — [arXiv:1804.03999](https://arxiv.org/abs/1804.03999) | 🟡 | Attention gate — biến thể được trích dẫn nhiều nhất. |
| B1.4 | **UNETR** — [arXiv:2103.10504](https://arxiv.org/abs/2103.10504) | 🟡 | Transformer encoder cho 3D. Có sẵn trong MONAI. |
| B1.5 | **Swin UNETR** — [arXiv:2201.01266](https://arxiv.org/abs/2201.01266) | 🟡 | Hierarchical transformer, SOTA BraTS một thời. Trong MONAI. |
| B1.6 | **TransUNet** — [arXiv:2102.04306](https://arxiv.org/abs/2102.04306) | ⚪ | Hybrid CNN-Transformer. Biết để đọc related work. |
| B1.7 | **TotalSegmentator** — [arXiv:2208.05868](https://arxiv.org/abs/2208.05868) | 🔴 | 104 cấu trúc giải phẫu trong CT, model + dataset mở. **Công cụ bạn sẽ dùng thật**, không chỉ đọc. |

### B2. Foundation model cho y tế (hướng nóng nhất hiện nay)

| # | Paper | Tầng | Vì sao đọc |
|---|---|---|---|
| B2.1 | **MedSAM** — Segment Anything in Medical Images — [arXiv:2304.12306](https://arxiv.org/abs/2304.12306) (Nature Communications 2024) | 🔴 | Điều chỉnh SAM cho y tế. Điểm khởi đầu cho rất nhiều project Phase 3. |
| B2.2 | **MedSAM2** — Segment Anything in 3D Medical Images and Videos — [arXiv:2504.03600](https://arxiv.org/abs/2504.03600) | 🔴 | Bản 3D + video (2025). Nếu bạn nhắm giao imaging ∩ surgical video → đọc kỹ. |
| B2.3 | **nnInteractive** — Redefining 3D Promptable Segmentation — [arXiv:2503.08373](https://arxiv.org/abs/2503.08373) | 🟡 | Segmentation tương tác 3D (click/scribble/lasso) — hướng thực dụng, dùng được trong lâm sàng. |
| B2.4 | **Models Genesis** — [arXiv:1908.06912](https://arxiv.org/abs/1908.06912) | 🟡 | Self-supervised pretraining chuyên cho ảnh y tế 3D. Kinh điển, vẫn hữu ích. |
| B2.5 | **RETFound** — Zhou et al., *Nature* 2023, "A foundation model for generalizable disease detection from retinal images" | 🟡 | Foundation model y tế đăng ở Nature — mẫu mực về cách *chứng minh* giá trị của FM y tế. |
| B2.6 | **UNI** & **CONCH** — Chen et al. / Lu et al., *Nature Medicine* 2024 (bệnh học số) | ⚪ | Nếu bạn quan tâm pathology. Mẫu mực về scale dữ liệu + đánh giá đa tác vụ. |

### B3. Registration & Reconstruction

| # | Paper | Tầng | Vì sao đọc |
|---|---|---|---|
| B3.1 | **VoxelMorph** — [arXiv:1802.02604](https://arxiv.org/abs/1802.02604) | 🔴 | Registration biến dạng bằng học sâu, **unsupervised**. Nền cho image-guided intervention (khớp CT/MRI trước mổ với cảnh trong mổ). |
| B3.2 | **fastMRI** — [arXiv:1811.08839](https://arxiv.org/abs/1811.08839) | 🟡 | Dataset + benchmark tái tạo MRI. Đọc để hiểu bài toán inverse problem trong y tế. |
| B3.3 | **MoDL** — Model Based Deep Learning for Inverse Problems — [arXiv:1712.02862](https://arxiv.org/abs/1712.02862) | ⚪ | Kết hợp mô hình vật lý + học sâu. Đọc nếu đi hướng reconstruction. |

### B4. Độ tin cậy lâm sàng — Uncertainty, Fairness, Federated

| # | Paper | Tầng | Vì sao đọc |
|---|---|---|---|
| B4.1 | **Kendall & Gal** — What Uncertainties Do We Need in Bayesian DL for CV? — [arXiv:1703.04977](https://arxiv.org/abs/1703.04977) | 🔴 | Phân biệt **aleatoric vs epistemic** uncertainty. Bắt buộc nếu nói tới an toàn lâm sàng. |
| B4.2 | **Deep Ensembles** — [arXiv:1612.01474](https://arxiv.org/abs/1612.01474) | 🟡 | Baseline uncertainty mạnh nhất và đơn giản nhất. Rất khó bị đánh bại — nhớ dùng làm baseline. |
| B4.3 | **Seyyed-Kalantari et al.** — *Nature Medicine* 2021, "Underdiagnosis bias of AI algorithms applied to chest radiographs in under-served patient populations" | 🔴 | Bằng chứng model chẩn đoán **sót nhiều hơn ở nhóm yếu thế**. Đọc để hiểu vì sao báo cáo phân tầng là bắt buộc. |
| B4.4 | **Rieke et al.** — *npj Digital Medicine* 2020, "The future of digital health with federated learning" | 🟡 | Vì sao dữ liệu bệnh viện không ra khỏi bệnh viện, và làm gì với thực tế đó. |
| B4.5 | **Metrics Reloaded** — [arXiv:2206.01653](https://arxiv.org/abs/2206.01653) / [Nature Methods 2024](https://www.nature.com/articles/s41592-023-02151-z) | 🔴 | Chọn metric đúng. Xem chi tiết `07-experiments-and-rigor.md`. |
| B4.6 | **Understanding metric-related pitfalls** — [arXiv:2302.01790](https://arxiv.org/abs/2302.01790) · **Common Limitations of Image Processing Metrics** — [arXiv:2104.05642](https://arxiv.org/abs/2104.05642) | 🟡 | Bản kể-bằng-hình của các cạm bẫy metric. Nhanh, đáng nhớ. |
| B4.7 | **CheXNet** — [arXiv:1711.05225](https://arxiv.org/abs/1711.05225) | ⚪ | Kinh điển lịch sử: "radiologist-level". Đọc **kèm phần phản biện** của nó — bài học về tuyên bố vượt bằng chứng. |

---

## Track C — Robotics & Robot Learning (mức nền tảng để bắc cầu)

### C1. Perception & 3D — phần liên quan nhất tới surgical vision

| # | Paper | Tầng | Vì sao đọc |
|---|---|---|---|
| C1.1 | **ORB-SLAM3** — [arXiv:2007.11898](https://arxiv.org/abs/2007.11898) | 🔴 | SLAM chuẩn mực. Cần để hiểu định vị camera nội soi. |
| C1.2 | **DROID-SLAM** — [arXiv:2108.10869](https://arxiv.org/abs/2108.10869) | 🟡 | SLAM học sâu, bền hơn trên cảnh ít texture — đúng vấn đề của nội soi. |
| C1.3 | **NeRF** — [arXiv:2003.08934](https://arxiv.org/abs/2003.08934) | 🔴 | Biểu diễn 3D ngầm. Nền trực tiếp của EndoNeRF (D3.x). |
| C1.4 | **3D Gaussian Splatting** — [arXiv:2308.04079](https://arxiv.org/abs/2308.04079) | 🔴 | Thay thế NeRF, **thời gian thực** → khả thi trong mổ. Hướng nóng cho tái tạo mô biến dạng. |

### C2. Robot learning — manipulation & policy

| # | Paper | Tầng | Vì sao đọc |
|---|---|---|---|
| C2.1 | **Diffusion Policy** — [arXiv:2303.04137](https://arxiv.org/abs/2303.04137) | 🔴 | Policy đa phương thức bằng diffusion. **Đang là nền của rất nhiều paper autonomous surgical subtask.** |
| C2.2 | **ACT / ALOHA** — Learning Fine-Grained Bimanual Manipulation with Low-Cost Hardware — [arXiv:2304.13705](https://arxiv.org/abs/2304.13705) | 🔴 | Action chunking + imitation cho tác vụ **hai tay tinh xảo** — chính là bài toán phẫu thuật. |
| C2.3 | **RT-2** — [arXiv:2307.15818](https://arxiv.org/abs/2307.15818) | 🟡 | Vision-Language-Action. Biết để hiểu làn sóng VLA. |
| C2.4 | **OpenVLA** — [arXiv:2406.09246](https://arxiv.org/abs/2406.09246) | 🟡 | VLA mã nguồn mở — bản bạn thực sự chạy được. |
| C2.5 | **π₀** — A VLA Flow Model for General Robot Control — [arXiv:2410.24164](https://arxiv.org/abs/2410.24164) | 🟡 | Thế hệ VLA mới nhất. Đọc pass-2 để nắm hướng đi của ngành. |
| C2.6 | **Contact-GraspNet** — [arXiv:2103.14127](https://arxiv.org/abs/2103.14127) | ⚪ | Grasp 6-DoF. Nền cho gắp/kẹp mô. |

### C3. RL & Sim-to-real

| # | Paper | Tầng | Vì sao đọc |
|---|---|---|---|
| C3.1 | **PPO** — [arXiv:1707.06347](https://arxiv.org/abs/1707.06347) | 🟡 | Thuật toán RL mặc định. Biết để dùng, không cần đọc chứng minh. |
| C3.2 | **SAC** — Soft Actor-Critic — [arXiv:1801.01290](https://arxiv.org/abs/1801.01290) | 🟡 | RL off-policy cho control liên tục — hay dùng trong SurRoL. |
| C3.3 | **Domain Randomization** — [arXiv:1703.06907](https://arxiv.org/abs/1703.06907) | 🟡 | Sim-to-real. Bắt buộc biết vì bạn sẽ huấn luyện trong mô phỏng phẫu thuật. |

---

## Track D — Surgical / Medical Robotics (mũi nhọn PhD đề xuất)

> **Đây là track quyết định hồ sơ PhD của bạn.** Đọc D1 trước tiên — chúng định nghĩa cả lĩnh vực.

### D1. Định hình lĩnh vực — đọc 5 bài này trước mọi thứ khác

| # | Paper | Tầng | Vì sao đọc |
|---|---|---|---|
| D1.1 | **Yang et al.** — *Science Robotics* 2017, "Medical robotics — Regulatory, ethical, and legal considerations for increasing levels of autonomy" — [DOI](https://www.science.org/doi/10.1126/scirobotics.aam8638) | 🔴🔴 | **Định nghĩa 6 mức tự chủ (Level 0–5)** cho robot y tế. Mọi paper surgical autonomy đều tham chiếu khung này. Đọc *rất* kỹ — nó cho bạn từ vựng để định vị nghiên cứu của mình. |
| D1.2 | **Maier-Hein et al.** — "Surgical Data Science: Enabling Next-Generation Surgery" — [arXiv:1701.06482](https://arxiv.org/abs/1701.06482) | 🔴 | Bài khai sinh khái niệm **Surgical Data Science**. Bản đồ toàn bộ lĩnh vực. |
| D1.3 | **Maier-Hein et al.** — "Surgical Data Science — from Concepts toward Clinical Translation" — [arXiv:2011.02284](https://arxiv.org/abs/2011.02284) (MedIA 2022) | 🔴 | Bản cập nhật: rào cản thực tế khi đưa vào lâm sàng. **Đây là nơi tìm câu hỏi PhD.** |
| D1.4 | **D'Ettorre et al.** — "Accelerating Surgical Robotics Research: 10 Years With the dVRK" — [arXiv:2104.09869](https://arxiv.org/abs/2104.09869) | 🔴 | Toàn cảnh nền tảng nghiên cứu chuẩn của ngành + ai đang làm gì. Dùng để **lần ra lab mục tiêu**. |
| D1.5 | **Jiang et al.** — "Robotic Ultrasound Imaging: State-of-the-Art and Future Perspectives" — [arXiv:2307.05545](https://arxiv.org/abs/2307.05545) | 🟡 | Review giao **robot ∩ ảnh y tế** rất sát sở trường bạn. Hướng ít cạnh tranh hơn phẫu thuật nội soi. |

### D2. Surgical scene understanding (lối vào dễ nhất cho paper đầu tiên)

| # | Paper | Tầng | Vì sao đọc |
|---|---|---|---|
| D2.1 | **EndoNet** — Twinanda et al., IEEE TMI 2017 — [arXiv:1602.03012](https://arxiv.org/abs/1602.03012) | 🔴 | Paper khai sinh **surgical phase recognition** + dataset **Cholec80**. Nền của cả nhánh nghiên cứu. |
| D2.2 | **TeCNO** — Multi-Stage TCN for Surgical Phase Recognition — [arXiv:2003.10751](https://arxiv.org/abs/2003.10751) | 🔴 | Mô hình hoá thời gian đúng cách. Baseline mạnh, dễ tái lập → **ứng viên tốt cho Phase 2 (reproduce)**. |
| D2.3 | **Rendezvous** — Nwoye et al., MedIA 2022 — [arXiv:2109.03223](https://arxiv.org/abs/2109.03223) | 🔴 | Nhận diện **action triplet** ⟨instrument, verb, target⟩ + dataset **CholecT50**. Bài toán giàu, chưa bão hoà. |
| D2.4 | **SurgVLP** — Learning Multi-modal Representations by Watching Hundreds of Surgical Video Lectures — [arXiv:2307.15220](https://arxiv.org/abs/2307.15220) | 🔴 | CLIP cho phẫu thuật, học từ video giảng trên YouTube. **Ý tưởng lấy dữ liệu cực thông minh** — bài học về vượt rào thiếu nhãn. |
| D2.5 | **Jaspers et al.** — "Scaling up self-supervised learning for improved surgical foundation models" — [arXiv:2501.09436](https://arxiv.org/abs/2501.09436) | 🔴 | So sánh **có kiểm soát** các surgical FM (EndoViT, GSViT…). Đọc để biết SOTA thật ở đâu, và để thấy một bài "so sánh nghiêm túc" trông thế nào. |

### D3. 3D & mô biến dạng trong nội soi (bài toán khó, mở, hợp nền vision của bạn)

| # | Paper | Tầng | Vì sao đọc |
|---|---|---|---|
| D3.1 | **EndoNeRF** — Neural Rendering for Stereo 3D Reconstruction of Deformable Tissues in Robotic Surgery — [arXiv:2206.15255](https://arxiv.org/abs/2206.15255) | 🔴 | NeRF cho mô mềm biến dạng. Giao chính xác giữa Track C1 và Track D. **Hướng rất tiềm năng cho đóng góp Phase 3.** |
| D3.2 | **SuPer** — A Surgical Perception Framework for Endoscopic Tissue Manipulation — [arXiv:1909.05405](https://arxiv.org/abs/1909.05405) | 🟡 | Theo dõi mô biến dạng + định vị dụng cụ trong một framework. |

> 💡 Sau khi đọc D3.1, tìm các paper **trích dẫn nó** (Semantic Scholar → "Citations") để thấy nhánh Gaussian-Splatting-cho-phẫu-thuật đang phát triển ra sao. Đây là cách tìm khoảng trống nghiên cứu **hiệu quả nhất**.

### D4. Autonomous surgical subtasks

| # | Paper | Tầng | Vì sao đọc |
|---|---|---|---|
| D4.1 | **Saeidi et al.** — *Science Robotics* 2022, "Autonomous robotic laparoscopic surgery for intestinal anastomosis" (**STAR**) | 🔴 | Cột mốc: khâu nối ruột tự chủ trên mô sống. Đọc để biết **trần thực tế** của tự chủ hiện nay ở đâu. |
| D4.2 | **Shademan et al.** — *Science Translational Medicine* 2016, "Supervised autonomous robotic soft tissue surgery" | 🟡 | Bài trước của STAR — đọc cặp với D4.1 để thấy tiến bộ 6 năm. |
| D4.3 | **SurRoL** — [arXiv:2108.13035](https://arxiv.org/abs/2108.13035) | 🔴 | Môi trường RL tương thích dVRK. **Bạn sẽ chạy nó thật** ở Phase 2–3 nếu không có phần cứng. |
| D4.4 | **ORBIT-Surgical** — [arXiv:2404.16027](https://arxiv.org/abs/2404.16027) | 🔴 | Framework học phẫu thuật trên Isaac Sim (GPU, ICRA 2024). Hiện đại hơn SurRoL — so sánh hai cái này rồi chọn. |

---

## Kế hoạch đọc 12 tuần (Phase 1) — bản gợi ý cụ thể

Mỗi tuần: **1 paper pass-3** (cột giữa) + **3–5 paper pass-1/2** (cột phải).

| Tuần | Pass-3 (đọc kỹ, có note đầy đủ) | Pass-1/2 (quét) |
|---|---|---|
| 1 | 0.1 Hamming + 0.2 Keshav | 0.3, 0.6 |
| 2 | A1.2 **U-Net** | A1.1 ResNet, B1.1 V-Net, B1.3 Attention U-Net |
| 3 | B1.2 **nnU-Net** (cả bản Nature Methods) | B1.4 UNETR, B1.5 Swin UNETR, B1.6 TransUNet |
| 4 | A1.3 **Attention Is All You Need** | A1.4 ViT, A1.5 ConvNeXt, A1.6 DETR |
| 5 | A2.4 **SAM** | A2.1 MAE, A2.2 CLIP, A2.3 DINOv2 |
| 6 | B2.1 **MedSAM** | A2.5 SAM 2, B2.2 MedSAM2, B2.3 nnInteractive, B1.7 TotalSegmentator |
| 7 | D1.1 **Levels of Autonomy (Yang 2017)** | D1.2, D1.3 Surgical Data Science (cả hai) |
| 8 | D1.4 **dVRK 10 years** | D1.5 Robotic US, D4.1 STAR, D4.2 Shademan |
| 9 | D2.1 **EndoNet** | D2.2 TeCNO, D2.3 Rendezvous |
| 10 | D2.4 **SurgVLP** | D2.5 Scaling SSL surgical FM, A2.1 MAE (đọc lại) |
| 11 | C1.3 **NeRF** | C1.4 3D Gaussian Splatting, C1.1 ORB-SLAM3, C1.2 DROID-SLAM |
| 12 | D3.1 **EndoNeRF** | D3.2 SuPer, D4.3 SurRoL, D4.4 ORBIT-Surgical |

**Song song từ tuần 1:** 0.3 → 0.4 → B4.5 (Metrics Reloaded). Ba bài này thay đổi cách bạn *đánh giá* mọi paper còn lại.

**Sau tuần 12 bạn sẽ có:** 12 note pass-3, ~40 paper trong literature map, và — quan trọng nhất — **2–3 câu hỏi mở cụ thể** để chọn làm project Phase 2/3.

---

## Cách mở rộng danh sách này (đừng chỉ đọc list của người khác)

1. **Đi ngược:** với mỗi paper 🔴, đọc 3–5 reference mà nó dựa vào nhiều nhất.
2. **Đi xuôi:** trên [Semantic Scholar](https://www.semanticscholar.org), xem ai **trích dẫn** nó — đó là biên giới hiện tại. Đây là cách tìm khoảng trống hiệu quả nhất.
3. **Bản đồ:** [Connected Papers](https://www.connectedpapers.com) cho một paper hạt giống → thấy ngay cả cụm nghiên cứu.
4. **Theo người, không theo từ khoá:** chọn 5–8 tác giả (từ D1.4 và các lab trong `09-community-labs-funding.md`), bật **Google Scholar alert** cho họ. Chất lượng cao hơn nhiều so với quét arXiv.
5. **Theo venue:** đọc **toàn bộ mục lục** MICCAI / IPCAI / MIDL năm gần nhất (chỉ tiêu đề + abstract, ~2 giờ). Bạn sẽ thấy ngay chủ đề nào đang đông, chủ đề nào trống.
6. **Theo challenge:** báo cáo tổng kết của EndoVis / BraTS / KiTS luôn có mục "hạn chế & hướng tương lai" — đó là danh sách câu hỏi mở được cộng đồng xác nhận.

> ⚠️ **Cảnh báo:** danh sách này sẽ cũ. Track D (surgical FM, 3D nội soi) thay đổi 6 tháng một lần. Coi file này là **điểm khởi đầu đã kiểm chứng**, không phải chân lý — và **tự cập nhật nó** mỗi cuối tháng. Đó chính là việc của một nhà nghiên cứu.

---

*Liên quan: `templates/paper-reading-note.md` · `templates/literature-map.md` · `07-experiments-and-rigor.md` (đọc phản biện phần đánh giá) · `09-community-labs-funding.md` (lần ra lab từ các paper trên).*
