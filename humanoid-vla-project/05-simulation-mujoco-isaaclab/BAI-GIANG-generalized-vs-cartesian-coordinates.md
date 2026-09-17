# Bài giảng: Generalized coordinates vs Cartesian coordinates

*(Thuộc mảng: Simulation: MuJoCo & Isaac Lab)*

## 🎯 Mục tiêu bài học

- Giải thích được vì sao mọi physics engine phải chọn một cách biểu diễn trạng thái hệ đa vật thể, và tại sao lựa chọn này quan trọng hơn vẻ ngoài "chi tiết kỹ thuật" của nó.
- Phân biệt được chính xác "generalized/joint coordinates" và "Cartesian/maximal coordinates" — biết cái nào "cộng thêm" DOF, cái nào "loại bỏ" DOF.
- Đếm tay được số bậc tự do (DOF) của một chuỗi động học đơn giản theo cả hai cách biểu diễn.
- Giải thích được vì sao MuJoCo dùng generalized coordinates lại tránh được lỗi "joint drift" mà các engine kiểu Cartesian (Bullet/PhysX/ODE-Gazebo) hay gặp.
- Nhận diện được hệ quả thực hành của lựa chọn này khi huấn luyện RL/WBC cho robot humanoid nhiều khớp.
- Phân biệt được khái niệm này với "contact dynamics" (khái niệm liên quan nhưng khác — có bài giảng riêng), tránh nhầm lẫn hai trục thiết kế độc lập của một physics engine.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Trước khi bạn viết một dòng MJCF nào, hay chạy `pip install mujoco`, có một quyết định kiến trúc đã được đưa ra sẵn cho bạn từ lúc engine được thiết kế: hệ vật thể nhiều khớp của bạn (robot humanoid G1/H1 với 19-29+ DOF) được biểu diễn trong bộ nhớ máy tính bằng con số gì? Đây không phải chi tiết cài đặt vô hại — nó quyết định trực tiếp việc mô phỏng của bạn có "trôi khớp" hay không, có cần timestep siêu nhỏ hay không, và vì sao MuJoCo lại được chọn làm nền tảng cho phần lớn nghiên cứu robot learning hiện nay thay vì các engine game cũ.

Khái niệm này là nền tảng đầu tiên bạn cần hiểu trước khi đọc bất kỳ tài liệu nào khác trong thư mục `05-simulation-mujoco-isaaclab/` — mọi thứ khác (MJCF, MJX, Isaac Lab) đều xây trên lựa chọn biểu diễn trạng thái này. Nó cũng là lý do trực tiếp khiến việc retarget chuyển động người → robot (`02-motion-retargeting/`) trên MuJoCo ít bị sai số động học cộng dồn hơn so với một số pipeline dùng engine kiểu Cartesian.

## 🧠 Trực giác

### Góc nhìn 1: "Con rối dây" (marionette) vs "đám mảnh ghép rời rạc buộc dây cao su"

Hãy tưởng tượng bạn cần mô phỏng một con rối tay có vai-khuỷu tay-cổ tay.

- **Cách generalized coordinates** giống như một con rối dây thật: cấu trúc xương của nó *đã được hàn nối sẵn* thành một chuỗi cứng — vai nối khuỷu tay, khuỷu tay nối cổ tay — và thứ duy nhất bạn "thêm vào" để mô tả tư thế là các góc xoay tại từng khớp. Không có cách nào để "vai tuột khỏi khuỷu tay" vì bản thân cấu trúc dữ liệu không cho phép biểu diễn trạng thái đó — góc khớp chỉ có thể xoay, không thể làm đứt chuỗi.
- **Cách Cartesian/maximal coordinates** giống như bạn có 3 mảnh xương (vai, cẳng tay, cổ tay) rời hoàn toàn, mỗi mảnh trôi tự do trong không gian 3D, và bạn dùng dây cao su (ràng buộc số học) để "cố giữ" chúng dính lại đúng vị trí khớp. Dây cao su đàn hồi — nếu bạn kéo mạnh hoặc nếu "người buộc dây" (solver số) tính không đủ chính xác, các mảnh xương có thể hơi tách rời nhau một chút. Đó chính là "joint drift".

**Giới hạn của phép loại suy này:** con rối dây thật không tính lực/mô-men chính xác theo nghĩa vật lý (đây chỉ là ẩn dụ cấu trúc, không phải ẩn dụ động lực học); và "dây cao su" trong game engine hiện đại thực ra là các ràng buộc được giải bằng thuật toán số học tinh vi (không đơn giản như dây cao su vật lý), độ "trôi" trong thực tế rất nhỏ nếu solver tốt — phép loại suy này phóng đại vấn đề để dễ hình dung, không phải mô tả đúng tỷ lệ sai số thực tế.

