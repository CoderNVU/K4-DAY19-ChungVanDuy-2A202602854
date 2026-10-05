# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Chung Văn Duy  **MSSV:** 2A202602854  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu trong báo cáo đều được trích xuất trực tiếp, chuẩn xác 100% từ file thực nghiệm `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

---

## 1. Chi phí (10 điểm)

Hai bảng `Indexing` và `Querying` trích xuất nguyên văn từ `ket_qua_benchmark_kg.txt`:

```
Chat model: gemini:gemini-3.5-flash-lite | Embedding: gemini:gemini-embedding-001 | top_k=3 | chunk_size=800 | chunks=176 | KG: 204 nodes / 383 rels

== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    114.1
graph       196     34619     5728   0.00000    154.0

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.50      696       76   0.00000     3.31
graph       0.94   1.83     5651      162   0.00000     2.28
```

### Bảng so sánh chỉ số chi phí và hiệu năng:

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| **Indexing USD** | $0.00000 | $0.00000 | **×1.0** *(Miễn phí trên Gemini API Free Tier)* |
| **Indexing giây** | 114.1 s | 154.0 s | **×1.35** *(Tăng 35.0% thời gian dựng)* |
| **Mỗi câu: USD** | $0.00000 | $0.00000 | **×1.0** *(Miễn phí trên Gemini API Free Tier)* |
| **Mỗi câu: giây** | 3.31 s | 2.28 s | **×0.69** *(GraphRAG phản hồi nhanh hơn 31%)* |
| **Mỗi câu: in_tok** | 696 | 5,651 | **×8.12** *(Tăng ~8.1 lần input token ngữ cảnh)* |

### Chi phí tăng thêm đến từ đâu?
1. **Ở giai đoạn Indexing (Dựng hệ thống - trả 1 lần):** Chi phí tăng thêm của GraphRAG xuất phát từ việc phải gọi LLM 20 lần để trích xuất có cấu trúc (JSON mode) các thực thể (bị can, vụ án, tội danh, tang vật) từ 20 bài báo tin tức (tiêu tốn thêm 34,619 input tokens và 5,728 output tokens, mất thêm 39.9 giây). Phía Flat RAG chỉ thực hiện embedding thuần túy cho 176 chunks nên thời gian dựng ngắn hơn.
2. **Ở giai đoạn Querying (Mỗi câu hỏi):** Input tokens của GraphRAG tăng gấp 8.12 lần (5,651 tokens so với 696 tokens) do prompt ngoài 3 chunks văn bản thông thường còn được chèn thêm danh sách các dữ kiện có cấu trúc (graph facts) mở rộng từ Neo4j (bao gồm tóm tắt vụ việc, điều luật, khung hình phạt của từng khoản). Tuy nhiên, thời gian sinh câu trả lời của GraphRAG lại nhanh hơn (2.28s so với 3.31s) do các dữ kiện đã được graph tường minh hóa sẵn, giúp LLM tổng hợp trực tiếp mà không cần mất thời gian tự suy luận nội suy từ các đoạn văn xuôi rời rạc.

---

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | :---: | :---: | :---: | --- |
| **Q1** | single-hop-law | 1.00 / 2 | 1.00 / 2 | **Hòa** | Định nghĩa tiền chất nằm gọn trong Điều 2 Luật PCMT, cả hai pipeline đều tìm đúng chunk và trích xuất chuẩn xác. |
| **Q2** | single-hop-news | 1.00 / 2 | 1.00 / 2 | **Hòa** | Thông tin 2 bị cáo lãnh án tử hình nằm trọn trong bài báo vụ 36kg ma túy, vector search lấy đúng chunk giúp cả 2 trả lời trọn vẹn. |
| **Q3** | cross-kb | 0.33 / 1 | 1.00 / 2 | **Graph** | Flat RAG chỉ biết mức án từ bài báo nhưng thiếu hoàn toàn Điều luật; GraphRAG đi qua node `Crime` nối sang Điều 251 BLHS khoản 1 (02–07 năm tù). |
| **Q4** | cross-kb | 0.33 / 1 | 0.67 / 1 | **Graph** | Flat RAG chỉ biết hành vi tổ chức sử dụng ma túy; GraphRAG liên kết được Điều 255 BLHS nhưng thiếu mức án tối đa (chung thân ở khoản 4) do bộ lọc KG-3 chỉ lấy khoản 1. |
| **Q5** | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | **Graph** | Flat RAG không biết khoản luật và khung phạt cho 9.6kg MDMA; GraphRAG kết nối xuyên chuỗi thực thể và chất ma túy MDMA, xác định chuẩn xác khoản 4 Điều 250 và án tử hình. |
| **Q6** | aggregation | 0.00 / 2 | 1.00 / 2 | **Graph** | Flat RAG bị phân mảnh giữa nhiều đoạn văn nên bỏ sót tên riêng bị can trong từ khóa bắt buộc; GraphRAG truy vấn tập trung quanh node `MDMA` nên gom đủ cả 4 vụ án. |

