# Project rules for Claude Code

Use the `akka-neo4j` subagent for Akka SDK and Neo4j work.

## Version traps

- Pin `akka-javasdk-parent` to 3.7.1. Maven Central shows a stray 3.15.19 as "latest". Do not use it.
- Samples for Akka SDK 3.5 or 3.6 can fail to compile on 3.7.
- Neo4j driver 6 removed `readTransaction`, `writeTransaction` and `RxSession`.
- Use the Cypher 25 `SEARCH` clause. `db.index.vector.queryNodes` is deprecated.

## Safety

- The Neo4j MCP server is read-only. It connects only to a dev database with test data.
- Claude Code blocks deploy, route and project commands in this repo. A person runs them.

Put team rules in this file, not in `AGENTS.md`. The Akka CLI overwrites `AGENTS.md` when it updates.
