# Case 1 - Support Ticket Triage

## Mục tiêu

Case này giúp học viên luyện 4 câu hỏi nên bật ra ngay khi gặp một AI task:

- Cái gì deterministic và nên chấm bằng code?
- Cái gì cần semantic judgment và nên giao cho LLM judge hoặc human?
- Cái gì high-risk nên cần gate chặt hơn?
- Sai ở đâu thì cần escalation sang người thật?

Chỉ cần thiết kế eval ban đầu, không cần code full system.

Case này nối trực tiếp từ track **AI Customer Support Agent** ở Day 18/19, nhưng đổi góc nhìn từ **thiết kế trải nghiệm** sang **thiết kế eval**.

---

## 1. Bối cảnh

Một công ty SaaS B2B dùng AI để đọc ticket support mới và tạo output triage cho hệ thống nội bộ.

Output này không gửi trực tiếp cho khách hàng, nhưng nó được dùng để:

- phân loại ticket,
- đánh dấu mức độ gấp,
- route đến đúng team,
- quyết định có cần người thật nhảy vào hay không.

Nếu AI route sai, ticket có thể bị trễ, bỏ sót escalation, hoặc đẩy sai sang team không xử lý được.

---

## 2. Workflow logic (ASCII)

```text
Khách hàng gửi ticket hỗ trợ
    ↓
AI đọc:
- tiêu đề
- nội dung ticket
- loại khách hàng
    ↓
Hệ thống phải quyết định:
- đây là loại vấn đề gì?
- mức độ khẩn cấp ra sao?
- có cần người thật xử lý ngay không?
- ticket nên vào hàng của team nào?
    ↓
UI inbox nội bộ hiển thị:
- nhãn loại yêu cầu
- mức độ khẩn
- team phụ trách
- cờ "cần xử lý ngay"
- lý do tóm tắt
    ↓
Nếu khách doanh nghiệp + có dấu hiệu chặn công việc
    ↓
Đẩy lên hàng ưu tiên cao / escalation
```

---

## 3. UI hiển thị dự kiến (ASCII)

```text
+----------------------------------------------------------------+
| Hộp thư hỗ trợ nội bộ                                           |
+----------------------------------------------------------------+
| Ticket: T-002                                                   |
| Khách hàng: Công ty ABC (Enterprise)                            |
| Tiêu đề: Thanh toán lỗi, tài khoản bị khóa                      |
|----------------------------------------------------------------|
| AI gợi ý                                                        |
| - Loại yêu cầu: [ ? ]                                           |
| - Mức độ khẩn: [ ? ]                                            |
| - Team phụ trách: [ ? ]                                         |
| - Cần người xử lý ngay: [ ? ]                                   |
| - Lý do tóm tắt: [ .......................................... ] |
|----------------------------------------------------------------|
| Hàng đợi hiện tại: [ Bình thường ] hoặc [ Ưu tiên cao ]         |
+----------------------------------------------------------------+
```

Học viên cần tự đề xuất output contract tối thiểu phía sau để màn hình này hiển thị được.

---

## 4. Input mẫu

```json
{
  "ticket_id": "T-001",
  "subject": "Cannot login after password reset",
  "message": "I reset my password twice but still cannot log in. This is blocking my work.",
  "customer_tier": "enterprise"
}
```

Một input khác:

```json
{
  "ticket_id": "T-002",
  "subject": "URGENT: payment failed and account disabled",
  "message": "Our team is locked out because your billing system failed. Fix this now.",
  "customer_tier": "enterprise"
}
```

---

## 5. Business rules / operational rules

- Output phải đúng schema và đúng allowed enums.
- `confidence` phải nằm trong khoảng `0-1`.
- Nếu `customer_tier = enterprise` và `urgency` là `high` hoặc `critical`, `requires_human` phải bằng `true`.
- Ticket billing không được route sang `product_team`.
- Ticket có dấu hiệu “blocking work”, “locked out”, hoặc “account disabled” không nên bị đánh `low`.
- `reason_codes` phải phản ánh được nội dung ticket, không được bốc thêm sự thật không có trong input.

---

## 6. Ví dụ full luồng để hình dung nhanh

### Tình huống

Khách hàng doanh nghiệp nhắn vào kênh hỗ trợ:

```text
Chị ơi bên em reset mật khẩu 2 lần rồi mà tài khoản admin vẫn không vào được.
Bên em đang bị chặn công việc từ sáng.
```

