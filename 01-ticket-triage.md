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

# Bài làm - Case 1: Support Ticket Triage

## 1. Unit of Work

unit of work là: một ticket support mới đi vào hệ thống, AI đọc nội dung ticket và metadata khách hàng, sau đó gán `category`, `urgency`, `route_to`, `requires_human` và lý do tóm tắt để nhân viên nội bộ xử lý. Đây là lát cắt đủ nhỏ vì nó chỉ đo một quyết định triage cho một ticket, nhưng vẫn chạm đúng rủi ro vận hành: route sai, bỏ sót escalation, hoặc đẩy ticket enterprise vào hàng thường. Output cuối cùng được dùng bởi support agent, support lead và hệ thống queue nội bộ. Nếu sai, khách có thể bị chậm xử lý, ticket đi sai team, hoặc case khẩn không được người thật tiếp nhận.

## 2. Quality Question

Câu hỏi chất lượng chính là: AI có gắn đúng loại vấn đề, mức độ khẩn, team phụ trách và cờ escalation để ticket không bị đi sai hàng xử lý không? Behavior bắt buộc là phải escalate khi khách enterprise gặp lỗi chặn công việc, locked out, account disabled hoặc billing critical. Behavior bị cấm là tự tin route sang team không xử lý được, hạ thấp mức khẩn của case có tín hiệu nghiêm trọng, hoặc tạo reason code không có trong input.

Nếu fail ở đây, trust của đội support sẽ giảm rất nhanh vì AI không chỉ "label sai" mà còn làm thay đổi thứ tự ưu tiên xử lý thật. Với khách enterprise, một ticket bị đánh medium thay vì high/critical có thể tạo hậu quả SLA và churn.

## 3. Output Contract tối thiểu

- `ticket_id`: cần để trace output về đúng ticket, render UI, và debug regression.
- `category`: enum tối thiểu gồm `technical`, `billing`, `product_question`, `account_access`, `feature_request`, `unknown`. Field này quyết định nhãn UI và hỗ trợ routing.
- `urgency`: enum `low`, `medium`, `high`, `critical`. Field này quyết định queue thường hay ưu tiên cao.
- `route_to`: enum `support_l1`, `technical_support`, `billing_ops`, `product_team`, `human_escalation`. Đây là quyết định vận hành quan trọng nhất vì nó đưa ticket vào team xử lý.
- `requires_human`: boolean để trigger escalation và tránh AI khiến ticket khẩn nằm lại automation queue.
- `confidence`: số từ `0` đến `1` để quyết định khi nào cần human review, đặc biệt với input thiếu thông tin.
- `reason_summary`: câu ngắn hiển thị trên UI để nhân viên hiểu vì sao AI route như vậy.
- `reason_codes`: list enum như `blocking_work`, `locked_out`, `payment_failed`, `account_disabled`, `enterprise_customer`, `low_information`, `ambiguous_category`. Field này giúp eval xem AI có dựa vào evidence thật hay không.
- `evidence_spans`: trích đoạn ngắn từ subject/message như "blocking my work" hoặc "account disabled". Field này giúp human review nhanh và chống hallucination.

## 4. Eval Decision Map

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
| --- | ---: | ---: | ---: | ---: | --- |
| Schema, required fields, enum hợp lệ | Có | Không | Không | Không | Đây là invariant kỹ thuật, code chấm nhanh và ổn định nhất. |
| `confidence` nằm trong `0-1` | Có | Không | Không | Không | Rule số học deterministic, không cần judgment. |
| Enterprise + high/critical phải `requires_human=true` | Có | Không | Có | Không | Code bắt hard rule; human review sample để kiểm tra rule có phản ánh vận hành thật. |
| Billing không route sang `product_team` | Có | Không | Không | Không | Đây là business rule rõ ràng, code assertion đủ tốt. |
| Category/urgency có hợp nghĩa với nội dung ticket | Không | Có | Có | Không | Cần đọc hiểu ngữ nghĩa; LLM judge có thể scale, human dùng để calibrate. |
| `reason_summary` grounded vào input | Không | Có | Có | Không | Code khó biết reason có bịa hay không; LLM và human đọc evidence tốt hơn. |
| Case ambiguous có biết hạ confidence hoặc route review không | Một phần | Có | Có | Không | Code bắt ngưỡng confidence, LLM/human đánh giá mức ambiguity. |
| Release gate cho case enterprise escalation | Có | Có | Có | Không | Đây là rủi ro vận hành cao nên cần kết hợp rule, semantic eval và review thật. |

