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
- `cron/`: scheduled scanners (`realtime-scan`, `daily-scan`, `weekly-scan`)
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
sudo ./cli/perm-setup
```

## Analyzer Configuration

Create local config files from examples:

```bash
cp ./config/old-man-bans-cloud.local.example ./config/old-man-bans-cloud.local
cp ./config/bad-clouds.local.example ./config/bad-clouds.local
cp ./config/good-clouds.local.example ./config/good-clouds.local
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
  - `SUBNET_SCAN_BAN=on|off` (default `on`)
  - `SUBNET_THRESHOLD=10`
- `config/bad-clouds.local`: `|`-delimited ban tokens for host/IP text
- `config/good-clouds.local`: `|`-delimited global allowlist for host/IP text
- `config/bad-agents.local`: `|`-delimited suspicious agent tokens
- `config/good-agents.local`: `|`-delimited global allowlist for agent text
- `config/sitemap.local`: `|`-delimited sitemap path tokens (default `sitemap`)

Global allowlists (`good-clouds.local`, `good-agents.local`) are applied to both
daily-agent checks and realtime-host checks.

Hostname lookups use a shared persistent cache (`logs/hostname-cache.tsv`) to
avoid repeated DNS lookups across cron runs.

Allowlist precedence notes:

- `bad-clouds.local` matches are ignored when `good-clouds.local` or
  `good-agents.local` match the same event's hostname, IP, or agent string.
- `bad-agents.local` matches are ignored when `good-clouds.local` or
  `good-agents.local` match the same event's hostname, IP, or agent string.

Matching behavior summary:

- Hostname and agent matching is case-insensitive token substring matching
  (partial matches; wildcard-like behavior without explicit `*` syntax).
- Sitemap path matching is case-insensitive token substring matching
  (for example, `sitemap` matches `sitemap.xml`, `sitemap1.xml`, `sitemap2.xml`).
- IPv4 matching is exact when an IP token is used.

## Realtime Host Scan

`cron/realtime-scan` reads nginx `access.log` and evaluates fixed
2-minute windows:

- `end = now - 60 seconds`
- `start = end - 120 seconds`

State is persisted in `logs/realtime-scan.state`, and access is serialized with
`logs/realtime-scan.lock` so concurrent runs do not corrupt interval tracking.

- Mode: hostname/IP detection
- Reverse lookup uses `host` with `dig` fallback.
- Hostname/IP tokens are checked from `bad-clouds.local`.
- Log/ban behavior:
  - `HOST_SCAN_LOG=on`: write matched events to log
  - `HOST_SCAN_BAN=on`: execute `perm-ban`
  - If one is `off`, only the other action occurs
  - If both are `off`, realtime scan performs no action for matches

## Daily Agent Scan

`cron/daily-scan` scans yesterday's log (`access.log.1`) and evaluates
agent tokens from `bad-agents.local`.

It does **not** scan the current live log (`access.log`); daily mode is
explicitly for `access.log.1`.

- Mode: agent detection
- For each match, hostname is resolved for logging and hostname allowlist checks
- Log/ban behavior:
  - `AGENT_SCAN_LOG=on`: write ban-event log and unique agent log
  - `AGENT_SCAN_BAN=on`: execute `perm-ban`
  - If one is `off`, only the other action occurs
  - If both are `off`, daily scan performs no action for matches

## Weekly Scan (6 days)

Weekly mode is designed to evaluate 6 days of completed rotated logs, so it
should be scheduled every 6 days.

`cron/weekly-scan` scans rotated logs (including `.gz`) and excludes the
current live `access.log`.

Detection focus:

- Only events whose request path matches `config/sitemap.local` tokens
  - default token `sitemap` matches `sitemap.xml`, `sitemap2.xml`, etc.
- Global allowlists still apply (`good-agents.local`, `good-clouds.local`)

Subnet promotion behavior:

- Group matched unique IPs by `/24`
- If unique IPs in a subnet are greater than `SUBNET_THRESHOLD` (default `10`)
  - ban subnet via `perm-ban` (example: `198.51.100.0/24`)
  - remove covered individual `/32` bans via `perm-unban`

Weekly log/ban toggles:

- `SITEMAP_SCAN_LOG=on`: write weekly sitemap-matched event logs
- `SITEMAP_SCAN_BAN=on`: allow sitemap matches to feed enforcement logic
- `SUBNET_SCAN_LOG=on`: write subnet promotion logs
- `SUBNET_SCAN_BAN=on`: perform subnet ban and `/32` cleanup
- same semantics as other scanners: log-only, ban-only, both-on, or both-off

Hostname placeholders used in logs:

- `-` when hostname is not identified
- `-TIMEOUT-` when DNS lookup times out

## Analyzer Output Logs

- `logs/realtime-host-ban-log`: realtime host/cloud match events
- `logs/daily-agent-ban-log`: daily agent match events
- `logs/daily-agent-log`: unique discovered daily agent strings
- `logs/weekly-subnet-ban-log`: weekly matched events for promoted subnets
- `logs/subnet-promotion-log`: subnet promotions (`subnet`, unique-ip count, removed `/32` count)

Ban-event logs use format:
  - `timestamp ip hostname agent-string`
  - columns are space-delimited; parse first three fields as timestamp, IP, hostname

If `*_LOG=off`, no log is written.
If `*_BAN=off`, no ban is executed.

## Cron Setup

Edit your crontab:

```bash
crontab -e
```

Add realtime scan every 2 minutes (use your actual absolute install path):

```cron
*/2 * * * * /home/ubuntu/old-man-bans-cloud/cron/realtime-scan >> /home/ubuntu/old-man-bans-cloud/logs/realtime-analyzer.log 2>&1

# Daily agent scan at 02:10 (yesterday access log)
10 2 * * * /home/ubuntu/old-man-bans-cloud/cron/daily-scan >> /home/ubuntu/old-man-bans-cloud/logs/daily-analyzer.log 2>&1

# Weekly subnet scan every 6 days at 03:30 (rotated logs, including .gz)
30 3 */6 * * /home/ubuntu/old-man-bans-cloud/cron/weekly-scan >> /home/ubuntu/old-man-bans-cloud/logs/weekly-analyzer.log 2>&1
```

Run full scan manually once (all access logs, including rotated `.gz` logs; 7-day coverage via today + yesterday + weekly rotated set):

```bash
./cli/scan-all
```

Promote existing dense `/32` bans to `/24` using current ban table:

```bash
./cli/promote-subnets
```

## CLI Commands

- `perm-ban`: add one or more IPs/CIDRs to `badnets_perm`
- `perm-unban`: remove one or more IPs/CIDRs from `badnets_perm`
- `perm-list`: show current blocklist entries (`/32` normalized for singleton IPs)
- `perm-setup`: one-time setup for `badnets_perm` and INPUT drop rule
- `promote-subnets`: promote dense `/32` bans into `/24` and remove covered `/32`
- `scan-all`: scan all access logs (including `.gz`) by running realtime + daily + weekly scanners in sequence
- `scan-today`: run realtime scan once (current log interval)
- `scan-yesterday`: run daily scan once (`access.log.1`)
