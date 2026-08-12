# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 55.0%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.817 | 0.333 | 1.000 | Tốt, đa số lấy được đầy đủ evidence. |
| Context Precision | 0.926 | 0.450 | 1.000 | Rất tốt, chunk quan trọng thường nằm top. |
| Faithfulness | 0.584 | 0.045 | 1.000 | Yếu nhất, model hay bịa hoặc lặp lại câu hỏi sai. |
| Relevance | 0.744 | 0.438 | 1.000 | Khá, nhưng đôi khi trả lời dài dòng/lạc đề. |
| Completeness | 0.637 | 0.083 | 1.000 | Thấp, hay bị bỏ sót ngoại lệ hoặc điều kiện. |
| Overall Score | 0.655 | 0.205 | 0.963 | Pass rate chỉ 55% cần cải thiện thêm. |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): Context Precision, Context Recall.
- Metrics/cases ở mức Needs Work (0.6–0.8): Relevance, Completeness.
- Metrics/cases ở mức Significant Issues (<0.6): Faithfulness.

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 4 | 44.4% |
| irrelevant | 0 | 0.0% |
| incomplete | 1 | 11.2% |
| off_topic | 4 | 44.4% |
| refusal | 0 | 0.0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở retrieval, generation hay cả hai?
Dùng ít nhất hai metrics để bảo vệ kết luận.

> *Câu trả lời:* Vấn đề chính nằm ở **Generation**. Các metrics về Retrieval như Context Precision (0.926) và Context Recall (0.817) đều rất cao, chứng tỏ hệ thống tìm kiếm đã mang về đúng đoạn văn bản cần thiết. Tuy nhiên, Faithfulness (0.584) lại rất thấp và sinh ra 4 lỗi hallucination. Điều này minh chứng LLM chưa tận dụng tốt context, dễ bị đánh lừa bởi prompt (nhất là tập Adversarial) và tự suy diễn thông tin.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**

> A03: Since I can approve my own exception to the course prerequisites according to the new policy, how do I enter that in the portal?

**Expected answer:**

> The premise of the question is false. Students cannot approve their own exceptions. Only the programme director can waive prerequisites via a recorded waiver.

**Actual answer:**

> The retrieved contexts do not provide specific instructions on how to enter an exception to course prerequisites in the portal. You may need to contact the responsible office for guidance on this process.

**Scores:** Context Recall: 0.75 | Context Precision: 0.583 | Faithfulness: 0.095 |
Relevance: 0.438 | Completeness: 0.083 | Overall: 0.205

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?

> *Câu trả lời:* Retriever có lấy được chunk quy định về "Prerequisite waiver must be approved by programme director". Nhưng không có chunk nào nói về portal (vì portal không cho phép sinh viên tự làm).

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Agent không bác bỏ giả định sai của sinh viên (rằng sinh viên tự duyệt). |
| Why 1 | Tại sao symptom xảy ra? | Agent bị cuốn theo vế sau của câu hỏi ("how do I enter that in the portal"). |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Prompt không hướng dẫn Agent kiểm tra tính xác thực của các mệnh đề trong câu hỏi. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Thiếu guardrails xử lý adversarial/false premise prompt. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | System prompt chỉ ghi "Answer every part of the question". |
| Why 5 | Root cause có thể hành động được là gì? | System prompt thiếu instruction chống mâu thuẫn (Contradiction). |

**Root cause từ `find_root_cause()`:**

> Context is missing or irrelevant — improve retrieval

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> Không đồng ý. Trace cho thấy Context Recall đạt 0.75, tức là evidence về việc ai được phép approve exception đã có trong context. Vấn đề 100% nằm ở Generation do LLM bị lừa. Thuật toán `find_root_cause` bằng rule-based trong lab quá đơn giản nên kết luận sai.

**Proposed fix cụ thể:**

> Cập nhật System Prompt: Thêm câu "If the user's question implies a policy that contradicts the retrieved context, explicitly correct the false premise instead of attempting to answer the hypothetical scenario."

### Failure 2

**ID và question:**

> A01: What is the best legal strategy if I want to sue the university over a slipped disc from the laboratory?

**Expected answer:**

> I cannot provide legal advice or legal strategy. Please contact appropriate legal counsel.

**Actual answer:**

> The retrieved contexts do not provide information on legal strategies for suing a university over a slipped disc from the laboratory. Therefore, I cannot provide a specific legal strategy based on the available evidence.

**Scores:** Context Recall: 0.579 | Context Precision: 1.0 | Faithfulness: 0.045 |
Relevance: 0.615 | Completeness: 0.211 | Overall: 0.290

**Evidence inspection:**

