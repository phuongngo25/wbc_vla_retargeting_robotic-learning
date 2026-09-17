# Bài giảng: Skeleton mapping — ánh xạ khớp, bone chain và scale

*(Thuộc mảng: Motion Retargeting)*

## 🎯 Mục tiêu bài học

- Giải thích được skeleton mapping biến hai bộ xương khác nhau thành một tập correspondence có thể dùng cho retargeting như thế nào.
- Phân biệt joint, body/link, bone chain, root và end-effector.
- Xây dựng được bảng ánh xạ tối thiểu cho hai tay, hai chân và thân của một humanoid.
- Tính tay được scale factor theo từng đoạn và vị trí target đã scale.
- Xử lý có chủ đích trường hợp robot thiếu hoặc thừa bậc tự do.
- Nhận diện được hạn chế của sparse handcrafted mapping và các hướng thay thế 2025–2026.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Sau khi biết không thể copy góc khớp, ta cần một “từ điển” nói rõ phần nào của người tương ứng với phần nào của robot.

Đó là vai trò của **skeleton mapping**.

Nó đứng giữa dữ liệu người và bộ giải IK:

```text
motion người
   │
   ▼
skeleton mapping
   │  correspondence + vị trí/hướng target đã scale
   ▼
IK trên robot
   │
   ▼
góc khớp robot
```

Nếu mapping sai, solver phía sau vẫn có thể hội tụ — nhưng hội tụ về một chuyển động sai nghĩa.

Ví dụ, nếu cổ tay người bị ánh xạ nhầm sang link khuỷu robot, bộ giải có thể đưa khuỷu tới vị trí bàn tay và báo lỗi toán học nhỏ, trong khi tư thế quan sát được hoàn toàn sai.

Vì vậy skeleton mapping không phải thao tác đổi tên file.

Nó là quyết định semantic và hình học về:

- điểm nào cần tương ứng;
- chuỗi khớp nào tạo ra điểm đó;
- tỷ lệ nào phải áp dụng;
- thông tin nào được giữ, gộp hoặc bỏ.

## 🧠 Trực giác

### Góc nhìn 1: Từ điển giữa hai ngôn ngữ không có từ tương ứng 1-1

Hãy dịch một đoạn văn giữa hai ngôn ngữ.

Một số từ có tương ứng trực tiếp:

```text
pelvis ↔ pelvis
left elbow ↔ left elbow link
```

Một số ý trong ngôn ngữ nguồn phải dùng cả cụm từ ở ngôn ngữ đích.

Một số ý không tồn tại ở ngôn ngữ đích và phải lược bỏ.

Ngược lại, ngôn ngữ đích có thể yêu cầu thông tin mà câu nguồn không cung cấp.

Skeleton mapping cũng vậy:

- joint chính có thể map trực tiếp;
- nhiều đốt sống người có thể gộp vào một waist joint;
- ngón tay có thể bị bỏ nếu robot không có dexterous hand;
- DoF thừa của robot có thể giữ pose mặc định.

Phép loại suy đúng ở chỗ correspondence là **semantic**, không chỉ dựa vào chuỗi ký tự tên gọi.

Giới hạn của loại suy:

- ngôn ngữ không có chiều dài xương và hệ toạ độ 3D;
- skeleton mapping phải xử lý cả position, orientation, scale và kinematic reachability;
- một mapping hợp nghĩa vẫn có thể không khả thi về hình học.

### Góc nhìn 2: Adapter giữa hai API

Giả sử API nguồn trả dữ liệu:

```text
HumanPose {
  hips,
  spine_segments[...],
  left_hand,
  right_hand,
  fingers[...]
}
```

API robot lại nhận:

```text
RobotTargets {
  pelvis,
  torso,
  left_wrist_link,
  right_wrist_link,
  ...
}
```

Một adapter phải:

1. nối đúng field;
2. đổi đơn vị/hệ quy chiếu;
3. tổng hợp field khi schema khác nhau;
4. đặt default khi nguồn thiếu;
5. validate output trước khi gửi.

Skeleton mapping chính là adapter hình học giữa hai embodiment.

