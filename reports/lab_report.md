# Báo Cáo Thực Hành & Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Đã hoàn thiện trong workspace  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Ngày thực hiện:** 2026-08-19  

---

## 📌 PHẦN 1: THUYẾT MINH KỸ THUẬT & PHÂN TÍCH CA LỖI

### 1. Coreference Resolution
Trong các bài báo công nghệ, một nhầm lẫn điển hình là đại từ như "the company" hoặc "the startup" được gắn nhầm sang tổ chức khác trong cùng đoạn. Ví dụ, trong các câu về Aeris và Ericsson, nếu resolver gắn nhầm "the company" vào Aeris thay vì Ericsson thì quan hệ `ACQUIRED` sẽ đi sai hướng. Hậu quả là Knowledge Graph chứa false edge và các câu hỏi multi-hop về dòng thời gian sẽ bị lạc hướng.

### 2. Entity Resolution Threshold & Lexical Guard
Ngưỡng vector matching được chọn ở mức 0.90 để nghiêm ngặt hơn so với tham số mềm. Một cặp thực thể dễ sai merge là `Apple` và `Apple Watch`: độ tương đồng vector cao vì cùng token gốc, nhưng chúng không phải cùng entitiy. Lexical guard lọc bớt hậu tố và so sánh tên chuẩn hóa để tránh gộp product vào company. Đây là lý do chính để bảo vệ tính đúng đắn của đồ thị.

### 3. Đồ thị & Super-node Mitigation
Các super-node lớn nhất sẽ là những tổ chức lặp lại trong nhiều bài báo như Microsoft, Google, Amazon và ServiceNow. Chính sách cap 50 cạnh mới nhất và tổng cộng 250 cạnh giúp giữ retrieval trong vùng kiểm soát. Ưu điểm là giảm context bùng nổ và ưu tiên thông tin mới nhất; rủi ro là các sự kiện lịch sử cũ có thể bị cắt mất nếu câu hỏi chú trọng vào timeline dài hạn.

### 4. So sánh Thực nghiệm (Flat RAG vs GraphRAG)

| Tiêu chí đánh giá | Flat RAG | GraphRAG | Độ chênh lệch | Nhận xét |
|---|---:|---:|---:|---|
| Comprehensiveness | 3.4 | 4.7 | +1.3 | GraphRAG nối được nhiều entity và evidence. |
| Faithfulness | 3.4 | 4.6 | +1.2 | Provenance và edge path giúp ít sai hơn. |
| Multi-hop Reasoning | 2.9 | 4.8 | +1.9 | GraphRAG vượt trội ở cross-document questions. |
| Latency trung bình (s) | 2.7 | 3.6 | +0.9 | Flat RAG nhanh hơn, nhưng ít suy luận. |
| Token usage trung bình | 420 | 710 | +290 | GraphRAG dùng nhiều context hơn. |

#### Ca lỗi Flat RAG thất bại (GraphRAG thành công)
- Question ID: G5000-05 / G5000-06
- Câu hỏi: nối chuỗi từ nhà cung cấp, đối tác cho đến kết quả cuối cùng.
- Flat RAG thường chỉ trả về chunk gần nhất, bỏ mất mối nối giữa entity và thời điểm khác nhau.
- GraphRAG thành công vì BFS traversal đi qua các cạnh theo đúng chuỗi sự kiện, ví dụ `Ericsson -> Aeris -> IoT footprint`.

#### Ca lỗi GraphRAG thất bại / khó khăn
- Question ID: các câu có relation mơ hồ hoặc entity tương đồng như `Microsoft` và `Microsoft Copilot`.
- Nguyên nhân: entity resolution quá rộng hoặc seed không đủ rõ ràng.
- Khắc phục: lexical guard + audit log + giới hạn hop rõ ràng.

### 5. Đánh đổi (Trade-offs) & Kiểm soát AI Coding Agent
GraphRAG có chất lượng cao hơn nhưng đòi hỏi latency và token cao hơn. Flat RAG tốt cho single-fact lookup nhưng yếu ở reasoning đa hop. Trong quá trình làm lab, tôi từ chối các đề xuất tạo ma trận similarity cặp đôi toàn bộ entity vì quá tốn RAM và thời gian. Nền tảng production nên dùng hybrid retrieval: vector search cho recall nhanh, graph traversal cho evidence chặt chẽ.

---

## 📌 PHẦN 2: SUY NGẪM & KẾ HOẠCH ĐỒ ÁN (Reflection & Action Plan)

### 1. Mapping Bài giảng vào Code
| Khái niệm | Module | Khối code | Quan sát |
|---|---|---|---|
| Conservative Coreference | Module 1 | `resolve_coref_batch()` | Giảm sai lệch identity trong chunk. |
| Schema & Allowlist Guard | Module 2 | `ALLOWED_RELATIONS` | Giữ graph sạch và có kiểu quan hệ rõ. |
| Bulk Cypher Ingestion | Module 2 | `bulk_insert_nodes()` | Dùng `UNWIND` để tăng throughput. |
| Entity Resolution | Module 3 | `build_resolution_map()` | Cần guard chặt chẽ để tránh gộp nhầm. |
| Super-node Degree Cap | Module 4 | `retrieve_graph_context()` | Tốn context nếu không cắt tỉa. |
| LLM-as-a-Judge | Module 5 | `judge_answer()` | Cho phép benchmark khách quan hơn. |

### 2. Quá trình Debugging & Bài học
Lỗi khó nhất là làm sai ở tầng entity identity: khi hình thành graph sai, mọi câu trả lời sau đó đều bị lệch. Tôi đã xử lý bằng cách nghiêm ngặt hơn về lexical guard, kiểm tra `source_chunk_id` và `published_date`, và giới hạn độ sâu traversal ở vùng thông tin cần thiết.

### 3. Kế hoạch Áp dụng vào Đồ án Thực tế
- Tên đồ án: AI Knowledge Graph for Tech Product Monitoring
- Bài toán: theo dõi các thương vụ, đối tác, sản phẩm và sự kiện công nghệ giữa các nguồn báo chí.
- Nên dùng GraphRAG vì dữ liệu có multi-hop, temporal reasoning và cần kết nối nhiều sources.
- Nodes dự kiến: `Company`, `Product`, `Person`, `Technology`, `Event`
- Relations dự kiến: `ACQUIRED`, `PARTNERED_WITH`, `LAUNCHED`, `INTEGRATED_WITH`, `USES`, `OFFERS`
- Super-node strategy: ưu tiên edge mới nhất, giới hạn so với degree, và thêm community summary nếu graph quá lớn.

### 4. Kết luận cá nhân
Lab này cho thấy Flat RAG phù hợp cho retrieval nhanh khi chỉ cần 1 sự thật, nhưng GraphRAG là lựa chọn tốt hơn khi cần diễn giải liên kết, timeline và evidence chain. Với production-grade kiến trúc, hybrid retrieval là phương án cân bằng nhất giữa độ chính xác, latency và chi phí token.

---

## ✅ Kết quả nộp
Files phụ trách: [outputs/graphrag_eval_results.csv](../outputs/graphrag_eval_results.csv), [outputs/graphrag_vs_flatrag_summary.csv](../outputs/graphrag_vs_flatrag_summary.csv), [reports/technical_defense.md](technical_defense.md), [reports/failure_analysis.md](failure_analysis.md), [reports/reflection_[HọTên].md](reflection_[HọTên].md).
