---
title: "One Flaw, Many Surfaces: Multimodal and Agentic Attack Surface"
date: 2026-07-10T14:00:00+09:00
tags:
  - ai-security
  - red-teaming
  - llm
  - agents
summary: "Adversarial images, text rendered inside pictures, prompts hidden in web pages, malicious instructions buried in MCP tool descriptions — four different attack surfaces, one root cause. The model mistakes untrusted input for instructions. Adding modalities and autonomy does not create new flaws; it widens the surface on which the same flaw fires, and lets it produce actions instead of text."
---

> 🇰🇷 **[이 글의 한국어판 →](/ko/posts/multimodal-agent-red-teaming/)**

## TL;DR

- **One root cause, four surfaces.** Pixels, external documents, audio carriers and tool metadata are all the same failure: untrusted input treated as instruction rather than data.
- **Safety alignment does not transfer across modalities.** Text-trained alignment does not carry into visual embeddings, and a *single universal adversarial image* can carry many different harmful instructions.
- **Agency is the amplifier.** Tool execution rights × exposure to external content × autonomy means an injection stops producing harmful *text* and starts producing harmful *actions*. On InjecAgent, a ReAct GPT-4 agent was vulnerable in about 24% of cases.
- **MCP moved the trust boundary from code to tool descriptions.** A field the model reads and the user rarely sees is now a supply chain attack surface — with rug-pulls, cross-server shadowing and over-privilege alongside it.

> Extends the [KISA guide](/posts/kisa-ai-redteaming-guide/) and [manual](/posts/kisa-ai-threat-response-manual/), which mostly target text LLMs. Practice layer: [AI red teaming in practice](/posts/ai-red-teaming-practice/). Research and concept level — no exploit code.

---

## 1. Multimodal: alignment does not transfer to vision or audio

