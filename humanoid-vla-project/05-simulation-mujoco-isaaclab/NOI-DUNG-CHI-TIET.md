# Nội dung chi tiết — Simulation: MuJoCo & Isaac Lab

> File này là phần "sách giáo trình": giải thích đầy đủ cơ chế kỹ thuật đứng sau 4 công cụ đã liệt kê ở `README.md`, để bạn hiểu **vì sao** chúng được thiết kế như vậy, không chỉ **cách dùng**. Các citation dùng lại đúng từ README (mục B): Todorov, Erez, Tassa (2012); MuJoCo Playground (arXiv:2502.08844); Makoviychuk et al. (2021, arXiv:2108.10470); HumanoidVerse (GitHub LeCAR-Lab).

---

## 1. Physics engine là gì, và MuJoCo khác gì các engine khác

### 1.1 Physics engine dùng để làm gì

Một physics engine mô phỏng dữ liệu số: cho một hệ vật thể (bodies) nối với nhau bằng khớp (joints), tại mỗi bước thời gian (timestep) nó tính ra vị trí, vận tốc, gia tốc mới của toàn hệ — dựa trên lực tác động (trọng lực, actuator, va chạm...). Với robotics, đây là nơi bạn "thử" một policy điều khiển trước khi đưa lên robot thật, vì mô phỏng sai lệch với thực tế ít thì bạn tiết kiệm được hàng nghìn giờ chạy robot thật (vốn dễ hỏng, chậm, nguy hiểm).

Có hai câu hỏi cốt lõi mọi physics engine phải trả lời, và cách trả lời khác nhau tạo ra sự khác biệt giữa MuJoCo với Bullet/PhysX/Gazebo(ODE):

1. **Biểu diễn trạng thái hệ đa vật thể (đa khớp) như thế nào?** → "generalized coordinates" (MuJoCo, các engine robotics/biomechanics cổ điển) vs "Cartesian/maximal coordinates" (đa số game engine).
2. **Tính va chạm (contact) như thế nào khi hai vật thể chạm nhau?** → "velocity-stepping / tối ưu hoá lồi (convex optimization)" (MuJoCo) vs "spring-damper / penalty method" (đa số game engine).

### 1.2 Generalized coordinates — toạ độ suy rộng

Đây là điểm khác biệt gốc rễ nhất, theo đúng cách paper Todorov et al. (2012) diễn giải:

- **Cách "Cartesian/maximal coordinates" (kiểu game engine — Bullet, PhysX, ODE):** mỗi body được cấp 6 bậc tự do (DOF) tự do hoàn toàn trong không gian (như đang bay tự do), rồi **khớp nối (joint) được áp đặt như một ràng buộc số học** để "khoá bớt" các DOF đó lại, buộc hai body dính vào nhau đúng như joint mô tả. Nói cách khác: mặc định là *rời rạc hoàn toàn*, joint là thứ **loại bỏ (remove)** bậc tự do. Vì ràng buộc này được giải bằng số (numerically), nó **có thể bị vi phạm** — dẫn tới hiện tượng khớp "trôi", "nứt" (joint drift) nếu solver không đủ chính xác hoặc timestep quá lớn — vấn đề rất quen thuộc với ai từng dùng Gazebo/ODE mô phỏng robot nhiều khớp phức tạp.
- **Cách "generalized/joint coordinates" (MuJoCo, và các engine robotics/biomechanics cổ điển như trong tài liệu gốc mô tả):** trạng thái hệ được biểu diễn trực tiếp bằng **góc khớp** (joint angles/positions) — nghĩa là joint **cộng thêm (add)** bậc tự do vào một hệ mặc định *đã hàn cứng (welded)* với nhau. Ràng buộc khớp vì vậy **là ẩn (implicit) trong chính cách biểu diễn** — về mặt toán học, không có cách nào để "vi phạm" một khớp bản lề khi bạn chỉ biểu diễn nó bằng 1 con số góc quay. Đây là cách các thuật toán động lực học đệ quy (recursive dynamics algorithms) hiệu quả và chính xác trong robotics/biomechanics vẫn dùng từ lâu.

