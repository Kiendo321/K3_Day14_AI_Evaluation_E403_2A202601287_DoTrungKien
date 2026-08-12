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

> *Câu trả lời:* Vấn đề cốt lõi chắc chắn nằm ở phần Generation. Theo kết quả, hai chỉ số của Retrieval là Context Precision (0.926) và Context Recall (0.817) đều rất cao, nghĩa là nó tìm đúng tài liệu rồi. Nhưng ngặt nỗi, điểm Faithfulness lại thấp thê thảm (0.584) và sinh ra tới 4 lỗi hallucination. Điều này chứng tỏ con LLM không thèm đọc kỹ context mà hay tự bịa, hoặc dễ bị đánh lừa bởi câu hỏi gài bẫy của người dùng.

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

> *Câu trả lời:* Retriever lấy được cái quy định về việc "chỉ có giám đốc chương trình mới duyệt được prerequisite waiver", nhưng dĩ nhiên là không có tài liệu nào nói về cái portal (vì sinh viên đâu có tự duyệt được).

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Agent hùa theo giả định sai của sinh viên thay vì bác bỏ nó. |
| Why 1 | Tại sao symptom xảy ra? | Agent bị cuốn theo vế sau "làm sao để nhập vào portal". |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Prompt hiện tại không hề bảo agent phải đi check xem câu hỏi có xạo hay không. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Do thiếu guardrails chống lại các câu hỏi gài bẫy (false premise). |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | System prompt chỉ ghi chung chung là "phải trả lời đầy đủ câu hỏi". |
| Why 5 | Root cause có thể hành động được là gì? | Prompt quá yếu, thiếu hụt instruction để xử lý mấy pha người dùng nói dối. |

**Root cause từ `find_root_cause()`:**

> Context is missing or irrelevant — improve retrieval

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> Mình không đồng ý chút nào. Trace cho thấy Context Recall là 0.75, tức là hệ thống đã lấy đủ evidence về chuyện ai được duyệt rồi. Lỗi 100% nằm ở Generation do LLM bị lừa. Thuật toán `find_root_cause` kiểu rule-based này chẩn đoán sai bét.

**Proposed fix cụ thể:**

> Sửa System Prompt ngay: Thêm câu "Nếu sinh viên đưa ra giả định sai trái so với tài liệu, hãy đính chính ngay lập tức thay vì cố gắng trả lời giả định đó."

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

> Retriever chỉ vớt đại vài chunk linh tinh vì câu này làm gì có trong quy chế (OOD). Agent đã làm đúng dặn dò là "nếu không có thì báo lại", nhưng khổ nỗi cách trả lời của nó lại không khớp với cái expected answer (từ chối thẳng).

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Tổng điểm rớt thê thảm dù Agent đã từ chối trả lời khá ổn. |
| Why 1 | Tại sao symptom xảy ra? | Vì hai điểm Faithfulness và Completeness bị chấm cực kỳ thấp. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Tại vì mấy cái metric này đo bằng cách đếm xem có trùng từ (word-overlap) hay không. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Câu từ chối của agent làm sao mà trùng từ vựng với context hay expected answer được. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Hệ thống lab dùng keyword overlap quá máy móc, không hiểu được nghĩa tương đồng (semantic equivalence) của hai cách từ chối. |
| Why 5 | Root cause có thể hành động được là gì? | Metric chấm thi hoàn toàn không phù hợp cho mấy ca Adversarial/Refusal này. |

**Root cause và proposed fix:**

> Root cause: Lỗi do chính cái Evaluator (dùng overlap heuristic). 
> Fix: Dẹp vụ dùng RAGAS overlap cho các câu OOD đi. Nên dùng LLM-as-a-Judge hoặc semantic similarity để xem nó từ chối có hợp lý không.

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

