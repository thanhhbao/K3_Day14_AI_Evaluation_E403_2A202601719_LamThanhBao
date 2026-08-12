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
| Faithfulness | Câu hỏi chung chung, context rộng, khó map từng từ chính xác (e.g., tóm tắt chính sách). Score 0.6–0.7 có thể chấp nhận khi answer đúng nhưng dùng paraphrase. | Dưới 0.5: answer chứa claim không có trong context → hallucination. Đặc biệt nguy hiểm với domain có số liệu (học phí, deadline). | Kiểm tra retrieval context có đủ không; thêm grounding guardrail; review các claim bịa. |
| Answer Relevance | Câu hỏi mơ hồ hoặc có nhiều cách hiển. Answer có thể giải thích thêm background, làm giảm overlap từ với question. Score 0.6 chấp nhận được. | Dưới 0.4: answer hoàn toàn lạc đề, không giải quyết intent của user. | Rà lại prompt instruction; thêm few-shot examples về cách trả lời đúng intent; kiểm tra routing. |
| Context Recall | Câu hỏi rộng cần nhiều document, retriever chỉ lấy được một phần. Score 0.6 chấp nhận được nếu phần lấy được đủ để generate câu trả lời chính. | Dưới 0.4: retriever bỏ sót evidence then chốt khiến generation thiếu thông tin quan trọng. | Tăng top-k, mở rộng chunking strategy, thêm query expansion hoặc hybrid search. |
| Context Precision | Corpus nhỏ hoặc câu hỏi rất cụ thể, noise chunk không làm lệch generation. Score thấp ít tác hại nếu recall cao. | Dưới 0.3: noise chunk đứng đầu ranking liên tục, generation bị dẫn dắt sai hướng. | Thêm reranker; điều chỉnh similarity threshold; lọc chunk theo metadata (source_doc). |
| Completeness | Câu hỏi phức tạp, chấp nhận partial answer; user có thể follow-up. Score 0.6 hợp lý. | Dưới 0.4: bỏ sót điều kiện quan trọng (deadline, mức phí, ngoại lệ) làm user hiểu sai chính sách. | Tăng context window; review expected_answer coverage; thêm instruction "list all conditions". |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:* Chọn 10 câu hỏi, mỗi câu có 2 candidate answers (A và B) với chất lượng khác nhau. Condition 1: trình bày [A trước, B sau] cho judge. Condition 2: đảo thứ tự [B trước, A sau] cho cùng judge. Nếu judge chọn answer đứng trước nhiều hơn đáng kể so với expected (dựa trên human label), thì tồn tại position bias. Để có ý nghĩa thống kê, cần ít nhất 20 cặp và so sánh tỷ lệ "chọn answer đứng trước" giữa hai condition.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:* Rubric cần có tiêu chí explicit về conciseness và penalize thông tin thừa. Ví dụ: thêm dimension "Precision" với mô tả "Điểm bị trừ nếu answer chứa thông tin không liên quan hoặc lặp lại không cần thiết." Tránh dùng rubric mơ hồ như "comprehensive = tốt". Đặt word-limit gợi ý trong prompt judge để chuẩn hoá kỳ vọng độ dài. Nếu có thể, cung cấp example answer ngắn gọn nhưng đầy đủ như golden reference.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:* LLM judge có thể có systematic bias riêng (verbosity, self-preference, cultural) không phản ánh chuẩn con người. Calibration so sánh score của LLM judge với human annotation trên cùng tập dữ liệu để đo inter-rater agreement (Cohen's kappa hoặc Pearson correlation). Nếu agreement thấp, ta biết judge đang đo sai thứ cần đo. Calibration cũng giúp điều chỉnh threshold — ví dụ nếu LLM judge luôn cho điểm cao hơn human 0.1, ta hạ threshold tương ứng. Không calibrate → metric trông đẹp nhưng không reflect chất lượng thực.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.70 | Dưới ngưỡng này risk hallucination cao; trong domain student services, thông tin sai về học phí/deadline có thể gây hậu quả nghiêm trọng cho student. |
| Answer Relevance | 0.60 | Dưới ngưỡng này answer thường không giải quyết được intent; student sẽ không tìm được thông tin cần, mất trust vào hệ thống. |
| Completeness | 0.65 | Dưới ngưỡng này answer bỏ sót điều kiện/ngoại lệ quan trọng; student có thể hiểu thiếu chính sách và bị ảnh hưởng tài chính hoặc học thuật. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:* **Offline evaluation** dùng trước mỗi code release hoặc prompt change — chạy toàn bộ golden dataset để đảm bảo không có regression trước khi deploy. **Online evaluation** chạy continuous trên real traffic để phát hiện distribution shift hoặc edge case mới mà golden dataset chưa cover; dùng sampling để không làm tăng latency đáng kể. **Human review** dùng khi cần calibrate LLM judge (quarterly), khi có high-stakes edge case (học viên khiếu nại, chính sách thay đổi lớn), hoặc khi offline và online metrics bất đồng và cần phán quyết cuối cùng. Ba loại bổ sung nhau — offline nhanh và rẻ, online real-time nhưng noisy, human chậm nhưng authoritative.

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
| Tổng số records | ____ / 20 |
| Easy | ____ / 5 |
| Medium | ____ / 7 |
| Hard | ____ / 5 |
| Adversarial | ____ / 3 |
| Source documents được sử dụng | ____ / 10 |
| Validator status | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

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
