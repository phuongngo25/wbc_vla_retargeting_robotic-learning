# Bài giảng: Tích hợp VLA + SONIC qua unified token space

*(Thuộc mảng: Vision-Language-Action: GR00T N1 → N1.7 & SONIC)*

## 🎯 Mục tiêu bài học

- Giải thích được **vì sao** một hệ thống humanoid cần một "unified token space" (không gian token thống nhất) thay vì xây hai policy tách biệt cho hai nguồn lệnh khác nhau (VR teleop và VLA).
- Phân biệt rạch ròi **vai trò của GR00T N1.x** (não cấp cao, quyết định *làm gì*) và **SONIC** (tủy sống/phản xạ vận động, quyết định *làm như thế nào* ở mức khớp) trong một vòng lặp loco-manipulation tự chủ.
- Vẽ và giải thích được **sơ đồ luồng dữ liệu hội tụ**: hai nguồn lệnh khác nhau (tay người vận hành qua VR, và action token do VLA sinh ra) đi vào **cùng một single policy** của SONIC.
- Nhận diện được **2 hiểu nhầm phổ biến nhất** về khái niệm này (nhầm là một model duy nhất được train chung; nhầm là VLA sinh trực tiếp góc khớp).
- Nêu được ít nhất 2 hệ thống khác (2024-2026) đang theo đuổi ý tưởng tương tự — "giao diện chuẩn hoá giữa policy cấp cao và bộ điều khiển cấp thấp" — và chỉ ra điểm giống/khác với SONIC.
- Biết chính xác **giới hạn hiểu biết hiện tại** của bản thân về cơ chế token hóa — phần nào đã xác minh được qua đọc paper, phần nào vẫn cần đọc thêm.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Toàn bộ dự án học tập này được tổ chức thành 8 thư mục, mỗi thư mục là một lớp trong "chồng" (stack) công nghệ của một humanoid robot hiện đại: retargeting dữ liệu người (02), skeleton mapping (02), reinforcement learning để huấn luyện policy chuyển động (04), whole-body control (01), và ở lớp trên cùng — vision-language-action, tức là "bộ não" quyết định robot nên làm gì khi nhận một câu lệnh ngôn ngữ (06). Nhưng có một câu hỏi mà cho tới tận thư mục 06 vẫn chưa được trả lời: **hai lớp này — "não" quyết định ý định, và "tủy sống" thực thi chuyển động — nói chuyện với nhau bằng cách nào?**

Đây không phải câu hỏi vặt. Nếu não và tủy sống dùng hai "ngôn ngữ" khác nhau, bạn cần một tầng dịch (adapter/glue code) giữa chúng — và tầng dịch đó thường là nơi mọi thứ vỡ: độ trễ tăng, lỗi tích lũy, và mỗi khi não hoặc tủy sống thay đổi kiến trúc, tầng dịch phải viết lại. Bài giảng này là về **giải pháp mà SONIC chọn**: thay vì có một tầng dịch, hãy thiết kế sao cho cả hai nguồn lệnh — người vận hành thật (qua VR teleoperation) và một model AI tự động (VLA) — đều nói cùng một "ngôn ngữ token" ngay từ đầu. Đây chính là **mảnh ghép cuối cùng nối hai nửa dự án lại với nhau**: nửa "học chuyển động từ dữ liệu người" (thư mục 01-02-03-04) và nửa "học ý định từ ngôn ngữ" (thư mục 06). Hiểu được cơ chế này, bạn hiểu được toàn cảnh dự án hội tụ ở đâu.

Lưu ý phạm vi: bài này **không** giải thích lại kiến trúc SONIC/WBC (đã có ở `01-whole-body-control/`) hay kiến trúc dual-system System 1/System 2 của GR00T N1.x (xem bài giảng riêng — dự kiến `BAI-GIANG-groot-n1-dual-system.md` trong thư mục này). Bài này chỉ tập trung đúng một điểm: **cơ chế tích hợp** giữa hai hệ thống đó.

## 🧠 Trực giác

### Góc nhìn 1: Một ngôn ngữ chung (lingua franca) giữa hai người nói tiếng khác nhau