> Retriever chỉ vớt được cái chunk nói về "term withdrawal" mà hụt mất cái chunk cụ thể về "medical leave pause". Kết quả là Context Recall rớt xuống còn 0.5.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Câu trả lời thiếu hụt trầm trọng và chém gió sai luật về nghỉ ốm. |
| Why 1 | Tại sao symptom xảy ra? | Agent bám vào cái chunk "term withdrawal" rồi tự biên tự diễn luôn cho "medical leave" (hallucination). |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Retriever tìm quá dở, không kéo được cái quy định "medical leave pause" lên Top 5. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Thuật toán BM25 chỉ đi so sánh từ khoá y xì nhau (exact keyword match). Chắc do tài liệu dùng từ lóng hay gì đó nên nó không khớp. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Đơn giản là BM25 không có khả năng hiểu được ngữ nghĩa (semantic search). |
| Why 5 | Root cause có thể hành động được là gì? | Retriever dựa vào từ khóa bị kém hiệu quả đối với các truy vấn phức tạp. |

**Root cause và proposed fix:**

> Root cause: Thuật toán tìm kiếm xài từ khóa quá nghèo nàn khi sinh viên hỏi câu lắt léo.
> Fix: Nâng cấp thẳng lên Hybrid Search (BM25 kết hợp với Vector Embeddings) và ráng tăng số lượng Top K lên cỡ 10 cho chắc ăn.

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

> *Câu trả lời:* Nên cho chạy tự động trong CI/CD (khi có Pull Request) mỗi khi anh em update code, đổi prompt, thêm bớt tài liệu hay update version của LLM trước khi cho merge vào nhánh chính.

**Câu 2: Threshold drop 0.05 có phù hợp Student Services không? Vì sao?**

> *Câu trả lời:* Không ổn tí nào. Làm mảng dịch vụ sinh viên dính dáng toàn tiền bạc học phí, rớt 0.05 điểm Faithfulness (5%) là quá rủi ro rồi. Nên set thật khắt khe (tầm 0.01) hoặc tạch luôn nếu có bất kỳ lỗi bịa chuyện (hallucination) nào.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:* 
> - **Block luôn:** Rớt điểm Faithfulness (chống chế cháo luật) và Context Recall (tránh mất thông tin xịn).
> - **Chỉ cần cảnh báo (alert):** Rớt Context Precision (vì cái này chỉ làm chậm và tốn tiền LLM chứ không sai luật), hoặc giảm Completeness một xíu.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Offline Eval (Chạy Golden Dataset)] → [Human Review cho mấy ca khó] → [Shadow Deployment / A/B Testing] → Deploy
```

> *Giải thích:* Khởi đầu thì cứ ốp bộ tự động lên Golden Dataset để cản mấy lỗi cũ cái đã. Sau đó team phải vào đọc lại (Human Review) mấy câu hỏi hóc búa. Xong xuôi thì chạy ngầm (Shadow Deployment) để hứng traffic thật xem có lòi ra lỗi lạ nào không rồi mới dám tung ra thật.

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

> *Câu trả lời:* Hồi đầu mình cứ đinh ninh là dùng thuật toán cổ lỗ sĩ BM25 thì điểm Context Precision chắc lè tè vì nó chỉ kiếm chính xác từ khóa. Ai dè Precision lên tới tận 0.926, còn Context Recall cũng khá khẩm (0.817). Hóa ra điểm chết lại nằm ở ông thần LLM (Generation) khi điểm Faithfulness lẹt đẹt (0.584). Rõ ràng là nó đọc không kỹ bài hoặc rất dễ bị người ta dắt mũi gài bẫy.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào
production, bạn sẽ thay hoặc bổ sung metric nào?**

> *Câu trả lời:* Trò đếm từ trùng lặp (word-overlap) này quá máy móc, nó chẳng hiểu từ đồng nghĩa hay kiểu viết lách khác xíu là tạch. Thảm nhất là nó toàn chấm trượt oan mấy câu "Từ chối trả lời" do câu từ chối chả có chữ nào giống tài liệu. Lên production thì mình bái bai trò này, phải đưa ngay **LLM-as-a-Judge (kiểu như G-Eval)** kết hợp xài Embedding để đo nghĩa (Semantic Similarity) thì mới chuẩn.
