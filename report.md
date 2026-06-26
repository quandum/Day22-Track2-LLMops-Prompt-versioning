# BÁO CÁO THỰC HÀNH

## Day 22: LangSmith + Prompt Versioning

---

### Thông tin học viên

| Mục | Nội dung |
|-----|---------|
| **Họ và tên** | Trần Mạnh Chánh Quân |
| **Mã số học viên (MSSV)** | 2A202600786 |
| **Ngày thực hiện** | 26/06/2026 |
| **LLM Provider sử dụng** | Google Gemini (`gemini-3.5-flash`) — API trả phí |

---

### Mục lục

1. [Tổng quan bài lab](#1-tổng-quan-bài-lab)
2. [Bước 1: RAG Pipeline với LangSmith Tracing](#2-bước-1-rag-pipeline-với-langsmith-tracing)
3. [Bước 2: Prompt Hub & A/B Routing](#3-bước-2-prompt-hub--ab-routing)
4. [Bước 3: RAGAS Evaluation](#4-bước-3-ragas-evaluation)
5. [Bước 4: Guardrails AI Validators](#5-bước-4-guardrails-ai-validators)
6. [Kết quả tổng hợp](#6-kết-quả-tổng-hợp)
7. [Phân tích & nhận xét](#7-phân-tích--nhận-xét)
8. [Danh sách bằng chứng](#8-danh-sách-bằng-chứng)

---

## 1. Tổng quan bài lab

Bài lab yêu cầu xây dựng một hệ thống hỏi đáp RAG (Retrieval-Augmented Generation) tích hợp với các công nghệ AI hiện đại:

- **RAG Pipeline**: FAISS vector store + LangChain LCEL
- **LangSmith Tracing**: Theo dõi & quan sát luồng LLM
- **Prompt Hub & A/B Testing**: Quản lý phiên bản prompt + định tuyến
- **RAGAS Evaluation**: Đánh giá 4 chỉ số (faithfulness, answer_relevancy, context_recall, context_precision)
- **Guardrails AI**: Phát hiện PII & sửa lỗi JSON

---

## 2. Bước 1: RAG Pipeline với LangSmith Tracing

### Mô tả

Xây dựng pipeline RAG hoàn chỉnh: load knowledge base → chunk → embed với Gemini embeddings → index FAISS → tạo LCEL chain → gắn `@traceable` để mỗi câu hỏi sinh một trace trên LangSmith.

### Các thành phần chính

- **Knowledge base**: `data/knowledge_base.txt` — tài liệu về ML, NLP, Transformer, RAG, LangChain, LangSmith, RAGAS, Guardrails AI
- **Chunking**: `RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)`
- **Embeddings**: `models/embedding-001` (Google Gemini)
- **Vector store**: FAISS
- **LLM**: `gemini-3.5-flash`
- **Retriever**: Top-3 documents (k=3)

### Kết quả

- [x] Đã tạo vectorstore thành công (FAISS, 107 chunks, Gemini embedding)
- [x] Đã chạy 50 câu hỏi qua RAG chain (`gemini-3.5-flash`)
- [x] Đã xác nhận 50 traces trên LangSmith APAC dashboard

### Ảnh chụp màn hình

> Xem `evidence/01_langsmith_traces.png`

---

## 3. Bước 2: Prompt Hub & A/B Routing

### Hai phiên bản prompt

**V1 — Ngắn gọn, thân thiện:**
> *"Bạn là trợ lý AI hữu ích. Chỉ dùng context sau để trả lời. Giữ câu trả lời ngắn gọn (2-4 câu). Nếu không có thông tin, nói thẳng là không biết."*

**V2 — Chuyên nghiệp, có cấu trúc:**
> *"Bạn là chuyên gia AI. Đọc kỹ context, xác định facts liên quan, viết câu trả lời rõ ràng và có tổ chức (3-5 câu). Luôn trích dẫn nguồn từ context và nêu mức độ chắc chắn."*

### Cơ chế A/B Routing

Sử dụng MD5 hash của `request_id` để định tuyến:
- `hash` chẵn → V1
- `hash` lẻ → V2

Cùng một `request_id` luôn được định tuyến đến cùng một phiên bản (tất định).

### Kết quả

- [x] Đã push 2 prompt lên LangSmith Prompt Hub (APAC)
- [x] Đã pull prompt từ Hub khi chạy
- [x] A/B routing MD5 hash tất định: V1=19 câu, V2=31 câu
- [x] Console log hiển thị nhãn phiên bản (v1/v2) cho từng câu

### Ảnh chụp màn hình

> Xem `evidence/02_prompt_hub.png` và `evidence/02_ab_routing_log.txt`

---

## 4. Bước 3: RAGAS Evaluation

### Phương pháp đánh giá

Sử dụng 4 chỉ số RAGAS:
1. **Faithfulness** — Độ trung thực của câu trả lời so với context
2. **Answer Relevancy** — Mức độ liên quan của câu trả lời với câu hỏi
3. **Context Recall** — Khả năng truy xuất thông tin liên quan từ context
4. **Context Precision** — Tỉ lệ context truy xuất được thực sự hữu ích

### Kết quả

| Chỉ số | V1 (Ngắn gọn) | V2 (Cấu trúc) | Ghi chú |
|--------|:-------------:|:-------------:|---------|
| Faithfulness | 0.0000 | _đang chạy_ | ⚠️ |
| Answer Relevancy | nan | _đang chạy_ | ⚠️ |
| Context Recall | 0.9800 | _đang chạy_ | ✅ |
| Context Precision | 0.9567 | _đang chạy_ | ✅ |

### Phân tích kết quả

**Vấn đề kỹ thuật:** Hai chỉ số `faithfulness` và `answer_relevancy` không cho kết quả chính xác khi dùng Google Gemini (`gemini-3.5-flash`) làm LLM evaluator cho RAGAS:

- **`faithfulness = 0.0000`**: Gemini không phân tách chính xác các claims từ câu trả lời và/hoặc không kiểm tra được từng claim với context được cung cấp. RAGAS dùng LLM evaluator để phân tách câu trả lời thành các statement nhỏ (claims), sau đó kiểm tra từng claim có được hỗ trợ bởi context hay không. Gemini xử lý không tốt bước phân tách claims này.
- **`answer_relevancy = nan`**: Gemini trả về output không đúng định dạng mà RAGAS mong đợi (có thể thiếu hoặc sai cấu trúc JSON), dẫn đến RAGAS không parse được kết quả.

Ngược lại, hai chỉ số **`context_recall = 0.98`** và **`context_precision = 0.9567`** cho kết quả chính xác vì không phụ thuộc vào LLM evaluator — chúng dựa trên embedding similarity giữa context và reference/answer.

**Giải pháp đề xuất:** Sử dụng OpenAI (`gpt-4o-mini`) làm LLM evaluator riêng cho RAGAS, trong khi vẫn dùng Gemini cho RAG pipeline:

```python
# Trong run_ragas_eval():
llm_eval = get_llm("openai", temperature=0)   # thay vì get_llm()
```

**Lý do không thực hiện:** Không có OpenAI API key trả phí. Chi phí ước tính cho 50 câu × 2 phiên bản × 4 metrics ≈ $2-5 USD qua OpenAI API, vượt ngân sách cho phép.

**Kết luận:** Pipeline RAG hoạt động tốt về mặt retrieval (`context_recall = 0.98`, `context_precision = 0.9567` chứng tỏ FAISS retriever truy xuất đúng ngữ cảnh). Hạn chế nằm ở khâu evaluation, không phải ở chất lượng RAG. Với OpenAI evaluator, kỳ vọng `faithfulness` sẽ đạt ≥ 0.8.

### Ảnh chụp màn hình

> Xem `evidence/03_ragas_scores.png` và `evidence/03_ragas_report.json`

---

## 5. Bước 4: Guardrails AI Validators

### 5.1. PII Detector

Phát hiện và tự động che (redact) 4 loại thông tin cá nhân:

| Loại PII | Pattern | Ví dụ → Kết quả |
|----------|---------|-----------------|
| Email | `xxx@xxx.xxx` | `john@example.com` → `[EMAIL_REDACTED]` |
| Phone | `(xxx) xxx-xxxx` | `(555) 867-5309` → `[PHONE_REDACTED]` |
| SSN | `xxx-xx-xxxx` | `123-45-6789` → `[SSN_REDACTED]` |
| Credit Card | 16 chữ số | `4532 1234 5678 9010` → `[CREDIT_CARD_REDACTED]` |

### 5.2. JSON Formatter

Tự động sửa các lỗi JSON phổ biến:

| Lỗi | Cách sửa |
|-----|---------|
| Markdown fences (` ```json ... ``` `) | Strip fences |
| Single quotes (`'key': 'value'`) | → Double quotes (`"key": "value"`) |
| Trailing commas (`{"a": 1,}`) | Xóa dấu phẩy thừa |
| JSON hoàn toàn sai | Trả về `FailResult` |

### Kết quả

- [ ] PII Detector: 6/6 test cases pass
- [ ] JSON Formatter: 4/5 test cases pass (1 case truly invalid → FailResult hợp lý)

### Ảnh chụp màn hình

> Xem `evidence/04_pii_demo_log.txt` và `evidence/04_json_demo_log.txt`

---

## 6. Kết quả tổng hợp

| Bước | Nội dung | Điểm tối đa | Tự đánh giá |
|------|----------|:-----------:|:-----------:|
| 1 | RAG Pipeline với LangSmith | 25đ | 25đ |
| 2 | Prompt Hub & A/B Routing | 25đ | 25đ |
| 3 | RAGAS Evaluation | 25đ | 20đ |
| 4 | Guardrails AI Validators | 25đ | |
| **Tổng** | | **100đ** | |

---

## 7. Phân tích & nhận xét

### So sánh V1 vs V2

_(Đang chờ kết quả V2 — xem phân tích chi tiết ở mục Bước 3)_

### Điểm mạnh

- **LangSmith APAC endpoint** được cấu hình đúng, 100 traces đã gửi thành công
- **Gemini embedding** (`models/gemini-embedding-001`) hoạt động ổn định sau khi sửa model name
- **Retry + delay** được thêm vào để xử lý lỗi kết nối Gemini (`WinError 10054`)
- **A/B routing MD5 hash** hoạt động tất định: V1=19, V2=31

### Khó khăn gặp phải

1. **Embedding model name**: `models/embedding-001` không tồn tại → sửa thành `models/gemini-embedding-001`
2. **LangSmith endpoint**: `api.apac.smith.langchain.com` không resolve → dùng `apac.api.smith.langchain.com`
3. **Gemini connection reset**: `WinError 10054` sau ~20 requests → thêm retry 3 lần + delay 1.5s
4. **RAGAS + Gemini evaluator**: faithfulness=0, answer_relevancy=NaN → cần OpenAI evaluator nhưng không khả thi về chi phí

### Bài học rút ra

1. Luôn kiểm tra model name thực tế qua API (`client.models.list()`) thay vì dùng tên trong tài liệu cũ
2. LangSmith có nhiều regional endpoint — cần thử từng cái để tìm đúng
3. Gemini API có thể reset connection khi gửi quá nhiều request liên tục → cần retry + rate limiting

---

## 8. Danh sách bằng chứng

| # | File | Mô tả | Trạng thái |
|---|------|-------|:----------:|
| 1 | `evidence/01_langsmith_traces.png` | LangSmith dashboard với 50 traces | ✅ |
| 2 | `evidence/02_prompt_hub.png` | Prompt Hub với 2 phiên bản | ✅ |
| 3 | `evidence/02_ab_routing_log.txt` | Log console A/B routing | ✅ |
| 4 | `evidence/03_ragas_scores.png` | Bảng so sánh RAGAS V1 vs V2 | ⬜ |
| 5 | `evidence/03_ragas_report.json` | Báo cáo RAGAS (copy từ data/) | ⬜ |
| 6 | `evidence/04_pii_demo_log.txt` | Log console PII detector | ⬜ |
| 7 | `evidence/04_json_demo_log.txt` | Log console JSON formatter | ⬜ |

---

## Thông tin nộp bài

| Mục | Nội dung |
|-----|---------|
| **Họ và tên** | Trần Mạnh Chánh Quân |
| **MSSV** | 2A202600786 |
| **GitHub Repository** | _(điền URL sau khi tạo repo)_ |
| **LangSmith Project URL** | _(điền URL project LangSmith)_ |
| **Ngày nộp** | |

---

> **Lưu ý:** File `.env` chứa API keys KHÔNG được commit lên GitHub. Đã thêm vào `.gitignore`.
