# Bài giảng: Contact dynamics — velocity-stepping/convex optimization vs spring-damper

*(Thuộc mảng: Simulation: MuJoCo & Isaac Lab)*

## 🎯 Mục tiêu bài học

- Giải thích được vì sao "tính va chạm" (contact) là câu hỏi kiến trúc thứ hai (độc lập với generalized/Cartesian coordinates) mà mọi physics engine phải trả lời.
- Phân biệt được cơ chế spring-damper/penalty method và velocity-stepping/convex optimization, biết trade-off cụ thể của mỗi cách.
- Giải thích được vì sao spring-damper cần timestep nhỏ để ổn định, còn cách MuJoCo dùng lại ổn định ở timestep lớn hơn.
- Tính tay được (ở mức minh hoạ) một bước penalty method đơn giản, thấy trực tiếp vấn đề "cứng lò xo vs nổ số".
- Liên hệ được chất lượng contact solver với sim-to-real gap khi huấn luyện locomotion cho humanoid.
- Không nhầm khái niệm này với "generalized vs Cartesian coordinates" (bài giảng riêng) — dù cả hai cùng xuất hiện trong 1 paper gốc.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Nếu generalized coordinates (bài giảng trước) là câu trả lời cho "hệ khớp được biểu diễn ra sao", thì contact dynamics là câu trả lời cho câu hỏi thứ hai, thực dụng hơn nhiều với robot đi lại: **khi bàn chân robot chạm sàn, điều gì xảy ra tiếp theo?** Đây là nơi phần lớn "cảm giác vật lý" của một mô phỏng đến từ — chân robot có "đứng vững như thật" hay "rung/nảy như cao su" phụ thuộc gần như hoàn toàn vào câu trả lời cho câu hỏi này, không phải vào độ chi tiết hình học hay độ phân giải mesh.

Với dự án humanoid của bạn, đây là khái niệm quyết định trực tiếp: (1) bạn có thể dùng timestep bao lớn mà mô phỏng vẫn ổn định (ảnh hưởng tốc độ training), và (2) sim-to-real gap khi chuyển policy từ MuJoCo/Isaac Lab sang robot G1/H1 thật lớn hay nhỏ — vì hành vi tiếp xúc chân-sàn là thứ khó match nhất giữa sim và real.

## 🧠 Trực giác

### Góc nhìn 1: "Đệm lò xo cũ" vs "trọng tài phân xử tức thời"

**Spring-damper/penalty method** giống như việc chặn một quả bóng đang lăn bằng một tấm đệm lò xo thật: quả bóng lún vào đệm một chút, lò xo bị nén sinh ra lực đẩy ngược tỉ lệ với độ lún — càng lún sâu, lực đẩy càng mạnh. Vấn đề: nếu đệm "mềm" (lò xo yếu), quả bóng lún sâu trông rất phi thực tế (chân robot "xuyên" vào sàn); nếu đệm "cứng" (lò xo rất mạnh) để tránh lún sâu, bạn cần "quay phim" (mô phỏng) ở tốc độ khung hình cực cao (timestep cực nhỏ) nếu không lực phản hồi sẽ dao động mất kiểm soát — giống như một lò xo thật bị nén-bật quá nhanh so với tốc độ máy quay, hình ảnh sẽ bị "giật, nổ".

**Velocity-stepping/convex optimization** giống như một trọng tài bóng đá cực nhanh: ngay khi 2 cầu thủ chạm nhau, trọng tài **giải ngay một bài toán** ("với luật không được xuyên qua nhau + luật ma sát, vận tốc hợp lý nhất của cả hai sau va chạm là gì?") và áp đặt trực tiếp kết quả đó — không có khái niệm "lún" tạm thời rồi bật ra, kết quả được tính ra thẳng bằng suy luận logic-số học tại đúng thời điểm chạm, nên không cần "quay chậm" (timestep nhỏ) để bắt kịp một quá trình lò xo nhanh.

