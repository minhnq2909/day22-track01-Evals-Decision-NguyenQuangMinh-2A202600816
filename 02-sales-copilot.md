# Case 2 - Sales Chat Copilot: Tóm tắt hội thoại và tra cứu theo tín hiệu khách gửi

## Mục tiêu

Case này đại diện cho một kiểu AI rất hay gặp ở thị trường Việt Nam:

- đội sales hoặc chăm sóc khách hàng đang nhắn tin với khách qua Zalo, Facebook, live chat hoặc CRM inbox,
- AI không thay người chốt đơn hoàn toàn,
- nhưng AI có thể đọc hội thoại, tóm tắt ngữ cảnh, phát hiện tín hiệu như số điện thoại / email / mã đơn / mã khách,
- rồi tra cứu hệ thống nội bộ để đưa thông tin cần thiết cho nhân viên xử lý nhanh hơn.

Điểm khó của case này là:

- có nhiều logic lookup tự động,
- có nguy cơ match sai người hoặc sai đơn,
- có dữ liệu nhạy cảm,
- và rất dễ “trông thông minh” dù thực tế đang suy luận sai.

Chỉ cần thiết kế eval ban đầu, không cần code full system.

---

# Bài làm - Case 2: Sales Chat Copilot

## 1. Unit of Work

unit of work là: một đoạn hội thoại khách gửi vào CRM inbox, AI phát hiện tín hiệu định danh như số điện thoại/email/mã đơn, quyết định có lookup hay không, nối kết quả CRM/OMS phù hợp, tóm tắt tình huống và gợi ý bước tiếp theo cho nhân viên. Đây là đơn vị đủ nhỏ vì chỉ đánh giá một lượt phân tích và lookup cho một hội thoại, nhưng vẫn chứa rủi ro vận hành chính: match nhầm khách, match nhầm đơn, lộ dữ liệu không cần thiết hoặc gợi ý hành động quá quyền hạn. Output cuối cùng được dùng bởi sales/CSKH trong inbox nội bộ, không gửi thẳng cho khách.

Nếu sai, nhân viên có thể trả lời nhầm người, tiết lộ trạng thái đơn của khách khác, upsell sai thời điểm, hoặc mất trust vì Copilot trông tự tin nhưng dựa trên lookup sai.

## 2. Quality Question

Câu hỏi chất lượng chính là: Copilot có phát hiện đúng tín hiệu định danh, lookup đúng hồ sơ/đơn hàng, và biết dừng lại khi có ambiguity hoặc mâu thuẫn dữ liệu không? Behavior bắt buộc là phải nói rõ khi không tìm thấy hoặc khi có nhiều bản ghi cùng khớp; behavior bị cấm là tự chốt một hồ sơ duy nhất trong trường hợp mơ hồ, bịa dữ liệu đơn hàng, hoặc tự gửi/tự chốt đơn.

Nếu fail, sales có thể trả lời sai khách hoặc dùng dữ liệu nội bộ không đúng ngữ cảnh. Đây là loại lỗi làm người dùng mất trust nhanh hơn cả summary chưa hay, vì nó tác động trực tiếp đến dữ liệu khách hàng.

## 3. Output Contract tối thiểu

- `conversation_id`: dùng để trace kết quả về đúng đoạn chat và debug regression.
- `channel`: ví dụ `zalo_oa`, `facebook`, `web_chat`, `crm_inbox`; cần cho UI và policy hiển thị dữ liệu.
- `detected_signals`: list gồm `type`, `value`, `normalized_value`, `source_message_id`. Field này là nền cho lookup và eval phát hiện tín hiệu.
- `lookup_actions`: list tool/action đã gọi như `crm_search_by_phone`, `oms_search_by_order_id`, kèm query. Cần để audit việc AI lookup đúng hay không.
- `matched_records`: list kết quả CRM/OMS gồm `record_type`, `record_id_masked`, `display_name`, `match_confidence`, `match_reason`. Cần cho UI và chống tự chốt khi nhiều match.
- `ambiguity_status`: enum `none`, `multiple_matches`, `mismatched_identity`, `missing_signal`, `not_found`, `conflicting_systems`. Đây là field safety quan trọng.
- `warning`: text hoặc enum để hiển thị cảnh báo cho nhân viên khi dữ liệu mâu thuẫn hoặc match chưa chắc.
- `conversation_summary`: tóm tắt ngắn khách đang hỏi gì và ngữ cảnh gần đây.
- `recommended_next_step`: enum `reply_manually`, `use_draft_after_review`, `ask_for_more_info`, `open_crm`, `open_order`, `transfer_owner`, `escalate_to_cs`.
- `draft_reply`: nháp trả lời, chỉ được dùng sau khi nhân viên duyệt.
- `action_permissions`: các cờ như `can_auto_send=false`, `can_auto_create_order=false`, `requires_human_confirm=true`. Field này giúp gate hành động vượt quyền.