### Data mẫu

- `customer_tier`: `enterprise`
- `account_name`: `Công ty Minh Phát Logistics`
- `previous_tickets_7d`: `0`
- `channel`: `Zalo OA`

### Workflow ASCII

```text
Khách nhắn vấn đề đăng nhập
    ↓
AI đọc nội dung + loại khách hàng
    ↓
AI phát hiện tín hiệu:
- login issue
- blocked work
- enterprise customer
    ↓
Hệ thống gợi ý:
- category = technical
- urgency = high hoặc critical
- requires_human = true
- route_to = technical_support
    ↓
UI nội bộ đẩy ticket lên hàng ưu tiên
    ↓
Nhân viên hỗ trợ xem lại rồi tiếp nhận
```

### UI trước khi AI xử lý (ASCII)

```text
+----------------------------------------------------------------+
| Hộp thư hỗ trợ nội bộ                                           |
+----------------------------------------------------------------+
| Ticket: T-115                                                   |
| Kênh: Zalo OA                                                   |
| Khách hàng: Minh Phát Logistics                                 |
|----------------------------------------------------------------|
| Nội dung khách nhắn:                                            |
| "Reset mật khẩu 2 lần rồi mà tài khoản admin vẫn không vào..."  |
|----------------------------------------------------------------|
| AI gợi ý: Chưa có                                               |
+----------------------------------------------------------------+
```

### UI sau khi AI xử lý (ASCII)

```text
+----------------------------------------------------------------+
| Hộp thư hỗ trợ nội bộ                                           |
+----------------------------------------------------------------+
| Ticket: T-115                                                   |
| Khách hàng: Minh Phát Logistics (Enterprise)                    |
|----------------------------------------------------------------|
| AI gợi ý                                                        |
| - Loại yêu cầu: Technical                                       |
| - Mức độ khẩn: High                                             |
| - Team phụ trách: Technical Support                             |
| - Cần người xử lý ngay: Có                                      |
| - Lý do tóm tắt: Lỗi đăng nhập đang chặn công việc              |
| - Hàng đợi: Ưu tiên cao                                         |
+----------------------------------------------------------------+
```

Ví dụ này giúp người đọc hình dung ngay:

- AI đang quyết định gì,
- quyết định nào hiển thị ra UI,
- và sai ở đâu thì ảnh hưởng vận hành.

---

## 7. Seed cases

Đây không phải full dataset. Đây chỉ là 3 seed cases để học viên hình dung phạm vi và failure modes.

### Seed A - Happy path

- `subject`: `Cannot login after password reset`
- Kỳ vọng: `category = technical`, `requires_human = true` nếu urgency đủ cao, route về `technical_support`

### Seed B - Ambiguous / low-info

- `subject`: `Help`
- `message`: `Please help asap`
- Kỳ vọng: AI không nên tự tin gán category quá mạnh; cần `unknown` hoặc route theo hướng cần review

### Seed C - High-risk / escalation

- `subject`: `URGENT: payment failed and account disabled`
- Kỳ vọng: `category = billing`, `urgency = critical`, `requires_human = true`, route về `billing_ops` hoặc `human_escalation`

---

## 8. Bạn phải đề xuất thêm 5 Dataset Edge Cases

Sau khi đọc seed cases ở trên, hãy đề xuất thêm 5 case cần đưa vào reference dataset version đầu.

Không cần nộp một bảng coverage riêng. Hãy chọn 5 case đại diện cho các lát cắt khác nhau, ví dụ: match rõ, thiếu tín hiệu, ambiguity, escalation, và regression.

1. Happy path: `"Cannot login after password reset"` (enterprise, đang chặn việc) → category=technical, urgency=high, requires_human=true, route=technical_support. — Bắt: luồng chuẩn chạy đúng, không under-triage tín hiệu blocking.
2. Ambiguous input: subject `"Help"`, message `"cần hỗ trợ gấp"`, không nêu vấn đề. — Bắt: AI gán category quá tự tin thay vì trả `unknown` + confidence thấp.
3. Missing information: ticket chỉ có mã đơn, không mô tả lỗi. — Bắt: AI đoán team thay vì đánh dấu cần bổ sung thông tin.
4. High-risk / escalation: `"payment failed and account disabled"` (enterprise). — Bắt: bỏ sót escalation — phải urgency=critical, requires_human=true, route billing_ops/human_escalation, KHÔNG product_team.
5. Regression case: ticket free-tier ghi `"blocking work"` nhưng nhẹ → kiểm rule "blocking ⇒ không low" vẫn đúng sau khi sửa prompt. — Bắt: tái phát lỗi under-triage cũ mỗi lần đổi prompt.

