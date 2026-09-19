# Bài giảng: LAFAN1 và định dạng BVH

*(Thuộc mảng: Human Motion Datasets)*

## 🎯 Mục tiêu bài học

- Mô tả chính xác cấu trúc file BVH: phần `HIERARCHY` (cây khớp + `OFFSET` + `CHANNELS`) và phần `MOTION` (`Frames`, `Frame Time`, ma trận góc Euler theo khung hình).
- Giải thích được 3 khác biệt căn bản giữa BVH và SMPL/SMPL-X (đã học ở các bài trước): không mesh, góc Euler tuyệt đối (gimbal lock), không tách shape/pose.
- Đọc tay được một đoạn file BVH tối giản và tính ra vị trí toàn cục của một khớp con từ `OFFSET` + góc xoay, dùng ma trận xoay.
- Mô tả được quy trình thu thập LAFAN1 (Ubisoft La Forge) và bài toán motion in-betweening mà nó phục vụ ban đầu.
- Nêu được vì sao LAFAN1 trở thành benchmark phổ biến cho retargeting/motion-tracking RL humanoid dù được tạo ra cho mục đích khác (animation game).
- Phân biệt được khi nào nên chọn nguồn dữ liệu dạng BVH (LAFAN1) thay vì SMPL-X (AMASS) cho một pipeline retargeting cụ thể.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Ba bài trước (SMPL, SMPL-X, AMASS) xây dựng nền tảng cho một **họ định dạng dữ liệu chuyển động người**: tham số hoá bằng shape (β) + pose (θ), có mesh, khả vi, dùng cho nghiên cứu học thuật. LAFAN1 đại diện cho một **họ định dạng hoàn toàn khác** — BVH, xuất thân từ công nghiệp game/hoạt hình, không có mesh, chỉ có khớp và góc xoay. Cả hai bài giảng GMR và SOMA-retargeter (`02-motion-retargeting/`) đều liệt kê BVH/LAFAN1 là một trong các định dạng input chính được hỗ trợ — học bài này để hiểu chính xác *tại sao* pipeline retargeting phải xử lý được cả hai họ định dạng khác nhau, và dữ liệu bên trong một file `.bvh` thực chất trông như thế nào trước khi nó được đưa vào bất kỳ bộ giải IK nào.

## 🧠 Trực giác

### Góc nhìn 1: Bản vẽ kỹ thuật khung xe đạp (chỉ khung, không vỏ bọc) so với mô hình 3D hoàn chỉnh có vỏ

SMPL/SMPL-X giống một mô hình 3D hoàn chỉnh của một chiếc xe đạp — có khung, có vỏ bọc (mesh), có thể "nhìn thấy" hình dáng thật khi render. BVH giống một **bản vẽ kỹ thuật chỉ có khung xe đạp** — biết chính xác chiều dài từng ống khung (`OFFSET`) và góc gập tại từng khớp nối (giá trị Euler trong `MOTION`), nhưng không biết vỏ bọc/sơn màu trông ra sao. Với mục đích retargeting (chỉ cần biết khung xương chuyển động thế nào để suy ra chuyển động khung robot), "bản vẽ khung" này đã đủ thông tin — không cần vỏ bọc.

**Giới hạn của loại suy này:** một bản vẽ kỹ thuật tĩnh chỉ có một trạng thái; file BVH có **hàng trăm/hàng nghìn khung hình** theo thời gian (phần `MOTION`), giống một cuốn phim hoạt hình về "khung xe đang chuyển động" hơn là một bản vẽ tĩnh duy nhất — loại suy bản vẽ chỉ nắm được phần `HIERARCHY`, chưa nắm được bản chất động của phần `MOTION`.

### Góc nhìn 2: Sổ ghi chép thao tác máy CNC (góc quay từng trục) so với toạ độ GPS