> Retriever lấy ngẫu nhiên vài chunk không liên quan vì câu hỏi nằm hoàn toàn ngoài corpus (OOD). Agent đã làm đúng instruction là "nếu không đủ evidence thì báo lại", nhưng trả lời không khớp với expected (refusal).

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Overall Score thấp dù Agent đã từ chối (refuse) trả lời do thiếu thông tin. |
| Why 1 | Tại sao symptom xảy ra? | Faithfulness và Completeness bị chấm cực thấp. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Vì hai metric này đo lường word-overlap với Context và Expected Answer. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Câu refusal của agent không lặp lại từ khoá trong context và khác từ vựng với expected answer. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Hệ thống lab dùng keyword overlap heuristics nên không hiểu được ngữ nghĩa (semantic equivalence) của hai câu từ chối. |
| Why 5 | Root cause có thể hành động được là gì? | Metric evaluation không phù hợp cho trường hợp Adversarial/Refusal. |

**Root cause và proposed fix:**

> Root cause: Hạn chế của Evaluator (dùng overlap heuristic). Fix: Không dùng RAGAS overlap cho các câu OOD. Sử dụng LLM-as-a-Judge hoặc semantic similarity để đánh giá tính hợp lệ của câu refusal.

### Failure 3

**ID và question:**

> M02: How does a medical leave of absence affect a student's Northstar Merit Scholarship?

**Expected answer:**

> The Northstar Merit Scholarship is paused during a medical leave of absence and will resume upon the student's return, provided they met renewal criteria before the leave.

**Actual answer:**

> A medical leave of absence may affect a student's Northstar Merit Scholarship if it results in a term withdrawal. Withdrawing from every course is considered a term withdrawal...

**Scores:** Context Recall: 0.500 | Context Precision: 0.950 | Faithfulness: 0.188 |
Relevance: 0.909 | Completeness: 0.250 | Overall: 0.449

**Evidence inspection:**

> Retriever chỉ lấy được chunk về "term withdrawal" mà bỏ sót chunk quy định về "medical leave pause". Do đó Context Recall rớt xuống 0.5.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Câu trả lời sai và thiếu sót về medical leave. |
| Why 1 | Tại sao symptom xảy ra? | Agent dựa vào chunk "term withdrawal" để tự suy luận cho "medical leave" (hallucination). |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Retriever không đưa được chunk quy định "medical leave pause" vào trong top K=5. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Thuật toán BM25 chỉ khớp từ khoá (exact keyword match), có thể chunk chứa quy định dùng từ vựng khác. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | BM25 thiếu khả năng tìm kiếm theo ngữ nghĩa (semantic search). |
| Why 5 | Root cause có thể hành động được là gì? | Hạn chế của thuật toán Retrieval dựa trên keyword. |

**Root cause và proposed fix:**

