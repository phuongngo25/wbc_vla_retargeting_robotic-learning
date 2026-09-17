# Bài giảng: N1.5 → N1.6: những gì thay đổi

*(Thuộc mảng: Vision-Language-Action: GR00T N1 → N1.7 & SONIC)*

## 🎯 Mục tiêu bài học

- Giải thích được vì sao N1.5 và N1.6 **không có paper kỹ thuật riêng** (không như N1 gốc và N1.7), chỉ có model card HuggingFace + GitHub release notes — và tại sao điều này quan trọng khi bạn trích dẫn thông tin về chúng ở nơi khác.
- Liệt kê được ít nhất **4 thay đổi có nguồn chính thức xác nhận** giữa N1.5 và N1.6 (không phải đoán, không phải nguồn thứ cấp) — kèm đúng link nguồn.
- Phân biệt rõ, cho từng tuyên bố về N1.5/N1.6, ba mức độ tin cậy: **xác nhận chính thức** (model card/blog NVIDIA) / **nguồn thứ cấp** (bài viết cộng đồng) / **suy đoán** (chưa có nguồn nào xác nhận).
- Vẽ đúng vị trí N1.5, N1.6 trên timeline phát triển GR00T: N1 (paper, 3/2025) → N1.5 → N1.6 → N1.7 (blog).
- Nhận ra và tự sửa được một hiểu lầm phổ biến: gọi N1.5/N1.6 là "chỉ cải thiện dữ liệu, không đổi kiến trúc" — sau khi tra cứu, tuyên bố này **đúng một phần cho N1.5 nhưng sai đáng kể cho N1.6** (N1.6 thực ra đổi kiến trúc không nhỏ).
- Biết đúng quy trình tự tra cứu (đọc model card HuggingFace + GitHub releases) để tự cập nhật khi có bản mới hơn xuất hiện sau khi bài giảng này được viết.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Trong lộ trình học `06-vla-groot-sonic/README.md` (mục D), bạn được khuyên đọc kỹ N1 gốc (bước 2), rồi đọc blog N1.7 (bước 3) — nhưng ở giữa hai mốc đó có hai bản phát hành mà tài liệu gốc của dự án (`NOI-DUNG-CHI-TIET.md` mục 3.1) đã thẳng thắn ghi nhận là "thiếu thông tin": N1.5 và N1.6. Đây không phải là hai bản phụ vô nghĩa để bỏ qua — chúng nằm đúng ở khoảng trống giữa "kiến trúc dual-system nền tảng" (N1) và "bước nhảy vọt về nguồn dữ liệu EgoScale" (N1.7), và là nơi NVIDIA thực sự tinh chỉnh cả kiến trúc lẫn công thức dữ liệu trước khi đặt cược lớn vào video egocentric ở N1.7.

Lý do bài giảng này xứng đáng có một file riêng, dù nguồn mỏng hơn các bài khác: đây là bài học tốt nhất trong toàn bộ dự án về **cách làm việc với thông tin về một hệ thống mã nguồn mở đang phát triển rất nhanh, không có paper bình duyệt cho mọi phiên bản**. Kỹ năng "phân biệt được cái gì đã xác nhận, cái gì mới là đồn đoán" quan trọng ngang với bản thân kiến thức kỹ thuật — vì bạn sẽ liên tục gặp tình huống này khi theo dõi các hệ thống robot học AI cập nhật theo tuần/tháng thay vì theo chu kỳ xuất bản paper (thường 6-12 tháng).

## 🧠 Trực giác

### Góc nhìn 1: "Bản cập nhật điểm" (point release) trong phần mềm

Trong quy ước đặt tên phiên bản phần mềm quen thuộc (semantic versioning), một bước nhảy `1.0 → 2.0` ("major version") thường đi kèm thay đổi lớn, có thể phá vỡ tương thích ngược; còn `1.0 → 1.1 → 1.2` ("minor/point release") thường chỉ là vá lỗi, cải thiện nhỏ, tối ưu hiệu năng — sản phẩm về cơ bản vẫn "là nó". Nhìn theo góc này, `N1 → N1.7` giống một bước nhảy lớn (đổi hẳn nguồn dữ liệu pretraining, phát hiện scaling law mới), còn `N1 → N1.5 → N1.6` giống chuỗi "point release" — cải thiện dần dần trên cùng một công thức nền.

- **Đúng ở đâu:** nắm đúng trực giác tổng thể — N1.5/N1.6 đúng là các bước đệm nhỏ hơn N1.7 về mức độ "gây chấn động" (không có phát hiện kiểu "scaling law" nào được công bố cho N1.5/N1.6).
- **Giới hạn quan trọng (xem mục Cập nhật hiện đại):** phép loại suy này **đánh giá thấp mức độ thay đổi kiến trúc thực tế ở N1.6** — số đo phiên bản `x.5 → x.6` trong tên gọi của NVIDIA **không tuân theo quy ước semantic versioning nghiêm ngặt** của ngành phần mềm. Nghiên cứu cho bài giảng này phát hiện N1.6 thực ra tăng gấp đôi kích thước diffusion transformer, đổi hẳn VLM backbone, và đổi cách biểu diễn hành động (relative thay vì absolute) — đây là mức thay đổi kiến trúc mà trong quy ước phần mềm thông thường đáng được gọi là "major version", không phải "point release". Đừng suy ra mức độ thay đổi kỹ thuật chỉ từ cách đặt tên số phiên bản.

