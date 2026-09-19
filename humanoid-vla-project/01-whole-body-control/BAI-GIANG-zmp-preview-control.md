# Bài giảng: ZMP preview control

*(Thuộc mảng: Whole-Body Control (WBC))*

## 🎯 Mục tiêu bài học

- Giải thích được vấn đề cụ thể mô hình cart-table để lại mà preview control giải quyết: sai lệch giữa mô hình đơn giản hoá và động lực học thật nhiều khớp.
- Mô tả được 3 thành phần của bộ điều khiển preview: phản hồi trạng thái, phản hồi tích luỹ sai số, và số hạng hành động dự đoán (preview action).
- Giải thích được vì sao "nhìn trước" một cửa sổ quỹ đạo ZMP tương lai lại giúp hệ bám tốt hơn so với chỉ phản ứng tại chỗ.
- Tính tay được một ví dụ số đơn giản minh hoạ cách preview control "chuẩn bị trước" cho một thay đổi ZMP tham chiếu sắp tới.
- Phân biệt được preview control với MPC (bài giảng tiếp theo) — cùng tư duy "nhìn trước" nhưng khác nhau ở điểm nào.
- Nêu được vị trí của khái niệm DCM (Divergent Component of Motion) như một hướng phát triển hiện đại song song với preview control.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Bài trước (ZMP và mô hình cart-table) đã học: Kajita đơn giản hoá cơ thể robot thành một khối lượng điểm trên "bàn" không khối lượng để tính ZMP nhanh, với công thức tuyến tính `p = x_com − (z_c/g)·ẍ_com`. Nhưng đơn giản hoá luôn có cái giá: mô hình cart-table chỉ là **xấp xỉ** — cơ thể robot thật có hàng chục khớp, tay chân di chuyển, phân bố khối lượng thay đổi liên tục, khác xa "một khối lượng điểm trên bàn". Preview control là câu trả lời cụ thể của Kajita cho câu hỏi: "làm sao dùng mô hình đơn giản đó mà vẫn bám sát được quỹ đạo ZMP mong muốn, bất chấp sai số xấp xỉ?" — câu trả lời không nằm ở việc làm mô hình phức tạp hơn, mà ở việc **tận dụng thông tin đã biết trước về tương lai gần**.

## 🧠 Trực giác

### Góc nhìn 1: Người đi bộ nhìn trước vài bước, không chỉ nhìn xuống chân mình

Khi đi bộ trên vỉa hè gồ ghề, một người có kinh nghiệm không nhìn chằm chằm xuống bàn chân mình tại từng bước — họ **nhìn trước vài mét**, thấy trước chỗ nào có ổ gà, rồi **điều chỉnh dáng đi ngay từ bây giờ** (hơi nghiêng người, đổi độ dài bước) để khi tới đó đã ở tư thế phù hợp, thay vì đợi tới đúng lúc đặt chân vào ổ gà mới phản ứng. Preview control làm đúng việc này với ZMP: "nhìn trước" quỹ đạo ZMP tham chiếu tương lai (đã biết trước vì dáng đi được lên kế hoạch trước) để chuẩn bị bù trừ ngay từ hiện tại.

**Giới hạn của loại suy này:** người đi bộ "nhìn trước" bằng mắt và xử lý thông tin thị giác phức tạp, không chắc chắn (chưa biết chính xác độ sâu ổ gà); preview control "nhìn trước" một quỹ đạo **đã biết chính xác trước** (vì chính hệ thống tự lên kế hoạch dáng đi, không phải quan sát môi trường bất định) — đây là khác biệt quan trọng: preview control tận dụng thông tin **đã có sẵn, xác định**, không phải dự đoán một tương lai bất định.

### Góc nhìn 2: Lái xe với hộp số tự động dự đoán trước khúc cua, so với chỉ dựa vào tốc độ hiện tại

Một số hệ thống hộp số ô tô hiện đại dùng GPS + bản đồ để biết trước "phía trước 200m có khúc cua gấp" và **chủ động chuyển số sớm** trước khi vào cua, thay vì chỉ phản ứng dựa trên tốc độ/độ dốc hiện tại tại đúng thời điểm vào cua. Bộ điều khiển preview cũng vậy: thay vì chỉ dùng trạng thái hiện tại (state feedback) để quyết định, nó cộng thêm một "số hạng dự đoán" dựa trên toàn bộ đoạn quỹ đạo ZMP tham chiếu sắp tới.

