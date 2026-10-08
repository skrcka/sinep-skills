---
name: architecture-docs
description: Write or update a project's architecture documentation in docs/architecture — how this service links to the other services around it, the containers inside it, and its layered dependency tree derived from the code, with Mermaid diagrams. Use when asked to document, map, explain or diagram a project's architecture, onboard someone to a codebase, or refresh architecture docs after changes.
---

# Architecture docs

Document a project so a newcomer (person or agent) can answer three questions fast, from the outside in:

1. **Where does this sit?** Which other services it talks to, in which direction, over what. Each one links to *that* service's architecture docs.
2. **What is it made of?** The things that run or store data: processes, databases, queues, jobs.
3. **How is the code layered?** The layers, which way dependencies may point, and the actual module dependency tree, including where the code breaks the rules.

Everything goes in `docs/architecture/`, in the same layout in every repo, so the docs of many services link into one navigable map.

## Layout

```
docs/
  architecture/
    README.md            entry point: one-paragraph summary, reading order, "verified at <commit>, <date>"
    context.md           level 1: this system among other services and actors
    services.yaml        the neighbours, machine-readable (see below)
    containers.md        level 2: deployable/runtime units inside this project
    layers.md            level 3: layers, allowed directions, module dependency tree, violations
    modules/<module>.md  one per major module/package: purpose, public surface, depends on, used by
    flows/<flow>.md      2-5 key runtime flows (requests, jobs, events) as sequence diagrams
    data.md              stores and the main entities (ER diagram), who owns what
    decisions/NNNN-<slug>.md   architecture decision records (only real, findable decisions)
    diagrams/<name>.mmd  every Mermaid diagram as its own file
```

Small project? Fold `containers.md`, `data.md` and `flows/` into fewer files, but keep `README.md`, `context.md`, `services.yaml` and `layers.md`: those four are the contract other repos and agents rely on. If `docs/README.md` exists, link `architecture/README.md` from it; if not, create a short one that does.

### Diagrams

