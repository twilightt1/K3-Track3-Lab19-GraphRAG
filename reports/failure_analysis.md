# Phân Tích Ca Lỗi (Failure Analysis) — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Phạm Văn Tâm  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Ngày thực hiện:** 20/08/2026  

---

## 1. Ca lỗi 1: Flat RAG thất bại (GraphRAG thành công)

- **Mã câu hỏi (ID):** `G02`
- **Nội dung câu hỏi:**  
  *"Which startups or AI initiatives were connected to former Microsoft employees or leadership and later received investment or partnership from major tech firms like Google or OpenAI?"*
- **Reference Answer:**  
  *Anthropic was co-founded by former OpenAI/Microsoft AI research executives (including Dario Amodei) and subsequently secured multi-billion dollar investment and cloud partnerships from Google in 2023.*

### Triệu chứng & Kết quả thực nghiệm:
- **Flat RAG Output:** Trích xuất được mẩu tin Google đầu tư vào Anthropic năm 2023 [chunk `art_0042::c0003`], nhưng hoàn toàn bỏ sót dữ kiện về đội ngũ sáng lập từng làm việc tại Microsoft/OpenAI vì dữ kiện này nằm ở một bài báo khác từ năm 2021 [chunk `art_0105::c0002`].
- **Điểm số Judge:**
  - Flat RAG: Comprehensiveness = 3/5, Multi-hop Reasoning = 3/5.
  - GraphRAG: Comprehensiveness = 5/5, Multi-hop Reasoning = 5/5.

### Phân tích nguyên nhân gốc rễ (Root Cause Analysis):
- Flat RAG sử dụng Vector Search (FAISS FlatIP) để tìm Top-6 chunks có cosine similarity cao nhất với toàn bộ câu query.
- Do câu hỏi chứa nhiều mệnh đề ("former Microsoft employees", "received investment from Google"), vector embedding của câu hỏi bị trung bình hóa ngữ nghĩa (semantic dilution). Kết quả là Top-6 chunks chỉ tập trung vào tin tức nổi bật nhất (thương vụ đầu tư của Google) mà không có chunk nào chứa lịch sử nhân sự của nhà sáng lập.
- **GraphRAG giải quyết như thế nào:**
  1. Trích xuất Seed Entities: `Microsoft`, `Google`, `OpenAI`.
  2. Duyệt đồ thị BFS 2-hop:
     ```
     (Microsoft:Company) <-[:WORKED_AT]- (Dario Amodei:Person) -[:FOUNDED]-> (Anthropic:Company) <-[:INVESTED_IN]- (Google:Company)
     ```
  3. Tuyến tính hóa đường đi này kèm `source_chunk_id` và `published_date` đưa vào context của LLM Generator. Nhờ vậy, câu trả lời tái hiện chính xác 100% chuỗi suy luận đa bước.

---

## 2. Ca lỗi 2: GraphRAG thất bại / gặp khó khăn (Macro-level Aggregation Query)

- **Mã câu hỏi (ID):** `G_AGG_01`
- **Nội dung câu hỏi:**  
  *"Có bao nhiêu công ty AI tại châu Âu được nhắc đến trong toàn bộ dataset và xu hướng doanh thu/đầu tư của họ trong năm 2023 ra sao?"*

### Triệu chứng & Kết quả thực nghiệm:
- **GraphRAG Output:** Trả về câu trả lời rỗng hoặc rất sơ sài, chỉ liệt kê 1–2 công ty ngẫu nhiên nếu may mắn bắt được từ khóa mơ hồ.
- **Điểm số Judge:** Comprehensiveness = 2/5, Faithfulness = 3/5.

### Phân tích nguyên nhân gốc rễ (Root Cause Analysis):
- Kiến trúc GraphRAG cục bộ (Local GraphRAG) phụ thuộc hoàn toàn vào bước **Seed Entity Extraction** để làm điểm khởi đầu duyệt đồ thị.
- Khi người dùng hỏi một câu hỏi mang tính tổng quan/vĩ mô (Macro-level / Aggregation question) không nhắc đến tên thực thể cụ thể nào ("các công ty AI tại châu Âu"), Seed Extractor không trích xuất được node hạt nhân hợp lệ trong Neo4j (Zero-Seed Match).
- Kết quả là pipeline không thể khởi tạo BFS frontier, dẫn đến đồ thị con rỗng.

### Giải pháp khắc phục (Mitigation & Architecture Enhancement):
1. **Community Detection & Global Search (Bonus A):**
   - Áp dụng thuật toán phân cụm cộng đồng (Leiden / Louvain / Greedy Modularity) để chia toàn bộ Knowledge Graph thành các cụm thứ bậc (Hierarchical Communities).
   - Dùng LLM sinh bản tóm tắt định kỳ cho từng cộng đồng (Community Summaries).
   - Khi gặp câu hỏi vĩ mô, hệ thống thực hiện **Global Search** bằng cách quét qua các bản tóm tắt cộng đồng thay vì duyệt đồ thị cục bộ.
2. **Self-Correction & Vector Fallback (Bonus B):**
   - Đánh giá tính đầy đủ của context đồ thị (`context_sufficient()`). Nếu phát hiện không có seed hoặc thiếu dữ kiện, tự động fallback sang Vector Search diện rộng ($k=12$) kết hợp Metadata Filtering (lọc theo thuộc tính địa lý / quốc gia).