Vấn đề lịch sử mà paper MuJoCo chỉ ra: các engine robotics/biomechanics dùng generalized coordinates *chính xác* nhưng theo truyền thống **hoặc bỏ qua hoàn toàn contact dynamics, hoặc chỉ mô phỏng contact bằng spring-damper** (xem mục 1.3) — vốn cần timestep cực nhỏ để ổn định số. MuJoCo là engine đầu tiên kết hợp được **generalized coordinates chính xác** *và* **contact dynamics dựa trên tối ưu hoá** trong cùng một hệ thống — đó chính là đóng góp cốt lõi của paper 2012.

**Ý nghĩa thực hành cho humanoid:** một robot humanoid có 20-40+ khớp nối liên hoàn (kinematic tree phức tạp: từ pelvis → spine → 2 tay, 2 chân, mỗi tay/chân nhiều khớp). Nếu dùng engine kiểu Cartesian, mỗi khớp là một ràng buộc có thể "trôi" — sai số cộng dồn qua toàn bộ chuỗi động học, đặc biệt nguy hiểm khi bạn cần độ chính xác cao để retarget chuyển động người → robot (xem `02-motion-retargeting/`). MuJoCo tránh được lớp sai số này từ gốc.

### 1.3 Tính contact: velocity-stepping / tối ưu hoá lồi, không phải spring-damper

Khi hai geometry (geom) trong mô phỏng chạm nhau, engine phải quyết định: đẩy nhau ra bao nhiêu lực, có ma sát bao nhiêu, có nảy (bounce) hay không.

- **Spring-damper / penalty method (cách cũ, nhiều game engine vẫn dùng biến thể của nó):** coi va chạm như một lò xo ảo cực cứng + giảm chấn (damper) được "nhúng" vào điểm tiếp xúc — vật thể lún vào nhau một chút, lò xo đẩy ngược lại. Vấn đề: để lò xo đủ "cứng" (không cho lún sâu phi thực tế) mà không nổ số (numerical explosion), bạn buộc phải dùng timestep rất nhỏ → chi phí tính toán cao, hoặc chấp nhận vật thể lún/rung không tự nhiên.
- **Velocity-stepping / contact như bài toán tối ưu hoá lồi (MuJoCo):** thay vì mô phỏng một lực lò xo tưởng tượng, MuJoCo tại mỗi bước thời gian **giải một bài toán tối ưu hoá (convex optimization problem)** để tìm ra trực tiếp lực tiếp xúc và vận tốc sau va chạm thoả mãn các ràng buộc vật lý (không xuyên thấu, ma sát Coulomb, v.v.) — cho ra kết quả được mô tả là "soft, convex, và analytically-invertible" (mềm dẻo về số học, lồi, và khả nghịch giải tích). Cách này ổn định hơn nhiều ở timestep lớn hơn, và là lý do MuJoCo được ưa chuộng cho các bài toán *model-based control* (điều khiển cần mô hình động lực học chính xác, ví dụ MPC) — đúng như tên paper gốc: "A Physics Engine for Model-Based Control".

**Vì sao quan trọng khi bạn train RL/WBC:** chất lượng và tốc độ của contact solver quyết định trực tiếp việc chân robot "đứng vững" hay "rung/nảy lung tung" trong mô phỏng — ảnh hưởng thẳng tới sim-to-real gap khi chuyển policy từ MuJoCo sang robot thật.

> Ghi chú xác minh: các mô tả trên dựa trực tiếp trên tài liệu chính thức MuJoCo (mujoco.readthedocs.io, mục Overview) đã được kiểm tra lại trong lúc viết file này — nội dung khớp với cách paper Todorov et al. 2012 trình bày vấn đề. Chi tiết toán học đầy đủ của bài toán tối ưu hoá (dạng LCP/QP cụ thể, các loại solver PGS/Newton/CG mà MuJoCo hỗ trợ) — cần đọc sâu thêm phần "Computation" trong docs nếu bạn cần cài đặt solver tuỳ chỉnh; ở đây chỉ giải thích ở mức khái niệm đủ dùng.

### 1.4 MJCF — định dạng mô tả robot của MuJoCo

