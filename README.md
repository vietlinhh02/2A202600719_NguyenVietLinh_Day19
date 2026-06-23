# 2A202600719_NguyenVietLinh_Day19 — GraphRAG Lab Report

> **Lab Day 19 — Building a GraphRAG System & Benchmarking against Flat RAG**
> Student: Nguyễn Viết Linh · ID: 2A202600719

[![repo](https://img.shields.io/badge/repo-2A202600719__NguyenVietLinh__Day19-blue)](https://github.com/vietlinhh02/2A202600719_NguyenVietLinh_Day19)

---

## 1. Mục tiêu bài lab

Xây dựng một pipeline **GraphRAG** hoàn chỉnh từ một tập văn bản thô, sau đó so sánh chất lượng trả lời với baseline **Flat RAG** truyền thống trên cùng một bộ câu hỏi.

Cụ thể hệ thống GraphRAG cần:

1. **Entity & Relation Extraction** — chuyển văn bản phi cấu trúc thành các triple `(subject, relation, object)`.
2. **Graph Construction** — khử trùng lặp thực thể và dựng đồ thị tri thức bằng **NetworkX** (có thể mirror sang **Neo4j**).
3. **Indexing** — embed các chunk vào chỉ mục vector để truy hồi.
4. **GraphRAG Querying** — liên kết câu hỏi với các thực thể trong đồ thị, duyệt **multi-hop subgraph**, rồi sinh câu trả lời từ graph facts + supporting text.
5. **So sánh Flat RAG vs GraphRAG** trên 20 câu hỏi benchmark.

---

## 2. Cấu trúc repository

| File | Mô tả |
|------|-------|
| `graphrag_lab.ipynb` | Notebook chính — toàn bộ pipeline GraphRAG (extract → graph → index → query). |
| `benchmark_20_questions.csv` | Bộ benchmark 20 câu hỏi về EV / năng lượng tái tạo, kèm câu trả lời của GraphRAG và FlatRAG. |
| `download.png` | Ảnh minh họa kết quả truy hồi / đồ thị từ notebook. |
| `Screenshot From 2026-06-23 22-26-47.png` | Ảnh chụp màn hình kết quả chạy thực nghiệm. |
| `README.md` | Báo cáo tổng kết bài lab (file này). |

---

## 3. Phương pháp

### 3.1 Cấu hình

Notebook hỗ trợ **3 chế độ**:

- **Offline (mặc định)** — `LLM_PROVIDER="ollama"` + `GRAPH_BACKEND="networkx"`:
  ```
  ollama serve
  ollama pull llama3.1
  ollama pull nomic-embed-text
  ```
  → chạy được không cần API key, không cần database.
- **OpenAI** — đổi `LLM_PROVIDER="openai"` và export `OPENAI_API_KEY`.
- **Neo4j (tùy chọn)** — đổi `GRAPH_BACKEND="neo4j"` và điền thông tin đăng nhập.
- **LangExtract** (khuyến nghị) — đổi `EXTRACTION_BACKEND="langextext"` để extraction ổn định hơn nhờ few-shot examples.

### 3.2 Pipeline GraphRAG

```
Raw corpus
  │
  ▼  (1) Chunk + Embed chunks  ─────────►  Vector Index
  │
  ▼  (2) LLM / LangExtract  ─► (subject, relation, object) triples
  │
  ▼  (3) Deduplicate entities, build directed multigraph
  │
  ▼  (4) Query time:
        • embed question
        • link question → top-k entities (cosine)
        • traverse N-hop neighborhood in graph
        • collect (entity, relation, entity) facts + supporting chunks
        • LLM generates answer from facts + chunks (source-grounded)
```

### 3.3 Flat RAG baseline

- Chỉ dùng vector retrieval (top-k chunks).
- Prompt LLM trả lời trực tiếp từ các chunk này, **không có graph facts**.

---

## 4. Bộ Benchmark — 20 câu hỏi

Chủ đề: **xe điện (EV), sạc, chính sách, nhà sản xuất, công nghệ pin, đối thủ Mỹ–Trung, IRA…**

| # | Câu hỏi (rút gọn) |
|---|---|
| 1  | EV sector kết nối với hạ tầng sạc và chính sách chính phủ? |
| 2  | Các công ty nào được nhắc tới trong tăng trưởng EV, liên kết ra sao? |
| 3  | Xu hướng tài chính / sentiment với đầu tư năng lượng tái tạo? |
| 4  | Chính sách ưu đãi liên quan tới áp dụng EV theo vùng? |
| 5  | Quan hệ giữa Tesla và thị trường EV rộng hơn? |
| 6  | Quy định ảnh hưởng tới số lượng mẫu EV khả dụng? |
| 7  | Vùng / thành phố nào dẫn đầu áp dụng EV và vì sao? |
| 8  | NVIDIA đóng vai trò gì trong hệ sinh thái EV / xe tự lái? |
| 9  | Các hãng pin thích ứng với yêu cầu nội địa hoá như thế nào? |
| 10 | Thách thức chính của chuỗi cung ứng EV? |
| 11 | Cạnh tranh EV Mỹ–Trung đang tiến hoá ra sao? |
| 12 | Quy định ZEV ảnh hưởng tới thị phần? |
| 13 | Ưu đãi người dùng (tax credit) tác động tới doanh số EV? |
| 14 | Ý nghĩa các thông báo tài chính / sản xuất gần đây của Polestar? |
| 15 | Mật độ trạm sạc công cộng tương quan với tỉ lệ nhận EV? |
| 16 | Tốc độ tăng trưởng dự kiến của thị trường EV? |
| 17 | Các hãng xe truyền thống (Ford, GM) phản ứng với EV startup? |
| 18 | Công nghệ cụ thể nào đang thúc đẩy tiến bộ của EV? |
| 19 | Inflation Reduction Act ảnh hưởng tới sản xuất EV? |
| 20 | Khác biệt áp dụng EV giữa các bang có / không có ZEV? |

---

## 5. Kết quả — GraphRAG vs Flat RAG

### 5.1 Thống kê định lượng

| Chỉ số | GraphRAG | FlatRAG |
|---|---|---|
| **Tổng ký tự trả lời (20 câu)** | **22,782** | 11,712 |
| **Trung bình / câu** | **1,139** | 586 |
| **Tổng proper-noun + số liệu** | **135** | 58 |
| **Tỉ lệ mật độ thực thể** | **2.33×** FlatRAG | 1× |

→ GraphRAG trả lời **dài gấp ~1.94×** và **chứa nhiều thực thể + số liệu gấp ~2.33×** so với FlatRAG.

### 5.2 Độ dài câu trả lời theo từng câu hỏi

| # | GraphRAG (chars) | FlatRAG (chars) | Δ |
|---:|---:|---:|---:|
|  1 | 1731 | 1025 | **+706** |
|  2 | 1169 |  803 | +366 |
|  3 |  945 |  691 | +254 |
|  4 | 1889 | 1117 | **+772** |
|  5 | 1701 |  610 | +1091 |
|  6 |  929 |  377 | +552 |
|  7 | 1199 |  621 | +578 |
|  8 | 1424 |  333 | +1091 |
|  9 |  438 |  279 | +159 |
| 10 |  274 |  870 | **−596** |
| 11 | 1055 | 1131 | −76 |
| 12 |  228 |  453 | **−225** |
| 13 | 1163 |  727 | +436 |
| 14 |  667 |  351 | +316 |
| 15 | 1586 |  469 | +1117 |
| 16 |  225 |  279 | −54 |
| 17 | 1594 |  450 | +1144 |
| 18 | 1352 |  338 | +1014 |
| 19 | 1443 |  113 | **+1330** |
| 20 | 1770 |  675 | +1095 |

### 5.3 Phân tích định tính

**GraphRAG thắng rõ ở các câu multi-hop / cần kết nối thực thể:**

- **Q5 (Tesla ↔ EV market)** — GraphRAG đưa ra mạng lưới Tesla ↔ Ford ↔ GM ↔ BYD ↔ IRA; FlatRAG chỉ nói chung chung.
- **Q8 (NVIDIA role)** — GraphRAG liên kết NVIDIA ↔ DRIVE Orin ↔ ADAS ↔ EV; FlatRAG trả lời mỏng (333 ký tự).
- **Q15 (sạc ↔ uptake)** — GraphRAG nêu được tương quan định lượng giữa số cổng sạc/1000 dân và tỉ lệ nhận EV ở các bang.
- **Q17 (Ford/GM ↔ startup)** — GraphRAG liệt kê đầy đủ quan hệ (acquisition, JV, đầu tư); FlatRAG bỏ sót.
- **Q19 (Inflation Reduction Act)** — GraphRAG dài gấp ~12.8× FlatRAG (1443 vs 113 chars) — chứng tỏ graph chứa nhiều triple liên quan tới IRA (chính sách ↔ bang ↔ nhà sản xuất ↔ khoáng sản).

**FlatRAG "thắng" ở vài câu mang tính mô tả đơn giản / corpus đã đủ thông tin trong top-k chunk:**

- **Q10** (chuỗi cung ứng) — FlatRAG trả lời dài hơn vì chunk trực tiếp nói về logistics; GraphRAG over-fit vào 1 triple quá hẹp.
- **Q12, Q16** — FlatRAG chỉ cần định nghĩa + 1-2 con số, không cần multi-hop; câu trả lời ngắn của GraphRAG là do graph có ít triple liên quan.

### 5.4 Minh hoạ trực quan

Ảnh trong repo:

- `download.png` — visualization đồ thị tri thức (NetworkX / Neo4j) sau khi extract.
- `Screenshot From 2026-06-23 22-26-47.png` — kết quả chạy notebook (cell output + bảng benchmark).

---

## 6. Nhận xét & Bài học

### 6.1 Ưu điểm của GraphRAG

- **Multi-hop reasoning**: trả lời được câu hỏi cần đi qua nhiều thực thể (Tesla → Ford → IRA → bang California).
- **Source-grounded**: mỗi fact trong câu trả lời đều ánh xạ được về 1 triple trong graph → dễ debug / audit.
- **Mật độ thực thể cao hơn ~2.3×** → câu trả lời cụ thể, có số liệu, đỡ chung chung hơn FlatRAG.

### 6.2 Hạn chế & hướng cải tiến

- **Phụ thuộc chất lượng extraction**: nếu LLM bỏ sót entity/relation, graph nghèo → câu trả lời ngắn (Q10, Q12, Q16).  
  → Cải thiện bằng **LangExtract + few-shot examples** đã có trong `CONFIG`.
- **Latency cao hơn**: phải embed + link entity + traverse graph + LLM. Có thể cache embedding của entity.
- **Chi phí token LLM**: GraphRAG đưa nhiều fact vào prompt → tốn token hơn. Có thể lọc top-k triple theo relevance trước khi sinh câu trả lời.
- **Hybrid RAG** (graph + vector kết hợp, không thay thế) có thể cho kết quả tốt nhất trên cả 2 loại câu hỏi.

---

## 7. Hướng dẫn chạy lại

```bash
# 1. Clone repo
git clone https://github.com/vietlinhh02/2A202600719_NguyenVietLinh_Day19.git
cd 2A202600719_NguyenVietLinh_Day19

# 2. Cài đặt dependency
pip install -q networkx numpy pandas matplotlib requests tqdm \
                google-generativeai neo4j langextract

# 3. (Nếu chạy offline) cài Ollama + pull model
ollama serve &
ollama pull llama3.1
ollama pull nomic-embed-text

# 4. Mở notebook
jupyter notebook graphrag_lab.ipynb
```

Chạy lần lượt các cell trong notebook để:
1. Extract triples từ corpus.
2. Dựng graph (`build_graph()`).
3. Index vector.
4. Chạy hàm `ask_graphrag(question)` và `ask_flatrag(question)` trên 20 câu hỏi.
5. So sánh output — file `benchmark_20_questions.csv` trong repo là kết quả tham chiếu.

---

## 8. Kết luận

Trong bộ benchmark 20 câu hỏi về EV / năng lượng tái tạo:

- **GraphRAG** cho câu trả lời dài hơn **~1.94×**, mật độ thực thể + số liệu **~2.33×**, thể hiện rõ khả năng **multi-hop reasoning** và **source-grounding**.
- **FlatRAG** vẫn hữu ích cho các câu hỏi mô tả đơn giản mà top-k chunk đã đủ thông tin.
- Hướng phát triển tiếp theo: kết hợp **Hybrid RAG** (vector + graph + reranker) và dùng **LangExtract** để ổn định chất lượng triple extraction.

---

*Repo: <https://github.com/vietlinhh02/2A202600719_NguyenVietLinh_Day19>*
*Tác giả: Nguyễn Viết Linh — 2A202600719*
*Ngày: 2026-06-23*
