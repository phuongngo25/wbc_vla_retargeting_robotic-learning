# Bài giảng: So sánh MJCF vs URDF vs USD

*(Thuộc mảng: Simulation: MuJoCo & Isaac Lab)*

## 🎯 Mục tiêu bài học

- Phân biệt được chính xác 3 định dạng theo 5 tiêu chí: ai tạo ra, kiểu định dạng, trọng tâm thiết kế, có mô tả contact/solver không, dùng ở đâu.
- Giải thích được vì sao URDF "thiếu" khái niệm actuator/solver không phải là lỗi thiết kế mà là hệ quả của mục tiêu thiết kế khác (ROS, mô tả kinematic chuẩn hoá).
- Mô tả được đúng hướng chuyển đổi thực dụng nhất trong pipeline dự án: URDF (gốc từ nhà sản xuất) → MJCF (debug MuJoCo) và URDF → USD (train Isaac Lab), không phải chuyển đổi tuỳ tiện giữa mọi cặp.
- Tính tay được một ví dụ minh hoạ về mất mát thông tin khi convert ngược MJCF → URDF.
- Nêu được vì sao USD không phải "chỉ một định dạng robot khác" mà là một hệ sinh thái scene-graph tổng quát hơn nhiều.
- Liên hệ được bảng so sánh này với toàn bộ các bài giảng trước trong thư mục (MJCF, Isaac Gym→Isaac Lab).

## 🧭 Vì sao cần học cái này? (bối cảnh)

Bạn đã học MJCF chi tiết (bài trước) và biết Isaac Lab dùng USD (bài "Isaac Gym→Isaac Lab"). Nhưng trong thực tế, **robot thật hiếm khi đến tay bạn ở dạng MJCF hay USD ngay từ đầu** — nhà sản xuất (Unitree, Booster...) gần như luôn cấp URDF trước, vì đây là chuẩn ROS phổ biến nhất. Học bài này để hiểu chính xác vì sao cần 3 định dạng khác nhau thay vì chỉ 1, và bước "convert định dạng" nào là bắt buộc, đường nào tổn thất thông tin, đường nào không — kiến thức thực dụng trực tiếp mỗi khi bạn chuyển một robot mới từ giai đoạn "debug trên MuJoCo" sang "train chính thức trên Isaac Lab".

## 🧠 Trực giác

### Góc nhìn 1: Bản vẽ kỹ thuật cơ khí (URDF) so với bản thiết kế game vật lý (MJCF) so với bản dựng phim hoạt hình Pixar (USD)

URDF giống một **bản vẽ kỹ thuật cơ khí** — đủ để biết chi tiết nào nối chi tiết nào, kích thước bao nhiêu, dùng cho việc lắp ráp/hiểu cấu trúc — nhưng không nói robot sẽ "diễn xuất" thế nào trong một cảnh phim. MJCF giống một **bản thiết kế cho game vật lý** — không chỉ có cấu trúc mà còn có luật chơi (solver, timestep, actuator lực bao nhiêu) để mô phỏng hành vi động. USD giống **bản dựng phim hoạt hình Pixar** — mô tả toàn bộ một cảnh phức tạp (ánh sáng, vật liệu, nhiều nhân vật/vật thể tương tác), trong đó một robot chỉ là một "diễn viên" trong cảnh lớn hơn nhiều.

**Giới hạn của loại suy này:** ba loại bản vẽ trong đời thực (cơ khí, game, phim) thường do ba đội ngũ hoàn toàn tách biệt tạo ra, ít khi cần chuyển đổi qua lại; trong khi robot học thực tế **bắt buộc phải chuyển đổi** giữa cả ba định dạng này liên tục trong cùng một pipeline (URDF gốc → MJCF để debug → USD để train) — mức độ cần "giao tiếp qua lại" giữa 3 định dạng này cao hơn nhiều so với 3 loại bản vẽ trong loại suy.

### Góc nhìn 2: Ngôn ngữ mẹ đẻ, ngôn ngữ chuyên ngành, và ngôn ngữ quốc tế