Case này không cần domain expert chuyên sâu vì taxonomy là nghiệp vụ support/SaaS, không phải medical/legal/financial advice. Support lead và ops lead đủ quyền xác nhận route, priority và escalation.

## 5. Kiểm tra tự động bằng code

- Kiểm tra: output là JSON hợp lệ, có đủ `ticket_id`, `category`, `urgency`, `route_to`, `requires_human`, `confidence`, `reason_summary`, `reason_codes`.
  Vì sao nên giao cho code: schema là deterministic, fail là không render được UI hoặc không route được.

- Kiểm tra: mọi enum thuộc allowed values.
  Vì sao nên giao cho code: enum sai sẽ làm hỏng routing hoặc dashboard.

- Kiểm tra: `confidence >= 0` và `confidence <= 1`.
  Vì sao nên giao cho code: đây là numeric invariant.

- Kiểm tra: nếu `customer_tier=enterprise` và `urgency` là `high` hoặc `critical` thì `requires_human=true`.
  Vì sao nên giao cho code: rule đã được business định nghĩa rõ.

- Kiểm tra: nếu subject/message chứa `locked out`, `account disabled`, `blocking my work` hoặc biến thể đã chuẩn hóa thì `urgency` không được là `low`.
  Vì sao nên giao cho code: đây là safety hard rule cho tín hiệu rõ ràng.

- Kiểm tra: nếu `category=billing` thì `route_to` không được là `product_team`.
  Vì sao nên giao cho code: route billing sang product là lỗi rule rõ ràng.

- Kiểm tra: nếu `requires_human=true` thì `route_to` phải là `human_escalation`, `billing_ops`, hoặc team có queue ưu tiên.
  Vì sao nên giao cho code: tránh cờ escalation có mà route vẫn vào hàng thường.

- Kiểm tra: nếu `category=unknown` hoặc `confidence < 0.6` thì `requires_human=true` hoặc `route_to=support_l1` với flag review.
  Vì sao nên giao cho code: ambiguity cần vào review, không để AI tự quyết quá mạnh.

- Kiểm tra: `reason_codes` không được chứa code không có trong taxonomy.
  Vì sao nên giao cho code: taxonomy ổn định và kiểm được bằng set membership.

- Kiểm tra: `reason_summary` không được rỗng, quá ngắn, hoặc dài quá ngưỡng UI.
  Vì sao nên giao cho code: đây là constraint hiển thị và trace.

- Kiểm tra: `evidence_spans` nếu có phải là substring hoặc gần-substring của input.
  Vì sao nên giao cho code: giúp bắt hallucination evidence ở mức cơ bản.

- Kiểm tra: regression cases T-002 billing/account disabled luôn phải `category=billing`, `urgency=critical`, `requires_human=true`.
  Vì sao nên giao cho code: đây là case từng được xác định high-risk nên phải khóa trong CI.

## 6. Tiêu chí chấm bằng LLM

- Tiêu chí: category có phản ánh đúng vấn đề chính của ticket không.
  Vì sao code không bắt tốt: cùng một lỗi có thể diễn đạt bằng nhiều cách, keyword đơn giản dễ miss hoặc match nhầm.

- Tiêu chí: urgency có hợp lý với tác động kinh doanh và ngôn ngữ khách dùng không.
  Vì sao code không bắt tốt: "blocking work" rõ, nhưng nhiều câu như "team cannot proceed" cần semantic judgment.

- Tiêu chí: route_to có đưa ticket tới team có khả năng xử lý nguyên nhân gốc không.
  Vì sao code không bắt tốt: route phụ thuộc ý nghĩa nội dung, không chỉ enum.

- Tiêu chí: reason_summary có đúng, đủ, không phóng đại và không bỏ sót chi tiết làm thay đổi priority không.
  Vì sao code không bắt tốt: cần hiểu nội dung ticket và mức nghiêm trọng.

- Tiêu chí: reason_codes có grounded vào input, không bịa thêm sự thật như "SLA breach" khi input không nói.
  Vì sao code không bắt tốt: cần so sánh ý nghĩa giữa input và output.

