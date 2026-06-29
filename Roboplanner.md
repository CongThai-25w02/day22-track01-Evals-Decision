# RoboPlanner ↔ 3 case eval — Bản rút gọn liên hệ dự án

> Nhóm **App‑022 · AI20K‑162 "RoboPlanner"**. File này rút **khung chung** từ 3 bài tập thiết kế eval — `01-ticket-triage`, `02-sales-copilot`, `03-medical-routing` — rồi map vào dự án RoboPlanner: **chỗ nào dự án đã làm đúng tinh thần đó, chỗ nào nên bổ sung.** Mục tiêu là biến 3 bài tập rời rạc thành một checklist dùng lại được cho chính sản phẩm của nhóm.

---

## 1. Khung eval chung rút từ 3 case

Cả 3 case lặp lại đúng một quy trình. Đây là "playbook" áp dụng được cho mọi AI task, kể cả RoboPlanner:

1. **4 câu hỏi mở đầu:** cái gì *deterministic* (code chấm)? cái gì *semantic* (LLM/human)? cái gì *high‑risk* cần gate chặt? sai ở đâu cần *escalation* sang người?
2. **Unit of Work** đủ nhỏ — một input → một quyết định, không ôm cả hệ thống.
3. **Quality Question** cụ thể — nếu fail ở đây thì mất gì (trust / an toàn / SLA)?
4. **Output Contract tối thiểu** — nhìn ngược từ UI / routing / safety gate.
5. **Eval Decision Map** — mỗi thành phần giao cho Code / LLM / Human / Expert + lý do.
6. **Code checks** — schema, enum, rule cứng, an toàn.
7. **LLM criteria** — faithfulness, severity, intent, ranh giới: những thứ code không hiểu nghĩa.
8. **Human / Expert review** + checkpoint — case high‑risk thì expert là bắt buộc.
9. **Release gate** — điều kiện chặn cứng + ngưỡng tối thiểu + khi nào cần người.
10. **Pilot plan** — giờ người + chi phí API thật (case y tế tính thêm giờ expert).

---

## 2. Map khung ↔ RoboPlanner (đã có sẵn trong repo)

| Khái niệm trong 3 case | RoboPlanner đã hiện thực | Nguồn trong repo |
|---|---|---|
| Unit of Work | 1 câu lệnh tiếng Việt → di chuyển 1 vật thể tới zone đích | `Coverage & candidate scenarios.md` §1, `PRD.md` §1 |
| Output Contract | `WorldState` + `task` + chuỗi tool‑call hợp lệ + `success` condition | `eval/scenarios/SPEC.md` |
| Code grader (chủ đạo) | oracle `check_object_moved`, `assert_invariants`, `valid_action_rate`, SPL vs A\* | `src/services/oracle.py`, `src/services/invariants.py` |
| Rule cứng / an toàn | guardrails lọc input + kiểm action **trước khi** chạm world; "0 đi xuyên người" | `agent/guardrails.py`, SPEC metric `safety_violations=0` |
| LLM grader (phụ) | chấm `parse_goal` + chất lượng summary/trace cho người đọc | `Coverage & candidate scenarios.md` §1 |
| Human / Expert | mentor review (12 vòng feedback), QA/an toàn đọc trace audit | `REVIEW.md` |
| "Biết nói không" | infeasible/abstention ~100%, `hallucinated_done = 0` | `eval/results/report_v2.md` |
| Human‑in‑loop | F3 "dừng & hỏi" (`ask_human`) khi bất định | `PRD.md` (F3) |
| Tách Code vs LLM | **Bảng A** (agent thật) vs **Bảng B** (solver A\*) — không trộn | `eval/results/report_v2.md` |

→ Nói cách khác, RoboPlanner **đã là một bản hiện thực gần đủ của khung 3 case** — đặc biệt là kỷ luật cốt lõi: *oracle độc lập chấm điểm, không tin agent tự khai "done".*