URDF giống **ngôn ngữ mẹ đẻ** của cộng đồng robot học (ROS) — ai làm robot cũng biết, đơn giản, phổ cập. MJCF giống một **ngôn ngữ chuyên ngành hẹp** (thuật ngữ y khoa, luật) — chính xác và giàu thông tin hơn nhiều trong lĩnh vực của nó (mô phỏng vật lý), nhưng không phổ biến ngoài phạm vi MuJoCo. USD giống một **ngôn ngữ quốc tế được thiết kế cho giao tiếp đa ngành** (như tiếng Anh trong khoa học) — không chuyên sâu về robot học như MJCF, nhưng đủ tổng quát để "nói chuyện" với nhiều hệ thống khác (rendering, animation, vật lý) trong cùng một hệ sinh thái lớn (Omniverse).

**Giới hạn của loại suy này:** ngôn ngữ tự nhiên có thể dịch qua lại tương đối trọn vẹn ý nghĩa; chuyển đổi giữa URDF/MJCF/USD có **mất mát thông tin không đối xứng** (một số hướng dễ, một số hướng khó/mất chi tiết) — không giống việc dịch ngôn ngữ tự nhiên vốn thường được coi là "hai chiều tương đối cân bằng".

## 📐 Định nghĩa chính xác

| | **URDF** | **MJCF** | **USD** |
|---|---|---|---|
| Ai tạo ra | Cộng đồng ROS (Willow Garage) | DeepMind/Roboti (MuJoCo) | Pixar, sau NVIDIA mở rộng cho robotics/Omniverse |
| Định dạng | XML | XML | Định dạng scene-graph riêng (`.usd`/`.usda`/`.usdc`), không phải XML thuần |
| Trọng tâm thiết kế | Mô tả **kinematic + hình học** robot đơn giản, dễ đọc, chuẩn hoá cho ROS | Mô tả robot **kèm chi tiết vật lý mô phỏng** (solver, actuator, contact params) | Mô tả **toàn bộ scene 3D phức tạp** (vật liệu, ánh sáng, animation, nhiều loại asset), robot chỉ là 1 use-case |
| Có mô tả contact/solver chi tiết? | Không (cần plugin ngoài, ví dụ Gazebo) | Có, native | Có, qua schema PhysX/USD Physics riêng |
| Dùng ở đâu trong dự án này | Robot thật thường được Unitree cấp URDF gốc; nhiều pipeline retargeting (`02-motion-retargeting/`) dùng URDF làm input chuẩn | MuJoCo/MuJoCo Playground | Isaac Sim/Isaac Lab |
| Chuyển đổi | URDF → MJCF: `compiler` của MuJoCo đọc trực tiếp URDF (hoặc `urdf2mjcf`); MJCF → URDF khó hơn (mất chi tiết đặc thù). URDF → USD: Isaac Sim URDF Importer | | |

**Ý nghĩa thực hành:** dữ liệu chuyển động sau retarget (`02-motion-retargeting/`) thường gắn với 1 mô hình robot ở URDF (từ nhà sản xuất) hoặc MJCF (nếu tool retargeting dùng MuJoCo làm viewer, như GMR). Khi chuyển từ "debug trên MuJoCo" sang "train chính thức trên Isaac Lab", bước bắt buộc là **convert model robot sang USD** — dùng URDF Importer của Isaac Sim (import trực tiếp từ URDF gốc, ít mất thông tin hơn convert ngược từ MJCF).

## ⚙️ Cơ chế hoạt động — từng bước