**Giới hạn của loại suy này:** hộp số dự đoán trước chỉ cần "chuyển số đúng lúc" (một quyết định rời rạc, đơn giản); bộ điều khiển preview phải tính toán một **tổ hợp tuyến tính có trọng số** của toàn bộ cửa sổ tương lai (không phải một quyết định nhị phân đơn lẻ) — độ phức tạp toán học cao hơn nhiều so với "chuyển số sớm hay muộn".

## 📐 Định nghĩa chính xác

**Vấn đề:** mô hình cart-table chỉ là xấp xỉ — sai số giữa mô hình đơn giản và động lực học thật (nhiều khớp) gây lệch ZMP.

**Preview control (Kajita et al. 2003):** bộ điều khiển không chỉ nhìn trạng thái hiện tại, mà **"nhìn trước" (preview) một cửa sổ quỹ đạo ZMP tham chiếu trong tương lai gần** (ví dụ 1–2 giây tới, đã biết trước vì dáng đi được lên kế hoạch trước), rồi dùng thông tin tương lai đó để tính bù ngay từ bây giờ.

Bộ điều khiển kết hợp **ba thành phần**:

```text
u(k) = −K_x · x(k)                            [1. phản hồi trạng thái]
       − K_i · Σᵢ₌₀ᵏ e(i)                      [2. phản hồi tích luỹ sai số theo dõi]
       + Σⱼ₌₁ᴺ Gp(j) · p_ref(k+j)               [3. số hạng hành động dự đoán/preview action]
```

trong đó:
- `x(k)` là trạng thái hiện tại (vị trí, vận tốc, gia tốc trọng tâm),
- `e(i) = p(i) − p_ref(i)` là sai số theo dõi ZMP tích luỹ theo thời gian,
- `p_ref(k+j)` là giá trị ZMP tham chiếu tại `j` bước **trong tương lai** (đã biết trước),
- `N` là độ dài cửa sổ preview (số bước nhìn trước),
- `K_x, K_i, Gp(j)` là các hệ số điều khiển (gain) được tính trước bằng lý thuyết điều khiển tối ưu (thường qua giải phương trình Riccati rời rạc cho bài toán LQR mở rộng có preview).

**Trực giác công thức:** số hạng thứ 3 (`Σ Gp(j)·p_ref(k+j)`) chính là phần "mới" so với điều khiển phản hồi cổ điển — nó cộng thêm một lượng điều chỉnh **tỷ lệ với các giá trị ZMP tham chiếu tương lai**, với trọng số `Gp(j)` thường **giảm dần** khi `j` tăng (tương lai càng xa càng ít ảnh hưởng tới quyết định ngay bây giờ) — đúng trực giác "chuẩn bị trước nhưng không phản ứng thái quá với thứ còn xa".

## ⚙️ Cơ chế hoạt động — từng bước

```text
┌────────────────────────────────────────────────────────────┐
│ INPUT: quỹ đạo ZMP tham chiếu p_ref(k), k=0...T ĐÃ BIẾT TRƯỚC  │
│  (vì dáng đi/footstep plan được lên kế hoạch trước, không     │
│   phải cảm biến theo thời gian thực)                          │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ Tại mỗi bước thời gian k:                                      │
│  1. Đọc trạng thái hiện tại x(k) (từ mô hình cart-table:       │
│     vị trí/vận tốc/gia tốc trọng tâm)                          │
│  2. Tính sai số tích luỹ e(k) = p(k) − p_ref(k)                 │
│  3. Lấy cửa sổ N giá trị TƯƠNG LAI: p_ref(k+1)...p_ref(k+N)     │
│  4. Tính u(k) = phản hồi trạng thái + phản hồi tích luỹ         │
│                 + Σ Gp(j)·p_ref(k+j)  (số hạng preview)         │
│  5. Áp u(k) làm gia tốc giật (jerk) điều khiển trọng tâm        │
│     (biến điều khiển trong công thức gốc Kajita là đạo hàm      │
│     bậc 3 của vị trí — jerk — để đảm bảo gia tốc mượt)          │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ Cập nhật trạng thái x(k+1) qua mô hình cart-table              │
│ Lặp lại cho bước k+1                                            │
└──────────────────────────────────────────────────────────────┘
```

**Vì sao các gain (`K_x, K_i, Gp(j)`) được tính TRƯỚC, không tính lại mỗi bước:** vì mô hình cart-table là tuyến tính bất biến theo thời gian (linear time-invariant), bài toán tối ưu hoá gain (thường dạng LQR — Linear Quadratic Regulator mở rộng với preview) chỉ cần giải **một lần duy nhất, offline**, cho ra một bộ hệ số cố định — khác với MPC (bài giảng tiếp theo), nơi bài toán tối ưu được **giải lại mỗi bước** với ràng buộc có thể thay đổi.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ đơn giản hoá cực độ, số tự chọn để thấy rõ trực giác "chuẩn bị trước" — không phải giải đầy đủ phương trình Riccati thật)*

