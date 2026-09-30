# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 14:15–17:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 14:15–14:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (14:30–14:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Câu hỏi adversarial (out-of-scope, prompt injection): trợ lý từ chối lịch sự bằng câu chữ riêng, không lặp lại evidence nên overlap với gold context thấp dù hành vi đúng. Câu trả lời diễn đạt lại (paraphrase) chính sách bằng từ đồng nghĩa. | Câu hỏi về chính sách có hệ quả tiền/pháp lý (hoàn tiền, bảo hành, phí restocking, thời hạn đổi trả) mà answer đưa ra số tiền, số ngày hoặc điều kiện không có trong nguồn — khách hàng hành động theo thông tin bịa. | Đọc answer cạnh gold evidence, đánh dấu từng claim không có nguồn. Nếu là bịa thật: siết system prompt "chỉ trả lời từ context", thêm bước kiểm tra claim/citation trước khi trả lời. Nếu chỉ do paraphrase: ghi nhận hạn chế của word-overlap, cân nhắc LLM judge. |
| Answer Relevance | Câu hỏi ngắn, answer đúng nhưng dùng từ khác câu hỏi (ví dụ hỏi "refund" nhưng answer nói "money back"); câu adversarial mà answer đúng là từ chối/chuyển hướng. | Answer trả lời một chủ đề khác (hỏi đổi trả nhưng trả lời bảo hành), hoặc trả lời chung chung không xử lý đúng tình huống khách nêu. | Kiểm tra retrieved chunks có đúng chủ đề không (lỗi retrieval kéo theo lệch chủ đề). Cải thiện prompt yêu cầu trả lời trực tiếp câu hỏi trước, sau đó mới bổ sung thông tin. |
| Context Recall | Câu adversarial/out-of-scope: evidence là quy tắc phạm vi trong `00_system_scope.md`, retriever có thể không lấy về nhưng trợ lý vẫn từ chối đúng nhờ system prompt. | Câu Medium/Hard cần điều kiện hoặc ngoại lệ (ví dụ ngoại lệ bảo hành, phiên bản chính sách mới) mà chunk chứa điều kiện đó không được retrieve — generator buộc phải thiếu ý hoặc đoán. | So sánh gold `source_doc` với `retrieved_contexts[].source_doc`. Điều chỉnh top_k, chunking (không cắt rời điều kiện khỏi quy tắc), query rewriting hoặc hybrid retrieval. |
| Context Precision | Recall đã đủ và chỉ có 1 chunk liên quan nằm ở hạng 2–3; generator vẫn dùng đúng evidence nên answer không bị ảnh hưởng. | Chunk liên quan bị đẩy xuống cuối, nhiều chunk nhiễu từ tài liệu gần chủ đề (ví dụ returns vs warranty) đứng đầu, khiến generator dùng nhầm chính sách. | Thêm reranker (cross-encoder hoặc lexical overlap), lọc theo metadata tài liệu, giảm top_k nếu noise nhiều. Đo lại precision trên cùng tập chunks. |
| Completeness | Expected answer có thêm chi tiết phụ (ví dụ kênh liên hệ) mà answer bỏ qua nhưng vẫn giải quyết được nhu cầu chính; answer paraphrase nên overlap thấp. | Answer bỏ sót điều kiện hoặc ngoại lệ bắt buộc (hạn chót, giấy tờ cần có, trường hợp bị loại trừ) — khách hiểu sai quyền lợi. | Xác định ý bị thiếu có nằm trong retrieved chunks không: nếu không → sửa retrieval; nếu có → sửa prompt/generation (yêu cầu liệt kê điều kiện và ngoại lệ), tăng giới hạn output token. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:* Lấy N cặp answer (A, B) cho cùng một câu hỏi, ví dụ 20 câu trong golden dataset, mỗi câu có hai answer từ hai phiên bản trợ lý. Chạy judge dạng pairwise với hai condition:
> - **Condition 1 (A trước):** prompt đặt A ở vị trí "Response 1", B ở "Response 2".
> - **Condition 2 (B trước):** giữ nguyên nội dung, chỉ đảo vị trí.
>
> Với mỗi cặp, ghi lựa chọn của judge ở cả hai condition. Judge không bias thì phải chọn cùng một answer (theo nội dung) ở cả hai lần. Đo **tỷ lệ inconsistency** (số cặp đổi lựa chọn khi đảo vị trí) và **tỷ lệ chọn "Response 1"** trên tổng 2N lần chấm. Nếu tỷ lệ chọn vị trí đầu cao hơn đáng kể 50% (ví dụ > 60%) hoặc inconsistency cao thì có position bias. Có thể thêm condition 3: hai answer giống hệt nhau, kỳ vọng hòa hoặc 50/50. Nếu judge vẫn thiên về vị trí đầu thì đó là bằng chứng rõ nhất.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> - Chấm theo **tiêu chí rời rạc, kiểm tra được**: mỗi mức điểm mô tả ý phải có (đúng điều kiện chính sách, đúng số ngày/số tiền, có bước hành động tiếp theo), không mô tả "chi tiết/đầy đủ" chung chung.
> - Ghi rõ trong rubric: *"Độ dài không phải là tiêu chí; thông tin thừa không liên quan hoặc không có trong nguồn bị trừ điểm"*. Mỗi claim ngoài evidence bị trừ điểm Correctness.
> - Tách **Conciseness/Clarity** thành một dimension riêng để answer dài dòng không được cộng điểm ở Correctness/Completeness.
> - Cung cấp ví dụ calibration: một answer ngắn nhưng đúng đủ được 5, một answer dài nhưng có một điều kiện sai chỉ được 2–3.
> - Yêu cầu judge liệt kê các ý đúng/sai trước khi cho điểm (chain-of-thought theo checklist), rồi mới quy đổi ra điểm.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:* LLM judge chỉ là một mô hình đo lường. Trước khi dùng điểm của nó làm quality gate, cần chứng minh điểm đó phản ánh đánh giá của chuyên gia. Calibrate bằng cách cho người (ví dụ nhân viên CSKH OrbitTech) chấm một tập mẫu theo cùng rubric, rồi so với judge qua agreement (Cohen's kappa, Spearman correlation, tỷ lệ lệch ≤ 1 điểm). Việc này giúp:
> 1. Phát hiện bias hệ thống (leniency, severity, verbosity, self-preference) mà judge không tự báo.
> 2. Phát hiện chỗ rubric mơ hồ: hai người lệch nhau thì judge cũng không ổn định.
> 3. Chọn ngưỡng block deploy có ý nghĩa thực tế: judge cho 0.7 phải tương ứng với mức người thấy chấp nhận được.
> 4. Theo dõi drift khi đổi model judge hoặc prompt. Không calibrate thì một thay đổi điểm có thể do judge chứ không do hệ thống.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.7 (trung bình) | Trợ lý CSKH trả lời về tiền, đổi trả, bảo hành. Thông tin bịa gây thiệt hại trực tiếp cho khách và cửa hàng, nên đây là metric chặn nghiêm nhất (theo bài giảng: faithfulness < 0.7 thì không deploy). Kèm điều kiện: không được giảm quá 0.05 so với baseline. |
| Answer Relevance | 0.6 (trung bình) | Word-overlap với câu hỏi bị ảnh hưởng bởi paraphrase và các câu từ chối adversarial, nên đặt ngưỡng thấp hơn để tránh chặn nhầm. Dưới 0.6 thường là trả lời lạc đề thật. |
| Completeness | 0.6 (trung bình) | Thiếu điều kiện hoặc ngoại lệ gây hiểu sai quyền lợi, nhưng expected answer do người viết thường dài và dùng từ khác answer, nên overlap khó đạt cao. Chặn ở 0.6, cảnh báo ở 0.6–0.7. Mọi regression > 0.05 phải được review. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline evaluation** (golden dataset cố định, chạy trong CI): trước mỗi lần merge hoặc release, khi đổi prompt, model, chunking, top_k hoặc retriever. Mục đích là phát hiện regression trên cùng input, lặp lại được và rẻ. Đây là quality gate chặn deploy.
> - **Online evaluation** (trên traffic thật sau deploy): theo dõi các tín hiệu như tỷ lệ escalate sang nhân viên, CSAT/thumbs-down, tỷ lệ từ chối, độ trễ, và chạy judge tự động trên mẫu hội thoại thật. Dùng để phát hiện drift và các câu hỏi mà golden dataset chưa bao phủ, rồi bổ sung chúng vào dataset (Augment).
> - **Human review:** calibrate LLM judge; xử lý các case judge không chắc hoặc hai metric mâu thuẫn; review định kỳ các chủ đề rủi ro cao (hoàn tiền, dữ liệu cá nhân, bảo mật tài khoản, thay đổi chính sách); duyệt expected answer khi chính sách cập nhật; điều tra 5 Whys cho failure nghiêm trọng.

---

## Part 2 — Core Coding (14:45–15:40)

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

## Part 3 — Golden Dataset & Real Benchmark (15:40–16:35)

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

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

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

### Exercise 3.4 — Framework Comparison (Bonus +5)

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

## Part 4 — Reflection (16:35–16:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 16:50–17:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
