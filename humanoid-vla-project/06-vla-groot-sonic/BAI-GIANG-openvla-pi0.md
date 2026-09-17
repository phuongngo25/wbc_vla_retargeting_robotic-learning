# Bài giảng: Dòng phát triển OpenVLA → π₀

*(Thuộc mảng: Vision-Language-Action: GR00T N1 → N1.7 & SONIC)*

## 🎯 Mục tiêu bài học

- Giải thích được vì sao OpenVLA ra đời (mã nguồn mở hóa kiến trúc VLA) và nó kế thừa/khác RT-2 ở điểm nào.
- Mô tả đúng kiến trúc OpenVLA: Llama 2 + fuse đặc trưng DINOv2/SigLIP, sinh hành động kiểu autoregressive-token rời rạc.
- Giải thích được bước chuyển kiến trúc của π₀: từ token rời rạc sang flow matching sinh action chunk liên tục, và vì sao đây là bước ngoặt quan trọng nhất dẫn tới GR00T.
- Đọc hiểu và diễn giải đúng ý nghĩa con số "7B so với 55B, cải thiện 16.5%" trong paper OpenVLA — không chỉ nhớ số mà hiểu nó chứng minh điều gì.
- Phân biệt được OpenVLA (gốc, 2024) với OpenVLA-OFT (2025) và π₀ (gốc, 2024) với π₀-FAST, π₀.5 (2025) — không nhầm bản gốc với bản cập nhật.
- Nêu được π₀ (flow-matching action head) đã trực tiếp truyền cảm hứng kiến trúc cho System 1 của GR00T N1/N1.7 như thế nào.

## 🧭 Vì sao cần học cái này? (bối cảnh)

Bài giảng này đứng giữa hai mắt xích trong dòng lịch sử VLA mà `NOI-DUNG-CHI-TIET.md` mục 1.2 đã vạch ra:

```text
RT-1 (2022) → RT-2 (2023) → OpenVLA (2024) → π₀ (2024) → GR00T N1 (2025) → N1.7
```

RT-1 chứng minh một transformer có thể "hấp thụ" dữ liệu robot thật đa dạng, nhưng chỉ học từ demo robot — tốn kém để scale. RT-2 giải quyết việc đó bằng cách ghép một VLM pretrain internet-scale vào, biến hành động thành "token ngôn ngữ" — nhưng RT-2 là mô hình **đóng** (closed), không công bố trọng số/kiến trúc chi tiết đầy đủ để cộng đồng học theo hay tái tạo.

**OpenVLA xuất hiện để lấp đúng khoảng trống đó**: một phiên bản mã nguồn mở, kiến trúc công khai rõ ràng, cho phép bất kỳ ai (kể cả một dự án học tập cá nhân như dự án này) đọc hiểu và tái tạo được một VLA "kiểu RT-2" từ đầu đến cuối. Vai trò của OpenVLA trong bài học này: **học kiến trúc VLA nền tảng** — cách ghép VLM với dữ liệu robot, cách sinh hành động bằng token.

Nhưng OpenVLA (và cả RT-2) vẫn mang một hạn chế cấu trúc: sinh hành động **autoregressive từng token rời rạc một** — giống hệt cách một LLM sinh từng từ trong câu văn. Cách này tự nhiên cho ngôn ngữ (rời rạc theo bản chất), nhưng gượng ép cho tín hiệu điều khiển robot (vốn liên tục, cần mượt và nhanh).

**π₀ là bước chuyển kiến trúc giải quyết đúng hạn chế đó**: giữ nguyên ý tưởng "VLM pretrain sẵn cung cấp tri thức internet-scale" (kế thừa từ RT-2), nhưng thay hoàn toàn cách sinh hành động — dùng **flow matching** để sinh ra cả một action chunk liên tục cùng lúc, thay vì từng token rời rạc tuần tự.

Đây chính xác là công thức mà GR00T N1 và N1.7 tiếp tục đi theo (System 2 là VLM pretrain kiểu RT-2/OpenVLA, System 1 là flow-matching action head kiểu π₀). Nói cách khác: **π₀ là tiền thân trực tiếp về mặt ý tưởng kiến trúc của System 1 trong GR00T** — hiểu π₀ là điều kiện tiên quyết để hiểu vì sao GR00T được thiết kế như hiện tại. Chi tiết kỹ thuật của flow matching và của GR00T System 1 được để dành cho các bài giảng riêng (`BAI-GIANG-flow-matching.md`, `BAI-GIANG-groot-n1-dual-system.md`) — ở đây chỉ tập trung vào bản thân sự chuyển đổi kiến trúc OpenVLA → π₀.

## 🧠 Trực giác

### Góc nhìn 1: Từ "đánh vần từng chữ" sang "viết cả câu cùng lúc bằng nét vẽ liên tục"

