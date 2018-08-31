# GitHub-Troubleshooting

Connect to your appliance via SSH:
```bash
ssh -p 122 admin@git.domain.com
```


## Exceptions
The exceptions log is JSON, and can be hard on the eyes. jq can help you make more sense of it.

Readable exception messages with times. The -c flag compacts the output so it's all on one line.

```bash
jq -c '{created_at, message}' /var/log/github/exceptions.log
```

Count exceptions that aren't slow requests or queries and find the most common ones.
```bash
grep -vi slow /var/log/github/exceptions.log | jq .message | sort | uniq -c | sort -nr | head
```
Print a readable backtrace for an exception:
```bash
grep -i -m1 "timeout" /var/log/github/exceptions.log | jq -r .backtrace
```
Print everything except the backtrace:
```bash
grep -i -m1 "timeout" /var/log/github/exceptions.log | jq 'del(.backtrace)'
```
Top repos with slow requests:
```bash
grep SlowRequest /var/log/github/exceptions.log | jq '.repo' | sed -e 's/.*\///' | sort | uniq -c | sort -rn
```

## babeld
All Git requests go through babeld, so it's a useful place to look for Git traffic. The babeld log format is a series of key=value pairs. Run the following commands on the appliance. This will give you some initial insight into Git operations on the appliance.

10 minute request counts overall, which can be useful spotting sudden increases in traffic that can be an indication of misconfigured or poorly written scripts for example.
```bash
cut -c 1-15 /var/log/babeld/babeld.log | uniq -c
```

1 minute request counts overall, which can be useful spotting sudden increases in traffic that can be an indication of misconfigured or poorly written scripts for example.
```bash
cut -c 1-16 /var/log/babeld/babeld.log | uniq -c
```

Top IP addresses, and often one IP or subnet stands out as misbehaving.
```bash
grep -o 'ip=[^ ]*' /var/log/babeld/babeld.log | sort | uniq -c | sort -nr | head
```

Filter by IP addresses, to see when it spiked
```bash
grep 172.18.1.201 /var/log/babeld/babeld.log | cut -c 1-13 | uniq -c
```

Top requested repositories
```bash
grep -o 'repo=[^ ]*' /var/log/babeld/babeld.log | sort | uniq -c | sort -nr | head
```

Top requested organizations or users
```bash
grep -o 'repo=[^/]*' /var/log/babeld/babeld.log | sort | uniq -c | sort -nr | head
```

You can also download a diagnostics file to your local machine (in the root of d:\ with the example below) by running:
```bash
ssh-p 122 admin@git.domain.com ghe-diagnostics > /d/ghe-diagnostics_$(DATE).txt
```

Only look at a specific protocol.
```bash
grep proto=http babeld-logs/babeld.log | grep -o 'repo=[^/]*' | sort | uniq -c | sort -nr | head
```

Extract the duration_ms value and sort to find out how much time is taken for the longest running operations
```bash
grep 'my/repo' babeld-logs/babeld.log | sed -e "s/.*duration_ms=\([^ ]*\).*/\1/" | sort -nr | head -n 20
```

Feed the duration_ms high score table back into grep to inspect the full babeld event
```bash
grep 'my/repo' babeld-logs/babeld.log | sed -e "s/.*duration_ms=\([^ ]*\).*/\1/" | sort -nr | head -n 5 | grep -f - babeld-logs/babeld.log
```

Number of pushes in 10 minute intervals:
```bash
grep receive-pack babeld-logs/babeld.log | cut -c 1-15 | uniq -c
```

Number of clones in 10 minute intervals:
```bash
grep upload-pack babeld-logs/babeld.log | cut -c 1-15 | uniq -c
```

Top 10 repos with pushes in a specific 10 minute interval:
```bash
grep -F 'Wed Jun  1 15:5' babeld-logs/babeld.log | grep -F receive-pack | grep -oP 'repo=[^ ]*' | sort | uniq -c | sort -rn | head
```

## GitHub auth
Users failing authentication - helpful in spotting misconfigured systems that may be tying up auth workers.
```bash
grep -F 'at=failure' github-logs/auth.log | grep -o 'login=[^ ]*'| sort | uniq -c | sort -nr | head
```

Authentication failure reason for a specific user.
```bash
grep -F 'at=failure' github-logs/auth.log | grep 'login=username' | grep -o 'failure_type=[^ ]*' | sort | uniq -c
```

Authentications failing because of invalid LDAP usernames.
```bash
grep "Invalid ldap username" github-logs/auth.log | grep -o "login=[^ ]* " | sort | uniq -c | sort -nr | head
```