```text
┌────────────────────────────────────────────────────────────┐
│ NGUỒN GỐC: Nhà sản xuất robot (Unitree...) cấp URDF gốc       │
│  (kinematic tree + hình học + mass/inertia cơ bản, chuẩn ROS) │
└─────────────┬─────────────────────────────┬────────────────────┘
              │                             │
   (nhánh A: debug nhẹ)          (nhánh B: train chính thức)
              ▼                             ▼
┌──────────────────────────┐   ┌──────────────────────────────┐
│ URDF → MJCF                │   │ URDF → USD                    │
│ (MuJoCo compiler / urdf2mjcf)│   │ (Isaac Sim URDF Importer)     │
│ → thêm solver/timestep/       │   │ → giữ physics fidelity,        │
│   actuator/sensor thủ công    │   │   joint limits, ROS topic      │
│   (không tự động suy ra hết)  │   │   mapping (theo tài liệu chính │
│                                │   │   thức, độ chính xác cao)      │
└──────────────┬─────────────────┘   └──────────────┬─────────────────┘
              ▼                                     ▼
     MuJoCo / MuJoCo Playground              Isaac Sim / Isaac Lab
     (debug, chạy nhẹ, không cần              (train GPU-parallel quy mô
      GPU NVIDIA mạnh)                          lớn, WBC/teleoperation)
```

**Vì sao KHÔNG khuyến nghị đường MJCF → URDF hoặc MJCF → USD trực tiếp:** MJCF chứa nhiều thông tin đặc thù MuJoCo (cách khai báo `<option>` solver, cấu trúc `<actuator>` riêng) không có khái niệm tương đương trực tiếp trong URDF/USD — convert theo hướng này thường **mất thông tin** hoặc cần ánh xạ thủ công phức tạp, ngược với hướng URDF→(MJCF hoặc USD) vốn đi từ định dạng "tối giản hơn" sang định dạng "giàu thông tin hơn" (chỉ cần *thêm* thông tin còn thiếu, không cần *loại bỏ* thông tin đặc thù không tương thích).

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số tự chọn để định lượng hoá "mất mát thông tin" khi convert — không phải phép đo chính thức từ bất kỳ công cụ cụ thể nào)*

Giả sử một file MJCF của một khớp gối robot có các thuộc tính sau (đã học ở bài MJCF):

```text
<joint name="gối" type="hinge" range="-150 0"
       damping="0.5" armature="0.01" frictionloss="0.2"/>
<actuator><motor joint="gối" gear="50" ctrlrange="-40 40"/></actuator>
```

Giả sử ta thử "convert ngược" file này thành URDF. URDF hỗ trợ trực tiếp:

```text
<joint type="revolute"> <limit lower="-2.618" upper="0" .../> ... </joint>
```

Đếm số thuộc tính **có tương đương trực tiếp** trong URDF chuẩn so với **không có tương đương chuẩn** (cần plugin/mở rộng riêng, ví dụ Gazebo plugin):

```text
Có tương đương trực tiếp trong URDF:
  - type (hinge → revolute)       : 1
  - range/limit (góc giới hạn)     : 1
  Tổng: 2/6 thuộc tính

Không có tương đương chuẩn (cần plugin ngoài hoặc bị bỏ qua):
  - damping (giảm chấn khớp)       : 1
  - armature (quán tính rotor)      : 1
  - frictionloss (ma sát khớp)      : 1
  - actuator gear/ctrlrange (tỷ số truyền động, giới hạn điều khiển) : 1
  Tổng: 4/6 thuộc tính
```

**Tỷ lệ thông tin "mất" khi convert MJCF → URDF (không dùng plugin bổ sung):**

```text
tỷ lệ mất ≈ 4/6 ≈ 66.7%
```

**So sánh chiều ngược lại (URDF → MJCF):** URDF chuẩn chỉ có `type`, `limit` (2 thuộc tính trong ví dụ) — khi convert sang MJCF, công cụ (`urdf2mjcf` hoặc MuJoCo `compiler`) chỉ cần **điền giá trị mặc định hợp lý** cho 4 thuộc tính còn thiếu (damping, armature, frictionloss, actuator params) — không có thông tin nào từ URDF gốc bị "vứt bỏ", chỉ có thông tin **được bổ sung thêm** (dù giá trị mặc định có thể cần tinh chỉnh thủ công sau đó để khớp robot thật). Đây chính là lý do định lượng cho khuyến nghị "URDF → MJCF/USD dễ hơn nhiều so với chiều ngược lại".