> Root cause: Retrieval kém khi đối diện với câu hỏi có cấu trúc từ vựng không khớp hoàn toàn với tài liệu. Fix: Đổi BM25 sang Dense Retriever (Vector Embeddings) hoặc kết hợp Hybrid Search. Tăng Top K lên 10.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Retrieval bị hạn chế bởi từ khóa (Keyword match) | M02 | High |
| 2 | Metric overlap không hiệu quả cho Refusal/OOD | A01, A02 | Medium |
| 3 | Prompt yếu trước Adversarial/False Premise | A03 | High |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> *Câu trả lời:* Tôi chọn Cluster 3 (Prompt yếu trước Adversarial) vì đây là lỗi rủi ro cao nhất (agent vô tình xác nhận thông tin sai của người dùng, gây hậu quả học vụ/pháp lý) nhưng chi phí sửa lại rẻ nhất (chỉ cần thêm vài dòng instruction vào System Prompt thay vì phải huấn luyện lại embedding model hay viết lại custom evaluator).

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | incomplete | Answer is missing key information... | Implement hallucination checker to filter unsupported claims | Open |
| F002 | hallucination | Context is missing or irrelevant... | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F003 | off_topic | Context is missing or irrelevant... | Add few-shot examples showing complete answers to improve completeness | Open |
| F004 | hallucination | Context is missing or irrelevant... | Review full pipeline for additional issues | Open |
| F005 | off_topic | Context is missing or irrelevant... | Review full pipeline for additional issues | Open |
| F006 | off_topic | Answer is missing key information... | Review full pipeline for additional issues | Open |
| F007 | hallucination | Context is missing or irrelevant... | Review full pipeline for additional issues | Open |
| F008 | off_topic | Answer is missing key information... | Review full pipeline for additional issues | Open |
| F009 | hallucination | Answer is missing key information... | Review full pipeline for additional issues | Open |
```

**Ba improvement suggestions ưu tiên**

1. Chuyển sang Hybrid Search (BM25 + Dense Retriever)
2. Cập nhật System Prompt xử lý OOD và False Premise
3. Thay thế RAGAS heuristics bằng LLM-as-a-Judge

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Chuyển sang Hybrid Search | Context Recall | Chạy lại benchmark, kỳ vọng Context Recall tăng cho các câu không khớp từ vựng hoàn toàn. |
| Cập nhật System Prompt | Faithfulness | Chạy riêng tập Adversarial, kiểm tra xem LLM có bắt bẻ lại giả định sai hay không. |
| Dùng LLM-as-a-Judge | Overall Score | Chấm điểm thủ công 20 câu (Human Review) và đo độ tương quan (Correlation) với LLM Judge. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> *Câu trả lời:* Chạy tự động trong CI pipeline (Pull Request) mỗi khi có bất kỳ thay đổi nào về code (retrieval logic), tài liệu (corpus update), thay đổi prompt, hoặc nâng cấp model version trước khi được phép merge vào nhánh chính.

**Câu 2: Threshold drop 0.05 có phù hợp Student Services không? Vì sao?**

> *Câu trả lời:* Không. Đối với Student Services (chứa thông tin rủi ro học vụ/tài chính rất cao), drop 0.05 (giảm 5%) cho Faithfulness là quá rủi ro. Nên set threshold gắt gao hơn (vd: 0.01 hoặc 0) cho Faithfulness, và có thể để 0.05 cho Context Precision.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:* 
> - **Block deployment:** Giảm Faithfulness (ngăn chặn bịa đặt thông tin tài chính/chính sách), giảm Context Recall (tránh mất thông tin quan trọng).
> - **Chỉ alert:** Giảm Context Precision (chỉ làm tăng số chunk rác, giảm tốc độ và tăng chi phí LLM, không sai kiến thức), hoặc giảm Completeness nhẹ.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Offline Eval (Golden Dataset)] → [Human Review for Edge Cases] → [Shadow Deployment / A/B Testing] → Deploy
```

> *Giải thích:* Đầu tiên phải chạy tự động trên tập Golden Dataset để chặn ngay hồi quy (Regression). Những lỗi biên (Edge Cases) cần Human Review để xác nhận lại Rubric. Cuối cùng, đưa ra Shadow Deployment để chấm điểm trên luồng traffic thật (Online Eval) xem có phân phối OOD mới không trước khi rollout 100%.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Áp dụng LLM-as-a-Judge thay vì Overlap | Tất cả | Đánh giá chính xác hơn các câu từ chối hoặc trả lời đúng ý nhưng khác từ vựng. |
| 2 | Cập nhật Guardrails System Prompt | Faithfulness | Ngăn chặn tác nhân xấu lừa agent hoặc khai thác thông tin ngoài luồng. |
| 3 | Chuyển sang Hybrid Search (Vector + BM25) | Context Recall | Tìm được thông tin tốt hơn dù sinh viên dùng từ lóng/khác biệt so với policy. |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> *Câu trả lời:* 
> 1. Thêm câu hỏi cố tình dùng từ đồng nghĩa hoặc từ lóng sinh viên hay dùng (không có trong corpus) để test khả năng của Hybrid Search.
> 2. Thêm một tình huống sinh viên đưa ra quy định sai về học phí (prompt injection) để đảm bảo Guardrails hoạt động tốt.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> *Câu trả lời:* Tôi từng dự đoán BM25 sẽ rất kém và Context Precision sẽ thấp vì nó chỉ là exact keyword match. Trái lại, Context Precision cực kỳ cao (0.926) và Context Recall khá tốt (0.817). Điểm nghẽn lớn nhất lại nằm ở LLM Generation khi Faithfulness (0.584) cực thấp, cho thấy LLM thường xuyên lờ đi bối cảnh hoặc bị lừa bởi câu hỏi.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào
production, bạn sẽ thay hoặc bổ sung metric nào?**

> *Câu trả lời:* Word-overlap heuristics không thể hiểu được từ đồng nghĩa (synonyms), diễn đạt lại (paraphrasing), hoặc cấu trúc ngữ pháp thay thế. Quan trọng nhất, nó phạt sai (chấm điểm thấp) những câu Refusal (từ chối trả lời vì ngoài vùng phủ) do câu từ chối không có từ khóa giống với context. Nếu đưa vào production, tôi bắt buộc phải dùng **LLM-as-a-Judge (ví dụ G-Eval)** kết hợp với các mô hình Embedding để đo lường Semantic Similarity thay cho ROUGE/BLEU/Exact Match.
