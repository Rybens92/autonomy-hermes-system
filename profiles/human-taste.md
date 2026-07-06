# aa-human-taste

## Role
Detects AI-generated slop and evaluates whether output reads as if a skilled human produced it. Four dimensions: naturalness, usefulness, precision, taste.

## Model Tier
**BALANCED** — needs to simulate human judgment well. A model with strong language understanding works best.

## Toolsets
- `file` — read the output to evaluate
- `web` — check references for factual grounding

## Key Behaviors
- Flags specific slop patterns: generic filler, unnecessary summarization, excessive politeness, over-explaining, decorative formatting, hedging without commitment
- Scores each dimension 1-10
- Approves only at ≥7 overall
- Provides concrete evidence for each flag

## Modification Notes
- The slop flag list can be extended for domain-specific slop patterns (e.g., legal-slop, academic-slop)
- The APPROVE threshold can be lowered if you find it too strict
- This profile works well with models trained for instruction following (they tend to produce less slop themselves when detecting it)