Hãy tưởng tượng hai người được yêu cầu viết ra một chuyển động tay (ví dụ vẽ một đường cong trên giấy).

Người thứ nhất (giống OpenVLA) làm việc như một người đánh vần: họ chia đường cong thành nhiều đoạn rời rạc, gán mỗi đoạn một "ký hiệu" từ một bảng ký hiệu hữu hạn (giống bảng chữ cái), rồi viết ra từng ký hiệu một, theo thứ tự, ký hiệu sau phụ thuộc vào ký hiệu trước (autoregressive). Cách này chắc chắn, dùng lại được đúng cơ chế đã thành công với văn bản (LLM), nhưng đường cong cuối cùng bị "răng cưa" theo độ phân giải của bảng ký hiệu, và viết chậm vì phải làm tuần tự từng ký hiệu.

Người thứ hai (giống π₀) làm việc như một họa sĩ: thay vì đánh vần, họ tưởng tượng một "trường lực" (vector field) dẫn cây bút di chuyển mượt từ một điểm xuất phát ngẫu nhiên đến hình dạng đường cong mong muốn, và vẽ ra toàn bộ đường cong trong một quá trình liên tục — không có "ký hiệu" trung gian nào cả.

Phép loại suy đúng ở chỗ: cả hai đều "biết" phải vẽ đường cong nào (nhờ cùng được huấn luyện/chỉ dẫn), khác nhau ở **cơ chế sinh ra kết quả cuối** — rời rạc hoá tuần tự so với một quá trình liên tục sinh cùng lúc cả chuỗi.

Giới hạn của loại suy:

- "trường lực" ở đây là một khái niệm toán học cụ thể (flow matching / vector field có điều kiện) chứ không phải phép ẩn dụ nghệ thuật đơn thuần — chi tiết toán học nằm trong `BAI-GIANG-flow-matching.md`.
- người họa sĩ trong đời thực không "tích phân ODE" — π₀ thực sự chạy một quá trình tích phân số học nhiều bước để đi từ nhiễu tới action chunk.
- phép loại suy không nói lên vì sao token rời rạc lại autoregressive chậm hơn tại inference — điều đó liên quan tới việc mỗi token phải chờ token trước sinh xong (decode tuần tự), trong khi flow matching có thể sinh toàn bộ chunk qua một số bước tích phân cố định, độc lập với độ dài chunk.

### Góc nhìn 2: Nâng cấp một chiếc xe — đổi khung gầm thay vì đổi động cơ

OpenVLA giống như lấy một khung gầm xe đã có sẵn (VLM Llama 2 pretrain) rồi gắn thêm hai camera cảm biến khác nhau (DINOv2 nhìn hình dạng/không gian, SigLIP nhìn ngữ nghĩa) để xe "nhìn" tốt hơn, nhưng vẫn giữ nguyên hộp số cũ: mỗi lệnh điều khiển vẫn phải "gõ" từng nấc số một, tuần tự.

π₀ đi xa hơn: vẫn dùng khung gầm VLM pretrain đó, nhưng **thay cả hộp số** bằng một cơ cấu truyền động mới (flow-matching action expert) cho phép ra lệnh điều khiển "mượt" liên tục thay vì gõ từng nấc.

Phép loại suy đúng ở chỗ: phần "nhìn và hiểu" (VLM backbone) không đổi về triết lý giữa hai thế hệ — cả OpenVLA và π₀ đều dựa trên một mô hình pretrain internet-scale. Thứ thay đổi là **bộ phận sinh hành động ở đầu ra**.

Giới hạn của loại suy:

- một chiếc xe thật không "học" cách lái từ dữ liệu — π₀ và OpenVLA đều phải huấn luyện lại (fine-tune hoặc huấn luyện đồng thời) trên dữ liệu robot, không phải chỉ "gắn thêm phụ kiện" vào một hệ thống bất biến.
- ẩn dụ "hộp số mượt" không truyền tải được rằng flow matching còn mang lại lợi ích tốc độ suy luận (inference) — ít bước hơn để sinh mẫu chất lượng tương đương — chứ không chỉ là "mượt hơn về cảm giác".

## 📐 Định nghĩa chính xác