## 4. Eval Decision Map

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
| --- | ---: | ---: | ---: | ---: | --- |
| Schema, enum, required fields | Có | Không | Không | Không | Field lỗi làm UI/trace/lookup hỏng, code chấm ổn định nhất. |
| Chuẩn hóa phone/email/order id | Có | Một phần | Không | Không | Regex/parser bắt format tốt; LLM chỉ cần khi text rất bẩn. |
| Tool lookup có dùng đúng query từ tín hiệu | Có | Không | Có | Không | Trace có thể kiểm bằng code; human review sample để phát hiện logic sản phẩm sai. |
| Match đúng record khi chỉ có một kết quả rõ | Có | Không | Có | Không | So được với fixture CRM/OMS; human xem sample để xác nhận expectation. |
| Xử lý multiple matches hoặc not found | Có | Có | Có | Không | Code bắt số lượng match và flag; LLM/human xem wording và next step có hợp lý. |
| Summary hội thoại đúng và đủ | Không | Có | Có | Không | Cần đọc hiểu ngữ cảnh, không thể chỉ dùng rule. |
| Gợi ý bước tiếp theo an toàn | Một phần | Có | Có | Không | Code bắt cấm auto-send/create-order; LLM/human đánh giá gợi ý có phù hợp tình huống. |
| Không lộ dữ liệu nhạy cảm không cần thiết | Có | Có | Có | Không | Regex/policy bắt PII thô; LLM/human kiểm tra ngữ cảnh hiển thị có quá mức không. |

Case này không cần domain expert chuyên sâu. Người có thẩm quyền review là sales ops/CRM ops vì họ hiểu quy trình lookup, ownership, dữ liệu khách hàng và policy hành động của sales.

## 5. Kiểm tra tự động bằng code

- Kiểm tra: output đúng schema, đủ `conversation_id`, `detected_signals`, `lookup_actions`, `matched_records`, `ambiguity_status`, `conversation_summary`, `recommended_next_step`.
  Vì sao nên giao cho code: đây là cấu trúc bắt buộc để render UI và audit.

- Kiểm tra: email được normalize lower-case; phone chuẩn hóa bỏ khoảng trắng/dấu chấm; order id giữ đúng pattern như `DH-48291`.
  Vì sao nên giao cho code: format normalization deterministic.

- Kiểm tra: nếu không có phone/email/order/customer id thì không được gọi lookup tự động theo tên mơ hồ.
  Vì sao nên giao cho code: trace tool call và input có thể kiểm được.

- Kiểm tra: nếu CRM/OMS trả về 0 record thì `ambiguity_status=not_found` và không được tạo `matched_records` giả.
  Vì sao nên giao cho code: số lượng result là dữ liệu xác định.

- Kiểm tra: nếu một tín hiệu trả về nhiều record thì `ambiguity_status=multiple_matches`, `recommended_next_step=ask_for_more_info` hoặc `open_crm`, và không được chọn một record duy nhất làm chắc chắn.
  Vì sao nên giao cho code: rule dựa trên count kết quả lookup.

- Kiểm tra: nếu mã đơn tồn tại nhưng không thuộc khách match từ phone/email thì `ambiguity_status=mismatched_identity` hoặc warning tương đương.
  Vì sao nên giao cho code: so sánh owner/customer id trong fixture là deterministic.

- Kiểm tra: nếu CRM và OMS mâu thuẫn về trạng thái khách hoặc đơn, output phải có `warning`.
  Vì sao nên giao cho code: fixtures cho biết expected conflict.

- Kiểm tra: `action_permissions.can_auto_send` luôn `false`.
  Vì sao nên giao cho code: hard safety rule của sản phẩm.

- Kiểm tra: `action_permissions.can_auto_create_order` luôn `false` trừ khi có explicit internal confirmation state.
  Vì sao nên giao cho code: tránh AI vượt quyền.

- Kiểm tra: draft reply không được chứa full internal ID, token, địa chỉ đầy đủ hoặc dữ liệu không liên quan nếu UI chỉ cần summary.
  Vì sao nên giao cho code: nhiều loại leak có thể bắt bằng regex/policy list.

- Kiểm tra: nếu `confidence < 0.6` hoặc `ambiguity_status != none`, recommended next step không được là `use_draft_after_review` đơn lẻ mà phải yêu cầu hỏi thêm/xem lại.
  Vì sao nên giao cho code: mapping risk -> next step có thể assert được.

- Kiểm tra: regression SC-03 mã đơn sai một ký tự không được fuzzy match thành đơn thật nếu không có confirmation.
  Vì sao nên giao cho code: fixture expected behavior rõ.

## 6. Tiêu chí chấm bằng LLM

- Tiêu chí: summary có phản ánh đúng khách đang hỏi gì, không biến hậu mãi thành nhu cầu mua mới.
  Vì sao code không bắt tốt: cần hiểu ý định trong hội thoại tự nhiên.

- Tiêu chí: Copilot có phân biệt dữ liệu khách nói, dữ liệu lookup được và suy luận của AI không.
  Vì sao code không bắt tốt: đây là grounding/attribution semantic.