**Giới hạn của phép loại suy này:** trọng tài trong đời thực không "tối ưu hoá lồi" theo nghĩa toán học chặt chẽ — phép loại suy chỉ giúp hình dung việc "giải trực tiếp" thay vì "mô phỏng một quá trình vật lý trung gian", không mô tả đúng bản chất bài toán quy hoạch lồi (convex program) với hàm mục tiêu và ràng buộc cụ thể.

### Góc nhìn 2: "Dây cao su treo vật" vs "ràng buộc cứng tức thời"

Một góc nhìn kỹ thuật hơn: hãy hình dung việc giữ 1 vật không rơi xuyên sàn bằng 2 cách. Cách 1 (spring-damper): buộc vật vào 1 sợi dây cao su gắn cố định ngay tại mặt sàn — vật có thể kéo dài dây cao su ra một chút (lún) trước khi bị kéo lại. Độ giãn của cao su tỉ lệ trực tiếp với lực phản hồi — đây là một hệ vật lý *có thật, liên tục*, và như mọi hệ lò xo-khối lượng, nó có tần số dao động riêng; nếu timestep mô phỏng dài hơn 1 phần nhỏ của chu kỳ dao động đó, thuật toán tích phân số (numerical integration) sẽ "bỏ lỡ" các dao động nhanh này và tích luỹ sai số theo cấp số nhân — đây chính là "numerical explosion". Cách 2 (velocity-stepping): tưởng tượng thay vì dây cao su, bạn có 1 thanh cứng tuyệt đối chỉ hoạt động *một phía* (không cho xuyên qua nhưng không cản khi tách ra) — bài toán "vật thể phải dừng lại đúng mức không xuyên sàn, với it nhất lực cần thiết" trở thành bài toán tối ưu có ràng buộc bất đẳng thức (complementarity), giải được trực tiếp bằng thuật toán quy hoạch lồi mà không cần mô hình hoá độ cứng liên tục nào.

**Giới hạn của phép loại suy này:** "thanh cứng một phía" không capture được ma sát Coulomb (một chiều tiếp tuyến khác với chiều pháp tuyến) — bài toán tối ưu thật của MuJoCo phải xử lý đồng thời cả ràng buộc không-xuyên-thấu (normal) và ràng buộc ma sát dạng nón (friction cone), phức tạp hơn một "thanh cứng" đơn giản; phép loại suy chỉ giúp hình dung phần "giải trực tiếp thay vì lò xo", không đủ để hình dung phần ma sát.

## 📐 Định nghĩa chính xác

Theo Todorov, Erez, Tassa (2012), khi 2 geometry (`<geom>`) chạm nhau, engine phải xác định lực tiếp xúc \(f_c\) (và do đó gia tốc/vận tốc tiếp theo của hệ) thoả mãn các ràng buộc vật lý: (1) không xuyên thấu (non-penetration), (2) ma sát Coulomb (lực tiếp tuyến bị giới hạn bởi hệ số ma sát × lực pháp tuyến — về mặt hình học tạo thành một "nón ma sát"), (3) lực tiếp xúc chỉ có thể đẩy ra, không thể "hút" (unilateral constraint).

