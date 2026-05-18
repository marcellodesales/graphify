---
name: planeo-ai-workspace-feeder
description: Maintain ANY Graphify output directory (graphify-out/) and Planeo supply-chain outputs (planeo-graph/) as a live, queryable Graphiti workspace stored in Neo4j via the Graphiti MCP server. Use this skill whenever the user asks to “feed”, “sync”, “publish”, “ingest”, “load”, or “keep updated” graphify-out/planeo-graph reports into Graphiti/Neo4j, when wiring the Planeo foundation after the planeo-ai skill generates graphify-out and planeo-graph, when exposing graph reports as MCP resources (graphify://reports…), or when building a repeatable workflow to keep multiple repositories’ graphs accessible to agents.
---

# Planeo AI Workspace Feeder (Graphify-out → Graphiti/Neo4j via MCP)

This skill is the **foundation step AFTER** `skills/planeo-ai` has generated:
- a Graphify source graph directory: `graphify-out/`
- and (optionally) a Planeo supply-chain graph directory: `planeo-graph/`

Its job is to turn those **local generated artifacts** into a **maintained Graphiti workspace**:
1) **Raw artifacts remain readable** via MCP resources (cheap, exact, no LLM cost).
2) **Curated summaries + gaps + classifications** are **ingested** into Graphiti memory (searchable, composable).
3) The feed is **repeatable and idempotent**, so you can re-run it whenever the outputs change.

## Mental model

Treat the outputs as two complementary “channels” of truth:

- **Channel A: File truth (source of record)**
  - `graphify-out/` and `planeo-graph/` are the canonical artifacts.
  - We expose them as **MCP resources** so any agent can read exact files on-demand.

- **Channel B: Graphiti memory truth (query + recall)**
  - We ingest **small, high-signal episodes** into Graphiti so an agent can:
    - search for “what are the top gaps?”
    - recall “what’s the repo architecture?”
    - compare “what changed since last run?”
  - We avoid ingesting massive raw graphs unless explicitly needed.

## Non-goals
- Do NOT generate `graphify-out/` or `planeo-graph/` here. If missing, run `planeo-ai` first.
- Do NOT assume you can prove supply-chain controls without evidence. If uncertain, store a “gap” or “missing evidence” note.
- Do NOT store or echo secrets found in repo values/config. Store only **keys**, **paths**, and **masked** indicators.
## Preconditions (must validate before feeding)
Before doing anything destructive (like clearing a Graphiti group), validate:
- `graphify-out/` exists and contains at least:
  - `GRAPH_REPORT.md`
  - `manifest.json`
  - `graph.json`
- If Planeo outputs are expected, `planeo-graph/` exists and contains:
  - `graph.json`
  - `gaps.json` and `classifications.json` (if your Planeo run generates them)
- Graphiti MCP server is reachable and healthy:
  - `GET /health` returns healthy
  - MCP endpoint responds on `/mcp/` (streamable HTTP)
If any of the inputs are missing, stop and run `planeo-ai` to regenerate the workspace outputs.
## Workspace identity and naming rules
Graphiti’s `group_id` is the primary namespace boundary. It must be deterministic.
Rules:
- Use the same `workspace_group_id` across re-feeds of the same repo (unless you intentionally want history).
- Prefer `planeo:workspace:<slug>` where `<slug>` is stable (repo name or org-repo).
- Avoid embedding absolute local paths.
- If you need environment-specific graphs, suffix with a stable environment label (e.g. `:dev`, `:prod`) rather than a timestamp.
Recommended episode name prefix:
- `planeo/workspace/<workspace_group_id>/...`
This makes it easy to filter/search episodes.
## Workspace registry (optional but recommended)
If you are managing many repos, keep a lightweight registry in Graphiti:
- Use a dedicated `group_id` like `planeo:registry`.
- Add/update one “workspace registry” episode per workspace with:
  - `workspace_group_id`
  - human name + repo remote (if available)
  - last feed timestamp
  - manifest hash
  - resource URI entry point (`graphify://reports`)

## Inputs (expected artifacts)

### Minimum viable input (Graphify)
- `graphify-out/GRAPH_REPORT.md` (human summary)
- `graphify-out/graph.json` (source graph)
- `graphify-out/manifest.json` (file hashes / change detection)

