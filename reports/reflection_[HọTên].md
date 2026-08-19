# Reflection & Action Plan

## 1. Mapping bài giảng vào code
| Khái niệm trong bài giảng | Module tương ứng | Hàm / Khối code cụ thể | Quan sát thực tế |
|---|---|---|---|
| Conservative Coreference | Module 1 | `resolve_coref_batch()` | Bảo vệ tránh giải thích sai `it/the company` trong cùng một chunk. |
| Schema & allowlist | Module 2 | `ALLOWED_RELATIONS` | Giữ cho các loại quan hệ hợp lệ và tránh edge sai kiểu. |
| Bulk Cypher ingestion | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | Dùng `UNWIND` để tránh insert từng row. |
| Entity Resolution | Module 3 | `build_resolution_map()`, union-find | Cần lexical guard để tránh gộp nhầm `Apple` với `Apple Watch`. |
| Super-node mitigation | Module 4 | `retrieve_graph_context()` | `degree > 100` phải bị cắt tỉa lịch sử và giới hạn số edge. |
| LLM-as-a-Judge | Module 5 | `judge_answer()` | Chấm điểm theo comprehensiveness, faithfulness, multi-hop. |

## 2. Quá trình Debugging & Bài học
Lỗi khó nhất là khi retrieval bị nhiễu vì graph quá rộng hoặc entity resolution quá aggressive. Trong nhiều trường hợp, tôi thấy câu trả lời tốt về một topic nhưng lại sai trong mối quan hệ vì các entity bị gộp quá rộng. Cách xử lý là thêm lexical guard, giới hạn hop và cắt tỉa theo published_date, đồng thời kiểm tra audit log trước khi biến dữ liệu thành final answer.

## 3. Kế hoạch áp dụng vào đồ án thực tế
- Tên đồ án: `AI Knowledge Graph for Tech Product Monitoring`
- Bài toán: theo dõi các thương vụ, đối tác, sản phẩm và sự kiện công nghệ từ nhiều nguồn tin.
- Tại sao cần GraphRAG: vì dữ liệu là đa nguồn, có nhiều sự kiện lặp lại và cần suy luận theo thời gian; Flat RAG sẽ bị vỡ trên multi-hop và temporal reasoning.
- Nodes dự kiến: `Company`, `Product`, `Person`, `Technology`, `Event`.
- Relations dự kiến: `ACQUIRED`, `PARTNERED_WITH`, `LAUNCHED`, `INTEGRATED_WITH`, `USES`, `OFFERS`.
- Chiến lược entity resolution: thu thập alias map cho các tên công ty, gộp theo vector + lexical guard, và giữ audit log cho các pair bị reject.
- Chiến lược super-node: ưu tiên các edge mới nhất, giới hạn độ sâu hop và áp dụng community summary cho các node lớn.

## 4. Tổng kết
Bài lab làm rõ rằng GraphRAG chỉ thực sự hiệu quả khi được xây dựng với kiểm soát chặt chẽ: dữ liệu sạch, chứng cứ rõ, và policy ngăn xung đột. Đó là nền tảng cho một hệ thống production-grade, không chỉ là một demo AI.
