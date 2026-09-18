# linked-fw

First-party packages of the **Linked** framework, published to npm under `@_linked`.

Community packages live in [linked-cm](https://github.com/linked-cm) under `@linked.cm`.

## Shared CI/CD

Every package repo calls two reusable workflows from this repository rather than carrying its
own copy:

| Workflow | Trigger | What it does |
|---|---|---|
| `publish.yml` | push to `main` | Opens and merges the changesets version PR, then publishes |
| `pr.yml` | PR to `main` | `Build & Test`, and a changeset reminder |

### Publishing

A package with a **trusted publisher** registered on npm publishes straight to `latest` over
OIDC — no token, no approval. A package without one falls back to `npm stage publish`, which
uploads the version and holds it for a maintainer to approve with 2FA. Either way the release
happens; the log line says which route it took.

To move a package onto the fast path, register a trusted publisher on npmjs.com for it with
owner `linked-fw`, the repo name, and workflow **`publish.yml`**. That filename is the one npm
validates, so a caller stub must never be renamed.