MJCF (MuJoCo XML format) là định dạng XML riêng của MuJoCo, được thiết kế "dễ đọc và dễ chỉnh sửa bằng tay nhất có thể" (human readable and editable). Nó không chỉ mô tả hình học như URDF, mà cho phép truy cập gần như toàn bộ khả năng tính toán của MuJoCo (solver options, sensor, actuator nâng cao...).

Các phần tử (element) XML cấp cao quan trọng nhất, mức thực hành:

| Element | Vai trò |
|---|---|
| `<mujoco>` | Element gốc, bắt buộc, có thể đặt tên model. |
| `<option>` | Cấu hình mô phỏng: timestep, gravity, integrator, solver. |
| `<compiler>` | Cấu hình lúc biên dịch model: hệ toạ độ, đường dẫn asset. |
| `<asset>` | Chứa mesh, texture, material, heightfield dùng trong model. |
| `<worldbody>` | Gốc của cây động học (kinematic tree) — chứa tất cả `<body>`. |
| `<body>` | Một vật thể cứng (rigid body) — có thể lồng nhau (body con bên trong body cha) để tạo chuỗi động học. |
| `<joint>` | Gắn vào 1 body, định nghĩa khớp nối với body cha (hinge, slide, ball, free...) — **đây chính là nơi "cộng thêm" bậc tự do** như giải thích ở mục 1.2. |
| `<geom>` | Hình học va chạm + hiển thị của 1 body (box, sphere, capsule, mesh...). |
| `<actuator>` | Định nghĩa động cơ điều khiển khớp (motor, position, velocity...). |
| `<sensor>` | Cảm biến ảo (accelerometer, gyro, joint position sensor...). |
| `<contact>` | Cấu hình cặp tiếp xúc, loại trừ va chạm giữa 2 geom cụ thể. |
| `<keyframe>` | Trạng thái mẫu định sẵn (ví dụ tư thế đứng ban đầu). |

Ví dụ MJCF tối giản (1 hộp rơi tự do dưới trọng lực):

```xml
<mujoco model="hop_don_gian">
  <option timestep="0.002" gravity="0 0 -9.81"/>
  <worldbody>
    <light pos="0 0 3"/>
    <geom type="plane" size="1 1 0.1"/>  <!-- mặt sàn -->
    <body name="hop" pos="0 0 1">
      <joint type="free"/>               <!-- rơi tự do 6-DOF -->
      <geom type="box" size="0.1 0.1 0.1" rgba="0.8 0.2 0.2 1"/>
    </body>
  </worldbody>
</mujoco>
```

Ví dụ robot 1 khớp bản lề (con lắc — minh hoạ khớp giới hạn 1-DOF, gần với cách 1 khớp gối/khuỷu tay của humanoid được mô tả):

```xml
<mujoco model="con_lac">
  <worldbody>
    <body name="tay_don" pos="0 0 1">
      <joint name="khop_1" type="hinge" axis="0 1 0" range="-90 90"/>
      <geom type="capsule" fromto="0 0 0  0 0 -0.5" size="0.02"/>
      <body name="qua_ta" pos="0 0 -0.5">
        <geom type="sphere" size="0.05" mass="1"/>
      </body>
    </body>
  </worldbody>
</mujoco>
```

Robot humanoid thật (G1, H1...) chỉ là mở rộng ý tưởng này: hàng chục `<body>` lồng nhau (pelvis → torso → tay/chân), mỗi body gắn 1-3 `<joint>`, cộng thêm `<actuator>` cho từng khớp — file MJCF của G1 có thể dài hàng trăm dòng nhưng cấu trúc gốc giống hệt 2 ví dụ trên.

---

## 2. MuJoCo Playground — kiến trúc GPU-accelerated (MJX/JAX)

### 2.1 Vấn đề: MuJoCo gốc chạy trên CPU, tuần tự

MuJoCo (bản gốc, "MuJoCo thuần" ở mục A của README) là một thư viện C/C++ chạy trên **CPU**, mô phỏng **một instance tại một thời điểm**. Với RL — vốn cần thu thập hàng triệu, hàng tỷ bước mô phỏng (environment steps) để policy hội tụ — chạy tuần tự từng instance một là điểm nghẽn tốc độ lớn nhất. Cách giải quyết truyền thống là chạy nhiều tiến trình CPU song song (multiprocessing) — nhưng vẫn bị giới hạn bởi số nhân CPU (vài chục, hiếm khi hơn 100).

