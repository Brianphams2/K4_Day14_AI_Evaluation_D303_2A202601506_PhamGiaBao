# Kế hoạch thực hiện Lab AI Evaluation

## 0. Quy ước chung

### Mục tiêu cuối

Hoàn thiện pipeline:

```text
Corpus -> Golden Dataset -> RAG actual answers -> Evaluation Core -> Benchmark Report -> Failure Analysis
```

### Deliverables bắt buộc

- `solution/solution.py`: evaluation core hoàn chỉnh.
- `golden_dataset.json`: 20 QA hợp lệ, có evidence provenance.
- `exercises.md`: worksheet, benchmark 3.2 và rubric 3.3.
- `reflection.md`: report, 3 failure analyses, 5 Whys và regression strategy.

### Artifact trung gian

- `artifacts/actual_answers.json`: output của RAG.
- `artifacts/benchmark_results.json`: output của evaluation.

Artifact chỉ được tạo sau khi phase trước đạt gate; không dùng artifact để thay thế deliverable.

### Quy tắc dữ liệu

- Corpus trong `data/technology_store/` là source of truth duy nhất.
- Không sửa corpus, `domain_assistant.py`, schema ID, difficulty hoặc `attack_type`.
- Evidence trong `contexts[].text` phải là substring nguyên văn của `source_doc` tương ứng.
- Không đưa `expected_answer` hoặc gold context vào lúc sinh actual answer để tránh leakage.
- Không commit `.env`, API key hoặc dữ liệu ngoài repo.
- Câu hỏi và expected answer viết bằng tiếng Anh để khớp corpus/prompt.

---

## Phase 1 — Khởi tạo và baseline

### Mục tiêu

Xác nhận repo, Python và dependency hoạt động trước khi sửa code.

### Nguồn

- `README.md`
- `guide_lab.md`
- `requirements.txt`
- `tests/`

### Thực hiện

```powershell
py -0p
py -3.11 -m venv .venv
.venv\\Scripts\\Activate.ps1
python -m pip install -r requirements.txt
python -c "import openai, dotenv, pytest; print('Environment OK')"
pytest tests/ -v
```

### Input / Output chuẩn hóa

| Input | Output | Điều kiện |
|---|---|---|
| Repo + Python 3.11+ | Môi trường chạy được | Không lỗi import |
| Starter code | Baseline test report | Test được collect; starter có thể fail |

### Bàn giao sang Phase 2

- Ghi nhận Python version và baseline.
- Không còn lỗi collection/import.
- Không tạo hoặc commit API key.

---

## Phase 2 — Hoàn thiện Evaluation Core

### Mục tiêu

Implement tất cả TODO bắt buộc trong `template.py`.

### Nguồn

- `template.py`: interface, dataclass và docstring.
- `tests/test_solution.py`: hành vi cần đạt.
- README/guide: định nghĩa metric và workflow.

### Logic cần triển khai

1. **Data models**
   - Hoàn thiện `QAPair`, `EvalResult`.
   - `overall_score()` là trung bình ba answer-side metrics: faithfulness, relevance, completeness.
   - Retrieval metrics không làm thay đổi overall score gốc.

2. **RAGASEvaluator**
   - Faithfulness: answer overlap với gold contexts.
   - Relevance: answer overlap với question.
   - Completeness: answer overlap với expected answer.
   - Context recall: expected answer overlap với union retrieved contexts.
   - Context precision: Average Precision@K, chunk relevant nếu overlap expected.
   - `run_full_eval(..., contexts=None)` luôn chạy 3 answer metrics; có contexts thì chạy thêm 2 retrieval metrics.

3. **LLMJudge**
   - Build prompt gồm question, answer và rubric.
   - Gọi `judge_llm_fn`, parse scores/rationale an toàn.
   - `detect_bias` trả về position, leniency và severity bias.

4. **BenchmarkRunner**
   - Chạy từng `QAPair` qua `agent_fn` và evaluator.
   - Truyền `pair.retrieved_contexts` vào `contexts`.
   - Report gồm pass rate, average answer metrics và average retrieval metrics.
   - Regression khi metric giảm quá `0.05`.
   - Lọc failure theo threshold.

5. **FailureAnalyzer**
   - Categorize: hallucination, irrelevant, incomplete, off_topic, refusal.
   - Tìm root cause theo metric thấp nhất.
   - Sinh suggestion và Markdown improvement log.

### Input / Output chuẩn hóa

| Component | Input | Output |
|---|---|---|
| `RAGASEvaluator` | `QAPair`, answer, optional contexts | `EvalResult` |
| `BenchmarkRunner` | list `QAPair`, `agent_fn`, evaluator | list results + report |
| `LLMJudge` | question, answer, rubric | scores + reasoning |
| `FailureAnalyzer` | failed results | categories, causes, suggestions, Markdown |

### Kiểm tra

```powershell
pytest tests/test_solution.py -v
```

### Bàn giao sang Phase 3

- Tất cả 42 tests pass.
- Không còn `NotImplementedError` ở TODO bắt buộc.
- `template.py` có type hints và không đọc dữ liệu gold ngoài interface được phép.

---

## Phase 3 — Xây dựng Golden Dataset

### Mục tiêu

Điền đủ 20 QA có chất lượng và evidence kiểm chứng được.

### Phân bổ cố định

- E01–E05: 5 easy.
- M01–M07: 7 medium.
- H01–H05: 5 hard.
- A01–A03: 3 adversarial.

