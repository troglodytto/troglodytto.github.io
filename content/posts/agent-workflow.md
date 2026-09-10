+++
date = '2026-09-10T17:20:00+05:30'
draft = false
title = 'Building an agent workflow from nothing'
slug = 'agent-workflow'
+++

I had 69 skills installed. Before I typed a single character into a session, roughly 9,600 tokens of context were already spent: skill descriptions, plugin hooks, standing instructions. None of it was doing anything for me yet. All of it was being paid for.

This post is about tearing that down to zero and rebuilding it, and mostly about the reasoning at each step, because the specific files matter far less than why they ended up shaped that way.

## Measuring before cutting

The first useful thing was finding out what a skill actually costs.

A skill on disk costs nothing. What costs is its `description` field, because that is what gets listed to the model on every single turn so it knows the skill exists. Sixty of my skills were model-visible and their descriptions came to about 4,700 tokens. Eight more were marked `disable-model-invocation`, which hides them from the model while keeping them typable as slash commands. Those eight were free.

That distinction reframed the whole exercise. The question was never "do I use this skill", it was "is this skill worth its description on every turn, including the several thousand turns where it is irrelevant".

Most were not. 107 skills across eight scattered directories. My main store held 63 of them, and when I traced their origins through a lock file, exactly three had no upstream. Every other one was a vendored copy of somebody else's repository, six of them in total.

## The wipe

I consolidated everything into one directory, deduplicated it, archived it, and then deleted the live set entirely. Skills, hooks, plugins, standing instructions. Startup context went from 9,583 tokens to zero.

That sounds dramatic and it was not. Everything was backed up twice. The point of going to zero was that curation had already failed, twice. The first pass took the store from 63 to 34. The second took it to 21. Both times the number came down and the set still felt arbitrary, because every decision was being made against its neighbors rather than against a need. Deciding what to add to an empty directory is a different question. Six weeks of accumulated installs came back as zero. The skills directory is still empty today. What replaced all of it was a rules file, one command line tool, and three sub-agents.

## Where instructions actually go

The first thing to rebuild was standing instructions, and here I learned something that cost me an hour of wrong assumptions.

I wanted the file to be called `AGENTS.md`, the emerging cross-agent convention, so the same rules could serve Claude Code, Codex, and Gemini. I wrote it, put it at `~/.claude/AGENTS.md`, and started filling it in.

Then I tested it. I put a marker string in the file and started a fresh session asking for it.

Nothing. The session had never seen the file.

The same marker in `~/.claude/CLAUDE.md` came back verbatim. At user scope, Claude Code reads `CLAUDE.md` and only `CLAUDE.md`. A global `AGENTS.md` is silently ignored, which is the worst possible failure mode: no error, no warning, just instructions that never load.

The fix was a symlink. The real file lives at `~/.agents/AGENTS.md` and `~/.claude/CLAUDE.md` points at it, along with `~/.codex/AGENTS.md` and `~/.gemini/GEMINI.md`. One file, four names, and I verified the symlink loads too rather than assuming.

> 💡 **Test the plumbing, not the documentation**
>
> Every claim in this post that could be tested was tested. The ones that could not be, I have said so. There is a large difference between "the docs say this works" and "I put a marker in a file and watched it come back".

## Instructions beat hooks, and it is not close

There is a second way to inject standing context: a `SessionStart` hook that prints text into the model's context at launch. I built one, and it worked, and I later deleted it.

The reason is framing. When `CLAUDE.md` loads, the harness wraps it in language of its own: *these instructions OVERRIDE any default behavior and you MUST follow them exactly as written*, along with a note saying whose instructions they are. That sentence is not written by me. It comes from Claude Code.

Hook output gets a flat label instead: `SessionStart hook additional context:` and then the payload. No override clause. No attribution. It reads as context rather than instruction.

A hook can borrow authority by shouting in its own payload, which is exactly what some plugins do by wrapping themselves in `<EXTREMELY_IMPORTANT>` tags. But that is the hook author shouting, not the system.

So the hard rules went in the file that inherits the override clause, and the hook that duplicated them got deleted. That change alone gave back about 5,100 tokens per session, because the hook had been injecting the entire ruleset a second time.

## Writing rules for a tool, not a character