- **Spring-damper / penalty method:** mô hình hoá lực tiếp xúc như \(f_c = -k \cdot d - c \cdot \dot d\), trong đó \(d\) là độ xuyên thấu (penetration depth, \(d > 0\) khi 2 vật lún vào nhau), \(k\) là hệ số cứng lò xo, \(c\) là hệ số giảm chấn (damping). Đây là một hàm liên tục, tường minh của trạng thái hiện tại — không cần giải bài toán tối ưu, chỉ cần "cắm số vào công thức". Nhưng vì đây là một hệ dao động điều hoà tắt dần với tần số riêng \(\omega \propto \sqrt{k/m}\), để mô phỏng ổn định bằng phương pháp tích phân số hiển (explicit integration), timestep \(\Delta t\) phải nhỏ hơn nhiều so với \(1/\omega\) — nghĩa là \(k\) càng lớn (để giảm lún phi thực tế) thì \(\Delta t\) càng phải nhỏ, dẫn tới chi phí tính toán tăng vọt hoặc phải dùng tích phân ẩn (implicit) phức tạp hơn.
- **Velocity-stepping / convex optimization:** tại mỗi bước thời gian, thay vì dùng công thức lực tường minh, MuJoCo **giải một bài toán tối ưu hoá lồi** (theo tài liệu chính thức mô tả là dạng liên quan tới Cone Complementarity Problem — CCP — tương đương một bài toán Second-Order Cone Programming, SOCP, khi dùng mô hình nón ma sát dạng elliptic) để tìm trực tiếp **vận tốc sau va chạm** (không phải lực trung gian) thoả mãn đồng thời tất cả ràng buộc không-xuyên-thấu và ma sát. Vì đây là một bài toán được giải "đúng" tại mỗi bước (trong giới hạn dung sai số của solver — PGS, CG, hoặc Newton, tuỳ cấu hình `<option solver=".."/>`), không có "hệ dao động riêng" nào cần né tránh bằng timestep nhỏ — kết quả được mô tả trong tài liệu MuJoCo là "soft, convex, và analytically-invertible".

> Ghi chú xác minh (giữ nguyên từ nguồn NOI-DUNG-CHI-TIET.md): chi tiết toán học đầy đủ của bài toán tối ưu hoá (dạng LCP/QP cụ thể, các loại solver PGS/Newton/CG mà MuJoCo hỗ trợ) cần đọc sâu thêm phần "Computation" trong docs chính thức nếu bạn cần cài đặt solver tuỳ chỉnh — bài giảng này chỉ giải thích ở mức khái niệm đủ dùng, không phải một hướng dẫn cài đặt solver.

## ⚙️ Cơ chế hoạt động — từng bước

```
═══════════════ SPRING-DAMPER / PENALTY METHOD ═══════════════

Bước 1: Phát hiện va chạm (collision detection)
         → đo độ xuyên thấu d giữa geom A và geom B

Bước 2: Tính lực phạt (penalty force) trực tiếp bằng công thức
         f_c = -k·d - c·d_dot     (k: độ cứng, c: giảm chấn)

Bước 3: Cộng f_c vào tổng lực tác động lên vật thể
         → tích phân số (Euler/RK4...) để ra vận tốc, vị trí mới

Bước 4: Vật thể "lún" một chút vào bước trước, giờ bị đẩy ngược lại
         → nếu k quá nhỏ: lún sâu, phi thực tế
         → nếu k quá lớn mà Δt không đủ nhỏ: dao động phát nổ (numerical explosion)

  Vấn đề cốt lõi: PHẢI chọn trước k, c, và Δt sao cho khớp nhau
                  → không có "công thức chuẩn" cho mọi tình huống,
                    phải tinh chỉnh (tune) thủ công theo từng bài toán

═══════════════ VELOCITY-STEPPING / CONVEX OPTIMIZATION (MuJoCo) ═══════════════

Bước 1: Phát hiện va chạm (collision detection) — giống hệt cách trên
         → xác định tập các cặp geom đang tiếp xúc/gần tiếp xúc

Bước 2: Xây dựng bài toán tối ưu hoá lồi cho TOÀN BỘ các điểm tiếp xúc
         cùng lúc, với ràng buộc:
           - không xuyên thấu (normal direction)
           - lực nằm trong nón ma sát Coulomb (friction cone)
           - lực chỉ đẩy ra, không hút (unilateral)

Bước 3: Giải bài toán bằng 1 trong 3 thuật toán MuJoCo hỗ trợ
         (PGS / Newton / CG — chọn qua <option solver="...">)
         → ra trực tiếp VẬN TỐC sau va chạm thoả mãn mọi ràng buộc

Bước 4: Áp trực tiếp vận tốc đó cho bước tiếp theo — không có khái niệm
         "lún rồi bật ra" như spring-damper

  Lợi ích cốt lõi: ổn định số ở Δt lớn hơn nhiều, vì không có
                    "tần số dao động riêng" ẩn nào cần né tránh
                    → phù hợp cho model-based control (MPC...)
                      cần mô hình động lực học tin cậy ở Δt thực dụng
```

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số tự chọn để dễ hình dung — không lấy từ 1 mô phỏng cụ thể nào, chỉ minh hoạ trực giác về vấn đề "cứng lò xo vs timestep".)*

