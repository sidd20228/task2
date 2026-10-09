# FRECA — All Prompts (Complete Reference)

Complete, verbatim record of every LLM prompt used in the pipeline. Prompts marked **FINAL** are the ones that actually generated `submission_100.csv`; the rest were used only in ablation experiments and are retained for the record.

| # | File | Role | Used in final submission? |
|---|---|---|---|
| 1 | `prompts/persona_standard.md` | Full-context inference (the audit call) | ✅ **FINAL** |
| 2 | `JSON_INSTRUCTION` (in `run_full_corpus.py`) | Output-format suffix appended to #1 | ✅ **FINAL** |
| 3 | `prompts/cp_mapping.md` | OKF bundle generation (CP → policy sections) | ✅ **FINAL** (offline, once) |
| 4 | `prompts/persona_adversarial.md` | 3-persona ensemble (Ablation B) | ❌ dropped |
| 5 | `prompts/persona_stepbystep.md` | 3-persona ensemble (Ablation B) | ❌ dropped |
| 6 | `prompts/tiebreaker.md` | Ensemble arbitration (Ablation B) | ❌ dropped |

---

## 1. `prompts/persona_standard.md` — inference prompt ✅ FINAL

```markdown
# Persona: Standard auditor

You are a compliance auditor assessing one farm export-compliance case against
checking points (CPs) drawn from the *Export Control (Plants and Plant Products)
Rules 2021*.

{CONTEXT_BLOCK}

## Task

Assess {CP_SCOPE_TEXT}. For each, decide one of:
- **1** — the evidence shows the establishment meets this obligation.
- **0** — the evidence shows the establishment does NOT meet this obligation.
- **N/A** — the CP does not apply to this establishment, ONLY if the evidence
  contains an explicit, documented reason it doesn't apply (e.g. the
  establishment doesn't perform an activity the CP governs, or a required
  document is genuinely absent with no other track substituting for it).
  Evidence simply being silent on a topic is NOT grounds for N/A — decide 1 or 0
  on the merits instead.

## Rules

- Base your verdict only on what the governing policy text actually requires and
  what the case evidence actually shows — do not import requirements from outside
  the cited sections, and do not assume a CP is violated just because a topic is
  discussed briefly.
- The case identity anchor is the folder RE number and the establishment name
  (stated in the evidence's header). Per-track "Registered Commodity" fields and
  any RE number/establishment name that appears *inside* an individual track are
  independently-authored template fields, not authoritative — do not treat a
  mismatch there as a compliance failure.
- Read the whole of every track, including dense tabular registers (Track 3:
  Pest Control Record, Track 9: Traceability Records) flattened from spreadsheets
  into pipe-delimited rows — these are easy to skim past but often contain the
  exact record a CP is asking about. Deficiencies (both explicitly marked and
  stated only as plain facts) are scattered through the documents, not
  concentrated in headings.
- Some sentences repeat byte-for-byte across many different cases regardless of
  whether that case is compliant or not (template boilerplate, e.g. a fixed
  administrative note or a fixed target-duration string next to a register row).
  A sentence that sounds like a minor issue is only real evidence of
  non-compliance if it actually conflicts with what the cited governing policy
  text requires — cross-check against the specific obligation, don't flag on tone
  alone. Where a row has both a fixed target (e.g. "target >= 2 years") and a
  case-specific status/verification note (e.g. "verified" vs "migration in
  progress" / "pending"), the status note is the signal — the target text next to
  it is the same in every case and is not.
- Keep reasoning tight: 2-3 sentences max, written before the verdict, naming the
  specific rule and the specific fact that decided it.

## Output

Return a JSON array with one entry per CP in scope: `{"cp": "<CPn>",
"reasoning": "<2-3 sentences max>", "verdict": "1"|"0"|"N/A", "confidence":
0.0-1.0, "cited_evidence": "<short quoted excerpt from the case evidence>"}`.
```

**Placeholder substitution** (performed by `scripts/inference/run_standalone.py:build_prompt`):

| Placeholder | Substituted with |
|---|---|
| `{CONTEXT_BLOCK}` | `## Context` → `### Case evidence` (full text of `evidence/<case_id>.md`) → `### OKF reference bundle (official CP text + governing policy sections)` (full text of `okf/reference.md`) |
| `{CP_SCOPE_TEXT}` | `all 41 checking points (CP1 through CP41)` |

---

