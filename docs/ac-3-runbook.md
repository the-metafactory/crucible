# The AC-3 runbook

From an inventory entry to the number that moves.

**AC-3, verbatim from the spec** ([§6, Phase 3](design-infrastructure-factory.md)):
the execution-boundary corpus runs on a factory VM against a cortex checkout
pinned to an exact SHA, the resulting `captured_on.cortex_commit` **equals the
SHA the inventory file declared**, **and `captured_on.environment_digest` is
non-null**.

Two clauses, both observed in one run. This document is the imperative-mood
counterpart to assay's
[worked example](https://github.com/the-metafactory/assay/blob/main/docs/worked-example-environment.md):
that document explains what the seam buys, this one is the sequence of
commands that exercises it.

> **Performed.** This runbook's first performance was 2026-09-03, on ProxMox,
> by Vincent —
> [`vpzed-dev/smithy` `evidence/ac-3.md`](https://github.com/vpzed-dev/smithy/blob/main/evidence/ac-3.md).
> All four checks green, backstop measured, both AC-3 clauses observed. That
> run also found this document's first flaw (section 7 relied on a file it
> never told the operator to check for), fixed in the revision you are
> reading.

## The run is an offer, not an assignment

Everything below needs a hypervisor. It is **offered** to whoever has one and
wants to spend an evening on it — today that is Vincent's ProxMox node, and
that is entirely his call — or it waits for the AWS day in Phase 2. Nothing
here assigns the run to anyone, and the epic
([crucible#24](https://github.com/the-metafactory/crucible/issues/24)) is
explicit about that: smithy merges are Vincent's, and PRs upstream are offers.

The runbook itself is not held. It can be written, reviewed and merged now,
because it is verifiable against the two roles it drives — both merged in
`vpzed-dev/smithy` main — without a VM existing anywhere. What is held is the
last two sections: the run, and its receipt.

## What has to be true before you start

On the **control node** (the machine that runs OpenTofu and Ansible — not the
VM):

- A working checkout of [vpzed-dev/smithy](https://github.com/vpzed-dev/smithy)
  at `main`, set up per its README: age key, `secrets.enc.json`, the Python
  venv, pinned collections, and a golden image on the node.
- The two roles this runbook drives, both present in that checkout:
  `ansible/roles/metafactory_cortex/` and `ansible/roles/assay_env/`.
- Network reachability from the control node to the VM over SSH — the
  fingerprint capture runs **from the control node**, not on the guest.

On the **VM**, after the playbook: `git`, `unzip` and `bun` (layer 1 packages
and the `bun` role), plus an arc-installed cortex checkout. The inventory entry
below asks for all of it.

Everything in this runbook was verified against `vpzed-dev/smithy` main at
`82359ef5d3c2` and `the-metafactory/assay` main at `81778d2cbcce` on
2026-09-03. Re-read the roles at HEAD before you run: naming above the seam is
upstream's call, and this document is a projection of that tree, not an
authority over it.

## 1. The inventory entry

One YAML file per VM in smithy's `inventory/`. The file name is the VM name and
must be a DNS label. Write `inventory/ac3.yaml`:

```yaml
# Pick a vm_id that is free on your node and above the fleet's VMID floor.
# 501 is an example: `tofu apply` fails at plan time against a VMID already in
# use, or one belonging to a VM not tagged `opentofu`, rather than touching it.
vm_id: 501
description: "AC-3: the execution-boundary corpus on a factory VM"

cpu_cores: 4
memory_mb: 8192
disk_gb: 32

ipv4: dhcp

# Pin apt to an instant so the package set is reproducible. Convention:
# midnight AFTER your image's build serial.
archive_snapshot: 20260721T000000Z

# unzip for the bun role; git for metafactory_arc and metafactory_cortex.
packages: [unzip, git]

ansible_roles: [bun, claude, metafactory_arc, metafactory_cortex, assay_env]

tags: ["ubuntu"]
```

Three things about that role list, each of which is a decision rather than a
formatting choice.

**`metafactory_cortex` goes before `assay_env`, not after.** Install the
target, then fingerprint. `assay_env` is the layer-2 capstone: it records the
identity of whatever the roles before it put on the machine, so anything listed
after it is software the recorded identity does not describe. The target is
deliberately *not* part of that identity — every path arc writes for it is
pruned from the capture or never walked — so the digest is the same either way,
and the ordering is chosen to match the story the receipt tells rather than to
change the number.

> **Correction to [crucible#27](https://github.com/the-metafactory/crucible/issues/27).**
> The issue's sketch ends the list `[…, assay_env, metafactory_cortex]`. That
> was written before either role existed. Both roles as merged state the
> opposite ordering in their own headers, and so does
> [`inventory-example.yaml`](https://github.com/vpzed-dev/smithy/blob/main/inventory-example.yaml):
> *"It needs `metafactory_arc` before it, and it goes before `assay_env`, which
> is the order the runbook reads in: install the target, then fingerprint."*
> This runbook follows the tree.

**`claude` goes before `metafactory_cortex`.** Not because the corpus needs
Claude Code, but because of a residual the cortex role documents rather than
hides: the fingerprint prunes `~/.local/share/metafactory` and `~/.local/bin`,
but `~/.local` and `~/.local/share` are themselves printed by the walk. On a VM
where they do not exist yet, the first arc install *creates* them and the core
digest moves by exactly those directory-name lines. The `claude` role is the
only one that creates both. A spec that omits it is still a valid spec; it just
pays a one-off, by-directory-name difference in its first capture. Declaring
this properly — role dependencies modelled and checked instead of written in
prose — is upstream's planned follow-up.

**`bun` before `metafactory_arc`.** Roles apply in list order and arc is
installed with `bun link`.

## 2. The pin, and why it is not in the file above

`metafactory_cortex` refuses to install anything unless
`metafactory_cortex_pin` is set to a **full 40-character lowercase commit
SHA**. Not a tag, not a branch: this factory does not identify what it
installed by anything mutable, and a moved tag makes the commit in a receipt
unreproducible later.

That key **does not reach Ansible from a VM spec today.** `spec` is a strict
object type in `modules/vm-pve/variables.tofu`, and `ansible/inventory/tofu.py`
builds hostvars from a fixed list (`ansible_host`, `ansible_user`, `vm_id`,
`vm_ansible_roles`, `vm_packages`, `vm_archive_snapshot`, `vm_timezone`). A key
in neither is dropped by OpenTofu's object conversion **with no error and no
warning** — verified upstream against OpenTofu 1.11.7. Writing
`metafactory_cortex_pin:` into the spec would document a lie: the file would
say one thing and the run would install nothing.

Until a spec passthrough exists, supply the pin at run time — either on the
command line:

```sh
ansible-playbook ansible/site.yaml --limit ac3 \
  -e metafactory_cortex_pin=01f43bb9029e9fbcf8dffd6c99201001434e921f
```

or from `ansible/inventory/host_vars/ac3.yaml`:

```yaml
metafactory_cortex_pin: 01f43bb9029e9fbcf8dffd6c99201001434e921f
```

The role fails loudly when it is unset, so a forgotten pin stops the run
instead of installing something nobody chose.

Resolve your own pin rather than copying the one above:

```sh
git ls-remote https://github.com/the-metafactory/cortex main
```

> `01f43bb9029e9fbcf8dffd6c99201001434e921f` is the head of
> `the-metafactory/cortex` main **at the time of writing (2026-09-03)** — use
> your own. The point of AC-3 is that the SHA the inventory declared is the SHA
> that reaches the result; which SHA it is does not matter.

## 3. Build

From the smithy repo root:

```sh
source tofu.env
tofu apply
```

Then apply the roles. Run the validation gate first — it asserts the dynamic
inventory's shape and that every declared role is backed by a real directory,
which is where a typo in the list above surfaces:

```sh
./scripts/check-ansible.sh
ansible-playbook ansible/site.yaml --limit ac3 \
  -e metafactory_cortex_pin=01f43bb9029e9fbcf8dffd6c99201001434e921f
```

What to look for in the play, beyond `failed=0` in the recap:

| Task | Why it matters |
|---|---|
| `metafactory_cortex : Assert the installed commit is the declared pin` | The role reads the checkout's HEAD back with git. `arc install --pin` is silently ignored for an already-installed package at arc v0.45.0, so a successful install is not evidence the pin took. This task is the evidence. |
| `metafactory_cortex : Assert the installed checkout is unmodified` | A checkout sitting at the declared pin with edits on top would otherwise be reported downstream as "running `<sha>`" while running something else. |
| `assay_env : Write the assay environment file` | `changed` on a first run. This is `/etc/assay/environment.json` — the second clause of AC-3 in a file. |

If the `assay_env` block fails anywhere, the role **withdraws**
`/etc/assay/environment.json` rather than leaving a stale identity standing,
and the failure message says whether the withdrawal was confirmed. Read it: a
file it could not remove is called out explicitly, and that host is untrusted
until the file is gone.

Read the file straight off the VM now — you will compare against it in check 4.
`10.0.0.51` below is an example address; `tofu apply` prints yours as the
`ssh_command` output for the VM.

```sh
ssh ubuntu@10.0.0.51 cat /etc/assay/environment.json
```

```json
{
    "core_digest": "sha256:47ab86461c26b15a1075e5ea643a287119e67b5e141911d3b3fc970479507a4b",
    "definition": "inventory/ac3.yaml",
    "provider": "proxmox-ve",
    "provider_digest": "sha256:61cb3c9398c74f84c2994fc40c5f1c3a817ddbbddbe61e1fdde307820bb3e992",
    "schema": 1
}
```

(Digests from upstream's own receipt for a different VM — yours will differ.
`schema` is the number `1`, not the string; assay refuses a schema it does not
know.)

## 4. Fetch assay onto the VM, at a pin

The corpus is not installed by any role — it is a checkout you make by hand, at
a ref you name, for the same reason the target is pinned. On the VM:

```sh
ssh ubuntu@10.0.0.51
```

```sh
git clone https://github.com/the-metafactory/assay
git -C assay checkout 81778d2cbcce5d3cd49441ba6d9cdf602d212fdc
cd assay
bun install --frozen-lockfile
```

> `81778d2cbcce5d3cd49441ba6d9cdf602d212fdc` is the head of
> `the-metafactory/assay` main **at the time of writing (2026-09-03)** — use
> your own, and record whichever you used in the receipt. The corpus content is
> the measuring instrument; a receipt that does not name its version is a
> reading without a scale.

This clone is deliberately taken **after** `assay_env` has written the
environment file, and it does not disturb it: `~/assay` is outside the trees
the capture walks (`~/.local` and `~/.bun`), and `~/.bun/install/cache` — which
`bun install` does populate — is pruned wholesale from both find passes since
smithy#27.

That is the reason to *expect* the clone to be inert, not evidence that it was.
Check 4 cannot supply the evidence either — it compares the runner's header
against the same `/etc/assay/environment.json` the header was read from, so it
closes a loop rather than opening one: if the clone had perturbed a walked
tree, the file would not have changed and check 4 would still pass. Section 7
is the step that actually measures it, and it is the one worth doing.

## 5. Run the corpus

The corpus needs to be pointed at the arc-installed cortex checkout.
`ASSAY_CORTEX_REPO_PATH` is that pointer, and assay accepts it only if the
directory looks like a cortex checkout — it probes for
`src/runner/hooks/path-guard.hook.ts`. A path that does not have it is silently
ignored, assay falls back to a sibling-checkout guess, finds nothing, and every
case skips.

Confirm where arc actually put it before you run:

```sh
arc list --json
```

Find the entry named `cortex` and read its `installPath`. On arc v0.45.0's
default layout it is `~/.local/share/metafactory/arc/repos/cortex`. Then, from
the assay checkout:

```sh
ASSAY_CORTEX_REPO_PATH=$HOME/.local/share/metafactory/arc/repos/cortex bun run eval:execution-boundary
```

## 6. The four things to check

The short form, for when it is late and the prose below is too much. Every row
is pass/fail; anything else is a failure — go to section 9.

| # | Look at | Pass when |
|---|---------|-----------|
| 1 | rollup line | `12/12 behaved as documented (fail=0 skip=0)` — **both** zeros |
| 2 | header | `cortex@` + first 12 hex of YOUR pin — no `-dirty`, no other SHA |
| 3 | header | `env@` + 12 hex — not `none`, not `unreadable` |
| 4 | `ssh <vm> grep core_digest /etc/assay/environment.json` | file digest starts with check 3's 12 hex |
| §7 | fresh capture's DIGESTS tail | `core` line equals the file's `core_digest` — full 64 hex |

The `SECURITY POSTURE ⚠️` and `ENVIRONMENT DRIFT ⚠️ 12 UNPINNED` lines are
**expected** — neither is a failure (explained at the end of this section).

The runner prints a four-line header and a rollup. Three of the four checks are
in those lines; the fourth is a comparison against the VM.

```text
execution-boundary corpus — 12 case(s)
environment: linux/x64  kernel 6.14.0-33-generic  bun 1.2.23  cortex@01f43bb9029e  env@47ab86461c26
substrate:   unknown  (no reliable substrate signal found in the environment. …)
captured:    2026-09-03T21:14:08.771Z
```

```text
────────────────────────────────────────────────────────────────────────
CORPUS INTEGRITY  12/12 behaved as documented   (fail=0 skip=0)
SECURITY POSTURE  ⚠️  4 finding(s) STILL OPEN — r2-f3, r2-f4, r2-f5, r2-f6
                  those cases PASS *because* the vulnerability still reproduces.
By documented status: fixed=7  accepted-residual=1  open=4
ENVIRONMENT DRIFT ⚠️  12 case(s) with an UNPINNED baseline (captured_on never recorded a comparable environment or substrate) — r1-f1, r1-f2, <snip — all 12 ids, r1-f1 through r2-f6>
```

### Check 1 — `CORPUS INTEGRITY 12/12`, and `skip=0`

```text
CORPUS INTEGRITY  12/12 behaved as documented   (fail=0 skip=0)
```

> **⚠️ `skip=0` IS LOAD-BEARING. NEVER READ THE EXIT CODE AS THE RESULT.**
>
> With no cortex checkout found, the corpus reports
> **`0/12 behaved as documented (fail=0 skip=12)`** and **exits 0**. That is
> correct behaviour on assay's part — "this case requires a cortex checkout and
> there isn't one" is a real, honestly-declared non-result, not an error — but
> it means a wrapper script, a CI job or a human reading `$?` sees a green run
> in which **nothing was verified at all**.
>
> The runner exits non-zero only when `fail > 0` (or `2` on a runner crash).
> Skips never affect it.
>
> **Check `fail=0` AND `skip=0` in the recap line. Not the exit code.**
> A run reporting `skip=12` has measured nothing. Paste it into a receipt and
> you have published a healthy trace over an empty run — the exact failure
> shape this ecosystem exists to name.

### Check 2 — the header shows the declared SHA, undirtied

```text
cortex@01f43bb9029e
```

The first 12 characters of the SHA the inventory declared. Two failure
renderings to know:

- `cortex@01f43bb9029e-dirty` — the checkout is at the pin with edits on top.
  The commit is no longer an honest name for the code.
- `cortex@<some other 12 hex>` — a different commit than the one you declared.

This is **AC-3's first clause, observed**: `captured_on.cortex_commit` equals
the SHA the inventory declared.

### Check 3 — the header shows a real environment digest

```text
env@47ab86461c26
```

The first 12 hex of the core digest, `sha256:` prefix stripped. Two failure
renderings, and they mean different things:

- `env@none (unfingerprinted machine)` — assay looked and established there is
  no file. A positive assertion, reserved for that one case.
- `env@unreadable (<reason>)` — there is a file assay declined to read a digest
  out of, and assay established nothing about the machine. The parenthesised
  reason is specific: `cannot open it (EACCES)`, `not a regular file`,
  `empty file`, `<n> bytes, over the 16384-byte cap`, malformed JSON, an
  unknown `schema`, or a missing/non-string `core_digest`. A future schema is
  refused rather than best-effort parsed.

This is **AC-3's second clause, observed**: `captured_on.environment_digest` is
non-null.

### Check 4 — the digest is the VM's own

The header's `env@` must be the first 12 hex of `core_digest` in
`/etc/assay/environment.json` **on that VM**, from step 3. Compare them by eye,
or:

```sh
ssh ubuntu@10.0.0.51 cat /etc/assay/environment.json | grep core_digest
```

A header digest that does not match the file means assay read a *different*
file — check `ASSAY_ENVIRONMENT_FILE`, which overrides the path — or that
something moved the machine between the capture and the run.

Note what this check does **not** establish. Both sides of it come from the
same file, so it confirms assay read the identity this VM published, and
nothing about whether that identity is still true of the machine. Section 7 is
the reading that can disagree.

### What is expected, and is not a failure

**`SECURITY POSTURE ⚠️ 4 finding(s) STILL OPEN`.** Four of the twelve cases
document findings that are still open in cortex. They **PASS because the
vulnerability still reproduces** — `PASS` means "reality matches what this case
documents", never "the code is good". Two signals, deliberately never merged
into one number.

**`ENVIRONMENT DRIFT ⚠️ 12 case(s) with an UNPINNED baseline`.** All twelve
cases were locked in before an infrastructure factory existed to publish a
digest, and their `captured_on` fields are honest `null`s. Re-establishing
those baselines is a deliberate, separate act on the assay side (S4 in the
epic) and is **out of scope for this runbook**. Twelve is the number that
starts falling once it happens — which is the whole point of the exercise, and
the reason this run has to happen first.

## 7. The backstop: prove the corpus run did not move the machine

Recommended, and the only step here that *measures* what section 4 argued.

Check 4 is a closed loop: it compares the runner's `env@` against the same
`/etc/assay/environment.json` the runner read it from. Both sides move together
or not at all, so a clone that perturbed a walked tree would leave the file
untouched and check 4 green. The open loop is to fingerprint the machine again
and compare that fresh reading against the frozen one.

The "before" reading is already frozen: `assay_env` computed `core_digest`
from its own capture in section 3 and wrote it into the environment file
**before** the assay clone existed. So the measurement is one fresh capture,
compared against the file:

```sh
./scripts/vm-fingerprint.sh ubuntu@10.0.0.51 fingerprints/ac3-post-corpus.txt
ssh ubuntu@10.0.0.51 grep _digest /etc/assay/environment.json
```

**Pass:** the `core` line in the DIGESTS tail equals the file's `core_digest`
— the **full 64-hex digest**, not the 12-hex short form (`provider` must match
`provider_digest` the same way). Digest equality IS the claim: nothing the
capture looks at moved across the assay clone, `bun install` and the corpus
run.

```text
fingerprint written to fingerprints/ac3-post-corpus.txt
##### DIGESTS #####
core      sha256:47ab86461c26b15a1075e5ea643a287119e67b5e141911d3b3fc970479507a4b
provider  sha256:61cb3c9398c74f84c2994fc40c5f1c3a817ddbbddbe61e1fdde307820bb3e992
combined  sha256:21a1b81246b5c9824e6d7554b87198a8041086429802cbee5cf80960c0a765ec
```

**Fail:** the digests differ. Now you want the line-level diff, and for that
you need the section-3 capture — the role wrote it on the control node at
`fingerprints/<vm-name>.txt` (the default `assay_env_capture_path`; an
operator overlay can point it elsewhere, and this runbook's first performance
found no such file at all — so `ls` it, never assume it):

```sh
ls -l fingerprints/ac3.txt
diff fingerprints/ac3.txt fingerprints/ac3-post-corpus.txt
```

The exact pattern to copy — a capture diffed against a capture, with the moved
`core` and `combined` lines at the tail as the verdict and `provider` visibly
holding — is upstream's
[`evidence/op-20260901-post-pr-27-fingerprint-diff.md`](https://github.com/vpzed-dev/smithy/blob/main/evidence/op-20260901-post-pr-27-fingerprint-diff.md).
That receipt is the negative case done properly: the diff is non-empty, every
line that moved is shown, and the digest change is explained by exactly those
lines and nothing else. If the section-3 capture is gone, the digest
mismatch still stands as the finding; record it and say the line-level
diagnosis was not possible.

If the digests do **not** match, the run is still a valid AC-3 observation —
the environment file was written before the clone, so the digest it records is
the digest of the machine the corpus was installed onto. What you have lost is
the claim that the corpus is inert. Record the mismatch (and the diff, if you
have it) in the receipt and say so; that is a finding about the runbook, not a
failed run.

`fingerprints/**` is gitignored upstream: a capture carries the login user's
`authorized_keys` and sshd drop-ins, so it is the operator's private overlay.
Quote the DIGESTS lines in the receipt, never the capture body.

## 8. The receipt

Save the run as an operational receipt in whichever repo the runner owns:
smithy's `evidence/` if Vincent runs it on ProxMox, crucible's `evidence/` if
we run it on AWS. Name it `evidence/op-<YYYYMMDD>-ac3-first-run.md`. Crucible
has no `evidence/` directory yet — this run would be the first thing in it,
which is the correct reason for one to exist.

Follow the op-receipt form already in use upstream —
[`evidence/op-20260901-post-pr-26-assay_env.md`](https://github.com/vpzed-dev/smithy/blob/main/evidence/op-20260901-post-pr-26-assay_env.md)
is the pattern to copy. What that form does, and what makes it a receipt rather
than a log:

- **One claim at the top**, as a title line. Not a summary of what happened; a
  statement of what the run proves.
- **The terminal session, pasted whole**, in one fenced block, prompts
  included. Long provider output is elided with `<snip>` — the elision is
  visible, and nothing that carries a digest, a SHA or a verdict is ever
  inside one.
- **Digests inline, in full**, never truncated to the 12-hex short form and
  never paraphrased. The receipt is the durable copy of the number.
- **The comparison is in the transcript, not in prose.** Upstream's receipt
  ends with a bare `diff` that printed nothing and a `cat` of the environment
  file — the reader sees the check being made, not a report that someone made
  it.

For this run, the block should carry, in order: the inventory entry; `tofu
apply`; the `ansible-playbook` invocation **including the `-e
metafactory_cortex_pin=<sha>`**; the play recap; `cat
/etc/assay/environment.json`; the assay clone and its `git checkout <ref>`; the
`ASSAY_CORTEX_REPO_PATH=… bun run eval:execution-boundary` invocation; and the
runner's full output — header and rollup, uncut.

Then, if you ran section 7, the fresh capture's DIGESTS tail beside the `grep`
of the environment file — and the line-level `diff`, if you took one.
**Including that comparison is what upgrades the receipt from argued to
measured.** Without it the receipt asserts that the corpus run left the machine
alone on the strength of a prune list read in a comment; with it, the receipt
shows a reading taken after the fact agreeing with the identity recorded before
it. Everything else in the receipt is a comparison someone can watch being
made — this is the one claim that would otherwise be taken on trust.

Then state the four checks against it explicitly, with the full digests and the
full SHA written out. Post it on
[crucible#27](https://github.com/the-metafactory/crucible/issues/27) and
[#24](https://github.com/the-metafactory/crucible/issues/24).

## 9. Failure triage

| Symptom | What it means | Where to look |
|---|---|---|
| `env@none (unfingerprinted machine)` | No `/etc/assay/environment.json` on the VM. The `assay_env` role did not run, or it ran and *withdrew* the file after a failure. | The play output for the `assay_env` block. A rescue that fired names the failing task and says whether the withdrawal was confirmed. If the role never ran, check it is in `ansible_roles` and that `--limit` matched the host. |
| `env@unreadable (<reason>)` | The file exists and assay refused to read a digest out of it. The reason in parentheses is specific — malformed JSON, unknown `schema`, missing `core_digest`, not a regular file, over the 16 KiB cap, or unopenable. | `cat /etc/assay/environment.json` on the VM. Also check `ASSAY_ENVIRONMENT_FILE` is not pointing assay somewhere else. A refusal also prints a line to **stderr**, which is easy to miss if you piped stdout. |
| `cortex@` mismatch — the header's SHA is not the declared pin | The role's own assertion should have caught this and failed the play. **Investigate before anything else**: either the play was not the play that built this VM, someone touched the checkout afterwards, or the assertion did not run. On the VM, take the `installPath` from `arc list --json` and run `git -C <that path> rev-parse HEAD`; then read the play output for `Assert the installed commit is the declared pin`. Do not re-run and hope. |
| `cortex@<sha>-dirty` | The checkout is at the pin with a modified working tree, so the commit is not an honest name for the code. Frequently `M bun.lock`: arc retries `bun install` without `--frozen-lockfile` when the frozen attempt fails on lockfile drift. | `git -C <checkout> status --porcelain`. This is a determinism hole in the thing being measured, not a false positive — record it rather than cleaning it away silently. |
| `skip=12`, and every case `[SKIP]` | `ASSAY_CORTEX_REPO_PATH` is wrong. Assay probes for `src/runner/hooks/path-guard.hook.ts` under it and silently ignores a path that does not have it. | `arc list --json` for the real `installPath`; confirm the marker file exists under it. Note the run **exited 0** — see the warning under check 1. |
| `fail=<n>` on a case | Reality diverged from what the case documents. On a `fixed` case that is a regression. On an `open` case it may be **good news** — the hole closed and the record needs updating. | The FAIL detail printed above the rollup. Then the drift lines: if the environment digest also moved, the *machine* moved and the failure is an environment question before it is a code accusation. |
| `metafactory_cortex_pin is not set` | The pin did not reach Ansible. Most likely it was written into the VM spec, where OpenTofu drops it silently. | Section 2. Use `-e metafactory_cortex_pin=<sha>` or `ansible/inventory/host_vars/<host>.yaml`. |
| `is already installed at … and its HEAD is <x> where the VM declares <y>` | The role is refusing to re-pin a checkout arc will not move: at v0.45.0 `arc install --pin` short-circuits on an already-installed package and returns success **without checking out the new ref**. | `arc remove cortex` and re-run the role — or, on a test VM, destroy and re-provision, which is this fleet's normal reset. Reset is destroy plus re-provision; a snapshot is a speed optimisation, never the attestation baseline. |

## Out of scope

- **Re-establishing case baselines** — giving the twelve cases real
  `captured_on` stamps. A deliberate, separate act on the assay side (S4).
- **Any cortex configuration beyond install.** No stack, no secrets, no NATS
  wiring. `metafactory_cortex` installs the target with `--skip-secrets` and
  stops.
- **Windmill orchestration.** Phase 4.

## Sources

Every command and variable name above was read out of these files, not
recalled. Re-read them at HEAD before a run.

- `vpzed-dev/smithy` — [`ansible/roles/metafactory_cortex/tasks/main.yaml`](https://github.com/vpzed-dev/smithy/blob/main/ansible/roles/metafactory_cortex/tasks/main.yaml),
  [`defaults/main.yaml`](https://github.com/vpzed-dev/smithy/blob/main/ansible/roles/metafactory_cortex/defaults/main.yaml),
  [`ansible/roles/assay_env/tasks/main.yaml`](https://github.com/vpzed-dev/smithy/blob/main/ansible/roles/assay_env/tasks/main.yaml),
  [`inventory-example.yaml`](https://github.com/vpzed-dev/smithy/blob/main/inventory-example.yaml),
  [`scripts/vm-fingerprint.sh`](https://github.com/vpzed-dev/smithy/blob/main/scripts/vm-fingerprint.sh),
  [`evidence/op-20260901-post-pr-26-assay_env.md`](https://github.com/vpzed-dev/smithy/blob/main/evidence/op-20260901-post-pr-26-assay_env.md),
  [`evidence/op-20260901-post-pr-27-fingerprint-diff.md`](https://github.com/vpzed-dev/smithy/blob/main/evidence/op-20260901-post-pr-27-fingerprint-diff.md)
- `the-metafactory/assay` — [`evals/execution-boundary/runner.ts`](https://github.com/the-metafactory/assay/blob/main/evals/execution-boundary/runner.ts),
  [`lib/environment.ts`](https://github.com/the-metafactory/assay/blob/main/evals/execution-boundary/lib/environment.ts),
  [`lib/cortex-repo.ts`](https://github.com/the-metafactory/assay/blob/main/evals/execution-boundary/lib/cortex-repo.ts),
  [`package.json`](https://github.com/the-metafactory/assay/blob/main/package.json),
  [`environments/README.md`](https://github.com/the-metafactory/assay/blob/main/environments/README.md),
  [`docs/worked-example-environment.md`](https://github.com/the-metafactory/assay/blob/main/docs/worked-example-environment.md)
- `the-metafactory/crucible` — [`docs/design-infrastructure-factory.md`](design-infrastructure-factory.md) §6,
  [`docs/stack-and-seams.md`](stack-and-seams.md)
- Known upstream gap: cortex's dependency repos install unpinned
  ([the-metafactory/arc#398](https://github.com/the-metafactory/arc/issues/398)).
  It does not move the core digest and does not affect the commit the role
  asserts, but the target's *transitive* source is not fixed by the pin. Worth
  a line in the receipt.
