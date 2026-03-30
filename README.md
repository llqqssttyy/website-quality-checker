# Website Quality Checker

**English** | [한국어](README_KO.md)

A Claude Code skill that evaluates webpage quality based on [Google Search Quality Rater Guidelines (QRG)](https://static.googleusercontent.com/media/guidelines.raterhub.com/en//searchqualityevaluatorguidelines.pdf). Provide a URL or paste HTML/text, and get a comprehensive quality report with a grade (S/A/B/C/D) across 6 categories.

## Key Features

- **QRG-aligned evaluation** — Applies Google's official Search Quality Rater Guidelines systematically, not just SEO heuristics
- **6-category inspection** — Page purpose, MC quality, website/author info, E-E-A-T, reputation, spam/abuse
- **YMYL-aware** — Automatically detects YMYL topics (health, finance, legal, safety) and adjusts scoring weights accordingly
- **Evidence-based scoring** — Every score must be backed by explicit evidence found in the content; inference is not accepted
- **Anti-optimism bias** — Built-in cap rules, conservative defaults for unverifiable items, and a Devil's Advocate self-check to prevent inflated scores
- **Actionable output** — Not just a grade, but a prioritized improvement plan with projected score gains
- **Flexible input** — Accepts URLs (fetched via `web_fetch`) or directly pasted HTML/text
- **Partial inspection** — Can run individual categories on demand (e.g., "just check E-E-A-T")

## Overview

### How It Works

The skill runs in 3 phases:

**Phase 1: Discovery**

1. Receive input (URL or pasted content)
2. Fetch page via `web_fetch` if URL is provided
3. Classify page type (informational, commercial, blog, forum, entertainment, government)
4. Determine YMYL status (clear YMYL / borderline / non-YMYL)
5. Lock in scoring weights based on YMYL determination

**Phase 2: Analysis** 6. Run 6 categories sequentially (A → B → C → D → E → F), scoring each 0–100

**Phase 3: Synthesis** 7. Compute weighted total score 8. Check for immediate D-grade triggers (harmful content, confirmed spam, fraud reputation) 9. Run Devil's Advocate self-check on all scores ≥ 75 10. Assign final grade and generate Markdown report

### Inspection Categories

| Category                      | Weight | What It Checks                                                       |
| ----------------------------- | ------ | -------------------------------------------------------------------- |
| A. Page Purpose & Harmfulness | 15%    | Is the purpose clear, beneficial, and non-deceptive?                 |
| B. Main Content (MC) Quality  | 25%    | Effort, originality, skill, accuracy, appropriate depth              |
| C. Website / Author Info      | 10%    | Operator transparency, author credentials, contact info              |
| D. E-E-A-T                    | 25%    | Experience, Expertise, Authoritativeness, Trust                      |
| E. Reputation                 | 10%    | Independent external sources, reviews, news coverage                 |
| F. Spam / Abuse Detection     | 15%    | Copied content, scaled AI content, deceptive design, ad interference |

Weights shift for YMYL topics — see [scoring-weights.md](skills/website-quality-checker/references/scoring-weights.md) for details.

### Grading Scale

| Score  | Grade | Google QRG Equivalent |
| ------ | ----- | --------------------- |
| 90–100 | S     | Highest               |
| 75–89  | A     | High                  |
| 55–74  | B     | Medium                |
| 35–54  | C     | Low                   |
| 0–34   | D     | Lowest                |

Certain signals trigger an **immediate D grade** regardless of score:

- Category A = 0 (harmful or deceptive purpose)
- Category C = 0 + YMYL (no site/author info on YMYL topic)
- Category E ≤ 10 (fraud/crime reputation)
- Category F = 0 (confirmed spam)

### Scoring Principles

1. **Evidence-based** — Scores derive from what is explicitly present in the content, not inference. "Seems like an expert" does not count.
2. **Unverifiable = conservative** — Missing info scores 35–40, not 50. "Can't verify" means "can't trust."
3. **Cap rules** — Any ❌ caps the category at 70; two or more cap it at 50.
4. **Devil's Advocate self-check** — After scoring all 6 categories, every score ≥ 75 is re-examined. If evidence is insufficient, it drops to ≤ 60.

## Plugin Structure

```
website-quality-checker/
├── .claude-plugin/
│   └── plugin.json                          # Plugin manifest
├── skills/
│   └── website-quality-checker/
│       ├── SKILL.md                         # Skill definition (3-phase workflow)
│       └── references/                      # Loaded on demand during inspection
│           ├── google-qrg-summary.md        # QRG core: YMYL, MC quality, reputation
│           ├── eeat-criteria.md             # E-E-A-T evaluation questions
│           ├── lowest-quality-signals.md    # Immediate Lowest triggers
│           └── scoring-weights.md           # Weight tables & score formula
├── README.md                                # English
├── README_KO.md                             # Korean
├── CHANGELOG.md
└── LICENSE
```

**Progressive disclosure**: The skill body (`SKILL.md`) stays under 500 lines. Detailed criteria are in `references/` and loaded only when the relevant category is being inspected, keeping context usage efficient.

## Usage

Ask Claude Code naturally:

```
"Check the quality of https://example.com"
"Is this site trustworthy?"
"Run a page quality analysis on this URL"
"Evaluate E-E-A-T for this page"
"Is this a spam site?"
"Just check MC quality and spam signals for this page"
```

You can also paste HTML or text directly:

```
"Evaluate the quality of this content: [paste text]"
```

When no URL is provided, the reputation category (E) is marked as "unverifiable" and scored conservatively (35 points).

### Partial Inspection

Request specific categories only:

- "Just check E-E-A-T" → runs Category D only
- "Is this spam?" → runs Category F only
- "Can I trust this site?" → runs Categories C + D + E

Partial inspections include only the requested categories in the report, with the overall score labeled as "partial."

## Output

Generates a Markdown report containing:

### Report Sections

1. **Header** — Target URL, inspection date, page type, YMYL status
2. **Overall Grade** — Grade (S/A/B/C/D), score out of 100, visual progress bar
3. **Score Summary Table** — All 6 categories with weights, scores, weighted contributions, and status labels
4. **Category Details** — Per-category breakdown with ✅/⚠️/❌ verdicts and evidence citations for each item
5. **Improvement Action Plan** — Prioritized into 3 tiers:
   - Immediate fixes (high impact, low effort)
   - Mid-term improvements (high impact, medium effort)
   - Strategic improvements (long-term)
6. **Projected Improvement** — Before/after score estimates per category if all recommendations are applied
7. **Devil's Advocate Log** — Documents any score adjustments made during self-check
8. **Key Takeaway** — 1 strength + 1–2 weaknesses in ≤ 3 sentences

### Sample Grade Output

```
## Overall Grade: B (68/100)

0         20        40        60        80        100
|---------|---------|---------|---------|---------|
██████████████████████████████████░░░░░░░░░░░░░░░░░ 68/100
                                  ^
```

## Installation

```bash
git clone https://github.com/llqqssttyy/website-quality-checker.git
```

## Reference Documents

The skill loads these on demand during inspection:

- [google-qrg-summary.md](skills/website-quality-checker/references/google-qrg-summary.md) — YMYL classification, MC quality axes, reputation standards
- [eeat-criteria.md](skills/website-quality-checker/references/eeat-criteria.md) — Detailed E-E-A-T evaluation questions per dimension
- [lowest-quality-signals.md](skills/website-quality-checker/references/lowest-quality-signals.md) — Immediate Lowest-quality triggers (harmful, untrustworthy, spammy)
- [scoring-weights.md](skills/website-quality-checker/references/scoring-weights.md) — Weight tables, YMYL adjustments, score calculation formula

## License

[MIT](LICENSE)
