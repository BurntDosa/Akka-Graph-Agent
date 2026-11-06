# Neo4j research for a Claude Code Neo4j agent (Java / Akka SDK backend)

Research date: 2026-11-06. All facts come from pages fetched on that date.
"Verified = yes" means I fetched the URL and the page gave the stated facts.
"UNVERIFIED" means the URL or fact came from a search result or a secondary page, and I did not fetch it.

---

## 0. Top findings

1. Neo4j server is now at **2026.10.1** (23 Oct 2026). The LTS line is **5.26** (5.26.32). A "2026 LTS" is announced but not yet released.
2. The Java driver is at **6.3.0** (`org.neo4j.driver:neo4j-java-driver:6.3.0`). It needs Java 17+. The newest 5.x LTS driver is 5.28.15.
3. **Cypher 25** is the current language. Cypher 5 has been frozen since 2025.06. From 2026.02, new databases default to Cypher 25. Aura orgs default to Cypher 25.
4. The official MCP server is **neo4j/mcp** (Go binary, v2.1.0, 2026-11-02). v2.0.0 removed the old `NEO4J_*` env vars. Use `NEO4J_MCP_*` now. Many blog posts and the Labs page still show the old names.
5. The Labs MCP servers (`mcp-neo4j-cypher`, `-memory`, `-data-modeling`, `-aura-manager`) still exist. They have no SLA. The data-modeling server has no official equivalent, so it is still useful.
6. Neo4j publishes **agent skills** for Claude Code (`neo4j-contrib/neo4j-skills`), including `neo4j-driver-java-skill` and `neo4j-cypher-skill`. It also publishes `llms.txt` and `llms-full.txt`.
7. Context7 has many Neo4j libraries. The best IDs for this agent are `/neo4j/docs-drivers`, `/websites/neo4j_cypher-manual_current`, `/neo4j/docs-operations`, `/websites/neo4j_apoc_current`, and `/llmstxt/neo4j_llms_txt`.
8. **Akka SDK 3.3.0+ runs component and endpoint code on Java 21 virtual threads.** The Akka migration guide says blocking I/O is now safe there. So the sync driver API is an option, and the async `CompletionStage` API is still a clean fit. See section 1.4.

---

## 1. Neo4j Java Driver

### 1.1 Version and coordinates

| Item | Value | Source URL | Verified |
|---|---|---|---|
| Latest stable | **6.3.0** (tag "Latest", 24 Sep). Adds driver-side Property Encryption | https://github.com/neo4j/neo4j-java-driver/releases | yes |
| Maven Central latest | 6.3.0 ("1 month ago"), Apache 2.0 | https://central.sonatype.com/artifact/org.neo4j.driver/neo4j-java-driver | yes |
| Latest 5.x (LTS) | 5.28.15 (12 Aug) | https://github.com/neo4j/neo4j-java-driver/releases | yes |
| 4.4 line | 4.4.27 (12 Aug) | same | yes |
| Java runtime | 6.x: Java 17. 5.x: Java 17. 4.4: Java 8/11 | https://github.com/neo4j/neo4j-java-driver | yes |
| Server compatibility | Driver 6.x works with Neo4j 4.4, 5.x and 2025/2026.x | https://neo4j.com/docs/java-manual/current/upgrade/ | yes |
| Alternative artifact | `neo4j-java-driver-all` (shaded, module name `org.neo4j.driver`) | https://github.com/neo4j/neo4j-java-driver | yes |
| BOM | Introduced in 6.0.0. `netty-bom` was removed from it again in 6.0.1 | https://github.com/neo4j/neo4j-java-driver/wiki/6.x-changelog | yes |

```xml
<dependency>
  <groupId>org.neo4j.driver</groupId>
  <artifactId>neo4j-java-driver</artifactId>
  <version>6.3.0</version>
</dependency>
```

Note: the manual's install snippet still pins 6.1.0. Use 6.3.0 from Maven Central.

### 1.2 Driver manual pages

| URL | What it is | Verified |
|---|---|---|
| https://neo4j.com/docs/java-manual/current/ | Manual home (Version 6 current, also 5 and 4.4) | yes |
| https://neo4j.com/docs/java-manual/current/query-simple/ | `driver.executableQuery(...)`, the recommended default API | yes |
| https://neo4j.com/docs/java-manual/current/transactions/ | Sessions, `executeRead`/`executeWrite`, explicit tx, `TransactionConfig` | yes |
| https://neo4j.com/docs/java-manual/current/async/ | Async API (`AsyncSession`, `CompletionStage`) | yes |
| https://neo4j.com/docs/java-manual/current/reactive/ | Reactive API (`ReactiveSession`, Flow or Reactive Streams) | yes |
| https://neo4j.com/docs/java-manual/current/performance/ | Performance recommendations | yes |
| https://neo4j.com/docs/java-manual/current/connect-advanced/ | URI schemes, auth, mTLS, logging, resolver | yes |
| https://neo4j.com/docs/java-manual/current/value-mapping/ | Map results to Java records (`Record.as`, `Value.as`, `@Property`) | yes |
| https://neo4j.com/docs/java-manual/current/upgrade/ | 5.x to 6.x changes, removed APIs, deprecations | yes |
| https://neo4j.com/docs/java-manual/current/install/ | Install page | UNVERIFIED (linked, not fetched) |
| https://neo4j.com/docs/java-manual/current/connect/ | Basic connect page | UNVERIFIED (linked, not fetched) |
| https://neo4j.com/docs/java-manual/current/result-summary/ | Result summary page | UNVERIFIED (linked, not fetched) |
| https://neo4j.com/docs/api/java-driver/current/org.neo4j.driver/org/neo4j/driver/Driver.html | Javadoc (all `Config` options, incl. pool settings) | UNVERIFIED (linked, not fetched) |
| https://neo4j.com/docs/java-manual/current/object-mapping/ | Does NOT exist (404). Use `value-mapping/` | yes (404) |

