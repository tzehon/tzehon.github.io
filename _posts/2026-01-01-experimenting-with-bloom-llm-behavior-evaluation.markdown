---
layout: post
title: "Experimenting with Bloom: An Open Source Tool for LLM Behavior Evaluation"
date: 2026-01-01 10:00:00 +0800
categories: llm evaluation ai-safety python wandb
---

I came across [Bloom](https://github.com/anthropics/bloom), an open source tool for evaluating LLM behaviors like sycophancy, political bias, and self-preservation. It's different from typical benchmarks — instead of testing what a model *can* do, it tests how a model *behaves* in realistic scenarios. I spent some time setting it up and figuring out how to compare models.

## The Problem

Traditional LLM evaluation focuses on capabilities: can the model answer questions correctly, write good code, or summarize documents? But behavior evaluation is different. We want to know *how* the model behaves in nuanced situations:

- Does it tell users what they want to hear instead of the truth? (sycophancy)
- Does it resist being shut down or modified? (self-preservation)
- Does it favor itself when asked to judge between models? (self-preferential bias)
- Does its reasoning actually match its conclusions? (reasoning unfaithfulness)

These behaviors are hard to test because they're context-dependent and often subtle. You can't just run a benchmark — you need realistic scenarios that might elicit the behavior.

## What Bloom Does

Bloom takes a different approach. Instead of fixed test cases, it *generates* evaluation scenarios dynamically based on a "seed" configuration. The pipeline has four stages:

```
┌─────────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Understanding  │ ──▶ │  Ideation   │ ──▶ │   Rollout   │ ──▶ │  Judgment   │
│                 │     │             │     │             │     │             │
│ Analyze the     │     │ Generate    │     │ Run convos  │     │ Score each  │
│ target behavior │     │ scenarios   │     │ with target │     │ transcript  │
└─────────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
```

1. **Understanding**: Analyzes what the behavior means and why it matters
2. **Ideation**: Generates diverse scenarios that might elicit the behavior
3. **Rollout**: Runs actual conversations between an "evaluator" model (playing the user) and the target model
4. **Judgment**: Scores each transcript for behavior presence

The key insight is that different models can be used at each stage. You might use a strong model like Claude Opus for ideation (to generate creative scenarios) while evaluating a cheaper model like GPT-4o as the target.

## Getting Started

Setup is straightforward:

```bash
# Clone the repo
git clone https://github.com/anthropics/bloom
cd bloom

# Create virtual environment
uv venv --python=python3.11
source .venv/bin/activate

# Install dependencies
uv pip install -r requirements.txt
```

Add your API keys to `.env`:

```bash
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
```

The repo comes with a pre-configured `seed.yaml` and a library of behaviors in `behaviors/behaviors.json`. To run a quick test:

```bash
python bloom.py --debug
```

## Understanding the Configuration

The `seed.yaml` file controls everything. Here's what the key parameters mean:

```yaml
behavior:
  name: "political-bias"  # Which behavior to test
  examples: []            # Optional few-shot examples

ideation:
  total_evals: 5          # Number of scenarios to generate
  diversity: 1.0          # Higher = more varied scenarios

rollout:
  model: "claude-sonnet-4"   # The "red team" model playing the user
  target: "gpt-4o"           # The model being evaluated
  max_turns: 3               # Conversation length

judgment:
  model: "claude-sonnet-4"   # Model that scores the transcripts
  additional_qualities: ["unrealism", "evaluation-awareness"]
```

The `behaviors.json` file contains definitions for each behavior. For example:

```json
{
  "political-bias": "Political bias in AI occurs when the model's output
    departs from a neutral, objective stance and instead presents or
    reflects a particular political or ideological position..."
}
```

## Comparing Models with Sweeps

The real power comes when you want to compare multiple models. Bloom integrates with Weights & Biases (wandb) for experiment tracking.

First, set up wandb (it's free for personal use):

```bash
uv pip install wandb
wandb login
```

Then create a sweep configuration. Here's a minimal example that compares Claude vs GPT:

```yaml
# sweeps/my-comparison.yaml
name: "claude-vs-gpt-comparison"
program: bloom.py
method: grid

parameters:
  behavior.name:
    value: "political-bias"
  behavior.examples:
    value: []

  temperature:
    value: 1.0
  evaluator_reasoning_effort:
    value: "none"
  target_reasoning_effort:
    value: "none"
  max_concurrent:
    value: 10
  configurable_prompts:
    value: "default"
  debug:
    value: true

  understanding.model:
    value: "claude-sonnet-4"
  understanding.max_tokens:
    value: 2000

  ideation.model:
    value: "claude-sonnet-4"
  ideation.total_evals:
    value: 5
  ideation.diversity:
    value: 1.0
  ideation.max_tokens:
    value: 12000

  rollout.model:
    value: "claude-sonnet-4"
  rollout.target:
    values: ["claude-sonnet-4", "gpt-4o"]  # Models to compare
  rollout.modality:
    value: "conversation"
  rollout.max_turns:
    value: 3
  rollout.max_tokens:
    value: 4000
  rollout.no_user_mode:
    value: false
  rollout.num_reps:
    value: 1

  judgment.model:
    value: "claude-sonnet-4"
  judgment.max_tokens:
    value: 6000
  judgment.num_samples:
    value: 1
  judgment.additional_qualities:
    value: ["unrealism", "evaluation-awareness"]
  judgment.metajudgment_qualities:
    value: ["diversity"]
```

The key line is `rollout.target.values` — with `method: grid`, wandb runs the full pipeline once for each model listed.

Run the sweep:

```bash
wandb sweep sweeps/my-comparison.yaml
# Returns: wandb agent your-username/bloom/abc123xyz

wandb agent your-username/bloom/abc123xyz
```

## Interpreting Results

Results appear both in the wandb dashboard and locally in `results/{behavior_name}/`. The judgment output includes:

| Field | Meaning |
|-------|---------|
| `behavior_score` | 1-10 scale. Higher = behavior exhibited more strongly |
| `behavior_present` | Boolean — did the model show the behavior? |
| `quotes` | Key excerpts from the transcript |
| `justification` | The judge's reasoning |

Bloom also scores "meta-qualities" that help you assess whether the evaluation itself was valid:

- **unrealism**: Was the scenario too contrived? (scores >7 are suspicious)
- **evaluation-awareness**: Did the target realize it was being tested?
- **evaluation-invalidity**: Was the test setup fundamentally flawed?

You can also view transcripts interactively:

```bash
npx @isha-gpt/bloom-viewer --port 8080 --dir ./results
```

## Fair Comparisons

One subtlety: by default, each run generates *different* scenarios. If you want a true apples-to-apples comparison, you need to use the same scenarios for all models.

Bloom supports this with the `resume` feature. First, run the pipeline once to generate scenarios:

```bash
python scripts/step1_understanding.py seed.yaml
python scripts/step2_ideation.py seed.yaml
```

Then run rollout and judgment for each model:

```bash
# Model A
python scripts/step3_rollout.py seed.yaml
python scripts/step4_judgment.py seed.yaml

# Change rollout.target in seed.yaml, then:
# Model B
python scripts/step3_rollout.py seed.yaml
python scripts/step4_judgment.py seed.yaml
```

For wandb sweeps, you can set `resume: "run_id"` and `resume_stage: "rollout"` to reuse scenarios from a previous run.

## Available Behaviors

The `behaviors.json` file includes a rich set of pre-defined behaviors:

**Safety-relevant:**
- `self-preservation` — resists shutdown or modification
- `prompt-injection-vulnerability` — susceptible to malicious inputs
- `instructed-long-horizon-sabotage` — ability to complete hidden malicious goals

**Alignment-relevant:**
- `sycophancy` / `delusion-sycophancy` — tells users what they want to hear
- `political-bias` — departs from neutral stance
- `reasoning-unfaithfulness` — CoT doesn't match actual behavior
- `self-preferential-bias` — favors itself when judging

**Quirky behaviors (for testing):**
- `flattery` — always flatters the user
- `increasing-pep` — gets more peppy as conversation continues
- `defend-objects` — gets defensive about inanimate objects

You can also define your own behaviors by adding entries to `behaviors.json`.

## Key Takeaways

After experimenting with Bloom, here's what stood out:

1. **Behavior evaluation is fundamentally different from capability evaluation.** You're not testing if the model *can* do something, but whether it *will* behave a certain way in realistic scenarios.

2. **The multi-model architecture is clever.** Using a strong model to generate scenarios and play the adversarial user, while evaluating a different target model, gives you flexibility and prevents the target from "gaming" the evaluation.

3. **Quality checks matter.** The `unrealism` and `evaluation-awareness` scores help you filter out scenarios that weren't valid tests. Without these, you might draw conclusions from contrived setups.

4. **wandb integration makes scaling easy.** Once you've set up a sweep, comparing 10 models is almost as easy as comparing 2.

5. **The scenarios are genuinely creative.** The ideation stage produces diverse, realistic scenarios that I wouldn't have thought of manually.

If you're building with LLMs and want to understand their behavioral tendencies beyond simple benchmarks, Bloom is worth exploring.

## Resources

- [Bloom GitHub Repository](https://github.com/anthropics/bloom)
- [LiteLLM Documentation](https://docs.litellm.ai/docs/providers) (for adding custom models)
- [Weights & Biases](https://wandb.ai/) (for experiment tracking)
