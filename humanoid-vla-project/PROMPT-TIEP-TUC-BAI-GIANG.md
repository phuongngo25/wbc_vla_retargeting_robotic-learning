# Prompt: Tiếp tục sinh các bài giảng (BAI-GIANG-*.md) còn thiếu (v2)

> **v2 — thay thế hoàn toàn bản trước.** Bản v1 cho phép gộp 2-3 khái niệm/file — đây là lỗi đã phát hiện (bài giảng quá ngắn, quá gộp, không cập nhật kiến thức mới). Bản này: **nghiêm cấm gộp**, mỗi khái niệm là 1 file riêng, bắt buộc tra cứu bổ sung kiến thức hiện đại, và có bước dọn dẹp các file đã gộp sai trước đó.
>
> Dùng file này khi bắt đầu một phiên hội thoại MỚI (Claude Code, đang mở repo `research-journey`) để tiếp tục công việc sinh bài giảng cho `humanoid-vla-project/`. Copy toàn bộ nội dung trong khối lệnh bên dưới, dán làm tin nhắn đầu tiên.

---

## PROMPT (copy từ đây)

```
Bạn đang làm việc trong repo `research-journey`, thư mục `humanoid-vla-project/`. Đây là dự án học tập cá nhân về humanoid robot (Whole-Body Control, Motion Retargeting, VLA/GR00T-SONIC...), có 8 thư mục con (01→08), mỗi thư mục đã có:
- `README.md` — mục lục các khái niệm + bảng paper nguồn đã kiểm chứng.
- `NOI-DUNG-CHI-TIET.md` — "giáo trình cô đặc": định nghĩa + cơ chế + công thức cho từng khái niệm, đây là NGUỒN SỰ THẬT cho phần khái niệm cốt lõi (không mâu thuẫn với nó), nhưng KHÔNG phải nguồn duy nhất — bài giảng phải bổ sung tra cứu hiện đại (xem bước 2 dưới).

Ở cấp `humanoid-vla-project/` có file `PROMPT-TAO-BAI-GIANG.md` (bản v2) — chứa prompt chuẩn + nguyên tắc bắt buộc + cấu trúc chi tiết + PHỤ LỤC liệt kê TOÀN BỘ khái niệm đơn lẻ (mỗi dòng trong bảng Phụ lục = 1 file bài giảng bắt buộc) của cả 8 thư mục. File này là checklist công việc chính thức, duy nhất — dùng đúng danh sách trong đó, đừng tự chia lại granularity theo cách khác.

## BƯỚC 0 — Dọn dẹp 4 file đã gộp sai (làm TRƯỚC, chỉ 1 lần)

Bản v1 trước đây đã tạo 4 file gộp nhiều khái niệm vào 1 file — VI PHẠM quy tắc "1 khái niệm = 1 file" của v2. Nếu các file sau còn tồn tại, hãy XOÁ chúng (sau khi xác nhận nội dung đã được thay thế bằng các file tách riêng ở bước sau, hoặc xoá luôn rồi tạo lại — không giữ song song bản gộp và bản tách):

- `01-whole-body-control/BAI-GIANG-operational-space-qp-hqp.md` → tách thành 3 file riêng: "Task-space/Operational-space control", "Quadratic Programming (QP) trong WBC", "Hierarchical QP (HQP)".
- `01-whole-body-control/BAI-GIANG-zmp-cart-table-mpc.md` → tách thành 3 file riêng: "ZMP và mô hình cart-table", "ZMP preview control", "MPC cho dáng đi".
- `02-motion-retargeting/BAI-GIANG-bai-toan-constraint-preservation.md` → tách thành 2 file riêng: "Vì sao không thể copy trực tiếp góc khớp", "Ý tưởng gốc: Gleicher (1998)".
- `02-motion-retargeting/BAI-GIANG-rang-buoc-vat-ly-va-cac-truong-phai.md` → tách thành 5 file riêng: "Foot contact stabilization", "Joint limit clamping & velocity limiting", "GMR — kiến trúc và pipeline", "SOMA-retargeter — kiến trúc và pipeline", "Retargeting học sâu/residual".
- `02-motion-retargeting/BAI-GIANG-skeleton-mapping-ik-per-frame.md` (tạo trước cả bản v1, từ yêu cầu riêng của user) → cũng vi phạm granularity mới, tách thành 3 file riêng: "Skeleton mapping", "Inverse Kinematics per-frame — thuật toán Jacobian-based/differential IK", "Inverse Kinematics per-frame — thuật toán FABRIK".

Sau khi xoá 4 file gộp này, cũng xoá dòng "🎓 Bài giảng đã có" tương ứng trong `README.md` của 2 thư mục 01 và 02 (sẽ được ghi lại đúng, từng dòng riêng cho từng file mới, ở bước 3 dưới).

## BƯỚC 1 — Xác định phạm vi còn thiếu

1. Mở `humanoid-vla-project/PROMPT-TAO-BAI-GIANG.md`, đọc kỹ toàn bộ (nguyên tắc cốt lõi, prompt, cấu trúc bắt buộc, và bảng Phụ lục).
2. Với mỗi thư mục 01→08 (thứ tự: hoàn thành nốt 02, rồi 01, 03, 04, 05, 06, 07, 08):
   - Liệt kê file `BAI-GIANG-*.md` hiện có (dùng `ls`/Glob).
   - Đối chiếu với đúng bảng Phụ lục của thư mục đó — mỗi DÒNG trong bảng là 1 khái niệm bắt buộc phải có 1 file riêng.
   - Xác định các dòng CHƯA có file tương ứng.

## BƯỚC 2 — Sinh từng file, ĐÚNG QUY TRÌNH trong PROMPT-TAO-BAI-GIANG.md

Với mỗi khái niệm còn thiếu, theo đúng thứ tự trong bảng Phụ lục:

1. Đọc đúng đoạn nội dung tương ứng khái niệm đó trong `NOI-DUNG-CHI-TIET.md` của thư mục (không cần dán ra ngoài, đọc trực tiếp từ đĩa).
2. **BẮT BUỘC chạy WebSearch/WebFetch** (tối thiểu 2-3 lượt) để tìm thông tin cập nhật 2024-2026 về khái niệm này — đây là yêu cầu KHÔNG được bỏ qua, khác biệt lớn nhất so với bản v1.
3. Viết bài giảng theo ĐÚNG cấu trúc 13 mục đã quy định trong `PROMPT-TAO-BAI-GIANG.md` (Mục tiêu, Bối cảnh, Trực giác 2 góc nhìn, Định nghĩa, Cơ chế từng bước, Ví dụ tính tay, So sánh, Sai lầm thường gặp, Ví dụ trong dự án, Cập nhật hiện đại có trích dẫn, Câu hỏi tự kiểm tra, Bài tập, Tóm tắt).
4. **KHÔNG gộp với bất kỳ khái niệm nào khác** — kể cả khi cùng 1 mục lớn trong nguồn, kể cả khi rất liên quan, kể cả khi nội dung nguồn ngắn. Nếu nội dung nguồn quá ngắn, bù bằng cách đào sâu hơn (ví dụ số, so sánh, cập nhật hiện đại) — không bù bằng cách gộp.
5. Đặt tên file mô tả ĐÚNG một khái niệm đó: `BAI-GIANG-<slug-kebab-case-không-dấu>.md`, lưu trong đúng thư mục chứa `NOI-DUNG-CHI-TIET.md` nguồn.
6. Độ dài: không giới hạn trần, sàn tối thiểu ~300-500 dòng cho khái niệm có công thức/thuật toán, ~200 dòng cho khái niệm còn lại — nếu bài viết ra ngắn hơn nhiều so với mức này, quay lại bổ sung thêm (ví dụ số, so sánh, sai lầm thường gặp, cập nhật hiện đại) trước khi coi là hoàn thành.
7. Sau khi tạo file, cập nhật `README.md` cùng thư mục: thêm 1 dòng blockquote mới NGAY DƯỚI các dòng "🎓 Bài giảng đã có" đã có sẵn (mỗi khái niệm 1 dòng riêng, đừng gộp nhiều link vào 1 dòng):
   `> 🎓 **Bài giảng đã có**: [\`BAI-GIANG-xxx.md\`](BAI-GIANG-xxx.md) — <mô tả ngắn, đúng 1 khái niệm>.`

## BƯỚC 3 — Nhịp độ làm việc

- Làm từng khái niệm một, không cần dừng hỏi xác nhận giữa chừng trừ khi gặp mâu thuẫn/không chắc chắn thật sự.
- Sau khi xong TOÀN BỘ khái niệm của 1 thư mục, báo cáo ngắn gọn rồi chuyển sang thư mục tiếp theo.
- Nếu không làm hết trong 1 phiên, báo cáo rõ đã xong khái niệm/thư mục nào — lần sau dùng lại đúng prompt này (`PROMPT-TIEP-TUC-BAI-GIANG.md`), bước 1 sẽ tự phát hiện phần đã làm (miễn KHÔNG có file gộp nào sót lại từ bản v1 — luôn kiểm tra lại Bước 0 nếu nghi ngờ).

## Phạm vi 1 phiên làm việc

Vì mỗi file giờ yêu cầu tra cứu web + độ sâu lớn hơn nhiều so với trước, số lượng file hoàn thành mỗi phiên sẽ ít hơn bản v1. Đây là đánh đổi CHỦ ĐỘNG và ĐÚNG YÊU CẦU — ưu tiên chất lượng/độ đầy đủ hơn số lượng file hoàn thành trong 1 phiên.
```

---

## Ghi chú khi dùng

- Prompt trên **tự chứa đủ ngữ cảnh** — phiên hội thoại mới không cần biết gì thêm, chỉ cần quyền đọc/ghi file + quyền dùng WebSearch/WebFetch trong repo này.
- Nếu muốn giới hạn phạm vi 1 phiên (ví dụ chỉ làm riêng 1 thư mục, hoặc chỉ làm bước dọn dẹp Bước 0), thêm 1 dòng vào cuối prompt trước khi gửi, ví dụ: `"Chỉ làm Bước 0 (dọn dẹp) và các khái niệm còn thiếu của 02-motion-retargeting/ trong phiên này."`
