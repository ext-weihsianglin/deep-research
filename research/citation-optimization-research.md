# Citation optimization: research findings and stack options

Date: 2026-10-01
Status: Research and proposed experiments; no dataset evaluation has been performed.

## 1. Recommendation

Use the historical citation dataset to test a **weakly supervised, query-conditioned content scorer**. Use that scorer to propose and prioritize edits, then validate the edits in a controlled multi-source answer-generation environment. Treat live-engine validation as a separate stage.

For a work-trial prototype, start with frozen representations, a small predictive model, an evidence-based LLM rubric, and bounded generation. Optimize reusable generator instructions only after the evaluation environment is working. Defer reinforcement learning and generator fine-tuning until measured intervention outcomes justify them.

This is an engineering proposal based on the project brief and cited research. The reported 9,700 rows and five parquet shards come from the user; they have not been independently verified. The linked Drive folder could not be retrieved through browsing. Model quality, costs, and citation uplift remain unknown.

## 2. Data semantics and estimands

The brief specifies five columns: `prompt`, `citation_category`, `href`, `hostname`, and `html_content`. It describes top/bottom labels relative to each hostname, at most five pages in each category per hostname, and prompt associations for both groups. Both groups were surfaced by answer engines to some degree.

The brief also mentions cleaned markdown, but the displayed five-column schema does not include it. Verify whether markdown is supplied separately or must be extracted from HTML.

| Target | Required evidence | Supported by the described dataset? |
| --- | --- | --- |
| Predict the supplied top/bottom label | Historical page–prompt associations | Yes, subject to held-out validation |
| Estimate citation probability for a query and engine | Repeated runs, uncited candidates, exposure counts, engine identity | No |
| Estimate the effect of a page edit | Outcomes under original and edited content | No |

The initial model estimates something closer to:

```text
P(top label | prompt, page content, hostname, selected into dataset)
```

The desired production target is:

```text
P(citation | query, engine, content, retrieval environment)
```

These are different estimands. The five-column schema does not include engine identity, citation counts, observation windows, retrieval ranks, or candidate sets. A high classification score is not a calibrated citation probability. Calibration on the selected dataset cannot recover the missing population prevalence.

## 3. Phase 1: test whether content signals transfer

### Dataset audit

- Count unique hostnames, canonical URLs, content hashes, and prompt clusters.
- Inspect label balance within hostnames and determine how labels were produced.
- Identify repeated pages, near-duplicate templates, scrape failures, empty bodies, language differences, and truncation.
- Inspect query intent and page type. Editorial pages versus careers or navigation pages may explain much of the label separation.
- Verify whether the HTML snapshot matches the content present when citation observations were collected.
- Establish whether top/bottom refers to aggregate page performance or performance for the associated query. This affects which training comparisons are valid.

9,700 rows are enough to attempt lightweight models. Effective sample size depends on independent pages, hostnames, queries, and label quality; row count alone does not establish feasibility.

### Baseline sequence

| Model | Inputs | Role | Main limitation |
| --- | --- | --- | --- |
| Simple baseline | Page type, content length, query relevance | Establish an inexpensive reference | May capture obvious page differences only |
| Regularized feature model | Structural features, semantic relevance, rubric traits | Inspect transferable associations | Feature definitions can encode shortcuts |
| Frozen embeddings with a small classifier | Query and body representations plus interactions | Test richer semantic information | Embeddings may miss fine factual distinctions |
| Rubric-based LLM judge | Query, body, fixed criteria, supporting spans | Extract semantic traits and edit feedback | Bias, instability, and inference cost |

Run a hostname/URL-only diagnostic to detect shortcuts. Compare content-only and metadata-aware models. Keep editable content signals distinguishable from immutable context such as hostname identity.

### Training and evaluation

- Use hostname-disjoint splits as the primary transfer test.
- Keep duplicate pages and near-identical templates together. Audit cross-host duplicates as well.
- Add a separate query-cluster holdout for unfamiliar intents. State which generalization axis each split measures.
- Report hostname-macro balanced accuracy and AUROC where both labels exist, with uncertainty bootstrapped by hostname. Report log loss only as prediction quality for the sampled labels.
- Use pairwise accuracy or ranking metrics only for meaningful comparable groups. Do not interpret binary-label NDCG as measured citation rank.
- Compare gains over query relevance, page type, and length; use ablations to determine what contributes.

Two pages from one hostname may have different associated queries. Their labels do not automatically create a preference pair for one shared query. Pairwise ranking is appropriate only when label provenance and query comparability support the comparison. Otherwise, classify the supplied labels first.

