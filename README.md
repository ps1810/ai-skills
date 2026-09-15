# eng-skills

A Claude Code plugin marketplace with three plugins:

- **`go-review`** — Go conventions while writing and automatic review afterwards. A `go-coding` skill that loads when Go is being written, a `go-review` skill with references on idiomatic Go, SOLID, concurrency/races, testing, and design patterns, a Sonnet-backed reviewer subagent, a concurrency auditor, and a `Stop` hook that gates the turn on blocking findings.
- **`ts-review`** — the same shape for TypeScript. `ts-coding` and `ts-review` skills with references on strict types and boundary validation, async correctness, SOLID, testing, and patterns; a reviewer subagent, an async auditor, and a `Stop` hook. The coding skill includes a "coming from Go" section.
- **`architect-council`** — five architect-level design skills (security & compliance, caching, database, infrastructure, backend services) plus a `design-review` coordinator that routes between them during feature planning.

Each language plugin splits **coding** from **review** on purpose. The coding skill carries the conventions at write time so the code is right first; the review skill carries the procedure, severity scale, and output format. Both point at the same reference files, so there is one source of truth per topic.

## Layout

```
.claude-plugin/marketplace.json          the registry Claude Code reads
plugins/
  go-review/
    .claude-plugin/plugin.json
    agents/go-reviewer.md                model: sonnet
    agents/go-concurrency-auditor.md     model: sonnet
    hooks/hooks.json                     PostToolUse marker + Stop hook (agent type)
    skills/go-coding/SKILL.md            loads when writing Go
    skills/go-review/SKILL.md            loads when reviewing Go; canonical severity + format
    skills/go-review/references/         idiomatic, concurrency, solid, patterns, testing
  ts-review/
    .claude-plugin/plugin.json
    agents/ts-reviewer.md                model: sonnet
    agents/ts-async-auditor.md           model: sonnet
    hooks/hooks.json                     PostToolUse marker + Stop hook (agent type)
    skills/ts-coding/SKILL.md            loads when writing TypeScript
    skills/ts-review/SKILL.md            loads when reviewing TypeScript; canonical severity + format
    skills/ts-review/references/         idiomatic, async, solid, patterns, testing
  architect-council/
    .claude-plugin/plugin.json
    skills/design-review/SKILL.md        coordinator
    skills/arch-security-compliance/SKILL.md
    skills/arch-caching/SKILL.md
    skills/arch-database/SKILL.md
    skills/arch-infrastructure/SKILL.md
    skills/arch-backend-services/SKILL.md
```

## Setup

Replace `CHANGE-ME` and `you@example.com` in `.claude-plugin/marketplace.json` and both `plugin.json` files, then push to a GitHub repo. Public or private both work: Claude Code fetches marketplaces with your existing git credential helpers (`gh auth login`, SSH agent, or `git-credential-store`).

## Install on any machine

```
/plugin marketplace add <your-user>/<your-repo>
/plugin install go-review@eng-skills
/plugin install ts-review@eng-skills
/plugin install architect-council@eng-skills
```

Or non-interactively, for a new dev box or a container image:

```bash
claude plugin marketplace add <your-user>/<your-repo>
claude plugin install go-review@eng-skills
claude plugin install ts-review@eng-skills
claude plugin install architect-council@eng-skills
```

Test locally before pushing by adding the directory as a marketplace:

```
/plugin marketplace add /path/to/this/repo
```

Validate the manifests:

```bash
claude plugin validate .
```

## Enable per project

Install the language plugins globally but enable each only where it belongs — the `Stop` hook fires on every turn otherwise. In a Go project's `.claude/settings.json`:

```json
{
  "enabledPlugins": {
    "go-review@eng-skills": true,
    "architect-council@eng-skills": true
  }
}
```

In a TypeScript project, `ts-review@eng-skills` instead of `go-review@eng-skills`. In a repo with both languages, enable both; the hooks key on file extension and stay out of each other's way. Add `.claude/.go-touched` and `.claude/.ts-touched` to that project's `.gitignore`.

## Using it