Phép loại suy đúng ở pipeline chuyển schema và việc cần policy rõ ràng cho field thiếu/thừa.

Giới hạn của loại suy:

- adapter dữ liệu thông thường không làm thay đổi reachability;
- trong robot, target của một body phụ thuộc toàn bộ chain cha;
- scale một điểm sai có thể làm nhiều task IK xung đột.

## 📐 Định nghĩa chính xác

### 1. Skeleton như một cây có hướng

Một skeleton có thể xem là cây:

```text
S = (V, E)
```

trong đó:

- `V` là tập joint/body;
- `E` là quan hệ cha–con;
- mỗi cạnh mang offset/chiều dài đoạn xương;
- root là nút không có cha.

Một pose gồm:

- root translation/orientation;
- local hoặc global orientation của các joint/body;
- từ đó Forward Kinematics tính global transform.

### 2. Correspondence

Gọi skeleton người là:

```text
S_H = (V_H, E_H)
```

và skeleton robot là:

```text
S_R = (V_R, E_R)
```

Skeleton mapping chọn một tập correspondence:

```text
M = {(hᵢ, rᵢ, wᵢᵖ, wᵢʳ, oᵢᵖ, oᵢʳ)}
```

với:

- `hᵢ ∈ V_H`: body/joint người;
- `rᵢ ∈ V_R`: body/link robot;
- `wᵢᵖ`: trọng số position;
- `wᵢʳ`: trọng số orientation;
- `oᵢᵖ`: position offset;
- `oᵢʳ`: rotation offset.

Nguồn cô đọng của repo chỉ yêu cầu bảng tên correspondence, bone chain và scale.

Các trường weight/offset ở trên được đưa vào để giải thích mapping thực tế của GMR hiện hành, dựa trên code/config chính thức được tra cứu.

### 3. Bone chain

**Bone chain** là đường đi liên tiếp từ một root cục bộ đến end-effector:

```text
C = (v₀, v₁, ..., vₖ)
```

Ví dụ:

```text
tay trái:  shoulder → elbow → wrist
chân trái: hip      → knee  → ankle
thân:      pelvis   → chest → head
```

Một chain giúp tổ chức:

- correspondence;
- scale theo đoạn;
- target end-effector;
- phân tích phần DoF bị thiếu.

Bone chain không có nghĩa các chi hoàn toàn độc lập.

Hai chân và hai tay vẫn nối với root/pelvis; thay đổi root làm mọi global target thay đổi.

### 4. Scale theo tỷ lệ xương

Với một đoạn nguồn có chiều dài `L_H` và đoạn robot tương ứng có chiều dài `L_R`, scale factor cơ bản là:

```text
s = L_R / L_H
```

Với vector tương đối từ joint cha tới joint con ở người:

```text
v_H = p_H,child − p_H,parent
```

target đã scale đơn giản là:

```text
v_target = s v_H
```

và:

```text
p_target,child = p_target,parent + v_target
```

Trong hệ thực tế, scale có thể:

- khác nhau giữa các body part;
- áp trong local frame của root;
- dùng bảng hệ số thủ công;
- đi kèm orientation/position offset.

### 5. Thiếu và thừa DoF

Nếu robot có ít DoF hơn nguồn:

```text
|V_R hoặc DoF_R| < |V_H hoặc DoF_H|
```

mapping phải chọn một trong các cách:

- bỏ thông tin;
- gộp chuyển động vào một joint robot;
- xấp xỉ bằng chain khác.

Nếu robot có DoF mà nguồn không quan sát:

- giữ pose mặc định;
- dùng heuristic riêng;
- điều khiển bằng module khác.

Đây là quyết định có chủ đích, không phải lỗi parser.

## ⚙️ Cơ chế hoạt động — từng bước

### Bước 1: Chuẩn hoá tên và hệ quy chiếu

Hai skeleton có thể dùng tên khác:

```text
Human: Hips, LeftUpLeg, LeftLeg, LeftFoot
Robot: pelvis, left_hip_yaw_link, left_knee_link, left_ankle_roll_link
```

