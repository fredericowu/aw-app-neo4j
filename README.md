# aw-app-neo4j

A dedicated Neo4j graph database your other apps can share. Install it once
and any app that declares it as a dependency gets a durable graph store on
the workspace's own container network — no setup beyond a password.

One container: the stock, pinned `neo4j:5.26.4-community` image — the exact
image an earlier production Neo4j (aw-knowledgeable's) already runs. No
wrapper, no custom code, no UI. The app's entire value is a hostname other
apps can rely on and the framework's own install/restart/resource-cap/backup
machinery around that one container.

Design: `docs/design/aw-app-neo4j.md` in the `aw-workspace` repo — the
contract frozen there (§2) is what this app ships, and what
`skills/aw-neo4j-dependency/SKILL.md` teaches a consumer to use.

## Using it

A consumer app declares the dependency:

```jsonc
"dependencies": {
  "apps": [
    { "id": "neo4j", "required": true,
      "reason": "Dedicated workspace graph store; this app's <X> graph lives in it." }
  ]
}
```

and connects to `bolt://aw-app-neo4j:7687`, database `neo4j`, user `neo4j`,
with a password carried in its own config (`x-secret: true`, no default) —
**never** hardcoded and never read from this app's config directly. Full
contract, including the runtime degrade-never-crash rule and the
namespacing convention every consumer must follow on a single shared
database: `skills/aw-neo4j-dependency/SKILL.md`.

## Configuring it

This app's own `auth` config field is the ONLY place the shared credential
is set — and it is not optional. Generate a strong password and set it
**before** first start:

```bash
# via the workspace API (see aw-autoskill-local-workspace-api-testing)
POST /api/apps/neo4j/config
{"auth": "neo4j/<strong-password>", "auto_start": true}
```

Left empty, the container boots fine with Neo4j's own locked default
credential (`neo4j/neo4j`) and every consumer 401s — while `docker ps` and
this app's own status both report healthy. **Verify with an authenticated
query, not a running container** — see the skill's "silent-failure trap"
section for why `${config.auth}` resolving to an empty string is dropped
silently rather than failing the install.

Inspect the database directly (there is no web UI window — Neo4j Browser
connects from the user's own browser over a bolt websocket, and that port is
deliberately unreachable from outside the shared network):

```bash
docker exec -it aw-app-neo4j cypher-shell -u neo4j -p <password>
```

## What this does NOT give you (yet)

- No per-consumer credential isolation — one shared password, one database.
- No backup. A consumer's data in here must be a rebuildable projection of
  its own system of record, not primary data, until a backup ships.
- No APOC/GDS plugins — plain Cypher only.

All three are named triggers in the design doc (§6), not oversights — see
that section before deciding one of them needs solving now.

## Development

```bash
python3 tests/validate_manifest.py aw-app.json
```

This app ships no Python package and no frontend, so there is nothing else
to test locally — `aw-app.json`, the skill, and this README are the entire
delivery.
