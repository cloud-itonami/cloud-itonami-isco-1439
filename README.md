# cloud-itonami-isco-1439

Open Occupation Blueprint for **ISCO-08 1439**: Services Managers Not Elsewhere Classified.

This repository designs a forkable OSS business for an independent services manager: a site-walkthrough robot performs service-quality checklist inspection under a governor-gated actor, so the practice keeps its own coordination and quality records instead of renting a closed services-management SaaS.

## Robotics premise

All cloud-itonami verticals are designed on the premise that a **robot performs
the physical domain work**. Here a site-walkthrough robot performs service-quality checklist inspection and evidence capture under an actor that proposes
actions and an independent **Services Management Governor** that gates them. The governor never
dispatches hardware itself; `:high`/`:safety-critical` actions (such as
clearing a service-quality failure without review, or approving a client-contract exception) require human sign-off.

A live sample of the operator console (robotics safety console, shared template) is rendered in [docs/samples/operator-console.html](docs/samples/operator-console.html) — pure-data HTML output of `kotoba.robotics.ui`.

## Core Contract

```text
service portfolio + client roster + quality standard
        |
        v
Services Advisor -> Services Management Governor -> coordinate/report, or human sign-off
        |
        v
robot actions (gated) + operating records + audit ledger
```

No automated advice can dispatch a robot action the governor refuses, suppress
an operating record, or disclose sensitive data without governor approval and
audit evidence.

## Capability layer

Resolves via [`kotoba-lang/occupation`](https://github.com/kotoba-lang/occupation)
(ISCO-08 `1439`). Required capabilities:

- :robotics
- :identity
- :forms
- :dmn
- :bpmn
- :audit-ledger

See [`docs/business-model.md`](docs/business-model.md) and
[`docs/operator-guide.md`](docs/operator-guide.md).

## License

AGPL-3.0-or-later.
