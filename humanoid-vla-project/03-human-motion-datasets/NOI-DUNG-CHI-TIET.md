# Nội dung chi tiết — Human Motion Datasets

> File này là phần "sách giáo trình": giải thích cơ chế, số liệu, và cách xây dựng của từng dataset/body model liệt kê ở `README.md`, đủ sâu để không cần tự mở paper gốc mới hiểu được. Các citation dùng lại đúng từ mục B của README — không tự chế thêm nguồn.

---

## 1. Body model SMPL là gì — cơ chế toán học ở mức trực giác

**Vấn đề SMPL giải quyết**: một hệ thống mocap (motion capture) chỉ ghi lại vị trí 3D của vài chục điểm đánh dấu (marker) gắn trên cơ thể diễn viên. Nhưng để dùng cho robot học, đồ hoạ, hay huấn luyện mô hình, ta cần một **mesh 3D đầy đủ** (hàng nghìn điểm bề mặt, tạo thành hình dạng cơ thể liên tục) tại mọi khung hình. SMPL (*Skinned Multi-Person Linear model*, Loper et al. 2015) là mô hình toán học biến một vài trăm con số thành mesh đó.

Trực giác, SMPL gồm 4 thành phần ghép lại:

1. **Template mesh**: một mesh cơ thể người trung bình cố định, khoảng 6,890 vertex (đỉnh) và ~23 khớp bên trong (skeleton ẩn dưới mesh). Đây là "cơ thể mặc định" trước khi biến dạng.
2. **Tham số shape β (beta)**: một vector ~10 số thực, mỗi chiều điều khiển một "hướng biến đổi hình dáng" học được từ dữ liệu quét 3D hàng nghìn người thật (cao/thấp, gầy/béo, tỷ lệ vai-hông...). Cộng β vào template theo các hướng này (gọi là *shape blend shapes*) cho ra mesh đúng hình dáng của một cá nhân cụ thể, ở tư thế "chuẩn" (T-pose hoặc A-pose).
3. **Tham số pose θ (theta)**: góc xoay 3D (dạng axis-angle) của từng khớp trong skeleton (vai, khuỷu tay, hông, gối...). Đây là phần "tư thế" — đổi θ theo từng khung hình mocap chính là cách biểu diễn chuyển động.
4. **Skinning (Linear Blend Skinning — LBS)**: sau khi có mesh hình dáng đúng (từ β) và góc khớp mong muốn (từ θ), thuật toán "gắn da" (skinning) sẽ di chuyển từng vertex của mesh theo tổ hợp trọng số của các khớp lân cận nó — giống hệt cách rigging nhân vật 3D trong game hoạt động: da ở gần khuỷu tay bị kéo theo chủ yếu bởi khớp khuỷu tay, một phần nhỏ bởi khớp vai.

**Vấn đề của LBS thuần tuý**: nếu chỉ dùng skinning tuyến tính, vùng khớp gập (khuỷu tay, đầu gối) sẽ bị "xẹp" hoặc méo phi thực tế (hiện tượng kinh điển "candy-wrapper artifact" trong đồ hoạ máy tính) — vì da thật không chỉ trượt theo khớp mà còn phồng lên do cơ bắp co, hoặc nhăn theo nếp gấp. SMPL sửa lỗi này bằng thành phần thứ 5:

5. **Pose-dependent blend shapes**: các hiệu chỉnh hình dạng mesh phụ thuộc vào chính tư thế θ hiện tại (ví dụ: khi khuỷu tay gập 90°, mesh được "phồng" thêm ở vùng cơ nhị đầu trước khi skinning). Các hiệu chỉnh này cũng được học từ dữ liệu scan 3D người thật ở nhiều tư thế khác nhau, không phải quy tắc thủ công.

Tóm gọn công thức trực giác:

```
mesh_cuoi_cung = LBS( template + shape_blend(β) + pose_blend(θ),  θ,  trọng_số_skinning )
```

