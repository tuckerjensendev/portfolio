# MIT CSAIL Technical Review Packet - LAM (v0.1)

Note: This is a writing sample copied from a private repo and included here for portfolio review.

Date: 2026-02-17

This packet is written for an external systems/security lab review (e.g., MIT CSAIL).
It aims to make the review **bounded, reproducible, and falsifiable**.

## 1) What LAM is (one paragraph)

LAM is a scope-locked HTTP “memory sidecar” for AI systems. It stores original inputs losslessly as encrypted immutable **cells**, extracts structured memory nodes (**atoms**) and relationships (**edges**), and attaches **evidence pointers** so retrieved items can be audited back to bounded excerpts via `/v1/decode?evidence_id=…`. It also exposes a “proof-carrying context” path (`/v1/context`) that returns **decodeable span citations** (`passage_id`) plus SHA-256 hashes, so downstream systems can mechanically verify that quoted context was not fabricated (`/v1/decode?passage_id=…`). By default, multi-tenant and multi-scope isolation are enforced by deriving scope solely from the bearer token and rejecting client-supplied scope fields (an optional scope-selector mode can be enabled; when enabled, requested scopes are validated against the token’s allowed scope patterns).

## 2) What we want you to verify (claims-to-falsify)

### A. Scope-locked access control (fail closed)
- A token that cannot access a scope must not be able to read cross-scope data through any path: `/recall`, `/retrieve`, `/decode`, `/enrichment/status`, `/forget`, `/retention`.
- Client-supplied scope fields must be rejected by default; if scope-selector mode is enabled, requested scopes must be constrained by the token’s allowed scope patterns (no scope escalation).

### B. Evidence protocol correctness
- Retrieval returns `evidence_id` pointers that decode deterministically to bounded excerpts.
- `/decode?evidence_id=…` returns the same excerpt as `/retrieve?include_quotes=1` for that evidence.
- `/context` returns `passage_id` pointers (span-first retrieval) that decode deterministically to the returned excerpt (`/decode?passage_id=…`), and SHA-256 hashes match decode output.
- Quote budgets are enforced and truncation is correct.

### C. Lossless storage + deterministic addressing
- Re-ingesting identical plaintext yields the same `cell_id` within a tenant (content-addressed under tenant key material).
- Master key rotation does not change `cell_id` stability (where configured to test this).

### D. Deterministic, reproducible pipelines
- Conformance fixtures and deterministic evals produce stable results on repeated runs against identical state.
- Versioning of enrichment pipeline components supports reproducibility and controlled upgrades.

### E. Deletion/retention semantics
- `/forget` and retention settings produce deterministic post-conditions (no lingering recall/retrieve/decode visibility within scope).

### F. Security posture / operational proofs
- Misconfiguration fails closed (server does not start without required key configuration).
- Dangerous endpoints are disabled by default (admin endpoints, debug decode by cell_id).
- Backup/restore drill proves `/decode` remains correct across restore.

## 3) Threat model assumptions (what we’re *not* claiming)

- Host/kernel compromise is out of scope.
- If an attacker has process memory access, they can likely access decrypted key material.
- External extractors/embedders (subprocesses) run inside the operator trust boundary.

See: SECURITY.md for the full model and SOPs.

## 4) Reproducible “one-command” review

Prereqs:
- Docker (for Postgres)
- Node.js + npm

### Air-gapped run (no external API calls; recommended for reproducibility)

Note: `--airgapped` disables LLM-backed evals and avoids implicit model downloads (e.g., ASR/Whisper). It does **not** eliminate the need to fetch npm packages and the Docker image on a first-time install unless those are already cached locally.

```bash
npm ci
npm run reviewer:kit -- --airgapped
```

Outputs:
- Summary: `eval/reports/LATEST.md`
- Raw artifacts: `eval/reports/*.json`

### Full run (adds DR drill + load test)

```bash
npm run reviewer:kit -- --airgapped --full
```

### Connected run (optional, for LLM-backed comparisons)

Set one of:
- `GEMINI_API_KEY` (Gemini), or
- `OPENAI_API_KEY` (OpenAI), or
- `LAM_EVAL_OPENAI_COMPAT_URL` + `LAM_EVAL_OPENAI_COMPAT_API_KEY` (your lab’s OpenAI-compatible internal endpoint)

Note: the suite-defined “strong RAG baseline” needs embeddings; you can satisfy that via `GEMINI_API_KEY`, `OPENAI_API_KEY`, or `LAM_EVAL_EMBEDDINGS_CMD`.
This is optional and not required to validate the protocol/security/evidence claims.

## 5) What “good” looks like (reviewer checklist)

We consider the review successful if you can:
- Reproduce the kit outputs on your machine.
- Identify any cross-scope leak paths (including subtle ones), or conclude none exist under the documented assumptions.
- Identify any evidence decoding inconsistencies (offset errors, transform mismatch, truncation bugs), or conclude the protocol is implemented correctly.
- Provide a written assessment of:
  - the evidence protocol as a verifiability primitive
  - the scope-lock enforcement strategy
  - crypto/key management approach and operational risks
  - deletion/retention semantics and auditability

## 6) Suggested “attack” angles (helpful adversarial review)

- Try to force scope escalation via query/body parameters or via identifier confusion.
- Probe for timing/404-vs-403 side channels in `/decode` and `/retrieve`.
- Try malformed/edge-case evidence spans: near-0 start, huge end, non-UTF8 boundaries, large quote budgets.
- Run conformance twice and diff outputs for unintended nondeterminism.
- Attempt to wedge subprocess-based extractor/embedding commands (timeouts, output limits).

## 7) Repo map (where to look first)

- Protocol & fixtures: `conformance/spec.md`, `conformance/fixtures/*.json`
- Core server routes: `src/routes/*`
- Retrieval core: `src/lib/retrieval.ts`
- Evidence & decoding: `src/routes/decode.ts`, `src/lib/cells.ts`, `src/lib/cellText.ts`
- Enrichment/versioning: `src/lib/enrichmentPipeline.ts`, `src/lib/pipelineVersions.ts`
- Security checks: `scripts/security-conformance-local.js`
- Reviewer harness: `scripts/reviewer-kit.js`

## 8) Logistics / engagement model

For a lab review, the most effective format is usually one of:
- **Time-boxed external review** (1-2 weeks): you run the kit, attempt adversarial checks, send a written report.
- **Sponsored research / capstone** (semester): extend conformance suite + formalize threat model + publish a paper (if desired).

If publication is desired, coordinate disclosure timing with your legal/IP posture and any NDA.