### Nguồn

- `data/technology_store/manifest.json`.
- 10 Markdown documents trong cùng thư mục.
- Template hiện có trong `golden_dataset.json`.

### Quy trình logic

1. Đọc manifest và toàn bộ corpus.
2. Chọn câu hỏi không trùng ý.
3. Viết expected answer ngắn nhưng đủ amount/date/condition/exception.
4. Copy evidence nguyên văn, không sửa punctuation/spacing.
5. Phân loại đúng độ khó.
6. Đảm bảo dùng đủ cả 10 source documents.
7. Với A01–A03, giữ nguyên attack type và dùng evidence scope phù hợp.

### Schema chuẩn

```json
{
  "id": "E01",
  "difficulty": "easy",
  "question": "...",
  "expected_answer": "...",
  "contexts": [{"source_doc": "01_product_catalog.md", "text": "..."}],
  "attack_type": null
}
```

### Kiểm tra và bàn giao

```powershell
python validate_golden_dataset.py
```

Chỉ bàn giao khi validator in:

```text
PASS: dataset structure and evidence provenance are valid.
```

Không được gọi OpenAI trước gate này.

---

## Phase 4 — Sinh actual answers bằng RAG

### Mục tiêu

Chạy system under evaluation độc lập trên 20 question.

### Nguồn và input

- `domain_assistant.py`.
- `golden_dataset.json` đã PASS.
- `data/technology_store/`.
- `.env` chứa `OPENAI_API_KEY` và model.

### Thực hiện

```powershell
Copy-Item .env.example .env
python domain_assistant.py --corpus-dir data/technology_store --dataset golden_dataset.json --output artifacts/actual_answers.json --top-k 5
```

### Output chuẩn

Mỗi record phải có tối thiểu:

- `id` khớp golden dataset.
- `question` khớp tuyệt đối.
- `actual_answer` không rỗng.
- `retrieved_contexts` có source/chunk/text/score.
- `error: null`.

### Quy tắc bàn giao

- Nếu dataset thay đổi, phải chạy lại toàn bộ actual answers.
- Kiểm tra đủ 20 ID trước Phase 5.
- Không sửa actual answer thủ công để tăng điểm.

---

## Phase 5 — Evaluation và Benchmark Report

### Mục tiêu

Đưa artifact vào evaluation core và tạo kết quả benchmark reproducible.

### Thực hiện

```powershell
python evaluate_answers.py --golden golden_dataset.json --actual artifacts/actual_answers.json --output artifacts/benchmark_results.json
```

### Output chuẩn

Report phải có:

- Tổng số cases và pass rate.
- Average Context Recall.
- Average Context Precision.
- Average Faithfulness, Relevance, Completeness.
- Overall score từng case.
- Failure type và ba case thấp nhất.

### Bàn giao sang Phase 6

- Artifact có đủ 20 kết quả.
- Không có `None` bất thường ở retrieval metrics khi retrieved contexts tồn tại.
- Ghi số liệu vào `exercises.md`, không chỉ copy pass rate.

---

## Phase 6 — Exercises, Failure Analysis và Reflection

### `exercises.md`

- Hoàn thành Exercise 3.1: distribution, coverage, representative cases.
- Hoàn thành Exercise 3.2: bảng 5 metrics, overall, passed, failure type, aggregate và 3 case thấp nhất.
- Hoàn thành Exercise 3.3: rubric LLM judge 1–5 domain-specific.
- Rubric phải nêu missing conditions, unsupported claims, privacy/safety failures và verbosity bias.
- Bonus 3.4/3.5 chỉ làm sau phần bắt buộc.

### `reflection.md`

Với ba case thấp nhất:

1. So sánh question, expected, actual và gold/retrieved contexts.
2. Nêu symptom và failure category.
3. Viết 5 Whys tới root cause có thể hành động.
4. Đối chiếu với `find_root_cause()`.
5. Đề xuất fix và metric xác minh.
6. Cluster lỗi và ưu tiên fix có tác động nhiều cases.

### Quy tắc bàn giao

- Mọi kết luận phải dẫn từ artifact hoặc evidence.
- Không kết luận chỉ dựa trên pass rate.
- Improvement log phải có owner/status/metric hoặc tiêu chí xác minh.

---

## Phase 7 — Đóng gói, kiểm tra cuối và nộp

### Đóng gói

```powershell
Copy-Item template.py solution/solution.py
pytest tests/ -v
python validate_golden_dataset.py
git status
git diff --check
```

### Checklist bàn giao cuối

- [ ] `solution/solution.py` là bản mới nhất của `template.py`.
- [ ] 42 tests pass.
- [ ] Golden dataset PASS validator.
- [ ] Đủ 5/7/5/3 theo difficulty.
- [ ] Dùng đủ 10 source documents.
- [ ] `exercises.md` hoàn chỉnh.
- [ ] `reflection.md` có 3 phân tích 5 Whys và regression strategy.
- [ ] Không có `.env`, API key hoặc secret trong diff.
- [ ] Artifacts chỉ là output hỗ trợ, không thay thế 4 deliverables.

### Quy tắc phiên bản và bàn giao

- Mỗi phase chỉ bàn giao output đã qua gate của phase đó.
- Khi sửa input của phase trước, phải rerun các phase downstream bị ảnh hưởng.
- Giữ tên file/schema/interface do starter quy định.
- Commit chỉ sau khi kiểm tra `git diff --check` và rà secret.

