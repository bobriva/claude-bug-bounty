---
name: web-llm-attacks
description: Identify and exploit vulnerabilities in LLM-integrated web applications — direct and indirect prompt injection, excessive agency (tool/API abuse beyond intended scope), system prompt leakage, training-data/sensitive-information disclosure, and insecure output handling chaining into XSS or data exfiltration — for authorized bug bounty and pentest work, mapped to the OWASP Top 10 for LLM Applications (2025). Covers recon (mapping direct vs. indirect inputs, enumerating available tools/APIs the model can invoke), real-world-pattern techniques (markdown/HTML image-rendering exfiltration, stored cross-user indirect injection via documents/tickets/knowledge-base content, classifier-bypass phrasing), and a validation gate that filters out "the bot said something odd" from confirmed, consequential findings — plus a report template. Use for any AI chatbot, AI search, AI agent/tool-calling workflow, or AI content-generation feature, or writing up an LLM-application finding for HackerOne, Bugcrowd, or YesWeHack.
---

# Web LLM Attack Testing

Same loop as the other skills in this set: **map → test → validate → report**. This is the newest category in this skill set, and also the one with the highest false-positive rate by far — getting a chatbot to say something off-script is trivial and proves almost nothing on its own. Documented research shows even well-defended models can be talked past their guardrails on a large fraction of repeated attempts; that's exactly why "I jailbroke it" isn't a finding here unless something with real consequence happened as a result. The discipline this skill enforces is distinguishing *novelty* (the model behaved unexpectedly) from *impact* (something a real attacker couldn't otherwise reach was exposed or executed).

## Core principle: the LLM is a confused deputy, not a target in itself

Nearly every serious finding in this category follows the same shape: the application gave the model access to something sensitive (data, an API, another user's content) and trusted the model's judgment as a security boundary. The model isn't the vulnerability — the trust placed in it is. When testing, always ask: *what does this LLM have access to that I don't have direct access to, and can I get it to use that access on my behalf or against another user?* That question is the throughline across prompt injection, excessive agency, and indirect injection alike.

## Before you start: map the attack surface

```
1. Direct inputs: chat prompts, search queries, feedback/review fields
   that feed an LLM, any field whose content the model will read.
2. Indirect inputs: anything the model might read *without* the
   current user typing it — uploaded documents, emails, support
   tickets, knowledge-base articles, web pages it fetches/summarizes,
   other users' comments or reviews, file attachments.
3. Tools/APIs the model can invoke: ask it directly ("what tools do
   you have access to?", "what APIs can you call?") — system prompts
   and tool descriptions are themselves part of the attack surface,
   not just a defense. Cross-reference what it admits to against
   actual network traffic (proxy the chat session) to find tools it
   doesn't volunteer.
4. Output destinations: where does the model's output get rendered —
   raw HTML, Markdown, a template engine, an email, a downstream
   automation/workflow? Each rendering context is a potential
   insecure-output-handling sink.
```

## The methodology

| Phase | Goal | Reference |
|---|---|---|
| 1. Recon & surface mapping | Direct/indirect inputs, tool/API enumeration, OWASP LLM Top 10 orientation | `references/owasp-llm-top10-and-recon.md` |
| 2. Direct prompt injection | Instruction override, system-prompt leakage, training-data extraction | `references/prompt-injection-and-extraction.md` |
| 3. Indirect injection & exfiltration | Planting payloads in documents/tickets/KB content, markdown/image-based exfiltration | `references/indirect-injection-and-exfiltration.md` |
| 4. Excessive agency & output handling | Tool/API abuse beyond intended scope, output rendered unsafely (XSS, template injection) | `references/excessive-agency-and-output-handling.md` |
| 5. Validate | Run every candidate through the gate below | This file, "Validation gate" |
| 6. Report | Real consequence demonstrated, reproducible, fast to triage | This file + `references/payloads-and-reporting.md` |

## Validation gate — run this before calling anything a finding

Every "no" means: keep digging, downgrade to "suspected," or drop it.

1. **Did something with real consequence happen — actual data exposed, an actual backend action executed — not just unusual text output?** "It said it would delete the user" is not the same as the user actually being deleted; confirm the latter independently (check the record is gone), not just that the model claimed success.
2. **For system prompt/training-data leakage: is what leaked actually sensitive** (credentials, internal tool definitions and their parameters, another user's real data, proprietary business logic) **rather than generic boilerplate** ("You are a helpful assistant for Acme Corp")?
3. **For excessive agency: did you confirm the tool call actually executed against the real backend**, with an effect you independently verified — not just that the model described what it could theoretically do?
4. **For indirect injection: is there cross-user impact** — a payload planted by one identity actually affecting a *different* victim's session or a system-level process — rather than only your own session following instructions from your own uploaded content?
5. **For markdown/image/link-based exfiltration: did the rendering actually fire an outbound request you captured on infrastructure you control**, ideally carrying real (or test) sensitive data in the URL — not just that the model agreed to output a link?
6. **Is this actually unintended?** Some "agentic" features genuinely are designed to let the model take the action you triggered (a documented "developer mode," an admin-only debug tool intentionally exposed to an admin chat session). Confirm the access level you used wasn't supposed to have that capability.
7. **Is it a duplicate, or just "the model can be jailbroken" with nothing attached?** General susceptibility to instruction override is extremely well-documented and not, by itself, novel or scoreable — the finding is what you got it to expose or do, not that you got past its guardrails at all.

### Patterns that usually die at this gate (don't submit these alone)

- Getting the model to say something off-topic, rude, or against content policy, with no data exposure or unauthorized action attached
- "System prompt leaked" where the leaked content is generic, non-sensitive boilerplate
- A tool/API abuse claim based only on the model's own narrated description of what it could do, with no confirmed real backend call
- Indirect injection demonstrated only against your own account/session, with no plausible or demonstrated path to a different victim
- A markdown/HTML image payload the model agreed to output, with no actual captured outbound request on infrastructure you control
- "Training data extraction" that produced plausible-sounding but unverifiable text, with no way to confirm it's real leaked data rather than a confident-sounding hallucination

### Patterns that become valid once chained

| Weak alone | Chain it with | Becomes |
|---|---|---|
| System prompt leak (often scored Low/Informational on its own) | The leaked prompt contains an internal API key, tool credential, or undocumented admin tool definition | Credential/internal-architecture exposure — meaningfully higher severity |
| Tool/API enumeration (just a list of names) | Actually invoking a destructive or data-disclosing tool end-to-end, with confirmed backend effect | Confirmed excessive agency, not just a capability inventory |
| A payload that follows instructions in your own uploaded content | The same content type (ticket, KB article, shared document) later processed in a *different* user's or an *admin's* session | Stored, cross-user indirect prompt injection — often the highest-severity variant in this category, since the attacker needed no special access at all |
| Model outputs raw, unescaped HTML/Markdown | That output is later rendered in a context other users or an admin view (shared chat history, an internal dashboard, a generated report) | Stored XSS via insecure output handling |
| Model agrees to include a markdown image/link pointing at attacker infrastructure | The resulting request, when actually rendered, carries real session/user data in its URL | Confirmed data exfiltration — this exact pattern (markdown image auto-render leaking data to an attacker-controlled URL) has been disclosed against multiple major chat products and is one of the most consistently rewarded findings in this category |

## Writing it up

**Title formula:** `[OWASP LLM category] in [exact feature] allows [actor] to [impact]`
Good: `Excessive Agency in support chatbot allows any user to execute arbitrary SQL via the debug_sql tool, including deleting other users' accounts`
Bad: `Chatbot can be jailbroken`

Full payload patterns, severity guidance, and report template: `references/payloads-and-reporting.md`.

## Reference files in this skill

- `references/owasp-llm-top10-and-recon.md` — OWASP Top 10 for LLM Applications (2025) mapped to tests, attack-surface recon methodology
- `references/prompt-injection-and-extraction.md` — direct prompt injection, system-prompt leakage, training-data extraction techniques
- `references/indirect-injection-and-exfiltration.md` — indirect injection via documents/tickets/knowledge bases, markdown/image-based data exfiltration, classifier-bypass phrasing
- `references/excessive-agency-and-output-handling.md` — tool/API enumeration and abuse, insecure output handling chaining into XSS/template injection
- `references/payloads-and-reporting.md` — payload patterns, severity guidance, full report template