Giả sử một khối nặng \(m = 1\) kg rơi xuống chạm sàn, và ta mô hình hoá tiếp xúc bằng spring-damper với hệ số cứng \(k\) và bỏ qua giảm chấn để đơn giản. Tần số dao động riêng của hệ lò xo-khối lượng là:

\[
\omega = \sqrt{k/m}
\]

Quy tắc kinh nghiệm phổ biến cho tích phân số hiển ổn định (không phải luật cứng, chỉ minh hoạ bậc độ lớn): timestep cần thoả \(\Delta t \lesssim \frac{2}{\omega}\) (điều kiện ổn định kiểu Euler hiển cho dao động điều hoà).

- Nếu chọn \(k = 100\) N/m (lò xo "mềm"): \(\omega = \sqrt{100/1} = 10\) rad/s → cần \(\Delta t \lesssim 0.2\) s — rất dễ đạt, nhưng \(k=100\) quá mềm khiến khối nặng 1kg dưới trọng lực (\(mg \approx 9.8\) N) lún tới \(d = mg/k \approx 0.098\) m ≈ 10 cm — phi thực tế cho một "sàn cứng".
- Nếu tăng \(k = 100{,}000\) N/m để giảm lún xuống còn \(d = 9.8/100000 \approx 0.098\) mm (hợp lý hơn): \(\omega = \sqrt{100000/1} \approx 316\) rad/s → cần \(\Delta t \lesssim 2/316 \approx 0.0063\) s. Với robot humanoid có hàng chục điểm tiếp xúc chân/tay xảy ra đồng thời, thường bạn sẽ cần \(k\) lớn hơn nhiều bậc để mô phỏng thực tế (bàn chân cứng gần như không lún) — kéo theo \(\Delta t\) phải nhỏ tương ứng, làm chậm mô phỏng đáng kể hoặc buộc dùng tích phân ẩn phức tạp hơn.

Với velocity-stepping, không có phép tính "\(\omega = \sqrt{k/m}\)" nào cả — bài toán được đặt ra là: "tìm vận tốc pháp tuyến sau va chạm sao cho không xuyên thấu và tổng năng lượng/động lượng thoả các ràng buộc ma sát" — được giải trực tiếp bằng 1 bước tối ưu hoá, không phụ thuộc vào việc chọn một "hệ số cứng" tuỳ ý nào, nên không có đánh đổi \(k\) vs \(\Delta t\) như trên.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Spring-damper / penalty method | Velocity-stepping / convex optimization (MuJoCo) |
|---|---|---|
| Cách tính lực tiếp xúc | Công thức tường minh \(f = -kd - c\dot d\) | Giải bài toán tối ưu lồi (CCP/SOCP) mỗi bước |
| Cần "lún" để sinh lực? | Có — bản chất cơ chế dựa trên độ xuyên thấu | Không — vận tốc sau va chạm được tính trực tiếp |
| Nhạy với timestep | Rất nhạy — phải cân bằng \(k\), \(c\), \(\Delta t\) | Ổn định hơn nhiều ở \(\Delta t\) lớn |
| Cần tinh chỉnh tham số thủ công | Có (k, c) — dễ mất nhiều thời gian tune | Có (dung sai solver, số vòng lặp) nhưng ít nhạy cảm hơn |
| Chi phí tính toán mỗi bước | Thấp (chỉ 1 công thức) nhưng cần nhiều bước (Δt nhỏ) | Cao hơn mỗi bước (giải optimization) nhưng cần ít bước hơn |
| Phù hợp cho model-based control (MPC) | Kém — mô hình động lực học kém tin cậy ở Δt thực dụng | Tốt — đúng mục tiêu thiết kế gốc của MuJoCo |
| Phổ biến ở đâu | Nhiều game engine/biến thể (Bullet cũ, một số chế độ PhysX) | MuJoCo (mặc định), Drake (cũng dùng CCP) |

