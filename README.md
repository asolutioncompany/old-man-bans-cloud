# old-man-bans-cloud

> **Warning**
> This system has not yet been fully reviewed, refined, or tested. Treat current
> scripts and configuration as work in progress.

Ubuntu/Debian Nginx access log analyzer like fail2ban for automatic banning of IP addresses and subnets by agent string patterns, by resolved DNS names of virtual servers often used by vulnerability scanning, spam bots, and data mining bots, and by detection of botnets flooding systems with whole subnets for the purpose of data mining.

It includes CLI and cron tooling for realtime, daily, and weekly nginx access-log scanning. It maintains persisted ipset/iptables to block traffic at the firewall before it reaches the webserver. Cached DNS lookups and automatic botnet detection are the stand out features.

## Project Layout

- `cli/`: user-invoked commands (add to `PATH`)
- `cron/`: scheduled scanners (`cron-realtime-scan`, `cron-daily-scan`, `cron-weekly-scan`)
- `config/`: main `.conf` defaults plus optional `.local` override files
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

Defaults are loaded from `config/old-man-bans-cloud.conf`.
To override values, add one or more `config/*.local` files (any filename ending in
`.local`). Local files are loaded after `.conf` files, so local values win.

Example:

```bash
cat > ./config/site.local <<'EOF'
LOG_DIR=/var/www/my-site-logs
SUBNET_THRESHOLD=15
EOF
```

Config files:

- `config/old-man-bans-cloud.conf`
  - `LOG_DIR` (default `/var/log/nginx`)
    - Supports multiple directories separated by whitespace (space, tab, or newline)
  - `ACCESS_LOG_BASENAME` (default `access.log`)
  - `DNS_TIMEOUT_SECONDS` (default `3`)
  - `HOST_SCAN_LOG=on|off` (default `on`)
  - `HOST_SCAN_BAN=on|off` (default `on`)
  - `AGENT_SCAN_LOG=on|off` (default `on`)
  - `AGENT_SCAN_BAN=on|off` (default `on`)
  - `SITEMAP_SCAN_LOG=on|off` (default `on`)
  - `SITEMAP_SCAN_BAN=on|off` (default `on`)
  - `SUBNET_PROMOTION_LOG=on|off` (default `on`)
  - `SUBNET_PROMOTION=on|off` (default `on`)
  - `SUBNET_THRESHOLD=10` (promote to `/24` when unique IP count is greater than this value)
  - `BAD_HOSTNAMES` (`|`-delimited host/IP ban tokens)
  - `GOOD_HOSTNAMES` (`|`-delimited host/IP allowlist tokens)
  - `BAD_AGENTS` (`|`-delimited suspicious agent tokens)
  - `GOOD_AGENTS` (`|`-delimited known-good agent tokens)
  - `SITEMAP_TOKENS` (`|`-delimited sitemap/path tokens; default `sitemap`)
- Optional `.local` overrides:
  - Any `config/*.local` file can override none, any, or all settings.
  - Token overrides are complete replacement values, not merge/append behavior.
    For example, setting `BAD_HOSTNAMES=example.com` in a `.local` file means only
    `example.com` is searched and default `BAD_HOSTNAMES` values are ignored.
  - Pipe-delimited token example:
    `BAD_HOSTNAMES=example.com|example.net`.

Global allowlists (`GOOD_HOSTNAMES`, `GOOD_AGENTS`) are applied to
hostname, agent, and sitemap scans.

Hostname lookups use a shared persistent cache (`logs/hostname-cache.tsv`) to
avoid repeated DNS lookups across cron and CLI runs.

Allowlist precedence notes:

- `BAD_HOSTNAMES` matches are ignored when `GOOD_HOSTNAMES` or
  `GOOD_AGENTS` match the same event's hostname, IP, or agent string.
- `BAD_AGENTS` matches are ignored when `GOOD_HOSTNAMES` or
  `GOOD_AGENTS` match the same event's hostname, IP, or agent string.

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

