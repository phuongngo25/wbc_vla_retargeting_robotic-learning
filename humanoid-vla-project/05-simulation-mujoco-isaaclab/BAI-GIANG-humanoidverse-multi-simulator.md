# Bài giảng: HumanoidVerse — lớp trừu tượng multi-simulator

*(Thuộc mảng: Simulation: MuJoCo & Isaac Lab)*

## 🎯 Mục tiêu bài học

- Giải thích được vấn đề cụ thể HumanoidVerse giải quyết: code huấn luyện viết thẳng theo API 1 simulator thì khó chuyển sang simulator khác.
- Mô tả chính xác kiến trúc 3 lớp độc lập: Simulator layer, Task layer, Algorithm layer, và interface chung giữa chúng.
- Giải thích được vì sao "policy đi vững trên nhiều engine khác nhau" là một dạng kiểm tra sim-to-sim, và mối liên hệ với sim-to-real gap.
- Liệt kê được các simulator và robot embodiment mà HumanoidVerse hỗ trợ.
- Nêu được đúng thời điểm nên bắt đầu dùng HumanoidVerse (sau khi đã quen ít nhất 2 simulator riêng lẻ), và vì sao dùng sớm quá có thể phản tác dụng.
- Liên hệ được HumanoidVerse với các khái niệm đã học (MuJoCo, MJX, Isaac Lab, PPO) như các "instance cụ thể" được lớp trừu tượng này bọc lại.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Ba bài trước (MJCF, MuJoCo Playground, Isaac Gym→Isaac Lab) giới thiệu hai hệ sinh thái riêng biệt: MuJoCo/MJX (nhẹ, JAX) và Isaac Sim/Isaac Lab (nặng hơn, USD/Omniverse). Một câu hỏi tự nhiên nảy sinh: nếu bạn đã viết code huấn luyện một policy trên Isaac Lab, cần bao nhiêu công sức để kiểm tra policy đó có hoạt động tương tự trên MuJoCo không? HumanoidVerse (LeCAR-Lab, CMU) trả lời câu hỏi này bằng một kiến trúc phần mềm cụ thể — học bài này để hiểu *tại sao* tách lớp (layering) là câu trả lời đúng cho bài toán "không muốn viết lại toàn bộ code khi đổi hạ tầng mô phỏng", một vấn đề kỹ thuật phần mềm quen thuộc ngoài phạm vi robot học nhưng ở đây được áp dụng cụ thể cho RL humanoid.

## 🧠 Trực giác

### Góc nhìn 1: Hợp đồng thuê xe (rental car interface), không quan tâm hãng xe cụ thể

Khi bạn thuê xe tại một công ty cho thuê, bạn tương tác với **một bộ điều khiển chuẩn** (vô-lăng, chân ga, chân phanh) — bất kể xe là Toyota hay Honda. Công ty cho thuê xe đã "trừu tượng hoá" sự khác biệt giữa các hãng xe thành một giao diện lái xe chung. HumanoidVerse làm đúng việc này cho simulator: Task layer và Algorithm layer tương tác với một "giao diện lái" chung (observation, action, reward theo chuẩn HumanoidVerse), không cần biết bên dưới là IsaacGym, Genesis hay Isaac Lab — giống bạn lái xe mà không cần học lại cách "vô-lăng Toyota khác vô-lăng Honda ở đâu".

**Giới hạn của loại suy này:** hai chiếc xe khác hãng thường có đặc tính lái (cảm giác phanh, độ nhạy ga) hơi khác nhau dù cùng vô-lăng chuẩn; tương tự, các simulator khác nhau có **contact solver khác nhau** (đã học ở bài Contact dynamics) nên một policy có thể "lái" (hoạt động) hơi khác nhau giữa các engine dù cùng giao diện — đây chính là điều HumanoidVerse muốn giúp bạn *phát hiện ra*, không phải xoá bỏ hoàn toàn sự khác biệt vật lý thật giữa các engine.

### Góc nhìn 2: Kiến trúc phần mềm plugin (như driver máy in) — ứng dụng không cần biết máy in hãng nào

Khi bạn nhấn "In" trên máy tính, ứng dụng của bạn không cần biết máy in là Canon hay HP — nó gọi một API in ấn chuẩn của hệ điều hành, và **driver** (lớp thích ứng riêng cho từng hãng máy in) dịch lệnh chuẩn đó thành lệnh cụ thể cho phần cứng thật. Simulator layer của HumanoidVerse đóng đúng vai trò "driver" này: dịch giữa API chung của HumanoidVerse và API riêng của IsaacGym/Genesis/Isaac Lab.

