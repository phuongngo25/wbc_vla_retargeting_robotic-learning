# ROADMAP — Từ AI Engineer đến Researcher (đích: PhD)

*Cập nhật lần đầu: 2026-07-28 · Cập nhật gần nhất: 2026-07-29 (bổ sung `resources/06`–`09` và 3 template mới)*

Tài liệu này là lộ trình ~15–18 tháng, thiết kế cho người **đã là AI Engineer** (robotics, thiết bị y tế, xử lý ảnh) muốn chuyển sang **tư duy nghiên cứu** và **apply PhD**. Nó không phải khóa học tuyến tính — mà là khung để bạn tự điều chỉnh theo tốc độ của mình.

---

## Phần 0 — Sự khác nhau cốt lõi: Engineer vs Researcher

Đây là phần quan trọng nhất. Nếu chỉ đọc được một phần, hãy đọc phần này.

| | **Tư duy Engineer** | **Tư duy Researcher** |
|---|---|---|
| Câu hỏi trung tâm | "Làm thế nào để build cái này?" | "Câu hỏi nào đáng trả lời, và tại sao?" |
| Định nghĩa thành công | Hệ thống chạy, đạt spec, ship đúng hạn | Tạo ra **tri thức mới, tổng quát hoá được**, thuyết phục được cộng đồng |
| Thái độ với vấn đề đã có lời giải | Dùng lại giải pháp tốt nhất | Hỏi "tại sao nó hoạt động? khi nào nó hỏng?" |
| Metric | Có sẵn, cố định (accuracy, latency, cost) | Phải **tự thiết kế** và bảo vệ được metric/thí nghiệm |
| Thất bại | Bug cần sửa | **Dữ liệu** — thường là phần thú vị nhất |
| Phạm vi | Một sản phẩm/khách hàng cụ thể | Một lớp bài toán, khái quát hoá cho nhiều trường hợp |
| Đầu ra | Code, sản phẩm, tính năng | Paper, kiến thức, phương pháp tái lập được |
| Quan hệ với "state of the art" | Tiêu dùng nó | **Đẩy nó tiến lên** một chút, và chứng minh được là mình đã đẩy |

**Bốn dịch chuyển tư duy bạn cần luyện:**

1. **Từ giải pháp sang câu hỏi.** Kỹ sư giỏi thấy vấn đề là nghĩ ngay giải pháp. Nhà nghiên cứu dừng lại: bài toán này *thực sự* là gì? Đã ai giải chưa? Nếu giải rồi thì thiếu gì? Câu hỏi con nào chưa ai chạm tới?
2. **Từ "chạy được" sang "hiểu tại sao".** Model đạt 92% accuracy chưa đủ. Tại sao 92%? 8% còn lại sai kiểu gì? Nếu đổi dữ liệu/điều kiện thì sao? Đâu là giả thuyết bạn đang kiểm chứng?
3. **Từ tiêu thụ tri thức sang tạo & bảo vệ tri thức.** Bạn phải đọc paper *một cách phản biện* (nó sai chỗ nào?), rồi tự tạo đóng góp và bảo vệ nó trước phản biện (peer review).
4. **Từ tối ưu ngắn hạn sang đầu tư dài hạn (Hamming).** Richard Hamming: "Are you working on important problems in your field?" Chọn bài toán quan trọng, không chỉ bài toán dễ.

> 📌 Đọc bắt buộc cho phần này: **Richard Hamming — "You and Your Research"** và **"The Craft of Research"**. Xem `resources/00-research-skills.md`.

---

## Phần 1 — Chọn hướng (làm ngay tuần đầu)

Bạn đã chọn cả 4 mảng: Medical Imaging, Robotics, Surgical/Medical Robotics, Research skills. Với **đích PhD**, đây là lời khuyên chiến lược:

- **Nền tảng chung (research skills)** → không phải "hướng", mà là **nền chạy song song** suốt cả lộ trình.
- Trong 3 mảng chuyên môn, **PhD đòi hỏi bạn hẹp lại thành một trục**. Không hẹp = hồ sơ loãng, khó có đóng góp sâu.

**Gợi ý trục chính (theo thế mạnh hiện tại của bạn):**

> **Medical Imaging làm trục chính** (gần nhất với kinh nghiệm xử lý ảnh + thiết bị y tế của bạn), và **Surgical/Medical Robotics làm điểm giao mở rộng** (image-guided intervention, vision cho robot phẫu thuật). Robotics thuần (control/planning) học ở mức nền tảng để hiểu, không cần là trục.

