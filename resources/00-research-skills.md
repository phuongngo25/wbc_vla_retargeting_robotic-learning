# 00 — Research Skills & Mindset (nền chạy suốt lộ trình)

*Cập nhật: 2026-07-29 — đã kiểm tra lại toàn bộ link.*

Đây là phần cốt lõi cho việc **đổi tư duy** từ engineer sang researcher. Học song song với chuyên môn, không phải học xong mới làm.

> 📚 File này là **tổng quan**. Ba phần đã được tách ra thành tài liệu chuyên sâu riêng:
>
> - **Viết, trình bày, peer review, rebuttal** → `06-writing-and-communication.md`
> - **Thiết kế thí nghiệm, thống kê, metric, reproducibility** → `07-experiments-and-rigor.md`
> - **Paper cụ thể để đọc, theo thứ tự** → `08-core-reading-list.md`

---

## A. Sách nền tảng về tư duy & phương pháp nghiên cứu

| Tài liệu | Vì sao nên đọc | Link |
|---|---|---|
| **The Craft of Research** — Booth, Colomb, Williams | Kinh điển: cách đặt câu hỏi nghiên cứu, xây lập luận, viết. Đọc Phần I–II trước. | [UChicago Press (5th ed., 2024)](https://press.uchicago.edu/ucp/books/book/chicago/C/bo215874008.html) |
| **The Unwritten Rules of PhD Research** — Rugg & Petre | "Luật ngầm" của đời nghiên cứu — thứ không ai dạy chính thức. Bản 3rd ed. (Open University Press) có thêm phần critical thinking & con đường tới độc lập nghiên cứu. | Tìm ISBN **9780335262120** (3rd ed.) trên thư viện/nhà sách |
| **How to Make Your PhD Work** — có case studies, worksheets | Bản đồ nghề nghiệp cho PhD/early-career researcher. | [Wiley](https://www.wiley.com/en-us/How+to+Make+Your+PhD+Work-p-9781394193165) |
| **A PhD Is Not Enough!** — Peter Feibelman | Ngắn, thẳng thắn, về chiến lược sự nghiệp khoa học. | [Amazon/Basic Books](https://www.basicbooks.com/titles/peter-j-feibelman/a-phd-is-not-enough/9780465022229/) |
| **The Sense of Style** — Steven Pinker | Viết học thuật rõ ràng, không sáo rỗng. | [stevenpinker.com](https://stevenpinker.com/publications/sense-style-thinking-persons-guide-writing-21st-century) |

---

## B. Bài viết / bài giảng kinh điển (miễn phí, đọc ngay)

| Tài liệu | Nội dung | Link |
|---|---|---|
| **Richard Hamming — "You and Your Research"** | Bài giảng huyền thoại: cách chọn *bài toán quan trọng*, làm việc để tạo tác động lớn. **Đọc đầu tiên.** Đọc lại mỗi 6 tháng. | [UVA mirror](https://www.cs.virginia.edu/~robins/YouAndYourResearch.html) |
| **S. Keshav — "How to Read a Paper"** | Phương pháp **three-pass** để đọc paper hiệu quả. | [PDF (SIGCOMM CCR)](http://ccr.sigcomm.org/online/files/p83-keshavA.pdf) |
| **Larry McEnerney — "The Craft of Writing Effectively"** | ⭐⭐ Bài giảng về viết học thuật hay nhất từng có (UChicago Leadership Lab, ~1h20). Xem chi tiết ở `06-writing-and-communication.md`. | YouTube: tìm **"Leadership Lab: The Craft of Writing Effectively"** (kênh UChicago Social Sciences) |
| **Simon Peyton Jones — "How to Write a Great Research Paper"** | Bài giảng vàng về viết paper (có video + slides). | [Microsoft Research](https://www.microsoft.com/en-us/research/academic-program/write-great-research-paper/) |
| **Simon Peyton Jones — "How to Give a Great Research Talk"** | Trình bày nghiên cứu thuyết phục. | [Microsoft Research](https://www.microsoft.com/en-us/research/academic-program/give-great-research-talk/) |
| **Michael Nielsen — "Principles of Effective Research"** | Xây thói quen & chiến lược nghiên cứu. | [michaelnielsen.org](https://michaelnielsen.org/blog/principles-of-effective-research/) |
| **John Schulman — "An Opinionated Guide to ML Research"** | Rất sát ngành AI: chọn bài toán, quản lý thời gian, tune. | [joschu.net](http://joschu.net/blog/opinionated-guide-ml-research.html) |
| **Andrej Karpathy — "A Survival Guide to a PhD"** | Kinh nghiệm thực tế cho PhD trong AI. | [karpathy.github.io](https://karpathy.github.io/2016/09/07/phd/) |
| **Jason Eisner — "How to Read a CS Research Paper"** | Bổ sung góc CS cho three-pass. | [JHU](https://www.cs.jhu.edu/~jason/advice/how-to-read-a-paper.html) |
| **Lipton & Steinhardt — "Troubling Trends in ML Scholarship"** | ⭐ Bốn bệnh của paper ML. Đọc sớm để không mắc phải. | [arXiv:1807.03341](https://arxiv.org/abs/1807.03341) |
| **Philip Guo — "The PhD Grind"** | Hồi ký PhD (Stanford CS) — đời sống nghiên cứu thật sự ra sao. ⚠️ Domain `pgbovine.net` **không còn phân giải DNS** (kiểm tra 2026-07-29). Tìm bản lưu: Wayback Machine cho `pgbovine.net/PhD-memoir.htm`, hoặc tìm tên file `pguo-PhD-grind.pdf`. | [Wayback](https://web.archive.org/web/2024/https://pgbovine.net/PhD-memoir.htm) |

---

## C. Đọc paper (áp dụng ngay từ Phase 1)

**Three-pass method (Keshav):**

1. **Pass 1 (5–10 phút):** đọc title, abstract, intro, heading các mục, kết luận, lướt qua toán. Trả lời: paper này về cái gì? có liên quan mình không?
2. **Pass 2 (~1 giờ):** đọc kỹ hình/bảng, nắm ý chính, bỏ qua chứng minh chi tiết. Sau pass 2 phải tóm tắt được bằng lời của mình.
3. **Pass 3 (vài giờ):** đọc để *tái tạo lại* — như thể mình là tác giả. Đặt câu hỏi phản biện ở từng bước.

**Đọc phản biện — luôn tự hỏi:**
- Câu hỏi/giả thuyết của paper là gì? Đóng góp *thực sự* mới là gì?
- Baseline & so sánh có công bằng không? Thiếu ablation nào?
- Kết quả có được lặp lại không? Có che giấu điều kiện thất bại không?
- Nếu là mình, mình sẽ làm khác chỗ nào? Câu hỏi mở nào còn lại?

> Dùng `templates/paper-reading-note.md` cho mỗi paper pass-3.

**Công cụ tìm & quản lý paper** (đã kiểm tra 2026-07-29):

- **Tìm & bản đồ:** [Google Scholar](https://scholar.google.com) · [Semantic Scholar](https://www.semanticscholar.org) (citation graph, API mở) · [Connected Papers](https://www.connectedpapers.com) (bản đồ paper liên quan — dùng khi bắt đầu một chủ đề mới).
- **Paper + code:** [Hugging Face Papers](https://huggingface.co/papers) — ⚠️ **Papers with Code đã đóng cửa 24/07/2025**; `paperswithcode.com` hiện redirect sang đây. Lưu ý: bản thay thế **không có leaderboard SOTA** như PwC cũ.
- **Thảo luận trên paper:** [alphaXiv](https://www.alphaxiv.org) — bình luận công khai trên từng bài arXiv.
- **Quản lý reference:** [Zotero](https://www.zotero.org) + plugin **Better BibTeX** (citation key ổn định, export `.bib` vào repo).
- **Cảnh báo tự động:** Google Scholar Alerts theo **tác giả** (hiệu quả hơn theo từ khoá) + arXiv theo category.
- ⚠️ `arxiv-sanity-lite.com` hiện **không phản hồi** (kiểm tra 2026-07-29) — dùng Hugging Face Papers hoặc alphaXiv thay thế.

---

## D. Viết & trình bày

> 📖 **Chi tiết đầy đủ: `06-writing-and-communication.md`** — bài giảng phải xem, công thức từng mục của paper, hình/bảng, peer review, rebuttal, đạo đức dùng AI.

Ba nguyên tắc cốt lõi:

- **Viết sớm:** bắt đầu viết Related Work + Method ngay khi làm thí nghiệm, không đợi có kết quả. Viết là cách tư duy, không phải bước báo cáo cuối.
- **Viết cho cộng đồng độc giả, không cho chính mình.** Người đọc đang tìm lý do để bỏ bài của bạn xuống. (Larry McEnerney — mục B.)
- **Mỗi đóng góp phải map 1-1 với một thí nghiệm chứng minh nó.** Không có thí nghiệm → không phải đóng góp.

Công cụ: **LaTeX** ([Overleaf](https://www.overleaf.com)), template hội nghị (IEEE, CVPR, MICCAI/Springer LNCS), `latexdiff` cho revision.

---

## E. Thiết kế thí nghiệm & tính tái lập (reproducibility)

> 📖 **Chi tiết đầy đủ: `07-experiments-and-rigor.md`** — rò rỉ dữ liệu, chọn metric theo Metrics Reloaded, thống kê, reporting guidelines cho AI y tế (FUTURE-AI, CLAIM, TRIPOD+AI), đạo đức dữ liệu bệnh nhân.

Bốn nguyên tắc cốt lõi:

- **Baseline trước, method sau** — và baseline phải được tune với **cùng ngân sách** như method của bạn. Trong ảnh y tế, **nnU-Net là baseline bắt buộc**.
- **Ablation study:** tắt/bật từng thành phần để biết cái gì *thực sự* tạo ra hiệu quả.
- **Nhiều seed (≥3) + báo cáo mean ± std.** Nếu khoảng dao động chồng lên baseline → bạn chưa chứng minh được gì.
- **Chia dữ liệu theo bệnh nhân, không theo slice/frame** — đây là kiểu rò rỉ dữ liệu phổ biến nhất và nguy hiểm nhất trong ảnh y tế.

Tham khảo nhanh: [ML Reproducibility Checklist (Pineau)](https://www.cs.mcgill.ca/~jpineau/ReproducibilityChecklist.pdf) · [ML Reproducibility Challenge](https://reproml.org) · [Weights & Biases](https://wandb.ai) / [MLflow](https://mlflow.org) · [Hydra](https://hydra.cc).

---

## F. Cộng đồng & giữ nhịp cập nhật

> 📖 **Chi tiết đầy đủ: `09-community-labs-funding.md`** — lab mục tiêu, seminar, summer school, mentorship, funding theo khu vực.

- **Feed:** [Hugging Face Papers](https://huggingface.co/papers) · [alphaXiv](https://www.alphaxiv.org) · [The Batch](https://www.deeplearning.ai/the-batch/) · [Import AI](https://importai.net).
- **arXiv theo category:** `cs.CV`, `cs.RO`, `eess.IV`.
- **Google Scholar Alerts theo tác giả** — tín hiệu chất lượng cao hơn nhiều so với quét từ khoá.
- Talk & tutorial hội nghị trên YouTube: MICCAI, CVPR, ICRA, RSS, CoRL. **Tutorial là tài liệu học miễn phí tốt nhất.**
- ⭐ Đóng góp mã nguồn mở cho [MONAI](https://github.com/Project-MONAI/MONAI) / [nnU-Net](https://github.com/MIC-DKFZ/nnUNet) — cách hiệu quả nhất để được cộng đồng biết đến khi bạn chưa ở trong academia.
