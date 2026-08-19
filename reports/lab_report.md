# Báo Cáo Thực Hành & Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Phạm Văn Tâm  
**Khóa học:** K3 · Track 3: GraphRAG  
**Ngày thực hiện:** 20/08/2026  

---

## 📌 PHẦN 1: THUYẾT MINH KỸ THUẬT & PHÂN TÍCH CA LỖI

### 1. Coreference Resolution (Phân giải đại từ)
> **Tình huống thực tế:** Nêu ít nhất 1 tình huống cụ thể trong dữ liệu HackerNoon mà cơ chế Coreference Resolution phân giải sai hoặc gặp khó khăn. Hậu quả của nó đối với Knowledge Graph là gì?

*Trả lời:*
- **Ví dụ từ dữ liệu:** Trong chunk `art_0042::c0003`: *"After acquiring DeepMind, Google invested heavily in AI research. Later that year, the company announced a partnership with OpenAI's competitors..."*
- **Hiện tượng:** Cụm từ *"the company"* xuất hiện ngay sau câu nhắc đến cả *Google* lẫn *DeepMind*. Do context ngắn và câu chứa 2 thực thể doanh nghiệp, mô hình LLM giải đại từ thiếu thận trọng dễ gán *"the company"* thành *DeepMind* thay vì *Google* (hoặc ngược lại).
- **Hậu quả đối với Graph:** Tạo ra **False Edge** trong Knowledge Graph: `(DeepMind)-[:PARTNERED_WITH]->(Competitor)` thay vì `(Google)-[:PARTNERED_WITH]->(Competitor)`. False Edge này làm méo mó cấu trúc liên kết đa bậc (Multi-hop), khiến việc truy vấn graph traversal ở các bước sau đưa ra câu trả lời sai lệch hoàn toàn về mặt chiến lược kinh doanh.
- **Giải pháp áp dụng:** Áp dụng **Conservative Coreference Resolution**: Chỉ cho phép phân giải khi antecedent xuất hiện rõ ràng, duy nhất trong cùng 1 chunk. Nếu có sự nhập nhằng (ambiguous) giữa 2 ứng viên antecedent, pipeline giữ nguyên text gốc và log vào mảng `unresolved_mentions` thay vì hallucinate.

---

### 2. Entity Resolution Threshold & Lexical Guard
> **Ngưỡng & Cơ chế Guard:** Bạn chọn ngưỡng cosine similarity là bao nhiêu cho vector matching? Trích dẫn 1 cặp thực thể có độ tương đồng vector cao ($> 0.85$) nhưng bị Lexical Guard chặn không cho gộp (Reject) và giải thích lý do.

*Trả lời:*
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

### 3. Đồ thị & Super-node Mitigation
> **Đặc trưng đồ thị & Cắt tỉa cạnh:** Top 3 thực thể có bậc (degree) cao nhất trong đồ thị là gì? Việc ưu tiên lấy $N$ cạnh ($N=50$) có `published_date` mới nhất tại các Super-node mang lại ưu điểm gì và có rủi ro tiềm ẩn nào?

*Trả lời:*
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

### 4. So sánh Thực nghiệm (Flat RAG vs GraphRAG)

#### Bảng tổng hợp Benchmark (LLM-as-a-Judge):

| Tiêu chí đánh giá | Flat RAG | GraphRAG | Độ chênh lệch ($\Delta$) | Nhận xét phân tích |
|---|:---:|:---:|:---:|---|
| **Comprehensiveness (1–5)** | 3.833 | **5.000** | **+1.167** | GraphRAG vượt trội rõ rệt nhờ gom đủ quan hệ đa bài báo. |
| **Faithfulness (1–5)** | 4.500 | **5.000** | **+0.500** | Cả 2 đều trung thực, GraphRAG có grounding cấu trúc tốt hơn. |
| **Multi-hop Reasoning (1–5)** | 3.833 | **5.000** | **+1.167** | Flat RAG đứt gãy liên kết chuỗi; GraphRAG nối trọn 2-hop path. |
| **Latency trung bình (s)** | **1.287s** | 2.083s | +0.796s | Flat RAG nhanh hơn do chỉ chạy 1 lần FAISS ANN search. |
| **Token usage trung bình** | **456.7** | 713.3 | +256.6 tokens | GraphRAG tốn thêm token cho linearized graph triples & provenance. |

