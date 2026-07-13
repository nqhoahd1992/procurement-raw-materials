# Hướng dẫn sử dụng — Max Biocare Procurement App

Tài liệu này mô tả các **case (tình huống) sử dụng** của ứng dụng theo từng vai trò, dùng làm nguồn để soạn guide hướng dẫn người dùng cuối. Tất cả nhãn (label), tên nút, tên trường được ghi **nguyên văn tiếng Anh** như trong ứng dụng (UI hiện tại chưa dịch tiếng Việt).

---

## 1. Tổng quan luồng xử lý (Status flow)

```
RequestFormScreen (Requester tạo yêu cầu, phải thêm ít nhất 1 raw material, phải chọn Manager Approver)
   │  Luôn bắt đầu ở "Pending Manager" — mọi yêu cầu đều qua đủ 2 cấp duyệt (Manager rồi Executive)
   ▼
Pending Manager ──(ManagerReviewScreen)
   │ Bất kỳ quyết định nào (Approved (within budget) / Needs clarification / Exceeds budget / unplanned) → Pending Executive
   ▼
Pending Executive ──(ExecutiveApprovalScreen)
   │ "Reject"                          → Rejected
   │ "Approve" / "Approve with conditions" → Pending Procurement
   ▼
Pending Procurement ──(ProcurementExecutionScreen)
   │ "Reject"                          → Rejected
   │ "Proceed"                         → Goods Receipt & Acceptance
   │   (đặt InvoiceMode: Direct / Deferred / ViaRequester)
   ▼
Goods Receipt & Acceptance ──(GoodsReceiptScreen)
   │ "Accepted"                        → Pending Invoice (nếu invoice chưa nộp) hoặc Pending Accounting (nếu đã nộp)
   │ "Rejected"                        → Rejected
   │ "Requires Supplier Follow-up"     → Pending Supplier Follow-up
   ▼
Pending Supplier Follow-up ──(SupplierFollowUpScreen, 2 vòng)
   │ Round 1 (Requester):
   │   "Accepted"                → Pending Invoice (nếu invoice chưa nộp) hoặc Pending Accounting (nếu đã nộp)
   │   "Accepted with Adjustment"→ tiếp tục Round 2 (Procurement) + Credit Note
   │ Round 2 (Procurement, chỉ khi Round 1 = Adjustment):
   │   Sau khi nộp Credit Note   → Pending Invoice (nếu invoice chưa nộp) hoặc Pending Accounting (nếu đã nộp)
   ▼
Pending Invoice ──(InvoiceSubmissionScreen [Procurement/Admin] hoặc RequesterInvoiceScreen [Requester])
   │ InvoiceSubmissionScreen nộp →
   │   Pending Supplier Follow-up (nếu Round 1 đã xong nhưng Round 2 chưa)
   │   hoặc Pending Accounting
   │ RequesterInvoiceScreen chỉ nộp lại file invoice, KHÔNG đổi Status
   ▼
Pending Accounting ──(AccountingScreen)
   │ Submit → Completed (đây là nơi DUY NHẤT trong app đặt Status = Completed)
   ▼
Completed
```

---

## 2. HomeScreen — Trang chủ

**Mọi vai trò** đăng nhập vào đều thấy trang này đầu tiên.

### Điều kiện truy cập
Đây là quy trình nội bộ (không còn dùng cho toàn bộ tenant) — nhân viên phải có **cả hai**:
1. Một dòng trong `Employee List`.
2. Một dòng trong `RM User` gán Role (kể cả Role "Requester" — mọi người dùng, kể cả Requester thường, đều phải được thêm thủ công vào `RM User`).

Nếu thiếu 1 trong 2, màn hình hiện thông báo: *"⚠ Your account was not found in the system. Please contact your administrator."* và không hiển thị bất kỳ nội dung nào khác.

