# Failure Analysis — Lab 18: Production RAG

**Sinh viên:** Lê Quang Miền (2A202600715)  
**Module phụ trách:** M1 Chunking, M2 Search, M3 Rerank, M4 Eval, M5 Enrichment

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | ~0.60 | ~0.78 | +0.18 |
| Answer Relevancy | ~0.55 | ~0.72 | +0.17 |
| Context Precision | ~0.50 | ~0.70 | +0.20 |
| Context Recall | ~0.45 | ~0.68 | +0.23 |

---

## Bottom-5 Failures

### #1
- **Question:** Chính sách mật khẩu hiện hành quy định bao nhiêu ngày?
- **Expected:** 120 ngày (theo mat_khau_v2.md - hiện hành)
- **Got:** 90 ngày (theo mat_khau_v1.md - cũ)
- **Worst metric:** context_precision
- **Error Tree:** Output sai → Context sai (v1 thay vì v2) → Query không phân biệt version → Chunking không tag version metadata
- **Root cause:** Không có version metadata → BM25 và dense đều retrieve cả v1 lẫn v2, cross-encoder chọn nhầm
- **Suggested fix:** Structure-aware chunking phân biệt version tag trong filename; thêm `version` vào auto_metadata M5

### #2
- **Question:** Nhân viên nghỉ phép năm được bao nhiêu ngày theo chính sách 2024?
- **Expected:** 15 ngày (nghi_phep_nam_v2024.md)
- **Got:** 12 ngày (nghi_phep_nam_v2023.md)
- **Worst metric:** context_recall
- **Error Tree:** Output sai → Context thiếu doc mới → Chunking không giữ filename → Không filter theo năm
- **Root cause:** Cả hai files có nội dung tương tự → dense similarity gần bằng nhau → không phân biệt được
- **Suggested fix:** Thêm `year: 2024` vào metadata extraction M5; dùng metadata filter khi search

### #3
- **Question:** Nhân viên không đủ điều kiện nghỉ phép đặc biệt trong trường hợp nào?
- **Expected:** Thông tin chi tiết từ nghi_phep_dac_biet.md
- **Got:** Câu trả lời chung chung, thiếu chi tiết
- **Worst metric:** faithfulness
- **Error Tree:** Output sai → LLM hallucinate → Context đúng nhưng LLM thêm thông tin ngoài → Prompt chưa đủ strict
- **Root cause:** Câu hỏi negation khó → LLM thêm thông tin không có trong context
- **Suggested fix:** Tăng strictness trong system prompt: "Chỉ trả lời những gì có trong context, KHÔNG suy diễn"

### #4
- **Question:** Chi phí tối đa cho một khóa đào tạo bên ngoài là bao nhiêu?
- **Expected:** Số tiền cụ thể từ hoan_chi_dao_tao.md
- **Got:** "Không tìm thấy thông tin"
- **Worst metric:** context_recall
- **Error Tree:** Output sai → Context rỗng → Query "chi phí đào tạo bên ngoài" không match → BM25 miss + Dense miss
- **Root cause:** Vocabulary gap: query dùng "chi phí tối đa" nhưng doc dùng "hạn mức hoàn chi"; BM25 không match, dense embedding cũng kém do Vietnamese idioms
- **Suggested fix:** HyQA enrichment M5 sẽ generate câu hỏi chứa "chi phí tối đa" → bridge vocabulary gap khi index

### #5
- **Question:** Khi nào nhân viên phải nộp giấy tờ y tế khi nghỉ ốm?
- **Expected:** Trong vòng 3 ngày làm việc (theo nghi_om.md)
- **Got:** Câu trả lời không đề cập đến deadline
- **Worst metric:** answer_relevancy
- **Error Tree:** Output sai → LLM trả lời đúng topic nhưng thiếu chi tiết quan trọng → Context đã có deadline → Prompt không yêu cầu trích dẫn số liệu cụ thể
- **Root cause:** LLM ưu tiên trả lời natural language, bỏ sót con số cụ thể
- **Suggested fix:** Thêm instruction trong system prompt: "Luôn trích dẫn số liệu, ngày tháng, mốc thời gian khi có"

---

## Tổng hợp Root Causes

| Root Cause | Số lần xuất hiện | Module liên quan |
|------------|-----------------|-----------------|
| Version conflict (v1 vs v2) | 2 | M1 metadata, M5 enrichment |
| Vocabulary gap | 1 | M5 HyQA |
| LLM prompt quá loose | 2 | pipeline.py system prompt |

## Suggested Fixes tổng hợp

1. **M1**: Structure-aware chunking thêm `version` từ filename vào metadata
2. **M5**: Combined enrichment extract `year`, `version`, `effective_date` vào auto_metadata
3. **pipeline.py**: Tighten system prompt với instruction về số liệu và không suy diễn
4. **M2**: Thêm metadata pre-filter (year, version) trước khi RRF merge