- Tiêu chí: với input thiếu thông tin, AI có thừa nhận không chắc chắn thay vì tự tin gán category mạnh không.
  Vì sao code không bắt tốt: ambiguity là judgment ngữ nghĩa.

## 7. Human / Expert Review

Người review chính là support lead hoặc ops lead vì họ hiểu queue, SLA, team ownership và hậu quả của route sai. Họ cần review toàn bộ case high-risk trong pilot, toàn bộ case confidence thấp, case customer enterprise, và sample ngẫu nhiên của case thường để calibrate LLM judge. Failure họ kiểm tra là: ticket có bị đi sai team không, escalation có đủ nhanh không, và reason có giúp agent xử lý tốt hơn không.

Không cần domain expert chuyên sâu. Đây là domain support operations, nên human review từ support lead là đủ để xác nhận taxonomy và release gate.

### 7A. Màn hình cho Domain Expert (ASCII)

Không áp dụng. Case này không cần domain expert vì quyết định thuộc nghiệp vụ support nội bộ, không phải chuyên môn high-stakes như y tế hoặc pháp lý.

### 7B. Tiêu chí review của Domain Expert

Không áp dụng.

## 8. Release Gate

Chặn release nếu bất kỳ hard rule nào fail: schema/enum sai, confidence ngoài `0-1`, enterprise high/critical không `requires_human=true`, billing route sang `product_team`, hoặc case có locked out/account disabled bị đánh `low`. Với reference dataset v0, yêu cầu tối thiểu: 100% pass code checks, ít nhất 90% đúng route trên toàn bộ dataset, ít nhất 95% đúng escalation cho enterprise/high-risk, và 0 lỗi bỏ sót critical escalation trong seed + regression cases.

Các output có `confidence < 0.6`, `category=unknown`, hoặc LLM judge chấm route không chắc chắn phải vào human review. Chỉ nên pilot nội bộ khi support lead duyệt ít nhất 20 case high-risk và không thấy failure nghiêm trọng lặp lại.

## 9. 5 Dataset Edge Cases

1. Happy path: Enterprise báo "admin cannot login and finance team blocked since morning"; kỳ vọng technical/account access, high, human required. Case này bắt failure hạ thấp urgency dù có blocked work.
2. Ambiguous input: Subject "Need help", message "This is urgent please call me"; kỳ vọng unknown hoặc support_l1 review, confidence thấp. Case này bắt failure tự bịa category.
3. Missing information: Khách SMB nói "Payment issue" nhưng không nói failed/disabled; kỳ vọng billing, medium hoặc review, không critical nếu thiếu evidence. Case này bắt failure over-escalation.
4. High-risk / escalation: Enterprise nói "payment failed and account disabled for all users"; kỳ vọng billing, critical, human required, billing_ops/human_escalation. Case này bắt failure giống mock outcome T-002.
5. Regression case: Ticket billing có từ "invoice question" nhưng message nói "account locked after failed payment"; kỳ vọng không route product_team và không đánh low. Case này bắt regression keyword "question" làm route sai.

## 10. Kế hoạch chạy thử và dự toán chi phí

Tôi đề xuất pilot 80 cases, gồm 40 case thường, 20 ambiguous/missing-info, 15 enterprise/high-risk, và 5 regression bắt buộc. Chạy 40 lần lặp trong giai đoạn prompt/model tuning, mỗi lần chạy trung bình 80 cases. Giả định mỗi case dùng khoảng 1,500 input tokens và 300 output tokens cho model chính; tổng 3,200 runs tương đương 4.8M input tokens và 0.96M output tokens.

Giá API dùng để tính: OpenAI API pricing chính thức, model `gpt-4.1-mini` là `$0.40 / 1M input tokens` và `$1.60 / 1M output tokens`. Chi phí API ước tính: input `4.8 * 0.40 = $1.92`, output `0.96 * 1.60 = $1.54`, tổng khoảng `$3.46`; cộng thêm LLM judge/overhead tôi làm tròn thành `$20`.