### Hiển thị theo vai trò
- Tiêu đề danh sách: "My Requests" (Requester/Manager), "All Requests" (Executive/Admin), "Procurement Requests" (Procurement), "Accounting Requests" (Accounting).
- Góc trên phải: tên nhân viên + vai trò, ví dụ "John Smith | Manager".

### Bộ lọc trạng thái (nút bấm)
"All", "Pending Manager", "Pending Executive", "Pending Procurement", "Pending Accounting", "Pending Invoice", "Goods Receipt", "Supplier Follow-up", "Completed", "Rejected".

### Ai thấy yêu cầu nào (case phân quyền)
| Vai trò | Thấy gì |
|---|---|
| Manager | Yêu cầu mình là ManagerApproverID, hoặc do mình tạo |
| Procurement / Accounting | Yêu cầu từ bước của họ trở đi, hoặc do mình tạo |
| Executive / Admin | Tất cả yêu cầu |
| Requester (mặc định) | Yêu cầu do mình tạo + yêu cầu được giao làm Goods Receipt / Supplier Follow-up Round 1 |

### Case: Nộp / xử lý invoice ngay tại Home (nút phụ trên card)
- Nếu invoice qua Requester (InvoiceMode = "ViaRequester") và chưa nộp và người xem là Requester: hiện nút **"Submit Invoice"** (cam) → mở RequesterInvoiceScreen.
- Nếu invoice ở dạng Deferred và Procurement/Admin xem:
  - Chưa có link invoice → nút **"Remind Requester"** (cam).
  - Đã có link invoice → nút **"Process Invoice"** (tím).

### Case: Bấm vào một request
- Nếu trạng thái = "Pending Invoice" và người xem là Procurement/Admin → mở **InvoiceSubmissionScreen**.
- Ngược lại → mở **RequestDetailScreen** (xem chi tiết).

### Nút "New Request"
Mở **RequestFormScreen** để tạo yêu cầu mới. Chỉ hiển thị cho vai trò được phép tạo (Requester và các vai trò khác đều có thể tạo cho bản thân).

---

## 3. RequestFormScreen — Tạo yêu cầu mới (Requester)

**Ai dùng:** Bất kỳ nhân viên nào muốn tạo yêu cầu mua hàng.

### Các trường bắt buộc (*)
1. **Procurement Type*** — radio: "Invoice Supplied" / "To be sourced by Procurement"
2. **Purchase Accordance*** — ví dụ: "Urgent", "Unplanned", "Standard"… (chỉ mang tính phân loại, không còn ảnh hưởng đến việc duyệt)
3. **Cost Center** — cố định "Port Melbourne Warehouse", chỉ xem, không chọn được
4. **Delivery Location** — cố định "Port Melbourne Warehouse", chỉ xem, không chọn được
5. **Required Delivery Date*** — chọn ngày
6. **Estimated Cost*** — phải là số
7. **Currency*** — dropdown chỉ 2 lựa chọn: "AUD" / "USD" (mặc định AUD)
8. **Procurement Description***
9. **Budget Reference** — không bắt buộc
10. **Related Product SKU** — không bắt buộc, cho phép chọn **nhiều** SKU từ danh mục sản phẩm liên quan
11. **Select Manager Approver*** — bắt buộc chọn người quản lý phê duyệt cho **mọi** yêu cầu (không còn trường hợp bỏ qua)

### Case: Raw Materials (bắt buộc ít nhất 1 dòng)
Bảng thêm nguyên liệu (**"+ Add Material"**), mỗi dòng gồm:
- **Trade Name*** — chọn từ danh mục `Raw Materials`; sau khi chọn sẽ hiện thêm **Code**, **Category** và **Supplier** của nguyên liệu đó (chỉ xem).
- **Unit*** — dropdown: "pcs", "kg", "box", "set", "liter", "meter".
- **Qty*** — số lượng, phải > 0.
- Nút **✕** để xóa dòng.

Không còn trường "Preferred Supplier" ở cấp yêu cầu — nhà cung cấp giờ gắn theo từng raw material trong danh mục, không chọn thủ công ở đây nữa.

