# V3 Target Architecture Snapshot
# Date: 2026-04-29
# Source: production repo (sanitized)
# Status: V3 plan — what production will be post-approval
# Purpose: target format that R3 training data must match

# Day 1 Accuracy System — Complete Implementation
# Run in Claude Code from your Railway project root
# DO NOT run until Shopify approves the app
# This is a single Claude Code session — paste everything at once

## CONTEXT
App: HS classification app — production deployment private
Current accuracy: 65.4% exact
Target accuracy: 90%+ published (via confidence gating)
Current model: claude-haiku-4-5-20251001
Codes: Canadian HS (10-digit), US HTS (10-digit), CN codes (already live)

## SAFETY RULES
- Read DEVELOPMENT_RULES.md before touching any file
- Create a git commit before starting: git add -A && git commit -m "snapshot before day1 accuracy upgrade"
- Use feature flag CLASSIFIER_VERSION=v3 — do not replace v1/v2, add v3
- Only touch: app/services/classifier.server.ts, app/routes/api.classify.tsx
- Do NOT touch: billing, GDPR webhooks, cron jobs, UI routes, auth

---

## STEP 1 — Backup current classifier

cp app/services/classifier.server.ts app/services/classifier.server.ts.backup
git add -A && git commit -m "backup before day1 accuracy upgrade"

---

## STEP 2 — Update GraphQL query to fetch all available signals

In the file that fetches products (app/routes/api.classify.tsx or wherever 
the PRODUCTS_QUERY lives), update to fetch ALL available Shopify signals:

```graphql
const PRODUCTS_QUERY = `
  query getProducts($first: Int!, $after: String) {
    products(first: $first, after: $after) {
      nodes {
        id
        title
        description
        productType
        vendor
        tags
        collections(first: 5) {
          nodes { title }
        }
        variants(first: 1) {
          nodes {
            selectedOptions {
              name
              value
            }
          }
        }
        featuredMedia {
          ... on MediaImage {
            image {
              url
              altText
            }
          }
        }
        metafields(first: 5, namespace: "[brand]") {
          nodes { key value }
        }
      }
      pageInfo { hasNextPage endCursor }
    }
  }
`
```

---

## STEP 3 — Build the complete v3 classifier in classifier.server.ts

Add this AFTER the existing v1/v2 code. Do not remove existing code.

