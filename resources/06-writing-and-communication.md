# 06 — Viết, Trình bày & Phản biện

*Cập nhật: 2026-07-29. Mọi link trong file này đã được kiểm tra còn sống.*

Nghiên cứu không tồn tại cho đến khi nó được **viết ra và thuyết phục được người khác**. Đây là kỹ năng bị engineer đánh giá thấp nhất và cũng là kỹ năng tạo khác biệt lớn nhất khi apply PhD.

> **Nguyên tắc gốc:** người đọc paper của bạn *không* muốn đọc nó. Họ đang tìm lý do để bỏ nó xuống. Việc viết không phải là "trình bày điều tôi đã làm" mà là **tạo giá trị cho một cộng đồng độc giả cụ thể**.

---

## A. Bốn bài giảng phải xem (theo thứ tự này)

| # | Bài giảng | Vì sao đây là bài hay nhất | Link |
|---|---|---|---|
| 1 | **Larry McEnerney — "The Craft of Writing Effectively"** (UChicago Leadership Lab, ~1h20) | ⭐⭐ **Bài giảng về viết học thuật hay nhất từng có** (8.8M views). Phá bỏ ảo tưởng "viết rõ ràng là đủ": bạn phải viết cho *cộng đồng độc giả*, tạo **giá trị** chứ không phải trình bày kiến thức. Thay đổi hoàn toàn cách bạn nghĩ về viết. | YouTube: tìm **"Leadership Lab: The Craft of Writing Effectively"** (kênh UChicago Social Sciences). Bản transcript có nhiều nơi đăng — tìm theo tên bài. |
| 2 | **Simon Peyton Jones — "How to Write a Great Research Paper"** | Bài giảng thực dụng nhất: viết paper *trước* khi làm research, cấu trúc intro 4 câu, cách kể "ý tưởng chính". Có slides + video. | [Microsoft Research](https://www.microsoft.com/en-us/research/academic-program/write-great-research-paper/) |
| 3 | **Jennifer Widom — "Tips for Writing Technical Papers"** (Stanford) | Checklist ngắn gọn, đi từng mục của paper. Đọc trước mỗi lần viết — như một quy trình QA. | [cs.stanford.edu](https://cs.stanford.edu/people/widom/paper-writing.html) |
| 4 | **Simon Peyton Jones — "How to Give a Great Research Talk"** | Nói chuyện nghiên cứu: mục tiêu của talk KHÔNG phải truyền tải hết nội dung, mà là làm người ta muốn đọc paper. | [Microsoft Research](https://www.microsoft.com/en-us/research/academic-program/give-great-research-talk/) |

> 🎯 **Việc làm ngay:** xem bài #1 tuần này. Ghi vào `logs/` ba điều bạn nhận ra mình đang viết sai.

---

## B. Loạt "Ten Simple Rules" (PLOS Computational Biology) — ngắn, miễn phí, chuẩn mực

Mỗi bài 2–4 trang. Đọc hết cả loạt mất một buổi tối và đáng giá hơn nhiều khóa học viết.

| Bài | Dùng khi nào | Link |
|---|---|---|
| **Ten Simple Rules for Structuring Papers** — Mensh & Kording | ⭐ Trước khi viết paper đầu tiên. Nguyên tắc "một bài — một thông điệp", logic C-C-C (Context–Content–Conclusion) áp dụng ở mọi cấp độ (paper / mục / đoạn). | [PLOS](https://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1005619) |
| **Ten Simple Rules for Better Figures** — Rougier et al. | ⭐ Trước khi vẽ hình. Hình là thứ reviewer xem đầu tiên. | [PLOS](https://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1003833) |
| **Ten Simple Rules for Writing a Literature Review** | Khi dựng literature map / mục Related Work. | [PLOS](https://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1003149) |
| **Ten Simple (Empirical) Rules for Writing Science** | Bằng chứng thực nghiệm về cái gì làm paper dễ đọc. | [PLOS](https://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1004205) |
| **Ten Simple Rules for Making Good Oral Presentations** | Trước talk đầu tiên. | [PLOS](https://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.0030077) |
| **Ten Simple Rules for a Good Poster Presentation** | Trước poster đầu tiên (workshop thường là poster). | [PLOS](https://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.0030102) |

---

## C. Sách về viết (đọc chậm, dùng lâu)

| Sách | Vì sao | Link |
|---|---|---|
| **The Sense of Style** — Steven Pinker | Văn phong "classic style": viết như đang chỉ cho người đọc thấy một thứ trong thế giới thực. Chữa bệnh viết vòng vo của dân kỹ thuật. | [stevenpinker.com](https://stevenpinker.com/publications/sense-style-thinking-persons-guide-writing-21st-century) |
| **Trees, Maps, and Theorems** — Jean-luc Doumont | ⭐ Giao tiếp khoa học có hệ thống: paper, slide, poster, email. Rất "kỹ sư": có nguyên lý, có cấu trúc. | [principiae.be](https://www.principiae.be) |
| **The Craft of Research** — Booth, Colomb, Williams et al. | Xây lập luận: claim → reason → evidence → acknowledgment. Bản **5th ed. (2024)** có thêm chương về trình bày và hướng dẫn dùng AI sinh tạo. | [UChicago Press](https://press.uchicago.edu/ucp/books/book/chicago/C/bo215874008.html) |
| **Fundamentals of Data Visualization** — Claus Wilke | Miễn phí online. Chuẩn mực để hình trong paper không bị reviewer chê. | [clauswilke.com/dataviz](https://clauswilke.com/dataviz/) |

---

## D. Công thức từng mục của paper (dùng như checklist)

### Abstract — 5 câu
1. **Bối cảnh:** lĩnh vực này quan tâm điều gì.
2. **Khoảng trống:** nhưng X vẫn chưa được giải quyết.
3. **Việc chúng tôi làm:** chúng tôi đề xuất/nghiên cứu Y.
4. **Bằng chứng:** trên dataset Z, Y đạt … (con số cụ thể, so với baseline nào).
5. **Ý nghĩa:** điều này cho thấy / mở ra …

### Introduction — 5 đoạn (Simon Peyton Jones)
1. **Vấn đề là gì** (1 đoạn, có ví dụ cụ thể — đừng bắt đầu bằng "Deep learning has revolutionized…").
2. **Tại sao nó khó / tại sao cách hiện tại không đủ.**
3. **Ý tưởng của chúng tôi** — nêu thẳng, đừng giữ bí mật tới mục 3.
4. **Đóng góp** — gạch đầu dòng, mỗi dòng *có thể kiểm chứng được* và **trỏ tới mục/hình cụ thể**.
5. **Kết quả chính** — một con số/hình then chốt.

> **Luật sắt:** mỗi đóng góp trong danh sách phải map 1-1 với một thí nghiệm chứng minh nó. Nếu không có thí nghiệm → không phải đóng góp, chỉ là mô tả.

### Related Work — định vị, không liệt kê
- Sai: "A làm X. B làm Y. C làm Z."
- Đúng: nhóm theo **cách tiếp cận**, rồi nói rõ **cái gì họ chưa làm mà bạn làm**. Mỗi nhóm kết bằng một câu "khác biệt của chúng tôi là…".
- Không hạ thấp người khác. Reviewer của bạn *chính là* các tác giả đó.

### Method — kỷ luật ký hiệu
- Bảng notation nếu > 8 ký hiệu. Một ký hiệu = một ý nghĩa, xuyên suốt.
- Một hình kiến trúc tổng quan mà người ta hiểu được **không cần đọc chữ**.
- Viết sao cho người khác **cài lại được** — đó là thước đo.

### Experiments — mapping claim → bằng chứng
Trước khi viết, làm bảng 2 cột: | Tôi khẳng định điều gì | Thí nghiệm/bảng/hình nào chứng minh |. Chỗ nào trống → hoặc bỏ claim, hoặc làm thêm thí nghiệm. Xem chi tiết `07-experiments-and-rigor.md`.

### Discussion / Limitations
- **Tự nêu hạn chế trước khi reviewer nêu.** Điều này *tăng* độ tin cậy, không giảm.
- Phân biệt rõ "kết quả của chúng tôi cho thấy" (đã đo) vs "chúng tôi cho rằng" (suy đoán).

---

## E. Hình & bảng — nơi reviewer quyết định trong 60 giây

- **Teaser figure (Hình 1):** phải kể được toàn bộ ý tưởng. Dành cho nó 10% thời gian viết paper.
- Chữ trong hình **không nhỏ hơn** chữ trong caption. Test: in ra giấy A4 đen trắng, đọc được không?
- **Caption tự đứng được:** đọc caption mà không đọc bài vẫn hiểu hình nói gì.
- **Palette an toàn cho người mù màu:** `viridis`, `cividis`, ColorBrewer. Không dùng jet/rainbow cho dữ liệu liên tục.
- Với ảnh y tế: luôn có **thanh tỉ lệ (scale bar)**, chỉ rõ modality/mặt cắt, và hiển thị **ca thất bại** — không chỉ ca đẹp nhất.
- Công cụ: matplotlib + [Inkscape](https://inkscape.org) (vector), [Excalidraw](https://excalidraw.com) (sơ đồ nhanh), TikZ (hình trong LaTeX, tái lập được), [3D Slicer](https://www.slicer.org) (render ảnh y tế).

---

## F. Peer review — kỹ năng "tăng tốc" bị bỏ qua nhiều nhất

Học review giúp bạn viết tốt hơn nhanh hơn bất kỳ cách nào khác, vì bạn thấy paper **từ phía người quyết định**.

**Cách bắt đầu khi chưa có PhD:**
1. Nhiều **workshop** tại MICCAI/CVPR/ICRA mở đăng ký reviewer cho ai có ≥1 preprint liên quan — email organizer, nói rõ background. Đây là lối vào thực tế nhất.
2. Đăng ký reviewer cho [ML Reproducibility Challenge](https://reproml.org).
3. Tự review: mỗi paper pass-3 trong `templates/paper-reading-note.md`, viết thêm một **review giả lập** (điểm số + 3 điểm mạnh + 3 điểm yếu + câu hỏi cho tác giả).

**Cấu trúc một review tốt:**
- **Tóm tắt** paper bằng lời của bạn (chứng minh bạn đã đọc).
- **Điểm mạnh** — cụ thể, không sáo.
- **Điểm yếu** — phân biệt rõ **lỗi nghiêm trọng** (kết luận không được bằng chứng ủng hộ, so sánh không công bằng, thiếu baseline) với **góp ý làm đẹp** (chính tả, trình bày).
- **Câu hỏi có thể trả lời được** trong thời gian rebuttal.
- **Điểm số + lý do**, nêu rõ điều gì sẽ làm bạn tăng điểm.

> ⚠️ Cạm bẫy của engineer khi review: đòi thí nghiệm tốn kém vô hạn ("hãy thử trên 10 dataset nữa"). Hãy hỏi: *thí nghiệm nào là cần thiết để kết luận của họ đứng được?* Chỉ đòi cái đó.

Đọc hướng dẫn reviewer chính thức của các venue lớn (miễn phí, rất chuẩn mực) trên website của **NeurIPS / CVPR / MICCAI** — tìm "reviewer guidelines" cho năm hiện tại. Chúng cũng là **checklist ngược** để tự kiểm paper của mình trước khi nộp.

---

## G. Rebuttal & phản hồi reviewer

**Rebuttal cho hội nghị (giới hạn 1 trang, có deadline gấp):**
1. Mở đầu 2 câu: cảm ơn + nêu **hai chỉnh sửa lớn nhất** bạn sẽ làm.
2. Gộp các phản biện trùng nhau thành mục chung (**R1&R3-Q1**), tránh lặp.
3. Ưu tiên: điều nào có thể **giải quyết bằng dữ liệu/số liệu mới** → làm trước, đưa bảng vào rebuttal.
4. Với phản biện bạn không đồng ý: nêu bằng chứng, không nêu cảm xúc. Nếu reviewer *hiểu sai*, hãy coi đó là **lỗi trình bày của bạn** và nói bạn sẽ viết lại chỗ nào.
5. Cam kết cụ thể ("chúng tôi sẽ bổ sung ablation ở Bảng 3 của bản final"), không cam kết chung chung.

**Response letter cho tạp chí (TMI/MedIA — dài, point-by-point):**
- Bảng 3 cột: **Nhận xét của reviewer (nguyên văn)** | **Phản hồi** | **Đã sửa ở đâu (số trang/dòng)**.
- Trích dẫn lại nguyên văn nhận xét — đừng diễn giải lại.
- Đánh dấu thay đổi trong bản thảo (dùng `latexdiff`).

**Luật sắt:** trả lời **mọi** nhận xét, kể cả khi chỉ để nói "chúng tôi đã sửa". Bỏ sót một nhận xét là lý do phổ biến để bị reject lần hai.

---

## H. Trung thực học thuật & những cạm bẫy của ngành

Đây là phần phân biệt researcher với người "làm ra số đẹp". Đọc để **không** vô tình rơi vào.

| Tài liệu | Nội dung | Link |
|---|---|---|
| **Troubling Trends in Machine Learning Scholarship** — Lipton & Steinhardt | ⭐ Bốn bệnh của paper ML: giải thích suy đoán trộn với sự thật, không tách nguồn gốc cải thiện, lạm dụng toán để gây oai, dùng từ ngữ mơ hồ. **Bắt buộc đọc trước khi viết paper đầu tiên.** | [arXiv:1807.03341](https://arxiv.org/abs/1807.03341) |
| **A Metric Learning Reality Check** — Musgrave et al. | Ví dụ điển hình: khi so sánh công bằng, phần lớn "tiến bộ" 10 năm biến mất. Học cách không tự lừa mình. | [arXiv:2003.08505](https://arxiv.org/abs/2003.08505) |
| **Leakage and the Reproducibility Crisis in ML-based Science** — Kapoor & Narayanan | ⭐ 8 loại rò rỉ dữ liệu làm hàng trăm paper (nhiều trong y tế) sai kết luận. Cực kỳ liên quan tới medical imaging. | [arXiv:2207.07048](https://arxiv.org/abs/2207.07048) |
| **Show Your Work: Improved Reporting of Experimental Results** — Dodge et al. | Tại sao báo cáo "con số tốt nhất" là gian lận vô ý, và báo cáo thế nào mới đúng. | [arXiv:1909.03004](https://arxiv.org/abs/1909.03004) |
| **Retraction Watch** | Theo dõi các ca gian lận/rút bài — để biết ranh giới ở đâu. | [retractionwatch.com](https://retractionwatch.com) |

**Danh sách tự kiểm trước khi nộp:**
- [ ] Mọi con số trong abstract đều tìm được trong bảng của bài.
- [ ] Không có claim nào không có thí nghiệm tương ứng.
- [ ] Baseline được tune với **cùng ngân sách** như method của mình.
- [ ] Test set chỉ dùng **một lần cuối** — không tune trên nó.
- [ ] Chia dữ liệu theo **bệnh nhân**, không theo slice/frame (rò rỉ kinh điển trong ảnh y tế).
- [ ] Đã nêu hạn chế thật, không phải hạn chế giả ("cần thêm dữ liệu" là hạn chế giả).

---

## I. Dùng AI trong viết paper — làm đúng cách

Bạn *sẽ* dùng LLM. Vấn đề là dùng ở đâu thì hợp lệ.

**Được và nên:** sửa ngữ pháp/văn phong, tóm tắt paper để sàng lọc, brainstorm phản biện với chính mình ("hãy đóng vai reviewer khó tính"), sinh code cho hình vẽ, dịch.

**Không được:**
- Để LLM sinh **nội dung khoa học** (kết quả, phân tích, related work) rồi coi là của mình.
- Để LLM sinh **trích dẫn** — nó bịa ra reference trông rất thật. **Luôn tự mở paper gốc ra kiểm tra.**
- Đưa **bản thảo của người khác đang review** vào công cụ AI — vi phạm bảo mật peer review (NeurIPS/CVPR/ACL đều cấm rõ).

**Khai báo:** hầu hết venue lớn (CVPR, NeurIPS, ICML, MICCAI, Elsevier, Springer, IEEE) hiện yêu cầu **khai báo mức độ dùng AI** và khẳng định tác giả chịu trách nhiệm hoàn toàn về nội dung. LLM **không được** là tác giả. Trước mỗi lần nộp: mở trang "Author Guidelines / Ethics Policy" của venue năm đó và làm theo — chính sách đang thay đổi hàng năm.

---

## J. Công cụ

| Công cụ | Vai trò |
|---|---|
| [Overleaf](https://www.overleaf.com) | LaTeX trên web, cộng tác. Dùng template chính thức của venue (IEEE, CVPR, Springer LNCS cho MICCAI). |
| [Zotero](https://www.zotero.org) + **Better BibTeX** | Quản lý reference, sinh citation key ổn định, đồng bộ `.bib` vào repo. |
| `latexdiff` | Sinh bản thảo có đánh dấu thay đổi cho revision — bắt buộc với tạp chí. |
| [arxiv-latex-cleaner](https://github.com/google-research/arxiv-latex-cleaner) | Làm sạch source LaTeX trước khi đăng arXiv (xoá comment, file rác). |
| [Vale](https://vale.sh) | Linter văn phong, chạy được trong CI — bắt câu bị động, từ mơ hồ. |
| [LanguageTool](https://languagetool.org) | Kiểm tra ngữ pháp, có bản offline. |
| [Connected Papers](https://www.connectedpapers.com) · [Semantic Scholar](https://www.semanticscholar.org) | Dựng bản đồ related work. |

---

## K. Nhịp luyện viết (bắt đầu ngay từ Phase 1, đừng đợi Phase 3)

| Tần suất | Việc |
|---|---|
| **Hàng tuần** | Cập nhật `logs/` — 3 điều học được, 1 điều mắc. Viết **thành câu hoàn chỉnh**, không gạch đầu dòng cụt. |
| **Mỗi paper pass-3** | Note đầy đủ + một review giả lập. |
| **Mỗi tháng** | Một bài viết ngắn 500–800 từ (blog/markdown) giải thích một khái niệm bạn vừa học cho người ngoài ngành. |
| **Mỗi project** | Bài "lessons learned" — vừa luyện viết, vừa là bằng chứng công khai cho hồ sơ PhD. |

> **Cách kiểm tra bạn đã hiểu:** giải thích được đóng góp của mình trong **một câu**, cho người không cùng chuyên ngành. Nếu chưa được — bạn chưa hiểu, chưa phải chưa viết được.

---

*Liên quan: `00-research-skills.md` (đọc paper, mindset) · `07-experiments-and-rigor.md` (bằng chứng cho điều bạn viết) · `05-phd-preparation.md` (SOP là một dạng viết thuyết phục).*
