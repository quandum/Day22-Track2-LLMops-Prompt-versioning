# Kế Hoạch Thực Hiện — Day 22: LangSmith + Prompt Versioning

**Học viên:** Trần Mạnh Chánh Quân  
**MSSV:** 2A202600786  
**LLM Provider:** Google Gemini (`gemini-3.5-flash`) — API trả phí, không giới hạn quota  
**Ngày thực hiện:** 26/06/2026

---

## Tổng quan dự án

Lab gồm 4 bước, mỗi bước 25 điểm, tổng 100 điểm:

| Bước | Nội dung | Điểm | Thời gian dự kiến |
|------|----------|------|--------------------|
| 1 | RAG Pipeline với LangSmith Tracing | 25đ | 25–45 phút |
| 2 | Prompt Hub & A/B Routing | 25đ | 20–30 phút |
| 3 | RAGAS Evaluation | 25đ | 20–35 phút |
| 4 | Guardrails AI Validators | 25đ | 20–30 phút |

---

## BƯỚC 0: Chuẩn bị môi trường

### 0.1. Kiểm tra cấu hình `.env`

File `.env` hiện có đã được cấu hình với:
- `PROVIDER=gemini` — dùng Google Gemini
- `GOOGLE_API_KEY` — đã được lưu vào biến môi trường
- `GEMINI_MODEL=gemini-3.5-flash`
- `GEMINI_EMBEDDING_MODEL=models/embedding-001`
- `LANGSMITH_API_KEY` — đã có
- `LANGSMITH_PROJECT=day22-lab`
- `LANGCHAIN_TRACING_V2=true`

### 0.2. Tạo virtual environment & cài dependencies

```bash
python -m venv venv
venv\Scripts\activate          # Windows
pip install -r requirements.txt
```

**Danh sách packages cần cài:**
- `langchain`, `langchain-core`, `langchain-community`, `langchain-text-splitters`
- `langchain-openai`, `langchain-google-genai`
- `langsmith`
- `faiss-cpu`
- `ragas>=0.4.0`
- `guardrails-ai>=0.5.0`
- `python-dotenv`, `tiktoken`, `datasets`, `numpy`

### 0.3. Xác minh cài đặt

```bash
cd src && python config.py
```

Kết quả mong đợi: `✅ Config OK | Provider: GEMINI | Project: day22-lab`

---

## BƯỚC 1: RAG Pipeline với LangSmith Tracing (25 điểm)

**File cần sửa:** `src/01_langsmith_rag_pipeline.py`  
**Mục tiêu:** Tạo FAISS vectorstore, xây dựng RAG chain, gắn `@traceable`, tạo ≥50 traces trên LangSmith

### Checklist triển khai

| # | Việc cần làm | Chi tiết |
|---|-------------|----------|
| 1.1 | `setup_vectorstore()` | Gọi `get_embeddings()` → `load_knowledge_base()` → `split_text(text, chunk_size=500, chunk_overlap=50)` → `build_vectorstore(chunks, embeddings)` |
| 1.2 | `RAG_PROMPT` | Tạo `ChatPromptTemplate.from_messages([("system", "..."), ("human", "{question}")])` |
| 1.3 | `build_rag_chain()` | `retriever = vectorstore.as_retriever(search_kwargs={"k": 3})` → `format_docs()` nối doc.page_content → LCEL chain: `{"context": retriever \| format_docs, "question": RunnablePassthrough()} \| RAG_PROMPT \| llm \| StrOutputParser()` |
| 1.4 | `@traceable` decorator | `@traceable(name="rag-query")` đặt ngay trên `def ask(chain, question)` |
| 1.5 | Hàm `ask()` | `return chain.invoke(question)` |
| 1.6 | `main()` | Gọi `setup_vectorstore()` → `build_rag_chain()` → lặp 50 câu từ `SAMPLE_QUESTIONS` → gọi `ask()` → in kết quả |

### Chạy & kiểm tra

```bash
cd src && python 01_langsmith_rag_pipeline.py
```

### Bằng chứng cần thu thập