**Khi nào dùng cái nào:** nếu bạn cần một hiệu ứng thị giác "coi được" cho game và không quan tâm độ chính xác động lực học tuyệt đối, penalty method đơn giản, dễ cài đặt, đủ dùng. Nếu bạn cần huấn luyện policy locomotion cho robot rồi chuyển sang robot thật (nơi hành vi tiếp xúc chân-sàn ảnh hưởng trực tiếp thành-bại), velocity-stepping/optimization-based là lựa chọn đúng — đây chính xác là lý do MuJoCo được chọn rộng rãi cho robot learning.

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "MuJoCo không có 'lún' tí nào — tiếp xúc luôn hoàn toàn cứng tuyệt đối."**
   Vì sao sai: bài toán tối ưu của MuJoCo được mô tả chính thức là "soft, convex" — nghĩa là vẫn cho phép một mức độ "mềm hoá" có kiểm soát (điều chỉnh qua tham số như `solref`/`solimp` trong MJCF) để giữ tính khả vi/ổn định số, không phải ràng buộc cứng tuyệt đối kiểu toán học thuần tuý. Sự khác biệt với spring-damper là MỨC ĐỘ và CÁCH kiểm soát độ "mềm" này — không phải việc có hay không có độ mềm.
   Hiểu đúng: MuJoCo cho phép bạn điều chỉnh độ "cứng cảm nhận được" của tiếp xúc qua tham số solver, nhưng cơ chế nền tảng là giải tối ưu hoá, không phải công thức lò xo tường minh.

2. **Hiểu nhầm: "Contact dynamics chỉ ảnh hưởng tới việc mô phỏng có bị 'nảy lung tung' hay không, không ảnh hưởng gì tới việc huấn luyện RL có hội tụ hay không."**
   Vì sao sai: nếu contact solver không ổn định (dùng penalty method với tham số chưa tune tốt), robot trong RL training có thể nhận reward/observation nhiễu loạn (chân "rung" giả tạo dù đang đứng yên) — điều này trực tiếp làm chậm hoặc phá hỏng quá trình học, vì policy phải học cách "đối phó" với nhiễu vật lý không có thật thay vì học hành vi thật sự hữu ích.
   Hiểu đúng: chất lượng contact solver là một phần của chất lượng dữ liệu huấn luyện, không chỉ là vấn đề thẩm mỹ hình ảnh.

3. **Hiểu nhầm: "Vì velocity-stepping ổn định hơn ở timestep lớn, nên luôn nên chọn timestep lớn nhất có thể để training nhanh nhất."**
   Vì sao sai: timestep quá lớn (dù không "nổ số") vẫn làm giảm độ chính xác vật lý tổng thể (bỏ lỡ chi tiết chuyển động nhanh, va chạm liên tiếp trong 1 bước bị gộp lại không chính xác) — ổn định số không đồng nghĩa với chính xác vật lý. Timestep phải cân bằng giữa tốc độ training và độ trung thực vật lý cần thiết cho bài toán cụ thể (locomotion nhanh cần timestep nhỏ hơn manipulation chậm, ví dụ).
   Hiểu đúng: ổn định số (không nổ) là điều kiện cần, không phải điều kiện đủ để chọn timestep — vẫn cần kiểm tra độ chính xác thực nghiệm.

## 🏗️ Ví dụ minh hoạ trong dự án này

