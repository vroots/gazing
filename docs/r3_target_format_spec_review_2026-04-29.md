# R3 Target Format Spec — Reviewer Critique

**Reviewer date:** 2026-04-29
**Spec under review:** `docs/r3_target_format_spec.md` (310 lines)
**Source-of-truth artifacts consulted:**
- `shared/v1_classifier_snapshot_2026-04-29.txt` — actual V1 production code
- `~/hs-code-classifiers/CLAUDE.md` — operating principles
- `~/hs-code-classifiers/ARCHITECTURE.md` — system context (read-only, freeze respected)

**Note:** The reviewer pattern document (`WRITER_REVIEWER.md`) referenced in the request was not found in any expected location. Review proceeds under the principle stated in the request: *assume nothing, verify everything, reject if uncertain.*

---

## TL;DR

**Verdict: NEEDS REWRITE.** The spec describes a *plausible* training format but it is NOT the format V1 production actually sends to the model. Training to this spec would produce a model whose performance on current production traffic is unpredictable — possibly worse than V2.5, not better.

**Findings: 4 critical, 4 high, 5 medium, 3 low.**

---

## Q1. Format match correctness — line-by-line diff

The spec's Section 3.2 claims to define the format "production sends to the model." It does not match the V1 production format from the snapshot. Concrete diffs:

| Spec Section 3.2 says | V1 production code (lines 100-126 of snapshot) actually sends |
|----------------------|----------------------------------------------------------------|
| `Tags: {comma-separated tags…}` | **No `Tags:` line exists** in V1's `productInfo` template (only title/description/type/vendor) |
| `Collections: {comma-separated collection names…}` | **No `Collections:` line exists** in V1 |
| `Product type hint: Likely Chapter {N}` | V1 uses `Product type detection: Likely Chapter {N}` followed by bulleted hint lines |
| (no warning line in spec) | V1 has `Common error to avoid: {warning}` when productTypeHints.warning is set |
| (no leading instruction in user message per spec) | V1 user message begins with `You are an international customs classification expert applying GRI methodology…` (everything is one user-role message in V1 — there is no separate system message) |
| (no closing focus paragraph in spec) | V1 includes verbatim `Classify using ALL the context above. Focus on: material, construction method, primary function, target user (men/women/children/unisex), and any technical specifications that affect the classification.` |
| Output schema includes `heading_desc`, `subheading_desc`, `cn_code` (8-digit) | V1 output schema has neither `heading_desc` nor `subheading_desc`; CN code is **13-digit** in V1's "all" mode (6 international + 2 national + 2 supervisory + 3 inspection/quarantine), not 8-digit |
| `confidence: 0.0-1.0` | V1 examples show `0.92`, `0.91`, `0.90` — model is anchored on high values; no spread guidance in V1 |

**Severity: CRITICAL.** A spec called "Target Prompt Format" that doesn't match the production format it claims to mirror will produce a model that under-performs on real merchant traffic. The R3 model would expect inputs that V1 production doesn't generate (tags, collections lines) and produce outputs that V1 production doesn't consume (`heading_desc`, `subheading_desc`).

**Root issue:** The spec implicitly targets the V3 production architecture (which adds tags/collections/image and is documented in `docs/day1_accuracy_system.md` of the production repo). V3 is NOT in production. The spec conflates "what V1 sends today" with "what V3 will send" without flagging the gap.

---

## Q2. Description style risk per Section 4.3

| Style | % | Verdict | Reasoning |
|-------|---|---------|-----------|
| 1. No description (just title) | 30% | **SAFE** | Mirrors real-world reality. Many Shopify merchants leave description empty. Model learns to handle this case. |
| 2. Short generic description | 30% | **RISKY** | "Available in multiple options", "Ships from Canada" become a stylistic tell. If the model sees these phrases only on synthesized records (never on real Shopify products), it learns "synthesized data has these markers" rather than learning useful features. Real merchant copy is more idiosyncratic. |
| 3. Spec-style description | 30% | **RISKY** | "Include material composition if title contains material words" — this echoes title content into the description field. The model learns description = title-with-material-amplified, which is leakage of title features into description. Real merchant descriptions often contradict or omit material words from the title. |
| 4. Official tariff-style | 10% | **DANGEROUS** | Tariff descriptions are *causally correlated* with the HS code. Even after rephrasing, technical commodity vocabulary ("knitted of cotton, men's") strongly signals chapter and heading. The model will learn "if description sounds like tariff schedule prose → match the code in the schedule." That's not classification — that's pattern-matching schedule prose to schedule codes. **At inference, real merchants do not write in tariff prose**, so this style's training signal does not transfer. |