The first draft of the voice section was built from my own writing style: vary sentence length, use contractions, take a stance, start sentences with "but" when it reads better.

That was wrong, and it took me a while to see why. Those rules exist to make *my* writing sound like a person. An agent answering questions is not writing under my name. It is retrieving, verifying, and reporting. Personality there is noise. It makes output harder to scan, and worse, it turns confidence and uncertainty into performance instead of fact.

So the section got rewritten around three ideas.

**No anthropomorphizing.** No claimed feelings, no "happy to", no "good catch", no apology, no praise of the question. The word "I" is permitted only as the actor of a verifiable action: *I ran X, it returned Y*. Never as the subject of a mental state.

**Determinism.** Same question, same shape of answer. Style carries no information, so it should not vary. One name for one thing, never rotating between check, verify, validate, and confirm for a single action.

**Uncertainty as a fact about evidence.** State what was verified and how. State what was not verified, before the conclusion that rests on it. An inconclusive test stays inconclusive rather than being rounded up into a result.

Underneath that sits a surprising ingredient: [ASD-STE100](https://asd-ste100.org), the controlled English standard written for aircraft maintenance manuals. It gives you active voice, simple tenses only, one instruction per sentence, a cap on sentence length, no phrasal verbs, and a rule that a multi-word noun may not exceed three words. It was designed so a technician whose first language is not English cannot misread a procedure that might kill someone. It turns out to be an excellent specification for a machine that answers questions.

## The rules that stop things going wrong

The engineering half of the file is shorter and more boring, and it is the half that prevents actual damage.

**Read before write.** Read the real documentation, not training recall, and check the version this project pins. Read the code around the task before editing it. Look at the target before overwriting or deleting it.

**Go-ahead.** Do not start executing until told to. A request to design something is not a request to build it. This one was added after the agent twice started implementing while I was still mid-explanation, which is a specific and infuriating failure mode: you are three sentences into describing a mechanism and the thing has already written a version of it.

The rule has two halves and the second matters as much as the first: once the go-ahead is given, carry the work to completion without checking back. An agent that asks permission every four turns is its own kind of broken.

**Git is read only.** Status, log, diff, show, blame. Nothing that writes. Not commit, not push, not even `git add`. At a natural commit point it says so and supplies the message, and I run it. This is absolute and written with no exceptions clause, because a rule with an escape hatch gets escaped.

**Environment and secrets.** Never set or export a variable. Never print one. Never run bare `env` or `printenv`, which dump everything at once. Reading is permitted only through a masked check that answers a question without revealing a value:

```bash
[ -n "${VAR:-}" ] && echo "VAR: set" || echo "VAR: unset"
printenv VAR | wc -c          # length only
```

The most detail that may ever be reported is the shape: *set, 40 characters, starts sk-*.

## State does not belong in prose

Here is a failure everyone using these tools will recognize. You are eight turns into a task. Every reply restates the full checklist so nothing is forgotten. Item three changed, so the other six get repeated to give it context. The list gets restated once per turn, forever, and the context window fills with the same seven lines.

The list is state. Prose is a terrible place to keep state.

So state moved into a small command line tool. Tasks live in a JSON file, persist across sessions, and a session binds to one. That last part matters more than it sounds:

```
todo task current
```

If the session already has a binding, that is the entire startup procedure. No searching, no asking, no re-establishing what we were doing. The binding is cached in the store keyed by session id, so it survives a restart, and I confirmed it survives `--resume` too.

Only a genuinely new session falls through to searching for a related task, proposing one, and asking before binding. The tool cannot know whether new work is the same work, so that one question is left to a human.

Tasks carry a type, chosen without asking, and can sit under a parent that acts as a folder with no clock of its own. The parent's total and the calendar range it spans are derived from its children rather than tracked separately.

![Tasks grouped under a parent, with type, item counts and elapsed time](/todo-task-ls.png)

## Time, and the problem of walking away

Tasks track time. Binding starts a period, switching to another task pauses the first one automatically, and time accumulates per task across every session that touches it.

The interesting problem is that an agent has no idea when you leave. You bind a task at nine, get pulled into something else at half past, and come back after lunch. Naively the task has been running for four hours.

The fix is a heartbeat. A hook fires on every message you send and stamps the task with the current time. Credit then accrues only while heartbeats are arriving, and it stops twenty minutes after the last one whether or not another ever comes.

Two properties make this work.

**The cap applies on read, not only on write.** If you walk away and no heartbeat ever arrives, nothing needs to write to the store for the number to stay honest. Asking how long a task has run computes the open period as ending at `min(now, last_beat + 20min)`. The total simply stops growing.

**A beat after a long gap splits the period.** It closes the old one at the twenty minute boundary, opens a fresh one at the present moment, and the interval between is never counted. Every period is kept. The total is derived by summing them, never stored, so there is no accumulator to drift out of sync with the timestamps that produced it.

Every period is kept, so the breakdown stays inspectable, and items can be listed across every open task at once.

![Time periods on one task beside open items across all tasks](/todo-time-and-items.png)

There is one case where the tool refuses to decide. If you deliberately pause or close a task after a long silence, it does not guess whether that silence was work. It prints the gap, exits without writing, and hands you the choice:

![The tool refusing to close a period across a gap](/todo-credit-prompt.png)

Silently truncating three hours is wrong. Silently crediting three hours is also wrong. The only correct move is to ask.

## Plugins, and why there is no operation log

At some point it became obvious that the tool was turning into a general version of a task tracker I had written before, one that pushed time entries to a remote service. Rather than merge the two, the tool became the product and the sync became a plugin.

The design question was how a plugin learns what changed. The instinct is an event stream: fire `task.created`, `span.closed`, and so on, and let plugins subscribe.

That instinct is wrong here, for one reason. If the sync target is unreachable when an event fires, that event is gone. Now you need a retry queue, delivery guarantees, and dead letter handling, all living inside a task tracking script.

The alternative is to make sync a function of state rather than of who was listening. Every task carries a monotonic revision number, bumped on any write. A plugin asks for everything above its own cursor:

```
todo export --since 41
```

Nothing is ever lost, because nothing was ever delivered. Retries are free. It is idempotent by construction. And folding comes free: five edits to one task between syncs export as a single row containing the final state, because operations are never what gets exported. There is no log to compact and no "increment, increment" to resolve into "plus two".

The only thing a state delta cannot express is a deletion, since absence is not observable. That needs tombstones, and it is the one place a log-shaped record survives.

Plugins are woken in the background after any write with a signal carrying no payload. They reconcile from the store themselves. They never write back.

And the rule that ties it together: the agent's entire surface is the `todo` command. It never invokes a plugin, never reads a plugin's config, never learns one exists. Which incidentally solves the credentials problem for free. The integration's token lives with the integration, out of band, and the rule that says the agent may never read credentials stays intact because there is no path to them.

## Sub-agents, tiered

The last piece was delegation. A sub-agent's tool output never enters the main context, which makes it the actual mechanism for preserving context rather than a slogan about it.

Three of them, and the tiering is the point:

| Agent | Model | Job |
| ----- | ----- | --- |
| `rtfm` | haiku | Read current official docs, return the answer with its source and the version checked |
| `verify` | haiku | Run the repo's own test command, return a verdict and the first real failure |
| `implement` | sonnet | Write code for one settled task, staying inside the files it was given |

Orchestration stays on the largest model, because holding the plan and deciding phases is the part that needs it. Writing code against an already-decided task does not. Retrieval and verification need it least of all.

Each returns a compressed, labelled block rather than prose, because the caller pays for every line that comes back. `verify` returns a verdict, the exact command, counts, and at most twenty lines around the failure. It is forbidden from fixing anything, because a sub-agent that both diagnoses and repairs gives you no way to check its work.

## What it costs now

Startup context went from 9,583 tokens to about 4,400, and the half that remains is doing work rather than advertising capability. Reads and searches are auto-approved through a permission allowlist, writes go to the classifier, and everything touching credentials is denied outright.

A status line reads the bound task straight out of the store:

![The status line reading the bound task out of the store](/todo-statusline.png)

None of this is finished. There is no sync plugin yet, several repositories still lack the per-project instructions that other rules depend on, and I am fairly sure the next irritating thing will produce another rule.

But the shape feels right, and I think the shape is the transferable part. Not the files. Instructions go where the harness gives them weight, state goes in a tool rather than in prose, totals are derived rather than stored, sync pulls rather than pushes, and the agent gets exactly one surface to touch.

Everything else is just a config file you will delete in six months anyway.
