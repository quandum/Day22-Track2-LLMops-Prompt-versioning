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

- [ ] Đã tạo vectorstore thành công
- [ ] Đã chạy 50 câu hỏi qua RAG chain
- [ ] Đã xác nhận ≥50 traces trên LangSmith dashboard

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

- [ ] Đã push 2 prompt lên LangSmith Prompt Hub
- [ ] Đã pull prompt từ Hub khi chạy
- [ ] Đã chạy A/B routing trên 50 câu hỏi
- [ ] Console log hiển thị nhãn phiên bản (v1/v2)

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

| Chỉ số | V1 (Ngắn gọn) | V2 (Cấu trúc) | Winner |
|--------|:-------------:|:-------------:|:------:|
| Faithfulness | _chờ kết quả_ | _chờ kết quả_ | |
| Answer Relevancy | _chờ kết quả_ | _chờ kết quả_ | |
| Context Recall | _chờ kết quả_ | _chờ kết quả_ | |
| Context Precision | _chờ kết quả_ | _chờ kết quả_ | |

**Mục tiêu:** Faithfulness ≥ 0.8 cho ít nhất một phiên bản.

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
| 1 | RAG Pipeline với LangSmith | 25đ | |
| 2 | Prompt Hub & A/B Routing | 25đ | |
| 3 | RAGAS Evaluation | 25đ | |
| 4 | Guardrails AI Validators | 25đ | |
| **Tổng** | | **100đ** | |

---

## 7. Phân tích & nhận xét

### So sánh V1 vs V2

_(Điền sau khi có kết quả RAGAS)_

- **V1 (Ngắn gọn):** ...
- **V2 (Cấu trúc):** ...
- **Kết luận:** ...

### Khó khăn gặp phải

1. ...
2. ...

### Bài học rút ra

1. ...
2. ...

---

## 8. Danh sách bằng chứng

| # | File | Mô tả | Trạng thái |
|---|------|-------|:----------:|
| 1 | `evidence/01_langsmith_traces.png` | LangSmith dashboard với ≥50 traces | ⬜ |
| 2 | `evidence/02_prompt_hub.png` | Prompt Hub với 2 phiên bản | ⬜ |
| 3 | `evidence/02_ab_routing_log.txt` | Log console A/B routing | ⬜ |
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
