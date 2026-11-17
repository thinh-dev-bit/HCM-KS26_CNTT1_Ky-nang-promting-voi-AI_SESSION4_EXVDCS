# BÁO CÁO THIẾT KẾ GIẢI PHÁP PROMPTING NÂNG CAO (LOGIC & CHUỖI)

---

## PHẦN 1: BÁO CÁO PHÂN TÍCH & THIẾT KẾ GIẢI PHÁP (BƯỚC 1)

### 1.1. Bối cảnh & Vấn đề Thực tế tại Cửa hàng
Tại cửa hàng bán lẻ thiết bị điện tử & gia dụng **TechHome**, mỗi ngày bộ phận Chăm sóc Khách hàng (CSKH) tiếp nhận hàng chục yêu cầu đổi trả, bảo hành và khiếu nại hoàn tiền. Nếu xử lý thủ công hoặc dùng một **Mega Prompt** duy nhất để yêu cầu AI vừa đọc tin nhắn, vừa tra cứu chính sách, tính tiền hoàn và soạn email trả lời, hệ thống sẽ gặp 3 lỗi nghiêm trọng:

1. **Ảo giác tính toán đa biến số:** AI phản xạ tức thì, tính sai số tiền hoàn do nhầm lẫn giữa giá niêm yết, mã giảm giá, phí vận chuyển và tỷ lệ khấu trừ.
2. **Lỗi lan truyền & bỏ sót dữ liệu rỗng:** Khi khách hàng gửi thiếu mã đơn, thiếu ngày nhận hàng hoặc hệ thống ERP xuất sai số âm, AI tự suy diễn (bịa dữ liệu) dẫn đến cam kết sai chính sách.
3. **Đứt gãy ngữ cảnh & trôi dạt quy tắc:** Khi xuất bảng tổng hợp đối soát cuối ngày (nhiều dòng dữ liệu) hoặc vận hành liên tục qua hàng chục lượt chat trong ca trực, AI bị ngắt giữa chừng do chạm giới hạn **Max Output Tokens** hoặc quên dần quy tắc cửa hàng (**Context Drift**).

### 1.2. Kiến trúc Giải pháp Kết hợp 3 Kỹ thuật Cốt lõi
Giải pháp chia quy trình thành một **Dây chuyền Prompt Chaining 3 Trạm**, tích hợp **Chain-of-Thought (CoT)** tại trạm tính toán và bộ quy tắc **Quản lý Ngữ cảnh** cho vận hành dài hạn:

| Trạm / Cơ chế | Kỹ thuật áp dụng | Vai trò & Tối ưu vận hành (Model Routing & Gates) |
| :--- | :--- | :--- |
| **Trạm 1: Tiếp nhận & Làm sạch** | **Prompt Chaining** (Task Decomposition + Pure JSON Handoff) | Dùng mô hình tốc độ cao, chi phí thấp (*Gemini Flash / GPT-4o-mini*) để trích xuất thông tin thô thành JSON chuẩn, loại bỏ hoàn toàn câu chào xã giao (*Handoff rác*). |
| **Chốt chặn kiểm định** | **Validation Gate** (Điều kiện bắt lỗi) | Kiểm tra các trường bắt buộc (`null`, rỗng) và tính hợp lệ logic số học (giá $\ge 0$, giảm giá $\le$ giá bán) trước khi chuyển sang Trạm 2 để chặn **Cascading Errors**. |
| **Trạm 2: Thẩm định & Tính tiền** | **Chain-of-Thought (CoT)** (`reasoning_path` + `[RESULT]`) | Dùng mô hình suy luận mạnh (*Gemini Pro / GPT-4o / Claude 3.5 Sonnet*) để lập luận từng bước điều kiện bảo hành, tính toán từng khoản tiền hoàn và xuất thẻ kết luận chuẩn ở dòng cuối. |
| **Trạm 3: Phản hồi & Duyệt tay** | **Prompt Chaining** + **HITL** + **Context Anchoring** | Soạn thư phản hồi theo đúng bộ nhận diện cửa hàng; tự động gắn cờ **Human-in-the-Loop (HITL)** nếu tiền hoàn $> 2.000.000\text{ VNĐ}$ hoặc dữ liệu thiếu. |
| **Vận hành lô lớn & Ca dài** | **Continue Prompting**, **Rolling Summary**, **Session Reset** | Mớm 3–5 từ cuối khi bảng tổng hợp bị ngắt dòng; tóm tắt cuộn sau mỗi 8–10 lượt và làm mới phiên sau 30 lượt để chống trôi dạt ngữ cảnh. |