You can export current bans using `perm-list`, for example:

```bash
cli/perm-list > logs/perm-list-export.log
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
- Uses `BAD_HOSTNAMES` for matches and `GOOD_HOSTNAMES`/`GOOD_AGENTS` for allowlisting
- Controlled by:
  - `HOST_SCAN_LOG=on|off`
  - `HOST_SCAN_BAN=on|off`

### Agent Scan

- Uses `BAD_AGENTS` for matches and `GOOD_HOSTNAMES`/`GOOD_AGENTS` for allowlisting
- Controlled by:
  - `AGENT_SCAN_LOG=on|off`
  - `AGENT_SCAN_BAN=on|off`

### Sitemap Scan

- Uses `SITEMAP_TOKENS` token matching against request paths
- Example: `sitemap` matches `sitemap.xml`, `sitemap2.xml`, etc.
- Controlled by:
  - `SITEMAP_SCAN_LOG=on|off`
  - `SITEMAP_SCAN_BAN=on|off`

### Subnet Promotion

- Groups matched IPs by `/24`
- Promotes when unique IPs in a subnet are greater than `SUBNET_THRESHOLD`
- Removes covered `/32` bans after subnet promotion
- Controlled by:
  - `SUBNET_PROMOTION_LOG=on|off`
  - `SUBNET_PROMOTION=on|off`

Use `perm-subnets` for automatic threshold-based promotions. For manual
promotion of known IP groups to a subnet ban, use `perm-ban <subnet>/24`.

Realtime does not perform subnet promotion; daily/weekly/full flows are where
subnet-focused processing is intended.

Hostname placeholders used in logs:

- `-` when hostname is not identified
- `-TIMEOUT-` when DNS lookup times out

## Analyzer Output Logs

- `logs/host-ban-log`: host/cloud match events
- `logs/agent-ban-log`: agent match events
- `logs/unique-agent-log`: unique discovered agent strings
- `logs/sitemap-ban-log`: sitemap-matched events considered for subnet promotion
- `logs/subnet-promotion-log`: subnet promotions (`subnet`, unique-ip count, removed `/32` count)

Ban-event logs use format:
  - `timestamp ip hostname agent-string`
  - columns are space-delimited; parse first three fields as timestamp, IP, hostname

If `*_LOG=off`, no log is written.
If `*_BAN=off`, no ban is executed.

## Subnet Detection

There are good subnets and bad ones. Examples of good botnets would be GoogleBot and a corporate network using your website for it's intended function. You generally want to allow this activity, and these types of botnets need to be configured to avoid automatic banning.

There are several types of botnets, but for large websites with large amounts of data, the general purpose is for data mining. The troublesome botnets are the ones which use a large number of IP addresses to mine your site, hiding in access logs as low volume activity, when as a whole, they are hammering your website.

Google will only use 3 IP addresses to index a website, whereas as a large data mining operation will use hundreds from multiple subnets.

One concern is corporate use where users might each have their own IP address. Another concern would be multiple users from an ISP being on the same subnet.

Therefore, it is recommended to set a higher thresold.

The sitemap detection was added as a good way to detect botnets being used for data mining. Large websites will have several sitemaps and botnets will hit these sitemaps from several IP addresses within a day, making it easy to detect and stop botnets performing data mining. It is also fairly safe to assume any access of your sitemap is likely from a bot performing data mining.

## Other Considerations

Daily and weekly scans may require significant processing on large sites,
especially when many DNS lookups are performed. Cached hostname lookups help,
but lookup-heavy workflows can still be costly at scale.

For larger environments, geolocation databases or ASN/ARIN-style network-owner
data may be a more efficient and stable signal than repeated reverse-DNS
lookups alone.

Implementing ban expiration (time-based unban after a long period) should be
considered so that bans can decay over time instead of remaining permanent
forever.