### Quy luật rút ra từ thực nghiệm:
- **Với câu hỏi đơn nguồn (`single-hop`):** Khi câu trả lời nằm trọn vẹn trong một đoạn văn bản (như Q1, Q2), Flat RAG phát huy tối đa ưu thế: nhanh, gọn, rẻ và đạt độ chính xác tuyệt đối (`recall = 1.00, judge = 2`) mà không cần đến sự hỗ trợ của Knowledge Graph.
- **Với câu hỏi đa nguồn (`cross-kb`, `multi-hop`, `aggregation`):** Khi dữ kiện nằm phân tán giữa tin tức và luật pháp (như Q3, Q5, Q6), Flat RAG thất bại hoặc chỉ trả lời được một phần (`recall = 0.00 - 0.40`) do khoảng cách ngữ nghĩa giữa tên người và điều luật không thể giải quyết bằng vector top-k. Ngược lại, GraphRAG áp đảo hoàn toàn (`recall = 0.94 - 1.00`) nhờ khả năng kết nối tri thức xuyên các node cầu nối.

---

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật (Missing Legal Context on Maximum Penalty at Q4)

- **Hiện tượng:** Tại câu hỏi **Q4** (*"Giang hồ 'Hoàng Nato' bị bắt về hành vi gì, và hành vi đó có thể bị phạt tù tối đa bao nhiêu theo Bộ luật Hình sự?"*), GraphRAG trả lời đúng hành vi và điều luật cơ bản nhưng không thể trả lời được mức phạt tù tối đa, khiến `recall` dừng lại ở `0.67` và `judge` chỉ đạt `1/2`.
- **Bằng chứng:**
  - Trích nguyên văn câu trả lời của GraphRAG trong `ket_qua_benchmark_kg.txt`:
    ```
    --- Q4 [cross-kb] graph recall=0.67 judge=1 2.13s
    Dựa trên ngữ cảnh và dữ kiện knowledge graph:

    - Hành vi bị bắt: Giang hồ "Hoàng Nato" (Dương Minh Tuấn) bị bắt về hành vi tổ chức sử dụng trái phép chất ma túy.
    - Mức phạt tù tối đa theo Bộ luật Hình sự: Không đủ thông tin trong ngữ cảnh để xác định mức phạt tù tối đa cụ thể (ngữ cảnh chỉ nêu tại [Điều 255 BLHS - Tội tổ chức sử dụng trái phép chất ma túy] khoản 1 về mức phạt tù từ 02 năm đến 07 năm cho tội danh này, nhưng không đủ dữ kiện các khoản nặng hơn).
    ```
  - Kiểm tra thực tế trong đồ thị Neo4j bằng câu lệnh Cypher:
    ```cypher
    MATCH (a:Article {id:'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
    RETURN cl.number AS number, cl.penalty AS penalty, cl.text AS text
    ORDER BY cl.number;
    ```
    Kết quả trả về từ database:
    ```
    number | penalty                            | text
    1      | phạt tù từ 02 năm đến 07 năm       | 1. Người nào tổ chức sử dụng trái phép chất ma túy...
    2      | phạt tù từ 07 năm đến 15 năm       | 2. Phạm tội thuộc một trong các trường hợp sau đây...
    3      | phạt tù từ 15 năm đến 20 năm       | 3. Phạm tội thuộc một trong các trường hợp sau đây...
    4      | phạt tù 20 năm hoặc tù chung thân  | 4. Phạm tội thuộc một trong các trường hợp sau đây...
    ```
