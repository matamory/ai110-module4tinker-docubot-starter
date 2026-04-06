# DocuBot Model Card

This model card is a short reflection on your DocuBot system. Fill it out after you have implemented retrieval and experimented with all three modes:

1. Naive LLM over full docs  
2. Retrieval only  
3. RAG (retrieval plus LLM)

Use clear, honest descriptions. It is fine if your system is imperfect.

---

## 1. System Overview

**What is DocuBot trying to do?**  
Describe the overall goal in 2 to 3 sentences.

> DocuBot is a documentation QA assistant for a small project docs folder. It supports three modes: naive LLM answering, retrieval-only snippet return, and retrieval-augmented generation (RAG). The goal is to improve factual grounding by retrieving relevant doc sections and refusing when evidence is weak.

**What inputs does DocuBot take?**  
For example: user question, docs in folder, environment variables.

> User question text, markdown/txt files in docs/, optional GEMINI_API_KEY for LLM modes, and mode choice (naive, retrieval-only, or RAG).

**What outputs does DocuBot produce?**

> Naive mode outputs free-form LLM text. Retrieval-only outputs top-ranked snippets with source filenames. RAG outputs a synthesized answer constrained by retrieved snippets, or a refusal string when evidence is insufficient.

---

## 2. Retrieval Design

**How does your retrieval system work?**  
Describe your choices for indexing and scoring.

- How do you turn documents into an index?
- How do you score relevance for a query?
- How do you choose top snippets?

> Documents are split into section-sized chunks by blank lines. An inverted index maps token -> section IDs. Query tokens are normalized and filtered through a simple stopword list; candidate sections come from index hits, then each section gets a count-based relevance score from keyword overlap frequency. Top-k sections are sorted by score desc with deterministic tie-breaking.

**What tradeoffs did you make?**  
For example: speed vs precision, simplicity vs accuracy.

> I favored simplicity and readability over advanced ranking quality. This is fast and easy to debug, but can still surface noisy snippets when common words overlap. The guardrail improves safety but can be over-conservative for wording mismatches (for example default vs defaults).

---

## 3. Use of the LLM (Gemini)

**When does DocuBot call the LLM and when does it not?**  
Briefly describe how each mode behaves.

- Naive LLM mode: always calls Gemini and answers directly from prompt text; currently it does not actually pass corpus text into the prompt, so grounding is weak.
- Retrieval only mode: never calls Gemini; returns retrieved snippets or refusal.
- RAG mode: retrieves snippets first, runs an evidence guardrail, then calls Gemini only when evidence passes; otherwise refuses.

**What instructions do you give the LLM to keep it grounded?**  
Summarize the rules from your prompt. For example: only use snippets, say "I do not know" when needed, cite files.

> In RAG mode, the prompt says to use only provided snippets, avoid inventing endpoints/functions/config values, refuse with exact text when snippets are insufficient, and mention which files were used.

---

## 4. Experiments and Comparisons

Run the **same set of queries** in all three modes. Fill in the table with short notes.

You can reuse or adapt the queries from `dataset.py`.

| Query | Naive LLM: helpful or harmful? | Retrieval only: helpful or harmful? | RAG: helpful or harmful? | Notes |
|------|---------------------------------|--------------------------------------|---------------------------|-------|
| Is GET /api/users only accessible to admins? | Mostly harmful: generic advisory answer and no direct project-grounded conclusion. | Helpful but raw: includes the correct admin snippet plus header-only noise. | Helpful: concise and grounded, answered yes and cited API_REFERENCE.md. | Best RAG case in this run. |
| Which endpoint returns all users? | Harmful: confidently gives generic GET /users pattern, not project-specific /api/users. | Mixed: contains the right sentence but also unrelated lines (projects, db helper). | Harmful in this run: refused despite available evidence. | Shows RAG can under-answer if model is too conservative with sparse snippets. |
| What is the capital of France? | Harmful for this system goal: answers Paris from world knowledge, not docs. | Helpful: correctly refused. | Helpful: correctly refused. | Guardrail behavior worked as intended. |
| What is TOKEN_LIFETIME_SECONDS default? | Harmful/misleading: says no universal default and hedges, while docs contain 3600. | Harmful in this run: refused though evidence exists. | Harmful in this run: same refusal. | Guardrail false negative from strict token overlap and morphology mismatch (default vs defaults). |

**What patterns did you notice?**  

- When does naive LLM look impressive but untrustworthy?  
- When is retrieval only clearly better?  
- When is RAG clearly better than both?

> Naive mode often sounds fluent and authoritative even when it is not grounded in this repo. Retrieval-only is transparent and auditable but hard to interpret because users must synthesize snippets themselves. RAG can be best when retrieval contains a crisp evidence line (admin-only users endpoint), but it still fails when retrieved evidence is partial or phrasing-sensitive.

---

## 5. Failure Cases and Guardrails

**Describe at least two concrete failure cases you observed.**  
For each one, say:

- What was the question?  
- What did the system do?  
- What should have happened instead?

> Failure case 1: Query "Which endpoint returns all users?". Naive answered generic GET /users and long best-practice boilerplate. Expected behavior: project-grounded endpoint (/api/users) or refusal if uncertain.

> Failure case 2: Query "What is TOKEN_LIFETIME_SECONDS default?". Retrieval-only and RAG both refused despite docs stating default 3600. Expected behavior: return the AUTH.md snippet and answer 3600 seconds.

**When should DocuBot say “I do not know based on the docs I have”?**  
Give at least two specific situations.

> It should refuse when no snippets are retrieved for informative query tokens, and when retrieved snippets do not contain enough keyword overlap to support a direct answer. It should also refuse when snippets are only section headers/labels with no factual statement.

**What guardrails did you implement?**  
Examples: refusal rules, thresholds, limits on snippets, safe defaults.

> Implemented query keyword filtering (stopwords + short-token filtering), section-level retrieval instead of whole-doc retrieval, and a meaningful-evidence gate requiring at least one snippet with sufficient distinct keyword overlap before answering. Both retrieval-only and RAG use the same refusal path.

---

## 6. Limitations and Future Improvements

**Current limitations**  
List at least three limitations of your DocuBot system.

1. Naive mode prompt currently ignores the full corpus text, so answers are weakly grounded.
2. Keyword retrieval is lexical only (no stemming/synonyms), causing misses like default vs defaults.
3. Ranking still admits noisy snippets due common terms and simplistic scoring.

**Future improvements**  
List two or three changes that would most improve reliability or usefulness.

1. Pass corpus context into naive mode (or deprecate naive mode) to avoid false confidence.
2. Add light normalization (stemming/lemmatization) and/or fuzzy token matching.
3. Improve snippet quality with better chunking and section-type weighting (prefer factual lines over headings).

---

## 7. Responsible Use

**Where could this system cause real world harm if used carelessly?**  
Think about wrong answers, missing information, or over trusting the LLM.

> In production support or security workflows, a confident but ungrounded answer could cause unsafe API use, missed auth constraints, or incorrect operational actions. Over-trusting naive mode is especially risky because fluent output can hide weak evidence.

**What instructions would you give real developers who want to use DocuBot safely?**  
Write 2 to 4 short bullet points.

- Treat retrieval snippets as primary evidence; do not trust fluent prose without source lines.
- Prefer RAG over naive mode, and treat refusals as a safety signal, not a bug.
- Add evaluation queries for critical endpoints/auth behavior and track regressions after retrieval changes.
- Review low-confidence or refusal-heavy queries manually and improve docs/chunking before deployment.

---