### Optional but recommended input (Planeo)
- `planeo-graph/graph.json`
- `planeo-graph/classifications.json`
- `planeo-graph/gaps.json` (or `planeo-graph/gaps.md` if used)
- `planeo-graph/evidence.json`
- `planeo-graph/report.txt` or `planeo-graph/GRAPH_REPORT.md`

### Optional convenience artifacts
- `graphify-out/*combined*.json` (a merged “one file” graph that references both layers)
- `graphify-out/cypher.txt` / `planeo-graph/cypher.txt` (optional Neo4j import scripts)

## Desired end state

A running Graphiti MCP server backed by Neo4j that provides:

1) MCP **resources**:
- `graphify://reports` → lists mounted artifacts
- `graphify://reports/json/{filename}` → reads JSON artifacts
- `graphify://reports/md/{filename}` → reads Markdown artifacts
- `graphify://reports/html/{filename}` → reads HTML artifacts
- `graphify://reports/txt/{filename}` and `graphify://reports/text/{filename}` → reads text artifacts

2) Graphiti **tools** populated for a workspace group:
- episodes added with `add_memory(..., group_id=<workspace_group_id>, ...)`
- searchable with `search_nodes` / `search_memory_facts`

## Core strategy (repeatable pipeline)

### Step 0 — Identify the workspace

Choose a deterministic `workspace_group_id` for Graphiti.

Recommendations:
- Use a stable “slug” derived from repository identity, not the absolute path.
- Keep it short and URL/Neo4j-friendly.

Good examples:
- `planeo:workspace:org-repo`
- `planeo:workspace:repo-name`
- `planeo:workspace:<repo-name>:<branch-or-environment>` (only if you intentionally want multiple graphs)

Also record:
- `generated_at` timestamps from outputs
- a `manifest_hash` (hash of `graphify-out/manifest.json`)
- the set of artifact filenames and sizes

### Step 1 — Ensure Graphiti (Neo4j) is running
Preferred architecture for Planeo foundation:
- Graphiti MCP server uses **Neo4j** as the persistence backend.
- Neo4j data persists across restarts via Docker volumes.
Operational notes:
- Concurrency is controlled by `SEMAPHORE_LIMIT`.
- You can run with local OpenAI-compatible endpoints (Ollama/LM Studio) to reduce cost.
#### Reference deployment (Docker Compose)
Use the Graphiti MCP server’s Neo4j compose file as the reference stack:
- Services: `neo4j` + `graphiti-mcp` (and optionally a `neo4j-cypher` MCP sidecar)
- Ports: Neo4j HTTP/Bolt + Graphiti MCP HTTP (streamable)
Mount contract (critical):
- Set `GRAPHIFY_OUT_HOST_DIR` to the project’s `graphify-out/` directory on the host
- Mount it into the `graphiti-mcp` container at `/data/graphify-out` read-only
- Set `GRAPHIFY_OUT_DIR=/data/graphify-out` inside the container
Example (generic):
```bash
GRAPHIFY_OUT_HOST_DIR="$(pwd)/graphify-out" \
  docker compose -f <graphiti_repo>/mcp_server/docker/docker-compose-neo4j.yml up -d --build
```
Validation commands (generic):
- `curl -sS http://localhost:8000/health`
- MCP inspector (optional) or a small MCP client probe to `http://localhost:8000/mcp/`
If you want reproducibility, prefer `--build` only when the server code changes. Otherwise, keep a stable tag for the patched server image (see caching section below).

### Step 2 — Mount graphify-out into Graphiti MCP and expose as resources (Channel A)

This is the critical “make the reports available” step.

#### A) Server capability requirement

The Graphiti MCP server must register **resource templates** (FastMCP `@mcp.resource`) that read files from a mounted directory.

The resource contract should be:
- safe (no path traversal)
- read-only
- minimal mime typing
- directory configured by env var (default `/data/graphify-out`)

Resource URIs (recommended, stable):
- `graphify://reports` (list)
- `graphify://reports/json/{filename}`
- `graphify://reports/md/{filename}`
- `graphify://reports/html/{filename}`
- `graphify://reports/txt/{filename}`
- `graphify://reports/text/{filename}`

#### B) Docker mount contract

Mount the host project’s `graphify-out/` directory into the container:
- Host: `${GRAPHIFY_OUT_HOST_DIR}`
- Container: `/data/graphify-out`
- Mount mode: `:ro`