### 1.3. Xác định Chi tiết Đầu vào (Input) & Đầu ra (Output)

#### A. Dữ liệu Đầu vào (Input)
* **Luồng 1 – Nội dung khiếu nại của khách hàng (Unstructured Text):** Tin nhắn/email mô tả lỗi sản phẩm, nguyện vọng xử lý (đổi mới/hoàn tiền), trạng thái đính kèm hình ảnh/video minh chứng.
* **Luồng 2 – Dữ liệu đơn hàng từ hệ thống bán hàng (Raw Order Data):** Mã đơn hàng (`order_id`), tên sản phẩm, ngày nhận hàng (`delivery_date`), ngày gửi khiếu nại (`claim_date`), giá niêm yết (`listed_price`), giá trị giảm giá (`discount_amount`), phí vận chuyển (`shipping_fee`), trạng thái tem niêm phong (`seal_intact`).
* **Luồng 3 – Bộ quy tắc nghiệp vụ cửa hàng (Business Rules):**
  * Thời hạn đổi/trả: $\le 7\text{ ngày}$ kể từ ngày nhận hàng.
  * Lỗi nhà sản xuất (`manufacturer`) & còn nguyên tem/hộp (`seal_intact = true`): Hoàn **100% giá thực thanh toán sản phẩm** + **100% phí vận chuyển**.
  * Lỗi nhà sản xuất nhưng mất tem/thiếu vỏ hộp (`seal_intact = false`): Khấu trừ **10%** trên giá thực thanh toán sản phẩm, hoàn **100% phí vận chuyển**.
  * Khách đổi ý (`customer_preference`) & còn nguyên seal: Khấu trừ **15%** phí xử lý trên giá thực thanh toán sản phẩm, **không hoàn** phí vận chuyển.
  * Ngưỡng kiểm duyệt thủ công (**HITL**): Tổng tiền hoàn $> 2.000.000\text{ VNĐ}$.

#### B. Cấu trúc Đầu ra (Output)
* **Output Trạm 1 (Data Handoff):** Duy nhất 1 khối **Pure JSON** chuẩn hóa, không kèm văn bản giải thích bên ngoài.
* **Output Trạm 2 (CoT Reasoning):**
  * Danh sách lập luận minh bạch từ `Bước 1` đến `Bước 5`.
  * Dòng cuối cùng bắt buộc tuân thủ định dạng máy đọc được: `[RESULT: <TRẠNG_THÁI> | REFUND_AMOUNT: <SỐ_TIỀN> VND | HITL: <TRUE/FALSE>]`.
* **Output Trạm 3 (Final Action):** Email/tin nhắn phản hồi khách hàng chuyên nghiệp hoặc Phiếu cảnh báo bổ sung thông tin / trình Quản lý duyệt.

### 1.4. Ràng buộc & Bẫy Dữ liệu Lường trước (Error Handling)

