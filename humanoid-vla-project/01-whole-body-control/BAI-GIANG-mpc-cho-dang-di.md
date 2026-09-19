# Bài giảng: MPC cho dáng đi (mức khái niệm)

*(Thuộc mảng: Whole-Body Control (WBC))*

## 🎯 Mục tiêu bài học

- Giải thích được ý tưởng "receding horizon" (cửa sổ trượt) — điểm khác biệt cốt lõi giữa MPC và preview control.
- Liệt kê được 2 khác biệt cụ thể giữa MPC và preview control: xử lý ràng buộc bất đẳng thức/phi tuyến, và tần suất giải bài toán tối ưu.
- Tính tay được một ví dụ đơn giản minh hoạ cơ chế "giải lại toàn bộ, chỉ dùng bước đầu tiên" của MPC.
- Giải thích được vì sao MPC phù hợp hơn preview control khi cần đưa trực tiếp giới hạn lực ma sát/vị trí đặt chân rời rạc vào bài toán.
- Nêu được 2 ví dụ thực tế MPC được dùng trong robot chân trước khi RL phổ biến (MIT Cheetah, ANYmal).
- Liên hệ được xu hướng hiện đại: MPC không biến mất mà đang được kết hợp với RL (RL-augmented MPC).

## 🧭 Vì sao cần học cái này? (bối cảnh)

Bài trước (ZMP preview control) đã học một bộ điều khiển "nhìn trước" nhưng có gain **cố định, tính một lần**. MPC là bước phát triển tự nhiên tiếp theo của đúng tư duy "nhìn trước" đó, nhưng tổng quát hoá triệt để hơn: thay vì tính gain một lần rồi áp mãi, MPC **giải lại toàn bộ bài toán tối ưu ở mỗi bước thời gian**. Đây là khái niệm cuối cùng của mảng "WBC cổ điển (model-based)" trong thư mục này trước khi chuyển sang mảng "WBC học sâu" — học kỹ MPC ở đây giúp hiểu chính xác **cái gì mà RL đang cố thay thế** (mục "Vì sao RL thay thế được model-based control", bài tiếp theo), thay vì so sánh mơ hồ.

## 🧠 Trực giác

### Góc nhìn 1: GPS chỉ đường tính lại tuyến đường mỗi khi bạn rẽ nhầm, không chỉ đưa một lộ trình cố định

Một bản đồ giấy in sẵn tuyến đường (giống preview control: tính trước, cố định) không thể phản ứng nếu bạn rẽ nhầm hoặc gặp đường tắc. Một GPS hiện đại thì khác: **mỗi khi có thông tin mới** (bạn rẽ nhầm, có tắc đường phía trước), nó **tính lại toàn bộ lộ trình từ vị trí hiện tại**, nhưng bạn chỉ thực sự đi theo **bước tiếp theo** của lộ trình mới đó (rẽ ở ngã tư kế tiếp) — rồi nếu có thêm thông tin mới, GPS lại tính lại từ đầu. MPC hoạt động đúng như vậy: giải lại toàn bộ bài toán tối ưu trên cửa sổ tương lai ở mỗi bước, chỉ dùng đúng bước điều khiển đầu tiên của lời giải.

**Giới hạn của loại suy này:** GPS tính lại lộ trình vì có **thông tin mới về thế giới bên ngoài** (tắc đường, rẽ nhầm); MPC cho dáng đi tính lại **ngay cả khi không có gì bất ngờ xảy ra** — đơn giản vì đó là cách hoạt động mặc định của thuật toán (receding horizon), không chỉ để phản ứng với sai lệch.

### Góc nhìn 2: Người chơi cờ tính trước nhiều nước đi nhưng chỉ thực sự đánh một nước, rồi tính lại từ đầu

Một kỳ thủ giỏi khi tính nước cờ không chỉ nghĩ "nước tiếp theo", mà **mô phỏng trước cả một chuỗi 5-10 nước tiếp theo** (bao gồm cả phản ứng dự kiến của đối thủ), chọn ra chuỗi tốt nhất, nhưng **chỉ thực sự đi đúng một nước đầu tiên** của chuỗi đó. Sau khi đối thủ phản hồi, kỳ thủ lại tính toán lại từ đầu một chuỗi 5-10 nước mới (không phải "tiếp tục" chuỗi cũ, mà tính lại hoàn toàn với thông tin mới). MPC làm đúng việc này với chuyển động robot: tối ưu một chuỗi hành động trên cửa sổ tương lai, chỉ thực thi bước đầu, rồi tính lại toàn bộ.