### Case: Chọn "Invoice Supplied"
Hiện thêm:
- **Invoice Type*** — radio: "Official Invoice" / "Proforma Invoice"
- **Attach Invoice File*** — bắt buộc đính kèm 1 file

### Lỗi thường gặp khi Submit
- "Please fill in all required fields"
- "Estimated Cost is required and must be a number"
- "Please select an invoice type"
- "Please attach the invoice file before submitting"
- "Please select a Manager Approver" — luôn bắt buộc, cho mọi yêu cầu
- "Please add at least one raw material" — chưa thêm dòng nguyên liệu nào
- "Please select a material for all rows" — có dòng chưa chọn Trade Name
- "Please select a unit for all rows" — có dòng chưa chọn Unit
- "Please enter a valid quantity for all rows" — có dòng Qty trống hoặc ≤ 0

### Kết quả sau khi Submit
- Trạng thái mới luôn là **"Pending Manager"** — mọi yêu cầu đều đi qua đủ Manager rồi Executive, không còn trường hợp nhảy thẳng lên Executive.
- Thông báo: "Request submitted successfully" → về HomeScreen.

---

## 4. ManagerReviewScreen — Quản lý duyệt (Manager)

**Ai dùng:** Manager được chỉ định là ManagerApproverID của yêu cầu, ở trạng thái "Pending Manager".

### Thông tin chỉ xem
Toàn bộ thông tin yêu cầu: người tạo, chi phí, cost center, loại mua hàng, loại invoice, mô tả.

### Review Checklist * (phải tick đủ 4 mục)
- "Business justification is valid and necessary"
- "Cost is reasonable and aligned with department objectives"
- "Within approved departmental budget"
- "Timing and urgency are appropriate"

### Trường nhập
- **Manager Remarks*** — bắt buộc
- **Manager Decision*** — radio: "Approved (within budget)" / "Needs clarification" / "Exceeds budget / unplanned"

### Case theo quyết định
Dù chọn quyết định nào ("Approved (within budget)" / "Needs clarification" / "Exceeds budget / unplanned"), yêu cầu **luôn chuyển sang "Pending Executive"** — Manager không còn có thể duyệt thẳng sang Pending Procurement mà bỏ qua Executive. Quyết định chỉ mang tính ghi nhận lý do/kết quả review của Manager.

### Lỗi thường gặp
"Please complete all checklist items", "Manager Remarks is required", "Please select a decision".

---

## 5. ExecutiveApprovalScreen — Ban điều hành duyệt (Executive)

**Ai dùng:** Executive, khi yêu cầu ở trạng thái "Pending Executive" — mọi yêu cầu đều qua Manager trước khi tới đây (không còn đường nhảy thẳng).

### Case: Manager đã review (mọi yêu cầu mới)
Hiện lại tóm tắt Manager Remarks, Manager Decision và 4 checklist đã tick (chỉ xem).

### Case: Manager Review bị bỏ qua — chỉ còn xảy ra với yêu cầu cũ
Banner cam **"⚡ Manager Review Skipped"** (kèm lý do "Urgent purchase" / "Unplanned purchase" / "Estimated cost exceeds threshold") chỉ hiện với các yêu cầu được tạo **trước** khi quy trình đổi sang "luôn 2 cấp duyệt" — những yêu cầu này từng được escalate thẳng lên Executive và không có review của Manager. Yêu cầu tạo mới sẽ không bao giờ thấy banner này.

### Trường nhập
- **Executive Decision*** — radio: "Approve" / "Approve with conditions" / "Reject"
- **Approval Conditions*** — chỉ hiện & bắt buộc khi chọn "Approve with conditions"
- **Rejection Reason*** — chỉ hiện & bắt buộc khi chọn "Reject" (chữ đỏ)

### Case theo quyết định
| Quyết định | Trạng thái tiếp theo |
|---|---|
| Approve | Pending Procurement |
| Approve with conditions | Pending Procurement (kèm điều kiện lưu vào ConditionsText) |
| Reject | Rejected |