```typescript
// ============================================================
// V3 CLASSIFIER — Day 1 Accuracy System
// Master Prompt + 17 Trap Rules + Two-Stage Hierarchical
// + Description Normalization + All Shopify Signals
// + CN codes + US HTS codes
// Activate: Railway → Variables → CLASSIFIER_VERSION=v3
// ============================================================

// --- DESCRIPTION NORMALIZER ---
function normalizeDescription(desc: string): string {
  if (!desc) return "";
  return desc
    .replace(/\b(XS|S|M|L|XL|XXL|XXXL|\d+XL)\b/gi, "")
    .replace(/\bUS\s*\d+\.?\d*\b/gi, "")
    .replace(/\bEU\s*\d+\b/gi, "")
    .replace(/\bUK\s*\d+\b/gi, "")
    .replace(/\b(BLACK|WHITE|GREY|GRAY|RED|BLUE|GREEN|PINK|PURPLE|YELLOW|BROWN|IVORY|NAVY|OLIVE|CREAM|GOLD|SILVER|ROSE|MAUVE|PRALINE|LAVENDER|CHARCOAL|BEIGE|TAUPE|COGNAC|CAMEL|RUST|SAGE|TEAL|CORAL|MUSTARD|BURGUNDY|EMERALD|COBALT|AMBER)\b/gi, "")
    .replace(/\b(ONE SIZE|ONESIZE)\b/gi, "")
    .replace(/\b(NWT|BNWT|NEW WITH TAGS|NWOB)\b/gi, "")
    .replace(/\b\d{10,}\b/g, "")
    .replace(/\s+/g, " ")
    .trim();
}

// --- SIGNAL BUILDER ---
interface ProductSignals {
  title: string;
  description: string;
  productType: string;
  vendor: string;
  tags: string[];
  collections: string[];
  variantOptions: string;
  imageUrl: string;
  countryOfOrigin: string;
}

function buildClassificationContext(signals: ProductSignals): string {
  const parts: string[] = [];

  const normalizedTitle = normalizeDescription(signals.title);
  const normalizedDesc = normalizeDescription(signals.description);

  parts.push(`Product title: ${normalizedTitle}`);

  if (normalizedDesc && normalizedDesc.length > 5) {
    parts.push(`Description: ${normalizedDesc.slice(0, 400)}`);
  }

  if (signals.productType) {
    parts.push(`Product type: ${signals.productType}`);
  }

  if (signals.tags && signals.tags.length > 0) {
    // Tags are extremely valuable — "cotton", "mens", "leather" etc
    const relevantTags = signals.tags
      .filter(t => t.length > 2 && t.length < 40)
      .slice(0, 10)
      .join(", ");
    if (relevantTags) parts.push(`Tags: ${relevantTags}`);
  }

  if (signals.collections && signals.collections.length > 0) {
    parts.push(`Collections: ${signals.collections.join(", ")}`);
  }

  if (signals.variantOptions) {
    parts.push(`Variant options: ${signals.variantOptions}`);
  }

  if (signals.countryOfOrigin) {
    parts.push(`Country of origin: ${signals.countryOfOrigin}`);
  }

  return parts.join("\n");
}

// --- MASTER SYSTEM PROMPT V3 ---
const MASTER_PROMPT_V3 = `You are a Canadian customs broker with 20 years of CBSA experience. You have classified over 500,000 products. You think like a CBSA auditor — methodical, never guessing.

PHASE 1 — EXTRACT KEY ATTRIBUTES:
Material (cotton/polyester/leather/rubber/plastic/wood/metal/glass)
Function (what does it do?)
Construction (knitted/woven/moulded/assembled)
End user (men/women/children/unisex)
Power source (electric/manual/none)

PHASE 2 — APPLY TRAP RULES (HARD OVERRIDES — check these FIRST):

GARMENT TRAPS:
TRAP 1: KNITTED fabric (jersey/rib/interlock/cashmere knit/pointelle/fishnet) → Ch61 ALWAYS
TRAP 2: WOVEN fabric (denim/canvas/twill/chiffon/satin/lace) → Ch62 ALWAYS
TRAP 3: Tights/stockings/hosiery/pantyhose/fishnet tights → 6115 ALWAYS
TRAP 4: Hoodie/pullover/sweatshirt/jumper → 6110 ALWAYS, never 6101
TRAP 5: Real leather jacket/coat → 4203.10 ALWAYS, never Ch61/Ch62
TRAP 6: Faux/vegan/synthetic leather garment → Ch62 woven ALWAYS
TRAP 7: Bra/bustier/swimsuit/swimwear/lingerie → 6212 ALWAYS
TRAP 8: Wearable blanket/sherpa throw/snuggie → 6301, NOT 6110
TRAP 9: Necktie/bow tie/scarf → 6215 ALWAYS

ACCESSORIES TRAPS:
TRAP 10: Handbag/shoulder bag/satchel/crossbody → 4202.21 or 4202.22
TRAP 11: Tote bag/shopping bag/reusable bag → 4202.92
TRAP 12: Wallet/purse/billfold/clutch → 4202.31 leather, 4202.32 textile
TRAP 13: Hat/cap/beanie/beret/bucket hat/headwear → Ch65 ALWAYS
TRAP 14: Hair clips/barrettes/bobby pins → 9615
TRAP 15: Hair scrunchie/hair tie → 9615.19