### Góc nhìn 2: "Toạ độ địa chỉ nhà" vs "toạ độ GPS tuyệt đối"

Một góc nhìn khác, gần với toán học hơn: hãy nghĩ generalized coordinates giống như việc mô tả vị trí một căn phòng bằng địa chỉ tương đối ("phòng 302, tầng 3, toà B") — vị trí luôn được neo theo cấu trúc phân cấp (toà nhà → tầng → phòng), nên không bao giờ có chuyện "phòng 302 nằm lơ lửng ngoài trời" vì địa chỉ chỉ có nghĩa khi gắn với cấu trúc mẹ. Ngược lại, Cartesian/maximal coordinates giống như việc mô tả mọi căn phòng bằng toạ độ GPS tuyệt đối (kinh độ, vĩ độ, độ cao) độc lập với nhau — bạn phải *thêm luật* ("phòng 302 phải cách sảnh tầng 3 đúng khoảng x mét") để đảm bảo các phòng ghép đúng vào nhau thành một toà nhà hợp lệ; nếu luật đó được thực thi không hoàn hảo, toà nhà "nứt".

**Giới hạn của phép loại suy này:** địa chỉ nhà trong đời thực không có "vận tốc" hay "gia tốc" đi kèm — trong khi generalized coordinates của MuJoCo còn phải theo dõi cả vận tốc suy rộng (\(\dot q\)) cho từng khớp, phức tạp hơn một địa chỉ tĩnh; phép loại suy này chỉ giúp hình dung tính "neo theo cấu trúc cha-con", không giúp hình dung phần động lực học (dynamics) thực sự.

## 📐 Định nghĩa chính xác

Theo cách paper Todorov, Erez, Tassa (2012) — *"MuJoCo: A Physics Engine for Model-Based Control"*, IROS 2012 — trình bày, mọi physics engine mô phỏng hệ đa vật thể (bodies nối bằng joints) phải trả lời câu hỏi: **trạng thái của toàn hệ được biểu diễn bằng vector số nào?**

- **Cartesian / maximal coordinates:** mỗi rigid body \(i\) trong hệ được cấp đầy đủ 6 bậc tự do độc lập trong không gian 3D — 3 tịnh tiến (position) + 3 xoay (orientation), tổng cộng với \(N\) body ta có vector trạng thái kích thước \(6N\) (hoặc 7N nếu dùng quaternion cho orientation). Mỗi joint nối 2 body được biểu diễn như một **ràng buộc đại số (algebraic constraint)** \(g(q) = 0\) áp lên vector trạng thái đầy đủ đó — ví dụ một khớp bản lề (hinge) là ràng buộc buộc 5 trong 6 bậc tự do tương đối giữa 2 body phải bằng 0 (chỉ giữ lại 1 bậc xoay). Vì các ràng buộc này được **giải bằng số** (numerically, thường qua stabilization như Baumgarte hoặc constraint projection), chúng có thể bị vi phạm ở mức nhỏ tại mỗi bước — đây là nguồn gốc "joint drift".
- **Generalized / joint coordinates (MuJoCo):** trạng thái toàn hệ được biểu diễn trực tiếp bằng một vector \(q \in \mathbb{R}^{n}\) các biến khớp (joint angles cho hinge, joint position cho slide, quaternion tương đối cho ball joint, v.v.), với \(n = \) tổng số bậc tự do thực sự của hệ (bằng đúng số DOF sau khi đã áp ràng buộc kinematic tree, không thừa). Không có "ràng buộc joint" nào cần giải riêng — cấu trúc cây động học (kinematic tree, neo từ `<worldbody>`) tự nó **ngầm định (implicit)** đảm bảo mọi cấu hình \(q\) hợp lệ đều tương ứng với một tư thế vật lý khả thi của hệ, không thể vi phạm liên kết khớp bằng cách chọn \(q\) bất kỳ.

Về mặt số học: nếu hệ Cartesian có kích thước trạng thái \(6N\) và phải giải thêm \(m\) phương trình ràng buộc (với \(m\) là tổng số bậc tự do bị joint loại bỏ), thì hệ generalized coordinates biểu diễn đúng \(n = 6N - m\) biến — không thừa, không cần giải ràng buộc khớp riêng. Đóng góp cốt lõi của MuJoCo (theo paper 2012) là kết hợp được cách biểu diễn generalized coordinates chính xác này **với** contact dynamics dựa trên tối ưu hoá lồi trong cùng một hệ thống (chi tiết contact dynamics — xem bài giảng riêng "Contact dynamics: velocity-stepping/convex optimization vs spring-damper").

