# quarkus-logging

An opinionated [Claude Skill](https://agentskills.io/specification) for log statements in Quarkus applications. When an AI coding agent is adding or modifying logging in a Quarkus codebase, this skill loads into context and tells it which API to use, how to format messages, what level to pick, and what never to put in a log line.

The full guidance lives in [`SKILL.md`](./SKILL.md). The short version:

- Use the static `io.quarkus.logging.Log` API. Don't declare per-class `LoggerFactory` loggers — Quarkus rewrites `Log` calls at build time.
- **Always** use the `*f` variants (`Log.debugf`, `Log.infof`, `Log.warnf`, `Log.errorf`) when there are arguments. They're lazy — the formatter runs only if the level is enabled.
- **Never** concatenate strings inside a log call. `Log.debug("x=" + x)` evaluates eagerly; `Log.debugf("x=%s", x)` doesn't.
- Pass the `Throwable` first when logging an exception: `Log.errorf(e, "failed for id=%s", id)`.
- Pick the level by **audience** (operations, on-call, developer), not by emphasis.
- Don't log PII, secrets, or whole entities. Log identifiers, action verbs, decision branches.

The skill also ships an Excuse / Reality table for resisting the rationalizations that show up at 5:48 PM on a Friday: "just for the weekend, we'll roll it back Monday," "it's only debug level," "I can't trace this customer without their email."

## Installation

The skill is a single `SKILL.md` file with YAML frontmatter. Drop the directory containing it into your agent's skills location.

### Claude Code

User-level (active across every project):

```bash
git clone https://github.com/jeremyrdavis/quarkus-skill-logging.git ~/.claude/skills/quarkus-logging
```

Project-level (active only in one project):

```bash
git clone https://github.com/jeremyrdavis/quarkus-skill-logging.git <your-project>/.claude/skills/quarkus-logging
```

### Other agents

| Agent | Skills directory |
|---|---|
| Codex | `~/.agents/skills/quarkus-logging/` |
| Copilot CLI | follow Copilot's plugin install docs; the skill is loaded by the same `Skill` tool |
| Gemini CLI | follow Gemini's `activate_skill` docs |

Only the directory containing `SKILL.md` is required. The other files in this repo (`README.md`, `LICENSE`) are repo metadata and don't affect skill loading.

## How triggering works

Agents read the `description:` field in `SKILL.md`'s YAML frontmatter to decide whether to load the body of the skill into context. The current trigger fires when the agent is:

- writing or modifying a call to `Log.info / debug / warn / error / trace` (or their `*f` variants)
- importing `io.quarkus.logging.Log`
- choosing a log level
- formatting a log message with placeholders
- deciding whether a value (entity, payload, request body) is safe to log
- reviewing a diff with string concatenation inside a log call, missing parameterized format, or PII appearing in log output

If you want to broaden or narrow that trigger, edit the `description:` field — that's the part agents read every time.

## Authoring methodology

This skill was developed and verified using the [`superpowers:writing-skills`](https://github.com/anthropics/claude-code-plugins) RED-GREEN-REFACTOR workflow:

1. **RED** — run a multi-pressure scenario against a subagent without the skill loaded; capture the verbatim rationalizations the agent uses to justify the wrong choice.
2. **GREEN** — write or edit the skill addressing those specific rationalizations; re-run; verify the agent now picks the right option and cites the rule.
3. **REFACTOR** — capture any new rationalizations that surface and add them to the rule list, the Anti-patterns table, or the Excuse / Reality table.

The Excuse / Reality table in `SKILL.md` is built from rationalizations that real subagents produced under simulated time / authority pressure (e.g. a manager demanding PII-bearing logs to trace a specific customer's incident).

## License

MIT — see [`LICENSE`](./LICENSE).

## Contributions

PRs welcome, especially:

- Pressure-test scenarios that surface rationalizations the current skill doesn't address.
- Coverage for `quarkus-logging-json` / structured logging when the call-site rules differ.
- Extensions to other JBoss Logging features (MDC, NDC, log filters) where the rules at the call site change.

When proposing a rule change, please include the pressure scenario you used to validate it — the methodology section above describes the format.
