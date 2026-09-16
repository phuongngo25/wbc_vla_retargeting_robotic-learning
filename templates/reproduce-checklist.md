# Reproduce Checklist — [Tên paper đang tái tạo]

*Dùng ở **Phase 2** của ROADMAP. Mục tiêu không phải "khớp số", mà là **học được nghiên cứu thực sự diễn ra thế nào**.*

- **Paper:** [Tác giả], "[Tiêu đề]", [Venue] [Năm] · [arXiv/DOI]
- **Code gốc:** [link hoặc "không có"]
- **Ngày bắt đầu:** YYYY-MM-DD · **Ngày kết thúc:** YYYY-MM-DD
- **Repo của tôi:** [link]

---

## 0. Trước khi viết dòng code đầu tiên

- [ ] Đã đọc paper tới **pass-3** và có note đầy đủ.
- [ ] Viết ra **con số cụ thể** tôi đang cố tái tạo (bảng nào, dòng nào, metric nào).
- [ ] Xác nhận dataset **tải được** và tôi có **đủ tài nguyên** (GPU-giờ, dung lượng đĩa).
- [ ] Ước lượng thời gian huấn luyện **một lần** chạy. Nếu > 3 ngày → chọn paper/tác vụ nhỏ hơn.
- [ ] Đặt **giới hạn thời gian** cho cả project (gợi ý: 4–6 tuần). Hết hạn thì viết lại những gì học được, dù chưa xong.

> ⚠️ **Cạm bẫy phổ biến nhất:** chọn paper quá to. Chọn paper có **code công khai** cho lần đầu tiên — mục tiêu là học quy trình, không phải chứng minh bản thân.

---

## 1. Ghi chép "chỗ paper không nói rõ" ⭐

> **Đây là phần giá trị nhất của toàn bộ bài tập này.** Mỗi lần bạn phải *đoán* một điều, ghi lại. Danh sách này chính là nội dung bài blog "lessons learned", và là bằng chứng bạn đã tư duy như nhà nghiên cứu.

| # | Chỗ paper không nói rõ | Tôi đã đoán / chọn gì | Vì sao chọn thế | Ảnh hưởng tới kết quả? |
|---|---|---|---|---|
| 1 | | | | |
| 2 | | | | |
| 3 | | | | |

**Những chỗ thường bị thiếu — kiểm tra từng cái:**

- [ ] Learning rate schedule chính xác (warmup? decay kiểu gì?)
- [ ] Chi tiết augmentation (thứ tự, xác suất, tham số)
- [ ] Cách chuẩn hoá dữ liệu (per-image? per-dataset? clip percentile nào?)
- [ ] Định nghĩa **split** train/val/test cụ thể (danh sách ca bệnh nào?)
- [ ] Tiêu chí chọn checkpoint (epoch cuối? val tốt nhất? theo metric nào?)
- [ ] Xử lý ảnh 3D: patch size, cách ghép (stitching), overlap
- [ ] Post-processing (có lấy connected component lớn nhất không? threshold bao nhiêu?)
- [ ] Cách **aggregate** metric (trung bình theo ca hay theo voxel? — xem `resources/07`)
- [ ] Số seed và cách báo cáo (mean? best?)

---

## 2. Hạ tầng tái lập (dựng NGAY, không để cuối)

- [ ] `git init`, commit thường xuyên; **tag** mỗi kết quả bạn ghi vào bảng.
- [ ] Môi trường khoá phiên bản: `environment.yml` / `requirements.txt` / `uv.lock`.
- [ ] Config tách khỏi code (Hydra/YAML); **mỗi run lưu lại config đã dùng**.
- [ ] Seed cố định; ghi lại phiên bản CUDA/cuDNN/PyTorch.
- [ ] Experiment tracking (W&B / MLflow) — **log cả run thất bại**.
- [ ] File split được **lưu vào repo**, không sinh lại ngẫu nhiên mỗi lần.
- [ ] `reproduce.sh` chạy từ đầu tới bảng kết quả.

---

## 3. Kết quả — bảng so sánh trung thực

| Metric | Paper báo cáo | Tôi đạt được | Chênh lệch | Giải thích khả năng nhất |
|---|---|---|---|---|
| | | | | |
| | | | | |

**Nếu lệch, kiểm tra theo thứ tự này** (từ nguyên nhân phổ biến nhất):

1. [ ] **Split khác nhau** — kiểm tra trước tiên, đây là nguyên nhân số 1.
2. [ ] Tiền xử lý / chuẩn hoá khác.
3. [ ] Cách **aggregate metric** khác (theo ca vs theo voxel).
4. [ ] Post-processing thiếu.
5. [ ] Ngân sách huấn luyện ít hơn (số epoch, batch size hiệu dụng).
6. [ ] Phiên bản dataset khác (dataset y tế **có** cập nhật nhãn theo thời gian).
7. [ ] Chỉ là dao động theo seed → chạy thêm 2 seed nữa trước khi kết luận.

> 💡 **Lệch không phải thất bại.** Một báo cáo trung thực "tôi đạt Dice 0.87 so với 0.91 của paper, và đây là 3 lý do khả năng nhất" **có giá trị nghiên cứu hơn** một con số khớp mà bạn không hiểu vì sao khớp.

---

## 4. Xem bằng mắt, không chỉ đọc số

- [ ] Đã xem **≥10 ca dự đoán tốt nhất** và **≥10 ca tệ nhất**.
- [ ] Ghi lại **kiểu lỗi** (failure mode) tôi thấy: …
- [ ] Kiểu lỗi này có được paper nêu không? (Thường là **không** — và đó là khoảng trống.)

---

## 5. ⭐ Câu hỏi mở tôi tìm được

> Đây là **hạt giống cho Phase 3**. Ghi cả những ý còn thô.

| # | Câu hỏi | Vì sao nó thú vị | Tôi có thể thử ngay không? |
|---|---|---|---|
| 1 | | | |
| 2 | | | |
| 3 | | | |

---

## 6. Đầu ra bắt buộc (definition of done của Phase 2)

- [ ] Repo công khai, `README` ghi rõ: cách chạy, hardware, thời gian, **và mọi khác biệt so với paper gốc**.
- [ ] Bảng kết quả ở mục 3 (kể cả khi không khớp).
- [ ] Bài viết 800–1500 từ: *"Những gì tôi học được khi reproduce [X]"* — tập trung vào **mục 1 và mục 5**, không phải vào con số.
- [ ] Đã gửi link cho **tác giả gốc** (email ngắn, lịch sự). Họ thường trả lời, và đây là cách tự nhiên nhất để bắt đầu một mối quan hệ nghiên cứu.
- [ ] Đã cập nhật `templates/literature-map.md` với những gì học được.

---

*Liên quan: `resources/07-experiments-and-rigor.md` (rò rỉ dữ liệu, metric, reproducibility) · `templates/paper-reading-note.md` · `templates/project-proposal.md` (bước tiếp theo, từ câu hỏi ở mục 5).*