Khi huấn luyện WBC/RL cho robot G1/H1 đi bộ trên MuJoCo hoặc MuJoCo Playground, mỗi bước chân chạm sàn là một sự kiện contact — nếu solver không ổn định, bạn sẽ quan sát thấy robot "rung chân" hoặc "trôi/trượt" bất thường ngay cả khi policy đã học đúng hành vi đứng yên, gây khó khăn khi debug (không biết lỗi do policy hay do vật lý mô phỏng). Đây cũng là lý do trực tiếp README của dự án (mục A) nhấn mạnh MuJoCo phù hợp cho *model-based control* — các bài toán MPC ở `01-whole-body-control/` (ZMP preview control, MPC cho dáng đi) cần một mô hình động lực học tiếp xúc đáng tin cậy để dự đoán trước nhiều bước, điều mà spring-damper khó đảm bảo ở cùng timestep thực dụng.

Khi đánh giá sim-to-real gap ở `07-policy-evaluation/`, sự khác biệt giữa hành vi tiếp xúc mô phỏng và tiếp xúc thật (ma sát bàn chân thật khác ma sát mô hình hoá, độ đàn hồi vật liệu sàn thật...) là một trong những nguồn sai lệch lớn nhất — hiểu cơ chế contact dynamics giúp bạn phân biệt "lỗi do domain randomization chưa đủ" với "lỗi do bản chất mô hình hoá contact khác nhau giữa 2 engine".

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **Cải tiến solver PGS bằng Nesterov momentum (MuJoCo, theo changelog chính thức 2024-2025).** Theo tài liệu MuJoCo Changelog (mujoco.readthedocs.io/en/stable/changelog.html), một bản cập nhật đã thêm cơ chế **Nesterov momentum extrapolation với adaptive gradient restart (kiểu O'Donoghue–Candès)** vào solver PGS (Projected Gauss-Seidel — một trong 3 thuật toán MuJoCo dùng để giải bài toán tối ưu contact), giúp PGS hội tụ nhanh hơn đáng kể — theo changelog, tổng số vòng lặp cần thiết giảm khoảng **2 lần**. Đây là cải tiến thuần thuật toán tối ưu hoá, không thay đổi công thức bài toán gốc (vẫn là CCP/SOCP như mô tả ở trên), chỉ giải nó nhanh hơn.
2. **MuJoCo áp dụng cách tiếp cận "bỏ qua hiệu chỉnh DeSaxcé" cho nón ma sát, quy về CCP tương đương SOCP.** Theo tổng hợp tài liệu kỹ thuật (mujoco.readthedocs.io, mục Computation, và bài tổng hợp trên emergentmind.com), MuJoCo cố tình đơn giản hoá mô hình ma sát Coulomb chuẩn (bỏ qua một số hiệu chỉnh toán học chặt chẽ như DeSaxcé) để bài toán vẫn giữ được tính **lồi** (convex) — đánh đổi một phần độ chính xác lý thuyết tuyệt đối để giữ khả năng giải nhanh, ổn định bằng các thuật toán tối ưu lồi tiêu chuẩn. Đây là một ví dụ cụ thể cho thấy MuJoCo ưu tiên tốc độ + ổn định số hơn độ chính xác vật lý tuyệt đối trong một số chi tiết ma sát — điều cần lưu ý khi đối chiếu sim-to-real.
3. **Newton physics engine (NVIDIA + Google DeepMind + Disney Research, công bố GTC 2025) tích hợp MuJoCo-Warp, tăng tốc contact solver 70-100+ lần trên GPU mà không đổi công thức toán học nền tảng.** Theo thông tin công bố tại GTC 2025 (session S72709, "Announcing Mujoco-Warp and Newton") và các bài tổng hợp báo chí công nghệ (Maginative.com, TechCrunch 18/3/2025, Electronic Specifier), Newton là một physics engine GPU-accelerated, differentiable, mã nguồn mở, xây trên NVIDIA Warp và tích hợp MuJoCo-Warp — báo cáo tăng tốc workload robotics "hơn 70 lần", riêng tác vụ thao tác bằng tay (in-hand manipulation) đạt khoảng 100 lần. Về bản chất toán học của contact dynamics (velocity-stepping/CCP), Newton không thay thế mà **kế thừa và tăng tốc phần cứng** cho đúng công thức MuJoCo đã dùng từ 2012 — xác nhận cách tiếp cận velocity-stepping/optimization-based vẫn là nền tảng chuẩn cho robot learning hiện đại, chỉ thay đổi ở tầng thực thi phần cứng (CPU tuần tự → GPU song song hàng loạt).

## ❓ Câu hỏi tự kiểm tra

1. Vì sao spring-damper cần timestep nhỏ hơn khi hệ số cứng \(k\) tăng, còn velocity-stepping/optimization thì không bị ràng buộc theo cách tương tự?
<details><summary>Đáp án gợi ý</summary>Vì spring-damper là một hệ dao động điều hoà có tần số riêng \(\omega \propto \sqrt{k/m}\) — timestep phải đủ nhỏ để "bắt kịp" dao động đó (ổn định tích phân số hiển). Velocity-stepping không mô hình hoá một quá trình dao động liên tục nào — nó giải trực tiếp trạng thái sau va chạm bằng tối ưu hoá, nên không có "tần số riêng" ẩn cần né tránh.</details>

2. Nếu bạn giảm hệ số cứng \(k\) trong penalty method để tránh "nổ số" ở timestep hiện tại, điều gì xảy ra với độ chân thực vật lý của mô phỏng?
<details><summary>Đáp án gợi ý</summary>Vật thể sẽ lún sâu hơn vào bề mặt tiếp xúc (độ xuyên thấu \(d = mg/k\) tăng khi \(k\) giảm) — mô phỏng trông kém chân thực hơn (chân robot "lún" xuống sàn thấy rõ). Đây chính là đánh đổi cố hữu của penalty method: giảm \(k\) để ổn định số nhưng mất độ chân thực; tăng \(k\) để chân thực hơn nhưng cần \(\Delta t\) nhỏ hơn.</details>

3. Tại sao nói contact dynamics và generalized/Cartesian coordinates là 2 trục thiết kế độc lập của một physics engine?
<details><summary>Đáp án gợi ý</summary>Vì một engine có thể chọn độc lập cho mỗi trục: ví dụ dùng generalized coordinates (đúng cấu trúc khớp) nhưng vẫn dùng spring-damper cho contact (nhiều engine robotics/biomechanics cổ điển làm vậy), hoặc dùng Cartesian coordinates nhưng có contact solver tối ưu hoá tốt. MuJoCo là engine kết hợp cả 2 lựa chọn "hiện đại" (generalized + optimization-based contact) trong cùng hệ thống — đó là đóng góp đặc biệt của nó, không phải vì 2 khái niệm này vốn dĩ phụ thuộc nhau về mặt lý thuyết.</details>

4. MuJoCo hỗ trợ 3 thuật toán giải bài toán tối ưu contact — hãy nêu tên và giải thích ngắn gọn vì sao có nhiều lựa chọn thay vì chỉ 1 thuật toán "tốt nhất".
<details><summary>Đáp án gợi ý</summary>PGS (Projected Gauss-Seidel), Newton, và CG (Conjugate Gradient). Có nhiều lựa chọn vì mỗi thuật toán có đánh đổi khác nhau giữa tốc độ hội tụ, độ chính xác, và chi phí tính toán mỗi vòng lặp — tuỳ bài toán cụ thể (số điểm tiếp xúc, độ cứng vật liệu, yêu cầu tốc độ real-time) mà một thuật toán có thể phù hợp hơn thuật toán khác; không có "one-size-fits-all" cho mọi tình huống contact-rich.</details>

5. Vì sao chất lượng contact solver ảnh hưởng trực tiếp tới sim-to-real gap của một policy locomotion, chứ không chỉ là vấn đề "hình ảnh mô phỏng trông đẹp hay xấu"?
<details><summary>Đáp án gợi ý</summary>Vì policy RL học hành vi dựa trên chuỗi trạng thái/reward mà nó quan sát được trong mô phỏng — nếu hành vi tiếp xúc chân-sàn trong sim khác xa với thật (ma sát, độ nảy, độ "cứng" cảm nhận được), policy sẽ học một chiến lược tối ưu cho VẬT LÝ SIM, không nhất thiết tối ưu cho vật lý thật, dẫn tới thất bại hoặc mất ổn định khi deploy lên robot thật.</details>

6. Newton physics engine (2025) tăng tốc contact solver 70-100+ lần — điều này có làm thay đổi công thức toán học (CCP/SOCP) mà MuJoCo dùng để giải contact hay không? Giải thích.
<details><summary>Đáp án gợi ý</summary>Không — theo thông tin công bố, Newton/MuJoCo-Warp kế thừa đúng công thức velocity-stepping/optimization-based contact của MuJoCo, chỉ thay đổi tầng thực thi (chạy song song hàng loạt trên GPU qua NVIDIA Warp thay vì tuần tự trên CPU). Tốc độ tăng đến từ song song hoá phần cứng, không phải từ một thuật toán toán học mới về bản chất.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể:** Lặp lại ví dụ ở mục "Ví dụ tính tay" nhưng với khối nặng \(m = 5\) kg và mục tiêu độ lún tối đa \(d_{max} = 0.5\) mm dưới trọng lực. Tính \(k\) cần thiết (\(k = mg/d_{max}\)), sau đó tính \(\omega = \sqrt{k/m}\) và timestep tối đa gợi ý \(\Delta t \lesssim 2/\omega\). So sánh con số này với timestep mặc định phổ biến trong ví dụ MJCF (0.002s) — bạn cần \(k\) lớn cỡ nào để 0.002s vẫn ổn định?
2. **Đọc tài liệu thật:** Mở `mujoco.readthedocs.io`, tìm mục "Computation" → phần mô tả `solref`/`solimp` (2 tham số cấu hình "độ mềm" của contact trong MJCF). Đọc và tóm tắt bằng lời của bạn: 2 tham số này kiểm soát điều gì trong bài toán tối ưu hoá contact, và chúng có tương đương trực tiếp với hệ số \(k\), \(c\) của spring-damper hay không (gợi ý: không hoàn toàn tương đương — tìm hiểu vì sao).

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

Khi 2 vật thể chạm nhau trong mô phỏng, engine phải quyết định lực/vận tốc tiếp theo theo 1 trong 2 triết lý: spring-damper/penalty method mô hình hoá va chạm như một lò xo-giảm chấn ảo dựa trên độ xuyên thấu — đơn giản để cài đặt nhưng buộc phải đánh đổi giữa độ chân thực (cần \(k\) lớn) và ổn định số (cần \(\Delta t\) nhỏ khi \(k\) lớn); velocity-stepping/convex optimization (cách MuJoCo chọn) giải trực tiếp một bài toán tối ưu hoá lồi (dạng Cone Complementarity Problem) mỗi bước để tìm ra vận tốc sau va chạm thoả mãn ràng buộc không-xuyên-thấu và ma sát Coulomb, ổn định hơn nhiều ở timestep lớn và phù hợp cho model-based control. Đây là trục thiết kế độc lập với cách engine biểu diễn toạ độ hệ khớp (generalized vs Cartesian — bài giảng riêng), và chất lượng của nó ảnh hưởng trực tiếp tới việc chân robot "đứng vững" hay "rung/nảy" trong huấn luyện RL/WBC, từ đó ảnh hưởng thẳng tới sim-to-real gap. Các phát triển 2024-2026 (cải tiến solver PGS bằng Nesterov momentum, và đặc biệt Newton — engine GPU-accelerated hợp tác NVIDIA/Google DeepMind/Disney) cho thấy công thức toán học nền tảng (velocity-stepping/optimization-based) vẫn được giữ nguyên, chỉ được tăng tốc phần cứng và tinh chỉnh thuật toán giải, khẳng định đây vẫn là hướng tiếp cận chuẩn cho robot learning hiện đại.
