# 03 — Robotics (học ở mức nền tảng để hiểu & mở rộng)

Với trục chính là medical imaging, bạn học robotics đủ để hiểu perception–planning–control và bắc cầu sang surgical robotics. Không cần thành chuyên gia control lý thuyết.

---

## A. Khóa học & sách nền tảng

| Tài liệu | Nội dung | Link |
|---|---|---|
| **Modern Robotics: Mechanics, Planning, and Control** — Lynch & Park | ⭐ Sách + specialization Coursera (Northwestern). Screw theory, kinematics, dynamics, control. Miễn phí PDF. | [Sách](http://hades.mech.northwestern.edu/index.php/Modern_Robotics) · [Coursera](https://www.coursera.org/specializations/modernrobotics) |
| **CS223A — Introduction to Robotics** (Stanford, Oussama Khatib) | Kinematics, dynamics, control cổ điển. Video miễn phí. | [Stanford SEE](https://see.stanford.edu/Course/CS223A) |
| **Underactuated Robotics** — Russ Tedrake (MIT 6.832) | ⭐ Điều khiển tối ưu, planning cho hệ phi tuyến. Có sách online + code (Drake). | [underactuated.mit.edu](https://underactuated.csail.mit.edu) · [github](https://github.com/RussTedrake/underactuated) |
| **Robotic Manipulation** — Russ Tedrake (MIT 6.4210) | Perception + manipulation hiện đại (rất liên quan surgical). | [manipulation.mit.edu](https://manipulation.csail.mit.edu) |
| **Probabilistic Robotics** — Thrun, Burgard, Fox | Kinh điển cho localization, SLAM, filtering. | [probabilistic-robotics.org](http://probabilistic-robotics.org/) |
| **Planning Algorithms** — Steven LaValle | Motion planning nền tảng, miễn phí. | [lavalle.pl/planning](http://lavalle.pl/planning/) |
| **Cyrill Stachniss — Mobile Sensing & Robotics / Photogrammetry** (Bonn) | ⭐⭐ **Bài giảng SLAM/perception rõ nhất trên internet**, toàn bộ video công khai. Nếu bạn cần hiểu SLAM, camera model, ICP, factor graph — học ở đây thay vì đọc sách. Rất liên quan tới surgical vision. | [ipb.uni-bonn.de/teaching](https://www.ipb.uni-bonn.de/teaching/) |
| **CS285 — Deep Reinforcement Learning** (Sergey Levine, Berkeley) | ⭐ Khoá RL chuẩn mực. Video + slides + homework công khai. Cần cho autonomous surgical subtask. | [rail.eecs.berkeley.edu/deeprlcourse](https://rail.eecs.berkeley.edu/deeprlcourse/) |
| **ETH RSL — Robot Dynamics / Legged Robotics** | Bài giảng ETH Zurich công khai, rất chắc về dynamics & control. | [rsl.ethz.ch/education-students/lectures](https://rsl.ethz.ch/education-students/lectures.html) |
| **Steve Brunton — Control Bootcamp** | Control theory bằng trực giác + code, playlist YouTube. Bổ trợ khi Modern Robotics quá hàn lâm. | Tìm "Steve Brunton Control Bootcamp" trên YouTube |

> 📖 **Paper cụ thể để đọc** (SLAM, NeRF/Gaussian Splatting, Diffusion Policy, ACT, VLA — kèm arXiv ID đã kiểm chứng): **`08-core-reading-list.md` Track C**.

---

## B. Công cụ & framework

| Công cụ | Vai trò | Link |
|---|---|---|
| **ROS 2** | Middleware chuẩn ngành robotics. | [docs.ros.org](https://docs.ros.org) |
| **Gazebo / Isaac Sim** | Mô phỏng robot (sim-to-real). | [gazebosim.org](https://gazebosim.org) · [NVIDIA Isaac](https://developer.nvidia.com/isaac/sim) |
| **PyBullet / MuJoCo** | Mô phỏng vật lý nhẹ cho RL/control. | [pybullet.org](https://pybullet.org) · [mujoco.org](https://mujoco.org) |
| **Drake** | Toolbox của Tedrake cho planning/control/optimization. | [drake.mit.edu](https://drake.mit.edu) |
| **MoveIt 2** | Motion planning cho manipulator (ROS). | [moveit.ai](https://moveit.ai) |

---

## C. Venue

**Hội nghị:**

- **ICRA** — IEEE Int. Conf. on Robotics and Automation (lớn nhất; tháng 5–6 hằng năm, deadline nộp ~tháng 9).
- **IROS** — IEEE/RSJ Int. Conf. on Intelligent Robots and Systems (tháng 10).
- **RSS** — Robotics: Science and Systems (single-track, theory-heavy, uy tín cao).
- **CoRL** — Conference on Robot Learning (giao robotics ∩ ML, đang rất nóng).

**Tạp chí:**

- **IEEE Transactions on Robotics (T-RO)** — top journal.
- **IEEE Robotics and Automation Letters (RA-L)** — nhanh, thường gắn với ICRA/IROS (lối vào tốt).
- **The International Journal of Robotics Research (IJRR)** — uy tín, theory-heavy.
- **Science Robotics** — tác động cao, liên ngành.

---

## D. Chủ đề nên nắm (đủ để bắc cầu sang surgical)

- **Perception:** camera calibration, pose estimation, tracking, SLAM ⭐ (rất cần cho surgical vision — học qua Stachniss ở mục A).
- **Kinematics/Dynamics** của cánh tay robot (manipulator); đặc biệt **remote center of motion (RCM)** — cơ chế đặc trưng của dụng cụ nội soi.
- **Control cơ bản:** PID → impedance/force control ⭐ (quan trọng khi robot tiếp xúc mô mềm — đây là chỗ robot y tế khác robot công nghiệp nhất).
- **Imitation Learning & RL** cho manipulation (nền cho tự động hoá tác vụ phẫu thuật — xem Diffusion Policy, ACT trong `08-core-reading-list.md`).
- **Sim-to-real transfer** — bạn sẽ huấn luyện trong mô phỏng phẫu thuật, nên phần này bắt buộc.
- **Biểu diễn 3D hiện đại:** NeRF → 3D Gaussian Splatting. Đang là công cụ chính cho tái tạo mô biến dạng trong mổ.

> **Gợi ý lộ trình tối thiểu:** Modern Robotics ch.1–6 + Robotic Manipulation (Tedrake) + bài giảng SLAM của Stachniss = đủ nền để đọc và làm paper surgical robotics. **Đừng học hết** — bạn cần đủ để hiểu và bắc cầu, không cần thành chuyên gia control.
>
> ⚠️ **Cạm bẫy của engineer ở đây:** dễ sa vào học robotics rất sâu vì nó thú vị và có cấu trúc rõ. Nhưng trục PhD của bạn là **imaging/vision**. Hãy giới hạn phần này ở mức "đọc được paper ICRA về surgical robotics" rồi quay lại trục chính.
