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

## Reference implementation (`:maturity :implemented`)

Full itonami Actor pattern (per ADR-2607011000 / CLAUDE.md's Actors
section, alongside `cloud-itonami-isco-6130`, `-8160`, `-2166`, `-2641`,
`-2651`, `-2652`, `-2654`, `-1219`, `-1223`, `-1330`, `-1341`, `-1349`
and `-1412`): a real
[`kotoba-lang/langgraph`](https://github.com/kotoba-lang/langgraph)
`StateGraph`, with the Advisor and Governor as distinct graph nodes and
human-in-the-loop interrupt/resume via checkpointing.

```text
:intake -> :advise -> :govern -> :decide -+-> :commit            (:ok? true)
                                           +-> :request-approval   (:escalate? true, interrupt-before)
                                           +-> :hold               (:hard? true)
```

- `src/services_management/store.kotoba` — `Store` protocol +
  `MemStore`: registered clients, committed records, an append-only
  audit ledger.
- `src/services_management/advisor.kotoba` — `Advisor` protocol;
  `mock-advisor` (deterministic, default) proposes a services operation
  from a request; `llm-advisor` wraps a `langchain.model/ChatModel` —
  either way the advisor only ever produces a `:propose`-effect
  proposal, never a committed record, and LLM parse failures always
  yield `confidence 0.0` (forces escalation, never fabricated
  confidence).
- `src/services_management/governor.kotoba` —
  `ServicesManagementGovernor/check`: a pure function, wired as its own
  `:govern` node. Hard invariants (unregistered client, a proposal
  whose `:effect` isn't `:propose`) always route to `:hold`. Escalation
  invariants (`:clear-quality-failure`, `:approve-contract-exception`,
  or low advisor confidence) always route to `:request-approval` — an
  `interrupt-before` node that the graph checkpoints and only resumes
  on explicit human approval (`actor/approve!`), matching the README's
  robotics-premise statement that clearing a service-quality failure
  without review and approving a client-contract exception always
  require human sign-off.
- `src/services_management/actor.kotoba` — `build-graph`,
  `run-request!`, `approve!`: the `langgraph.graph/state-graph` wiring
  itself.

```bash
kbb -M:test
```

This is what backs this repo's `:maturity :implemented` entry in
[`kotoba-lang/occupation`](https://github.com/kotoba-lang/occupation).

## License

AGPL-3.0-or-later.