**Giới hạn của loại suy này:** kỳ thủ cờ vua đối mặt với một đối thủ có ý chí riêng (adversarial, khó dự đoán hoàn toàn); MPC cho dáng đi thường giả định **mô hình động lực học đã biết** (không có "đối thủ" chủ động, dù có thể có nhiễu/địa hình bất định) — bài toán tối ưu MPC vì vậy thường dễ giải hơn nhiều so với việc "đọc vị" một đối thủ có chiến lược.

## 📐 Định nghĩa chính xác

**Model Predictive Control (MPC)** là bước phát triển tiếp theo của tư duy "nhìn trước" ở preview control, nhưng tổng quát hơn: tại mỗi bước thời gian, MPC **giải lại một bài toán tối ưu trên toàn bộ cửa sổ tương lai** (ví dụ tối ưu vị trí chân đặt xuống, lực tiếp xúc, quỹ đạo trọng tâm trong 1 giây tới), chỉ áp dụng **bước điều khiển đầu tiên** của lời giải, rồi **lặp lại toàn bộ quá trình tối ưu ở bước kế tiếp** với thông tin cập nhật mới nhất (vị trí thật, nhiễu vừa đo được).

Dạng tổng quát của bài toán tối ưu tại mỗi bước `k`:

```text
minimize_{u(k),...,u(k+H-1)}   Σᵢ₌₀ᴴ⁻¹ ℓ(x(k+i), u(k+i)) + ℓ_f(x(k+H))

subject to    x(k+i+1) = f(x(k+i), u(k+i))     (động lực học, i=0..H-1)
              g(x(k+i), u(k+i)) ≤ 0             (ràng buộc bất đẳng thức, i=0..H-1)
              x(k) = x_hiện_tại (đo được)
```

trong đó `H` là **horizon** (độ dài cửa sổ tối ưu, ví dụ tương đương 1 giây), `ℓ` là chi phí từng bước, `ℓ_f` là chi phí cuối cửa sổ, `g(·)≤0` là các ràng buộc bất đẳng thức (giới hạn lực ma sát, giới hạn động cơ, vị trí đặt chân rời rạc).

**Khác biệt cốt lõi so với preview control** (đã học ở bài trước):

1. **Ràng buộc bất đẳng thức/phi tuyến trực tiếp:** preview control chỉ có mô hình tuyến tính thuần (không xử lý được ràng buộc dạng bất đẳng thức trong công thức gain cố định); MPC đưa thẳng các ràng buộc này (`g(·)≤0`) vào cùng một bài toán tối ưu.
2. **Giải lại mỗi bước (receding horizon):** preview control tính gain một lần offline; MPC giải một bài toán tối ưu mới hoàn toàn ở **mỗi bước thời gian** — tốn chi phí tính toán hơn nhưng linh hoạt hơn nhiều khi ràng buộc/mục tiêu có thể thay đổi.

MPC là hướng được dùng rộng rãi trong WBC cổ điển hiện đại (ví dụ MIT Cheetah, ANYmal) **trước khi RL trở nên phổ biến**.

## ⚙️ Cơ chế hoạt động — từng bước

```text
┌────────────────────────────────────────────────────────────┐
│ Tại bước thời gian k:                                          │
│  1. Đo trạng thái hiện tại x(k) (vị trí/vận tốc thật, từ        │
│     cảm biến/estimator)                                         │
│  2. Giải bài toán tối ưu trên cửa sổ [k, k+H]:                  │
│     tìm chuỗi u(k), u(k+1), ..., u(k+H-1) tối ưu hoá chi phí      │
│     VÀ thoả mãn ràng buộc động lực học + bất đẳng thức           │
│     (thường quy về QP nếu mô hình/ràng buộc tuyến tính hoá được) │
│  3. CHỈ áp dụng u(k) — bước điều khiển ĐẦU TIÊN của lời giải     │
│     (KHÔNG dùng u(k+1)...u(k+H-1) dù đã tính ra)                 │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ Robot thực thi u(k), thời gian trôi qua 1 bước                 │
│ Đo trạng thái MỚI x(k+1) (đã bao gồm nhiễu/sai lệch thực tế)    │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ VỨT BỎ toàn bộ lời giải cũ (u(k+1)...u(k+H-1) không dùng nữa)  │
│ Cửa sổ tối ưu "TRƯỢT" tới [k+1, k+1+H]                          │
│ Quay lại bước 1 với k := k+1                                    │
└──────────────────────────────────────────────────────────────┘
```

