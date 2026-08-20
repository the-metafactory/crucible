## Architecture

Two planes, one seam (`docs/design-infrastructure-factory.md`):

- **Backend plane** (persistent): otel-lgtm + Windmill containers.
- **Test plane** (ephemeral): one YAML per VM — provision → configure →
  install target at an exact ref → test with receipts → destroy → prove the
  rebuild identical via the fingerprint digest.
- **The seam**: the provider-blind inventory `spec` object and output
  contract. ProxMox VE is the reference implementation
  (vpzed/opentofu-pve-template); AWS is the second provider. Agnosticism is
  achieved at the contract, never via an abstraction layer, and is measured
  by the provider-invariant core fingerprint digest.