## 4. LLM judges and verifiable rubrics

An LLM can serve as a prototype scorer, but historical-label prediction and editorial assessment are separate jobs. A judge may identify useful content defects without predicting citation labels accurately.

LLM judges have documented position, verbosity, and self-preference biases. Evidence of agreement with human answer preferences does not establish citation prediction validity. [S1](https://arxiv.org/abs/2306.05685)

Recommended procedure:

1. Define a fixed trait schema using training data and domain reasoning.
2. Ask the judge to extract traits and supporting spans without seeing the historical label.
3. Fit a small model mapping traits to historical labels.
4. Evaluate against simpler baselines on grouped holdouts.
5. Freeze the judge, rubric, and scorer before generator optimization.
6. Evaluate generated edits separately from historical-label prediction.

Do not show a label and ask the judge to explain why the page deserved it. Such explanations may rationalize the supplied answer rather than reveal predictive mechanisms.

| Trait | Proposed verification | Verification limit |
| --- | --- | --- |
| Direct query answer | Locate an answer span; assess relevance | Semantic assessment remains model- or human-dependent |
| Intent coverage | Compare with an independently defined subquestion checklist | The checklist itself needs validation |
| Fact preservation | Map edited claims to original or independently verified evidence | Matching text alone does not prove entailment |
| Heading structure and valid link syntax | Parse deterministically | Correct syntax does not imply usefulness |
| Supported statistics and quotations | Check original source and claim support | A plausible URL or quotation is insufficient |
| Low redundancy | Detect duplicate passages and semantic repetition | Some repetition is useful context |

Explanations improve inspectability, not causal identification. Use deterministic checks where possible and evidence-backed semantic judgments where necessary. Unsupported added facts should fail a quality gate even if the learned score improves.

## 5. Counterfactual evaluation environment

The five columns cannot reproduce the original answer engine. Counterfactual learning from logged feedback generally relies on logging-policy information and support for evaluated actions. Unseen rewrites have no observed outcomes here; propensity weighting cannot reconstruct missing intervention evidence. [S2](https://jmlr.org/papers/v16/swaminathan15a.html)

Build a controlled source-substitution experiment:

```text
Query + fixed relevant competing sources
  ├─ original target page → answer generation → supported citation metrics
  └─ edited target page   → answer generation → supported citation metrics

Compare paired outcomes across repetitions, source orders, and models.
```

1. Assemble relevant competing sources for each query; log how they were chosen.
2. Run the original and edited target in separate contexts, retaining the same source identity.
3. Change only target content within a comparison; keep competing content, prompt, model configuration, and context budget fixed.
4. Randomize source order and repeat stochastic generations. Match conditions across variants where the API permits.
5. Measure target citation rate, supported attribution, answer correctness, and completeness.
6. Evaluate transfer on a held-out answer model and unfamiliar hostnames.

Do not put original and edited versions in the same context: that changes the competition. Account for document truncation and length changes, and add length-matched comparisons where appropriate.

This measures citation behavior **conditional on inclusion in the supplied source set**. It does not measure indexing, retrieval, or total live citation probability.

GEO provides a precedent for controlled source-content interventions. Its original experiments found visibility improvements for several transformations, with variation across domains. Importantly, its Perplexity evaluation supplied source texts through file uploads; it did not establish the effect of publishing edits and waiting for normal web retrieval. [S3](https://arxiv.org/html/2311.09735v3)

For live validation, publish approved edits, verify retrieval of the updated content, and use randomized comparable pages or a staggered rollout with controls. Log query panel, engine/version where available, dates, repeated outcomes, and indexing status. The simulator remains a proposal-testing environment until transfer is demonstrated.

## 6. Phase 2: generation and prompt optimization

Proposed interface:

```text
generate(
  instructions,
  hostname,
  target_query,
  extracted_source_content,
  evidence_inventory,
  intent_checklist,
  edit_constraints
) → improved_markdown + claim_evidence_map + edit_notes
```

Better instructions cannot supply missing evidence. If new facts are needed, add a separately verified research step or return an evidence gap. Preserve source facts, qualifications, and important content that the target query does not cover.

Keep raw HTML features separate from the normalized body used to score rewrites. Comparing a raw-HTML original score to a markdown candidate score introduces a representation change. Score both through the same extraction and normalization pipeline. Markdown generation also cannot directly change HTML metadata or structured data; those require a separate rendering or publishing step.

There are two optimization loops:

- Across pages: optimize reusable instructions on development examples, then freeze them for the test set.
- Within a page: generate a small, bounded candidate set and select an acceptable edit.

Best-of-N selection is itself optimization pressure and can exploit an imperfect reward model. Increasing candidate count is not automatically beneficial. [S4](https://arxiv.org/abs/2210.10760)

### Optimizer options

| Option | Use when | Benefits | Risks and limits |
| --- | --- | --- | --- |
| Fixed prompt plus bounded candidates | Establishing the first generation baseline | Simple and auditable | Limited adaptation; selection can still exploit a proxy |
| DSPy MIPROv2 | Optimizing instructions and demonstrations against a stable metric | Systematic prompt search | Can overfit an unreliable development metric |
| GEPA | Feedback includes evidence-backed failure descriptions | Reflective instruction evolution fits the rubric proposal | Feedback can be wrong; requires independent evaluation |
| TextGrad | Exploring local text or instruction refinement | Natural-language feedback supports targeted changes | Another optimization loop around an imperfect evaluator |
| Supervised generator fine-tuning | Verified successful edit pairs accumulate | Can amortize generation behavior | Synthetic edits are not proof of citation improvement |
| Reinforcement learning | A validated environment supplies enough reliable outcomes | Direct optimization of a learned behavior | Higher operational complexity and reward exploitation risk |

MIPRO optimizes instructions and demonstrations for LM programs. GEPA uses trajectories and natural-language reflection to evolve prompts; its published results are on other tasks, so citation gains remain untested here. TextGrad offers text-based optimization feedback. [S5](https://aclanthology.org/2024.emnlp-main.525/) [S6](https://arxiv.org/abs/2507.19457) [S7](https://arxiv.org/abs/2406.07496)

## 7. Reward design and OOD behavior

Use constrained candidate selection: among edits that preserve supported facts and meet quality requirements, prefer those with independently measured citation performance.

Before intervention data exists, label the historical model output a **proxy score** and use it as a proposal filter. Do not present it as citation probability or predicted causal uplift.

Once the controlled environment works, prioritize supported citation uplift and retain answer correctness and completeness as quality constraints. ALCE provides useful methods for evaluating citation support and coverage, though its task differs from predicting whether a particular page is cited. [S8](https://arxiv.org/abs/2305.14627)

Suggested stress tests:

- Keyword repetition and unnecessary length expansion.
- Fabricated statistics, quotations, sources, and implied authority.
- Instructions embedded in page content telling the evaluator to cite it.
- Simple reorganization and length-matched rewrites as controls.
- Judge disagreement and sensitivity to source order or irrelevant formatting.

For OOD detection, use unfamiliarity indicators, grouped validation errors, and model disagreement. These are warning signals, not reliable guarantees. Fall back to conservative edits or abstain when evidence is insufficient. Keep the original as an eligible outcome.

## 8. Recommended implementation stack

These are proposed engineering choices, not measured winners for this dataset. Pin versions when implementing; no dependencies have been installed or tested for this document.

| Component | Initial choice | Alternative | Decision criterion |
| --- | --- | --- | --- |
| Parquet inspection | DuckDB | Polars or pandas with a parquet backend | Prefer SQL for audit queries; dataframes for transformations |
| HTML extraction | Trafilatura plus preserved raw HTML | Beautiful Soup with explicit extraction rules | Inspect representative pages; quantify dropped facts and boilerplate |
| Lightweight modeling | scikit-learn regularized classifier | Boosted trees or a small ranking model | Promote complexity only for grouped held-out gains |
| Semantic features | Frozen embeddings plus query–page interactions | Cross-encoder scoring | Compare marginal quality against inference cost |
| Rubric evaluation | Structured LLM outputs with evidence spans | Human-reviewed rubric subset | Measure label prediction, stability, and evidence validity |
| Generation | Frozen LLM with fixed prompt | GEPA or MIPROv2 after baseline | Optimize only against a working development evaluation |
| Citation environment | Explicit multi-source answer harness | Multiple independently configured harnesses | Log source sets, configuration, and repeated outcomes |
| Experiment records | JSONL plus parquet summaries | Experiment-tracking service | Record provenance and reproducibility before adding infrastructure |
| Demo | CLI or minimal local web interface | Streamlit | Show markdown, evidence map, and separate scores clearly |

Minimum trace fields should include case ID, query, hostname, source-content hashes, extraction version, source-set IDs and order, prompt version, model configuration, candidate edits, gate results, proxy score, answer text, citations, and evaluation results.

## 9. Experiment plan and decision gates

| Hypothesis | Experiment | Evidence required to proceed |
| --- | --- | --- |
| H1: content adds transferable historical signal | Compare models with relevance/type/length baselines on unseen hostnames | A practically useful held-out gain with uncertainty and shortcut audits |
| H2: scorer-guided edits help beyond generic rewriting | Compare original, fixed-prompt rewrite, and guided rewrite in the source-substitution environment | Supported attribution improves without quality degradation |
| H3: gains transfer | Hold out an answer model and test unfamiliar hostnames/intents | Gains persist outside the optimization environment |
| H4: controlled gains predict live outcomes | Approved controlled publishing experiment | Repeated live outcomes improve under an appropriate control design |

Start with approximately 30–50 page–query cases to debug interventions and estimate variance. This is a pilot size, not a powered success test. Determine the larger sample from variance, clustering, and the smallest meaningful uplift. Treat cases or hostnames as the independent unit; repeated generations are not independent new pages.

Work-trial deliverables:

1. Dataset audit and reproducible grouped splits.
2. Baseline comparison with error analysis and feature ablations.
3. Frozen generation baseline and one guided method.
4. Held-out intervention results with uncertainty, quality checks, and failure examples.
5. Demo producing markdown, evidence-backed edit notes, and clearly separated proxy and measured scores.

If H1 fails, retain useful quality rubrics and focus on intervention evidence rather than forcing a historical reward model. If H2 or H3 fails, constrain optimization and investigate evaluator artifacts. Defer autonomous recurring updates until live evidence and monitoring support them.

## 10. Recurring framework roadmap

Proposed cycle:

```text
Collect current observations → audit drift → propose bounded edits
→ verify evidence → evaluate interventions → approve deployment
→ observe live outcomes → version and update models
```

Store engine- and time-specific outcomes instead of collapsing all observations into one label. Distinguish discovery, retrieval, and attribution where instrumentation permits. Google's documentation reinforces that eligibility and indexing remain relevant to appearance in its AI search features; an in-context citation score does not cover these stages. [S9](https://developers.google.com/search/docs/appearance/ai-features)

Keep champion and candidate versions, an untouched evaluation panel, rollbackable content versions, and explicit promotion criteria. Refresh historical-label models using observed outcomes rather than treating generated scores as ground truth.

## 11. Evidence summary and sources

| ID | Source | Relevance | Boundary |
| --- | --- | --- | --- |
| S1 | LLM-as-a-Judge | Judge bias and evaluation design | Human preference agreement is not citation prediction |
| S2 | Counterfactual Risk Minimization | Requirements for logged-feedback evaluation | Missing edit outcomes cannot be recovered from five columns |
| S3 | GEO | Controlled source-content interventions | Visibility metrics and supplied-source experiments do not prove live retrieval uplift |
| S4 | Reward Model Overoptimization | Proxy exploitation under RL and best-of-N | Synthetic results motivate controls, not a dataset-specific threshold |
| S5 | MIPRO | Instruction and demonstration optimization | Task-specific citation gains require evaluation |
| S6 | GEPA | Reflective prompt optimization with language feedback | Published task results do not validate this citation reward |
| S7 | TextGrad | Text-based optimization feedback | Feedback quality remains a dependency |
| S8 | ALCE | Citation support and coverage evaluation | Citation correctness differs from target-page citation likelihood |
| S9 | Google AI features documentation | Retrieval and eligibility boundary | Applies to Google; other engines need separate validation |

- [S1: Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena](https://arxiv.org/abs/2306.05685)
- [S2: Batch Learning from Logged Bandit Feedback through Counterfactual Risk Minimization](https://jmlr.org/papers/v16/swaminathan15a.html)
- [S3: GEO: Generative Engine Optimization](https://arxiv.org/html/2311.09735v3)
- [S4: Scaling Laws for Reward Model Overoptimization](https://arxiv.org/abs/2210.10760)
- [S5: Optimizing Instructions and Demonstrations for Multi-Stage Language Model Programs](https://aclanthology.org/2024.emnlp-main.525/)
- [S6: GEPA: Reflective Prompt Evolution Can Outperform Reinforcement Learning](https://arxiv.org/abs/2507.19457)
- [S7: TextGrad: Automatic Differentiation via Text](https://arxiv.org/abs/2406.07496)
- [S8: Enabling Large Language Models to Generate Text with Citations](https://arxiv.org/abs/2305.14627)
- [S9: AI Features and Your Website](https://developers.google.com/search/docs/appearance/ai-features)

Input: `/Users/ext-weihsiang.lin/Downloads/project-brief.md`. The input file is local and not required to view this report.
