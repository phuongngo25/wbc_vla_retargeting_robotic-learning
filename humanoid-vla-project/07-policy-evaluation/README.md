# 07 — Policy Evaluation

> Chạy được policy chưa đủ — bạn cần **chứng minh bằng số liệu** nó tốt tới đâu, tốt hơn baseline nào, và thất bại ở đâu. Đây là mảng dễ bị bỏ qua nhất khi làm robot learning, nhưng lại chính là nơi quyết định một kết quả có thuyết phục được reviewer/mentor hay không.

> 📖 **Giải thích chi tiết đầy đủ** (công thức metric, cách đo sim-to-real gap, benchmark) cho từng khái niệm ở mục A: xem `NOI-DUNG-CHI-TIET.md`.

---

## A. Khái niệm cần nắm, theo thứ tự

1. **Metric cho motion tracking:** tracking error (khoảng cách pose robot vs. pose tham chiếu theo từng khớp/từng frame), success rate (hoàn thành task hay không, ví dụ đi hết quãng đường mà không ngã), độ mượt chuyển động (jerk/gia tốc), độ lệch quỹ đạo gốc (root trajectory error).
   > 📚 **Đọc thêm:** các metric này được định nghĩa formal nhất trong phần Evaluation của **DeepMimic** ([arXiv:1804.02717](https://arxiv.org/abs/1804.02717)) và **AMP** ([arXiv:2104.02180](https://arxiv.org/abs/2104.02180)) — đã dẫn ở `../04-imitation-learning-rl/`. Đọc lại đúng phần "Evaluation Metrics" thay vì phần Method.
2. **Metric cho VLA/loco-manipulation:** success rate theo từng loại task ngôn ngữ, generalization (task chưa thấy trong huấn luyện), robustness (thay đổi ánh sáng, vật thể, môi trường).
   > 📚 **Đọc thêm:** phần Experiments của **GR00T N1** ([arXiv:2503.14734](https://arxiv.org/abs/2503.14734), đã dẫn ở `../06-vla-groot-sonic/`) — họ báo cáo success rate theo nhiều nhóm task khác nhau kèm so sánh baseline, là mẫu tốt để học cách phân nhóm metric.
3. **Sim-to-real gap:** đo bằng cách so sánh performance policy *trong simulation* vs. *trên robot thật* cùng 1 tập task — SONIC báo cáo "100% success rate trên 50 trajectory thật, zero-shot" — đây là một cách trình bày sim-to-real gap = 0 rất mạnh, cần hiểu **thiết kế thí nghiệm** đằng sau con số này (bao nhiêu lần thử, điều kiện gì, so với baseline nào).
   > 📚 **Đọc thêm:** **Zhao, Queralta, Westerlund (2020)** — *"Sim-to-Real Transfer in Deep Reinforcement Learning for Robotics: A Survey"*, IEEE SSCI 2020. Survey tổng quan cách cộng đồng đo và báo cáo sim-to-real gap trước khi đọc cách SONIC làm cụ thể — giúp nhận ra SONIC đang làm gì khác biệt (zero-shot, không fine-tune) so với đa số phương pháp trong survey. [arXiv:2009.13303](https://arxiv.org/abs/2009.13303)
4. **So sánh công bằng (fair comparison):** cùng 1 bộ test set, cùng số lần thử, cùng điều kiện môi trường — nguyên tắc chung đã có ở `../../resources/07-experiments-and-rigor.md`, áp dụng trực tiếp vào robot learning (nơi rất dễ "cherry-pick" 1 lần chạy đẹp).
5. **Benchmark chuẩn của cộng đồng:** một số paper trong `humanoid-wbc-review` (`../01-whole-body-control/`) có kèm benchmark suite chuẩn hoá (ví dụ HumanoidVerse có sẵn eval pipeline) — dùng benchmark có sẵn thay vì tự chế để so sánh được với paper khác.
   > 📚 **Đọc thêm:** **Sferrazza, Huang et al. (UC Berkeley, 2024)** — *"HumanoidBench: Simulated Humanoid Benchmark for Whole-Body Locomotion and Manipulation"*, RSS 2024. Benchmark chuẩn hoá đầu tiên riêng cho humanoid (27 task, nhiều loại robot/tay), dùng MuJoCo — nếu bạn cần so sánh policy của mình với paper khác một cách công bằng, đây là bộ test set nên dùng thay vì tự chế. [humanoid-bench.github.io](https://humanoid-bench.github.io/) · [Paper PDF](https://www.roboticsproceedings.org/rss20/p061.pdf)

---

## B. Tài liệu chính

| Tài liệu | Vì sao đọc | Link |
|---|---|---|
| `../../resources/07-experiments-and-rigor.md` | 🔴 Nền tảng bắt buộc — thiết kế thí nghiệm, thống kê, chọn metric đúng, các cạm bẫy phổ biến (đã viết sẵn trong repo chính, áp dụng được trực tiếp). | Đọc trong repo này |
| **Metrics Reloaded** — [arXiv:2206.01653](https://arxiv.org/abs/2206.01653) | 🔴 Tuy viết cho ảnh y tế nhưng nguyên tắc chọn metric đúng theo bài toán là tổng quát — áp dụng tư duy tương tự khi chọn metric cho robot. | Đã có trong `../../resources/08-core-reading-list.md` B4.5 |
| **SONIC — phần thực nghiệm (Section Experiments)** | 🔴 Đọc kỹ *cách họ thiết kế* thí nghiệm zero-shot real-world (50 trajectory, điều kiện thử, baseline so sánh) — đây là mẫu thực nghiệm chuẩn cho lĩnh vực. | [arXiv:2511.07820](https://arxiv.org/abs/2511.07820) |
| **HumanoidVerse — eval pipeline** | 🟡 Xem code eval có sẵn để hiểu benchmark thực tế trông như thế nào (không chỉ lý thuyết). | [GitHub](https://github.com/LeCAR-Lab/HumanoidVerse) |

---

## C. Video hướng dẫn

Không có video chuyên biệt cho "đánh giá policy humanoid" — đây là kỹ năng học tốt nhất qua đọc kỹ phần Experiments của các paper đã nêu, kết hợp với nguyên tắc thiết kế thí nghiệm tổng quát:

| Video | Ngôn ngữ | Nội dung |
|---|---|---|
| CS285 (Levine) — các bài giảng về evaluation trong RL | Tiếng Anh | Cách đánh giá policy RL đúng cách (variance giữa seeds, số lần chạy tối thiểu). |

---

## D. Lộ trình từng bước cho người mới bắt đầu

1. **(2–3 ngày)** Đọc `../../resources/07-experiments-and-rigor.md` toàn bộ — đây là nền chung cho mọi thí nghiệm bạn làm sau này trong dự án.
2. **(2–3 ngày)** Đọc kỹ phần Experiments của SONIC — liệt kê ra: họ đo metric gì, bao nhiêu lần thử, so sánh với baseline nào, report variance/confidence interval ra sao (nếu có).
3. **(3–5 ngày)** Tự thiết kế 1 bộ metric nhỏ (3–5 con số) cho policy bạn đã huấn luyện ở `04-imitation-learning-rl/`/`05-simulation-mujoco-isaaclab/` — ví dụ: tracking error trung bình, success rate trên 10 lần chạy với seed khác nhau, độ mượt chuyển động.
4. **(checkpoint)** Chạy đánh giá này trên policy của bạn, report kết quả kèm variance (không chỉ 1 con số duy nhất) — tập thói quen này ngay từ đầu, nó là khác biệt lớn nhất giữa "demo đẹp" và "kết quả nghiên cứu đáng tin".

---

## Liên kết chéo

- Đánh giá policy WBC từ `01-whole-body-control/`/`04-imitation-learning-rl/` huấn luyện ở `05-simulation-mujoco-isaaclab/`.
- Đánh giá VLA từ `06-vla-groot-sonic/`.
- Kết quả đánh giá sim-to-real gap dẫn thẳng tới quyết định deploy ở `08-real-robot-deployment/`.