Một máy CNC (điều khiển số) ghi lại chuyển động bằng góc quay tuyệt đối của từng trục động cơ theo thời gian — không ghi toạ độ điểm cuối mũi khoan trong không gian, mà ghi "trục X quay bao nhiêu độ, trục Y quay bao nhiêu độ". Muốn biết mũi khoan đang ở đâu trong không gian, phải **tính lại** từ các góc quay đó cộng với chiều dài các bộ phận máy — đúng như BVH: file chỉ lưu góc Euler từng khớp (`MOTION`) và chiều dài xương (`OFFSET` trong `HIERARCHY`); muốn biết vị trí toàn cục một khớp bất kỳ (ví dụ cổ tay), phải tự tính bằng Forward Kinematics (đã học ở `02-motion-retargeting/`).

**Giới hạn của loại suy này:** máy CNC thường chỉ có vài trục độc lập không lồng nhau phức tạp; skeleton BVH là một **cây phân cấp** (hierarchy) — góc xoay của khớp cha ảnh hưởng tới vị trí toàn cục của mọi khớp con phía sau nó, một mức độ phụ thuộc lồng nhau mà loại suy máy CNC đơn trục không thể hiện đầy đủ.

## 📐 Định nghĩa chính xác

**BVH (BioVision Hierarchy)** là định dạng file mocap dạng văn bản, ra đời từ ngành công nghiệp game/hoạt hình thập niên 1990, gồm đúng hai phần:

**Phần `HIERARCHY`** — khai báo cấu trúc skeleton dạng cây, bắt đầu từ khớp gốc `ROOT` (thường là hông/pelvis):

```text
HIERARCHY
ROOT Hips
{
    OFFSET 0.00 0.00 0.00
    CHANNELS 6 Xposition Yposition Zposition Zrotation Yrotation Xrotation
    JOINT LeftUpLeg
    {
        OFFSET 3.5 -2.1 0.0
        CHANNELS 3 Zrotation Yrotation Xrotation
        JOINT LeftLeg
        {
            OFFSET 0.0 -18.5 0.0
            CHANNELS 3 Zrotation Yrotation Xrotation
            End Site
            {
                OFFSET 0.0 -20.0 0.0
            }
        }
    }
    ...
}
```

- `OFFSET x y z`: độ dịch chuyển tĩnh (chiều dài xương) từ khớp cha đến khớp này, **không đổi trong suốt file** — đây chính là "kích thước cơ thể" của skeleton cụ thể đó.
- `CHANNELS`: số và loại kênh dữ liệu sẽ ghi trong `MOTION` cho khớp này — khớp thường có 3 kênh xoay (Euler `Z-Y-X` theo thứ tự áp dụng cụ thể của file), riêng `ROOT` có thêm 3 kênh dịch chuyển vị trí toàn cục.
- `End Site`: điểm cuối một nhánh (đầu ngón tay, đỉnh đầu), không có khớp con, không xuất hiện trong `MOTION`.

**Phần `MOTION`** — dữ liệu theo thời gian:

```text
MOTION
Frames: 3
Frame Time: 0.033333
0.0 90.0 0.0  0.0 0.0 0.0   10.0 0.0 0.0   5.0 0.0 0.0
0.0 90.0 0.0  0.0 0.0 2.0   10.5 0.0 0.0   5.2 0.0 0.0
0.0 90.0 0.0  0.0 0.0 4.0   11.0 0.0 0.0   5.4 0.0 0.0
```

Mỗi dòng là **một khung hình**; mỗi cột là giá trị của một kênh đã khai báo ở `HIERARCHY` (đúng thứ tự khai báo). `Frame Time` (ví dụ `0.033333 s ≈ 1/30 s`) xác định tần số khung hình (ở đây 30Hz).

**Ba khác biệt căn bản với SMPL/SMPL-X** (đã học):

1. **Không mesh** — BVH chỉ là skeleton thuần (xương + khớp), không biết hình dáng da thịt.
2. **Góc Euler tuyệt đối theo từng khớp** — dễ gặp **gimbal lock** (mất một bậc tự do khi hai trục xoay trùng nhau), trong khi SMPL dùng axis-angle.
3. **Không tách shape/pose** — chiều dài xương (`OFFSET`) cố định cho một skeleton cụ thể; muốn đổi sang người khác phải đổi cả file offset, khác hẳn cơ chế β (shape) độc lập với θ (pose) của SMPL.

