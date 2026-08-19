# Thuyết Minh Kỹ Thuật (Technical Defense) — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Phạm Văn Tâm  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Ngày thực hiện:** 20/08/2026  

---

### Câu 1: Coreference Resolution (Phân giải đại từ)
- **Tình huống thực tế:** Trong chunk `art_0042::c0003`: *"After acquiring DeepMind, Google invested heavily in AI research. Later that year, the company announced a partnership with OpenAI's competitors..."*
- **Hiện tượng:** Cụm từ *"the company"* xuất hiện ngay sau câu nhắc đến cả *Google* lẫn *DeepMind*. Do context ngắn và câu chứa 2 thực thể doanh nghiệp, mô hình LLM giải đại từ thiếu thận trọng dễ gán *"the company"* thành *DeepMind* thay vì *Google* (hoặc ngược lại).
- **Hậu quả đối với Graph:** Tạo ra **False Edge** trong Knowledge Graph: `(DeepMind)-[:PARTNERED_WITH]->(Competitor)` thay vì `(Google)-[:PARTNERED_WITH]->(Competitor)`. False Edge này làm méo mó cấu trúc liên kết đa bậc (Multi-hop), khiến việc truy vấn graph traversal ở các bước sau đưa ra câu trả lời sai lệch hoàn toàn về mặt chiến lược kinh doanh.
- **Giải pháp áp dụng:** Áp dụng **Conservative Coreference Resolution**: Chỉ cho phép phân giải khi antecedent xuất hiện rõ ràng, duy nhất trong cùng 1 chunk. Nếu có sự nhập nhằng (ambiguous) giữa 2 ứng viên antecedent, pipeline giữ nguyên text gốc và log vào mảng `unresolved_mentions` thay vì hallucinate.

---

### Câu 2: Entity Resolution Threshold & Lexical Guard
- **Ngưỡng cosine similarity:** `threshold = 0.90` (sử dụng model embedding `sentence-transformers/all-MiniLM-L6-v2` với cosine similarity / inner product trên normalized vectors).
- **Cặp thực thể bị Guard chặn:** 
  1. `Apple` vs `Apple Music` (Cosine similarity: `0.887` → Bị `merge_guard()` từ chối: `REJECT_GUARD`).
  2. `Sam Altman` vs `Steve Altman` (Cosine similarity: `0.864` → Bị `merge_guard()` từ chối: `REJECT_GUARD`).
  3. `Google` vs `Google Cloud` (Cosine similarity: `0.912` → Bị `merge_guard()` từ chối: `REJECT_GUARD`).
- **Lý do chặn:** 
  - Trong không gian vector embedding ngữ nghĩa, tên công ty và tên sản phẩm/dịch vụ của công ty đó (hoặc 2 người cùng họ trong cùng lĩnh vực công nghệ) nằm rất gần nhau.
  - Nếu chỉ dựa vào vector similarity, thuật toán sẽ gộp (False Merge) `Apple` (Company) và `Apple Music` (Product/Technology) thành 1 node duy nhất.
  - **Lexical Guard** chuẩn hóa tên, loại bỏ hậu tố doanh nghiệp (`Inc, Corp, Ltd, LLC`), sau đó kiểm tra tỷ lệ trùng khớp từ vựng chuỗi ký tự (`SequenceMatcher.ratio >= 0.72`). Khi tên thực thể có thêm từ tố chỉ sản phẩm ("Music", "Cloud") hoặc tên riêng khác biệt ("Sam" vs "Steve"), Lexical Guard phát hiện sự khác biệt từ vựng cấu trúc và lập tức chặn merge (`REJECT_GUARD`), bảo vệ tính toàn vẹn của đồ thị.

---

### Câu 3: Đồ thị & Super-node Mitigation
- **Top 3 Super-nodes:**

| Hạng | Tên thực thể | Loại thực thể (Type) | Bậc kết nối (Degree) |
|:---:|:---|:---|:---:|
| 1 | **Google** (Alphabet) | `Company` | **184** |
| 2 | **Microsoft** | `Company` | **162** |
| 3 | **OpenAI** | `Company` | **128** |

- **Ưu điểm & Rủi ro của Temporal Mitigation:**
  - *Ưu điểm:* 
    1. Ngăn chặn triệt để hiện tượng **Context Explosion**: Khi duyệt BFS qua các node trung tâm như Google/Microsoft, nếu không cắt tỉa, số lượng cạnh bùng nổ lên hàng trăm cạnh, vượt quá context window của LLM và gây tràn token (OOM / Rate limit).
    2. Giữ lại các sự kiện công nghệ và quan hệ đối tác, đầu tư mới nhất (`published_date DESC`), phản ánh chính xác trạng thái hiện tại của thị trường.
    3. Giảm độ trễ truy vấn Cypher và giảm chi phí token cho bước generation.
  - *Rủi ro tiềm ẩn:* 
    1. **Historical Amnesia (Mất dấu lịch sử):** Nếu người dùng hỏi về các sự kiện trong quá khứ xa (ví dụ: *"Ai là người sáng lập Google năm 1998?"* hoặc *"Thương vụ thâu tóm Android diễn ra khi nào?"*), chính sách lấy 50 cạnh mới nhất sẽ cắt bỏ hoàn toàn các cạnh lịch sử này.
    2. **Khắc phục:** Kết hợp Hybrid Retrieval (Vector Search sẽ bù đắp các chunk lịch sử) hoặc áp dụng **Query-Aware Edge Filtering** (lọc quan hệ dựa trên relation type hoặc khoảng thời gian được hỏi thay vì chỉ `LIMIT 50` đơn thuần).

---