---

## 3. Mỗi case soi ra điều gì cho RoboPlanner

**Case 1 — deterministic vs semantic + escalation gate.** Đúng cách RoboPlanner tách **rule cứng** (oracle, invariants, schema tool‑call) khỏi **phán đoán semantic** (parse goal tiếng Việt, summary). Bài học: giữ ranh giới này thật rõ — đừng để LLM chấm những gì oracle đã chấm được, và ngược lại.

**Case 2 — lookup → ambiguity → action‑safety.** Map thẳng sang `grounded_action_rate` (chỉ `move/pick/drop` sau khi `perceive/locate`) và `guardrails` (cấm act vượt quyền). "Một tín hiệu khớp nhiều bản ghi" của case 2 tương đương **vật trùng tên / mục tiêu mơ hồ** trong kho → nên có seed eval bắt agent **hỏi lại thay vì chọn bừa** (t17 "mục tiêu mơ hồ" đã chạm, nên bổ sung thêm ca trùng tên).

**Case 3 — high‑risk, expert bắt buộc, red flag, tách "nói/tra/suy luận".** Đây chính là bản thiết kế cho phần **an toàn / human‑in‑loop** mà v2 đã **cắt khỏi scope đo**. Red flag ~ người sát robot / nguy cơ đi xuyên; tách "khách nói / hệ thống tra / AI suy luận" ~ trace `grounded` vs `inferred` mà RoboPlanner đã hiển thị; expert ~ đội an toàn/QA (và nếu lên robot thật thì kỹ sư an toàn ISO 3691‑4). Khi bật lại scope an toàn, dùng đúng checklist case 3: **gate red flag recall, expert duyệt taxonomy, human trong vòng lặp.**

---

## 4. Khoảng trống & việc nên làm tiếp (rút từ 3 case, đối chiếu `REVIEW.md`)

- **Khóa release gate thành gate CI.** SPEC đã có target (`success > 80–90%`, `safety_violations = 0`, `infeasible_correct > 90%`) nhưng `REVIEW.md` ghi nhận các chỉ số năng lực còn dưới mục tiêu (success ~62–70% tùy lần chạy/seed, SPL ~0.40–0.57, grounded < 95%) và CI gốc chưa dựng. → biến target thành **điều kiện chặn merge**, đúng tinh thần "release gate" của cả 3 case.
- **Mở rộng reference dataset.** Cả 3 case đều yêu cầu reference dataset + edge cases đại diện. RoboPlanner `n` còn nhỏ → thêm: ca **ambiguity vật trùng tên** (case 2), ca **robustness tiếng Việt không dấu** (đã có t‑series, giữ làm regression), và bộ **safety / red‑flag** khi bật lại scope (case 3).
- **Pilot costing minh bạch.** 3 case bắt tính giờ người + API thật (case 3 thêm giờ expert). `REVIEW.md` ghi "chưa đo token" → RoboPlanner nên công bố **chi phí token thật** và, nếu bật human‑in‑loop / an toàn, thêm **giờ chuyên gia an toàn** vào dự toán.

---

## 5. Chốt một câu

3 case là **khung**; RoboPlanner là **nơi áp dụng**. Những điểm mạnh nhất của RoboPlanner — oracle độc lập, tách agent/solver, công bố `n` + giới hạn, `hallucinated_done = 0`, biết "nói không" — chính là hiện thực đúng những gì 3 case dạy. Phần còn thiếu — đạt ngưỡng gate, `n` lớn hơn, bật lại an toàn có expert — cũng chính là các bước tiếp theo mà 3 case đã chỉ sẵn.

---

*Nguồn (file trong repo App‑022): `PRD.md`, `README.md`, `eval/scenarios/SPEC.md`, `eval/results/report_v2.md`, `Coverage & candidate scenarios.md`, `REVIEW.md`, `src/services/oracle.py`, `src/services/invariants.py`, `agent/guardrails.py`.*