## ⚙️ Cơ chế hoạt động — từng bước

```text
┌────────────────────────────────────────────────────────────┐
│ Đọc HIERARCHY: dựng cây skeleton                             │
│  ROOT → JOINT → JOINT → ... → End Site                       │
│  mỗi cạnh mang OFFSET (chiều dài xương cố định)               │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ Đọc MOTION: với mỗi frame t                                  │
│  đọc đúng số cột theo thứ tự CHANNELS đã khai báo             │
│  ROOT: (x,y,z) vị trí toàn cục + (Rz,Ry,Rx) hướng toàn cục    │
│  Mỗi JOINT con: (Rz,Ry,Rx) góc xoay LOCAL so với khớp cha     │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ Forward Kinematics: tính vị trí toàn cục từng khớp            │
│  global_transform(joint) =                                   │
│    global_transform(parent) × T(OFFSET) × R(góc_local)        │
│  (nhân ma trận đồng nhất — homogeneous transform — dọc cây)   │
└──────────────────────────────────────────────────────────────┘
```

Điểm quan trọng: bản thân file BVH **không lưu vị trí toàn cục** của các khớp không phải root — mọi vị trí toàn cục phải được **tính lại** bằng Forward Kinematics đi từ root xuống, nhân dồn các phép biến đổi `OFFSET` (tĩnh) và góc xoay `local` (theo frame) — đúng cơ chế mà các bài IK ở `02-motion-retargeting/` giả định làm nền tảng.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ, số tự chọn để dễ hình dung — đơn giản hoá về 2D, một phần nhỏ của ví dụ HIERARCHY ở trên)*

Xét chain: `Hips (root)` → `LeftUpLeg` (OFFSET `(3.5, −2.1)` cm, tính 2D) → `LeftLeg` (OFFSET `(0.0, −18.5)` cm).

Tại một frame, dữ liệu MOTION cho:
- `Hips`: vị trí toàn cục `(0, 90)` cm (từ ví dụ ở trên, cột 1-2 — bỏ qua trục z và các góc root để đơn giản 2D), góc xoay root `0°`.
- `LeftUpLeg`: góc xoay local `Rz = 0°` (dùng frame đầu tiên trong bảng MOTION ở trên).
- `LeftLeg`: góc xoay local `Rz = 10°` *(giả định minh hoạ, khác số 0 để thấy rõ phép tính xoay)*.

**Bước 1 — vị trí toàn cục của `LeftUpLeg`:** vì góc xoay của `Hips` và của chính `LeftUpLeg` đều `0°` trong ví dụ này, vị trí toàn cục đơn giản là cộng offset trực tiếp:

```text
p_LeftUpLeg = p_Hips + OFFSET_LeftUpLeg
            = (0, 90) + (3.5, −2.1)
            = (3.5, 87.9) cm
```

**Bước 2 — vị trí toàn cục của `LeftLeg`:** offset của `LeftLeg` là `(0.0, −18.5)` cm tính **trong hệ toạ độ local của `LeftUpLeg`** — vì `LeftUpLeg` xoay `10°` (giả định ở trên) so với hệ toạ độ cha, phải **xoay vector offset trước khi cộng**:

```text
Ma trận xoay 2D góc 10°:
R(10°) = [cos10° −sin10°]   [0.9848 −0.1736]
         [sin10°  cos10°] = [0.1736  0.9848]

vector offset LeftLeg trong hệ cha: v = (0.0, −18.5)

v_xoay = R(10°) · v
       = (0.9848×0.0 + (−0.1736)×(−18.5),  0.1736×0.0 + 0.9848×(−18.5))
       = (3.2116, −18.2188) cm
```

```text
p_LeftLeg = p_LeftUpLeg + v_xoay
          = (3.5, 87.9) + (3.2116, −18.2188)
          = (6.7116, 69.6812) cm
```

**Ý nghĩa:** nếu bỏ qua bước xoay vector offset (một lỗi lập trình phổ biến khi tự viết parser BVH) và chỉ cộng thẳng `(0.0, −18.5)` không xoay, kết quả sai sẽ là `(3.5, 69.4)` cm — lệch so với kết quả đúng `(6.7116, 69.6812)` một khoảng:

```text
‖Δp‖ = √((6.7116−3.5)² + (69.6812−69.4)²) = √(3.2116² + 0.2812²) ≈ √(10.31+0.079) ≈ 3.22 cm
```

Đây chính xác là lý do phần "Cơ chế hoạt động" nhấn mạnh: `OFFSET` phải được **xoay theo hệ toạ độ hiện tại của khớp cha** trước khi cộng dồn — bỏ qua bước này là lỗi cụ thể, định lượng được (ở đây 3.22 cm cho một khớp đơn giản, và sẽ khuếch đại dọc theo chain dài hơn — đúng hiệu ứng khuếch đại sai số đã học ở bài "Vì sao không thể copy góc khớp" trong `02-motion-retargeting/`).

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | BVH (LAFAN1) | SMPL/SMPL-X (AMASS) |
|---|---|---|
| Có mesh/bề mặt? | Không — skeleton thuần | Có — N=10,475 vertex (SMPL-X) |
| Biểu diễn xoay | Góc Euler tuyệt đối theo khớp (gimbal lock có thể xảy ra) | Axis-angle, có blend shapes |
| Tách shape/pose? | Không — `OFFSET` cố định theo skeleton cụ thể | Có — β (shape) độc lập θ (pose) |
| Nguồn gốc ngành | Game/hoạt hình (Ubisoft La Forge) | Nghiên cứu học thuật (AMASS hợp nhất 15 bộ mocap) |
| Quy mô (LAFAN1 vs AMASS) | 496,672 khung @30Hz (~4.6 giờ), 5 diễn viên, 77 sequence | >40 giờ, >11.000 motion, >300 chủ thể |
| Chất lượng mocap | Rất cao, chuẩn sản xuất game AAA, ít nhiễu | Không đồng đều (phụ thuộc 15 nguồn mocap gốc khác nhau) |
| Có tương tác vật thể? | Không | Không (đây là khoảng trống mà OMOMO bổ sung) |
| Được GMR/SOMA-retargeter hỗ trợ? | Có (input BVH trực tiếp) | Có (input SMPL-X trực tiếp) |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "file BVH đã có sẵn vị trí toàn cục của mọi khớp, chỉ cần đọc thẳng cột dữ liệu".** Vì sao sai: như ví dụ tính tay cho thấy, `MOTION` chỉ ghi **góc xoay local** (trừ vị trí/hướng của riêng `ROOT`) — vị trí toàn cục của mọi khớp khác phải được tính bằng Forward Kinematics, nhân dồn `OFFSET` (đã xoay đúng theo hệ khớp cha) qua toàn bộ chain. **Hiểu đúng:** đọc file BVH đúng cách luôn cần một bước dựng cây + tính FK, không phải chỉ đọc số thô.
2. **Hiểu nhầm: "LAFAN1 và AMASS có thể dùng thay thế cho nhau tuỳ thích vì cả hai đều là 'dữ liệu chuyển động người'".** Vì sao sai: hai định dạng có tính chất khác nhau về chất — BVH không mesh/không tách shape-pose, SMPL-X có cả hai; một pipeline cần biết hình dạng bề mặt (ví dụ để tính tương tác tay-vật thể chi tiết) sẽ cần SMPL-X, còn một pipeline chỉ cần khung xương chuyển động mượt/chất lượng cao (ví dụ benchmark motion-tracking RL) có thể dùng LAFAN1 hiệu quả hơn dù thiếu mesh. **Hiểu đúng:** lựa chọn dataset phụ thuộc đúng nhu cầu thông tin của bài toán cụ thể, không phải "dataset nào lớn hơn/mới hơn thì dùng cái đó".

## 🏗️ Ví dụ minh hoạ trong dự án này

LAFAN1 là một trong ba nguồn dữ liệu retargeting chính của pipeline dự án (cùng AMASS/OMOMO và BONES-SEED — bài giảng tiếp theo):