| Loại bẫy dữ liệu | Biểu hiện thực tế | Câu lệnh điều kiện xử lý lỗi trong Prompt |
| :--- | :--- | :--- |
| **1. Dữ liệu bị trống (`null` / rỗng)** | Khách quên ghi mã đơn; hệ thống chưa cập nhật ngày giao hàng (`delivery_date = null`). | Tuyệt đối không tự đoán hoặc bịa giá trị. Gán giá trị `null`, đưa tên trường thiếu vào mảng `missing_fields` và chuyển trạng thái sang `INCOMPLETE_DATA`. |
| **2. Dữ liệu số học sai lệch** | Giá trị giảm giá lớn hơn giá bán (`discount > price`), hoặc phí vận chuyển/giá bán bị âm (`< 0`). | Thiết lập chốt chặn logic: Nếu `listed_price <= 0`, `discount_amount < 0`, `discount_amount > listed_price` hoặc `shipping_fee < 0` $\rightarrow$ Dừng tính toán ngay lập tức, trả về lỗi `DATA_ANOMALY` để tránh ra số tiền hoàn âm hoặc vượt giá trị đơn. |
| **3. Mâu thuẫn mốc thời gian** | Ngày khiếu nại sớm hơn ngày nhận hàng (`claim_date < delivery_date`). | Bắt lỗi `INVALID_DATE_RANGE`, yêu cầu kiểm tra lại trạng thái vận đơn trước khi xét duyệt. |
| **4. Thiếu bằng chứng xác thực** | Khách báo lỗi kỹ thuật nhưng không gửi ảnh/video (`evidence_attached = false`). | Không từ chối ngay và cũng không duyệt chi ngay; chuyển luồng `PENDING_EVIDENCE` và sinh mẫu tin nhắn hướng dẫn khách bổ sung video. |
| **5. Ngụy biện chuỗi suy luận CoT** | AI tính đúng bước đầu nhưng cộng nhầm ở bước cuối hoặc chôn vùi đáp án trong văn bản. | Bắt buộc kiểm tra chéo công thức ở Bước 5 và khóa cứng định dạng dòng cuối `[RESULT: ...]`. |

---

## PHẦN 2: BỘ PROMPT THỰC THI HOÀN CHỈNH (BƯỚC 2A)

### 2.1. Prompt Trạm 1 – Trích xuất Dữ liệu & Chốt chặn Kiểm định (Data Handoff)

````markdown
### VAI TRÒ
Bạn là Bộ phân tích & Chuẩn hóa Dữ liệu (Data Parser) của hệ thống tự động hóa CSKH cửa hàng TechHome.

### NHIỆM VỤ
Đọc nội dung khiếu nại của khách hàng và dữ liệu đơn hàng thô được cung cấp trong thẻ `<input_data>`, trích xuất thông tin và thực hiện kiểm định tính hợp lệ dữ liệu.

### QUY TẮC XỬ LÝ LỖI & RÀNG BUỘC NGHIÊM NGẶT
1. **Không suy diễn dữ liệu trống:** Nếu thông tin nào không có hoặc bị trống, BẮT BUỘC điền giá trị `null` (không tự bịa mã đơn, ngày tháng hay con số).
2. **Kiểm tra dữ liệu thiếu (`missing_fields`):** Liệt kê tên các trường bắt buộc bị `null` hoặc rỗng trong danh sách: `["order_id", "delivery_date", "claim_date", "listed_price", "fault_type"]`.
3. **Kiểm tra sai lệch số học & logic (`anomalies`):**
   - Nếu `listed_price <= 0` hoặc `discount_amount < 0` hoặc `shipping_fee < 0`: thêm `"NEGATIVE_OR_ZERO_AMOUNT"`.
   - Nếu `discount_amount > listed_price`: thêm `"DISCOUNT_EXCEEDS_PRICE"`.
   - Nếu `claim_date < delivery_date`: thêm `"INVALID_DATE_ORDER"`.
4. **Xác định trạng thái chốt chặn (`validation_status`):**
   - Nếu `missing_fields` không rỗng -> `"INCOMPLETE_DATA"`.
   - Nếu `anomalies` không rỗng -> `"DATA_ANOMALY"`.
   - Nếu `evidence_attached == false` và `fault_type == "manufacturer"` -> `"PENDING_EVIDENCE"`.
   - Nếu tất cả hợp lệ -> `"VALID"`.