Sơ đồ minh hoạ "cửa sổ trượt" (receding horizon):

```text
Bước k:    [────────── cửa sổ tối ưu H bước ──────────]
           k  k+1  k+2  ...              k+H-1
           ▲
           chỉ dùng u(k), rồi trượt cửa sổ

Bước k+1:      [────────── cửa sổ tối ưu H bước ──────────]
               k+1  k+2  k+3  ...              k+H
               ▲
               chỉ dùng u(k+1), rồi trượt tiếp...
```

## 🧮 Ví dụ tính tay / walkthrough cụ thể

*(Ví dụ minh hoạ tối giản — bài toán MPC 1 chiều với chi phí bậc hai, không ràng buộc, để thấy rõ cơ chế "giải lại, chỉ dùng bước đầu" — không phải bài toán MPC dáng đi thật với đầy đủ ràng buộc lực tiếp xúc)*

Xét hệ động lực học đơn giản `x(k+1) = x(k) + u(k)` (tích phân đơn), mục tiêu đưa `x` về 0, horizon `H=2`, chi phí `ℓ(x,u) = x² + 0.5u²`.

**Bước k=0, trạng thái đo được `x(0) = 4`:**

Bài toán tối ưu: chọn `u(0), u(1)` để tối thiểu `x(1)² + 0.5u(0)² + x(2)² + 0.5u(1)²`, với `x(1)=x(0)+u(0)=4+u(0)`, `x(2)=x(1)+u(1)`.

Để đơn giản hoá tính tay, giả sử ta giới hạn bài toán chỉ chọn `u(0)` sao cho tối thiểu hoá chi phí 2 bước, với heuristic đơn giản "chia đều": đưa `x` về 0 qua 2 bước đều nhau, tức muốn `x(1) = 2` (đi được nửa đường):

```text
u(0) = x(1) − x(0) = 2 − 4 = −2
```

Kiểm tra chi phí bước này: `x(1)² + 0.5u(0)² = 2² + 0.5×(−2)² = 4 + 2 = 6`.

**So sánh với phương án "đi hết luôn trong 1 bước" (`u(0) = −4`, đưa `x(1)=0` ngay):**

```text
x(1)² + 0.5u(0)² = 0² + 0.5×(−4)² = 0 + 8 = 8
```

**Kết luận cục bộ:** phương án "chia đều 2 bước" (chi phí bước 1 = 6) tốt hơn phương án "đi hết ngay" (chi phí bước 1 = 8) vì chi phí điều khiển (`0.5u²`) tăng theo bình phương — MPC với horizon H=2 "nhìn thấy" rằng dùng lực nhỏ hơn trong 2 bước rẻ hơn dùng lực lớn trong 1 bước, đúng bản chất tối ưu hoá trên cả cửa sổ tương lai, không chỉ bước hiện tại.

**Bước k=0 → thực thi:** robot áp `u(0) = −2`. Giả sử có nhiễu nhỏ, trạng thái thực tế đo được sau đó là `x(1) = 2.3` (không đúng khớp 2.0 như tính toán, do nhiễu).

**Bước k=1 (GIẢI LẠI TỪ ĐẦU với trạng thái thực tế mới `x(1)=2.3`, horizon vẫn H=2, cửa sổ trượt tới [1,2]):** áp cùng heuristic "chia đều":

```text
u(1) = 0 − 2.3 = −2.3  (nếu horizon còn lại chỉ 1 bước tại biên cuối cùng của kế hoạch gốc)
```

