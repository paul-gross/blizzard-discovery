# Story map

One box per epic from the [registry](./epics.md) — canonical id plus a short name — with arrows pointing to the next work, in [roadmap](./roadmap.md) milestone order. An epic appearing twice lands in slices.

```text
milestone:mvp — the local fleet: the walking skeleton

┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│ epic:hub         │───▶│ epic:store       │───▶│ epic:supervisor  │───▶│ epic:adapters    │
│ orchestrator     │    │ runner store     │    │ reconcile loop   │    │ claude code      │
└──────────────────┘    └──────────────────┘    └──────────────────┘    └─────────┬────────┘
                                                                                  │
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐    ┌─────────▼────────┐
│ epic:ask-answer  │◀───│ epic:delivery    │◀───│ epic:review      │◀───│ epic:workflow    │
│ Q&A at the hub   │    │ deliver node     │    │ review node      │    │ graph engine     │
└─────────┬────────┘    └──────────────────┘    └──────────────────┘    └──────────────────┘
          │
┌─────────▼────────┐
│ epic:board       │
│ local web app    │
└─────────┬────────┘
          │
          │  milestone:centralized-hub — control plane + spokes
          ▼
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│ epic:hub         │───▶│ epic:board       │───▶│ epic:ask-answer  │───▶│ epic:chat        │
│ remote deployment│    │ remote board/PWA │    │ remote Q&A       │    │ bot + auth       │
└─────────┬────────┘    └──────────────────┘    └──────────────────┘    └──────────────────┘
          │
          │  later — in any order; the mvp core's contracts stay frozen
          ▼
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│ epic:batching    │    │ epic:adapters    │    │ epic:gates       │
│ packer → planner │    │ opencode + codex │    │ gate nodes+dial  │
└──────────────────┘    └──────────────────┘    └──────────────────┘
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│ epic:config      │    │ epic:ci-feedback │    │ epic:team        │
│ all-as-config    │    │ CI → session     │    │ multi-operator   │
└──────────────────┘    └──────────────────┘    └──────────────────┘
┌──────────────────┐
│ epic:migration   │
│ graph lifecycle  │
└──────────────────┘
```
