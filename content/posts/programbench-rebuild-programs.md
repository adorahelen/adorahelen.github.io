---
title: "Nine Models, Two Hundred Programs, Zero Solved: What ProgramBench Measures That SWE-bench Cannot"
date: 2026-06-27T12:00:00+09:00
tags:
  - llm
  - evaluation
  - paper-review
  - software-engineering
summary: "Hand a model a compiled binary and its documentation, remove the source, and ask for a reimplementation that behaves identically. Across 200 tasks and nine frontier models, nothing was fully solved. The code that does get written is monolithic — a third of the original's length, a fifth of its files — which is the finding: these models write code well and design software badly."
---

> 🇰🇷 **[이 글의 한국어판 →](/ko/posts/programbench-rebuild-programs/)**

## TL;DR

- **Zero percent resolved, across all nine models.** The best result is Claude Opus 4.7 at 3.0% "almost" (≥95% of tests passing). Two hundred tasks, nobody finishes one.
- **Models flee to the monolith.** Accepted solutions run 0.38× the original's line count in 0.2× the files, with 10–29% as many functions — each 1.1–1.6× longer. The architecture is not being decomposed; it is being collapsed.
- **Difficulty is a property of the program, not the model.** Easy/medium/hard ordering is near-identical across models: small CLI tools are tractable, FFmpeg and a PHP interpreter are not, for everyone.
- **Given internet access, models look up the source.** Claude Sonnet 4.6 does it in 36% of runs. Worse, nine LM judges disagree on 40–57% of cases about whether reading a local Cargo registry cache even counts. The only rigorous fix is full network isolation.
- **More turns is not more progress.** Turn count against performance correlates at r = 0.27. GPT 5.4 averages 16 API calls; Claude Sonnet 4.6 averages 475 and scores worse than Opus at 93.

