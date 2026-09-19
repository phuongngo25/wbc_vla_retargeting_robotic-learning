# Bài giảng: MJCF — định dạng mô tả robot của MuJoCo

*(Thuộc mảng: Simulation: MuJoCo & Isaac Lab)*

## 🎯 Mục tiêu bài học

- Liệt kê và giải thích được vai trò của 10 element XML cấp cao quan trọng nhất trong MJCF: `<mujoco>`, `<option>`, `<compiler>`, `<asset>`, `<worldbody>`, `<body>`, `<joint>`, `<geom>`, `<actuator>`, `<sensor>`, `<contact>`, `<keyframe>`.
- Tự viết và đọc hiểu được một file MJCF tối giản (vật rơi tự do, con lắc 1 khớp).
- Giải thích được vì sao `<joint>` trong MJCF chính là nơi hiện thực hoá "cộng thêm bậc tự do" (generalized coordinates) đã học ở bài trước, không phải một ràng buộc số học áp đặt lên body tự do.
- Tính tay được số bậc tự do (DoF) tổng của một hệ đơn giản từ khai báo `<joint>`.
- Phân biệt được MJCF với URDF ở mức "cái gì MJCF mô tả được mà URDF không mô tả được trực tiếp".
- Đọc hiểu được cấu trúc tổng quát của một file MJCF robot humanoid thật (G1/H1) dù file dài hàng trăm dòng.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Hai bài trước (Generalized coordinates vs Cartesian, Contact dynamics) giải thích *triết lý* khiến MuJoCo khác các engine khác. MJCF là nơi triết lý đó được **hiện thực hoá thành một file cụ thể mà bạn phải viết/đọc/chỉnh sửa** mỗi khi làm việc với MuJoCo — kể cả khi chỉ dùng GMR (đã học ở `02-motion-retargeting/`) để retarget và xem kết quả qua MuJoCo viewer, file MJCF của robot đích (G1, H1...) chính là thứ định nghĩa robot đó trông như thế nào, có bao nhiêu khớp, giới hạn góc bao nhiêu. Học bài này để không còn coi file `.xml` của robot là một "hộp đen" chỉ tải lên rồi chạy, mà hiểu được từng phần tử bên trong nó ánh xạ trực tiếp tới khái niệm vật lý nào.

## 🧠 Trực giác

### Góc nhìn 1: Bản thiết kế lắp ráp đồ nội thất kiểu module (IKEA), có ghi rõ cả "khớp nối xoay được bao nhiêu độ"

Một bản hướng dẫn lắp tủ IKEA thông thường chỉ ghi "tấm A nối vào tấm B tại vị trí này" (giống URDF — mô tả hình học/kết nối). MJCF giống một bản hướng dẫn lắp ráp **chi tiết hơn nhiều**: không chỉ ghi tấm nào nối tấm nào, mà còn ghi rõ "bản lề tại đây xoay được tối đa 90°" (`range` trong `<joint>`), "vật liệu bề mặt là gỗ, hệ số ma sát X" (`<geom>` + friction), "có một cảm biến đo góc gắn ở đây" (`<sensor>`), và "có một động cơ tự động đóng/mở cánh cửa với lực Y" (`<actuator>`).

**Giới hạn của loại suy này:** đồ nội thất IKEA là vật tĩnh sau khi lắp xong (không "mô phỏng" tiếp); MJCF mô tả một hệ sẽ được **mô phỏng liên tục theo thời gian** (`<option timestep=...>`) — bản thân file MJCF không phải "trạng thái cuối" mà là "luật chơi" cho một quá trình động sau này.

### Góc nhìn 2: Mã nguồn khai báo (declarative code) của một scene game, không phải ảnh chụp scene

MJCF giống mã nguồn HTML/CSS khai báo cấu trúc một trang web (thẻ lồng nhau, mỗi thẻ có thuộc tính) hơn là một bức ảnh chụp kết quả cuối cùng. `<worldbody>` giống thẻ `<body>` gốc của HTML; các `<body>` lồng nhau bên trong giống các `<div>` lồng nhau — vị trí cuối cùng trên màn hình (hoặc trong không gian 3D) được tính ra từ toàn bộ cây lồng nhau đó cộng với "trạng thái động" hiện tại (góc khớp), giống như layout CSS được tính lại mỗi khi nội dung thay đổi.

