---
name: codex
description: "Hand a task to a headless Codex session from Claude, review its PR and resume the session until it is ready."
---

# Run a Codex Session

The user asked for a Codex session or agent on a task. Codex writes the code and you review it: launch it headless with `codex exec`, review what it delivers, and resume the same session whenever something is left to change, until its PR is ready. Don't make those changes yourself. A running Codex session counts as a subagent toward any limit your instructions put on running them in parallel.

Every run sets its model and reasoning effort explicitly, so the user's own Codex config never decides them. Unless the user names others, use `medium` effort and the newest listed Sol model, read once per conversation from the model list Codex keeps locally (no network call):

```bash
python3 -c 'import json,os,re;m=json.load(open(os.path.expanduser("~/.codex/models_cache.json")))["models"];v=[(tuple(map(int,x.group(1).split("."))),s) for s in (e["slug"] for e in m if e.get("visibility")=="list") if (x:=re.fullmatch(r"gpt-([\d.]+)-sol",s))];print(max(v)[1])'
```

It prints a slug such as `gpt-6.1-sol`. If the file is missing or the command fails, say so and ask the user which model to use rather than guessing one.

Every run also has a speed, SPEED. Use the standard speed unless the user asks for another: leave `-c service_tier=SPEED` out of the commands below. A faster speed is the id of one of the model's service tiers, such as `priority`, the tier Codex calls Fast, which the model list rates at 1.5x to 2x the standard speed for more usage. Codex has no `--fast` flag; a tier goes in as `-c service_tier=SPEED`. The same model list says which tiers the model offers and what each claims:

```bash
python3 -c 'import json,os,sys;m={e["slug"]:e for e in json.load(open(os.path.expanduser("~/.codex/models_cache.json")))["models"]};[print(t["id"],"-",t["name"],"-",t.get("description","")) for t in m[sys.argv[1]].get("service_tiers",[])]' MODEL
```

On 2026-10-10 `gpt-6.1-sol` listed `priority` ("Fast", "2x speed, increased usage") and `ultrafast`; `gpt-6-sol` and `gpt-5.6-sol` listed `priority` at 1.5x. Measured end to end on `gpt-6.1-sol`, eight alternating runs of one 600-word answer took 50.4 s (median) at `priority` against 67.7 s standard, 1.34x: Codex's startup is not sped up, and neither is time a session spends waiting on CI. When the user asks for a tier the model does not list, say so and use the standard speed. A tier the model does not offer is not an error: Codex prints `Configured service tier … is not advertised as supported … and will be omitted from requests` in the log and runs at the standard speed, so read the log's opening lines once after a launch with a tier.

Reuse the same MODEL, EFFORT and SPEED for every run and resume of a session, and name them when you report its launch. A session that is already running keeps its speed: to change it, stop the run and resume the session with the new tier, telling it to pick up where it stopped.

For an opinion only, with no code change and no follow-up, skip the steps and run `codex exec -m MODEL -c model_reasoning_effort=EFFORT -c service_tier=SPEED --ephemeral -s read-only -C DIR -o ANSWER - < PROMPT > LOG 2>&1`.

## 1. Write the brief

Save it as `<working-dir>/.plans/YYYY-MM-DD-<topic>-codex-brief.md`, kept out of git as [design-brainstorming](../design-brainstorming/SKILL.md) describes, so the user can open it. Codex follows the same global rules and has the same skills, so the brief carries only the task: the problem with its evidence, what done looks like, what is out of scope. It also tells Codex to:

- work in the worktree it starts in and on the branch it finds there, making no other worktree: step 2 cuts both from the latest `origin/<default>` for it;
- stop once its non-draft PR is open, with no babysit-pr and no waiting for a review, because you are the reviewer and start only when its run exits;
- not merge or deploy;
- end its final message with a report block: PR URL, one-line summary, each command it ran with its result, anything the reviewer or the user must know or answer.

## 2. Launch

Make the session's worktree yourself, where your instructions put worktrees, on a new branch cut from the latest `origin/<default>`. Name the worktree `codex-<topic>` after the brief, and the branch the way your instructions name branches, `codex/<topic>` when they don't say.

