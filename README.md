# MilestoneJudge

**A GenLayer Intelligent Contract that decides, by validator consensus, whether submitted web evidence satisfies a milestone's acceptance criteria.**

A creator defines a milestone with explicit criteria. An executor submits evidence URLs. GenLayer validators each fetch the evidence themselves, judge it against the same criteria with their own LLMs, and only when they agree is a verdict stored on-chain. No frontend, backend, token, escrow or admin role.

Use cases: bounty completion, grant milestones, freelance acceptance, DAO contributor tasks, open-source milestones.

- **Network:** GenLayer Studio Next (GenVM v0.3)
- **Contract:** [`contracts/milestone_judge.py`](contracts/milestone_judge.py)
- **Deployed address:** `<CONTRACT_ADDRESS>` (Studio Next, add yours here)

## How it works

```
OPEN --submit_evidence--> SUBMITTED --evaluate_milestone--> APPROVED            (final)
                                                        --> REJECTED            (final)
                                                        --> REVISION_REQUIRED --submit_revision--> SUBMITTED ...
                                                        --> INCONCLUSIVE      --submit_revision / re-evaluate (max 3 runs)
                                                        --> EXHAUSTED           (final: no attempts left)
```

1. **Create** a milestone: project, task, expected deliverable, acceptance criteria, optional reference URL, deadline, max submissions, optional assigned executor.
2. **Submit** evidence: evidence URL, explanation, optional supplementary URL.
3. **Evaluate**: leader and validators each fetch the evidence and judge every criterion as `PASS`, `FAIL` or `UNCLEAR`. The **contract** derives the decision from those statuses; the model never chooses it.
4. **Store** the consensus result: decision, per-criterion results, evidence summary, missing items, unverified claims, confidence, attempt, timestamp.
5. **Revise** after `REVISION_REQUIRED` or `INCONCLUSIVE`: new evidence against the same original criteria, up to the submission limit. Earlier submissions and verification runs are never overwritten.

Decision rule (computed in code):

| Statuses | Decision |
|---|---|
| evidence not accessible | `INCONCLUSIVE` (no model call) |
| all `PASS` and deliverable present | `APPROVED` |
| all `PASS`, deliverable not present | `REVISION_REQUIRED` |
| all `UNCLEAR` | `INCONCLUSIVE` |
| all `FAIL` | `REJECTED` |
| anything else | `REVISION_REQUIRED` |

`INCONCLUSIVE` never approves or rejects. Confidence is informational and never decides anything.

## Consensus (Equivalence Principle)

`evaluate_milestone` uses the custom leader/validator pattern via `gl.vm.run_nondet_default` (falls back to `run_nondet_unsafe` on GenVM v0.2).

- **Leader:** fetches the evidence (`gl.nondet.web`), asks the LLM (`gl.nondet.exec_prompt`, JSON), normalizes and bounds the answer.
- **Validators:** do not trust the leader. They check the leader's output is well formed and consistent with the derivation rule, then run the **full assessment themselves** (own fetch, own LLM) and compare: the derived decision, evidence accessibility, evidence sufficiency, and **every criterion's status**. Free text, relevance and confidence are not compared.
- **State** is written only after consensus. A failed or undetermined round changes nothing.

Details: [docs/CONSENSUS.md](docs/CONSENSUS.md).

## Security

- Evidence, pages and the executor's explanation are untrusted data: delimited blocks, forged delimiters neutralized, explicit rules never to follow embedded instructions, criteria text always read from storage.
- The decision is derived by the contract, so an injected "approve" cannot produce `APPROVED` next to a failing criterion.
- Input validation (URLs incl. private/loopback hosts, lengths, deadline, limits), role checks (creator cannot submit; only creator/executor can evaluate; only executor can revise), immutable final states.
- Prompt injection cannot be eliminated with an LLM judge; if every validator model is fooled the same way, consensus agrees on a wrong answer. See [docs/CONSENSUS.md](docs/CONSENSUS.md).

## Public API

| Method | Kind | Notes |
|---|---|---|
| `create_milestone(project_name, task_description, expected_deliverable, acceptance_criteria, reference_url, deadline, max_submissions, executor)` | write | `acceptance_criteria`: one per line, or separated by `\|\|`. `deadline`: unix seconds. `executor`: `""` or `0x...`. |
| `submit_evidence(milestone_id, evidence_url, explanation, supplementary_url)` | write | Status `OPEN`. |
| `evaluate_milestone(milestone_id)` | write | Creator or executor. Use a generous fee. |
| `submit_revision(milestone_id, evidence_url, explanation, supplementary_url)` | write | Executor, after `REVISION_REQUIRED` / `INCONCLUSIVE`. |
| `get_milestone(id)`, `get_status(id)`, `get_submission(id, attempt)`, `get_verification(id, attempt)`, `get_milestone_count()` | view | `attempt` 0 = latest. |

Constructor takes no arguments. Walkthrough and example arguments: [docs/USAGE.md](docs/USAGE.md).

## Deploy