- [ ] Mở [smith.langchain.com](https://smith.langchain.com) → project `day22-lab` → tab Runs → chụp ảnh ≥50 traces → lưu `evidence/01_langsmith_traces.png`

---

## BƯỚC 2: Prompt Hub & A/B Routing (25 điểm)

**File cần sửa:** `src/02_prompt_hub_ab_routing.py`  
**Mục tiêu:** Tạo 2 system prompt, push lên LangSmith Prompt Hub, pull về, định tuyến A/B tất định bằng MD5 hash

### Checklist triển khai

| # | Việc cần làm | Chi tiết |
|---|-------------|----------|
| 2.1 | Đổi tên prompt | `PROMPT_V1_NAME = "tran-manh-chanh-quan-rag-v1"`, `PROMPT_V2_NAME = "tran-manh-chanh-quan-rag-v2"` |
| 2.2 | `SYSTEM_V1` | Phong cách ngắn gọn, thân thiện, 2-4 câu: *"Bạn là trợ lý AI hữu ích. Chỉ dùng context sau để trả lời. Giữ câu trả lời ngắn gọn (2-4 câu). Nếu không có thông tin, nói thẳng là không biết."* |
| 2.3 | `SYSTEM_V2` | Phong cách chuyên nghiệp, có cấu trúc, 3-5 câu: *"Bạn là chuyên gia AI. Đọc kỹ context, xác định facts liên quan, viết câu trả lời rõ ràng và có tổ chức (3-5 câu). Luôn trích dẫn nguồn từ context và nêu mức độ chắc chắn."* |
| 2.4 | `push_prompts_to_hub()` | Tạo `Client(api_key=...)` → `client.push_prompt(PROMPT_V1_NAME, object=PROMPT_V1, ...)` → `client.push_prompt(PROMPT_V2_NAME, object=PROMPT_V2, ...)`, bọc try/except |
| 2.5 | `pull_prompts_from_hub()` | `client.pull_prompt(PROMPT_V1_NAME)` + `client.pull_prompt(PROMPT_V2_NAME)`, fallback local |
| 2.6 | `get_prompt_version()` | `hash_int = int(hashlib.md5(request_id.encode()).hexdigest(), 16)` → chẵn=V1, lẻ=V2 |
| 2.7 | `@traceable` decorator | `@traceable(name="ab-rag-query", tags=["ab-test", "step2"])` |
| 2.8 | `ask_ab()` | Retrieve docs → ghép context → `(prompt \| llm \| StrOutputParser()).invoke({"context": ..., "question": ...})` → trả về `{"question": ..., "answer": ..., "version": ...}` |
| 2.9 | `main()` | Tạo client → push prompts → pull prompts → tạo vectorstore/retriever/llm → lặp 50 câu hỏi, gọi `get_prompt_version()` → `ask_ab()` → in log có nhãn v1/v2 |

### Chạy & kiểm tra

```bash
cd src && python 02_prompt_hub_ab_routing.py | tee ../evidence/02_ab_routing_log.txt
```

### Bằng chứng cần thu thập

- [ ] Chụp ảnh Prompt Hub hiển thị 2 phiên bản prompt → lưu `evidence/02_prompt_hub.png`
- [ ] File log `evidence/02_ab_routing_log.txt` chứa output console với nhãn v1/v2

---

## BƯỚC 3: RAGAS Evaluation (25 điểm)

**File cần sửa:** `src/03_ragas_evaluation.py`  
**Mục tiêu:** Chạy 50 QA pairs qua cả 2 phiên bản prompt, đánh giá bằng 4 chỉ số RAGAS, đạt faithfulness ≥ 0.8

**Lưu ý:** API Gemini đã trả phí nên tốc độ nhanh, dự kiến ~10–15 phút cho mỗi phiên bản.

### Checklist triển khai

| # | Việc cần làm | Chi tiết |
|---|-------------|----------|
| 3.1 | Copy `SYSTEM_V1`, `SYSTEM_V2` | Copy y hệt từ file `02_prompt_hub_ab_routing.py` — không được khác |
| 3.2 | `run_rag()` | `docs = retriever.invoke(question)` → `contexts = [doc.page_content for doc in docs]` (list[str]!) → `ctx_str = "\n\n".join(contexts)` → `(prompt \| llm \| StrOutputParser()).invoke({"context": ctx_str, "question": question})` → trả về `{"answer": ..., "contexts": contexts}` |
| 3.3 | `collect_rag_outputs()` | Lặp qua 50 QA_PAIRS → gọi `run_rag()` → append vào list {question, reference, answer, contexts} |
| 3.4 | `build_ragas_dataset()` | Tạo `SingleTurnSample(user_input=..., response=..., retrieved_contexts=..., reference=...)` → wrap trong `EvaluationDataset(samples=...)` |
| 3.5 | `run_ragas_eval()` | `evaluate(dataset, metrics=[faithfulness, answer_relevancy, context_recall, context_precision], llm=llm_eval, embeddings=emb_eval)` → `np.mean(result[metric])` cho từng metric |
| 3.6 | `main()` | Tạo vectorstore → `collect_rag_outputs("v1")` + `collect_rag_outputs("v2")` → `run_ragas_eval(v1)` + `run_ragas_eval(v2)` → in bảng so sánh → lưu `data/ragas_report.json` |

### Chạy

```bash
cd src && python 03_ragas_evaluation.py
```

### Bằng chứng cần thu thập

- [ ] Chụp ảnh terminal hiển thị bảng so sánh V1 vs V2 → lưu `evidence/03_ragas_scores.png`
- [ ] Copy `data/ragas_report.json` → `evidence/03_ragas_report.json`

---

## BƯỚC 4: Guardrails AI Validators (25 điểm)

**File cần sửa:** `src/04_guardrails_validator.py`  
**Mục tiêu:** Triển khai PIIDetector (phát hiện email, phone, SSN, credit card) và JSONFormatter (sửa JSON lỗi)

### Checklist triển khai — PII Detector

| # | Việc cần làm | Chi tiết |
|---|-------------|----------|
| 4.1 | `PIIDetector` class | Đã có sẵn `@register_validator(name="custom/pii-detector", data_type="string")` và `PII_PATTERNS` |
| 4.2 | `validate()` | Lặp `PII_PATTERNS.items()` → `re.findall(pattern, value)` → `redacted_text.replace(match, f"[{pii_type}_REDACTED]")` → `found_pii.append((pii_type, match))` |
| 4.3 | Return | Nếu có PII → `PassResult(value_override=redacted_text)`, không có → `PassResult(value_override=value)` |
| 4.4 | `demo_pii_guard()` | `guard = Guard().use(PIIDetector(on_fail=OnFailAction.FIX))` → test 6 cases: Email, Phone, SSN, Credit Card, Multi-PII, Clean |

### Checklist triển khai — JSON Formatter

| # | Việc cần làm | Chi tiết |
|---|-------------|----------|
| 4.5 | `JSONFormatter._repair()` | strip whitespace → xóa markdown fences `r'^```(?:json)?\s*'` → `text.replace("'", '"')` → `re.sub(r',\s*([}\]])', r'\1', text)` |
| 4.6 | `validate()` | Thử `json.loads(value)` → nếu fail: `_repair()` rồi thử lại → `PassResult(value_override=json.dumps(parsed, indent=2))` → nếu vẫn fail: `FailResult(...)` |
| 4.7 | `demo_json_guard()` | `guard = Guard().use(JSONFormatter(on_fail=OnFailAction.FIX))` → test 5 cases: Valid JSON, Markdown fences, Single quotes, Trailing comma, Truly invalid |

