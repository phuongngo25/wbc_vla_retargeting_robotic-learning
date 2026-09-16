# 07 — Thiết kế thí nghiệm, Thống kê & Thẩm định

*Cập nhật: 2026-07-29.*

Đây là file **quan trọng thứ hai** sau `00-research-skills.md`, và là nơi engineer dễ mắc lỗi nhất. Engineer đo để **biết hệ thống có chạy không**. Researcher đo để **kết luận một điều đúng về thế giới** — và phải bảo vệ được kết luận đó trước người muốn chứng minh bạn sai.

> **Sự thật khó chịu:** phần lớn "cải tiến +2% Dice" trong paper là nhiễu, tuning không công bằng, hoặc rò rỉ dữ liệu. Nếu bạn học được cách **không tự lừa mình**, bạn đã hơn phần lớn người viết paper.

---

## A. Từ câu hỏi đến thí nghiệm — chuỗi bắt buộc

```
Câu hỏi nghiên cứu
   └─> Giả thuyết (phát biểu có thể SAI được)
         └─> Dự đoán cụ thể ("nếu giả thuyết đúng, tôi sẽ thấy X")
               └─> Thí nghiệm phân biệt được X với không-X
                     └─> Metric đo được X
                           └─> Kết luận (chỉ mạnh đến mức bằng chứng cho phép)
```

**Test giả thuyết tốt:** viết ra **kết quả nào sẽ bác bỏ nó**. Nếu không viết được → đó không phải giả thuyết, chỉ là mong đợi.

**Tự hỏi trước khi chạy thí nghiệm (không phải sau):**
- Nếu kết quả *ngược lại* điều tôi mong, tôi có coi đó là phát hiện không? (Nếu không → thiết kế sai.)
- Có cách giải thích nào khác cho kết quả này ngoài giả thuyết của tôi? (Đó là confound cần kiểm soát.)
- Bao nhiêu là "đủ khác biệt để đáng quan tâm"? Quyết định **trước**, không sau khi thấy số.

---

## B. Baseline, ablation, control — ba chân của bằng chứng

### Baseline
- **Baseline yếu là gian lận vô ý.** Baseline phải được tune với **cùng ngân sách** (số lần thử hyperparameter, cùng augmentation, cùng số epoch) như method của bạn.
- Trong medical imaging, baseline **bắt buộc** phải có: **nnU-Net** với cấu hình mặc định. Nó thắng rất nhiều method "mới" — nếu bạn không so với nó, reviewer sẽ hỏi.
- Luôn có **baseline tầm thường** (trivial): dự đoán trung bình, atlas cố định, hoặc hình học đơn giản. Nó cho biết bài toán *thực sự* khó cỡ nào.

### Ablation
Mục tiêu: **cô lập nguyên nhân**. Method của bạn có 3 thành phần → cần ≥3 dòng ablation, mỗi dòng tắt một thành phần.

Cạm bẫy: ablation "cộng dồn" (baseline → +A → +A+B → +A+B+C) **không** cho biết B có tác dụng độc lập hay chỉ hữu ích khi có A. Nếu quan trọng, làm cả hai chiều (bỏ từng cái ra khỏi bản đầy đủ).

### Control
- **Seed control:** cùng seed cho mọi phương pháp trong một lần so sánh.
- **Compute control:** cùng số tham số / FLOPs, hoặc nêu rõ nếu không.
- **Data control:** cùng split, cùng tiền xử lý.

---

## C. Nhiều seed, phương sai, và tại sao "một con số" là vô nghĩa

- Chạy **≥3 seed, tốt nhất 5**, báo cáo **trung bình ± độ lệch chuẩn** (hoặc khoảng tin cậy).
- **Không** báo cáo "best of N" trừ khi baseline cũng được lấy best of N — và nói rõ N.
- Nếu khoảng ±std của bạn chồng lên baseline → **bạn chưa chứng minh được gì.** Hãy nói thẳng điều đó; nó vẫn là một kết quả có giá trị.

