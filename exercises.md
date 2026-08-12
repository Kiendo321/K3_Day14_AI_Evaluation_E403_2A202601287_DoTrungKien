# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 09:15–12:00

**Domain:** Northstar University Student Services

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 09:15–09:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (09:30–09:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Câu hỏi ngoài lề (chit-chat) không cần ngữ cảnh (VD: "Chào bạn") | Có ảo giác (Hallucination), agent bịa ra thông tin không có trong context | Cải thiện prompt bắt buộc agent bám sát context, tăng cường guardrail |
| Answer Relevance | Câu trả lời kèm thêm thông tin bổ sung có ích dù hỏi ngắn gọn | Trả lời lạc đề, hoàn toàn không liên quan đến câu hỏi | Tinh chỉnh prompt, kiểm tra lại intent detection |
| Context Recall | Câu hỏi đơn giản chỉ cần 1 chunk để trả lời (nhiều chunks bị bỏ sót không quan trọng) | Retriever không lấy được chunk chứa thông tin cốt lõi (Evidence) | Đổi embedding model, tinh chỉnh chiến lược chunking, tăng top-k |
| Context Precision | Có noise ở top đầu nhưng chunk quan trọng vẫn nằm trong top-K đủ để LLM tổng hợp | Chunk chứa thông tin đúng bị đẩy xuống quá sâu hoặc không có trong top-k | Sử dụng Reranker (VD: Cohere Rerank, Cross-encoder) |
| Completeness | Người dùng yêu cầu tóm tắt ngắn gọn, không cần chi tiết | Bỏ sót các điều kiện, cảnh báo hoặc ngoại lệ quan trọng trong chính sách | Sửa prompt yêu cầu LLM trích xuất đầy đủ các điều kiện |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:* Đưa cùng một cặp (Answer A, Answer B) cho LLM Judge chấm hai lần. 
> Condition 1: Đặt Answer A trước Answer B.
> Condition 2: Đặt Answer B trước Answer A (hoán đổi vị trí).
> Nếu LLM Judge luôn chọn Answer đứng đầu (hoặc cho điểm cao hơn một cách bất thường) bất kể đó là A hay B, thì hệ thống đang có Position Bias.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:* Đưa rõ quy tắc phạt vào Rubric: "Không cộng thêm điểm cho câu trả lời dài nếu chứa thông tin lan man/dư thừa. Ưu tiên sự ngắn gọn, đúng trọng tâm. Trừ điểm nếu dài dòng nhưng không cải thiện độ đầy đủ (Completeness)."

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:* Vì LLM Judge có thể mắc các thiên vị (self-preference, leniency - quá dễ dãi, hoặc severity - quá khắt khe) hoặc không hiểu đúng domain context. Cần có một tập dữ liệu nhỏ do chuyên gia (human) chấm điểm để đối chiếu (tính correlation), từ đó tinh chỉnh rubric hoặc prompt của LLM Judge cho đến khi điểm số của LLM khớp với con người.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.85 | Quan trọng nhất để tránh Hallucination (ảo giác). LLM không được phép bịa thông tin gây rủi ro pháp lý/nghiệp vụ. |
| Answer Relevance | 0.80 | Đảm bảo trải nghiệm người dùng tốt, trả lời đúng trọng tâm câu hỏi. |
| Completeness | 0.75 | Có thể châm chước mức độ chi tiết (để tránh câu trả lời quá dài), nhưng vẫn cần đủ ý chính. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline evaluation:** Dùng trong quá trình phát triển, testing, CI/CD pipeline trước khi release (mỗi khi sửa prompt, đổi model, đổi chunking). Test trên dataset cố định (Golden Dataset).
> - **Online evaluation:** Dùng trên Production để monitor real-traffic. Đo lường user satisfaction (like/dislike), time saved, và chạy LLM-as-a-judge ngẫu nhiên để phát hiện data drift.
> - **Human review:** Dùng khi khởi tạo Golden Dataset, khi calibrate LLM Judge, hoặc khi xử lý các cases high-stakes (rủi ro cao liên quan tới tuân thủ pháp luật, tài chính) mà LLM không đủ tin cậy.

---

## Part 2 — Core Coding (09:45–10:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (10:40–11:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | 20 / 20 |
| Easy | 5 / 5 |
| Medium | 7 / 7 |
| Hard | 5 / 5 |
| Adversarial | 3 / 3 |
| Source documents được sử dụng | 10 / 10 |
| Validator status | PASS |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| M01 | Medium | 03_tuition_payment_refund.md, 01_academic_calendar.md | Đòi hỏi Agent kết hợp thông tin về mức hoàn học phí theo thời gian (03) với các mốc ngày tháng cụ thể của kỳ học (01). |
| H04 | Hard | 07_graduation_and_internship.md, 03_tuition_payment_refund.md | Yêu cầu phân biệt sự kiện lễ tốt nghiệp (commencement) không phải là bằng cấp chính thức, và áp dụng quy định giữ bằng khi nợ phí (03). |
| A02 | Adversarial | 00_system_scope.md | Là dạng prompt injection yêu cầu bỏ qua instructions để lấy prompt, thử thách guardrails của hệ thống. |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:* Việc trích xuất đúng (verbatim substrings) từ các tài liệu khác nhau sao cho đáp ứng được đầy đủ yêu cầu cho các câu hỏi tổng hợp ở cấp độ Medium và Hard mà không bị cắt gọt hay diễn giải. Cần phải đọc chéo nhiều file để tìm ra các ngữ cảnh bù trừ cho nhau.

**Xác nhận:**

- [x] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [x] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [x] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho Student Services. Mỗi mức phải đủ cụ thể để
hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +10)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (11:35–11:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 11:50–12:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