### 1.3 Async and reactive APIs (key facts)

Async (`https://neo4j.com/docs/java-manual/current/async/`):
- Open with `driver.session(AsyncSession.class, SessionConfig.builder().withDatabase("<db>").build())`.
- Use `session.executeReadAsync(tx -> tx.runAsync(query, params).thenCompose(cursor -> cursor.listAsync(...)))`. The callback returns a `CompletionStage`.
- You cannot use try-with-resources for async sessions. Close with `session.closeAsync()` in `whenComplete(...)`.
- Process the result inside the callback. Do not leak the `ResultCursor`.
- The doc example calls `.get()` "for demonstration purposes". Never do that in an Akka component.

Reactive (`https://neo4j.com/docs/java-manual/current/reactive/`):
- Two flavours: `org.neo4j.driver.reactivestreams` (Reactive Streams, needs `reactor-core`) and `org.neo4j.driver.reactive` (JDK `Flow`).
- `RxSession` was removed in 6.0. Use `ReactiveSession`.
- The manual recommends reactive only for apps that are already reactive and need backpressure. Otherwise use sync or async. It says async "works well with virtual threads on Java 21+".
- Always create the session lazily inside `Flux.usingWhen(...)`. An eager session that nobody subscribes to stays busy and can exhaust the pool.

### 1.4 Akka SDK fit

| Fact | Source | Verified |
|---|---|---|
| Akka SDK **3.3.0** "introduces support for Java 21 virtual threads across all user component and endpoint logic". It is "now safe to perform blocking I/O operations without occupying underlying OS threads." | https://doc.akka.io/reference/migration-guide.html | yes |
| Service Setup, Endpoints, Consumers, Timed Actions and Workflows can accept a `java.util.concurrent.Executor` (virtual-thread executor) for parallel blocking work | same | yes |
| Component clients: `.invoke()` (blocking) and `.invokeAsync()` (`CompletionStage`) both exist | https://doc.akka.io/sdk/errors-and-failures.html | UNVERIFIED (search snippet only) |

Guidance for the agent (inference from the facts above, not a Neo4j or Akka statement):
- Create **one** `Driver` per service (for example, in Akka Service Setup) and close it at shutdown. Sessions are cheap and not thread-safe. Open one per unit of work.
- On Akka SDK 3.3+, the sync `executableQuery`/`executeRead` calls run on virtual threads, so they do not block OS threads. The async API (`executeReadAsync` + `CompletionStage`) also works and composes with Akka's async APIs.
- Do not call Neo4j from inside Event Sourced Entity or Key Value Entity command handlers. Entities should stay pure. Put Neo4j writes in Consumers (projection from entity events) or Workflows, and reads in Endpoints, Views-adjacent services, or Agents' tools. (Design advice. Confirm against the Akka SDK docs.)
- Transaction functions retry. Make Consumer writes idempotent (`MERGE` on a constrained key).
- The JDK version matters for virtual-thread pinning. I did not check whether the driver's internal locks pin carrier threads. Test under load.

### 1.5 Connection, session and transaction best practices (from the manual)

- **One `Driver` per application.** Share it across threads. (llms-full.txt: "One driver instance per application process.")
- **Always set the database** (`.withDatabase("neo4j")` in `QueryConfig` or `SessionConfig`). Without it, the driver makes an extra round trip on every query.
- **Sessions are cheap and not thread-safe.** Close them (try-with-resources for sync). One active transaction per session.
- **Use managed transactions** (`executeRead`/`executeWrite`). They retry transient errors up to `withMaxTransactionRetryTime`. The callback must be idempotent. Do not return `Result` from the callback. Return processed data.
- **Explicit transactions** (`beginTransaction`) do not retry. Use them when a side effect cannot be repeated.
- **Auto-commit** (`session.run`) gives the highest throughput and no retries. Driver 6.1.0 added retries for auto-commit queries marked idempotent.
- **`TransactionConfig`**: `withTimeout(Duration)` and `withMetadata(Map)`. Metadata appears in `query.log` and `SHOW TRANSACTIONS`.
- **Route reads** with `RoutingControl.READ` or `executeRead`. Read mode is NOT access control.
- **Always use `$parameters`.** They prevent Cypher injection and enable plan caching.
- **Batch writes** with `UNWIND $rows AS row ...`. For first-time bulk loads, use `neo4j-admin database import`.
- Use `CREATE` instead of `MERGE` when the data is known to be new.
- URI schemes: `neo4j://` (routing), `neo4j+s://` (TLS, Aura default), `neo4j+ssc://` (self-signed), `bolt://`, `bolt+s://`, `bolt+ssc://` (direct), `bolt+unix://` (Neo4j 2025.08+, not Aura).
- Optional: Netty native transports (for example, `netty-transport-native-io_uring`). The version must match the driver's Netty 4.2.
- Errors: subclasses of `Neo4jException`. Use `e.code()` or GQL status (`e.gqlStatus()`, `e.containsGqlStatus("42001")`). Do not parse messages.

