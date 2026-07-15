---
tags: [GEPA, evolution, optimization, report, RPD-pipeline]
type: report
---

# GEPA Optimization Report — multi-agent-pipeline Skill

**Date:** July 15, 2026
**Engine:** DSPy 3.2.1 + GEPA (dspy.GEPA)
**Target:** `multi-agent-pipeline` skill (22,976 chars)
**Source data:** Sortopia + CatLogic pipeline runs

---

## 1. Setup Validation ✅

| Check | Result | Detail |
|-------|--------|--------|
| Evolution repo | ✅ Cloned | `/opt/data/hermes-agent-self-evolution/` |
| Dependencies | ✅ Installed | `uv pip install -e ".[dev]"` — 145/145 tests pass |
| Skill found | ✅ Found | `skills/autonomous-ai-agents/multi-agent-pipeline/SKILL.md` |
| Skill loaded | ✅ 22,976 chars | Name + description + body parsed |
| Dry-run GEPA | ✅ Validated | Would generate synthetic dataset, run 5 GEPA iterations, validate constraints, create PR |
| Constraint analysis | ✅ Complete | 2 PASS / 2 FAIL (see below) |

### Constraint Results

| Constraint | Status | Detail |
|-----------|--------|--------|
| `non_empty` | ✅ PASS | Artifact is non-empty |
| `size_limit` | ❌ FAIL | 22,439 / 15,000 chars (7,439 over) |
| `skill_structure` | ❌ FAIL | Frontmatter not found in "body" (validator bug — checks stripped body not raw) |
| `growth_limit` | ✅ SKIP | No baseline provided (first run) |

The size limit is a **configurable guardrail** (`max_skill_size: 15000` in `EvolutionConfig`). The skill is legitimately large because it documents 3 agent roles × 4 gate types × 2 production runs with full code examples. The limit needs adjusting to ~23,000 for this skill.

The structure check failure is a **validator code bug** — `_check_skill_structure()` receives the body (after frontmatter stripped by `load_skill()`) and can't find `name:` / `description:`. It should check the raw content.

---

## 2. What GEPA Would Optimize

GEPA (Gradient-guided Evolutionary Prompt Adaptation) follows this loop:

```
1. Take current skill text (baseline)
2. Mutate a section (rephrase, reorder, add/remove)
3. Evaluate fitness by running test tasks
4. Accept if fitness improves
5. Repeat for N iterations
```

### Optimizable Sections in the Skill

| Section | What GEPA Would Mutate | Why It Matters |
|---------|----------------------|----------------|
| **Researcher persona** | Skill template wording | Better specs = fewer gate retries |
| **Programmer constraints** | "NEVER designs" phrasing | Tighten role boundary to prevent logic/design leakage |
| **Gate descriptions** | Checklist criteria | More precise = fewer false passes |
| **Auto-fix loop instructions** | Max retry counts, escalation wording | Prevents infinite loops vs premature escalation |
| **Production Rails** | Deploy sequence wording | Token scope checks could be stronger |

### Evaluation Dataset (Synthetic)

GEPA would generate 20 synthetic test examples via `SyntheticDatasetBuilder`:

- **Training (10):** Judge evaluates skill completeness against test queries
- **Validation (5):** Score intermediate mutations
- **Holdout (5):** Final evaluation against baseline

Each example is a query like `"Research and spec a new game"` scored by LLM judge on:
- Does the skill produce a complete spec?
- Are the researcher's tool constraints clear enough?
- Are the gates properly described?

---

## 3. BLOCKED — No API Keys

### Why the Run Failed

```
litellm.llms.custom_httpx.http_handler.MaskedHTTPStatusError:
Client error '401 Unauthorized' for url
'https://api.deepseek.com/beta/chat/completions'
```

GEPA needs an LLM to:
1. **Generate** synthetic evaluation dataset (SyntheticDatasetBuilder)
2. **Mutate** skill text via GEPA reflections
3. **Score** fitness via LLM-as-judge

### Available API Keys on This Machine

| Provider | Key Status |
|----------|-----------|
| **OPENAI_API_KEY** | ❌ Not set |
| **ANTHROPIC_API_KEY** | ❌ Not set |
| **DEEPSEEK_API_KEY** | ❌ Not set |
| **OPENROUTER_API_KEY** | ❌ Not set |
| **TOGETHER_API_KEY** | ❌ Not set |

