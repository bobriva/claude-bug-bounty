# OWASP Top 10 for LLM Applications (2025) & Recon

## Recon methodology

```
1. Identify the LLM surface: chatbot, AI search, content generator,
   AI-powered scanner/assistant, agentic workflow.
2. Identify direct inputs: anything the current user types that the
   model reads (chat messages, search queries, form fields that get
   summarized/processed by the model).
3. Identify indirect inputs: anything the model might read that
   the *current* user didn't type — uploaded documents, fetched web
   pages, other users' content (reviews, tickets, comments), email,
   knowledge-base articles, RAG-retrieved chunks.
4. Map tools/APIs: ask directly — "What tools do you have access
   to?", "What APIs can you call?", "List your available functions."
   Treat the model's own description as a starting inventory, then
   verify against actual proxied network traffic during a chat
   session, since a model may under-report or be instructed not to
   disclose certain tools.
5. Map output destinations: where does a response get rendered —
   directly as HTML, as Markdown converted to HTML, into an email
   template, into a downstream workflow/automation step?
```

## OWASP Top 10 for LLM Applications (2025) — quick map

| # | Category | One-line test | Reference |
|---|---|---|---|
| LLM01 | Prompt Injection | Override instructions via direct input or content the model reads | `prompt-injection-and-extraction.md`, `indirect-injection-and-exfiltration.md` |
| LLM02 | Sensitive Information Disclosure | Extract PII, credentials, or proprietary data the model has access to | `prompt-injection-and-extraction.md` |
| LLM03 | Supply Chain | Compromised/malicious third-party model, plugin, or training data source | Largely out of scope for black-box web testing — note if a plugin/tool integration looks unvetted, but this is primarily a build-time/governance risk |
| LLM04 | Data and Model Poisoning | Injecting malicious content into training/fine-tuning data or a RAG vector store | Relevant mainly if the app lets users contribute content that gets ingested into a knowledge base/embedding store — test whether your contributed content can skew retrieval or later outputs for other users |
| LLM05 | Improper Output Handling | Model output rendered unsafely downstream (HTML/Markdown/template/workflow) | `excessive-agency-and-output-handling.md` |
| LLM06 | Excessive Agency | Model has more tool access/permissions/autonomy than its task requires | `excessive-agency-and-output-handling.md` |
| LLM07 | System Prompt Leakage | System prompt contains secrets, internal logic, or credentials and can be extracted | `prompt-injection-and-extraction.md` |
| LLM08 | Vector and Embedding Weaknesses | RAG vector store poisoning or cross-tenant access control gaps | Test multi-tenant RAG systems for retrieval crossing tenant boundaries — conceptually a BOLA-style test (see the `api-security` skill) applied to embedding retrieval rather than a REST object |
| LLM09 | Misinformation | Model produces confident, wrong output (hallucinated facts/citations) | Generally a product-quality issue, not a security finding, unless the hallucination is *attacker-steerable toward a specific harmful outcome* (e.g., consistently recommending a malicious package name) — test for steerability, not just inaccuracy |
| LLM10 | Unbounded Consumption | No limits on request volume/complexity, enabling cost/resource abuse | Same testing approach as the `business-logic` skill's resource-consumption angle and the `api-security` skill's "Unrestricted Resource Consumption" — applied to expensive model calls instead of a REST endpoint |

Prompt Injection (LLM01) has held the #1 spot since this list began — start there, since it's both the highest-frequency root cause and the gateway into most of the other categories (a successful injection is usually what makes LLM02/LLM05/LLM06 actually exploitable rather than theoretical).

## Why this category resists traditional web testing

Standard web vulnerability scanning doesn't detect prompt injection, excessive agency, or embedding-store weaknesses — there's no signature or fixed syntax to match the way there is for `' OR 1=1`. Every test in this skill is necessarily semantic and manual: you're testing whether *meaning*, not *syntax*, can be hijacked. Budget testing time accordingly — this category requires more iterative, judgment-driven probing per finding than syntax-driven classes like SQL injection.