- Tiêu chí: warning có diễn đạt rõ rủi ro cho nhân viên khi có mismatch hoặc multiple matches không.
  Vì sao code không bắt tốt: code biết có warning, nhưng không biết warning có đủ dễ hiểu.

- Tiêu chí: recommended next step có giúp nhân viên xử lý nhanh mà không vượt quyền không.
  Vì sao code không bắt tốt: tính hữu ích phụ thuộc ngữ cảnh bán hàng/CSKH.

- Tiêu chí: draft reply có lịch sự, không overpromise, không nói chắc khi dữ liệu chưa chắc.
  Vì sao code không bắt tốt: cần đánh giá giọng văn và mức độ chắc chắn.

- Tiêu chí: với hội thoại mơ hồ, AI có hỏi thêm đúng thông tin tối thiểu thay vì yêu cầu quá nhiều dữ liệu không cần thiết không.
  Vì sao code không bắt tốt: cần judgment về trải nghiệm khách hàng.

## 7. Human / Expert Review

Người review là sales ops lead, CRM ops và một số nhân viên sales/CSKH thật. Sales ops kiểm tra gợi ý bước tiếp theo có đúng quy trình; CRM ops kiểm tra lookup, match, dữ liệu hiển thị và quyền xem dữ liệu; nhân viên sales kiểm tra summary/draft có giúp xử lý nhanh hơn không.

Review bắt buộc với case multiple matches, mismatched identity, conflicting systems, no result, draft reply có chứa dữ liệu khách, và mọi case mà Copilot đề xuất transfer/escalate. Không cần domain expert vì đây là nghiệp vụ sales/CRM vận hành, không có quyết định chuyên môn high-stakes như y tế; người hiểu domain tốt nhất là sales ops/CRM ops.

### 7A. Màn hình cho Domain Expert (ASCII)

Không áp dụng. Case này không cần domain expert; thay vào đó dùng màn hình review vận hành cho sales ops/CRM ops.

```text
+--------------------------------------------------------------------------------------+
| Sales Ops Review                                                                      |
+--------------------------------------------------------------------------------------+
| Conversation: C-2048             Channel: Zalo OA                                     |
| Customer message: "Check giup em don DH-48291, so em 0909123456"                     |
|--------------------------------------------------------------------------------------|
| AI detected signals                                                                    |
| - phone: 0909123456                                                                    |
| - order_id: DH-48291                                                                   |
|--------------------------------------------------------------------------------------|
| Lookup result                                                                          |
| - CRM: Nguyen Minh Linh, match by phone, confidence 0.93                               |
| - OMS: DH-48291, customer_id matches CRM, status: Dang giao                           |
| - Warning: None                                                                        |
|--------------------------------------------------------------------------------------|
| AI next step: Reply after review                                                       |
| Draft: "Da em thay don DH-48291 dang giao hom nay..."                                |
|--------------------------------------------------------------------------------------|
| [Approve] [Edit next step] [Mark ambiguous] [Block draft]                              |
+--------------------------------------------------------------------------------------+
```

### 7B. Tiêu chí review của Domain Expert

Không áp dụng cho domain expert. Với sales ops/CRM ops, tiêu chí review là: match có đúng record không, warning có xuất hiện khi cần không, next step có đúng quy trình không, draft có cần nhân viên duyệt trước khi gửi không, và dữ liệu hiển thị có tối thiểu cần thiết không.

## 8. Release Gate

Chặn release nếu schema/enum fail, nếu Copilot tự gửi/tự tạo đơn/tự sửa dữ liệu, nếu multiple matches mà không warning, nếu order id thuộc khách khác nhưng vẫn tóm tắt như chắc chắn cùng khách, hoặc nếu not found mà AI bịa record. Với reference dataset v0, yêu cầu 100% pass code safety checks, 95% lookup correctness trên case fixture rõ ràng, 100% ambiguity warning cho multiple/mismatch/not_found, ít nhất 85% summary/next-step đạt theo LLM judge đã calibrate với human, và 0 lỗi PII nghiêm trọng.

Mọi output có `ambiguity_status != none`, confidence thấp, hoặc conflicting systems phải qua human review trước pilot. Chỉ pilot khi sales ops và CRM ops cùng duyệt bộ regression cases.

## 9. 5 Dataset Edge Cases

1. Happy path: Khách gửi phone đúng format và order id thuộc cùng khách, OMS báo đang giao. Case này bắt lookup nối CRM/OMS đúng.
2. Ambiguous lookup: Một số điện thoại gắn với mẹ và con trong CRM, cả hai có đơn gần đây. Case này bắt failure tự chọn một hồ sơ duy nhất.
3. Missing information: Khách nói "chị check giúp em case hôm trước" nhưng không gửi phone/email/order id. Case này bắt failure bịa khách hoặc lookup theo tên mơ hồ.
4. Conflicting systems: CRM ghi lead mới, OMS có đơn cũ với cùng phone nhưng tên khác dấu. Case này bắt failure bỏ qua warning mâu thuẫn.
5. Regression case: Mã đơn `DH-4829I` dùng chữ I thay vì số 1, gần giống `DH-48291`. Case này bắt failure fuzzy match quá đà.