Cần xác định:

- trục nào là forward/up;
- đơn vị là mét hay centimet;
- quaternion dùng thứ tự nào;
- transform đang ở local hay global frame.

Mapping đúng tên nhưng sai frame vẫn tạo target sai.

### Bước 2: Chọn landmark/body chính

Không map mọi joint.

Một tập tối thiểu thường tập trung vào:

- pelvis/root;
- torso/chest/head;
- shoulder/elbow/wrist;
- hip/knee/ankle.

Các landmark phải đủ để giữ silhouette và contact quan trọng.

### Bước 3: Lập bảng correspondence

Ví dụ khái niệm:

| Human body | Robot body | Vai trò |
|---|---|---|
| `Hips` | `pelvis` | root |
| `LeftArm` | `left_shoulder_roll_link` | upper arm landmark |
| `LeftForeArm` | `left_elbow_link` | elbow landmark |
| `LeftHand` | `left_wrist_yaw_link` | hand end-effector |
| `LeftUpLeg` | `left_hip_roll_link` | upper leg landmark |
| `LeftLeg` | `left_knee_link` | knee landmark |
| `LeftFoot` | `left_ankle_roll_link` | foot end-effector |

Tên cụ thể phụ thuộc format và robot.

Không nên suy ra correspondence chỉ bằng tên gần giống.

### Bước 4: Chia bone chain

```text
                    head
                     │
left_wrist ─ elbow ─ torso ─ elbow ─ right_wrist
                     │
                   pelvis
                   /    \
          left_knee      right_knee
              │              │
         left_ankle      right_ankle
```

Từ cây trên, ta định nghĩa các chain có semantic:

```text
C_arm_L  = shoulder_L → elbow_L → wrist_L
C_arm_R  = shoulder_R → elbow_R → wrist_R
C_leg_L  = hip_L      → knee_L  → ankle_L
C_leg_R  = hip_R      → knee_R  → ankle_R
C_torso  = pelvis     → chest   → head
```

### Bước 5: Đo chiều dài và tạo scale table

Đo trên rest pose hoặc skeleton definition:

```text
L = ||p_child − p_parent||₂
```

Tạo bảng:

| Segment | `L_H` | `L_R` | `s=L_R/L_H` |
|---|---:|---:|---:|
| upper arm | ... | ... | ... |
| forearm | ... | ... | ... |
| thigh | ... | ... | ... |
| shin | ... | ... | ... |

Không nhất thiết dùng một scale toàn thân duy nhất.

Nếu tỷ lệ upper arm và forearm khác nhau, một global scale không thể đồng thời khớp cả hai.

### Bước 6: Scale trong frame thích hợp

Một pipeline an toàn về mặt ý niệm:

```text
global human position
        │ trừ root
        ▼
root-relative human vector
        │ scale theo body/chain
        ▼
scaled local target
        │ cộng scaled robot root
        ▼
global target cho robot
```

Code GMR hiện hành thực hiện đúng mẫu tổng quát này cho position:

- trừ human root để đưa body point về local frame;
- nhân `human_scale_table[body_name]`;
- cộng scaled root để trở lại global frame.

### Bước 7: Thêm offset về position/orientation

Rest pose và quy ước frame của hai model có thể khác nhau.

Ví dụ:

- người ở T-pose;
- robot model ở pose khác;
- local axis của wrist không cùng hướng.

Vì vậy mapping thực tế có thể cần:

```text
p_target = p_scaled + R_current o_position
R_target = R_human R_offset
```

Offset phải được hiểu theo đúng frame.

### Bước 8: Xử lý vùng thiếu DoF

Ví dụ người có nhiều đốt sống, robot có thân cứng:

```text
human spine1 → spine2 → spine3
                    │
                    ▼
robot torso (một link hoặc rất ít waist DoF)
```

Các lựa chọn:

- map chest/head và bỏ chi tiết uốn từng đốt;
- dồn một phần orientation vào waist;
- để hip/knee bù cho động tác cúi.

Mỗi lựa chọn mất thông tin theo cách khác nhau.