## ⚙️ Cơ chế hoạt động — từng bước

Hãy xem cách một chuỗi động học 2 khớp (ví dụ vai → khuỷu tay của một cánh tay robot, mỗi khớp là hinge 1-DOF) được biểu diễn theo 2 cách:

```
Chuỗi vật lý thật:  [Thân] --khớp vai (hinge)--> [Cánh tay trên] --khớp khuỷu (hinge)--> [Cẳng tay]

═══════════════════════ CÁCH 1: Cartesian/maximal coordinates ═══════════════════════

  Thân (world, cố định)
      │  (không phải constraint, world luôn tại gốc)
      ▼
  [Cánh tay trên]  ← trạng thái riêng: (x1,y1,z1, quat1)  — 6 DOF thô
      │
      │  RÀNG BUỘC #1 (giải bằng số mỗi bước):
      │  "vị trí điểm nối của Cánh tay trên phải trùng vị trí khớp vai trên Thân"
      │  → nếu solver không hội tụ đủ tốt: khe hở nhỏ xuất hiện (drift)
      ▼
  [Cẳng tay]       ← trạng thái riêng: (x2,y2,z2, quat2)  — 6 DOF thô
      │
      │  RÀNG BUỘC #2 (giải bằng số mỗi bước):
      │  "vị trí điểm nối của Cẳng tay phải trùng vị trí khớp khuỷu trên Cánh tay trên"
      │  → nguồn drift thứ 2, CỘNG DỒN với drift #1
      ▼
  Tổng: 12 biến thô (2 body × 6 DOF) + 2 ràng buộc phải giải mỗi bước
  (mỗi ràng buộc = nguồn sai số tích luỹ tiềm tàng)

═══════════════════════ CÁCH 2: Generalized coordinates (MuJoCo) ═══════════════════════

  q = [ q_vai, q_khuyu ]     ← CHỈ 2 số (2 góc khớp), không hơn không kém

  Vị trí Cẳng tay trong không gian được TÍNH RA (forward kinematics)
  trực tiếp từ q, không phải một biến độc lập cần "ràng buộc giữ dính":

      pos(Cẳng tay) = f(q_vai, q_khuyu)   ← luôn đúng bằng cấu trúc, không cần giải gì thêm

  Tổng: 2 biến (đúng bằng số DOF thật), 0 ràng buộc khớp cần giải mỗi bước
```

Có thể thấy: Cartesian coordinates biểu diễn *thừa* (12 số cho một hệ chỉ có 2 DOF thật) rồi dùng ràng buộc để "cắt bớt" — mỗi ràng buộc là một điểm có thể lỗi. Generalized coordinates biểu diễn *đúng, tối giản* — không có gì để "lỗi" về mặt cấu trúc khớp.

Về mặt thuật toán: MuJoCo và các engine robotics/biomechanics dùng recursive dynamics algorithms (ví dụ họ thuật toán kiểu Articulated-Body Algorithm — ABA — dùng cấu trúc cây để tính động lực học \(O(n)\) theo số khớp, thay vì phải giải một hệ ràng buộc kích thước lớn như cách Cartesian).

### Vì sao generalized coordinates "rẻ" hơn về mặt tính toán khi hệ có nhiều khớp

Một điểm quan trọng ít được nhắc tới trong mô tả cấp khái niệm: lợi ích của generalized coordinates không chỉ nằm ở việc "không trôi khớp" mà còn ở **độ phức tạp tính toán (computational complexity)** của bước tính động lực học (forward dynamics — từ lực/mô-men suy ra gia tốc).

- Với hệ **Cartesian/maximal coordinates**, để tính động lực học đúng, engine phải giải một hệ phương trình tuyến tính có kích thước phụ thuộc vào tổng số ràng buộc \(m\) (ví dụ dùng phương pháp nhân tử Lagrange hoặc Baumgarte stabilization) — với hệ có nhiều ràng buộc lồng nhau (chuỗi động học dài), chi phí này có thể tăng nhanh nếu dùng thuật toán tổng quát (dense solve, \(O(m^3)\) trong trường hợp xấu nhất).
- Với **generalized coordinates**, robotics/biomechanics từ lâu đã có các thuật toán đệ quy khai thác đúng cấu trúc cây (kinematic tree): ví dụ **Articulated-Body Algorithm (ABA)** hoặc **Composite Rigid Body Algorithm (CRBA)** — các thuật toán này tính forward dynamics với chi phí \(O(n)\) (ABA) hoặc \(O(n^2)\)-\(O(n^3)\) tuỳ biến thể (CRBA) theo đúng số khớp \(n\), bằng cách "quét" một lần từ gốc cây ra ngọn rồi một lần ngược lại — không cần giải hệ phương trình ràng buộc tổng quát vì cấu trúc ràng buộc đã "có sẵn" trong cách biểu diễn.