### Góc nhìn 2: "Luyện tập thêm với bộ kỹ năng cũ" vs "học kỹ năng hoàn toàn mới"

Hãy tưởng tượng một vận động viên: giai đoạn `N1.5` giống như vận động viên tiếp tục **luyện tập thêm với đúng bộ bài tập cũ** (cùng loại dữ liệu, cùng ý tưởng huấn luyện của HLV) để làm mượt kỹ thuật đã có — còn giai đoạn `N1.7` giống như vận động viên **chuyển sang một chương trình huấn luyện hoàn toàn khác** (xem hàng nghìn giờ video người khác thi đấu, thay vì chỉ tự tập lặp lại). Theo cách nhìn này, `N1 → N1.5` là "luyện tập thêm", còn `N1 → N1.7` là "đổi chương trình huấn luyện".

- **Đúng ở đâu:** truyền tải đúng ý "N1.5 vẫn dùng đúng loại dữ liệu (teleoperation robot thật) như N1, chỉ nhiều/kỹ hơn" — điều này khớp với những gì model card N1.5 mô tả (cải thiện connector + mục tiêu huấn luyện, không đổi loại dữ liệu).
- **Giới hạn:** loại suy này **không giải thích được N1.6** — vì N1.6 không chỉ "luyện tập thêm", nó còn đổi hẳn "dụng cụ tập luyện" (VLM backbone mới, DiT lớn gấp đôi) VÀ mở rộng sang loại robot/tình huống mới (bimanual arms, mobile manipulator, locomanipulation toàn thân) trước khi N1.7 mới thực sự đổi nguồn dữ liệu gốc (video người thay vì robot). Nói cách khác, ranh giới "luyện thêm kỹ năng cũ" vs "học kỹ năng mới" không nằm gọn ở chỗ `N1.6 → N1.7` như trực giác ban đầu gợi ý — nó đã bắt đầu dịch chuyển từ `N1.5 → N1.6`.

## 📐 Định nghĩa chính xác