**Severity: CRITICAL on style 4.** Recommend dropping style 4 entirely. Consider replacing with "merchant copy with errors/typos" to teach robustness instead of memorization.

**For styles 2 and 3:** the spec gives no guidance on randomization between them. If the same product always gets the same description style, that's a data-prep deterministic leak (model could learn `hash(title) → style → code`). Spec must specify **random per-example** style assignment with a fixed seed for reproducibility.

---

## Q3. Tags synthesis circular dependency

**Spec Section 4.6:** "50% of examples: synthesize 2-4 tags from the title using common patterns (material, gender, product type, descriptor). Synthesized tags must be plausible Shopify-style tags (lowercase, hyphenated, like 'cotton', 'mens-tshirt', 'summer-collection')."

**Critique:**

1. **Circular leakage:** Synthesizing tags FROM the title means tags are a function of title. The model would learn `tags = f(title)` — but at inference, real tags are independent of (or inconsistent with) the title. The trained model treats tags as confirmatory when they're actually a noisy independent signal in production.

2. **Stylistic uniformity:** "lowercase, hyphenated, like 'cotton', 'mens-tshirt'" is **wrong** about real Shopify tags. Real tags are wildly inconsistent: "Spring 2024", "On Sale", "Best Seller", "MENS", "men's clothing", "TIKTOK FAMOUS", arbitrary internal codes. The synthesized format would teach the model to expect a clean tag namespace that production doesn't have.

3. **50/50 ratio is not grounded:** Spec offers no data on what % of real Shopify products have tags. Anecdotal: most published Shopify catalogs have 30-70% tagged products, and tag content varies. Picking 50% with no reference point is arbitrary.

4. **Bigger problem — Q1's format diff:** V1 production doesn't send a `Tags:` line at all. So this entire section is moot for matching V1 — and only relevant if the goal is V3 (which should be stated explicitly).

**Severity: HIGH.** If we're committing to tags being in the prompt (V3-style), we need real Shopify tag distributions, not synthesis-from-title.

**Concrete fix:** sample tags from a public dump of Shopify-store-tag distributions if available, or include 0-tag examples at production-realistic rates. Don't synthesize from title.

---

## Q4. RAG synthetic examples

**Spec Section 4.7:** 80% omit, 20% include 1-2 synthetic examples from same chapter as target with similarity 0.4-0.7.

**Critique:**