Nói ngắn gọn: generalized coordinates không chỉ đúng hơn về mặt vật lý, mà còn thường **nhanh hơn** khi hệ có cấu trúc cây rõ ràng (đúng như hầu hết robot humanoid) — đây là lý do kép khiến robotics/biomechanics chọn cách này từ trước cả khi MuJoCo ra đời, và MuJoCo chỉ là engine đầu tiên kết hợp thêm được contact dynamics hiện đại vào cùng khung này.

### Trường hợp đặc biệt: ball joint và quaternion trong generalized coordinates

Không phải mọi khớp trong generalized coordinates đều là 1 con số duy nhất. Một khớp cầu (ball joint — ví dụ mô hình hoá khớp vai/hông của humanoid ở mức đơn giản hoá) có 3 DOF xoay tự do, và MuJoCo biểu diễn phần "biến khớp" tương ứng bằng 1 quaternion 4 thành phần (dù chỉ có 3 DOF thật — quaternion dùng 4 số với 1 ràng buộc chuẩn hoá `‖q‖=1` để tránh hiện tượng "gimbal lock" của góc Euler). Đây là điểm dễ gây nhầm lẫn: số phần tử trong vector `qpos` của MuJoCo (dùng để lưu trạng thái) có thể lớn hơn số DOF thật của hệ (`qvel`/`nv` mới đúng bằng số DOF) — vì mỗi ball/free joint "tốn" thêm 1 số dư để biểu diễn quaternion không suy biến. Đây KHÔNG phải là quay lại kiểu Cartesian — ràng buộc chuẩn hoá quaternion là ràng buộc nội tại của phép biểu diễn xoay 3D, tách biệt hoàn toàn với ràng buộc liên kết khớp (joint constraint) mà Cartesian coordinates phải giải.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số tự chọn để dễ hình dung — không lấy từ 1 paper cụ thể, chỉ minh hoạ cách đếm DOF.)*

Xét một cánh tay robot đơn giản hoá gồm 3 rigid body: `Thân` (world, không tính), `CanhTayTren`, `CamTay` — nối bằng 2 khớp hinge 1-DOF (vai, khuỷu tay).

**Đếm theo Cartesian/maximal coordinates:**
- Số body cần cấp DOF tự do (không tính `Thân` vì world cố định): 2 body → mỗi body 6 DOF thô = \(2 \times 6 = 12\) biến.
- Số ràng buộc cần áp: mỗi khớp hinge loại bỏ 5 trong 6 bậc tự do tương đối (chỉ giữ 1 trục xoay) → mỗi khớp cần \(5\) phương trình ràng buộc → 2 khớp = \(2 \times 5 = 10\) phương trình ràng buộc.
- Kiểm tra: DOF thật = biến thô − số ràng buộc = \(12 - 10 = 2\). Khớp với 2 khớp hinge 1-DOF thật. ✅ (đúng số DOF cuối, nhưng phải "đi qua" 12 biến và giải 10 phương trình để tới được đó — đây chính là nơi drift có thể sinh ra nếu 10 phương trình không được giải chính xác tuyệt đối).

**Đếm theo generalized coordinates (MuJoCo):**
- Vector trạng thái \(q = [q_1, q_2]\) — đúng 2 số, một cho mỗi khớp hinge.
- Không có phương trình ràng buộc nào cần giải riêng cho cấu trúc khớp — "10 phương trình ràng buộc" ở trên biến mất hoàn toàn vì cấu trúc cây (`<body>` lồng trong `<body>` của MJCF) đã tự động đảm bảo tính liên kết.
- Kiểm tra: 2 biến, đúng bằng 2 DOF thật, không thừa không thiếu, không cần bước "giải ràng buộc khớp" nào. ✅