**N1.5 và N1.6** là hai bản phát hành (release) checkpoint trung gian của họ mô hình GR00T, nằm giữa **N1 gốc** (paper chính thức, [arXiv:2503.14734](https://arxiv.org/abs/2503.14734), 3/2025 — xem bài giảng riêng về kiến trúc dual-system) và **N1.7** (blog chính thức HuggingFace, EgoScale + scaling law cho dexterity — xem bài giảng riêng). Điểm định nghĩa quan trọng nhất, khác biệt N1.5/N1.6 với hai mốc kia:

> ⚠️ **N1.5 và N1.6 không có bài báo khoa học (paper) riêng công bố trên arXiv hoặc bất kỳ hội nghị/tạp chí nào.** Nguồn thông tin duy nhất về chúng là: (1) model card trên HuggingFace (`nvidia/GR00T-N1.5-3B`, `nvidia/GR00T-N1.6-3B`), (2) GitHub release notes của repo `NVIDIA/Isaac-GR00T`, (3) một bài blog kỹ thuật của NVIDIA Developer về N1.6, và (4) trang tóm tắt nghiên cứu tại `research.nvidia.com/labs/gear/`. Đây là điểm khác biệt căn bản về **"độ tin cậy nguồn"** so với N1 (có paper, có bình duyệt cộng đồng rộng) và N1.7 (có blog chi tiết chính thức, dù cũng chưa có paper riêng).

Đây chính xác là điều mà tài liệu nội bộ gốc của dự án (`NOI-DUNG-CHI-TIET.md` mục 3.1) đã ghi nhận trước khi bài giảng này được viết:

> *"Tại thời điểm viết tài liệu này, N1.5 và N1.6 chưa có paper kỹ thuật riêng — thông tin chính thức duy nhất là changelog/model card trên HuggingFace (`nvidia/GR00T-N1.5-*`, `nvidia/GR00T-N1.6-*`) và tài liệu repo `GR00T-WholeBodyControl`. [...] N1.5 giữ cùng công thức kiến trúc với N1 gốc — VLM (Eagle-2) trích đặc trưng thị giác-ngôn ngữ làm điều kiện, DiT làm action head huấn luyện theo flow matching — với các cải tiến chủ yếu về dữ liệu/chất lượng huấn luyện hơn là thay đổi kiến trúc nền tảng."*
>
> *"Cần xác minh thêm: các chi tiết cụ thể về N1.5/N1.6 [...] dựa trên mô tả tổng quan từ nguồn thứ cấp [...], chưa được kiểm tra trực tiếp trong changelog/model card chính thức."*

Bài giảng này được viết đúng để lấp phần "cần xác minh thêm" đó — bằng cách trực tiếp tra cứu model card và blog chính thức (kết quả đầy đủ ở mục 🔥 Cập nhật hiện đại bên dưới). Kết luận ngắn gọn ngay tại đây: **giả định "N1.5 giữ cùng kiến trúc" hoá ra đúng phần lớn cho N1.5, nhưng KHÔNG còn đúng cho N1.6** — N1.6 có thay đổi kiến trúc được xác nhận chính thức, không chỉ là cải thiện dữ liệu.

## ⚙️ Cơ chế hoạt động — từng bước

Đây không phải một "cơ chế" theo nghĩa thuật toán (không có công thức toán riêng cho "N1.5→N1.6"), mà là một **trình tự phát triển phiên bản** — vẫn đáng để trình bày từng bước rõ ràng, ghi rõ ở mỗi mũi tên cái gì đã xác nhận và cái gì còn cần kiểm tra thêm:

```
┌──────────────────────────────────────────────────────────────────────────┐
│ N1 (gốc)                                                                  │
│  • Paper: arXiv:2503.14734 (nộp 3/2025)                                   │
│  • VLM: Eagle-2 (Li et al., 2025), chạy 10Hz                              │
│  • Action head: DiT flow-matching, chạy 120Hz                            │
│  • Code release trên GitHub: "n1-release" (theo GitHub Releases, ước     │
│    tính ~6/2025 — ⚠️ năm suy ra từ thứ tự phát hành, chưa tự tay xác      │
│    nhận timestamp chính xác)                                             │
└──────────────────────────────┬─────────────────────────────────────────┘
                                │  ĐÃ XÁC NHẬN (model card N1.5):
                                │  - MLP connector (nối đặc trưng VLM → DiT)
                                │    được sửa "để cải thiện hiệu năng trên sim
                                │    benchmark của NVIDIA"
                                │  - Huấn luyện đồng thời với 2 mục tiêu: flow
                                │    matching + "world-modeling objectives"
                                │  - Vision encoder mô tả trong card: SigLip2
                                │    (224×224) + text encoder T5
                                │  CẦN XÁC MINH THÊM:
                                │  - Card N1.5 không dùng lại nhãn "Eagle-2" —
                                │    chưa rõ đây là mô tả lại đúng thành phần
                                │    bên trong Eagle-2, hay là một thay đổi
                                │    backbone thực sự (xem mục Cập nhật hiện đại)
                                │  - Số liệu dữ liệu huấn luyện cụ thể: KHÔNG có
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ N1.5                                                                      │
│  • HF: nvidia/GR00T-N1.5-3B — GitHub release "n1.5-release" (~12/2025)   │
│  • KHÔNG có paper riêng                                                  │
└──────────────────────────────┬─────────────────────────────────────────┘
                                │  ĐÃ XÁC NHẬN (blog NVIDIA + research.nvidia.com):
                                │  - VLM đổi sang biến thể Cosmos-2B, hỗ trợ
                                │    độ phân giải/tỉ lệ khung hình linh hoạt
                                │    (khác SigLip2 cố định 224×224 ở N1.5)
                                │  - DiT tăng gấp đôi: 32 layer (so với 16 layer
                                │    ở N1.5)
                                │  - Bỏ tầng adapter transformer sau VLM; thay
                                │    vào đó "un-freeze" 4 layer trên cùng của
                                │    VLM khi pretrain
                                │  - Action chunk chuyển sang biểu diễn
                                │    "state-relative" (tương đối theo trạng thái
                                │    hiện tại) thay vì góc khớp tuyệt đối
                                │  - Dữ liệu: thêm hàng nghìn giờ teleoperation
                                │    mới — YAM (tay đôi), AGIBot Genie-1, mô
                                │    phỏng Galaxea R1 Pro (bộ BEHAVIOR), Unitree
                                │    G1 (locomanipulation toàn thân)
                                │  - Pretrain: 300K bước, batch size toàn cục
                                │    16384 (theo trang research.nvidia.com)
                                │  CẦN XÁC MINH THÊM:
                                │  - Con số benchmark định lượng cụ thể (N1.6
                                │    tốt hơn N1.5 bao nhiêu %) — KHÔNG công bố
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ N1.6                                                                      │
│  • HF: nvidia/GR00T-N1.6-3B — GitHub release "n1.6-release" (~4/2026)    │
│  • Blog kỹ thuật: NVIDIA Developer Blog "sim-to-real workflow"           │
│  • Bản vá n1.6.1 (~4/2026, vài ngày sau): chỉ dọn dẹp                    │
│    Dockerfile/pyproject.toml — KHÔNG phải thay đổi kỹ thuật              │
│  • KHÔNG có paper riêng                                                  │
└──────────────────────────────┬─────────────────────────────────────────┘
                                │  → xem bài giảng riêng
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ N1.7 — EgoScale (20,854 giờ video egocentric người) + scaling law cho    │
│ dexterity — nguồn: blog HuggingFace chính thức                          │
│ (chi tiết: BAI-GIANG-n17-egoscale-scaling-law.md)                        │
└──────────────────────────────────────────────────────────────────────────┘
```

Lưu ý quan trọng về ngày tháng trong sơ đồ trên: các mốc thời gian lấy từ tóm tắt trang GitHub Releases của `NVIDIA/Isaac-GR00T` — công cụ tra cứu chỉ trả về tháng/ngày, **năm được suy luận theo thứ tự phát hành hợp lý** (không tự tay đối chiếu từng timestamp gốc). Trước khi trích dẫn ngày tháng cụ thể ở nơi khác, nên tự mở trực tiếp `https://github.com/NVIDIA/Isaac-GR00T/releases` để xác nhận.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

Vì N1.5/N1.6 không có số liệu định lượng đầy đủ để "tính tay" theo nghĩa thông thường (không có công thức toán, không có bảng benchmark đầy đủ công khai), bài tập ở đây là một **walkthrough suy luận có cấu trúc**: với mỗi giả thuyết về "điều gì đã thay đổi", đánh giá đúng mức độ tin cậy dựa trên các nguồn đã tra cứu được cho bài giảng này.

| # | Giả thuyết | Mức độ tin cậy | Bằng chứng / nguồn |
|---|---|---|---|
| H1 | N1.5 chỉ cải thiện dữ liệu/chất lượng huấn luyện, không đổi kiến trúc nền tảng | **Đúng một phần, đã xác nhận có sửa nhỏ** | Model card N1.5 xác nhận rõ: MLP connector VLM→DiT "đã được sửa để cải thiện hiệu năng trên sim benchmark", cộng thêm mục tiêu huấn luyện "world-modeling" mới — đây LÀ một thay đổi kiến trúc, dù nhỏ, không phải "giữ nguyên 100%" như giả định gốc |
| H2 | N1.6 chỉ là "còn dựa trên vài nghìn giờ teleoperation robot thật" giống N1.5, chưa có gì đột phá | **Đúng phần dữ liệu, sai phần kiến trúc** | Đúng: N1.6 vẫn dùng teleoperation robot thật (chưa chuyển sang video người ở quy mô EgoScale). Sai: kiến trúc N1.6 đổi không nhỏ — DiT gấp đôi, VLM đổi, biểu diễn action đổi (xem H3-H5) |
| H3 | N1.6 tăng kích thước diffusion transformer (DiT) so với N1.5 | **Xác nhận chính thức** | NVIDIA Dev Blog + trang research.nvidia.com/labs/gear/gr00t-n1_6: "32 layers vs 16 layers in N1.5" |
| H4 | N1.6 đổi VLM backbone so với N1.5 | **Xác nhận chính thức** | NVIDIA Dev Blog: "a variant of Cosmos-Reason-2B VLM with native resolution support"; model card N1.5 mô tả SigLip2 (224×224 cố định) — hai mô tả này khác nhau rõ ràng |
| H5 | N1.6 dùng action chunk biểu diễn tương đối (relative) thay vì tuyệt đối | **Xác nhận chính thức** | Trang research.nvidia.com: "state-relative action predictions", nêu rõ đây là điểm khác so với N1.5 |
| H6 | N1.6 tốt hơn N1.5 một tỉ lệ phần trăm cụ thể trên benchmark | **Chưa xác nhận — không có số liệu công khai** | Cả 3 nguồn (model card, dev blog, research page) đều chỉ nói "outperforms N1.5" theo định tính, không kèm bảng số |
| H7 | N1.5 vẫn dùng đúng VLM "Eagle-2" như N1 gốc | **Không xác nhận được rõ ràng — mâu thuẫn tiềm ẩn** | Model card N1.5 mô tả thành phần là "SigLip2 vision transformer + T5 text encoder", không nhắc tên "Eagle-2" — có thể đây chỉ là cách mô tả chi tiết hơn các thành phần bên trong Eagle-2 (Eagle-2 cũng dùng vision tower dạng SigLIP), hoặc có thể là một thay đổi backbone thật sự chưa được công bố rõ ràng. **Cần đọc trực tiếp phần "Model Architecture" đầy đủ của cả hai card và, nếu cần độ chắc chắn cao, đối chiếu `config.json` của checkpoint** trước khi khẳng định theo hướng nào |

Điểm rút ra từ bảng: khi nguồn mỏng, cách làm việc đúng không phải là "chọn đại một câu trả lời nghe hợp lý", mà là liệt kê rõ các khả năng và gắn đúng nhãn tin cậy cho từng khả năng — đúng tinh thần "trung thực về khoảng trống thông tin" mà bài giảng này muốn truyền đạt.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | N1 (gốc) | N1.5 | N1.6 | N1.7 |
|---|---|---|---|---|
| Có paper chính thức? | Có — [arXiv:2503.14734](https://arxiv.org/abs/2503.14734) | Không | Không | Không (có blog chính thức chi tiết) |
| Nguồn thông tin chính | Paper arXiv | Model card HF + GitHub release | Model card HF + NVIDIA Dev Blog + research.nvidia.com | Blog HuggingFace |
| VLM backbone | Eagle-2 (10Hz) | SigLip2 + T5 (theo card — quan hệ với "Eagle-2" **cần xác minh thêm**, xem H7) | Biến thể Cosmos-Reason-2B, độ phân giải linh hoạt (xác nhận chính thức) | Cosmos-Reason2-2B (theo nguồn thứ cấp trong `NOI-DUNG-CHI-TIET.md`, **cần xác minh thêm**) |
| Action head | DiT flow-matching, 120Hz, action tuyệt đối | DiT flow-matching, connector đã sửa | DiT flow-matching, **32 layer** (gấp đôi N1.5), action **tương đối** (state-relative) | DiT flow-matching (số layer: nguồn thứ cấp không thống nhất, **cần xác minh thêm**) |
| Nguồn dữ liệu chính | Real-robot + human video + synthetic | Teleoperation robot thật (chưa rõ số giờ cụ thể) | Vài nghìn giờ teleoperation mới (YAM, AGIBot Genie-1, Galaxea R1 Pro sim, Unitree G1) — **vẫn là robot, chưa phải video người quy mô lớn** | **EgoScale — 20.854 giờ video egocentric người**, không cần thêm dữ liệu robot |
| Điểm nhấn | Kiến trúc dual-system (System 1/2) nền tảng | Cải thiện connector + mục tiêu huấn luyện world-model | Đổi VLM, tăng gấp đôi DiT, action tương đối, mở rộng embodiment (bimanual/mobile/locomanipulation) | Scaling law đầu tiên cho dexterity từ video người |
| Điểm số benchmark cụ thể công bố? | Có (trong paper) | Không tìm thấy | Không — chỉ nói định tính "outperforms N1.5" | Có (ví dụ "1k→20k giờ tăng hơn gấp đôi tỉ lệ hoàn thành tác vụ") |

**Điểm khác biệt cốt lõi rút ra:** nếu chỉ nhìn số phiên bản (`N1 → N1.5 → N1.6 → N1.7`), ta dễ tưởng tượng một đường thẳng đều dần tăng độ lớn thay đổi. Thực tế, từ nguồn đã xác nhận: bước nhảy kiến trúc lớn nhất về **DiT + VLM + biểu diễn action** xảy ra ở `N1.5 → N1.6` (đã xác nhận chính thức), còn bước nhảy lớn nhất về **loại nguồn dữ liệu** (robot → video người) xảy ra ở `N1.6 → N1.7`. Hai loại "bước nhảy lớn" này không trùng nhau ở cùng một mốc phiên bản.

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: N1.5/N1.6 chỉ là các bản vá nhỏ, không đáng học kỹ, có thể bỏ qua để đi thẳng từ N1 sang N1.7.**
   Vì sao sai: như bảng so sánh ở trên cho thấy, N1.6 có thay đổi kiến trúc được xác nhận chính thức không hề nhỏ — DiT tăng gấp đôi (16→32 layer), VLM backbone đổi, cách biểu diễn action đổi từ tuyệt đối sang tương đối. Nếu bạn tự fine-tune hoặc so sánh checkpoint N1.5 với N1.6, những khác biệt này ảnh hưởng trực tiếp tới cách bạn diễn giải kết quả (ví dụ: action tương đối cần xử lý khác action tuyệt đối khi ghép với bộ điều khiển hạ tầng như SONIC).
   Hiểu đúng: N1.5/N1.6 đáng học kỹ ngang các bước khác trong dòng phát triển — chỉ khác là nguồn tài liệu mỏng hơn, nên phải tự tra cứu nhiều hơn (model card, blog, GitHub release) thay vì có sẵn một paper tổng hợp.

2. **Hiểu nhầm: số phiên bản `x.5`/`x.6` tự động nghĩa là "nửa chặng đường" hoặc mức độ thay đổi tỉ lệ thuận với số thập phân (N1.6 "lớn hơn" N1.5 đúng bằng 0.1 đơn vị thay đổi).**
   Vì sao sai: cách đặt tên phiên bản của NVIDIA cho GR00T không tuân theo semantic versioning nghiêm ngặt của ngành phần mềm (nơi số sau dấu chấm có quy ước rõ ràng về mức độ thay đổi). Bằng chứng: `N1 → N1.5` chỉ sửa một connector nhỏ + mục tiêu huấn luyện phụ, trong khi `N1.5 → N1.6` đổi cả VLM, gấp đôi DiT, và đổi cách biểu diễn action — hai bước nhảy "0.5 đơn vị" này rõ ràng không cùng độ lớn thay đổi kỹ thuật.
   Hiểu đúng: luôn đọc trực tiếp model card/changelog để biết mức độ thay đổi thực tế, không suy luận từ tên số phiên bản.

3. **Hiểu nhầm: vì N1.5/N1.6 "chưa có paper", nên mọi thông tin về chúng đều là đồn đoán, không đáng tin.**
   Vì sao sai: đây là thái độ quá cực đoan theo hướng ngược lại. Model card HuggingFace chính thức và blog kỹ thuật NVIDIA Developer **là nguồn chính thức** (do chính NVIDIA công bố), chỉ là không có định dạng bình duyệt như paper — chúng vẫn đáng tin hơn hẳn "bài viết cộng đồng suy đoán". Sự phân biệt đúng không phải "có paper = tin, không có paper = không tin", mà là ba mức: xác nhận chính thức (model card/blog hãng) > nguồn thứ cấp (cộng đồng) > suy đoán chưa có nguồn.

## 🏗️ Ví dụ minh hoạ trong dự án này

Trong `06-vla-groot-sonic/README.md` mục A, mục 4 và mục D (lộ trình từng bước), N1.5/N1.6 được xếp là bước cải tiến trung gian trước khi học sâu N1.7 — bài giảng này chính là phần mở rộng chi tiết cho đúng mục 4 đó (README chỉ tóm tắt một câu, khuyến khích đọc changelog/model card, mà không có cấu trúc phân tích độ tin cậy từng tuyên bố). Khi bạn đi theo lộ trình D:

- Ở bước 2 (đọc N1 pass-3 đầy đủ), hãy ghi chú riêng phần VLM backbone (Eagle-2, mục 2.2 của `NOI-DUNG-CHI-TIET.md`) — đây là điểm neo để so sánh khi bạn đọc model card N1.5/N1.6 sau này (chính là bài tập thực hành số 1 bên dưới).
- Ở bước 3 (đọc blog N1.7), khi so sánh N1.7 với "N1 gốc", đừng bỏ qua rằng N1.7 thực chất kế thừa trực tiếp các thay đổi kiến trúc từ N1.6 (DiT lớn, VLM Cosmos họ) chứ không nhảy thẳng từ kiến trúc N1 gốc — hiểu đúng bậc thang N1→N1.5→N1.6→N1.7 giúp bạn đọc đúng phần "Related Work"/so sánh trong blog N1.7 thay vì tưởng N1.7 tự dưng đổi hết mọi thứ một lúc.
- Khi bạn tự chạy inference checkpoint (bước 4, dùng `NVIDIA/Isaac-GR00T`), nếu chọn thử cả checkpoint N1.5 và N1.6, hãy chú ý sự khác biệt biểu diễn action (tuyệt đối vs tương đối) — đây là chi tiết kỹ thuật thực sự ảnh hưởng tới cách bạn hậu xử lý output trước khi đưa vào SONIC (`01-whole-body-control/`).

## 🔥 Cập nhật hiện đại / SOTA gần đây

Đây là phần được tra cứu kỹ nhất cho bài giảng này — tối thiểu 5 lượt WebSearch/WebFetch nhắm trực tiếp vào model card HuggingFace, GitHub releases, và blog kỹ thuật chính thức. Kết quả **có nhiều thông tin xác thực hơn** những gì `NOI-DUNG-CHI-TIET.md` (soạn trước đó) có được — đây là một trường hợp hiếm trong dự án mà "cập nhật hiện đại" thực sự đẩy lùi được ranh giới "cần xác minh thêm" của tài liệu gốc, chứ không chỉ xác nhận lại.

1. **Model card `nvidia/GR00T-N1.5-3B`** ([HuggingFace](https://huggingface.co/nvidia/GR00T-N1.5-3B)) xác nhận chính thức: kiến trúc gồm SigLip2 vision transformer (khung hình 224×224) + T5 text encoder + MLP proprioception theo embodiment ID + DiT flow-matching action head; **"MLP connector giữa đặc trưng vision-language và DiT đã được sửa để cải thiện hiệu năng trên sim benchmark"**; huấn luyện đồng thời với mục tiêu flow matching **và** "world-modeling objectives" (mục tiêu phụ mới, không có ở N1 gốc theo mô tả trong `NOI-DUNG-CHI-TIET.md`). Đây là xác nhận chính thức đầu tiên có được cho câu hỏi "N1.5 đổi gì" — trước đây chỉ có phỏng đoán từ nguồn thứ cấp.

2. **Model card `nvidia/GR00T-N1.6-3B`** ([HuggingFace](https://huggingface.co/nvidia/GR00T-N1.6-3B)) và đặc biệt hai nguồn kỹ thuật chi tiết hơn — **NVIDIA Developer Blog** ["Building Generalist Humanoid Capabilities with NVIDIA Isaac GR00T N1.6 Using a Sim-to-Real Workflow"](https://developer.nvidia.com/blog/building-generalist-humanoid-capabilities-with-nvidia-isaac-gr00t-n1-6-using-a-sim-to-real-workflow/) và trang tóm tắt nghiên cứu [research.nvidia.com/labs/gear/gr00t-n1_6](https://research.nvidia.com/labs/gear/gr00t-n1_6/) — xác nhận rõ ràng, có thể trích dẫn:
   - VLM đổi sang **biến thể Cosmos-Reason-2B / "Cosmos-2B"** hỗ trợ độ phân giải và tỉ lệ khung hình linh hoạt ("native resolution support").
   - DiT tăng gấp đôi: **32 layer so với 16 layer ở N1.5**.
   - Bỏ tầng adapter transformer sau VLM, thay bằng un-freeze 4 layer trên cùng của VLM trong pretraining.
   - Chuyển sang **action chunk biểu diễn tương đối theo trạng thái (state-relative)** thay vì góc khớp tuyệt đối — theo blog, cải thiện độ mượt chuyển động và tốc độ hội tụ so với N1.5.
   - Dữ liệu mở rộng: thêm hàng nghìn giờ teleoperation mới từ tay đôi YAM, AGIBot Genie-1, mô phỏng Galaxea R1 Pro (bộ BEHAVIOR), Unitree G1 (locomanipulation toàn thân), cộng dữ liệu mô phỏng BEHAVIOR/RoboCasa.
   - Pretraining: 300.000 bước, batch size toàn cục 16.384 (theo trang research.nvidia.com).
   - Kết quả định tính: "outperforms N1.5 on both simulated manipulation benchmarks and on real bimanual robots including YAM, Agibot Genie-1, and Unitree G1" — **không kèm bảng số cụ thể** trong các nguồn đã tra cứu.
   - Ghi nhận thẳng thắn từ chính nguồn: "multi-task language following and out-of-distribution task generalization continue to be challenging" — tức NVIDIA tự thừa nhận N1.6 chưa giải quyết xong các bài toán khó này.

3. **GitHub Releases của `NVIDIA/Isaac-GR00T`** ([github.com/NVIDIA/Isaac-GR00T/releases](https://github.com/NVIDIA/Isaac-GR00T/releases)) xác nhận có tồn tại một bản vá **n1.6.1**, chỉ "loại bỏ các tham chiếu lỗi thời (dangling references) trong Dockerfile và pyproject.toml" — tức là một patch đóng gói/hạ tầng, **không phải thay đổi kỹ thuật về model**. Điểm này đáng chú ý: không phải mọi con số phiên bản đều tương ứng với thay đổi model — cần phân biệt patch hạ tầng với bản phát hành model thật.

4. **Điều CHƯA tìm được, dù đã tra cứu kỹ (giữ nguyên mức độ dè dặt):** (a) không tìm thấy bảng benchmark định lượng nào so sánh trực tiếp N1.5 vs N1.6 bằng số cụ thể (chỉ có mô tả định tính "outperforms"); (b) chưa xác nhận được rõ liệu VLM của N1.5 có thực sự là "Eagle-2" (như giả định trong `NOI-DUNG-CHI-TIET.md`) hay là một thành phần khác — model card N1.5 mô tả riêng lẻ "SigLip2 + T5" mà không nhắc tên thương hiệu "Eagle-2", và đây là điểm mâu thuẫn tiềm ẩn với giả định gốc mà bài giảng này **không tự ý giải quyết** — cần đọc trực tiếp phần kiến trúc đầy đủ của cả hai card, hoặc đối chiếu file `config.json` của từng checkpoint, để kết luận chắc chắn; (c) không tìm thấy số liệu chính xác "bao nhiêu giờ dữ liệu" cho riêng N1.5 (chỉ có mô tả định tính). Với các điểm (a)-(c), **đúng như hướng dẫn của bài giảng này: không bịa số, giữ nguyên trạng thái "cần xác minh thêm"**.

## ❓ Câu hỏi tự kiểm tra

1. Vì sao N1.5 và N1.6 không có paper kỹ thuật riêng, và điều đó có nghĩa gì khi bạn muốn trích dẫn một con số cụ thể về chúng trong một tài liệu khác?
<details><summary>Gợi ý đáp án</summary>NVIDIA phát hành N1.5/N1.6 nhanh hơn chu kỳ xuất bản paper bình duyệt — thông tin chỉ có qua model card HuggingFace, blog kỹ thuật, và GitHub release notes. Khi trích dẫn số liệu cụ thể, nên link trực tiếp tới model card/blog đó (không link tới "paper" vì không tồn tại), và luôn ghi rõ đây là nguồn hãng công bố chứ không phải công trình bình duyệt.</details>

2. Liệt kê 4 thay đổi kiến trúc/dữ liệu đã được **xác nhận chính thức** giữa N1.5 và N1.6 (không phải đoán).
<details><summary>Gợi ý đáp án</summary>(1) VLM đổi sang biến thể Cosmos-Reason-2B hỗ trợ độ phân giải linh hoạt; (2) DiT tăng gấp đôi (32 layer so với 16 layer); (3) bỏ tầng adapter sau VLM, un-freeze 4 layer trên của VLM; (4) action chunk chuyển từ tuyệt đối sang biểu diễn tương đối theo trạng thái (state-relative); có thể thêm (5) mở rộng dữ liệu sang các embodiment mới (YAM, AGIBot Genie-1, Galaxea R1 Pro sim).</details>

3. Tại sao gọi N1.5/N1.6 là "chỉ cải thiện dữ liệu, không đổi kiến trúc" là một giả định đã bị điều chỉnh sau khi tra cứu kỹ hơn?
<details><summary>Gợi ý đáp án</summary>Giả định đó gần đúng cho N1.5 (chỉ sửa một connector nhỏ + thêm mục tiêu huấn luyện phụ), nhưng sai đáng kể cho N1.6 — N1.6 đổi VLM backbone, tăng gấp đôi kích thước DiT, và đổi cách biểu diễn action, đều là thay đổi kiến trúc đã xác nhận chính thức, không chỉ là cải thiện dữ liệu huấn luyện.</details>

4. Cho một ví dụ về thông tin liên quan tới N1.5/N1.6 mà bài giảng này CHƯA thể xác nhận, dù đã tra cứu kỹ.
<details><summary>Gợi ý đáp án</summary>Ví dụ: bảng benchmark định lượng so sánh trực tiếp N1.5 vs N1.6 bằng số cụ thể (%, mm, hay bất kỳ đơn vị nào) — các nguồn chỉ nói định tính "N1.6 outperforms N1.5". Một ví dụ khác: liệu VLM của N1.5 có đúng là Eagle-2 hay là một thành phần khác — model card mô tả SigLip2+T5 mà không dùng tên "Eagle-2".</details>

5. Bản vá "n1.6.1" trên GitHub thay đổi gì? Vì sao nó không nên bị nhầm là "một phiên bản model mới có cải tiến kỹ thuật"?
<details><summary>Gợi ý đáp án</summary>n1.6.1 chỉ loại bỏ các tham chiếu lỗi thời trong Dockerfile và pyproject.toml — đây là một bản vá đóng gói/hạ tầng (packaging), không đổi trọng số model hay kiến trúc. Không nên nhầm mọi số phiên bản (kể cả số sau hai dấu chấm) với một thay đổi kỹ thuật về model.</details>

6. Nếu bạn đọc được một bài viết cộng đồng (blog, forum) khẳng định một con số cụ thể về N1.5/N1.6 mà bài giảng này không có, bạn nên làm gì trước khi tin và trích dẫn lại?
<details><summary>Gợi ý đáp án</summary>Kiểm tra xem bài viết đó dẫn nguồn từ đâu — nếu chỉ là suy đoán hoặc dẫn lại nguồn thứ cấp khác (không phải model card/blog/GitHub chính thức của NVIDIA), nên coi là "nguồn thứ cấp, chưa xác nhận" và tự tra lại model card HuggingFace hoặc GitHub releases tương ứng trước khi trích dẫn con số đó ở nơi khác.</details>

## 📝 Bài tập thực hành

1. **Tự tra model card HuggingFace và đối chiếu:** mở trực tiếp `https://huggingface.co/nvidia/GR00T-N1.5-3B` và `https://huggingface.co/nvidia/GR00T-N1.6-3B`, đọc kỹ mục "Model Architecture"/"Training Data" của cả hai. Ghi lại **3 điều xác nhận được** mà bài giảng này chưa nhắc tới (nếu model card đã được cập nhật thêm kể từ khi bài giảng này được viết), và **1 điều bạn thấy mâu thuẫn hoặc chưa rõ ràng** giữa hai card — đặc biệt kiểm tra lại câu hỏi H7 (quan hệ giữa VLM của N1.5 và "Eagle-2").
2. **Đối chiếu GitHub release notes:** mở `https://github.com/NVIDIA/Isaac-GR00T/releases`, tìm đúng 2 mục "n1.5-release" và "n1.6-release" (và "n1.6.1-release" nếu có), đọc toàn bộ nội dung release note (không chỉ tiêu đề). Viết lại thành bảng 2 cột "Đã xác nhận trong bài giảng này" / "Mới phát hiện thêm" — nếu bảng "Mới phát hiện thêm" có mục nào, đó chính là dấu hiệu bài giảng này cần được cập nhật lại.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

N1.5 và N1.6 là hai bản phát hành trung gian của GR00T nằm giữa N1 gốc (có paper, kiến trúc dual-system nền tảng) và N1.7 (có blog chi tiết, bước nhảy EgoScale) — nhưng khác biệt căn bản của chúng là **không có paper kỹ thuật riêng**, chỉ có model card HuggingFace và GitHub release notes làm nguồn chính thức duy nhất. Tra cứu trực tiếp cho bài giảng này xác nhận: N1.5 chủ yếu sửa nhỏ (connector VLM→DiT, thêm mục tiêu huấn luyện world-model), nhưng N1.6 thực ra có thay đổi kiến trúc không nhỏ — DiT tăng gấp đôi (32 so với 16 layer), VLM đổi sang biến thể Cosmos-Reason-2B, và action chunk chuyển sang biểu diễn tương đối — điều này **điều chỉnh lại** giả định ban đầu rằng "N1.5/N1.6 chỉ là cải thiện dữ liệu, không đổi kiến trúc nền tảng". Dù vậy, một số câu hỏi quan trọng vẫn còn để ngỏ dù đã tra cứu kỹ — không có bảng benchmark định lượng N1.5 vs N1.6 công khai, và quan hệ chính xác giữa VLM của N1.5 với "Eagle-2" của N1 gốc chưa được xác nhận rõ ràng. Đây vẫn là mắt xích còn thiếu tài liệu chính thức nhất trong toàn bộ dòng phát triển GR00T tính đến thời điểm soạn bài giảng này, và bài học quan trọng nhất không phải là một con số cụ thể, mà là kỹ năng phân biệt rạch ròi giữa "đã xác nhận chính thức", "nguồn thứ cấp", và "suy đoán" khi làm việc với một hệ thống đang phát triển nhanh hơn chu kỳ xuất bản khoa học truyền thống.
