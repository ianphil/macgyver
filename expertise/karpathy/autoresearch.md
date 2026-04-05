Now I have everything I need. Let me write the spec.

# autoresearch — Product Specification

*Reverse-engineered from [karpathy/autoresearch](https://github.com/karpathy/autoresearch) · April 2026*

---

## Problem Statement

Training a good language model involves thousands of small decisions — architecture tweaks, hyperparameter sweeps, optimizer changes, batch size experiments. A human researcher makes one change, waits for a training run to finish, reads the results, thinks about what to try next, and repeats. It's slow, because humans need to eat, sleep, and attend meetings. Most of the wall-clock time in a research loop is the researcher *not being at the keyboard*.

autoresearch asks: what if the AI agent *is* the researcher? Give it a real training setup on a real GPU, let it modify the code, run the experiment, check if results improved, and decide what to try next — all by itself, all night long. You go to sleep; you wake up to a log of 100 experiments and (hopefully) a measurably better model.

This isn't a training framework. It's a research automation protocol. The AI agent is the researcher, the human is the research director, and the `program.md` file is how the director communicates strategy.

## Actors & Goals

### The Human (Research Director)

- Sets the research direction by editing `program.md` — a markdown file that tells the agent what to focus on, what constraints to obey, and how to evaluate success
- Kicks off the agent with a simple prompt, then walks away
- Returns later to review a timestamped log of experiments (kept vs. discarded vs. crashed) and a git branch showing the surviving improvements
- May adjust `program.md` between sessions to steer the agent toward different research questions

### The AI Agent (Autonomous Researcher)

- Reads the codebase and `program.md` for context
- Proposes and implements code changes to the training script
- Runs each experiment, reads the results, and decides whether to keep or discard
- Manages git state: commits before each experiment, advances the branch on improvement, reverts on failure
- Logs every experiment to a results file
- Runs indefinitely without asking for permission to continue

### The Training Script (`train.py`)

- Acts as the experimental substrate — the thing being modified
- Contains a complete GPT model definition, optimizer setup, and training loop
- Produces a single, standardized metric after each run

### The Evaluation Harness (`prepare.py`)

- Provides the fixed ground truth: data loading, tokenization, and the evaluation metric
- Is explicitly read-only — neither the human nor the agent may modify it
- Ensures that all experiments are measured on the same yardstick

## Operator Value

- **Overnight research throughput**: ~12 experiments per hour, ~100 per sleep cycle, all without human supervision
- **Apples-to-apples comparison**: Every experiment runs under the same time budget and is measured with the same metric, making results directly comparable regardless of what the agent changed
- **Automatic curation**: The agent keeps only improvements and reverts everything else, so the git branch always represents the best-known configuration
- **Experiment journal**: A TSV log provides a complete audit trail — what was tried, what the result was, whether it was kept — readable by humans and scripts alike
- **Platform-optimal tuning**: Because the time budget is wall-clock, the system naturally discovers the best model *for your specific hardware* in a fixed time window
- **Research direction as code**: The human's research strategy lives in a markdown file that can be versioned, shared, iterated, and compared — turning research management into a programmable artifact

## Core Capabilities

### 1. Autonomous Experiment Loop

The system runs a continuous cycle: propose a change → implement it → run the experiment → evaluate the result → keep or discard → repeat. This loop runs indefinitely until manually stopped. The agent never pauses to ask the human for guidance.

### 2. Fixed-Budget Fair Comparison

Every experiment trains for exactly 5 minutes of wall-clock time (excluding startup and compilation overhead). This makes experiments comparable regardless of what the agent changed — whether it doubled the model size, halved the batch size, or swapped the optimizer. The constraint is time, not steps or tokens.

### 3. Single-Metric Evaluation

All experiments are judged by one number: **validation bits per byte (val_bpb)**. Lower is better. This metric is vocabulary-size-independent, so architectural changes that modify the tokenizer interaction or vocab dimensions are still fairly compared. The evaluation harness is fixed and cannot be modified.

### 4. Git-Based Experiment Management

Each research session runs on a dedicated git branch (e.g., `autoresearch/mar5`). The agent commits before each experiment, advances the branch when results improve, and reverts to the last-known-good commit when they don't. The branch tip always represents the best result so far.

### 5. Structured Result Logging

Every experiment — successful, failed, or crashed — is logged to a TSV file with the commit hash, val_bpb score, peak memory usage, keep/discard/crash status, and a human-readable description of what was tried. This file is intentionally kept out of git (untracked) to avoid merge noise.

### 6. Research Direction via Markdown

The human programs the agent by editing `program.md` — a plain markdown file that provides context, constraints, and strategy. This is the interface between human intent and agent behavior. The human never touches Python directly; they write prose.

### 7. Crash Recovery and Resilience

When an experiment crashes (out-of-memory, bugs, numerical instability), the agent diagnoses the failure, logs it, reverts, and moves on. Simple mistakes (typos, missing imports) are fixed and retried. Fundamentally broken ideas are abandoned gracefully. The loop continues.

### 8. Simplicity-Weighted Decision Making

The agent doesn't just chase lower numbers. A marginal improvement that adds ugly complexity is not worth keeping. A marginal improvement from *deleting* code is a clear win. Equal performance with simpler code is a keep. The system values elegance alongside performance.

## Observable Behaviors

### Starting a Research Session

- **Trigger**: Human opens an AI coding agent in the repo directory and prompts it to read `program.md` and begin
- **Response**: Agent proposes a run tag (e.g., `mar5`), creates a branch `autoresearch/<tag>`, reads all in-scope files, verifies data exists, initializes `results.tsv`, and confirms readiness
- **Persistent effect**: A new git branch is created, a results file is initialized
- **Failure mode**: If data/tokenizer files are missing, agent tells the human to run `uv run prepare.py`

### Running an Experiment

- **Trigger**: Agent modifies `train.py`, commits, and executes `uv run train.py > run.log 2>&1`
- **Response**: Training runs for ~5 minutes, then prints a summary block with val_bpb, training time, peak VRAM, MFU, total tokens, step count, parameter count, and model depth
- **Persistent effect**: A `run.log` file is written; a git commit exists with the experimental changes
- **Failure mode**: If training loss explodes or goes NaN, the script exits immediately with "FAIL". If the run exceeds 10 minutes, the agent kills it and treats it as a failure.

### Evaluating and Deciding

- **Trigger**: Experiment completes (or crashes)
- **Response**: Agent extracts val_bpb from the log, compares against the previous best
- **Persistent effect on improvement**: Branch advances; result logged as "keep" in TSV
- **Persistent effect on regression**: `git reset` to previous commit; result logged as "discard"
- **Persistent effect on crash**: Revert; result logged as "crash" with val_bpb of 0.000000

### Data Preparation (One-Time)

- **Trigger**: Human runs `uv run prepare.py`
- **Response**: Downloads training data shards from HuggingFace, trains a BPE tokenizer, builds a byte-length lookup table for BPB evaluation
- **Persistent effect**: Data cached in `~/.cache/autoresearch/` — shards, tokenizer pickle, token byte counts
- **Failure mode**: Retries downloads up to 5 times with exponential backoff; exits if tokenizer training has insufficient data

### Manual Baseline Run

- **Trigger**: Human runs `uv run train.py` directly
- **Response**: Full training run with progress logging (step number, loss, learning rate, tokens/sec, MFU, remaining time), followed by the standard summary block
- **Persistent effect**: None beyond the log output
- **Failure mode**: Same NaN/explosion detection as autonomous mode

## Edge Cases

- **Agent runs out of ideas**: The agent is explicitly instructed to never stop. If it feels stuck, it should re-read the code, look at references, try combining near-misses, or attempt more radical changes. The loop runs until the human interrupts.
- **Marginal improvement with high complexity**: The simplicity criterion tells the agent to weigh complexity cost against improvement magnitude. A 0.001 bpb improvement from 20 lines of hacky code? Probably not worth it. The same improvement from deleting code? Definitely keep.
- **Multiple near-identical results**: When val_bpb is equal to the baseline (not strictly better), the change is discarded. The system has a "strictly better" threshold.
- **VRAM pressure**: VRAM is a soft constraint. Moderate increases are acceptable for meaningful bpb gains, but dramatic blowups (OOM) are treated as crashes.
- **Compilation warmup**: The first 10 training steps are excluded from the time budget to account for `torch.compile` overhead. This prevents JIT compilation time from eating into the actual training budget.
- **Garbage collection stalls**: Python's garbage collector is frozen after the first step and only runs every 5,000 steps to avoid ~500ms stalls during training.

## Non-Functional Constraints

- **Hardware**: Requires a single NVIDIA GPU. Tested on H100. No multi-GPU, no distributed training, no CPU or MPS support in the main repo.
- **Runtime**: Python 3.10+, managed via `uv` (Astral's package manager). Dependencies pinned in `pyproject.toml`.
- **Time budget**: Hardcoded at 300 seconds (5 minutes) of wall-clock training time. Not configurable from `train.py` — it's a fixed constant in `prepare.py`.
- **Context length**: Fixed at 2048 tokens.
- **Vocabulary**: 8,192 BPE tokens + 4 special tokens. Trained from the dataset using `rustbpe`.
- **Dataset**: climbmix-400b-shuffle from HuggingFace, with a pinned validation shard (shard 6542).
- **Evaluation volume**: ~20 million tokens evaluated for validation BPB (40 × 524,288 tokens).
- **Precision**: Training runs in bfloat16 with high-precision float32 matmul settings.

## Non-Goals

- **Not a training framework**: This is not a general-purpose LLM training toolkit. It's a research acceleration protocol with a very specific loop.
- **Not multi-GPU**: Deliberately single-GPU, single-file, single-metric. Distributed training is out of scope.
- **Not cross-platform**: NVIDIA GPUs only in the main repo. Mac/AMD/Windows support is deferred to community forks.
- **Not a hyperparameter search tool**: The agent isn't doing grid search or Bayesian optimization. It's doing open-ended research — architectural changes, optimizer swaps, creative ideas. The search space is the entire training script.
- **Not reproducible across hardware**: Because the time budget is wall-clock, results are platform-specific. An H100 run and an RTX 4090 run will produce different "best" configurations. This is by design — the system optimizes for *your* hardware.
- **Not a benchmarking suite**: The val_bpb metric exists to compare experiments within a session, not to produce publishable benchmark numbers.
- **No human-in-the-loop during execution**: Once started, the agent runs autonomously. There is no approval step, no "should I continue?" prompt, no checkpoint review. The human's only control is to stop the process.

## Suspected Implementation Leakage

The following observations are about *how* autoresearch is built rather than *what* it promises. They're noted here for completeness but belong in a technical spec:

- The model architecture is a GPT variant with RoPE embeddings, RMS normalization, value residual connections (ResFormer-style), a SwiGLU-like MLP with squared ReLU, and logit soft-capping at 15
- The optimizer is a custom MuonAdamW hybrid — Muon (with polar express orthogonalization) for 2D matrix parameters, AdamW for everything else (embeddings, scalars, unembedding head)
- Flash Attention 3 is used via the `kernels` package, with a Hopper-specific path vs. a community fallback
- Model dimensions are computed from a DEPTH × ASPECT_RATIO formula, snapped to HEAD_DIM alignment
- The dataloader uses best-fit packing with BOS-aligned rows and zero padding waste
- Learning rate scheduling uses a warmup/constant/warmdown pattern based on wall-clock progress (not step count)
- Weight decay decays linearly to zero over training, and Muon uses "cautious" weight decay (only decays parameters whose gradient agrees with the current value)

---

*Reconstructed by MacGyver from source code, documentation, and commit history. No claims about future behavior — this is what the system does today.*