**OpenVLA** ([arXiv:2406.09246](https://arxiv.org/abs/2406.09246)) là một Vision-Language-Action model mã nguồn mở, kiến trúc gồm:

- **Backbone ngôn ngữ**: Llama 2.
- **Visual encoder**: fuse (kết hợp) đặc trưng từ hai encoder pretrain riêng biệt — **DINOv2** (mạnh về đặc trưng không gian/hình học) và **SigLIP** (mạnh về đặc trưng ngữ nghĩa gắn với ngôn ngữ).
- **Dữ liệu huấn luyện**: 970k demo robot thật, đa dạng nhiều robot/embodiment.
- **Cách sinh hành động**: autoregressive-token — hành động liên tục được rời rạc hóa (discretize) thành token, sinh tuần tự từng token một, giống cách RT-2 biểu diễn hành động như "ngôn ngữ".
- **Quy mô**: 7B tham số.

**π₀** ([arXiv:2410.24164](https://arxiv.org/abs/2410.24164), *"π₀: A Vision-Language-Action Flow Model for General Robot Control"*, Physical Intelligence) là một VLA:

- xây trên một **VLM pretrain sẵn** để thừa hưởng tri thức internet-scale (giống triết lý RT-2/OpenVLA ở phần "nhìn và hiểu");
- nhưng phần **sinh hành động là một mô-đun flow-matching riêng** (action expert), không phải phần tiếp nối autoregressive của chính VLM;
- sinh ra **toàn bộ action chunk liên tục** (một chuỗi hành động số thực, nhiều bước thời gian, sinh cùng lúc qua một quá trình tích phân trường vận tốc), thay vì từng token rời rạc một.

Về mặt định nghĩa, sự khác biệt cốt lõi cần nhớ chính xác: **OpenVLA = VLM + autoregressive discrete action tokens**; **π₀ = VLM (pretrain) + flow-matching continuous action chunk**. Đây là đúng nội dung nguồn `NOI-DUNG-CHI-TIET.md` mục 1.2, không mở rộng thêm diễn giải riêng.

Một hệ quả trực tiếp của khác biệt kiến trúc này là **hàm mục tiêu huấn luyện (training objective) cũng khác nhau về bản chất**:

- OpenVLA huấn luyện giống một LLM chuẩn: **cross-entropy loss** trên chuỗi token hành động rời rạc — mô hình học dự đoán đúng token tiếp theo, y hệt cách huấn luyện mô hình ngôn ngữ dự đoán từ tiếp theo.
- π₀ huấn luyện action expert bằng mục tiêu **flow matching** — về bản chất là hồi quy một trường vận tốc (regressing a vector field) sao cho quá trình tích phân từ nhiễu ra đúng phân bố action chunk quan sát được trong dữ liệu. Đây không phải bài toán phân loại token như OpenVLA, mà là bài toán hồi quy có điều kiện trên không gian liên tục.

Sự khác biệt về hàm mục tiêu này giải thích vì sao hai kiến trúc không thể "trộn lẫn nửa vời" — action head kiểu flow matching không tương thích trực tiếp với việc chỉ thêm một lớp phân loại token vào cuối một LLM, mà cần một mô-đun huấn luyện theo mục tiêu hoàn toàn khác.

## ⚙️ Cơ chế hoạt động — từng bước

### Kiến trúc OpenVLA

```text
Ảnh RGB đầu vào          Câu lệnh ngôn ngữ
      │                         │
      ▼                         ▼
┌───────────┐            ┌─────────────┐
│  DINOv2   │            │   SigLIP    │
│(không gian│            │(ngữ nghĩa)  │
│/hình học) │            │             │
└─────┬─────┘            └──────┬──────┘
      │                         │
      └──────────┬──────────────┘
                  ▼
       fuse đặc trưng thị giác
       (kết hợp 2 nguồn feature)
                  ▼
       ┌─────────────────────┐
       │      Llama 2 (7B)    │  ← backbone ngôn ngữ, nhận
       │   (autoregressive)   │    feature thị giác + text token
       └──────────┬───────────┘
                  ▼
     sinh TỪNG token hành động một,
     tuần tự, token sau phụ thuộc
     token trước (giống sinh văn bản)
                  ▼
     de-tokenize → giá trị hành động
     rời rạc hoá (vd. vị trí/độ mở
     gripper tại từng bước thời gian)
```

Các bước cụ thể:

1. Ảnh quan sát hiện tại được đưa qua cả DINOv2 và SigLIP; hai tập đặc trưng được fuse lại thành một biểu diễn thị giác duy nhất.
2. Biểu diễn thị giác này cùng với token của câu lệnh ngôn ngữ được đưa vào Llama 2 như một chuỗi input, giống hệt cách một LLM đa phương thức (multimodal) xử lý ảnh + văn bản.
3. Llama 2 sinh output autoregressive: mỗi bước sinh ra một token hành động, token này được đưa lại vào chuỗi input để sinh token tiếp theo — đúng cơ chế decode tuần tự của một LLM.
4. Các token hành động được de-tokenize (giải mã ngược) thành giá trị số thực rời rạc hoá — ví dụ tọa độ end-effector hoặc trạng thái gripper tại một bước thời gian.
5. Toàn bộ mô hình được huấn luyện end-to-end trên 970k demo robot thật, học ánh xạ trực tiếp từ (ảnh, lệnh) sang chuỗi token hành động.

### Kiến trúc π₀

```text
Ảnh RGB (có thể nhiều camera)    Câu lệnh ngôn ngữ    Trạng thái robot
      │                               │                     │
      └───────────────┬───────────────┴──────────┬──────────┘
                       ▼                          │
            ┌─────────────────────┐               │
            │   VLM pretrain sẵn   │              │
            │  (internet-scale)    │              │
            └──────────┬───────────┘              │
                       ▼                          │
              vector điều kiện (context           │
              embedding: hiểu cảnh + lệnh)         │
                       │                           │
                       ▼                           ▼
            ┌───────────────────────────────────────┐
            │     Flow-matching action expert        │
            │  (mô-đun riêng, KHÔNG phải phần tiếp   │
            │   nối autoregressive của VLM)           │
            │                                         │
            │  nhận: nhiễu ngẫu nhiên + điều kiện     │
            │  học: trường vận tốc dẫn nhiễu → action │
            └──────────────────┬──────────────────────┘
                                ▼
                  tích phân trường vận tốc
                  (nhiều bước ODE, không phải
                   sinh token tuần tự)
                                ▼
              action CHUNK liên tục — toàn bộ
              chuỗi hành động nhiều bước thời
              gian, được sinh ra CÙNG LÚC
```

Các bước cụ thể:

1. VLM pretrain sẵn xử lý ảnh + lệnh ngôn ngữ (và trong một số biến thể, cả trạng thái robot) để tạo ra một biểu diễn điều kiện (context/conditioning embedding) — đây là bước "hiểu cảnh", tương tự vai trò của Llama 2 trong OpenVLA nhưng **không** dùng phần này để tự sinh hành động.
2. Một mô-đun action expert riêng (kiến trúc dạng transformer nhỏ hơn, huấn luyện theo mục tiêu flow matching) nhận biểu diễn điều kiện đó cùng với một điểm nhiễu ngẫu nhiên ban đầu.
3. Action expert học một trường vận tốc (vector field) mô tả hướng "chảy" từ phân bố nhiễu tới phân bố action chunk hợp lệ, có điều kiện theo ngữ cảnh ở bước 1.
4. Tại inference, một bộ giải ODE tích phân trường vận tốc này qua một số bước cố định (thường ít hơn nhiều so với số token cần sinh nếu làm autoregressive) để ra được toàn bộ action chunk — một chuỗi hành động liên tục nhiều bước thời gian, sinh ra cùng lúc chứ không tuần tự từng token.
5. Vì hành động được sinh theo chunk liên tục, π₀ tự nhiên hơn cho tín hiệu điều khiển motor thật — không bị giới hạn bởi độ phân giải rời rạc hoá và không cần chờ tuần tự từng bước decode.

Chi tiết toán học đầy đủ của flow matching (vector field, ODE solver, so sánh với diffusion cổ điển) được trình bày trong `BAI-GIANG-flow-matching.md` — ở đây chỉ dùng đủ để thấy vì sao đây là lựa chọn kiến trúc khác hẳn OpenVLA.

## 🧮 Ví dụ tính tay / walkthrough cụ thể

### Ví dụ 1 — diễn giải con số từ chính paper OpenVLA (không tự bịa)

Nguồn trích: OpenVLA "outperforming closed models such as RT-2-X (55B) by 16.5% in absolute task success rate across 29 tasks and multiple robot embodiments, with 7x fewer parameters".

Tính tay:

```text
Số tham số RT-2-X:     55.000.000.000  (55B)
Số tham số OpenVLA:     7.000.000.000  (7B)

Tỷ lệ giảm tham số  = 55B / 7B ≈ 7,86 lần
                    → paper làm tròn/báo cáo là "7x fewer parameters"

Cải thiện success rate tuyệt đối = 16,5 điểm phần trăm
(ví dụ: nếu RT-2-X đạt X% thành công trung bình trên 29 task,
 OpenVLA đạt khoảng X% + 16,5 điểm phần trăm — đây là ĐIỂM PHẦN TRĂM
 tuyệt đối, không phải % tương đối, cần đọc đúng đơn vị)
```

Ý nghĩa của con số này (đây là diễn giải, không phải trích nguyên văn thêm số liệu mới): một mô hình **nhỏ hơn gần 8 lần** vẫn đạt kết quả tốt hơn đáng kể một mô hình đóng lớn hơn nhiều. Điều này cho thấy: (a) chất lượng và độ đa dạng của dữ liệu huấn luyện (970k demo, nhiều embodiment) có thể quan trọng hơn việc chỉ tăng số tham số; (b) việc chọn đúng cặp vision encoder (DINOv2 + SigLIP bổ sung cho nhau) là một quyết định kiến trúc có tác động thực sự, không chỉ là chi tiết kỹ thuật phụ.

### Ví dụ 2 — minh hoạ khác biệt sinh token rời rạc vs action chunk liên tục

*(Đây là ví dụ tự chọn để minh hoạ cơ chế, không phải số liệu lấy từ paper.)*

Giả sử cần sinh một hành động gồm 4 bước thời gian, mỗi bước là 1 giá trị vị trí gripper theo trục x, giá trị mong muốn xấp xỉ: `[0.10, 0.14, 0.18, 0.21]` (đơn vị mét, minh hoạ).

**Kiểu OpenVLA (autoregressive-token, rời rạc hoá)**: giả sử không gian giá trị được chia thành các "bin" (khoang) rời rạc, ví dụ bước lượng tử 0.02m → giá trị `0.10` được gán token tương ứng bin gần nhất (ví dụ bin số 5), rồi model sinh token cho bước tiếp theo *có điều kiện theo token vừa sinh* — nghĩa là 4 lượt decode tuần tự, mỗi lượt tốn một lượt forward-pass qua Llama 2, và giá trị cuối cùng bị giới hạn theo độ phân giải của bin (0.02m), không thể chính xác hơn mức đó.

**Kiểu π₀ (flow matching, liên tục)**: 4 giá trị này được xem là một điểm duy nhất trong không gian 4 chiều (một "action chunk"). Action expert học một trường vận tốc dẫn từ một điểm nhiễu ngẫu nhiên trong không gian 4 chiều đó tới đúng điểm `[0.10, 0.14, 0.18, 0.21]` (có điều kiện theo ảnh + lệnh). Tại inference, một số bước tích phân cố định (ví dụ 10 bước, con số minh hoạ) sẽ ra toàn bộ 4 giá trị **cùng lúc**, không bị giới hạn bởi lượng tử hoá bin, và số bước tích phân không nhất thiết tăng theo độ dài chunk như số lượt decode autoregressive.

Đây là lý do cốt lõi để nói "OpenVLA sinh từng token" còn "π₀ sinh cả chunk liên tục" — khác nhau về đơn vị sinh ra (token rời rạc và tuần tự vs. vector liên tục và đồng thời), chứ không chỉ khác nhau về "công thức toán học phía sau".

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | OpenVLA | π₀ |
|---|---|---|
| Paper / năm | [arXiv:2406.09246](https://arxiv.org/abs/2406.09246), 2024 | [arXiv:2410.24164](https://arxiv.org/abs/2410.24164), 2024 |
| Tổ chức | Stanford + cộng đồng mở | Physical Intelligence |
| Backbone ngôn ngữ/VLM | Llama 2 | VLM pretrain sẵn (kiến trúc nền dạng PaliGemma theo tài liệu công khai của Physical Intelligence; cần xác minh thêm chi tiết phiên bản chính xác nếu trích dẫn sâu hơn) |
| Visual encoder | Fuse DINOv2 + SigLIP | Encoder thị giác bên trong VLM pretrain (không phải trọng tâm khác biệt chính so với OpenVLA) |
| Cách sinh hành động | Autoregressive, token rời rạc (giống RT-2) | Flow matching, action chunk liên tục |
| Đơn vị sinh ra mỗi lượt | 1 token hành động | Toàn bộ action chunk (nhiều bước thời gian) cùng lúc |
| Độ phân giải hành động | Giới hạn bởi rời rạc hoá (bin) | Liên tục, không lượng tử hoá |
| Tốc độ suy luận cho chuỗi dài | Chậm hơn (decode tuần tự từng token) | Nhanh hơn về nguyên lý (số bước tích phân không tăng tuyến tính theo độ dài chunk như autoregressive) — mức độ nhanh cụ thể phụ thuộc triển khai, cần xác minh thêm nếu so sánh số liệu cụ thể |
| Dữ liệu huấn luyện | 970k demo robot thật, đa robot/embodiment | Dữ liệu robot đa dạng của Physical Intelligence (quy mô/nguồn cụ thể không nằm trong đoạn trích nguồn của bài này — xem `NOI-DUNG-CHI-TIET.md` nếu cần thêm) |
| Quy mô tham số | 7B | Không nêu trong đoạn nguồn trích dẫn của bài này — cần xác minh thêm nếu cần trích dẫn con số cụ thể |
| Vai trò trong dòng lịch sử tới GR00T | Dạy kiến trúc VLA nền tảng (VLM + dữ liệu robot đa dạng, mã nguồn mở để học) | Tiền thân trực tiếp về ý tưởng cho System 1 (flow-matching action head) của GR00T N1/N1.7 |
| Quan hệ với RT-2 | Kế thừa cách sinh token hành động (autoregressive) | Từ bỏ cách sinh token của RT-2, giữ lại triết lý "VLM pretrain cung cấp tri thức" |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Nhầm OpenVLA (bản gốc) cũng dùng flow matching.** Sai — OpenVLA bản gốc (arXiv:2406.09246) sinh hành động autoregressive-token rời rạc, giống RT-2. Chỉ có biến thể sau này là **OpenVLA-OFT** (arXiv:2502.19645, 2025) mới thay bằng đầu ra hành động liên tục (continuous action head, hồi quy L1) — và đó là một paper/recipe khác, công bố sau, không phải OpenVLA gốc.

2. **Nhầm π₀ không dùng VLM pretrain, tưởng nó chỉ là "một mô hình flow matching thuần túy học từ đầu".** Sai — π₀ vẫn xây trên một VLM đã pretrain sẵn để thừa hưởng tri thức internet-scale; điểm khác biệt so với OpenVLA/RT-2 nằm ở **phần sinh hành động** (flow-matching action expert), không phải ở việc bỏ qua VLM.

3. **Nhầm π₀-FAST (2025) với π₀ gốc (2024) là cùng một cơ chế suy luận.** π₀-FAST dùng tokenizer FAST (nén DCT + BPE) để biểu diễn hành động rời rạc và sinh **autoregressive**, khác hẳn cơ chế flow matching của π₀ gốc — dù cùng tên "π₀" và cùng tổ chức Physical Intelligence, đây là hai cách sinh hành động khác nhau, phục vụ mục tiêu khác nhau (huấn luyện nhanh hơn, xử lý tác vụ khéo léo/dexterous hơn, đánh đổi lấy suy luận autoregressive chậm hơn tại inference so với flow matching).

4. **Nhầm con số "16.5%" là cải thiện tương đối (relative improvement) thay vì tuyệt đối (absolute).** Paper nêu rõ đây là "absolute task success rate" — tức 16,5 điểm phần trăm cộng thêm, không phải "tăng 16,5% so với baseline". Đọc nhầm đơn vị này dẫn tới đánh giá sai mức độ cải thiện thực sự.

5. **Nhầm rằng GR00T N1 "dùng lại nguyên khối" mã nguồn/mô hình π₀.** Sai — GR00T N1 kế thừa **công thức kiến trúc** (VLM điều kiện + action head flow-matching sinh chunk liên tục), không phải trực tiếp tái sử dụng checkpoint hay codebase của π₀. π₀ nhắm một tay máy cố định với VLM nền riêng của Physical Intelligence; GR00T N1 dùng VLM riêng (Eagle-2) và phải mở rộng bài toán sang toàn bộ cơ thể humanoid — đây là hai hệ thống độc lập, cùng chia sẻ một *ý tưởng thiết kế*, không phải cùng một mô hình.

## 🏗️ Ví dụ minh hoạ trong dự án này

Trong dự án học tập này, GR00T N1 (mục 2 của `NOI-DUNG-CHI-TIET.md`) có hai "System": System 2 là một VLM pretrain internet-scale (Eagle-2) đóng vai trò "hiểu cảnh + hiểu lệnh" — đúng vị trí kiến trúc mà VLM đóng trong cả OpenVLA lẫn π₀. System 1 là một Diffusion Transformer (DiT) huấn luyện theo mục tiêu **flow matching**, sinh action chunk liên tục cho toàn bộ cơ thể humanoid — đây chính là ý tưởng kiến trúc **kế thừa trực tiếp từ π₀**, không phải một phát minh độc lập của GR00T.

Nói cách khác, chuỗi kế thừa cụ thể là:

```text
π₀:      VLM pretrain (điều kiện) + flow-matching action expert
                              │
                              │  cùng công thức kiến trúc,
                              │  áp dụng cho bài toán khó hơn
                              ▼
GR00T N1: System 2 = VLM pretrain (Eagle-2, điều kiện)
          System 1 = DiT huấn luyện theo flow matching
                      (sinh action chunk cho TOÀN THÂN humanoid,
                       không chỉ một tay máy cố định như π₀)
```

Khác biệt quan trọng cần nhớ (đã nêu trong nguồn `NOI-DUNG-CHI-TIET.md` mục so sánh): π₀ nhắm tới một tay máy cố định (fixed-base manipulator), còn GR00T N1/N1.7 phải sinh hành động phối hợp cho toàn bộ cơ thể humanoid (chân đi lại/giữ thăng bằng + tay thao tác), nên cần thêm một hệ điều khiển toàn thân riêng (SONIC) ở tầng dưới — điều mà π₀ không cần vì bài toán của nó đơn giản hơn về số bậc tự do. Chi tiết dual-system của GR00T được để dành cho `BAI-GIANG-groot-n1-dual-system.md`.

## 🔥 Cập nhật hiện đại / SOTA gần đây

- **OpenVLA-OFT** — *"Fine-Tuning Vision-Language-Action Models: Optimizing Speed and Success"*, [arXiv:2502.19645](https://arxiv.org/abs/2502.19645) (Kim, Finn, Liang; 2025; trang dự án [openvla-oft.github.io](https://openvla-oft.github.io/); code [github.com/moojink/openvla-oft](https://github.com/moojink/openvla-oft)). Đây là một **recipe fine-tuning**, không phải kiến trúc mới từ đầu: kết hợp parallel decoding, action chunking, biểu diễn hành động **liên tục** (thay cho token rời rạc gốc), và một hàm mất mát hồi quy L1 đơn giản, đồng thời đóng băng (freeze) vision tower và dùng LoRA cho backbone LLM. Kết quả: nâng success rate trung bình trên benchmark LIBERO từ 76,5% lên 97,1%, đồng thời tăng thông lượng sinh hành động (throughput) lên tới 26 lần. Trên robot ALOHA hai tay thật, OpenVLA-OFT vượt qua cả π₀ và RDT-1B cùng nhiều policy imitation learning khác, tới 15% success rate trung bình. Điểm đáng chú ý cho bài học này: OpenVLA-OFT là bằng chứng cho thấy chính cộng đồng nghiên cứu VLA cũng nhận ra hạn chế của token rời rạc autoregressive và đã "kéo" OpenVLA về gần hướng continuous action gần giống triết lý π₀ — dù vẫn là hai dòng phát triển tách biệt.

- **π₀-FAST** — dựa trên tokenizer **FAST**: *"FAST: Efficient Action Tokenization for Vision-Language-Action Models"*, [arXiv:2501.09747](https://arxiv.org/abs/2501.09747) (Physical Intelligence, 2025; trang nghiên cứu [pi.website/research/fast](https://www.pi.website/research/fast)). FAST nén chuỗi hành động bằng biến đổi tần số (DCT) rồi mã hoá bằng Byte Pair Encoding (BPE), đạt nén khoảng 10 lần so với cách rời rạc hoá (binning) đơn giản trước đó, cho phép huấn luyện một VLA **autoregressive** trên các tác vụ khéo léo (dexterous) mà cách binning cũ không xử lý nổi, và huấn luyện nhanh hơn tới 5 lần so với π₀ gốc (dựa trên flow matching/diffusion). Đánh đổi: suy luận (inference) autoregressive của π₀-FAST chậm hơn đáng kể so với suy luận qua flow matching của π₀ gốc — đúng như trực giác đã nêu ở phần cơ chế hoạt động.

- **π₀.5** — *"π0.5: a Vision-Language-Action Model with Open-World Generalization"*, [arXiv:2504.16054](https://arxiv.org/abs/2504.16054) (Physical Intelligence, 2025; công bố tại [pi.website/blog/pi05](https://www.pi.website/blog/pi05)). π₀.5 giữ nền tảng kiến trúc từ π₀ nhưng tập trung vào **co-training trên dữ liệu không đồng nhất** (heterogeneous tasks/data) để đạt khả năng tổng quát hoá sang môi trường hoàn toàn mới — ví dụ điều khiển một robot di động dọn dẹp bếp/phòng ngủ trong những ngôi nhà chưa từng xuất hiện trong dữ liệu huấn luyện, thực hiện các hành vi nhiều giai đoạn kéo dài 10–15 phút. Nhóm nghiên cứu ghi nhận việc tăng số lượng nhà huấn luyện từ 3 lên 104 giúp cải thiện rõ rệt khả năng tổng quát hoá — cho thấy độ đa dạng môi trường quan trọng không kém độ đa dạng tác vụ. π₀.5 được báo cáo vượt trội hơn cả π₀ gốc lẫn một bộ lập kế hoạch (planner) dùng GPT-4 trên các tác vụ cấp cao, đặc biệt trong môi trường lộn xộn/bừa bộn. Đây là hướng đi liên quan gián tiếp tới việc GR00T N1.7 mở rộng nguồn dữ liệu huấn luyện (không chỉ teleoperation robot thật mà còn hướng tới quy mô lớn hơn) — nhưng chi tiết N1.7 thuộc phạm vi bài giảng riêng, không mở rộng thêm ở đây.

- **Toàn bộ mã nguồn và checkpoint π₀/π₀.5/π₀-FAST** được Physical Intelligence công bố mở tại repo [github.com/Physical-Intelligence/openpi](https://github.com/Physical-Intelligence/openpi), cho phép sử dụng ngay hoặc fine-tune trên dữ liệu tùy chỉnh.

## ❓ Câu hỏi tự kiểm tra

1. OpenVLA khác RT-2 ở điểm nào quan trọng nhất, và giống RT-2 ở điểm nào về cách sinh hành động?
2. Vì sao nói π₀ là "bước chuyển kiến trúc quan trọng nhất đối với GR00T", chứ không phải OpenVLA?
3. Con số "16.5%" trong paper OpenVLA là cải thiện tuyệt đối hay tương đối? Giải thích vì sao việc phân biệt này quan trọng.
4. OpenVLA-OFT khác OpenVLA gốc ở điểm nào cụ thể? Nó có làm cho OpenVLA "giống π₀ hơn" không, và vì sao vẫn là hai dòng phát triển tách biệt?
5. π₀-FAST và π₀ gốc, cái nào dùng flow matching, cái nào dùng autoregressive-token? Đánh đổi (trade-off) giữa hai cách này là gì?
6. Nếu phải giải thích cho một người mới vì sao GR00T N1 không dùng kiến trúc kiểu OpenVLA cho action head, bạn sẽ nói gì bằng đúng lý do kỹ thuật (không dùng phép ẩn dụ)?

## 📝 Bài tập thực hành

1. **Đọc và tóm tắt kiến trúc**: Vào abstract/phần method của hai paper [OpenVLA](https://arxiv.org/abs/2406.09246) và [π₀](https://arxiv.org/abs/2410.24164), viết một bảng so sánh do chính bạn tự lập (không nhìn bảng trong bài giảng này) gồm ít nhất 5 tiêu chí, sau đó đối chiếu lại với bảng so sánh ở mục "So sánh" phía trên — ghi ra những chỗ bạn hiểu sai hoặc bỏ sót lần đầu.

2. **Truy vết một cải tiến 2025**: Chọn một trong ba cập nhật (OpenVLA-OFT, π₀-FAST, π₀.5), đọc phần method chi tiết hơn phần đã tóm tắt trong bài giảng này, rồi viết 3–5 câu giải thích: (a) nó thay đổi cụ thể điều gì so với bản gốc, (b) vì sao thay đổi đó giải quyết đúng một hạn chế đã nêu trong bài giảng này, (c) một câu hỏi bạn còn chưa chắc chắn cần xác minh thêm.

3. **Vẽ lại sơ đồ bằng lời của bạn**: Không nhìn hai sơ đồ ASCII ở mục "Cơ chế hoạt động", tự vẽ lại (trên giấy hoặc text) luồng dữ liệu của OpenVLA và của π₀ từ đầu vào (ảnh + lệnh) tới đầu ra (hành động), ghi rõ ở bước nào dữ liệu chuyển từ "liên tục" sang "rời rạc" (OpenVLA) hoặc giữ nguyên "liên tục" xuyên suốt (π₀). Sau đó so lại với sơ đồ gốc trong bài và liệt kê những chỗ bạn quên hoặc vẽ sai thứ tự — đây thường là dấu hiệu của một khái niệm chưa nắm chắc.

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

OpenVLA (arXiv:2406.09246) mã nguồn mở hóa công thức kiến trúc VLA của RT-2 — ghép Llama 2 với đặc trưng thị giác fuse từ DINOv2 và SigLIP, huấn luyện trên 970k demo robot thật, đạt kết quả vượt RT-2-X dù chỉ dùng 7B tham số so với 55B (cải thiện 16,5 điểm phần trăm tuyệt đối) — nhưng vẫn sinh hành động theo kiểu autoregressive-token rời rạc, thừa hưởng cả điểm mạnh lẫn giới hạn tốc độ/độ mượt của cách sinh này. π₀ (arXiv:2410.24164, Physical Intelligence) tạo ra bước chuyển kiến trúc quyết định bằng cách giữ nguyên triết lý "VLM pretrain cung cấp tri thức internet-scale" nhưng thay hoàn toàn cơ chế sinh hành động bằng flow matching, sinh ra toàn bộ action chunk liên tục thay vì từng token — và chính công thức "VLM (điều kiện) + action head chuyên biệt sinh liên tục" này được GR00T N1/N1.7 kế thừa trực tiếp làm nền tảng cho kiến trúc dual-system của mình. Các cập nhật 2025 (OpenVLA-OFT, π₀-FAST, π₀.5) cho thấy cả hai dòng vẫn tiếp tục tiến hoá — OpenVLA dịch chuyển gần hơn về phía continuous action, còn π₀ mở rộng theo cả hướng tokenization hiệu quả hơn (FAST) lẫn khả năng tổng quát hoá môi trường mở (π₀.5) — nhưng bản thân sự phân nhánh kiến trúc gốc giữa "token rời rạc autoregressive" và "flow matching liên tục" vẫn là bài học cốt lõi cần nắm trước khi học tiếp về GR00T.
