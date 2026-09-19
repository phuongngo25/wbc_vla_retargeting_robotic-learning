# Bài giảng: BONES-SEED

*(Thuộc mảng: Human Motion Datasets)*

> **Ghi chú thận trọng bắt buộc:** đây là dataset công bố **gần đây (2026)**, thông tin công khai còn hạn chế tại thời điểm viết bài. Bài giảng này chỉ ghi lại đúng những gì xác minh được qua trang phát hành chính thức (Hugging Face) và các repo/dataset liên quan trực tiếp của NVIDIA (dự án Kimodo) — không suy diễn thêm chi tiết không có nguồn.

## 🎯 Mục tiêu bài học

- Nêu chính xác ai công bố BONES-SEED, vai trò thực sự của NVIDIA trong dataset này (công cụ, không phải chủ sở hữu dữ liệu gốc).
- Nhớ được đúng 4 con số quy mô: 142.220 chuyển động, ~288 giờ, 522 diễn viên, 3 định dạng xuất.
- Giải thích được vì sao việc dataset đã có sẵn bản retarget G1 (CSV) là một lợi thế quan trọng, liên hệ trực tiếp tới bài giảng SOMA-retargeter đã học.
- Tính tay được một ví dụ so sánh chi phí tính toán/thời gian nếu phải tự retarget BONES-SEED từ đầu bằng GMR (CPU, streaming) so với việc đã có sẵn bản retarget.
- Phân biệt được BONES-SEED với AMASS/LAFAN1/OMOMO theo đúng các tiêu chí đã học ở các bài trước (định dạng, quy mô, loại chuyển động, annotation).
- Nêu được ít nhất 1 hạn chế/rủi ro cụ thể khi dùng một dataset mới công bố, ít được kiểm chứng độc lập.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Ba dataset trước (AMASS, OMOMO, LAFAN1) đều đã tồn tại nhiều năm, được cộng đồng học thuật kiểm chứng rộng rãi. BONES-SEED khác hẳn: quy mô lớn nhất trong nhóm (~288 giờ, gấp hơn 7 lần LAFAN1 và gần 7 lần AMASS), nhưng mới công bố năm 2026 và **đã có sẵn bản retarget sang Unitree G1** — nghĩa là với dataset này, một phần công việc mà toàn bộ mảng `02-motion-retargeting/` mô tả (GMR/SOMA-retargeter) **đã được thực hiện sẵn ở quy mô lớn**. Học bài này để hiểu chính xác dataset này cung cấp gì, giới hạn thông tin công khai hiện tại ra sao, và vì sao việc "đã retarget sẵn" lại quan trọng tới mức được nhấn mạnh riêng thành một khái niệm.

## 🧠 Trực giác

### Góc nhìn 1: Mua đồ ăn đã chế biến sẵn (meal kit đã nấu chín) so với mua nguyên liệu thô

AMASS/LAFAN1/OMOMO giống việc mua **nguyên liệu thô** (rau, thịt) về nhà tự nấu (tự chạy GMR/SOMA-retargeter để retarget sang robot cụ thể). BONES-SEED, với việc cung cấp sẵn định dạng "Unitree G1 (CSV)", giống một **suất ăn đã nấu chín sẵn** — bạn có thể dùng ngay mà không cần tự thực hiện bước retargeting, tiết kiệm thời gian đáng kể khi cần khối lượng lớn (142.220 "suất ăn").

**Giới hạn của loại suy này:** đồ ăn nấu sẵn thường không thể "nấu lại theo khẩu vị riêng"; trong khi dữ liệu SOMA (BVH) vẫn được cung cấp song song với bản G1 đã retarget trong BONES-SEED — nghĩa là bạn *vẫn có thể* tự retarget lại theo cấu hình robot khác nếu muốn (dùng đúng SOMA-retargeter đã học), khác với đồ ăn nấu sẵn không thể "undo" để lấy lại nguyên liệu thô.

### Góc nhìn 2: Một cửa hàng bán lại hàng của nhiều nhà sản xuất, dán thêm nhãn phụ