Giả sử ZMP tham chiếu đang ở `p_ref = 0` cm (ổn định), nhưng tại bước `k+5` (tức 5 bước × Δt sau, ví dụ Δt=0.02s → 0.1 giây sau) robot cần bắt đầu chuyển ZMP sang `p_ref = 3` cm (chuẩn bị dồn trọng lượng sang chân phải để bước tiếp).

**Không có preview (chỉ phản hồi trạng thái + tích luỹ sai số):** tại bước `k`, vì `p_ref(k) = 0` và trạng thái hiện tại đang khớp `p_ref`, bộ điều khiển **không làm gì cả** — nó chỉ "phát hiện" sự thay đổi khi `p_ref` thực sự nhảy lên 3cm tại bước `k+5`, lúc đó mới bắt đầu phản ứng, gây ra một **độ trễ bám (tracking lag)**.

**Có preview (cửa sổ N=10 bước, giả sử `Gp(j)` giảm tuyến tính từ `Gp(1)=0.15` xuống `Gp(10)=0.02`, đơn vị minh hoạ):**

```text
u(k) = 0 (phản hồi trạng thái, giả sử đang ổn định)
     + 0 (phản hồi tích luỹ, giả sử sai số tích luỹ = 0)
     + Σⱼ₌₁¹⁰ Gp(j)·p_ref(k+j)
```

Vì `p_ref(k+1)...p_ref(k+4) = 0` và `p_ref(k+5)...p_ref(k+10) = 3` cm (giả sử giữ nguyên 3cm sau khi chuyển):

```text
u(k) = Gp(5)·3 + Gp(6)·3 + Gp(7)·3 + Gp(8)·3 + Gp(9)·3 + Gp(10)·3
```

Giả sử `Gp(5)=0.10, Gp(6)=0.08, Gp(7)=0.06, Gp(8)=0.05, Gp(9)=0.035, Gp(10)=0.02` (giá trị minh hoạ giảm dần theo khoảng cách tới tương lai):

```text
u(k) = 3×(0.10+0.08+0.06+0.05+0.035+0.02)
     = 3×0.345
     = 1.035 (đơn vị jerk minh hoạ)
```

**Ý nghĩa:** ngay tại bước `k` — **5 bước trước khi ZMP tham chiếu thực sự thay đổi** — bộ điều khiển đã bắt đầu tạo ra một tín hiệu điều khiển khác 0 (`u(k)=1.035`), tức là **đã bắt đầu di chuyển trọng tâm chuẩn bị trước**, dù `p_ref(k)` vẫn đang là 0. Đây chính là cơ chế cụ thể hoá câu "chuẩn bị trước cho một thay đổi sắp tới" — nếu không có preview, robot phải đợi đúng bước `k+5` mới bắt đầu phản ứng, dẫn tới lag và có thể gây giật/mất cân bằng khi ZMP tham chiếu thay đổi đột ngột.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | ZMP preview control (Kajita 2003) | MPC cho dáng đi (bài tiếp theo) | Chỉ phản hồi (feedback thuần, không preview) |
|---|---|---|---|
| Có "nhìn trước" tương lai không? | Có, cửa sổ cố định N bước | Có, cửa sổ trượt (receding horizon) | Không |
| Gain/hệ số điều khiển | Tính TRƯỚC (offline), cố định | Giải lại MỖI BƯỚC (online) | Tính trước, cố định |
| Xử lý ràng buộc bất đẳng thức trực tiếp? | Không (mô hình tuyến tính thuần) | Có (đưa thẳng vào bài toán tối ưu) | Không |
| Chi phí tính toán online | Thấp (chỉ nhân-cộng theo gain có sẵn) | Cao hơn (giải QP/tối ưu mỗi bước) | Thấp nhất |
| Độ trễ bám khi tham chiếu thay đổi | Giảm đáng kể nhờ preview | Giảm tương tự, còn linh hoạt hơn với ràng buộc | Cao nhất (chỉ phản ứng khi đã xảy ra) |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "preview control 'dự đoán' tương lai giống như một mô hình machine learning dự báo".** Vì sao sai: preview control không **dự đoán** gì cả — nó dùng một quỹ đạo tham chiếu **đã biết chính xác trước** (vì chính hệ thống lập kế hoạch dáng đi/footstep đã tạo ra quỹ đạo đó từ trước), không có yếu tố bất định hay xác suất nào. **Hiểu đúng:** "preview" ở đây nghĩa là "đã có sẵn thông tin về tương lai gần, chỉ chưa tới thời điểm đó" — khác hoàn toàn với "dự báo" một tương lai chưa biết chắc chắn.
2. **Hiểu nhầm: "vì đã tính gain trước (offline), preview control không thể xử lý nhiễu/thay đổi bất ngờ".** Vì sao sai: mặc dù gain cố định, hệ thống vẫn có **phản hồi trạng thái** (`K_x`) và **phản hồi tích luỹ sai số** (`K_i`) hoạt động theo thời gian thực dựa trên trạng thái đo được thực tế — hai thành phần này xử lý sai lệch thực tế so với mô hình, độc lập với phần preview (vốn chỉ xử lý phần "đã biết trước"). **Hiểu đúng:** preview control là sự kết hợp giữa phần phản ứng thời gian thực (feedback, xử lý bất định) và phần chuẩn bị trước (feedforward/preview, xử lý phần đã biết) — không phải chỉ có một trong hai.