---

## 6. ProcurementExecutionScreen — Phòng mua hàng xử lý (Procurement)

**Ai dùng:** Procurement/Admin, khi yêu cầu ở "Pending Procurement".

### Procurement Decision* — radio
- "Proceed" (mặc định)
- "Reject" → hiện **Rejection Reason*** bắt buộc (nền đỏ nhạt)

### Case "Proceed" — Procurement Execution Checklist * (bắt buộc tick hết)
- "Supplier sourcing completed"
- "Price comparison and/or negotiation completed (where applicable)"
- "Purchase Order issued (if applicable)"
- "Supplier coordination and delivery tracking in place"
- "Official supplier invoice collected" (tùy theo invoice mode)
- "Invoice verified against approved procurement request and delivery status" (tùy theo invoice mode)

### Các trường khác khi "Proceed"
- **Supplier Selection / Negotiation Summary*** — bắt buộc
- **Purchase Order Link** — không bắt buộc

### Case xử lý invoice — 3 nhánh
1. **Deferred (Invoice Handling*)**: radio "Submit Invoice Now" / "Defer Invoice". Nếu "Submit Invoice Now" → phải upload **Order Confirmation Document***.
2. **Via Requester (Remittance*)**: upload **Remittance Advice Document***; đồng thời hiện phần kiểm tra invoice của Requester đã nộp (link xem invoice + radio "Is this invoice correct?" Yes/No).
3. **Official Invoice** (khi không phải Via Requester và không phải Official Invoice Type có sẵn): upload file rồi bấm **"🔍 Extract Invoice"** để AI trích xuất dữ liệu (xem case ở mục 7).

### Case: Executive có kèm điều kiện phê duyệt
Hiện banner cam "⚠ Executive Approval Conditions" hiển thị nội dung điều kiện.

### Nút submit và kết quả
- **"Submit →"**: tạo log bước 1 (ExecutionLog StepNumber 1), chuyển trạng thái sang **"Goods Receipt & Acceptance"** (hoặc Pending Invoice nếu invoice defer).
- **"Reject"**: chuyển trạng thái sang **Rejected**.

---

## 7. InvoiceSubmissionScreen — Xử lý & trích xuất invoice (Procurement/Admin)

**Khi nào xuất hiện:** Trạng thái "Pending Invoice", người xem là Procurement/Admin (bấm vào request từ Home).

### Case: Invoice qua Requester nhưng Requester chưa nộp
Hiện banner cam: *"Waiting for Requester to submit the corrected supplier invoice."* + nút **"Remind Requester"** (gửi thông báo nhắc lại).

### Case: Requester đã nộp invoice — cần xác minh
Hiện link file Requester đã nộp + câu hỏi **"Is this invoice correct?"**:
- **"Yes — proceed"** → cho phép tiếp tục trích xuất dữ liệu.
- **"No — request re-upload"** → hiện banner đỏ + nút **"Request Re-upload"** (xóa link cũ, gửi thông báo yêu cầu Requester nộp lại).

### Case: Invoice Deferred — Procurement tự upload
Upload file ở **"Upload Supplier Invoice *"**, sau đó dùng nút **"🔍 Extract Invoice"** để chạy AI (Parse_Invoice flow), tự điền các trường: Invoice Date, Supplier Name, Billed To, Invoice Number, ABN, Attention, Description, Total Amount, GST/Tax, Currency. Nếu AI không trích được, phải điền tay.

Hệ thống cũng gợi ý **Suggested Filename** kèm Confidence Score.

### Trường bắt buộc trước khi Submit
**Assign to Accounting Staff*** — chọn nhân viên kế toán xử lý tiếp.

### Lỗi thường gặp
- "Please assign an Accounting Staff member before submitting."
- "Please provide a valid SharePoint link (maxbiocare)."
- "Please click 'Extract Invoice' to extract invoice data before submitting."
- "Some invoice fields could not be extracted. Please fill them in manually before submitting."