### 1.6 Driver 5.x to 6.x breaking changes (for code review)

Removed in 6.0: `Session.readTransaction()/writeTransaction()` (use `executeRead/executeWrite`), `RxSession` (use `ReactiveSession`), `Session.asyncSession()/reactiveSession()` (use `driver.session(Class)`), `TrustStrategy.certFile()`, `Notification.severity()`, `Session.lastBookmark()`.
Deprecated: `.id()` (use element IDs), driver `Logging` (use `System.Logger`), status-code notifications (use GQL status objects), `ProfiledPlan`/`profile()`/`plan()` (6.2, use `QueryProfile`).
New in 6.x: record/value mapping, `Vector` type (GA), GQL status objects, `bolt+unix`, Netty native transports, UUID type (6.2, Bolt 6.1), Property Encryption (6.3).
Source: https://neo4j.com/docs/java-manual/current/upgrade/ and https://github.com/neo4j/neo4j-java-driver/wiki/6.x-changelog (both verified).

---

## 2. Core documentation and current versions

### 2.1 Versions

| Product | Current version | LTS / other | Source | Verified |
|---|---|---|---|---|
| Neo4j DBMS | **2026.10.1** (23 Oct 2026) | 5.26 LTS (5.26.32, 9 Oct 2026). "2026.LTS" announced, no date | https://neo4j.com/release-notes/ | yes |
| Operations manual | 2026.10 (current) | 5.26 (LTS), 4.4 | https://neo4j.com/docs/operations-manual/current/ | yes |
| Cypher manual | Cypher 25 | Cypher 5, 4.4 | https://neo4j.com/docs/cypher-manual/current/ | yes |
| APOC Core | 2026.10 | 5.26 LTS | https://neo4j.com/docs/apoc/current/ | yes |
| GDS | 2026.10 | 2.x line down to 2.5 | https://neo4j.com/docs/graph-data-science/current/ | yes |
| Java driver | 6.3.0 | 5.28.15 | see section 1 | yes |
| Neo4j MCP (official) | v2.1.0 (2026-11-02) | docs "Version 2 (Current)", "Version 1" | https://github.com/neo4j/mcp/releases | yes |

### 2.2 Doc URLs

| URL | What it is | Verified |
|---|---|---|
| https://neo4j.com/docs/cypher-manual/current/ | Cypher manual (Cypher 25) | yes |
| https://neo4j.com/docs/cypher-manual/current/queries/select-version/ | Select Cypher 5 or 25 (prefix, config, `ALTER DATABASE`) | yes |
| https://neo4j.com/docs/cypher-manual/current/deprecations-additions-removals-compatibility/ | Deprecations, additions, removals | yes |
| https://neo4j.com/docs/cypher-cheat-sheet/25/all/ | Cypher 25 cheat sheet | UNVERIFIED (linked from manual, not fetched) |
| https://neo4j.com/docs/cypher-manual/current/indexes/ | Index overview (performance vs search indexes) | yes |
| https://neo4j.com/docs/cypher-manual/current/indexes/performance-indexes/ | Range, text, point, token lookup | UNVERIFIED (linked, not fetched) |
| https://neo4j.com/docs/cypher-manual/current/indexes/search-indexes/ | Full-text and vector | UNVERIFIED (linked, not fetched) |
| https://neo4j.com/docs/cypher-manual/current/indexes/semantic-indexes/vector-indexes/ | Vector indexes (page loaded at this older path) | yes |
| https://neo4j.com/docs/cypher-manual/current/schema/constraints/ | Constraints overview | yes |
| https://neo4j.com/docs/cypher-manual/current/schema/constraints/create-constraints/ | Create constraints syntax | UNVERIFIED (linked, not fetched) |
| https://neo4j.com/docs/cypher-manual/current/schema/graph-types/ | Graph types (new schema model, recommended over plain constraints) | UNVERIFIED (linked, not fetched) |
| https://neo4j.com/docs/operations-manual/current/ | Operations manual | yes |
| https://neo4j.com/docs/operations-manual/current/docker/introduction/ | Docker intro | yes |
| https://neo4j.com/docs/operations-manual/current/docker/plugins/ | Docker `NEO4J_PLUGINS` | yes |
| https://neo4j.com/docs/getting-started/data-modeling/ | Data modeling guide | yes |
| https://neo4j.com/docs/getting-started/data-modeling/tutorial-data-modeling/ | Modeling tutorial | UNVERIFIED (linked, not fetched) |
| https://neo4j.com/docs/getting-started/data-modeling/modeling-designs/ | Modeling designs | UNVERIFIED (linked, not fetched) |
| https://neo4j.com/docs/getting-started/data-modeling/relational-to-graph-modeling/ | Relational to graph | UNVERIFIED (linked, not fetched) |
| https://neo4j.com/docs/getting-started/data-modeling/modeling-tips/ | Modeling tips | UNVERIFIED (linked, not fetched) |
| https://graphacademy.neo4j.com/courses/modeling-fundamentals/ | GraphAcademy modeling course | UNVERIFIED (linked, not fetched) |
| https://neo4j.com/docs/apoc/current/ | APOC Core manual | yes |
| https://neo4j.com/docs/apoc/current/installation/ | APOC install and version-matching rule | yes |
| https://neo4j.com/docs/apoc/current/apoc-and-cypher/cypher-migration/ | Migrate APOC calls to native Cypher | UNVERIFIED (linked, not fetched) |
| https://neo4j.com/labs/apoc/5/overview | APOC Extended (Labs) | UNVERIFIED (linked, not fetched) |
| https://neo4j.com/docs/graph-data-science/current/ | GDS manual | yes |
| https://neo4j.com/docs/graph-data-science/current/installation/ | GDS install, editions | yes |
| https://neo4j.com/docs/genai/plugin/current/ | GenAI plugin (LLM calls from Cypher) | yes |
| https://neo4j.com/docs/genai/tutorials/current/ | GenAI tutorials | UNVERIFIED (from docs llms.txt) |
| https://neo4j.com/docs/neo4j-graphrag-python/current/ | neo4j-graphrag Python docs | yes |
| https://neo4j.com/docs/aura/quotas-and-limits/ | Aura quotas and limits | yes |

