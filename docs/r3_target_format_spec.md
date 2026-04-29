# R3 Training — Target Prompt Format Specification

**Status:** Draft for review
**Date:** 2026-04-29
**Goal:** Define the exact prompt format that R3 training data must match
so the fine-tuned model performs well on real Shopify products.

---

## 1. The Core Problem

The R2 model was trained on a simple prompt format:
```
Classify for CA:
Product: {short customs-style description}
Country of Origin: {2-letter code}
```

But real Shopify merchants have richer product data: descriptions, tags,
product types, vendor names, materials. The production app extracts these
signals and sends them to the model. If the model never saw them during
training, it cannot use them.

**Fix:** R3 training data must use the same prompt format that production
sends to the model.

---

## 2. Three Layers of Truth

Before anything else, distinguish these:

### Layer 1 — What Shopify makes available
The Shopify GraphQL Admin API exposes these fields on every Product:
title, description, descriptionHtml, vendor, productType, tags,
collections, variants (with selectedOptions), featuredMedia, metafields.

Source: shopify.dev/docs/api/admin-graphql/latest/objects/Product

### Layer 2 — What our app fetches
The V3 production query (per docs/day1_accuracy_system.md) requests:
title, description, productType, vendor, tags, collections,
variants/selectedOptions, featuredMedia, metafields(namespace: [brand]).

These are all available without additional Shopify scopes beyond
`read_products` which the app already has.

### Layer 3 — What we send to the model
This is what `buildClassificationPrompt()` builds from the fetched data
plus three derived signals:
- materials (regex extraction from title+description)
- productTypeHints (keyword chapter detection)
- ragExamples (similar past corrections from merchant flywheel)

**The training data must match Layer 3.**

---

## 3. Target Prompt Format (V3)

### 3.1 System message
```
You are an international customs classification expert applying GRI
methodology. Classify this product with a 10-digit Canadian HS code,
a 10-digit US HTS code, and an 8-digit Chinese HS code.
```

(System prompt content stays in production code — it does not need to be
in training data identically. The model learns the task from
input→output pairs, not from the system prompt being identical.)

### 3.2 User message — full schema
```
Extracted product attributes:
Materials detected: {comma-separated material list with percentages}
Primary material: {single material or "none"}

Product title: {title, max 200 chars}
Key details: {description, max 600 chars, plain text after HTML strip}
Type: {productType from Shopify, or "Not specified"}
Vendor: {vendor from Shopify, or "Not specified"}
Tags: {comma-separated tags, max 10, or "Not specified"}
Collections: {comma-separated collection names, max 5, or "Not specified"}

Product type hint: {chapter detection result from detectProductType, or omit}

Reference classifications from verified merchant data:
{0-3 RAG examples, or omit entire section if none}

Classify using ALL the context above. Focus on:
material, construction method, primary function,
target user (men/women/children/unisex), and any
technical specifications that affect the classification.

Return ONLY valid JSON with no other text:
{output schema}
```

### 3.3 Assistant message — full schema
```json
{
  "hs_code": "10-digit CA HS code",
  "us_hts_code": "10-digit US HTS code",
  "cn_code": "8-digit CN HS code (or null if mode != all)",
  "chapter": "2-digit chapter",
  "category": "short product category name",
  "confidence": 0.0-1.0,
  "reasoning": "2-3 sentence explanation",
  "rules_applied": ["GRI 1", "GRI 6", ...],
  "heading_desc": "official heading description from CA tariff schedule",
  "subheading_desc": "official subheading description from CA tariff schedule"
}
```

---

## 4. Training Data Construction