```text
LAFAN1 (.bvh, Ubisoft La Forge)
        │
        ▼
GMR --bvh_lafan1_to_g1.json--> Unitree G1 (đã học ở BAI-GIANG-gmr-kien-truc-pipeline.md)
        │                       (config human_scale_table/ik_match_table
        │                        được thiết kế riêng cho định dạng BVH-LAFAN1)
        ▼
Reference motion cho:
  - Benchmark motion-tracking RL (BeyondMimic, xem Cập nhật hiện đại)
  - Huấn luyện policy ở 01-whole-body-control/, 04-imitation-learning-rl/
```

Chính config `bvh_lafan1_to_g1.json` đã xuất hiện cụ thể ở bài Skeleton mapping (`02-motion-retargeting/`) — nay ta đã hiểu rõ *dữ liệu đầu vào* của config đó (một file `.bvh` LAFAN1) có cấu trúc chính xác ra sao trước khi đi vào bước skeleton mapping/scale.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **LAFAN1 đã trở thành benchmark chuẩn cho motion-tracking RL humanoid, vượt xa mục đích gốc (animation in-betweening).** BeyondMimic — *"From Motion Tracking to Versatile Humanoid Control via Guided Diffusion"* ([arXiv:2508.08241](https://arxiv.org/pdf/2508.08241)) dùng một **tập con "high-dynamic" của LAFAN1** (các động tác chạy nước rút, lộn nhào, xoay người) làm benchmark khó để đánh giá policy tracking trên Unitree G1 thật, báo cáo success rate **70.04%** trên tập con này — cho thấy ngay cả các policy hiện đại nhất cũng chưa "giải quyết xong" LAFAN1, dù dataset này đã tồn tại từ 2020. [arXiv:2508.08241](https://arxiv.org/pdf/2508.08241)
2. **Paper GMR chính thức dùng LAFAN1 làm nền tảng thực nghiệm chính để cô lập ảnh hưởng của chất lượng retargeting.** Araujo et al. (2025), *Retargeting Matters* ([arXiv:2510.02252](https://arxiv.org/abs/2510.02252)) huấn luyện policy qua BeyondMimic trên LAFAN1 để so sánh GMR với các baseline khác — việc chọn LAFAN1 (thay vì AMASS) cho thí nghiệm này liên quan trực tiếp tới chất lượng mocap rất cao và ít nhiễu của LAFAN1 (đã nêu ở mục Định nghĩa), giúp kết quả so sánh phản ánh đúng ảnh hưởng của *retargeting*, không bị nhiễu lẫn bởi chất lượng dữ liệu nguồn kém.
3. **Xu hướng chung:** dữ liệu BVH chất lượng game AAA (LAFAN1) và dữ liệu SMPL-X quy mô lớn (AMASS/BONES-SEED) đang được dùng **song song, bổ sung cho nhau** trong các benchmark humanoid 2025-2026 — không có dấu hiệu một định dạng "thắng thế" hoàn toàn, vì mỗi định dạng phục vụ mục đích khác nhau (LAFAN1: chất lượng mượt/độ khó động tác; SMPL-X: quy mô, khả vi, có mesh).

## ❓ Câu hỏi tự kiểm tra

1. File BVH có lưu trực tiếp vị trí toàn cục của khớp `LeftLeg` không? Nếu không, làm sao tính được?
   <details><summary>Gợi ý đáp án</summary>Không — chỉ `ROOT` có vị trí toàn cục trực tiếp trong `MOTION`; vị trí của `LeftLeg` phải tính bằng Forward Kinematics, nhân dồn `OFFSET` (đã xoay đúng hệ khớp cha) từ root xuống.</details>
2. Trong ví dụ tính tay, nếu bỏ qua việc xoay vector `OFFSET` của `LeftLeg` theo góc `10°` của `LeftUpLeg`, sai số vị trí là bao nhiêu?
   <details><summary>Gợi ý đáp án</summary>≈3.22 cm, tính bằng khoảng cách Euclid giữa `(3.5, 69.4)` (sai, không xoay) và `(6.7116, 69.6812)` (đúng, có xoay).</details>
3. Nêu 3 khác biệt căn bản giữa BVH và SMPL/SMPL-X.
   <details><summary>Gợi ý đáp án</summary>(1) Không mesh vs có mesh; (2) góc Euler tuyệt đối/gimbal lock vs axis-angle; (3) không tách shape/pose (OFFSET cố định) vs tách β/θ độc lập.</details>
4. Vì sao LAFAN1 được chọn làm benchmark thực nghiệm chính trong paper GMR thay vì AMASS, dù AMASS có quy mô lớn hơn nhiều?
   <details><summary>Gợi ý đáp án</summary>Vì LAFAN1 có chất lượng mocap rất cao, ít nhiễu (chuẩn sản xuất game AAA) — giúp cô lập rõ ảnh hưởng của chất lượng retargeting lên policy, không bị nhiễu lẫn bởi chất lượng dữ liệu nguồn kém như một số bộ mocap học thuật cũ trong AMASS.</details>
5. Vì sao dùng góc Euler tuyệt đối trong BVH có thể gây "gimbal lock", và điều này ảnh hưởng gì tới việc đọc/ghi dữ liệu chuyển động nhanh?
   <details><summary>Gợi ý đáp án</summary>Gimbal lock xảy ra khi hai trong ba trục xoay Euler trùng phương, làm mất một bậc tự do xoay tại đúng cấu hình đó — với chuyển động nhanh/phức tạp (nhào lộn, xoay người), xác suất đi qua các cấu hình gần gimbal lock cao hơn, có thể gây nội suy/tính toán sai lệch cục bộ nếu không xử lý cẩn thận (một lý do các hệ thống hiện đại thường ưu tiên quaternion/axis-angle hơn Euler thuần khi có thể).</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** với cùng cấu trúc chain `Hips → LeftUpLeg (OFFSET (4.0, −2.5)) → LeftLeg (OFFSET (0.0, −20.0))`, `Hips` ở `(5, 95)` cm không xoay, `LeftUpLeg` xoay `−15°`. Tính vị trí toàn cục của `LeftLeg` (nhớ xoay vector offset của `LeftLeg` theo đúng góc của `LeftUpLeg` trước khi cộng).
2. **Đọc dữ liệu thật:** tải một file `.bvh` mẫu từ repo chính thức [ubisoft/ubisoft-laforge-animation-dataset](https://github.com/ubisoft/ubisoft-laforge-animation-dataset), mở bằng trình soạn thảo văn bản, xác định: số khớp trong `HIERARCHY`, giá trị `Frame Time` (suy ra tần số Hz), và tổng số `Frames` — so sánh với con số tổng "496,672 khung @30Hz" đã học, tính xem file mẫu này chiếm bao nhiêu % tổng số khung của cả dataset.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

BVH là định dạng mocap dạng cây khớp thuần tuý (không mesh), gồm `HIERARCHY` khai báo cấu trúc xương + chiều dài đoạn cố định (`OFFSET`) và `MOTION` ghi góc Euler local theo từng khung hình — muốn biết vị trí toàn cục bất kỳ khớp nào phải tự tính Forward Kinematics, xoay đúng `OFFSET` theo hệ toạ độ khớp cha trước khi cộng dồn dọc chain, như ví dụ tính tay minh hoạ (bỏ qua bước xoay này gây sai số vài centimet, khuếch đại theo chain dài hơn). LAFAN1 (Harvey et al. 2020, Ubisoft La Forge) là bộ dữ liệu BVH chất lượng sản xuất game AAA (496.672 khung @30Hz, 5 diễn viên, 77 sequence, 15 theme động tác) ban đầu phục vụ bài toán motion in-betweening, nhưng nay đã trở thành benchmark chuẩn cho retargeting (GMR/SOMA-retargeter đều hỗ trợ trực tiếp) và motion-tracking RL humanoid (BeyondMimic báo cáo success rate 70.04% trên tập con high-dynamic của chính LAFAN1) — một minh chứng cho việc dữ liệu chất lượng cao từ ngành công nghiệp game có thể tái sử dụng hiệu quả cho nghiên cứu robot học dù định dạng (BVH) khác hẳn triết lý dữ liệu học thuật (SMPL/SMPL-X, AMASS).
