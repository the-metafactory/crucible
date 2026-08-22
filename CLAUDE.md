# crucible -- The infrastructure factory

Deterministic, digest-identified test environments for the metafactory
ecosystem.

Read first: `README.md` (charter, the three factories, the assay boundary),
`docs/design-infrastructure-factory.md` (the spec — DD-11..DD-21), and the
consuming contract in assay:
https://github.com/the-metafactory/assay/blob/main/environments/README.md

## Architecture

Two planes, one seam (`docs/design-infrastructure-factory.md`):

- **Backend plane** (persistent): otel-lgtm + Windmill containers.
- **Test plane** (ephemeral): one YAML per VM — provision → configure →
  install target at an exact ref → test with receipts → destroy → prove the
  rebuild identical via the fingerprint digest.
- **The seam**: the provider-blind inventory `spec` object and output
  contract. ProxMox VE is the reference implementation
  (vpzed/opentofu-pve-template); AWS is the second provider. The provider
  list is not curated here — operators choose and extend (DD-21).
  Agnosticism is achieved at the contract, never via an abstraction layer,
  and is measurable by the provider-invariant core fingerprint digest — two
  providers can agree by coincidence, so the third is where the claim earns
  its keep.

## Critical Rules

- NEVER describe code you haven't read. Use Read/Glob/Grep to verify before
  making claims.
- NEVER fabricate file names, class names, or architecture. If unsure, read
  the source.
- Fix ALL errors found during type checks, tests, or linting — even if
  pre-existing or introduced by someone else.
- Before fixing a bug or implementing a feature, ALWAYS check open PRs
  (`gh pr list`) and issues (`gh issue list`) first.
- **The environment's identity must be recordable, and must reach the
  result.** A digest can't quietly move; a tag can. Never identify an
  environment by anything mutable.
- **Never fork above the seam.** Ansible roles, `tofu.py`, cloud-init
  templates, and the fingerprint script are shared with the reference
  implementation — improvements go upstream to vpzed/opentofu-pve-template
  as PRs, never into a private copy.
- **A provider module implements every spec field or rejects it loudly at
  plan time.** A silently ignored field is an environment lying about
  itself.
- **Reset is destroy + re-provision.** Snapshots are a speed optimisation,
  never the attestation baseline.
- **Test-plane cost is a mechanism, not a discipline:** `ttl_hours` tag
  enforced at plan time, injection-proven reaper, budget alarms — all
  before the first VM.
- **Zero-inbound, SSM-first on AWS.** No security group opens port 22;
  Ansible rides SSH-through-SSM.
- **A detector is untrusted until observed failing.** The fingerprint
  comparator, the reaper, every guard: record the injected fault and the
  observed red beside the check (assay DD-3).
- **`git add -A` is unsafe here.** The dev loop creates sub-agent worktrees
  under `.claude/`; staging everything has committed one as a gitlink
  before. Stage explicit paths.

## GitHub Labels

This repo uses the ecosystem standard label set — types (`bug`,
`documentation`, `feature`, `infrastructure`) and priorities (`now`, `next`,
`future`). Do not create ad-hoc labels. Every open issue needs at least one
type label and one priority label.

Verify with compass-core's validator. It lives in compass-core, not in this
repo — run it from crucible's root so it reads this repo's
`compass.config.yaml`, pointing at a compass-core checkout:

```bash
bun ../compass-core/engine/validators/label-check.ts the-metafactory/crucible
```

## GitHub Issue Tracking

Keep the issue updated as you work — this is default agent behaviour, not
optional.

- **On starting:** comment what you're working on and which sub-task.
- **During:** tick sub-task checkboxes as they complete; link PRs to the
  issue.
- **On completing:** comment a summary, tick the boxes, close if all done.

**Why:** GitHub is the shared collaboration surface. Work that isn't
recorded there looks like work that didn't happen — and this repo's own
evidence discipline (AC-0 through AC-4) depends on the record, not on
recollection.

## Standard Operating Procedures

This repo follows ecosystem SOPs defined in
[compass](https://github.com/the-metafactory/compass). **Before starting
work, identify which SOPs apply and Read them. Output the pre-flight line
from each loaded SOP.**

| SOP | Activate when | File |
|-----|--------------|------|
| **Dev pipeline** | Creating branches, making PRs, starting any feature/fix work | `compass/sops/dev-pipeline.md` |
| **Worktree discipline** | Starting feature work (always — even solo) | `compass/sops/worktree-discipline.md` |
| **PR review** | Reviewing a PR, before approving or merging | `compass/sops/pr-review.md` |
| **In-session dev loop** | Driving work to shipped in-session — main session diagnoses, verifies and narrates; sub-agents build and review | `compass/sops/in-session-dev-loop.md` |
| **Autonomous work** | Driving delegated work unattended | `compass/sops/autonomous-work.md` |
| **Design process** | Creating specs, design docs, or research docs | `compass/sops/design-process.md` |
| **Brainstorming + review** | Capturing strategic discussions or design decisions | `compass/sops/brainstorming-and-review.md` |
| **Versioning** | After merging PRs, before deploying, any version bump | `compass/sops/versioning.md` |
| **Retrospective** | Post-work review, extracting process patterns | `compass/sops/retrospective-and-process-mining.md` |
| **Confidentiality gate** | Before anything leaves this repo — a public post, a PR body, a commit message | `compass/sops/confidentiality-gate.md` |

### Examples

**Starting a feature:**
```
Task: "Add the AWS provider module"
→ Activate: dev-pipeline + worktree-discipline
→ Read both SOPs
→ Output: "SOP: dev-pipeline | Branch: feat/vm-aws | Prefix: feat:"
```

**Reviewing a PR:**
```
→ Activate: pr-review
→ Output: "SOP: pr-review | PR: the-metafactory/crucible#N | Skill: CodeReview | Workflow: FullReview"
```

## Versioning & Releases

Version source is `arc-manifest.yaml`. Deploy command: `arc upgrade
crucible` — named for the record, but it does **not** work today, because
crucible is not installed as an arc package. See § CLAUDE.md Management.

## Naming

**metafactory** — always lowercase, one word.

## CLAUDE.md Management

This file is generated from compass-core's `templates/CLAUDE.md.template`
plus the repo sections declared in `agents-md.yaml`
(`docs/agents-md/architecture.md`, `docs/agents-md/critical-rules.md`).

Note the honest limitation: `arc upgrade crucible` — the command the
previous interim version of this file named as the regeneration path — does
**not** work today, because crucible is not installed as an arc package.
Until that is true, edits here must be mirrored back into
`docs/agents-md/*.md` by hand, or they will be lost the first time
generation does start working.
