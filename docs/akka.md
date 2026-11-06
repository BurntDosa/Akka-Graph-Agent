# Akka / Akka SDK research for a Claude Code expert agent

Research date: 2026-11-06. Every URL marked "yes" returned HTTP 200 (curl or WebFetch) on this date. The tables mark failed URLs "no" (404) and URLs that were not fetched "UNVERIFIED".

Raw copies of the key files are in `research/dl/`: `AGENTS.md` (the official Akka template, 48 KB), `llms-full.txt` (1.76 MB, 30,739 lines), and the akka-sdk repo `CLAUDE.md` and `AGENTS.md`.

---

## 0. TL;DR for the agent build

1. **Use the official plugin. Do not build the core yourself.** Akka ships a Claude Code plugin (`akka` in marketplace `akka/ai-marketplace`). The plugin has about 22 `/akka:*` commands, the Akka MCP server (`akka mcp serve`), and on-demand SDK docs.
2. **The best single context file** is the official `AGENTS.md` template at https://doc.akka.io/_attachments/AGENTS.md. Use it as the core of the subagent system prompt. It has rules, patterns, a "Critical Gotchas" section, and a self-review checklist.
3. **Docs for retrieval:** `https://doc.akka.io/llms.txt` (index). Append `.md` to any doc page URL to get markdown. Use `llms-full.txt` or `akka-docs-md.zip` for a local `akka-context/` folder.
4. **Context7 has Akka:** `/akka/akka-sdk` (repo, 12k snippets) and `/llmstxt/doc_akka_io_llms_txt` (docs).
5. **Current SDK version is 3.7.1** (Nov 2026, runtime 1.7.4). The parent POM is `io.akka:akka-javasdk-parent:3.7.1`. Java 25 is required in the dev environment. The parent compiles with `maven.compiler.release=21`.

---

## 1. Akka SDK: version, Maven, Java, CLI, samples

### Version
| Item | Value | Source |
|---|---|---|
| Latest SDK | **3.7.1** (Nov 2026), Akka Runtime 1.7.4 | release notes |
| Previous | 3.7.0 (Oct 2026, governance release: guardrails, classifiers, evaluators, ledger, sanitizers, SPIFFE ACLs. **Breaking:** removed 3.5.x deprecated APIs, renamed config keys, TestKit moved to JUnit 6), 3.6.6 (EvalKit in TestKit), 3.6.5, 3.6.3 | release notes |
| Maven Central anomaly | `maven-metadata.xml` for `akka-javasdk-parent` reports `<latest>3.15.19`. That artifact was published 2026-05-21, before 3.7.x, and is not in the release notes. **Pin 3.7.1. Do not trust "latest" from metadata.** | repo1.maven.org |