## 🔍 So sánh với phương pháp/khái niệm liên quan

*(Bảng chính đã trình bày đầy đủ ở mục Định nghĩa chính xác — dưới đây là góc so sánh bổ sung theo "luồng công việc điển hình")*

| Giai đoạn pipeline | Định dạng phù hợp nhất | Vì sao |
|---|---|---|
| Nhận robot mới từ nhà sản xuất | URDF | Chuẩn phổ biến nhất, ROS hỗ trợ sẵn |
| Debug nhanh, xem retarget bằng MuJoCo | MJCF (convert từ URDF) | Nhẹ, không cần GPU mạnh, đã học ở bài trước |
| Train RL quy mô lớn trên GPU NVIDIA | USD (convert từ URDF) | Isaac Lab yêu cầu USD, cần GPU-parallel training |
| Chia sẻ scene phức tạp (nhiều robot + vật thể + ánh sáng) | USD | Chỉ USD mô tả được scene tổng quát ở mức này |
| Triển khai lên robot thật qua ROS2 | URDF | ROS2 (đã cài theo `HUONG-DAN-THUC-HANH-RETARGETING.md`) dùng URDF làm chuẩn mô tả robot |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "URDF thiếu actuator/solver là do định dạng cũ/lạc hậu, cần thay thế hoàn toàn bằng MJCF/USD".** Vì sao sai: URDF được thiết kế có chủ đích để mô tả **kinematic + hình học** một cách tối giản, chuẩn hoá cho hệ sinh thái ROS — việc "thiếu" solver/actuator chi tiết là hệ quả của mục tiêu thiết kế (đơn giản, phổ cập, tách biệt vật lý mô phỏng khỏi mô tả robot), không phải một khiếm khuyết cần sửa. Rất nhiều hệ thống (bao gồm ROS2 điều khiển robot thật) vẫn dùng URDF hiệu quả chính vì tính tối giản này. **Hiểu đúng:** ba định dạng phục vụ ba mục đích khác nhau trong cùng một pipeline, không phải ba phiên bản "tốt dần lên" của cùng một ý tưởng — URDF vẫn là lựa chọn đúng cho vai trò của nó (giao tiếp với ROS2, chuẩn hoá từ nhà sản xuất).
2. **Hiểu nhầm: "USD chỉ là 'MJCF phiên bản NVIDIA', làm cùng một việc".** Vì sao sai: MJCF được thiết kế *chuyên biệt* cho việc mô tả robot/vật thể để mô phỏng vật lý trong MuJoCo; USD là một **định dạng scene-graph tổng quát** ra đời từ ngành công nghiệp phim hoạt hình (Pixar), mô tả được vật liệu PBR, ánh sáng, animation, nhiều loại asset — robot chỉ là một trong rất nhiều loại "asset" mà USD có thể mô tả, thông qua các schema mở rộng riêng (USD Physics) cho vật lý. **Hiểu đúng:** USD rộng hơn nhiều so với phạm vi "mô tả robot để mô phỏng" mà MJCF/URDF tập trung vào — đây là lý do Isaac Sim/Omniverse chọn USD làm nền tảng (cần mô tả scene phức tạp hơn robot đơn thuần), không phải vì USD "giỏi mô tả robot hơn" MJCF.

## 🏗️ Ví dụ minh hoạ trong dự án này

```text
Unitree cấp URDF gốc cho G1
        │
        ├──▶ (nhánh debug) MuJoCo compiler → MJCF
        │       │
        │       ▼
        │    GMR dùng MJCF này làm viewer (02-motion-retargeting/,
        │    BAI-GIANG-gmr-kien-truc-pipeline.md — giới hạn góc/DoF
        │    lấy từ chính file MJCF này)
        │
        └──▶ (nhánh train chính thức) Isaac Sim URDF Importer → USD
                │
                ▼
             Isaac Lab huấn luyện RL GPU-parallel
             (05-simulation-mujoco-isaaclab/BAI-GIANG-isaac-gym-den-isaac-lab.md)
                │
                ▼
             Deploy lên robot thật qua ROS2
             (dùng lại chính URDF gốc ban đầu — vòng tròn khép kín)
```

