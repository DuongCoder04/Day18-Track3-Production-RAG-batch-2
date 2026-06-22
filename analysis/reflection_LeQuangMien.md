# Individual Reflection — Lab 18

**Tên:** Lê Quang Miền (2A202600715)  
**Module phụ trách:** M1 · M2 · M3 · M4 · M5 (cá nhân)

---

## 1. Đóng góp kỹ thuật

- **Module đã implement:** M1 Chunking, M2 Hybrid Search, M3 Reranking, M4 RAGAS Eval, M5 Enrichment
- **Các hàm/class chính đã viết:**
  - M1: `chunk_semantic()`, `chunk_hierarchical()`, `chunk_structure_aware()`
  - M2: `segment_vietnamese()`, `BM25Search.index/search()`, `DenseSearch.index/search()`, `reciprocal_rank_fusion()`
  - M3: `CrossEncoderReranker._load_model()`, `CrossEncoderReranker.rerank()`
  - M4: `evaluate_ragas()`, `failure_analysis()`
  - M5: `summarize_chunk()`, `generate_hypothesis_questions()`, `contextual_prepend()`, `extract_metadata()`, `_enrich_single_call()`
- **Số tests pass:** 32/32

---

## 2. Kiến thức học được

- **Khái niệm mới nhất:** Reciprocal Rank Fusion (RRF) — cách merge hai ranked list từ BM25 và dense mà không cần normalize scores về cùng scale. Công thức `1/(k + rank + 1)` đơn giản nhưng robust hơn weighted sum vì không bị dominate bởi outlier scores.

- **Điều bất ngờ nhất:** underthesea nối từ ghép tiếng Việt bằng `_` (ví dụ `nghỉ_phép`), khiến BM25 không match được query `nghỉ phép` (2 tokens) với corpus `nghỉ_phép` (1 token). Một dòng `.replace("_", " ")` fix toàn bộ vấn đề — bug "im lặng" nhất trong lab vì không raise error, chỉ trả về kết quả sai.

- **Kết nối với bài giảng:**
  - Slide "Why Hybrid > Dense-only": BM25 bắt exact match số liệu (12 ngày, 90 ngày) mà dense embedding bỏ sót → giải thích tại sao hybrid cần thiết cho corpus policy
  - Slide "Contextual Embeddings (Anthropic)": implement trực tiếp thành `contextual_prepend()` — chunk đứng độc lập thiếu context, prepend 1 câu mô tả → embedding vector "biết" chunk nằm ở đâu
  - Slide "RAGAS Diagnostic Tree": implement thành `failure_analysis()` với mapping metric → root cause → fix

---

## 3. Khó khăn & Cách giải quyết

- **Khó khăn lớn nhất:** qdrant-client v2.0 breaking change — `client.search()` không còn tồn tại, thay bằng `client.query_points()` với signature khác. Error message không rõ ràng (`AttributeError` thay vì deprecation warning).

- **Cách giải quyết:** Đọc CHANGELOG của qdrant-client trên GitHub, tìm migration guide từ v1 → v2. Fix: đổi sang `self.client.query_points(collection_name=collection, query=query_vector, limit=top_k)` và đọc result qua `.points` attribute.

- **Thời gian debug:** ~25 phút cho bug qdrant, ~10 phút cho bug underthesea `_`, ~15 phút cho CrossEncoder vs FlagReranker compatibility issue với transformers>=5.0.

---

## 4. Nếu làm lại

- **Sẽ làm khác điều gì:** Implement M5 enrichment trước M2 indexing — vì enriched text cần được index, không phải raw text. Thứ tự trong timeline (M1→M2→M3→M4→M5) khiến mình index raw chunks trước, sau đó phải re-index lại khi M5 xong. Thứ tự đúng về pipeline là M1→M5→M2→M3→M4.

- **Module nào muốn thử tiếp:** M4 với custom metrics — RAGAS faithfulness dùng LLM-as-judge khá tốn kém (1 OpenAI call/statement). Muốn thử implement lightweight version dùng NLI model local (cross-encoder/nli-deberta-v3-small) để eval offline không tốn API cost.

---

## 5. Tự đánh giá

| Tiêu chí | Tự chấm (1-5) |
|----------|---------------|
| Hiểu bài giảng | 4 |
| Code quality | 4 |
| Problem solving | 4 |
