# ErrCalBench: Error Calibration Benchmark for Code LLMs

**ErrCalBench** (Error Calibration Benchmark) is a diagnostic evaluation benchmark designed to assess both **code generation quality** and **error self-diagnosis accuracy** in code language models.

Existing code benchmarks evaluate only functional correctness (pass/fail). ErrCalBench goes further by measuring whether a model can accurately predict *why* its solution fails — a metacognitive ability critical for reliable AI-assisted software engineering.

## Key Features

- **178 manually authored Python problems** across three difficulty tiers
- **20 verified test cases per problem** (3,560 total), covering normal inputs, boundary conditions, and stress tests
- **Dual-axis evaluation**: functional correctness (pass@1) + metacognitive calibration (CCS, diagnosis accuracy, quadrant fractions)
- **Five error types**: Passed, Runtime Error, Wrong Answer, Timeout, Compile Error
- **Designed for blindspot detection**: test cases are crafted to trigger specific error patterns

## Dataset Statistics

| Tier | # Problems | Topics | Base Pass Rate (Qwen3-1.7B) |
|------|-----------|--------|------------------------------|
| **Easy** | 78 | Prefix sums, binary search, sorting, string ops, BFS/DFS, GCD/LCM, primality, Caesar cipher, matrix ops | ~45% |
| **Medium** | 50 | Two sum, merge intervals, knapsack, coin change, edit distance, LIS/LCS, topological sort, Dijkstra, MST, SCC, Euler path, modular exponentiation, CRT | ~18% |
| **Hard** | 50 | Segment trees (lazy propagation), BIT, bitmask DP (TSP), digit DP, convex hull, max bipartite matching, articulation points/bridges, LCA, HLD, Dinic's max flow, 2-SAT, suffix arrays, Manacher's, matrix exponentiation, Mo's algorithm, meet-in-the-middle, Kadane 2D | ~6% |

### Error Type Distribution (Base Qwen3-1.7B)

| Passed | Runtime Error | Wrong Answer | Timeout | Compile Error |
|--------|--------------|--------------|---------|---------------|
| 28% | 18% | 35% | 12% | 7% |

## Data Format

The dataset is in **JSONL** format (one JSON object per line). Each entry contains:

| Field | Type | Description |
|-------|------|-------------|
| `input` | `List[Dict]` | Conversational messages (`role: "user"`, `content: problem description`) |
| `label` | `JSON String` | Contains `inputs` (20 test inputs), `outputs` (20 expected outputs), `testtype` ("stdin"), `fn_name` (""), `time_limit` (2 seconds) |
| `datasource` | `String` | `"custom_problems"` |
| `reference_solution` | `String` | Verified reference implementation in Python |

### Example Entry (Simplified)

```json
{
  "input": [
    {
      "role": "user",
      "content": "You are a coding expert. ... Two Sum\nGiven N integers and a target value T, find two distinct indices..."
    }
  ],
  "label": "{\"inputs\": [\"5 -584594\\n-438641 ...\", ...], \"outputs\": [\"1 4\", ...], \"testtype\": \"stdin\", \"fn_name\": \"\", \"time_limit\": 2}",
  "datasource": "custom_problems",
  "reference_solution": "import sys\ndef p1_s(inp):\n    lines = inp.strip().split('\\n')\n    n, t = int(lines[0].split()[0]), int(lines[0].split()[1])\n    a = list(map(int, lines[1].split()))\n    seen = {}\n    for i, x in enumerate(a):\n        if t - x in seen:\n            j = seen[t - x]\n            return f\"{min(j,i)+1} {max(j,i)+1}\"\n        seen[x] = i\n    return \"-1\"\nprint(p1_s(sys.stdin.read()))\n"
}
```

## Evaluation Metrics

ErrCalBench supports evaluation on two axes:

### 1. Functional Correctness
- **pass@1**: Whether a single sampled solution passes all test cases

### 2. Metacognitive Calibration

- **CCS (Calibrated Confidence Score)**: Measures confidence calibration (higher is better)
- **Diagnosis Accuracy**: Percentage of correct error type predictions across the 5 error categories
- **Four Quadrant Fractions** (predicted vs. actual execution outcome):

| Quadrant | Predicted | Actual | Interpretation |
|----------|-----------|--------|----------------|
| **A** (desired) | Pass | Pass | Correct and knows it |
| **B** (blindspot) | Pass | Fail | Overconfident failure — most harmful |
| **C** (underconfident) | Fail | Pass | Missed opportunity |
| **D** (aware) | Fail | Fail | Knows it doesn't know |

### Error Type Codes

| Code | Type | Typical Triggers |
|------|------|-----------------|
| 0 | Passed | Correct solution |
| 1 | Runtime Error | IndexError, TypeError, ValueError, KeyError, ZeroDivisionError, RecursionError |
| 2 | Wrong Answer | Incorrect output |
| 3 | Timeout | Exceeds time limit (inefficient algorithm) |
| 4 | Compile Error | SyntaxError, IndentationError, ImportError, NameError |

## Quality Assurance

- All 178 problems have **reference solutions verified against all 20 test cases**
- Two authors **independently solved 20% of problems** (36 problems) to verify:
  - Problem statements are unambiguous
  - Test cases are correct
  - Difficulty labels match intended tier
- **Baseline validation**: Three Qwen3 models (1.7B, 4B, 8B) were run to verify all five error types appear in the output distribution

## Quick Start

### Loading the Dataset

```python
import json

problems = []
with open("data/ErrCalBench.jsonl", "r") as f:
    for line in f:
        problems.append(json.loads(line))

print(f"Total problems: {len(problems)}")  # 178

# Access a problem
problem = problems[0]
prompt = problem["input"][0]["content"]
label = json.loads(problem["label"])
test_inputs = label["inputs"]    # 20 test inputs
test_outputs = label["outputs"]  # 20 expected outputs
time_limit = label["time_limit"] # 2 seconds
reference = problem["reference_solution"]
```

### Running Evaluation

```python
import subprocess
import json

def evaluate_solution(code: str, test_input: str, expected_output: str, timeout: int = 10) -> str:
    """
    Execute a solution and classify the result.
    Returns one of: 'passed', 'runtime_error', 'wrong_answer', 'timeout', 'compile_error'
    """
    try:
        result = subprocess.run(
            ["python3", "-c", code],
            input=test_input,
            capture_output=True,
            text=True,
            timeout=timeout
        )
    except subprocess.TimeoutExpired:
        return "timeout"

    if result.returncode != 0:
        stderr = result.stderr
        for err in ["SyntaxError", "IndentationError", "ImportError", "NameError"]:
            if err in stderr:
                return "compile_error"
        return "runtime_error"

    if result.stdout.strip() == expected_output.strip():
        return "passed"
    return "wrong_answer"
```

## File Structure

```
ErrCalBench/
├── README.md
├── LICENSE
├── .gitattributes          # Git LFS tracking rules
├── data/
│   └── ErrCalBench.jsonl   # 178 problems, ~225 MB (tracked by Git LFS)
```

## Citation

If you use ErrCalBench in your research, please cite:

```bibtex
@inproceedings{score2026,
  title={SCORE: Self-Calibrated On-Policy Reinforcement Learning with Execution Feedback for Code Generation},
  booktitle={IEEE/ACM International Conference on Automated Software Engineering (ASE)},
  year={2026}
}
```

## License

This dataset is released under the [MIT License](LICENSE).