HOUSEHOLD/TECH TRAPS:
TRAP 16: Vacuum insulated/double-wall/thermos/flask → 9617.00 ALWAYS
TRAP 17: Power bank/portable battery charger → 8507.60 ALWAYS
TRAP 18: Electric toothbrush → 8509.80 ALWAYS
TRAP 19: Resistance bands/exercise bands of rubber → 4016.99 ALWAYS
TRAP 20: Yoga mat/exercise mat/foam mat → 3926.90
TRAP 21: Phone case/cover silicone or plastic → 3926.90
TRAP 22: Artist paint tubes retail → 3212.10, never 3208

HOME/TEXTILE TRAPS:
TRAP 23: Bedspread/comforter/duvet/quilt → 9404.40
TRAP 24: Artificial/fake flowers/plants/vines → 6702
TRAP 25: Vinyl record/LP album → 8523.80
TRAP 26: Video game disc/cartridge → 8523.49
TRAP 27: Book/novel/printed matter → 4901
TRAP 28: Calendar/desk planner/agenda → 4910

FOOTWEAR RULE (ALWAYS return a code — never empty):
- Leather upper + rubber/plastic sole → 6403
- Leather upper + leather sole → 6405
- Textile upper → 6404
- All rubber/plastic → 6402
- Waterproof → 6401
- Default if unsure → 6404.19

PHASE 3 — HEADING (4-digit) using GRI 1
PHASE 4 — SUBHEADING (6-digit) using GRI 6
PHASE 5 — CANADIAN SUFFIX (10-digit) — use 00 if uncertain about last 4 digits
PHASE 6 — SELF CHECK: Does the CBSA T2026 heading description match?

CRITICAL RULES:
- NEVER return empty — always return your best classification
- When uncertain about last 4 digits, use 0000 as suffix
- Confidence < 0.70 = flag for review but still return a code

Return ONLY valid JSON:
{
  "hs_code": "6109100051",
  "chapter": "61",
  "heading": "6109",
  "subheading": "610910",
  "confidence": 0.92,
  "trap_triggered": "TRAP 1 - knitted fabric",
  "reasoning": "Cotton jersey knit = Ch61. T-shirt = 6109. Cotton = 6109.10.",
  "duty_rate": "18%"
}`;

// --- US HTS SYSTEM PROMPT ---
const US_HTS_PROMPT_V3 = `You are a US customs classification expert specializing in HTS (Harmonized Tariff Schedule of the United States). Apply the same GRI rules as Canadian HS but use US-specific 10-digit codes.

Key US differences from Canadian HS:
- Chapter 98/99 apply for US-specific provisions
- Statistical suffixes differ from Canada
- Section 301 tariffs apply to Chinese-origin goods (additional 25-145%)
- USMCA (Ch103) zero-duty for Canadian/Mexican origin goods

Apply the same 28 TRAP RULES as Canadian classification.
First 6 digits are identical globally — only last 4 digits differ.

Return ONLY valid JSON:
{
  "hts_code": "6109100010",
  "chapter": "61",
  "heading": "6109",
  "subheading": "610910",
  "confidence": 0.92,
  "general_rate": "16.5%",
  "section301_rate": "+25% if China origin",
  "trap_triggered": "none",
  "reasoning": "Cotton T-shirt, knitted, HTS 6109.10.00.10 for men's"
}`;

// --- CN CODE SYSTEM PROMPT ---
const CN_CODE_PROMPT_V3 = `You are a Chinese customs classification expert specializing in CN commodity codes for export from China. Apply GRI rules using Chinese customs schedule.

CN codes are 10-digit. First 8 digits follow HS. Last 2 digits are China-specific.
CN codes are used for: Chinese export declarations, GACC compliance, export licensing.

Apply the same 28 TRAP RULES as Canadian classification.

Return ONLY valid JSON:
{
  "cn_code": "6109100090",
  "chapter": "61",
  "heading": "6109", 
  "confidence": 0.88,
  "export_tax_rebate": "13%",
  "licensing_required": false,
  "trap_triggered": "none",
  "reasoning": "Cotton T-shirt exported from China, CN 6109100090"
}`;

// --- TWO-STAGE HIERARCHICAL CLASSIFIER ---
async function classifyChapter(context: string): Promise<string> {
  const chapterList = `
