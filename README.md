# old-man-bans-cloud

> **Warning**
> This system has not yet been fully reviewed, refined, or tested. Treat current
> scripts and configuration as work in progress.

CLI + cron tooling for realtime, daily, and weekly nginx access-log scanning to
automatically identify and ban IP addresses and botnet subnets based on
hostnames, agent strings, and sitemap paths, backed by a persistent
`ipset`/`iptables` ban list. It works like fail2ban, but adds cached hostname
lookups and automatic subnet detection to handle problematic bots that are hard
to detect without false positives.

## Project Layout

- `cli/`: user-invoked commands (add to `PATH`)
- `cron/`: scheduled scanners (`cron-realtime-scan`, `cron-daily-scan`, `cron-weekly-scan`)
- `config/`: user-owned `.local` configuration files
- `logs/`: runtime logs and scanner state files (created automatically)

## Prerequisites

- `ipset`
- `iptables`
- `netfilter-persistent` (for persistence in `perm-*` scripts)
- DNS lookup tools for realtime hostname checks:
  - `host` (usually from `dnsutils` or `bind-utils`)
  - `dig` (fallback reverse lookup)
- `timeout` and `flock` (normally available on Linux)

Install example (Debian/Ubuntu):

```bash
sudo apt-get update
sudo apt-get install -y ipset iptables iptables-persistent dnsutils
```

## Initial Firewall Setup

Run once to initialize `badnets_perm` and INPUT drop rule:

```bash
cli/perm-setup
```

## Analyzer Configuration

Create local config files from examples:

```bash
cp ./config/old-man-bans-cloud.local.example ./config/old-man-bans-cloud.local
cp ./config/bad-hostnames.local.example ./config/bad-hostnames.local
cp ./config/good-hostnames.local.example ./config/good-hostnames.local
cp ./config/bad-agents.local.example ./config/bad-agents.local
cp ./config/good-agents.local.example ./config/good-agents.local
cp ./config/sitemap.local.example ./config/sitemap.local
```

Config files:

- `config/old-man-bans-cloud.local`
  - `LOG_DIR` (default `/var/log/nginx`)
  - `ACCESS_LOG_BASENAME` (default `access.log`)
  - `DNS_TIMEOUT_SECONDS` (default `3`)
  - `HOST_SCAN_LOG=on|off` (default `on`)
  - `HOST_SCAN_BAN=on|off` (default `on`)
  - `AGENT_SCAN_LOG=on|off` (default `on`)
  - `AGENT_SCAN_BAN=on|off` (default `on`)
  - `SITEMAP_SCAN_LOG=on|off` (default `on`)
  - `SITEMAP_SCAN_BAN=on|off` (default `on`)
  - `SUBNET_SCAN_LOG=on|off` (default `on`)
  - `SUBNET_PROMOTION=on|off` (default `on`)
  - `SUBNET_THRESHOLD=10` (promote to `/24` when unique IP count is greater than this value)
- `config/bad-hostnames.local`: `|`-delimited ban tokens for host/IP text
- `config/good-hostnames.local`: `|`-delimited global allowlist for host/IP text
- `config/bad-agents.local`: `|`-delimited suspicious agent tokens
- `config/good-agents.local`: `|`-delimited global allowlist for agent text
- `config/sitemap.local`: `|`-delimited sitemap path tokens (default `sitemap`)

Global allowlists (`good-hostnames.local`, `good-agents.local`) are applied to
hostname, agent, and sitemap scans.

Hostname lookups use a shared persistent cache (`logs/hostname-cache.tsv`) to
avoid repeated DNS lookups across cron and CLI runs.

Allowlist precedence notes:

- `bad-hostnames.local` matches are ignored when `good-hostnames.local` or
  `good-agents.local` match the same event's hostname, IP, or agent string.
- `bad-agents.local` matches are ignored when `good-hostnames.local` or
  `good-agents.local` match the same event's hostname, IP, or agent string.

Matching behavior summary:

- Hostname and agent matching is case-insensitive token substring matching
  (partial matches; wildcard-like behavior without explicit `*` syntax).
- Sitemap path matching is case-insensitive token substring matching
  (for example, `sitemap` matches `sitemap.xml`, `sitemap1.xml`, `sitemap2.xml`).
- IPv4 matching is exact when an IP token is used.

## CLI Commands

- `perm-ban`: add one or more IPs/CIDRs to `badnets_perm`
- `perm-unban`: remove one or more IPs/CIDRs from `badnets_perm`
- `perm-list`: show current blocklist entries (`/32` normalized for singleton IPs)
- `perm-setup`: one-time setup for `badnets_perm` and INPUT drop rule
- `perm-subnets`: automatically promote dense `/32` bans into `/24` using `SUBNET_THRESHOLD`
- `perm-scan-all`: scan all access logs (including `.gz`) by running realtime + daily + weekly scanners in sequence
- `perm-scan-today`: run realtime scan once (current log interval)
- `perm-scan-yesterday`: run daily scan once (`access.log.1`)