BONES-SEED giống một cửa hàng (Bones Studio) bán một sản phẩm (dữ liệu mocap gốc do chính họ thu thập), nhưng có một đối tác khác (NVIDIA) đóng góp thêm dịch vụ gia công (retarget bằng SOMA-retargeter, gắn nhãn phân đoạn thời gian cho dự án Kimodo) rồi dán thêm nhãn phụ đó lên sản phẩm trước khi bán ra. Người mua cần phân biệt rõ: ai là **nhà sản xuất gốc** (Bones Studio — dữ liệu mocap) và ai là **đơn vị gia công thêm giá trị** (NVIDIA — công cụ retarget + annotation), vì trách nhiệm/license của mỗi phần có thể khác nhau.

**Giới hạn của loại suy này:** trong thương mại, "cửa hàng bán lại" thường không phải bên tạo giá trị chính; ở đây NVIDIA tuy chỉ đóng góp công cụ nhưng phần đóng góp đó (bản retarget G1 sẵn có, annotation ngôn ngữ) lại là phần **quyết định trực tiếp** dataset này có hữu ích ngay lập tức cho pipeline VLA/RL hay không — vai trò "công cụ" ở đây quan trọng hơn nhiều so với vai trò của một cửa hàng bán lẻ thông thường.

## 📐 Định nghĩa chính xác

**BONES-SEED** là dataset chuyển động người quy mô lớn do **Bones Studio** phát hành (không phải nội bộ NVIDIA), host công khai trên Hugging Face tại `bones-studio/seed` (gated — cần chấp nhận điều khoản license `bones-seed-license`, liên hệ `licensing@bones.studio` để biết chi tiết truy cập, hiện dành cho tổ chức phi lợi nhuận qua Bones Research Network).

**Vai trò của NVIDIA trong dataset này:** NVIDIA đóng góp **công cụ**, không phải chủ sở hữu dữ liệu gốc:
- Retarget chuyển động gốc sang **SOMA-skeleton** và **Unitree G1**, dùng chính công cụ mã nguồn mở `NVIDIA/soma-retargeter` (đã học ở `02-motion-retargeting/`).
- Tạo **temporal segmentation** (chia mỗi chuyển động thành các đoạn/pha có ý nghĩa kèm mốc thời gian) "cho dự án Kimodo" của NVIDIA — Kimodo là mô hình diffusion sinh chuyển động động học (kinematic motion diffusion model) của NVIDIA (`nv-tlabs/kimodo` trên GitHub), có nhiều biến thể huấn luyện trên dữ liệu liên quan tới BONES-SEED (`nvidia/Kimodo-G1-SEED-v1`, `nvidia/Kimodo-SOMA-RP-v1`, `nvidia/Kimodo-SMPLX-RP-v1`...).

**Số liệu quy mô** (theo trang phát hành chính thức trên Hugging Face):

```text
142.220 chuyển động  =  71.132 chuyển động gốc  +  71.088 bản mirror/lật gương
~288 giờ dữ liệu (tính ở 120 fps)
522 diễn viên (253 nữ / 269 nam), độ tuổi 17–71
3 định dạng xuất: SOMA Uniform (BVH) | SOMA Proportional (BVH) | Unitree G1 (CSV, tương thích MuJoCo)
Annotation: tới 6 mô tả ngôn ngữ tự nhiên/chuyển động + temporal segmentation
```

**Không có bằng chứng công khai** cho thấy BONES-SEED trực tiếp gắn với GR00T hay SONIC — mối liên hệ với pipeline WBC/VLA cụ thể của dự án này (nếu có) cần xác minh thêm khi tới giai đoạn huấn luyện thực tế, không nên mặc định.

## ⚙️ Cơ chế hoạt động — từng bước

Sơ đồ dưới đây mô tả *quy trình tạo ra* BONES-SEED (dựa trên các thông tin đã xác minh), không phải một thuật toán người dùng tự chạy:

