# Web LLM Attacks Knowledge

## Core Mindset

Treat the LLM as a privileged proxy.

The attacker cannot directly access:

- Internal APIs
- Internal data
- System prompts

But the LLM might.

---

## Main Categories

1. Prompt Injection

2. Indirect Prompt Injection

3. Excessive Agency

4. Data Leakage

5. Insecure Output Handling

6. Training Data Poisoning

7. Tool Abuse

---

## Similarity to SSRF

LLM attacks are often conceptually similar to SSRF.

In SSRF:

Attacker -> Server -> Internal Resource

In LLM attacks:

Attacker -> LLM -> Internal Resource

The LLM acts as a trusted intermediary.