# Symptom Priors

Typical cause orderings for common symptoms, with a good *first* read-only check per platform. These are priors for your private ranking — never paste the list to the user. The user's evidence (error text, recent changes, what they've already checked) always overrides the ordering here.

## Contents
1. Disk full / "No space left on device"
2. Service won't start / keeps crashing
3. Can't connect to a host or port (SSH, RDP, HTTP, DB)
4. DNS not resolving
5. 502 / 503 / 504 from a proxy or load balancer
6. High CPU
7. Memory exhaustion / OOM kills
8. TLS / certificate errors
9. Permission denied
10. Slow host
11. Container restarting / CrashLoopBackOff
12. Scheduled job (cron / Task Scheduler) not running

---

## 1. Disk full / "No space left on device"
Likely causes, in order:
1. Logs grown out of control (`/var/log`, journald, app logs, Docker container logs)
2. Docker images/volumes/build cache
3. Deleted-but-still-open files held by a running process (df full, du doesn't add up)
4. Inode exhaustion (df -h fine, df -i at 100%) — many tiny files (sessions, mail queue, cache)
5. Backups/dumps/core files left in home or /tmp

First checks:
- Linux: `sudo du -xh / --max-depth=2 2>/dev/null | sort -rh | head -20`
  - if du total ≪ df used: `sudo lsof +L1 | head -20`
  - if "No space" but df shows free: `df -i`
- Windows: `Get-PSDrive -PSProvider FileSystem` then `Get-ChildItem C:\ -Directory -Force -ErrorAction SilentlyContinue | ForEach-Object { [pscustomobject]@{Path=$_.FullName; GB=[math]::Round((Get-ChildItem $_.FullName -Recurse -File -Force -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum/1GB,2)} } | Sort-Object GB -Descending | Select-Object -First 10`
- macOS: `sudo du -xh -d 2 / 2>/dev/null | sort -rh | head -20`

Safe fixes: `journalctl --vacuum-size=500M`, `docker system df` then `docker image prune` (confirm first), logrotate config, restart the process holding deleted files.

## 2. Service won't start / keeps crashing
Likely causes:
1. Config error after a recent edit/upgrade
2. Port already in use
3. Missing/incorrect file permissions or ownership (data dir, socket, cert)
4. Disk full / read-only filesystem
5. Dependency not up (DB, mount, network)

First checks:
- Linux (systemd): `sudo journalctl -u <service> --no-pager -n 50`
- Windows: `Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='Service Control Manager'} -MaxEvents 20 | Format-List TimeCreated,Message`, or the app's own log
- macOS (launchd): `log show --last 30m --predicate 'process == "<name>"' | tail -50`
- Many daemons have a config test: `nginx -t`, `apachectl configtest`, `sshd -t`, `named-checkconf`.

## 3. Can't connect to a host or port
Likely causes (ping works but service doesn't → skip straight to 2–4):
1. Service not running / not listening on that interface (bound to 127.0.0.1)
2. Host firewall (ufw/firewalld/iptables/Windows Firewall)
3. Cloud security group / NSG / network ACL
4. Service-level restriction (sshd AllowUsers, RDP disabled, NLA/cert, max sessions)
5. Routing/VPN/DNS pointing at the wrong IP

First checks:
- From the client: Linux/macOS `nc -vz <host> <port>`; Windows `Test-NetConnection <host> -Port <port>`
- On the server: Linux `sudo ss -tlnp | grep <port>`; Windows `Get-NetTCPConnection -LocalPort <port> -State Listen`
- RDP specifically: `Get-Service TermService`, `Get-ItemProperty 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name fDenyTSConnections`, firewall rule group "Remote Desktop".

## 4. DNS not resolving
Likely causes:
1. Resolver config wrong/unreachable (`/etc/resolv.conf`, systemd-resolved, VPN-pushed DNS)
2. Record actually missing/wrong at the authoritative server, or recently changed (TTL)
3. Local override (`/etc/hosts`, `C:\Windows\System32\drivers\etc\hosts`)
4. Firewall blocking UDP/TCP 53

First checks:
- Linux/macOS: `dig <name> +short` then `dig @1.1.1.1 <name> +short` to compare
- Linux (systemd-resolved): `resolvectl status | head -30`
- Windows: `Resolve-DnsName <name>` and `Resolve-DnsName <name> -Server 1.1.1.1`

## 5. 502 / 503 / 504 from a proxy or load balancer
Likely causes:
1. Upstream/backend process down or crashed (502)
2. Proxy pointing at wrong host/port/socket, or backend bound elsewhere
3. Backend too slow → timeout (504)
4. Upstream socket permission / SELinux denying proxy connect
5. LB health checks failing → no healthy targets (503)

First checks:
- nginx: `sudo tail -n 30 /var/log/nginx/error.log` (the line names the upstream and the errno: "connection refused" vs "timed out" vs "permission denied")
- Backend directly: `curl -sv http://127.0.0.1:<port>/ -o /dev/null`
- Cloud LB: target health in console / `aws elbv2 describe-target-health --target-group-arn <arn>`

## 6. High CPU
Likely causes:
1. One runaway process (stuck loop, crawler, backup, AV scan)
2. Load from traffic spike / bot
3. I/O wait masquerading as load (check `wa`)
4. Crypto-miner / compromise (unknown process names, high CPU from /tmp)

First checks:
- Linux: `top -b -n 1 | head -20`
- Windows: `Get-Process | Sort-Object CPU -Descending | Select-Object -First 10 Name,Id,CPU,WS`
- macOS: `top -l 1 -o cpu -n 10`

## 7. Memory exhaustion / OOM kills
Likely causes:
1. One process leaking/growing
2. Too many workers configured for RAM size
3. No swap on a small VM
4. Page cache misread as "used" (not a real problem)

First checks:
- Linux: `free -h` then `sudo dmesg -T | grep -i -E 'killed process|out of memory' | tail -10`
- Windows: `Get-Process | Sort-Object WS -Descending | Select-Object -First 10 Name,Id,@{n='MB';e={[int]($_.WS/1MB)}}`
- Containers: `docker stats --no-stream`, `kubectl describe pod <pod> | grep -A3 'Last State'`

## 8. TLS / certificate errors
Likely causes:
1. Certificate expired (auto-renew silently failing)
2. Incomplete chain (missing intermediate)
3. Hostname mismatch (SAN doesn't cover the name used)
4. Client clock wrong
5. Client trust store missing a private/corporate CA

First check:
- `echo | openssl s_client -connect <host>:443 -servername <host> 2>/dev/null | openssl x509 -noout -dates -subject -ext subjectAltName`
- Windows: `$c=[Net.Sockets.TcpClient]::new('<host>',443); $s=[Net.Security.SslStream]::new($c.GetStream(),$false,{$true}); $s.AuthenticateAsClient('<host>'); $s.RemoteCertificate | Format-List Subject,NotAfter`

## 9. Permission denied
Likely causes:
1. Wrong owner/mode on the file or a parent directory
2. Process running as a different user than assumed
3. SELinux/AppArmor denial (Linux) despite correct mode
4. Read-only mount / noexec mount
5. Windows ACL or UAC elevation

First checks:
- Linux: `namei -l <path>` (shows every directory in the path), then `sudo ausearch -m avc -ts recent | tail` if SELinux
- Windows: `Get-Acl <path> | Format-List`
- `mount | grep <mountpoint>` for ro/noexec

## 10. Slow host
Likely causes:
1. Disk I/O saturation (high `wa`, swap thrashing)
2. Memory pressure → swapping
3. CPU-bound process
4. Network latency/packet loss to a dependency
5. Noisy neighbor / burst credits exhausted (cloud t-class instances)

First checks:
- Linux: `vmstat 1 5` (look at `wa`, `si/so`, `r`)
- Windows: `Get-Counter '\Processor(_Total)\% Processor Time','\Memory\Available MBytes','\PhysicalDisk(_Total)\% Disk Time' -SampleInterval 1 -MaxSamples 3`

## 11. Container restarting / CrashLoopBackOff
Likely causes:
1. App exits on startup (bad env var/config, can't reach dependency)
2. OOMKilled (memory limit too low)
3. Failing liveness probe killing a slow-starting app
4. Image/entrypoint wrong after a rebuild

First checks:
- Docker: `docker logs --tail 50 <container>` and `docker inspect <container> --format '{{.State.ExitCode}} {{.State.OOMKilled}}'`
- Kubernetes: `kubectl logs <pod> --previous --tail=50`, then `kubectl describe pod <pod>` (Last State, Events)

## 12. Scheduled job not running
Likely causes:
1. Environment differs from interactive shell (PATH, working dir, env vars)
2. Job runs but fails silently (no output captured)
3. Schedule/syntax wrong or timezone confusion
4. Permissions / wrong user's crontab

First checks:
- Linux: `grep CRON /var/log/syslog | tail -20` (Debian/Ubuntu) or `journalctl -u cron --no-pager -n 30`
- Windows: `Get-ScheduledTaskInfo -TaskName '<name>' | Format-List LastRunTime,LastTaskResult,NextRunTime`
