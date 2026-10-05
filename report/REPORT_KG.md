# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Hoàng Trung Hiếu  **MSSV:** 2A202602945  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     55.1
graph       196     91958     4702   0.00933    124.1

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     1.72
graph       0.69   1.33     3254       76   0.00053     2.54
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | 0.00112 | 0.00933 | ×8.33 |
| Indexing giây | 55.1 | 124.1 | ×2.25 |
| Mỗi câu: USD | 0.00013 | 0.00053 | ×4.08 |
| Mỗi câu: giây | 1.72 | 2.54 | ×1.48 |
| Mỗi câu: in_tok | 694 | 3254 | ×4.69 |

**Chi phí tăng thêm đến từ đâu?**
> Chi phí tăng thêm của GraphRAG ở giai đoạn **Indexing** đến chủ yếu từ việc gọi LLM (`gpt-4o-mini`) để trích xuất có cấu trúc (JSON entities, relations, cases) trên toàn bộ 20 bài báo tin tức (tốn 4.702 output tokens và ~35.800 input tokens bổ sung ngoài embedding thông thường). Ở giai đoạn **Querying**, chi phí tăng gấp ~4 lần do prompt ngữ cảnh được mở rộng: ngoài các chunks văn bản từ vector search, hệ thống bổ sung thêm danh sách facts văn bản trích xuất từ đồ thị tri thức (multi-hop graph facts), khiến số lượng token đầu vào trung bình mỗi câu hỏi tăng từ 694 lên 3.254 tokens (gấp ~4.7 lần).

---

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| **Q1** | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai pipeline đều lấy đúng và đủ định nghĩa tiền chất trong Luật PCMT qua vector chunk hoặc node luật Điều 2. |
| **Q2** | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Bản án tử hình của Trần Thanh Tuấn và Trần Minh Tâm nằm trọn vẹn trong một bài báo nên Flat RAG và GraphRAG đều truy xuất hoàn hảo. |
| **Q3** | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Flat RAG bị đứt liên kết giữa bài báo về Lê Minh Thành và Điều 251 BLHS nên báo "Không đủ thông tin", trong khi GraphRAG duyệt đồ thị multi-hop trả lời chính xác trọn vẹn. |
| **Q4** | cross-kb | 0.00 / 0 | 0.00 / 0 | Hòa (cùng 0) | Cả hai pipeline đều trả lời "Không đủ thông tin" do thực thể được hỏi bằng biệt danh "Hoàng Nato" và đối tượng liên quan nhiều đường dây với nhiều tội danh khác nhau. |
| **Q5** | cross-kb-multi-hop | 0.60 / 1 | 0.80 / 1 | Graph | GraphRAG vượt trội về recall (0.80 vs 0.60) nhờ duyệt qua đồ thị liên kết Cái Quang Huy $\to$ vận chuyển $\to$ Điều 250 và các khung phạt tù kịch khung (20 năm, chung thân, tử hình). |
| **Q6** | aggregation | 0.00 / 1 | 0.33 / 1 | Graph | GraphRAG thắng về recall (0.33 vs 0.00) nhờ cơ chế truy vấn đồ thị gom đủ cả 3 vụ án có liên quan đến ma túy MDMA (Huy, Thành, và Viện Pháp y tâm thần Trung ương) trong khi Flat RAG chỉ lấy được tên vụ rời rạc. |

---

## 3. Phân tích lỗi (20 điểm)

### Lỗi E1: Cầu nối gãy — Vụ án không nối được sang luật

- **Hiện tượng:** Có vụ án trong tin tức báo chí được trích xuất thành node `Case` nhưng không tạo được bất kỳ quan hệ `CHARGED_WITH` nào tới node `Crime`, khiến vụ án bị cô lập hoàn toàn và không thể duyệt multi-hop sang KB luật.
- **Bằng chứng:** Truy vấn Cypher tìm các node `Case` không có quan hệ `CHARGED_WITH`:

```cypher
MATCH (k:Case) WHERE NOT (k)-[:CHARGED_WITH]->() RETURN k.name AS name, k.doc_id AS doc_id
```