### Chạy

```bash
cd src && python 04_guardrails_validator.py 2>&1 | tee ../evidence/04_pii_demo_log.txt
```

### Bằng chứng cần thu thập

- [ ] Output PII demo → lưu `evidence/04_pii_demo_log.txt`
- [ ] Output JSON demo → lưu `evidence/04_json_demo_log.txt`

---

## BƯỚC 5: Chạy toàn bộ & kiểm tra tổng thể

```bash
cd src && python run_all.py
```

Hoặc từng bước:
```bash
cd src && python run_all.py --step 1
cd src && python run_all.py --step 2
cd src && python run_all.py --step 3
cd src && python run_all.py --step 4
```

---

## Danh sách bằng chứng cần nộp (7 files)

| # | File | Nguồn |
|---|------|-------|
| 1 | `evidence/01_langsmith_traces.png` | Chụp màn hình LangSmith dashboard ≥50 traces |
| 2 | `evidence/02_prompt_hub.png` | Chụp màn hình Prompt Hub với 2 phiên bản |
| 3 | `evidence/02_ab_routing_log.txt` | Output console Step 2 với nhãn v1/v2 |
| 4 | `evidence/03_ragas_scores.png` | Output terminal bảng so sánh V1 vs V2 |
| 5 | `evidence/03_ragas_report.json` | Copy từ `data/ragas_report.json` |
| 6 | `evidence/04_pii_demo_log.txt` | Output console PII test cases |
| 7 | `evidence/04_json_demo_log.txt` | Output console JSON test cases |

---

## Checklist trước khi nộp bài

- [ ] Tất cả 4 file `.py` trong `src/` chạy không lỗi
- [ ] `data/ragas_report.json` tồn tại, chứa điểm V1 và V2
- [ ] Đủ 7 file trong `evidence/`
- [ ] Tạo GitHub repository public, push toàn bộ code (KHÔNG push `.env`!)
- [ ] Nộp URL GitHub repository + URL LangSmith project
- [ ] File `.env` đã được thêm vào `.gitignore`
- [ ] Không có API key nào trong mã nguồn

---

## Các lưu ý quan trọng

### Về RAGAS (Bước 3)
- `contexts` phải là `list[str]`, không phải string đã ghép
- `evaluate()` nhận `llm=` và `embeddings=`, không truyền vào metric constructor
- `result[metric_name]` trả về list → dùng `np.mean()` để tính trung bình

### Về Guardrails AI (Bước 4)
- `on_fail` truyền vào **constructor của Validator**: `PIIDetector(on_fail=OnFailAction.FIX)` ✓
- KHÔNG truyền vào `Guard.use()`: `Guard().use(PIIDetector, on_fail=...)` ✗
- `Guard.use()` nhận validator **instance**, không phải class

### Về Gemini Provider
- API đã trả phí → tốc độ cao, không bị giới hạn 15 req/phút như free tier
- Cần `pip install langchain-google-genai`
- Embedding dùng `models/embedding-001` — không cần OpenAI key
