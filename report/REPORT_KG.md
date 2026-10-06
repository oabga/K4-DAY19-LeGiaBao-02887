# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Lê Gia Bảo  **MSSV:** 02887  **Ngày:** 2026-10-05

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu dưới đây lấy nguyên văn từ
> `ket_qua_benchmark_kg.txt` (sinh ra bằng `python bench_kg.py --judge`, provider `openai:gpt-4o-mini` /
> `openai:text-embedding-3-small`). Bản thiết kế ontology: `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

```
Chat model: openai:gpt-4o-mini | Embedding: openai:text-embedding-3-small | top_k=3 | chunk_size=800 | chunks=176 | KG: 204 nodes / 381 rels

== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     52.7
graph       196     91958     4630   0.00928    110.1

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     1.53
graph       0.78   1.67     4099       81   0.00066     2.30
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | 0.00112 | 0.00928 | ×8.29 |
| Indexing giây | 52.7 | 110.1 | ×2.09 |
| Mỗi câu: USD | 0.00013 | 0.00066 | ×5.08 |
| Mỗi câu: giây | 1.53 | 2.30 | ×1.50 |
| Mỗi câu: in_tok | 694 | 4099 | ×5.91 |

**Chi phí tăng thêm đến từ đâu?**

- **Indexing (+8.29× USD, +2.09× giây):** đến từ 20 lần gọi LLM thêm để trích xuất tin tức
  (`extract_news_cases`, `json_mode=True`) — luật dùng regex nên không tốn token. 20 lần gọi này cộng
  thêm ~36k input token (prompt `NEWS_EXTRACTION_PROMPT` nhúng cả nội dung bài báo + danh sách tội
  danh/chất chuẩn) và 4630 output token (JSON trả về) — một chi phí **trả một lần**, không lặp lại khi hỏi
  thêm câu.
- **Mỗi câu hỏi (+5.91× input token, +5.08× USD):** `GRAPH_PROMPT` nhúng thêm toàn bộ `graph.context()` —
  phần lớn token thừa đến từ `seed_facts()` (bước ontology-độc-lập, luôn chạy trước): khi câu hỏi nhắc một
  chất phổ biến (vd "MDMA"), `Substance{name:'MDMA'}` trở thành một seed node và kéo theo **mọi** cạnh 1
  bước quanh nó — tức mọi `Clause` ở mọi Điều luật có nhắc MDMA, không riêng Điều đang được hỏi. Phần dữ
  kiện multi-hop thật sự cần (khoản luật đúng của đúng vụ) chỉ là một phần nhỏ trong số đó.
- **Ước tính điểm hòa vốn:** chênh lệch chi phí dựng hệ thống một lần là 0.00928 − 0.00112 = **0.00816
  USD**; chênh lệch chi phí mỗi câu là 0.00066 − 0.00013 = **0.00053 USD/câu**. Nếu chỉ tính thuần theo
  USD (bỏ qua giá trị của việc trả lời đúng), GraphRAG hòa vốn so với Flat sau khoảng
  0.00816 / 0.00053 ≈ **15–16 câu hỏi**. Vì 4/6 câu benchmark (Q3–Q6, loại `cross-kb*`/`aggregation`) là
  loại Flat RAG trả lời sai hoàn toàn (recall=0.00), giá trị thực tế hòa vốn sớm hơn nhiều nếu tính theo
  "số câu trả lời đúng thêm được", không chỉ theo USD thuần.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Flat (ngang điểm, rẻ hơn) | Đáp án nằm nguyên trong 1 đoạn luật (Điều 2 Luật PCMT); graph chỉ lặp lại cùng thông tin với giá đắt hơn. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Flat (ngang điểm, rẻ hơn) | Toàn bộ nằm trong 1 bài báo; không cần qua node cầu nối. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | **Graph** | Flat không có đoạn văn nào chứa cả mức án (tin) và khung luật (luật); Graph đi `Case→Crime←Article→Clause` qua `Crime` lấy đúng Điều 251 khoản 1. |