Add `cli/` to your `PATH` (so commands can be run without `cli/` prefix):

```bash
echo 'export PATH="$PATH:/home/ubuntu/old-man-bans-cloud/cli"' >> ~/.bashrc
source ~/.bashrc
```

## Cron Setup

Edit your crontab:

```bash
sudo crontab -e
```

Add realtime scan every 2 minutes (use your actual absolute install path):

```cron
# Realtime scan every 2 minutes (current access log interval)
*/2 * * * * /home/ubuntu/old-man-bans-cloud/cron/cron-realtime-scan >> /home/ubuntu/old-man-bans-cloud/logs/realtime-analyzer.log 2>&1

# Daily agent scan at 02:10 (yesterday access log)
10 2 * * * /home/ubuntu/old-man-bans-cloud/cron/cron-daily-scan >> /home/ubuntu/old-man-bans-cloud/logs/daily-analyzer.log 2>&1

# Weekly subnet scan every 6 days at 03:30 (rotated logs, including .gz)
30 3 */6 * * /home/ubuntu/old-man-bans-cloud/cron/cron-weekly-scan >> /home/ubuntu/old-man-bans-cloud/logs/weekly-analyzer.log 2>&1
```

### Realtime Scan

`cron/cron-realtime-scan` reads current `access.log` in fixed 2-minute windows:

- `end = now - 60 seconds`
- `start = end - 120 seconds`

State is tracked in `logs/realtime-scan.state` and locked by
`logs/realtime-scan.lock`.

### Daily Scan

`cron/cron-daily-scan` scans `access.log.1` (yesterday) only. It does not scan live
`access.log`.

### Weekly Scan (6 days)

`cron/cron-weekly-scan` scans completed rotated logs (including `.gz`) and is meant
to run every 6 days.

## Scan Types

### Hostname Scan

- Reverse lookup uses `host` with `dig` fallback
- Uses `bad-hostnames.local` for matches and `good-hostnames.local`/`good-agents.local` for allowlisting
- Controlled by:
  - `HOST_SCAN_LOG=on|off`
  - `HOST_SCAN_BAN=on|off`

### Agent Scan

- Uses `bad-agents.local` for matches and `good-hostnames.local`/`good-agents.local` for allowlisting
- Controlled by:
  - `AGENT_SCAN_LOG=on|off`
  - `AGENT_SCAN_BAN=on|off`

### Sitemap Scan

- Uses `sitemap.local` token matching against request paths
- Example: `sitemap` matches `sitemap.xml`, `sitemap2.xml`, etc.
- Controlled by:
  - `SITEMAP_SCAN_LOG=on|off`
  - `SITEMAP_SCAN_BAN=on|off`

### Subnet Promotion

- Groups matched IPs by `/24`
- Promotes when unique IPs in a subnet are greater than `SUBNET_THRESHOLD`
- Removes covered `/32` bans after subnet promotion
- Controlled by:
  - `SUBNET_SCAN_LOG=on|off`
  - `SUBNET_PROMOTION=on|off`

Use `perm-subnets` for automatic threshold-based promotions. For manual
promotion of known IP groups to a subnet ban, use `perm-ban <subnet>/24`.

Realtime does not perform subnet promotion; daily/weekly/full flows are where
subnet-focused processing is intended.

Hostname placeholders used in logs:

- `-` when hostname is not identified
- `-TIMEOUT-` when DNS lookup times out

## Considerations

`SUBNET_THRESHOLD` is inherently an arbitrary value. You should decide what
unique-IP count in a `/24` is high enough to justify banning the entire subnet
for your own traffic profile and false-positive tolerance.

You can also run in a manual-review mode by enabling log settings while keeping
ban/promotion settings off, then reviewing logged events before applying manual
bans and subnet promotions. This is a safer approach when tuning initial rules
or operating in environments with mixed legitimate bot traffic.

## Analyzer Output Logs

- `logs/host-ban-log`: host/cloud match events
- `logs/agent-ban-log`: agent match events
- `logs/agent-unique-log`: unique discovered agent strings
- `logs/sitemap-ban-log`: sitemap-matched events considered for subnet promotion
- `logs/subnet-promotion-log`: subnet promotions (`subnet`, unique-ip count, removed `/32` count)

Ban-event logs use format:
  - `timestamp ip hostname agent-string`
  - columns are space-delimited; parse first three fields as timestamp, IP, hostname

If `*_LOG=off`, no log is written.
If `*_BAN=off`, no ban is executed.
