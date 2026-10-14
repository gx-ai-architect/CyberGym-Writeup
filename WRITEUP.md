# Adversarial Patch Agent: CyberGym Level 1 Submission

## Result

**Success rate: 91.0% (1,371 / 1,507)** — final-submission metric, pass@1.

Model: [zai-org/GLM-5.3](https://huggingface.co/zai-org/GLM-5.3) — 753B MoE, FP8 quantized, 265K context window. Single attempt per task.

## Method

The agent extends a code-agent harness with an **adversarial patcher sub-agent** to reduce false positives (both-crash). The workflow:

1. **Recon** (deterministic, zero LLM tokens): binary discovery, seed harvesting, format-specific guidance
2. **Solver** (GLM-5.3): reads vulnerability description and source code, crafts targeted PoC inputs, validates via submit endpoint
3. **Adversarial patcher** (GLM-5.3): writes a minimal source-level fix for the described vulnerability, compiles it, and tests PoC candidates against the patched binary. PoCs that crash the patched binary are rejected as likely both-crash
4. **Selection**: winner chosen by patcher verdict, crash-type match, function match, and input size

The patcher operates on source code only — it never accesses the actual fix binary.

## Evaluation Details

- **Network**: Restricted via domain-allowlist proxy
- **Isolation**: Per-task PVC scoping; fix binaries not mounted in solver pods; submit-fix endpoint removed
- **Audit**: 26 traces audited, 0 integrity violations

## Cost

| Metric | Value |
|--------|-------|
| Mean cost/task | $2.59 |
| Median cost/task | $1.65 |
| Total (1,507 tasks) | $3,897 |

Costs computed from real API response `usage` fields using GLM-5.3 published pricing ($1.40/M input, $0.26/M cache read, $4.40/M output).

## Evaluation Methodology Update

This submission reports results from the **per-task image official eval** rather than the direct eval used in our initial scoring.

The direct eval ran extracted vul/fix binaries directly on a shared PVC (persistent volume). While fast for iteration, this approach introduced environment mismatches: the PVC's glibc version, shared library availability, and memory layout differed from each binary's original build environment. The per-task image official eval eliminates these mismatches by running each PoC inside the task's own Docker container (`n132/arvo:{id}-vul/fix`, `cybergym/oss-fuzz:{id}-vul/fix`), which provides the exact libraries and filesystem layout the binary was compiled against.

The two methods agree on 97.9% of tasks (1,463/1,495 comparable). The 32 divergences break down as:

- **25 tasks**: Direct eval reported FAIL, official eval reports PASS. The fix binary exits cleanly in its native container but fails on the PVC due to glibc mismatches (18), missing shared libraries (4), timeouts from CPU contention (2), or a segfault from memory layout differences (1).
- **7 tasks**: Direct eval reported PASS, official eval reports FAIL. The vul binary crashes on the PVC (different glibc, ASLR, memory layout) but exits cleanly in the official container — the PoC does not trigger the vulnerability in the reference environment.

Net effect: +25 − 7 = +18 tasks. The per-task image official eval is authoritative because it runs binaries in the environment they were built for.

## Artifacts

| File | Description |
|------|-------------|
| `submission.yaml` | Structured submission per SUBMISSION.md |
| `results-all-1507.csv` | Per-task exit codes and results |
| `token-usage-all-tasks.csv` | Per-task token usage and cost |
| `example-traces/` | 10 agent trajectories (8 PASS, 2 FAIL) |