```text
┌────────────────────────────────────────────────────────────┐
│ Bones Studio: thu thập mocap gốc                             │
│  522 diễn viên (253 nữ/269 nam, 17–71 tuổi)                  │
│  71.132 chuyển động gốc                                       │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ Data augmentation: mirror/lật gương                           │
│  71.132 gốc + 71.088 mirror = 142.220 tổng                   │
│  (lật trái-phải một chuyển động → mẫu huấn luyện hợp lệ mới) │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ Chuẩn hoá & xuất SOMA-skeleton (Bones Studio)                 │
│  → SOMA Uniform (BVH) + SOMA Proportional (BVH)               │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ NVIDIA: retarget bằng SOMA-retargeter (đã học)                │
│  → Unitree G1 (CSV, tương thích MuJoCo)                       │
│  + temporal segmentation cho dự án Kimodo                     │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ Annotation ngôn ngữ tự nhiên (tới 6 mô tả/chuyển động)        │
└──────────────────────────────────────────────────────────────┘
                            ▼
                  Xuất bản: bones-studio/seed (Hugging Face, gated)
```

Điểm mấu chốt: bước "NVIDIA retarget bằng SOMA-retargeter" trong sơ đồ trên chính là ứng dụng thực tế, ở quy mô 142.220 chuyển động, của đúng pipeline 5 bước đã học chi tiết ở [`BAI-GIANG-soma-retargeter-kien-truc-pipeline.md`](../02-motion-retargeting/BAI-GIANG-soma-retargeter-kien-truc-pipeline.md) — đây là bằng chứng thực tế cho câu hỏi tự kiểm tra #5 của bài đó ("SEED xác nhận vai trò thực tế của SOMA-retargeter").

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số tự chọn — tái sử dụng đúng phương pháp tính đã học ở bài SOMA-retargeter, áp dụng cho quy mô thật của BONES-SEED)*

Giả sử một người muốn **tự retarget lại** toàn bộ 142.220 chuyển động BONES-SEED sang G1 bằng GMR (thay vì dùng bản CSV đã có sẵn) — dù đây không phải cách dùng khuyến nghị (dataset đã cho sẵn bản retarget), tính thử chi phí để thấy giá trị của việc "có sẵn":

Giả sử trung bình mỗi chuyển động dài **288 giờ / 142.220 chuyển động ≈ 7.29 giây** (giả định đơn giản hoá: chia đều tổng thời lượng cho tổng số chuyển động), ở 120fps:

```text
số frame trung bình mỗi chuyển động = 7.29 × 120 ≈ 875 frame
tổng số frame toàn dataset = 142.220 × 875 ≈ 124.442.500 frame  (~124.4 triệu frame)
```

**Nếu retarget bằng GMR ở tốc độ trung bình 50 FPS (CPU, tuần tự — dải 35-70 FPS đã học):**

```text
thời gian = 124.442.500 / 50 = 2.488.850 giây
          ≈ 41.481 phút
          ≈ 691.4 giờ
          ≈ 28.8 ngày chạy liên tục 24/24
```