**Go review** runs on its own. A `PostToolUse` hook drops a marker file (`.claude/.go-touched`) whenever Claude edits a `.go` file. The `Stop` hook returns immediately unless that marker exists, so your own uncommitted work in progress does not trigger a review. When it does run, it executes `gofmt`/`go vet`/`go build`, reads the changed files against the `go-review` skill, and blocks the turn with findings if anything is BLOCKING. Claude fixes and tries to stop again; a finding that was already reported does not block a second time. You can also invoke the reviewers directly:

```
@go-reviewer look at the changes on this branch
@go-concurrency-auditor audit the worker pool in internal/queue
```

**TypeScript review** works the same way with `.claude/.ts-touched`, `tsc --noEmit`, the project's `lint` script, and the `ts-review` skill. The hook detects the package manager from the lockfile and uses the project's own scripts where they exist.

```
@ts-reviewer look at the changes on this branch
@ts-async-auditor audit the queue consumer in src/jobs
```

**Writing code** needs no invocation. `go-coding` loads when Claude is about to write Go, `ts-coding` when it is about to write TypeScript. Each is the short form of the conventions the reviewer will hold the code to, with a table of which reference to read for the ground being covered.

**Architecture review** is invoked when you are planning:

```
/design-review we need to add per-org usage metering to the API
```

The coordinator explores the repo first, asks only what the code cannot answer, picks which specialist perspectives apply, names the conflicts between them, always fills in a verification-and-rollout section, and writes the result to `docs/design/<feature>.md` so the decision survives the session. Severity (BLOCKING / WARNING / CONSIDER) is defined once in `design-review`; each specialist says what qualifies in its domain and shows a good-versus-bad finding pair. Or call one directly for its full output template: `/arch-database`, `/arch-caching`, `/arch-security-compliance`, `/arch-infrastructure`, `/arch-backend-services`.

## Tuning

**The Stop hook is the part most likely to annoy you.** It lives in `plugins/go-review/hooks/hooks.json`. Three things to know:

1. Claude Code overrides a `Stop` hook after it blocks **8 consecutive times**. If your reviewer keeps finding new things, you hit the cap and the turn ends with a warning. Raise it with `CLAUDE_CODE_STOP_HOOK_BLOCK_CAP`, or tighten the prompt's definition of BLOCKING.
2. `Stop` hooks fire whenever Claude finishes responding, not only after coding. The hook agent returns early unless `.claude/.go-touched` exists, and that marker is only written by the `PostToolUse` hook when Claude edits a `.go` file — keep both halves if you edit it. Add `.claude/.go-touched` to your `.gitignore`.
3. The hook agent has no memory between invocations. The prompt tells it to read the transcript when `stop_hook_active` is true and to downgrade already-reported findings to WARNING, which is what stops it from looping on a finding you decided to keep. If you rewrite the prompt, keep that section.
4. Agent-type hooks are marked experimental in the Claude Code docs. If the behaviour changes, the fallback is to drop the hook and invoke `@go-reviewer` manually, or switch to `"type": "prompt"` (single LLM call, no tool access — it can check whether a review happened but cannot read the diff itself).

**Model pinning.** The reviewer subagents set `model: sonnet` in frontmatter, so they stay on Sonnet while your main session runs Opus. The docs do not currently say which model an agent-type hook runs on or how to pin it; assume it follows the session's model.

**Who reviews when.** Per language, four things could touch the same edit: the coding skill, the review skill, the reviewer subagent, and the `Stop` hook. They are deliberately given different jobs. The coding skill loads at write time and carries the conventions. The review skill loads only on an explicit review request and carries the procedure, severity scale, and output format; the hook and the subagents read it rather than restating it. The hook is the automatic gate. The subagents are on-request only (`@go-reviewer ...`), so the main session does not delegate a review that the hook is about to run anyway.

**Skill triggering.** The five architect skills have deliberately distinct descriptions so they do not all fire on "help me plan a feature" — each names its own concrete triggers, and `design-review` is the entry point that pulls in the relevant ones. If one over- or under-triggers, the `description` field is the only thing that controls it. Rewriting a description is the fix; the skill body has no effect on when it loads.

**References** are in `plugins/go-review/skills/go-review/references/` and `plugins/ts-review/skills/ts-review/references/`. Add your team's conventions there and point to the file from both the coding and review `SKILL.md` tables — that keeps each `SKILL.md` small, since it loads in full whenever the skill triggers while references load only when read. The coding skills reach the references by relative path (`../go-review/references/`), so keep the two skill directories side by side.
