# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Lê Thanh Truong  **MSSV:** 2A202602492  **Ngày:** 2026-10-05

> Số liệu trong báo cáo này lấy từ `ket_qua_benchmark_kg.txt` (sinh bởi `python bench_kg.py --judge`).
> Bản thiết kế ontology ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Chat model: `aibox:qwen3.7-flash` | Embedding: `aibox:qwen3.7-text-embedding` | top_k=3 | chunk_size=800 | chunks=176 | KG: 212 nodes / 392 rels

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     35582        0   0.00050     61.0
graph       196     68110     6339   0.00166    125.0

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.62   1.00      728      171   0.00003     2.84
graph       0.83   1.83     3455      167   0.00008     2.80
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | 0.00050 | 0.00166 | ×3.32 |
| Indexing giây | 61.0 | 125.0 | ×2.05 |
| Mỗi câu: USD | 0.00003 | 0.00008 | ×2.67 |
| Mỗi câu: giây | 2.84 | 2.80 | ×0.99 |
| Mỗi câu: in_tok | 728 | 3455 | ×4.75 |

**Chi phí tăng thêm đến từ đâu?**

Indexing: graph tốn thêm **20 lần gọi LLM** (196 − 176) — đúng bằng 20 bài báo, mỗi bài một lần gọi
trích xuất. Số token tăng tương ứng **+32.528 in_tok** và **+6.339 out_tok** (JSON trích xuất trả về,
flat RAG không có phần này vì chỉ embed). Nói cách khác: **chi phí dựng graph = chi phí regex luật
(bằng 0) + chi phí LLM đọc toàn bộ KB tin**.

Querying: graph tốn **4,75× in_tok** vì mỗi prompt phải mang thêm khối dữ kiện từ graph — riêng các
khoản luật đã ≈2.000 token (xem quyết định 3 ở `ONTOLOGY.md`: tôi giữ nguyên văn khoản luật để có
ngưỡng khối lượng). Bù lại **độ trễ gần như bằng nhau** (2.80s vs 2.84s, ×0.99) và **out_tok không
tăng** (167 vs 171) — vì câu trả lời vẫn ngắn, chỉ có ngữ cảnh đầu vào dài hơn.

**Điểm hòa vốn:** chi phí dựng graph tăng thêm `$0.00116`, mỗi câu hỏi tăng thêm `$0.00005` →
`0.00116 / 0.00005 = **~23 câu hỏi**`. Hệ thống này chỉ chạy 6 câu, nên **Flat RAG rẻ hơn tuyệt đối**
ở quy mô lab. Graph chỉ bắt đầu rẻ hơn về tổng chi phí từ câu thứ ~23 trở đi.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Đáp án nằm gọn trong 1 chunk; Luật PCMT không có `Crime` nên graph không đóng góp gì |
| Q2 | single-hop-news | 1.00 / 1 | 1.00 / 2 | Graph | Cùng recall nhưng graph trả lời đủ ý hơn (judge 2 vs 1) nhờ có thêm Điều 251 trong ngữ cảnh |
| Q3 | cross-kb | 0.33 / 1 | **1.00 / 2** | **Graph** | Flat lấy nhầm Điều 258 ("lôi kéo") nên trả lời "không đủ thông tin"; graph đi cầu nối ra đúng Điều 251 |
| Q4 | cross-kb | 0.67 / 1 | 0.67 / 1 | Hòa (cùng sai) | Cả hai đều thiếu khung **tối đa**; graph trả lời tự tin "tối đa 07 năm" (sai, phải là tù chung thân) — lỗi E2 |
| Q5 | cross-kb-multi-hop | 0.40 / 0 | **1.00 / 2** | **Graph** | Flat bỏ cuộc ("không đủ thông tin"); graph lọc khoản 4 Điều 250 theo MDMA → đúng ngưỡng 100g + tử hình |
| Q6 | aggregation | 0.33 / 1 | 0.33 / 2 | Graph (judge) | Graph liệt kê **4 vụ trong đó 2 vụ là một** (trùng node, lỗi E3); flat liệt kê 3 vụ không trùng nhưng thiếu Điều luật |

