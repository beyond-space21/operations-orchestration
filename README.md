# AI Operations Platform

A hierarchical workforce: admins delegate objectives to a central agent, worker agents coordinate bound human assistants, and PostgreSQL holds the authoritative operational state.

Phase 1 is identity and agent configuration. The person who registers an organization is its Super Admin, and the organization keeps at least one. A Super Admin can add an Admin; an Admin can add a human assistant. Phase 2 is the work itself. An admin assigns a worker agent, the bound human assistant responds and submits evidence, and the platform verifies completion against the work item's criteria. A human saying the work is done does not verify it.

Phase 3 is the agent runtime. One loop serves every agent. Gemini plans a central turn; GLM handles the tool turn that follows and every worker turn. A tool runs only after the agent's allowlist and the organization policy agree. `request_verification` records a proposal. An admin approval is what calls verification, and the work stays open when the evidence is missing. `POST /api/agents/{id}/turns` starts a turn. `GET /api/agents/{id}/runs` reads the stored provider, tools, and output. `POST /api/proposals/{id}/decision` approves or rejects a proposal.

Phase 4 is memory. `POST /api/memory/records` stores a fact only when the claim matches a source row the actor can see. A claim that work is verified is rejected while the work item is still open. A later fact supersedes the earlier one, and the earlier row stays. `POST /api/memory/recall` filters by organization and scope before it ranks. Facts outrank summaries. A turn receives at most five matching memories, and both model calls in that turn receive that same slice.

Phase 5 is the desk. `make desk` serves it at `http://127.0.0.1:5173` and proxies `/api` to the API on port 8000. An admin talks with the central agent. A human assistant talks only with the worker agents they are assigned to, and switches among those agents when they have more than one. An agent may have several assistants, or none. A task that needs a person names that person; removing the assignment hands open tasks to another assistant or escalates them to the agent. Tasks stay in the left panel. The worker manages those tasks. A reply can use Markdown and HTML. A diagram is drawn from a Mermaid fence after the agent loads that diagram type's reference. A declaration is stored. It does not verify the work. Times are shown in Asia/Kolkata.

Phase 6 is a derived relationship graph. FalkorDB stores Graphiti-shaped edges. The platform writes them from committed work, bindings, and memory rows after the outbox commit. Valid time and recorded time stay distinct. `relate` answers a fixed set of questions for the actor's authorized scopes, and every edge cites its source row. A graph outage leaves the PostgreSQL records in place. Replaying those records rebuilds the graph.

Follow-ups and deadline checks are rows in `scheduled_actions`. The execution worker leases them, so a restart continues a follow-up that was already due. Applying the same lease twice does not write a second event. PostgreSQL remains the business state.

## Local run

```bash
python3 -m venv .venv
.venv/bin/pip install -e "./backend[dev]"
docker compose up -d --wait
make migrate
make test
make api
make worker
make desk
```

The API listens on `http://127.0.0.1:8000`. Interactive docs are at `/docs`. PostgreSQL is published on port 5434 so it does not collide with another local database.

Copy `.env.example` to `.env` in the repository root for local overrides and the model keys. The API reads that file. Leave `.env.example` without secrets.

## Build order

| Phase | What it adds |
| --- | --- |
| 1 | Identity, agent configuration, permissions, events, outbox |
| 2 | Task engine and durable follow-up. The worker process resumes due work after a restart |
| 3 | Central and worker runtime, GLM and Gemini behind one router, tools checked by policy |
| 4 | Memory API on PostgreSQL full-text search and pgvector |
| 5 | React admin oversight and the human-assistant desk |
| 6 | Graphiti when workloads need temporal relationship queries |

GLM and Gemini are the only model providers. Set `GLM_API_KEY` and `GEMINI_API_KEY` before a live turn. A missing key is stored on the run as "The model provider is not configured" and the key is never written into the error. The models do not store organizational memory. FalkorDB is a derived index and can be rebuilt from PostgreSQL.
# operations-orchestration