```
name: "Vụ tông cảnh sát giao thông ở An Giang"
doc_id: "news-100260926112415229"
```

Mở bài báo gốc `data/drug_news/news-100260926112415229.md`: Đây là bài báo phản ánh đối tượng tông xe vào tổ tuần tra giao thông, sau đó công an phát hiện chất bột màu trắng nghi ma túy trong túi và đang tạm giữ để trưng cầu giám định.
- **Nguyên nhân:** Nằm ở khâu nội dung bài báo và hàm `link_entity`: Tại thời điểm đưa tin, cơ quan chức năng chưa khởi tố bị can về tội danh cụ thể trong BLHS (chỉ tạm giữ điều tra hành vi chống người thi hành công vụ và giám định chất bột). LLM không tìm thấy tội danh ma túy chuẩn trong văn bản nên trả về danh sách `charges` rỗng, hoặc trích xuất tên tội không nằm trong danh mục luật khiến `link_entity` trả về `None`.
- **Đề xuất sửa:**
  1. Trong ontology: Bổ sung thêm thuộc tính trạng thái tố tụng `status` trên `Case` (`"đang_điều_tra"` / `"đã_khởi_tố"`) và cho phép liên kết dự phòng sang luật qua node `Substance` (`INVOLVES` $\to$ `Substance` $\leftarrow$ `MENTIONS` $\leftarrow$ `Clause`).
  2. Fallback sang vector search khi `charges` bị rỗng để tận dụng ngữ cảnh văn bản gốc.
  *Đánh đổi*: Nếu liên kết dự phòng chỉ qua `Substance` mà chưa biết rõ hành vi (tàng trữ, vận chuyển hay mua bán), đồ thị sẽ trích xuất quá nhiều Điều luật khả dĩ, làm tăng token và gây nhiễu prompt cho LLM.

---

### Lỗi E3: Trùng thực thể — Tên chất ma túy bị phân mảnh thành nhiều node

- **Hiện tượng:** Cùng một chất ma túy ngoài đời thực nhưng bị phân tách thành nhiều node `Substance` khác nhau trong đồ thị do viết hoa/thường hoặc dùng tên lóng/tên thương mại không đồng nhất giữa 2 KB.
- **Bằng chứng:** Truy vấn Cypher liệt kê danh sách các node `Substance` trong cơ sở dữ liệu:

```cypher
MATCH (s:Substance) RETURN s.name AS name ORDER BY toLower(s.name)
```

```
[
  'Amphetamine', 'chất ma túy', 'Cocaine', 'côca', 'cần sa', 
  'etomidate', 'Heroine', 'Ketamine', 'ketamine', 'ma túy', 
  'ma túy tổng hợp', 'MDMA', 'Methamphetamine', 'methamphetamine', 
  'thuốc lắc', 'thuốc phiện', 'XLR-11'
]
```

Nhận thấy rõ ràng sự trùng lặp và phân mảnh:
- `'Ketamine'` (viết hoa từ luật) và `'ketamine'` (viết thường từ báo).
- `'Methamphetamine'` và `'methamphetamine'`.
- `'thuốc lắc'` (tiếng lóng báo chí) và `'MDMA'` (tên khoa học trong Bộ luật Hình sự).
- Các cụm từ chung chung như `'chất ma túy'`, `'ma túy'`, `'ma túy tổng hợp'` cũng bị biến thành node `Substance`.

- **Nguyên nhân:**
  1. Lệnh tạo node `MERGE (sub:Substance {name: s.name})` trên Neo4j phân biệt hoa thường (`case-sensitive`).
  2. KB Luật sử dụng danh pháp pháp lý/hóa học chuẩn (`MDMA`), trong khi phóng viên viết bài thường sử dụng tên đường phố/tiếng lóng (`thuốc lắc`, `kẹo`, `đá`).
  3. Hàm `link_entity` ở baseline chỉ mới được áp dụng cho tội danh (`Crime`), chưa được áp dụng để chuẩn hóa tên chất (`Substance`).