**Quy luật rút ra:** *GraphRAG thắng khi câu hỏi đòi hỏi **nối hai KB** hoặc **gom nhiều nguồn**
(Q3 ×3 recall, Q5 ×2.5 recall); hòa khi đáp án nằm gọn trong một đoạn (Q1, Q2); và **có thể trả lời
sai tự tin hơn** khi ontology thiếu chiều thông tin (Q4) hoặc graph bị trùng thực thể (Q6).*

Đáng chú ý: `judge` và `recall` **bất đồng ở Q6**. Judge cho graph 2 điểm (câu trả lời đủ ý, văn phong
tốt) trong khi recall chỉ 0.33 — vì judge **không phát hiện được lỗi trùng vụ**. Đây là lỗi E4 (phép
đo sai) và là lý do phải đọc cả hai cột, không chỉ tin một cột.

## 3. Phân tích lỗi (20 điểm)

### Lỗi E1: Cầu nối gãy — 4/16 vụ án không nối được sang luật

- **Hiện tượng:** 4 `Case` không có đường đi tới bất kỳ `Article` nào, nên mọi câu hỏi
  `cross-kb` về các vụ này sẽ không nhận được Điều luật từ graph.
- **Bằng chứng:**

```cypher
MATCH (k:Case) WHERE NOT EXISTS { MATCH (k)-[:CHARGED_WITH]->(:Crime)<-[:DEFINES]-(:Article) }
RETURN k.name, k.doc_id;
```

```
'Bắt giữ Nguyễn Minh Đức'                          <- news-100260918220613301
'Chuyên án triệt phá đường dây ma túy liên tỉnh'   <- news-100260924101703641
'Vụ tông cảnh sát giao thông ở An Giang'           <- news-100260926112415229
'Biên phòng phát hiện bao tải dạt vào bờ biển...'  <- news-100260927182621527
```

- **Nguyên nhân:** **hai nguyên nhân khác nhau, phải tách ra** — đây là phần đáng chú ý nhất:
  - **Không phải lỗi (2 vụ):** `news-100260927182621527` là tin *bao tải ma túy trôi dạt*, chưa xác
    định được ai, chưa có tội danh → **đúng** là không nối được. `news-100260924101703641` có 2 `Case`
    trong cùng bài, một case nối được, case "chuyên án" là mô tả chuyên án chung.
  - **Lỗi trích xuất thật (2 vụ):** `news-100260918220613301` và `news-100260926112415229` — LLM tạo
    `Case` nhưng **không điền `charges`**, dù bài có mô tả hành vi. Với vụ An Giang, LLM còn gán chất
    là `"ma túy"` (chuỗi chung, không có trong `SUBSTANCES`) nên `Substance` cũng không tạo được.
- **Điều tra loại trừ:** tôi kiểm tra giả thuyết "cutoff 0.9 của KG-1 quá chặt nên chặn mất tội danh".
  **Sai.** Cả 3 bài không hề chứa tên tội danh chuẩn nào (`0 canonical charge name(s)`), nên không
  cutoff nào cứu được. Và `"tổ chức sử dụng trái phép chất ma túy"` — tội **có thật** — link đúng ở
  **1.0000**, hoàn toàn không bị cutoff chặn. Nâng 0.8 → 0.9 **không mất cạnh nào**.
- **Đề xuất sửa:** đây là lỗi ở **prompt trích xuất**, không phải ở `link_entity`. Sửa: cho LLM thấy
  **đoạn văn chứa hành vi** thay vì cả bài, và thêm ví dụ ít-shot cho trường hợp "mới bắt giữ, chưa
  khởi tố" → `charges: []`. *Đánh đổi:* prompt dài hơn ~300 token/bài × 20 bài, tăng chi phí indexing
  khoảng 15%.

### Lỗi E3: Trùng thực thể — một vụ án thành hai node `Case`

- **Hiện tượng:** vụ Viện Pháp y tâm thần Trung ương (được nhiều bài báo đưa tin) tồn tại thành **hai
  node `Case` riêng biệt**. Vì Q6 là câu hỏi **tổng hợp**, graph đếm vụ này **hai lần** trong câu trả lời.