**Ý nghĩa:** dù kế hoạch ban đầu tại bước k=0 đã "dự tính" `x(1)=2`, MPC **không cố chấp bám theo kế hoạch cũ** khi thực tế lệch thành `2.3` — nó **vứt bỏ hoàn toàn** phần còn lại của kế hoạch cũ và **tính lại từ đầu** dựa trên trạng thái thực tế mới nhất. Đây chính là điểm khác biệt cơ bản với một bộ điều khiển "mở vòng" (open-loop) chỉ chạy theo một kế hoạch cố định đã lập từ đầu — MPC luôn đóng vòng (closed-loop) qua việc giải lại liên tục.

## 🔍 So sánh với phương pháp/khái niệm liên quan

| Tiêu chí | MPC | ZMP preview control (bài trước) | RL end-to-end (bài tiếp theo) |
|---|---|---|---|
| Tần suất giải bài toán tối ưu | Mỗi bước (online) | Một lần (offline), gain cố định | Không giải tối ưu online — policy đã học sẵn, chỉ forward pass mạng |
| Xử lý ràng buộc bất đẳng thức/phi tuyến | Trực tiếp, native | Khó/không xử lý trực tiếp | Học ngầm qua reward/huấn luyện, không đảm bảo tường minh |
| Chi phí tính toán online | Cao hơn preview, thấp hơn RL forward pass thường | Thấp nhất | Rất thấp (chỉ forward pass, có thể chạy hàng trăm Hz) |
| Đảm bảo toán học (constraint satisfaction) | Có, tường minh | Có, nhưng hạn chế (không ràng buộc bất đẳng thức) | Không đảm bảo tường minh, chỉ "khả năng cao" nếu huấn luyện tốt |
| Cần mô hình động lực học chính xác | Có, bắt buộc | Có (mô hình đơn giản hoá) | Không cần mô hình tường minh (học từ tương tác/dữ liệu) |
| Khả năng tổng quát hoá sang tình huống mới | Hạn chế bởi mô hình đã giả định | Hạn chế hơn MPC | Tốt hơn nếu huấn luyện đa dạng, nhưng không đảm bảo |

## ⚠️ Sai lầm thường gặp / hiểu nhầm phổ biến

1. **Hiểu nhầm: "MPC lãng phí tính toán vì tính ra cả chuỗi H bước nhưng chỉ dùng 1 bước".** Vì sao sai: chính việc "nhìn thấy" các bước tương lai trong bài toán tối ưu là điều làm cho **bước đầu tiên** được chọn đúng — như ví dụ tính tay cho thấy, nếu chỉ tối ưu 1 bước (không nhìn xa hơn), phương án được chọn có thể tệ hơn (chi phí 8 so với 6). **Hiểu đúng:** các bước H-1 còn lại "bị bỏ" không phải lãng phí — chúng là công cụ trung gian bắt buộc để tính đúng bước đầu tiên, giống việc kỳ thủ cờ phải tính trước nhiều nước để chọn đúng nước đi kế tiếp, dù không thực sự đi hết chuỗi đó.
2. **Hiểu nhầm: "MPC đã lỗi thời, giờ ai cũng dùng RL".** Vì sao sai: các nghiên cứu 2025 (xem mục Cập nhật hiện đại) cho thấy MPC vẫn có ưu thế rõ ràng ở khả năng đảm bảo ràng buộc tường minh và không cần dữ liệu huấn luyện — trong khi RL có ưu thế về khả năng xử lý nhiễu và không cần mô hình chính xác nhưng khó tổng quát hoá sang địa hình/mô hình chưa từng thấy. **Hiểu đúng:** xu hướng thực tế 2025-2026 là **kết hợp** (RL-augmented MPC, MPC-compensated RL) chứ không phải "RL thay thế hoàn toàn MPC" — mỗi phương pháp bù đắp điểm yếu của phương pháp kia.

## 🏗️ Ví dụ minh hoạ trong dự án này