| Tài liệu | Nội dung | Link |
|---|---|---|
| **Torch.manual_seed(3407) is all you need** — Picard | Cho thấy chỉ đổi seed đã tạo dao động lớn hơn nhiều "cải tiến" được công bố. Vui nhưng nghiêm túc. | [arXiv:2109.08203](https://arxiv.org/abs/2109.08203) |
| **Show Your Work** — Dodge et al. | Cách báo cáo có tính đến ngân sách tuning. | [arXiv:1909.03004](https://arxiv.org/abs/1909.03004) |
| **A Metric Learning Reality Check** — Musgrave et al. | Khi so sánh công bằng, tiến bộ nhiều năm bốc hơi. | [arXiv:2003.08505](https://arxiv.org/abs/2003.08505) |

---

## D. Rò rỉ dữ liệu (data leakage) — sát thủ số 1 trong ảnh y tế

**Đây là lỗi khiến paper y tế bị rút nhiều nhất.** Học kỹ phần này.

| Loại rò rỉ | Ví dụ trong ảnh y tế | Cách tránh |
|---|---|---|
| **Rò rỉ theo bệnh nhân** | Chia slice/frame ngẫu nhiên → slice của cùng một bệnh nhân nằm ở cả train và test. | **Luôn chia theo patient ID.** Với video nội soi: chia theo ca mổ. |
| **Rò rỉ tiền xử lý** | Tính normalization/statistics trên toàn bộ dataset trước khi chia. | Tính chỉ trên train, áp dụng cho test. |
| **Rò rỉ chọn model** | Chọn epoch/threshold tốt nhất dựa trên test set. | Dùng validation set riêng; test set mở **một lần**. |
| **Rò rỉ theo thời gian / theo máy** | Train và test cùng một máy scan / cùng bệnh viện → thổi phồng hiệu năng. | Có ít nhất một **external test set** khác trung tâm. |
| **Rò rỉ qua nhãn** | Đặc trưng nào đó chứa thông tin nhãn (ví dụ marker vật lý trên phim X-quang chỉ có ở ca dương tính). | Kiểm tra hình ảnh thủ công; test trên dữ liệu che vùng nghi vấn. |
| **Rò rỉ qua tăng cường dữ liệu** | Augment rồi mới chia. | Chia trước, augment sau — chỉ trên train. |

> ⭐ Đọc bắt buộc: **Kapoor & Narayanan — "Leakage and the Reproducibility Crisis in ML-based Science"** — [arXiv:2207.07048](https://arxiv.org/abs/2207.07048). Có taxonomy 8 loại rò rỉ + template "model info sheet" để tự kiểm.

---

## E. Metric — chọn sai metric làm vô nghĩa toàn bộ nghiên cứu

Đây là chỗ mảng ảnh y tế đã có **tài liệu chuẩn mực bậc nhất** trong toàn ngành ML. Nếu bạn nắm phần này, bạn nghe như người đã làm nghề nhiều năm.

| Tài liệu | Vì sao quan trọng | Link |
|---|---|---|
| **Metrics Reloaded: recommendations for image analysis validation** — Maier-Hein et al., *Nature Methods* 2024 | ⭐⭐ Framework đồng thuận quốc tế: mô tả bài toán bằng "problem fingerprint" rồi chọn metric phù hợp, kèm cảnh báo cạm bẫy. **Tài liệu tham chiếu số 1 khi thiết kế đánh giá cho ảnh y tế.** | [Nature Methods](https://www.nature.com/articles/s41592-023-02151-z) · [arXiv:2206.01653](https://arxiv.org/abs/2206.01653) · [Code (MONAI)](https://github.com/Project-MONAI/MetricsReloaded) |
| **Understanding metric-related pitfalls in image analysis validation** — Reinke et al., *Nature Methods* 2024 | ⭐ Bài chị em của trên: liệt kê cụ thể từng cạm bẫy và ví dụ hình ảnh. | [arXiv:2302.01790](https://arxiv.org/abs/2302.01790) |
| **Common Limitations of Image Processing Metrics: A Picture Story** — Reinke et al. | Phiên bản "kể bằng hình" — dễ nhớ, treo cạnh bàn làm việc. | [arXiv:2104.05642](https://arxiv.org/abs/2104.05642) |

**Những điều cụ thể phải biết:**

- **Dice (DSC) đơn độc là không đủ.** Dice thiên vị cấu trúc lớn; với cấu trúc nhỏ (tổn thương vài voxel) nó dao động dữ dội và không phản ánh ý nghĩa lâm sàng.
- Ghép **metric dựa trên vùng** (Dice, IoU) với **metric dựa trên biên** (Normalized Surface Distance NSD, HD95). Với can thiệp phẫu thuật, sai số biên mới là cái quan trọng.
- **Không dùng Hausdorff thô** (rất nhạy outlier) — dùng **HD95**.
- Với **phát hiện tổn thương**: dùng metric theo *tổn thương* (per-lesion sensitivity, FROC), không chỉ theo voxel.
- Với **mất cân bằng lớp nặng** (điển hình trong y tế): accuracy vô nghĩa; dùng **AUPRC** thay AUROC khi lớp dương rất hiếm; báo cáo sensitivity ở specificity cố định (điều bác sĩ thực sự cần).
- **Aggregate đúng cấp:** trung bình Dice theo *ca bệnh*, không theo tổng voxel toàn dataset — nếu không, một ca lớn sẽ chi phối kết quả.
- Với **ca âm tính hoàn toàn** (không có tổn thương): Dice không định nghĩa được. Phải nêu rõ cách xử lý (thường là báo cáo riêng).

**Hiệu chuẩn (calibration) — bắt buộc nếu nói tới ứng dụng lâm sàng:**
Model nói "90% chắc chắn" thì có đúng 90% số lần không? Đo bằng **ECE** + reliability diagram. Xem `08-core-reading-list.md` mục Uncertainty.

---

## F. Thống kê cho ML researcher — vừa đủ, làm đúng

**Nguyên tắc:** báo cáo **effect size + khoảng tin cậy**, không chỉ p-value.

| Tình huống | Test phù hợp |
|---|---|
| So 2 method trên **cùng** tập ca bệnh | **Paired** test: Wilcoxon signed-rank (không giả định phân phối) hoặc paired t-test. |
| So 2 method trên tập độc lập | Mann–Whitney U / unpaired t-test. |
| So > 2 method | Friedman test + post-hoc (Nemenyi), **có** hiệu chỉnh đa so sánh (Holm/Bonferroni). |
| Khoảng tin cậy cho bất kỳ metric | **Bootstrap** (lấy mẫu lại ca bệnh có hoàn lại, 1000–10000 lần). Đơn giản, không giả định gì, luôn dùng được. |
| Không biết chọn gì | Bootstrap CI + permutation test. Hai công cụ này giải quyết 90% trường hợp. |

**Cạm bẫy:**
- **Đa so sánh:** thử 20 biến thể rồi báo cái p<0.05 → gần như chắc chắn là nhiễu.
- **HARKing** (đặt giả thuyết sau khi thấy kết quả) và **p-hacking**: chốt kế hoạch phân tích **trước** khi xem test set. Với project quan trọng, viết pre-registration ngắn vào `templates/project-proposal.md`.
- **n nhỏ:** dataset y tế thường 30–200 ca. Với n=30, khoảng tin cậy rất rộng — hãy trung thực rằng bạn không thể phát hiện khác biệt nhỏ.

**Học thống kê (chọn một, học cho xong):**

| Tài liệu | Vì sao | Link |
|---|---|---|
| **Statistical Rethinking** — Richard McElreath | ⭐ Bài giảng (YouTube) + sách hay nhất để *hiểu* thống kê thay vì thuộc công thức. Cách tiếp cận Bayes, rất phù hợp tư duy nghiên cứu. | [xcelab.net/rm](https://xcelab.net/rm/) |
| **An Introduction to Statistical Learning** (ISL / ISLP) | Miễn phí, có bản Python. Nền ML thống kê + cross-validation làm đúng. | [statlearning.com](https://www.statlearning.com) |
| **Introduction to Probability** — Blitzstein (Harvard Stat 110) | Xác suất trực giác. | [Stat 110](https://projects.iq.harvard.edu/stat110) |

---

## G. Reporting guidelines cho AI y tế — dấu hiệu của người chuyên nghiệp

Y học có văn hoá **checklist báo cáo** mà ML thuần không có. Biết và dùng chúng làm paper của bạn nổi bật ngay lập tức, và là điều **bắt buộc** với tạp chí y khoa.

| Guideline | Dùng cho | Link |
|---|---|---|
| **FUTURE-AI** (BMJ 2025) | ⭐ Đồng thuận quốc tế (117 chuyên gia, 50 nước) về AI y tế đáng tin cậy: 6 nguyên tắc (fairness, universality, traceability, usability, robustness, explainability) + 30 best practice cho toàn vòng đời. Đọc để định hình cách bạn *thiết kế* nghiên cứu, không chỉ báo cáo. | [PubMed 39961614](https://pubmed.ncbi.nlm.nih.gov/39961614/) |
| **CLAIM** (Checklist for AI in Medical Imaging) | Checklist báo cáo cho paper AI ảnh y tế (bản cập nhật 2024 trên *Radiology: AI*). Dùng như checklist cuối trước khi nộp. | Tìm "CLAIM 2024 checklist Radiology Artificial Intelligence" |
| **TRIPOD+AI** (BMJ 2024) | Báo cáo model dự đoán lâm sàng — chuẩn nếu paper của bạn có yếu tố tiên lượng/chẩn đoán. | [tripod-statement.org](https://www.tripod-statement.org) |
| **PROBAST+AI** | Đánh giá **risk of bias** của model dự đoán — dùng khi làm systematic review. | Tìm trên [Equator Network](https://www.equator-network.org) |
| **STARD-AI** | Nghiên cứu độ chính xác chẩn đoán. | [Equator Network](https://www.equator-network.org) |
| **CONSORT-AI / SPIRIT-AI / DECIDE-AI** | Thử nghiệm lâm sàng có AI, và đánh giá giai đoạn đầu tại giường bệnh. | [Equator Network](https://www.equator-network.org) |
| **BIAS** (Biomedical Image Analysis ChallengeS) | Báo cáo **challenge** — quan trọng nếu bạn tham gia hoặc phân tích kết quả challenge. | Tìm "BIAS reporting guideline biomedical image analysis challenges" |
| **Equator Network** | Trung tâm tra cứu **mọi** guideline báo cáo y sinh. Bookmark. | [equator-network.org](https://www.equator-network.org) |

> 💡 **Về challenge/leaderboard:** thứ hạng trong challenge **rất không ổn định** — đổi cách aggregate metric là đổi người thắng. Đọc Maier-Hein et al., *"Why rankings of biomedical image analysis competitions should be interpreted with care"* (Nature Communications 2018) trước khi tự hào về rank, hoặc trước khi tin rank của người khác.

---

## H. Tính tái lập (reproducibility) — hạ tầng, không phải thiện chí

**Chuẩn tối thiểu cho mỗi project (áp dụng từ Phase 2):**

- [ ] `git` cho code; **tag** commit tương ứng với mỗi kết quả trong paper.
- [ ] Môi trường khoá phiên bản: `environment.yml` / `requirements.txt` (hoặc `uv.lock`), tốt nhất là **Dockerfile**.
- [ ] Config tách khỏi code — [Hydra](https://hydra.cc) hoặc YAML. **Mỗi run lưu lại config đã dùng.**
- [ ] Seed cố định và ghi lại; ghi cả phiên bản CUDA/cuDNN (kết quả GPU không hoàn toàn tất định).
- [ ] Phiên bản dữ liệu: ghi rõ nguồn, ngày tải, checksum; lưu **file split** vào repo (không phải sinh lại ngẫu nhiên).
- [ ] Experiment tracking: [Weights & Biases](https://wandb.ai) hoặc [MLflow](https://mlflow.org). Log **mọi** run, kể cả run thất bại.
- [ ] Một script `reproduce.sh` chạy được **từ đầu đến bảng kết quả**.
- [ ] `README` ghi rõ: hardware, thời gian chạy, và **những chỗ bạn không tái lập được** của paper gốc.

**Tài liệu chuẩn:**
- [ML Reproducibility Checklist — Joelle Pineau](https://www.cs.mcgill.ca/~jpineau/ReproducibilityChecklist.pdf) — bản gốc, ngắn.
- [ML Reproducibility Challenge](https://reproml.org) — tham gia để luyện (và có thể có publication).
- **NeurIPS Paper Checklist** — tìm trên site NeurIPS năm hiện tại; đây là checklist bắt buộc khi nộp, rất tốt để tự soi.
- **Datasheets for Datasets** — Gebru et al., [arXiv:1803.09010](https://arxiv.org/abs/1803.09010): mô tả dataset chuẩn mực.
- **Model Cards for Model Reporting** — Mitchell et al., [arXiv:1810.03993](https://arxiv.org/abs/1810.03993).

---

## I. Đạo đức & dữ liệu bệnh nhân

Không phải hình thức — đây là điều kiện để nghiên cứu của bạn dùng được trong thực tế.

- **Nguồn dữ liệu:** chỉ dùng dataset công khai có license rõ ràng ở Phase 2–3. Đọc license: một số dataset **cấm dùng thương mại**, một số **cấm redistribute**, một số yêu cầu **trích dẫn cụ thể**.
- **Credentialed access:** MIMIC-CXR / PhysioNet yêu cầu hoàn thành khoá **CITI "Data or Specimens Only Research"** + ký DUA. Bắt đầu sớm, mất vài tuần.
- **Ẩn danh hoá:** DICOM chứa PHI trong metadata (tên, ngày sinh, ID) **và** có thể trong pixel (burned-in annotation). Dùng công cụ ẩn danh hoá chuyên dụng, đừng tự viết.
- **Nghiên cứu trên dữ liệu mới:** cần **IRB / hội đồng đạo đức** duyệt trước khi thu thập. Nếu bạn cộng tác với bệnh viện, đây là bước dài nhất — hỏi ngay từ đầu.
- **Công bằng (fairness):** báo cáo hiệu năng **phân tầng** theo nhóm (tuổi, giới, chủng tộc, máy scan, trung tâm) khi dữ liệu cho phép. Có bằng chứng rõ ràng rằng model ảnh y tế **chẩn đoán sót nhiều hơn ở nhóm yếu thế** — xem mục Fairness trong `08-core-reading-list.md`.
- **Không tuyên bố lâm sàng vượt bằng chứng.** "Đạt Dice 0.91 trên BraTS" ≠ "dùng được trong phòng mổ". Reviewer y tế rất nhạy với chuyện này.

---

## J. Danh sách kiểm tra cuối — dán lên tường

Trước khi tin vào kết quả của **chính mình**:

```
[ ] Split theo bệnh nhân/ca mổ, không theo slice/frame
[ ] Test set chỉ mở một lần, sau khi chốt mọi lựa chọn
[ ] Baseline được tune cùng ngân sách với method của tôi
[ ] nnU-Net (hoặc baseline mạnh chuẩn ngành) đã có trong bảng
[ ] ≥3 seed, báo cáo mean ± std
[ ] Metric chọn theo Metrics Reloaded, có cả region-based và boundary-based
[ ] Có khoảng tin cậy (bootstrap) cho khác biệt chính
[ ] Có ablation cô lập từng thành phần
[ ] Có xem ca thất bại bằng mắt (không chỉ đọc số)
[ ] Có ít nhất một test set ngoại vi / khác phân phối
[ ] Đã tự trả lời: "cách giải thích nào khác cho kết quả này?"
[ ] reproduce.sh chạy được từ máy sạch
```

> Nếu một dòng không tick được — hãy **viết ra trong mục Limitations** thay vì im lặng. Reviewer tha cho hạn chế được nêu rõ; họ không tha cho hạn chế bị che.

---

*Liên quan: `06-writing-and-communication.md` (viết ra bằng chứng này) · `08-core-reading-list.md` (paper nền về metric, uncertainty, fairness) · `templates/project-proposal.md` · `templates/reproduce-checklist.md`.*