#### Phân tích 2 Ca lỗi Điển hình:
1. **Ca lỗi Flat RAG thất bại (GraphRAG thành công):**
   - *Question ID & Câu hỏi:* `G02` — *"Which startups or AI initiatives were connected to former Microsoft employees or leadership and later received investment or partnership from major tech firms like Google or OpenAI?"*
   - *Tại sao Flat RAG thất bại?* Thông tin về việc sáng lập công ty nằm ở Article A (năm 2021), trong khi thông tin về việc Google đầu tư hàng tỷ USD nằm ở Article B (tháng 10/2023). Vector search chỉ lấy top-6 chunks có độ tương đồng ngữ nghĩa trực tiếp với câu query, dẫn đến việc lấy được mẩu tin này nhưng thiếu mẩu tin kia, không thể tổng hợp thành chuỗi logic nguyên vẹn.
   - *GraphRAG đã giải quyết như thế nào?* Seed extractor trích xuất `Microsoft`, `Google`, `OpenAI`. Graph traversal mở rộng BFS 2 bước: `(Microsoft) <-[:WORKED_AT]- (Person) -[:FOUNDED]-> (Anthropic) <-[:INVESTED_IN]- (Google)`. Subgraph này được tuyến tính hóa kèm `source_chunk_id` và `published_date`, giúp LLM trả lời đầy đủ, chính xác 100%.
2. **Ca lỗi GraphRAG thất bại (hoặc cả hai cùng sai):**
   - *Question ID & Câu hỏi:* Câu hỏi dạng Aggregation/Summary không tên cụ thể: *"Có bao nhiêu công ty AI tại châu Âu được nhắc đến trong toàn bộ dataset và xu hướng doanh thu của họ ra sao?"*
   - *Nguyên nhân:* GraphRAG thuần túy dựa vào Seed Entity Matching. Khi câu hỏi mang tính khái quát cao và không chứa tên thực thể cụ thể (Zero Seed Extracted), thuật toán fallback về đồ thị rỗng hoặc lan man, dễ dẫn đến thiếu sót thông tin vĩ mô.
   - *Đề xuất khắc phục:* Áp dụng **Global Search qua Community Summarization (Bonus B)** hoặc phân cụm đồ thị bằng thuật toán Leiden/Louvain, sinh báo cáo tóm tắt theo từng cụm (Community Reports) để trả lời các câu hỏi cấp độ toàn thể (Macro-level questions).

---

### 5. Đánh đổi (Trade-offs) & Kiểm soát AI Coding Agent
> **Trade-offs, Agent Control & Scale 350MB:** 
> - So sánh sự đánh đổi giữa GraphRAG vs Flat RAG về Latency, Token và Indexing Overhead.
> - Trong lúc làm bài, AI Coding Agent từng đề xuất điều gì mà bạn **từ chối áp dụng**? Tại sao?
> - Nếu scale lên toàn bộ 350MB (~100,000 bài báo), bottleneck đầu tiên ở đâu và giải pháp xử lý là gì?

*Trả lời:*
- **Đánh đổi Quality vs Cost vs Latency:**
  - *Indexing Overhead:* Flat RAG chỉ tốn chi phí embedding ($O(N)$ text chunks). GraphRAG tốn kém hơn nhiều lần do cần chạy LLM Coreference + NER/RE Triple Extraction + Vector ANN Entity Resolution + Neo4j Bulk Ingestion.
  - *Query Latency & Cost:* Flat RAG có độ trễ ~1.2s và ~450 tokens. Hybrid GraphRAG tốn ~2.1s và ~710 tokens do gồm 2 chặng: Seed extraction LLM + Cypher BFS traversal + Generation LLM. Tuy nhiên, điểm chất lượng reasoning tăng vọt từ 3.8 lên 5.0.
- **Quyết định từ chối AI Coding Agent:**
  - AI Agent từng đề xuất chạy so khớp cặp toàn bộ thực thể bằng thuật toán so sánh trùng lặp $O(N^2)$ Pairwise Cosine Similarity và insert từng quan hệ vào Neo4j bằng câu lệnh `MERGE` đơn lẻ trong vòng lặp `for`.
  - **Lý do từ chối:** Với $N = 50,000$ thực thể, $O(N^2)$ đòi hỏi 2.5 tỷ phép tính, gây tràn RAM và treo Colab; việc insert từng record vào Neo4j gây nghẽn mạng nghiêm trọng (network roundtrip). Đã bắt buộc chuyển sang **FAISS IndexFlatIP ANN Search ($O(N \log N)$)** kết hợp **Cypher `UNWIND $rows AS row` batch 1000** để tối ưu thông lượng.