1. **Always-correct RAG is unrealistic.** Production RAG comes from `findSimilarCorrections()` which returns merchant corrections. Merchant corrections can be **wrong** — merchants make mistakes too. Production has a `correctionQuality` score per shop (V1 snapshot's `findSimilarCorrections` weighting) precisely to discount low-quality corrections. The spec ignores this entirely.

2. **The 80/20 split is plausible for a young app** (few corrections in DB → RAG rarely fires). But as the merchant flywheel grows, RAG fires more often. Training at 20% rate locks the model to early-stage behavior.

3. **Same-chapter only is too generous.** Real RAG can return cross-chapter examples (Jaccard similarity on title text doesn't respect chapter boundaries). The model needs to learn to discount cross-chapter RAG, not just trust same-chapter RAG.

**Severity: HIGH.** Recommend:
- Of the 20% with RAG: 70% should be correct same-chapter, 20% should be wrong same-chapter, 10% should be wrong cross-chapter — to teach the model to override bad RAG.
- Document that the 20% rate is a snapshot of *current* RAG-fire frequency and will need re-baselining as the corrections corpus grows.

---

## Q5. Country distribution + system prompt placement

**Spec Section 5:** 60/25/15 CA/US/CN, country in system prompt.

**Critique — this is the big one:**

V1 production does NOT put country anywhere as an explicit field. V1 has four modes (`ca_only`, `us_only`, `both`, `all`) chosen via `Plan.classificationMode`, and each mode has a *mode-specific preamble* embedded in the user message:

- `ca_only` → user message starts with "Classify this product with a 10-digit Canadian HS code"
- `us_only` → "with a 10-digit US HTS code"
- `both` → "with BOTH a 10-digit Canadian HS code AND a 10-digit US HTS code"
- `all` → "with ALL THREE codes…"

So country selection is implicit via the mode-specific preamble in the *user* message — not via a system prompt field, not via a "Classify for {country}" prefix.

The R3 spec proposes country in system prompt. Two problems:

1. **System prompt is empty in V1 production.** The Anthropic API call uses `messages: [{ role: "user", content: promptContent }]` with no `system` parameter. There IS no system prompt for the country to live in.

2. **V3 plan documents the same thing.** Per the V3 plan in production's `docs/day1_accuracy_system.md`, V3 also doesn't put country in a system prompt — it routes via the `mode` parameter (which feeds the user message preamble) and uses `countryOfOrigin` only as a *signal* for whether to include CN classification.

**The spec is forcing R3 training to assume a system-prompt country field that neither V1 nor V3 plan actually uses.** The trained model would not understand what production sends and vice versa.

**Severity: CRITICAL.** Recommended fix: remove the system-prompt-country idea entirely. Match V1's mode-specific preamble approach. Train each example with the appropriate preamble for its target country mix.

**Also:** 15% CN coverage is too low to teach CN reliably. The model has to learn the 13-digit CN code structure (6+2+2+3 per V1 snapshot's "all" mode guidelines) AND that the last 4-7 digits are China-specific. With only 15% CN data, the loss signal on CN-specific digits is weak. Recommend either 25-30% CN OR drop CN training entirely until we have a quality CN corpus equal to the CA corpus.

---

## Q6. heading_desc / subheading_desc lookup-table memorization

**Spec Section 3.3 + Section 6:** Assistant message includes `heading_desc` and `subheading_desc` from public tariff lookup files.

**Critique:**

1. **Not in V1 production output.** V1's JSON schema has: `hs_code`, `us_hts_code`, `chapter`, `category`, `confidence`, `reasoning`, `rules_applied`, `alternatives`, `suggestions`. NO `heading_desc`, NO `subheading_desc`. Adding them to training output makes the model produce fields production doesn't read — wasted output tokens × every classification × forever.

2. **Yes, this teaches lookup-table memorization.** Each unique HS code maps to ONE heading_desc and ONE subheading_desc by definition (these come from the official tariff schedule). So the training pairs are `(code) → (deterministic_string)`. The model learns a static lookup table, not classification reasoning. The spec author's claim ("the model learns to produce the official text, which reinforces correct classification") is unsupported — there's no mechanism by which producing canned strings reinforces the upstream classification step. If anything, it trains the model to confidently produce official-sounding prose for whatever code it picks (right or wrong).

3. **Cost analysis:** A 10-digit CA HS code has ~17,000 valid values. Each has heading_desc (~80 chars) and subheading_desc (~80 chars). At 4 chars/token, that's ~40 output tokens × every inference at production. At Wayne's projected scale (~30K classifications/month at 100 merchants), heading_desc bloat alone adds ~$3-5/month in output token cost for zero merchant value.

**Severity: CRITICAL.** Drop `heading_desc` and `subheading_desc` from training output entirely. If we want post-classification description, look it up server-side from the same tariff files — don't ask the LLM to regurgitate it.

---

## Q7. Validation gate (100 examples)

**Spec Section 9:** 100 example manual inspection covering 6 yes/no checks.

**Critique:**

1. **100 is too small for the variation matrix.** The spec defines: 4 description styles × 3 countries × tags-yes/tags-no × RAG-yes/RAG-no = 48 cells. 100 examples means ~2 per cell. Some cells will have 0 — undetectable failure modes.

2. **Manual inspection is not adversarial.** The 6 checks in the spec are confirmatory ("Do they look right?"). They miss negative cases: did extractMaterials fire when it shouldn't have? Did detectProductType give a wrong-chapter hint that we then trained on?

3. **No automated validation.** Spec proposes no scripts to verify:
   - Synthesized text passes through `extractMaterials` and produces the expected `materials` field
   - Synthesized text passes through `detectProductType` and produces the expected `productTypeHints`
   - HS code in label has 10 digits and matches a valid 2026 code (CBSA delta validator)
   - User message format byte-for-byte matches `buildClassificationPrompt(...)` output for the same inputs
   - No example has `heading_desc` revealing answer in input (data leakage)
   - Distribution actually matches stated percentages (no off-by-N bugs in style picker)

4. **No held-out eval mentioned in Section 9** (Section 10 Q6 raises this as an open question — should be Section 9's job to mandate it).

**Severity: HIGH.** Recommend:
- Increase to 500 examples for manual inspection (still cheap), distributed by stratified sampling across cells
- Add automated validation script that runs all checks on the full dataset before training
- Mandate held-out 1000-example "production-like" eval with full Shopify-style synthetic data

---

## Q8. Open Questions — reviewer answers

**Q1 (country signal).** Reviewer answer: country goes in user-message preamble matching V1's mode-specific approach. NOT in system prompt. Production has no system prompt today.

**Q2 (CN coverage 15% vs higher).** Reviewer answer: increase to 25-30% CN OR drop CN training entirely. 15% is below the threshold where the model can learn the 13-digit CN structure (per V1 snapshot's "all" mode: 6 international + 2 national subheading + 2 supervisory + 3 inspection/quarantine). Half-hearted CN training will produce confidently-wrong CN codes — the worst outcome for compliance.

**Q3 (style 4 tariff-style).** Reviewer answer: drop entirely. The risk of teaching tariff-prose-to-code memorization outweighs the benefit. Replace with "merchant-copy-with-errors" style at the same 10% to teach typo robustness instead.

**Q4 (RAG synthetic always-correct vs include some wrong).** Reviewer answer: of the 20% with RAG, include ~30% wrong-RAG examples (split: 20% wrong-same-chapter, 10% wrong-cross-chapter). Otherwise the model overtrusts RAG and production-side bad corrections poison classifications.

**Q5 (tag count deterministic vs random).** Reviewer answer: random count, weighted toward 0-2 tags (most real Shopify products have 0-3 tags). DO NOT enforce "tags-from-title" synthesis pattern — sample from real-world Shopify tag patterns instead.

**Q6 (production-like eval set).** Reviewer answer: yes, mandatory, not optional. Hold out 1000 examples with full Shopify-style synthetic structure (tags + collections + descriptions of varied quality). Eval on this set is the real metric for production performance. The current title-only 86.4% / 86.8% numbers are ceiling estimates that will not transfer.

---

## Q9. What's MISSING from the spec entirely

**Decisions that must be made before training but aren't covered:**

1. **Train/val/test split strategy.** The spec describes how to construct training data but never mentions how to hold out validation/test sets. Risk of accidental contamination.

2. **Data versioning / audit trail.** What random seed? Which version of `extractMaterials` and `detectProductType` were used? If those functions change in production after training, the trained model is implicitly stale.

3. **Negative examples.** No "deliberately wrong title" or "ambiguous inputs that should return REVIEW_NEEDED" — model never learns to refuse.

4. **Edge case handling.** Very short titles (< 5 chars), very long titles (200+ chars), non-English titles, titles with only emojis/numbers, gibberish ("Item #4471" example). Spec is silent.

5. **Code coverage analysis.** Of the ~17,000 valid 10-digit CA codes, how many appear in source data? Codes with 0 examples will never be predicted. Spec doesn't address rare-code strategy.

6. **Excluded data.** Section 5 says US examples come from "CA codes mapped to US 10-digit via 6-digit overlap (only 1:1 maps)". What % of CA codes have 1:1 US maps? If most CA codes have multiple US sub-codes (likely for textiles), the US training set could be heavily biased toward simple categories.

7. **Hyperparameter spec.** The README says R3 is `r=32, alpha=64, lr=2e-5 to 5e-5, 2-3 epochs` — but the spec doesn't reference this. Single source of truth missing.

8. **Output schema match validation.** Section 4 builds inputs but never validates that the synthesized assistant message JSON parses cleanly and matches the production schema field-by-field.

9. **Confidence calibration target.** V1 production uses raw model confidence. The R3 model will produce its own confidence. Should R3 confidence be re-calibrated post-training (Platt scaling, isotonic regression)? Spec doesn't mention.

10. **Failure mode for missing dependencies.** If `extractMaterials` returns null because the regex doesn't fire, what does the synthesized example show? Spec says "Materials detected: none / Primary material: none" but the V1 production code SKIPS the entire `Extracted product attributes:` block when materials is empty. Mismatch.

11. **Order-of-fields stability.** Does "Tags" come before "Collections" or after? Before "Type" or after? Spec doesn't lock the order. If training order != production order, the model could fail to use either.

12. **Multilingual handling.** Shopify is global. French/Chinese/Spanish product titles exist. Spec is English-only by assumption.

---

## Q10. Verdict + required edits

### Verdict: **NEEDS REWRITE**

The spec's intent is sound (match training format to inference format) but the execution doesn't actually achieve that intent. Training data per this spec would be **structurally divergent** from V1 production in 6 of 7 fields (tags/collections/output schema/country handling/style 4 leak/no system prompt). Shipping the resulting model would produce:

- **Lower accuracy on real merchant traffic** than V2.5 baseline (which is title-only and at least matches V1 inputs)
- **Confidently wrong on CN** (insufficient training data)
- **Wasted output tokens** on `heading_desc`/`subheading_desc` that production discards
- **Unpredictable behavior on style-4 tariff-prose-like inputs** (because real merchants don't write in tariff prose, the model never sees its tariff-style trigger at inference)

### Critical edits required before this is actionable

**C1. Resolve "V1 vs V3 target" ambiguity in Section 1.** Explicitly declare which production format R3 trains for. If V3, state that V3 is not yet in production and R3 will need to wait for V3 deploy OR R3 model will be deployed alongside V3.

**C2. Rewrite Section 3.2 to byte-match V1 production output.** Delete `Tags:`, `Collections:`. Add `Common error to avoid:` (conditional). Add user-message-leading sentence per mode. Fix `Product type detection:` (not `hint:`). Add closing `Classify using ALL the context above…` paragraph.

**C3. Remove `heading_desc` and `subheading_desc` from Section 3.3 output schema.** Look up server-side if needed.

**C4. Drop description Style 4 entirely.** Replace with "merchant-copy-with-errors" style.

**C5. Rewrite Section 5 country handling.** Remove "country in system prompt." Use V1 mode-specific preamble approach. Either increase CN to 25-30% or drop CN.

### High edits

**H1. Replace tag synthesis-from-title.** Sample from real-world Shopify tag distributions (or omit tags entirely if matching V1).

**H2. Increase validation set to 500.** Add automated checks per Q7.

**H3. RAG: include wrong examples.** Per Q8 Q4 answer.

**H4. Mandate held-out 1000-example production-like eval set in Section 9.** Not optional.

### Medium / Low

(See Q9 for the 12 missing items. Each warrants a paragraph in the rewrite.)

---

## Findings count

| Severity | Count |
|----------|-------|
| Critical | 4 |
| High | 4 |
| Medium | 5 |
| Low | 3 |
| **Total** | **16** |

---

## Reviewer's confidence in this critique

I have **HIGH confidence** in the critical findings (C1-C5) — they're grounded in line-by-line diffs against the V1 snapshot, which is byte-identical to production code as of 2026-04-29.

I have **MEDIUM confidence** in some HIGH findings — particularly the "real Shopify tag distributions" claim (Q3). I do not have access to a real-world Shopify tag corpus and am inferring distribution from anecdotal/secondary signals. Spec author may have better information.

I have **LOW confidence** that all 12 missing items in Q9 are equally important — some are nice-to-have (multilingual), others are blocking (train/val/test split).

**The spec author should address all 4 CRITICAL items before this is workable.** HIGH/MEDIUM/LOW can be triaged in a second pass.