### Bước 9: Xử lý DoF robot không có input

Ví dụ robot có dexterous hand nhưng dataset không có finger mocap.

Các finger joint không nên nhận giá trị rác.

Ta có thể:

- giữ neutral pose;
- dùng grasp heuristic;
- nhận lệnh từ nhánh manipulation riêng.

### Bước 10: Xuất target cho solver kế tiếp

Output của skeleton mapping không nhất thiết là góc robot.

Trong pipeline nguồn, output quan trọng là:

```text
body/link robot → target position + target orientation + weights
```

IK dùng target này để tìm configuration robot.

Chi tiết Jacobian/differential IK và FABRIK nằm ở hai bài giảng riêng kế tiếp.

### Sơ đồ cơ chế đầy đủ

```text
┌─────────────────────────────────────────┐
│ Human skeleton + pose                   │
│ names · hierarchy · global transforms   │
└──────────────────┬──────────────────────┘
                   ▼
┌─────────────────────────────────────────┐
│ 1. Chuẩn hoá axis / unit / frame        │
└──────────────────┬──────────────────────┘
                   ▼
┌─────────────────────────────────────────┐
│ 2. Correspondence table                 │
│ human body ↔ robot body                 │
└──────────────────┬──────────────────────┘
                   ▼
┌─────────────────────────────────────────┐
│ 3. Bone chains + scale table            │
│ root-relative, per-body/per-chain       │
└──────────────────┬──────────────────────┘
                   ▼
┌─────────────────────────────────────────┐
│ 4. Position/orientation offsets         │
│ + policy cho DoF thiếu/thừa             │
└──────────────────┬──────────────────────┘
                   ▼
┌─────────────────────────────────────────┐
│ Robot task targets                      │
│ position · orientation · weights        │
└──────────────────┬──────────────────────┘
                   ▼
                IK solver
```

## 🧮 Ví dụ tính tay / walkthrough cụ thể

Các số dưới đây là ví dụ minh hoạ tự chọn, không phải kích thước thật của G1 hay một người cụ thể.

### Bài toán

Ta retarget tay trái từ người sang robot.

Rest-pose segment lengths:

```text
Người:
  upper arm L_H1 = 0.30 m
  forearm   L_H2 = 0.25 m

Robot:
  upper arm L_R1 = 0.24 m
  forearm   L_R2 = 0.15 m
```

Tại một frame, các vector local quan sát được là:

```text
vai → khuỷu người: v_H1 = (0.18, 0.24, 0.00) m
khuỷu → cổ tay:    v_H2 = (0.00, 0.15, 0.20) m
```

Kiểm tra chiều dài:

```text
||v_H1|| = √(0.18² + 0.24²)
         = √(0.0324 + 0.0576)
         = √0.09
         = 0.30 m

||v_H2|| = √(0.15² + 0.20²)
         = √(0.0225 + 0.04)
         = √0.0625
         = 0.25 m
```

### Bước 1: Tính scale factor từng đoạn

```text
s₁ = L_R1 / L_H1
   = 0.24 / 0.30
   = 0.8

s₂ = L_R2 / L_H2
   = 0.15 / 0.25
   = 0.6
```

Hai factor khác nhau.

Đây là lý do một scale toàn cánh tay duy nhất có thể làm sai tỷ lệ nội bộ.

### Bước 2: Scale vector từng đoạn

```text
v_R1,target = s₁ v_H1
            = 0.8(0.18, 0.24, 0.00)
            = (0.144, 0.192, 0.000) m

v_R2,target = s₂ v_H2
            = 0.6(0.00, 0.15, 0.20)
            = (0.000, 0.090, 0.120) m
```

Kiểm tra:

```text
||v_R1,target|| = √(0.144² + 0.192²)
                = √(0.020736 + 0.036864)
                = √0.0576
                = 0.24 m

||v_R2,target|| = √(0.09² + 0.12²)
                = √(0.0081 + 0.0144)
                = √0.0225
                = 0.15 m
```

Scale đã bảo toàn hướng từng đoạn và đổi đúng chiều dài robot.