Toàn bộ hàm này là **khả vi (differentiable)** — nghĩa là có thể tính đạo hàm ngược từ mesh về lại (β, θ). Đây là lý do SMPL trở thành "ngôn ngữ chung": thay vì mỗi dataset mocap lưu dữ liệu theo định dạng riêng (số lượng marker khác nhau, vị trí gắn marker khác nhau giữa các phòng lab), người ta có thể "fit" (khớp) bất kỳ dữ liệu mocap nào về một bộ (β, θ) sao cho mesh SMPL sinh ra khớp với marker quan sát được — biến mọi nguồn dữ liệu khác nhau thành cùng một không gian tham số nhỏ gọn, dễ lưu trữ, dễ đưa vào mạng neural, và dễ retarget sang skeleton robot sau này (xem `02-motion-retargeting/`).

> Nguồn: Loper, Mahmood, Romero, Pons-Moll, Black (2015), *"SMPL: A Skinned Multi-Person Linear Model"*, ACM ToG 34(6). [smpl.is.tue.mpg.de](https://smpl.is.tue.mpg.de)

---

## 2. SMPL-X mở rộng gì so với SMPL

SMPL nguyên bản chỉ mô hình hoá thân và tay/chân ở mức thô — bàn tay là một "khối cụt" không có ngón, khuôn mặt không biểu cảm. Với robot học tay khéo léo (dexterous manipulation) hay VLA cần hiểu ý định qua cử chỉ/biểu cảm, mức chi tiết này không đủ.

**SMPL-X** (*SMPL eXpressive*, Pavlakos et al. 2019) hợp nhất ba mô hình từng tách biệt (thân SMPL, bàn tay MANO, khuôn mặt FLAME) thành **một** mô hình thống nhất, dùng chung cùng một cơ chế toán học (blend skinning + pose-dependent blend shapes) đã mô tả ở mục 1, chỉ mở rộng số lượng vertex/khớp:

- **N = 10,475 vertex** (so với 6,890 của SMPL) — đủ chi tiết để biểu diễn từng ngón tay và các đường nét mặt.
- **K = 54 khớp** (so với 23 của SMPL) — bao gồm khớp cổ, hàm, hai mắt (điều khiển hướng nhìn), và các khớp ngón tay của cả hai bàn tay.
- Thêm tham số biểu cảm khuôn mặt (expression) tách biệt khỏi shape, cho phép cùng một người có nhiều biểu cảm khác nhau độc lập với hình dáng cơ thể.

**Vì sao AMASS và OMOMO chọn SMPL-X (hoặc biến thể SMPL-H — chỉ thêm tay, không thêm mặt) thay vì SMPL gốc**: dữ liệu mocap dùng để huấn luyện robot loco-manipulation cần vị trí bàn tay chính xác (cầm, nắm, đặt vật) — chi tiết mà SMPL thô không nắm bắt được. OMOMO đặc biệt phụ thuộc vào việc này vì bài toán của nó *là* tương tác tay–vật thể.

> Nguồn: Pavlakos, Choutas, Ghorbani, Bolkart, Osman, Tzionas, Black (2019), *"Expressive Body Capture: 3D Hands, Face, and Body from a Single Image"*, CVPR 2019. [CVF Open Access](https://openaccess.thecvf.com/content_CVPR_2019/html/Pavlakos_Expressive_Body_Capture_3D_Hands_Face_and_Body_From_a_CVPR_2019_paper.html)

---

## 3. AMASS — cách xây dựng, cấu trúc, ý nghĩa

**Bài toán**: tính đến 2019, cộng đồng nghiên cứu có hàng chục bộ mocap optical marker rải rác (CMU Mocap, HumanEva, KIT, TotalCapture, BioMotionLab...), mỗi bộ dùng số lượng marker khác nhau, vị trí gắn marker khác nhau, định dạng file khác nhau — không thể gộp chung để huấn luyện một mô hình duy nhất. AMASS (*Archive of Motion Capture as Surface Shapes*, Mahmood et al. 2019) giải quyết việc này bằng cách quy **tất cả** về cùng một tham số hoá SMPL/SMPL-X.

**Cách làm — MoSh/MoSh++**: kỹ thuật cốt lõi là **MoSh** (*Motion and Shape capture*), và phiên bản cải tiến **MoSh++** dùng riêng cho AMASS. Ý tưởng: với một khung hình mocap gồm N điểm marker 3D, thuật toán tối ưu hoá (tương tự bài toán inverse kinematics) tìm bộ tham số (β, θ) của SMPL sao cho khi gắn các "marker ảo" tương ứng lên đúng vị trí trên mesh SMPL và render ra, chúng khớp gần nhất có thể (tối thiểu hoá sai số bình phương) với vị trí marker thật đo được. Vì β (hình dáng) không đổi trong suốt một phiên mocap của một người, MoSh++ ước lượng β một lần từ nhiều khung hình rồi tối ưu θ (tư thế) riêng cho từng khung — kết quả là một chuỗi θ theo thời gian, tức chuyển động, cộng với soft-tissue coefficients (DMPL) mô phỏng dao động mô mềm khi chuyển động nhanh.

Kết quả: **15 bộ dữ liệu mocap optical-marker khác nhau** được "dịch" về cùng một định dạng SMPL/SMPL-X thống nhất — mỗi khung hình giờ chỉ là một vector (β, θ) ngắn gọn thay vì hàng chục toạ độ marker rời rạc.

**Số liệu chính thức** (theo abstract paper gốc và trang chính thức amass.is.tue.mpg.de, đã kiểm tra lại): **hơn 40 giờ mocap, hơn 11,000 chuyển động (motions), hơn 300 chủ thể (subjects)** — trích nguyên văn: *"more than 40 hours of motion data, spanning over 300 subjects, more than 11000 motions"*. Một số tài liệu thứ cấp (không phải bản thân paper/trang chủ) trích con số cụ thể hơn là **11,265 motions / 344 subjects** — con số này khớp về độ lớn với "over 300" nhưng chưa xác minh được trực tiếp trong văn bản chính thức, nên coi là **cần xác minh thêm** nếu cần độ chính xác tuyệt đối.

> ⚠️ **Phát hiện khi kiểm tra**: README.md gốc của dự án trước đây ghi AMASS có **"480+ người"** — con số này KHÔNG khớp với số liệu chính thức ("hơn 300 chủ thể"). Đã sửa lại trong README.md ở bước cuối của tài liệu này.

**Giới hạn/thiên lệch dữ liệu cần lưu ý**:
- Vì AMASS hợp nhất các bộ mocap học thuật cũ, phân bố loại chuyển động **lệch nặng về các động tác phòng lab** (đi bộ, các bài kiểm tra vận động cơ bản, một số điệu nhảy/thể thao) — thiếu các chuyển động sinh hoạt tự nhiên, chuyển động tương tác vật thể phức tạp (đây chính là khoảng trống mà OMOMO và LAFAN1 bổ sung một phần).
- Chất lượng khớp (fit) phụ thuộc vào chất lượng marker gốc — một số bộ mocap cũ có ít marker hơn, dẫn đến θ ước lượng kém chính xác hơn ở bàn tay/ngón chân.
- Không có object trajectory — chỉ có chuyển động người đơn thuần, không phù hợp trực tiếp cho loco-manipulation (khác với OMOMO).

> Nguồn: Mahmood, Ghorbani, Troje, Pons-Moll, Black (2019), *"AMASS: Archive of Motion Capture as Surface Shapes"*, ICCV 2019. [CVF Open Access](https://openaccess.thecvf.com/content_ICCV_2019/papers/Mahmood_AMASS_Archive_of_Motion_Capture_As_Surface_Shapes_ICCV_2019_paper.pdf) · [amass.is.tue.mpg.de](https://amass.is.tue.mpg.de)

---

## 4. OMOMO — nội dung, cách thu thập, bài toán phục vụ

**Bài toán "object motion guided human motion synthesis"**: cho trước **chỉ quỹ đạo chuyển động của một vật thể** (ví dụ: một chiếc ghế bị kéo lê, một hộp được nhấc lên rồi di chuyển), hãy **sinh ra chuyển động toàn thân của người** (bao gồm cả tư thế tay cầm nắm) sao cho hợp lý về mặt vật lý và tự nhiên. Đây là bài toán ngược lại với "sinh chuyển động vật thể từ chuyển động người" — và khó hơn, vì từ một quỹ đạo vật thể có thể có nhiều cách người cầm/di chuyển nó khác nhau đều hợp lý.

**Cách thu thập dữ liệu**: nhóm nghiên cứu (Stanford, Li et al. 2023) thực hiện mocap đồng thời cả người **và** vật thể khi diễn viên thao tác với 15 loại vật thể sinh hoạt hằng ngày (máy hút bụi, cây lau nhà, đèn cây, giá treo quần áo, vali, thùng nhựa, ghế gỗ, ghế trắng, bàn lớn, bàn nhỏ, hộp lớn, hộp nhỏ, thùng rác, màn hình...). Dữ liệu gồm: (1) chuyển động người dạng SMPL-H/SMPL-X, (2) quỹ đạo 6-DOF (vị trí + hướng) của vật thể theo thời gian, và (3) mesh 3D quét sẵn của từng vật thể để biết chính xác hình học lúc tiếp xúc (ví dụ điểm tay chạm vào vật). Tổng cộng khoảng **10 giờ** mocap tương tác toàn thân.

**Vì sao quan trọng cho loco-manipulation**: hầu hết dataset mocap thuần tuý (AMASS) hoặc dataset skeleton game (LAFAN1) chỉ ghi chuyển động người mà không biết người đang tương tác với vật gì, ở đâu. Với robot humanoid cần vừa di chuyển (locomotion) vừa thao tác vật thể (manipulation) — bài toán loco-manipulation trung tâm của VLA — cần dữ liệu **ràng buộc rõ giữa chuyển động cơ thể và trạng thái vật thể**, để mô hình học được quan hệ nhân-quả (tay ở đâu thì vật di chuyển thế nào), không chỉ học chuyển động người "trong chân không". OMOMO là một trong số ít dataset công khai cung cấp đúng cặp dữ liệu này.

> Nguồn: Li, Clegg, Mottaghi, Wu, Puig, Liu (2023), *"Object Motion Guided Human Motion Synthesis"*, ACM ToG (SIGGRAPH Asia 2023). [Trang dự án](https://lijiaman.github.io/projects/omomo/) · [GitHub](https://github.com/lijiaman/omomo_release)

---

## 5. LAFAN1 — định dạng BVH giải thích chi tiết

**BVH (BioVision Hierarchy)** là định dạng mocap "cây khớp + góc Euler", ra đời từ ngành công nghiệp game/phim hoạt hình từ thập niên 1990, khác hẳn triết lý của SMPL. Một file `.bvh` gồm 2 phần:

**Phần `HIERARCHY`**: khai báo cấu trúc skeleton dạng cây (tree), bắt đầu từ khớp gốc `ROOT` (thường là hông/pelvis). Mỗi khớp con được khai báo lồng trong khớp cha bằng từ khoá `JOINT`, kèm:
- `OFFSET x y z`: độ dịch chuyển tĩnh (chiều dài xương) từ khớp cha đến khớp này, tính bằng đơn vị chiều dài cố định — đây chính là "kích thước cơ thể" của skeleton, không đổi trong suốt file.
- `CHANNELS`: khai báo bậc tự do (degrees of freedom) của khớp này sẽ được ghi trong phần MOTION — thường là 3 kênh xoay (`Zrotation Yrotation Xrotation`), riêng khớp gốc có thêm 3 kênh dịch chuyển (`Xposition Yposition Zposition`) để ghi vị trí toàn cục của cả cơ thể trong không gian.
- Kết thúc mỗi nhánh là `End Site` (đầu ngón tay, đỉnh đầu...) — điểm cuối không có khớp con.

**Phần `MOTION`**: bắt đầu bằng `Frames:` (tổng số khung hình) và `Frame Time:` (khoảng cách thời gian giữa 2 khung, ví dụ 1/30 giây cho 30Hz). Sau đó là các dòng dữ liệu số — mỗi dòng là **một khung hình**, mỗi cột là giá trị của một kênh đã khai báo ở HIERARCHY (theo đúng thứ tự khai báo), tức là **góc Euler** (độ) xoay của từng khớp tại khung hình đó.

**Khác biệt căn bản với SMPL**:
- BVH chỉ là **skeleton thuần** (xương + khớp), **không có mesh/bề mặt cơ thể** — không biết hình dáng da thịt, không thể render nhân vật trực tiếp mà không gắn thêm một mesh/rig riêng.
- BVH dùng **góc Euler tuyệt đối theo từng khớp** (dễ bị "gimbal lock" — hiện tượng mất một bậc tự do khi hai trục xoay trùng nhau), trong khi SMPL dùng axis-angle và có blend shapes để tránh méo mesh.
- BVH không có khái niệm "shape" tách biệt khỏi "pose" — chiều dài xương (OFFSET) đã cố định cho một skeleton cụ thể, muốn đổi sang người khác phải đổi cả file offset.
- Vì đơn giản và không phụ thuộc mesh, BVH là định dạng phổ biến bậc nhất trong pipeline game/phim công nghiệp — đây là lý do LAFAN1 (dữ liệu do Ubisoft La Forge sản xuất) dùng BVH thay vì SMPL.

**Nguồn gốc — bài toán "motion in-betweening"**: LAFAN1 được tạo ra để phục vụ paper *"Robust Motion In-Betweening"* (Harvey et al. 2020, SIGGRAPH/ACM ToG). Motion in-betweening là bài toán kinh điển trong animation: cho trước một tư thế bắt đầu (keyframe) và một tư thế kết thúc (keyframe) cách nhau vài chục/trăm khung hình, hãy **tự động sinh ra các khung hình ở giữa** sao cho chuyển động mượt và tự nhiên — giống việc animator chuyên nghiệp vẽ tay các khung trung gian giữa hai "pose chủ đạo". Bài toán này cực kỳ quan trọng trong sản xuất game (nhân vật phải chuyển mượt giữa hàng trăm animation clip theo thời gian thực) nhưng mạng neural học in-betweening cần dữ liệu mocap **chất lượng sản xuất thực tế** (không phải mocap phòng thí nghiệm rời rạc) để học được sự mượt mà và đa dạng phong cách chuyển động thật. Vì vậy Ubisoft tự thu thập LAFAN1 với quy trình mocap chuẩn game AAA: **5 diễn viên, 77 sequence, 15 chủ đề động tác (theme)** — đi bộ, chạy, nhảy múa, đánh nhau, ngã/đứng dậy, bò, nhắm súng (aiming), vượt chướng ngại vật (obstacles)... — tổng **496,672 khung hình ở 30Hz** (~4.6 giờ).

> Nguồn: Harvey, Yurick, Nowrouzezahrai, Pal (2020), *"Robust Motion In-Betweening"*, ACM ToG (SIGGRAPH) 39(4). [Ubisoft La Forge](https://www.ubisoft.com/en-us/studio/laforge/news/2NBPwJzPl3DwCAzGTav7Tg/robust-motion-inbetweening) · [GitHub Ubisoft](https://github.com/ubisoft/ubisoft-laforge-animation-dataset)

---

## 6. BONES-SEED — những gì đã xác minh được, và giới hạn thông tin công khai

> ⚠️ Đây là dataset công bố **gần đây (2026)**, thông tin công khai còn hạn chế. Mục này chỉ ghi lại đúng những gì xác minh được qua trang phát hành chính thức — **không suy diễn/bịa thêm chi tiết**.

**Ai công bố**: BONES-SEED do **Bones Studio** phát hành (không phải nội bộ NVIDIA như ghi chú cũ trong README — cần sửa lại, xem cập nhật ở bước cuối). Dataset được host công khai trên Hugging Face (`bones-studio/seed`), dạng **gated** — người dùng cần chấp nhận điều khoản license (`bones-seed-license`, liên hệ `licensing@bones.studio` để biết chi tiết) trước khi tải.

**Vai trò của NVIDIA trong dataset này**: NVIDIA đóng góp công cụ, không phải chủ sở hữu dữ liệu gốc — cụ thể là retarget SOMA và Unitree G1 (qua công cụ mã nguồn mở `NVIDIA/soma-retargeter`), và phần temporal segmentation (chia mỗi chuyển động thành các đoạn/pha có ý nghĩa kèm mốc thời gian) được tạo ra "cho dự án Kimodo" của NVIDIA. Không có bằng chứng công khai cho thấy BONES-SEED trực tiếp gắn với GR00T hay SONIC — mối liên hệ với pipeline WBC/VLA trong dự án này (nếu có) cần xác minh thêm khi tới Phase huấn luyện thực tế.

**Số liệu quy mô** (theo trang phát hành chính thức trên Hugging Face):
- **142,220 chuyển động** (71,132 chuyển động gốc + 71,088 bản mirror/lật gương — kỹ thuật tăng cường dữ liệu phổ biến: lật trái-phải một chuyển động tạo thêm một mẫu huấn luyện hợp lệ)
- **~288 giờ** dữ liệu (tính ở tốc độ khung hình 120fps)
- **522 diễn viên** (253 nữ / 269 nam), độ tuổi 17–71
- Cung cấp ở 3 định dạng: SOMA Uniform (BVH), SOMA Proportional (BVH), và **Unitree G1 (CSV, tương thích MuJoCo)** — tức đã có sẵn bản retarget sang robot G1, không cần tự retarget từ đầu.
- Có annotation ngôn ngữ tự nhiên (tới 6 mô tả văn bản mỗi chuyển động) và temporal segmentation — phù hợp trực tiếp cho huấn luyện VLA (liên hệ `06-vla-groot-sonic/`).

**Ý nghĩa cho pipeline**: việc BONES-SEED cung cấp sẵn bản retarget G1 ở quy mô hàng trăm nghìn chuyển động minh hoạ đúng luận điểm ở `01-whole-body-control/`: retargeting (từ chuyển động người sang skeleton robot cụ thể) phải được làm **trước, ở quy mô lớn, bằng pipeline tự động** (như SOMA retargeter) — không thể làm thủ công từng file khi cần huấn luyện policy RL/imitation learning trên hàng trăm giờ dữ liệu.

---

## 7. Bảng so sánh tổng hợp 5 dataset

| Dataset | Định dạng | Quy mô | Loại chuyển động | Ưu điểm cho retargeting/WBC | Nhược điểm/giới hạn |
|---|---|---|---|---|---|
| **AMASS** | SMPL/SMPL-X (β + θ tham số hoá) | >40 giờ, >11,000 motion, >300 chủ thể (15 bộ mocap hợp nhất) | Đa dạng nhất về nguồn nhưng thiên về động tác phòng lab (đi, chạy, bài kiểm tra vận động, một số thể thao) | Tham số hoá gọn, khả vi, dễ đưa vào mạng neural; là "mẫu số chung" mọi công cụ retargeting (GMR, SOMA) hỗ trợ sẵn | Thiếu tương tác vật thể; chất lượng fit phụ thuộc mocap gốc; không có annotation ngôn ngữ |
| **SMPL-X** | *(body model, không phải dataset)* | N=10,475 vertex, K=54 khớp | — | Chuẩn biểu diễn chung có tay + mặt, cần thiết khi retarget cần vị trí ngón tay chính xác (cầm nắm) | Không tự nó có dữ liệu chuyển động — chỉ là "khuôn" để AMASS/OMOMO đổ dữ liệu vào |
| **OMOMO** | SMPL-H/SMPL-X + quỹ đạo vật thể + mesh 3D vật thể | ~10 giờ, 15 loại vật thể | Tương tác người–vật thể (loco-manipulation): nâng, kéo, mang | Duy nhất trong nhóm có ràng buộc người↔vật rõ ràng — thiết yếu để học loco-manipulation | Quy mô nhỏ (10 giờ so với hàng chục/trăm giờ của AMASS/BONES-SEED); chỉ 15 vật thể, chưa đa dạng đủ cho generalization rộng |
| **LAFAN1** | BVH (skeleton + góc Euler, không mesh) | 496,672 khung @30Hz (~4.6 giờ), 5 diễn viên, 77 sequence, 15 theme | Chuyển động chất lượng sản xuất game: đi, chạy, nhảy múa, đánh nhau, ngã, nhắm súng, vượt chướng ngại | Chất lượng mocap rất cao (chuẩn AAA game), mượt, ít nhiễu — benchmark phổ biến để test motion tracking RL | Không có mesh/shape; quy mô nhỏ hơn nhiều so với AMASS/BONES-SEED; không có tương tác vật thể |
| **BONES-SEED** | SOMA (BVH) + **Unitree G1 retarget sẵn (CSV)** | ~288 giờ, 142,220 chuyển động, 522 diễn viên | Đa dạng, có annotation ngôn ngữ + temporal segmentation | Quy mô lớn nhất trong nhóm; **đã retarget sẵn sang G1** — bỏ qua bước retargeting thủ công; có ngôn ngữ tự nhiên đi kèm, hợp huấn luyện VLA | Công bố gần đây, thông tin/độ ổn định license còn hạn chế; cần chấp nhận điều khoản gated access; ít paper/tài liệu học thuật kiểm chứng độc lập so với AMASS |

---

## Bảng thuật ngữ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| **Mocap (motion capture)** | Quá trình ghi lại chuyển động thật của diễn viên bằng cảm biến (thường là camera quang học + marker phản quang gắn trên cơ thể). |
| **Marker (optical marker)** | Điểm đánh dấu phản quang gắn trên cơ thể diễn viên, được nhiều camera hồng ngoại theo dõi vị trí 3D theo thời gian. |
| **SMPL** | *Skinned Multi-Person Linear model* — body model tham số hoá cơ thể người bằng shape (β) + pose (θ) + blend skinning. |
| **SMPL-X** | Bản mở rộng của SMPL, thêm bàn tay (từ MANO) và khuôn mặt (từ FLAME), N=10,475 vertex, K=54 khớp. |
| **SMPL-H** | Biến thể trung gian: SMPL + bàn tay (MANO), không có khuôn mặt — OMOMO dùng biến thể này. |
| **β (beta) — shape parameters** | Vector số thực điều khiển hình dáng cơ thể (cao/thấp, gầy/béo...) của một cá nhân, không đổi theo thời gian trong một chuyển động. |
| **θ (theta) — pose parameters** | Vector góc xoay (axis-angle) của từng khớp, thay đổi theo từng khung hình — chính là "chuyển động". |
| **Blend skinning (LBS)** | Thuật toán di chuyển từng vertex mesh theo tổ hợp trọng số của các khớp lân cận khi khớp xoay. |
| **Pose-dependent blend shapes** | Hiệu chỉnh hình dạng mesh phụ thuộc vào tư thế hiện tại, sửa lỗi méo mesh (ví dụ cơ bắp phồng khi gập khuỷu tay) mà LBS thuần tuý không xử lý được. |
| **MoSh / MoSh++** | Kỹ thuật tối ưu hoá tìm (β, θ) của SMPL sao cho marker ảo trên mesh khớp với marker mocap thật đo được — cách AMASS "dịch" 15 bộ mocap về cùng định dạng. |
| **DMPL** | Hệ số dao động mô mềm (soft-tissue) bổ sung trong AMASS, mô phỏng rung động da/mỡ khi chuyển động nhanh. |
| **BVH (BioVision Hierarchy)** | Định dạng file mocap dạng cây khớp (HIERARCHY, có OFFSET xương) + dữ liệu góc Euler theo khung hình (MOTION) — không có mesh. |
| **Motion in-betweening** | Bài toán sinh tự động các khung hình chuyển động ở giữa hai tư thế keyframe cho trước, sao cho mượt và tự nhiên. |
| **Loco-manipulation** | Bài toán robot vừa di chuyển toàn thân (locomotion) vừa thao tác vật thể bằng tay (manipulation) cùng lúc. |
| **Retargeting** | Quá trình chuyển đổi chuyển động từ skeleton nguồn (người, SMPL, BVH...) sang skeleton đích khác (robot humanoid cụ thể như Unitree G1) — xem chi tiết ở `02-motion-retargeting/`. |
| **Gated dataset** | Dataset công khai nhưng yêu cầu người dùng đăng ký/chấp nhận điều khoản license trước khi tải (như AMASS, BONES-SEED). |

---

## Nguồn đã kiểm tra khi viết tài liệu này (ngoài 5 citation gốc ở README mục B)

- Trang chính thức AMASS: [amass.is.tue.mpg.de](https://amass.is.tue.mpg.de) — xác nhận "over 40 hours... over 300 subjects... more than 11000 motions... 15 optical marker-based mocap datasets".
- Trang chính thức LAFAN1 (README GitHub Ubisoft): xác nhận 5 subjects, 77 sequences, 496,672 frames @30fps, 15 theme.
- Trang chính thức OMOMO (arXiv abstract 2309.16237): xác nhận 15 objects, ~10 giờ.
- Trang Hugging Face `bones-studio/seed`: xác nhận đơn vị công bố (Bones Studio), quy mô 142,220 clip / ~288 giờ / 522 diễn viên, vai trò công cụ của NVIDIA (SOMA retargeter, temporal segmentation cho dự án Kimodo).
