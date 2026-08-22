## Architecture

Two planes, one seam (`docs/design-infrastructure-factory.md`):

- **Backend plane** (persistent): otel-lgtm + Windmill containers.
- **Test plane** (ephemeral): one YAML per VM — provision → configure →
  install target at an exact ref → test with receipts → destroy → prove the
  rebuild identical via the fingerprint digest.
- **The seam**: the provider-blind inventory `spec` object and output
  contract. ProxMox VE is the reference implementation
  (vpzed/opentofu-pve-template); AWS is the second provider. The provider
  list is not curated here — operators choose and extend (DD-21). Agnosticism is
  achieved at the contract, never via an abstraction layer, and is measured
  by the provider-invariant core fingerprint digest — two providers can agree
  by coincidence, so the third is where the claim earns its keep.