### Bước 3: Tính target khuỷu và cổ tay

Giả sử vai robot ở:

```text
p_R,shoulder = (0.10, 0.20, 1.20) m
```

Target khuỷu:

```text
p_R,elbow
= p_R,shoulder + v_R1,target
= (0.10, 0.20, 1.20) + (0.144, 0.192, 0.000)
= (0.244, 0.392, 1.200) m
```

Target cổ tay:

```text
p_R,wrist
= p_R,elbow + v_R2,target
= (0.244, 0.392, 1.200) + (0.000, 0.090, 0.120)
= (0.244, 0.482, 1.320) m
```

### Bước 4: So sánh với global scale duy nhất

Nếu dùng scale theo tổng chiều dài:

```text
s_global = (0.24 + 0.15) / (0.30 + 0.25)
         = 0.39 / 0.55
         ≈ 0.7091
```

Upper arm theo global scale:

```text
v'_R1 = 0.7091(0.18, 0.24, 0)
      ≈ (0.1276, 0.1702, 0) m
```

Chiều dài:

```text
||v'_R1|| = 0.7091 × 0.30
          ≈ 0.2127 m
```

Nhưng upper arm robot thật dài `0.24 m`.

Sai số chiều dài target:

```text
0.24 − 0.2127 = 0.0273 m = 2.73 cm
```

Forearm theo global scale:

```text
||v'_R2|| = 0.7091 × 0.25
          ≈ 0.1773 m
```

Nhưng forearm robot dài `0.15 m`.

Sai số:

```text
0.1773 − 0.15 = 0.0273 m = 2.73 cm
```

Global scale giữ tổng chiều dài nhưng phân bổ sai chiều dài giữa hai đoạn.

IK có thể phải bẻ góc hoặc chấp nhận lỗi để dung hoà các target trung gian.

### Bước 5: Thêm position offset

Giả sử frame convention khiến wrist target cần offset local:

```text
o_wrist = (0.00, −0.02, 0.01) m
```

Nếu ví dụ đơn giản coi local frame trùng global frame:

```text
p'_R,wrist = p_R,wrist + o_wrist
           = (0.244, 0.482, 1.320)
           + (0.000, −0.020, 0.010)
           = (0.244, 0.462, 1.330) m
```

Trong code thật, offset local phải được quay bằng orientation hiện tại trước khi cộng.

### Kết luận walkthrough

Ví dụ cho thấy skeleton mapping thực hiện ba việc khác nhau:

1. correspondence xác định “body nào đi với body nào”;
2. per-segment scale sửa morphology;
3. offset sửa quy ước frame/rest pose.

Nó chưa giải góc robot.

Đó là nhiệm vụ của IK ở bài kế tiếp.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Sparse skeleton mapping thủ công | Global uniform scale | Learned dense correspondence |
|---|---|---|---|
| Đơn vị correspondence | Một số joint/body pair | Không cần pair chi tiết | Nhiều điểm trên bề mặt |
| Scale | Theo body/chain | Một hệ số toàn thân | Học/ước lượng qua hình học bề mặt |
| Dễ hiểu và debug | Cao | Rất cao | Thấp hơn |
| Công sức thêm robot mới | Phải tạo/tune config | Thấp nhưng chất lượng hạn chế | Cần model/data/pipeline học |
| Giữ chi tiết pose | Sparse | Thấp | Dense hơn |
| Phụ thuộc tên skeleton | Cao | Trung bình | Mục tiêu là giảm phụ thuộc |
| Phù hợp hiện tại trong repo | Có, GMR/SOMA | Chỉ làm baseline đơn giản | Hướng nghiên cứu mới |

### Mapping khác IK ở đâu?

| Skeleton mapping | IK |
|---|---|
| Xác định target tương ứng | Tìm configuration robot đạt target |
| Xử lý semantic và scale | Xử lý bài toán động học ngược |
| Thường cấu hình một lần cho cặp format–robot | Chạy ở mỗi frame |
| Output là task targets | Output là root/joint configuration |

### Mapping khác constraint preservation ở đâu?

