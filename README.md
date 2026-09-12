# eng-skills

A Claude Code plugin marketplace with two plugins:

- **`go-review`** — automatic Go code review after Claude writes code. A Sonnet-backed reviewer subagent, a concurrency auditor, a `Stop` hook that gates the turn on blocking findings, and a `go-review` skill with references on idiomatic Go, SOLID, concurrency/races, and design patterns.
- **`architect-council`** — five architect-level design skills (security & compliance, caching, database, infrastructure, backend services) plus a `design-review` coordinator that routes between them during feature planning.

## Layout

```
.claude-plugin/marketplace.json          the registry Claude Code reads
plugins/
  go-review/
    .claude-plugin/plugin.json
    agents/go-reviewer.md                model: sonnet
    agents/go-concurrency-auditor.md     model: sonnet
    hooks/hooks.json                     PostToolUse marker + Stop hook (agent type)
    skills/go-review/SKILL.md
    skills/go-review/references/         idiomatic, concurrency, solid, patterns
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
/plugin install architect-council@eng-skills
```

Or non-interactively, for a new dev box or a container image:

```bash
claude plugin marketplace add <your-user>/<your-repo>
claude plugin install go-review@eng-skills
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

Install `go-review` globally but enable it only where it belongs — the `Stop` hook fires on every turn otherwise. In a Go project's `.claude/settings.json`:

```json
{
  "enabledPlugins": {
    "go-review@eng-skills": true,
    "architect-council@eng-skills": true
  }
}
```

## Using it

**Go review** runs on its own. A `PostToolUse` hook drops a marker file (`.claude/.go-touched`) whenever Claude edits a `.go` file. The `Stop` hook returns immediately unless that marker exists, so your own uncommitted work in progress does not trigger a review. When it does run, it executes `gofmt`/`go vet`/`go build`, reads the changed files against the `go-review` skill, and blocks the turn with findings if anything is BLOCKING. Claude fixes and tries to stop again; a finding that was already reported does not block a second time. You can also invoke the reviewers directly:

```
@go-reviewer look at the changes on this branch
@go-concurrency-auditor audit the worker pool in internal/queue
```

**Architecture review** is invoked when you are planning:

```
/design-review we need to add per-org usage metering to the API
```

The coordinator picks which specialist perspectives apply and names the conflicts between them. Or call one directly: `/arch-database`, `/arch-caching`, `/arch-security-compliance`, `/arch-infrastructure`, `/arch-backend-services`.

## Tuning

**The Stop hook is the part most likely to annoy you.** It lives in `plugins/go-review/hooks/hooks.json`. Three things to know:

1. Claude Code overrides a `Stop` hook after it blocks **8 consecutive times**. If your reviewer keeps finding new things, you hit the cap and the turn ends with a warning. Raise it with `CLAUDE_CODE_STOP_HOOK_BLOCK_CAP`, or tighten the prompt's definition of BLOCKING.
2. `Stop` hooks fire whenever Claude finishes responding, not only after coding. The hook agent returns early unless `.claude/.go-touched` exists, and that marker is only written by the `PostToolUse` hook when Claude edits a `.go` file — keep both halves if you edit it. Add `.claude/.go-touched` to your `.gitignore`.
3. The hook agent has no memory between invocations. The prompt tells it to read the transcript when `stop_hook_active` is true and to downgrade already-reported findings to WARNING, which is what stops it from looping on a finding you decided to keep. If you rewrite the prompt, keep that section.
4. Agent-type hooks are marked experimental in the Claude Code docs. If the behaviour changes, the fallback is to drop the hook and invoke `@go-reviewer` manually, or switch to `"type": "prompt"` (single LLM call, no tool access — it can check whether a review happened but cannot read the diff itself).

**Model pinning.** The reviewer subagents set `model: sonnet` in frontmatter, so they stay on Sonnet while your main session runs Opus. The docs do not currently say which model an agent-type hook runs on or how to pin it; assume it follows the session's model.

**Who reviews when.** Three things could review the same edit: the `go-review` skill loaded into the main session, the `go-reviewer` subagent, and the `Stop` hook. They are deliberately given different jobs. The skill is the knowledge and loads when Go is being written or reviewed. The hook is the automatic gate. The subagent is on-request only (`@go-reviewer ...`), so the main session does not delegate a review that the hook is about to run anyway.

**Skill triggering.** The five architect skills have deliberately distinct descriptions so they do not all fire on "help me plan a feature" — each names its own concrete triggers, and `design-review` is the entry point that pulls in the relevant ones. If one over- or under-triggers, the `description` field is the only thing that controls it. Rewriting a description is the fix; the skill body has no effect on when it loads.

**Go references** are in `plugins/go-review/skills/go-review/references/`. Add your team's conventions there and point to the file from `SKILL.md`'s table — that keeps `SKILL.md` small, since it loads in full whenever the skill triggers while references load only when read.