### 2.2 MJX — MuJoCo viết lại bằng JAX để chạy trên GPU

**MJX** là một cài đặt lại (reimplementation) phần lớn thuật toán vật lý của MuJoCo bằng **JAX** — thư viện tính toán số của Google hỗ trợ chạy vector hoá song song (vectorized) trực tiếp trên GPU/TPU và tự động vi phân (autodiff). Khác biệt cốt lõi so với MuJoCo CPU gốc:

- **MuJoCo CPU:** 1 vòng lặp mô phỏng = 1 instance robot, chạy tuần tự trên 1 nhân CPU.
- **MJX trên GPU:** hàng nghìn instance robot (mỗi instance là 1 bản sao độc lập của cùng 1 môi trường, ví dụ 4096 con robot humanoid đang tập đi song song) được xếp thành **một tensor lớn**, và toàn bộ phép tính vật lý (tích phân động lực học, giải contact) được thực hiện **đồng thời trên tất cả instance** nhờ kiến trúc SIMD/song song hàng loạt (massively parallel) của GPU.

**MuJoCo Playground** (Google DeepMind, arXiv:2502.08844, dẫn ở README mục B) là bộ môi trường RL đóng gói sẵn trên nền MJX — tức bạn không cần tự viết code JAX, chỉ cần gọi task có sẵn (`humanoid-stand`, `humanoid-walk`, `humanoid-run`...) và một thuật toán RL (PPO/SAC có sẵn) sẽ tự động huấn luyện song song trên GPU của bạn.

### 2.3 Vì sao song song hoá tăng tốc RL training

Trong một vòng lặp RL kinh điển: policy hành động → môi trường trả về trạng thái mới + reward → policy cập nhật. Nếu bạn có 4096 bản sao môi trường chạy song song, mỗi "bước" thu thập được 4096 mẫu dữ liệu cùng lúc thay vì 1 mẫu — giảm thời gian thu thập dữ liệu (wall-clock time) theo cấp số nhân so với chạy tuần tự, dù tổng số phép tính không đổi. Đây là lý do các bài toán mà trước kia mất nhiều ngày/tuần huấn luyện trên CPU cluster giờ có thể xong trong vài giờ trên 1 GPU cá nhân — chính là điểm hấp dẫn của MuJoCo Playground với người nghiên cứu không có hạ tầng lớn.

### 2.4 Task humanoid có sẵn hoạt động ra sao (mức khái niệm)

Các task như `humanoid-stand`/`humanoid-walk`/`humanoid-run` trong MuJoCo Playground đã đóng gói sẵn: (1) file MJCF của 1 robot humanoid chuẩn, (2) hàm reward (ví dụ: `stand` thưởng giữ thân thẳng đứng ổn định, `walk`/`run` thưởng theo tốc độ tiến + giữ thăng bằng), (3) điều kiện kết thúc episode (robot ngã → reset). Khi bạn chạy training, hàng nghìn bản sao robot này cùng "thử" hành động ngẫu nhiên ban đầu, phần lớn ngã ngay — nhưng vì có hàng nghìn mẫu mỗi bước, policy học rất nhanh cách đứng vững rồi tiến tới đi được, chỉ sau vài chục phút tới vài giờ tuỳ độ khó task (liên hệ mục D-3 trong README: quan sát "policy học đi từ đầu tới ổn định").

---

## 3. Isaac Gym → Isaac Lab: kiến trúc GPU-based simulation của NVIDIA

### 3.1 Ý tưởng cốt lõi từ paper Isaac Gym (Makoviychuk et al. 2021)

Trước Isaac Gym, pipeline RL cho robot tiêu chuẩn là: **physics simulation chạy trên CPU**, còn **neural network training (forward/backward pass) chạy trên GPU**. Ở mỗi bước, dữ liệu (trạng thái robot, reward) phải được **truyền qua lại giữa CPU và GPU** (qua PCIe bus) — với robot phức tạp và hàng nghìn môi trường song song, chi phí truyền dữ liệu này (CPU↔GPU transfer) trở thành **điểm nghẽn (bottleneck)** lớn hơn cả chi phí tính toán vật lý hay tính toán mạng nơ-ron.