**Giới hạn của loại suy này:** HTML/CSS không có khái niệm vật lý (trọng lực, va chạm, ma sát) — MJCF phải mô tả thêm toàn bộ các thuộc tính vật lý này (`<option>`, `mass`, `friction`...) mà một ngôn ngữ đánh dấu tài liệu thuần tuý không cần tới.

## 📐 Định nghĩa chính xác

**MJCF (MuJoCo XML Format)** là định dạng XML riêng của MuJoCo, thiết kế để "dễ đọc và dễ chỉnh sửa bằng tay nhất có thể" (human readable and editable) — khác với URDF (chỉ mô tả hình học/kinematic), MJCF cho phép truy cập gần như toàn bộ khả năng tính toán của MuJoCo (solver options, sensor, actuator nâng cao).

**Bảng 10 element cấp cao quan trọng nhất** (đã học từ nguồn — nhắc lại có hệ thống hoá):

| Element | Vai trò |
|---|---|
| `<mujoco>` | Element gốc, bắt buộc, có thể đặt tên model. |
| `<option>` | Cấu hình mô phỏng: timestep, gravity, integrator, solver. |
| `<compiler>` | Cấu hình lúc biên dịch model: hệ toạ độ, đường dẫn asset. |
| `<asset>` | Chứa mesh, texture, material, heightfield dùng trong model. |
| `<worldbody>` | Gốc của cây động học (kinematic tree) — chứa tất cả `<body>`. |
| `<body>` | Một vật thể cứng (rigid body), lồng nhau để tạo chuỗi động học. |
| `<joint>` | Gắn vào 1 body, định nghĩa khớp nối với body cha (hinge/slide/ball/free) — nơi "cộng thêm" bậc tự do. |
| `<geom>` | Hình học va chạm + hiển thị của 1 body. |
| `<actuator>` | Định nghĩa động cơ điều khiển khớp (motor/position/velocity). |
| `<sensor>` | Cảm biến ảo (accelerometer, gyro, joint position sensor...). |
| `<contact>` | Cấu hình cặp tiếp xúc, loại trừ va chạm giữa 2 geom cụ thể. |
| `<keyframe>` | Trạng thái mẫu định sẵn (ví dụ tư thế đứng ban đầu). |

**Bốn loại `<joint>` cơ bản và số DoF mỗi loại cộng thêm:**

```text
hinge  (bản lề, 1 trục xoay)         → +1 DoF
slide  (trượt tịnh tiến, 1 trục)     → +1 DoF
ball   (khớp cầu, xoay tự do 3 trục) → +3 DoF
free   (tự do hoàn toàn 6-DOF)       → +6 DoF (3 tịnh tiến + 3 xoay)
```

Đây chính là "nơi cộng thêm bậc tự do" mà bài giảng "Generalized coordinates vs Cartesian" mô tả ở mức khái niệm — trong MJCF, mỗi `<joint>` là một dòng khai báo cụ thể quyết định chính xác bao nhiêu số (biến trạng thái `qpos`) được thêm vào vector trạng thái tổng của hệ.

## ⚙️ Cơ chế hoạt động — từng bước

```text
┌────────────────────────────────────────────────────────────┐
│ MuJoCo compiler đọc file .xml                                 │
│  1. Đọc <compiler>, <option> → cấu hình toàn cục               │
│  2. Đọc <asset> → nạp mesh/texture/material                    │
│  3. Duyệt <worldbody> theo cây lồng nhau:                       │
│     mỗi <body> → tạo 1 rigid body mới                          │
│     mỗi <joint> trong body đó → CỘNG bậc tự do vào qpos/qvel    │
│     mỗi <geom> → gắn hình dạng va chạm + hiển thị               │
│     mỗi <actuator> → gắn động cơ điều khiển tới 1 joint cụ thể  │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ Kết quả: vector trạng thái toàn hệ                             │
│  qpos = [góc/vị trí mọi joint, theo đúng thứ tự khai báo]       │
│  qvel = [vận tốc tương ứng]                                     │
│  → đây chính là "generalized coordinates" của TOÀN BỘ hệ         │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ Mỗi bước mô phỏng (timestep):                                   │
│  tính lực (gravity, actuator, contact — velocity-stepping)      │
│  tích phân → cập nhật qpos, qvel mới                            │
└──────────────────────────────────────────────────────────────┘
```