## 🏗️ Ví dụ minh hoạ trong dự án này

```text
Kế hoạch bước chân (footstep planner) — biết trước vị trí đặt chân
  N bước tiếp theo
        │
        ▼
Sinh quỹ đạo ZMP tham chiếu p_ref(k) cho toàn bộ chuỗi bước
        │
        ▼
ZMP preview control: tính quỹ đạo trọng tâm x_com(k) sao cho
  ZMP thực tế bám sát p_ref, dùng preview window
        │
        ▼
Chuyển thành lệnh vị trí/vận tốc cho các khớp chân
  (qua IK hoặc kết hợp với Task-space control, bài đã học)
        │
        ▼
Trong dự án này: preview control là NỀN TẢNG CỔ ĐIỂN mà các
  policy RL hiện đại (04-imitation-learning-rl/, PHC→OmniH2O→ASAP→SONIC)
  đang dần thay thế — nhưng hiểu preview control giúp hiểu TẠI SAO
  RL cần học "giữ thăng bằng khi di chuyển" — đây chính là bài toán
  mà preview control giải bằng công thức tường minh, RL giải bằng
  học từ dữ liệu/thử-sai
```

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **DCM (Divergent Component of Motion) — hướng phát triển song song, không thay thế hoàn toàn preview control.** DCM được định nghĩa `ξ = c + ċ/ω` (với `c` là vị trí trọng tâm, `ω` là hằng số thời gian tự nhiên của con lắc ngược tuyến tính — LIPM). Điểm mấu chốt: DCM **tự nhiên phân kỳ (diverge) ra khỏi ZMP**, trong khi trọng tâm được đảm bảo **hội tụ về DCM mà không cần điều khiển tường minh** — nghĩa là thay vì phải tính preview cho cả quỹ đạo ZMP, một số phương pháp hiện đại chỉ cần tính **quỹ đạo DCM tham chiếu khả thi** (dùng MPC tuyến tính trên cửa sổ preview ngắn cho cả pha một chân/hai chân chạm đất), rồi để động lực học tự nhiên đưa trọng tâm hội tụ theo. Nghiên cứu 2025 kết hợp cả hai: dùng preview control để sinh mẫu hình dáng đi (pattern generation) và phản hồi DCM để ổn định hoá — cho thấy hai tư tưởng (preview control cổ điển và DCM) đang được **kết hợp**, không loại trừ nhau, trong cùng một kiến trúc điều khiển phân lớp.
2. **Nghiên cứu về đi trên địa hình gồ ghề/nhiễu biến thiên theo thời gian** dựa trên DCM cho phép robot duy trì cân bằng động khi gặp nhiễu lớn thay đổi theo thời gian trên bề mặt gồ ghề — mở rộng trực tiếp khả năng của preview control cổ điển (vốn giả định địa hình phẳng, biết trước hoàn toàn) sang các tình huống có bất định về địa hình.
3. **Xu hướng chung 2025-2026:** preview control/DCM tiếp tục là lớp điều khiển "đế" (base layer) ổn định, chi phí tính toán thấp, dùng làm baseline hoặc lớp an toàn (safety layer) kết hợp với các phương pháp học sâu hiện đại hơn (MPC học tăng cường, RL end-to-end) — không biến mất khỏi thực tiễn dù RL (mục 2 của thư mục này) ngày càng phổ biến cho các hành vi phức tạp hơn.

## ❓ Câu hỏi tự kiểm tra

