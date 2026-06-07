# LLM Security Testing Methodology

## Phase 1 - Recon

Identify:

- Chat interfaces
- AI assistants
- AI search
- AI workflows
- Agentic systems

Determine:

- Model provider
- Tool usage
- Plugin support
- Function calling

---

## Phase 2 - Attack Surface Mapping

Identify:

Direct Inputs

- Chat prompts
- Search prompts
- Feedback prompts

Indirect Inputs

- Documents
- Emails
- Comments
- Knowledge bases
- Web pages

---

## Phase 3 - Capability Mapping

Ask:

- What tools are available?
- What APIs can be accessed?
- What actions can be performed?

Map:

- User APIs
- Admin APIs
- Search APIs
- File APIs
- Internal services

---

## Phase 4 - Security Testing

Test:

- Prompt injection
- Indirect prompt injection
- Tool abuse
- Data leakage
- Output handling
- Cross-user impact

---

## Phase 5 - Impact Validation

Verify:

- Sensitive data access
- Unauthorized actions
- API misuse
- User compromise