Điểm quan trọng cần khớp nối với bài trước: khi bạn khai báo `<joint type="hinge">` bên trong một `<body>`, bạn **không** đang tạo ra một body 6-DOF tự do rồi "khoá" nó lại — bạn đang trực tiếp nói "body này chỉ có đúng 1 con số tự do (góc xoay quanh 1 trục) so với body cha", đúng bản chất "cộng thêm DoF" của generalized coordinates, không phải "ràng buộc số học áp lên 6-DOF mặc định" của cách Cartesian.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Dựa trên đúng ví dụ "con lắc" trong `NOI-DUNG-CHI-TIET.md`, mở rộng thêm để tính DoF)*

Xét file MJCF:

```xml
<mujoco model="con_lac_kep">
  <worldbody>
    <body name="tay_don_1" pos="0 0 1">
      <joint name="khop_1" type="hinge" axis="0 1 0" range="-90 90"/>
      <geom type="capsule" fromto="0 0 0  0 0 -0.5" size="0.02"/>
      <body name="tay_don_2" pos="0 0 -0.5">
        <joint name="khop_2" type="hinge" axis="0 1 0" range="-120 120"/>
        <geom type="capsule" fromto="0 0 0  0 0 -0.3" size="0.02"/>
        <body name="qua_ta" pos="0 0 -0.3">
          <joint name="khop_3" type="free"/>
          <geom type="sphere" size="0.05" mass="1"/>
        </body>
      </body>
    </body>
  </worldbody>
</mujoco>
```

**Bước 1 — đếm số `<joint>` và loại của mỗi joint:**

```text
khop_1: hinge  → +1 DoF
khop_2: hinge  → +1 DoF
khop_3: free   → +6 DoF
```

**Bước 2 — tính tổng số bậc tự do (DoF) của toàn hệ:**

```text
DoF_tổng = 1 + 1 + 6 = 8
```

**Bước 3 — so sánh với cách biểu diễn Cartesian/maximal coordinates** (đã học ở bài trước): nếu dùng cách Cartesian, mỗi trong 3 body (`tay_don_1`, `tay_don_2`, `qua_ta`) sẽ mặc định có 6-DOF tự do:

```text
DoF_Cartesian_thô = 3 body × 6 DoF = 18 DoF
```

rồi 2 khớp hinge sẽ áp ràng buộc số học để "khoá bớt" xuống còn đúng 1 DoF mỗi khớp (mỗi hinge khoá đi 5 trong 6 DoF tương đối giữa hai body):

```text
DoF_Cartesian_sau_ràng_buộc = 18 − (5 khoá bởi khop_1) − (5 khoá bởi khop_2) − (0 khoá bởi khop_3, vì free không khoá gì)
                             = 18 − 5 − 5 − 0 = 8
```

**Ý nghĩa:** cả hai cách biểu diễn đều cho ra đúng **8 DoF vật lý thật** cho hệ này — nhưng cách MJCF/generalized coordinates đạt con số đó bằng cách **cộng trực tiếp** (`1+1+6=8`, không có bước "khoá" nào, không có ràng buộc nào có thể bị vi phạm), còn cách Cartesian đạt cùng con số bằng cách **trừ đi từ một hệ tự do quá mức** (`18 − 5 − 5 = 8`, có 10 ràng buộc số học đang hoạt động, mỗi ràng buộc là một nơi tiềm ẩn "trôi" nếu solver không đủ chính xác) — đây chính là ví dụ số cụ thể hoá lại đúng khác biệt đã học ở bài "Generalized coordinates vs Cartesian".

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | MJCF | URDF (xem bài riêng "MJCF vs URDF vs USD") |
|---|---|---|
| Mô tả kinematic/hình học | Có | Có |
| Mô tả solver/timestep/integrator | Có, native (`<option>`) | Không — cần plugin ngoài (Gazebo) |
| Mô tả actuator chi tiết (motor/position/velocity control) | Có, native (`<actuator>`) | Hạn chế, thường cần `<transmission>` + plugin |
| Mô tả sensor ảo | Có, native (`<sensor>`) | Không native |
| Độ phổ biến làm input gốc từ nhà sản xuất robot | Thấp hơn | Cao nhất (chuẩn ROS) |
| Dễ chỉnh tay | Cao (thiết kế mục tiêu rõ ràng) | Trung bình |
| Dùng trực tiếp bởi | MuJoCo, MuJoCo Playground (MJX) | ROS, nhiều pipeline retargeting (input gốc trước khi convert) |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "thêm một `<joint>` là thêm một ràng buộc giới hạn chuyển động, giống như khoá một cánh cửa".** Vì sao sai: trong MJCF/generalized coordinates, `<joint>` không "khoá" gì — nó là nguồn duy nhất tạo ra bậc tự do chuyển động giữa 2 body. Nếu một `<body>` không có `<joint>` nào bên trong (hoặc joint của body cha), nó bị **hàn cứng (welded)** hoàn toàn với body cha — không di chuyển được chút nào. **Hiểu đúng:** không có `<joint>` = không có chuyển động tương đối; có `<joint>` = có đúng số DoF mà loại joint đó cung cấp, không hơn không kém.
2. **Hiểu nhầm: "`range` trong `<joint>` chỉ là gợi ý hiển thị, không ảnh hưởng vật lý thật".** Vì sao sai: `range` (kèm `limited="true"`) là một ràng buộc vật lý thật được đưa vào bài toán tối ưu hoá contact/constraint mà solver giải mỗi bước (liên hệ bài "Contact dynamics" — MuJoCo giải các ràng buộc này cùng lúc với contact trong một bài toán tối ưu hoá thống nhất) — không phải chỉ để vẽ đẹp trên viewer. **Hiểu đúng:** `range` là giới hạn góc khớp thật, tương đương khái niệm "joint limit clamping" đã học ở `02-motion-retargeting/`, được MuJoCo tự động thực thi trong quá trình mô phỏng, không cần code hậu xử lý riêng.