### ĐỊNH DẠNG ĐẦU RA (PURE JSON)
Chỉ trả về DUY NHẤT một khối JSON hợp lệ theo schema dưới đây. Tuyệt đối KHÔNG viết lời chào, không giải thích trước hay sau khối JSON.

{
  "order_id": "string | null",
  "customer_name": "string | null",
  "product_name": "string | null",
  "delivery_date": "YYYY-MM-DD | null",
  "claim_date": "YYYY-MM-DD | null",
  "listed_price": "number | null",
  "discount_amount": "number | null",
  "shipping_fee": "number | null",
  "seal_intact": "boolean | null",
  "user_damage": "boolean | null",
  "fault_type": "manufacturer | customer_preference | unknown | null",
  "evidence_attached": "boolean",
  "validation_status": "VALID | INCOMPLETE_DATA | DATA_ANOMALY | PENDING_EVIDENCE",
  "missing_fields": [],
  "anomalies": []
}

<input_data>
{{DAN_DU_LIEU_THO_TAI_DAY}}
</input_data>
````

---

### 2.2. Prompt Trạm 2 – Lập luận Logic & Tính toán Hoàn tiền (Chain-of-Thought)

````markdown
### VAI TRÒ
Bạn là Chuyên viên Thẩm định Hoàn tiền & Bảo hành cấp cao tại TechHome.

### NHIỆM VỤ
Nhận dữ liệu JSON từ Trạm 1 trong thẻ `<handoff_json>` và áp dụng bộ quy tắc nghiệp vụ để tính toán chính xác phương án xử lý.

### QUY TẮC NGHIỆP VỤ CỬA HÀNG TECHHOME
1. **Chốt chặn đầu vào:** Nếu `validation_status` khác `"VALID"`, DỪNG tính toán tiền ngay lập tức, nêu rõ mã lỗi và hướng xử lý tương ứng.
2. **Điều kiện thời hạn:** Số ngày sử dụng = `claim_date - delivery_date`. Nếu $> 7$ ngày $\rightarrow$ Từ chối hoàn tiền (`REJECTED_OVERDUE`), chuyển sang bảo hành sửa chữa.
3. **Công thức tính giá thực thanh toán sản phẩm:**
   $$\text{Actual\_Product\_Paid} = \text{listed\_price} - \text{discount\_amount}$$
4. **Tỷ lệ hoàn & khấu trừ:**
   - Trường hợp `fault_type == "manufacturer"` và `user_damage == false`:
     - Nếu `seal_intact == true`: Hoàn $100\%$ `Actual_Product_Paid` + $100\%$ `shipping_fee`.
     - Nếu `seal_intact == false`: Khấu trừ $10\%$ `Actual_Product_Paid` (hoàn $90\%$) + hoàn $100\%$ `shipping_fee`.
   - Trường hợp `fault_type == "customer_preference"` và `user_damage == false`:
     - Nếu `seal_intact == true`: Khấu trừ $15\%$ `Actual_Product_Paid` (hoàn $85\%$) + $0\text{ VNĐ}$ phí vận chuyển.
     - Nếu `seal_intact == false`: Từ chối hoàn tiền (`REJECTED_UNSEALED`).
   - Trường hợp `user_damage == true`: Từ chối hoàn tiền (`REJECTED_USER_DAMAGE`).
5. **Quy tắc phê duyệt (HITL):** Nếu tổng tiền hoàn $> 2.000.000\text{ VNĐ}$, đặt `HITL = TRUE`; ngược lại đặt `HITL = FALSE`.

