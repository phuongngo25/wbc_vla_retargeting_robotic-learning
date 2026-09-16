# 08 — Teleoperation & Triển khai trên robot thật

> Bước cuối cùng của pipeline: dùng VR để **thu dữ liệu** trên robot thật (teleoperation), và **triển khai** policy đã huấn luyện trong simulation lên phần cứng thật. Đây là phần **cần phần cứng** (robot G1, kính VR) — nếu chưa có, vẫn học được lý thuyết + thử phần simulation-only.

---

## A. Khái niệm cần nắm, theo thứ tự

1. **Vì sao cần teleoperation:** để thu dữ liệu chuyển động/thao tác *thật* trên robot thật (không qua retargeting từ dữ liệu người ngoài) — dữ liệu này dùng để fine-tune VLA hoặc bổ sung vào tập huấn luyện motion-tracking, đặc biệt cho các thao tác tay tinh xảo mà retargeting từ mocap không nắm bắt tốt.
2. **Kiến trúc teleoperation qua VR:** người đeo kính VR → tracking đầu/tay/tay cầm → ánh xạ (qua IK, giống retargeting nhưng real-time) sang tư thế robot → robot thực thi theo thời gian thực, có phản hồi hình ảnh stereoscopic (nhìn qua camera robot) gửi ngược lại kính VR.
3. **XRoboToolkit** (khớp với "XToolRobotKits" mentor nhắc — tên chính xác là **XRoboToolkit**): framework teleoperation mã nguồn mở, chuẩn OpenXR, hỗ trợ tracking đầu/tay/controller/tracker phụ, độ trễ thấp, đã test trên Ubuntu 22.04/24.04. Hỗ trợ nhiều nền tảng robot (tay máy UR5, ARX R5, và cả humanoid Galaxea R1-Lite).
4. **Quy trình teleop chính thức trong GR00T-WholeBodyControl:** setup kính VR → kết nối với policy WBC (SONIC) qua token space chung → điều khiển whole-body robot trực tiếp bằng chuyển động cơ thể người vận hành.
5. **Từ dữ liệu teleop tới fine-tuning:** dữ liệu thu được (ảnh + trạng thái khớp + hành động) dùng để fine-tune GR00T N1.x hoặc để bổ sung reward/data cho WBC — đây là vòng lặp "data flywheel" thực sự của các hệ thống VLA hiện đại.
6. **Triển khai (deployment):** xuất policy đã huấn luyện sang ONNX (định dạng nhẹ, chạy trên phần cứng biên như Jetson gắn trên robot), kiểm tra độ trễ thực thi, kiểm tra an toàn (giới hạn tốc độ, emergency stop) trước khi chạy trên robot thật.

---

## B. Tài liệu chính