Đóng góp cốt lõi của Isaac Gym: chạy **cả physics simulation lẫn neural network training trên cùng GPU**, với dữ liệu ở dạng **PyTorch tensor nằm sẵn trên GPU memory**, không bao giờ phải rời khỏi GPU giữa bước mô phỏng và bước cập nhật mạng. Nói cách khác: loại bỏ hoàn toàn round-trip CPU↔GPU khỏi vòng lặp huấn luyện. Đây chính là lý do paper báo cáo tăng tốc **2-3 bậc độ lớn (orders of magnitude)** so với setup CPU-sim + GPU-train truyền thống — không phải vì GPU tính vật lý "thông minh hơn" CPU, mà vì **loại bỏ được chi phí truyền dữ liệu** vốn chiếm phần lớn thời gian ở setup cũ, đồng thời tận dụng được khả năng song song hàng nghìn instance giống MJX ở mục 2.

### 3.2 Isaac Lab — bước kế thừa hiện đại hơn Isaac Gym

Theo tài liệu chính thức NVIDIA (developer.nvidia.com/isaac/lab, đã kiểm tra lúc viết file này), Isaac Lab là **"a lightweight, open-source framework built on top of Isaac Sim, specifically optimized for robot learning workflows"** — tức không phải một engine vật lý độc lập mới, mà là một lớp framework xây **trên nền Isaac Sim/Omniverse** (nền tảng mô phỏng robot tổng quát của NVIDIA, vốn dùng cho cả sinh dữ liệu tổng hợp, kiểm định thiết kế, không chỉ RL). Isaac Lab kế thừa đúng ý tưởng gốc "physics + training cùng trên GPU" từ Isaac Gym, nhưng bổ sung:

- **USD (Universal Scene Description)** thay vì chỉ URDF/MJCF thuần — vì Omniverse (nền tảng NVIDIA dựa trên chuẩn USD của Pixar) dùng USD làm định dạng scene gốc, cho phép mô tả scene phức tạp hơn (vật liệu PBR, ánh sáng, nhiều loại asset) chứ không chỉ robot đơn thuần — xem so sánh chi tiết ở mục 5.
- **GPU-optimized simulation paths built on Warp và CUDA-graphable environments** — Warp là ngôn ngữ/runtime tính toán GPU của NVIDIA (tương tự vai trò JAX với MJX ở mục 2), cho phép chạy môi trường song song quy mô lớn từ máy trạm cá nhân tới cloud data-center, hỗ trợ cả multi-GPU/multi-node.
- **Tích hợp sẵn nhiều thư viện RL**, không khoá cứng vào 1 thuật toán: "customize workflows... integrate custom libraries (e.g., skrl, RLLib, rl_games...)" — trong đó **RSL-RL** là thư viện được dùng phổ biến nhất cho locomotion/WBC (xem mục 6).
- **Isaac Lab 2.3** (bản mới nhất được README dẫn) bổ sung tính năng **whole-body control (WBC) và teleoperation nâng cao** chính thức — đây là lý do README gọi Isaac Lab là "nền chính thức mà hệ sinh thái GR00T dùng": pipeline retargeting → training → deploy của NVIDIA cho robot G1 dựa trực tiếp trên framework này.

**Tóm gọn quan hệ:** Isaac Gym (2021, paper nghiên cứu, đã deprecated) → chứng minh ý tưởng "all-GPU pipeline" → Isaac Lab (framework production/nghiên cứu hiện hành, xây trên Isaac Sim/Omniverse, mở rộng thêm USD, đa thư viện RL, WBC/teleop). NVIDIA khuyến nghị người dùng Isaac Gym cũ chuyển sang Isaac Lab để có tính năng mới nhất.

> Ghi chú xác minh: trang developer.nvidia.com/isaac/lab không giải thích chi tiết USD là gì hay cơ chế GPU-sim cụ thể ở mức thuật toán (loại solver PhysX dùng) — phần "USD dùng để làm gì cụ thể" ở mục 5 dưới đây dựa trên hiểu biết chung về hệ sinh thái Omniverse/USD, không trích trực tiếp từ trang đã fetch; nếu cần độ chính xác tuyệt đối về USD, nên đọc thêm tài liệu OpenUSD chính thức (openusd.org).