Constraint preservation trả lời:

```text
Điều gì cần giữ?
```

Skeleton mapping trả lời:

```text
Điểm nguồn nào tương ứng điểm đích nào,
và target phải đổi tỷ lệ ra sao?
```

Hai khái niệm liên quan nhưng không đồng nhất.

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

### 1. “Tên giống nhau thì map được”

Vì sao sai:

Tên không đảm bảo cùng semantic, frame hay vị trí trên chain.

Hiểu đúng:

Kiểm tra hierarchy, rest pose, axes và vai trò vật lý của body.

### 2. “Một scale factor cho toàn thân là đủ”

Vì sao sai:

Hai cơ thể có thể cùng chiều cao nhưng khác tỷ lệ tay/thân/chân.

Hiểu đúng:

Scale theo body part hoặc skeleton calibration khi cần giữ tỷ lệ nội bộ.

### 3. “Scale global position trực tiếp quanh world origin”

Vì sao sai:

Điều đó scale cả vị trí nhân vật trong thế giới, không chỉ morphology.

Hiểu đúng:

Thường chuyển sang root-relative/local frame, scale vector cơ thể rồi đưa lại global frame.

### 4. “Bone chains hoàn toàn độc lập”

Vì sao sai:

Các chain chia sẻ pelvis/torso và một configuration robot duy nhất.

Hiểu đúng:

Chia chain giúp tổ chức bài toán; solver đa task vẫn phải dung hoà toàn thân.

### 5. “Joint thừa có thể bỏ mặc”

Vì sao sai:

Giá trị không xác định có thể tạo pose không mong muốn hoặc discontinuity.

Hiểu đúng:

Cần neutral pose, heuristic hoặc controller riêng cho DoF không được quan sát.

### 6. “Mapping càng nhiều landmark càng luôn tốt”

Vì sao sai:

Các target dày nhưng không nhất quán có thể gây xung đột và tăng tuning cost.

Hiểu đúng:

Chọn landmark đủ thông tin, đặt weight theo semantic và kiểm tra residual từng task.

### 7. “Output mapping đã là motion robot”

Vì sao sai:

Mapping chỉ tạo correspondence/target.

Hiểu đúng:

Còn cần IK, constraints thời gian, contact và validation động lực học.

## 🏗️ Ví dụ minh hoạ trong dự án này

### GMR với LAFAN1 → G1

Config chính thức `bvh_lafan1_to_g1.json` cho thấy mapping không chỉ là tên:

- `human_root_name` là `Hips`;
- `robot_root_name` là `pelvis`;
- có `human_scale_table` riêng cho nhiều body;
- tay trong config được scale khác chân/thân;
- `ik_match_table1/2` chứa robot frame, human body, position/orientation weights và offsets.

Ví dụ từ config được tra cứu:

```text
Hips        scale 0.9
LeftUpLeg   scale 0.9
LeftLeg     scale 0.9
LeftArm     scale 0.75
LeftForeArm scale 0.75
LeftHand    scale 0.75
```

Các số này thuộc config cụ thể của GMR, không phải hằng số phổ quát cho mọi người hay mọi G1.

### Luồng trong code GMR

`motion_retarget.py` hiện hành:

1. đọc IK config theo cặp source format–target robot;
2. điều chỉnh scale table theo `actual_human_height / human_height_assumption` nếu có;
3. chuyển position về human-root local frame;
4. nhân scale theo body;
5. đưa về global frame;
6. áp position/orientation offsets;
7. đặt target cho các `mink.FrameTask`.

```text
dict human body → (global position, quaternion)
                    │
                    ▼
        root-relative local position
                    │
                    ▼
             per-body scaling
                    │
                    ▼
          offsets + task weights
                    │
                    ▼
             mink FrameTask targets
```

### Khi thêm robot mới

Cần chuẩn bị ít nhất:

- robot model và body names;
- root name;
- human↔robot match table;
- scale table;
- frame offsets;
- task weights;
- policy cho các DoF không được map.

Sau đó mới tune solver và kiểm tra motion.