1. Ba thành phần của bộ điều khiển preview là gì, và thành phần nào là "mới" so với điều khiển phản hồi cổ điển?
   <details><summary>Gợi ý đáp án</summary>Phản hồi trạng thái, phản hồi tích luỹ sai số theo dõi, và số hạng hành động dự đoán (preview action dựa trên quỹ đạo ZMP tham chiếu tương lai) — thành phần thứ ba là điểm mới cốt lõi.</details>
2. Trong ví dụ tính tay, vì sao `u(k)` khác 0 ngay cả khi `p_ref(k)=0` và trạng thái hiện tại đang ổn định?
   <details><summary>Gợi ý đáp án</summary>Vì số hạng preview `Σ Gp(j)·p_ref(k+j)` nhìn thấy trước rằng `p_ref` sẽ thay đổi thành 3cm tại các bước tương lai (k+5 trở đi), nên đã bắt đầu tạo tín hiệu điều khiển chuẩn bị trước, dù giá trị hiện tại vẫn là 0.</details>
3. Vì sao gain của preview control được tính trước (offline) một lần, trong khi MPC phải giải lại mỗi bước?
   <details><summary>Gợi ý đáp án</summary>Vì mô hình cart-table là tuyến tính bất biến theo thời gian (LTI) và không có ràng buộc bất đẳng thức trong công thức — bài toán tối ưu gain (LQR mở rộng preview) có nghiệm giải tích cố định; MPC đưa thêm ràng buộc bất đẳng thức/phi tuyến trực tiếp vào bài toán, nên cần giải lại mỗi bước với thông tin cập nhật mới nhất.</details>
4. DCM khác ZMP ở điểm nào về mặt động lực học?
   <details><summary>Gợi ý đáp án</summary>DCM (`ξ = c + ċ/ω`) tự nhiên phân kỳ ra khỏi ZMP theo động lực học con lắc ngược tuyến tính, trong khi trọng tâm hội tụ về DCM MÀ KHÔNG CẦN điều khiển tường minh — cho phép một số phương pháp chỉ cần lập kế hoạch DCM tham chiếu thay vì toàn bộ quỹ đạo ZMP.</details>
5. Vì sao "preview" trong preview control không nên hiểu là "dự đoán" theo nghĩa machine learning?
   <details><summary>Gợi ý đáp án</summary>Vì quỹ đạo tương lai dùng trong preview đã được biết chính xác trước (do chính hệ thống lập kế hoạch dáng đi tạo ra), không có yếu tố bất định/xác suất như một mô hình dự báo — preview chỉ là "đã có sẵn thông tin, chưa tới lúc dùng", không phải "đoán trước điều chưa biết".</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** với cùng cấu trúc gain `Gp(j)` đã cho, tính `u(k)` nếu `p_ref` chuyển từ 0 sang **5cm** (thay vì 3cm) bắt đầu từ bước `k+3` (thay vì k+5) — giữ nguyên các giá trị `Gp(j)` cho `j=3..10`, tự chọn thêm `Gp(3), Gp(4)` hợp lý (nội suy giữa các giá trị đã cho) theo xu hướng giảm dần.
2. **Đọc paper/tài liệu thật:** tìm bản dịch hoặc tóm tắt paper gốc Kajita et al. 2003 ("Biped Walking Pattern Generation by using Preview Control of Zero-Moment Point") — xác định chính xác biến điều khiển thật sự dùng trong công thức gốc là gì (gợi ý: đạo hàm bậc 3 của vị trí trọng tâm — jerk), và giải thích bằng lời tại sao dùng jerk làm biến điều khiển (thay vì trực tiếp dùng gia tốc) lại cho quỹ đạo trọng tâm mượt hơn.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

ZMP preview control (Kajita et al. 2003) giải quyết hạn chế của mô hình cart-table (chỉ là xấp xỉ, gây sai lệch ZMP so với động lực học thật) bằng cách kết hợp ba thành phần điều khiển — phản hồi trạng thái, phản hồi tích luỹ sai số, và một số hạng "hành động dự đoán" dựa trên cửa sổ quỹ đạo ZMP tham chiếu tương lai đã biết trước — như ví dụ tính tay minh hoạ, số hạng preview cho phép hệ thống bắt đầu điều chỉnh trọng tâm nhiều bước TRƯỚC khi ZMP tham chiếu thực sự thay đổi, giảm đáng kể độ trễ bám so với chỉ phản hồi thuần tuý. Khác với MPC (bài tiếp theo), gain của preview control được tính một lần offline nhờ mô hình tuyến tính bất biến theo thời gian, đổi lại không xử lý trực tiếp được ràng buộc bất đẳng thức; hướng phát triển hiện đại (DCM) không thay thế mà thường được kết hợp song song với preview control trong các kiến trúc điều khiển dáng đi 2025-2026.