### Câu 4: Schema & Bulk Ingestion Strategy
- Sử dụng Schema Allowlist chặt chẽ:
  - Node Types: `Company`, `Person`, `Technology`.
  - Relations: `ACQUIRED`, `DEVELOPED`, `INVESTED_IN`, `FOUNDED`, `WORKED_AT`, `PARTNERED_WITH`, `USES`, `LEADS`.
- Sử dụng câu lệnh Cypher `UNWIND $rows AS row` theo từng batch 1,000 records giúp giảm thiểu network round-trips từ hàng ngàn cuộc gọi xuống chỉ còn vài chục cuộc gọi, tăng thông lượng nạp dữ liệu hơn 40 lần.
- Thiết lập Unique Constraint trên `(n:Entity).id` và Index trên `(n:Entity).name_norm`.
- Toàn bộ các cạnh nạp vào đều có 100% provenance metadata (`source_chunk_id`, `published_date`, `evidence`, `confidence`). Kết quả kiểm tra sanity check: `invalid_provenance_edges = 0`.

---

### Câu 5: Seed Entity Matching & Fuzzy Fallback
- Quá trình Seed Extraction trích xuất thực thể hạt nhân trực tiếp từ câu hỏi của người dùng kèm loại thực thể mong đợi.
- Kết hợp 2 tầng đối sánh:
  1. *Exact Match:* So khớp trên `name_norm` và danh sách `aliases_norm`.
  2. *Fuzzy Vector Fallback:* Nếu không khớp chính xác, tính cosine similarity giữa embedding của seed và toàn bộ danh mục thực thể trong đồ thị với ngưỡng `fuzzy_threshold = 0.66`.
- Đảm bảo độ phủ seed tối đa kể cả khi người dùng gõ sai chính tả hoặc sử dụng tên viết tắt thông dụng.

---

### Câu 6: BFS Traversal & Subgraph Linearization
- Sử dụng thuật toán Breadth-First Search (BFS) mở rộng tối đa 2 bước (`max_hops = 2`) từ các seed entities.
- Áp dụng các tầng chặn an toàn:
  - Cắt tỉa Super-node: Node có bậc $> 100$ chỉ lấy tối đa 50 cạnh mới nhất.
  - Chặn toàn cục: `GLOBAL_EDGE_CAP = 250` cạnh.
  - Chặn ký tự ngữ cảnh: `MAX_GRAPH_CONTEXT_CHARS = 14,000` ký tự.
- Chuyển đồ thị thành văn bản có cấu trúc rõ ràng với đầy đủ trích dẫn xuất xứ từng sự kiện (`source_chunk_id`, `published_date`, `evidence`).

---

### Câu 7: Benchmark Performance Analysis
- **Factoid:** Cả Flat RAG và GraphRAG đều đạt điểm tối đa (5.0/5.0). Flat RAG nhanh hơn (~1.12s vs ~1.84s).
- **Multi-hop:** GraphRAG vượt trội hoàn toàn (5.0/5.0 vs 3.5/5.0). Flat RAG thất bại vì context vector search bị phân mảnh qua nhiều chunk độc lập.
- **Cross-document:** GraphRAG đạt 4.5–5.0/5.0 nhờ khả năng tổng hợp các cạnh sự kiện theo dòng thời gian (`published_date`), trong khi Flat RAG chỉ đạt 3.0–3.5/5.0.

---

### Câu 8: Trade-offs (Chất lượng vs Chi phí vs Độ trễ)
- *Flat RAG:* Chi phí indexing thấp ($O(N)$ text embedding), độ trễ truy vấn thấp (~1.28s), token tiêu thụ thấp (~456 tokens). Phù hợp với tra cứu đơn giản.
- *GraphRAG:* Chi phí indexing cao (gọi LLM trích xuất NER/RE, Entity Resolution, nạp Neo4j), độ trễ cao hơn (~2.08s), token tiêu thụ cao hơn (~713 tokens). Đổi lại, chất lượng trả lời cho câu hỏi phức tạp (multi-hop, cross-doc) tăng vọt từ 3.5 lên 5.0.

---

### Câu 9: Kiểm Soát AI Coding Agent & Đề Xuất Bị Từ Chối
- **Đề xuất bị từ chối:** AI Agent từng đề xuất so khớp trùng lặp thực thể bằng thuật toán $O(N^2)$ Pairwise Similarity và insert từng quan hệ vào Neo4j bằng câu lệnh `MERGE` đơn lẻ trong vòng lặp `for`.
- **Lý do từ chối:** Với $N = 50,000$ thực thể, $O(N^2)$ gây tràn RAM (2.5 tỷ phép so sánh); chèn từng dòng vào Neo4j gây nghẽn mạng nghiêm trọng. Bắt buộc chuyển sang **FAISS IndexFlatIP ($O(N \log N)$)** và **Cypher `UNWIND` batch 1000**.

---

### Câu 10: Scalability Blueprint (Quy mô 350MB / ~100,000 bài báo)
1. **Bottleneck 1 (LLM Extraction):** ~300,000 chunks đòi hỏi thông lượng xử lý cao.  
   *Giải pháp:* Dùng Ray/Celery distributed worker pool kết hợp mô hình Local SLM (như Qwen-2.5-7B/14B quantized vLLM) chạy batch extraction song song.
2. **Bottleneck 2 (Entity Resolution):** Không gian $N$ quá lớn.  
   *Giải pháp:* Dùng MinHash LSH blocking / Annoy Indexing trước khi chạy FAISS ANN.
3. **Bottleneck 3 (Database Ingestion):** Dùng Neo4j Admin Import Tool cho đợt nạp ban đầu, sau đó duy trì hàng đợi Kafka cho ingestion thời gian thực.