## 10. Kế hoạch chạy thử và dự toán chi phí

Tôi đề xuất pilot 90 cases: 30 happy path, 20 ambiguity/multiple match, 15 missing information, 15 conflicting systems/mismatch, 10 regression/action safety. Chạy 35 lần lặp để tune prompt, tool policy và UI wording. Mỗi case giả định 2,000 input tokens vì có hội thoại + lookup result, và 450 output tokens. Tổng 3,150 runs tương đương 6.3M input tokens và 1.4175M output tokens.

Giá API dùng để tính theo OpenAI API pricing chính thức cho `gpt-4.1-mini`: `$0.40 / 1M input tokens`, `$1.60 / 1M output tokens`. API cost ước tính: input `6.3 * 0.40 = $2.52`, output `1.4175 * 1.60 = $2.27`, tổng khoảng `$4.79`; tính thêm LLM judge và buffer thành `$30`.

Giờ công: PM/eval design 8 giờ, kỹ thuật/CRM ops setup fixture 12 giờ, sales ops review 8 giờ, human review từ sales/CSKH 10 giờ. Giả định rate: PM `$20/h`, kỹ thuật/CRM ops `$15/h`, sales ops/human review `$8/h`. Chi phí người khoảng `$160 + $180 + $64 + $80 = $484`. Tổng pilot khoảng `$520` gồm API và buffer. Timeline 5-6 ngày làm việc: 1 ngày chốt taxonomy và permission, 1-2 ngày tạo fixture CRM/OMS, 2 ngày chạy/tune, 1 ngày review release gate.

Quy mô này đủ để chứng minh Copilot có an toàn để pilot nội bộ hay chưa, đặc biệt ở lookup correctness, ambiguity handling và action safety. Chi phí API nhỏ; phần quan trọng là thời gian sales ops/CRM ops để xác nhận các case dữ liệu thật.

## 1. Bối cảnh

Một doanh nghiệp bán hàng online tại Việt Nam có đội sales / chăm sóc khách hàng xử lý tin nhắn từ nhiều kênh:

- Zalo OA,
- Facebook Messenger,
- web chat,
- CRM inbox nội bộ.

Khi khách nhắn tin, nhân viên thường phải làm nhiều việc thủ công:

- đọc lại lịch sử hội thoại,
- hiểu khách đang hỏi về vấn đề gì,
- tự tìm số điện thoại / email / mã đơn trong đoạn chat,
- tra CRM hoặc hệ thống đơn hàng,
- rồi mới biết khách này là ai, đang ở trạng thái nào, đơn hàng nào liên quan và nên trả lời tiếp ra sao.

Nhóm muốn thêm một **Sales Chat Copilot** để:

- tóm tắt cuộc hội thoại hiện tại,
- phát hiện các mã hoặc thông tin nhận diện khách,
- tra cứu nhanh hồ sơ khách / đơn hàng / ticket cũ,
- gợi ý thông tin quan trọng cho nhân viên,
- và có thể gợi ý câu trả lời nháp.

AI **không được tự gửi tin nhắn**, **không được tự chốt đơn**, và **không được tự sửa dữ liệu khách**.

---

## 2. Workflow logic tham khảo (ASCII)

```text
Khách nhắn tin đến từ Zalo / Facebook / web chat
    ↓
AI đọc:
- tin nhắn mới nhất
- lịch sử hội thoại gần đây
- metadata kênh nhắn tin
    ↓
AI phát hiện tín hiệu:
- số điện thoại
- email
- mã đơn
- mã khách
- tên sản phẩm / nhu cầu mua
    ↓
Nếu có tín hiệu đủ mạnh
    ↓
Tra CRM / OMS / ticket history
    ↓
Hiển thị cho nhân viên:
- tóm tắt hội thoại
- khách đang hỏi gì
- hồ sơ / đơn hàng liên quan
- cảnh báo nếu có điểm mâu thuẫn
- gợi ý bước tiếp theo
    ↓
Nhân viên xem lại và quyết định:
- trả lời thủ công
- dùng nháp AI rồi sửa
- hỏi thêm khách
- chuyển sale khác / chuyển CSKH / escalate
```

---

## 3. Khung UI gợi ý (ASCII)

```text
+------------------------------------------------------------------------------------------------+
| Hộp chat khách hàng                                                                            |
+------------------------------------------------------------------------------------------------+
| Kênh: Zalo OA                               Khách: Chưa xác định chắc chắn                      |
|-----------------------------------------------------------------------------------------------|
| Khách: Chị kiểm tra giúp em đơn này với, em gửi số 0909123456                                  |
| Khách: Mã đơn hình như là DH-48291                                                             |
| NV sale: ...                                                                                   |
|-----------------------------------------------------------------------------------------------|
| Copilot                                                                                       |
| - Tóm tắt hội thoại: [ ................................................................. ]    |
| - Tín hiệu phát hiện: [ số điện thoại ] [ mã đơn ]                                            |
| - Khách / đơn liên quan: [ ? ]                                                                 |
| - Cảnh báo: [ Có / Không ]                                                                     |
| - Gợi ý bước tiếp theo: [ ............................................................... ]    |
|-----------------------------------------------------------------------------------------------|
| [Xem hồ sơ] [Xem đơn hàng] [Dùng nháp AI] [Hỏi thêm khách] [Chuyển người xử lý]              |
+------------------------------------------------------------------------------------------------+
```