### 2.3 Indexes and constraints

- Index types: performance indexes (range, text, point, token lookup) and search indexes (full-text, vector).
- Constraints: property uniqueness (Community and Enterprise). Property existence, property type and key constraints are **Enterprise only**.
- The manual now recommends **graph types** as the schema mechanism, with "more sophisticated constraint types".

### 2.4 Vector indexes / GenAI

- Syntax: `CREATE VECTOR INDEX moviePlots IF NOT EXISTS FOR (m:Movie) ON m.embedding OPTIONS { indexConfig: { `vector.dimensions`: 1536, `vector.similarity_function`: 'cosine' } }`.
- 2026.01+: multi-label (`FOR (n:Movie|Actor)`) and in-index filter properties (`WITH [n.title, ...]`).
- Options: `vector.dimensions` (1-4096), `vector.similarity_function` (`cosine` default, `euclidean`), `vector.quantization.type` (2026.06+, `none`/`scalar`/`binary`, default `binary` since 2026.08), `vector.hnsw.m` (16), `vector.hnsw.ef_construction` (100).
- Query: the **`SEARCH` clause** is preferred from 2026.01. `db.index.vector.queryNodes()` and `queryRelationships()` are **deprecated since 2026.04**.
- Native `VECTOR` property type from 2025.10 (Cypher 25). It needs block format (Enterprise and Aura). Community indexes `LIST<FLOAT>` properties.
- GenAI plugin: enabled by default on Aura. Self-managed: move the jar from `products/` to `plugins/`, or Docker `NEO4J_PLUGINS='["genai"]'`. "Most GenAI features are available only in Cypher 25." Providers: OpenAI, Azure OpenAI, VertexAI, Amazon Bedrock.

### 2.5 APOC and GDS

- APOC Core ships in `$NEO4J_HOME/labs`. Move it to `plugins/`. **The APOC year.month must match the Neo4j year.month.**
- Restrict with `dbms.security.procedures.allowlist` and `dbms.security.procedures.unrestricted`.
- Many APOC procedures are deprecated in favour of native Cypher. Example: `apoc.create.uuids()` is not available in Cypher 25.
- GDS Community: free plugin, limited to 4 CPUs and 3 models (pricing page). GDS Enterprise needs a license key. On Aura, use Aura Graph Analytics or AuraDS.

---

## 3. Neo4j MCP servers

### 3.1 Official server: neo4j/mcp

| Item | Value | Source | Verified |
|---|---|---|---|
| Repo | https://github.com/neo4j/mcp (Go: `go.mod`, `cmd/neo4j-mcp`) | https://github.com/neo4j/mcp | yes |
| Docs | https://neo4j.com/docs/mcp/current/ | same | yes |
| Latest | v2.1.0 (2026-11-02). v2.0.0 (2026-10-28) was a breaking release | https://github.com/neo4j/mcp/releases | yes |
| Prerequisite | Neo4j instance with **APOC** installed (docs intro page) | https://neo4j.com/docs/mcp/current/ | yes |
| Docker image | `neo4j/mcp:latest` (about 7.5 MB, no overview text) | https://hub.docker.com/r/neo4j/mcp | yes |

