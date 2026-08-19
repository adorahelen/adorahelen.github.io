---
title: "Five Documents Are Enough to Poison a RAG System — and What It Takes to Catch Them"
date: 2026-06-27T10:00:00+09:00
tags:
  - ai-security
  - rag
  - llm
  - paper-review
summary: "Inject five malicious texts into a knowledge base of 2.6 million and the model returns the attacker's answer over 90% of the time (PoisonedRAG, USENIX Security 2025). Perplexity detection fails because the injected text is well written. I read the attack and a defence (EcoSafeRAG) together, and worked out what the pair means for anyone running RAG over security documents."
---

> 🇰🇷 **[이 글의 한국어판 →](/ko/posts/rag-security-poisonedrag-ecosaferag/)**

## TL;DR

- **The knowledge base is an attack surface.** Everyone hardens the retriever and the model; the corpus is where the actual injection happens. Five documents against millions is enough for a 90–99% attack success rate.
- **The obvious defences do not work.** Perplexity detection scores AUC 0.30 — the malicious text was written by a strong model, so it reads better than the corpus average. Deduplication does nothing because each injected document is generated separately.
- **The defence that works refuses to ask the model.** EcoSafeRAG splits retrieved documents into sentences, flags sentences suspiciously similar to the query, and checks context diversity with planted bait — no reliance on the model's own knowledge, which is the point of using RAG in the first place. Attack success drops to 0–3% while token use falls by up to 80%.
- **If you run RAG over security documents, this is your problem.** A poisoned corpus that says "this IP is benign" is an attack on your detection pipeline, not a hallucination.

---

## Paper 1: PoisonedRAG — poisoning the knowledge base

> **Venue**: USENIX Security 2025
> **Authors**: Wei Zou et al. (Penn State, Illinois Tech)
> **Claim**: inject five malicious texts into a RAG knowledge base and the model produces the attacker's chosen answer with over 90% probability.

### How the attack works

Of the three components of a RAG system — knowledge base, retriever, model — the **knowledge base is the new attack surface**.

```
Attacker:
  1. Pick a target question:  "Who is the CEO of OpenAI?"
  2. Pick a target answer:    "Tim Cook"  (false)
  3. Craft malicious text and inject it into the knowledge base

malicious text = S ⊕ I
  S: makes it retrievable for the target question   (retrieval condition)
  I: induces the model to emit the target answer    (generation condition)
```

### The two conditions

| Condition | Requirement | How it is met |
|---|---|---|
| **Retrieval** | The malicious text must be retrieved for the target question | Black box: prepend the target question itself. White box: optimise embedding similarity with HotFlip |
| **Generation** | The retrieved text must convince the model to emit the target answer | Use a strong model to write "a document in which this question has this answer" — two or three attempts on average |

Note what the black-box retrieval trick is: the attacker simply puts the question in the document. It works because similarity search is doing exactly what it was asked to do.

### Attack performance

| Dataset | KB size | Black-box ASR | White-box ASR | Texts injected |
|---|---|---|---|---|
| NQ | 2,681,468 | **97%** | **97%** | 5 |
| HotpotQA | 5,233,329 | **99%** | **94%** | 5 |
| MS-MARCO | 8,841,823 | **91%** | **90%** | 5 |

- Consistently high across eight models (GPT-4, PaLM 2, LLaMA-2, Vicuna and others)
- Still 70–87% against advanced RAG variants (Self-RAG, CRAG)
- 94–100% against a Wikipedia-backed chatbot over 21 million texts

Five documents. Against 8.8 million.

### Why the standard defences fail

| Defence | ASR reduction | Why it falls short |
|---|---|---|
| Paraphrasing | 97% → 87% | Still high |
| Perplexity detection | AUC 0.30 | The text was written by a strong model, so its perplexity looks *better* than the corpus |
| Deduplication | No change | Each malicious text is generated independently |
| Knowledge expansion (larger k) | 97% → 41% at k=50 | Cost explodes and it is still not a defence |

The perplexity result is the one worth sitting with. The classic assumption — injected content looks statistically odd — inverts once the injection is written by a language model.

### Why this is dangerous in practice

- The attack costs two or three model queries and under a second of compute
- Black box: no access to the retriever or the model required
- Injection paths are mundane — edit a wiki, publish a page, host a document that gets crawled
- Effect on non-targeted questions is 0–0.4%, so the system looks healthy

That last point is the operationally nasty one. A poisoned corpus does not degrade generally. It answers one question wrongly and everything else correctly.

---

