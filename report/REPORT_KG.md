# Báo cáo Day 19 — Flat RAG so với GraphRAG

**Họ tên:** Nguyễn Văn Duy · **MSSV:** 2A202602729 · **Ngày:** 05/10/2026

Kết quả dưới đây lấy từ lần chạy thật `python bench_kg.py --judge` cuối cùng, lưu nguyên văn tại `ket_qua_benchmark_kg.txt`: Gemini 3.5 Flash-Lite cho chat/judge, Gemini Embedding 2 cho vector, `top_k=3`, đoạn 800 ký tự, 176 chunk, 203 node và 384 cạnh. Hai pipeline dùng cùng model, cùng dữ liệu, cùng top-k.

## 1. Chi phí

Hai bảng sau sao chép từ file benchmark cuối:

~~~text
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     37435        0   0.00749    133.5
graph       196     72054     5730   0.03220    224.1

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.58   1.17      692       71   0.00038     4.24
graph       1.00   2.00     5052      136   0.00185     4.92
~~~

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | ---: | ---: | ---: |
| Dựng chỉ mục, USD | 0,00749 | 0,03220 | 4,30× |
| Dựng chỉ mục, giây | 133,5 | 224,1 | 1,68× |
| Mỗi câu, USD | 0,00038 | 0,00185 | 4,87× |
| Mỗi câu, giây | 4,24 | 4,92 | 1,16× |
| Mỗi câu, input token | 692 | 5.052 | 7,30× |

Graph tốn thêm 20 lần gọi LLM để trích vụ việc từ tin tức khi dựng chỉ mục. Khi hỏi, prompt phải chứa cả chunk và các dữ kiện mở rộng nhiều bước, làm input token tăng mạnh. Giá USD là **ước tính tương đương bậc trả phí** theo `src/llm.py`; tài khoản Gemini hiện chạy trong quota miễn phí. Gemini không trả usage cho embedding qua endpoint tương thích OpenAI, nên số input token embedding trong code được **ước tính bằng UTF-8 bytes/4**. Năm mẫu văn bản của corpus được kiểm bằng Gemini countTokens cho tỉ lệ 3,56–4,47 bytes/token. Vì vậy USD indexing là ước tính, không phải hóa đơn thực tế.

Graph đắt hơn cả khi dựng lẫn khi hỏi, nên **không có điểm hòa vốn về tiền thuần** nếu chỉ cộng phí API. Với giả định 100 câu có chi phí trung bình như sáu câu thử, tổng Flat khoảng 0,04549 USD, Graph khoảng 0,21720 USD; chênh khoảng 0,17171 USD. Lợi ích cần cân bằng khoản chênh đó là khả năng trả lời câu hỏi xuyên hai KB.

## 2. Từng câu hỏi

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Lý do |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1,00 / 2 | 1,00 / 2 | Hòa | Định nghĩa tiền chất đã nằm trong một đoạn luật |
| Q2 | single-hop-news | 1,00 / 2 | 1,00 / 2 | Hòa | Hai tên bị cáo cùng ở một bài tin |
| Q3 | cross-kb | 0,00 / 0 | 1,00 / 2 | Graph | Đường Person → Case → Crime ← Article đưa Điều 251 vào ngữ cảnh |
| Q4 | cross-kb | 0,33 / 1 | 1,00 / 2 | Graph | Truy vấn graph mở các khoản của Điều 255 để thấy khung tối đa |
| Q5 | cross-kb-multi-hop | 0,80 / 1 | 1,00 / 2 | Graph | Case gắn MDMA, Crime dẫn tới Điều 250/khoản 4 |
| Q6 | aggregation | 0,33 / 1 | 1,00 / 2 | Graph | Substance MDMA nối tới nhiều Case thay vì chỉ một chunk |

Ba câu xuyên KB Q3–Q5 có recall trung bình khoảng **0,38 Flat** so với **1,00 Graph** và judge **0,67/2 Flat** so với **2,00/2 Graph**. Với Q1–Q2 một nguồn, hai pipeline cùng đạt điểm tối đa: đồ thị chỉ tăng chi phí.

## 3. Phân tích lỗi

### E3 — Một sự kiện thành nhiều Case

- **Hiện tượng:** ba bài báo về cùng đợt triệt phá tám đường dây liên quan Hoàng Nato tạo ba node `Case` khác nhau.
- **Bằng chứng:** truy vấn và ba dòng trả về từ Neo4j sau lần benchmark cuối:

~~~cypher
MATCH (k:Case)
WHERE k.name CONTAINS '8 đường dây'
RETURN k.name AS name, k.doc_id AS doc_id
ORDER BY doc_id;
~~~

~~~text
Vụ triệt phá 8 đường dây ma túy liên quan 'Hoàng Nato' tại TP.HCM
  news-100260920221957595
Vụ triệt phá 8 đường dây ma túy liên quan TikToker Phannhibeauty và 'Hoàng Nato' tại TP.HCM
  news-100260922111804786
Vụ triệt phá 8 đường dây ma túy liên quan đến ‘Hoàng Nato’ tại TP.HCM
  news-100260925144412498
~~~

- **Nguyên nhân:** hàm `add_news_case` dùng `MERGE (k:Case {name: $name})`, trong khi `name` do LLM tóm tắt khác nhau ở mỗi bài. Khóa này không đại diện ổn định cho sự kiện thực.
- **Đề xuất sửa:** tạo node `CaseMention` theo `doc_id + vị trí trong bài`, sau đó liên kết đến `Event` chuẩn bằng ngày, địa điểm, người, loại vụ và đối chiếu nguồn. Chỉ gộp khi đủ bằng chứng; trường hợp mơ hồ để người xem duyệt. Đánh đổi: thêm node, truy vấn và bước entity resolution.