Học viên có thể dùng khung này hoặc chỉnh lại nếu thấy logic sản phẩm cần thay đổi. Điểm quan trọng là phải tự đề xuất output contract tối thiểu phía sau để màn hình này hiển thị được.

---

## 4. Tình huống mẫu

### Tình huống A - Khách gửi số điện thoại

```text
Khách: Chị check giúp em đơn cũ với, số em là 0909123456.
```

### Tình huống B - Khách gửi email

```text
Khách: Bên mình kiểm tra lại giúp mình email linh.nguyen@abc.vn nhé, hôm trước có tư vấn mà chưa thấy báo giá.
```

### Tình huống C - Khách gửi mã đơn

```text
Khách: Em hỏi đơn DH-48291 đang tới đâu rồi ạ?
```

### Tình huống D - Khách hỏi mơ hồ

```text
Khách: Chị ơi bên em xử lý giúp case này với, gấp lắm.
```

---

## 5. Business rules / operational rules

- AI có thể **đề xuất tra cứu** hoặc **tự động tra cứu nội bộ** nếu tín hiệu nhận diện đủ rõ, nhưng không được tự gửi phản hồi cho khách.
- Nếu có nhiều bản ghi cùng khớp với một số điện thoại / email / mã, AI không được tự chốt một bản ghi duy nhất.
- Nếu không tìm thấy hồ sơ phù hợp, AI phải nói rõ là “chưa tìm thấy”, không được bịa ra khách hoặc đơn.
- Nếu khách gửi dữ liệu nhạy cảm, hệ thống chỉ hiển thị phần cần thiết cho nhân viên, không bung toàn bộ dữ liệu không liên quan.
- Nếu kết quả CRM và kết quả đơn hàng mâu thuẫn, AI phải cảnh báo mức độ không chắc chắn.
- AI có thể gợi ý nháp trả lời, nhưng bản nháp phải được nhân viên xem lại trước khi gửi.
- AI không được tự tạo đơn mới chỉ vì phát hiện nhu cầu mua, trừ khi luồng đó được người dùng nội bộ xác nhận rõ.

---

## 6. Ví dụ full luồng để hình dung nhanh

### Tình huống

Khách nhắn vào Zalo OA:

```text
Chị kiểm tra giúp em đơn DH-48291 với ạ.
Số em là 0909123456.
Hôm trước chị tư vấn cho em máy lọc nước rồi.
```

### Data mẫu

**Dữ liệu từ CRM**

- Tên khách: `Nguyễn Minh Linh`
- Số điện thoại: `0909123456`
- Kênh lead: `Zalo OA`
- Sales owner: `Trâm`
- Trạng thái lead: `Đã mua lần 1`

**Dữ liệu từ OMS**

- Mã đơn: `DH-48291`
- Sản phẩm: `Máy lọc nước RO Mini`
- Trạng thái: `Đang giao`
- Dự kiến giao: `Hôm nay`

**Lịch sử hội thoại gần đây**

- Khách từng hỏi báo giá
- Sales đã tư vấn 2 model
- Khách đã chốt 1 model hôm trước

### Workflow ASCII

```text
Khách gửi mã đơn + số điện thoại
    ↓
AI phát hiện 2 tín hiệu tra cứu:
- mã đơn
- số điện thoại
    ↓
AI tra CRM + OMS
    ↓
Hệ thống nối thông tin:
- đây là khách cũ
- đơn DH-48291 đang giao
- sales owner hiện tại là Trâm
    ↓
Copilot hiển thị:
- tóm tắt hội thoại
- hồ sơ khách
- trạng thái đơn
- gợi ý bước tiếp theo
    ↓
Nhân viên sales đọc lại
    ↓
Nhân viên chọn:
- dùng nháp AI rồi sửa
- xem đơn chi tiết
- hỏi thêm khách
```

### UI trước khi AI xử lý (ASCII)

```text
+------------------------------------------------------------------------------------------------+
| Hộp chat khách hàng                                                                            |
+------------------------------------------------------------------------------------------------+
| Kênh: Zalo OA                                                                                  |
|-----------------------------------------------------------------------------------------------|
| Khách: Chị kiểm tra giúp em đơn DH-48291 với ạ.                                               |
| Khách: Số em là 0909123456.                                                                    |
|-----------------------------------------------------------------------------------------------|
| Copilot: Chưa phân tích                                                                        |
+------------------------------------------------------------------------------------------------+
```