- **Đề xuất sửa:**
  1. Viết hàm `normalize_substance` đưa tên chất về chữ thường và tích hợp từ điển từ đồng nghĩa (synonym dictionary: `"thuốc lắc"` $\to$ `"MDMA"`, `"đá"` $\to$ `"Methamphetamine"`).
  2. Bổ sung hàm `link_entity` cho `Substance` trước khi thực hiện `MERGE` vào đồ thị.
  *Đánh đổi*: Cần duy trì và cập nhật liên tục từ điển tiếng lóng ma túy; nếu quy đổi sai các loại ma túy pha trộn (ví dụ nước vui, trà sữa chứa nhiều hợp chất) có thể dẫn tới sai lệch định khung hình phạt.

---

## 4. Kết luận (5 điểm)

**Khi nào nên dùng KG, khi nào Flat RAG là đủ?**

1. **Khi nào Flat RAG là đủ:**
   - Khi các bài toán tra cứu thuộc dạng **đơn chặng (single-hop)**, trong đó câu hỏi và thông tin giải đáp nằm trọn vẹn trong một văn bản hoặc một đoạn trích cục bộ (như câu Q1 và Q2).
   - Ở hai câu này, Flat RAG đạt kết quả hoàn hảo (**recall 1.00, judge 2.0**) với chi phí rẻ hơn **8.33 lần** ở khâu Indexing ($0.00112 so với $0.00933) và thời gian phản hồi nhanh hơn **1.48 lần** (1.72s so với 2.54s). Với các hệ thống FAQ, tra cứu chính sách đơn giản, Flat RAG là lựa chọn tối ưu về chi phí và tài nguyên.

2. **Khi nào bắt buộc phải dùng Knowledge Graph (GraphRAG):**
   - Khi nghiệp vụ đòi hỏi suy luận **liên kết chéo (Cross-KB)** và **đa chặng (Multi-hop)** giữa các nguồn tri thức tách rời (như liên kết giữa đối tượng/vụ án trong tin tức và các khung khoản quy định trong Bộ luật Hình sự ở Q3 và Q5). Ở câu Q3, Flat RAG hoàn toàn thất bại (recall 0.00, judge 0 - "Không đủ thông tin"), trong khi GraphRAG đạt độ chính xác tuyệt đối (**recall 1.00, judge 2.0**).
   - Khi câu hỏi yêu cầu **tổng hợp toàn diện (Aggregation)** trên nhiều tài liệu (như câu Q6 tìm tất cả vụ việc liên quan đến MDMA). GraphRAG giúp gom nhóm quan hệ trên toàn đồ thị vượt qua giới hạn độ dài của top-k vector chunk (recall GraphRAG 0.33 so với Flat RAG 0.00).
   - Tổng thể trên toàn bộ tập benchmark, GraphRAG nâng recall trung bình từ **0.43 lên 0.69** (+60%) và điểm LLM Judge từ **1.00 lên 1.33** (+33%), là sự đầu tư hoàn toàn xứng đáng cho các hệ thống pháp lý, phân tích tội phạm và điều tra tri thức chuyên sâu.

---

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.14s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openai:gpt-4o-mini | embedding = openai:text-embedding-3-small
[OK] KG-2 build_graph: 146 node / 289 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 13 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00064. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.  
Người đã chọn cho `kg_my_case.png`: **Cái Quang Huy** (Vụ vận chuyển ma túy từ Đức về Việt Nam qua sân bay Nội Bài).

---

## Vấn đề gặp phải (không tính điểm)

- File `.env` ban đầu khi tạo bằng PowerShell trên Windows bị dính tiền tố Byte Order Mark (`UTF-8 BOM` `\xef\xbb\xbf`), khiến thư viện `python-dotenv` đọc biến môi trường thành `\ufeffOPENAI_API_KEY` và báo lỗi thiếu key. Đã xử lý triệt để bằng script Python viết lại file theo định dạng `utf-8` chuẩn không BOM.
- Docker Desktop trên Windows cần được khởi động ở phiên bản GUI của người dùng để mở socket `npipe` cho container Neo4j kết nối ổn định.