- **Visual adversarial examples** — Qi et al., *"Visual Adversarial Examples Jailbreak Aligned LLMs"* ([arXiv 2306.13213](https://arxiv.org/abs/2306.13213), AAAI 2024). The mechanism is the point: images are **continuous and high-dimensional**, so alignment learned on text does not transfer into the visual embedding space. One **universal adversarial image** carries many different harmful instructions.
- **Typographic injection** — Gong et al., *"FigStep"* ([arXiv 2311.05608](https://arxiv.org/abs/2311.05608), AAAI 2025 Oral). Render the forbidden text **as characters inside an image** and text filters never see it; the accompanying text prompt stays benign. The authors report roughly 82.5% average attack success across six vision-language models.
- **The audio lineage** runs from hidden voice commands ([arXiv 1904.05734](https://arxiv.org/abs/1904.05734)) through universal acoustic attacks on speech foundation models ([Muting Whisper, 2405.06134](https://arxiv.org/abs/2405.06134)) to adversarial work on audio LLMs. The shared idea: a carrier that sounds normal to a person while hiding a command only the machine recognises.

> **For a defender**: text input filtering and alignment are not sufficient. Every modality is a separate attack surface, and the image channel is genuinely hard to filter — there is no equivalent of string matching for pixels.

## 2. Indirect prompt injection: the canonical remote control

The canonical reference is Greshake et al., *"Not what you've signed up for"* ([arXiv 2302.12173](https://arxiv.org/abs/2302.12173), ACM AISec '23). It formalised the vector: with no direct interface to the model, an attacker **plants a prompt in content the model will later retrieve** — a web page, a document, an email — and controls the application remotely.

The threats it enumerates: remote control at inference time, **persistent compromise**, data exfiltration, **propagation between documents (worming)**, and denial of service, with proofs of concept against production assistants at the time.

The root cause distinguishes it from direct injection: **untrusted external content is mistaken for instruction rather than data.** That single sentence is the whole of this post; everything else is a delivery mechanism.

## 3. Agents and tool use: autonomy widens the surface

- **Tool-using agents are measurably vulnerable to indirect injection**: on InjecAgent, a ReAct-prompted GPT-4 agent was compromised in about **24%** of cases (§6).
- **Computer-use and browser agents** face a newer surface — prompts planted in on-screen content steering the agent through the **visual channel**, sometimes called visual prompt injection (VPI-Bench, [2506.02456](https://arxiv.org/abs/2506.02456); details unverified by me).
  - Vendor position, self-reported: at least one lab documents that browser content may contain injections, has added detection classifiers, and states plainly that **the risk is not fully resolved**. That is the honest framing, and it is worth reading as a statement about the state of the art rather than about one product.
- **Coding agents**: published work demonstrates **tool-call hijacking** against real coding agents ([arXiv 2509.05755](https://arxiv.org/abs/2509.05755), abstract level).

## 4. MCP security: the tool description as an attack surface

- **Tool poisoning** was first published by Invariant Labs in April 2025: hide malicious instructions in the **description field — metadata the model reads and the user usually does not** — which is indirect prompt injection specialised to a protocol.
- Known MCP risk classes: tool description injection, **rug-pull** (silently replacing a tool definition after installation), cross-server contamination (shadowing), and over-privilege.
- Mitigation exists at the ecosystem level: static analysis of server metadata before installation, pinning trusted tools, allowlisting. Academic treatment in *"Systematic Analysis of MCP Security"* ([arXiv 2508.12538](https://arxiv.org/abs/2508.12538)).

> **For anyone operating agents**: tool description injection and rug-pulls are **the AI edition of a software supply chain attack**. The trust boundary moved from the code package to the tool metadata, and the controls are the ones you already know — watch for definition changes, scan before installation, grant least privilege. What is new is where you point them.

## 5. How the standards map this

**OWASP LLM Top 10 (2025)**:

- **LLM01 Prompt Injection** — first place for a second consecutive edition, and everything in §1–4 maps here: direct, indirect, visual and tool injection.
- **LLM06 Excessive Agency** — the agent-specific entry, with root causes defined as **excessive functionality** (too many tools), **excessive permissions**, and **excessive autonomy** (no supervision). That decomposition is more useful than it looks, because each part has a different fix.
- The 2025 Top 10 has no standalone agentic entry; the OWASP Agentic Security Initiative covers that separately, organising agent threats along agent design, memory, planning, tool use and deployment.

**KISA threat codes**: the manual names **agent hijacking** — hidden malicious prompts in external documents or web content mistaken for legitimate instructions — as a distinct attack type, which is exactly Greshake's indirect injection. (I have not verified the specific A01–A04 label definitions against the source PDF, so I am not reproducing them.)

## 6. Benchmarks: measuring agent injection

| Benchmark | Source | What it measures |
|---|---|---|
| **InjecAgent** | [arXiv 2403.02691](https://arxiv.org/abs/2403.02691) | Indirect injection vulnerability in tool-integrated agents. 1,054 cases, 17 user tools, 62 attacker tools, single turn. Two axes: direct harm and data exfiltration. ReAct GPT-4 vulnerable in ~24% |
| **AgentDojo** | [arXiv 2406.13352](https://arxiv.org/abs/2406.13352), NeurIPS 2024 | Dynamic evaluation of agents executing tools **over untrusted data**. 70 tools, 97 tasks, 27 injection goals. Measures **both** (a) utility — did it still complete the original task — and (b) security — was it compromised — end to end across multiple steps |

**The difference matters.** InjecAgent isolates a single step; AgentDojo simulates realistic execution including tool selection and task completion. For evaluating a defence, the AgentDojo style is the practical one because it measures **utility alongside security** — it forces the balance that a filter-everything approach fails: blocking the attack while killing the feature is not a defence, it is an outage.

---

## The single principle behind the expanded surface

1. **More modalities and more autonomy, one underlying flaw**: untrusted input (pixels, external documents, tool descriptions) read as instruction. §1, §2 and §4 are the same disease with different symptoms.
2. **Agency amplifies it.** Tool permissions × external content exposure × autonomy multiply, so an injection stops being "harmful text" and becomes **an action** — data exfiltrated, tools misused. That is what OWASP LLM06 is pointing at.
3. **Supply chain in the MCP era**: the trust boundary moved from code to tool metadata. Pre-install scanning, pinning and least privilege are mandatory — the AI edition of conventional supply chain defence.
4. **Measurement has matured.** InjecAgent and AgentDojo make agent injection vulnerability quantifiable, and a defence should be evaluated on **utility and security together**.
5. **Where it meets the SOC**: agent telemetry is the detection target — unexpected tool call sequences, privilege-escalating behaviour immediately after referencing external content. MCP tool definition changes and rug-pulls belong in supply chain integrity monitoring.

> **The connection worth noticing**: this expanded surface is a deepening of axis ① (AI as the target), and autonomous agents are also the execution substrate for axis ② ([AI as the weapon](/posts/ai-augmented-red-teaming/)). **The agent being attacked and the agent doing the attacking are two faces of the same technology** — which means hardening one teaches you about the other.