- **Bằng chứng:**

```cypher
MATCH (k:Case) WHERE k.name CONTAINS 'Pháp y'
OPTIONAL MATCH (k)-[:INVOLVES]->(s:Substance)
RETURN k.name, k.doc_id, collect(DISTINCT s.name);
```

```
'Vụ tổ chức sử dụng trái phép chất ma túy tại Viện Pháp y tâm thần Trung ương và bãi biển Sầm Sơn'
    <- news-100260930085028036   substances: [cần sa, ketamine, methamphetamine, MDMA]
'Vụ án sai phạm tại Viện Pháp y tâm thần Trung ương'
    <- news-100260924105118645   substances: [Methamphetamine, cần sa, MDMA, Ketamine]
```

Chứng minh đây **cùng một vụ** — 3 người xuất hiện ở cả hai node:

```cypher
MATCH (p:Person)-[:INVOLVED_IN]->(k:Case) WHERE k.name CONTAINS 'Pháp y'
WITH p.name AS person, collect(DISTINCT k.name) AS cases
WHERE size(cases) > 1 RETURN person, cases;
```

```
Nguyễn Thị Mai Anh   [2 cases]     Lê Văn Đông   [2 cases]     Trần Quốc An   [2 cases]
```

- **Nguyên nhân:** **lỗi thiết kế ontology.** Khóa định danh của `Case` là `name` do **LLM tự đặt**,
  và LLM đặt tên khác nhau cho hai bài về cùng vụ (`MERGE` theo `name` không gộp được). Đây đúng là
  điểm yếu mà `LAB_GUIDE.md` Bước 2 đã cảnh báo ("`Case` và `Person` khóa theo tên do LLM tự đặt,
  nên dễ trùng").
- **Đề xuất sửa:** thêm `case_key` ổn định làm khóa `MERGE` — ví dụ `type + ngày + tỉnh`, hoặc
  `(charges, location, date)` chuẩn hóa. *Đánh đổi:* cần thêm một bước LLM sinh khóa (thêm ~20 lần
  gọi), và khóa ghép có thể gộp nhầm hai vụ khác nhau cùng ngày cùng tỉnh. Cách rẻ hơn: giữ `name`
  làm khóa nhưng thêm bước **dedup sau khi dựng graph** bằng Cypher `apoc.refactor.mergeNodes`.

### Lỗi E2: Thiếu ngữ cảnh luật — trả lời sai khung hình phạt *tối đa* dù graph có đủ Điều luật

- **Hiện tượng:** Q4 hỏi mức phạt tù **tối đa**. GraphRAG trả lời **"tối đa 07 năm"** — sai, đáp án
  đúng là **tù chung thân**. Câu trả lời **tự tin, có trích dẫn Điều luật đầy đủ**, nên trông rất
  đáng tin. Đây là lỗi nguy hiểm nhất trong 6 nhóm: sai mà không có dấu hiệu nào để người đọc nghi ngờ.
- **Bằng chứng** (trích nguyên văn `ket_qua_benchmark_kg.txt`, dòng `Q4 graph`):

> 2. **Mức phạt tù tối đa:** Theo **[Điều 255 BLHS - Tội tổ chức sử dụng trái phép chất ma túy]**
> khoản 1 được cung cấp trong dữ kiện, người nào tổ chức sử dụng trái phép chất ma túy thì bị phạt tù
> từ 02 năm đến 07 năm. Do đó, mức phạt tù tối đa là **07 năm**.

Đối chiếu đáp án chuẩn (`data/benchmark_kg.json`, Q4): *"khung cao nhất là tù 20 năm hoặc tù chung thân."*

Kiểm chứng graph có đủ dữ liệu — khoản 4 Điều 255 **có thật** trong graph, chỉ là không được lấy ra:

```cypher
MATCH (a:Article {id:'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
RETURN cl.number, cl.penalty ORDER BY cl.number;
```

```
1  | phạt tù từ 02 năm đến 07 năm
2  | phạt tù từ 07 năm đến 15 năm
3  | phạt tù từ 15 năm đến 20 năm
4  | phạt tù 20 năm hoặc tù chung thân        <- đáp án đúng, nằm trong graph
```

- **Nguyên nhân:** nằm ở **quy tắc lọc khoản trong KG-3** (`context()`), không phải ở graph và không
  phải ở LLM trả lời. Quy tắc của tôi chỉ giữ *khoản 1 + khoản `MENTIONS` chất có trong câu hỏi*.
  Q4 ("hành vi đó có thể bị phạt tù **tối đa** bao nhiêu") **không nhắc tên chất nào** → `substances`
  rỗng → chỉ khoản 1 lọt qua. Từ khóa quyết định ("tối đa") **không nằm trong cơ chế lọc**.
- **Đề xuất sửa:** bắt tín hiệu "tối đa / cao nhất / nặng nhất" trong câu hỏi và khi đó lấy **khoản
  cao nhất** thay vì khoản 1:
  `re.search(r"tối đa|cao nhất|nặng nhất", question)` → `ORDER BY cl.number DESC LIMIT 1`.
  *Đánh đổi:* thêm khoảng 1–2 khoản nữa vào prompt (mỗi khoản ~400 token), tức tăng in_tok mỗi câu
  khoảng 10–20% cho các câu dạng này — đổi lấy việc sửa một câu trả lời sai thành đúng.

## 4. Kết luận (5 điểm)

**Khi nào nên dùng Knowledge Graph:**

1. **Khi câu hỏi bắt buộc phải nối ≥2 nguồn dữ liệu.** Đây là điều kiện rõ nhất và có số liệu mạnh
   nhất: Q3 flat 0.33 → graph **1.00** (×3), Q5 flat 0.40 → graph **1.00** (×2.5). Ở Q3, Flat RAG
   không chỉ thiếu thông tin — nó lấy **nhầm Điều 258** ("lôi kéo người khác sử dụng") rồi kết luận
   "không đủ thông tin", tức là nó *thất bại một cách có ý thức*. Graph đi đúng cầu nối `Crime` và ra
   Điều 251. Lý do: không có đoạn văn nào chứa cả tên bị cáo *và* Điều luật, nên **không chunk nào**
   giúp được vector search — chỉ quan hệ mới nối được hai KB.

2. **Khi câu hỏi dạng tổng hợp/gom nhiều bài.** Q6 judge: graph 2 vs flat 1. Flat RAG chỉ thấy được
   các chunk trong top-3 của *một* truy vấn; graph đếm được trên toàn bộ tập `Case`.

**Khi nào Flat RAG là đủ (và nên chọn nó):**

1. **Khi đáp án nằm gọn trong một đoạn văn.** Q1 và Q2 hòa tuyệt đối (recall 1.00 cả hai, judge 2
   cả hai). Luật PCMT (Q1) thậm chí **không thể** nối vào graph vì không có `Crime` — thêm KG vào đây
   là chi phí thuần túy không đổi lại gì.

2. **Khi số câu hỏi còn ít.** Điểm hòa vốn đo được là **~23 câu** (mục 1). Lab này chạy 6 câu, nên
   Flat RAG rẻ hơn về tổng chi phí. Graph chỉ thắng về **chất lượng**, không thắng về tiền, ở quy mô này.

**Điều kiện cụ thể để chọn:** dùng KG khi (a) tỉ lệ câu hỏi `cross-kb`/`aggregation` trong workload
cao — ở đây 4/6 câu = 67%, và (b) số câu hỏi vượt ~23 để hoàn vốn chi phí dựng. Với dữ liệu này cả
hai điều kiện đều đúng, nên **KG đáng tiền** — nhưng phải chấp nhận đánh đổi: prompt dài gấp 4.75 lần,
và hai dạng lỗi mới mà Flat RAG không có (E2 sai khung tối đa, E3 trùng thực thể).

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.16s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = aibox:qwen3.7-flash | embedding = aibox:qwen3.7-text-embedding
[OK] KG-2 build_graph: 146 node / 289 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 10 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00009. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

> Lưu ý: `--check` **xoá graph và chỉ dựng lại luật + 1 bài báo**. Để xem graph đầy đủ trên Neo4j
> Browser (và để chụp ảnh), phải chạy `python bench_kg.py --build` sau đó — nếu không, ảnh Q-A sẽ chỉ
> hiện 1 `Case` thay vì 16.

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: **Cái Quang Huy** — nhân vật của Q5, câu `cross-kb-multi-hop`
mà GraphRAG thắng đậm nhất (recall 0.40 → 1.00), nên ảnh chứng minh trực tiếp đường đi đã tạo ra
chiến thắng đó: `Cái Quang Huy → Vụ vận chuyển ma túy từ Đức về Việt Nam → tội vận chuyển trái phép
chất ma túy → Điều 250 BLHS`.

## Vấn đề gặp phải (không tính điểm)

1. **Provider không có trên danh sách lab.** Lab hỗ trợ OpenAI/OpenRouter/Gemini/Anthropic; tôi dùng
   gateway OpenAI-compatible khác (`ai-box`). Đã thêm một mục vào `PROVIDERS` trong `src/llm.py`.
2. **Mọi model của gateway đều bật reasoning mặc định.** Một câu hỏi một dòng đốt **3608 reasoning
   token** và mất **28,9 giây** — với ~32 lần gọi LLM của lab thì vừa hết giờ vừa hỏng số liệu chi phí.
   Đã sửa bằng `EXTRA_BODY = {"aibox": {"enable_thinking": False}}` trong `src/llm.py`: còn **384
   token / 3,5 giây**, nhanh gấp 8 lần. (Phải dùng `extra_body=` của SDK `openai`; truyền thẳng
   `enable_thinking=False` bị SDK chặn.)
3. **Gateway trả JSON bọc trong ```` ```json ````.** `extract_news_cases` gọi `llm_fn(prompt)` không có
   `json_mode=True` → `json.loads` fail → hàm nuốt lỗi và trả `[]` → **toàn bộ KB tin biến mất, không
   một exception nào**. Đã sửa: `build_graph` truyền `lambda prompt: llm_fn(prompt, json_mode=True)`.
   *Đây là loại lỗi nguy hiểm nhất gặp phải: sai âm thầm.*
4. **Mất mạng giữa lúc chạy benchmark.** `httpx.ConnectError: getaddrinfo failed` sau khi đã embed
   được một phần. Không có retry nên cả lần chạy hỏng; phải chạy lại từ đầu.

5. **Hai bug "sai âm thầm" trong KG-3 do tôi tự tìm ra khi review lại code** — cả hai đều không crash,
   không exception, chỉ lặng lẽ cho kết quả thiếu. Đáng ghi lại vì chúng là dạng lỗi khó phát hiện nhất:
   - **`ENDS WITH` thay vì `STARTS WITH`:** `Article.id` là `"Điều 251 BLHS"`, tức số Điều nằm ở **đầu**
     chuỗi. Tôi viết `a.id ENDS WITH ('Điều ' + $n)` nên nhánh "câu hỏi nhắc thẳng Điều X" **chưa bao
     giờ chạy**. Phát hiện bằng cách test `context("Điều 250 quy định gì?", [])` → ra **0 fact**; sau
     khi sửa → **2 fact** đúng Điều 250.
   - **`MATCH (k)-[:INVOLVES]->(ks:Substance)` là bắt buộc:** vụ nào LLM không trích được chất thì mất
     **toàn bộ** Điều luật, kể cả khoản 1. Đây chính là cơ chế của lỗi E2 ở mục 3. Đã đổi thành
     `OPTIONAL MATCH`.

   Cả hai đều được sửa **trước** lần chạy benchmark cuối cùng, nên `ket_qua_benchmark_kg.txt` khớp với
   code hiện tại. (Một thay đổi nhỏ sau đó — bỏ một biến Cypher thừa trong `WITH` — đã được kiểm lại:
   `context()` cho cả 6 câu hỏi vẫn ra đúng các Điều như trước.)
