## What does this PR do?

<!-- One sentence. Reference the spec artifact it answers: T-xx of tasks.md, a CA, or an ADR. -->

## Type of change

- [ ] New functionality (`feat`)
- [ ] Bug fix (`fix`)
- [ ] Documentation only (`docs`)
- [ ] Refactor, no behavior change (`refactor`)
- [ ] Tests (`test`)
- [ ] Tooling, dependencies, CI (`chore`)

## Spec traceability

<!-- Required. Which requirement does this satisfy? -->

| Field | Value |
|---|---|
| Task | `T-xx` — <!-- from spec-python/tasks.md --> |
| Requirement | <!-- HU-xx · CA-xx.x · RN-x · RNF-x · Article N · ADR-00x --> |
| Source document | <!-- e.g. spec-python/data-model.md §3 --> |

## How was it verified?

<!-- Article XII: evidence that runs, not "done". Paste the command and its output. -->

```bash
# command
# output
```

- [ ] Evidence exists and **passes** when executed
- [ ] No secret, `.env` or credential is committed (Article IX)

## Constitution

<!-- Tick only what this PR actually proves. -->

- [ ] I — `Domain` imports nothing external (no `Illuminate\`, no `Carbon\`)
- [ ] II — ports are declared by the consumer, no framework type in a port signature
- [ ] III — no infrastructure is instantiated outside `Bootstrap/Composition/PortBindings.php`
- [ ] V — all DDL lives in migrations; `infra` has no `CREATE TABLE`
- [ ] VI — every engine-expressible invariant is declared in the engine
- [ ] VII — nothing derived is stored (`status`, `days_late`)
- [ ] VIII — frozen values never read the live catalog
- [ ] IX — no secret in the repo or in a log
- [ ] XI — code, columns and routes in English; user messages in Spanish
- [ ] XII — the task is done only with executable evidence

## Scope guard

- [ ] The project scope is unchanged
- [ ] The data model is unchanged: no table, column, index or constraint added or removed
- [ ] The HTTP contract is unchanged: no route, status code or error code added

## Checklist

- [ ] Branch is off `dev`, not off `main`
- [ ] Subject follows `[type]([scope]): lowercase imperative, no trailing period`
- [ ] Branch name follows `[type]/[description-in-kebab-case]`
- [ ] Green CI
- [ ] Reviewed (1 approval)

## Screenshots or output

<!-- For UI changes or console evidence. -->