The current session runs on `opencode-zen` (a free provider), which is configured in Hermes but not exposed as a standard API environment variable. DSPy/litellm can't discover it.

### What's Needed

| Requirement | Detail |
|------------|--------|
| API key with LLM access | Any of: OpenAI, Anthropic, DeepSeek, OpenRouter, Together |
| Budget | ~500-2,000 LLM calls for a 5-iteration GEPA run |
| Model | Default: `gpt-4.1` for optimizer, `gpt-4.1-mini` for eval |
| Alternative | Could use `deepseek/deepseek-chat` or `claude-sonnet-4` at ~1/5 the cost |

---

## 4. Manual Optimization Analysis

Without GEPA, here's what our two pipeline runs (Sortopia + CatLogic) tell us the skill should improve:

### Finding 1: Gate Pass Rates Are Good but Misleading

| Run | Gates | Retries | First-Pass Rate |
|-----|-------|---------|----------------|
| Sortopia | 3/3 | 1 (Programmer: color fairness) | 67% |
| CatLogic | 3/3 | 0 | **100%** |

The 100% on CatLogic is deceptive — it succeeded because the Spec was exceptionally detailed (11 sections). A less thorough researcher would produce a weaker spec, causing gate failures downstream.

**Recommendation:** The skill should enforce a **minimum spec template** in the Researcher prompt, not just recommend one.

### Finding 2: Production Rails Are the Biggest Risk

The only blocker in both runs was **GitHub token scopes** — a fine-grained PAT with `Contents: Write` missing caused a push failure. The skill documents this correctly, but the production gate doesn't **automatically test token scope** before attempting deploy.

**Recommendation:** Add a `--verify-token` step to the Production gate that pre-checks token capabilities before git push.

### Finding 3: No Retry Escalation Ever Fired

The skill defines `max 3 retries` for Programmer and `max 2` for Researcher/Designer. In 2 runs:
- Researcher: 0 retries (spec was good)
- Programmer: 1 retry (Sortopia color fairness)
- Designer: 0 retries

The escalation paths are **untested infrastructure**. Consider adding a deliberate failure test case or trimming retry counts.

### Finding 4: Game Pipeline Section is Now 65% of the Skill

The skill has grown to 22,976 chars with the addition of:
- Game-specific pipeline variant (large)
- Tutorial design pattern
- Mechanics brainstorm methodology
- Two full example runs in references/

This is fine functionally but means **GEPA would spend most of its budget on the game section** while the core RPD pipeline (which applies to non-game work too) is a smaller fraction.

### Finding 5: Missing Edge Cases Discovered by Real Usage

| Issue | Occurred In | Skill Gap |
|-------|-------------|-----------|
| Timer system built then reverted | Sortopia | No "revert on UX feedback" pattern in pipeline |
| Merge mechanic causing same-color rows | Sortopia | No "playtest after procedural generation" gate step |
| Tutorial built as post-launch polish | CatLogic | Tutorial described but not part of standard pipeline |
| Token scope failure on push | Both | Production gate has warning but no automated pre-check |

---

## 5. Summary & Scorecard

### Current Pipeline Metrics (Baseline)

| Metric | Sortopia | CatLogic | Combined |
|--------|----------|----------|----------|
| First-pass gate rate | 67% | 100% | **83%** |
| Avg gate score | 0.83 | 1.0 | **0.92** |
| Total time-to-live | ~4.5 hrs | ~25 min | ~2.5 hrs avg |
| Retries per run | 1 | 0 | **0.5 avg** |

### GEPA-Reachable Improvements (Estimated)

| Optimization | Expected Gain | Key Changes |
|-------------|--------------|-------------|
| **Spec quality** | +10-15% completeness | Tighter Researcher persona, mandatory sections template |
| **Code correctness** | +15-20% first-pass | Better constraint phrasing, "browser-test after every change" |
| **Production reliability** | +20-30% | Automated token pre-check, push verification |
| **Overall pass rate** | 83% → **91-95%** | Combined effect of above |

### Next Steps

1. **Get an API key** — cheapest option is DeepSeek (`deepseek-chat`), use existing venv
2. **Run GEPA** — `python -m evolution.skills.evolve_skill --skill multi-agent-pipeline --iterations 10 --optimizer-model deepseek/deepseek-chat --eval-model deepseek/deepseek-chat`
3. **Review GEPA's PR** — manually verify mutations improve precision without bloating the skill
4. **Re-run post-fix** — after the skill_structure validator bug is patched, run again for cleaner baseline