### 4.1 Source data fields
Each source record provides:
- `product_description` — short customs-style description (becomes title)
- `hs_code_ca` — verified 10-digit CA HS code (the label)
- `country_of_origin` — NOT used in user message (production doesn't send it)

### 4.2 Generated user message — natural variation
Real Shopify products have wildly different description quality. The
training data must mirror this distribution to avoid teaching the model
that descriptions are always present:

| Description style | % of training examples |
|------------------|------------------------|
| No description (just title) | 30% |
| Short generic description | 30% |
| Spec-style description | 30% |
| Official tariff-style description | 10% |

For each example, the description style is chosen randomly per the
distribution above.

### 4.3 Generated user message — per-style construction

**Style 1: No description (30%)**
```
Product title: {title from source}
Key details:
Type: Not specified
Vendor: Not specified
```

**Style 2: Short generic description (30%)**
Build a description that mimics typical low-effort merchant copy:
- Take title
- Add 1-2 generic phrases ("Available in multiple options", "Ships from Canada", etc.)
- 50-150 chars total

**Style 3: Spec-style description (30%)**
Build a description that mimics detail-oriented merchants:
- Include material composition if title contains material words
- Include construction method if known (knit/woven/molded/etc.)
- Include dimensions or capacity if applicable
- 200-400 chars total

**Style 4: Official tariff-style (10%)**
Lift the official tariff description text (already in the lookup files)
and rephrase slightly so the model learns to handle technical commodity
language without memorizing the tariff text verbatim.

### 4.4 Materials field
For ALL examples, run the production `extractMaterials()` regex against
the constructed (title + description). Use whatever it returns.

If extractMaterials finds nothing, output:
```
Materials detected: none
Primary material: none
```

This matches what production does when the regex matches nothing — the
model must learn to handle this case.

### 4.5 Product type hint
Run the production `detectProductType()` keyword detector against the
title. If it returns a result, include this line:
```
Product type hint: Likely Chapter {N} — {warning if any}
```
If no match, omit the line entirely (matches production behavior).

### 4.6 Tags and Collections
The source dataset doesn't have tags or collections. For training:
- 50% of examples: omit (Type: Not specified pattern)
- 50% of examples: synthesize 2-4 tags from the title using common
  patterns (material, gender, product type, descriptor)

Synthesized tags must be plausible Shopify-style tags (lowercase,
hyphenated, like "cotton", "mens-tshirt", "summer-collection").

### 4.7 RAG examples
The source dataset doesn't have merchant corrections to RAG against.
For training:
- 80% of examples: omit the "Reference classifications" section entirely
- 20% of examples: include 1-2 SYNTHETIC reference examples from the
  same chapter as the target code, with similarity scores 0.4-0.7

This trains the model to use RAG context when present and ignore it
when absent.

---

## 5. Country Distribution

R3 trains a single model on all three markets. Distribution:

| Market | % of training examples | Source |
|--------|------------------------|--------|
| CA | 60% | Source dataset (largest, highest quality) |
| US | 25% | CA codes mapped to US 10-digit via 6-digit overlap (only 1:1 maps) |
| CN | 15% | CA codes truncated to 8-digit when CN equivalent exists |

US and CN examples use the same user message format with:
- "Classify for {country}" prefix removed (model infers from system prompt
  variant or assistant output `country` field)
- Same description style distribution as CA

**Decision needed:** does the model see country in the user message
or system message? Production behavior must match training. Default
plan: country in system prompt only, model outputs country in JSON.

---

## 6. Assistant Output — Approach A confirmed

The assistant message includes the official tariff descriptions in
`heading_desc` and `subheading_desc`. These come from the public
tariff lookup files, looked up by the verified HS code.

This is safe because it appears in the OUTPUT not the input. The
model learns to produce the official text, which reinforces correct
classification. Reviewer confirmed this is non-corrupting.

---

## 7. What This Spec Is NOT

This spec is intentionally not:
- A change to the production app (production app stays frozen)
- A change to the source data labels (labels remain ground truth)
- An attempt to inject answers into the user message
- A use of any unverified data source

This spec IS:
- A way to make training data look like inference data
- A way for the model to learn signals it currently ignores
- A repeatable process from existing source data + lookup files

---

## 8. Files Required

To execute this spec, the following files must be readable:

| File | Purpose | Status |
|------|---------|--------|
| Source dataset (CA records) | Title + verified CA code | Available privately |
| ca_tariff_lookup.json | Official CA descriptions + rates | Available privately |
| us_hts_lookup.json | Official US descriptions + rates | Available privately |
| china_hs_with_desc.json | Official CN descriptions | Available privately |
| extractMaterials regex | Production code — port to Python | In v1 snapshot |
| detectProductType keywords | Production code — port to Python | In v1 snapshot |

All files exist. None need to be created.

---

## 9. Validation Before Training

Before running the full data prep, build 100 examples and manually inspect:
1. Do the user messages look like real Shopify products would?
2. Are descriptions varied (some empty, some short, some long, some technical)?
3. Are tags/collections plausibly synthesized?
4. Does the materials field match what extractMaterials would output?
5. Does the productType hint appear when expected?
6. Is the assistant output schema correct?

If 100 examples look right, scale to full dataset.

---

## 10. Open Questions for Reviewer

Q1. Country signal: in system prompt or user message? Production = system.
    Training = match.

Q2. CN coverage: 15% may be too low for the model to learn CN reliably.
    Should we increase to 20-25%?

Q3. Description style 4 (tariff-style at 10%): is rephrasing safe enough
    to prevent the model from learning "if description is technical, copy
    the tariff text"? Or should we drop this style entirely?

Q4. RAG synthetic examples: should they always be CORRECT examples, or
    should some be deliberately wrong (to teach the model to override
    bad RAG suggestions)?

Q5. Tags synthesis: should we use a deterministic rule (always 3 tags)
    or random count (1-5)? Real Shopify tags vary widely.

Q6. Should we hold out a 1000-example "production-like" eval set with
    full Shopify-style synthetic data to measure real-world accuracy
    instead of the title-only eval we have now?