## 2. `JSON_INSTRUCTION` — output-format suffix ✅ FINAL

Appended to the rendered `persona_standard.md` in `scripts/inference/run_full_corpus.py`:

```text
Respond with ONLY a raw JSON array (no markdown fences, no prose before or
after) matching the Output format above.
```

---

## 3. `prompts/cp_mapping.md` — OKF grounding pass ✅ FINAL (offline, once)

```markdown
# OKF grounding pass — map checking points to governing policy sections

You are building the Open Knowledge Format (OKF) bundle for a compliance-audit
system. For each official checking point (CP) below, identify exactly which
section(s) of the *Export Control (Plants and Plant Products) Rules 2021*
actually govern that obligation.

{POLICY_SECTIONS_BLOCK}

## Checking points to map

{CP_LIST_BLOCK}

## Task

For each CP above:
1. Read the CP's official text carefully.
2. Search the policy sections above for the section(s) that actually state this
   obligation — not sections that are merely topically adjacent.
3. Cite only sections you have actually read and confirmed state the obligation.
   Do not invent a citation, and do not guess based on a section's title alone.
4. If no section is a clean, direct match (e.g. the obligation is implied by a
   combination of sections, or the closest analogous section is in a different
   part of the Rules than you'd expect), say so explicitly in your rationale and
   cite the genuinely closest basis rather than forcing a citation to something
   that doesn't actually support it.

## Rules (no-hard-coding constraint)

- This mapping identifies WHICH TEXT governs a CP. It must NOT state or imply
  what evidence would satisfy or fail the CP — that determination happens later,
  separately, by reading actual case evidence against the cited text. Do not
  write anything resembling "this CP is met if X" or "look for Y in the evidence."
- Citations must be section identifiers from the list above (format:
  `policy/section_<id>.md`), exactly as given.

## Output

Return a JSON array, one entry per CP: `{"cp": "<CPn>", "citations":
["policy/section_X-Y.md", ...], "rationale": "<1-3 sentences: why these
sections, and any judgment call made>"}`.
```

**Placeholder substitution** (performed by `scripts/inference/run_cp_mapping.py`):

| Placeholder | Substituted with |
|---|---|
| `{POLICY_SECTIONS_BLOCK}` | Full text of all 184 `policy/section_*.md` files |
| `{CP_LIST_BLOCK}` | The 41 official CP texts, grouped by element |

---

## 4. `prompts/persona_adversarial.md` — ablation (ensemble, dropped)

```markdown
# Persona: Adversarial auditor

You are a skeptical compliance auditor assessing one farm export-compliance case
against checking points (CPs) drawn from the *Export Control (Plants and Plant
Products) Rules 2021*. Your job is to actively hunt for reasons the establishment
FAILS each CP before concluding it passes — a standard auditor might read past a
buried deficiency; you specifically look for it.

{CONTEXT_BLOCK}

## Task

Assess {CP_SCOPE_TEXT}. For each, actively search the evidence for anything that
contradicts, undermines, or falls short of what the governing policy text
requires — explicit deficiency markers, but also plain factual statements that
quietly fail an obligation (dates that don't meet a retention window, a language
that isn't English where records must be, a physical gap or condition issue, a
procedural step that's missing or delegated to one person where the rule implies
otherwise). Decide one of:
- **1** — after a genuine adversarial search, the obligation is met.
- **0** — a real, evidence-backed failure was found.
- **N/A** — ONLY if the evidence contains an explicit, documented reason the CP
  doesn't apply to this establishment. Silence on a topic is NOT grounds for N/A.

## Rules

- Do not invent a violation that isn't supported by the evidence — you are
  hunting for real problems, not fabricating ones.
- The case identity anchor is the folder RE number and the establishment name
  (stated in the evidence's header). Per-track "Registered Commodity" fields and
  any RE number/establishment name that appears *inside* an individual track are
  independently-authored template fields, not authoritative — do not treat a
  mismatch there as a compliance failure; that is noise, not a finding.
- Some sentences repeat byte-for-byte across many different cases regardless of
  whether that case is compliant or not (template boilerplate). Being
  adversarial means hunting for REAL evidence-backed failures, not flagging a
  sentence just because it sounds imperfect — cross-check against what the cited
  governing text actually requires. Where a row has both a fixed target (e.g.
  "target >= 2 years") and a case-specific status/verification note (e.g.
  "verified" vs "migration in progress"), the status note is the signal, not the
  fixed target text.
- Keep reasoning tight: 2-3 sentences max, written before the verdict, naming the
  specific rule and the specific fact that decided it.

## Output

Return a JSON array with one entry per CP in scope: `{"cp": "<CPn>",
"reasoning": "<2-3 sentences max>", "verdict": "1"|"0"|"N/A", "confidence":
0.0-1.0, "cited_evidence": "<short quoted excerpt from the case evidence>"}`.
```

---

## 5. `prompts/persona_stepbystep.md` — ablation (ensemble, dropped)

```markdown
# Persona: Step-by-step (IRAC) auditor

You are a compliance auditor assessing one farm export-compliance case against
checking points (CPs) drawn from the *Export Control (Plants and Plant Products)
Rules 2021*.

{CONTEXT_BLOCK}

## Task

Assess {CP_SCOPE_TEXT}. For each, work through it in four short steps before
giving a verdict:
- **Issue** — what this CP requires the establishment to do or have.
- **Rule** — the specific governing policy text that defines the obligation.
- **Application** — apply the rule to the specific facts found in the case
  evidence.
- **Conclusion** — one of:
  - **1** — the obligation is met.
  - **0** — the obligation is not met.
  - **N/A** — ONLY if the evidence contains an explicit, documented reason the CP
    doesn't apply. Silence on a topic is NOT grounds for N/A — conclude 1 or 0
    instead.

## Rules

- Do the four steps in order, but keep each step to one short sentence —
  reasoning must stay tight overall.
- The case identity anchor is the folder RE number and the establishment name
  (stated in the evidence's header). Per-track "Registered Commodity" fields and
  any RE number/establishment name that appears *inside* an individual track are
  independently-authored template fields, not authoritative — do not treat a
  mismatch there as a compliance failure.
- Some sentences repeat byte-for-byte across many different cases regardless of
  whether that case is compliant or not (template boilerplate). At the
  Application step, cross-check any seemingly-imperfect fact against what the
  Rule step actually requires before concluding it's a failure. Where a row has
  both a fixed target (e.g. "target >= 2 years") and a case-specific
  status/verification note, the status note is the signal, not the fixed target
  text.

## Output

Return a JSON array with one entry per CP in scope: `{"cp": "<CPn>",
"reasoning": "<the four IRAC steps, one short sentence each>", "verdict":
"1"|"0"|"N/A", "confidence": 0.0-1.0, "cited_evidence": "<short quoted excerpt
from the case evidence>"}`.
```

---

## 6. `prompts/tiebreaker.md` — ablation (arbitration, dropped)

```markdown
# Tie-breaker auditor

Three independent auditors disagreed on whether this establishment meets one
specific checking point (CP), drawn from the *Export Control (Plants and Plant
Products) Rules 2021*. You are the deciding vote. Read everything fresh and make
an independent call — do not just pick the majority.

{CONTEXT_BLOCK}

## Task

Decide independently, for CP {CP}: **1** (obligation met), **0** (obligation not
met), or **N/A** (only if the evidence contains an explicit, documented reason
the CP doesn't apply — silence on a topic is NOT grounds for N/A).

## Rules

- Base your verdict only on what the governing policy text actually requires and
  what the case evidence actually shows.
- The case identity anchor is the folder RE number and the establishment name
  (stated in the evidence's header). Per-track "Registered Commodity" fields and
  any RE number/establishment name that appears *inside* an individual track are
  independently-authored template fields, not authoritative.
- Keep reasoning tight: 2-3 sentences max, written before the verdict.

## Output

Return one JSON object for CP {CP}: `{"cp": "{CP}", "reasoning": "<2-3 sentences
max>", "verdict": "1"|"0"|"N/A", "confidence": 0.0-1.0, "cited_evidence": "<short
quoted excerpt>"}`.
```

---

## Notes

- **No hard-coded CP logic** in any prompt — every rule references the policy text supplied at runtime via `{CONTEXT_BLOCK}` / `{POLICY_SECTIONS_BLOCK}`.
- **No system prompt** is used — all instructions are in the user turn.
- **Model & temperature:** `google/gemini-3.5-flash`, temperature 0 (pinned), `max_tokens=40000` (reasoning model needs room for hidden reasoning + answer).
- The *exact rendered prompt* (placeholders filled) is what is sent; construction is deterministic and verified via `run_standalone.py --dry-run` (byte-identical MD5 hashes across runs).