### YÊU CẦU SUY LUẬN (CHAIN-OF-THOUGHT)
Hãy suy nghĩ từng bước một và trình bày rõ ràng theo đúng 5 bước sau trước khi kết luận:
- **Bước 1 (Kiểm tra Chốt chặn Validation):** Xác nhận `validation_status` từ JSON đầu vào.
- **Bước 2 (Kiểm tra Thời hạn đổi/trả):** Tính chính xác số ngày từ `delivery_date` đến `claim_date` và đối chiếu ngưỡng 7 ngày.
- **Bước 3 (Xác định Điều kiện ngoại quan & Nguyên nhân lỗi):** Đánh giá `fault_type`, `seal_intact`, `user_damage` để chọn công thức khấu trừ.
- **Bước 4 (Tính toán chi tiết từng khoản tiền):** Tính `Actual_Product_Paid`, khoản khấu trừ (nếu có), `shipping_fee` được hoàn và tổng cộng.
- **Bước 5 (Kiểm tra chéo & Xét ngưỡng HITL):** Đối chiếu lại phép cộng/trừ ở Bước 4 và kiểm tra ngưỡng $2.000.000\text{ VNĐ}$.

### ĐỊNH DẠNG KẾT LUẬN BẮT BUỘC Ở DÒNG CUỐI CÙNG
Sau 5 bước lập luận, dòng cuối cùng của phản hồi BẮT BUỘC phải viết đúng định dạng sau (không thêm ký tự nào phía sau):
`[RESULT: <APPROVED | REJECTED | HOLD_FOR_FIX> | REFUND_AMOUNT: <SO_NGUYEN> VND | HITL: <TRUE | FALSE>]`

<handoff_json>
{{DAN_JSON_TRAM_1_TAI_DAY}}
</handoff_json>
````

---

### 2.3. Prompt Trạm 3 – Soạn Phản hồi CSKH & Điều phối HITL (Context Anchoring)

````markdown
[Neo bối cảnh]: Bạn là Trưởng nhóm Chăm sóc Khách hàng tại TechHome. Giữ vững văn phong lịch sự, đồng cảm, minh bạch con số và tuyệt đối tuân thủ kết quả thẩm định từ hệ thống, không tự ý hứa hẹn ngoài chính sách.

### NHIỆM VỤ
Dựa trên dữ liệu JSON từ Trạm 1 và kết quả lập luận `[RESULT]` từ Trạm 2, hãy tạo phản hồi đầu ra gồm 2 phần:
1. **Thông báo nội bộ (Internal Action):**
   - Nếu `HITL: TRUE`: Xuất cảnh báo `[CẦN QUẢN LÝ DUYỆT CHI]` kèm lý do vượt hạn mức $2.000.000\text{ VNĐ}$.
   - Nếu trạng thái là `HOLD_FOR_FIX`: Xuất phiếu yêu cầu khách bổ sung thông tin/video hoặc báo bộ phận Kỹ thuật sửa dữ liệu ERP.
2. **Mẫu tin nhắn/email gửi khách hàng:** Giải thích rõ ràng từng khoản tiền (giá thực trả, phí ship, khấu trừ nếu có) hoặc hướng dẫn bước tiếp theo một cách nhẹ nhàng, chuyên nghiệp.

<station_1_json>
{{DAN_JSON_TRAM_1}}
</station_1_json>

<station_2_output>
{{DAN_KET_QUA_TRAM_2}}
</station_2_output>
````

---

### 2.4. Bộ Lệnh Xử lý Đứt gãy Ngữ cảnh & Duy trì Nhất quán (Vận hành Thực tế)

* **Tình huống A – Khi bảng báo cáo đối soát cuối ngày bị ngắt ngang do chạm trần `Max Output Tokens` (Continue Prompting):**

````markdown
Phản hồi trước của bạn bị ngắt ngang do giới hạn độ dài. Hãy viết tiếp ngay sau cụm từ: "{{COPY_3_DEN_5_TU_CUOI_TRUOC_DIEM_NGAT}}".
Yêu cầu bắt buộc:
1. Bắt đầu ngay dòng kế tiếp của bảng Markdown, giữ nguyên số lượng cột và căn lề.
2. Tuyệt đối không lặp lại tiêu đề bảng hay các dòng đã xuất trước đó.
3. Xuất nốt cho đến hết danh sách đơn hàng và ghi `[END_OF_REPORT]` ở dòng cuối.
````

