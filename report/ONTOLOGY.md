# Thiết kế ontology — Day 19

**Họ tên:** Nguyễn Văn Duy · **MSSV:** 2A202602729

**Lựa chọn:** [x] Dùng ontology gợi ý. Bài này không nhận bonus tự thiết kế ontology.

## 1. Sơ đồ

~~~mermaid
flowchart LR
    P[Person] -- "INVOLVED_IN: role, sentence, charge" --> K[Case]
    K -- CHARGED_WITH --> C((Crime: cầu nối))
    K -- "INVOLVES: amount" --> S[Substance]
    K -- LOCATED_IN --> L[Location]
    A[Article] -- DEFINES --> C
    A -- HAS_CLAUSE --> CL[Clause]
    CL -- MENTIONS --> S
~~~

Tin tức sinh Case/Person/Location; luật sinh Article/Clause. Crime nối hai KB. Substance cũng dùng chung nhưng không phải cầu nối bắt buộc vì có tội không gắn với một chất cụ thể.

## 2. Entity types

| Label | Ý nghĩa | Khóa MERGE | Properties | KB / cách trích |
| --- | --- | --- | --- | --- |
| Article | Điều luật | id | id, title, law, doc_id | Luật: metadata và regex |
| Clause | Khoản luật | id | id, number, penalty, text, doc_id | Luật: regex |
| Crime | Tội danh chuẩn | name | name | Luật tạo chuẩn; tin được liên kết |
| Case | Vụ việc | name | name, summary, date, doc_id, source_title | Tin: LLM JSON |
| Person | Người trong vụ | name | name, aliases | Tin: LLM JSON |
| Substance | Chất | name | name | Luật: danh sách chuẩn; tin: LLM |
| Location | Địa điểm | name | name | Tin: LLM JSON |

Article, Clause và Case thuộc một tài liệu cụ thể nên mang doc_id. Crime, Person, Substance và Location được dùng chung giữa tài liệu nên không nhận một doc_id duy nhất; Case vẫn giữ doc_id bài báo nguồn.

## 3. Relationships

| Type | Từ → Đến | Properties trên cạnh | Ý nghĩa |
| --- | --- | --- | --- |
| DEFINES | Article → Crime | Không | Điều luật định nghĩa tội |
| HAS_CLAUSE | Article → Clause | Không | Điều luật gồm khoản |
| MENTIONS | Clause → Substance | Không | Khoản nhắc đến chất |
| CHARGED_WITH | Case → Crime | Không | Vụ gắn với tội danh |
| INVOLVES | Case → Substance | amount | Chất và khối lượng được nêu |
| LOCATED_IN | Case → Location | Không | Địa điểm vụ |
| INVOLVED_IN | Person → Case | role, sentence, charge | Vai trò, mức án, tội của người |

## 4. Node cầu nối giữa hai KB

- **Node:** Crime, khóa name là tên tội chuẩn từ tiêu đề Điều luật.
- **Lý do:** Case → Crime ← Article đi từ bài báo sang luật mà không đoán số Điều trực tiếp từ bài báo.
- **Khớp tên:** normalize_crime bỏ tiền tố “Tội”, chuẩn hóa chữ thường và khoảng trắng. link_entity khớp chính xác trước, sau đó mới so gần đúng ở ngưỡng 0,8. LLM được cấp danh sách tên chuẩn nhưng code vẫn kiểm tra kết quả.
- **Khi gãy:** LLM bỏ sót tội hoặc tên báo viết khác xa tên luật làm link_entity trả None. Không nối bừa; xem lại văn bản nguồn, thêm alias có bằng chứng hoặc sửa prompt rồi dựng lại graph.

## 5. Competency questions

| Câu | Đường đi trên graph | Mức trả lời |
| --- | --- | --- |
| Q1 tiền chất | Article thuộc Luật PCMT → HAS_CLAUSE → Clause.text chứa định nghĩa | Có, đọc text khoản |
| Q2 bị cáo tử hình | Person → INVOLVED_IN với sentence “tử hình” → Case liên quan | Có nếu LLM trích đủ người |
| Q3 Lê Minh Thành | Person → Case → Crime ← Article → Clause khoản 1; sentence trên INVOLVED_IN | Có khi Crime được liên kết |
| Q4 Hoàng Nato | Person.aliases → Case → Crime ← Article → các Clause để tìm mức tối đa | Có nếu alias được trích |
| Q5 Cái Quang Huy/MDMA | Person → Case → Substance MDMA ← Clause ← Article | Một phần: ngưỡng 100g còn là text, cần đối chiếu 9,6kg |
| Q6 các vụ MDMA | Substance MDMA ← INVOLVES ← Case ← INVOLVED_IN ← Person | Có với các vụ được trích đủ chất |

## 6. Quyết định thiết kế và đánh đổi

1. **Crime là cầu nối thay vì đoán Điều bằng từ khóa.** Tách được dữ kiện hai KB; đổi lại phụ thuộc chất lượng trích tội danh và link_entity.
2. **Luật dùng regex, tin dùng LLM.** Luật có cấu trúc đều nên rẻ và tái lập được; tin văn xuôi cần LLM nhưng có nguy cơ bỏ sót.
3. **Khối lượng và mức án là property của cạnh, không thành node.** Graph gọn nhưng chưa so sánh ngưỡng khối lượng bằng phép toán Cypher; Q5 cần đọc Clause.text.
4. **MERGE Person và Case theo tên.** Ít code nhưng có thể gộp nhầm người trùng tên hoặc tách đôi một vụ khi LLM đặt tên khác. doc_id trên Case hỗ trợ truy nguồn để phát hiện.

## 7. So với ontology gợi ý

Không xin bonus: các label và quan hệ giữ theo gợi ý. Lọc các khoản tối đa theo câu hỏi là logic truy vấn, không phải ontology mới.

## 8. Hạn chế còn lại

- Person và Case chưa có định danh ổn định ngoài tên; Substance chưa gộp đủ tên đồng nghĩa.
- Ngưỡng khối lượng và giai đoạn tố tụng chưa được mô hình hóa thành node/quan hệ riêng.
- LLM có thể bỏ sót người, chất hoặc tội. Cần đối chiếu bài báo và luật nguồn trước khi coi kết quả là kết luận pháp lý.