Install (from https://neo4j.com/docs/mcp/current/installation/, verified):
```bash
brew install neo4j-mcp
uvx neo4j-mcp-server                # PyPI wrapper
# or download the binary from https://github.com/neo4j/mcp/releases and put neo4j-mcp on PATH
neo4j-mcp -v
```
The repo README shows `pip install neo4j-mcp-server` and `python -m neo4j_mcp_server`. The docs and the README disagree slightly. Both point to the same PyPI package.

Claude Code (from https://neo4j.com/docs/mcp/current/client-configuration/, verified, reassembled from the page):
```bash
claude mcp add neo4j-mcp \
  --env NEO4J_MCP_URI=bolt://localhost:7687 \
  --env NEO4J_MCP_USERNAME=neo4j \
  --env NEO4J_MCP_PASSWORD=password \
  --env NEO4J_MCP_DATABASE=neo4j \
  --env NEO4J_MCP_READ_ONLY=true \
  -- neo4j-mcp

# Aura-hosted MCP endpoint:
claude mcp add --transport http neo4j-mcp https://<instance-id>.mcp-instances.neo4j.io
```

Tools (https://neo4j.com/docs/mcp/current/tools/, verified):

| Tool | Read-only | Notes |
|---|---|---|
| `get-schema` | yes | Labels, relationship types, property keys (samples `NEO4J_MCP_SCHEMA_SAMPLE_SIZE` nodes, default 100) |
| `read-cypher` | yes | Rejects writes, schema/admin ops and PROFILE. Uses an extra `EXPLAIN` round trip. Custom procedures that Neo4j classifies as read-only can bypass the check |
| `write-cypher` | no | Hidden when read-only is on. Docs: use in development only |
| `list-gds-procedures` | yes | Shown only when GDS is installed |

Env vars / flags (https://neo4j.com/docs/mcp/current/configuration/, verified):

| Env var | Flag | Default |
|---|---|---|
| `NEO4J_MCP_URI` | `--uri` | required (STDIO) |
| `NEO4J_MCP_USERNAME` / `NEO4J_MCP_PASSWORD` | `--username` / `--password` | required (STDIO) |
| `NEO4J_MCP_DATABASE` | `--database` | required in STDIO since v2.0.0 (no "neo4j" default) |
| `NEO4J_MCP_READ_ONLY` | `--read-only` | `false` |
| `NEO4J_MCP_TOOLS` | `--tools` | all tools (filter can only shrink the set) |
| `NEO4J_MCP_TELEMETRY` | `--telemetry` | `true` |
| `NEO4J_MCP_LOG_LEVEL` / `NEO4J_MCP_LOG_FORMAT` | `--log-level` / `--log-format` | `info` / `text` |
| `NEO4J_MCP_SCHEMA_SAMPLE_SIZE` | `--schema-sample-size` | `100` |
| `NEO4J_MCP_TRANSPORT_MODE` | `--transport-mode` | `stdio` (`http` also) |
| `NEO4J_MCP_REQUEST_TIMEOUT` | `--request-timeout` | `3m` (max `30m`) |
| `NEO4J_MCP_HTTP_HOST` / `_PORT` / `_ALLOWED_ORIGINS` / `_TLS_*` | `--http-*` | `127.0.0.1` / 80 or 443 |
| `NEO4J_MCP_HTTP_ALLOWED_URIS` | | required in HTTP mode since v2.0.0 (release notes) |

HTTP mode: endpoint `/db/{databaseName}/mcp`. Credentials come from request headers. Headers `X-Neo4j-MCP-URI`, `X-Neo4j-MCP-ReadOnly`, `X-Neo4j-MCP-Tools` and `X-Neo4j-MCP-Request-Timeout` work per request.

**Gotcha:** v2.0.0 removed the deprecated `NEO4J_URI`/`NEO4J_USERNAME`/... env vars and `--neo4j-*` flags. The Labs page https://neo4j.com/labs/genai-ecosystem/agent-skills/coding-skills/ and community guides still show the old names.

### 3.2 Labs servers: neo4j-contrib/mcp-neo4j (no SLA, MIT)

Repo: https://github.com/neo4j-contrib/mcp-neo4j (verified). The README points to neo4j/mcp as "the official product Neo4j MCP server".

| Server | Run command | Key env vars | Tools | Verified |
|---|---|---|---|---|
| `mcp-neo4j-cypher` | `uvx mcp-neo4j-cypher@0.6.0 --transport stdio` | `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD`, `NEO4J_DATABASE`, `NEO4J_READ_ONLY`, `NEO4J_READ_TIMEOUT` (30), `NEO4J_RESPONSE_TOKEN_LIMIT`, `NEO4J_SCHEMA_SAMPLE_SIZE` (1000), `NEO4J_NAMESPACE`, `NEO4J_TRANSPORT` | `read_neo4j_cypher`, `write_neo4j_cypher`, `get_neo4j_schema` (needs APOC) | yes |
| `mcp-neo4j-data-modeling` | `uvx mcp-neo4j-data-modeling@0.8.2 --transport stdio` | transport/namespace vars only (no DB connection) | `validate_node`, `validate_relationship`, `validate_data_model`, `get_mermaid_config_str`, `load_from_arrows_json`, `export_to_arrows_json`, `load_from_owl_turtle`, `export_to_owl_turtle`, `export_to_pydantic_models`, `export_to_neo4j_graphrag_pkg_schema`, `load_from_neo4j_graphrag_pkg_schema`, `list_example_data_models`, `get_example_data_model`, `get_constraints_cypher_queries`, `get_node_cypher_ingest_query`, `get_relationship_cypher_ingest_query`. Prompt: `create_new_data_model` | yes |
| `mcp-neo4j-memory` | `uvx mcp-neo4j-memory@0.4.5` | `NEO4J_URI` (README examples use `NEO4J_URL`), `NEO4J_USERNAME`, `NEO4J_PASSWORD`, `NEO4J_DATABASE` | `read_graph`, `search_nodes`, `find_nodes`, `create_entities`, `delete_entities`, `create_relations`, `delete_relations`, `add_observations`, `delete_observations` | yes |
| `mcp-neo4j-aura-manager` (folder `mcp-neo4j-cloud-aura-api`) | `uvx mcp-neo4j-aura-manager@0.4.8` | `NEO4J_AURA_CLIENT_ID`, `NEO4J_AURA_CLIENT_SECRET` | `list_instances`, `get_instance_details`, `get_instance_by_name`, `create_instance`, `update_instance_name`, `update_instance_memory`, `update_instance_vector_optimization`, `pause_instance`, `resume_instance`, `delete_instance`, `list_tenants`, `get_tenant_details` | yes |

Other MCP options listed on https://neo4j.com/developer/genai-ecosystem/model-context-protocol-mcp/ (verified page): GDS Agent (https://github.com/neo4j-contrib/gds-agent), Sandbox MCP (https://mcp.sandbox.neo4j.com/sse, https://github.com/neo4j-contrib/sandbox-mcp-server), Google MCP Toolbox (https://github.com/googleapis/genai-toolbox), Docker MCP catalog (https://hub.docker.com/mcp/server/neo4j/overview). I did not fetch these sub-URLs, so they are UNVERIFIED.

### 3.3 Recommendation for the agent

- Use **neo4j/mcp** with `NEO4J_MCP_READ_ONLY=true` as the default database tool. Enable `write-cypher` only for a local dev database.
- Add **mcp-neo4j-data-modeling** for model design and validation. It needs no DB credentials.
- Skip memory and aura-manager unless the user needs them.
- Note from a community guide: a natively registered MCP server loads its tool schemas into every conversation. Scope the server to the project `.mcp.json` or to the subagent.

---

## 4. AI-assisted development resources

| URL | What it is | Verified |
|---|---|---|
| https://neo4j.com/llms.txt | Agent-oriented index: drivers, Cypher essentials incl. deprecated patterns, MCP, GraphRAG, agent skills | yes |
| https://neo4j.com/llms-full.txt | Long companion (about 35-45 KB). Has "Common Cypher Footguns for LLM-Generated Queries" | yes |
| https://neo4j.com/docs/llms.txt | Docs index (all manuals) | yes |
| https://github.com/neo4j-contrib/neo4j-skills | Official agent skills (about 25+ skills), Claude Code plugin marketplace | yes |
| https://github.com/neo4j-contrib/neo4j-skills/tree/main/neo4j-driver-java-skill | Java driver skill (driver v6, Java 17+, executableQuery, tx, async, reactive, pooling) | yes (README only, SKILL.md not read) |
| https://github.com/neo4j-contrib/neo4j-skills/blob/main/neo4j-cypher-skill/SKILL.md | Cypher skill: rules for LLM-written Cypher 25 | yes |
| https://neo4j.com/labs/genai-ecosystem/agent-skills/coding-skills/ | Labs page: skills and MCP for coding tools (uses old env var names) | yes |
| https://neo4j.com/labs/genai-ecosystem/agent-skills/neo4j-skills/ | Labs skills page | UNVERIFIED |
| https://skills.sh/neo4j-contrib/neo4j-skills | skills.sh listing | UNVERIFIED |
| https://neo4j.com/labs/genai-ecosystem/ | GenAI ecosystem index | yes |
| https://neo4j.com/labs/genai-ecosystem/langchain4j/ | LangChain4j page | UNVERIFIED (linked) |
| https://neo4j.com/labs/genai-ecosystem/spring-ai/ | Spring AI page | UNVERIFIED (linked) |
| https://docs.langchain4j.dev/integrations/embedding-stores/neo4j/ | LangChain4j Neo4j integration docs | yes |
| https://central.sonatype.com/artifact/dev.langchain4j/langchain4j-community-neo4j | Latest `1.23.0-beta33`, depends on neo4j-java-driver 6.3.0 | yes |
| https://docs.spring.io/spring-ai/reference/api/vectordbs/neo4j.html | Spring AI Neo4j vector store (Spring AI 2.0.2) | yes |
| https://pypi.org/project/neo4j-graphrag/ | neo4j-graphrag 1.23.0 (2026-10-28) | yes |
| https://neo4j.com/docs/neo4j-graphrag-python/current/ | neo4j-graphrag docs. Needs Python 3.10+, Neo4j 5.18.1+ | yes |
| https://github.com/neo4j-labs/agent-memory | Neo4j agent memory | UNVERIFIED (from llms.txt) |

Install the skills in Claude Code (from the repo README, verified):
```
/plugin marketplace add https://github.com/neo4j-contrib/neo4j-skills.git
/plugin install neo4j-skills@neo4j-skills-marketplace
# or a single skill:
npx skills add https://github.com/neo4j-contrib/neo4j-skills --skill neo4j-driver-java-skill
```
Relevant skills for this project: `neo4j-driver-java-skill`, `neo4j-cypher-skill`, `neo4j-modeling-skill`, `neo4j-migration-skill`, `neo4j-vector-index-skill`, `neo4j-genai-plugin-skill`, `neo4j-mcp-skill`, `neo4j-query-tuning-skill` (folder only, no README text), `neo4j-security-skill` (folder only).

Java GraphRAG options:
- **LangChain4j** (`dev.langchain4j:langchain4j-community-neo4j`, `...-neo4j-retriever`, `...-neo4j-spring-boot-starter`): `Neo4jEmbeddingStore`, `Neo4jText2CypherRetriever`, `Neo4jGraph`, `KnowledgeGraphWriter`, `Neo4jChatMemoryStore`. Text2Cypher reads the schema with `apoc.meta.data`, so it needs APOC.
- **Spring AI** (`org.springframework.ai:spring-ai-neo4j-store`, starter `spring-ai-starter-vector-store-neo4j`): `Neo4jVectorStore`. Needs Neo4j 5.15+. Schema init is opt-in (`initialize-schema=true`). It still queries with the deprecated `db.index.vector.queryNodes()`.
- There is no official Java port of neo4j-graphrag that I could find. I did not search hard for one.

Cypher rules for LLMs (from `neo4j-cypher-skill` and llms-full.txt, verified):
1. Prefix with `CYPHER 25` (top-level only).
2. Read the schema first. Never guess labels or properties.
3. `$parameters` always. Labels and types cannot be parameters except with Cypher 25 dynamic `$($label)`.
4. `MERGE` only on a constrained key. Bind both endpoints before a relationship `MERGE`. Never nest node `MERGE` in relationship `MERGE`.
5. `SET n += {...}` for partial updates. `SET n = {...}` wipes properties.
6. `elementId(n)`, not `id(n)`.
7. `CALL (x) { ... }`, not `CALL { WITH x ... }`.
8. Prefer QPP `(()-[:R]->()){1,5}` and `SHORTEST 1` over `[:R*1..5]` and `shortestPath()`.
9. No label-free `MATCH (n)`.
10. `CALL (row) { ... } IN TRANSACTIONS OF N ROWS` for bulk writes (auto-commit only). `DISJOINT BY` from 2026.06.
11. `EXPLAIN` before writes. `PROFILE` to look for `AllNodesScan`, `CartesianProduct`, `Eager`.
12. Unknown server version: stick to 2025.01-safe features. `SEARCH`, `ACYCLIC`, `GROUP BY` and conditional `WHEN/THEN` CALL need newer servers.

---

## 5. Context7

Search API https://context7.com/api/v1/search?query=neo4j works (verified). Relevant library IDs:

| Library ID | Content | Snippets | Trust |
|---|---|---|---|
| `/neo4j/docs-drivers` | All driver manuals incl. Java (verified page https://context7.com/neo4j/docs-drivers, updated "1 day ago", has 6.3.0 content) | 1768 | 9 |
| `/websites/neo4j_cypher-manual_current` | Cypher manual | 85 (small) | 10 |
| `/neo4j/docs-operations` | Operations manual | 6449 | 9 |
| `/websites/neo4j_apoc_current` | APOC | 1794 | 10 |
| `/llmstxt/neo4j_llms_txt` | neo4j.com llms.txt | 3603 | 10 |
| `/websites/neo4j` | neo4j.com site | 1688 | 10 |
| `/neo4j/docs-aura` | Aura | 2477 | 9 |
| `/neo4j/docs-status-codes` | Status and GQL codes | 2133 | 9 |
| `/neo4j/cypher-dsl` | Java Cypher-DSL | 2483 | 9 |
| `/spring-projects/spring-data-neo4j` | Spring Data Neo4j | 506 | 9.7 |
| `/websites/neo4j_neo4j-graphrag-python` | neo4j-graphrag Python | 1953 | 10 |
| `/neo4j/neo4j` | Server source (version tag 2025.08.0) | 8452 | 8.8 |

No dedicated GDS, GenAI plugin or MCP library appeared in the top 30 results. The Cypher manual entry has only 85 snippets, so `/llmstxt/neo4j_llms_txt` plus the neo4j-cypher-skill are better Cypher sources.

---

## 6. Local dev and testing

| URL | What it is | Verified |
|---|---|---|
| https://hub.docker.com/_/neo4j | Official image | yes |
| https://java.testcontainers.org/modules/databases/neo4j/ | Testcontainers Neo4j module | yes |
| https://neo4j.com/docs/aura/quotas-and-limits/ | Aura Free limits | yes |
| https://neo4j.com/pricing/ | Tiers and editions | yes |
| https://neo4j.com/cloud/aura-free/ | Aura Free landing | UNVERIFIED (from llms.txt) |

Docker (verified):
- Tags: `neo4j:2026.10.1` / `latest` / `community` (Community), `neo4j:2026.10.1-enterprise` / `enterprise`, `neo4j:5.26` / `5.26-enterprise` (LTS), `4.4`. Base OS suffixes `-trixie` (default) and `-ubi10`. amd64 and arm64.
- `NEO4J_AUTH=neo4j/<password>` or `NEO4J_AUTH=none`. It has no effect if `/data` already holds a database.
- `NEO4J_PLUGINS='["apoc"]'` (names: `apoc`, `apoc-extended`, `bloom`, `genai`, `graph-data-science`, `n10s`). Docs: dev only. Mount jars in production.
- Enterprise needs `NEO4J_ACCEPT_LICENSE_AGREEMENT=yes`.
```bash
docker run --publish=7474:7474 --publish=7687:7687 \
  --env NEO4J_AUTH=neo4j/devpassword \
  --env NEO4J_PLUGINS='["apoc"]' \
  neo4j:2026.10.1
```

Testcontainers (verified):
- `org.testcontainers:testcontainers-neo4j:2.0.5` (test scope). In 1.x the artifactId was `neo4j`.
- `new Neo4jContainer("neo4j:2026.10.1").withAdminPassword("...")`, `.withPlugins("apoc")`, `.withoutAuthentication()`, `.withRandomPassword()`, `.withNeo4jConfig(key, value)`.
- Enterprise 5.x+: `.withEnv("NEO4J_ACCEPT_LICENSE_AGREEMENT", "yes")`.
- The page's own examples still use `neo4j:4.4` and driver 4.4.13. Pin a 2026.x image and driver 6.3.0 yourself.

Aura Free (quotas page verified): 200,000 nodes, 400,000 relationships, 1 GB storage, 1 database, one Free instance per account, no SLA, on-demand backup (1 kept).
Pause after 3 days (72 h) without write activity, deletion after 30 days paused: UNVERIFIED (search snippets from neo4j.com FAQ and support.neo4j.com, not fetched).

---

## 7. Gotchas

### 7.1 Licensing (https://neo4j.com/pricing/, verified)
- Community Edition: **GPLv3**, free. No clustering, no fine-grained security. GDS limited to 4 CPUs and 3 models.
- Enterprise Edition: commercial license. Clustering/HA, RBAC, Parallel Runtime, CDC, unlimited GDS. Neo4j Desktop includes a free developer Enterprise license.
- Enterprise-only Cypher features that bite in Community: existence, property type and key constraints. Native `VECTOR` storage (block format). Multiple databases.
- Java driver is Apache 2.0. The Labs MCP servers are MIT.
- Agent rule: generate code that works on Community unless the user confirms Enterprise or Aura.

### 7.2 Cypher 5 vs Cypher 25
- Cypher 25 arrived in 2025.06. Cypher 5 is frozen (fixes only). Support ends "no earlier than two additional server LTS cycles".
- Default: `db.query.default_language=CYPHER_5` if unset. New databases on 2026.02+ get `CYPHER_25`. Aura orgs default to Cypher 25.
- Per query: `CYPHER 25 ...` or `CYPHER 5 ...`. Per DB: `ALTER DATABASE x SET DEFAULT LANGUAGE CYPHER 25`.
- Removed in Cypher 25: `SET n = r` / `SET n += r` with an entity on the right (use `properties(r)`). MERGE that reads properties of another entity in the same clause. `indexProvider` option. `db.create.setVectorProperty()`, `db.index.vector.createNodeIndex()`. `$` in unescaped identifiers.
- Deprecated: `db.index.vector.queryNodes/queryRelationships` (2026.04, use `SEARCH`), `db.index.fulltext.queryNodes/queryRelationships` (2026.10, use `SEARCH ... FULLTEXT`), `vector.quantization.enabled` (2026.06).
- Behaviour change 2026.05: `stDev()`/`stDevP()` of null return `null`, not `0`.
- Deprecated in practice (skill and llms.txt): `id()`, `CALL { WITH x }`, `[:R*m..n]`, `shortestPath()`, `PERIODIC COMMIT`. The deprecations page I fetched did not list `id()`, var-length or `CALL { WITH }`. Treat them as "avoid", and check the manual for exact status.

### 7.3 Other gotchas
- APOC version must match the server year.month. Several APOC functions are not available in Cypher 25 (for example, `apoc.create.uuids()`).
- `read-cypher` read-only checks can be bypassed by custom procedures that are wrongly classified as read-only. Use a read-only DB user as the real guard.
- Driver read routing is not access control.
- Spring AI's Neo4j store still uses the deprecated vector procedures. Expect deprecation notifications on 2026.04+.
- The manual's `install` snippet lags (6.1.0). The Testcontainers docs lag (4.4 image, driver 4.4.13).
- Neo4j MCP v2 needs `NEO4J_MCP_DATABASE` set explicitly in STDIO mode.

---

## 8. Not covered or not checked

- Spring Data Neo4j docs (https://docs.spring.io/spring-data/neo4j/reference/) not fetched.
- Neo4j Java driver Javadoc `Config` pool defaults (`maxConnectionPoolSize`, `connectionAcquisitionTimeout`, `maxConnectionLifetime`) not fetched.
- GDS compatibility matrix and APOC-on-Aura availability not found on the fetched pages.
- Virtual-thread pinning behaviour of the driver under Akka SDK not checked.
- Aura-hosted MCP (`*.mcp-instances.neo4j.io`) availability by tier not checked.