Hãy tưởng tượng một hội nghị quốc tế nơi một người nói tiếng Việt (đại diện cho **VR teleoperation** — lệnh từ tay/đầu người vận hành đọc qua headset) và một người nói tiếng Nhật (đại diện cho **VLA** — lệnh cấp cao do GR00T N1.x sinh ra) đều cần truyền đạt ý muốn cho một người điều phối duy nhất (đại diện cho **SONIC single policy**). Cách thông thường là thuê hai phiên dịch viên riêng — một người dịch Việt→điều phối, một người dịch Nhật→điều phối. Nhưng nếu cả hội nghị đồng ý dùng chung một ngôn ngữ trung gian (ví dụ tiếng Anh chuẩn hóa) mà cả hai bên đều được huấn luyện để nói ngay từ đầu, thì người điều phối chỉ cần biết **một** ngôn ngữ duy nhất — không cần hai phiên dịch viên, không cần biết ai đang nói. Đó chính là ý tưởng: token hóa cả lệnh VR lẫn lệnh VLA vào cùng một định dạng, để SONIC (người điều phối) chỉ cần hiểu một "ngôn ngữ" và không cần phân biệt nguồn gốc.

**Giới hạn của loại suy này:** đây là ẩn dụ về *giao diện* (interface), không phải một cơ chế xử lý ngôn ngữ tự nhiên (NLP) thật. "Ngôn ngữ chung" ở đây không mang ngữ nghĩa/cú pháp như ngôn ngữ con người — nó là một không gian vector rời rạc hóa (discrete latent space) học được từ dữ liệu, không có "từ vựng" hay "ngữ pháp" theo nghĩa NLP. Đừng suy diễn rằng SONIC "hiểu ngôn ngữ" theo nghĩa GR00T hiểu ngôn ngữ — hai chữ "ngôn ngữ" ở đây là hai khái niệm hoàn toàn khác nhau.

### Góc nhìn 2: Cổng giao tiếp chuẩn hóa — kiểu USB cắm-là-chạy

Trước khi có chuẩn USB, mỗi thiết bị ngoại vi (chuột, bàn phím, máy in, ổ cứng ngoài) thường cần một loại cổng và driver riêng — máy tính phải "biết" từng loại thiết bị để giao tiếp đúng cách. USB giải quyết vấn đề này bằng cách định nghĩa **một giao thức chuẩn** mà bất kỳ thiết bị nào tuân theo đều có thể cắm vào cùng một cổng vật lý, và máy tính xử lý mọi thiết bị qua cùng một tầng giao thức, bất kể bên trong thiết bị là chuột quang hay ổ SSD. Tương tự, "unified token space" của SONIC là một "cổng USB" cho hành động: bất kể tín hiệu đến từ đâu (bàn tay người vận hành qua VR, hay action token do VLA sinh ra), miễn là được mã hóa đúng "chuẩn cổng" (định dạng token thống nhất), SONIC (đóng vai máy tính) xử lý chúng theo đúng một quy trình duy nhất.

**Giới hạn của loại suy này:** USB là một chuẩn giao thức được thiết kế thủ công, cố định, công bố công khai (ai cũng có thể làm thiết bị tuân theo chuẩn USB). Unified token space của SONIC thì ngược lại — nó là một không gian **học được** (learned latent space, qua các encoder được huấn luyện cùng dữ liệu chuyển động), không phải một đặc tả kỹ thuật cố định viết tay. Hai hệ thống bên ngoài (VR teleop, VLA) không "tự tuân theo chuẩn" như một nhà sản xuất USB — chúng phải được **huấn luyện hoặc thiết kế riêng** để tạo ra token tương thích với không gian mà SONIC đã học.

## 📐 Định nghĩa chính xác