**So sánh chi phí:** với robot humanoid thật có ~29 DOF (ví dụ Unitree G1, theo MuJoCo Menagerie) và ~15-20 rigid body, cách Cartesian sẽ cần khoảng \(15 \times 6 = 90\) biến thô và khoảng \(90 - 29 = 61\) phương trình ràng buộc phải giải ổn định mỗi bước mô phỏng — mỗi phương trình là một điểm drift tiềm tàng. Cách generalized coordinates chỉ cần đúng 29 biến, không có ràng buộc khớp nào phải giải. Đây là lý do trực tiếp khiến sai số động học của MuJoCo trên robot nhiều khớp ổn định hơn nhiều so với engine kiểu Cartesian ở cùng độ chính xác solver.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Cartesian / maximal coordinates (Bullet, PhysX cổ điển, ODE/Gazebo) | Generalized / joint coordinates (MuJoCo) |
|---|---|---|
| Số biến trạng thái | Thừa (6-7 DOF/body, không phụ thuộc cấu trúc khớp) | Tối giản (đúng bằng số DOF thật của hệ) |
| Ràng buộc khớp | Tường minh, giải bằng số mỗi bước → có thể vi phạm (drift) | Ngầm định trong cấu trúc biểu diễn → không thể vi phạm về mặt cấu trúc |
| Độ phức tạp tính toán động lực học | Thường \(O(n)\) tới \(O(n^3)\) tuỳ solver ràng buộc, phụ thuộc số ràng buộc | Recursive algorithms thường \(O(n)\) theo số khớp (n nhỏ hơn, không có ràng buộc phụ) |
| Dễ thêm/xoá 1 vật thể rời rạc (không nối khớp) vào scene | Rất tự nhiên — mọi vật thể vốn đã "rời", chỉ cần không thêm ràng buộc | Cần xử lý đặc biệt (ví dụ `<joint type="free">` — bản chất là quay lại 6-DOF Cartesian *cho riêng vật thể đó*, còn hệ có khớp vẫn generalized) |
| Vòng lặp mô phỏng phim/game với vật thể rời rạc số lượng lớn (mảnh vỡ, ragdoll đơn giản) | Phù hợp tự nhiên, engine game tối ưu cho trường hợp này | Không phải use-case chính, dù MuJoCo vẫn hỗ trợ qua free joint |
| Robot nhiều khớp liên hoàn, cần độ chính xác động học cao (humanoid, tay robot công nghiệp) | Dễ "trôi khớp" nếu solver/timestep không đủ tốt | Không có nguồn sai số này từ gốc — phù hợp hơn cho model-based control |
| Use-case lịch sử | Game engine, mô phỏng vật lý giải trí | Robotics, biomechanics (từ trước MuJoCo), điều khiển mô hình chính xác |
| Thuật toán tính động lực học tiêu biểu | Lagrange multiplier / constraint solver tổng quát trên toàn bộ hệ ràng buộc | Recursive: Articulated-Body Algorithm (ABA, \(O(n)\)), Composite Rigid Body Algorithm (CRBA) |
| Biểu diễn xoay 3-DOF (ví dụ khớp cầu vai/hông) | Quaternion/ma trận xoay đầy đủ trên từng body, ràng buộc khớp áp riêng | Quaternion cục bộ trong `qpos` của khớp đó, không cần ràng buộc liên kết riêng |
| Chi phí thêm 1 vật thể KHÔNG nối khớp vào scene (ví dụ 1 quả bóng rơi tự do) | Không đổi bản chất — vốn đã là "mọi thứ đều rời" | Cần khai báo `<joint type="free">` — về bản chất cục bộ quay lại 6-DOF như Cartesian cho riêng vật thể đó |

**Khi nào dùng cái nào:** nếu bạn cần mô phỏng hàng trăm vật thể rời rạc va chạm hỗn loạn (mảnh vỡ trong game), Cartesian/maximal tự nhiên hơn. Nếu bạn cần mô phỏng chính xác động học của một robot nhiều khớp liên hoàn để huấn luyện policy rồi chuyển sang robot thật (sim-to-real), generalized coordinates là lựa chọn đúng — đây chính là lý do gần như toàn bộ hệ sinh thái robot learning hiện đại (MuJoCo, MJX, và cả PhysX trong Isaac Sim khi mô phỏng robot có khớp) đều ưu tiên cách biểu diễn theo khớp cho phần robot, dù dùng maximal coordinates ở tầng thấp cho một số phép tính.

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "Generalized coordinates chỉ là một chi tiết cài đặt nội bộ, không ảnh hưởng gì tới kết quả huấn luyện RL."**
   Vì sao sai: chi tiết này ảnh hưởng trực tiếp tới độ ổn định số và sai số tích luỹ qua kinematic tree — với robot humanoid 20-29+ DOF, sai số cộng dồn qua nhiều khớp (nếu dùng engine kiểu Cartesian với solver yếu) có thể khiến chân robot "trôi" khỏi vị trí lý tưởng đủ để ảnh hưởng reward/quan sát trong RL, và ảnh hưởng trực tiếp độ chính xác khi retarget chuyển động người → robot.
   Hiểu đúng: đây là lựa chọn kiến trúc nền tảng ảnh hưởng tới toàn bộ chất lượng mô phỏng phía sau, không phải chi tiết vô hại.