---

## 4. HumanoidVerse — lớp trừu tượng multi-simulator

### 4.1 Vấn đề nó giải quyết

Mỗi công cụ ở mục 1-3 (MuJoCo/MJX, Isaac Gym/Isaac Lab) có API, định dạng model, và quy ước reward/observation riêng. Nếu code huấn luyện của bạn viết thẳng theo API của 1 simulator, việc chuyển sang simulator khác (để kiểm tra xem policy có phụ thuộc quá mức vào đặc thù vật lý của 1 engine cụ thể hay không) đòi hỏi viết lại gần như toàn bộ.

### 4.2 Ý tưởng kiến trúc: 3 lớp độc lập

Theo README kiến trúc trên GitHub LeCAR-Lab/HumanoidVerse (dẫn ở README mục B), HumanoidVerse tách hệ thống thành 3 lớp tách biệt, giao tiếp qua interface chung:

1. **Simulator layer** — lớp adapter cho từng physics engine cụ thể (IsaacGym, Genesis, Isaac Lab...): chịu trách nhiệm dịch giữa API chung của HumanoidVerse và API riêng của từng engine.
2. **Task layer** — định nghĩa bài toán (ví dụ: motion tracking cho robot Unitree H1/G1) độc lập với việc bài toán đó đang chạy trên engine nào — cùng 1 định nghĩa reward, observation, episode logic dùng chung.
3. **Algorithm layer** — thuật toán RL/IL (PPO và các biến thể) tách khỏi cả simulator lẫn task, có thể swap qua lại.

Nhờ tách 3 lớp này, bạn có thể giữ nguyên **task** và **algorithm**, chỉ đổi **simulator adapter**, để chạy đúng cùng 1 policy trên nhiều engine khác nhau.

### 4.3 Lợi ích cụ thể khi nghiên cứu

- **Kiểm tra robustness (tính vững) của policy:** nếu một policy huấn luyện trên IsaacGym vẫn đi vững khi chuyển sang chạy trên MuJoCo/Genesis (2 engine có contact solver khác nhau — xem mục 1.3), đó là bằng chứng policy không bị "overfit" vào đặc thù vật lý riêng của 1 engine (một dạng kiểm tra sim-to-sim trước khi thử sim-to-real).
- **Không khoá cứng hạ tầng ngay từ đầu học:** người mới có thể bắt đầu học trên MuJoCo (nhẹ, không cần GPU mạnh) rồi chuyển pipeline sang Isaac Lab sau, mà không phải viết lại toàn bộ code task/reward.
- **So sánh tốc độ/độ ổn định huấn luyện** giữa các engine trên cùng 1 bài toán — hữu ích khi quyết định hạ tầng nào phù hợp cho dự án dài hạn.

Đây cũng chính là lý do README gợi ý dùng HumanoidVerse ở bước "checkpoint nâng cao" (mục D) — cần đã quen ít nhất 2 simulator riêng lẻ trước khi lớp trừu tượng này thực sự có ích, vì nếu chưa hiểu rõ MuJoCo/Isaac Lab hoạt động độc lập ra sao, lớp trừu tượng sẽ che mất chi tiết cần thiết để debug khi có sai lệch giữa các engine.

---

## 5. So sánh MJCF vs URDF vs USD