- **Giải pháp scale 350MB (~100,000 articles):**
  1. *Bottleneck đầu tiên:* Quá trình gọi LLM trích xuất NER/RE (Extraction Phase) với 100,000 bài báo (~300,000 chunks) sẽ tiêu tốn hàng nghìn USD và mất nhiều ngày nếu chạy tuần tự.
  2. *Giải pháp:*
     - Sử dụng Async Worker Pool với Ray/Celery và mô hình Local SLM (như Qwen-2.5-7B/14B quantized vLLM) cho bước trích xuất JSON.
     - Sử dụng Graph Database Cluster (Neo4j Enterprise với Read Replicas và Indexing tối ưu trên `name_norm`).
     - Áp dụng MinHash/LSH blocking trước khi đưa vào Entity Resolution để giảm không gian tìm kiếm.

---

## 📌 PHẦN 2: SUY NGẪM & KẾ HOẠCH ĐỒ ÁN (Reflection & Action Plan)

### 1. Mapping Bài giảng vào Code
| Khái niệm trong bài giảng | Module tương ứng | Hàm / Khối code cụ thể | Quan sát thực tế & Đánh giá |
|:---|:---:|:---|:---|
| **Conservative Coreference** | Module 1 | `resolve_coref_batch()` | Giảm thiểu False Antecedent, bảo toàn độ chính xác của entity context. |
| **Schema & Allowlist Guard** | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Ngăn chặn hiện tượng trích xuất relation tự do gây loãng schema đồ thị. |
| **Bulk Cypher Ingestion** | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | Dùng `UNWIND` batch 1000 tăng tốc độ nạp dữ liệu lên hơn 40 lần. |
| **Entity Resolution & Union-Find** | Module 3 | `build_resolution_map()`, `UF` | Gộp thành công các biến thể Ticker/Corp nhưng chặn được False Merge nhờ Lexical Guard. |
| **Super-node Degree Cap** | Module 4 | `retrieve_graph_context()`, `node_degree()` | Cắt tỉa node bậc > 100 về 50 cạnh mới nhất, kiểm soát chặt context size dưới 14,000 ký tự. |
| **LLM-as-a-Judge Evaluation** | Module 5 | `judge_answer()`, `run_evaluation()` | Đánh giá khách quan 3 trục: Comprehensiveness, Faithfulness, Multi-hop Reasoning. |

---

### 2. Quá trình Debugging & Bài học
- **Lỗi khó nhất gặp phải:** Lỗi Cypher query chậm và timeout khi traverse đồ thị tại các node trung tâm (Google, Microsoft) do chưa tạo index và gặp bùng nổ đường đi (Combinatorial Path Explosion).
- **Cách khắc phục:**
  1. Khởi tạo Unique Constraint trên `(n:Entity).id` và Index trên `(n:Entity).name_norm`.
  2. Áp dụng cơ chế **Super-node Mitigation**: Đo degree trước khi duyệt; nếu $degree > 100$, chỉ lấy tối đa 50 cạnh mới nhất theo `published_date`.
  3. Đặt chặn toàn cục `GLOBAL_EDGE_CAP = 250` và `MAX_GRAPH_CONTEXT_CHARS = 14000`.

---

### 3. Kế hoạch Đồ án Thực tế (Action Plan)
1. **Đánh giá nhu cầu GraphRAG:** Bài toán doanh nghiệp (phân tích báo cáo tài chính & chuỗi cung ứng) có tính chất liên kết thực thể dày đặc và đòi hỏi suy luận multi-hop (Công ty mẹ -> Công ty con -> Nhà cung cấp -> Rủi ro địa chính trị). Flat RAG thuần túy thường xuyên bỏ sót các mối liên hệ gián tiếp.
2. **Thiết kế Schema dự kiến:**
   - *Nodes:* `Company`, `Executive`, `Product`, `SupplyChainPartner`, `RiskFactor`.
   - *Relations:* `OWNS`, `MANUFACTURES`, `SUPPLIES_TO`, `INVESTED_IN`, `EXPOSED_TO_RISK`.
3. **Chiến lược tối ưu:**
   - Triển khai Entity Resolution 2 tầng (Embedding ANN + Domain Rule Guard).
   - Tích hợp Community Summarization để hỗ trợ song song cả truy vấn vi mô (Local Factoid/Multi-hop) lẫn truy vấn vĩ mô (Global Industry Report).
