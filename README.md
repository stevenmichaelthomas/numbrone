# numbrone

> numbrone is an independent community project built on top of [herdr](https://herdr.dev).
> It is not made by or affiliated with the herdr team.

**A chief of staff for your agents.** You give one agent requests in plain language: code
changes, research, documents, emails, whatever your projects need.
It works out which of your projects each request belongs to and hands it to that project's
manager. The manager runs workers in parallel, each in its own git worktree, and everyone
reports back up. All of it runs in [herdr](https://herdr.dev), so you can watch any agent at any time.

We all need a numbrone at work. This is yours.

```
            you, or sources the chief watches (Linear, GitHub issues, ...)
                        │  "add rate limiting to the api"
                        ▼
                 ┌─────────────┐
                 │    chief    │   routes, delegates, keeps track       ~/.numbrone
                 └──────┬──────┘
           ┌────────────┴─────────────┐
           ▼                          ▼
   ┌───────────────┐          ┌───────────────┐   ◀── sources each manager watches
   │  manager-api  │          │  manager-web  │       (Sentry, Datadog, ...)
   └───────┬───────┘          └───────┬───────┘   one per project: herdr workspace numbrone/<project>
      ┌────┴─────────┐                │
      ▼              ▼                ▼
 worker-rate-   worker-fix-     worker-new-       one per task: herdr tab and git worktree
 limit          login           navbar                               on branch numbrone/<task>

  reports flow back up:  worker ──▶ manager ──▶ chief ──▶ you
```

- **Chief**: the agent you talk to. Beyond quick answers, it doesn't do the work itself. It
  routes and delegates requests, and it tells you what's happening.
- **Managers**: one per project, living in that project's herdr workspace. A manager answers
  small things itself and spawns a worker for everything else: code, research, writing, emails.
  It reviews their work before reporting it as ready.
- **Workers**: one per task, each in its own tab and git worktree, so many tasks run at once
  without stepping on each other or on your own checkout. Work that isn't a change to the
  project (findings, drafts) goes in `notes/<task>.md`.

Small things (a quick question, a status check) get answered in the agent's own thread. Everything
else gets its own tab by default, so you can watch any piece of work.

It's one skill, one bash script, and four Markdown role files. State is plain text files you
can read and edit.

## Requirements

- [herdr](https://herdr.dev) (`curl -fsSL https://herdr.dev/install.sh | sh`)
- `git` and `jq`
- A coding agent herdr supports. The default is [Claude Code](https://claude.com/claude-code);
  Codex, Gemini, OpenCode and others work too (see [Using another agent](#using-another-agent)).
- macOS or Linux

## Setup

### 1. Install the skill

With the [skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add stevenmichaelthomas/numbrone -g
```

Or by hand:

```bash
git clone https://github.com/stevenmichaelthomas/numbrone.git
cp -r numbrone/skills/numbrone ~/.claude/skills/
```

### 2. Start herdr and your agent

```bash
herdr                                # launch or attach to herdr
herdr integration install claude     # once: lets herdr track Claude sessions
claude                               # start Claude Code in a herdr pane
```

### 3. Run the setup skill

In Claude Code, type:

```
/numbrone
```

The skill walks you through:

1. **Checks**: that herdr, git and jq are present.
2. **The numbrone home**: the folder where the chief lives (default `~/.numbrone`).
3. **Your projects**: give it paths, or a parent folder to pick git repositories from.
4. **Context**: it reads each project (README, manifests, layout, recent history) and writes
   `projects/<name>/context.md` so the chief can route requests accurately.
5. **Inputs** (optional): sources the chief or a manager should watch, such as Linear tickets
   or Sentry errors (see [Inputs](#inputs-you-or-anything-else)).
6. **Knowledge** (optional): a few questions about your goals and standards, to give the chief a head start.
7. **Agent settings**: which agent to use, and how many permission prompts you want (see below).
8. **Managers**: it opens one herdr workspace per project and starts a manager in each.

### 4. Start your chief of staff

Open a new herdr pane and run:

```bash
cd ~/.numbrone && claude
```

The chief reads `~/.numbrone/CLAUDE.md` and registers itself. From then on, just ask for things:

> Fix the flaky login test in the API, and add a dark mode toggle to the web application.

It splits that into two tasks, hands each to the right manager, and tells you. Managers spawn
workers, and the workers' tabs appear in each project's workspace in herdr. When something
needs you (a question, or work ready for review), the chief gets a `[numbrone] …` message and
passes it on. herdr also shows a notification.

Ask "what's going on?" at any time, or run `numbrone status` yourself. To use the
`numbrone` command from your own shell, add it to your PATH:

```bash
export PATH="$HOME/.numbrone/bin:$PATH"   # in ~/.zshrc or ~/.bashrc
```

The status looks like this:

```
TASK                     PROJECT          STATUS   WORKER   LAST UPDATE
fix-flaky-login          api              review   done     2026-11-08 16:01 [manager-api] review: Branch numbrone/fix-flaky-login …
dark-mode-toggle         web              working  working  2026-11-08 16:00 [worker-dark-mode-toggle] working: Add toggle to …

managers: manager-api=done manager-web=idle
```

## Inputs: you, or anything else

Work doesn't have to come from you. At each level, an agent can run a loop that watches a
source and turns new items into tasks:

- **The chief** watches sources that span projects, such as a Linear team, GitHub issues across
  an org, or a shared inbox. It routes each new item to the right project, just like a request from you.
- **A manager** watches sources its project owns, such as that service's Sentry errors, its
  Datadog monitors, or its CI failures. It triages each item and either handles it, spawns a
  worker, or asks the chief (and so you) when it needs a decision.

Sources are listed under a **Watch** heading: in `~/.numbrone/CLAUDE.md` for the chief, and in
`projects/<name>/context.md` for a manager. One line per source:

```markdown
## Watch
- Linear: new issues in team ENG labelled "agent", every 15 minutes
- Sentry: new unresolved issues in project "api", every 30 minutes
```

Agents start these loops when they launch (in Claude Code, with `/loop`), so watches come back
after a restart. You can also add one just by asking, for example "have the api manager keep an eye on
Sentry". The manager adds it to its `context.md` itself.

Agents read sources with whatever tools they have: MCP connectors (Linear, Sentry, Datadog, ...)
or CLIs such as `gh`. Connect those in your agent first. The item's ID goes in the task brief,
so the same ticket or error is never picked up twice.

## Teaching the chief

Not every conversation with the chief needs to be a task. You can also just give it context:

> For context: we're prioritizing enterprise customers this quarter, so anything touching SSO
> or audit logs comes first. And we decided last month to stop supporting IE.

The chief files it rather than delegating it, and confirms in one line where it went:

- **Context about your world** (goals, customers, people, past decisions, your standards) goes in
  `~/.numbrone/knowledge/`, one Markdown file per topic, with an `index.md` it reads at the start
  of every session.
- **Context about one project** goes in that project's `context.md`, and the chief passes it to
  that project's manager.
- **How you want the chief itself to work** goes under **Learned** (see below).

The more it knows, the better its work gets. It routes ambiguous requests correctly, writes briefs
that carry the "why" (managers only see the brief), and judges finished work against your goals
and standards before it reaches you. Paste documents, explain history, or correct a wrong assumption.
It keeps the files organized and up to date, and asks when it's missing something it needs.

## A chief that learns from you

The chief is meant to act like an extension of you: opinionated and quick to decide. It
recommends one option instead of listing five, makes routine calls itself, and only brings you
real decisions, with its recommendation attached. It also gives feedback, for example when a
request is vague or the same problem keeps happening.

It gets better by taking your feedback. Whenever you correct it or tell it how you like things
done, it adds one line to the **Learned** section of `~/.numbrone/CLAUDE.md`, which every future
session reads first. Early on (until it has about ten lessons), it asks you a short question
after each finished task to calibrate faster. You can edit **Learned** yourself at any time.

**Learned** is kept short on purpose: the chief merges overlapping lessons, replaces ones that
no longer hold, and moves project-specific detail into that project's `context.md`. The work log
keeps the full history of your feedback.

## Work log

`~/.numbrone/log/<date>.md` is the record of what the chief and its teams got done each day,
one Markdown file per day. It holds outcomes and decisions, not progress chatter:

- **Milestones**, written by the script as managers report them: queued, blocked, review, done, closed.
- **Outcomes**, written by the chief when a task finishes: what was delivered, how it went
  (rework, blocks, time taken), and a one-line lesson.
- **Decisions**: your answers to blocked questions, priority calls, work sent back. Managers
  log the decisions they make on your behalf, too.
- **Feedback and context** you gave the chief.

Workers' step-by-step progress stays in each task file (`numbrone status <task>`), so a day's log
stays short enough to read in a minute.

```markdown
# Work log 2026-11-08

- 15:59 [chief] add-name: queued for demo
- 16:01 [manager-demo] add-name: review: Branch numbrone/add-name (c880b98): greet.sh takes an
  optional name and falls back to "world". Tests pass.
- 16:02 [chief] outcome: add-name (demo): optional name argument, one pass, no rework. Lesson: none
- 16:20 [chief] decision: fix-login goes before new-navbar (launch Friday)
- 16:21 [chief] add-name: closed: accepted (branch numbrone/add-name kept)
- 17:05 [chief] q4-update-email: closed: changed: user cut the second paragraph; wanted it under 100 words
```

Ask the chief "what got done today?" and it answers by project from this file. To skim
outcomes across days yourself: `grep -h "outcome:" ~/.numbrone/log/*.md`.

Entries are only ever appended, so the log is an audit trail you can trust. It's also how the
setup gets better over time: see [Dreams](#dreams). You can also ask the chief for a retro any time
("how did this week go?"). It reads the log, tells you where work stalled or was redone and how
often, and proposes changes. It only makes them once you agree.

## Dreams

Once a day, the team looks back at its own work and gets better at it. The chief checks every
hour (`numbrone dream`, on its loop), and the first check after 3am on a day with new activity starts a
short-lived **dreamer** agent in a tab next to the chief. A chief that runs for weeks still dreams
daily, and a restart never dreams twice in one day.

The dreamer reads:

- **The numbers**, from `numbrone stats`: for each task, its kind of work, how long it took to
  deliver, how often it was sent back, how many questions workers asked, how often it reached you
  as a decision, and your verdict when it closed. Work on a branch also shows commits made after
  hand-off and whether it was merged.
- **The work log and task files**: your feedback, decisions, and every `sent back:` with what was wrong.
- **What you changed or rejected** after work was handed to you: the note on each `changed` or
  `dropped` verdict, and for code, the commits you added. That's the gap between what the team
  delivered and what you wanted.
- **The previous dream**, to check whether yesterday's changes moved the numbers.

Every role learns, each from its own file:

| Who | What gets better | How |
| --- | --- | --- |
| Workers | Conventions, checking their work, setup problems that keep coming back | `lessons.md` files, read before every task |
| Managers | What to catch in review, and their own drafts, answers and emails | The same `lessons.md` files, read at startup and checked in every review |
| Chief | Routing, briefs, what it lets through, what it asks you | Proposals for `CLAUDE.md` and `knowledge/` |
| Dreamer | Which lessons actually helped | Proposals for `roles/dream.md` |

Lessons live in two places, grouped by kind of work (`## Code`, `## Research`, `## Email`, ...):
`projects/<name>/lessons.md` for one project, and `lessons.md` in the numbrone home for what holds
everywhere, such as how you like research written up or the tone of your emails.

The dreamer **applies** lessons about how work gets done (the `lessons.md` files, fixes to a project's
`context.md`, merging duplicates in **Learned**). It **proposes** everything else, as exact
edits: changes to the chief's instructions, the role files, settings, or knowledge. The chief
brings you the headline and the proposals with its recommendation, and applies the ones you accept.

Each dream is saved to `log/dreams/<date>.md`. Read one, or run `numbrone stats` yourself:

```
TASK                     PROJECT        KIND       STATUS   DELIVERED  CLOSED    BACK  ASKED ESC  VERDICT   EDITED  MERGED
fix-flaky-login          api            code       closed   1h11m      3h40m     1     1     0    accepted  0       yes
churn-research           growth         research   closed   2h05m      1d2h      0     0     1    changed   -       -
q4-update-email          growth         email      review   12m        -         0     0     0    -         -       -

since 2026-11-09: 3 tasks, 3 delivered, 2 first time (66%), median time to deliver 1h11m
send-backs: 1, worker questions: 1, escalations to the user: 1
verdicts: 1 accepted, 1 changed, 0 dropped, 1 without one
by kind (first time / delivered): code 0/1, research 1/1, email 1/1
branches: 0 changed after hand-off, 1 merged
```

For this to work, two things go on the record:

- **Rework.** Managers and the chief send work back with `numbrone send-back <task> "<what to fix>"`,
  never a bare prompt, so it lands in the task file and the log.
- **Your verdict.** The chief closes every task once you've responded to it, whatever the kind of
  work: `accepted`, `changed` (with what you changed), or `dropped` (with why). It reads the verdict
  from what you say, and asks in a few words if it can't tell.

## Day to day

| You want to… | Do this |
| --- | --- |
| Give work | Tell the chief in plain language |
| See progress | Ask the chief, or `numbrone status` |
| See one task in full | `numbrone status <task>` (brief, status, full log) |
| Watch an agent work | Click its workspace and tab in herdr |
| Accept finished work | Tell the chief, and say what you changed if anything. For code, review the `numbrone/<task>` branch and merge it your usual way first |
| Add a project | Run `/numbrone` again, or `numbrone add` + write its `context.md` + `numbrone up <name>` |
| Restart managers | `numbrone up` (after a reboot, for example) |
| Review how things went | Ask the chief for a retro ("how did this week go?"), or read `log/dreams/` |
| See the numbers | `numbrone stats [days]` |
| Send work back | Tell the chief what's wrong (it runs `numbrone send-back`) |
| Dream now | `numbrone dream --force` |

Nobody merges, pushes, deploys, or sends, posts or publishes anything to other people (emails,
messages, shared documents) unless you say so. Asking for exactly that, such as "send the update
to the team", counts. `numbrone close` refuses to remove a worktree that has uncommitted changes,
and it always keeps the branch.

## How it works

Everything is plain files in the numbrone home folder:

```
~/.numbrone/
├── CLAUDE.md               the chief's instructions (AGENTS.md links to it)
├── settings.sh           which agent to run, with which arguments
├── projects.json           project index: name → path + one-line summary
├── projects/<name>/
│   ├── context.md          what the project is, its commands, which requests belong to it
│   └── lessons.md          what the dreams learned about working here (managers and workers read it)
├── lessons.md              what the dreams learned that holds across projects, by kind of work
├── tasks/<task>.md         one file per task: brief, status, append-only log
├── notes/<task>.md         what workers deliver that isn't a change to a project: findings, drafts
├── log/<date>.md           the daily work log (see above)
├── log/dreams/<date>.md    each day's dream: numbers, patterns, what changed, proposals
├── knowledge/              what the chief knows about your world, by topic (index.md lists them)
├── roles/
│   ├── manager.md          instructions every manager reads at startup
│   ├── worker.md           instructions every worker reads at startup
│   └── dream.md            instructions for the daily dreamer
└── bin/numbrone            the script everyone uses
```

A task file looks like this:

```markdown
# fix-flaky-login
project: api
kind: code
status: review
branch: numbrone/fix-flaky-login
start: 4f1c2e9
worktree: /code/api/.numbrone/worktrees/fix-flaky-login
created: 2026-11-08 15:58

## Brief
The login test fails about 1 in 5 runs on CI. Find out why and fix it ...

## Log
- 2026-11-08 15:58 [chief] queued
- 2026-11-08 15:59 [manager-api] spawned worker-fix-flaky-login on numbrone/fix-flaky-login
- 2026-11-08 16:04 [worker-fix-flaky-login] review: Race between session write and redirect ...
- 2026-11-08 16:05 [manager-api] review: Branch numbrone/fix-flaky-login: awaits the session write ...
```

The `numbrone` script is a thin wrapper around the herdr CLI:

| Command | What it does in herdr / git |
| --- | --- |
| `numbrone up [project]` | `herdr workspace create --label numbrone/<project>`, then `herdr agent start manager-<project>` |
| `numbrone delegate <project> <task> "<brief>" [kind]` | Writes `tasks/<task>.md` (kind is one word: `code`, `research`, `email`, ...), then `herdr agent prompt manager-<project>` |
| `numbrone spawn <task>` | `git worktree add <project>/.numbrone/worktrees/<task> -b numbrone/<task>`, `herdr tab create`, `herdr agent start worker-<task>` |
| `numbrone report <task> <status> "<note>"` | Updates the task file; on `blocked`/`review`/`done` it messages one level up (worker → manager → chief) and shows a herdr notification |
| `numbrone status [task]` | Reads task files plus live agent states (`herdr agent get`) |
| `numbrone send-back <task> "<what to fix>"` | Records the rework in the task file and log, then `herdr agent prompt` one level down (manager → worker, chief → manager) |
| `numbrone close <task> <accepted\|changed\|dropped> ["<note>"]` | Records your verdict; for work with a worker, `herdr tab close` it and `git worktree remove` (keeps the branch) |
| `numbrone stats [days]` | Per-task numbers and verdicts from the task files, plus `git log` / `git merge-base` for branches |
| `numbrone dream [--force]` | Once a day: `herdr tab create` next to the chief, then `herdr agent start dreamer` |
| `numbrone chief` | `herdr agent rename <this pane> chief`, so reports can reach it |
| `numbrone log ["<note>" \| <date>]` | Appends a note to today's work log, or prints a day's log |

Worktrees go in `<project>/.numbrone/worktrees/`, listed in the repository's local `.git/info/exclude`. They
never appear in `git status` or get committed, and agents started there inherit the folder
trust you already gave the project.

Run `numbrone` with no arguments for the full command list.

## Configuration

`~/.numbrone/settings.sh`:

```bash
AGENT_KIND=claude       # any `herdr agent start --kind` value
AGENT_ARGUMENTS=""      # extra arguments for the agent
BASE_BRANCH=""          # branch new worktrees start from ("" = project's current HEAD)
PING_CHIEF=1            # 0 = don't type reports into the chief; just log and notify
DREAM_HOUR=3            # the daily dream runs at the first hourly check after this hour
```

**Permission prompts.** Workers stop at every permission prompt and show as `blocked` in herdr
until you answer. That's the safe default. To cut down on prompts with Claude Code, for example:

```bash
AGENT_ARGUMENTS="--permission-mode acceptEdits"
```

Pick what you're comfortable with. Agents run on your machine with your access.

**Folder trust.** The first time Claude Code opens a project, it asks whether you trust the
folder. If a manager hits this, `numbrone` shows a notification and waits up to 5 minutes for you to
answer in herdr. Workers inherit their project's trust, so they won't ask again.

## Using another agent

Set `AGENT_KIND` in `settings.sh` (for example `codex`, `gemini`, `opencode`) and install its herdr
integration (`herdr integration install codex`). The role files are plain Markdown and the numbrone
home folder has an `AGENTS.md` symlink, so any agent that reads `AGENTS.md` can be the chief. You
can install the skill for that agent with `npx skills add … -a <agent>`, or run
`bash skills/numbrone/scripts/numbrone init ~/.numbrone` yourself and follow `skills/numbrone/SKILL.md` by hand.

## Customizing

- **Change how roles behave**: edit `~/.numbrone/roles/manager.md` or `worker.md`.
- **Upgrade**: pull the repository and run `numbrone init ~/.numbrone` again. It updates `bin/numbrone`
  and never overwrites files you may have edited (`CLAUDE.md`, `roles/`, `settings.sh`,
  `projects.json`). To take a newer template, delete that file and run `init` again. If your
  `CLAUDE.md` or `roles/manager.md` predates dreams, `init` tells you the line to add.
- **Change how the chief behaves**: edit `~/.numbrone/CLAUDE.md` directly.
- **Better routing**: improve the "Route here" and "Not here" sections in each project's `context.md`.

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| `numbrone: no numbrone home found` | Run `~/.numbrone/bin/numbrone …` (not a copy elsewhere), or set `NUMBRONE_HOME=~/.numbrone` |
| A manager or worker shows `blocked` | It's waiting for an answer. Click its tab in herdr, or `herdr agent read <name> --source recent-unwrapped --lines 80` |
| Reports don't reach the chief | Run `numbrone chief` in the chief's session (it registers the name `chief`) |
| `manager-x already running` but it seems stuck | `herdr agent read manager-x …` to see its screen; close its tab in herdr and run `numbrone up x` |
| Managers gone after a reboot | `numbrone up` |
| Agent states look wrong | `herdr integration status`; `herdr agent explain <name>` |

## Uninstall

```bash
rm -rf ~/.claude/skills/numbrone ~/.numbrone
```

Then close the `numbrone/*` workspaces in herdr. Task branches (`numbrone/*`) stay in your repositories until you
delete them; any leftover worktrees can be removed with `git worktree remove <path>`.

## License

MIT