* **Tình huống B – Khi phiên chat xử lý khiếu nại kéo dài qua 10–30 lượt (Rolling Summary & Session Reset):**

````markdown
[Rolling Summary]: Chúng ta đã xử lý 10 lượt khiếu nại trong ca làm việc. Hãy tóm tắt ngắn gọn dưới 150 từ thành gạch đầu dòng:
1. Các quy tắc tính hoàn tiền và ngưỡng HITL (> 2.000.000 VNĐ) của TechHome.
2. Danh sách mã đơn đang treo chờ bổ sung dữ liệu hoặc chờ Quản lý duyệt.
(Bản tóm tắt này sẽ được dùng để Reset sang phiên chat mới nhằm giải phóng Context Window).
````

---

## PHẦN 3: KẾT QUẢ CHẠY THỬ THỰC TẾ (BƯỚC 2B)

### 3.1. Test Case 1: Dữ liệu Hợp lệ Đa biến số (Kiểm chứng CoT + Prompt Chaining + HITL)

#### Dữ liệu Input đưa vào Trạm 1:
> **Tin nhắn khách hàng:** "Chào shop, mình là Nguyễn Minh Anh. Hôm 12/11/2026 mình có nhận chiếc Tai nghe chống ồn ProSound X2 (mã đơn DH-20261117-089). Hôm nay 17/11/2026 mình dùng thì phát hiện micro bị rè không thu tiếng. Hộp và tem niêm phong mình vẫn giữ nguyên vẹn, mình đã đính kèm video test mic bên dưới. Shop hỗ trợ hoàn tiền giúp mình nhé."
> **Dữ liệu hệ thống ERP:** `listed_price: 2500000`, `discount_amount: 250000` (mã giảm 10%), `shipping_fee: 30000`, `seal_intact: true`, `user_damage: false`, `evidence_attached: true`.

#### Kết quả phản hồi thực tế của AI – Trạm 1 (Pure JSON Handoff):
````json
{
  "order_id": "DH-20261117-089",
  "customer_name": "Nguyễn Minh Anh",
  "product_name": "Tai nghe chống ồn ProSound X2",
  "delivery_date": "2026-11-12",
  "claim_date": "2026-11-17",
  "listed_price": 2500000,
  "discount_amount": 250000,
  "shipping_fee": 30000,
  "seal_intact": true,
  "user_damage": false,
  "fault_type": "manufacturer",
  "evidence_attached": true,
  "validation_status": "VALID",
  "missing_fields": [],
  "anomalies": []
}
````

#### Kết quả phản hồi thực tế của AI – Trạm 2 (Chain-of-Thought Reasoning):
* **Bước 1 (Kiểm tra Chốt chặn Validation):** Trạng thái `validation_status` là `"VALID"`, không có trường dữ liệu bị trống (`missing_fields: []`) và không có sai lệch số học (`anomalies: []`). Đủ điều kiện tiếp tục thẩm định.
* **Bước 2 (Kiểm tra Thời hạn đổi/trả):** Khoảng cách từ ngày nhận hàng (`2026-11-12`) đến ngày gửi khiếu nại (`2026-11-17`) là $17 - 12 = 5\text{ ngày}$. Vì $5\text{ ngày} \le 7\text{ ngày}$ theo quy định TechHome $\rightarrow$ **Đạt điều kiện thời hạn**.
* **Bước 3 (Xác định Điều kiện ngoại quan & Nguyên nhân lỗi):** Sản phẩm gặp lỗi kỹ thuật từ nhà sản xuất (`fault_type: "manufacturer"`), không có tác động hư hại từ người dùng (`user_damage: false`) và còn nguyên hộp/tem niêm phong (`seal_intact: true`). Áp dụng chính sách hoàn $100\%$ giá thực thanh toán sản phẩm và $100\%$ phí vận chuyển.
* **Bước 4 (Tính toán chi tiết từng khoản tiền):**
  * Giá thực thanh toán sản phẩm: $\text{Actual\_Product\_Paid} = 2.500.000 - 250.000 = 2.250.000\text{ VNĐ}$.
  * Phí khấu trừ ngoại quan: $0\text{ VNĐ}$.
  * Phí vận chuyển được hoàn lại ($100\%$): $30.000\text{ VNĐ}$.
  * Tổng số tiền hoàn trả: $2.250.000 + 30.000 = 2.280.000\text{ VNĐ}$.
