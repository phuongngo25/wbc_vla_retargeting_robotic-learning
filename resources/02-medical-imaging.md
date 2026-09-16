# 02 — Medical Imaging (trục chính đề xuất)

Xử lý ảnh y tế: segmentation, detection, classification, registration, reconstruction trên CT/MRI/X-ray/siêu âm/nội soi/bệnh học số.

---

## A. Khóa học & sách

| Tài liệu | Nội dung | Link |
|---|---|---|
| **AI for Medical Diagnosis** (DeepLearning.AI, Coursera) | Vào cửa nhanh, thực hành DL cho chẩn đoán. | [coursera.org](https://www.coursera.org/specializations/ai-for-medicine) |
| **MICCAI Educational Challenge** | ⭐⭐ Tutorial do chính cộng đồng MICCAI viết (video, notebook, blog) — **tài liệu học tốt nhất và miễn phí của ngành**. Kho các năm trước là một khoá học hoàn chỉnh. Lần thứ 13 hướng tới MICCAI 2026 (Strasbourg). **Nên tự nộp một bài.** | [miccai-sb.github.io/challenge](https://miccai-sb.github.io/challenge.html) |
| **Medical Imaging with Deep Learning (MIDL)** — proceedings | Đọc paper của venue chuyên về DL + medical imaging. Toàn bộ mở, có review công khai (học được nhiều từ chính review). | [midl.io](https://midl.io) |
| **UCL — Medical Image Computing** (tài liệu mở) | Nền computing cho ảnh y tế. | [ucl.ac.uk/medical-image-computing](https://www.ucl.ac.uk/medical-image-computing/) |
| **"Deep Learning for Medical Image Analysis"** — Zhou, Greenspan, Shen (sách) | Tổng hợp kỹ thuật DL cho ảnh y tế. | [Elsevier](https://www.elsevier.com/books/deep-learning-for-medical-image-analysis/zhou/978-0-12-810408-8) |
| **MONAI Bootcamp & community meeting** | Học trực tiếp từ nhóm phát triển framework chuẩn ngành. Cũng là nơi để được cộng đồng nhận diện. | [Project-MONAI](https://github.com/Project-MONAI) |

> 📖 **Paper cụ thể để đọc** (kèm arXiv ID đã kiểm chứng, xếp theo thứ tự + kế hoạch 12 tuần): **`08-core-reading-list.md` Track B**.
> 📖 **Chọn metric & thẩm định đúng cách** (Metrics Reloaded, FUTURE-AI, CLAIM, rò rỉ dữ liệu): **`07-experiments-and-rigor.md`** — phần này là điều phân biệt paper y tế nghiêm túc với paper bị reject.

---

## B. Framework & repo cốt lõi (dùng ở Phase 2 để reproduce)

| Repo | Vì sao quan trọng | Link |
|---|---|---|
| **MONAI** | ⭐ Framework PyTorch chuẩn ngành cho healthcare imaging: transforms, models, pipelines. | [github.com/Project-MONAI/MONAI](https://github.com/Project-MONAI/MONAI) |
| **MONAI Tutorials** | Notebook thực hành từ cơ bản → nâng cao. | [github.com/Project-MONAI/tutorials](https://github.com/Project-MONAI/tutorials) |
| **nnU-Net** | ⭐ Baseline segmentation tự cấu hình, state-of-the-art nhiều năm. Bắt buộc biết. | [github.com/MIC-DKFZ/nnUNet](https://github.com/MIC-DKFZ/nnUNet) |
| **MedSAM / MedSAM2** | Segment Anything cho ảnh y tế. **MedSAM2** (2025) mở rộng sang **3D và video** — quan trọng nếu bạn đi hướng surgical video. | [github.com/bowang-lab/MedSAM](https://github.com/bowang-lab/MedSAM) · [medsam2.github.io](https://medsam2.github.io/) |
| **TotalSegmentator** | ⭐ Segment **104+ cấu trúc giải phẫu** trong CT bằng một lệnh. Dùng làm tiền xử lý, làm nhãn giả (pseudo-label), hoặc baseline. **Công cụ bạn sẽ dùng thật.** | [arXiv:2208.05868](https://arxiv.org/abs/2208.05868) |
| **MONAI Label** | Gán nhãn ảnh y tế có AI hỗ trợ, cắm vào 3D Slicer. Giải quyết đúng nút thắt của mọi project y tế: **thiếu nhãn**. | [github.com/Project-MONAI/MONAILabel](https://github.com/Project-MONAI/MONAILabel) |
| **TorchIO** | Tiền xử lý & augmentation ảnh y tế 3D. | [github.com/fepegar/torchio](https://github.com/fepegar/torchio) |
| **3D Slicer** | Phần mềm chuẩn để xem/annotate ảnh y tế. Cũng là nơi tốt để làm hình cho paper. | [slicer.org](https://www.slicer.org) |
| **SimpleITK / ITK** | Xử lý ảnh y tế nền tảng (registration, I/O, DICOM/NIfTI). | [simpleitk.org](https://simpleitk.org) |
| **MetricsReloaded** | ⭐ Cài đặt tham chiếu của framework chọn metric chuẩn ngành (Nature Methods 2024). **Dùng cái này thay vì tự viết Dice.** | [github.com/Project-MONAI/MetricsReloaded](https://github.com/Project-MONAI/MetricsReloaded) |

---

## C. Dataset & challenge (để reproduce và làm project)

| Nguồn | Nội dung | Link |
|---|---|---|
| **MedMNIST v2** | ⭐ **Bắt đầu ở đây.** 18 dataset y tế 2D/3D đã chuẩn hoá, nhẹ, tải trong vài phút. Hoàn hảo để chạy thử pipeline & học nhanh trước khi đụng dữ liệu thật nặng. | [medmnist.com](https://medmnist.com) |
| **Medical Segmentation Decathlon** | 10 tác vụ segmentation chuẩn — điểm khởi đầu lý tưởng cho Phase 2. | [medicaldecathlon.com](http://medicaldecathlon.com) |
| **Grand Challenge** | Nền tảng tổng hợp các challenge ảnh y tế. Bookmark và duyệt định kỳ. | [grand-challenge.org](https://grand-challenge.org) |
| **The Cancer Imaging Archive (TCIA)** | Kho ảnh ung thư công khai lớn. | [cancerimagingarchive.net](https://www.cancerimagingarchive.net) |
| **MICCAI Challenges** (BraTS, KiTS, AMOS, v.v.) | Bài toán chuẩn của cộng đồng. **Đọc báo cáo tổng kết challenge** — mục "limitations & future work" là danh sách câu hỏi mở đã được cộng đồng xác nhận. | tìm theo tên trên [grand-challenge.org](https://grand-challenge.org) |
| **PhysioNet / MIMIC-CXR** | X-quang ngực + báo cáo văn bản. ⚠️ Cần **credentialed access**: hoàn thành khoá CITI + ký DUA, mất vài tuần → **xin sớm**. | [physionet.org](https://physionet.org) |
| **fastMRI** | Dataset + benchmark tái tạo MRI (NYU). Cho hướng inverse problem. | [arXiv:1811.08839](https://arxiv.org/abs/1811.08839) |

> ⚠️ **Trước khi dùng bất kỳ dataset nào:** đọc license (một số cấm dùng thương mại / cấm redistribute), ghi lại **ngày tải + phiên bản**, và **lưu file split vào repo**. Xem `07-experiments-and-rigor.md` mục I.

---

## D. Venue (nơi công bố — nhắm ở Phase 4)

**Hội nghị:**

- **MICCAI** — hàng đầu về medical image computing & computer-assisted intervention. Deadline nộp thường ~tháng 2. **MICCAI 2026 tổ chức tại Strasbourg** (cùng thành phố với CAMMA). [Springer LNCS](https://link.springer.com/conference/miccai)
- **IPMI** — Information Processing in Medical Imaging (nhỏ, theory-heavy, uy tín rất cao, 2 năm/lần).
- **ISBI** — IEEE International Symposium on Biomedical Imaging (dễ vào hơn, tốt để bắt đầu).
- **MIDL** — Medical Imaging with Deep Learning (review công khai — đọc review để học).
- **CVPR / ICCV / ECCV / NeurIPS** — cho đóng góp CS/ML mạnh (thường có track/workshop y tế).

**Tạp chí:**

- **IEEE Transactions on Medical Imaging (TMI)** — top journal, giới hạn ~10 trang initial submission.
- **Medical Image Analysis (MedIA)** — Elsevier, top journal của ngành.
- **IEEE Journal of Biomedical and Health Informatics (JBHI)**.

> 💡 **Lối vào dễ nhất cho bài đầu tiên:** **workshop tại MICCAI/CVPR** hoặc **ISBI**. Đừng nhắm thẳng TMI/MICCAI main cho bài đầu.
>
> 💡 **Chiến thuật ít người dùng:** đọc **toàn bộ mục lục** MICCAI/MIDL năm gần nhất (chỉ tiêu đề + abstract, ~2 giờ). Bạn thấy ngay chủ đề nào đông, chủ đề nào trống — hiệu quả hơn nhiều so với tìm từ khoá.

---

## E. Hướng nghiên cứu đang nóng (để tìm câu hỏi)

- **Foundation model & promptable segmentation** cho y tế (SAM/SAM2 → MedSAM2, nnInteractive). Câu hỏi mở: khi nào FM *thua* nnU-Net chuyên biệt, và vì sao?
- **Học với ít nhãn:** self/semi-supervised, weakly-supervised — dữ liệu y tế đắt để gán nhãn. Đây là hướng bền vững nhất.
- **Domain generalization / robustness** giữa các máy scan, bệnh viện, quốc gia. Rất cần thiết về lâm sàng, dễ thiết kế thí nghiệm sạch.
- **Uncertainty quantification & calibration** — model biết khi nào nó không biết. Bắt buộc cho triển khai thật.
- **Multimodal** (ảnh + báo cáo text + genomics), sinh báo cáo tự động.
- **Federated learning** — dữ liệu bệnh viện không ra khỏi bệnh viện.
- **Fairness & bias:** hiệu năng phân tầng theo nhóm bệnh nhân. Có bằng chứng model chẩn đoán **sót nhiều hơn ở nhóm yếu thế**.
- **Validation methodology** — chọn metric, thiết kế đánh giá. Ít hào nhoáng nhưng **rất được trọng dụng**, và khớp hoàn hảo với tư duy kỹ sư của bạn. Xem `07-experiments-and-rigor.md`.

> 🎯 **Cách biến "hướng nóng" thành **câu hỏi** nghiên cứu:** chọn một hướng ở trên → đọc 5 paper mới nhất → tìm điều **cả 5 bài đều giả định mà không kiểm chứng**. Đó là câu hỏi của bạn.