Giờ công dự kiến: PM/eval design 8 giờ, kỹ thuật/ops setup 8 giờ, support lead review 6 giờ, human reviewer 8 giờ. Giả định rate nội bộ để xin budget thử nghiệm: PM `$20/h`, kỹ thuật/ops `$15/h`, support lead/human review `$8/h`. Chi phí người khoảng `$160 + $120 + $48 + $64 = $392`. Tổng pilot khoảng `$420` sau khi cộng API và buffer nhỏ. Thời gian dự kiến 4-5 ngày làm việc: 1 ngày chốt taxonomy/output contract, 1 ngày build reference set, 1-2 ngày chạy/tune, 1 ngày review gate.

Với quy mô này, team chứng minh được hướng triage có đủ chính xác để pilot nội bộ hay không, đặc biệt ở route và escalation. Chi phí API rất nhỏ so với chi phí review; khoản đáng xin budget chủ yếu là thời gian của support lead để xác nhận quality gate.

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

1. Happy path:
2. Ambiguous input:
3. Missing information:
4. High-risk / escalation:
5. Regression case:

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

> ...

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

> ...

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

> ...

### 4. Eval Decision Map

Ở phần này, bạn phải **tự quyết định** đâu là các thành phần thật sự cần chấm.

Đừng chép lại toàn bộ business rules hay toàn bộ UI. Hãy chọn ra những thành phần quan trọng nhất, bám vào:

- `Output Contract` bạn đã đề xuất
- quyết định nào thật sự làm thay đổi route, escalation, hoặc safety
- chỗ nào nếu sai sẽ gây hậu quả vận hành rõ ràng

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
| --- | ---: | ---: | ---: | ---: | --- |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
|  |  |  |  |  |  |

Bạn có thể thêm hoặc bớt dòng nếu cần, nhưng không nên biến bảng này thành một danh sách rất dài.

Không chấp nhận bảng chỉ tick `Yes/No`. Cột `Lý do` phải nêu được vì sao bạn chọn nguồn chấm đó.

### 5. Kiểm tra tự động bằng code

Liệt kê **đầy đủ** các rule kiểm tra tự động mà bạn cho rằng case này cần có.

Không giới hạn số lượng. Hãy coi như bạn đang thiết kế bộ eval thật cho chính bài toán này, không phải chỉ chọn vài ý tiêu biểu.

Ưu tiên các kiểm tra mà nếu fail thì ticket sẽ đi sai hàng, thiếu escalation, hoặc vỡ schema.

Mỗi ý nên viết theo dạng:

- Kiểm tra: [rule]
  Vì sao nên giao cho code:

### 6. Tiêu chí chấm bằng LLM

Liệt kê **đầy đủ** các tiêu chí semantic mà case này cần có và code không chấm tốt.

Không giới hạn số lượng. Hãy coi như đây là bộ tiêu chí bạn thật sự sẽ dùng để chấm case này.

Chỉ giữ những tiêu chí mà cần đọc hiểu nghĩa của ticket hoặc mức độ hợp lý của lý do tóm tắt.

Mỗi ý nên viết theo dạng:

- Tiêu chí: [criterion]
  Vì sao code không bắt tốt:

### 7. Human / Expert Review

- Ai cần review?
- Review những case nào?
- Có cần domain expert không? Nếu không, vì sao?

**Trả lời của bạn:**

Đừng chỉ ghi tên team review. Hãy giải thích vì sao đúng nhóm người đó cần xem, và failure nào cần họ xem.

> ...

Nếu chọn **có domain expert**, bạn phải làm thêm 2 phần dưới đây. Nếu **không cần domain expert**, hãy ghi `Không áp dụng` và giải thích 1 câu.

#### 7A. Màn hình cho Domain Expert (ASCII)

Mock một màn hình review cho expert.

Expert cần thấy tối thiểu:

- AI đã route hoặc gắn nhãn gì,
- dấu hiệu hoặc evidence nào khiến case bị đẩy sang expert,
- expert có thể duyệt / sửa / escalation ở đâu.

**Trả lời của bạn:**

```text
...
```

#### 7B. Tiêu chí review của Domain Expert

Liệt kê các tiêu chí domain expert sẽ dùng để duyệt case này.

### 8. Release Gate

Đề xuất release gate phù hợp cho case này. Nêu rõ điều kiện chặn, ngưỡng chất lượng tối thiểu, và trường hợp cần human review.

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

---