* **Bước 5 (Kiểm tra chéo & Xét ngưỡng HITL):** Kiểm tra lại phép tính: $2.250.000 + 30.000 = 2.280.000\text{ VNĐ}$ (khớp với tổng tiền thực tế khách đã thanh toán). Vì $2.280.000\text{ VNĐ} > 2.000.000\text{ VNĐ}$ (ngưỡng tự động), hệ thống kích hoạt cờ phê duyệt thủ công `HITL = TRUE`.

`[RESULT: APPROVED | REFUND_AMOUNT: 2280000 VND | HITL: TRUE]`

#### Kết quả phản hồi thực tế của AI – Trạm 3 (Điều phối HITL & Soạn Thư):
> **1. Thông báo nội bộ:**
> `[CẦN QUẢN LÝ DUYỆT CHI]` – Đơn hàng **DH-20261117-089** (Khách hàng: Nguyễn Minh Anh) đủ điều kiện hoàn tiền **2.280.000 VNĐ** do lỗi kỹ thuật nhà sản xuất. Yêu cầu Quản lý CSKH xác nhận phê duyệt trên hệ thống trước khi gửi phản hồi chính thức vì vượt hạn mức tự động 2.000.000 VNĐ.
>
> **2. Bản thảo gửi khách hàng (Chờ duyệt):**
> Chào bạn Nguyễn Minh Anh,
> TechHome rất xin lỗi về trải nghiệm chưa trọn vẹn với sản phẩm **Tai nghe chống ồn ProSound X2** thuộc đơn hàng **DH-20261117-089**. Qua kiểm tra video bạn cung cấp, bộ phận kỹ thuật xác nhận lỗi micro xuất phát từ nhà sản xuất và sản phẩm vẫn trong thời hạn 7 ngày đổi trả.
> TechHome hỗ trợ hoàn tiền **100%** cho đơn hàng của bạn với chi tiết như sau:
> * Giá trị sản phẩm thực thanh toán (sau ưu đãi 250.000 VNĐ): **2.250.000 VNĐ**
> * Phí vận chuyển hoàn lại: **30.000 VNĐ**
> * **Tổng số tiền hoàn trả:** **2.280.000 VNĐ**
>
> Yêu cầu của bạn đang được Trưởng bộ phận xác nhận lệnh chi và sẽ hoàn tất chuyển khoản trong vòng 24 giờ làm việc. Cảm ơn bạn đã tin tưởng TechHome!

---

### 3.2. Test Case 2: Bẫy Dữ liệu Trống & Sai lệch Số học (Kiểm chứng Validation Gate)

#### Dữ liệu Input đưa vào Trạm 1 (Chứa bẫy dữ liệu):
> **Tin nhắn khách hàng:** "Shop ơi cái máy xay sinh tố hôm bữa mình mua bị kẹt lưỡi dao rồi, xem hoàn tiền cho mình nhé." *(Không ghi mã đơn, không có ảnh/video đính kèm)*.
> **Dữ liệu hệ thống ERP lỗi:** `order_id: ""`, `delivery_date: null`, `claim_date: "2026-11-17"`, `listed_price: 850000`, `discount_amount: 900000` *(giảm giá lớn hơn giá bán)*, `shipping_fee: -15000` *(số âm)*, `evidence_attached: false`.