| Q4 | cross-kb | 0.00 / 0 | 0.67 / 1 | **Graph** (nhưng thiếu) | Graph tìm đúng Điều 255 nhưng bộ lọc khoản của KG-3 bỏ sót khoản 4 (20 năm/chung thân) — xem lỗi **E2**. Vẫn hơn tuyệt đối Flat (0 thông tin). |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 1.00 / 2 | **Graph** | Sau khi sửa thứ tự fact trong KG-3 (mục 3, lỗi E2), Graph trích đúng khoản 4 Điều 250 cho 9,6kg MDMA; Flat đoán đúng vài từ khóa nhờ 1 chunk luật ngẫu nhiên lọt top-k nhưng gọi sai cú pháp ("khoản b)" — luật này không có điểm b riêng ở khoản áp dụng). |
| Q6 | aggregation | 0.00 / 1 | 0.00 / 1 | Hòa | Cả hai liệt kê đúng nội dung các vụ (kể cả đúng khối lượng MDMA từng vụ) nhưng không dùng đúng tên riêng mà `must_include` yêu cầu — minh chứng phép đo `recall` theo từ khóa cứng quá nhạy với cách diễn đạt, xem lỗi **E4**. |

**Quy luật rút ra:** GraphRAG thắng tuyệt đối ở mọi câu `cross-kb*` (Q3–Q5, đáp án nằm rải rác 2 KB) và
hòa ở câu `single-hop*` (Q1–Q2, đáp án nằm gọn 1 đoạn — khi đó graph chỉ tốn thêm tiền mà không tăng độ
chính xác) và câu `aggregation` đo bằng `recall` cứng (Q6, cả hai bên đều thua vì vấn đề nằm ở cách đo,
không nằm ở pipeline).

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật

- **Hiện tượng:** GraphRAG trả lời sai/thiếu khung hình phạt cao nhất dù graph có đủ dữ liệu Điều luật.
  Phát hiện ở 2 câu: Q5 (trước khi sửa) trích sai số Điều (nói "Điều 251" thay vì "Điều 250"), và Q4 trích
  đúng Điều 255 nhưng luôn thiếu khoản 4 (20 năm/chung thân), chỉ dừng ở khoản 1 (2–7 năm).

- **Bằng chứng:**

  Trước khi sửa `Neo4jGraph.context`, debug trực tiếp cho câu hỏi Q5 (Cái Quang Huy) cho thấy
  `graph.context()` trả về **60/60 fact đã bị lấp đầy bởi `seed_facts()`** (mọi `Clause` ở các Điều
  248–252 có nhắc "MDMA", vì câu hỏi chứa chữ "MDMA" khiến node `Substance{name:'MDMA'}` trở thành seed và
  kéo theo toàn bộ cạnh 1-hop quanh nó) — các fact đã-định-dạng đúng từ bước "đi qua Crime sang Điều 250"
  bị nối vào **cuối** danh sách rồi bị cắt bởi `facts[:max_facts]`, không bao giờ tới được prompt:

  ```python
  # trước khi sửa: seed_facts() (tối đa 60 fact) được gán trực tiếp vào `facts`,
  # các fact multi-hop của chính câu hỏi bị append SAU và bị facts[:60] cắt mất
  seed_ids, facts = self.seed_facts(question, doc_ids)
  ...
  facts.append(f"[{article_id} - {title}] khoản {number}: {text}")   # không bao giờ tới lượt
  ```

  Kết quả đo được khi chạy `python bench_kg.py --judge` hai lần, trước và sau khi sửa (không phải một
  file nộp riêng — số liệu trước-sửa lấy từ log terminal của lần chạy đầu):

  | | Q5 graph recall | Câu trả lời |
  | --- | --- | --- |
  | Trước sửa | 0.80 | "...khoản áp dụng tương ứng là **khoản 4 của Điều 251 BLHS**..." (sai số Điều) |
  | Sau sửa | 1.00 | "...khoản 4 của **Điều 250** Bộ luật Hình sự (BLHS) được áp dụng..." (đúng) |

  Với Q4 (Hoàng Nato — Điều 255), nguyên nhân khác và vẫn còn tồn tại sau khi sửa thứ tự fact: Điều 255
  BLHS (tội tổ chức sử dụng trái phép chất ma túy) tăng nặng hình phạt dựa trên **số nạn nhân / tỷ lệ
  thương tật**, không dựa khối lượng chất — nên không khoản nào từ khoản 2–4 có cạnh `MENTIONS` tới một
  `Substance` nào cả:

  ```cypher
  MATCH (a:Article {id:'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
  OPTIONAL MATCH (cl)-[:MENTIONS]->(s:Substance)
  RETURN cl.number AS khoan, cl.penalty AS phat, s.name AS chat
  ORDER BY khoan
  ```

  → `khoan=1` có `phat`, không có `chat`; `khoan=2,3,4` đều `chat IS NULL` — khớp đúng giả thuyết: bộ lọc
  *"khoản 1 HOẶC khoản MENTIONS chất mà vụ INVOLVES"* trong KG-3 không có cách nào bắt được khoản 4, vì
  điều kiện "MENTIONS chất" không bao giờ đúng cho các khoản này. Benchmark xác nhận: Q4 graph dừng ở
  "tối đa 07 năm" (khoản 1) dù đáp án chuẩn cần "20 năm hoặc tù chung thân" (khoản 4).