- One diagram per file in `docs/architecture/diagrams/*.mmd`, embedded where it is discussed with
  `![Context](diagrams/context.mmd)`. Viewers that render Mermaid files (sinep's preview, many IDEs) show the image. Add a one-line text summary under each diagram for everything else.
- Plain Mermaid only: `flowchart`, `sequenceDiagram`, `erDiagram`, `classDiagram`, `stateDiagram-v2`. No `click`, no HTML in labels, no theme directives.
- At most ~25 nodes per diagram. Split by layer or subsystem rather than drawing one wall.
- Node ids are stable identifiers (`api`, `billing_db`); labels are human (`api["HTTP API (axum)"]`).

### services.yaml

```yaml
# Neighbours of this service. Other repos' agents read this to link back.
service: billing            # this project's name, as other services call it
docs: https://git.example.com/acme/billing/src/branch/main/docs/architecture/README.md
neighbours:
  - name: auth
    repo: https://git.example.com/acme/auth
    docs: https://git.example.com/acme/auth/src/branch/main/docs/architecture/README.md
    direction: outbound      # outbound = we call them, inbound = they call us, both
    protocol: HTTPS/JSON     # HTTP, gRPC, AMQP topic x, Postgres, S3 bucket y, webhook, ...
    purpose: validate session tokens
    evidence: server/src/auth_client.rs, AUTH_URL in deploy/.env.example
```

`docs` links point at the neighbour's `docs/architecture/README.md` when its repo is known, even if that file doesn't exist yet: once that repo is documented with this same skill, the link resolves. Unknown repo → leave `repo`/`docs` empty and say so in `context.md`.

## How to do it

Work from evidence, not from names. Cite file paths for every claim a reader might doubt.

### 1. Survey (read, don't write yet)

- Manifests and entry points: `Cargo.toml`, `package.json`, `go.mod`, `pyproject.toml`, `pom.xml`/`build.gradle`, `Dockerfile*`, `docker-compose*.yml`, k8s/helm, CI files, `main`/`cmd/`/`bin/`.
- **External links**: config and env (`.env.example`, settings modules, helm values), HTTP/gRPC clients, SDK imports (`stripe`, `aws-sdk`, `@google-cloud/*`), queue topics, webhook routes, database URLs, other repos named in README/CI. Every one is a candidate neighbour; keep the file that proves it.
- **Inbound**: routes/handlers, gRPC services, consumers, cron entries, CLI commands — who can call this.
- Existing docs (README, `docs/`, ADRs, comments at the top of modules). Reuse what is still true; correct what isn't.

### 2. The dependency tree (generate it, don't guess)

Get the real import graph between the project's own modules with the language's tooling, falling back to grep:

| Stack | Module graph |
|---|---|
| Rust | workspace crates: `cargo metadata --format-version 1 --no-deps`; inside a crate: `grep -rn "use crate::" src/` (or `cargo modules dependencies` if installed) |
| Go | `go list -f '{{.ImportPath}}: {{join .Imports " "}}' ./...` |
| TS/JS | `npx --yes madge --json src` or `npx --yes depcruise --output-type json src` |
| Python | `grep -rnE "^(from|import) <pkg>" <pkg>/` (or `pydeps --show-deps`) |
| Java/Kotlin | `grep -rn "^import <base.package>"` per package (or `jdeps -verbose:package`) |
| C# | project references in `*.csproj`, then `using` per namespace |

Collapse it to the level a reader thinks in (top-level modules/packages, not files). Then:

1. **Name the layers** the code actually has (e.g. `http → services → domain → storage`, or `ui → state → api client`). Prefer the project's own vocabulary.
2. **State the allowed direction** (usually outer → inner; domain depends on nothing app-specific).
3. **Draw the tree** (`diagrams/layers.mmd`): a `flowchart TB` with one `subgraph` per layer, edges = module imports.
4. **List violations honestly**: every edge that points the wrong way or skips a layer, with the importing file. Don't hide them and don't fix them here — documenting is the job. Note cycles explicitly.
5. Record the exact command you ran in `layers.md` so the next update can regenerate it.

### 3. Write, outside in

- `context.md`: short paragraph, the context diagram (this system in the middle, neighbours and human actors around it, edges labelled with protocol and direction), then a table: neighbour, direction, protocol, purpose, link to its docs, evidence.
- `containers.md`: each runnable/storage unit — what it is, how it is started/deployed, what it talks to, where its config is.
- `layers.md`: layers table (layer, modules, may depend on), the tree diagram, violations, regeneration command.
- `modules/*.md`: only for modules a newcomer must understand; each one in ~10-30 lines: purpose, key types/functions with paths, depends on, used by, gotchas.
- `flows/*.md`: the 2-5 paths that matter most (main request, main background job, auth). Sequence diagram + numbered steps with file references.
- `data.md`: stores, the main entities and relations (ER diagram), and which module owns writes to which table/bucket/topic.
- `decisions/`: only decisions you can point to (a commit, a comment, an existing doc). Format: Context, Decision, Consequences, Status, source link.
- `README.md` last: summary, the context diagram again, links to every file, and `Verified at <git rev-parse --short HEAD>, <date>`.

### 4. Check

- Every relative link and every `![](diagrams/…mmd)` resolves; every `.mmd` parses (if a Mermaid renderer is available, render each one; sinep users can open them in the preview).
- Every neighbour in `context.md` is in `services.yaml` and vice versa.
- Nothing invented: anything you couldn't confirm is marked `TODO(verify): …` with what would confirm it.
- No secrets, tokens, passwords or private keys — refer to config keys by name, never by value.

## Updating existing docs

1. Read `docs/architecture/README.md` for the last verified commit, then `git diff --stat <commit>..HEAD` and `git log --oneline <commit>..HEAD`.
2. Re-run the dependency-graph command from `layers.md`; diff against the documented tree.
3. Touch only what changed (new neighbours, moved modules, new violations or fixed ones), then bump the verified commit/date.

## Style

- Plain, short sentences; present tense; facts with paths (`server/src/orch.rs`).
- Tables for inventories, diagrams for shape, prose only for *why*.
- Write for someone joining tomorrow: what to read first, where the sharp edges are.
- When done, tell the user what you documented, what you couldn't verify, and the layer violations you found.
