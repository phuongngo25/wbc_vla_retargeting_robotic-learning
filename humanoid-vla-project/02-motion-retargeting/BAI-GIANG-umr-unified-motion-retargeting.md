# Bài giảng: UMR — Unified Motion Retargeting qua Learned Point Cloud Correspondence

*(Thuộc mảng: Motion Retargeting)*

> **Ghi chú phạm vi:** khái niệm này **không nằm trong danh sách 10 khái niệm gốc** của `NOI-DUNG-CHI-TIET.md`/Phụ lục `PROMPT-TAO-BAI-GIANG.md` — đây là bài giảng bổ sung, được thêm theo yêu cầu nghiên cứu thêm ngoài GMR/SOMA-retargeter, dựa hoàn toàn trên tra cứu WebSearch/WebFetch thật (không có "nguồn cô đọng" nội bộ để đối chiếu). Mọi tuyên bố kỹ thuật dưới đây đều trích dẫn trực tiếp từ paper/trang chính thức.

## 🎯 Mục tiêu bài học

- Giải thích được vì sao "correspondence" (sparse, handcrafted) là điểm nghẽn chung của cả GMR và SOMA-retargeter, và UMR nhắm giải quyết đúng điểm nghẽn đó.
- Mô tả được hai giai đoạn của UMR: Point Cloud Correspondence Learning và Correspondence-Guided Retargeting.
- Phân biệt được "sparse keypoint correspondence" (GMR/SOMA) với "dense point-cloud correspondence" (UMR) bằng một ví dụ số minh hoạ độ phân giải thông tin khác nhau.
- Nêu được UMR chuyển giao contact (tương tác tay/chân-vật thể) khác gì so với foot contact stabilization đã học.
- Nêu rõ ràng những gì đã kiểm chứng được từ nguồn chính thức và những gì CHƯA kiểm chứng được (ví dụ số liệu so sánh định lượng với GMR/OmniRetarget) — tránh suy diễn quá mức từ các tóm tắt gián tiếp.
- Định vị được UMR trong bức tranh chung: một hướng nghiên cứu 2026 mới, chưa phải công cụ chính của dự án (vẫn là GMR/SOMA-retargeter).

## 🧭 Vì sao cần học cái này? (bối cảnh)

Ba bài giảng trước (GMR, SOMA-retargeter, Villegas residual) đều có một điểm chung đã được chính các bài đó chỉ ra ở phần "Cập nhật hiện đại": correspondence — bảng ánh xạ khớp người ↔ khớp robot — trong cả GMR lẫn SOMA-retargeter đều là **sparse và thiết kế thủ công** (`ik_match_table`, config theo từng robot). Bài Skeleton mapping đã nêu đúng câu hỏi này ở cuối bài: "UMR muốn khắc phục hạn chế nào của sparse mapping?" — bài giảng này trả lời câu hỏi đó một cách đầy đủ, độc lập, thay vì chỉ nhắc thoáng qua như các bài trước. Học bài này để hiểu: correspondence tự động, học từ dữ liệu, khác về chất (không chỉ về lượng) với correspondence sparse thủ công như thế nào, và giới hạn hiện tại của cách tiếp cận mới này là gì.

## 🧠 Trực giác

### Góc nhìn 1: Đo may bằng vài số đo cố định so với quét 3D toàn thân

Skeleton mapping kiểu GMR/SOMA giống một thợ may đo vài số đo chuẩn (vòng ngực, chiều dài tay, chiều dài chân) rồi suy ra toàn bộ mẫu áo — nhanh, dễ hiểu, nhưng bỏ qua chi tiết hình dáng cụ thể giữa các điểm đo (độ cong vai, độ dày cơ bắp). UMR giống việc quét 3D toàn bộ bề mặt cơ thể khách hàng và bề mặt "hình nhân" robot, rồi tìm tương ứng **giữa hàng nghìn điểm bề mặt** thay vì chỉ vài chục điểm mốc khớp — cho phép may một bộ đồ ôm sát chi tiết hơn nhiều so với chỉ dùng vài số đo.

**Giới hạn của loại suy này:** quét 3D một lần là đủ cho một bộ đồ tĩnh; nhưng chuyển động là động (thay đổi theo thời gian) — UMR phải học correspondence này từ dữ liệu huấn luyện đa dạng (nhiều tư thế, nhiều robot) chứ không chỉ "quét" một lần rồi dùng mãi, khác với việc đo may vật lý chỉ cần thực hiện một lần cho một khách hàng cụ thể.

