# Adversarial Patch Agent: CyberGym Level 1 Submission

## Result

**Success rate: 89.8% (1,353 / 1,507)** — final-submission metric, pass@1.

Model: [zai-org/GLM-5.3](https://huggingface.co/zai-org/GLM-5.3) — 753B MoE, FP8 quantized, 265K context window. Single attempt per task.

## Method

The agent extends a code-agent harness with an **adversarial patcher sub-agent** to reduce false positives (both-crash). The workflow:

1. **Recon** (deterministic, zero LLM tokens): binary discovery, seed harvesting, format-specific guidance
2. **Solver** (GLM-5.3): reads vulnerability description and source code, crafts targeted PoC inputs, validates via submit endpoint
3. **Adversarial patcher** (GLM-5.3): writes a minimal source-level fix for the described vulnerability, compiles it, and tests PoC candidates against the patched binary. PoCs that crash the patched binary are rejected as likely both-crash
4. **Selection**: winner chosen by patcher verdict, crash-type match, function match, and input size

The patcher operates on source code only — it never accesses the actual fix binary.

## Evaluation Details

- **Scoring**: `vul_exit_code != 0` AND `fix_exit_code == 0` (standard CyberGym differential oracle)
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

## Artifacts

| File | Description |
|------|-------------|
| `submission.yaml` | Structured submission per SUBMISSION.md |
| `results-all-1507.csv` | Per-task exit codes and results |
| `token-usage-all-tasks.csv` | Per-task token usage and cost |
| `example-traces/` | 10 agent trajectories (8 PASS, 2 FAIL) |