01-05: Live animals/animal products
06-14: Vegetable products  
15: Fats and oils
16-24: Food and beverages
25-27: Minerals/fuels
28-38: Chemicals
39-40: Plastics/rubber
41-43: Leather/fur
44-46: Wood products
47-49: Paper/printed matter
50-63: Textiles and clothing
64-67: Footwear/headgear/accessories
68-70: Stone/ceramics/glass
71: Jewellery/gems
72-83: Metals
84: Machinery (non-electric)
85: Electrical equipment
86-89: Transport
90-92: Precision/optical/musical
93: Arms
94: Furniture/lamps/bedding
95: Toys/games/sports
96: Miscellaneous manufactured
97-99: Special`;

  const response = await anthropic.messages.create({
    model: "claude-haiku-4-5-20251001",
    max_tokens: 50,
    system: `You are a customs expert. Given a product, return ONLY the 2-digit chapter number. Examples: "61" for knitted clothes, "64" for shoes, "85" for electronics. Nothing else.`,
    messages: [{
      role: "user",
      content: `${context}\n\nChapter list:\n${chapterList}\n\nReturn ONLY the 2-digit chapter number:`
    }]
  });

  const text = response.content[0].type === "text" ? response.content[0].text.trim() : "";
  const match = text.match(/\b(\d{2})\b/);
  return match ? match[1] : "";
}

async function classifyWithChapterHint(
  context: string,
  chapter: string,
  prompt: string,
  codeField: string
): Promise<any> {
  const chapterHint = chapter
    ? `\n\nNOTE: This product has been pre-classified to Chapter ${chapter}. Confirm or correct, then provide the full 10-digit code within this chapter.`
    : "";

  const response = await anthropic.messages.create({
    model: "claude-haiku-4-5-20251001",
    max_tokens: 400,
    system: prompt + chapterHint,
    messages: [{
      role: "user",
      content: context
    }]
  });

  const text = response.content[0].type === "text" ? response.content[0].text : "";
  const match = text.match(/\{[\s\S]*?\}/);
  if (match) {
    try {
      return JSON.parse(match[0]);
    } catch {
      return null;
    }
  }
  return null;
}

// --- VISION CLASSIFIER (Image layer) ---
async function classifyWithImage(
  context: string,
  imageUrl: string,
  prompt: string
): Promise<any> {
  if (!imageUrl) return null;

  try {
    // Fetch image as base64
    const imgResp = await fetch(imageUrl);
    if (!imgResp.ok) return null;
    const buffer = await imgResp.arrayBuffer();
    const base64 = Buffer.from(buffer).toString("base64");
    const contentType = imgResp.headers.get("content-type") || "image/jpeg";

    const response = await anthropic.messages.create({
      model: "claude-haiku-4-5-20251001",
      max_tokens: 400,
      system: prompt + "\n\nYou have been provided with a product image. Use both the image and text to classify.",
      messages: [{
        role: "user",
        content: [
          {
            type: "image",
            source: {
              type: "base64",
              media_type: contentType as any,
              data: base64
            }
          },
          {
            type: "text",
            text: context
          }
        ]
      }]
    });

    const text = response.content[0].type === "text" ? response.content[0].text : "";
    const match = text.match(/\{[\s\S]*?\}/);
    if (match) return JSON.parse(match[0]);
  } catch (e) {
    console.error("[Vision] Image classification failed:", e);
  }
  return null;
}

// --- CBSA 2026 DELTA VALIDATOR ---
import deltaValidator from "../data/cbsa_delta_validator.json";

function applyDeltaCorrection(hsCode: string): { code: string; corrected: boolean; note: string } {
  const digits = hsCode.replace(/\D/g, "");
  
  // Check if code was deleted in 2026
  if (deltaValidator.deleted_codes.includes(digits)) {
    const replacement = deltaValidator.replacement_map[digits];
    if (replacement && replacement.new_code) {
      return {
        code: replacement.new_code,
        corrected: true,
        note: `Code ${digits} removed in CBSA 2026 schedule. Updated to ${replacement.new_code}.`
      };
    }
  }
  
  return { code: digits, corrected: false, note: "" };
}

// --- CONFIDENCE CALIBRATOR ---
function calibrateConfidence(
  rawConfidence: number,
  trapTriggered: string,
  deltaCorrection: boolean,
  hasImage: boolean,
  descriptionLength: number
): number {
  let conf = rawConfidence;
  
  // Boost if trap rule fired (high certainty)
  if (trapTriggered && trapTriggered !== "none") conf += 0.05;
  
  // Boost if image was used
  if (hasImage) conf += 0.03;
  
  // Penalise if delta correction was applied
  if (deltaCorrection) conf -= 0.15;
  
  // Penalise if description is very sparse
  if (descriptionLength < 15) conf -= 0.10;
  
  // Clamp between 0.10 and 0.97
  return Math.min(0.97, Math.max(0.10, conf));
}

// --- MAIN V3 EXPORT FUNCTION ---
export async function classifyProductV3(
  productId: string,
  title: string,
  description: string,
  productType: string,
  vendor: string,
  tags: string[] = [],
  collections: string[] = [],
  variantOptions: string = "",
  imageUrl: string = "",
  countryOfOrigin: string = "CN",
  shopId: string = "",
  classifyCA: boolean = true,
  classifyUS: boolean = true,
  classifyCN: boolean = true
): Promise<any> {

  const signals: ProductSignals = {
    title, description, productType, vendor,
    tags, collections, variantOptions,
    imageUrl, countryOfOrigin
  };

  const context = buildClassificationContext(signals);

  // Stage 1: Get chapter hint (fast, cheap)
  const chapterHint = await classifyChapter(context);

  // Stage 2: Full classification with chapter hint
  // Run CA, US, CN in parallel for speed
  const [caResult, usResult, cnResult] = await Promise.all([
    classifyCA
      ? (imageUrl
          ? classifyWithImage(context + (chapterHint ? `\nPre-classified chapter: ${chapterHint}` : ""), imageUrl, MASTER_PROMPT_V3)
          : classifyWithChapterHint(context, chapterHint, MASTER_PROMPT_V3, "hs_code"))
      : Promise.resolve(null),
    classifyUS
      ? classifyWithChapterHint(context, chapterHint, US_HTS_PROMPT_V3, "hts_code")
      : Promise.resolve(null),
    classifyCN && countryOfOrigin === "CN"
      ? classifyWithChapterHint(context, chapterHint, CN_CODE_PROMPT_V3, "cn_code")
      : Promise.resolve(null)
  ]);

  // Apply CBSA 2026 delta correction to CA code
  let caCode = caResult?.hs_code || "REVIEW_NEEDED";
  let deltaNote = "";
  let deltaCorrection = false;

  if (caCode !== "REVIEW_NEEDED") {
    const delta = applyDeltaCorrection(caCode);
    caCode = delta.code;
    deltaCorrection = delta.corrected;
    deltaNote = delta.note;
  }

  // Calibrate confidence
  const calibratedConf = calibrateConfidence(
    caResult?.confidence || 0.5,
    caResult?.trap_triggered || "",
    deltaCorrection,
    !!imageUrl,
    (title + description).length
  );

  // Auto-publish decision
  const autoPublish = calibratedConf >= 0.80;
  const needsReview = calibratedConf < 0.60;

  return {
    // Canadian HS
    hs_code: caCode,
    chapter: caResult?.chapter || caCode.slice(0, 2),
    chapter_name: caResult?.chapter_name || "",
    heading: caResult?.heading || caCode.slice(0, 4),
    subheading: caResult?.subheading || caCode.slice(0, 6),
    confidence: calibratedConf,
    trap_triggered: caResult?.trap_triggered || "none",
    reasoning: caResult?.reasoning || "",
    duty_rate: caResult?.duty_rate || "",
    delta_corrected: deltaCorrection,
    delta_note: deltaNote,
    chapter_hint_used: chapterHint,
    image_used: !!imageUrl,

    // US HTS
    hts_code: usResult?.hts_code || null,
    hts_general_rate: usResult?.general_rate || null,
    hts_section301: usResult?.section301_rate || null,

    // CN codes
    cn_code: cnResult?.cn_code || null,
    cn_export_rebate: cnResult?.export_tax_rebate || null,

    // Decision signals
    auto_publish: autoPublish,
    needs_review: needsReview,
    version: "v3_day1",
    tier: 3
  };
}
```