### E4 — Điểm judge tối đa che khuất thông tin thừa và lời đáp thiếu nhất quán

- **Hiện tượng:** Q6 GraphRAG được `recall=1,00`, `judge=2/2` nhưng câu trả lời liệt kê riêng “bãi biển Sầm Sơn và Viện Pháp y tâm thần” bên cạnh vụ Viện Pháp y, trong khi đáp án chuẩn chỉ mô tả ba nhóm vụ Huy, Thành, Viện Pháp y. Q3 GraphRAG cũng được `judge=2/2` dù vừa nói “không đủ thông tin ... Điều luật” rồi nêu chính Điều 251.
- **Bằng chứng:** nguyên văn hai câu trả lời trong `ket_qua_benchmark_kg.txt`:

~~~text
--- Q3 [cross-kb] graph recall=1.00 judge=2 6.10s
Dựa trên ngữ cảnh và dữ kiện từ knowledge graph:

- **Số tháng tù:** Lê Minh Thành bị tuyên **36 tháng tù**.
- **Về tội:** Tội mua bán trái phép chất ma túy.
- **Quy định tại điều luật:** Không đủ thông tin trong ngữ cảnh chỉ rõ Điều luật quy định riêng cho tội mua bán trái phép chất ma túy của Lê Minh Thành, tuy nhiên ngữ cảnh có cung cấp **Điều 251 BLHS** quy định về "Tội mua bán trái phép chất ma túy".
- **Khung hình phạt cơ bản:** Theo khoản 1 **Điều 251 BLHS**, khung hình phạt cơ bản đối với tội mua bán trái phép chất ma túy là **phạt tù từ 02 năm đến 07 năm**.

--- Q6 [aggregation] graph recall=1.00 judge=2 4.89s
Dựa trên ngữ cảnh và dữ kiện từ knowledge graph, các vụ việc liên quan đến ma túy MDMA bao gồm:

1. **Vụ tổ chức sử dụng trái phép chất ma túy tại bãi biển Sầm Sơn và Viện Pháp y tâm thần Trung ương** (liên quan 0,686g MDMA).
2. **Vụ án sai phạm tại Viện Pháp y tâm thần Trung ương** (liên quan đến MDMA).
3. **Vụ vận chuyển ma túy qua sân bay Nội Bài liên quan đến Cái Quang Huy** / **Vụ vận chuyển hơn 10kg ma túy từ Đức về Việt Nam qua sân bay Nội Bài** (liên quan hơn 9,6kg MDMA).
4. **Vụ mua bán trái phép chất ma túy liên quan đến Lê Minh Thành và đồng phạm** (liên quan 5 viên MDMA).
~~~

- **Nguyên nhân:** `must_include` chỉ đếm từ khóa bắt buộc, không phạt thông tin thừa/trùng. Judge chỉ nhìn câu hỏi, gold và câu trả lời, không nhìn nguồn hay graph để quyết định mục thừa có thật hay do trích xuất sai. Từ kết quả này **chưa thể kết luận** mục Sầm Sơn sai; chỉ có thể kết luận phép đo hiện chưa phân xử được.
- **Đề xuất sửa:** lập danh sách sự kiện chuẩn có ID và nguồn, kiểm tra tính đầy đủ **và** độ chính xác của từng mục trong câu tổng hợp; đưa source span/đường Cypher cho judge, thêm luật phát hiện câu tự mâu thuẫn. Đánh đổi: tốn thêm công gán gold và token chấm.

## 4. Kết luận

GraphRAG đáng dùng khi câu hỏi cần nối người/vụ trong tin với Điều/khoản của luật (Q3–Q5) hoặc tổng hợp nhiều vụ theo cùng một chất (Q6): recall tăng tới 1,00 và judge đạt 2/2 ở cả bốn câu này. Với định nghĩa hoặc chi tiết nằm gọn một văn bản (Q1–Q2), Flat RAG đã đạt 1,00 và 2/2; GraphRAG tăng chi phí truy vấn khoảng 4,87 lần mà không tăng điểm.

Kết quả là benchmark kỹ thuật trên corpus và đáp án chuẩn của lab, **không phải tư vấn pháp lý**. Vì E3/E4, điểm judge cao không thay thế kiểm tra nguồn và entity resolution.

## 5. Tự kiểm

~~~text
$ .venv/Scripts/python -m pytest tests/ -q
................................................                         [100%]
48 passed in 0.11s

$ .venv/Scripts/python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-2
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 23 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00250. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
~~~

`--check` cố ý chỉ dựng graph luật + một bài tin; vì vậy số node của lần check không trùng với benchmark đầy đủ.

Ảnh Neo4j thật trong `report/img/`: `kg_count.png`, `kg_cross_kb.png`, `kg_my_case.png`. Người tự chọn cho ảnh thứ ba là **Cái Quang Huy**.

## Vấn đề gặp phải

- Model mặc định Gemini 2.5 Flash-Lite trong repo đề trả 404 với tài khoản hiện tại. Chuyển sang Gemini 3.5 Flash-Lite và cập nhật bảng giá trong `src/llm.py`.
- Gemini miễn phí giới hạn 15 chat và 100 embedding requests/phút; thêm pacing và retry có giới hạn trong `src/llm.py`. Lần benchmark thất bại bị loại; file kết quả là từ lần chạy cuối thành công.
- API embedding tương thích OpenAI bỏ trống usage; dùng ước lượng token có kiểm tra mẫu như mục 1, không trình bày USD là số tiền đã bị trừ.