And set:
- `GRAPHIFY_OUT_DIR=/data/graphify-out`

#### C) Verification (must do)
After starting Graphiti MCP:
- list resource templates and confirm `graphify://reports/...` templates exist
- read `graphify://reports`
- read a known file, e.g. `graphify://reports/json/<some-json-file>`
If resources are missing, stop and fix the server build or deployment before proceeding.
Concrete verification approach (recommended):
- Use a real MCP client (streamable HTTP), not raw curl.
- If Graphiti is running in Docker, you can run a client probe from inside the container’s venv.
Probe outline:
- connect to `http://localhost:8000/mcp`
- `list_resource_templates()`
- `list_resources()`
- `read_resource('graphify://reports')`
- `read_resource('graphify://reports/json/<filename>.json')`
This should work even when the graph database/LLM is slow, because it exercises only the resource layer.

### Step 3 — Ingest curated episodes into Graphiti memory (Channel B)

Goal: make “high-signal” information searchable, without paying to ingest huge JSON graphs.

#### What to ingest (recommended default)
Ingest these as episodes (in order):
1) **Workspace metadata episode** (small, always)
- Includes workspace_group_id, generation timestamps, manifest hash, file list, and pointers to resource URIs.
- Store as JSON for easy downstream parsing.
Suggested metadata shape:
```json
{
  "workspace_group_id": "planeo:workspace:my-repo",
  "generated_at": {
    "graphify_out": "<timestamp if known>",
    "planeo_graph": "<timestamp if known>"
  },
  "manifest": {
    "path": "graphify-out/manifest.json",
    "hash": "<sha256>"
  },
  "resources": {
    "reports_index": "graphify://reports",
    "combined_graph": "graphify://reports/json/<combined>.json"
  }
}
```
2) **Graphify report episode**
- `graphify-out/GRAPH_REPORT.md` as `source=text`
3) **Planeo supply-chain report episode** (if present)
- `planeo-graph/report.txt` or `planeo-graph/GRAPH_REPORT.md` as `source=text`
4) **Gaps episode** (if present)
- `planeo-graph/gaps.json` as `source=json`
5) **Classifications episode** (if present)
- `planeo-graph/classifications.json` as `source=json`
6) **Evidence index episode** (if present)
- `planeo-graph/evidence.json` as `source=json`
#### Tool call shapes (Graphiti MCP tools)
When feeding Graphiti memory, use these tool calls:
- Clear (Policy A):
  - `clear_graph({"group_ids": [workspace_group_id]})`
- Add episode:
  - `add_memory({
      "name": "planeo/workspace/<gid>/graphify/GRAPH_REPORT.md",
      "episode_body": "<file contents as string>",
      "group_id": "<gid>",
      "source": "text" | "json" | "message",
      "source_description": "graphify-out/GRAPH_REPORT.md"
    })`
- Verify:
  - `get_episodes({"group_ids": [workspace_group_id], "max_episodes": 10})`
  - `search_nodes({"query": "SBOM gaps", "group_ids": [workspace_group_id], "max_nodes": 10})`
Important:
- For `source=json`, `episode_body` must be a JSON string (already serialized), not a Python dict.
- Treat `clear_graph` as destructive: only use it when the user intent is “refresh/rebuild” or when you are establishing a brand-new workspace group.
- `add_memory` typically queues work asynchronously; searches may be empty for a short period even though ingestion is in progress. Poll `get_episodes` (and then `search_nodes`) with a short backoff before declaring failure.

#### What NOT to ingest by default

- `graphify-out/graph.json` or `*combined*.json` (often large)

Instead:
- keep them as MCP resources
- optionally ingest a *digest* (counts, top hubs, top communities, top gaps) if you need searchability

#### Update policy (idempotency)

Choose ONE update policy:

- **Policy A (simple, recommended): clear-and-refeed**
  - call `clear_graph(group_ids=[workspace_group_id])`
  - re-add the curated episodes
  - benefit: deterministic state
  - cost: deletes history

- **Policy B (audit/history): append new run**
  - keep the workspace group stable
  - add episodes with a timestamp in the name
  - store “current pointer” in a metadata episode
  - benefit: timeline
  - cost: graph grows