LDAP connection issues or request saturation through polling. Reported when gitauth worker process attempting LDAP authentication takes too long to respond. This can contribute to load as other requests may be forced to wait during the timeout period.
```bash
grep "unexpected return code from _gitauth" babeld-logs/babeld.log | cut -c 1-15 | uniq -c
```

Gitauth workers terminated due to timeout. Could be a failing or slow to respond LDAP server. Is also possible to adjust LDAP authentication timeout settings.
```bash
grep "unicorn worker killed for taking too long" github-logs/exceptions.log | grep ldap | jq .created_at | cut -c 1-15 | uniq -c
```

## github.log
Top Web or API requests by User agent
```bash
awk '{print $12}' web-logs/github.log | sort | uniq -c | sort -nr | head
```

haproxy.log
API requests in 10 minute intervals:
```bash
grep -F 'api/v3' system-logs/haproxy.log | cut -c 1-11 | uniq -c
```

Top API endpoints:
```bash
grep -F 'api/v3' system-logs/haproxy.log | grep -o "} \".*" | sort | uniq -c | sort -nr | head -n 10
```

Top API endpoints in a specific 10 minute interval:
```bash
grep -F 'Jun  1 16:0' system-logs/haproxy.log | grep -F 'api/v3' | grep -o "} \".*" | sort | uniq -c | sort -nr | head -n 10
```

Top 10 API IPs in a specific 10 minute interval
```bash
grep -F 'Jun  1 16:0' system-logs/haproxy.log | grep -F 'api/v3' | grep -oP "]: [^:]*" | sort | uniq -c | sort -nr | head -n 10
```

Top IP request counts in 10 minute intervals:
```bash
grep '[IP]' system-logs/haproxy.log.1 | cut -c 1-11 | uniq -c
```

## redis.log
Check for slow disk IO:
```bash
grep "disk is busy" redis-logs/redis.log
```

## syslog
Strip syslog noise to look for abnormalities:
```bash
grep -v -e UFW -e CRON -e syslog-ng system-logs/syslog | less
```

## Kernel logs
Hypervisor CPU stealing:
```bash
grep "soft lockup" system-logs/systemd/journalctl-k.log
```

## Replication
### Breaking Replication
Break down the previous replication pair by running:

```bash
ghe-repl-stop && ghe-repl-teardown
```

on the previous replica, before setting up a new replica. Running both in parallel is not supported.

### Replication verbose information
Run this on the replica instance:

```bash
ghe-repl-status -vv
```

and this on the primary

```bash
ghe-spokes status
```

### Diagnose repositories
We can address the repositories with bad checksums. To gain more insight into the problems there, please send support the output of the following from the primary instance:
For organization repos

```bash
ghe-spokes diagnose ORG/Repo e.g. ghe-spokes diagnose GR/HBS
```

For user repos
```bash
ghe-spokes diagnose user/repo. e.g. ghe-spokes diagnose markri/personal1
```

## Get the UUID of the Node
In some cases an old replica remains. Use the command below to get the UUID of the valid primary and replica servers.

```bash
cat /data/user/common/uuid
```

## Destroy old server instances
In some cases an old replica remains. Use the command above to get the UUID of the valid primary and replica servers, and then remove the old entry by destroying it using the foillowing command.

```bash
ghe-spokes server destroy git-server-590345ea-1453-11e7-a18e-0050569d7f1a
```

## gpgverify failure
The gpgverify service on the appliance might be having trouble starting correctly, likely due to a known issue which manifests when the instance is not cleanly restarted. The issue is fixed in the 2.10.4 release, but please run the following on the primary to resolve it locally:

```bash
sudo systemctl stop gpgverify
sudo rm /data/gpgverify/current/tmp/sockets/gpgverify.sock
sudo systemctl start gpgverify
```

ghe-support-bundle errors
Sometimes when you try generate logs you get the following:
```bash
ssh -p 122 admin@git.domain.com -- 'ghe-support-bundle -x -o' > /d/syslog/support/support-bundle.tgz
mkdir: cannot create directory '/var/log/haproxy.log.1': File exists
```

```bash
$ ssh -p 122 admin@git.domain.com -- 'ghe-support-bundle -o' > /d/syslog/support/support-bundle.tgz
mkdir: cannot create directory '/var/log/haproxy.log.1': File exists
```

This is fixed in 2.10.4, but the workaround is:
```bash
sudo sed -i 's#sanitize_logs /var/log/haproxy.log\*#sanitize_logs "/var/log/haproxy.log*"#' /usr/local/bin/ghe-support-bundle
```
