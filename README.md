# Linux nginx Port Conflict Lab

A hands-on troubleshooting lab on a Debian VM: nginx refuses to start even though its configuration is valid, because another process already holds port 80. I diagnose the cause from logs and system tools, restore service, and document the incident.

## What this shows
- Telling a config error apart from a port conflict (`nginx -t` passes, but the service still fails)
- Reading systemd status and journal logs to find a `bind() ... Address already in use` error
- Finding which process owns a port with `ss -tlnp`
- Verifying the response actually comes from nginx (the `Server:` header), not just a 200 OK
- Writing a clear incident report: timeline, evidence, root cause, resolution, prevention

## Report
[Incident report (PDF)](incident-report.pdf)

## Environment
- Debian 13 VM (VirtualBox)
- nginx, managed by systemd
- Tools: systemctl, journalctl, nginx -t, ss, curl, kill

## Method
Baseline, introduce the fault, gather evidence, find the root cause, fix, verify, document. The fault was introduced deliberately in an isolated lab VM. This is a troubleshooting exercise, not an attack.

## Findings
- **Symptom:** `systemctl start nginx` failed, even though `nginx -t` reported the configuration was valid.
- **Root cause:** a Python web server (PID 2944) already held TCP port 80, so nginx's `bind()` call failed with error 98 (address already in use).
- **Key clue:** the `ExecStartPre` config test succeeded (`status=0`) while `ExecStart` failed (`status=1`), which pointed away from a config problem. `curl` also returned 200 OK, but the `Server:` header showed `SimpleHTTP`, not nginx.
- **Fix:** identified the process with `ss -tlnp`, stopped it with `kill`, restarted nginx, and confirmed nginx owned port 80 again.

## How I reproduced it
1. **Baseline:** confirm nginx owns port 80 and responds.
   `sudo ss -tlnp | grep :80` and `curl -I http://127.0.0.1`
2. **Create the conflict:** stop nginx, then start a plain Python web server on port 80 to stand in for an unexpected process.
   `sudo systemctl stop nginx` and `sudo python3 -m http.server 80 &`
3. **Trigger the outage:** try to start nginx (it fails).
   `sudo systemctl start nginx`
4. **Investigate:** check service state, logs, and the config test.
   `systemctl status nginx --no-pager`, `sudo journalctl -u nginx -n 20 --no-pager`, `sudo nginx -t`
5. **Identify the culprit:** find which process owns port 80 and check what is actually answering.
   `sudo ss -tlnp | grep :80` and `curl -I http://127.0.0.1`
6. **Fix:** stop the conflicting process and start nginx.
   `sudo kill <PID>` and `sudo systemctl start nginx`
7. **Verify:** confirm nginx owns port 80 and the response comes from nginx.
   `sudo ss -tlnp | grep :80`, `systemctl status nginx --no-pager`, `curl -I http://127.0.0.1`

## Evidence

**1. Baseline: nginx owns port 80 and responds with 200 OK**
![Baseline](screenshots/1-baseline.PNG)

**2. nginx start fails**
![Start failed](screenshots/2-start-failed.PNG)

**3. Service status: config test passed, start step failed**
![Status failed](screenshots/3-status-failed.PNG)

**4. Journal: bind() to 0.0.0.0:80 failed (98: Address already in use)**
![Journal error](screenshots/4-journal-error.PNG)

**5. nginx -t passes: the config is not the problem**
![nginx -t OK](screenshots/5-nginx-t-ok.PNG)

**6. Culprit found: python3 (PID 2944) owns port 80**
![Culprit found](screenshots/6-culprit-found.PNG)

**7. Conflicting process stopped, port 80 free**
![Port freed](screenshots/7-port-freed.PNG)

**8. Recovery: nginx back on port 80, 200 OK from nginx**
![Recovered](screenshots/8-recovered.PNG)