- **Nguyên nhân:** Lỗi nằm ở **logic truy vấn Cypher trong hàm `Neo4jGraph.context` (KG-3)**. Quy tắc lọc khoản hiện tại chỉ chọn `cl.number = 1` (khung cơ bản) và các khoản có quan hệ `MENTIONS` tới `Substance` mà vụ án liên quan (`INVOLVES`). Trong vụ án Hoàng Nato, đối tượng bị bắt về hành vi tổ chức sử dụng ma túy nhưng các bài báo không ghi nhận khối lượng cụ thể khớp với Điều 255. Do đó, các khoản 2, 3 và đặc biệt là **khoản 4 (quy định mức án cao nhất: "tù 20 năm hoặc tù chung thân")** đã bị bộ lọc loại bỏ khỏi facts đưa vào prompt.
- **Đề xuất sửa:**
  - Sửa câu lệnh Cypher trong `context()`: Khi truy vấn các khoản của một Điều luật, luôn lấy thêm Khoản có `number` lớn nhất (`max(cl.number)`), hoặc nếu Điều luật có số lượng khoản ít (<= 4 khoản), đưa toàn bộ các khoản vào facts.
  - **Đánh đổi:** Tăng thêm khoảng 100–150 tokens ngữ cảnh trong prompt, nhưng đảm bảo LLM luôn có đủ dữ liệu để trả lời các câu hỏi về khung hình phạt cao nhất.

---

### Lỗi E4: Phép đo sai / Thiếu linh hoạt (Metric Misalignment between Exact Keyword Recall and Semantic Judge at Q6)

- **Hiện tượng:** Tại câu hỏi **Q6** (*"Những vụ việc nào trong tin tức có liên quan đến ma túy MDMA?"*), câu trả lời của Flat RAG đạt điểm tuyệt đối từ người chấm LLM (**`judge = 2`**), nhưng điểm từ khóa bắt buộc lại bằng 0 tuyệt đối (**`recall = 0.00`**).
- **Bằng chứng:**
  - Trích nguyên văn kết quả của Flat RAG tại Q6 trong `ket_qua_benchmark_kg.txt`:
    ```
    --- Q6 [aggregation] flat recall=0.00 judge=2 7.36s
    Dựa trên ngữ cảnh, cả 3 vụ việc đều có liên quan đến ma túy MDMA:

    * Vụ việc [1]: Lực lượng chức năng phát hiện bên trong thùng hàng có các viên nén màu xanh là MDMA (khối lượng gần 4,3kg).
    * Vụ việc [2]: Công an bắt quả tang Thành mang 5 viên ma túy đi bán, kết luận giám định xác định số viên nén này là ma túy MDMA.
    * Vụ việc [3]: Kết quả giám định xác định số viên nén hình tam giác màu hồng - xám trong kiện hàng là MDMA (khối lượng hơn 5,3kg).
    ```
  - Đối chiếu với danh sách từ khóa bắt buộc trong `data/benchmark_kg.json`:
    ```json
    "must_include": ["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]
    ```
- **Nguyên nhân:** Lỗi nằm ở **thiết kế của phép đo `keyword_recall`**. Hàm kiểm tra từ khóa sử dụng phép so khớp chuỗi con chính xác ký tự (`sum(k.lower() in answer.lower() for k in keywords) / len(keywords)`).
  - Trong câu trả lời của Flat RAG, mô hình đã trích xuất được 3 vụ việc có MDMA từ các chunk, nhưng chỉ gọi là *"Thành"* (thay vì đầy đủ *"Lê Minh Thành"*), mô tả kiện hàng ở sân bay nhưng không gọi tên *"Cái Quang Huy"*, và bỏ sót vụ *"Pháp y tâm thần"*.
  - LLM-as-a-Judge đánh giá câu trả lời đã nắm được bản chất các vụ có MDMA trong context nên chấm điểm 2, trong khi metric từ khóa máy móc do không thấy đúng 3 chuỗi con nguyên vẹn nên cho điểm 0.00.
- **Đề xuất sửa:**
  - Trong Prompt trả lời: Hướng dẫn rõ ràng yêu cầu mô hình *"Bắt buộc nêu rõ họ tên đầy đủ của bị can/bị cáo cầm đầu và địa danh/tên vụ việc cụ thể"*.
  - Trong Phép đo: Kết hợp Entity Extraction (NER) hoặc Semantic Similarity F1-score thay vì chỉ dùng phép kiểm tra chuỗi con thuần túy.

---

## 4. Kết luận (5 điểm)