```bash
git -C MAIN_CHECKOUT fetch origin
git -C MAIN_CHECKOUT worktree add --no-track -b BRANCH WORKTREE origin/DEFAULT
```

A repository's own command for making a worktree comes first: when its instructions give one, run it with the branch name in place of these two commands, and use the path it prints as the worktree.

Then start Codex in it, in a background Bash with `timeout: 7200000`. Two hours is the maximum, and the default stops it after 30 minutes.

```bash
codex exec -m MODEL -c model_reasoning_effort=EFFORT -c service_tier=SPEED -C WORKTREE --approve-for-me --color never -o SCRATCHPAD/NAME-last.md - < BRIEF > SCRATCHPAD/NAME.log 2>&1
```

- Never pass `--worktree`. Codex would make a worktree of its own at `<its worktree root>/<4 hex digits>/<repo>`, detached at the main checkout's HEAD: outside the place your instructions and the project's tooling expect a worktree, and on a base that can be behind.
- `--approve-for-me` keeps the `workspace-write` sandbox and sends approval requests to Codex's automatic reviewer, since nobody is there to answer a prompt. Never pass `--dangerously-bypass-approvals-and-sandbox`.
- Never pass `--ephemeral`: it keeps no session to resume.

The log opens with `stale rollout path` errors, which are noise, and within seconds a header with the `session id:` line a resume needs. Read it and tell the user the session id, the worktree and the brief.

**When auto mode refuses.** The first launch in a conversation is usually refused as `[Create Unsafe Agents]` when the user's request said no more than "run codex". Whenever a launch or a resume is refused, don't try variants: show the user the exact command, say what `--approve-for-me` does, and re-issue it unchanged on their go. If it is refused again, the command is theirs to run.

## 3. Review

You are re-invoked when the run exits.

1. Read the `-o` file. A run that was cut off leaves none, so read the log's tail instead.
2. Attach the PR it opened to your session, once, when your runtime can show a PR beside the conversation, as the Claude desktop app does: the user follows its checks and review state there. Attach it and nothing more: no watch goes on with it, as the next step says.
3. Review the PR with [review-pr](../review-pr/SKILL.md), against the brief as well as the diff. You are this PR's external AI reviewer, so keep [babysit-pr](../babysit-pr/SKILL.md) and any Auto-fix watch off for it: both would have you fix what you review.
4. Read the PR's checks and mergeability yourself afterwards: review-pr leaves CI out.
5. Pass the user any question Codex left that the brief doesn't answer.

## 4. Resume

Resume the session when anything is left for it to do: an open review thread, a failed or still-running check, a conflict with the base, or a run that stopped short of its PR. Use the same background Bash and timeout.

```bash
codex exec -m MODEL -c model_reasoning_effort=EFFORT -c service_tier=SPEED -C WORKTREE --approve-for-me --color never -o SCRATCHPAD/NAME-last-2.md resume SESSION_ID - < FOLLOW_UP > SCRATCHPAD/NAME-2.log 2>&1
```

- Always pass `-C` with the session's worktree. A resumed run works in the directory it is given (the current one by default), not where the session began.
- Keep the flags before the word `resume`, which takes neither `-C` nor `--approve-for-me` after it.
- Pass the session's MODEL, EFFORT and SPEED again, so a resume never falls back to the user's config.

The follow-up says what is left: for one [address-pr](../address-pr/SKILL.md) pass (fix or argue, reply without resolving, push), the links to each open thread and failed check and the text of any finding that has no thread; or what a run that stopped short still owes. It always ends by asking Codex, once that is done, to wait for the checks on the head it leaves and to report them with what changed. Re-review that head when the run exits.

## 5. Finish

Done is the four conditions from babysit-pr's *When to stop*, with your review as the approval. Tell the user then; merging and deploying stay their call. When a resume leaves an item as it was (a thread you still hold open, a check still red, a run cut off or blocked again), bring that item to the user rather than resuming for it again.