#### Kết quả phản hồi thực tế của AI – Trạm 1 (Pure JSON Handoff):
````json
{
  "order_id": null,
  "customer_name": null,
  "product_name": "Máy xay sinh tố",
  "delivery_date": null,
  "claim_date": "2026-11-17",
  "listed_price": 850000,
  "discount_amount": 900000,
  "shipping_fee": -15000,
  "seal_intact": null,
  "user_damage": null,
  "fault_type": "manufacturer",
  "evidence_attached": false,
  "validation_status": "INCOMPLETE_DATA",
  "missing_fields": [
    "order_id",
    "delivery_date"
  ],
  "anomalies": [
    "NEGATIVE_OR_ZERO_AMOUNT",
    "DISCOUNT_EXCEEDS_PRICE"
  ]
}
````

#### Kết quả phản hồi thực tế của AI – Trạm 2 (Kích hoạt Chốt chặn Dừng Tính Toán):
* **Bước 1 (Kiểm tra Chốt chặn Validation):** Phát hiện `validation_status` là `"INCOMPLETE_DATA"` kèm các cảnh báo nghiêm trọng:
  * Trường bị trống (`missing_fields`): `order_id`, `delivery_date`.
  * Dữ liệu số học bất thường (`anomalies`): `NEGATIVE_OR_ZERO_AMOUNT` (`shipping_fee = -15000`) và `DISCOUNT_EXCEEDS_PRICE` (`discount_amount = 900000 > listed_price = 850000`).
  * Thiếu bằng chứng xác thực lỗi (`evidence_attached: false`).
* **Bước 2 – Bước 5:** Dừng quy trình tính toán hoàn tiền theo Quy tắc nghiệp vụ số 1 để ngăn chặn ảo giác số học (ra tiền âm) và sai lệch chính sách thời hạn.

`[RESULT: HOLD_FOR_FIX | REFUND_AMOUNT: 0 VND | HITL: TRUE]`

---

### 3.3. Test Case 3: Khôi phục Bảng Tổng hợp Cuối ngày Bị Ngắt Ngang (`Continue Prompting`)

#### Tình huống mô phỏng sự cố chạm trần `Max Output Tokens`:
AI đang xuất bảng tổng hợp đối soát 20 đơn khiếu nại cuối ngày cho Kế toán thì bị ngắt đột ngột ở dòng thứ 18:

````markdown
| STT | Mã đơn hàng | Khách hàng | Tiền hoàn | Trạng thái |
| :--- | :--- | :--- | :--- | :--- |
| 17 | DH-20261117-017 | Lê Hoàng Nam | 320.000 VNĐ | Đã duyệt |
| 18 | DH-20261117-018 | Trần Quốc Bảo | 450.000 VNĐ | Đã duyệt |
````
*(Phản hồi dừng ngang tại đây)*

#### Câu lệnh thực thi khôi phục (Mớm đúng 4 từ cuối trước điểm ngắt):

````markdown
Phản hồi trước của bạn bị ngắt ngang do giới hạn độ dài. Hãy viết tiếp ngay sau cụm từ: "| 450.000 VNĐ | Đã duyệt |".
Yêu cầu bắt buộc:
1. Bắt đầu ngay dòng kế tiếp (STT 19) của bảng Markdown, giữ nguyên số lượng cột và căn lề.
2. Tuyệt đối không lặp lại tiêu đề bảng hay các dòng từ 1 đến 18.
3. Xuất nốt cho đến STT 20 và ghi `[END_OF_REPORT]` ở dòng cuối.
````

#### Kết quả phản hồi nối tiếp của AI (Liền mạch 100%, không vỡ cột):

````markdown
| 19 | DH-20261117-019 | Phạm Thu Hà | 0 VNĐ | Từ chối (Quá 7 ngày) |
| 20 | DH-20261117-020 | Đỗ Minh Quân | 2.280.000 VNĐ | Chờ Quản lý duyệt (HITL) |

`[END_OF_REPORT]`
````