**So sánh:** việc BONES-SEED đã cung cấp sẵn bản retarget G1 tiết kiệm trực tiếp khoảng **691 giờ tính toán** (gần 29 ngày máy chạy liên tục) so với việc tự retarget lại từ đầu bằng GMR — đây là con số cụ thể hoá cho nhận định ở mục Định nghĩa: "việc đã có sẵn bản retarget G1 ở quy mô hàng trăm nghìn chuyển động minh hoạ đúng luận điểm retargeting phải được làm trước, ở quy mô lớn, bằng pipeline tự động". *(Lưu ý: đây là ước tính minh hoạ dựa trên giả định thời lượng trung bình đơn giản hoá — thời lượng thực tế từng chuyển động trong BONES-SEED có thể phân bố không đều.)*

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | AMASS | LAFAN1 | OMOMO | **BONES-SEED** |
|---|---|---|---|---|
| Định dạng | SMPL/SMPL-X | BVH | SMPL-H/SMPL-X + object | SOMA (BVH) + **G1 CSV đã retarget sẵn** |
| Quy mô | >40 giờ, >11.000 motion | 496.672 khung (~4.6 giờ) | ~10 giờ | **~288 giờ, 142.220 chuyển động** |
| Số diễn viên | >300 (15 bộ mocap hợp nhất) | 5 | không nêu cụ thể trong nguồn đã học | **522** (253 nữ/269 nam) |
| Annotation ngôn ngữ | Không | Không | Không | **Có, tới 6 mô tả/chuyển động** |
| Đã retarget sẵn sang robot? | Không | Không | Không | **Có — Unitree G1 (CSV)** |
| Temporal segmentation | Không | Không | Không | **Có** (cho dự án Kimodo) |
| Độ trưởng thành/kiểm chứng | Cao (2019, nhiều năm sử dụng học thuật) | Cao (2020, chuẩn game AAA) | Trung bình (2023) | **Thấp — mới 2026, ít paper độc lập kiểm chứng** |
| Access | Gated (đăng ký) | Mở (GitHub) | Mở (Google Drive) | **Gated** (license `bones-seed-license`, phi lợi nhuận) |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "BONES-SEED là dataset nội bộ của NVIDIA, gắn liền với GR00T/SONIC".** Vì sao sai: nguồn chính thức xác nhận rõ ràng đơn vị phát hành là **Bones Studio**, không phải NVIDIA — NVIDIA chỉ đóng góp công cụ (SOMA-retargeter để tạo bản G1, temporal segmentation cho dự án Kimodo riêng của họ). Không có bằng chứng công khai liên kết trực tiếp dataset này với GR00T hoặc SONIC. **Hiểu đúng:** cần phân biệt rõ "ai sở hữu dữ liệu gốc" và "ai đóng góp công cụ xử lý", tránh suy diễn mối liên hệ với các sản phẩm khác của cùng một công ty (NVIDIA) khi không có bằng chứng trực tiếp.
2. **Hiểu nhầm: "vì BONES-SEED lớn nhất và mới nhất, nên luôn ưu tiên dùng nó thay AMASS/LAFAN1".** Vì sao sai: dataset mới công bố (2026) có **ít paper/tài liệu học thuật kiểm chứng độc lập** hơn nhiều so với AMASS (từ 2019, hàng trăm paper trích dẫn) — quy mô lớn không tự động đồng nghĩa với độ tin cậy đã được kiểm chứng rộng rãi về chất lượng mocap, độ chính xác retarget, hay tính đại diện của annotation ngôn ngữ. **Hiểu đúng:** với các nghiên cứu cần độ tin cậy đã được cộng đồng xác nhận (ví dụ benchmark so sánh phương pháp), AMASS/LAFAN1 vẫn có giá trị riêng; BONES-SEED phù hợp hơn khi cần **quy mô lớn + đã retarget sẵn + có ngôn ngữ** cho huấn luyện VLA, chấp nhận đánh đổi độ kiểm chứng thấp hơn.

## 🏗️ Ví dụ minh hoạ trong dự án này

BONES-SEED là nguồn dữ liệu phù hợp nhất trong 4 dataset đã học cho giai đoạn huấn luyện VLA (`06-vla-groot-sonic/`), vì đã kết hợp đúng 3 yếu tố cần thiết cùng lúc mà không dataset nào khác trong nhóm có đủ:

```text
Quy mô lớn (~288 giờ)          →  đủ dữ liệu cho huấn luyện policy quy mô lớn
      +
Đã retarget sẵn sang G1 (CSV)  →  bỏ qua bước retargeting thủ công/tính toán lại
      +
Annotation ngôn ngữ tự nhiên   →  huấn luyện trực tiếp policy language-conditioned
                                    (liên hệ 06-vla-groot-sonic/)
```