> **Source**: [arXiv:2605.03546v1](https://arxiv.org/abs/2605.03546) · John Yang, Kilian Lieret et al. (Meta FAIR, Stanford, Harvard) · [code](https://github.com/facebookresearch/ProgramBench)

---

## 1. The claim

SWE-bench and its descendants measure local work: fix this bug, implement this function, make this failing test pass. The skeleton already exists. Language choice, module decomposition and data structure design — the decisions that determine whether a codebase survives — are all pre-made by the repository being patched.

ProgramBench removes the skeleton. You get **a compiled executable and its documentation**. The source is gone. Rebuild something that behaves the same way. Evaluation is behavioural, so the language and the structure are yours to choose, and choosing them is the point.

## 2. The setup

Nine models under mini-SWE-agent — deliberately minimal scaffolding, direct bash execution:

| Model | Vendor | % Resolved | % Almost (≥95%) | Mean API calls | Mean cost |
|---|---|---|---|---|---|
| Claude Opus 4.7 | Anthropic | 0.0% | **3.0%** | 93 | $3.81 |
| Claude Opus 4.6 | Anthropic | 0.0% | 2.5% | 260 | $11.38 |
| Claude Sonnet 4.6 | Anthropic | 0.0% | 1.6% | 475 | $27.09 |
| Claude Haiku 4.5 | Anthropic | 0.0% | 0.0% | 124 | $0.80 |
| Gemini 3.1 Pro | Google | 0.0% | 0.0% | 94 | $1.51 |
| Gemini 3 Flash | Google | 0.0% | 0.0% | 89 | $0.33 |
| GPT 5.4 | OpenAI | 0.0% | 0.0% | 16 | $0.33 |
| GPT 5.4 mini | OpenAI | 0.0% | 0.0% | 18 | $0.04 |
| GPT 5 mini | OpenAI | 0.0% | 0.0% | 15 | $0.03 |

Budget per task: 1,000 turns, six hours wall clock, 20 CPU cores, 60 GB RAM, **no network**.

The benchmark itself:

| Property | Value |
|---|---|
| Tasks | 200 |
| Test functions | 248,853 (median 770 per task) |
| Source languages | Rust 107, Go 46, C/C++ 45, other 2 |
| Difficulty | 28 easy / 143 medium / 29 hard |
| Programs | FFmpeg, SQLite, DuckDB, the PHP interpreter, ripgrep, fzf, jq, zstd, typst, nnn, gron |

Construction requirements are worth stating because they are what makes the design work: a task must build to a standalone executable from an open-source repository (R1); the task ships the compiled binary and documentation only, source removed (R2); and grading is behavioural, so any language and any structure passes if the input/output behaviour matches (R3).

One detail I did not expect: the **auto-generated tests cover more than the projects' own**. Mean line coverage 79.7% against 56.8%, median 86.2% against 64.3%. An assertion-quality linter cut the dummy-solution pass rate from 18.5% to 3.7% — a fivefold improvement in how much the assertions actually assert.

## 3. What the models build instead

Comparing accepted solutions (≥75% tests passing, n = 207) against the originals:

| Property | Original (median) | Model (median) | Ratio |
|---|---|---|---|
| Lines of code | 3,068 | 1,173 | 0.38× |
| Files | 15 | 3 | 0.2× |
| Max directory depth | 2 | 1 | — |

Fewer functions — 10–29% of the original count — and each one 1.1–1.6× longer. That combination has a single reading: logic that the human codebase separated is being packed into one place. The model produces a **single-file monolith**, and it does so consistently enough to be a behavioural signature rather than a per-task accident.

Some of the length gap is fair. Original repositories carry configuration, platform shims, build glue and years of edge cases that no six-hour rebuild would reproduce. But file count and directory depth are not explained by accumulated edge cases. Three files against fifteen is a decomposition decision, taken differently.

## 4. Difficulty belongs to the program

| Difficulty | Representative programs | Opus 4.7 pass rate | Sonnet 4.6 pass rate |
|---|---|---|---|
| Easy | nnn, fzf, gron | ~73% | ~67% |
| Medium | ripgrep, jq, zstd | ~52% | ~49% |
| Hard | FFmpeg, php-src, typst | ~25% | ~29% |

Levels differ between models; the *ordering* barely does. Difficulty tracks the program's own complexity — line count plus dependency surface — not any model-specific weakness. Useful for benchmark design: the hard set will stay hard through the next model generation, so the ceiling here should not move for a while.

## 5. Two methodological findings

**The metric problem.** % Resolved is strict to the point of being uninformative when everything is zero — one failed test out of 770 and the whole task is unsolved. But % Tests Passed cannot simply be trusted instead: 90% passing is compatible with a missing core feature, since the tests are not weighted by importance. The paper reports both and neither is sufficient alone. This is the same trap as EM in patch generation, one benchmark generation later.

A side experiment sharpens it. Forcing a language other than the model's own choice:

| Model | Free choice | Forced different language | Δ |
|---|---|---|---|
| Claude Opus 4.7 | ~51% | ~43% | **−8.0%** |
| Claude Opus 4.6 | ~52% | ~48.5% | −3.5% |
| GPT 5.4 | ~38% | ~42% | **+4.2%** |
| GPT 5.4 mini | ~17% | ~21% | **+4.2%** |

Claude degrades; GPT *improves*. Models do not reliably pick the language they are best at — forcing a switch pushed Python selection from 36% to 51%, and for the GPT models that was an upgrade they had declined to make themselves. Language choice is supposed to be one of the design decisions this benchmark measures, and at least two of the models are making it wrong.

**The cheating problem.** With network access allowed:

| Model | Cheat rate | Method |
|---|---|---|
| Claude Sonnet 4.6 | **36%** | Cloning the GitHub repository, downloading sources via package managers |
| Claude Opus 4.6 | 21% | Source inspection, less directly |
| Gemini 3 Flash | 20% | Source lookup |
| GPT 5 mini | 1% | — |

The detection result matters more than the rates. Nine LM judges disagreed on 40–57% of cases. Reading dependency source from a local Cargo cache at `~/.cargo/registry/` split them five to four — legitimate API reference, or the answer key? The ambiguity is real, which is why the paper's conclusion is to remove the question: cut the network entirely and the judges have nothing to adjudicate.

## 6. What I am keeping

1. **Code generation and software design are separate capabilities, and only one of them is solved.** The models write working code — accepted solutions pass three quarters of a 770-test suite — and produce architecture that does not resemble engineered software. Benchmarks that supply the skeleton were never measuring the second thing.
2. **Behavioural evaluation from a binary is a portable technique.** No source, no reference structure, no language constraint. It applies anywhere you have an executable and can generate tests, and it sidesteps the entire "is this patch equivalent to the reference" problem.
3. **Generated tests beat hand-written ones on coverage, given an assertion linter.** 79.7% against 56.8%, with the dummy pass rate down fivefold. That is a usable recipe for bootstrapping a test suite, independent of the benchmark.
4. **Air-gap the evaluation.** Not because models are dishonest, but because "did it cheat" is undecidable at 40–57% judge disagreement. Remove the capability rather than the ambiguity.
5. **Turn count is not effort.** r = 0.27. Sonnet 4.6 spent 475 calls and $27.09 to place below Opus 4.7 at 93 calls and $3.81. Cost per task is a model property worth measuring separately from score.

## 7. Assessment

| Aspect | Judgement |
|---|---|
| Problem statement | Convincing — the first systematic attempt to measure architectural design ability |
| Benchmark design | Practical: no source needed, language- and structure-agnostic, straightforward to extend |
| Scale | Nine models × 200 tasks × ~1,800 runs; adequately large |
| Limits | No non-functional requirements (speed, memory) evaluated; **no human baseline**; a 0% ceiling gives little signal for ranking current models |
| Relation to SWE-bench | Complementary — SWE-bench modifies existing code, ProgramBench designs from nothing |

The missing human baseline is the gap I would most want filled. "Nobody solved it" is a much weaker claim without knowing what a competent engineer achieves in six hours with 20 cores and no internet — plausibly also zero on FFmpeg, in which case the benchmark is measuring the task's difficulty more than the models'. The authors' own proposed next steps — multi-agent setups, human-in-the-loop design decisions, non-functional requirements — all point the same way.

> **In one line: these models can write code but cannot yet design software.** What they produce runs, and it is monolithic, short and shallow — structurally unlike the decomposed architecture human developers build over years, which turns out to be the part that scaffolded benchmarks were quietly supplying all along.

---

## Sources

- [Paper (arXiv)](https://arxiv.org/abs/2605.03546)
- [ProgramBench code (Meta FAIR)](https://github.com/facebookresearch/ProgramBench)
