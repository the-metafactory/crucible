# crucible

**The infrastructure factory** — deterministic, digest-identified test
environments for systems built at the pace software is now built.

## Why "crucible"

In a fire assay, the sample is fused **in a crucible** under known,
controlled conditions before anything is weighed. An assay performed in a
contaminated or unknown crucible is invalid — the vessel's identity is part
of the result. That is exactly this repo's job: hold the thing under test at
known conditions, prove those conditions are what they claim to be, and be
emptied and refired between uses.

The name extends the register the ecosystem's instruments already use —
[assay](https://github.com/the-metafactory/assay) (the analysis), miner,
forge. crucible is the vessel the assay needs.

## The three factories

metafactory is a factory of factories. Three of them:

| Factory | Produces | Home |
|---|---|---|
| **Software factory** | the systems themselves | cortex, arc, pilot, the dev loop |
| **Test factory** | verdicts — claims paired with executable comparisons that can fail | [assay](https://github.com/the-metafactory/assay) |
| **Infrastructure factory** | environments — declared in git, built to a digest, never mutated in place, destroyed after use, identity recorded | **crucible** |

The boundary with assay is a contract, deliberately: assay **consumes**
environment identity (a digest that reaches every result's `captured_on`);
crucible **produces** it. The contract lives on assay's side
([`environments/README.md`](https://github.com/the-metafactory/assay/blob/main/environments/README.md))
so the instrument never depends on one implementation of the thing it
measures. This repo implements that contract.

## The shape

Two planes, one seam (full spec:
[`docs/design-infrastructure-factory.md`](docs/design-infrastructure-factory.md)):

- **Backend plane** (persistent): otel-lgtm for telemetry, Windmill for
  orchestration.
- **Test plane** (ephemeral): one YAML file per VM — provision → configure →
  install the target at an exact ref → test with receipts → destroy →
  **prove** the next one is identical.
- **The seam**: a provider-blind inventory + output contract. ProxMox VE is
  the reference implementation
  ([vpzed-dev/smithy](https://github.com/vpzed-dev/smithy),
  by Vincent Zontini — the work this repo grew from); AWS is the second
  provider — but the provider list is not curated here: whoever runs the
  factory picks what suits them and adds a module if one is missing (DD-21).
  Hyperscaler/hypervisor agnosticism is achieved at the contract,
  not through an abstraction layer, and it is **measurable**: an environment
  definition is proven portable when its provider-invariant fingerprint
  digest matches across providers — though two providers can agree by
  coincidence, so the third implementation is where that stops being a claim.

## Status

Early, and deliberately so — the spec precedes the modules, the same way
assay's charter preceded its corpus. Current contents: the design spec and
the iteration plan. First target systems: cortex and myelin.

## Naming

**metafactory** — always lowercase, one word.
