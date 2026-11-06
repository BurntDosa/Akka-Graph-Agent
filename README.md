# Akka-Graph-Agent

A Claude Code kit for Akka SDK services that store data in Neo4j. It gives each teammate the same agent, the same docs and the same safety rules.

## What is in the kit

| Path | What it does |
|---|---|
| `.claude-plugin/marketplace.json` | Makes this repo a plugin marketplace named `akka-neo4j-kit` |
| `plugins/akka-neo4j/agents/akka-neo4j.md` | The `akka-neo4j` subagent |
| `plugins/akka-neo4j/skills/` | 5 Neo4j skills, copied from `neo4j-contrib/neo4j-skills` at a fixed commit. See `SOURCE.md`. |
| `plugins/akka-neo4j/.mcp.json` | The Neo4j MCP server, read-only, telemetry off |
| `template/.claude/settings.json` | Project settings: installs the plugins, blocks deploy and database writes |
| `template/CLAUDE.md` | Project rules: version traps and safety rules |
| `docs/` | Research notes for Akka and Neo4j (6 Nov 2026) |

The template also installs the official Akka plugin `akka@ai-marketplace`, pinned to tag `v2.9.4`. That plugin gives the `/akka:*` commands, the Akka MCP server and the `AGENTS.md` rules.

## Where the kit lives

The kit is at `https://github.com/BurntDosa/Akka-Graph-Agent`. Projects use the tag `v1.0.0`.

To install only the plugin, without the project template, run `/plugin marketplace add BurntDosa/Akka-Graph-Agent@v1.0.0`. Then run `/plugin install akka-neo4j@akka-neo4j-kit`. This way does not add the deny rules, so use the template for real projects.

## Set up a machine (each teammate, one time, 20 min)

### 1. Install the Neo4j MCP server v2.1.0

Windows (PowerShell):

```powershell
$v = "2.1.0"; $f = "neo4j-mcp_Windows_x86_64.zip"
$u = "https://github.com/neo4j/mcp/releases/download/v$v"
curl.exe -sSLO "$u/$f"; curl.exe -sSLO "$u/neo4j-mcp_${v}_checksums.txt"
$want = ((Select-String -SimpleMatch $f "neo4j-mcp_${v}_checksums.txt").Line -split "\s+")[0]
if ((Get-FileHash $f -Algorithm SHA256).Hash.ToLower() -ne $want) { throw "Checksum does not match. Do not use this file." }
Expand-Archive $f -DestinationPath "$HOME\bin\neo4j-mcp" -Force
```

Then add `%USERPROFILE%\bin\neo4j-mcp` to your user PATH.

macOS or Linux (bash). Use `Darwin_arm64` for an Apple silicon Mac:

```bash
v=2.1.0; f=neo4j-mcp_Linux_x86_64.tar.gz
u=https://github.com/neo4j/mcp/releases/download/v$v
curl -sSLO "$u/$f" && curl -sSLO "$u/neo4j-mcp_${v}_checksums.txt"
sha256sum --check --ignore-missing "neo4j-mcp_${v}_checksums.txt"   # macOS: shasum -a 256 -c --ignore-missing
mkdir -p ~/.local/bin && tar -xzf "$f" -C ~/.local/bin neo4j-mcp
```

Done when: `neo4j-mcp --help` runs in a new terminal.

### 2. Set the Neo4j connection variables

Use a dev database with test data only. Aura Free is a good choice.

Windows (PowerShell). Do not type the password on the command line:

```powershell
[Environment]::SetEnvironmentVariable("NEO4J_MCP_URI", "neo4j+s://xxxx.databases.neo4j.io", "User")
[Environment]::SetEnvironmentVariable("NEO4J_MCP_USERNAME", "neo4j", "User")
$p = Read-Host "Neo4j dev password"; [Environment]::SetEnvironmentVariable("NEO4J_MCP_PASSWORD", $p, "User"); Remove-Variable p
```

macOS or Linux: add the three `export NEO4J_MCP_...=...` lines to `~/.profile`.

Optional: set `NEO4J_MCP_DATABASE`. The default is `neo4j`.

Done when: a new terminal shows the URI with `echo $env:NEO4J_MCP_URI` (Windows) or `echo $NEO4J_MCP_URI`.

## Add the kit to an Akka project (each project, 10 min)

1. Copy `template/.claude/settings.json` and `template/CLAUDE.md` to the project root. If the project already has these files, merge the keys and the text.
2. Add `akka-context/` to the project `.gitignore`.
3. Commit the files, so that teammates get them when they clone.
4. Start `claude` in the project root and accept the trust dialog. Claude Code then installs both plugins.
5. Run `/akka:setup`. It installs the Akka CLI, Java, Maven and the Akka download token if they are missing.

Done when: `/mcp` shows `plugin:akka:akka` and `plugin:akka-neo4j:neo4j` as connected, and `/agents` lists `akka-neo4j`.

## Check the setup (15 min)

| Test | Ask Claude | Expected result |
|---|---|---|
| Read schema | "Use the akka-neo4j agent. Show the Neo4j schema." | The agent calls `get-schema` and shows labels |
| No writes | "Delete all nodes in Neo4j." | No `write-cypher` tool exists. Claude Code blocks the call. |
| No deploy | "Deploy this service to Akka." | Claude Code blocks `akka_services_deploy` and `akka services` |
| Build | "Make an Event Sourced Entity `Customer` and a Consumer that writes customers to Neo4j." | `mvn verify` passes |

## Safety rules and their limits

- **Read-only database access.** The kit hard-codes `NEO4J_MCP_READ_ONLY=true`. Neo4j Community has no read-only user roles, so connect only to a dev database.
- **No deploy.** Deny rules block the Akka deploy, route, project and backoffice tools. They also block `akka services`, `akka routes` and `akka projects` in Bash and PowerShell.
- **Deny rules are not a sandbox.** A Bash deny rule does not match a command inside a script or `sh -c`. Review what the agent runs.
- **No bypass mode.** `disableBypassPermissionsMode` stops bypass mode in projects that use the template. Remove that line only if the team agrees.
- **No secrets in git.** The MCP config reads the password from an environment variable. Never commit a password.
- **Pinned versions.** Auto-update is off. The Akka plugin is pinned to `v2.9.4`. The Neo4j skills are copies from one commit.

## Update the kit (maintainer, 30 min)

1. Akka plugin: read the new release in `akka/ai-marketplace`. Look for new hooks and new MCP tools. Then change `"ref": "v2.9.4"` in `template/.claude/settings.json`.
2. Neo4j skills: clone `neo4j-contrib/neo4j-skills`, then copy the 5 skill folders over `plugins/akka-neo4j/skills/`. Update the commit in `SOURCE.md`.
3. Neo4j MCP server: change `v` in the install commands above. Read the release notes for changed env vars.
4. Raise `version` in `plugins/akka-neo4j/.claude-plugin/plugin.json`, then tag a new release such as `v1.1.0`.
5. Change the kit `ref` in each project `.claude/settings.json`.

Run `claude plugin validate .` before each tag.

## Remove the kit from a project

1. Delete the `extraKnownMarketplaces`, `enabledPlugins` and `permissions` keys that the template added.
2. Run `/plugin` and uninstall `akka` and `akka-neo4j`.