Điểm mấu chốt: **URDF gốc chỉ cần tồn tại một lần**, mọi nhánh (MJCF cho debug, USD cho train) đều xuất phát từ đúng file URDF đó — tránh tình trạng "hai bản mô tả robot không đồng bộ" (ví dụ MJCF chỉnh tay một chỗ, USD chỉnh tay chỗ khác, dẫn tới hai mô phỏng chạy trên hai robot "hơi khác nhau" mà không ai nhận ra).

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **NVIDIA đang dẫn đầu nỗ lực chuẩn hoá USD cho robotics qua roadmap "OpenUSD for Robotics".** Theo NVIDIA Technical Blog, nỗ lực này bao gồm việc "mapping data models từ các định dạng robotics (URDF, MJCF, SDFormat) sang OpenUSD" và đề xuất **schema robot chính thức** để lấp các khoảng trống khái niệm còn thiếu — nghĩa là bài toán "convert qua lại giữa URDF/MJCF/USD" mà bài giảng này mô tả đang được chuẩn hoá dần ở cấp độ ngành, không chỉ dựa vào từng công cụ importer riêng lẻ như hiện tại. [NVIDIA Technical Blog](https://developer.nvidia.com/blog/using-openusd-for-modular-and-scalable-robotic-simulation-and-development/)
2. **Isaac Sim URDF Importer hiện được mô tả là giữ độ chính xác cao khi convert.** Theo tài liệu chính thức, pipeline import URDF/SDF sang USD của Isaac Sim "strictly preserve physics fidelity, joint limits, và ROS topic mappings" — ánh xạ chính xác các tham số vật lý (khối lượng, tensor quán tính, mesh va chạm, hệ số ma sát khớp) sang đúng thuộc tính PhysX tương ứng. Đây là bằng chứng cụ thể cho khuyến nghị "URDF → USD qua Isaac Sim URDF Importer là đường ít mất thông tin nhất" đã nêu ở mục Cơ chế hoạt động.
3. **Isaac Sim 5.0 nâng cấp lên Omniverse Kit SDK 107 và OpenUSD 24.05**, cải thiện mô phỏng cảm biến và tương thích nhị phân (binary compatibility) — cho thấy bản thân USD (qua các phiên bản OpenUSD) cũng đang tiến hoá nhanh, không phải một chuẩn tĩnh — người dùng cần chú ý version OpenUSD khi làm việc với các công cụ khác nhau trong cùng pipeline để tránh lỗi tương thích.

## ❓ Câu hỏi tự kiểm tra

1. Vì sao URDF "thiếu" khái niệm actuator/solver không nên coi là một khiếm khuyết cần sửa?
   <details><summary>Gợi ý đáp án</summary>Vì URDF được thiết kế có chủ đích để mô tả kinematic + hình học tối giản, chuẩn hoá cho ROS — tách biệt mô tả robot khỏi chi tiết mô phỏng vật lý là một lựa chọn thiết kế, không phải lỗi; nhiều ứng dụng (như điều khiển robot thật qua ROS2) không cần thông tin đó.</details>
2. Trong ví dụ tính tay, vì sao chiều URDF → MJCF được coi là "dễ" hơn chiều MJCF → URDF?
   <details><summary>Gợi ý đáp án</summary>Vì URDF → MJCF chỉ cần BỔ SUNG thêm giá trị mặc định cho các thuộc tính MJCF-specific còn thiếu (không mất thông tin nào từ URDF gốc), trong khi MJCF → URDF phải LOẠI BỎ các thuộc tính không có tương đương chuẩn trong URDF (ví dụ damping, armature, frictionloss, actuator params — trong ví dụ chiếm tới 66.7%).</details>
3. Vì sao USD không nên được hiểu đơn giản là "MJCF phiên bản của NVIDIA"?
   <details><summary>Gợi ý đáp án</summary>Vì USD là một định dạng scene-graph tổng quát (từ ngành phim hoạt hình Pixar) mô tả được vật liệu, ánh sáng, animation, nhiều loại asset — robot chỉ là một trong nhiều loại asset USD mô tả được qua schema mở rộng (USD Physics), khác hẳn MJCF vốn được thiết kế chuyên biệt chỉ cho mô phỏng vật lý robot/vật thể trong MuJoCo.</details>
4. Trong pipeline dự án, tại sao nên luôn xuất phát từ đúng một file URDF gốc cho cả nhánh debug (MJCF) và nhánh train (USD), thay vì chỉnh tay riêng từng định dạng?
   <details><summary>Gợi ý đáp án</summary>Để tránh tình trạng hai bản mô tả robot không đồng bộ (một bên chỉnh tay MJCF, một bên chỉnh tay USD) dẫn tới hai mô phỏng thực chất chạy trên hai robot "hơi khác nhau" mà không ai nhận ra — giữ một nguồn sự thật duy nhất (URDF gốc) và luôn convert lại từ đó khi cần.</details>
5. Nỗ lực "OpenUSD for Robotics" của NVIDIA đang giải quyết vấn đề gì mà bài giảng này mô tả?
   <details><summary>Gợi ý đáp án</summary>Đang chuẩn hoá việc mapping dữ liệu giữa URDF/MJCF/SDFormat và OpenUSD ở cấp độ ngành (đề xuất schema chính thức), thay vì để mỗi công cụ tự viết importer/converter riêng lẻ như hiện tại — hướng tới giảm mất mát/không nhất quán thông tin khi chuyển đổi giữa các định dạng.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** với một khớp khuỷu tay có 8 thuộc tính MJCF (`type`, `range`, `damping`, `armature`, `frictionloss`, `stiffness`, actuator `gear`, actuator `ctrlrange`), giả sử URDF chuẩn chỉ hỗ trợ trực tiếp 2 thuộc tính đầu (`type`, `range`) — tính tỷ lệ % thông tin "mất" khi convert MJCF → URDF, so sánh với tỷ lệ 66.7% đã tính trong bài.
2. **Đọc tài liệu/thử nghiệm thật:** nếu đã cài GMR (theo `HUONG-DAN-THUC-HANH-RETARGETING.md`), tìm file URDF gốc của Unitree G1 (thường có trong repo chính thức Unitree hoặc kèm theo Isaac Lab) và file MJCF tương ứng trong assets của GMR — so sánh 3 khớp bất kỳ, liệt kê thuộc tính nào chỉ xuất hiện trong MJCF mà không có trong URDF, xác nhận lại đúng loại thông tin "chỉ MJCF mới có" đã học ở mục Định nghĩa chính xác.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

URDF (chuẩn ROS, tối giản, kinematic + hình học), MJCF (chuẩn MuJoCo, giàu chi tiết vật lý mô phỏng — solver/actuator/sensor native), và USD (chuẩn scene-graph tổng quát của Pixar/NVIDIA, robot chỉ là một loại asset) phục vụ ba mục đích khác nhau trong cùng một pipeline, không phải ba phiên bản "tốt dần lên" của cùng một ý tưởng; như ví dụ tính tay minh hoạ, hướng chuyển đổi URDF→(MJCF hoặc USD) chỉ cần bổ sung thông tin còn thiếu (không mất mát), trong khi chiều ngược lại (MJCF→URDF) có thể mất tới ~2/3 thông tin đặc thù không có tương đương chuẩn — đây là lý do pipeline thực dụng luôn xuất phát từ một file URDF gốc duy nhất (từ nhà sản xuất robot), rồi convert riêng sang MJCF để debug trên MuJoCo và sang USD (qua Isaac Sim URDF Importer, được xác nhận giữ độ chính xác cao về physics fidelity và joint limits) để train chính thức trên Isaac Lab, trong khi nỗ lực "OpenUSD for Robotics" của NVIDIA đang dần chuẩn hoá việc ánh xạ này ở cấp độ ngành.
