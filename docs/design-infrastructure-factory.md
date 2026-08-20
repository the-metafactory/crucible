# Design: the infrastructure factory

**Status:** Draft for review — decisions DD-11..DD-20 are proposals, not settled
**Home:** this repo (crucible) is the infrastructure factory this spec describes; the consuming contract lives in [assay](https://github.com/the-metafactory/assay/blob/main/environments/README.md)
**Author:** Luna (with Andreas)
**Contributors whose work this specifies:** Vincent Zontini (the factory itself), Robert Chuvala (the digest criterion), Magnús Smárason (the injection protocol)
**Refs:** [`assay:docs/design-testing-factory.md`](https://github.com/the-metafactory/assay/blob/main/docs/design-testing-factory.md) (DD-1..DD-10, which this continues) · [`assay:ideas/2026-08-01-environment-tier.md`](https://github.com/the-metafactory/assay/blob/main/ideas/2026-08-01-environment-tier.md) (the tray note this hardens into a spec) · [`assay:CONTEXT.md`](https://github.com/the-metafactory/assay/blob/main/CONTEXT.md) (language — *environment* and *substrate* are used in their canonical senses throughout) · [vpzed/opentofu-pve-template](https://github.com/vpzed/opentofu-pve-template) (the reference implementation)

---

## 1. Problem

DD-7 rules that environment work precedes corpus work: *the agent's world must
be deterministic and green before assistant behaviour is meaningfully
testable*. Today that prerequisite is unmet in three specific ways:

1. **All 12 corpus cases report an unpinned baseline.** `captured_on` exists
   and is honest, but nothing produces an environment worth pinning to. The
   fix is not more fields — it is a factory that builds environments whose
   identity can be recorded.
2. **The factory exists, but on one provider and one operator's hardware.**
   Vincent has built and proven a four-component stack (OpenTofu, Ansible,
   otel-lgtm, Windmill) against ProxMox VE in his homelab. It cannot yet be
   picked up by anyone else, which fails DD-8's day-one test.
3. **`environments/` does not exist in this repo.** The charter names the
   tier; there is no directory, no contract, and nowhere for the factory's
   output to land.

**Scope of "up and running", stated so it can fail:** a contributor with an
AWS account can provision a pinned test VM from one YAML file, install a
target system (cortex, myelin, arc) at an exact git ref, run a graded corpus
against it with telemetry receipts, destroy everything, and **prove** the
next VM is identical — where "prove" means an executable comparison, not a
claim.

---

## 2. What exists — verified, not assumed

Everything below was read or executed, not inferred.

| Piece | State | Evidence |
|---|---|---|
| OpenTofu VM module (ProxMox) | working | `vpzed/opentofu-pve-template@496bb67` — typed `spec` contract, plan-time guards, encrypted state |
| Ansible layer-2 roles | working, idempotent | `nats_server`, `bun`, `claude`, `docker`, `metafactory_arc` + dynamic inventory `tofu.py` reading `tofu output -json vms` |
| Environment fingerprint | working, text-only | `scripts/vm-fingerprint.sh` — packages, apt config, units, users, layer-2 trees; per-instance noise deliberately excluded |
| otel-lgtm backend | containers up | compose files + two Grafana dashboards |
| Windmill backend | containers up, **no factory code yet** | `windmill/local-dev/` is bare CLI scaffolding |
| assay attestation | implemented | `EnvironmentStamp` / `CapturedOnStamp` / `DriftAssessment` (`evals/execution-boundary/lib/environment.ts`) |
| arc ref-pinning | shipped in v0.45.0 | `--pin` accepts any git ref; `bun.lock` tracked (verified via GitHub API against the tag) |

**Constraints inherited from measurement** (tray note; two cost real time to learn):

1. **`bwrap` fails inside a container even as root** — the sandbox tier must
   be a real VM, never a container.
2. **macOS only runs on Apple hardware** — the tier model has a permanent
   hole shaped like a laptop; EC2 Mac makes it expensive rather than
   impossible (§8).
3. **A digest can't quietly move; a tag can** — environment identity must be
   content-addressed.
4. **Reset by recreation beats rollback** — snapshots of a drifted machine
   make the known-good not.

**Known gaps in arc that this spec must design around, not ignore:**
the `--pin` is **install-time only** — nothing on disk records it, and a
later `arc upgrade` silently moves the checkout off the pinned commit;
`self-upgrade` requires bun ≥ 1.2 (arc#394); a modified tracked lockfile
slips the dev-checkout filter (arc#393).

---

## 3. Architecture — two planes and one seam

```
            BACKEND PLANE (persistent, patched, boring)
   ┌────────────────────────────────────────────────────┐
   │  otel-lgtm (traces, logs, dashboards)               │
   │  Windmill  (Scripts + Flows: orchestration, reaper) │
   └───────────────▲────────────────────────────────────┘
                   │ telemetry + receipts
            TEST PLANE (ephemeral, never patched, destroyed)
   ┌───────────────┴────────────────────────────────────┐
   │  VM per inventory/*.yaml                            │
   │  cloud-init (layer 1: identity, apt snapshot pin)   │
   │  ansible    (layer 2: bun, claude, docker, arc)     │
   │  arc --pin  (layer 3: target — cortex, myelin)      │
   │  fingerprint → digest → reaches the assay result    │
   └────────────────────────────────────────────────────┘
   ══════════════════ THE SEAM ═══════════════════════════
   modules/vm/variables.tofu  (the `spec` object)
   modules/vm/outputs.tofu    (vm_id, ipv4_addresses,
                               ssh_command, ansible_roles)
   ┌──────────────────────┐      ┌──────────────────────┐
   │ modules/vm-pve       │      │ modules/vm-aws       │
   │ (Vincent, reference) │      │ (this spec)          │
   └──────────────────────┘      └──────────────────────┘
```

The split Vincent stated explicitly: backend containers are **persistent**;
test VMs are **ephemeral**. Everything above the seam — the ansible roles,
`tofu.py`, `site.yaml`, the fingerprint script, every future Windmill Flow —
is provider-blind and must stay that way. Providers are swapped below the
seam only.

---

## 4. The environment identity contract

The tray note's single hard requirement, now mechanized:

> The environment's identity must be recordable, and must reach the result.

**The mechanism, end to end:**

1. **Capture.** `vm-fingerprint.sh` (extended, not replaced) emits the
   canonical text: os-release, kernel, package set with versions, apt
   configuration, enabled units, login user, layer-2 file trees. Per-instance
   noise (host keys, machine-id, MACs, timestamps) stays excluded — that
   exclusion list is itself part of the contract and lives beside the script.
2. **Digest.** `environment_digest = sha256(canonical text)`. The text is
   committed for humans to `git diff`; the digest is what machines compare.
3. **Record.** Each environment definition lives in `environments/<name>/`
   with its fingerprint text and digest committed at lock time.
4. **Reach the result.** `EnvironmentStamp` gains
   `environment_digest: string | null`. The honest-null rule is unchanged: a
   corpus run on an unfingerprinted machine (a laptop) records `null` plus a
   note — never a guess. `DriftAssessment` compares run-time digest against
   the case's `captured_on` and reports **match / drift / unpinned** exactly
   as it does today for the other fields.
5. **Prove the comparator can fail** (DD-3 applied to the factory itself).
   Before the digest is trusted: inject a fault — change one pinned package
   version — rebuild, recapture, and observe the non-empty diff and changed
   digest. Record the injection and the observed red beside the script. A
   fingerprint that has never been seen to differ is a detector nobody has
   watched fire.

**Comparability boundary, stated up front:** a pve environment and an aws
environment produce *different* digests by design (different kernels,
different cloud-init datasources). Comparability is **within** an
environment definition, across rebuilds. Cross-provider comparability is
OQ-8, deliberately deferred.

---

## 5. Design decisions (continuing DD-1..DD-10)

### DD-11 — Adopt Vincent's repo as the reference implementation; port at the seam, never fork above it
**Decision:** the ProxMox implementation remains Vincent's and remains
canonical for the pattern. Our AWS work reuses `ansible/`, `tofu.py`,
cloud-init templates, and the fingerprint script **verbatim**, freezing the
module output contract (`vm_id`, `ipv4_addresses`, `ssh_command`,
`ansible_roles`, `ansible_user`) byte-compatible. Improvements to shared
layers go **upstream to his repo as PRs**, never into a private copy.

**Why:** the seam exists by construction — he built it that way. Forking
above it hands a merge burden to a contributor who is short on health and
time, and forks the practice into two dialects.

**Rejected:** rewriting in CDK/Pulumi — a new language, discards proven work,
and Pulumi does not replace Ansible anyway (Vincent's own finding, with the
vendor's page as the source).

### DD-12 — AWS is the second provider: `modules/vm-aws`, same `spec` object
**Decision:** implement the seam over `aws_instance`.

- **user-data is native.** The `proxmox_virtual_environment_file` resource
  and the hypervisor-SSH machinery exist only because ProxMox has no snippet
  API; on EC2 they are deleted, not ported. Same rendered cloud-init, passed
  as an argument.
- **The one honest contract break:** `cpu_cores`/`memory_mb` have no direct
  AWS equivalent. The spec gains `instance_type = optional(string)`; when
  unset, a documented lookup table maps the requested shape to the nearest
  type and the plan **names the mapping** so nobody is surprised silently.
- **`guards.tofu` dissolves rather than ports.** There is no shared VMID
  space to defend. The intent — never touch what we don't own — maps to a
  mandatory `managed = assay-factory` tag plus an IAM policy that scopes
  destroy/terminate to that tag condition: a stronger mechanism than a
  plan-time precondition.
- **Base image pinned by AMI ID** — immutable and content-addressed, which
  is strictly better identity than a datastore file path (constraint 3).
  Canonical's published Ubuntu AMI, one pinned ID per region, recorded in
  the fingerprint. The apt `snapshot.ubuntu.com` pin carries over unchanged.
- **Test-plane instances are EC2 VMs (Nitro), never Fargate/ECS** —
  constraint 1: the sandbox question needs a real VM.

**Rejected:** containers for the test plane (constraint 1); building custom
AMIs with Packer at this stage — the pinned public AMI + cloud-init +
snapshot pin gives the same determinism with one less toolchain ("dirt
simple but it worked").

### DD-13 — A dedicated AWS account, with the reaper and budget alarm landing before the first VM
**Decision:** the factory runs in its own AWS account — never an existing
work or personal account. Before any test VM is provisioned:

- a **budget alarm** exists;
- every test-plane instance carries a `ttl_hours` tag, **enforced at plan
  time** (a validation refuses an instance without one);
- a scheduled **reaper terminates expired instances** — implemented as a
  Windmill Flow, so the factory's own orchestration layer polices the
  factory's own cost, and the reaper is proven by injection like any other
  detector: launch an instance with an expired TTL, observe termination.

**Why:** a ProxMox VM left running is free; an EC2 instance left running is
a bill. Ephemeral-by-discipline must become ephemeral-by-mechanism before
the discipline is ever tested. The persistent plane is one small stoppable
instance (t4g-class) running the same compose files as Vincent's backend.

**Mechanisms, imported from the prior estates rather than invented:**

- **Two independent budgets, not one budget with two notification levels** —
  a warning budget and a hard-ceiling budget, each firing at 100% of its own
  limit. Cleaner semantics ("the threshold I declared was crossed") and
  independently tunable.
- **The persistent plane gets the estates' auto-shutdown module shape:** a
  time-based stop schedule as the hard floor, plus an on-host idle monitor
  (installed via SSM association, re-applied on a schedule to heal drift)
  that stops the instance early when no sessions exist. The backend plane
  only needs to be up while tests run or someone is reading Grafana.
- **The test plane keeps the TTL reaper** — instances there are measured in
  hours by design, and the reaper terminates rather than stops.

### DD-14 — Targets install via arc at an exact ref, and every install can fail on drift
**Decision:** layer 3 (the software under test) is installed by arc:

- `metafactory_arc` role: version pin `0.44.3 → 0.45.0`, and
  `bun install --production` gains `--frozen-lockfile`. Without the flag,
  the role remains the last unpinned install path — the tracked-lockfile fix
  landed on arc's CI surface and left the provisioning surface open, the
  exact one-surface drift class the execution-boundary corpus catalogues.
- New roles `metafactory_cortex` / `metafactory_myelin` install via
  `arc install <pkg> --pin <ref>` — the v0.45.0 capability, so a SHA or
  branch works and nobody mints rc-tags on public repos.
- **Constraint designed around, not ignored:** the pin is install-time only.
  On an ephemeral VM this is acceptable — the VM dies before any `arc
  upgrade` can move it. On the **persistent plane it must not be relied on**;
  persisting the pin is raised upstream as an arc issue, not worked around
  silently here.

### DD-15 — Reset is destroy + re-provision; snapshots are never the attestation baseline
**Decision:** desired-state reset means `tofu destroy` + `tofu apply` from
the declared definition. Snapshots/AMIs of a *built* machine may be used to
make rebuilds faster, but the fingerprint of a snapshot-restored machine
must equal the fingerprint of a from-scratch build — checked, not assumed —
before any snapshot path is used in anger.

**Why:** constraint 4. You can snapshot a drifted machine, and then your
known-good isn't.

### DD-16 — State: S3 backend for the team; Vincent's encrypted local state stays valid for the reference
**Decision:** the AWS realization uses an S3 state backend (native lockfile
locking) and **keeps** the client-side state encryption block (pbkdf2 +
aes_gcm, `enforced = true`) so state at rest in the bucket is ciphertext.
The reference implementation's sops + local encrypted state is unchanged —
it is correct for a single operator.

### DD-17 — A run produces a receipt, and the receipt is the artifact
**Decision:** Vincent's phrase — *"automatable with receipts"* — becomes a
defined object. A **receipt** is the record a test-plane run leaves behind
after the VM is destroyed:

```
{ environment_digest, provider, ami_or_image_id,
  arc_ref, target_refs {cortex: <sha>, ...},
  corpus_results (the runner's three signals, never collapsed),
  otel trace ids, started/finished }
```

Test VMs export telemetry to the backend plane's otel-lgtm (OTel GenAI
semantic conventions, per DD-6); the receipt binds the corpus result to the
environment digest and the trace. Receipts are files — portable, greppable,
and independent of which backend plane collected them.

### DD-18 — ARM Linux is a first-class tier via Graviton
**Decision:** the AWS module supports arm64 instance types from day one; the
fingerprint and `EnvironmentStamp.arch` already record architecture.

**Why:** everything captured so far was on Apple Silicon and nothing recorded
it (OQ-5 in the testing-factory spec). Graviton closes the ARM gap for an
instance-hour, with no hardware purchase and no waiting on anyone's homelab.

---

DD-19 and DD-20 import patterns proven in two of this project's operators'
existing private OpenTofu estates on AWS (referenced here by pattern, not by
name — they carry client context that does not belong in a public repo).
None of these is speculative; each has run in production and at least one
exists because of a failure that already happened once.

### DD-19 — AWS access is SSM-first; nothing in the factory opens port 22
**Decision:** no security group in either plane allows inbound SSH.
Interactive access is **SSM Session Manager**; Ansible reaches test-plane
VMs over **SSH-through-SSM** (`ProxyCommand aws ssm start-session
--document-name AWS-StartSSHSession`), so every ansible role, `tofu.py`, and
the fingerprint script run **unchanged** — the transport is tunnelled, the
tool never knows. Session output/recording lands in S3 as in the prior
estates. *(Resolves OQ-6.)*

**Why:** this is the established house posture — the prior estates run
zero-inbound bastions where the SSM session count is the only liveness
signal — and SSH-over-SSM is the variant that gets that posture without
forking Vincent's provider-blind layer, which DD-11 forbids. The reference
implementation keeps plain SSH on the homelab LAN, where the posture
question is different and is Vincent's.

### DD-20 — Pipeline: OIDC-only auth, plan/apply split, and a delete-guard that reads the saved plan
**Decision:** four CI mechanisms, imported as a set:

- **GitHub Actions OIDC → AWS; no long-lived keys anywhere.** Trust policy
  scoped to this one repo, `sub` claim pinned to the protected environment
  and `main` — never the `pull_request` context.
- **Plan on PR; apply only by human `workflow_dispatch`** bound to a
  protected environment with a typed confirmation phrase. The apply consumes
  the exact saved plan file — never a re-plan that could diverge from what
  was reviewed.
- **Delete/replace guard on the persistent plane:** before apply, read the
  saved plan JSON and **refuse** if any resource's actions include
  `delete`. An unattended apply that can destroy is the same shape as an
  exit code accepted as evidence. **Deliberately not applied to the test
  plane** — there, destruction is the job (DD-15), and the plane is driven
  by Windmill, not CI; the two planes get separate tofu roots so the guard
  can bind to one without lying about the other.
- **Credential-less `tofu validate` + `fmt -check` over every root, with
  dynamic root discovery** (any directory containing a `.tf`/`.tofu` file is
  a root). Discovery is dynamic because a hardcoded matrix is a check that
  silently stops covering — a new root would merge with no CI and read as
  green, the aggregate-green shape applied to the pipeline itself.

**Why:** each mechanism exists because its absence already burned one of the
prior estates once (a stale-branch squash-merge silently reverting
intervening work; uncovered tofu roots merging broken HCL silently). The
stale-base guard those repos also run is already this repo's CLAUDE.md rule;
CI should enforce it here too rather than trusting recall.

---

## 5b. Hyperscaler- and hypervisor-agnostic: contract, not abstraction

The tempting road to "agnostic" is an abstraction layer — one module with
`if provider == "aws"` branches, or a tool that promises to speak every
cloud. That road is rejected here: HCL has no interfaces, so such modules
rot into the union of every provider's quirks; and switching languages
(Pulumi, Crossplane) relocates the problem without shrinking it. An
abstraction layer is also exactly where a spec field would get **silently
ignored** on one provider — a healthy trace over an environment that isn't
what its YAML declared.

Instead, agnosticism lives at **three fixed contracts**, with everything
below them written natively per provider:

1. **The inventory contract** — the `spec` object, one YAML per VM. A
   provider module must implement every field **or reject it loudly at plan
   time**. Silent acceptance of an unimplementable field is a conformance
   failure, not a convenience: `archive_snapshot` quietly doing nothing on
   some future provider would unpin apt while the YAML still claims the pin.
2. **The output contract** — `vm_id`, `ipv4_addresses`, `ssh_command`,
   `ansible_roles`, `ansible_user` — byte-compatible, because `tofu.py` and
   every future Windmill Flow consume it blind.
3. **The result contract** — the fingerprint, split into two sections:
   a **provider-invariant core** (package set and versions, apt
   configuration, enabled units, login user, layer-2 file trees) and a
   **provider section** (kernel flavour, cloud-init datasource, image
   identity). Core and provider sections are digested separately.

The third contract is what makes agnosticism *measurable* instead of
asserted: an environment definition is **proven portable** when its
core-section digest matches across providers, and the provider section is
where the honest differences live on the record. That is the claim/
comparison pairing applied to the portability claim itself. *(This
supersedes OQ-8's divergent-by-design recommendation: the split gives
within-provider comparability and cross-provider comparability from the
same capture.)*

**Conformance is executable.** A provider module is conformant when:

- the reference inventory file plans cleanly against it;
- an inventory file using a spec field the module cannot honor **fails at
  plan time**, named;
- `ansible-playbook site.yaml` runs against its VMs with zero edits;
- its VMs' **core fingerprint digest matches the reference
  implementation's** for the same definition.

Adding a provider — hypervisor or hyperscaler, Proxmox today, AWS next,
Hetzner or libvirt or GCP whenever someone needs one — is one module under
the seam plus a conformance run. Nothing above the seam changes, and the
conformance run is the proof rather than the promise.

---

## 6. Phases, acceptance criteria, and predictions

Every criterion below is an executable comparison that can fail. Predictions
are recorded now, before any run, per the Specification gate.

### Phase 0 — Close the loop on the reference implementation
*Owner: Vincent; two small assists offered from our side.*

- [ ] `metafactory_arc` role: `0.45.0` + `--frozen-lockfile` (two-line PR)
- [ ] fingerprint script emits `sha256` digest alongside the text
- [ ] **injection proof recorded**: one pinned package changed → rebuild →
      non-empty diff and changed digest, observed and committed

**AC-0:** two captures across destroy/recreate are byte-identical (empty
`git diff`); the injected fault produces a visibly red comparison.

### Phase 1 — `environments/` lands in assay
- [ ] `environments/README.md` — the contract (declared in git, built to a
      digest, never mutated in place, destroyed after use, identity recorded)
- [ ] `EnvironmentStamp.environment_digest` added, honest-null preserved
- [ ] runner reports match/drift/unpinned per case for the new field

**AC-1:** a corpus run on a factory VM yields a non-null digest in
`captured_on`; a run on an unfingerprinted laptop yields `null` **plus a
note**, and the rollup still prints the unpinned count loudest.

### Phase 2 — `modules/vm-aws`
- [ ] provision from an unchanged `inventory/*.yaml`; `tofu.py` and
      `site.yaml` run with zero edits
- [ ] TTL validation + reaper + budget alarm live **before** the first VM
      (DD-13 ordering is part of the criterion)
- [ ] reaper proven by injection: expired-TTL instance observed terminated

**AC-2:** `tofu apply` with one three-line inventory file yields an
instance reachable over SSH-through-SSM with **no inbound rules in any
security group**; `ansible-playbook site.yaml` is green; two rebuilds of
the same definition produce identical digests; an instance without a
`ttl_hours` tag fails **at plan time**; the §5b conformance run passes —
including the negative half: an inventory field the module cannot honor
fails the plan, named, and the **core** fingerprint digest matches the
reference implementation's for the same definition.

### Phase 3 — Target enablement (cortex, myelin)
- [ ] `metafactory_cortex`, `metafactory_myelin` roles using
      `arc install --pin <ref>`
- [ ] the inherited six-step smoke loop runs end to end:
      *provision → substrate → arc → target → smoke test*, stopping at
      first failure

**AC-3 (the criterion the whole spec exists for):** the execution-boundary
corpus runs on a factory VM against a cortex checkout pinned to an exact
SHA, and the resulting `captured_on.cortex_commit` **equals the SHA the
inventory file declared**. When that holds, the factory's pin has reached
the assay result, and "unpinned baseline: 12 of 12" starts falling.

### Phase 4 — Orchestration
- [ ] first Windmill Script drives `tofu` + `ansible` (Vincent's stated next
      step — note that on AWS the container-drives-IaC obstacle he flagged
      disappears: user-data is an API call, so a container with an instance
      role and outbound HTTPS drives everything; no hypervisor SSH key)
- [ ] one Flow: provision → install → run corpus → write receipt → destroy

**AC-4:** a single Windmill run produces a receipt (DD-17) and leaves zero
test-plane instances behind — checked by listing, not assumed from exit
codes (an exit code accepted as evidence is the shape this ecosystem has now
found six times).

### Predictions — including predicted failures

- **P-1:** the first AWS fingerprint will be noisy (EC2 kernel flavour,
  cloud-init datasource differences). Expect one or two iterations on the
  exclusion list before AC-2's identical-digest criterion holds.
- **P-2:** Windmill-in-a-container driving IaC **fails on ProxMox** (needs a
  long-lived SSH key to the hypervisor) and **succeeds on AWS** (API-only).
  If P-2 holds, it is an argument discovered before a weekend was spent on
  it; if it fails, the spec's AWS-advantage claim weakens and says so.
- **P-3:** the apt `snapshot.ubuntu.com` pin must survive AWS's
  region-mirror defaults in the Ubuntu cloud image. Verify explicitly; a
  silently unpinned apt would be a healthy trace over stale input.
- **P-4:** someone will be surprised by the `instance_type` mapping
  (asked-for shape ≠ granted shape). The named-mapping plan output exists
  because of this prediction.

---

## 7. Security and confidentiality

- Dedicated account; credentials never in the repo; sops stays on the
  reference side, IAM/SSM parameters on the AWS side.
- No public ingress to the test plane: SSH restricted to the operator's
  address, or SSM Session Manager (OQ-6).
- This repo's confidentiality gate applies to every artifact the factory
  emits: receipts and fingerprints carry no live platform IDs, no real
  hostnames outside the test plane's own ephemeral names, placeholders in
  anything shippable.

---

## 8. Non-goals

- **A macOS tier.** EC2 Mac exists (24-hour minimum, dedicated-host
  pricing); the hole is named and deliberately left open until a case needs
  it.
- **Multi-host and physical-OT tiers.** The charter names them; nothing
  drives them yet.
- **Replacing the ProxMox path.** It is the reference implementation, not
  a legacy one.
- **Discovery.** The factory builds environments for locking in known-good;
  it is not a bypass-discovery engine (charter boundary, restated because a
  factory is exactly where that confusion would creep in).

---

## 9. Open questions

**OQ-6 — resolved by DD-19.** SSM-first, zero inbound; Ansible over
SSH-through-SSM so nothing above the seam changes. (Was: SSH restricted by
security group.)

**OQ-7 — Which account and region?** Needs Andreas and JC. Region trades
operator latency (ap-southeast-2) against instance-family breadth
(us-east-1). *Recommendation: decide with the budget alarm, as one
decision.*

**OQ-8 — superseded by §5b.** The fingerprint splits into a
provider-invariant core section and a provider section, digested
separately: within-provider comparability and cross-provider portability
proof from the same capture. (Was: divergent-by-design, revisit later.)

**OQ-9 — resolved: this repo.** The factory lives in
**the-metafactory/crucible** — named for the fire-assay vessel, extending
assay's register (an assay performed in an unknown crucible is invalid,
which is the unpinned-baseline finding restated). The consuming contract
stays in assay (`environments/README.md`) so the instrument never depends
on one implementation of what it measures. Vincent's repo remains his and
remains the reference implementation; work migrates here at his pace, with
attribution preserved. (Was: Vincent's twice-asked, unanswered question.)

**OQ-10 — One shared backend plane or one per operator?** *Recommendation:
one per operator (Vincent's homelab, our AWS instance) — receipts are files
and portable, so nothing forces a shared Grafana.*

---

## 10. Delivery

Ordered by DD-7, environment before corpus. Phases 0 and 1 are independent
and can run in parallel; 2 depends on 1's contract; 3 on 2; 4 on 3.

1. **Phase 0 assists** — the two-line arc-role PR and the digest+injection
   PR to `vpzed/opentofu-pve-template`, offered upstream (DD-11).
2. **Phase 1** — `environments/` contract + `environment_digest` in the
   stamp. Small, entirely in this repo, unblocks the AC-3 criterion.
3. **Phase 2** — `modules/vm-aws` with reaper-first ordering (DD-13).
4. **Phase 3** — cortex/myelin roles + the smoke loop; AC-3 is the exit.
5. **Phase 4** — Windmill Script/Flow; tests P-2 explicitly.
6. **Upstream issues raised, not worked around:** arc persisted-pin;
   anything the seam freeze surfaces goes to Vincent's repo as a PR.

Tracking: one iteration umbrella issue in this repo with one sub-issue per
phase, per the repo's sub-issue convention.

**Definition of done for the spec itself:** every DD accepted, rejected with
reasoning, or converted to an OQ with a recommendation — and every
prediction in §6 either confirmed or falsified **in writing** once the runs
happen. A prediction nobody scores is a claim without a comparison.