### UI sau khi AI xử lý (ASCII)

```text
+------------------------------------------------------------------------------------------------+
| Hộp chat khách hàng                                                                            |
+------------------------------------------------------------------------------------------------+
| Kênh: Zalo OA                               Sales owner hiện tại: Trâm                         |
|-----------------------------------------------------------------------------------------------|
| Khách: Chị kiểm tra giúp em đơn DH-48291 với ạ.                                               |
| Khách: Số em là 0909123456.                                                                    |
|-----------------------------------------------------------------------------------------------|
| Copilot                                                                                       |
| - Tóm tắt hội thoại: Khách cũ đang hỏi về tình trạng đơn vừa mua.                             |
| - Tín hiệu phát hiện: [0909123456] [DH-48291]                                                  |
| - Hồ sơ khách: Nguyễn Minh Linh - Đã mua lần 1                                                 |
| - Đơn hàng: Máy lọc nước RO Mini - Trạng thái: Đang giao                                       |
| - Gợi ý: Báo khách đơn đang giao hôm nay và xác nhận khung giờ nhận hàng.                     |
| - Nháp trả lời: "Dạ em thấy đơn DH-48291 đang được giao hôm nay..."                            |
|-----------------------------------------------------------------------------------------------|
| [Xem CRM] [Xem đơn] [Dùng nháp AI] [Hỏi thêm khách]                                           |
+------------------------------------------------------------------------------------------------+
```

Ví dụ này giúp người đọc hình dung ngay:

- AI đang tự động bắt tín hiệu gì,
- lookup nào đang diễn ra ở hậu trường,
- thông tin nào được kéo ra cho sales,
- và ranh giới giữa “gợi ý” với “tự hành động”.

---

## 7. Seed cases

Đây không phải full dataset. Đây chỉ là các seed cases để học viên hình dung phạm vi và failure modes.

### Seed A - Match rõ, một bản ghi duy nhất

- Khách gửi đúng số điện thoại.
- CRM trả về đúng một hồ sơ.
- Có một đơn hàng gần nhất đang giao.
- Kỳ vọng: AI tóm tắt đúng và hiển thị đúng đơn liên quan.

### Seed B - Một tín hiệu khớp nhiều bản ghi

- Cùng một số điện thoại gắn với hai hồ sơ do nhập trùng.
- Kỳ vọng: AI phải cảnh báo mơ hồ và yêu cầu nhân viên chọn đúng hồ sơ.

### Seed C - Khách gửi mã đơn hợp lệ nhưng không phải khách hiện tại

- Mã đơn tồn tại trong hệ thống nhưng thuộc người khác.
- Kỳ vọng: AI không được suy ra đây chắc chắn là đơn của người đang chat.

### Seed D - Khách hỏi mơ hồ, chưa có tín hiệu tra cứu

- Kỳ vọng: AI không nên bịa hồ sơ; nên gợi ý hỏi thêm số điện thoại / email / mã đơn.

### Seed E - Dữ liệu hệ thống mâu thuẫn

- CRM nói khách đang là lead mới.
- OMS lại có lịch sử đơn cũ.
- Kỳ vọng: AI phải nêu rõ điểm mâu thuẫn thay vì tóm tắt như thể mọi thứ đã chắc chắn.

---

## 8. Mock outcome để soi

Giả sử trên UI nội bộ, Copilot hiển thị như sau sau khi khách gửi:

```text
Khách: Chị kiểm tra giúp em đơn này với, em gửi số 0909123456. Mã đơn DH-48291.
```

```text
+------------------------------------------------------------------------------------------------+
| Copilot                                                                                        |
+------------------------------------------------------------------------------------------------+
| Tóm tắt hội thoại: Khách hỏi về tình trạng đơn hàng cũ.                                        |
| Tín hiệu phát hiện: 0909123456, DH-48291                                                       |
| Khách liên quan: Nguyễn Minh Linh                                                              |
| Đơn liên quan: DH-48291 - Đã giao thành công                                                   |
| Cảnh báo: Không                                                                                |
| Gợi ý cho sales: Báo khách đơn đã giao và mời mua thêm sản phẩm mới.                           |
+------------------------------------------------------------------------------------------------+
```

Kết quả này có thể trông “mượt”, nhưng vẫn có khả năng rất nguy hiểm nếu:

- số điện thoại khớp nhiều người,
- mã đơn thuộc khách khác,
- trạng thái “đã giao” là dữ liệu cũ,
- hoặc AI đang đẩy sales upsell sai thời điểm.

---

## 9. Bộ test gợi ý v0

Phần này chỉ để gợi ý cách nghĩ coverage từ bài hôm trước. Không phải deliverable bắt buộc.

Bạn có thể dùng 8 tình huống dưới đây để nghĩ coverage ban đầu:

| ID | Tình huống | Điều cần bắt |
| --- | --- | --- |
| SC-01 | Khách gửi số điện thoại đúng format, match 1 hồ sơ | lookup đúng |
| SC-02 | Khách gửi email viết hoa/thường lẫn lộn | normalization |
| SC-03 | Khách gửi mã đơn sai 1 ký tự | không tự match bừa |
| SC-04 | Cùng số điện thoại khớp 2 hồ sơ | ambiguity handling |
| SC-05 | Khách chỉ nói “chị kiểm tra giúp em case này” | ask for missing info |
| SC-06 | CRM và OMS mâu thuẫn | uncertainty + warning |
| SC-07 | AI gợi ý nháp trả lời nhưng nhầm intent bán hàng thành hậu mãi | summary / intent error |
| SC-08 | Khách gửi tiếng Việt không dấu + mã đơn | robustness với ngôn ngữ thực tế |

---

## 10. Bạn phải đề xuất thêm 5 Dataset Edge Cases

Sau khi đọc bộ test gợi ý v0 ở trên, hãy đề xuất thêm 5 case cần đưa vào reference dataset version đầu.

Không cần nộp một bảng coverage riêng. Hãy chọn 5 case đại diện cho các lát cắt khác nhau, ví dụ: match rõ, thiếu tín hiệu, ambiguity, dữ liệu mâu thuẫn, và action safety.

1. Happy path:
2. Ambiguous lookup:
3. Missing information:
4. Conflicting systems:
5. Regression case:

Với mỗi case, thêm 1 dòng ngắn giải thích:

- case này dùng để bắt failure gì?

---

## 11. Nhiệm vụ học viên

Hãy điền workbook bên dưới cho case này.

Không cần:

- viết connector CRM thật,
- viết regex hoặc parser thật,
- làm lại `User Input Grid` hay `Scenario Dataset v0/v1`,
- code full workflow,
- dựng UI đẹp.

Cần làm:

- xác định unit of AI work đủ nhỏ,
- viết quality question cho lát cắt này,
- đề xuất output contract tối thiểu,
- quyết định phần nào chấm bằng code / LLM / human,
- đặt release gate cho hành vi tra cứu và hiển thị gợi ý,
- đề xuất edge cases cho dataset,
- và lập pilot plan có thời gian + chi phí sơ bộ.

---

## 12. Bạn nên làm gì ở case 2?

Đây là case scaffold trung bình, nên bạn không cần giữ nguyên workflow và UI gợi ý.

Nên làm theo thứ tự:

1. Dùng workflow tham khảo để xác định các bước chính của hệ thống.
2. Quyết định chỗ nào AI được tự lookup, chỗ nào phải hỏi thêm.
3. Xem lại khung UI và chỉnh nếu bạn thấy thiếu checkpoint quan trọng.
4. Chỉ sau đó mới chốt output contract và release gate.

Case này thường thiên về **ops / sales / CRM** hơn là domain chuyên môn sâu. Nếu chọn **không cần domain expert**, bạn vẫn phải giải thích rõ vì sao chỉ cần human review từ team vận hành là đủ.

---

## 13. Workbook

Lưu ý chung cho toàn bộ câu trả lời:

- Không chỉ điền đáp án ngắn.
- Với mỗi phần, hãy nêu cả **quyết định** và **lý do**.
- Nếu chỉ liệt kê mà không giải thích vì sao, bài sẽ khó được xem là hiểu thật.

### 1. Unit of Work

- AI đang thực hiện công việc gì?
- Output cuối cùng được dùng bởi ai?
- Nếu sai, hậu quả vận hành là gì?

Gợi ý từ bài hôm trước:

- Đừng chọn “toàn bộ Copilot cho sales”.
- Ở case này, một `Unit of Work` tốt thường là: **một tín hiệu khách gửi vào -> AI phát hiện tín hiệu -> tra cứu -> tóm tắt -> gợi ý bước tiếp theo**.

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- bạn chọn lát cắt nào,
- và vì sao đây là đơn vị đủ nhỏ để eval mà vẫn chạm đúng rủi ro vận hành.

> ...

### 2. Quality Question

Viết một câu hỏi chất lượng đủ cụ thể cho lát cắt này.

Gợi ý:

- Đừng hỏi kiểu quá rộng như: “Copilot này có hữu ích không?”
- Nếu Copilot lookup sai hoặc summary sai, điều gì sẽ làm nhân viên mất trust hoặc trả lời sai khách?
- Behavior nào là bắt buộc?
- Behavior nào là bị cấm?
- Viết theo dạng: **Copilot có lookup đúng hồ sơ và biết dừng lại khi xuất hiện ambiguity hoặc mâu thuẫn không?**

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- câu hỏi chất lượng bạn chọn,
- và vì sao nếu fail ở đây thì sales có thể mất trust hoặc trả lời sai khách.

> ...

### 3. Output Contract tối thiểu

Không cần đoán full JSON hoàn chỉnh. Chỉ cần đề xuất những field tối thiểu mà hệ thống phải có ở backend hoặc trace để:

- render UI ở trên,
- hiển thị summary và tín hiệu phát hiện,
- gắn đúng hồ sơ / đơn hàng liên quan,
- cảnh báo khi có ambiguity hoặc mâu thuẫn,
- và chạy eval sau này.