### Kết quả Submit
Cập nhật `InvoiceSubmitted = true`, chuyển trạng thái sang **"Pending Accounting"** (hoặc "Pending Supplier Follow-up" nếu đang chờ Round 2 SFU), lưu log bước 6 và gọi flow Submit_Invoice.

---

## 8. RequesterInvoiceScreen — Requester nộp/nộp lại invoice

**Ai dùng:** Requester, khi cần nộp invoice do được chỉ định "ViaRequester" hoặc do bị yêu cầu nộp lại.

### Case: Đã có invoice và đã được Procurement xử lý
Hiện thông báo xanh: *"Invoice has been processed by Procurement — no further changes allowed."* → không cho sửa, nút chỉ còn **"Back to Home"**.

### Case: Đã có invoice nhưng chưa xử lý (còn sửa được)
Hiện link file hiện tại + nút **"Remove Invoice"** để xóa và nộp lại.

### Case: Chưa có invoice hoặc bị yêu cầu nộp lại
Hiện **"Upload Corrected Supplier Invoice *"** — bắt buộc chọn 1 file, bấm **"Submit Invoice →"**.

### Kết quả
Lưu link file vào RequesterInvoiceURL, gọi flow thông báo Procurement (Procurement_Notify_Invoice_Provided), báo *"Invoice submitted. Procurement has been notified."*, quay về Home.

---

## 9. AccountingScreen — Kế toán xử lý (Accounting)

**Ai dùng:** Accounting, khi trạng thái "Pending Accounting".

### Thông tin tham chiếu
- **Official Invoice Reference** — link invoice chính thức
- **Supplier Selection / Negotiation Summary** — tóm tắt từ bước Procurement
- **Credit Note** (nền xanh lá, chỉ hiện nếu có) — nếu request này từng có Supplier Follow-up phát sinh Credit Note.

### Accounting Checklist * (bắt buộc tick hết 5 mục)
- "Invoice amount matches approved procurement request"
- "VAT and tax compliance verified"
- "Payment terms and due date confirmed"
- "Invoice recorded in accounting system"
- "Supporting documents complete and filed"

### Trường nhập
**Accounting Notes*** — bắt buộc.

### Kết quả Submit
Luôn chuyển trạng thái sang **"Completed"** — đây là màn hình duy nhất trong app đặt trạng thái này. Thông báo: *"Accounting handover completed. Request is now Completed."*

---

## 10. GoodsReceiptScreen — Nhận hàng & xác nhận

**Ai dùng:** Requester (mặc định) hoặc người được Requester chỉ định (assignee), khi trạng thái "Goods Receipt & Acceptance".

### Case: Phân công người nhận hàng (chỉ hiện nếu chưa có log Round 1)
Hai nút chuyển đổi:
- **"I will receive"** — Requester tự nhận, mở checklist bên dưới.
- **"Assign to someone else"** — hiện **"Assign to *"** (chọn nhân viên) + nút **"Save Assignment"**. Danh sách chọn **chỉ hiện nhân viên đã có trong `RM User`** (đã được cấp quyền dùng app) — vì app hiện giới hạn truy cập theo `RM User`, không thể giao việc cho người chưa có quyền vào app.
  - Khi lưu: gửi thông báo (email + Teams Adaptive Card) tới người được giao qua flow `Procurement_Notify_Receipt_Assignee` (notificationType = "GoodsReceipt").

### Case: Người được giao bị hủy phân công (assignee đang làm mà bị đổi)
Hệ thống cảnh báo: *"This assignment has been reassigned to someone else. Your submission has been cancelled."* → quay về Home. (Đây là edge case quan trọng cần lưu ý trong guide.)

### Case: Requester nhận lại việc (bỏ assignee cũ)
Bấm "I will receive" khi đang có người được giao → gọi flow với notificationType = "Unassigned" (chỉ gửi email, không Teams card) để báo người cũ không cần làm nữa.