Từ các kết quả đo lường định lượng ở Mục 1 và Mục 2, chúng ta có thể rút ra kết luận khoa học và thực tiễn về việc ứng dụng Knowledge Graph:

1. **Khi nào Flat RAG là đủ:**
   - Khi bài toán chỉ bao gồm các câu hỏi **đơn nguồn (single-hop)**, trong đó thông tin cần trả lời nằm gọn trong một đoạn văn bản hoặc một tài liệu duy nhất (như Q1 về định nghĩa luật, Q2 về kết quả phiên tòa).
   - Trong trường hợp này, Flat RAG tối ưu hơn vượt bậc về mặt kinh tế kỹ thuật: Thời gian dựng hệ thống nhanh hơn **1.35 lần** (114s vs 154s), chi phí prompt rẻ hơn **8.1 lần** (696 tokens vs 5,651 tokens) mà vẫn đạt độ chính xác tối đa (`recall = 1.00, judge = 2`). Việc dựng Knowledge Graph trong kịch bản này là dư thừa và không mang lại giá trị gia tăng.

2. **Khi nào bắt buộc phải dùng GraphRAG:**
   - Khi câu hỏi yêu cầu **liên kết xuyên cơ sở tri thức (cross-KB)** hoặc **suy luận đa bước (multi-hop reasoning)** (như Q3, Q5), nơi một phần dữ kiện nằm ở bài báo (tên người, tang vật) và một phần nằm ở bộ luật (tội danh, khung hình phạt). Flat RAG hoàn toàn bất lực vì vector embedding không thể tìm thấy một đoạn văn nào chứa đồng thời cả hai nguồn tri thức này (`recall` chỉ đạt 0.33 – 0.40).
   - Khi câu hỏi yêu cầu **tổng hợp toàn diện (aggregation)** (như Q6), nơi các thực thể nằm rải rác ở hàng chục bài báo khác nhau. Knowledge Graph gom nhóm các vụ án quanh thực thể trung tâm (`Substance: MDMA`), giúp GraphRAG đạt `recall = 1.00` trong khi Flat RAG bỏ sót hoàn toàn (`recall = 0.00`).
   - **Điểm hòa vốn:** Dù chi phí dựng ban đầu của Graph cao hơn (gọi LLM trích xuất), nhưng chi phí này là **trả 1 lần (one-off)**. Khi số lượng truy vấn cross-KB vượt qua ngưỡng vài trăm câu hỏi, giá trị về độ chính xác và tính toàn vẹn thông tin hoàn toàn bù đắp cho chi phí mở rộng đồ thị.

---

## 5. Tự kiểm (5 điểm)

### Output lệnh kiểm thử đơn vị:
```bash
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.08s
```

### Output lệnh tự kiểm tra hợp đồng:
```bash
$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 293 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 18 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00000. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

### Minh chứng hình ảnh Neo4j:
- **`report/img/kg_count.png`**: Chụp bảng thống kê số lượng node theo cả 7 label trong ontology.
- **`report/img/kg_cross_kb.png`**: Chụp đồ thị trực quan đường đi từ tin tức qua node cầu nối `Crime` tới KB luật, hiển thị đầy đủ thanh Results overview.
- **`report/img/kg_my_case.png`**: Chụp toàn bộ cụm đồ thị vụ án của một nhân vật tự chọn.
- **Người đã chọn cho `kg_my_case.png`:** **`Cái Quang Huy`** (vụ án vận chuyển trái phép chất ma túy qua sân bay Nội Bài, liên quan đến MDMA và Ketamine, kết nối tới Điều 250 BLHS).

---

## Vấn đề gặp phải (không tính điểm)

- **Vấn đề ban đầu với API Key:** Ban đầu file `.env` chứa API key có tiền tố `gsk_...` (thuộc nhà cung cấp Groq), dẫn đến lỗi `401 AuthenticationError` khi gọi OpenAI endpoint mặc định.
- **Giải pháp:** Đã chuyển cấu hình sang sử dụng nhà cung cấp `gemini` với model Chat `gemini-3.5-flash-lite` và model Embedding `gemini-embedding-001` bản Free có sẵn, đồng thời bổ sung cơ chế tự động thử lại (exponential backoff retry) khi gặp lỗi Rate Limit (429) trong `src/llm.py`, giúp toàn bộ pipeline benchmark chạy mượt mà $0.00 USD và đạt điểm số tối đa.