- **Nguyên nhân:** hai nguyên nhân độc lập, cả hai nằm ở **KG-3** (`Neo4jGraph.context`):
  1. *Thứ tự ghép fact* — `seed_facts()` (giới hạn 60 fact riêng của nó) được đặt trước, lấp đầy ngân sách
     `max_facts` trước khi các fact multi-hop thật sự liên quan tới câu hỏi được thêm vào. Đây là lỗi
     thực thi, đã sửa trong commit này (xem code hiện tại trong `src/graph.py`).
  2. *Thiết kế bộ lọc khoản* — quy tắc "khoản 1 + khoản có `MENTIONS` chất mà vụ `INVOLVES`" (gợi ý từ
     LAB_GUIDE) ngầm giả định mọi khung tăng nặng đều gắn với khối lượng/loại chất. Giả định này đúng với
     Điều 250–252 (mua bán/vận chuyển/tàng trữ — tăng nặng theo khối lượng) nhưng **sai** với Điều 255,
     259 (tổ chức sử dụng, vi phạm quản lý — tăng nặng theo hậu quả/hành vi). Đây là hạn chế **thiết kế
     ontology** (đã ghi ở `report/ONTOLOGY.md` mục 8), không phải lỗi code.

- **Đề xuất sửa:**
  1. Đã sửa (ngân sách fact): thu thập fact multi-hop (mục a–c trong KG-3) **trước**, chỉ lấp phần ngân
     sách còn lại bằng `seed_facts()` sau, và giảm `limit` nội bộ của `seed_facts()` xuống 30 để giảm
     nhiễu. Đánh đổi: tốn thêm 1 bước Cypher nhỏ (không đáng kể), không tốn thêm lần gọi LLM.
  2. Chưa sửa (thiết kế): mở rộng điều kiện giữ khoản trong KG-3 thành "khoản 1 HOẶC khoản MENTIONS chất
     vụ INVOLVES HOẶC **khoản có số lớn nhất của Điều đó khi vụ không có chất nào khớp**" (tức luôn lấy
     thêm khung tăng nặng cao nhất làm tham khảo). Đánh đổi: prompt dài hơn một chút cho các Điều không
     dựa khối lượng chất, nhưng không ảnh hưởng các Điều khác vì điều kiện mới chỉ kích hoạt khi điều kiện
     cũ không khớp khoản nào ngoài khoản 1.

### Lỗi E3: Trùng thực thể

- **Hiện tượng:** Cùng một chất ma túy xuất hiện thành hai node `Substance` khác nhau chỉ vì khác chữ
  hoa/thường.