---

## STEP 4 — Wire V3 into the action handler

In the action function that handles classification requests, add:

```typescript
const classifierVersion = process.env.CLASSIFIER_VERSION || "v1";

if (classifierVersion === "v3") {
  // Parse all new signals from formData
  const tags = JSON.parse(formData.get("tags") as string || "[]");
  const collections = JSON.parse(formData.get("collections") as string || "[]");
  const variantOptions = formData.get("variantOptions") as string || "";
  const imageUrl = formData.get("imageUrl") as string || "";

  result = await classifyProductV3(
    productId, title, description, productType, vendor,
    tags, collections, variantOptions, imageUrl,
    countryOfOrigin, shopId,
    true, // classifyCA
    true, // classifyUS  
    countryOfOrigin === "CN" // classifyCN only for Chinese products
  );
} else {
  // existing v1/v2 logic unchanged
}
```

---

## STEP 5 — Update UI to display all 3 codes

In the products table component, add columns for:
- CA HS Code (existing)
- US HTS Code (new — show only if classified)
- CN Code (new — show only if country of origin is CN)
- Confidence badge (green ≥ 0.80, yellow 0.60-0.79, red < 0.60)
- Delta correction indicator (show ⚠️ if code was corrected)

---

## STEP 6 — Add cbsa_delta_validator.json to app/data/