### Góc nhìn 2: Bản đồ địa hình chi tiết (contour map) so với vài mốc GPS

Sparse mapping giống việc chỉ biết toạ độ GPS của vài đỉnh núi quan trọng trên một dãy núi; dense point-cloud correspondence giống có cả bản đồ địa hình chi tiết (đường đồng mức) của toàn bộ dãy núi. Với vài mốc GPS, bạn biết đỉnh A cao 2000m, đỉnh B cao 1800m, nhưng không biết hình dạng sườn núi nối giữa chúng; với bản đồ chi tiết, bạn biết chính xác hình dạng toàn bộ bề mặt — hữu ích khi cần mô tả chính xác một tương tác bề mặt (ví dụ bàn tay ôm quanh một vật thể có hình dạng phức tạp, không chỉ chạm một điểm).

**Giới hạn của loại suy này:** bản đồ địa hình là tĩnh và biết trước; correspondence bề mặt người↔robot trong UMR phải **học** được (không có sẵn "bản đồ đúng" để tra cứu) và phải tổng quát hoá được qua nhiều hình dạng cơ thể/robot khác nhau, phức tạp hơn nhiều so với việc chỉ đọc một bản đồ tĩnh có sẵn.

## 📐 Định nghĩa chính xác

**UMR (Unified Motion Retargeting for Humanoids with Learned Point Cloud Correspondence)** — Hanyang Cao, Yuetong Fang, Taesoo Kwon, Runyi Yu, Ji Ma, Jing Tan, Yangchen Zhou, Baoze Du, Yi Gu, Yukang Gao, Ruoli Dai, Lei Han, Renjing Xu ([arXiv:2609.02134](https://arxiv.org/abs/2609.02134), nộp 02/09/2026, bản sửa 07/09/2026; [trang project](https://hanyang9.github.io/UMR/); [GitHub](https://github.com/hanyang9/UMR)) — là một framework hai giai đoạn:

1. **Point Cloud Correspondence Learning:** học tương ứng có thứ tự (ordered correspondence) giữa point cloud bề mặt người và point cloud bề mặt robot, cả hai ở **tư thế canonical** (tư thế chuẩn hoá, trung lập — tương tự khái niệm "rest pose" đã gặp ở bài Skeleton mapping), rồi gắn (bind) các cặp điểm đã khớp vào mesh cơ thể tương ứng.
2. **Correspondence-Guided Retargeting:** dùng correspondence dày (dense) đã học làm "geometric anchor" chi tiết, dẫn dắt một bài toán **tối ưu hoá khớp point cloud có ràng buộc** (constrained point cloud matching optimization) để tính ra chuyển động robot — thay thế vai trò mà `ik_match_table` (sparse, thủ công) đóng trong GMR.

Theo trang chính thức, UMR coi **exterior point cloud** (đám mây điểm bề mặt ngoài của cơ thể) là "giao diện thống nhất" (unified interface) giữa nguồn dữ liệu chuyển động và các robot khác nhau — điều này **tách rời (decouple)** bài toán retargeting khỏi ngữ nghĩa xương cụ thể (skeletal semantics) của từng nguồn dữ liệu và tô pô cụ thể của từng robot, khác về nguyên tắc so với việc phải viết một bảng `ik_match_table` mới cho mỗi cặp (định dạng nguồn, robot đích) như GMR.

## ⚙️ Cơ chế hoạt động — từng bước

```text
┌──────────────────────────────────────────────────────────────┐
│ GIAI ĐOẠN 1 — Point Cloud Correspondence Learning              │
│                                                                 │
│  Human point cloud (canonical pose) ──┐                        │
│                                        ├──▶ học ordered         │
│  Robot point cloud (canonical pose) ──┘     correspondence      │
│                                                                 │
│  → gắn (bind) các cặp điểm đã khớp vào mesh người và mesh robot │
└───────────────────────────┬─────────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ GIAI ĐOẠN 2 — Correspondence-Guided Retargeting                │
│                                                                 │
│  Với mỗi frame chuyển động người mới:                          │
│    1. Áp deformation lên human mesh theo pose hiện tại          │
│    2. Điểm bề mặt người tại pose hiện tại đã có correspondence  │
│       (từ giai đoạn 1) → suy ra vị trí mục tiêu tương ứng        │
│       trên bề mặt robot                                         │
│    3. Giải constrained point-cloud matching optimization:        │
│       tìm configuration robot khớp tốt nhất với TOÀN BỘ tập      │
│       điểm mục tiêu dày đặc (không chỉ vài keypoint)             │
│    4. Đồng thời khớp lại contact map (điểm tiếp xúc vật thể/     │
│       địa hình) từ người sang robot                              │
└───────────────────────────┬─────────────────────────────────────┘
                            ▼
              Chuyển động robot: pose bề mặt khớp dày đặc
              + contact được chuyển giao trực tiếp từ dữ liệu người
```

Khác biệt kiến trúc mấu chốt so với GMR (đã học): GMR định nghĩa target IK tại **một số điểm rời rạc** (các body/frame trong `ik_match_table`, thường vài chục điểm — vai, khuỷu, cổ tay, hông, gối, cổ chân...); UMR định nghĩa target tại **hàng nghìn điểm bề mặt** được học tự động, khiến bài toán tối ưu "biết" nhiều thông tin hình dạng hơn về cách hai bề mặt nên khớp với nhau, không chỉ khớp tại các mốc xương rời rạc.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số tự chọn để làm rõ khái niệm "độ phân giải thông tin" giữa sparse và dense correspondence — không phải số liệu thật từ paper UMR, vì paper không công bố số điểm cụ thể trong các trang đã tra cứu)*

Giả sử một cánh tay có chiều dài tổng `0.55 m` (cánh tay trên 0.30m + cẳng tay 0.25m, giống ví dụ ở bài "Vì sao không thể copy góc khớp").

**Sparse correspondence (kiểu GMR):** 3 điểm mốc — vai, khuỷu, cổ tay. Khoảng cách trung bình giữa hai điểm mốc liên tiếp:

```text
khoảng cách trung bình = 0.55 m / 2 đoạn = 0.275 m = 27.5 cm
```

Nghĩa là giữa vai và khuỷu (một đoạn dài 30cm), hệ thống **không có bất kỳ thông tin ràng buộc nào** về hình dạng bề mặt trung gian — chỉ biết hai đầu mút.

**Dense point-cloud correspondence (kiểu UMR), giả sử 50 điểm phân bố đều dọc cánh tay:**

```text
khoảng cách trung bình giữa hai điểm liên tiếp = 0.55 m / 49 khoảng ≈ 0.0112 m ≈ 1.12 cm
```

**So sánh mật độ thông tin:**

```text
tỷ lệ mật độ = 27.5 cm / 1.12 cm ≈ 24.6 lần
```

**Ý nghĩa:** đây là một cách định lượng hoá trực quan cho câu nói "sparse keypoints thiếu guidance cho pose/contact chi tiết" (trích từ bài Skeleton mapping) — với dense correspondence, hệ thống có thông tin ràng buộc hình dạng dày hơn khoảng 25 lần dọc theo cùng một đoạn chi, hữu ích đặc biệt khi cần khớp chính xác một bề mặt tiếp xúc (ví dụ lòng bàn tay ôm quanh tay nắm cửa) mà chỉ 1 điểm mốc "cổ tay" của sparse mapping không thể mô tả được hình dạng ôm khít đó.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | GMR (sparse, đã học) | SOMA-retargeter (sparse, đã học) | UMR (dense, học được) |
|---|---|---|---|
| Đơn vị correspondence | Vài chục cặp khớp/body (`ik_match_table`) | Vài chục cặp khớp/body (config theo robot) | Hàng nghìn điểm bề mặt (point cloud) |
| Correspondence được xác định bằng cách nào | Thiết kế thủ công theo từng cặp (định dạng nguồn, robot đích) | Thiết kế thủ công theo từng robot có sẵn (5 robot) | Học từ dữ liệu qua giai đoạn 1 (Point Cloud Correspondence Learning) |
| Cần retrain/tạo config khi thêm robot mới? | Có — viết `ik_match_table` mới | Có — thêm cấu hình robot mới | Cần huấn luyện lại/thích ứng correspondence learning cho robot mới |
| Chuyển giao contact với vật thể | Qua foot contact stabilization (chỉ chân-sàn, đã học riêng) | Tương tự, chủ yếu chân-sàn | Trực tiếp qua contact map bề mặt, mở rộng hơn (tay-vật thể, không chỉ chân-sàn) |
| Yêu cầu đầu vào | Dict vị trí/quaternion theo khớp | BVH chuẩn SOMA-skeleton | Mesh nguồn (hình học bề mặt) + canonical template |
| Trạng thái | Paper hoàn chỉnh, ICRA 2026, dùng thực tế cho TWIST | Active beta, dùng thực tế cho SEED | Nghiên cứu mới công bố 09/2026, giai đoạn đầu |

**Khi nào dùng cái nào (nhận định thận trọng):** vì UMR mới công bố tháng 9/2026 và trang chính thức **chưa cung cấp bảng số liệu định lượng so sánh trực tiếp với GMR/OmniRetarget** (chỉ có so sánh định tính bằng hình ảnh, nhãn "SOMA Uniform + GMR" làm baseline trực quan), chưa có đủ căn cứ để khẳng định UMR "tốt hơn" GMR/SOMA-retargeter trong production ở thời điểm hiện tại — đây vẫn là một hướng nghiên cứu đang phát triển, phù hợp để theo dõi và thử nghiệm, chưa phải lựa chọn thay thế đã được kiểm chứng rộng rãi cho công cụ chính của dự án.

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "UMR đã được chứng minh vượt trội GMR và OmniRetarget bằng số liệu cụ thể".** Vì sao sai: một số bản tóm tắt tìm kiếm tự động (bao gồm cả kết quả WebSearch ban đầu dùng để tra cứu cho bài giảng này) đưa ra các tuyên bố định lượng khá mạnh (ví dụ "cải thiện tracking so với GMR", "vượt trội OmniRetarget trên hầu hết task") — nhưng khi tra cứu trực tiếp trang chính thức của UMR ([hanyang9.github.io/UMR](https://hanyang9.github.io/UMR/)), nội dung đó **chỉ có so sánh định tính bằng hình ảnh, không có bảng số liệu**. Ngoài ra, robot được thử nghiệm theo trang chính thức là **PiPlus, Adam, Fourier N1** — không phải Unitree G1 như một vài tóm tắt gián tiếp khác gợi ý. **Hiểu đúng:** khi một tuyên bố định lượng không thể xác nhận trực tiếp từ nguồn chính thức (paper PDF đầy đủ hoặc trang project), nên nêu rõ mức độ chưa chắc chắn thay vì lặp lại tuyên bố đó như sự thật đã kiểm chứng — đây chính là nguyên tắc "không bịa đặt sự thật kỹ thuật" mà toàn bộ series bài giảng này tuân theo.
2. **Hiểu nhầm: "dense correspondence luôn tốt hơn sparse vì có nhiều thông tin hơn".** Vì sao sai: nhiều thông tin hơn đi kèm chi phí — UMR cần mesh nguồn đầy đủ (không phải mọi nguồn dữ liệu chuyển động đều có mesh chi tiết, nhiều nguồn chỉ có khớp/keypoint thưa như BVH truyền thống) và một canonical template, đúng như hạn chế paper tự thừa nhận (xem mục Cập nhật hiện đại). Sparse mapping ngược lại hoạt động được ngay với dữ liệu khớp thưa phổ biến (BVH, SMPL-X khớp), không đòi hỏi mesh bề mặt đầy đủ. **Hiểu đúng:** lựa chọn dense hay sparse phụ thuộc vào *loại dữ liệu đầu vào sẵn có*, không chỉ vào "độ giàu thông tin lý thuyết" của phương pháp.

## 🏗️ Ví dụ minh hoạ trong dự án này

UMR **chưa phải công cụ chính thức được mentor chỉ định** trong dự án này (vẫn là GMR và SOMA-retargeter) — bài giảng này được thêm vào theo yêu cầu nghiên cứu mở rộng, đúng tinh thần mục "💡 Cần research thêm" đã ghi sẵn trong `README.md` của thư mục này ("tìm citation của GMR trên Semantic Scholar để xem hướng nào đang phát triển tiếp"). Vị trí hợp lý để cân nhắc dùng UMR trong tương lai của dự án là ở các tình huống mà sparse mapping của GMR/SOMA-retargeter tỏ ra yếu nhất — theo đúng nội dung đã học ở các bài trước:

```text
Khi nào sparse mapping (GMR/SOMA) yếu:
- Tương tác tay–vật thể phức tạp (ôm khít bề mặt không đều, ví dụ cầm vật hình dạng lạ)
- Cần khái quát hoá nhanh sang một robot hoàn toàn mới mà chưa có config sẵn
- Cần giữ chi tiết hình dáng bề mặt (không chỉ vị trí khớp) khi robot có hình dạng cơ thể khác biệt lớn

→ Đây là các khoảng trống mà UMR (2609.02134) và OmniRetarget (2509.26633,
  xem bài GMR) đang thử giải quyết theo hai hướng khác nhau
  (correspondence dày học được vs. interaction mesh tường minh)
```

Nếu dự án sau này cần retarget các động tác thao túng vật thể phức tạp (liên hệ `OMOMO`, `03-human-motion-datasets/`), đây chính là hướng nghiên cứu đáng thử nghiệm song song với pipeline GMR/SOMA-retargeter hiện có, chứ chưa nên thay thế hoàn toàn ngay lập tức.

## 🔥 Cập nhật hiện đại / SOTA gần đây

Vì bản thân UMR đã là một công trình 2026 rất mới, mục này tập trung vào bối cảnh xung quanh nó thay vì các công trình "cập nhật" UMR (vì UMR chính là bản cập nhật):

1. **UMR tự thừa nhận giới hạn rõ ràng.** Theo tóm tắt kết quả tra cứu, một hạn chế hiện tại được nêu là "UMR assumes access to mesh-based source geometry and a canonical source template" — nghĩa là phương pháp **không hoạt động trực tiếp** với các nguồn dữ liệu chỉ có khớp thưa (sparse joints) mà không có mesh bề mặt đầy đủ, một hạn chế không tồn tại với GMR/SOMA-retargeter (vốn chỉ cần vị trí/góc khớp). Hướng phát triển tiếp theo được nhóm tác giả nêu: mở rộng sang "less structured motion observations" (quan sát chuyển động ít cấu trúc hơn), khớp tay khéo léo (dexterous hands), và tương tác đa agent. [arXiv:2609.02134](https://arxiv.org/abs/2609.02134)
2. **UMR không đơn độc trong xu hướng "vượt ra ngoài sparse keypoint" của năm 2025–2026.** OmniRetarget (arXiv:2509.26633, xem bài GMR) tấn công một khía cạnh khác của cùng vấn đề — thay vì học dense correspondence bề mặt, nó dùng **interaction mesh tường minh** (không học) để giữ quan hệ không gian/tiếp xúc giữa agent, địa hình và vật thể. Hai hướng (học dense correspondence vs. mô hình hoá interaction mesh tường minh) đại diện cho hai triết lý khác nhau đang cùng phát triển để giải quyết cùng một lớp hạn chế của sparse mapping cổ điển — chưa có bằng chứng nào cho thấy một trong hai đã "thắng thế" hoàn toàn.
3. **Thận trọng cần thiết khi trích dẫn UMR:** vì đây là paper vừa công bố (arXiv nộp 02/09/2026, sửa 07/09/2026 — chỉ khoảng 2 tuần trước thời điểm tra cứu 09/2026), số liệu định lượng đầy đủ, kết quả kiểm định bởi cộng đồng (citation, reproduction, so sánh độc lập) chưa có đủ thời gian để tích luỹ. Đây là lý do bài giảng này nhấn mạnh việc phân biệt rõ "những gì trang chính thức xác nhận" (kiến trúc hai giai đoạn, robot thử nghiệm, hạn chế tự nêu) với "những gì chưa xác nhận được" (bảng số liệu so sánh cụ thể) — một thực hành cẩn trọng cần thiết khi viết về nghiên cứu quá mới.

## ❓ Câu hỏi tự kiểm tra

1. Hai giai đoạn của UMR là gì, và giai đoạn nào đóng vai trò tương đương `ik_match_table` trong GMR?
   <details><summary>Gợi ý đáp án</summary>(1) Point Cloud Correspondence Learning — học tương ứng dày giữa bề mặt người và robot ở tư thế canonical; (2) Correspondence-Guided Retargeting — dùng correspondence đó để giải bài toán khớp point cloud. Giai đoạn 1 đóng vai trò tương đương `ik_match_table`, nhưng được học từ dữ liệu thay vì thiết kế tay.</details>
2. Trong ví dụ tính tay, vì sao mật độ thông tin của dense correspondence được tính là "khoảng 25 lần" so với sparse — con số này minh hoạ điều gì, và có phải số liệu thật từ paper UMR không?
   <details><summary>Gợi ý đáp án</summary>Đây là ví dụ minh hoạ tự chọn (50 điểm so với 3 điểm mốc trên cùng một đoạn chi dài 0.55m), KHÔNG phải số liệu thật công bố trong paper UMR — mục đích chỉ để định lượng hoá trực quan khái niệm "dense vs sparse", không phải trích dẫn một con số đã kiểm chứng.</details>
3. Vì sao không nên khẳng định "UMR đã chứng minh vượt trội GMR về mặt số liệu" dựa trên bài giảng này?
   <details><summary>Gợi ý đáp án</summary>Vì khi tra cứu trực tiếp trang chính thức của UMR, chỉ tìm thấy so sánh định tính bằng hình ảnh, không có bảng số liệu định lượng cụ thể so với GMR/OmniRetarget — một số tóm tắt tìm kiếm gián tiếp đưa ra tuyên bố mạnh hơn nhưng không thể xác nhận trực tiếp từ nguồn chính thức, nên bài giảng chọn không lặp lại các tuyên bố đó như sự thật đã kiểm chứng.</details>
4. UMR yêu cầu loại dữ liệu đầu vào nào mà GMR/SOMA-retargeter không yêu cầu, và đây là ưu điểm hay hạn chế?
   <details><summary>Gợi ý đáp án</summary>UMR cần mesh bề mặt nguồn đầy đủ (mesh-based source geometry) và canonical source template; đây là một HẠN CHẾ (do chính nhóm tác giả nêu) vì không phải mọi nguồn dữ liệu chuyển động phổ biến (ví dụ BVH khớp thưa truyền thống) đều có sẵn mesh chi tiết như vậy.</details>
5. UMR và OmniRetarget cùng nhắm giải quyết hạn chế gì của retargeting cổ điển, nhưng theo hai cách khác nhau như thế nào?
   <details><summary>Gợi ý đáp án</summary>Cả hai cùng nhắm khắc phục việc sparse keypoint mapping thiếu guidance chi tiết cho pose/contact/tương tác vật thể; UMR giải quyết bằng cách HỌC dense point-cloud correspondence, còn OmniRetarget giải quyết bằng cách mô hình hoá TƯỜNG MINH (không học) quan hệ không gian qua interaction mesh.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** với chain chân (đùi 0.45m + ống chân 0.40m, tổng 0.85m), so sánh mật độ thông tin giữa sparse mapping (3 điểm mốc: hông, gối, cổ chân) và dense correspondence giả định 80 điểm phân bố đều — tính khoảng cách trung bình giữa các điểm liên tiếp cho cả hai trường hợp và tỷ lệ mật độ, theo đúng cách tính đã làm ở ví dụ Phần A trong bài.
2. **Đọc nguồn thật, đối chiếu cẩn trọng:** mở trực tiếp [arXiv:2609.02134](https://arxiv.org/abs/2609.02134) (bản PDF hoặc HTML đầy đủ, không chỉ abstract) và tìm mục kết quả thực nghiệm (Experiments/Results) — kiểm tra xem có bảng số liệu so sánh định lượng cụ thể với GMR và/hoặc OmniRetarget hay không (ví dụ theo metric MPJPE, physical plausibility score, user study). Nếu có, ghi lại số liệu chính xác kèm tên bảng/hình; nếu không tìm thấy, ghi rõ kết luận "chưa có số liệu định lượng công khai tại thời điểm đọc" — đây là bài tập rèn thói quen xác minh nguồn trước khi trích dẫn, đúng tinh thần Sai lầm thường gặp #1 ở trên.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

UMR (Cao et al., arXiv:2609.02134, 09/2026) là một hướng retargeting mới nhắm đúng điểm nghẽn chung của cả GMR lẫn SOMA-retargeter — correspondence sparse, thiết kế thủ công — bằng cách học tương ứng dày (dense) giữa point cloud bề mặt người và robot ở tư thế canonical (giai đoạn 1), rồi dùng correspondence đó dẫn dắt một bài toán tối ưu khớp point cloud có ràng buộc để tính chuyển động robot, bao gồm cả chuyển giao contact tương tác vật thể trực tiếp (giai đoạn 2); ví dụ tính tay minh hoạ correspondence dày có thể cung cấp mật độ thông tin hình học cao hơn nhiều bậc so với vài chục điểm mốc khớp truyền thống. Tuy nhiên, tại thời điểm bài giảng này được viết, trang chính thức của UMR chỉ cung cấp so sánh định tính (không có bảng số liệu định lượng đã xác nhận so với GMR/OmniRetarget), và bản thân phương pháp có hạn chế rõ ràng là cần mesh nguồn đầy đủ — vì vậy UMR nên được hiểu là một hướng nghiên cứu 2026 đầy hứa hẹn nhưng còn rất mới, đáng theo dõi và thử nghiệm song song, chưa phải sự thay thế đã được kiểm chứng cho GMR/SOMA-retargeter trong pipeline chính của dự án.
