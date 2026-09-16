# Research Journey — Từ Engineer đến Researcher

> Repo cá nhân ghi lại lộ trình chuyển từ tư duy kỹ sư (engineer) sang tư duy nghiên cứu (researcher), với đích đến là học lên PhD.
> Lĩnh vực: **Medical Imaging · Robotics · Surgical/Medical Robotics**.

---

## Mục đích của repo này

Repo này có 3 vai trò:

1. **Bản đồ** — một lộ trình rõ ràng để mình không bị lạc lối (`ROADMAP.md`).
2. **Kho tài liệu** — danh sách sách, khóa học, repo, paper, venue được tuyển chọn (`resources/`).
3. **Nhật ký** — nơi ghi lại quá trình học, paper đã đọc, thí nghiệm đã làm (`logs/`, `templates/`).

Nguyên tắc: **học công khai (learn in public)**. Mỗi tuần commit ít nhất một lần. Repo càng "sống", động lực càng bền.

---

## Cấu trúc thư mục

```text
research-journey/
├── README.md                          # File này — tổng quan & hướng dẫn dùng
├── ROADMAP.md                         # ⭐ Lộ trình chính, chia theo giai đoạn
├── resources/
│   ├── 00-research-skills.md          # Phương pháp NC, đọc paper, mindset (tổng quan)
│   ├── 01-foundations.md              # Nền tảng Toán + ML + Computer Vision
│   ├── 02-medical-imaging.md          # Tài liệu chuyên sâu xử lý ảnh y tế
│   ├── 03-robotics.md                 # Tài liệu chuyên sâu robotics
│   ├── 04-surgical-robotics.md        # Giao của robot + y tế + vision
│   ├── 05-phd-preparation.md          # Chuẩn bị hồ sơ & apply PhD
│   ├── 06-writing-and-communication.md  # ⭐ Viết paper, talk, peer review, rebuttal
│   ├── 07-experiments-and-rigor.md      # ⭐ Thiết kế thí nghiệm, thống kê, metric, đạo đức
│   ├── 08-core-reading-list.md          # ⭐ 67 paper nền tảng, xếp thứ tự + kế hoạch 12 tuần
│   └── 09-community-labs-funding.md     # Lab mục tiêu, summer school, mentorship, funding
├── templates/
│   ├── paper-reading-note.md          # Mẫu ghi chú đọc paper (three-pass)
│   ├── weekly-log.md                  # Mẫu nhật ký học tập hàng tuần
│   ├── project-proposal.md            # Mẫu đề cương nghiên cứu mini
│   ├── literature-map.md              # Bản đồ tài liệu → tìm khoảng trống nghiên cứu
│   ├── reproduce-checklist.md         # Checklist tái tạo paper (Phase 2)
│   └── lab-shortlist.md               # Theo dõi lab, deadline, thư giới thiệu (Phase 4–5)
├── logs/
│   └── 2026-W31-example.md            # Ví dụ nhật ký tuần đầu tiên
└── humanoid-vla-project/              # 🤖 Dự án phụ (mentor giao): humanoid robot, WBC, retargeting, VLA (GR00T/SONIC)
    ├── README.md                      # Tổng quan, sơ đồ pipeline, hệ sinh thái GR00T
    ├── ROADMAP.md                     # Lộ trình 14 tuần riêng cho dự án này
    └── 01…08-*/                       # 8 thư mục theo từng mảng (WBC, retargeting, dataset, IL/RL, sim, VLA, evaluation, deployment)
```

**Bốn file quan trọng nhất, theo thứ tự:** `ROADMAP.md` → `00-research-skills.md` → `08-core-reading-list.md` → `07-experiments-and-rigor.md`.

> 🤖 **Dự án phụ do mentor giao (tách riêng khỏi trục PhD chính):** `humanoid-vla-project/` — deep research về humanoid robot, whole-body control, motion retargeting, và VLA (hệ sinh thái NVIDIA GR00T/SONIC). Xem `humanoid-vla-project/README.md` để bắt đầu.

---

## Bắt đầu từ đâu?

**Tuần này:**

1. Đọc `ROADMAP.md` từ đầu tới cuối một lượt để nắm bức tranh tổng thể.
2. Đọc `resources/00-research-skills.md` — quan trọng nhất cho việc **đổi tư duy**.
3. Chọn **một** hướng hẹp làm trục chính (gợi ý trong ROADMAP: Medical Imaging làm trục, Surgical Robotics làm điểm giao mở rộng). Viết 3–5 câu vào `logs/`.
4. Đọc **hai bài** này — không cần gì hơn để bắt đầu:
   - Hamming, *"You and Your Research"* (chọn bài toán quan trọng)
   - Keshav, *"How to Read a Paper"* (three-pass method)

**Tuần 2 trở đi:**

1. Mở `resources/08-core-reading-list.md`, làm theo **kế hoạch đọc 12 tuần** ở cuối file. Mỗi tuần: 1 paper pass-3 + 3–5 paper pass-1/2.
2. Copy `templates/weekly-log.md` vào `logs/` mỗi tuần, và `templates/paper-reading-note.md` cho mỗi paper pass-3.
3. Dựng `templates/literature-map.md` — cập nhật mỗi cuối tháng. Đây là nơi câu hỏi nghiên cứu của bạn sẽ xuất hiện.

**Xem thêm khi tới lúc:** `06` (khi bắt đầu viết) · `07` (khi bắt đầu chạy thí nghiệm — **đọc trước, không đọc sau**) · `09` (khi muốn xây mạng lưới) · `05` (Phase 4–5).

---

## Cách khởi tạo repo trên GitHub

```bash
cd research-journey
git init
git add .
git commit -m "chore: khởi tạo research journey"
git branch -M main
git remote add origin git@github.com:<username>/research-journey.git
git push -u origin main
```

---

## Nguyên tắc nhắc mình mỗi ngày

- **Question > Solution.** Kỹ sư hỏi "làm thế nào?"; nhà nghiên cứu hỏi "câu hỏi đúng là gì, và tại sao nó quan trọng?".
- **Thất bại là dữ liệu.** Một thí nghiệm không chạy ra kết quả kỳ vọng vẫn là kiến thức.
- **Đọc có hệ thống, không đọc lan man.** Ba lượt (three-pass), ghi chú lại.
- **Viết sớm, viết thường xuyên.** Viết là cách tư duy, không phải bước cuối.
- **Reproduce trước, sáng tạo sau.** Tái tạo được kết quả người khác là chứng chỉ đầu vào của nghiên cứu.