| | **URDF** | **MJCF** | **USD** |
|---|---|---|---|
| Ai tạo ra | Cộng đồng ROS (Willow Garage) | DeepMind/Roboti (MuJoCo) | Pixar, sau NVIDIA mở rộng cho robotics/Omniverse |
| Định dạng | XML | XML | Định dạng scene-graph riêng (`.usd`/`.usda`/`.usdc`), không phải XML thuần |
| Trọng tâm thiết kế | Mô tả **kinematic + hình học** robot đơn giản, dễ đọc, chuẩn hoá cho ROS | Mô tả robot **kèm chi tiết vật lý mô phỏng** (solver, actuator, contact params...) — như mục 1.4 | Mô tả **toàn bộ scene 3D phức tạp** (vật liệu, ánh sáng, animation, nhiều loại asset), robot chỉ là 1 use-case trong hệ sinh thái lớn hơn |
| Có mô tả contact/solver chi tiết? | Không (URDF không có khái niệm actuator/solver — cần plugin bên ngoài, ví dụ Gazebo) | Có, native | Có, qua schema PhysX/USD Physics riêng |
| Dùng ở đâu (trong dự án này) | Robot thật thường được nhà sản xuất (Unitree) cấp URDF gốc; nhiều pipeline retargeting (`02-motion-retargeting/`) cũng dùng URDF làm input chuẩn vì phổ biến nhất | MuJoCo/MuJoCo Playground | Isaac Sim/Isaac Lab |
| Chuyển đổi | URDF → MJCF: MuJoCo có công cụ `compiler` đọc trực tiếp URDF rồi convert nội bộ (hoặc dùng `urdf2mjcf`); MJCF → URDF khó hơn (mất chi tiết đặc thù MuJoCo). URDF → USD: NVIDIA cung cấp Isaac Sim URDF Importer (đọc URDF, sinh USD tương ứng). | | |

**Ý nghĩa thực hành cho pipeline của bạn:** dữ liệu chuyển động sau khi retarget (`02-motion-retargeting/`) thường gắn với 1 mô hình robot cụ thể ở định dạng URDF (từ Unitree) hoặc MJCF (nếu retargeting tool như GMR dùng MuJoCo làm viewer — đúng như README mục A ghi chú). Khi bạn chuyển từ giai đoạn "debug trên MuJoCo" (mục D bước 1-3) sang "train chính thức trên Isaac Lab" (mục D bước 4), bước bắt buộc là **convert model robot sang USD** — dùng URDF Importer của Isaac Sim (import trực tiếp từ URDF gốc là đường đi phổ biến nhất, ít mất thông tin hơn convert ngược từ MJCF).

---

## 6. RSL-RL và PPO trong Isaac Lab

**RSL-RL** là thư viện reinforcement learning mã nguồn mở của **Robotic Systems Lab (RSL), ETH Zürich** — nhóm nghiên cứu đứng sau nhiều công trình locomotion nổi tiếng (bao gồm paper "Learning to Walk in Minutes..." đã dẫn ở `04-imitation-learning-rl/` mục A-2). Thư viện được thiết kế tối giản, tối ưu riêng cho bài toán locomotion/legged robot chạy trên GPU quy mô lớn (khớp thẳng với kiến trúc "physics + training cùng GPU" của Isaac Lab đã giải thích ở mục 3.1) — khác với các thư viện RL tổng quát (Stable-Baselines3, RLlib) vốn ưu tiên tính tổng quát hơn tốc độ tối đa cho 1 use-case cụ thể.

Trong Isaac Lab, **RSL-RL được tích hợp làm lựa chọn mặc định phổ biến nhất cho locomotion**, và thuật toán mặc định là **PPO (Proximal Policy Optimization)** — đã giải thích chi tiết ở `04-imitation-learning-rl/` mục A-1 (Schulman et al. 2017, arXiv:1707.06347). Việc chọn PPO không phải ngẫu nhiên: PPO ổn định hơn các phương pháp policy-gradient cũ (ít nhạy với learning rate, không cần second-order optimization như TRPO), phù hợp với training on-policy tốc độ cao khi bạn có hàng nghìn môi trường song song liên tục sinh dữ liệu mới (đúng bối cảnh GPU-parallel training của Isaac Lab/MuJoCo Playground) — với on-policy method, dữ liệu cũ nhanh chóng "lỗi thời" so với policy hiện tại, nên tốc độ sinh dữ liệu song song lớn giúp bù đắp việc không tái sử dụng được nhiều dữ liệu cũ (khác với off-policy như SAC).

Isaac Lab không khoá cứng vào RSL-RL — như mục 3.2 đã trích, bạn có thể tích hợp skrl, RLlib, rl_games... nhưng với riêng bài toán humanoid locomotion/WBC, RSL-RL + PPO là combo mặc định bạn sẽ gặp trong phần lớn code mẫu chính thức và playlist Skyentific (README mục C).

---