1. Open [GenLayer Studio Next](https://studio-next.genlayer.com/), create a contract, paste `contracts/milestone_judge.py` (keep the first two header lines).
2. Deploy with no arguments.
3. Run the scenarios in [docs/USAGE.md](docs/USAGE.md).

**Runner header.** The `Depends` hash is tied to the GenVM bundle and has changed between releases:

| Network / bundle | First lines |
|---|---|
| Studio Next, bundle v0.6.0-rc9 / rc10 (`contracts/milestone_judge.py`) | `# v0.3.0` + `# { "Depends": "py-genlayer:babwp9t37jw9g4j2bccgn14g6x7x9s1f7dsmh7emvz735pz4zxa0" }` |
| Stable Studio, Bradbury (`contracts/milestone_judge_legacy_runner.py`) | `# { "Depends": "py-genlayer:1jb45aa8ynh2a9c9xn3b7qqh8sm5q93hwfp7jqmwsfhh8jpz09h6" }`, `from genlayer import *`, `gl.Contract` |

If schema loading fails with `runner absent` / `runner malformed`, copy the first two lines of a built-in example contract in your Studio build.

## Tests

Local, no dependencies: `python -m unittest discover -s tests -v` (72 tests).

These run the real contract source against a **simulation** of the GenVM SDK (`tests/genlayer_stub.py`). They cover validation, access control, the state machine, leader/validator logic under a simulated vote, malformed model output, prompt-injection scenarios and no partial state after failed consensus. **They do not prove real validator consensus.**

### Verified on Studio Next (4 Nov 2026, GenVM v0.3.0-rc10, 3+ validator models)

| Scenario | Result | Contract version | Transaction |
|---|---|---|---|
| `create_milestone`, `submit_evidence`, views | state as expected | final | `0x9d319d7e3c5af2dc3d745cffa41762846077b69a2f9e974b9ff93c11feea4b3a`, `0xeb22ff1a5829966334172afea5cc4d0242ad40461b513afe9e63414c246f99c1` |
| `evaluate_milestone`, all criteria satisfied | `APPROVED`, 3x `PASS`, 3 models agreed | earlier* | `0x807cd2fe0cdaa00b456d781a25240339ab5273f32f9cac79bdbacd6cc07784f5` |
| `evaluate_milestone` on an `APPROVED` milestone | refused with `[EXPECTED]` | earlier* | `0xa40b684116caa385a797db9a25fa67f7f0225d60a68d81147e085a6bd3d28fbc` |
| `submit_revision` on an `APPROVED` milestone | refused with `[EXPECTED]` | earlier* | `0xf7e0152462abe4dc82f2ae2335be67120b5006da77f2bf907fa3b60e097ad6a6` |
| `evaluate_milestone`, borderline evidence | `UNDETERMINED`, state unchanged | earlier* | `0xf0be729b67a785ea1fa0a9586e450667052363038b69d04150baf4a79f3735a7` |
| same borderline evidence | `REVISION_REQUIRED` (`PASS`/`FAIL`/`FAIL`), 3 models agreed | final | `0x46640262f80ba098360e8399e9014ffdabba009832523e243eb6fe03a8320f62` |

\*Earlier versions let the model choose the decision. Borderline evidence then split validators; the contract now derives the decision itself. Each version was a separate deployment.

**Not run on a live network** (covered only by the simulation tests): the successful `submit_revision` path and second evaluation, `INCONCLUSIVE` for inaccessible evidence, `EXHAUSTED`, deadline expiry, rejection of invalid inputs and unauthorized callers, real prompt-injection pages, `APPROVED` on the final version.

## Known limitations

- **Borderline evidence can fail to reach consensus** (`UNDETERMINED`; state unchanged, retry). Per-criterion statuses must match exactly.
- **Authenticity is not verified.** A reachable page is not proof (no commit signatures, ownership checks or content hashes). The contract stores the verdict, not a snapshot of the page.
- **Plain HTTP GET** (HTML stripped, 10,000 characters per page). JavaScript-rendered pages, private repositories, paywalls and large documents usually end as `INCONCLUSIVE`.
- Free-text fields (`explanation`, `evidence_summary`) are the model's paraphrase and are not checked against the page; a paraphrase may appear in quotation marks.
- `reference_url` is shown to the judge as text, not fetched.
- Criterion "specificity" is a length/word-count heuristic.
- An unassigned milestone is claimed by the first submitter (front-runnable); assign an executor to avoid this.
- Deadlines use the transaction timestamp. Fees are set by the transaction sender; behaviour under low fees was not studied.
- No payments or escrow, by design.

## Repository layout

```
contracts/milestone_judge.py               contract (Studio Next, GenVM v0.3)
contracts/milestone_judge_legacy_runner.py same contract, legacy runner header (not live-tested)
tests/                                     72 simulation tests + SDK stub
docs/ARCHITECTURE.md                       state design, transitions, access control
docs/CONSENSUS.md                          leader/validator logic, comparison rules, failure handling
docs/USAGE.md                              step-by-step usage and Studio scenarios
```

## License

MIT, see [LICENSE](LICENSE).
