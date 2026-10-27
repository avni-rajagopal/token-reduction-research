# Applying Token Reduction Strategies To Reduce Reasoning Inefficiency In Agentic LLM Systems

**Using attention-based prompt compression to decrease tokens in a multi-agent system**

Avni Rajagopal - Synopsys Science Fair 2026 ASEI Award Winner

📄 [Poster](poster.pdf)

---

## Overview

Agentic LLM systems use multiple agents, reasoning, and tool use, making them powerful but resource-intensive. Long prompts and long outputs require more tokens, which increases the cost and processing time of running agentic AI systems and prevents scalability.

This project aims to maintain agentic AI's performance while reducing the cost by using fewer tokens. 

## Method

I built a multi-agent system consisting of nodes with different tasks using GPT-4o mini. I applied attention-based prompt compression for prompts and between each node, using an open source auxiliary model to compute scores. I compared pruning sentences to meet a budget and to meet a percentage of kept sentences. I tested compression at several levels. I compared two auxiliary models for attention scores. Each of these experiments used the [AgentIF](https://arxiv.org/abs/2505.16944) general agentic benchmark for evaluation. 

### Agentic LLM System

AgentIF was used for the prompts. Each prompt was passed to a planner node, a researcher node, a builder node, and finally a critic node that first critiqued the draft, and then produced the final output for a total of five calls.

### Attention-based compression

- Auxiliary LLM computed attention, weights from prompt text as keys and, for intermediary inputs, node-specific probe as queries. Compression asks: which spans are most important to the specific node?
- **Initial prompt:** scored once and stored separately, used generic `"Answer: "` probe.
- **Between nodes:** output of previous node compressed during the run with node-specific probes (`"Research:"`, `"Deliverable:"`, `"Critique:"`, `"Response:"`, etc).
- Aggregated token scores, removed sentences to either meet a budget or reach a sentence percentage kept

### Two pruning strategies

- **Budget-based:** removed sentences until a token budget was met. For each node, budgets were set based on the 25th percentile, median, and 75th percentile length of that node's output in the uncompressed run.
- **Percentage-based:** removed sentences until a percentage of sentences, either 25%, 50%, or 75%, was kept. 

### Data Collected

Each experiment tested the system on 128 of AgentIF's 707 queries, a sample stratified by prompt length and constraint number. Each experiment's outputs were evaluated using AgentIF's code-based and LLM-based checks to determine if the constraints associated with each query were met. 

For each experiment, I collected these numbers:

- CSR (constraint success rate): proportion of all constraints were met
- ISR (instruction success rate): proportion of all instructions for which all constraints were met
- Mean length of text passed to LLMs (input, plan, research, build, and critique) in characters

### Experiments

Pipeline notebooks (00–09) are in experiments; the other notebooks are in notebooks.

| # | Auxiliary model | Strategy | Level | Initial prompt | Planner | Researcher | Builder | Critic |
|---|---|---|---|---|---|---|---|---|
| 0 | — | Control (no compression) | — | — | — | — | — | — |
| 1 | Qwen2.5-7B | Budget | Median | 859 | 323 | 150 | 370 | 516 |
| 2 | Qwen2.5-7B | Budget | 25th percentile (P25) | 405 | 77 | 58 | 69 | 339 |
| 3 | Qwen2.5-7B | Budget | 75th percentile (P75) | 2572 | 742 | 251 | 733 | 734 |
| 4 | Qwen2.5-7B | Percentage | 50% of sentences | 0.50 | 0.50 | 0.50 | 0.50 | 0.50 |
| 5 | Qwen2.5-7B | Percentage | 25% of sentences | 0.25 | 0.25 | 0.25 | 0.25 | 0.25 |
| 6 | Qwen2.5-7B | Percentage | 75% of sentences | 0.75 | 0.75 | 0.75 | 0.75 | 0.75 |
| 7 | Llama-3.1-8B-Instruct | Budget | Median | 859 | 323 | 150 | 370 | 516 |
| 8 | Llama-3.1-8B-Instruct | Budget | P25 | 405 | 77 | 58 | 69 | 339 |
| 9 | Llama-3.1-8B-Instruct | Budget | P75 | 2572 | 742 | 251 | 733 | 734 |


## Results

| Run | Chars passed (vs. control) | CSR (change) | ISR (change) |
|---|---|---|---|
| Control | 15,106 | 62.5% | 30.7% |
| Qwen median budget | 6,995 (−54%) | 52.3% (−16%) | 27.3% (−11%) |
| Qwen P25 budget | 3,606 (−76%) | 47.1% (−25%) | 24.2% (−21%) |
| Qwen P75 budget | 10,681 (−29%) | 55.4% (−11%) | 28.9% (−6%) |
| Qwen 50% of sentences | 9,964 (−34%) | 55.1% (−12%) | 16.7% (−46%) |
| Qwen 25% of sentences | 6,119 (−59%) | 51.8% (−17%) | 11.2% (−64%) |
| Qwen 75% of sentences | 13,140 (−13%) | 56.6% (−9%) | 28.2% (−8%) |
| Llama median budget | 6,897 (−54%) | 56.2% (−10%) | 28.8% (−6%) |
| Llama P25 budget | 3,722 (−75%) | 46.4% (−26%) | 19.5% (−36%) |
| Llama P75 budget | 10,842 (−28%) | 56.7% (−9%) | 25.2% (−18%) |

Percent changes are relative to the control: (control − run) / control.

### Takeaways

- Prompt compression notably decreased average total length passed to GPT
- Performance drops, but even less aggressive compression saves
- 25th percentile budget compression is cheapest (76% reduction in length) but also 25% CSR reduction, 21% ISR reduction
- Median budget similar performance to 75th percentile, but much cheaper
- For CSR, percentage-based marginally better than budget-based, but for ISR, budget notably better than percentage → choose budget
- Better performance not always greater length; sometimes more compression → equal or better results

## Limitations

- **Small sample, single runs.** With experiments run once on just 128 instructions, small differences of a few points may not be significant; no significance testing was done. GPT-4o mini and the GPT-4o judge are also not fully deterministic.
- **Characters passed is approximate.** Each prompt and intermediate output was counted once, but the goal was sent to all five model calls, plan and draft were used twice, and each node's fixed instructions and generated outputs were not counted. Savings may therefore be smaller than the input character reduction suggests.
- **Sentence splitting is simplistic.** Text is split only on `.`, `!`, `?`, and blank lines, which often failed to break up Chinese prompts that used different punctuation or broke up other structures that used punctuation, such as numbered lists. Budgets also used a rough 4-characters-per-token estimate rather than exact token counts.
- **Budgets matched evaluation data.** Budget sizes were directly derived from the control run, which used the same 128 instructions used to evaluate the system when compressed.
- **Attention scoring.** Longer sentences are favored because they have more tokens and more scores, overlapping attention windows resulted in double-counting, and layer count and probe wording were not varied in this set of experiments, leaving an area for future research.
- **Results reflect this specific pipeline.** AgentIF was designed for single models, and other agent architectures, flows, or models may not reproduce these results.

## Repository organization

| Notebook | Purpose |
|---|---|
| `01_AgentIF_Dataset_Process.ipynb` | Downloads the AgentIF dataset and saves it as `eval_criteria.pkl` |
| `02_Attention_Score_Retrieval.ipynb` | Scores each prompt's sentences using an auxiliary model; can switch between Qwen and Llama with `AUX_CHOICE`. Scores are saved locally |
| `03_Downsampling_Data.ipynb` | Gets the stratified 128-instruction sample from the full dataset |
| `04_Evaluate_With_Constraints.ipynb` | Runs AgentIF's evaluator on an experiment's outputs, select with EXPERIMENT_NUM |
| `05_Data_Analysis.ipynb` | Computes CSR/ISR and characters passed, plots data; also used to calculate length percentiles for budgets |
| `00_GPT_DEFAULT_Experiment_Pipeline.ipynb` | Control experiment / the agentic system with no compression |
| `0{1–9}_GPT_TOKENRED_{1–9}_Experiment_Pipeline.ipynb` | Experiments 1–9 (see experiments table) |
| `00_GPT_DEFAULT_Prompt_Get.ipynb`, `0{1–6}_GPT_TOKENRED_{1–6}_Prompt_Get.ipynb` | Save the compressed initial prompts; experiments 7–9 do this inside their pipeline notebooks |

Budgets in experiments 1–3 and 7–9 are in approximate tokens (`_approx_tokens` = characters / 4).

## How to reproduce

The notebooks were written for **Google Colab** with Google Drive mounted. Notebooks were run in Google Colab in early 2026 with the library versions current at the time, and newer versions may require minor changes.

### 1. Requirements

- **GPU:** an A100 (or similar large-memory GPU). The auxiliary models run with full attention outputs, and experiments 1–9 also load the auxiliary model to compress outputs between nodes.
- **API keys** (add under **Secrets**, the key icon in Colab's left sidebar, with notebook access enabled):
  - `OPENAI_API_KEY`: used by GPT-4o mini in the pipelines and the GPT-4o judge in evaluation
  - `HF_TOKEN`: used to download the auxiliary models
- **Llama access:** `meta-llama/Llama-3.1-8B-Instruct` is gated. Accept Meta's license on its Hugging Face page before running.
- **Cost:** each pipeline run makes five GPT-4o mini calls per instruction, and evaluation makes GPT-4o judge calls.

### 2. Folder setup

All paths point to `/content/drive/MyDrive/TokenReductionProject/`, which resolves to your own Drive once mounted. Create this structure first:

```python
from google.colab import drive
drive.mount('/content/drive')

import os
for d in ["Data", "Results/AgentOutputs", "Results/Prompts", "Results/ConstraintEvalResults"]:
    os.makedirs(f"/content/drive/MyDrive/TokenReductionProject/{d}", exist_ok=True)
```

### 3. Run order

1. **`01_AgentIF_Dataset_Process`**: creates `Data/eval_criteria.pkl`.
2. **`02_Attention_Score_Retrieval`** with `AUX_CHOICE = "qwen"`: scores all 707 instructions with Qwen2.5-7B.
3. **`03_Downsampling_Data`**: run all cells. This creates the 128-instruction Qwen-scored file and `eval_criteria_DOWNSAMPLED.pkl`.
4. **`02_Attention_Score_Retrieval`** with `AUX_CHOICE = "llama"`: scores the 128-instruction sample with Llama-3.1-8B-Instruct.
5. **`00_GPT_DEFAULT_Experiment_Pipeline`** (control).
6. **`05_Data_Analysis`** ("Compression budgets" bottom section): prints the control-run length percentiles. These are the budgets hard-coded in experiments 1–3 and 7–9.
7. **`01`–`09` `_Experiment_Pipeline`** notebooks.
8. **Prompt_Get notebooks** (`00`–`06`): save the initial prompts used for length analysis.
9. **`04_Evaluate_With_Constraints`**: once per experiment, setting `EXPERIMENT_NUM` each time.
10. **`05_Data_Analysis`** (top section only): computes the results and plots.

GPT-4o mini at temperature 0 and the GPT-4o judge are not fully deterministic, so re-runs will give similar but not identical numbers.

## References

1. Qi, Y., et al. (2025). AGENTIF: Benchmarking Instruction Following of Large Language Models in Agentic Scenarios. arXiv:2505.16944.
2. Kong, Z., et al. (2025). Token Reduction Should Go Beyond Efficiency in Generative Models — From Vision, Language to Multimodality. arXiv:2505.18227.
3. Hassid, M., et al. (2025). Don't Overthink it. Preferring Shorter Thinking Chains for Improved LLM Reasoning. arXiv:2505.17813.
4. Zeng, Y., et al. (2025). S²-MAD: Breaking the Token Barrier to Enhance Multi-Agent Debate Efficiency. NAACL 2025.
5. Dong, P., et al. (2025). Can Compressed LLMs Truly Act? An Empirical Evaluation of Agentic Capabilities in LLM Compression. ICML 2025.

Some of the code in this project was generated with generative AI (GPT-5.2).
