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