So với việc phải tự lấy AMASS (không có ngôn ngữ, chưa retarget) rồi tự chạy GMR/SOMA-retargeter (tốn thời gian tính toán như ví dụ trên) rồi tự gắn annotation ngôn ngữ (công việc riêng, tốn kém), BONES-SEED cung cấp một "đường tắt" hợp lệ cho pipeline dữ liệu VLA — với điều kiện chấp nhận rủi ro về độ kiểm chứng thấp hơn (mục Sai lầm thường gặp #2) và các ràng buộc license gated.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **Dự án Kimodo (NVIDIA) là bằng chứng ứng dụng thực tế trực tiếp nhất của BONES-SEED tại thời điểm tra cứu.** Kimodo là "kinematic motion diffusion model" mã nguồn mở của NVIDIA ([GitHub: nv-tlabs/kimodo](https://github.com/nv-tlabs/kimodo)) — mô hình diffusion sinh chuyển động động học (không phải chỉ retargeting) có nhiều biến thể huấn luyện trực tiếp trên dữ liệu liên quan BONES-SEED: **`Kimodo-G1-SEED-v1`** (mô hình diffusion 282 triệu tham số, nhận đầu vào text + duration + pose constraint, huấn luyện trên biến thể G1 của SEED), cùng các biến thể `Kimodo-SOMA-RP-v1(.1)`, `Kimodo-SMPLX-RP-v1`. NVIDIA cũng công bố **NVIDIA Kimodo Motion Generation Benchmark** trên Hugging Face, dùng chính BONES-SEED và "BONES RP01" làm ground truth để đánh giá các mô hình sinh chuyển động trong hệ sinh thái mở — xác nhận BONES-SEED không chỉ là dữ liệu tĩnh mà đã trở thành **nền tảng benchmark** cho một dòng nghiên cứu sinh chuyển động cụ thể.
2. **Cấu trúc train/test 90%/10% đã được thiết lập cho biến thể SOMA của SEED**, theo thông tin liên quan tới Kimodo — cho thấy dataset đã được tổ chức sẵn theo quy ước phù hợp cho việc huấn luyện/đánh giá mô hình học máy, không chỉ là một kho lưu trữ chuyển động thô.
3. **Thận trọng cần thiết:** vì cả BONES-SEED lẫn dòng mô hình Kimodo đều là công bố rất gần đây (2026), số lượng nghiên cứu độc lập (ngoài chính NVIDIA/Bones Studio) sử dụng và đánh giá lại dataset này còn hạn chế tại thời điểm viết bài — nên xử lý các con số/tuyên bố về chất lượng dataset với mức độ tin cậy tương xứng với một nguồn mới, đúng tinh thần đã áp dụng nhất quán ở bài giảng UMR (`02-motion-retargeting/`).

## ❓ Câu hỏi tự kiểm tra

1. Ai là đơn vị phát hành BONES-SEED, và NVIDIA đóng vai trò gì trong dataset này?
   <details><summary>Gợi ý đáp án</summary>Bones Studio phát hành dataset gốc; NVIDIA chỉ đóng góp công cụ — retarget bằng SOMA-retargeter sang G1, và temporal segmentation cho dự án Kimodo riêng của họ — không phải chủ sở hữu dữ liệu gốc.</details>
2. Vì sao 142.220 chuyển động không đồng nghĩa với 142.220 lần thu thập mocap độc lập?
   <details><summary>Gợi ý đáp án</summary>Vì con số này gồm 71.132 chuyển động gốc + 71.088 bản mirror/lật gương (kỹ thuật tăng cường dữ liệu) — chỉ khoảng một nửa là dữ liệu mocap thực sự thu thập mới.</details>
3. Trong ví dụ tính tay, nếu GMR chạy nhanh hơn (giả sử 70 FPS thay vì 50 FPS), thời gian tự retarget lại toàn bộ dataset thay đổi thế nào (tính theo giờ)?
   <details><summary>Gợi ý đáp án</summary>`124.442.500 / 70 ≈ 1.777.750 giây ≈ 493.8 giờ` (≈20.6 ngày) — giảm so với 691.4 giờ ở 50 FPS, nhưng vẫn là một khối lượng tính toán rất lớn so với việc dùng bản CSV đã có sẵn (0 giờ tính toán thêm).</details>
4. Vì sao không nên mặc định BONES-SEED có liên hệ trực tiếp với GR00T/SONIC dù cùng thuộc hệ sinh thái NVIDIA?
   <details><summary>Gợi ý đáp án</summary>Vì nguồn chính thức không có bằng chứng công khai xác nhận mối liên hệ này — NVIDIA chỉ đóng góp công cụ retarget/annotation cho dự án Kimodo (một mô hình diffusion sinh chuyển động), không phải cho GR00T/SONIC; suy diễn liên hệ khi không có bằng chứng là vi phạm nguyên tắc không bịa đặt sự thật kỹ thuật.</details>
5. So với AMASS, điều gì khiến BONES-SEED phù hợp hơn cho huấn luyện policy language-conditioned trong `06-vla-groot-sonic/`?
   <details><summary>Gợi ý đáp án</summary>BONES-SEED có annotation ngôn ngữ tự nhiên (tới 6 mô tả/chuyển động) và temporal segmentation đi kèm sẵn, trong khi AMASS không có annotation ngôn ngữ nào — một yếu tố thiết yếu để huấn luyện trực tiếp policy điều kiện hoá theo ngôn ngữ.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** giả sử một nhóm nghiên cứu khác chỉ có 50.000 chuyển động (không phải toàn bộ 142.220) với cùng tỷ lệ thời lượng trung bình mỗi chuyển động (~7.29 giây ở 120fps), muốn tự retarget bằng SOMA-retargeter (GPU batch, giả định batch size 512, mỗi lượt batch mất 0.05s — dùng đúng công thức đã học ở bài SOMA-retargeter). Tính tổng số frame, số lượt batch, và tổng thời gian xử lý.
2. **Đọc nguồn thật, phân biệt cẩn trọng:** mở trực tiếp trang [huggingface.co/datasets/bones-studio/seed](https://huggingface.co/datasets/bones-studio/seed) và trang [huggingface.co/nvidia/Kimodo-G1-SEED-v1](https://huggingface.co/nvidia/Kimodo-G1-SEED-v1). Lập một bảng ngắn phân biệt rõ: (a) thông tin nào thuộc về dataset gốc BONES-SEED (Bones Studio sở hữu), (b) thông tin nào thuộc về mô hình Kimodo-G1-SEED (NVIDIA huấn luyện, dùng dữ liệu liên quan) — đây là bài tập rèn thói quen không nhầm lẫn "dữ liệu" với "mô hình được huấn luyện trên dữ liệu đó".

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

BONES-SEED là dataset chuyển động người quy mô lớn nhất trong nhóm 4 dataset đã học (~288 giờ, 142.220 chuyển động từ 522 diễn viên, gồm cả bản gốc và mirror), do Bones Studio phát hành và host gated trên Hugging Face, với đóng góp công cụ quan trọng từ NVIDIA: retarget sẵn sang Unitree G1 (CSV) bằng chính SOMA-retargeter đã học, cùng temporal segmentation cho dự án Kimodo — một mô hình diffusion sinh chuyển động động học có nhiều biến thể huấn luyện trên dữ liệu liên quan (Kimodo-G1-SEED, Kimodo-SOMA-RP...). Việc đã có sẵn bản retarget G1 tiết kiệm một khối lượng tính toán khổng lồ (ước tính minh hoạ ~691 giờ nếu tự retarget lại bằng GMR ở 50 FPS) và kết hợp với annotation ngôn ngữ tự nhiên khiến dataset này đặc biệt phù hợp cho huấn luyện VLA language-conditioned; tuy nhiên, vì mới công bố năm 2026, dataset còn thiếu độ kiểm chứng độc lập rộng rãi so với AMASS/LAFAN1, và cần phân biệt rõ vai trò "công cụ" của NVIDIA với vai trò "chủ sở hữu dữ liệu gốc" của Bones Studio khi trích dẫn hoặc sử dụng.