Theo abstract của paper SONIC (Luo, Yuan, Wang, Li, Castañeda và 23 đồng tác giả khác — NVIDIA/GEAR, [arXiv:2511.07820](https://arxiv.org/abs/2511.07820)):

> "...a unified token space that supports virtual reality (VR) teleoperation and vision-language-action (VLA) models with a single policy."

Diễn giải chính xác từng thành phần của định nghĩa này:

- **"Unified token space"** — một không gian biểu diễn hành động (action representation) chung, mà nhiều nguồn tín hiệu điều khiển khác nhau đều được ánh xạ (mã hóa/encode) vào đó, dưới dạng các **token rời rạc** (discrete tokens) cùng định dạng — cùng số chiều, cùng kiểu lượng tử hóa (quantization).
- **"Supports VR teleoperation and VLA models"** — hai loại nguồn lệnh cụ thể mà không gian token này phục vụ: (1) lệnh từ người vận hành thật, đọc qua thiết bị VR (vị trí tay/đầu/thân trên); (2) lệnh cấp cao do một model VLA (GR00T N1.x) sinh ra một cách tự động, không có người can thiệp.
- **"With a single policy"** — điểm mấu chốt: **chỉ một** mạng neural (một tập trọng số) nhận token từ không gian thống nhất này và sinh ra góc khớp/lệnh motor cấp thấp — không có hai policy riêng biệt cho hai nguồn lệnh, không có tầng chuyển đổi/thích ứng (adapter) trung gian giữa "token VR" và "token VLA" vì bản thân chúng đã cùng định dạng ngay từ khâu tạo ra.

Hệ quả trực tiếp của định nghĩa này: khi GR00T N1.x sinh ra một action token cấp cao, SONIC diễn giải token đó **giống hệt** cách nó diễn giải token đến từ tay người vận hành VR — không có sự phân biệt đối xử giữa hai nguồn. Đây là điều kiện tiên quyết cho phép chế độ vận hành gọi là **"autonomous VLA-driven whole-body loco-manipulation"** (paper gọi task ứng dụng minh họa là loại yêu cầu "coordinated hand and foot placement" — phối hợp đặt tay và đặt chân đồng thời).

## ⚙️ Cơ chế hoạt động — từng bước

Sơ đồ dưới đây thể hiện hai luồng dữ liệu, khác nguồn hoàn toàn, nhưng **hội tụ tại đúng một điểm**: SONIC single policy.

```
NHÁNH 1 — VR teleoperation (người vận hành thật):

  [VR headset + controllers]
          │  (vị trí/hướng tay, đầu, thân trên của người vận hành)
          ▼
  [Human encoder ℰh]  ── mã hóa tư thế người (vd. SMPL joint positions)
          │
          ▼
  [Token hóa: FSQ — Finite Scalar Quantization]
          │  → token rời rạc, cùng định dạng với nhánh 2
          ▼
          └──────────────┐
                          │
NHÁNH 2 — VLA tự động:    │
                          │
  [Camera + câu lệnh ngôn ngữ, vd. "nhặt cốc nước trên bàn"]
          │
          ▼
  [GR00T N1.x — System 2, ~7-10Hz]
          │  (VLM/multimodal, hiểu ngữ cảnh + ý định, sinh latent/embedding)
          ▼
  [GR00T N1.x — System 1, ~120Hz]
          │  (action head tần số cao, sinh action chunk)
          ▼
  [Token hóa: cùng định dạng FSQ/universal motion token]
          │
          ▼
          └──────────────┐
                          │
                          ▼
              ╔═══════════════════════════╗
              ║   SONIC — SINGLE POLICY    ║
              ║  (không phân biệt nguồn    ║
              ║   token đến từ đâu)        ║
              ╚═══════════════════════════╝
                          │
                          ▼
              [Robot motion decoder ℜr]
                          │
                          ▼
            Góc khớp / lệnh motor cấp thấp
            (chân, tay, thân — full-body)
                          │
                          ▼
                    [Robot G1 chuyển động]
```

Các bước cụ thể, theo đúng trình tự thời gian trong một vòng lặp:

1. **Bước 1 — Sinh tín hiệu điều khiển ở nguồn.** Với VR teleop: người vận hành di chuyển tay/đầu, thiết bị VR đọc tư thế 3D. Với VLA: camera chụp ảnh cảnh hiện tại, người dùng đưa ra câu lệnh ngôn ngữ.
2. **Bước 2 — Mã hóa vào không gian latent chuyên biệt cho từng nguồn.** SONIC dùng các encoder khác nhau tùy loại input (encoder cho tư thế người, encoder cho trạng thái robot, encoder cho input "hybrid" — xem chi tiết đã xác minh ở mục "Cập nhật hiện đại" bên dưới) để đưa dữ liệu thô về một không gian latent liên tục.
3. **Bước 3 — Lượng tử hóa (quantize) thành token rời rạc.** Không gian latent liên tục được rời rạc hóa thành một tập token hữu hạn — đây chính là bước tạo ra "unified token space": dù input gốc là tư thế người hay là output của VLA, sau bước này chúng có **cùng định dạng token**.
4. **Bước 4 — SONIC single policy nhận token, không cần biết nguồn gốc.** Đây là điểm hội tụ trung tâm của toàn bộ cơ chế: cùng một mạng neural, cùng một tập trọng số, xử lý token bất kể nó đến từ VR hay từ VLA.
5. **Bước 5 — Giải mã (decode) token thành lệnh motor.** SONIC sinh ra góc khớp/mô-men cho toàn thân (chân + tay + thân trên), tại tần số điều khiển riêng của nó (tần số này thuộc phạm vi bài giảng WBC ở `01-whole-body-control/`, không nhắc lại chi tiết ở đây).
6. **Bước 6 — Vòng lặp phản hồi.** Robot chuyển động, trạng thái mới (proprioception) được đọc lại, quay về bước 1 cho chu kỳ tiếp theo — với VLA, System 2 chạy chậm hơn (khoảng 7-10Hz) trong khi System 1 và SONIC chạy nhanh hơn nhiều, tạo ra kiến trúc bất đồng bộ (asynchronous) nhiều tầng tần số.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

**Cảnh báo quan trọng:** phần dưới đây là một **walkthrough định tính minh họa luồng dữ liệu**, dùng các con số thời gian/tần số hợp lý dựa trên kiến trúc đã biết của GR00T N1.x và SONIC (đã mô tả ở các bài giảng khác trong dự án) — đây **không phải** số liệu thực nghiệm trích trực tiếp từ một bảng kết quả cụ thể trong paper SONIC cho đúng kịch bản này. Mục đích là giúp bạn hình dung luồng thời gian, không phải để trích dẫn như số liệu chính thức.

**Kịch bản:** Robot G1 nhận lệnh ngôn ngữ "nhặt cốc nước trên bàn và mang tới người dùng", không có người vận hành VR (chế độ autonomous VLA-driven).

| Thời điểm (mốc minh họa) | Thành phần | Diễn ra gì |
|---|---|---|
| t = 0 ms | System 2 (GR00T N1.x) | Nhận ảnh camera (cốc nước, bàn, người dùng trong khung hình) + câu lệnh ngôn ngữ. Bắt đầu forward pass qua VLM backbone. |
| t = 0 → ~100-140 ms | System 2 | Sinh ra một embedding/latent biểu diễn "ý định": ví dụ "di chuyển tới gần bàn, đưa tay phải tới vị trí cốc, cầm nắm, xoay người, đưa tay ra phía người dùng". Do chạy ở ~7-10Hz, một lần forward pass mất khoảng 100-140ms. |
| t ≈ 100 ms | System 1 | Nhận latent mới nhất từ System 2 + quan sát proprioception hiện tại, bắt đầu sinh **action chunk** ở tần số cao hơn nhiều (ví dụ ~120Hz — tức action chunk có horizon H, giả sử H=16 bước, được sinh liên tục). |
| t ≈ 100 → 108 ms | System 1 → Token hóa | Action chunk (thô, ở dạng continuous) được mã hóa qua encoder tương ứng rồi lượng tử hóa (FSQ) thành chuỗi token rời rạc — cùng định dạng với token mà nhánh VR teleop sẽ tạo ra nếu có người vận hành. |
| t ≈ 108 ms | SONIC single policy | Nhận token (không "biết" và không cần biết token này đến từ VLA chứ không phải từ tay người vận hành VR). Chạy forward pass ở tần số điều khiển riêng của WBC (thường cao hơn nhiều so với 120Hz của System 1, ví dụ ~500Hz-1kHz tùy cấu hình — chi tiết tần số WBC thuộc phạm vi `01-whole-body-control/`). |
| t ≈ 108+ ms | SONIC → Robot decoder | Sinh góc khớp cụ thể cho từng khớp (chân trái/phải để giữ thăng bằng và bước tới bàn, tay phải để với và nắm cốc, tay trái/thân trên để giữ thăng bằng bù trừ). |
| Vòng lặp tiếp theo | System 2 (chậm hơn, ~100ms/lần) | Vẫn tiếp tục "suy nghĩ" ở tần số thấp — ví dụ sau khi tay đã nắm cốc, System 2 mới cập nhật ý định sang "xoay người, đưa cốc tới người dùng" — trong khi System 1 + SONIC đã chạy hàng chục vòng lặp phản xạ nhanh ở giữa hai lần cập nhật của System 2. |

Điểm cần rút ra từ walkthrough này: **SONIC không "biết" nó đang phục vụ một VLA tự động hay một người vận hành thật** — với nó, mọi thứ chỉ là một chuỗi token đến ở một tần số nhất định. Sự khác biệt về tần số giữa System 2 (chậm, ~10Hz) và System 1 + SONIC (nhanh, hàng trăm Hz) là điều tạo ra cảm giác robot "phản xạ mượt" dù "suy nghĩ" chậm — đây chính là lý do kiến trúc dual-system + unified token interface phối hợp hiệu quả.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | Kiến trúc 2 policy tách biệt (VR-policy riêng + VLA-policy riêng) | Unified token space (1 single policy — cách SONIC làm) |
|---|---|---|
| **Độ phức tạp engineering** | Cao — phải xây, huấn luyện, bảo trì 2 hệ thống độc lập, cộng thêm một tầng "adapter"/glue code để chuyển đổi giữa 2 định dạng lệnh khác nhau. | Thấp hơn về lâu dài — chỉ 1 policy để huấn luyện/bảo trì, nhưng đòi hỏi thiết kế không gian token dùng chung ngay từ đầu (chi phí thiết kế ban đầu cao hơn). |
| **Khả năng mở rộng (thêm nguồn lệnh mới, vd. thêm một loại VLA khác, hoặc human video)** | Mỗi nguồn lệnh mới cần một adapter mới → chi phí tăng tuyến tính theo số nguồn lệnh. | Chỉ cần huấn luyện một encoder mới ánh xạ nguồn lệnh mới vào không gian token đã có — bản thân SONIC (single policy) không cần thay đổi. Thực tế, theo mô tả kỹ thuật của SONIC, không gian token này được thiết kế hỗ trợ cả VR teleop, human video, và VLA — tức đã tính tới việc mở rộng đa nguồn ngay từ đầu. |
| **Độ trễ (latency)** | Có nguy cơ thêm độ trễ ở tầng adapter (thời gian chuyển đổi định dạng). | Không cần tầng chuyển đổi trung gian giữa VLA và WBC — về lý thuyết giảm được một khâu trễ, nhưng đổi lại độ trễ dồn vào bước lượng tử hóa/token hóa (vẫn tồn tại, chỉ là không lặp lại cho từng nguồn). |
| **Rủi ro** | Rủi ro phân tán: lỗi có thể nằm ở policy A, policy B, hoặc ở chính tầng adapter — khó debug vì nhiều điểm hỏng độc lập. | Rủi ro tập trung: nếu không gian token thiết kế kém (ví dụ không đủ biểu đạt cho một loại chuyển động), **mọi nguồn lệnh** đều bị ảnh hưởng — một điểm hỏng duy nhất nhưng ảnh hưởng toàn hệ thống. |
| **Bằng chứng thực nghiệm về chất lượng** | (không phải trọng tâm so sánh của paper SONIC — không có số liệu trực tiếp so sánh 2-policy vs 1-policy) | Paper SONIC có ablation cho thấy dùng **discrete token action space** (cách tiếp cận unified-token) vượt trội rõ rệt so với dùng **explicit human pose** (SMPL) làm action space cho VLA — xem chi tiết định lượng ở mục "Cập nhật hiện đại" bên dưới. Đây là bằng chứng gián tiếp ủng hộ việc chọn biểu diễn token rời rạc, dù không phải so sánh trực tiếp "1 policy vs 2 policy". |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Nhầm rằng SONIC và GR00T N1.x là MỘT model duy nhất được huấn luyện chung (end-to-end).** Thực tế: đây là **hai hệ thống hoàn toàn khác nhau**, huấn luyện riêng biệt, với mục tiêu và dữ liệu huấn luyện khác nhau (GR00T N1.x học từ video + ngôn ngữ để hiểu ý định; SONIC học từ dữ liệu motion-capture quy mô lớn để tạo chuyển động tự nhiên). Chúng chỉ **chia sẻ một giao diện/định dạng token** — giống như hai công ty khác nhau đồng ý dùng chung một chuẩn API, không có nghĩa là họ dùng chung một cơ sở dữ liệu hay một đội kỹ sư.
2. **Nhầm rằng VLA (GR00T N1.x) trực tiếp sinh ra góc khớp cho robot.** Thực tế: GR00T N1.x sinh ra **action token cấp cao** (một biểu diễn trừu tượng của ý định chuyển động), sau đó token này được **SONIC** — chứ không phải VLA — giải mã (decode) thành góc khớp/lệnh motor cụ thể. VLA không "biết" và không cần biết cấu trúc cơ khí chi tiết của robot (bao nhiêu khớp, giới hạn góc quay từng khớp...) — đó là trách nhiệm của SONIC.
3. **Nhầm rằng "unified token space" nghĩa là bất kỳ loại tín hiệu điều khiển nào cũng tự động tương thích.** Thực tế: một nguồn lệnh mới (ví dụ một VLA khác, hoặc một giao diện điều khiển hoàn toàn mới) **phải được thiết kế/huấn luyện riêng** để tạo ra token đúng định dạng và đúng phân bố thống kê mà SONIC đã học — không phải cứ "rời rạc hóa" là tự nhiên tương thích. Đây là điểm khác biệt quan trọng với ẩn dụ USB ở phần Trực giác (USB là chuẩn cố định công khai; ở đây là một không gian học được, không có đặc tả kỹ thuật cố định để bên thứ ba tự tuân theo).

## 🏗️ Ví dụ minh hoạ trong dự án này

Đây chính là điểm mà **toàn bộ 8 thư mục của dự án hội tụ lại**. Sơ đồ toàn cảnh:

```
02-motion-retargeting/          03-.../ (dataset chuẩn bị)
  (retarget chuyển động người              │
   sang skeleton robot, giải quyết          │
   "vì sao không thể copy góc khớp")        │
              │                             │
              ▼                             ▼
        04-imitation-learning-rl/  ──────────
        (huấn luyện SONIC bằng RL,
         dùng dữ liệu đã retarget
         làm mục tiêu bắt chước)
              │
              ▼
        01-whole-body-control/
        (SONIC — single policy,
         nhận token thống nhất,
         sinh góc khớp toàn thân)
              ▲
              │  (unified token space — ĐÂY LÀ BÀI GIẢNG NÀY)
              │
        06-vla-groot-sonic/
        (GR00T N1.x — "não cấp cao",
         hiểu ảnh + ngôn ngữ,
         sinh action token cấp cao)
              │
              ▼
        07-policy-evaluation/
        (đánh giá chất lượng toàn hệ thống —
         ví dụ dùng MPJPE và các biến thể
         để đo độ chính xác bám chuyển động)
```

Nói cách khác: thư mục 02-03-04 trả lời câu hỏi "làm sao dạy SONIC di chuyển tự nhiên như người". Thư mục 01 trả lời "SONIC vận hành nội bộ như thế nào". Thư mục 06 (bao gồm chính bài giảng này) trả lời "làm sao một model AI hiểu ngôn ngữ có thể *ra lệnh* cho SONIC mà không cần xây lại một bộ điều khiển riêng". Và thư mục 07 trả lời "làm sao biết toàn bộ chuỗi này hoạt động tốt". Bài giảng này — unified token space — là **khớp nối** (the missing link) giữa nửa "học chuyển động" và nửa "học ý định" của dự án.

## 🔥 Cập nhật hiện đại / SOTA gần đây

**Xác minh thêm qua đọc trực tiếp bản HTML đầy đủ của paper SONIC (arXiv:2511.07820, v4, cập nhật 13/8/2026):** so với mức abstract-only ban đầu, đã tìm được các chi tiết cơ chế cụ thể hơn (lưu ý: trích qua công cụ đọc/tóm tắt tự động bản HTML của paper — nếu cần trích dẫn số liệu để công bố/báo cáo chính thức, nên đối chiếu lại trực tiếp với bản PDF gốc):

- **Kỹ thuật lượng tử hóa:** SONIC dùng **Finite Scalar Quantization (FSQ)** — không phải VQ-VAE (Vector-Quantized VAE) truyền thống — để tạo token rời rạc. Cấu hình cụ thể được nêu là **FSQ-32-32** (32 mức lượng tử, 32 chiều mỗi token). Lý do chọn FSQ thay vì VQ-VAE: FSQ tránh được **codebook collapse** (hiện tượng phần lớn "từ điển" mã embedding học được không bao giờ được dùng tới) — và theo paper, FSQ vượt trội hơn VQ-VAE khoảng 8.7mm MPJPE-L trên tập test-content (chỉ số MPJPE đã có bài giảng riêng ở `07-policy-evaluation/BAI-GIANG-mpjpe-va-cac-bien-the.md`).
- **Cấu trúc encoder đa nguồn:** ba encoder chuyên biệt ánh xạ các loại input khác nhau vào cùng không gian latent trước khi lượng tử hóa — encoder cho robot (trạng thái khớp/vận tốc robot), encoder cho người (tư thế 3D dạng SMPL, dùng cho VR teleop/human video), và encoder "hybrid" (kết hợp keypoint thân trên thưa + chuyển động robot phần dưới) — đây chính là cơ chế kỹ thuật hiện thực hóa "unified token space" mà abstract chỉ mô tả ở mức khái niệm.
- **Bằng chứng định lượng ủng hộ token rời rạc:** ablation so sánh dùng **FSQ token** làm action space cho VLA (GR00T) so với dùng **explicit SMPL pose** (tư thế người tường minh, liên tục) làm action space — trên các task như nhặt cà rốt, mở thùng rác, bỏ lon nước ngọt vào thùng rác — token rời rạc cho tỷ lệ thành công trung bình vượt trội rõ rệt so với biểu diễn pose liên tục (paper giải thích: không gian pose liên tục có số chiều cao khiến sai số dự đoán nhỏ bị khuếch đại thành thất bại lớn khi bám theo, trong khi token rời rạc, số chiều thấp, dễ học hơn từ dữ liệu demo qua teleoperation).
- **Phạm vi input rộng hơn mức abstract ban đầu ghi nhận:** không chỉ VR teleop + VLA, không gian token thống nhất của SONIC còn được thiết kế hỗ trợ cả **human video** làm nguồn input, và qua tích hợp với một model chuyển động người tổng quát tên **GEM**, SONIC có thể nhận điều khiển từ cả video, text prompt, và nhạc — mở rộng đáng kể so với mô tả "chỉ VR + VLA" ở mức tóm tắt.
- **Tích hợp thực tế trong GR00T N1.7:** repo `GR00T-WholeBodyControl` (NVlabs) xác nhận N1.7 hỗ trợ whole-body control qua embodiment tag `UNITREE_G1_SONIC` và bộ điều khiển **GEAR-SONIC**, nơi VLA sinh ra "compact latent action token" mà whole-body controller giải mã thành lệnh khớp toàn thân (chân, tay, tay cầm) — khớp đúng với mô tả cơ chế ở bài giảng này.

**Các hệ thống khác (2024-2026) theo đuổi ý tưởng "giao diện chuẩn hóa giữa policy cấp cao và bộ điều khiển cấp thấp":**

- **Helix / Helix 02 — Figure AI** ([figure.ai/news/helix](https://www.figure.ai/news/helix), [figure.ai/news/helix-02](https://www.figure.ai/news/helix-02)): kiến trúc "System 2" (VLM 7B, chạy ~7-9Hz, "não") + "System 1" (chạy ~200Hz, điều khiển toàn bộ phần trên cơ thể) — và Helix 02 mở rộng thêm "System 0" (chạy ở tần số kilohertz, xử lý thăng bằng/phối hợp kiểu phản xạ tủy sống). Điểm giống SONIC: cùng ý tưởng phân tầng tần số não-chậm/tủy-nhanh. Điểm khác quan trọng: Helix không nhấn mạnh một "unified token space" phục vụ *nhiều nguồn lệnh khác nhau* (VR teleop + VLA) như SONIC — trọng tâm của Helix là sự bất đồng bộ (asynchrony) giữa hai tầng của chính một VLA, không phải giao diện dùng chung giữa người vận hành và AI.
- **JAEGER — Dual-Level Humanoid Whole-Body Controller** ([arXiv:2505.06584](https://arxiv.org/pdf/2505.06584)): một hướng tiếp cận khác cho whole-body control, tách riêng điều khiển phần trên/dưới cơ thể ở hai cấp độ — hữu ích để đối chiếu với cách tiếp cận "một policy duy nhất xử lý toàn thân" của SONIC.
- **ℳ²Tok — Multi-head Multi-codebook Discrete Action Tokenization for VLA models** ([arXiv:2609.18259](https://arxiv.org/html/2609.18259)): cho thấy xu hướng rộng hơn trong 2025-2026 — dùng token rời rạc đa-codebook làm action representation chuẩn cho VLA nói chung (không riêng humanoid) — củng cố thêm luận điểm rằng lựa chọn "discrete token action space" của SONIC nằm trong một xu hướng lớn hơn của cả lĩnh vực, không phải một lựa chọn riêng lẻ.
- **"Asynchronous Fast-Slow Vision-Language-Action Policies for Whole-Body Robotic Manipulation"** ([arXiv:2512.20188](https://arxiv.org/pdf/2512.20188)): một hướng nghiên cứu 2025-2026 khác trực tiếp giải quyết bài toán "VLA (chậm) + whole-body control (nhanh)" phối hợp không đồng bộ — đáng đọc thêm để so sánh trực tiếp cách tiếp cận với SONIC.

## ❓ Câu hỏi tự kiểm tra

1. Vì sao một hệ thống có thể cần "hai policy khác nhau" nếu không có unified token space — hãy mô tả cụ thể vấn đề kỹ thuật (không chỉ nói chung chung "phức tạp hơn").
2. Nếu SONIC nhận một token, làm sao nó biết token đó nên diễn giải thành "đi tới bàn" hay "với tay nhặt cốc"? Câu trả lời đúng phải chỉ ra: SONIC **không cần phân biệt nguồn gốc** token — nó chỉ cần token đúng định dạng đã học; ý nghĩa ngữ nghĩa cấp cao (đi tới bàn hay với tay) đã được mã hóa sẵn trong chính giá trị token, không phải một cờ (flag) đánh dấu "đây là lệnh từ VLA".
3. Phân biệt: FSQ (Finite Scalar Quantization) khác VQ-VAE (Vector-Quantized VAE) ở điểm nào, và vì sao sự khác biệt đó quan trọng cho bài toán "codebook collapse"?
4. Tại sao dùng discrete token (rời rạc) làm action space cho VLA lại hoạt động tốt hơn dùng pose liên tục (ví dụ SMPL) — giải thích bằng chính lập luận "khuếch đại sai số" đã nêu trong bài.
5. Helix (Figure AI) cũng có kiến trúc "hai tầng tần số" (System 2 chậm, System 1 nhanh) giống GR00T N1.x + SONIC. Điểm khác biệt cốt lõi giữa hai cách tiếp cận là gì, xét riêng về khái niệm "phục vụ nhiều nguồn lệnh khác nhau qua một giao diện chung"?
6. Nếu ngày mai NVIDIA muốn thêm một nguồn lệnh thứ ba (ví dụ điều khiển bằng cử chỉ tay từ một camera thường, không phải VR headset) vào hệ thống này, theo đúng nguyên lý unified token space, bước kỹ thuật chính cần làm là gì (và bước nào KHÔNG cần làm)?

## 📝 Bài tập thực hành

1. **Đọc sâu phần method của paper SONIC.** Tự đọc trực tiếp bản PDF/HTML đầy đủ tại [arXiv:2511.07820](https://arxiv.org/abs/2511.07820) (đặc biệt phần mô tả kiến trúc FSQ, ba encoder ℰr/ℰh/ℰm, và bảng ablation FSQ-token vs SMPL-pose). Ghi lại: (a) công thức/thuật toán chính xác của FSQ (khác VQ-VAE ở phép tính cụ thể nào, không chỉ ở ý tưởng); (b) số liệu chính xác của bảng ablation (đối chiếu lại với số liệu đã nêu ở mục "Cập nhật hiện đại" của bài giảng này — nếu có sai lệch, ghi chú lại vì số liệu ở đây được trích qua công cụ đọc tự động, có thể cần hiệu chỉnh).
2. **Tự vẽ lại sơ đồ hội tụ token** (phần "Cơ chế hoạt động") bằng công cụ vẽ tay hoặc draw.io, sau đó tự giải thích cho một người khác (hoặc ghi âm tự giải thích) toàn bộ luồng — từ lúc camera nhận ảnh tới lúc khớp chân robot chuyển động — mà không nhìn lại bài giảng. Đây là bài kiểm tra hiểu bài tốt nhất theo đúng gợi ý ở `README.md` mục D bước 6 của thư mục này.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

SONIC giải quyết bài toán "một robot humanoid cần nhận lệnh từ cả người vận hành thật (qua VR teleoperation) lẫn từ một model AI tự động (VLA, cụ thể là GR00T N1.x)" bằng cách xây một **unified token space**: mọi nguồn lệnh đều được mã hóa (qua các encoder chuyên biệt) rồi lượng tử hóa (bằng FSQ — Finite Scalar Quantization, thay vì VQ-VAE, để tránh codebook collapse) thành cùng một định dạng token rời rạc, để một **single policy duy nhất** của SONIC xử lý — không phân biệt token đến từ tay người vận hành hay từ action head của GR00T N1.x, và không cần một tầng adapter riêng biệt giữa hai hệ thống. Cơ chế này là mảnh ghép cuối cùng nối "não cấp cao" (GR00T N1.x, hiểu ảnh + ngôn ngữ, quyết định ý định) với "tủy sống/phản xạ" (SONIC, sinh góc khớp toàn thân), cho phép chế độ vận hành tự chủ hoàn toàn gọi là autonomous VLA-driven whole-body loco-manipulation. Quan trọng: đây là **hai hệ thống riêng biệt chia sẻ một giao diện học được**, chứ không phải một model duy nhất huấn luyện chung, và bằng chứng thực nghiệm trong paper (ablation FSQ-token vs SMPL-pose) cho thấy lựa chọn biểu diễn token rời rạc mang lại lợi thế rõ rệt về khả năng học từ dữ liệu demo — một lựa chọn thiết kế nằm trong xu hướng rộng hơn của toàn ngành robot learning giai đoạn 2025-2026 (so sánh với Helix của Figure AI, ℳ²Tok, và các kiến trúc fast-slow VLA khác).