### Maven coordinates (verified in `akka-samples/helloworld-agent/pom.xml` and in the POM on Maven Central)
```xml
<parent>
  <groupId>io.akka</groupId>
  <artifactId>akka-javasdk-parent</artifactId>
  <version>3.7.1</version>
</parent>
```
- The sample POM declares no dependencies, plugins, repositories, or Java properties. The parent supplies all of them.
- The parent POM sets `maven.compiler.release=21` with the comment "dependent on runtime environment and cannot be customized by users". It also sets `akka-runtime.version=1.7.4`, `akka.grpc.version=2.5.12`, and the docker-maven-plugin 0.45.1 base image `alpine:3.24.1`.
- The parent is on Maven Central (`https://repo1.maven.org/maven2/io/akka/akka-javasdk-parent/3.7.1/`).
- The setup docs list an **Akka download token** (free, https://account.akka.io/token) as a prerequisite. `akka code token` "provisions an Akka download token". `/akka:setup` configures it automatically. The fetched pages do not document the exact Maven repository URL or `settings.xml` layout (UNVERIFIED).

### Java and Maven
- Manual setup page: **Java 25** (Adoptium) and **Maven 3.9 or later**.
- Runtime 1.6.3 (with SDK 3.6.0) moved to a Java 25 base image. User code runs on virtual threads (since 3.3.0).

### Akka CLI install
| OS | Command |
|---|---|
| macOS | `brew install akka/brew/akka` (upgrade: `brew update && brew upgrade akka`. On Homebrew 7+: `brew trust --formula akka/brew/akka`) |
| Windows | `winget install Akka.Cli` (upgrade: `winget upgrade Akka.Cli`). Alternative: https://downloads.akka.io/latest/akka_windows_amd64.zip |
| Linux | `curl -1sLf 'https://downloads.akka.io/setup.deb.sh' \| sudo -E bash && sudo apt install akka`. Alternative: `curl -sL https://doc.akka.io/install-cli.sh \| bash` |
| Verify | `akka version` / `akka help` |

Key CLI commands for development:
- `akka code init` (pick a sample, download `akka-context`, pick an assistant)
- `akka code init --name helloworld-agent --repo akka-samples/helloworld-agent.git`
- `akka code context-update . [--assistant claude-code|generic] [--claude-md-name X] [--force]`
- `akka code token`
- `akka specify init <dir> [--agent claude-code|cursor|vscode-copilot|codex|...]`
- `akka local start|status|stop|console|services ...`
- `akka services deploy|list|components ...`
- `akka mcp serve [--toolsets ...]`

### Samples (github.com/akka-samples has 28 repos, all verified on the org page)
Agentic samples:
- `helloworld-agent`, `helloworld-agent-kotlin`
- `multi-agent`, `adaptive-multi-agent`
- `ask-akka-agent` (RAG)
- `trip-agent` (Anthropic and tools), `travel-agent`
- `real-estate-cs-agent`, `medical-tagging-agent`, `healthcare-agent`
- `temperature-monitoring-agent`, `changelog-agent`, `order-reconciliation-agent`
- `agentic-haiku`, `autonomous-agent-playground`

Non-agent samples:
- `shopping-cart-quickstart`, `shopping-cart-with-view`
- `transfer-workflow`, `transfer-workflow-compensation`, `transfer-workflow-orchestration`
- `event-sourced-customer-registry`, `event-sourced-counter-brokers`
- `choreography-saga-quickstart`, `endpoint-jwt`, `akka-chess`
- `empty`, `empty-kotlin` (template)

The akka-sdk repo also has `samples/` and spec examples at `samples/autonomous-agent-playground/specs`.

| URL | What | Verified |
|---|---|---|
| https://doc.akka.io/reference/release-notes.html.md | Release notes | yes |
| https://doc.akka.io/reference/migration-guide.html.md | Migration guide | UNVERIFIED |
| https://raw.githubusercontent.com/akka-samples/helloworld-agent/main/pom.xml | Sample POM (parent 3.7.1) | yes |
| https://repo1.maven.org/maven2/io/akka/akka-javasdk-parent/3.7.1/akka-javasdk-parent-3.7.1.pom | Parent POM | yes |
| https://doc.akka.io/getting-started/set-up-env.html.md | Setup options overview | yes |
| https://doc.akka.io/getting-started/set-up-manually.html.md | Manual setup (Java 25, Maven 3.9+) | yes |
| https://doc.akka.io/getting-started/set-up-container.html | Containerized dev env | yes |
| https://doc.akka.io/getting-started/install-cli.html.md | CLI install | yes |
| https://doc.akka.io/reference/cli/akka-cli/index.html | CLI reference | yes |
| https://doc.akka.io/reference/cli/akka-cli/akka_code_token.html | `akka code token` | yes |
| https://account.akka.io/token | Download token | yes |
| https://github.com/orgs/akka-samples/repositories | Samples org | yes |
| https://github.com/akka/akka-sdk | SDK source (BSL 1.1). Has CLAUDE.md and AGENTS.md for contributors | yes |
| https://github.com/akka/akka-sdk/tree/main/samples | In-repo samples | yes |
| https://github.com/akka/akka-sdk/tree/main/samples/autonomous-agent-playground/specs | Example SDD specs | yes |
| https://doc.akka.io/getting-started/samples.html.md | Samples catalog (dynamic, little content in markdown) | yes |

---

## 2. Docs structure (append `.html.md` for markdown, `.html` for HTML)

| Area | URL | Verified |
|---|---|---|
| Index for LLMs | https://doc.akka.io/llms.txt | yes |
| Components overview / choosing | https://doc.akka.io/sdk/components/index.html.md | UNVERIFIED (listed in llms.txt) |
| Agents | https://doc.akka.io/sdk/agents.html | yes |
| Autonomous Agents | https://doc.akka.io/sdk/autonomous-agents.html.md | yes |
| Event Sourced Entities | https://doc.akka.io/sdk/event-sourced-entities.html | yes |
| Key Value Entities | https://doc.akka.io/sdk/key-value-entities.html | yes |
| Workflows | https://doc.akka.io/sdk/workflows.html | yes |
| Views | https://doc.akka.io/sdk/views.html | yes |
| Views query reference | https://doc.akka.io/reference/views/syntax/query.html.md | UNVERIFIED |
| Consumers | https://doc.akka.io/sdk/consuming-producing.html | yes |
| Timed Actions | https://doc.akka.io/sdk/timed-actions.html | yes |
| HTTP Endpoints | https://doc.akka.io/sdk/http-endpoints.html | yes |
| gRPC Endpoints | https://doc.akka.io/sdk/grpc-endpoints.html | yes |
| MCP Endpoints | https://doc.akka.io/sdk/mcp-endpoints.html | yes |
| Setup / DI (`ServiceSetup`) | https://doc.akka.io/sdk/setup-and-dependency-injection.html.md | UNVERIFIED |
| Service config reference | https://doc.akka.io/reference/config/reference.html.md | UNVERIFIED |
| Run locally | https://doc.akka.io/sdk/running-locally.html.md | UNVERIFIED |
| Testing overview | https://doc.akka.io/testing/index.html | yes |
| Unit testing | https://doc.akka.io/testing/unit.html.md | UNVERIFIED |
| Integration testing | https://doc.akka.io/testing/integration.html.md | UNVERIFIED |
| Deploy a service | https://doc.akka.io/operations/services/deploy-service.html | yes |
| Deployment options | https://doc.akka.io/deployment/index.html.md | UNVERIFIED |
| API docs (Javadoc) | https://doc.akka.io/reference/api-docs.html | yes |
| Design best practices | https://doc.akka.io/sdk/dev-best-practices.html.md | UNVERIFIED |

Testing summary (from AGENTS.md and the guidelines):
- Use `TestKitSupport` for integration tests (`*IntegrationTest` suffix).
- Unit tests use `EventSourcedTestKit`, `KeyValueEntityTestKit`, and `TimedActionTestKit`.
- `TestModelProvider` mocks LLMs.
- Endpoint tests use `httpClient`, not `componentClient`.
- View tests use `Awaitility.await()`.
- Tests use AssertJ. The TestKit moved to JUnit 6 in 3.7.0.

---

## 3. Akka Agent component

### Definition (request-based Agent)
```java
@Component(id = "my-agent")
public class MyAgent extends Agent {
  public Effect<String> query(String question) {
    return effects().systemMessage("You are ...").userMessage(question).thenReply();
  }
}
```
- An Agent has **one** command handler, takes 0 or 1 parameter (use a record for more), and holds no mutable state.
- An Agent binds to a **session id**, not an instance id.
- Use `StreamEffect` for token streaming.
- Call an Agent with `componentClient.forAgent().inSession(sessionId).method(MyAgent::query).invoke(arg)`.

### Autonomous Agent (newer, durable, model-driven loop)
```java
@Component(id = "question-answerer", description = "Answers questions ...")
public class QuestionAnswerer extends AutonomousAgent {
  @Override public AgentDefinition definition() {
    return define().capability(TaskAcceptance.of(QuestionTasks.ANSWER).maxIterationsPerTask(3));
  }
}
// Task<Answer> ANSWER = Task.name("Answer").description("...").resultConformsTo(Answer.class);
// componentClient.forAutonomousAgent(QuestionAnswerer.class, instanceId).runSingleTask(QuestionTasks.ANSWER.instructions(q));
// componentClient.forTask(taskId).get(QuestionTasks.ANSWER)
```
- Capabilities: `TaskAcceptance`, `Delegation.to(...)`, handoff, `TeamLeadership`, `Moderation`.
- An Autonomous Agent has no command handlers.

### Model providers
- Supported values: `anthropic`, `azure-openai`, `bedrock`, `googleai-gemini`, `hugging-face`, `local-ai`, `mistral-ai`, `ollama`, `openai`, `vertex-ai`, and custom (an FQCN that implements `ModelProvider.Custom`, or a LangChain4J `ChatModel` through `ModelProvider.custom`).
- Default selection: `akka.javasdk.agent.model-provider = anthropic`. This points to the sibling section `akka.javasdk.agent.anthropic`.

```hocon
akka.javasdk.agent {
  model-provider = anthropic
  anthropic {
    model-name = "claude-opus-4-6"     # docs example
    api-key = ${?ANTHROPIC_API_KEY}
    max-tokens = 5000
  }
}
```

Anthropic keys and their defaults:

| Key | Default |
|---|---|
| `provider` | `"anthropic"` |
| `api-key` | `${?ANTHROPIC_API_KEY}` |
| `model-name` | `""` |
| `base-url` | `""` |
| `temperature` | NaN |
| `top-p` | NaN |
| `top-k` | -1 |
| `max-tokens` | -1 |
| `connection-timeout` | 15s |
| `response-timeout` | 1m |
| `max-retries` | 2 |
| `thinking-budget-tokens` | 0 (thinking off) |
| `cache-system-messages` | false |
| `cache-tools` | false |
| `additional-model-request-headers` | (inherited) |
| `identity-headers` | (inherited) |

Other ways to choose a model:
- Per call: `effects().model(ModelProvider.openAi().withModelName(...))`
- From config: `ModelProvider.fromConfig("...")`
- Inherit a named section: `gpt-o3 = ${akka.javasdk.agent.openai}`
- Autonomous agents: `modelProvider(...)` on the definition

### Memory
- Session memory is on by default and keyed by session id. Agents that share a session id share the memory.
- The backing store is `SessionMemoryEntity`, an Event Sourced Entity.
- Config keys:
  - `akka.javasdk.agent.memory.enabled`
  - `akka.javasdk.agent.memory.limited-window.max-size`
- `effects().memory(...)` options:
  - `MemoryProvider.none()`
  - `limitedWindow()` with `.readOnly()`, `.readLast(n[, filter])`, or `.filtered(...)`
  - `custom(SessionMemory)`
  - `fromConfig()`
  - `.withInterceptor(SessionMemoryInterceptor)`
- Filters: `MemoryFilter.include/excludeFromAgentId/AgentRole`.
- Read history with `SessionMemoryEntity::getHistory`.
- Compact history with `CompactionCmd`.

### Tools and function calling
- Annotate methods with `@FunctionTool(description=...)` and parameters with `@Description`.
- Methods in the agent class register automatically. Register other tools with `.tools(instanceOrClass)`. Registering a class needs a `DependencyProvider`.
- Entities, Workflows, and Views can be tools. **Agents cannot be tools.** Use a Workflow or an AutonomousAgent with `Delegation` to coordinate agents.
- Tools can return `MessageContent` (image or PDF).
- Limit: `akka.javasdk.agent.max-tool-call-steps` (default 100).

### MCP support
- **Consume** remote MCP tools:
  - `.mcpTools(RemoteMcpTools.fromService("svc"), RemoteMcpTools.fromServer("https://...").addClientHeader(...).withAllowedToolNames(Set.of(...)))`
  - Add `RemoteMcpTools.ToolInterceptor` to intercept calls.
- **Expose** an MCP server: `@McpEndpoint(serverName, serverVersion)` + `@Acl`, with `@McpTool`, `@McpResource`, and `@McpPrompt`.
  - The default path is `/mcp`.
  - Transport is stateless Streamable HTTP (spec 2025-03-26).
  - Tool parameters must be primitives, boxed primitives, or String. Use `Optional<T>` for optional parameters.
  - MCP OAuth 2.1 is not supported.

### Multi-agent orchestration
- **Static:** a Workflow with `@StepName` steps that call `componentClient.forAgent().inSession(workflowId)`.
  - Set `WorkflowSettings.builder().stepTimeout(...)`.
  - Set `defaultStepRecovery(RecoverStrategy.maxRetries(2).failoverTo(...))`.
- **Dynamic:** an `AutonomousAgent` with `Delegation.to(A.class, B.class)`. The model chooses the worker from the `@Component(description=...)` text.

### Evaluation and guardrails
- **Guardrails** (3.7.0+) are configured under `akka.javasdk.agent.guardrails."<name>" { class, agents, agent-roles, category, report-only, fail-mode, tools, control-id }`.
  - Interfaces: `ModelCallGuardrail`, `AgentResponseGuardrail`, `ToolCallGuardrail`.
  - A guardrail returns `Decision.Allow`, `Decision.Deny`, or `Decision.Fail`.
  - Built-in: `ModelCallSimilarityGuard` (jailbreak examples).
  - **Gotcha:** the runtime applies no timeout to guardrails.
- **Classifiers, Evaluators, ledger, and sanitizers** also exist (see the URL table).
- **Offline eval:** package `akka.javasdk.testkit.eval`.
  - Classes: `EvalCase`, `Evaluators`, `ExperimentRunner`, `Gate`, `EvalReport`, `Judge`, and `EvalCaseParser` (replay).
  - Reports go to `target/eval-reports`.
  - **The docs say access requires contacting Akka support.**

| URL | What | Verified |
|---|---|---|
| https://doc.akka.io/sdk/agents.html.md | Agents | yes |
| https://doc.akka.io/sdk/model-provider-details.html.md | Model provider config (all keys) | yes |
| https://doc.akka.io/sdk/agents/extending.html.md | Function tools and MCP tools | yes |
| https://doc.akka.io/sdk/agents/memory.html.md | Session memory | yes |
| https://doc.akka.io/sdk/agents/orchestrating.html.md | Multi-agent orchestration | yes |
| https://doc.akka.io/sdk/autonomous-agents.html.md | Autonomous agents | yes |
| https://doc.akka.io/sdk/agents/guardrails.html.md | Guardrails | yes |
| https://doc.akka.io/testing/evaluation/index.html.md | Evaluation (EvalKit) | yes |
| https://doc.akka.io/sdk/mcp-endpoints.html.md | MCP endpoints | yes |
| https://doc.akka.io/sdk/agents/prompt.html.md | Prompts | UNVERIFIED |
| https://doc.akka.io/sdk/agents/calling.html.md | Agent invocation | UNVERIFIED |
| https://doc.akka.io/sdk/agents/structured.html.md | Structured responses | UNVERIFIED |
| https://doc.akka.io/sdk/agents/streaming.html.md | Streaming | UNVERIFIED |
| https://doc.akka.io/sdk/agents/testing.html.md | Testing agents | UNVERIFIED |
| https://doc.akka.io/sdk/agents/classifiers.html.md | Classifiers | UNVERIFIED |
| https://doc.akka.io/sdk/evaluators.html.md | Runtime evaluators and the ledger | UNVERIFIED |
| https://doc.akka.io/sdk/sanitization.html.md | Data sanitization | UNVERIFIED |
| https://doc.akka.io/testing/evaluation/judges.html.md | Model judges | UNVERIFIED |
| https://doc.akka.io/concepts/ai-agents.html.md | Agent concepts | UNVERIFIED |

---

## 4. AI-assisted development resources from Akka

### 4a. Claude Code plugin (recommended path)
```
/plugin marketplace add akka/ai-marketplace@stable    # docs page omits @stable. akka.ai and the README pin it.
/plugin install akka@ai-marketplace
/reload-plugins
/akka:setup                                            # in a new empty folder: installs CLI, JVM, Maven, download token
```
- **Selector:** `akka@ai-marketplace`.
  - A third-party listing (claudepluginhub) and one search snippet show `akka@akka-ai-marketplace`.
  - The repo `marketplace.json` declares `"name": "ai-marketplace"`, so `akka@ai-marketplace` is correct.
- **Update:** `/plugin marketplace update` then `/plugin update akka`.
- **Offline fallback:** `git clone https://github.com/akka/ai-marketplace.git`, then `/plugin marketplace add /path/to/ai-marketplace`.
- **Plugin version:** `akka` 2.9.4. The old `akka-specify` 2.1.0 plugin is **deprecated**. Uninstall it.
- **The plugin includes:**
  - About 22 commands: setup, constitution, specify, clarify, plan, tasks, analyze, checklist, implement, harnesses, converge, review, build, inspect, reliability, deploy, issues, status, conform, ship, mode, docs.
  - The Akka MCP server, registered automatically.
  - On-demand SDK docs.
- **Command syntax:** the plugin uses `/akka:<name>`. CLI-installed commands use `/akka.<name>` (in `.claude/commands/`). Codex uses `$akka:*`.
- **Repo license:** Apache-2.0.

### 4b. Spec-Driven Development (Akka Specify, based on spec-kit)
- **Flow:** setup, constitution, specify, clarify, plan, tasks, implement, build, inspect, deploy.
- **Without the plugin:** run `akka specify init <dir> --agent claude-code`. This writes `.akka/` (templates and constitution), slash commands, and MCP config.
- **Without `--agent`:** the command writes `AGENTS.md` and `.mcp.json`.
- **Modes:**
  - À la carte (default).
  - Enforced (`/akka:mode enforced`): exit conditions, auditors, and a coverage gate. `akka specify conform` / `/akka:conform` gives a verdict.
- **Feature specs** go in `specs/00N-feature/`, each with its own git branch.

### 4c. Akka MCP server
- Command: stdio `akka mcp serve`. The `akka mcp` command group is marked "[Incubating]".
- Flag: `--toolsets` with values `projects, services, routes, descriptor, backoffice, sdd, local, build, git, browser, exit-conditions` (default `all`).
- Generic config:
```json
{ "mcpServers": { "akka": { "command": "akka", "args": ["mcp", "serve"] } } }
```
- Tool names from AGENTS.md:
  - SDD: `akka_sdd_constitution`, `akka_sdd_get_template`, `akka_sdd_create_spec`, `akka_sdd_list_specs`, `akka_sdd_list_templates`, `akka_sdd_init`
  - Build: `akka_maven_compile`, `akka_maven_test`, `akka_maven_verify`
  - Local: `akka_local_start`, `akka_local_run_service`, `akka_local_status`, `akka_local_logs`, `akka_local_request`, `akka_local_stop_service`
  - Deploy: `akka_build_image`, `akka_services_deploy`, `akka_services_logs`, `akka_routes_create`
  - Inspect: `akka_backoffice_*`
  - Other: `akka_refresh`, `akka_ec_conform`
- Prerequisites: the CLI is on PATH and an `AGENTS.md` file is at the project root.

### 4d. Context files
- `AGENTS.md` template (48 KB):
  - Sections: MCP driving loop, docs strategy, component selection, package structure, naming, imports, patterns for every component, test patterns, "Critical Gotchas", and a self-review checklist.
  - **This is the best seed for the subagent prompt.**
- `CLAUDE.md`: the CLI generates it. It points to AGENTS.md and the guidelines. Put your own rules in CLAUDE.md so that updates to AGENTS.md do not overwrite them.
- `akka-context/` folder: created by `akka code init` and refreshed by `akka code context-update`. The bundle is `akka-docs-md.zip` (about 1.56 MB, updated 2026-11-03). Add it to `.gitignore`, except for Codex.

| URL | What | Verified |
|---|---|---|
| https://akka.ai/ | Harness install instructions (all assistants), CLI install, MCP | yes |
| https://akka.ai/llms.txt | akka.ai LLM index | yes |
| https://akka.ai/setup | Alias for the setup skill (per README) | **no (404)** |
| https://github.com/akka/ai-marketplace | Plugin marketplace repo | yes |
| https://raw.githubusercontent.com/akka/ai-marketplace/main/.claude-plugin/marketplace.json | Marketplace manifest (akka 2.9.4) | yes |
| https://www.claudepluginhub.com/plugins/akka-akka-plugins-akka | Third-party plugin listing | yes |
| https://doc.akka.io/sdk/ai-coding-assistant.html | "Using an AI coding assistant" | yes |
| https://doc.akka.io/java/ai-coding-assistant.html | Older path of the same page | yes |
| https://doc.akka.io/sdk/ai-coding-assistant-guidelines.html.md | Coding guidelines for AI | yes |
| https://doc.akka.io/_attachments/AGENTS.md | Official AGENTS.md template | yes |
| https://doc.akka.io/sdk/_attachments/AGENTS.md | (inferred path) | **no (404)** |
| https://doc.akka.io/sdk/_attachments/akka-docs-md.zip | akka-context docs bundle | yes |
| https://doc.akka.io/llms.txt | Docs index | yes |
| https://doc.akka.io/llms-full.txt | Full docs, about 1.76 MB | yes |
| https://doc.akka.io/llms-ctx.txt | Docs without optional parts | yes |
| https://doc.akka.io/llms-ctx-full.txt | Docs with optional parts | yes |
| https://akka.io/llms.txt | Corporate llms.txt | yes |
| https://doc.akka.io/sdk/register-mcp.html.md | Register the Akka MCP server | yes |
| https://doc.akka.io/reference/cli/akka-cli/akka_mcp_serve.html | `akka mcp serve` reference (the `.html.md` variant gives 404) | yes |
| https://doc.akka.io/reference/cli/akka-cli/akka_code_context-update.html | `akka code context-update` | yes |
| https://doc.akka.io/getting-started/set-up-ai-install.html | Install through an AI harness | yes |
| https://doc.akka.io/getting-started/set-up-harness.html | Manual harness setup (Claude Code tab) | yes |
| https://doc.akka.io/sdk/spec-driven-development.html | SDD guide and command list | yes |
| https://doc.akka.io/sdk/enforced-mode.html | Enforced mode | yes |
| https://doc.akka.io/sdk/a-la-carte-mode.html | À la carte mode | yes |
| https://doc.akka.io/reference/specify/index.html | Specify reference | yes |
| https://doc.akka.io/getting-started/spec-your-first-agent.html.md | Spec-first tutorial | UNVERIFIED |
| https://akka.io/blog/introducing-akka-specify | Akka Specify blog (Mar 2026) | yes |
| https://raw.githubusercontent.com/akka/akka-sdk/main/CLAUDE.md | Contributor CLAUDE.md (SDK repo only, not for users) | yes |
| https://raw.githubusercontent.com/akka/akka-sdk/main/AGENTS.md | Contributor AGENTS.md. Points to `docs/src/modules/ROOT/attachments/AGENTS.md`, the user template source | yes |

---

## 5. Context7

These IDs come from https://context7.com/api/v1/search?query=akka (verified):

| Library ID | What | Snippets | Updated | Page verified |
|---|---|---|---|---|
| `/akka/akka-sdk` | Akka SDK repo | 12,217 (1.38M tokens) | 2026-10-12 | yes (https://context7.com/akka/akka-sdk) |
| `/llmstxt/doc_akka_io_llms_txt` | doc.akka.io llms.txt | 4,849 | 2026-09-27 | yes (https://context7.com/llmstxt/doc_akka_io_llms_txt) |
| `/akka/akka-core` | Akka core libraries (actors etc.) | 6,910 | 2026-09-01 | id in search only |
| `/websites/doc_akka_io_libraries_akka-core_2_4` | Akka core docs (old 2.4) | 9,345 | 2026-10-20 | id in search only |
| `/websites/doc_akka_io_alpakka-kafka_current` | Alpakka Kafka (only 4 snippets) | 4 | 2026-09-12 | id in search only |

Avoid `/akkadotnet/akka.net`, which is the .NET port.

---

## 6. Gotchas

### Licensing
- Akka (libraries and SDK) uses **BSL 1.1**. Each release becomes Apache 2.0 after **3 years**.
- Dev, test, and pre-production use is free.
- **Production needs a commercial license** from Lightbend/Akka. Many subscriptions are free, and paid ones start at $0.25 per core hour.
- Non-profits and governments with a budget over $25M must pay.
- The akka-sdk repo LICENSE is BSL 1.1.
- The `akka/ai-marketplace` plugin is Apache-2.0.
- Source: https://akka.io/bsl-license-faq (yes). https://akka.io/license-faq gives **404**.

### SDK compared with the Akka libraries
- The **Akka SDK** (`io.akka:akka-javasdk-*`, imports `akka.javasdk.*`) is a high-level component model: Agents, Entities, Workflows, Views, Consumers, Timed Actions, and Endpoints. It runs on the Akka runtime (local or Akka Automated Operations).
- The **Akka libraries** are Actors/Typed, Streams, Cluster, Persistence, HTTP 10.7.6, gRPC 2.5.13, Alpakka, and Projections 1.6.25. The libraries version set is 25.10.18 and core is 2.10.x. They are the classic low-level toolkit. Index: https://doc.akka.io/libraries/index.html (yes).
- The agent must not mix these up. Training data is full of classic actor and Lagom/Kalix code.
- From AGENTS.md: "Use `io.akka.*` imports → use `akka.*`".

### Kalix legacy
- The SDK was formerly Kalix. The base image still comes from `us-docker.pkg.dev/kalix-images/...`. Old Kalix samples are not valid for SDK 3.x.

### API drift that LLMs often get wrong (from AGENTS.md "Critical Gotchas")
- Use `.invoke()`, not `.invokeAsync()`.
- Use synchronous endpoints, not `CompletionStage`.
- Workflows use `settings()` and step methods with method references. Do not use `definition()`.
- Do not put `@Component` on endpoints. Every endpoint needs `@Acl`.
- `ComponentClient` is not allowed in Entities or Views.
- View multi-row results need `SELECT * AS items` and a wrapper record.
- Put `@Consume` on the `TableUpdater`, not on the View.
- Events need `@TypeName` and a sealed interface.
- `applyEvent` must be pure.
- Use a `Bootstrap implements ServiceSetup`, not a `Main` class.
- An Agent has one command handler. An AutonomousAgent has only `definition()`.
- Static web UI files go in `src/main/resources/static-resources`.

### Other gotchas
- **Model names in the docs are examples.** The Agents page config uses `claude-opus-4-6`, but its prose mentions `claude-sonnet-4`. Set `model-name` explicitly.
- **Breaking changes in 3.7.0:** the release removed APIs deprecated in 3.5.x, renamed config keys, and moved the TestKit to JUnit 6. Code from 3.5/3.6 samples can fail to compile.
- **Maven Central "latest" is wrong.** It reports 3.15.19. Pin 3.7.1.
- **Docs have two command styles:** `/akka.specify` (CLI-installed) and `/akka:specify` (plugin).
- **EvalKit access** may need Akka support.