Với mỗi case, thêm 1 dòng ngắn giải thích:

- case này dùng để bắt failure gì?

---

## 9. Mock outcome để soi

Giả sử trên UI nội bộ, hệ thống hiển thị kết quả gợi ý như sau cho `T-002`:

```text
+----------------------------------------------------------------+
| Ticket: T-002                                                   |
| Khách hàng: Enterprise                                          |
|----------------------------------------------------------------|
| AI gợi ý                                                        |
| - Loại yêu cầu: Product question                                |
| - Mức độ khẩn: Medium                                           |
| - Team phụ trách: Support L1                                    |
| - Cần người xử lý ngay: Không                                   |
| - Lý do tóm tắt: Có vấn đề thanh toán                           |
| - Độ tin cậy: 0.91                                              |
+----------------------------------------------------------------+
```

Kết quả này trông có vẻ “ổn” nếu chỉ nhìn bề mặt, nhưng khả năng cao là sai về judgment vận hành.

---

## 10. Nhiệm vụ học viên

Hãy điền workbook bên dưới cho case này.

Không cần:

- viết eval runner,
- viết prompt judge thật,
- làm lại `User Input Grid` đầy đủ như bài test inputs hôm trước,
- tạo full dataset lớn,
- code full system.

Cần làm:

- chọn đúng nguồn chấm cho từng thành phần,
- viết các rule kiểm tra đủ cụ thể để có thể implement sau,
- đặt release gate có ý nghĩa vận hành,
- đề xuất 5 edge cases cần đưa vào reference dataset,
- và lập một pilot plan có thời gian + chi phí sơ bộ.

---

## 11. Bạn nên làm gì ở case 1?

Đây là case scaffold cao, nên cách làm tốt nhất là:

1. Đọc ví dụ full luồng trước để hiểu “một output tốt trông như thế nào”.
2. So mock outcome với ví dụ full luồng để thấy lỗi đang nằm ở đâu.
3. Nhìn từ UI để suy ra các field tối thiểu hệ thống phải có.
4. Điền `Eval Decision Map` trước, rồi mới quay lại viết các kiểm tra tự động và gate.

Case này thường **không bắt buộc phải có domain expert chuyên sâu**. Nếu chọn không cần expert, bạn vẫn phải giải thích vì sao human review vận hành là đủ.

---

## 12. Workbook

Lưu ý chung cho toàn bộ câu trả lời:

- Không chỉ điền đáp án ngắn.
- Với mỗi phần, hãy nêu cả **quyết định** và **lý do**.
- Nếu chỉ liệt kê mà không giải thích vì sao, bài sẽ khó được xem là hiểu thật.

### 1. Unit of Work

- AI đang thực hiện công việc gì?
- Output cuối cùng được dùng bởi ai?
- Nếu sai, hậu quả vận hành là gì?

Gợi ý từ bài hôm trước:

- Đừng chọn “toàn bộ hệ thống hỗ trợ khách hàng”.
- Ở case này, một `Unit of Work` tốt thường là: **một ticket đi vào -> AI gán nhãn, đánh mức ưu tiên, đề xuất route và cờ escalation**.

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- bạn chọn lát cắt nào,
- và vì sao đây là đơn vị đủ nhỏ để eval.

Lát cắt chọn: **một ticket đi vào → AI gán {category, urgency, route_to, requires_human, reason} cho riêng ticket đó**. Output dùng bởi hệ thống inbox nội bộ và nhân viên hỗ trợ (không gửi thẳng cho khách). Nếu sai: ticket bị trễ, bỏ sót escalation, hoặc đẩy sang team không xử lý được — với khách enterprise đang bị chặn việc thì mất trust và vỡ SLA. Đây là đơn vị đủ nhỏ vì chỉ chấm **một quyết định phân loại + route cho một ticket**, không ôm cả hệ thống hỗ trợ.

### 2. Quality Question

Viết một câu hỏi chất lượng đủ cụ thể cho lát cắt này.

Gợi ý:

- Đừng hỏi kiểu quá rộng như: “AI có triage tốt không?”
- Nếu AI làm sai ở đây, điều gì sẽ khiến khách hàng mất trust hoặc không hoàn thành mục tiêu?
- Behavior nào là bắt buộc?
- Behavior nào là bị cấm?
- Viết theo dạng: **AI có gắn đúng route và escalation để ticket không bị đi sai hàng xử lý không?**

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- câu hỏi chất lượng bạn chọn,
- và vì sao nếu fail ở đây thì ticket sẽ đi sai hoặc gây mất trust.

Câu hỏi chất lượng: **"AI có gán đúng category + urgency + route_to và bật requires_human đúng lúc để ticket không đi sai hàng và không bỏ sót escalation không?"** Nếu fail ở đây, một ticket enterprise đang bị chặn việc có thể bị đánh `low` hoặc route nhầm `product_team` → xử lý trễ, khách mất trust, vỡ SLA. Behavior bắt buộc: enterprise + high/critical ⇒ `requires_human=true`. Behavior bị cấm: route billing sang `product_team`, hoặc đánh `low` khi có dấu hiệu blocking/locked out.

### 3. Output Contract tối thiểu

Không cần đoán full JSON hoàn chỉnh. Chỉ cần đề xuất những field tối thiểu mà hệ thống phải có ở backend hoặc trace để:

- render UI ở trên,
- route đúng hàng xử lý,
- trigger escalation nếu cần,
- và chạy eval sau này.

Mẹo lấy từ ví dụ full luồng:

- Hãy nhìn ngược từ UI và mock outcome.
- Field nào không làm thay đổi màn hình, routing hoặc gate thì chưa cần đưa vào.

**Trả lời của bạn:**

Đừng chỉ liệt kê field. Với mỗi field bạn giữ lại, hãy giải thích ngắn vì sao nó cần cho UI, routing, escalation, hoặc eval.

Các field tối thiểu (nhìn ngược từ UI + routing + gate):
- `ticket_id` — nối kết quả về đúng ticket để render và audit.
- `category` (enum) — quyết định route + nhãn hiển thị.
- `urgency` (enum) — xếp hàng ưu tiên + điều kiện escalation.
- `route_to` (enum) — đẩy đúng team; sai là hỏng vận hành.
- `requires_human` (bool) — trigger escalation / human-in-loop.
- `confidence` (0–1) — hạ tự tin thì đẩy review; phục vụ release gate.
- `reason_summary` (text ngắn) — để người duyệt hiểu vì sao, và để chấm "có bịa không".
- `reason_codes` (enum[]) — để code kiểm lý do có bám tín hiệu thật trong input.

Field nào không làm đổi UI, routing, escalation hay eval thì chưa cần đưa vào.

### 4. Eval Decision Map

Ở phần này, bạn phải **tự quyết định** đâu là các thành phần thật sự cần chấm.

Đừng chép lại toàn bộ business rules hay toàn bộ UI. Hãy chọn ra những thành phần quan trọng nhất, bám vào:

- `Output Contract` bạn đã đề xuất
- quyết định nào thật sự làm thay đổi route, escalation, hoặc safety
- chỗ nào nếu sai sẽ gây hậu quả vận hành rõ ràng

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
| --- | ---: | ---: | ---: | ---: | --- |
| Schema + enum hợp lệ (category/urgency/route/reason_codes) | x |  |  |  | Nhị phân, có schema rõ; sai là vỡ downstream nên để code chặn. |
| `confidence` ∈ [0,1] | x |  |  |  | Kiểm khoảng số thuần, không cần hiểu nghĩa. |
| Rule cứng: enterprise + high/critical ⇒ requires_human=true | x |  |  |  | Bất biến vận hành, phải luôn đúng; code kiểm tuyệt đối. |
| Billing không route product_team; dấu hiệu 'blocking/locked out' ⇒ không `low` | x |  |  |  | Rule route/ưu tiên cứng, deterministic. |
| `category` đúng bản chất ticket | | x | x |  | Cần đọc hiểu nội dung; ca biên để human chốt. |
| `urgency` tương xứng mức ảnh hưởng | | x | x |  | Mức 'khẩn' là phán đoán semantic + vận hành. |
| `reason_summary` trung thực, không bịa | | x |  |  | So khớp ý với input — semantic faithfulness. |

Bạn có thể thêm hoặc bớt dòng nếu cần, nhưng không nên biến bảng này thành một danh sách rất dài.