## 🏗️ Ví dụ minh hoạ trong dự án này

Khi dùng GMR (`02-motion-retargeting/BAI-GIANG-gmr-kien-truc-pipeline.md`) để retarget và xem kết quả, lệnh `vis_robot_motion.py` mở một cửa sổ MuJoCo viewer — cửa sổ đó đang đọc **chính file MJCF của robot đích** (ví dụ `unitree_g1.xml`) để biết: robot có bao nhiêu khớp actuated (khớp nào GMR cần giải IK tới), giới hạn góc mỗi khớp là bao nhiêu (dùng bởi bước joint limit clamping), và hình dạng va chạm (`<geom>`) để hiển thị + phát hiện contact với sàn (dùng bởi bước foot contact stabilization). Nói cách khác: mọi con số `θ_min, θ_max` xuất hiện trong các bài giảng retargeting trước đó đều **lấy trực tiếp từ file MJCF** của robot, không phải hằng số tách biệt được lập trình viên gõ tay ở nơi khác.

```text
File MJCF của G1 (unitree_g1.xml)
        │
        ├──▶ Số khớp actuated, giới hạn góc  → dùng bởi GMR (IK + joint limit clamping)
        ├──▶ <geom> hình dạng va chạm         → dùng bởi foot contact stabilization
        └──▶ Toàn bộ cấu trúc cây <body>       → dùng bởi MuJoCo viewer để hiển thị
```

## 🔥 Cập nhật hiện đại / SOTA gần đây

Vì bản thân định dạng MJCF là một đặc tả kỹ thuật tương đối ổn định (đã tồn tại từ khi MuJoCo ra đời, được duy trì bởi Google DeepMind từ khi mua lại MuJoCo năm 2021), phần lớn "cập nhật" liên quan tới MJCF thực chất là các công cụ/hệ sinh thái xây dựng xung quanh nó:

1. **MJX (MuJoCo XML runtime GPU) đọc trực tiếp cùng định dạng MJCF, không cần một định dạng file riêng.** Đây là điểm thiết kế quan trọng: cùng một file MJCF viết cho MuJoCo CPU gốc có thể tái sử dụng gần như nguyên vẹn cho MuJoCo Playground (chạy trên MJX/GPU, xem bài giảng riêng) — không cần viết lại mô tả robot khi chuyển từ debug trên CPU sang huấn luyện song song trên GPU, một lợi thế thực dụng lớn so với việc phải convert định dạng khi chuyển hệ sinh thái (như trường hợp URDF→USD khi chuyển sang Isaac Lab, xem bài "MJCF vs URDF vs USD").
2. **Đây là kiến thức nền tảng ổn định, chưa tìm thấy thay đổi lớn về bản thân đặc tả XML gần đây** — phần mở rộng thực sự đang diễn ra là ở các *công cụ đọc/ghi/convert* MJCF (URDF→MJCF importer, các thư viện Python thao tác MJCF theo chương trình như `dm_control`), không phải ở cú pháp element cốt lõi đã học ở bài này.