## Bảng thuật ngữ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| **Physics engine** | Phần mềm mô phỏng động lực học vật thể (vị trí, vận tốc, lực) theo thời gian rời rạc (timestep). |
| **Generalized coordinates (toạ độ suy rộng)** | Cách biểu diễn trạng thái hệ đa khớp bằng góc/vị trí khớp trực tiếp, thay vì toạ độ Cartesian từng vật thể — ràng buộc khớp là ẩn, không thể vi phạm. MuJoCo dùng cách này. |
| **Cartesian / maximal coordinates** | Mỗi body có 6-DOF tự do độc lập, joint là ràng buộc số học áp lên — dễ "trôi" khớp. Nhiều game engine (Bullet, PhysX cổ điển, ODE/Gazebo) dùng cách này. |
| **Contact dynamics** | Cách engine tính lực/vận tốc khi 2 vật thể va chạm/tiếp xúc. |
| **Spring-damper / penalty method** | Cách tính contact cũ: mô phỏng va chạm bằng lò xo ảo + giảm chấn — cần timestep nhỏ để ổn định. |
| **Velocity-stepping / convex optimization (contact)** | Cách MuJoCo tính contact: giải trực tiếp 1 bài toán tối ưu hoá lồi để ra lực/vận tốc sau va chạm — ổn định hơn ở timestep lớn. |
| **MJCF** | Định dạng XML mô tả model của MuJoCo (body, joint, geom, actuator...). |
| **MJX** | Bản viết lại MuJoCo bằng JAX, chạy vector hoá song song trên GPU/TPU. |
| **JAX** | Thư viện tính toán số của Google hỗ trợ autodiff + chạy song song trên GPU/TPU. |
| **MuJoCo Playground** | Bộ môi trường RL đóng gói sẵn trên nền MJX, có task humanoid sẵn (Google DeepMind, 2025). |
| **Isaac Gym** | Framework nghiên cứu 2021 của NVIDIA, chứng minh ý tưởng chạy physics + training RL cùng trên GPU bằng PyTorch tensor — tiền thân của Isaac Lab, nay đã deprecated. |
| **Isaac Sim / Omniverse** | Nền tảng mô phỏng robot tổng quát của NVIDIA (không chỉ RL) mà Isaac Lab được xây trên đó, dựa trên chuẩn USD. |
| **Isaac Lab** | Framework robot learning hiện hành của NVIDIA, xây trên Isaac Sim, hỗ trợ GPU-parallel training, USD, RSL-RL, WBC/teleoperation (bản 2.3+). |
| **Warp** | Runtime/ngôn ngữ tính toán GPU của NVIDIA dùng trong Isaac Lab — vai trò tương tự JAX với MJX. |
| **USD (Universal Scene Description)** | Định dạng scene-graph gốc của Pixar, được Omniverse/Isaac Sim dùng làm định dạng model/scene chính. |
| **URDF** | Định dạng XML mô tả robot chuẩn của ROS — phổ biến nhất làm input gốc từ nhà sản xuất robot. |
| **HumanoidVerse** | Framework của LeCAR-Lab (CMU) trừu tượng hoá simulator/task/algorithm thành 3 lớp độc lập, hỗ trợ chạy đa simulator. |
| **RSL-RL** | Thư viện RL của ETH Zürich (Robotic Systems Lab), tích hợp mặc định trong Isaac Lab cho locomotion, tối ưu cho GPU-parallel training. |
| **PPO (Proximal Policy Optimization)** | Thuật toán RL on-policy phổ biến nhất cho locomotion/WBC hiện nay — chi tiết ở `04-imitation-learning-rl/`. |
| **Domain randomization** | (Nhắc lại, liên quan gián tiếp) Kỹ thuật ngẫu nhiên hoá tham số mô phỏng (ma sát, khối lượng...) để policy tổng quát hoá tốt hơn khi chuyển sang robot thật — xem `04-imitation-learning-rl/`. |

---

## Liên kết chéo

- Cách dùng thực hành từng công cụ, lộ trình học, và bảng so sánh nhanh: xem `README.md` cùng thư mục.
- Thuật toán RL/IL chạy bên trong các simulator này (PPO, DeepMimic, AMP, SONIC): `../04-imitation-learning-rl/`.
- Kết quả huấn luyện được đánh giá bằng metric nào: `../07-policy-evaluation/`.
- Bước chuyển từ Isaac Lab sang robot thật: `../08-real-robot-deployment/`.