2. **Hiểu nhầm: "MuJoCo dùng generalized coordinates nên không cần giải bài toán tối ưu/ràng buộc nào cả — nó 'miễn phí' về mặt tính toán so với engine khác."**
   Vì sao sai: generalized coordinates loại bỏ được ràng buộc *khớp* (joint constraint), nhưng KHÔNG loại bỏ được việc phải giải bài toán tối ưu hoá cho **contact** (va chạm) — đây là một trục thiết kế hoàn toàn khác (xem bài giảng riêng "Contact dynamics"). MuJoCo vẫn phải giải một bài toán tối ưu lồi mỗi bước cho phần tiếp xúc, chỉ là không phải cho phần khớp.
   Hiểu đúng: generalized coordinates giải quyết vấn đề "khớp có thể trôi", còn contact dynamics là vấn đề riêng biệt về "va chạm được tính như thế nào" — hai khái niệm độc lập, dễ bị gộp nhầm vì cùng nằm trong 1 paper gốc.

3. **Hiểu nhầm: "Free joint (`<joint type='free'/>`) trong MJCF có nghĩa là MuJoCo quay lại dùng Cartesian coordinates cho vật thể đó."**
   Vì sao gần đúng nhưng dễ hiểu sai phạm vi: đúng là một body gắn free joint có 6 DOF độc lập giống hệt Cartesian — nhưng đây là lựa chọn *có chủ đích* cho MỘT body cụ thể (ví dụ: robot đang "bay" tự do trước khi tiếp đất, hoặc pelvis gốc của humanoid làm điểm neo 6-DOF), không có nghĩa là toàn bộ hệ quay lại Cartesian. Các khớp còn lại trong kinematic tree (ví dụ tất cả khớp gối/khuỷu tay của humanoid) vẫn là generalized coordinates thuần tuý.
   Hiểu đúng: generalized coordinates của MuJoCo là một khung biểu diễn linh hoạt — free joint chỉ là 1 loại joint đặc biệt (thêm đúng 6 DOF ngay tại gốc chuỗi) trong framework generalized đó, không phải một "chế độ khác" của engine.

## 🏗️ Ví dụ minh hoạ trong dự án này

Robot humanoid G1 (Unitree, 29 DOF theo MuJoCo Menagerie) khi được mô phỏng trong MuJoCo được biểu diễn bằng đúng 29 biến khớp `q` (cộng thêm 7 biến nếu pelvis gắn free joint để robot có thể di chuyển tự do trong không gian — 3 vị trí + 4 quaternion). Khi bạn dùng GMR hoặc SOMA-retargeter (`02-motion-retargeting/`) để ánh xạ chuyển động người sang G1 rồi mở kết quả bằng MuJoCo viewer (README mục A, D bước 2), việc kiểm tra "quỹ đạo có khớp với model không" tin cậy được một phần chính vì generalized coordinates đảm bảo không có drift khớp giả tạo làm nhiễu kết quả retargeting — sai lệch bạn thấy (nếu có) phản ánh đúng vấn đề thuật toán IK/retargeting, không phải nhiễu từ physics engine.