```text
Kế hoạch bước chân (footstep planning)
        │
        ▼
MPC: tại mỗi bước thời gian (ví dụ 100-500Hz tuỳ triển khai),
  giải bài toán tối ưu trên cửa sổ ~1 giây tương lai:
  - biến quyết định: lực tiếp xúc chân, vị trí đặt chân (có thể
    rời rạc), quỹ đạo trọng tâm
  - ràng buộc: friction cone, giới hạn động cơ, không nhấc/trượt chân
        │
        ▼
CHỈ áp dụng bước điều khiển đầu tiên → lệnh lực/vị trí khớp
        │
        ▼
Đo trạng thái mới, GIẢI LẠI toàn bộ ở bước tiếp theo
        │
        ▼
Trong dự án này: MPC là "chuẩn cổ điển" mà pipeline RL hiện đại
  (04-imitation-learning-rl/, PHC→OmniH2O→ASAP→SONIC) được so sánh với —
  hiểu MPC giúp đánh giá đúng khi nào một policy RL "học lại" đúng
  hành vi mà MPC vốn đã giải tường minh (ví dụ giữ ZMP/DCM trong đế chân),
  và khi nào RL vượt trội (hành vi phức tạp không thể viết công thức)
```

## 🔥 Cập nhật hiện đại / SOTA gần đây