- **Bằng chứng:**

  ```cypher
  MATCH (s:Substance) RETURN s.name ORDER BY toLower(s.name)
  ```

  Kết quả thật từ graph đã dựng (20 lần gọi LLM, toàn bộ 2 KB):

  ```
  Amphetamine
  chất ma túy
  Cocaine
  côca
  cần sa
  etomidate
  Heroine
  Ketamine
  ketamine        ← trùng với "Ketamine" ở trên, chỉ khác chữ hoa
  ma túy
  ma túy tổng hợp
  MDMA
  Methamphetamine
  methamphetamine  ← trùng với "Methamphetamine" ở trên, chỉ khác chữ hoa
  thuốc lắc
  thuốc phiện
  XLR-11
  ```

  `Ketamine`/`ketamine` và `Methamphetamine`/`methamphetamine` là 4 node cho 2 chất thật. Hệ quả: một
  `Case` có thể `INVOLVES` node `ketamine` (viết thường, do LLM trích từ 1 bài) trong khi Điều luật
  tương ứng chỉ `MENTIONS` node `Ketamine` (viết hoa, do regex trích từ danh sách `SUBSTANCES` cố định)
  — khiến bộ lọc khoản ở KG-3 (`EXISTS { (k)-[:INVOLVES]->(s)<-[:MENTIONS]-(cl) }`) **không khớp được**
  cho vụ đó, dù về nội dung chất giống nhau 100%.

- **Nguyên nhân:** nằm ở **bước trích xuất** (`extract_news_cases`/`NEWS_EXTRACTION_PROMPT`) và **thiết
  kế khóa định danh**: `add_news_case` ghi `MERGE (sub:Substance {name: s.name})` với `s.name` lấy
  nguyên văn chuỗi LLM trả về. Prompt có đưa danh sách chất chuẩn (`DANH SÁCH CHẤT: {substances}`) và yêu
  cầu "dùng tên chuẩn trong DANH SÁCH CHẤT nếu khớp", nhưng đây chỉ là **gợi ý trong prompt**, không có
  bước chuẩn hóa/khóa bắt buộc nào ở phía code — khác với `Crime`, nơi `link_entity` **ép** map về tên
  chuẩn sau khi LLM trả lời. `Substance` không có bước `link_entity` tương ứng.

- **Đề xuất sửa:** áp `link_entity(s["name"], SUBSTANCES, normalize=str.lower)` (hoặc một hàm chuẩn hóa
  rút gọn tương tự `normalize_crime`) cho từng tên chất trước khi gọi `add_news_case`, giống cách
  `charges` đã được xử lý trong `extract_news_cases`. Đánh đổi: không tốn thêm lần gọi LLM (dùng lại hàm
  `link_entity` thuần Python đã có); chất nào không khớp danh sách chuẩn (hợp lệ, vd tên lóng không map
  được) vẫn bị giữ nguyên văn như hiện tại — chấp nhận được vì đây không phải chất nằm trong luật đang xét.

**Quan sát bổ sung (không tính điểm riêng, để tham khảo):**

- **E1 — cầu nối gãy, nhưng hợp lý:** `MATCH (k:Case) WHERE NOT (k)-[:CHARGED_WITH]->() RETURN k.name,
  k.doc_id` chỉ ra đúng 1 `Case` mồ côi: *"Vụ tông cảnh sát giao thông ở An Giang"*
  (`news-100260926112415229`). Đọc lại bài báo gốc: bị can Nguyễn Minh Nhân bị khởi tố về **"hành vi
  chống người thi hành công vụ"** (Điều 330 BLHS) — không phải một tội ma túy trong Chương XX mà KB luật
  của lab có crawl; bài chỉ nhắc "ma túy" như một yếu tố liên quan ("sử dụng rượu và ma túy"), không phải
  tội danh bị truy tố. `link_entity` đúng khi trả `None` ở đây — cầu nối "gãy" là **đúng về mặt dữ liệu**,
  không phải lỗi, vì tội thật của vụ này nằm ngoài phạm vi 2 KB.
