# Suy Ngẫm Cá Nhân & Kế Hoạch Đồ Án (Reflection & Action Plan)

**Học viên:** Phạm Văn Tâm  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Ngày thực hiện:** 20/08/2026  

---

## 1. Bảng Mapping Bài Giảng vào Code Thực Tế

| Khái niệm trong bài giảng | Module tương ứng | Hàm / Khối code cụ thể | Quan sát thực tế & Đánh giá |
|:---|:---:|:---|:---|
| **Conservative Coreference** | Module 1 | `resolve_coref_batch()` | Giảm thiểu False Antecedent, bảo toàn độ chính xác của entity context. |
| **Schema & Allowlist Guard** | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Ngăn chặn hiện tượng trích xuất relation tự do gây loãng schema đồ thị. |
| **Bulk Cypher Ingestion** | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | Dùng `UNWIND` batch 1000 tăng tốc độ nạp dữ liệu lên hơn 40 lần. |
| **Entity Resolution & Union-Find** | Module 3 | `build_resolution_map()`, `UF` | Gộp thành công các biến thể Ticker/Corp nhưng chặn được False Merge nhờ Lexical Guard. |
| **Super-node Degree Cap** | Module 4 | `retrieve_graph_context()`, `node_degree()` | Cắt tỉa node bậc > 100 về 50 cạnh mới nhất, kiểm soát chặt context size dưới 14,000 ký tự. |
| **LLM-as-a-Judge Evaluation** | Module 5 | `judge_answer()`, `run_evaluation()` | Đánh giá khách quan 3 trục: Comprehensiveness, Faithfulness, Multi-hop Reasoning. |

---

## 2. Quá Trình Debugging & Bài Học Kỹ Thuật

- **Lỗi khó nhất gặp phải:** Lỗi Cypher query chạy chậm và timeout khi traverse đồ thị tại các node trung tâm (Google, Microsoft) do chưa tạo index và gặp bùng nổ đường đi (Combinatorial Path Explosion).
- **Cách khắc phục:**
  1. Khởi tạo Unique Constraint trên `(n:Entity).id` và Index trên `(n:Entity).name_norm`.
  2. Áp dụng cơ chế **Super-node Mitigation**: Đo degree trước khi duyệt; nếu $degree > 100$, chỉ lấy tối đa 50 cạnh mới nhất theo `published_date`.
  3. Đặt chặn toàn cục `GLOBAL_EDGE_CAP = 250` và `MAX_GRAPH_CONTEXT_CHARS = 14000`.
- **Bài học rút ra:** 
  - Đồ thị tri thức trong thực tế tuân theo phân phối mạng không tỷ xích (Scale-Free Network / Power-Law Distribution), nơi một số ít node trung tâm (Hubs) sở hữu phần lớn các liên kết.
  - Xây dựng hệ thống GraphRAG cho production bắt buộc phải có cơ chế kiểm soát bậc kết nối (Degree Capping) và chặn kích thước context nếu không muốn hệ thống bị sập do tràn bộ nhớ và chi phí token bùng nổ.

---

## 3. Kế Hoạch Đồ Án Thực Tế (Action Plan)

### 1. Đánh giá bài toán thực tế:
- Trong bài toán phân tích báo cáo tài chính doanh nghiệp và kiểm toán chuỗi cung ứng (Corporate Due Diligence & Supply Chain Risk), dữ liệu có tính chất liên kết thực thể chằng chịt và các câu hỏi luôn mang tính chất suy luận đa bước (Ví dụ: *Tập đoàn X sở hữu những công ty con nào đang nhập khẩu chip từ nhà cung cấp Y tại quốc gia có rủi ro cấm vận?*).
- Flat RAG thuần túy hoàn toàn bất lực trước dạng bài toán này. Kiến trúc Hybrid GraphRAG là bắt buộc để vừa trích xuất được facts chính xác vừa tổng hợp được các mối quan hệ gián tiếp.

### 2. Thiết kế Schema dự kiến:
- **Node Types:** `Company`, `Executive`, `Product`, `SupplyChainPartner`, `RegulatoryRisk`.
- **Relation Types:** `OWNS`, `MANUFACTURES`, `SUPPLIES_TO`, `INVESTED_IN`, `EXPOSED_TO_RISK`, `OPERATES_IN`.

### 3. Chiến lược triển khai Production:
- **Entity Resolution 2 Tầng:** Sử dụng MinHash LSH Blocking ở tầng 1 để lọc ứng viên tiềm năng, sau đó áp dụng mô hình Cross-Encoder Domain kết hợp Regex Lexical Guard ở tầng 2.
- **Hierarchical Graph Indexing:** Kết hợp cả Local Subgraph Traversal (cho câu hỏi vi mô) và Global Community Summarization (cho câu hỏi tổng hợp vĩ mô).
- **Continuous Ingestion:** Thiết lập pipeline nạp dữ liệu phi đồng bộ (Async Worker Pool) với kiểm tra 100% Edge Provenance trước khi cập nhật vào Neo4j.