### Goods Receipt Checklist * (bắt buộc tick hết 5 mục, chỉ hiện khi "I will receive")
- "Items physically received"
- "Items checked against approved order / delivery documents"
- "Quantity and condition verified"
- "Requester acceptance confirmed"
- "Damaged or incorrect items reported to Procurement (if applicable)"

### Bảng nhận từng nguyên liệu (Raw Materials, bắt buộc)
Danh sách raw material của yêu cầu (Trade Name kèm dòng phụ nhỏ màu xám hiện Category · Supplier, Unit, Qty đã đặt). Mỗi dòng hiển thị 2 dòng nhập liệu:
- Dòng 1: **Received Qty** (số lượng thực nhận), **Batch Number** (bắt buộc nếu Received Qty > 0), **Expiry Date** (bắt buộc nếu Received Qty > 0)
- Dòng 2: **QC Number** (bắt buộc nếu Received Qty > 0), **RM/PK Code** (bắt buộc nếu Received Qty > 0), **Link to COA** (bắt buộc nếu Received Qty > 0)

Lỗi thường gặp: "Please enter received quantity for at least one material", "Please enter batch number for all received materials", "Please enter expiry date for all received materials", "Please enter QC number for all received materials", "Please enter RM/PK code for all received materials", "Please enter Link to COA for all received materials".

### Trường nhập
- **Receipt Date*** — chọn ngày
- **Receipt Status*** — dropdown
- **Acceptance Decision*** — dropdown: có thể là "Accepted" / "Rejected" / "Requires Supplier Follow-up"
- **Remarks*** — bắt buộc
- **Receipt Photos*** — bắt buộc ít nhất 1 ảnh (tối đa 5)

### Case theo Acceptance Decision
| Quyết định | Trạng thái tiếp theo |
|---|---|
| Accepted | "Pending Invoice" (nếu invoice chưa nộp) hoặc "Pending Accounting" (nếu đã nộp) |
| Rejected | Rejected |
| Requires Supplier Follow-up | Pending Supplier Follow-up |

---

## 11. SupplierFollowUpScreen — Theo dõi giao hàng lại với nhà cung cấp

**Khi nào xuất hiện:** Trạng thái "Pending Supplier Follow-up" — 2 vòng (Round/Step) riêng biệt.

### Tham chiếu Round 1 (đọc lại từ Goods Receipt)
Hiện lại: Received By, Receipt Date, Receipt Status, Acceptance Decision, Remarks của lần nhận hàng đầu.

### Round 2 — Bước 1: Requester nhận hàng lần 2
- Cách phân công giống GoodsReceiptScreen: **"I will receive"** / **"Assign to someone else"** + **"Save Assignment"** (gọi flow với notificationType = "SupplierFollowUp"). Danh sách chọn cũng chỉ hiện nhân viên đã có trong `RM User`.
- Checklist * giống Goods Receipt (5 mục).
- **Bảng nhận từng nguyên liệu (bắt buộc)** — giống GoodsReceiptScreen: mỗi dòng raw material nhập lại **Received Qty**, **Batch Number**, **Expiry Date**, **QC Number**, **RM/PK Code**, **Link to COA** (5 trường sau đều bắt buộc nếu Received Qty > 0).
- Trường: **Receipt Date***, **Receipt Status*** (theo danh mục FollowUpReceiptStatus riêng), **Acceptance Decision*** (FollowUpAcceptanceDecision — ví dụ "Accepted" / "Accepted with Adjustment"), **Remarks***, **Round 2 Receipt Photos*** (bắt buộc).

### Case theo Acceptance Decision Round 2
- **"Accepted"** → trạng thái chuyển sang **"Pending Invoice"** (nếu invoice chưa nộp) hoặc **"Pending Accounting"** (nếu đã nộp) — **không** đi thẳng đến Completed.
- **"Accepted with Adjustment"** → trạng thái giữ "Pending Supplier Follow-up", chuyển tiếp sang Bước 2 (Procurement) kèm yêu cầu **Upload Credit Note ***.

