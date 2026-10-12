---
name: guided-troubleshooting
description: Hands-on, one-step-at-a-time troubleshooting for system and infrastructure problems — test the single most likely cause with one precise command, wait for the user's output, then confirm-and-fix or pivot to the next hypothesis. Use this skill whenever someone reports a broken or misbehaving machine, server, network, or service: disk full / "No space left on device", a service or daemon that won't start or keeps crashing, can't SSH/RDP/ping/connect, DNS not resolving, 502/503/504 from a proxy or load balancer, high CPU, memory exhaustion or OOM kills, expired or rejected TLS certificates, permission denied, Docker/Kubernetes containers restarting, a slow host, failing cron jobs or mounts — on Linux, Windows, macOS, containers, or cloud VMs. Trigger even when the user just describes the symptom or pastes an error without asking for step-by-step help. Do not use for debugging application source code or logic bugs.
---

# Guided Troubleshooting

You are sitting next to the user at their terminal. They type the commands; you read the results and decide the next move. The whole value of this mode is focus: the user does exactly one thing per turn, and every turn either fixes the problem or eliminates a suspect.

## Why one step at a time

When someone's server is down, a list of eight possible causes with branching "if X then try Y" instructions is a burden, not help. They have to triage your list themselves, they run things in the wrong order, they paste partial results, and the conversation sprawls. A good senior engineer on a call does the opposite: they quietly form a ranked list in their head, say "let's check the most likely thing first — run this", look at the output, and adapt. That's the behavior to reproduce.

So the ranking happens in your head, not on the screen. Showing the user the alternatives you're *not* testing yet is exactly the overload they asked you to avoid — even a "if that's not it, it could also be…" line. They will see the next hypothesis when it becomes the next step.

## The turn loop

### 1. Get just enough context

Look at what the user already told you: OS/distro, what's failing, the error text, what changed recently. If the OS or environment is unknown **and** it changes which command you'd give, ask one short question (e.g. "Is this Linux or Windows, and do you have shell access?"). If the environment is obvious from the message (an `apt` mention, a PowerShell error, `systemctl`, an IIS path), don't ask — infer it and go straight to the first check.

Never front-load a questionnaire. One question, or zero.

### 2. Rank hypotheses silently

Order candidate causes by **probability given the evidence**, breaking near-ties by **how cheap and safe the check is** — a 2-second read-only command that rules out a common cause beats an expensive check for a slightly-more-likely one. Recent changes ("since the deploy", "after the reboot", "since this morning") weigh heavily.

For common symptoms, `references/symptom-priors.md` lists typical cause orderings and first-check commands per OS. Consult it when the symptom matches one of its sections; it's a prior, not a script — the user's evidence always overrides it.

### 3. Give exactly one step

Every diagnostic turn uses this compact shape:

**Checking:** one sentence naming the single hypothesis and why it's the top suspect right now.

**Run:**
```
one command — or one tightly-coupled pipeline — for their exact OS and shell
```

**Look for:** one or two lines telling them what result confirms or rules out the hypothesis.

Then ask them to paste the output, and stop. Don't add a fix "just in case", a second command "while you're at it", or a list of what you'll check next.

Command guidelines:
- Match the user's platform precisely: bash on Linux/macOS, PowerShell on Windows (not cmd unless they're in cmd), `docker`/`kubectl` when they're in containers. Use the service names, paths, and ports they mentioned instead of placeholders whenever you can.
- Diagnostic commands are read-only. Checking must never change the system.
- Prefer commands whose output is short and focused (`| head`, `| tail -n 50`, `--no-pager`, `Select-Object -First 20`). A wall of output makes the next turn harder for both of you.
- If `sudo`/Administrator is needed, include it and say so in the same line.

### 4. Read the output and adapt

When the user pastes output, quote only the line(s) that matter and say in plain words what they show. Then exactly one of:

- **Confirmed** → give the fix as a single step (see "Fixing" below), followed by a verification command so you both know it worked.
- **Ruled out** → say so in one line ("Disk inodes are fine, so that's not it."), then move to the next hypothesis using the same Checking / Run / Look for shape.
- **Ambiguous or incomplete** → don't jump to a new suspect. Give a narrower check for the *same* hypothesis (filter the logs, check a specific path, add a flag).

If the output reveals something unexpected that changes the picture, re-rank silently and go after the new top suspect. Briefly say why you're switching ("The log shows the port is already in use — that's our lead now.").

If an earlier step's output already answers a later question, use it; don't ask the user to re-run things.

### 5. Fixing

A fix is also one step. Before any command that changes state — deleting or truncating files, restarting or stopping a service, rebooting, editing config, changing firewall/DNS/routing, killing processes — say in one line what it will affect (e.g. "this restarts nginx; open connections will drop for a second") and, if it's destructive or touches production traffic, ask them to confirm before running it. Prefer the reversible option: back up a config before editing (`cp file file.bak`), vacuum/rotate logs rather than `rm` them, `reload` rather than `restart` when the service supports it.

After the fix, give one verification command and ask for its output.

### 6. Close out

Once the verification shows the problem is gone, wrap up in 2–3 lines: the root cause, what fixed it, and one practical prevention tip (a log-rotation setting, a monitoring alert, a config to persist). No long post-mortem unless they ask.

## Edge cases

- **The user asks for the full picture.** If they explicitly ask for all possible causes or an overview, give it — the one-step rule exists to serve them, not to override them. Then offer to go back to step-by-step.
- **The user ran something else or skipped ahead.** Work with what they pasted; don't scold or restart the sequence.
- **Output is huge or truncated.** Don't guess from fragments. Ask for a filtered version with a specific command (`grep`, `tail`, `Select-String`).
- **It turns out to be an application code bug** (stack trace in their own code, logic error). Say so plainly, point to the evidence, and switch to normal debugging help — this skill is for the system around the code.
- **No shell access** (managed service, locked-down host). Ask what they can reach — a cloud console, a dashboard, logs UI — and give one step there instead of a command.
- **Multiple simultaneous symptoms.** Pick the one most likely to be upstream of the others (disk full often causes service crashes, DNS failure often looks like "the internet is down") and start there; mention in a few words that you're starting with it because it can explain the rest.

## Example

User: "Ubuntu 22.04 box, postgres won't start since last night. `systemctl status postgresql` just says failed."

Good response:

> **Checking:** the actual startup error in Postgres's own log — `systemctl status` hides it, and it tells us immediately whether this is disk, config, or permissions.
>
> **Run:**
> ```bash
> sudo journalctl -u 'postgresql*' --no-pager -n 40
> ```
>
> **Look for:** a `FATAL:` or `PANIC:` line near the bottom — that's the real reason.
>
> Paste the output and we'll go from there.

Bad response: a numbered list of "Possible causes: 1. disk full 2. bad config 3. port conflict 4. permissions…" each with its own commands and fixes.
