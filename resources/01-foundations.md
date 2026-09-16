# 01 — Nền tảng: Toán · Machine Learning · Computer Vision

Bạn đã là AI Engineer nên **không cần học lại từ đầu**. Dùng file này để lấp lỗ hổng có mục tiêu (targeted gaps), đủ để đọc và làm nghiên cứu.

---

## A. Toán cho ML (chỉ đọc chương còn yếu)

| Tài liệu | Nội dung | Link |
|---|---|---|
| **Mathematics for Machine Learning** — Deisenroth, Faisal, Ong | Linear algebra, calculus, probability, optimization — đúng phần ML cần. Miễn phí PDF. | [mml-book.github.io](https://mml-book.github.io) |
| **Linear Algebra** — MIT 18.06 (Gilbert Strang) | Nếu cần chắc lại đại số tuyến tính. | [MIT OCW](https://ocw.mit.edu/courses/18-06-linear-algebra-spring-2010/) |
| **Probability** — "Introduction to Probability" (Blitzstein, Harvard Stat110) | Xác suất trực giác, rất tốt cho ML. | [stat110.net](https://projects.iq.harvard.edu/stat110) |
| **Convex Optimization** — Boyd & Vandenberghe | Nền cho tối ưu (đọc khi cần cho control/robotics). | [Stanford EE364a](https://web.stanford.edu/~boyd/cvxbook/) |

---

## B. Machine Learning / Deep Learning

| Tài liệu | Nội dung | Link |
|---|---|---|
| **CS231n — Deep Learning for Computer Vision** (Stanford) | ⭐ Trục CV cốt lõi. Làm **assignments** để hiểu sâu backprop, CNN, training. | [cs231n.stanford.edu](https://cs231n.stanford.edu) |
| **Deep Learning for Computer Vision** — Justin Johnson (Michigan EECS 498/598) | ⭐⭐ Bản giảng hiện đại hơn CS231n, **video công khai đầy đủ** (Justin Johnson là đồng tác giả CS231n gốc). Nếu chỉ chọn một khoá CV — chọn cái này. | Tìm "Justin Johnson Deep Learning for Computer Vision" trên YouTube |
| **Neural Networks: Zero to Hero** — Andrej Karpathy | ⭐ Xây backprop/transformer **từ số 0** bằng code, live. Không có cách nào hiểu sâu hơn. Làm trước khi đọc paper transformer. | [karpathy.ai/zero-to-hero](https://karpathy.ai/zero-to-hero.html) |
| **Deep Learning: Foundations and Concepts** — Bishop & Bishop (2024) | ⭐ Sách giáo khoa DL **hiện đại nhất** — thay thế Goodfellow cho phần lý thuyết. Có bản đọc online. | [bishopbook.com](https://www.bishopbook.com) |
| **MIT 6.S191 — Introduction to Deep Learning** | Khoá nhanh, cập nhật mỗi năm, có lecture về AI y sinh. Tốt để bắt kịp trong 2 tuần. | [introtodeeplearning.com](http://introtodeeplearning.com) |
| **Stanford CS25 — Transformers United** | Seminar mời tác giả các paper transformer lớn tự trình bày. Bắt kịp hiện đại nhanh nhất. | [web.stanford.edu/class/cs25](https://web.stanford.edu/class/cs25/) |
| **Dive into Deep Learning (D2L)** | Sách tương tác, có code (PyTorch), miễn phí. | [d2l.ai](https://d2l.ai) |
| **fast.ai — Practical Deep Learning** | Cách tiếp cận code-first, bổ trợ CS231n. | [course.fast.ai](https://course.fast.ai) |
| **CS229 — Machine Learning** (Andrew Ng, Stanford) | Nền ML cổ điển (nếu cần chắc lại lý thuyết). | [cs229.stanford.edu](https://cs229.stanford.edu) |
| **Deep Learning** — Goodfellow, Bengio, Courville | Sách tham khảo lý thuyết (2016 — đã cũ ở phần kiến trúc, vẫn tốt ở phần nền). | [deeplearningbook.org](https://www.deeplearningbook.org) |
| **Probabilistic ML** — Kevin Murphy | Tham khảo nâng cao, hiện đại. | [probml.github.io](https://probml.github.io/pml-book/) |

**Chủ đề hiện đại nên nắm** — đọc **paper gốc**, đã có danh sách kèm arXiv ID đã kiểm chứng trong **`08-core-reading-list.md` Track A**:

- Transformer & ViT → attention, tokenization ảnh.
- Self-supervised learning (MAE, DINOv2) → **quan trọng nhất cho y tế**, nơi nhãn đắt.
- Vision-language (CLIP) → nền của mọi foundation model y tế/phẫu thuật.
- Promptable segmentation (SAM, SAM 2) → nền của MedSAM/MedSAM2.
- Diffusion models (DDPM, Latent Diffusion) → sinh dữ liệu, tái tạo, chuyển modality.

---

## C. Computer Vision cổ điển (đừng bỏ — rất cần cho robotics & imaging)

| Tài liệu | Nội dung | Link |
|---|---|---|
| **Computer Vision: Algorithms and Applications** — Richard Szeliski | Sách CV chuẩn mực, miễn phí PDF (2nd ed.). | [szeliski.org/Book](https://szeliski.org/Book/) |
| **Multiple View Geometry** — Hartley & Zisserman | Hình học đa góc nhìn — nền cho SLAM & surgical vision. | [robots.ox.ac.uk](https://www.robots.ox.ac.uk/~vgg/hzbook/) |
| **First Principles of Computer Vision** (Shree Nayar, Columbia) | Series video giảng CV nền tảng cực rõ. | [YouTube / fpcv.cs.columbia.edu](https://fpcv.cs.columbia.edu) |

---

## D. Kỹ năng công cụ (tooling) của researcher

- **PyTorch** ([pytorch.org](https://pytorch.org)) — framework mặc định cho NC hiện nay; + **PyTorch Lightning** để tổ chức code sạch.
- **Python khoa học:** NumPy, SciPy, scikit-learn, pandas, matplotlib.
- **Git & GitHub** thành thạo (branch, PR, issue) — nghiên cứu ngày càng open-source.
- **LaTeX / Overleaf** cho viết paper.
- **Experiment tracking:** Weights & Biases / MLflow; **Hydra** cho config.
- **Môi trường tái lập:** Docker / conda, `requirements.txt` hoặc `environment.yml`.

> Gợi ý thứ tự Phase 1: (1) lấp toán yếu bằng MML → (2) CS231n + assignments → (3) đọc 3–4 paper transformer/ViT/SAM để bắt kịp hiện đại.