### Round 2 — Bước 2: Procurement hoàn tất theo dõi (chỉ hiện khi Bước 1 xong và có Adjustment)
- **Upload Credit Note *** — bắt buộc.
- **Remarks*** — mô tả kết quả làm việc với nhà cung cấp.
- Sau khi Submit: trạng thái chuyển sang **"Pending Invoice"** hoặc **"Pending Accounting"** (tùy invoice đã nộp hay chưa); Credit Note được lưu và hiển thị lại ở AccountingScreen (mục 9).

### Case: Đã hoàn tất theo dõi
Nếu log Bước 2 đã tồn tại, hiện banner xanh **"Supplier Follow-up Complete"** kèm ngày hoàn tất & người thực hiện.

---

## 12. RequestDetailScreen — Xem chi tiết yêu cầu

**Ai dùng:** Mọi vai trò, khi bấm vào 1 request từ Home (trừ trường hợp Procurement mở request "Pending Invoice" sẽ vào InvoiceSubmissionScreen thẳng).

Hiển thị toàn bộ dữ liệu của yêu cầu ở dạng chỉ xem:
- Thông tin cơ bản: Requester, Procurement Type, Purchase Accordance, Cost Center (luôn "Port Melbourne Warehouse"), Delivery Location (luôn "Port Melbourne Warehouse"), Required Delivery Date, Estimated Cost, Budget Reference.
- **Related Product SKU** — danh sách SKU sản phẩm liên quan (nếu có chọn khi tạo yêu cầu); hiện "—" nếu không chọn SKU nào.
- **Raw Materials** — bảng danh sách nguyên liệu của yêu cầu (Trade Name, Category, Supplier, Unit, Qty), chỉ xem.
- **Escalated to Executive**: "Yes"/"No" — chỉ có ý nghĩa với yêu cầu cũ (trước khi đổi sang luôn 2 cấp duyệt); yêu cầu mới luôn là "No".
- Invoice Type (màu xanh nếu "Official Invoice", cam nếu "Proforma Invoice").
- Link Invoice của Requester (nếu invoice qua Requester) và Official Invoice Link (nếu Procurement xử lý) — bấm để mở file.
- Approval Conditions (Executive) — nếu có điều kiện phê duyệt.
- Accounting Handler — người kế toán được assign.
- **Goods Receipt & Acceptance — Round 1**: người nhận, ngày nhận, trạng thái, quyết định, remarks, ảnh đính kèm, người được assign.
- **Goods Receipt & Acceptance — Round 2**: tương tự, hiện khi có dữ liệu Follow-up.

Đây là màn hình trung gian để người dùng bấm nút chuyển tiếp đến màn hình hành động phù hợp theo vai trò và trạng thái hiện tại (Manager Review, Executive Approval, Procurement Execution, Accounting, Goods Receipt, Supplier Follow-up).

---

## 13. Bảng màu trạng thái (để nhận biết nhanh trên Home/Detail)

| Trạng thái | Màu badge |
|---|---|
| Completed | Xanh lá |
| Rejected | Đỏ |
| Pending Manager / Pending Executive / Pending Procurement / Pending Accounting | Cam |
| Goods Receipt & Acceptance | Tím / xám tùy màn hình |
| Khác | Xám |

---

## 14. Các lỗi/thông báo phổ biến cần biết khi hướng dẫn người dùng

- "Please fill in all required fields" — thiếu trường bắt buộc.
- "Please complete all checklist items" — chưa tick hết checklist.
- "[Field] is required" — trường cụ thể còn trống.
- "Failed to [action]. Please try again." — lỗi hệ thống khi lưu, thử lại.
- "[Action] submitted successfully" — thao tác thành công.
- Với các bước có **assignment** (Goods Receipt, Supplier Follow-up Round 1): nếu bị đổi người trong lúc đang làm, dữ liệu đang nhập sẽ bị hủy — cần lưu ý người dùng nên hoàn tất nhanh sau khi được giao.