1. **Benchmark trực tiếp MPC vs RL trên cùng nền tảng mô phỏng (MuJoCo).** *"Benchmarking Model Predictive Control and Reinforcement Learning Based Control for Legged Robot Locomotion in MuJoCo Simulation"* ([arXiv:2501.16590](https://arxiv.org/abs/2501.16590), 2025) so sánh trực tiếp hai cách tiếp cận: kết quả cho thấy RL "vượt trội trong việc xử lý nhiễu và duy trì hiệu quả năng lượng, nhưng gặp khó khăn tổng quát hoá sang địa hình mới do phụ thuộc vào policy đã học cho môi trường cụ thể" — một xác nhận thực nghiệm trực tiếp cho đánh đổi đã nêu ở mục So sánh (MPC: đảm bảo tường minh nhưng hạn chế bởi mô hình; RL: linh hoạt hơn nhưng khó tổng quát hoá).
2. **RL-compensated MPC — xu hướng hybrid nổi bật 2025-2026.** Một hướng nghiên cứu gần đây dùng RL để **bù trừ (compensate)** cho các thành phần khó mô hình hoá chính xác trong MPC — ví dụ một policy RL học cách điều chỉnh gia tốc tuyến tính/góc, tần số dáng đi, và vị trí đặt chân, kết hợp với MPC làm khung tối ưu chính — báo cáo cải thiện hiệu năng trên địa hình không đều so với MPC thuần. Một hướng khác kết hợp **online learning of residual dynamics** (học phần dư của mô hình động lực học mà MPC dùng) để MPC thích nghi tốt hơn với sai lệch mô hình theo thời gian thực — đây chính là ứng dụng cụ thể của tư tưởng "residual learning" đã gặp ở retargeting (`02-motion-retargeting/`, Villegas et al.) nhưng áp dụng cho điều khiển thay vì retargeting.
3. **Xu hướng chung:** không có bằng chứng MPC bị loại bỏ khỏi thực tiễn robot chân hiện đại — thay vào đó, ranh giới giữa "model-based (MPC)" và "learning-based (RL)" đang mờ dần qua các kiến trúc lai, mỗi phương pháp đóng góp đúng thế mạnh riêng (MPC: đảm bảo ràng buộc tường minh, không cần dữ liệu; RL: xử lý nhiễu/tình huống chưa mô hình hoá được).

## ❓ Câu hỏi tự kiểm tra

1. "Receding horizon" (cửa sổ trượt) nghĩa là gì, và nó khác gì so với việc chỉ lập kế hoạch một lần từ đầu rồi chạy theo?
   <details><summary>Gợi ý đáp án</summary>Receding horizon nghĩa là bài toán tối ưu được giải lại ở MỖI bước thời gian trên một cửa sổ tương lai mới (dịch chuyển theo thời gian), chỉ dùng bước điều khiển đầu tiên mỗi lần — khác với lập kế hoạch một lần rồi chạy theo (open-loop), MPC luôn cập nhật kế hoạch dựa trên trạng thái thực tế mới nhất, closed-loop.</details>
2. Trong ví dụ tính tay, vì sao phương án "chia đều 2 bước" (chi phí 6) tốt hơn "đi hết trong 1 bước" (chi phí 8)?
   <details><summary>Gợi ý đáp án</summary>Vì chi phí điều khiển `0.5u²` tăng theo bình phương độ lớn `u` — dùng lực nhỏ hơn trải trên nhiều bước có tổng chi phí thấp hơn dùng lực lớn trong một bước, một hiệu ứng chỉ "nhìn thấy" được khi tối ưu hoá trên cả cửa sổ nhiều bước (horizon H=2), không chỉ bước hiện tại.</details>
3. Hai khác biệt cốt lõi giữa MPC và ZMP preview control là gì?
   <details><summary>Gợi ý đáp án</summary>(1) MPC xử lý ràng buộc bất đẳng thức/phi tuyến trực tiếp trong bài toán tối ưu, preview control thì không; (2) MPC giải lại bài toán tối ưu mỗi bước (online), preview control tính gain một lần offline và giữ cố định.</details>
4. Theo benchmark arXiv:2501.16590, RL vượt trội MPC ở điểm nào và kém hơn ở điểm nào?
   <details><summary>Gợi ý đáp án</summary>RL vượt trội trong xử lý nhiễu và hiệu quả năng lượng; RL kém hơn ở khả năng tổng quát hoá sang địa hình mới, vì phụ thuộc vào policy đã học riêng cho môi trường huấn luyện cụ thể.</details>
5. Vì sao xu hướng "RL-compensated MPC" cho thấy MPC và RL không phải hai lựa chọn loại trừ nhau?
   <details><summary>Gợi ý đáp án</summary>Vì các kiến trúc hybrid này dùng MPC làm khung tối ưu chính (đảm bảo ràng buộc tường minh) trong khi để RL bù trừ đúng những phần khó mô hình hoá chính xác (ví dụ điều chỉnh gia tốc, tần số dáng đi trên địa hình không đều) — mỗi phương pháp đóng góp đúng thế mạnh riêng thay vì thay thế hoàn toàn nhau.</details>

## 📝 Bài tập thực hành

1. **Tính tay biến thể khác:** với cùng hệ động lực học `x(k+1)=x(k)+u(k)`, chi phí `ℓ=x²+0.5u²`, horizon H=3, trạng thái ban đầu `x(0)=6` — dùng heuristic "chia đều 3 bước" (`x(1)=4, x(2)=2, x(3)=0`), tính `u(0), u(1), u(2)` và tổng chi phí 3 bước, so sánh với phương án "đi hết trong 1 bước tại k=0" (`u(0)=−6`, các bước sau `u=0`).
2. **Đọc paper thật:** đọc phần "Results" của [arXiv:2501.16590](https://arxiv.org/abs/2501.16590) — tìm ít nhất 2 con số/kết luận định lượng cụ thể (không phải chỉ nhận định định tính) so sánh MPC và RL trên cùng một địa hình thử nghiệm, ghi chú lại metric đo (thời gian, năng lượng, độ ổn định...).

## 🧩 Tóm tắt bằng 1 đoạn duy nhất

MPC mở rộng tư duy "nhìn trước" của preview control bằng cách giải lại toàn bộ bài toán tối ưu trên một cửa sổ tương lai (horizon) ở MỖI bước thời gian, chỉ áp dụng bước điều khiển đầu tiên rồi lặp lại (receding horizon) — như ví dụ tính tay minh hoạ, việc "nhìn thấy" các bước tương lai trong bài toán tối ưu giúp chọn đúng bước hiện tại tốt hơn nhiều so với tối ưu cận thị (chỉ 1 bước), đồng thời khả năng đưa trực tiếp ràng buộc bất đẳng thức (lực ma sát, giới hạn động cơ, vị trí đặt chân) vào bài toán tối ưu là ưu thế cốt lõi so với preview control. MPC là chuẩn WBC cổ điển được dùng rộng rãi (MIT Cheetah, ANYmal) trước khi RL phổ biến; benchmark trực tiếp 2025 (arXiv:2501.16590) xác nhận đánh đổi rõ ràng giữa hai cách tiếp cận (MPC: đảm bảo tường minh nhưng hạn chế tổng quát hoá; RL: linh hoạt hơn nhưng khó đảm bảo ràng buộc), và xu hướng 2025-2026 đang hội tụ về các kiến trúc hybrid (RL-compensated MPC) kết hợp thế mạnh của cả hai thay vì để một phương pháp thay thế hoàn toàn phương pháp kia.