Không chấp nhận bảng chỉ tick `Yes/No`. Cột `Lý do` phải nêu được vì sao bạn chọn nguồn chấm đó.

### 5. Kiểm tra tự động bằng code

Liệt kê **đầy đủ** các rule kiểm tra tự động mà bạn cho rằng case này cần có.

Không giới hạn số lượng. Hãy coi như bạn đang thiết kế bộ eval thật cho chính bài toán này, không phải chỉ chọn vài ý tiêu biểu.

Ưu tiên các kiểm tra mà nếu fail thì ticket sẽ đi sai hàng, thiếu escalation, hoặc vỡ schema.

Mỗi ý nên viết theo dạng:

- Kiểm tra: [rule]
  Vì sao nên giao cho code:

- Kiểm tra: output đúng JSON schema và mọi field enum nằm trong allowed set.
  Vì sao nên giao cho code: nhị phân, rẻ, chặn vỡ hệ thống phía sau.
- Kiểm tra: `confidence` ∈ [0,1].
  Vì sao nên giao cho code: range check thuần số.
- Kiểm tra: NẾU customer_tier=enterprise VÀ urgency ∈ {high,critical} THÌ requires_human=true.
  Vì sao nên giao cho code: bất biến logic, không được phụ thuộc phán đoán mô hình.
- Kiểm tra: category=billing ⇒ route_to ≠ product_team.
  Vì sao nên giao cho code: rule routing cứng.
- Kiểm tra: message chứa {blocking, locked out, account disabled} ⇒ urgency ≠ low.
  Vì sao nên giao cho code: rule từ khoá deterministic, chống under-triage.
- Kiểm tra: route_to ∈ taxonomy hợp lệ; reason_codes ⊆ enum và không rỗng khi requires_human=true.
  Vì sao nên giao cho code: schema + điều kiện, code kiểm chắc chắn.
- Kiểm tra: mỗi reason_code map tới ≥1 tín hiệu có trong input (chống bịa).
  Vì sao nên giao cho code: kiểm tập hợp cơ học, không cần hiểu nghĩa.

### 6. Tiêu chí chấm bằng LLM

Liệt kê **đầy đủ** các tiêu chí semantic mà case này cần có và code không chấm tốt.

Không giới hạn số lượng. Hãy coi như đây là bộ tiêu chí bạn thật sự sẽ dùng để chấm case này.

Chỉ giữ những tiêu chí mà cần đọc hiểu nghĩa của ticket hoặc mức độ hợp lý của lý do tóm tắt.

Mỗi ý nên viết theo dạng:

- Tiêu chí: [criterion]
  Vì sao code không bắt tốt:

- Tiêu chí: category có phản ánh đúng vấn đề thật của ticket không (login vs billing vs product).
  Vì sao code không bắt tốt: cần đọc hiểu nghĩa câu chữ.
- Tiêu chí: urgency có tương xứng mức ảnh hưởng mô tả trong ticket không (chặn cả team vs hỏi nhẹ).
  Vì sao code không bắt tốt: mức khẩn là phán đoán ngữ cảnh.
- Tiêu chí: reason_summary có trung thực với nội dung, không thêm sự thật mới không.
  Vì sao code không bắt tốt: faithfulness ngữ nghĩa.
- Tiêu chí: với ticket mơ hồ/thiếu tin, AI có hạ tự tin / chọn `unknown` thay vì đoán mạnh không.
  Vì sao code không bắt tốt: cần đánh giá đủ thông tin hay chưa.

### 7. Human / Expert Review

- Ai cần review?
- Review những case nào?
- Có cần domain expert không? Nếu không, vì sao?

**Trả lời của bạn:**

Đừng chỉ ghi tên team review. Hãy giải thích vì sao đúng nhóm người đó cần xem, và failure nào cần họ xem.

Người review: **ops lead / team hỗ trợ** — soi các ca AI bật `requires_human=true`, các ca `confidence` thấp, và một mẫu ngẫu nhiên để bắt drift. Họ kiểm route và mức khẩn có khớp thực tế vận hành không — vì chính họ chịu hậu quả SLA nên là "chuẩn vàng" cho case này. **Không cần domain expert chuyên sâu**: triage support là phán đoán vận hành, không cần chuyên môn ngành; human review từ team ops đủ để bắt sai route / sót escalation.

Nếu chọn **có domain expert**, bạn phải làm thêm 2 phần dưới đây. Nếu **không cần domain expert**, hãy ghi `Không áp dụng` và giải thích 1 câu.