## Paper 2: EcoSafeRAG — a defence that does not consult the model

> **Source**: [arXiv:2505.13506](https://arxiv.org/abs/2505.13506) (May 2025)
> **Authors**: Ruobing Yao et al. (Institute of Information Engineering, Chinese Academy of Sciences)
> **Claim**: sentence-level segmentation plus bait-guided context diversity checking detects malicious content without relying on the model's internal knowledge.

### What was wrong with earlier defences

| Defence | Problem |
|---|---|
| RobustRAG | Degrades once there are two or more malicious documents |
| TrustRAG stage 2 | Relies on the model's internal knowledge — which contradicts the reason you deployed RAG |
| InstructRAG | 7.7× token consumption and requires fine-tuning |

The TrustRAG objection is the interesting one. If your defence asks the model "does this look right to you?", you have reintroduced the dependency on parametric knowledge that RAG exists to avoid.

### The architecture

```
query → [1. sentence segmentation] → [2. core sentence screening] → [3. bait-guided diversity check] → filtered context → model
```

**Stage 1 — sentence-level segmentation.** Split retrieved documents into sentences. Most attacks are optimised at the document level, so segmentation dismantles the structure. It also reduces redundancy in legitimate content and surfaces the load-bearing sentences. **On its own** this yields 26% higher accuracy than vanilla RAG at 80% fewer tokens.

**Stage 2 — core sentence screening**, with two thresholds:

```
θ      = τ × max(sim)   ← adaptive, relative threshold
τ_abs  = 0.92           ← absolute threshold

candidates A  = { s | sim(s, Q) ≥ θ }
suspicious Ip = { s | sim(s, Q) ≥ τ_abs }   ← too similar to the query is itself suspicious
```

99.97% of legitimate data sits below 0.92, so false positives are rare — and PoisonedRAG's `S = Q ⊕ I` construction necessarily scores high, because the attacker put the query in the document.

**Stage 3 — bait-guided diversity check.** The insight: legitimate documents have diverse context (multiple sources, different phrasings), while malicious ones generated from a template have uniform context.

```
1. Embed the context of each candidate (the other sentences in its document)
2. Add bait samples: 4 types × 4 = 16
   (emotional manipulation, forced choice, role-play, prompt injection)
3. DBSCAN clustering (ε = 0.6, min_samples = 4)
4. Anything clustering with the bait is judged malicious
```

The bait solves a cold-start problem: with only one malicious document, clustering classifies it as noise. The bait raises local density around malicious content so a cluster forms at all. At N=1, ASR drops from 35% to **0%**.

### Results

With five malicious documents injected:

| Model | Defence | Tokens | GCG ACC | PIA ACC/ASR | Poison ACC/ASR | Clean ACC |
|---|---|---|---|---|---|---|
| **Vicuna** | Vanilla RAG | 1315 | 24 | 2/57 | 14/72 | 39 |
| | TrustRAG S1 | 1078 | 29 | 40/3 | 47/2 | 40 |
| | **EcoSafeRAG** | **266 ↓80%** | **63** | **62/3** | **63/0** | **63 ↑23%** |
| **Llama2** | Vanilla RAG | 1315 | 17 | 14/82 | 23/70 | 51 |
| | TrustRAG S1 | 1078 | 29 | 54/3 | 48/3 | 54 |
| | **EcoSafeRAG** | **266 ↓80%** | **63** | **63/1** | **63/1** | **63 ↑14%** |
| **Llama3** | Vanilla RAG | 1315 | 6 | 28/67 | 13/77 | 57 |
| | InstructRAG ICL | 2320 ↑77% | 15 | 73/23 | 49/47 | 74 ↑17% |
| | **EcoSafeRAG** | **452 ↓66%** | **64** | **64/0** | **63/0** | **64 ↑7%** |

| Metric | EcoSafeRAG |
|---|---|
| Peak ASR | **3%** across models and attacks |
| Clean accuracy gain | +23% (Vicuna), +14% (Llama2), +7% (Llama3) |
| Token reduction | 48–80% versus vanilla RAG |
| Latency | 1.2× vanilla RAG |
| Clustering time | 0.29 s |

The result I did not expect: **clean accuracy goes up**. Segmentation removes redundancy, so the model sees denser context. The defence pays for itself even with no attacker present — which is the rarest property a security control can have.

Per-stage contribution (Vicuna):

| Stage | Clean ACC | PIA ASR (N=1) | PIA ASR (N=5) | Poison ASR (N=5) |
|---|---|---|---|---|
| Stage 1 (segmentation only) | 65 (+26) | 35 | 73 | 0 |
| Stage 2 (+ screening) | 59 | 2 | 2 | 0 |
| Stage 3 (+ bait) | 63 | **0** | **0** | **0** |

---

## Reading the pair together

```
PoisonedRAG (attack)                     EcoSafeRAG (defence)
─────────────────────────────────────────────────────────────────────────
Q ⊕ Md structure forces retrieval   →    segmentation splits Q from Md; high similarity flagged
Md generated from a model template  →    context diversity check catches the template pattern
Black box, no access needed          →    no use of model internal knowledge (keeps RAG's premise)
5 texts for 90%+ ASR                 →    ASR down to 0–3%
Neutralises prior defences           →    plug and play, no fine-tuning
```

Each defence stage lands on a specific attack property:

1. **Segmentation** splits `Q ⊕ Md` into `Q` and the sentences of `Md`, dismantling the construction.
2. **Screening** flags `Q` itself, because embedding the query in the document is what made it retrievable.
3. **Diversity checking** catches `Md`'s sentences, because template generation makes their context uniform.
4. **Bait** makes detection work even at N=1, where clustering alone sees noise.

---

## Limits and open questions

**PoisonedRAG**: focused on close-ended questions; open-ended is untested. Attacks are optimised per target question rather than jointly. Not every case reaches 100% ASR.

**EcoSafeRAG**:

| Limit | Detail |
|---|---|
| DBSCAN ε | Needs per-dataset tuning; fixed at 0.6 here |
| Bait design | Depends on knowledge of existing attack patterns, so it needs updating for novel ones |
| Segmentation overhead | Preprocessing cost at scale, mitigable with precomputed indexing |
| Adaptive attackers | Untested — what happens when the attacker knows EcoSafeRAG is there? |

The open questions I would want answered:

- Can an attacker generate malicious documents with **deliberately diverse context** and walk past the diversity check?
- What about attacks at word or token level rather than sentence level?
- What does any of this look like for **multimodal RAG**, where the injected content is an image?

---

## Why this matters if you run RAG over security documents

I maintain a RAG system that answers questions over security documentation, so I read this as an operational problem rather than an academic one. The [on-premise agent console](https://github.com/adorahelen/ai-console-public) I maintain has this architecture — retrieve from a document corpus, generate an answer — and it has exactly the surface this paper describes.

```
Attack scenario:
  inject malicious text into the security knowledge base
  → induce the answer "this IP is benign"
  → real attack traffic gets classified as normal

Defensive application:
  insert EcoSafeRAG's segmentation and diversity check
  into the RAG pipeline as middleware
```

Concretely:

| EcoSafeRAG component | Applied to a security RAG pipeline |
|---|---|
| Sentence segmentation | Split retrieved security documents into sentences and re-rank |
| Core sentence screening | Flag sentences with query similarity above 0.92 |
| Diversity check | Verify context diversity across the reference corpus |
| Bait injection | Use known attack patterns as bait to guide clustering |

The distinction I want to hold onto: a poisoned corpus is not a hallucination. Hallucination is the model failing. This is the model faithfully reporting what the corpus says, and the corpus having been edited by someone else. Those need different controls, and only one of them is fixed by a better model.

---

## Assessment

| | PoisonedRAG | EcoSafeRAG |
|---|---|---|
| **Venue** | USENIX Security (top tier) | arXiv (unpublished) |
| **Contribution** | First systematic definition of the RAG attack surface | First defence that does not lean on model internal knowledge |
| **Practicality** | Attack cost is trivial, immediately executable | Plug and play, no fine-tuning |
| **Rigour** | 8 models × 3 knowledge bases, systematically evaluated | 3 attacks × 3 knowledge bases × 3 models |
| **Weakness** | Demonstrates that defence is hard without offering one | Adaptive attackers not evaluated |

> **A RAG system depends on external knowledge, so the integrity of that knowledge is the security boundary.** PoisonedRAG proved the point with five documents and a 90%-plus success rate. EcoSafeRAG showed you can push that to 0–3% with sentence segmentation and context diversity analysis — without asking the model what it already believes.

Anyone operating RAG in production needs a plan for corpus poisoning. Most do not have one.

---

## Sources

- [PoisonedRAG (USENIX Security 2025)](https://www.usenix.org/conference/usenixsecurity25/presentation/zou-poisonedrag) · [arXiv 2402.07867](https://arxiv.org/abs/2402.07867)
- [EcoSafeRAG (arXiv 2505.13506)](https://arxiv.org/abs/2505.13506)