Khi huấn luyện WBC/RL trên MuJoCo Playground hoặc Isaac Lab (dùng PhysX, cũng theo mô hình generalized coordinates cho phần robot articulation), 29 giá trị `q` này chính là một phần cốt lõi trong observation vector đưa vào policy — hiểu rõ ý nghĩa của chúng giúp bạn debug được khi policy học sai (ví dụ kiểm tra xem `q` có đúng range vật lý hay không, xem `01-whole-body-control/` để hiểu cách các giá trị này được dùng trong task-space control).

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **MuJoCo-Warp / Newton (2025) — vẫn giữ generalized coordinates nhưng tăng tốc GPU khổng lồ.** Tại GTC 2025, NVIDIA, Google DeepMind và Disney Research công bố hợp tác mã nguồn mở **Newton** — một physics engine GPU-accelerated, differentiable, xây trên NVIDIA Warp, tích hợp MuJoCo-Warp. Theo thông tin công bố (NVIDIA GTC25 session S72709, "Announcing Mujoco-Warp and Newton"), MuJoCo-Warp tăng tốc workload robotics **hơn 70 lần**, với báo cáo riêng cho tác vụ thao tác bằng tay (in-hand manipulation) đạt **100 lần**, và một số nguồn thứ cấp (robolabs.ai, flywing-tech) trích số liệu benchmark humanoid tới 152x trên các tác vụ locomotion cụ thể trên RTX 4090 — con số cụ thể nên được đối chiếu thêm với báo cáo kỹ thuật chính thức khi cần trích dẫn chính xác cho mục đích nghiên cứu. Điểm quan trọng: Newton **không** thay đổi triết lý generalized coordinates + optimization-based contact của MuJoCo gốc, nó chỉ viết lại engine để chạy song song hàng loạt trên GPU — đúng như hướng MJX đã đi trước đó (xem bài giảng riêng "MuJoCo Playground — kiến trúc MJX/JAX"). Nguồn: NVIDIA GTC 2025 session S72709; Maginative.com, "NVIDIA, Google DeepMind, and Disney Research Team Up for Open-Source Physics Engine" (2025); Isaac Lab docs, mục "Newton Physics Integration" (isaac-sim.github.io/IsaacLab).
2. **Isaac Lab 3.0 Beta (đang phát triển) tích hợp Newton làm backend thay thế/bổ sung cho PhysX.** Theo tài liệu Isaac Lab chính thức (mục "Newton Physics Integration", nhánh `develop`), việc tích hợp Newton — vốn dùng chung ý tưởng generalized-coordinates + contact tối ưu hoá với MuJoCo — vào Isaac Lab cho thấy xu hướng hội tụ: cả hệ sinh thái NVIDIA và Google DeepMind đang đồng thuận về cách biểu diễn trạng thái robot bằng generalized coordinates là hướng đi chuẩn cho robot learning quy mô lớn, thay vì tiếp tục dựa hoàn toàn vào maximal coordinates kiểu game engine truyền thống của PhysX cũ. Đây là bằng chứng gián tiếp cho thấy tranh luận "generalized vs Cartesian" trong bài này không chỉ là lịch sử 2012 mà vẫn định hình các quyết định kiến trúc lớn năm 2025.
3. **Genesis simulator (2024-2025) tuyên bố tốc độ vượt trội nhưng KHÔNG thay thế tranh luận về coordinates.** Theo bài viết tổng hợp trên Spheron Blog (2026) và các nguồn thứ cấp khác, Genesis báo cáo tốc độ nhanh hơn 10-80 lần so với Isaac Sim và MuJoCo trên một số workload — tuy nhiên đây chủ yếu là tuyên bố hiệu năng tổng thể (bao gồm cả tối ưu contact/rendering), chưa có phân tích công khai chi tiết so sánh riêng cách Genesis biểu diễn toạ độ hệ đa vật thể so với MuJoCo/PhysX ở mức thuật toán — nên xem đây là điểm cần theo dõi/kiểm chứng thêm hơn là một "cải tiến đã xác nhận" về mặt biểu diễn toạ độ.

## ❓ Câu hỏi tự kiểm tra

1. Một hệ có 4 rigid body nối liên hoàn bằng 4 khớp hinge 1-DOF. Hỏi: theo Cartesian/maximal coordinates cần bao nhiêu biến thô và bao nhiêu phương trình ràng buộc? Theo generalized coordinates cần bao nhiêu biến?
<details><summary>Đáp án gợi ý</summary>Cartesian: 4 body × 6 DOF = 24 biến thô; mỗi khớp hinge loại 5 DOF → 4×5=20 ràng buộc; kiểm tra 24-20=4 DOF thật. Generalized: đúng 4 biến (một góc mỗi khớp).</details>

2. Vì sao "joint drift" gần như không thể xảy ra về mặt cấu trúc trong generalized coordinates, trong khi vẫn có thể xảy ra trong Cartesian coordinates dù solver rất tốt?
<details><summary>Đáp án gợi ý</summary>Vì trong generalized coordinates, ràng buộc khớp là *ngầm định trong chính cách biểu diễn* — không tồn tại phương trình số nào cần giải cho cấu trúc khớp nên không có gì để "giải sai". Trong Cartesian, ràng buộc là phương trình số tường minh, luôn có sai số dư (residual) dù nhỏ tới đâu, do bản chất số học của việc giải phương trình lặp.</details>

3. Một `<joint type="free">` gắn vào pelvis của robot humanoid trong MJCF có ý nghĩa gì về mặt DOF, và nó có mâu thuẫn với việc MuJoCo dùng generalized coordinates không?
<details><summary>Đáp án gợi ý</summary>Free joint thêm đúng 6 DOF (3 vị trí + 3 xoay, thường biểu diễn bằng quaternion) cho pelvis để nó di chuyển tự do trong không gian. Không mâu thuẫn — đây vẫn là generalized coordinates, chỉ là loại joint đặc biệt cho phép 6 DOF tại 1 điểm neo của cây động học, các khớp con (gối, khuỷu tay...) vẫn dùng ít DOF hơn theo đúng cấu trúc khớp thật.</details>