- **Policy C (advanced): stable UUIDs per artifact**
  - compute deterministic UUID per (workspace, artifact name, content hash)
  - re-add only changed artifacts
  - benefit: incremental
  - cost: more logic

Default to **Policy A** unless the user explicitly wants history.

#### Minimum verification

After ingestion:
- `get_episodes(group_ids=[workspace_group_id], max_episodes=10)` should return the episodes
- `search_nodes(query="gaps" | "SBOM" | "Helm", group_ids=[workspace_group_id])` should return something meaningful

## Multi-workspace strategy

Graphiti needs to serve many repos over time. There are two patterns:

### Pattern 1 — One Graphiti instance, many group_ids (recommended)
- Keep ONE Neo4j + Graphiti MCP server.
- For each repo, pick a distinct `workspace_group_id`.
- Ingest curated episodes per group.

For MCP resources, you still need file access. Options:
- re-mount the active workspace’s `graphify-out/` when you work on it
- OR mount a parent directory containing multiple workspaces and add a `workspace` parameter to resource URIs (requires server code)

### Pattern 2 — One Graphiti instance per workspace (simple but heavy)
- run multiple Graphiti containers on different ports
- each mounts exactly one `graphify-out/`
- mostly useful for isolated demos

## Practical improvements (performance, cost, reliability)

### Docker image caching / reproducibility
Goal: make “workspace feeding” fast enough to be used constantly.
Recommendations:
- Build a dedicated image tag for the Planeo-patched Graphiti MCP server (resources enabled).
- Pin versions:
  - Graphiti MCP server version
  - graphiti-core version
  - Neo4j image version
- Prefer a local stable tag (or a registry tag) so you can restart without rebuilding.
- Use BuildKit caching:
  - keep a local cache for Python dependencies (uv already supports cache mounts in Docker builds)
  - avoid rebuilds unless server code changes
- If you manage many workspaces, consider a “base image” that rarely changes plus a thin “planeo patch” layer.
- If you have CI, publish the patched image to GHCR and pull it locally; this makes setup reproducible across machines.

### Avoid expensive ingestion

- Prefer resources for large JSON
- Ingest “thin” digests:
  - node/edge counts
  - top communities
  - top gaps summary
  - directory classification summary

### Change detection

- Hash `graphify-out/manifest.json` and store it in the workspace metadata episode.
- If unchanged since last feed, skip the ingestion.

### Security hardening

- Always mount `graphify-out/` read-only.
- Do not allow slashes in filenames for resource reads.
- Do not ingest plaintext secrets. If detected, create a Gap episode entry that points to file path + line range without including the secret value.

## Troubleshooting quick hits

- If HTTP returns 406 “must accept text/event-stream”: you are calling the endpoint incorrectly; MCP clients should use streamable HTTP.
- If resources list is empty: the server likely doesn’t implement resources; fix server build first.
- If `/data/graphify-out` is missing in container: your volume mount is wrong or `${GRAPHIFY_OUT_HOST_DIR}` wasn’t set.
- If tool calls succeed but searches return nothing: LLM/embedder may be misconfigured; still keep resources as the fallback source-of-truth.

## Reference implementation contract (what the Graphiti server must support)
This skill assumes the Graphiti MCP server supports **both**:
- Graphiti memory tools (add/search/clear)
- Planeo “reports as resources” (graphify://reports...)
Minimum required server behaviors:
- Streamable HTTP MCP endpoint at `/mcp/`
- Resource templates:
  - `graphify://reports/json/{filename}`
  - `graphify://reports/md/{filename}`
  - `graphify://reports/html/{filename}`
  - `graphify://reports/txt/{filename}`
  - `graphify://reports/text/{filename}`
- Reads from `GRAPHIFY_OUT_DIR` (default `/data/graphify-out`)
- Enforces safe filename rules (no slashes, no traversal)
If any of these are missing, fix the Graphiti deployment first; do not “paper over” missing resources by ingesting huge artifacts into memory.
## Optional advanced: Neo4j native import of Graphify graphs
If you want direct Cypher queries over the code graph (outside Graphiti memory semantics):
- import `graphify-out/cypher.txt` into Neo4j using a separate label namespace (e.g., `:Code`, `:Rationale`) to avoid collisions.
- keep Graphiti’s internal nodes/edges separate.
This is optional and should only be done if you have a concrete query/use-case.
