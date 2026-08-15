# SKILLS

> **This repository has moved to [github.com/ghafarallahi/SKILLS](https://github.com/ghafarallahi/SKILLS).**
>
> This copy is archived and read-only. It stays here so old links keep working, but it
> receives no further changes. The new repository has the same history and the same tags,
> plus everything released after v0.2.0.
>
> ```bash
> git remote set-url origin https://github.com/ghafarallahi/SKILLS.git
> ```

[![release](https://img.shields.io/github/v/release/ghafarallahi/SKILLS)](https://github.com/ghafarallahi/SKILLS/releases/latest)
[![tests](https://github.com/ghafarallahi/SKILLS/actions/workflows/tests.yml/badge.svg)](https://github.com/ghafarallahi/SKILLS/actions/workflows/tests.yml)

Claude Code customizations that make an independent model — OpenAI's Codex — confirm work
before it's reported as done. Claude doesn't self-certify.

## Quick start

```bash
npm install -g @openai/codex && codex login
git clone https://github.com/ghafarallahi/SKILLS.git ~/MyProject/SKILLS
bash ~/MyProject/SKILLS/hooks/install.sh
```

Restart Claude Code. From then on every turn that leaves changed files gets reviewed before
it can be called done — a rejection is fed back for a fix, not shown to you as a suggestion.

```bash
bash ~/MyProject/SKILLS/hooks/selftest.sh   # 17 cases, offline, confirms it all works
```

[`install.sh`](hooks/install.sh) is idempotent and non-destructive: it skips anything that
already exists and isn't its own symlink, and merges into `settings.json` rather than
replacing it. See [Install](#install) for what it touches and how to do it by hand.

---

Skills are advisory — they trigger on relevance. The hooks are enforcing — they run whether
or not anyone remembers them:

| Path | What it is |
|---|---|
| [`skills/codex-check/SKILL.md`](skills/codex-check/SKILL.md) | A skill Claude follows when asked to verify work ("codex check this"). Advisory — it triggers on relevance. |
| [`skills/target/SKILL.md`](skills/target/SKILL.md) | `/target <task>` — break the task down, ask everything up front, then run to the end without check-ins. |
| [`skills/commit-message/SKILL.md`](skills/commit-message/SKILL.md) | Writes the message from the staged diff, not from intent — and never claims a test it didn't run. |
| [`skills/review-changes/SKILL.md`](skills/review-changes/SKILL.md) | Reviews a diff by hand: severity-ranked, every finding refuted first, silence when it's clean. |
| [`skills/write-tests/SKILL.md`](skills/write-tests/SKILL.md) | Tests proven to fail against the broken code first — otherwise they pass for the wrong reason. |
| [`skills/root-cause/SKILL.md`](skills/root-cause/SKILL.md) | Debugging: reproduce, halve the space, observe values, fix where the callers converge. |
| [`skills/refactor/SKILL.md`](skills/refactor/SKILL.md) | Structure changes that provably don't change behavior — green in between every step. |
| [`skills/code-comments/SKILL.md`](skills/code-comments/SKILL.md) | Comments that carry what code can't: why, invariants, domain rules — and stay true as it changes. |
| [`skills/context-budget/SKILL.md`](skills/context-budget/SKILL.md) | Read the smallest slice that answers the question. Search first. Do not read unchanged data two times. |
| [`skills/write-docs/SKILL.md`](skills/write-docs/SKILL.md) | Docs you can act on: every command run for real, limits stated, updated in the same commit. |
| [`skills/pr-description/SKILL.md`](skills/pr-description/SKILL.md) | PR bodies written for the reviewer: why, where to start reading, how it was verified, what's out of scope. |
| [`skills/ci-verify/SKILL.md`](skills/ci-verify/SKILL.md) | Runs what the pipeline runs, on the commit you're about to push, before you push it. |
| [`skills/security-audit/SKILL.md`](skills/security-audit/SKILL.md) | Trust boundaries, injection sinks, per-object authorization, business-logic abuse, dependency risk. |
| [`skills/release/SKILL.md`](skills/release/SKILL.md) | Ship so you can get back: honest bumps, expand/contract migrations, abort criteria set in advance. |
| [`hooks/codex-review.sh`](hooks/codex-review.sh) | A `Stop` hook. Runs on **every** turn end, no discretion involved. |
| [`hooks/record-edit.sh`](hooks/record-edit.sh) | A `PostToolUse` hook that logs which files got written, so the review works outside git. |
| [`hooks/reset-count.sh`](hooks/reset-count.sh) | Clears the per-session block counters so the hook will block again. |
| [`hooks/selftest.sh`](hooks/selftest.sh) | Regression check for the hooks and the installer. Stub `codex`, throwaway repos, no network. |
| [`hooks/install.sh`](hooks/install.sh) | Idempotent installer — symlinks, plus a non-destructive merge into `settings.json`. |
| [`CHANGELOG.md`](CHANGELOG.md) | What each version does to your machine, what it costs, and how to uninstall. |
| [`CODEX-REVIEW.md`](CODEX-REVIEW.md) | Codex's independent verdict on every skill, what changed because of it, and what was declined. |

## The skill files use Simplified Technical English

Every file in `skills/` is written in ASD-STE100 Simplified Technical English: short
sentences, the active voice, one meaning for each word, and no metaphor. The purpose is an
instruction that an agent cannot read in two ways.

A rewrite into a controlled language loses content if nobody checks it. Each file was
compared against its previous version, and the instructions that the first rewrite lost were
put back. [`CODEX-REVIEW.md`](CODEX-REVIEW.md) records what was lost and what was restored.

If you add a skill, keep the style: a maximum of 25 words for each sentence, a list for each
enumeration, and the same word for the same thing.

## How the hook behaves

On each turn end it collects the work in progress, hands it to `codex exec`, and reads the
verdict off the last line of Codex's reply. What counts as "work in progress" depends on
where you are:

- **In a git repo** — `git diff HEAD` plus untracked files. Committed work is invisible to
  it, so committing is what ends a review loop.
- **Anywhere else** — the files this session wrote, recorded by `record-edit.sh`. There's no
  diff to show, so Codex judges the files as they stand. At most 30 files go into one
  review; the rest stay queued for the next turn. Reviewed paths are dropped on APPROVE —
  nothing else would clear them, and every later turn would re-review the same files.

The verdict decides the turn:

- **APPROVE** → silent, turn ends.
- **REJECT** → the turn is blocked and the findings are fed back to Claude to fix.
- **Anything else** → a loud `changes are UNVERIFIED` warning, turn ends.

It no-ops when there's nothing to review — clean tree, or no files written — so it only
costs a Codex run when there's something to look at.

### Deliberate limits

- **Fails open.** Codex missing, crashed, or unparseable → warn, never block. A hook that
  can wedge a session is worse than an unverified diff.
- **Two blocks per session, max.** Then it steps aside and hands the disagreement to you
  rather than looping. Run `hooks/reset-count.sh` to re-arm it.

## Install

The repo is canonical; `~/.claude` holds symlinks to it. `install.sh` (see
[Quick start](#quick-start)) links every directory under `skills/` plus the two hook
scripts the harness invokes —

```
~/.claude/skills/<every skill in skills/>
~/.claude/hooks/codex-review.sh
~/.claude/hooks/record-edit.sh
```

— and adds one `PostToolUse` and one `Stop` entry to `~/.claude/settings.json`, backing the
file up to `settings.json.bak` first. Nothing else in that file is touched. It honors
`$HOME`, so you can rehearse the whole thing against a scratch directory before running it
for real.

<details>
<summary>Doing it by hand instead</summary>

```bash
for s in ~/MyProject/SKILLS/skills/*/; do ln -s "${s%/}" ~/.claude/skills/; done
ln -s ~/MyProject/SKILLS/hooks/codex-review.sh ~/.claude/hooks/codex-review.sh
ln -s ~/MyProject/SKILLS/hooks/record-edit.sh ~/.claude/hooks/record-edit.sh
```

Then wire both hooks up in `~/.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          { "type": "command", "command": "bash ~/.claude/hooks/record-edit.sh", "timeout": 10 }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/codex-review.sh",
            "timeout": 300,
            "statusMessage": "Codex reviewing changes..."
          }
        ]
      }
    ]
  }
}
```

</details>

Requires `codex` (`npm install -g @openai/codex`), authenticated, plus `jq` and `git`.

## Verifying it

```bash
bash hooks/selftest.sh
```

Seventeen cases covering both hooks, the installer, and `reset-count.sh`. Every case stubs
`codex` and gets its own `TMPDIR` and `HOME`, so the suite is deterministic, offline, and
costs nothing. Non-zero exit if anything fails.

```
install
  ok   the old settings.json is kept as .bak before it's rewritten
  ok   install links the files and wires both hooks
  ok   install is idempotent and keeps existing settings
  ok   a skipped link fails loudly instead of reporting success
  ok   empty skills/ makes no dangling link
  ...

17 passed, 0 failed
```

### By hand

The hook reads its working directory and session id from the JSON on stdin, so you can run
it against any folder without waiting for a turn to end. Plant a defect, then:

```bash
# in a git repo — reviews the diff
echo '{"cwd":"/path/to/repo","session_id":"manual"}' | bash ~/.claude/hooks/codex-review.sh

# in a plain folder — reviews the files listed for that session id
S="${TMPDIR:-/tmp}/claude-codex-review"
echo /path/to/folder/thing.py > "$S/manual.files"
echo '{"cwd":"/path/to/folder","session_id":"manual"}' | bash ~/.claude/hooks/codex-review.sh
```

A rejection comes back as the JSON the hook feeds to Claude:

```json
{
  "decision": "block",
  "reason": "... After the final failed attempt, the function sleeps and implicitly
             returns None, swallowing the exception ...\n\nCODEX_VERDICT: REJECT",
  "systemMessage": "Codex rejected the changes — Claude is fixing them"
}
```

Silence and exit 0 means approved. Use a throwaway `session_id` — the real one carries the
block counter and the queued file list.