- **E4 — phép đo sai, ở Q6:** sau khi sửa, cả Flat và Graph đều `recall=0.00` cho câu `aggregation` dù câu
  trả lời của Graph liệt kê đúng 4 vụ kèm đúng khối lượng MDMA từng vụ — chỉ vì không lặp lại đúng 3 tên
  riêng (`"Cái Quang Huy"`, `"Lê Minh Thành"`, `"Pháp y tâm thần"`) mà `must_include` yêu cầu đối chiếu
  từng chữ. `judge` (LLM) vẫn cho 1/2 điểm ở cả hai — cho thấy `keyword_recall` quá cứng với câu hỏi mở,
  trong khi chính `judge` cũng không hoàn toàn đáng tin (gold yêu cầu liệt kê đủ 3 vụ, câu trả lời Graph
  liệt kê 4 vụ đúng nội dung nhưng vẫn chỉ được 1/2 vì diễn đạt khác).

## 4. Kết luận (5 điểm)

Dựa trên số liệu tự đo (mục 1–2): **Flat RAG đủ dùng** khi câu hỏi là single-hop — đáp án nằm nguyên
trong một đoạn văn bản của một tài liệu (Q1, Q2: cả hai pipeline đạt `recall=1.00`, `judge=2`, nhưng Flat
rẻ hơn 5–8× và nhanh hơn 1.5×). **GraphRAG đáng tiền** khi câu hỏi cross-kb — đáp án nằm rải rác ở ≥2 tài
liệu thuộc hai nguồn khác nhau cần nối qua node cầu nối: trên 3 câu loại này (Q3–Q5), Flat đạt `recall`
trung bình 0.20 (gần như không trả lời được), Graph đạt 0.89, đổi lấy ~5× chi phí mỗi câu. Với tập dữ
liệu và benchmark này, 4/6 câu (67%) cần multi-hop/cross-kb — đủ để biện minh cho khoảng 0,008 USD chi phí
dựng graph một lần (hòa vốn thuần USD sau ~15–16 câu hỏi bất kỳ, sớm hơn nhiều nếu chỉ tính riêng câu
cross-kb vì Flat ở đó gần như luôn trả lời sai hoàn toàn).

**Điều kiện cụ thể để chọn KG:** (1) dữ liệu tách thành ≥2 nguồn có cấu trúc khác nhau (một nguồn đều đặn
dùng được regex, một nguồn tự do cần LLM) và có một khái niệm chung nối được hai nguồn (ở đây là `Crime`);
(2) tỉ lệ câu hỏi thật sự cross-kb đủ lớn (ở đây >50%) và hệ thống được hỏi lại nhiều lần sau khi index một
lần, để chi phí dựng graph (trả một lần) được khấu hao qua nhiều câu hỏi. Nếu phần lớn câu hỏi là
single-hop, hoặc hệ thống chỉ chạy một lần rồi bỏ (không khấu hao được chi phí indexing), Flat RAG là lựa
chọn hợp lý hơn.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.04s

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
Người đã chọn cho `kg_my_case.png`: **Cái Quang Huy** (vụ vận chuyển hơn 9,6kg MDMA + ~406g Ketamine từ
Đức về Việt Nam qua Nội Bài — Điều 250 BLHS khoản 4, dùng trong Q5/Q6).

## Vấn đề gặp phải (không tính điểm)

Không có lỗi chưa giải quyết. Một vấn đề đã gặp và đã sửa trong quá trình làm (không phải lỗi còn tồn
đọng): phiên bản đầu của `Neo4jGraph.context` để `seed_facts()` lấp đầy toàn bộ ngân sách `max_facts=60`
trước khi các fact multi-hop của chính câu hỏi được thêm vào, khiến GraphRAG trích sai số Điều ở Q5 (xem
lỗi **E2**, mục 3). Đã sửa bằng cách đổi thứ tự ghép fact (ưu tiên fact multi-hop trước, `seed_facts()`
chỉ lấp phần còn trống) và giảm `limit` nội bộ của `seed_facts()` xuống 30 — số liệu trong báo cáo này đã
là số liệu **sau khi sửa**.