| Tài liệu | Vì sao đọc | Link |
|---|---|---|
| **VR Teleop Setup** (GR00T-WholeBodyControl, chính thức) | 🔴 Hướng dẫn cài đặt phần cứng/phần mềm teleop từ đầu — bắt buộc theo đúng thứ tự. | [Docs](https://nvlabs.github.io/GR00T-WholeBodyControl/getting_started/vr_teleop_setup.html) |
| **VR Whole-Body Teleop tutorial** (GR00T-WholeBodyControl, chính thức) | 🔴 Hướng dẫn thực hành điều khiển whole-body qua VR, kết nối trực tiếp với SONIC. | [Docs](https://nvlabs.github.io/GR00T-WholeBodyControl/tutorials/vr_wholebody_teleop.html) |
| **XRoboToolkit** paper — A Cross-Platform Framework for Robot Teleoperation | 🔴 Hiểu kiến trúc framework teleop (OpenXR, IK tối ưu hoá, tracking đa phương thức) trước khi cài đặt. | [arXiv:2508.00097](https://arxiv.org/abs/2508.00097) |
| **XRoboToolkit** — GitHub (code + sample teleop Python/C++) | 🔴 Code thực hành, có sẵn sample cho nhiều robot. | [GitHub XR-Robotics](https://github.com/XR-Robotics) |
| **GR00T-WholeBodyControl — Reference: ONNX models & deployment** | 🟡 Phần tài liệu tham khảo về xuất/chạy model ONNX trên phần cứng thật. | [Docs](https://nvlabs.github.io/GR00T-WholeBodyControl/) |
| **Isaac Lab 2.3 — Enhanced Teleoperation blog** | 🟡 Bối cảnh: teleop trong simulation trước khi ra robot thật (Isaac Lab hỗ trợ sẵn). | [NVIDIA Tech Blog](https://developer.nvidia.com/blog/streamline-robot-learning-with-whole-body-control-and-enhanced-teleoperation-in-nvidia-isaac-lab-2-3) |

---

## C. Video hướng dẫn

| Video | Ngôn ngữ | Nội dung |
|---|---|---|
| Video demo trên trang **XRoboToolkit**/PICO Developer blog | Tiếng Anh | Minh hoạ trực quan teleoperation qua kính VR (PICO đã tích hợp XRoboToolkit). |
| Video/GIF minh hoạ trong tutorial VR Whole-Body Teleop chính thức | Tiếng Anh | Xem trước khi tự làm để biết kết quả mong đợi trông như thế nào. |

> ⚠️ Chưa có video/tài liệu tiếng Việt cho teleoperation robot humanoid qua VR. Nếu bạn chưa từng dùng kính VR/OpenXR, nên xem qua bất kỳ video giới thiệu OpenXR cơ bản nào trước (không chuyên robot) để làm quen khái niệm tracking đầu/tay.

---

## D. Lộ trình từng bước cho người mới bắt đầu

**Nếu CÓ phần cứng (G1 + kính VR):**

1. **(2–3 ngày)** Làm theo đúng **VR Teleop Setup** chính thức — cài đặt từng bước, không bỏ qua phần kiểm tra kết nối mạng/độ trễ.
2. **(3–5 ngày)** Làm theo **VR Whole-Body Teleop tutorial** — thử điều khiển robot đứng yên trước, sau đó di chuyển đơn giản.
3. **(1 tuần)** Thu 1 bộ dữ liệu teleop nhỏ (vài phút, vài task đơn giản: cầm vật, đi vài bước) — lưu lại đúng định dạng mà `Isaac-GR00T` yêu cầu để fine-tune.
4. **(checkpoint)** Thử fine-tune nhẹ N1.7 (hoặc chỉ chạy inference so sánh trước/sau) trên dữ liệu vừa thu — quan sát sự khác biệt.

**Nếu CHƯA có phần cứng (chỉ simulation):**

1. **(2–3 ngày)** Đọc kỹ cả 2 tutorial chính thức để hiểu quy trình, dù chưa chạy được.
2. **(1 tuần)** Cài XRoboToolkit, chạy sample teleop trong simulation (không cần robot thật, nhiều sample chạy được với robot ảo trong Isaac Sim/MuJoCo).
3. **(3–5 ngày)** Thử xuất 1 policy đã huấn luyện ở `05-simulation-mujoco-isaaclab/` sang ONNX, chạy inference thử trong simulation để hiểu bước "deploy" mà không cần robot thật.
4. **(checkpoint)** Viết note mô tả: nếu có G1 thật, bạn sẽ cần thêm bước gì so với những gì đã làm trong simulation (an toàn, độ trễ, calibration)?

---

## Liên kết chéo

- Nhận policy WBC đã huấn luyện/đánh giá từ `01-whole-body-control/`, `05-simulation-mujoco-isaaclab/`, `07-policy-evaluation/`.
- Dữ liệu thu được có thể fine-tune ngược lại `06-vla-groot-sonic/`.
- Đây là bước tổng kết toàn bộ pipeline — quay lại `../README.md` để xem lại sơ đồ đầy đủ.