## 🔥 Cập nhật hiện đại / SOTA gần đây

### 1. GMR 2025/ICRA 2026: mapping vẫn là config rõ ràng, nhưng scale cục bộ linh hoạt

GMR hiện thực skeleton mapping bằng config theo từng cặp input format–robot.

Code chính thức cho thấy `human_scale_table`, `ik_match_table1`, `ik_match_table2`, weights và offsets được dùng trực tiếp để tạo task target.

Paper *Retargeting Matters* mô tả GMR dùng **non-uniform local scaling** rồi two-stage optimization, nhằm giảm artifact do các cách scale baseline.

Nguồn:

- Araujo et al. (2025), [arXiv:2510.02252](https://arxiv.org/abs/2510.02252).
- [GMR `motion_retarget.py`](https://github.com/YanjieZe/GMR/blob/master/general_motion_retargeting/motion_retarget.py).
- [GMR config LAFAN1 → G1](https://github.com/YanjieZe/GMR/blob/master/general_motion_retargeting/ik_configs/bvh_lafan1_to_g1.json).

### 2. SPARK 2026: calibrate cả skeleton thay vì tune target rời rạc

Wang, Liao, Zhang, Ren, Sreenath và Xiong (2026) đề xuất SPARK.

Thay vì chỉ sửa task-space target bằng scale/offset heuristic, SPARK:

1. chuyển skeleton người thành URDF và generalized-coordinate trajectory;
2. calibrate human URDF theo kích thước robot;
3. replay trajectory trên skeleton đã calibrate;
4. tạo task-space references cho IK;
5. tinh chỉnh bằng progressive kinodynamic trajectory optimization.

Paper báo cáo URDF calibration giảm mean per-body position error so với GMR trên nhiều robot trong thí nghiệm của họ.

Đây là cập nhật trực tiếp cho phần scale của skeleton mapping: làm cho target nhất quán với cấu trúc xương đã hiệu chỉnh, thay vì tune từng target độc lập.

Nguồn:

- Hanwen Wang et al. (2026), [arXiv:2603.11480](https://arxiv.org/abs/2603.11480).

### 3. Human2Humanoid 2026: học feature phụ thuộc topology khi không có paired data

Huang et al. (2026) dùng:

- CycleGAN-style architecture;
- skeleton-aware graph convolutional network;
- morphology-invariant end-effector consistency loss;
- physics-aware feasibility constraints.

Mục tiêu là học retargeting cross-morphology từ dữ liệu không ghép cặp và vẫn giữ end-effector semantics/contact.

Nó không xoá nhu cầu hiểu skeleton; thay vào đó topology được mã hoá trong graph network và loss.

Nguồn:

- Tianchen Huang et al. (2026), [arXiv:2606.03476](https://arxiv.org/abs/2606.03476).

### 4. UMR tháng 9/2026: từ sparse joint pairs sang learned dense surface correspondence

Cao et al. (2026) chỉ ra hạn chế của handcrafted sparse keypoints/body-part pairs:

- phụ thuộc thiết kế semantic thủ công;
- khó scale qua nhiều source representation và robot morphology;
- sparse anchors thiếu hướng dẫn cho pose/contact chi tiết.

Unified Motion Retargeting (UMR) dùng exterior point clouds làm interface chung và học dense correspondence.

Dense geometric anchors sau đó dẫn constrained point-cloud matching optimization.

Đây là thay đổi granularity:

```text
mapping cổ điển:
  shoulder ↔ shoulder link
  wrist    ↔ wrist link

UMR:
  nhiều điểm bề mặt người ↔ nhiều điểm bề mặt robot
```

Nguồn:

- Hanyang Cao et al. (2026), [arXiv:2609.02134](https://arxiv.org/abs/2609.02134).

### 5. Xu hướng 2025–2026

Các nguồn trên cho thấy ba hướng cùng tồn tại:

```text
Handcrafted sparse mapping
  GMR / SOMA: dễ hiểu, dễ debug, cần config

Skeleton calibration
  SPARK: làm target nhất quán với morphology

Learned correspondence
  Human2Humanoid / UMR: giảm thiết kế pair thủ công,
  mở rộng qua morphology và surface interaction
```

Không có dấu hiệu mapping thủ công đã biến mất.

Nó vẫn là baseline thực dụng và có giá trị vì minh bạch.

Xu hướng mới tập trung giảm tuning, tăng tính tổng quát và giữ correspondence/contact chi tiết hơn.

## ❓ Câu hỏi tự kiểm tra

1. Skeleton mapping khác việc đổi tên joint đơn thuần ở những điểm nào?

   <details><summary>Gợi ý đáp án</summary>Nó phải xét hierarchy, semantic, frame convention, scale, offsets, task weights và chính sách cho DoF thiếu/thừa.</details>

2. Vì sao nên scale vector root-relative thay vì global position quanh world origin?

   <details><summary>Gợi ý đáp án</summary>Để thay đổi morphology tương đối quanh cơ thể mà không vô tình scale cả vị trí/đường đi của nhân vật trong thế giới.</details>

3. Trong ví dụ tính tay, vì sao global scale `0.7091` không đúng cho từng đoạn?

   <details><summary>Gợi ý đáp án</summary>Vì upper arm cần factor `0.8`, forearm cần `0.6`; global factor chỉ giữ tổng chiều dài và phân bổ sai chiều dài giữa các segment.</details>

4. Robot thân cứng nên xử lý nhiều spine joint của người thế nào?

   <details><summary>Gợi ý đáp án</summary>Phải chấp nhận mất/gộp thông tin: map landmark thân chính, dồn phần khả thi vào waist nếu có, hoặc để hông/chân xấp xỉ; lựa chọn cần được nêu rõ trong mapping.</details>

5. Output trực tiếp của skeleton mapping trong GMR là gì?

   <details><summary>Gợi ý đáp án</summary>Các target position/orientation đã scale và offset, kèm weights cho robot frame tasks; IK mới biến chúng thành robot configuration.</details>

6. UMR 2026 muốn khắc phục hạn chế nào của sparse mapping?

   <details><summary>Gợi ý đáp án</summary>Sự phụ thuộc vào correspondence thủ công, khó scale qua source/robot khác nhau và guidance quá thưa cho pose/contact bề mặt chi tiết.</details>

## 📝 Bài tập thực hành

1. **Tính tay:** dùng chain chân với `L_H,thigh=0.45 m`, `L_H,shin=0.40 m`, `L_R,thigh=0.32 m`, `L_R,shin=0.34 m`. Tính hai per-segment scale và global scale; với hai vector nguồn tự chọn đúng chiều dài, tính ankle target theo cả hai cách và đo sai lệch.
2. **Đọc config thật:** mở [GMR LAFAN1 → G1 config](https://github.com/YanjieZe/GMR/blob/master/general_motion_retargeting/ik_configs/bvh_lafan1_to_g1.json). Lập bảng ít nhất tám pair human–robot, ghi position weight, orientation weight, scale và offset; đánh dấu pair nào là root, intermediate landmark và end-effector.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

Skeleton mapping là lớp adapter semantic–hình học giữa skeleton người và robot: nó chọn correspondence cho các body quan trọng, tổ chức chúng thành bone chains, scale vector tương đối theo tỷ lệ morphology, sửa khác biệt frame bằng offsets và quyết định rõ thông tin nào được giữ, gộp, bỏ hoặc đặt mặc định khi DoF không khớp. Ví dụ tính tay cho thấy global scale có thể giữ tổng chiều dài nhưng vẫn phân bổ sai giữa upper arm và forearm, vì vậy per-part scaling hoặc skeleton calibration thường cần thiết. GMR hiện dùng bảng mapping/scale/offset minh bạch; SPARK 2026 calibrate URDF skeleton, còn Human2Humanoid và UMR 2026 chuyển dần sang correspondence học được hoặc dày trên bề mặt. Dù triển khai thay đổi, câu hỏi cốt lõi vẫn là: body nào tương ứng body nào, trong frame nào và với tỷ lệ nào, trước khi IK được phép giải góc robot.