**Giới hạn của loại suy này:** driver máy in không ảnh hưởng tới *nội dung* bản in (chữ vẫn là chữ đó dù in trên máy nào); trong khi driver simulator của HumanoidVerse có thể ảnh hưởng tới **kết quả huấn luyện thực tế** (một policy học trên engine A với contact solver khác engine B có thể học ra hành vi hơi khác) — sự khác biệt giữa các "driver" ở đây không trong suốt hoàn toàn như driver máy in, mà chính là thứ nghiên cứu robustness muốn đo lường.

## 📐 Định nghĩa chính xác

**Vấn đề HumanoidVerse giải quyết:** mỗi công cụ (MuJoCo/MJX, Isaac Gym/Isaac Lab, Genesis...) có API, định dạng model, và quy ước reward/observation riêng. Nếu code huấn luyện viết thẳng theo API của 1 simulator, chuyển sang simulator khác (để kiểm tra policy có phụ thuộc quá mức vào đặc thù vật lý của 1 engine hay không) đòi hỏi viết lại gần như toàn bộ.

**Kiến trúc 3 lớp độc lập** (theo README kiến trúc [LeCAR-Lab/HumanoidVerse](https://github.com/LeCAR-Lab/HumanoidVerse)):

```text
┌─────────────────────────────────────────────────────────────┐
│ ALGORITHM LAYER                                                │
│  Thuật toán RL/IL (PPO và biến thể) — tách khỏi cả simulator   │
│  lẫn task, có thể swap qua lại                                 │
└──────────────────────────┬──────────────────────────────────────┘
                           │ interface chung
┌──────────────────────────▼──────────────────────────────────────┐
│ TASK LAYER                                                      │
│  Định nghĩa bài toán (ví dụ: motion tracking cho H1/G1) ĐỘC LẬP │
│  với engine đang chạy — cùng 1 định nghĩa reward, observation,  │
│  episode logic dùng chung                                       │
└──────────────────────────┬──────────────────────────────────────┘
                           │ interface chung
┌──────────────────────────▼──────────────────────────────────────┐
│ SIMULATOR LAYER                                                 │
│  Adapter riêng cho TỪNG physics engine cụ thể:                  │
│  ┌──────────┐  ┌─────────┐  ┌───────────┐                       │
│  │ IsaacGym │  │ Genesis │  │ Isaac Lab │  ...                  │
│  └──────────┘  └─────────┘  └───────────┘                       │
│  Dịch giữa API chung của HumanoidVerse và API riêng của engine   │
└──────────────────────────────────────────────────────────────────┘
```

**Robot embodiment hỗ trợ** (theo repo chính thức): Unitree H1-10DoF, H1-19DoF, G1-12DoF, G1-23DoF — nhiều mức độ tự do khác nhau cho cùng dòng robot.

**Lợi ích cụ thể khi nghiên cứu:**
1. **Kiểm tra robustness của policy:** nếu policy huấn luyện trên IsaacGym vẫn đi vững khi chuyển sang MuJoCo/Genesis (2 engine có contact solver khác nhau), đó là bằng chứng policy không "overfit" vào đặc thù vật lý của 1 engine — một dạng kiểm tra sim-to-sim trước khi thử sim-to-real.
2. **Không khoá cứng hạ tầng ngay từ đầu học:** bắt đầu học trên MuJoCo (nhẹ, không cần GPU mạnh) rồi chuyển sang Isaac Lab sau, không cần viết lại toàn bộ code task/reward.
3. **So sánh tốc độ/độ ổn định huấn luyện** giữa các engine trên cùng 1 bài toán.

## ⚙️ Cơ chế hoạt động — từng bước

```text
┌────────────────────────────────────────────────────────────┐
│ Người dùng viết 1 lần: định nghĩa Task (reward, obs, episode) │
│  KHÔNG import bất kỳ thứ gì riêng của IsaacGym/MuJoCo/Genesis  │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ Chọn Simulator adapter qua config (ví dụ: simulator=isaacgym) │
│  → Simulator layer khởi tạo đúng engine đó, dịch API           │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ Chọn Algorithm qua config (ví dụ: algo=ppo)                   │
│  → Algorithm layer nhận đúng interface Task đã định nghĩa      │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ Vòng lặp huấn luyện chạy bình thường                          │
│  (Algorithm ↔ Task ↔ Simulator, qua interface chung)          │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ ĐỔI simulator=mujoco_mjx trong config, GIỮ NGUYÊN Task+Algo   │
│  → chạy lại, so sánh kết quả — policy có còn đi vững không?   │
└──────────────────────────────────────────────────────────────┘
```

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ định lượng hoá "chi phí chuyển đổi simulator" — số tự chọn, không phải benchmark thật của HumanoidVerse)*

Giả sử một nhóm nghiên cứu có codebase huấn luyện **không** dùng lớp trừu tượng nào — viết thẳng theo API IsaacGym, tổng cộng khoảng **2.000 dòng code**, trong đó:

```text
Task-specific logic (reward, observation, episode) : 800 dòng
Simulator-specific glue code (API IsaacGym cụ thể)  : 900 dòng
Algorithm (PPO cụ thể)                              : 300 dòng
```

**Chi phí chuyển sang MuJoCo/MJX KHÔNG dùng HumanoidVerse:** phải viết lại toàn bộ 900 dòng glue code (khác API hoàn toàn) + có thể phải điều chỉnh một phần logic reward nếu observation format khác (giả sử 200/800 dòng task logic bị ảnh hưởng gián tiếp):

```text
dòng code cần viết lại ≈ 900 + 200 = 1.100 dòng (55% tổng codebase)
```

**Với HumanoidVerse (giả sử đã có sẵn Simulator adapter cho cả IsaacGym và MJX):** chỉ cần đổi 1 dòng config (`simulator: isaacgym` → `simulator: mujoco_mjx`), với điều kiện 800 dòng Task logic đã được viết đúng theo interface chung ngay từ đầu:

```text
dòng code cần viết lại ≈ 0 (chỉ đổi config)
Chi phí "trả trước" (upfront cost): phải học/tuân theo interface chung của
  HumanoidVerse khi viết Task logic lần đầu — không miễn phí hoàn toàn,
  nhưng trả một lần, tái sử dụng nhiều lần
```

**Ý nghĩa:** đây là đánh đổi kinh điển trong kỹ thuật phần mềm — trừu tượng hoá (abstraction) có chi phí thiết kế ban đầu (phải tuân theo interface, đôi khi interface chung không "vừa khít" 100% với mọi tính năng đặc thù của một engine cụ thể), nhưng tiết kiệm rất lớn (ở ví dụ này, ~55% codebase không cần viết lại) khi cần chuyển đổi hạ tầng nhiều lần — đúng nhận định trong README HumanoidVerse rằng công cụ này phù hợp nhất khi bạn *đã biết* mình sẽ cần so sánh/chuyển đổi giữa nhiều simulator, không phải khi chỉ dùng đúng 1 engine từ đầu tới cuối.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Viết thẳng theo API 1 simulator | HumanoidVerse (3 lớp trừu tượng) |
|---|---|---|
| Tốc độ bắt đầu dự án nhỏ, 1 simulator | Nhanh hơn (không cần học interface chung) | Chậm hơn ban đầu (chi phí học interface) |
| Chi phí chuyển sang simulator khác | Cao — viết lại phần lớn glue code | Thấp — chỉ đổi config, nếu adapter đã có sẵn |
| Kiểm tra robustness đa engine | Khó, tốn công | Dễ, gần như miễn phí sau chi phí ban đầu |
| Độ "trong suốt" khi debug sai lệch engine | Cao (thấy trực tiếp code của engine) | Thấp hơn — lớp trừu tượng có thể "che" chi tiết cần debug |
| Phù hợp cho ai | Người mới, dự án chỉ dùng 1 engine dài hạn | Người đã quen ≥2 simulator, cần robustness/so sánh |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "nên học HumanoidVerse ngay từ đầu để khỏi phải học riêng MuJoCo và Isaac Lab".** Vì sao sai: chính README của dự án này (và nhận định trong `NOI-DUNG-CHI-TIET.md`) chỉ rõ: nếu chưa hiểu rõ MuJoCo/Isaac Lab hoạt động độc lập ra sao, lớp trừu tượng sẽ **che mất chi tiết cần thiết để debug** khi có sai lệch giữa các engine — ví dụ nếu policy đi vững trên IsaacGym nhưng ngã trên MuJoCo, bạn cần hiểu rõ contact solver của cả hai (đã học ở bài "Contact dynamics") để biết đây là khác biệt vật lý thật hay lỗi trong Simulator adapter của HumanoidVerse. **Hiểu đúng:** thứ tự học hợp lý là: hiểu rõ ít nhất 2 simulator riêng lẻ trước (đúng như bài MJCF, MuJoCo Playground, Isaac Gym→Isaac Lab đã học), rồi mới dùng HumanoidVerse như công cụ *nâng cao* để so sánh/kiểm tra robustness.
2. **Hiểu nhầm: "nếu policy đi vững trên cả 2-3 simulator qua HumanoidVerse, nghĩa là nó chắc chắn hoạt động tốt trên robot thật (sim-to-real đã giải quyết)".** Vì sao sai: sim-to-sim robustness (đi vững qua nhiều *mô phỏng*) là một bằng chứng hữu ích nhưng **không tương đương** sim-to-real robustness — tất cả các simulator vẫn là mô phỏng, có thể cùng thiếu chung một số hiệu ứng vật lý thật (độ trễ cảm biến/actuator thật, độ đàn hồi vật liệu thật, nhiễu môi trường thật) mà không simulator nào trong bộ so sánh mô hình hoá đúng. **Hiểu đúng:** sim-to-sim robustness là bước kiểm tra trung gian hữu ích, giảm rủi ro overfit vào một engine cụ thể, nhưng không thay thế được bước kiểm chứng sim-to-real thật (xem `07-policy-evaluation/`).

## 🏗️ Ví dụ minh hoạ trong dự án này

```text
Giai đoạn học (checkpoint mục D, README 05):
  Bước 1-3: học MuJoCo/MJX thuần (nhẹ, không cần GPU mạnh)
  Bước 4: chuyển sang Isaac Lab (train chính thức, cần GPU NVIDIA)
        │
        ▼
Sau khi đã quen CẢ HAI — dùng HumanoidVerse để:
  - Huấn luyện policy motion-tracking (dữ liệu đã retarget từ 02)
    trên IsaacGym/Isaac Lab trước (tốc độ GPU-parallel training)
  - Chạy lại ĐÚNG policy đó (không train lại) trên MuJoCo/MJX
    qua Simulator adapter tương ứng
  - Nếu policy vẫn đi vững trên MuJoCo → tăng độ tin cậy trước khi
    thử tiếp bước sim-to-real (07-policy-evaluation/, 08-real-robot-deployment/)
```

Đây chính là ứng dụng cụ thể của "checkpoint nâng cao" mà README gợi ý — HumanoidVerse không thay thế bước học riêng từng simulator, mà là bước *tổng hợp* sau khi đã có nền tảng vững từ các bài giảng trước trong cùng thư mục này.

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **HumanoidVerse (CMU LeCAR-Lab) công bố công khai từ 02/2025**, mô tả chính thức là "Multi-Simulator Framework for Humanoid Robot Sim-to-Real Learning", hỗ trợ IsaacGym, Genesis, và Isaac Lab, cùng nhiều embodiment Unitree (H1-10DoF, H1-19DoF, G1-12DoF, G1-23DoF) và pipeline Sim-to-Sim/Sim-to-Real cho motion tracking task. [GitHub: LeCAR-Lab/HumanoidVerse](https://github.com/LeCAR-Lab/HumanoidVerse)
2. **LeCAR-Lab là nhóm nghiên cứu đứng sau ASAP** — *"Aligning Simulation and Real-World Physics for Learning Agile Humanoid Whole-Body Skills"* (RSS 2025, [GitHub: LeCAR-Lab/ASAP](https://github.com/LeCAR-Lab/ASAP)), đã được nhắc tới trong "Đường phát triển PHC → OmniH2O → ASAP → SONIC" (`04-imitation-learning-rl/`) — cùng một nhóm nghiên cứu xây dựng cả công cụ hạ tầng đa-simulator (HumanoidVerse) lẫn thuật toán sim-to-real cụ thể (ASAP), cho thấy nhu cầu thực tế: nghiên cứu về "làm sao thu hẹp khoảng cách sim-to-real" (ASAP) và nghiên cứu về "làm sao kiểm tra robustness qua nhiều mô phỏng" (HumanoidVerse) đi cùng nhau trong cùng một chương trình nghiên cứu.
3. **Cộng đồng đã mở rộng/fork HumanoidVerse** (ví dụ `NathanWu7/HumanoidVerse3`, mô tả "an all-in-one project for humanoid robots from locomotion to motion tracking") — dấu hiệu framework đang được cộng đồng tiếp nhận và mở rộng thêm ngoài phạm vi gốc của LeCAR-Lab, dù các bản fork/mở rộng này chưa nhất thiết đã qua kiểm chứng đồng cấp (peer review) như bản gốc.

## ❓ Câu hỏi tự kiểm tra

1. Ba lớp của HumanoidVerse là gì, và lớp nào chịu trách nhiệm "dịch" giữa API chung và API riêng của từng engine?
   <details><summary>Gợi ý đáp án</summary>Simulator layer (dịch API), Task layer (định nghĩa bài toán độc lập engine), Algorithm layer (thuật toán RL/IL độc lập cả simulator lẫn task). Simulator layer là lớp dịch API.</details>
2. Vì sao "policy đi vững trên nhiều simulator" chỉ là một dạng kiểm tra sim-to-sim, không phải bằng chứng đầy đủ cho sim-to-real?
   <details><summary>Gợi ý đáp án</summary>Vì tất cả các simulator so sánh vẫn là mô phỏng, có thể cùng thiếu chung một số hiệu ứng vật lý thật (độ trễ cảm biến/actuator, đàn hồi vật liệu, nhiễu môi trường) mà không simulator nào mô hình hoá đúng — đi vững qua nhiều mô phỏng giảm rủi ro overfit vào 1 engine, nhưng không đảm bảo hoạt động đúng trên robot thật.</details>
3. Trong ví dụ tính tay, điều gì quyết định "chi phí trả trước" khi dùng HumanoidVerse có đáng hay không?
   <details><summary>Gợi ý đáp án</summary>Tuỳ vào việc dự án có thực sự cần chuyển đổi/so sánh giữa nhiều simulator hay không — nếu chỉ dùng đúng 1 engine từ đầu tới cuối, chi phí học interface chung không được "hoàn vốn"; nếu cần chuyển đổi nhiều lần (như ví dụ tiết kiệm 55% codebase), chi phí ban đầu rất đáng.</details>
4. Vì sao nên học riêng MuJoCo và Isaac Lab trước khi dùng HumanoidVerse, thay vì học HumanoidVerse ngay từ đầu?
   <details><summary>Gợi ý đáp án</summary>Vì lớp trừu tượng có thể che mất chi tiết cần thiết để debug khi có sai lệch giữa các engine (ví dụ khác biệt contact solver) — cần hiểu rõ từng engine độc lập trước để phân biệt được "khác biệt vật lý thật" với "lỗi trong Simulator adapter".</details>
5. LeCAR-Lab (CMU) còn được biết đến với công trình nào khác liên quan trực tiếp tới sim-to-real cho humanoid, đã học ở thư mục `04-imitation-learning-rl/`?
   <details><summary>Gợi ý đáp án</summary>ASAP — "Aligning Simulation and Real-World Physics for Learning Agile Humanoid Whole-Body Skills" (RSS 2025), một mắt xích trong đường phát triển PHC → OmniH2O → ASAP → SONIC.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** với codebase 3.000 dòng (Task 1.200, Simulator glue 1.500, Algorithm 300), tính % codebase cần viết lại khi chuyển simulator KHÔNG dùng lớp trừu tượng (giả sử 300/1.200 dòng task logic bị ảnh hưởng gián tiếp), so sánh với cách dùng HumanoidVerse.
2. **Đọc code thật:** mở [repo HumanoidVerse](https://github.com/LeCAR-Lab/HumanoidVerse), tìm thư mục chứa Simulator adapter cho 2 engine khác nhau (ví dụ IsaacGym và Isaac Lab) — so sánh cấu trúc code của 2 adapter này, xác định phần nào giống hệt nhau về interface (chứng minh chúng cùng tuân theo 1 chuẩn chung) và phần nào khác nhau (chứng minh chúng thực sự gọi API riêng của từng engine).

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

HumanoidVerse (CMU LeCAR-Lab, công bố 02/2025) giải quyết bài toán code huấn luyện bị khoá cứng vào API của 1 simulator cụ thể, bằng kiến trúc 3 lớp độc lập (Simulator, Task, Algorithm) giao tiếp qua interface chung — như ví dụ tính tay minh hoạ, cách này có thể tiết kiệm phần lớn công sức viết lại code (ước tính ~55% trong ví dụ) khi cần chuyển đổi/so sánh giữa nhiều engine (IsaacGym, Genesis, Isaac Lab), với cái giá là một chi phí học interface chung ban đầu. Lợi ích chính là kiểm tra robustness của policy qua nhiều mô phỏng có contact solver khác nhau (một dạng kiểm tra sim-to-sim), nhưng điều này không thay thế được kiểm chứng sim-to-real thật, và công cụ này chỉ thực sự hữu ích sau khi đã hiểu rõ ít nhất 2 simulator riêng lẻ — dùng quá sớm sẽ che mất chi tiết cần thiết để debug sai lệch giữa các engine.