Điểm giao này rất "PhD-friendly": nó nằm đúng chỗ MICCAI ∩ ICRA — image-guided surgery, surgical vision, autonomous surgical subtasks — một hướng đang nóng và thiếu người có cả nền vision lẫn robotics như bạn.

**Việc cần làm tuần này:** viết 3–5 câu vào `logs/` trả lời: (1) Hướng trục của mình là gì? (2) Vì sao mình quan tâm? (3) Mình muốn 2 năm nữa mình *giỏi cái gì mà bây giờ chưa*?

> ⚠️ Chọn trục *không* có nghĩa là khóa cứng. Bạn được phép điều chỉnh sau Phase 2 khi đã đọc đủ paper để biết mình thực sự thích gì.

---

## Phần 2 — Lộ trình theo giai đoạn

Mỗi phase có: **mục tiêu**, **việc làm**, và **tiêu chí hoàn thành (definition of done)**. Thời lượng là gợi ý — điều chỉnh theo quỹ thời gian thực tế (giả định ~10–12 giờ/tuần ngoài giờ làm).

### Phase 0 — Setup & orientation (Tuần 1–2)

**Mục tiêu:** dựng hạ tầng và thói quen trước khi học.

- Tạo repo này trên GitHub, commit lần đầu.
- Đọc hết `ROADMAP.md` và `resources/00-research-skills.md`.
- Lập tài khoản: [Google Scholar](https://scholar.google.com), [Semantic Scholar](https://www.semanticscholar.org), [arXiv](https://arxiv.org), [Papers with Code](https://paperswithcode.com), [Zotero](https://www.zotero.org) (quản lý tài liệu tham khảo).
- Chọn công cụ ghi chú (Obsidian/Notion/plain markdown trong repo này).
- Viết "research direction statement" 3–5 câu (Phần 1).

**Done khi:** repo đã push, có statement định hướng, đã cài Zotero và biết dùng three-pass method.

---

### Phase 1 — Research literacy + refresh nền tảng (Tháng 1–3)

**Mục tiêu:** biết đọc paper như nhà nghiên cứu, và lấp lỗ hổng nền tảng.

- **Kỹ năng đọc:** áp dụng **three-pass method (Keshav)** cho mọi paper. Mỗi tuần đọc kỹ (pass 3) **1 paper**, đọc lướt (pass 1–2) **3–5 paper**. Dùng `templates/paper-reading-note.md`.
- **⭐ Đọc gì, theo thứ tự nào:** làm theo **kế hoạch 12 tuần** trong `resources/08-core-reading-list.md` — 67 paper nền tảng đã được kiểm chứng, chia theo tầng ưu tiên (T1/T2/T3) và theo track. Đây là phần cụ thể hoá toàn bộ Phase 1.
- **Nền tảng (chọn theo lỗ hổng của bạn, không cần học lại từ đầu):**
  - Toán ML: *Mathematics for Machine Learning* (chỉ đọc chương còn yếu).
  - Computer Vision + Deep Learning: **CS231n** (Stanford) hoặc **EECS 498 của Justin Johnson** (bản hiện đại hơn, video đầy đủ) — làm assignments là bắt buộc nếu muốn hiểu sâu.
  - Hiểu sâu từ gốc: **Karpathy — Neural Networks: Zero to Hero** (xây backprop/transformer từ số 0).
  - Nếu yếu robotics: **Modern Robotics** (Lynch & Park) chương 1–6; SLAM/perception qua **bài giảng của Cyrill Stachniss**.
- **Đọc mindset:** *The Craft of Research* (Phần I–II), Hamming, và **Larry McEnerney — "The Craft of Writing Effectively"** (xem `resources/06`).
- **⭐ Đọc sớm về tính nghiêm ngặt:** *Troubling Trends in ML Scholarship* + *Leakage and the Reproducibility Crisis* + *Metrics Reloaded* (`resources/07`). Đọc **trước** khi chạy thí nghiệm đầu tiên — chúng thay đổi cách bạn đánh giá mọi paper còn lại.
- Bắt đầu một **"literature map"**: dùng `templates/literature-map.md`, gom 15–25 paper nền của trục bạn chọn, nhóm **theo cách tiếp cận** (không theo thời gian) và ghi lại khoảng trống.

**Done khi:** đọc kỹ ≥ 10 paper (có note), dựng được literature map sơ bộ **có ≥3 khoảng trống ghi ra được**, làm xong assignment CS231n hoặc tương đương.

---

### Phase 2 — Đào sâu domain + reproduce một paper (Tháng 3–6)

**Mục tiêu:** từ "đọc hiểu" sang "làm lại được". Đây là bước quan trọng nhất để có tư duy nghiên cứu thực chiến.

- **⭐ Dùng `templates/reproduce-checklist.md` ngay từ đầu** — nó có sẵn danh sách "những chỗ paper thường không nói rõ" và bảng ghi lại các lựa chọn bạn buộc phải đoán. Phần đó là giá trị thật của Phase này.
- Chọn **1 paper** kinh điển/gần đây trong trục của bạn và **tái tạo (reproduce)** kết quả của nó từ đầu — dùng dữ liệu công khai. **Chọn paper có code công khai cho lần đầu tiên.**
  - Medical imaging: bắt đầu với **MONAI** + **nnU-Net** trên một dataset của [Medical Segmentation Decathlon](http://medicaldecathlon.com) hoặc [Grand Challenge](https://grand-challenge.org). Chạy thử pipeline trên [MedMNIST](https://medmnist.com) trước cho nhanh.
  - Surgical: **Cholec80/CholecT50** (CAMMA) là lối vào dễ nhất — TeCNO là ứng viên reproduce tốt. Hoặc **SurRoL/ORBIT-Surgical** nếu đi hướng mô phỏng.
- Ghi lại: chỗ nào paper không nói rõ? bạn phải đoán gì? kết quả có khớp không? vì sao lệch?
- Dùng **experiment tracking** (Weights & Biases hoặc MLflow) — log **cả run thất bại**.
- Học **reproducibility**: seed, config, môi trường, versioning dữ liệu → `resources/07-experiments-and-rigor.md` mục H.
- **⭐ Trước khi tin bất kỳ con số nào:** đọc `resources/07` mục D (rò rỉ dữ liệu) và mục E (chọn metric). Riêng một lỗi — chia dữ liệu theo slice thay vì theo bệnh nhân — đủ làm vô nghĩa cả project.
- **Xem bằng mắt, không chỉ đọc số:** xem ≥10 ca tốt nhất và ≥10 ca tệ nhất. Kiểu lỗi bạn thấy mà paper không nêu **chính là khoảng trống**.

**Done khi:** có 1 repo tái tạo được (dù chỉ gần đúng) kết quả một paper, kèm README ghi rõ khác biệt so với bản gốc, một bài viết ngắn "những gì tôi học được khi reproduce X", **và ≥2 câu hỏi mở ghi ra được**.

> 💡 **Lệch số không phải thất bại.** Báo cáo trung thực "tôi đạt 0.87 so với 0.91, đây là 3 lý do khả năng nhất" có giá trị nghiên cứu **hơn** một con số khớp mà bạn không hiểu vì sao khớp.
>
> 💡 Đây thường là thời điểm bạn nhận ra một câu hỏi mở — hạt giống cho đóng góp riêng ở Phase 3.

---

### Phase 3 — Đóng góp nghiên cứu mini + tập viết (Tháng 6–10)

**Mục tiêu:** tạo một đóng góp nhỏ nhưng *của bạn*, và viết nó ra.

- Từ câu hỏi mở tìm được ở Phase 2, đề xuất một **cải tiến/thí nghiệm nhỏ** (dùng `templates/project-proposal.md`). Ví dụ: đổi loss, thêm một module, test tính bền vững (robustness) trên phân phối dữ liệu khác, hoặc **so sánh có kiểm soát mà chưa ai làm** (loại này bị đánh giá thấp nhưng rất được trọng dụng — và khớp hoàn hảo với tư duy kỹ sư).
- Nguyên tắc scope: **một câu hỏi, một giả thuyết, một thí nghiệm sạch**. Đừng làm quá to.
- **⭐ Thiết kế thí nghiệm nghiêm túc** → `resources/07-experiments-and-rigor.md`: baseline tune cùng ngân sách, ablation cô lập từng thành phần, ≥3 seed với mean ± std, metric chọn theo Metrics Reloaded, bootstrap CI cho khác biệt chính. Dùng checklist ở mục J trước khi tin kết quả của chính mình.
- **⭐ Viết song song** → `resources/06-writing-and-communication.md`: bắt đầu viết Related Work và Method ngay khi làm, không đợi có kết quả. Mỗi đóng góp phải map 1-1 với một thí nghiệm.
- Tìm một **mentor/cộng tác viên** để review. Ba đường thực tế nhất: (1) **RISE-MICCAI mentorship** — ghép mentor 1 năm với mục tiêu cùng viết paper MICCAI; (2) cộng tác với lab AI/bệnh viện trong nước; (3) reach out nghiên cứu sinh bạn đã gặp qua open-source. Xem `resources/09`.
- **Tự review như reviewer:** trước khi đưa ai đọc, viết một review giả lập cho chính bài của mình (`resources/06` mục F). Bạn sẽ tìm ra 80% vấn đề mà reviewer thật sẽ nêu.

**Done khi:** có bản nháp một bài ngắn (workshop-style, 4–8 trang) với đóng góp rõ ràng, kết quả có ablation, và ít nhất một người ngoài đã đọc phản biện.

> ⚠️ **Xử lý thư giới thiệu từ Phase này, không phải Phase 5.** Đây là mắt xích yếu nhất của hồ sơ người tự học: bạn cần người *biết rõ năng lực nghiên cứu* của mình, và điều đó cần thời gian để xây. Xem `templates/lab-shortlist.md` mục D.

---

### Phase 4 — Nhắm một publication + khởi động hồ sơ PhD (Tháng 10–15)

**Mục tiêu:** biến bản nháp thành submission, và chuẩn bị hồ sơ.

- Chọn một **workshop** (dễ vào hơn main conference) tại MICCAI/CVPR/ICRA hoặc một venue phù hợp — nộp bài. Workshop paper / arXiv preprint đã là tín hiệu mạnh cho hồ sơ PhD.
- Nếu chưa đủ chín, đăng **arXiv preprint** — vẫn rất giá trị.
- Song song, làm theo `resources/05-phd-preparation.md` và điền `templates/lab-shortlist.md`:
  - Lập danh sách 10–20 giáo sư/lab phù hợp trục của bạn (đọc paper của họ, không chọn theo ranking trường). Điểm khởi đầu: `resources/09` mục A.
  - Chuẩn bị CV học thuật, xin thư giới thiệu (**bắt đầu sớm!**).
  - Thi English (IELTS/TOEFL) nếu apply nước ngoài; kiểm tra yêu cầu GRE của từng trường.
  - Đọc 5–10 **SOP thật** trên [cs-sop.notion.site](https://cs-sop.notion.site/) trước khi viết dòng đầu tiên.
  - **Đăng ký chương trình review hồ sơ miễn phí** (MIT EECS GAAP và tương tự — mở khoảng tháng 9–10). Xem `resources/05` mục F.
  - Nếu nhắm châu Âu: theo dõi [EURAXESS Jobs](https://euraxess.ec.europa.eu/jobs) — vị trí PhD có lương, tuyển quanh năm, phù hợp người đã đi làm.

**Done khi:** đã submit/preprint ít nhất một bài; có shortlist lab; CV học thuật xong; đã liên hệ ≥ 2 người viết thư giới thiệu.

---

### Phase 5 — Apply PhD (Tháng 15–18, theo mùa tuyển)

**Mục tiêu:** nộp hồ sơ mạnh.

- **Cold email** giáo sư mục tiêu (dùng mẫu trong `resources/05-phd-preparation.md`) từ mùa hè/đầu thu.
- Viết **Statement of Purpose** kể một câu chuyện nghiên cứu mạch lạc: engineer → tự học nghiên cứu → reproduce → đóng góp riêng → tại sao lab này.
- Nộp đúng deadline (Mỹ: thường tháng 12; châu Âu/Á: rải rác quanh năm, nhiều vị trí có lương theo dự án).

**Done khi:** đã nộp hồ sơ tới danh sách trường/lab mục tiêu.

---

## Phần 3 — Nhịp độ hàng tuần (giữ đều còn hơn học dồn)

Một tuần mẫu (~10–12h):

- **Đọc (3–4h):** 1 paper pass-3 + vài paper pass-1/2. Ghi note (`templates/paper-reading-note.md`).
- **Làm (5–6h):** code/thí nghiệm cho project hiện tại.
- **Viết & phản tư (1–2h):** cập nhật `logs/` tuần — 3 điều học được, 1 điều còn mắc. Viết **thành câu hoàn chỉnh**, không gạch đầu dòng cụt (viết là cách tư duy).

**Cuối mỗi tháng (1–2h):**

- Cập nhật `templates/literature-map.md` — đặc biệt mục "khoảng trống tôi nhìn thấy".
- Viết một bài ngắn 500–800 từ giải thích một khái niệm vừa học cho người ngoài ngành.
- Rà lại ROADMAP: hướng có thay đổi không? Nếu có, sửa file này — nó là tài liệu sống.
- Kiểm tra `resources/08-core-reading-list.md` còn cập nhật không (Track D thay đổi ~6 tháng/lần).

---

## Phần 4 — Sai lầm thường gặp khi engineer chuyển sang research

1. **Over-engineering.** Xây hệ thống hoành tráng thay vì trả lời một câu hỏi rõ ràng. Nghiên cứu cần thí nghiệm *sạch và tối giản*, không cần production-grade.
2. **Đọc quá nhiều, làm quá ít.** Đọc là để phục vụ một câu hỏi, không phải để "biết hết".
3. **Sợ scope nhỏ.** Đóng góp PhD-worthy đầu tiên thường rất hẹp. Hẹp mà sâu > rộng mà nông.
4. **Bỏ qua viết.** Viết là tư duy. Đợi "xong hết rồi mới viết" là công thức để không bao giờ viết.
5. **Chọn bài toán theo độ dễ, không theo tầm quan trọng.** (Hamming) Hãy hỏi: nếu giải được, nó có đáng không?
6. **Chọn lab theo ranking trường thay vì theo advisor & hướng.** Advisor và fit quan trọng hơn tên trường rất nhiều.

---

## Phần 5 — Cột mốc kiểm tra tiến độ (checklist)

- [ ] Có repo + statement định hướng (Phase 0)
- [ ] Đọc kỹ ≥ 20 paper có note, dựng literature map **có ≥3 khoảng trống ghi ra được** (Phase 1)
- [ ] Reproduce được ≥ 1 paper, có blog "lessons learned" (Phase 2)
- [ ] Có đề cương một đóng góp riêng + bản nháp bài viết (Phase 3)
- [ ] Có ≥1 người ngoài đã review nghiêm túc công việc của mình (Phase 3)
- [ ] Submit/preprint ≥ 1 bài (Phase 4)
- [ ] Shortlist 10–20 lab, CV học thuật, thư giới thiệu (Phase 4)
- [ ] Nộp hồ sơ PhD (Phase 5)

**Cột mốc "mềm" nhưng quan trọng không kém** (làm được sớm là dấu hiệu rất tốt):

- [ ] ≥1 PR được merge vào một repo chuẩn ngành (MONAI / nnU-Net / TorchIO / dVRK)
- [ ] Đã làm reviewer cho ≥1 workshop
- [ ] Đã ứng tuyển ≥1 summer school hoặc chương trình mentorship (RISE-MICCAI là ưu tiên số 1)
- [ ] Giải thích được đóng góp của mình trong **một câu**, cho người không cùng chuyên ngành

---

## Bản đồ tài liệu — dùng file nào khi nào

| Khi bạn đang… | Đọc file |
|---|---|
| Muốn nắm bức tranh & đổi tư duy | `resources/00-research-skills.md` |
| Lấp lỗ hổng toán/ML/CV | `resources/01-foundations.md` |
| Cần **paper cụ thể để đọc**, theo thứ tự | ⭐ `resources/08-core-reading-list.md` |
| Đào sâu trục chính | `resources/02-medical-imaging.md` |
| Cần nền robotics để bắc cầu | `resources/03-robotics.md` |
| Làm mũi nhọn surgical | `resources/04-surgical-robotics.md` |
| **Chuẩn bị chạy thí nghiệm** (đọc TRƯỚC, không đọc sau) | ⭐ `resources/07-experiments-and-rigor.md` |
| Bắt đầu viết / làm talk / trả lời reviewer | ⭐ `resources/06-writing-and-communication.md` |
| Muốn xây mạng lưới, tìm lab, tìm funding | `resources/09-community-labs-funding.md` |
| Chuẩn bị hồ sơ & apply | `resources/05-phd-preparation.md` |

---

*Xem chi tiết tài liệu cho từng phần trong thư mục `resources/`. Toàn bộ link và arXiv ID đã được kiểm chứng ngày 2026-07-29 — nếu bạn gặp link chết sau này, sửa ngay và ghi ngày kiểm tra lại.*
