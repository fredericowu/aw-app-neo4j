---
name: aw-neo4j-dependency
description: How an aw-workspace app declares and consumes the shared aw-app-neo4j graph store — the dependency entry, the bolt URI, credential handling, the degrade-never-crash runtime rule, the slug-prefix namespacing/guard-test expectation, and the credential rotation runbook. Use whenever building or reviewing an app that wants its own graph data in Neo4j, or when rotating the shared Neo4j password.
---

# aw-neo4j-dependency — consuming the shared aw-app-neo4j

`aw-app-neo4j` is a Tier-2 app whose container **is** the stock pinned
`neo4j:5.26.4-community` image — no wrapper, no custom code, no UI. Its only
job is to be a dedicated Neo4j reachable at a fixed, documented hostname on
the shared container network, owned by the app framework (install, restart
policy, resource caps, durable data dir, `doctor` visibility), and consumable
by any app that declares it as a dependency. This skill is the frozen
contract — written once, here, so the next consumer's Architect/Coder reads
it instead of re-deriving it.

Full design, alternatives rejected, and the risks behind each rule:
`docs/design/aw-app-neo4j.md` in the `aw-workspace` repo.

## The five contract rules

1. **URI**: `bolt://aw-app-neo4j:7687` (`neo4j://` scheme works equally —
   there is no routing, this is a single instance). Resolvable from the
   workspace process (every Tier-1 app) and from every Tier-2 app container
   on the shared podman network, by aardvark-dns container-name resolution.
   **Not published to the host** — there is no `127.0.0.1:7687` anywhere, by
   design; the network boundary is the shared podman network itself.

2. **Database `neo4j`, user `neo4j`.** Community Edition has exactly one
   user database, so there is nothing to name. The **credential** is a
   single shared password, generated once at install time and stored
   twice-per-consumer:
   - in `aw-app-neo4j`'s own config, field `auth` — the literal
     `neo4j/<password>` pair the image's `NEO4J_AUTH` env var wants (the
     placeholder grammar is whole-value, so this one field holds the full
     pair);
   - in each consumer's own `config_schema`, as a `neo4j_password` field
     (`x-secret: true`, no default).

   Both live in the app-config store (`<AW_WORKSPACE_HOME>/app-config/`),
   which survives app update/uninstall/redeploy.

3. **Install order** — a consumer declares:
   ```jsonc
   "dependencies": {
     "apps": [
       { "id": "neo4j", "required": true,
         "reason": "Dedicated workspace graph store; this app's <X> graph lives in it." }
     ]
   }
   ```
   The reconciler auto-installs `aw-app-neo4j` first and protects it from
   uninstall while a dependent app is loaded.

4. **Runtime posture: degrade, never crash.** This inverts the
   aw-knowledgeable 10-hour outage (a hard runtime dependency on Neo4j took
   the whole app down when Neo4j went unhealthy, and the rollback crashed
   too). A consumer whose `neo4j_password` is empty, or whose bolt
   connection fails, **must boot normally, disable its own graph features,
   and surface the degradation in its own health/status endpoint** — never
   fail its own startup over this dependency. Install-time dependency is
   hard (§3); runtime dependency is soft. Bolt readiness lags
   container-running by 10–30 seconds (JVM boot) — retry with backoff rather
   than failing the first call after install.

5. **Namespacing.** Community is one database, shared by every consumer —
   there is no per-tenant database to isolate behind. Every consumer
   prefixes its node labels with its own slug (e.g. `rs__Paper`, `rs__Theme`
   for a `research-search` consumer), its relationship types likewise
   (`RS_CITES`, `RS_HAS_THEME`), and names every constraint/index
   `<slug>_...`. This is convention, not enforcement — the same
   property/prefix tenancy pattern aw-knowledgeable already runs in its own
   Neo4j instance. Each consumer ships its **own static guard test**
   (aw-knowledgeable's "K5" pattern: grep/AST-check that every Cypher string
   literal in the consumer's codebase stays inside its own label/relationship
   prefix) so a future edit can't silently write outside its slice.

That is the whole contract: a hostname, a database name, a password handed
through config, an install-order declaration, a crash-isolation rule, and a
naming convention.

## What this does NOT give you

- **No per-consumer isolation.** One shared credential — any consumer that
  can reach the container can read/write any other consumer's subgraph
  (bounded only by the namespacing convention above, which is advisory, not
  enforced by Neo4j). Acceptable at today's scale (one consumer); see the
  rotation list below for why this matters more as consumers are added.
- **No backup.** `aw-app-neo4j` ships with none in v1. **A consumer that
  wants to store primary (non-rebuildable) data in this instance must ship
  its own backup first** (a stop-the-container dump to S3, modeled on
  `aw-stack/scripts/backup-neo4j.sh`) — this instance is designed for data
  that is a *projection*, rebuildable from a Postgres (or other) system of
  record by re-running an export.
- **No APOC/GDS plugins.** Plain Cypher only, until a consumer names a
  concrete need (`NEO4J_PLUGINS` is a one-line manifest addition when that
  happens).
- **No human-facing UI.** Neo4j Browser connects from the user's own browser
  over a bolt websocket, which is deliberately unreachable from outside the
  shared network — there is no window for this app. Inspect the database
  directly: `docker exec -it aw-app-neo4j cypher-shell -u neo4j -p <password>`.

## Credential rotation runbook

Rotating the shared password touches **every place it is stored** — today
that is 2 places, and it grows by one for every new consumer. Keep this list
current whenever a consumer is added or removed:

| # | Where | What to set |
|---|---|---|
| 1 | `aw-app-neo4j`'s own config (`POST /api/apps/neo4j/config`) | `auth`: the full `neo4j/<new-password>` pair |
| 2 | *(pending — no consumer live yet)* `research-search`'s own config | `neo4j_password`: the new password half, once that app's dependency wiring ships (tracked separately, card `4025bf3b-9510-81c9-8daa-ec144a45a9b4`) |

After updating #1, restart `aw-app-neo4j` so the container actually picks up
the new `NEO4J_AUTH` value, **then** update every consumer — a consumer still
holding the old password degrades per rule 4 above rather than crashing, but
it does lose graph access until its own config is updated too.

Generate the new password, write it to secret-store-backed config, never to
a file or a transcript — same discipline as any other production credential
rotation in this workspace (see the `rotating-production-neo4j-touches-five-places-not-two`
lesson for what happens when a rotation list like this one goes stale).

## The silent-failure trap

`expand_env` (aw-workspace core, `src/apps/containers.py`) **drops an
unresolved `${config.auth}` placeholder entirely** rather than failing the
install. If `aw-app-neo4j`'s `auth` config field is ever empty when the
container starts, the container boots fine with Neo4j's own default locked
credential (`neo4j/neo4j`, forced password change) — and every consumer then
401s against a container that `docker ps` and the app's own status report as
healthy. This is this workspace's canonical silent-degradation failure mode.
**Never trust "container is running" as proof the credential took** — verify
with an authenticated query (`RETURN 1` over bolt, or the HTTP transaction
API) every time the `auth` config changes.
