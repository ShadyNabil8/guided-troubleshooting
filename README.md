# guided-troubleshooting

A Claude skill for fixing system and infrastructure problems one step at a time. Each reply tests the single most likely cause with one command, then waits for your output.

## Example

**You:**
> Our Ubuntu 22.04 web server throws "No space left on device". `df -h` shows `/` at 100%.

**Claude:**
> **Checking:** which directories on `/` are eating the space. Runaway logs are the usual culprit on a web server.
>
> **Run:**
> ```bash
> sudo du -xh / --max-depth=2 2>/dev/null | sort -rh | head -20
> ```
>
> **Look for:** one or two paths much bigger than the rest.
>
> Paste the output and we'll go from there.

**You:**
> ```
> 58G   /
> 44G   /var
> 42G   /var/log
> 41G   /var/log/journal
> ```

**Claude:**
> That's our culprit: `41G /var/log/journal`. The systemd journal is using most of the disk.
>
> **Fix:** this permanently deletes older journal entries. Nothing restarts. Reply "go" or run it when ready:
> ```bash
> sudo journalctl --vacuum-size=1G
> ```
>
> **Verify:**
> ```bash
> df -h / && sudo journalctl --disk-usage
> ```
>
> Paste the output.

## What you won't get

Without the skill, the same first message gets a long reply: a table of ten possible causes, eight command blocks, and cleanup commands run before anything was diagnosed.

## More examples it handles

| You say | First step it gives |
|---|---|
| "nginx returns 502, Node API runs as `api.service` on port 3000" | `sudo systemctl status api.service --no-pager -l` |
| "Can't RDP into Windows Server 2022, ping works" | PowerShell check that `TermService` is running and port 3389 is listening |
| "Pod keeps restarting" | `kubectl logs <pod> --previous --tail=50` |

It works on Linux, Windows, macOS, containers, and cloud VMs. It's not meant for debugging application code.

## Install

```bash
git clone https://github.com/ShadyNabil8/guided-troubleshooting ~/.claude/skills/guided-troubleshooting
```

Claude Code loads it in new sessions. It triggers on its own when you describe a broken server, service, or network.