## ❓ Câu hỏi tự kiểm tra

1. Một `<body>` không có `<joint>` nào bên trong thì di chuyển thế nào so với body cha?
   <details><summary>Gợi ý đáp án</summary>Không di chuyển được chút nào — nó bị hàn cứng (welded) hoàn toàn với body cha, vì không có joint nào cộng thêm bậc tự do.</details>
2. Trong ví dụ tính tay, nếu thay `khop_3` từ `free` (6 DoF) thành `ball` (3 DoF), tổng DoF của hệ thay đổi thế nào?
   <details><summary>Gợi ý đáp án</summary>Giảm từ 8 xuống `1+1+3=5` DoF — mất đi 3 DoF tịnh tiến (translation) mà `free` có nhưng `ball` không có.</details>
3. Vì sao `range` trong `<joint>` là một ràng buộc vật lý thật chứ không chỉ là gợi ý hiển thị?
   <details><summary>Gợi ý đáp án</summary>Vì MuJoCo đưa `range` (khi `limited="true"`) vào bài toán tối ưu hoá ràng buộc/contact mà solver giải mỗi bước thời gian, cùng cơ chế với việc giải lực tiếp xúc — không phải xử lý hậu kỳ riêng biệt cho mục đích hiển thị.</details>
4. Vì sao cùng một file MJCF có thể dùng lại cho cả MuJoCo CPU gốc lẫn MuJoCo Playground (MJX/GPU)?
   <details><summary>Gợi ý đáp án</summary>Vì MJX là một runtime đọc cùng định dạng MJCF (không phải định dạng file riêng) — chỉ khác cách MJX thực thi phép tính (vector hoá song song trên GPU thay vì tuần tự trên CPU), không khác cách mô tả robot.</details>
5. Trong pipeline retargeting của dự án, giới hạn góc khớp (`θ_min, θ_max`) dùng trong bước joint limit clamping lấy từ đâu?
   <details><summary>Gợi ý đáp án</summary>Lấy trực tiếp từ thuộc tính `range` trong khai báo `<joint>` của file MJCF robot đích (ví dụ `unitree_g1.xml`), không phải một hằng số riêng lập trình viên tự định nghĩa ở nơi khác.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** viết (trên giấy, không cần chạy) một MJCF mô tả một cánh tay robot 3 khớp: `vai (ball)` → `khuỷu (hinge)` → `cổ_tay (hinge)`. Tính tổng DoF theo cách cộng trực tiếp (generalized coordinates), rồi tính lại theo cách Cartesian (3 body × 6 DoF trừ đi số DoF bị khoá bởi mỗi loại joint) — xác nhận hai cách cho cùng kết quả.
2. **Đọc file thật:** cài GMR (theo `HUONG-DAN-THUC-HANH-RETARGETING.md`), mở file MJCF của `unitree_g1` trong thư mục assets của repo — đếm số `<joint>` loại `hinge`, ghi lại `range` của 3 khớp bất kỳ (ví dụ khớp gối, khớp hông, khớp khuỷu tay), so sánh với số bậc tự do "23–43 khớp tuỳ cấu hình" đã học ở bài "Vì sao không thể copy góc khớp" (`02-motion-retargeting/`).

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

MJCF là định dạng XML riêng của MuJoCo, hiện thực hoá trực tiếp triết lý generalized coordinates: mỗi `<joint>` bên trong một `<body>` không phải một ràng buộc khoá bớt chuyển động mà là nguồn **cộng thêm** bậc tự do (hinge/slide: +1, ball: +3, free: +6) vào vector trạng thái tổng của hệ — như ví dụ tính tay minh hoạ, cách cộng trực tiếp này (1+1+6=8) cho cùng kết quả DoF vật lý thật với cách Cartesian truyền thống (18 trừ 10 ràng buộc = 8), nhưng không có ràng buộc số học nào có thể "trôi". Ngoài `<joint>`, MJCF còn native hỗ trợ solver/timestep (`<option>`), actuator, sensor và contact configuration mà URDF cần plugin ngoài mới có — và vì MJX (runtime GPU của MuJoCo Playground) đọc cùng định dạng này, một file MJCF viết một lần dùng được cả khi debug trên CPU lẫn huấn luyện song song trên GPU sau này.