4. Tại sao nói "generalized coordinates" và "contact dynamics" là hai trục thiết kế độc lập, dù cùng được giải quyết trong cùng 1 paper MuJoCo 2012?
<details><summary>Đáp án gợi ý</summary>Vì chúng trả lời 2 câu hỏi khác nhau: (1) biểu diễn trạng thái hệ khớp như thế nào (generalized vs Cartesian) và (2) tính lực/vận tốc khi 2 vật va chạm như thế nào (velocity-stepping/optimization vs spring-damper). Một engine về lý thuyết có thể chọn generalized coordinates nhưng vẫn dùng spring-damper cho contact (nhiều engine robotics/biomechanics cổ điển làm vậy) — MuJoCo là engine đầu tiên kết hợp cả hai lựa chọn "hiện đại" trong cùng hệ thống.</details>

5. Nếu bạn phải thiết kế một physics engine chỉ để mô phỏng 10.000 mảnh gạch vỡ rơi tự do trong một game phá huỷ công trình (không nối khớp với nhau), bạn sẽ chọn generalized hay Cartesian coordinates? Vì sao?
<details><summary>Đáp án gợi ý</summary>Cartesian/maximal — vì không có cấu trúc khớp nào cần biểu diễn ngầm định (mỗi mảnh gạch độc lập hoàn toàn), generalized coordinates không mang lại lợi ích gì trong trường hợp này và thêm phức tạp không cần thiết. Đây đúng là lý do game engine truyền thống ưu tiên Cartesian.</details>

6. MuJoCo-Warp/Newton (2025) tăng tốc mô phỏng hàng chục tới hàng trăm lần bằng cách nào, và điều đó có thay đổi việc MuJoCo vẫn dùng generalized coordinates hay không?
<details><summary>Đáp án gợi ý</summary>Tăng tốc chủ yếu đến từ việc chạy song song hàng loạt (batched, GPU-native) qua NVIDIA Warp — không phải từ việc đổi cách biểu diễn toạ độ. Generalized coordinates vẫn được giữ nguyên làm nền tảng; đây là ví dụ cho thấy lựa chọn kiến trúc "biểu diễn toạ độ" và "tối ưu tốc độ phần cứng" là hai lớp vấn đề tách biệt.</details>

## 📝 Bài tập thực hành

1. **Đếm tay:** Vẽ một chuỗi động học tưởng tượng gồm 6 rigid body nối liên hoàn, trong đó 3 khớp là hinge (1-DOF) và 2 khớp là ball joint (3-DOF, ví dụ khớp vai/hông). Tính: (a) số biến thô theo Cartesian/maximal coordinates, (b) số phương trình ràng buộc cần giải, (c) số biến theo generalized coordinates. Kiểm tra chéo bằng công thức DOF thật = biến thô − số ràng buộc.
2. **Đọc code thật:** Tải một file MJCF humanoid từ MuJoCo Menagerie (github.com/google-deepmind/mujoco_menagerie, thư mục `unitree_g1` hoặc `unitree_h1`), đếm số thẻ `<joint>` xuất hiện trong file (không tính `<geom>`/`<site>`), cộng với 6 hoặc 7 nếu có `<joint type="free">` ở gốc, rồi so sánh tổng đó với con số DOF chính thức được ghi trong README của Menagerie (19 DOF cho H1, 29 DOF cho G1) để tự kiểm chứng cách đếm bạn vừa học.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

Mọi physics engine phải chọn cách biểu diễn trạng thái một hệ nhiều vật thể nối khớp: hoặc cấp dư thừa 6 DOF cho từng vật thể rồi dùng ràng buộc số học để "khoá" chúng lại đúng theo joint (Cartesian/maximal coordinates — dễ "trôi khớp" nếu solver không hoàn hảo, kiểu Bullet/PhysX cổ điển/ODE-Gazebo), hoặc biểu diễn trực tiếp bằng đúng số biến khớp cần thiết, khiến ràng buộc liên kết trở thành ngầm định trong chính cấu trúc dữ liệu, không thể vi phạm (generalized/joint coordinates — cách MuJoCo chọn, kế thừa truyền thống recursive dynamics của robotics/biomechanics). Với robot humanoid 20-29+ DOF, lựa chọn này quyết định trực tiếp việc sai số động học có cộng dồn hay không qua toàn bộ kinematic tree — ảnh hưởng thẳng tới độ tin cậy khi retarget chuyển động và huấn luyện policy. Đây là một trục thiết kế hoàn toàn tách biệt với cách engine tính contact (va chạm) — MuJoCo là engine đầu tiên kết hợp cả generalized coordinates chính xác lẫn contact dựa trên tối ưu hoá lồi, và ngay cả các phát triển GPU-tăng tốc mới nhất (MJX, MuJoCo-Warp, Newton 2025) cũng chỉ thay đổi tốc độ thực thi, không thay đổi triết lý biểu diễn toạ độ nền tảng này.