#### 7A. Màn hình cho Domain Expert (ASCII)

Mock một màn hình review cho expert.

Expert cần thấy tối thiểu:

- AI đã route hoặc gắn nhãn gì,
- dấu hiệu hoặc evidence nào khiến case bị đẩy sang expert,
- expert có thể duyệt / sửa / escalation ở đâu.

**Trả lời của bạn:**

Không áp dụng — case này không dùng domain expert (lý do ở mục 7: triage là phán đoán vận hành, team ops review là đủ).

#### 7B. Tiêu chí review của Domain Expert

Không áp dụng — case này không dùng domain expert (xem mục 7).

### 8. Release Gate

Đề xuất release gate phù hợp cho case này. Nêu rõ điều kiện chặn, ngưỡng chất lượng tối thiểu, và trường hợp cần human review.

**Release gate đề xuất:**
- **Chặn cứng (bất kỳ vi phạm = không release):** vi phạm rule cứng > 0 (enterprise+high⇒human; billing không sang product; blocking⇒không low); invalid schema > 0%; false-`low` trên ca high-risk > 0.
- **Ngưỡng chất lượng tối thiểu:** đúng route + urgency (LLM + human chấm) ≥ 90% trên reference set nhóm feasible; reason_summary faithfulness ≥ 95%.
- **Bắt buộc human review trước khi vào hàng:** mọi ca `confidence` < 0.6 hoặc `category=unknown`, và mọi ca `requires_human=true`.

### 9. Kế hoạch chạy thử và dự toán chi phí

Làm phần này với giả định team của bạn vừa nhận đề bài triage này từ công ty.

Bạn là PM phụ trách đề xuất cách xây bộ eval, cách chạy thử, và chi phí cần xin để trả lời câu hỏi:

- hướng làm này hiện chính xác tới đâu
- cần thêm những checkpoint nào trước khi đề xuất triển khai tiếp
- và với một khoản budget thử nghiệm nhỏ, team có thể chứng minh được gì

README của folder này chỉ cho khung tính. Hãy giữ cách tính đơn giản: phần người tính bằng `giờ công`, phần máy tính bằng `chi phí API key` tính từ **giá thật** của model / dịch vụ bạn chọn.

Để làm phần này, bạn cần tự tính và nêu rõ:

- giá API thật bạn dùng để tính
- tổng số cases pilot dự kiến
- tổng số lần chạy / lặp lại dự kiến
- tổng giờ PM / thiết kế eval
- tổng giờ vận hành / kỹ thuật
- tổng giờ human review
- nếu có `domain expert`, tổng số giờ expert
- tổng chi phí API key
- tổng chi phí pilot
- tổng thời gian dự kiến

Có thể lấy mốc tham khảo để nhẩm nhanh:

- khoảng `50-100 cases`
- khoảng `30-50 lần chạy / lặp lại`

Không cần trình bày thành bảng. Hãy tự chọn cách trình bày miễn là người đọc nhìn vào hiểu được bạn đã tính gì và chi phí tổng rơi vào đâu.

Sau phần này, viết thêm 2-4 câu ngắn:

- bạn dùng giá API thật từ đâu để tính,
- với quy mô này chi phí tổng rơi vào khoảng nào,
- và vì sao plan này đủ để chứng minh case có thể pilot được.

**Kế hoạch chạy thử + dự toán (gọn):**
- **Giá API thật:** Claude Haiku 4.5 — **$1.00 / 1M token input, $5.00 / 1M token output** (bảng giá Anthropic, 2026). Mỗi ticket ~1.5K token in + 0.3K token out.
- **Quy mô:** ~80 cases × ~40 lần chạy/lặp = **3,200 lần gọi** → ~5–6M token in + ~1M token out → **chi phí API ≈ $6–10**.
- **Giờ người:** PM/thiết kế eval ~16h; kỹ thuật dựng harness + rule ~20h; human review (ops) ~12h. Không cần expert.
- **Tổng pilot:** ~**48 giờ công + ~$10 API**, thời gian ~**1–1.5 tuần**.

Giá lấy từ trang pricing chính thức của Anthropic; ở quy mô này chi phí gần như chỉ là **giờ người (~48h)**, API gần như không đáng kể. Plan đủ để đo độ chính xác route/escalation và quyết định có mở rộng hay không.

---
