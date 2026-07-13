# Architecture rules

The code-level rules blizzard is held to.
These carry over the CLEAN-architecture aspects practiced in winter-cli, plus a repository-access rule specific to blizzard.
Once `blizzard-harness` exists these rules graduate there ([ecosystem.md](./ecosystem.md)); until then this file is the owner.

## CLEAN aspects (as practiced in winter-cli)

- **Screaming architecture.** Functionality is grouped by domain and the grouping is *named after the domain* — the top-level layout announces what the system does, not what framework it uses.
- **Domain layer at the core.** Business rules live in a domain layer with no outward dependencies; frameworks, storage, and transport sit outside it.
- **Dependency inversion.** Concepts are separated by interfaces owned at the inner layer; outer layers implement them. The winter-harness repository-pattern exemplar (I-prefix Protocol seam + `internal/` adapter + factory-injected errors) is the reference shape.
- **Dependency injection.** Wiring happens at the composition root; nothing constructs its own collaborators.

## Repository access rules

The repository pattern is split into **read-only repositories** and **write repositories**, and access is layer-gated:

| Layer | Read-only repositories | Write repositories |
|-------|------------------------|--------------------|
| Controllers (API/CLI edges) | allowed | **forbidden** |
| Domain layer | allowed | allowed |

All mutation flows through the domain layer — a controller can answer a query directly from a read model, but it can never write around the business rules.

## Domain takes objects, not identifiers

Domain-layer operations receive **domain objects**, never raw identifiers.
Resolution from identifier to object is an edge concern (controller or application service); by the time the domain is invoked, the entities it operates on are already loaded and typed.