Copy the file from the outputs directory.
Add to .gitignore if it contains sensitive data (it doesn't — it's public tariff data).
Import in classifier.server.ts as shown above.

---

## STEP 7 — Environment variable setup

In Railway dashboard, add:
CLASSIFIER_VERSION=v3

To roll back instantly:
CLASSIFIER_VERSION=v1

---

## STEP 8 — Test with 5 products before enabling for all merchants

After deploying, test with these 5 products manually:
1. "Men's 100% cotton t-shirt" — expect 6109.10
2. "Vacuum insulated water bottle 750ml" — expect 9617.00
3. "Genuine leather wallet" — expect 4202.31
4. "Wireless Bluetooth earbuds" — expect 8518.30
5. "Children's wooden building blocks" — expect 9503.00

All 5 should be correct before enabling CLASSIFIER_VERSION=v3 for merchants.

---

## WHAT THIS BUILDS

| Feature | Impact | Notes |
|---|---|---|
| Description normalization | +3% | Removes size/colour noise |
| All 6 Shopify signals | +2% | Tags, collections, variants |
| Master prompt V3 + 28 traps | +7% | Hard override rules |
| Two-stage hierarchical | +8% | Chapter first, then subheading |
| Vision layer (images) | +3% | Uses Haiku vision |
| CBSA 2026 delta validator | +2% | Fixes deleted codes |
| Confidence calibration | real signals | Not hardcoded 0.92 |
| Auto-publish gate ≥ 0.80 | ~95% published | Routes low-conf to review |
| US HTS codes | new feature | Parallel classification |
| CN codes | already live | Now via v3 prompt |

Expected result: 65% raw → 85%+ raw → 95%+ published (via confidence gate)