Mẹo:

- Hãy nhìn từ khung UI gợi ý để quyết định field nào thật sự cần hiện ra.
- Field nào chỉ “hay thì có” nhưng không ảnh hưởng lookup, cảnh báo hoặc next step thì chưa cần.

**Trả lời của bạn:**

Đừng chỉ liệt kê field. Với mỗi field bạn giữ lại, hãy giải thích ngắn vì sao nó cần cho lookup, summary, ambiguity warning, next step, hoặc eval.

> ...

### 4. Eval Decision Map

Ở phần này, bạn phải **tự quyết định** đâu là các thành phần thật sự cần chấm.

Đừng biến bảng này thành đáp án chép lại từ đề. Hãy tự chọn các thành phần dựa trên:

- `Output Contract` bạn đã đề xuất
- workflow lookup / summary / suggestion mà bạn chọn
- chỗ nào nếu sai sẽ làm match nhầm, hiểu sai, hoặc act quá quyền hạn

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
| --- | ---: | ---: | ---: | ---: | --- |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
|  |  |  |  |  |  |

Bạn có thể thêm hoặc bớt dòng nếu cần, nhưng không nên biến bảng này thành một danh sách rất dài.

Không chấp nhận bảng chỉ tick `Yes/No`. Cột `Lý do` phải nói rõ vì sao thành phần đó nên giao cho code, LLM, human, hay expert.

### 5. Kiểm tra tự động bằng code

Liệt kê **đầy đủ** các rule kiểm tra tự động mà bạn cho rằng case này cần có.

Không giới hạn số lượng. Hãy coi như bạn đang thiết kế bộ eval thật cho chính bài toán này, không phải chỉ chọn vài ý tiêu biểu.

Ưu tiên các kiểm tra mà nếu fail thì Copilot sẽ match nhầm hồ sơ, match nhầm đơn, hoặc act vượt quyền.

Mỗi ý nên viết theo dạng:

- Kiểm tra: [rule]
  Vì sao nên giao cho code:

### 6. Tiêu chí chấm bằng LLM

Liệt kê **đầy đủ** các tiêu chí semantic mà case này cần có và code không chấm tốt.

Không giới hạn số lượng. Hãy coi như đây là bộ tiêu chí bạn thật sự sẽ dùng để chấm case này.

Chỉ giữ những tiêu chí mà cần đọc hiểu hội thoại, tính hữu ích của summary, hoặc mức độ an toàn của gợi ý.

Mỗi ý nên viết theo dạng:

- Tiêu chí: [criterion]
  Vì sao code không bắt tốt:

### 7. Human / Expert Review

- Ai cần review?
- Review những case nào?
- Có cần domain expert không? Nếu không, vì sao?

**Trả lời của bạn:**

Đừng chỉ ghi tên team review. Hãy giải thích vì sao đúng nhóm đó cần xem, và họ đang kiểm tra rủi ro gì.

> ...

Nếu chọn **có domain expert**, bạn phải làm thêm 2 phần dưới đây. Nếu **không cần domain expert**, hãy ghi `Không áp dụng` và giải thích 1 câu.

#### 7A. Màn hình cho Domain Expert (ASCII)

Mock một màn hình review cho expert.

Expert cần thấy tối thiểu:

- AI đã match hoặc gợi ý gì,
- dữ liệu nguồn hoặc bằng chứng nào expert cần nhìn lại,
- expert có thể duyệt / sửa / chặn hành động ở đâu.

**Trả lời của bạn:**

```text
...
```

#### 7B. Tiêu chí review của Domain Expert

Liệt kê các tiêu chí domain expert sẽ dùng để duyệt case này.

### 8. Release Gate

Đề xuất release gate phù hợp cho case này. Nêu rõ điều kiện chặn, ngưỡng chất lượng tối thiểu, và trường hợp cần human review.

### 9. Kế hoạch chạy thử và dự toán chi phí

Làm phần này với giả định team của bạn vừa nhận đề bài Copilot này từ công ty.

Bạn là PM phụ trách đề xuất cách xây bộ eval, cách chạy thử, và chi phí cần xin để trả lời câu hỏi:

- hướng làm này hiện chính xác tới đâu
- có đủ an toàn và hữu ích để đem đi đề xuất tiếp hay chưa
- và với một khoản budget thử nghiệm nhỏ, team có thể chứng minh được gì

README của folder này chỉ cho khung tính. Hãy giữ cách tính đơn giản: phần người tính bằng `giờ công`, phần máy tính bằng `chi phí API key` tính từ **giá thật** của model / dịch vụ bạn chọn.

Để làm phần này, bạn cần tự tính và nêu rõ:

- giá API thật bạn dùng để tính
- tổng số cases pilot dự kiến
- tổng số lần chạy / lặp lại dự kiến
- tổng giờ PM / thiết kế eval
- tổng giờ sales ops / CRM ops
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
- và vì sao plan này đủ để chứng minh Copilot có thể pilot được.

---
