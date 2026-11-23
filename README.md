# CrowdSec for the Homelab

A hands-on lab that shows how [CrowdSec](https://www.crowdsec.net/) detects attacks in your logs and blocks the attacker, step by step, on a real self-hosted web service.

You'll put a small web app on the internet, scan it with a real vulnerability scanner, and watch CrowdSec spot the scan and ban the scanner's IP within seconds. You won't write a single firewall rule.

In [Part 2](#part-2-mikrotik-bouncer-jellyfin-and-blocklists) you expose Jellyfin, let CrowdSec read the app's own log, push every ban to a **MikroTik router** (dropped in the RAW table, before it enters your network) and subscribe to a curated blocklist.

The diagram below is the Part 1 lab; Part 2 has its own.

```
          Internet: visitors, bots, scanners
                         │
                         ▼   whoami.example.com  →  A record  →  your public IP
                ┌──────────────────┐
                │      Router      │   forwards TCP 443 → homelab (blocks nothing, for now)
                └────────┬─────────┘
                         ▼
┌──────────────────────── Homelab host ──────────────────────────┐
│                                                                │
│   nftables  ◄── firewall bouncer ◄── CrowdSec Local API        │
│      │ drops banned IPs                    ▲                   │
│      ▼                                     │ decisions         │
│   Caddy (TLS, access log) ──► logs/access.log ──► CrowdSec     │
│      │                                     Security Engine     │
│      ▼                                                         │
│   whoami (Docker)                                              │
└────────────────────────────────────────────────────────────────┘
```

> **The golden rule of this series: CrowdSec decides, bouncers enforce.**
> CrowdSec reads logs and decides who to ban. A *bouncer* turns those decisions into actual blocks. Without a bouncer, nothing gets blocked.

## The series

| Part | Topic | Status |
|------|-------|--------|
| **1** | **CrowdSec basics: detect on the host, block on the host (nftables)** | ✅ [Step-by-step guide](#step-by-step-guide) |
| **2** | **Jellyfin + MikroTik bouncer: read the app's log, block on the router, add blocklists** | ✅ [Part 2](#part-2-mikrotik-bouncer-jellyfin-and-blocklists) |
| 3 | Services behind Cloudflare, where your server no longer sees the real client IP | Coming soon |
| 4 | AppSec / WAF: block malicious requests before they reach the app | Coming soon |

---

## Contents

- [Before you start](#before-you-start)
- [What's in this repo](#whats-in-this-repo)
- [Quick start](#quick-start)
- [Step-by-step guide](#step-by-step-guide)
  - [1. Install CrowdSec](#1-install-crowdsec)
  - [2. (Optional) Enroll in the CrowdSec Console](#2-optional-enroll-in-the-crowdsec-console)
  - [3. How CrowdSec works](#3-how-crowdsec-works)
  - [4. Look inside: acquisition, parsers, scenarios](#4-look-inside-acquisition-parsers-scenarios)
  - [5. Test: SSH brute force](#5-test-ssh-brute-force)
  - [6. Decisions](#6-decisions)
  - [7. Install a bouncer](#7-install-a-bouncer)
  - [8. Expose a web service](#8-expose-a-web-service)
  - [9. Teach CrowdSec to read Caddy logs](#9-teach-crowdsec-to-read-caddy-logs)
  - [10. Attack: scan the site and watch the ban](#10-attack-scan-the-site-and-watch-the-ban)
  - [11. Why some requests still got through](#11-why-some-requests-still-got-through)
- [Part 2: MikroTik bouncer, Jellyfin and blocklists](#part-2-mikrotik-bouncer-jellyfin-and-blocklists)
  - [12. Expose Jellyfin behind Caddy](#12-expose-jellyfin-behind-caddy)
  - [13. The proxy trap: Known proxies](#13-the-proxy-trap-known-proxies)
  - [14. Teach CrowdSec to read Jellyfin's log](#14-teach-crowdsec-to-read-jellyfins-log)
  - [15. Why block on the router, and why in RAW](#15-why-block-on-the-router-and-why-in-raw)
  - [16. Prepare the router: encrypted API and a restricted user](#16-prepare-the-router-encrypted-api-and-a-restricted-user)
  - [17. Run the MikroTik bouncer](#17-run-the-mikrotik-bouncer)
  - [18. Subscribe to a curated blocklist](#18-subscribe-to-a-curated-blocklist)
  - [19. Attack: brute-force Jellyfin and get banned at the edge](#19-attack-brute-force-jellyfin-and-get-banned-at-the-edge)
- [What to expose, and how](#what-to-expose-and-how)
- [Troubleshooting](#troubleshooting)
- [cscli cheat sheet](#cscli-cheat-sheet)
- [Clean up](#clean-up)

---

## Before you start

> ⚠️ **Only scan and attack servers you own or have explicit written permission to test.** Scanning other people's systems without permission is illegal.

### You'll need

| What | Why |
|------|-----|
| A Linux host (Ubuntu/Debian) with Docker and Docker Compose | Runs CrowdSec, Caddy and the demo app |
| A domain where you can add DNS records | Caddy needs a hostname to get a Let's Encrypt certificate |
| A router where you can forward TCP 443 to the host | So the internet can reach Caddy |
| **A machine outside your network** (a cheap cloud VM works, or a laptop on mobile data) | To act as the "attacker". Attacks from your home network are whitelisted and **will never be banned** |
| On the attacker machine: `curl`, [`ffuf`](https://github.com/ffuf/ffuf), `sshpass`, and a wordlist such as [SecLists `common.txt`](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/common.txt) | For the tests |
| **Part 2 only:** a MikroTik router (RouterOS 7) | The RouterOS bouncer pushes bans to it over the API |

### Placeholders used in this guide

Replace these with your own values:

| Placeholder | Meaning |
|-------------|---------|
| `whoami.example.com` | Your demo hostname |
| `jellyfin.example.com` | Your Jellyfin hostname (Part 2) |
| `<PUBLIC_IP>` | Your home/server public IP |
| `192.168.1.10` | The homelab host's LAN IP |
| `<ROUTER_IP>` | The MikroTik router's LAN IP (Part 2) |
| `<ATTACKER_IP>` | Public IP of your test/attacker machine |
| `~/crowdsec-demo` | Where you cloned this repo on the homelab host (in the video Part 2 uses `~/jellyfin` and `~/crowdsec-routeros-bouncer`; here they're subfolders of the repo) |

---

## What's in this repo

| File | Purpose |
|------|---------|
| [`docker-compose.yaml`](docker-compose.yaml) | Caddy (reverse proxy, port 443) + `traefik/whoami` (the demo app, not exposed directly) |
| [`Caddyfile`](Caddyfile) | JSON access log, proxy `/` to whoami, **404 for everything else**, Let's Encrypt TLS. Part 2 adds a `jellyfin` site block |
| [`jellyfin/docker-compose.yaml`](jellyfin/docker-compose.yaml) | Part 2: Jellyfin on the same Docker network, no public port (Caddy is its only door) |
| [`routeros-bouncer/docker-compose.yaml`](routeros-bouncer/docker-compose.yaml) | Part 2: [cs-routeros-bouncer](https://github.com/jmrplens/cs-routeros-bouncer), pushes decisions to the MikroTik router |
| [`routeros-bouncer/.env.example`](routeros-bouncer/.env.example) | Template for the router password and bouncer API key (copy to `.env`, `chmod 600`) |
| `logs/` | Caddy writes `access.log` here; CrowdSec reads it |
| [`diagrams/`](diagrams/) | Architecture diagrams used in the videos (open the `.html` files in a browser); Part 2 in [`diagrams/part2/`](diagrams/part2/) |

---

## Quick start

Already know CrowdSec and just want the lab running? Here's the short version. Each step is explained below.

```bash
# On the homelab host
curl -s https://install.crowdsec.net -o install-crowdsec.sh   # read it first!
sudo sh install-crowdsec.sh
sudo apt install crowdsec crowdsec-firewall-bouncer-nftables
sudo cscli collections install crowdsecurity/caddy

git clone https://github.com/filip-lebiecki/crowdsec.git ~/crowdsec-demo && cd ~/crowdsec-demo
# edit Caddyfile: replace the hostname with yours
docker compose up -d

sudo tee /etc/crowdsec/acquis.d/caddy-demo.yaml >/dev/null <<EOF
source: file
filenames:
  - $HOME/crowdsec-demo/logs/*.log
labels:
  type: caddy
EOF
sudo systemctl restart crowdsec
```

Then add a DNS A record (no proxy) and forward TCP 443 on your router to the host, and run the [attack](#10-attack-scan-the-site-and-watch-the-ban) from an outside machine.

---

## Step-by-step guide

### 1. Install CrowdSec

**Don't use your distro's package.** Check what Ubuntu ships:

```bash
apt-cache policy crowdsec
```

It's usually an old release (e.g. 1.4.x). Use CrowdSec's official repository instead. Their docs pipe the install script straight into `sudo sh`. Don't do that with any script you haven't read. Download it, read it, then run it:

```bash
curl -s https://install.crowdsec.net -o install-crowdsec.sh
less install-crowdsec.sh        # it only adds the CrowdSec apt repo and signing key
sudo sh install-crowdsec.sh
sudo apt install crowdsec
```

The installer detects services on the host (syslog, SSH, and Nginx/Apache if present) and sets up log collection for them. Note the yellow line in the output:

> *CrowdSec alone will not block any IP address.*

That's the most important line on the screen. We'll come back to it.

Check that both APIs are healthy:

```bash
sudo cscli lapi status    # Local API: your local decision database
sudo cscli capi status    # Central API: the community ("the crowd")
```

### 2. (Optional) Enroll in the CrowdSec Console

The [web console](https://app.crowdsec.net) gives you a map of alerts, extra blocklists, and a view of what your bouncers block. It's optional.

```bash
sudo cscli console enroll --quick
```

Open the printed URL, click **Enroll engine**, then confirm:

```bash
sudo cscli console status
```

Good to know:
- **Central API ≠ Console.** The Central API shares threat intel and gives you the community blocklist. The Console is only a dashboard.
- Alerts about **private IPs are never sent** to the console, and the free plan keeps only the **last 500 alerts**. If your console looks empty while testing, that's probably why.

### 3. How CrowdSec works

Detection and blocking are **two separate jobs**.

```
 Logs ──► Security Engine ─────────────────────────────────────► Local API ──► Bouncers ──► block
          Acquisition → Parsers → Enrichment → Scenarios → Profiles    │
                                                                       ▼
                                                                  Central API
                                                          (share bans, get the community blocklist)
```

| Component | What it does |
|-----------|--------------|
| **Acquisition** | Reads log sources: files, Docker containers, journald and more |
| **Parsers** | Turn raw log lines into structured fields (IP, path, status code, user...) |
| **Enrichment** | Adds country / network owner (ASN), and **drops whitelisted IPs** (private ranges by default) |
| **Scenarios** | Look for *patterns* of bad behaviour using "leaky buckets". An overflow becomes an **alert** |
| **Profiles** | Turn an alert into a **decision**, by default "ban this IP for 4 hours" |
| **Local API (LAPI)** | Local database of decisions. `cscli` talks to it |
| **Bouncers** | Poll the LAPI (about every 10 s) and actually block traffic |
| **Central API (CAPI)** | Anonymously shares your bans, gives you the community blocklist (about 15,000 IPs) |

> Everything in Part 1 is **log-based**: CrowdSec reads a request *after* it hit your server, so a few malicious requests get through before the ban. To stop the very first bad request you need AppSec (CrowdSec's WAF), covered in Part 4.

### 4. Look inside: acquisition, parsers, scenarios

#### Acquisition: which logs, and what format

```bash
sudo cscli metrics show acquisition
ls /etc/crowdsec/acquis.d/
cat /etc/crowdsec/acquis.d/setup.sshd.yaml
```

Each file lists the **paths** to read and a **`type` label**. The label tells CrowdSec which parser to use (`syslog`, `nginx`, `caddy`...).

#### Parsers: raw text into structured events

```bash
sudo cscli parsers list
```

Parsing runs in stages:

| Stage | Example file | Job |
|-------|--------------|-----|
| `s00-raw` | `/etc/crowdsec/parsers/s00-raw/syslog-logs.yaml` | Clean-up: timestamp, program, message |
| `s01-parse` | `/etc/crowdsec/parsers/s01-parse/sshd-logs.yaml` | Real parsing: source IP, user, event type |
| `s02-enrich` | `/etc/crowdsec/parsers/s02-enrich/geoip-enrich.yaml` | Adds GeoIP / ASN data |
| `s02-enrich` | `/etc/crowdsec/parsers/s02-enrich/whitelists.yaml` | **Drops private ranges** (`10/8`, `172.16/12`, `192.168/16`, `127/8`) |
| `s02-enrich` | `/etc/crowdsec/parsers/s02-enrich/public-dns-allowlist.yaml` | Never bans big public DNS resolvers |

A parser **doesn't decide what's bad**. It only makes every log look the same. Scenarios do the deciding.

> 💡 **Testing tip:** attacks from your home network are whitelisted and never banned. Always test from outside: mobile data or a cloud VM.
>
> 💡 **Don't lock yourself out:** whitelist the public IPs you use (office, VPN exit, etc.). Create `/etc/crowdsec/parsers/s02-enrich/my-whitelist.yaml`:
>
> ```yaml
> name: my/whitelist
> description: "My own public IPs"
> whitelist:
>   reason: "my own IPs"
>   ip:
>     - "198.51.100.7"
>   cidr:
>     - "198.51.100.0/24"
> ```
>
> Then run `sudo systemctl reload crowdsec`.

Browse more parsers on the Hub with `sudo cscli parsers list -a`.

#### Scenarios: patterns of bad behaviour

```bash
sudo cscli scenarios list
sudo less /etc/crowdsec/scenarios/ssh-bf.yaml
```

Most scenarios are **leaky buckets**: every IP gets its own bucket and every matching event adds a drop.

| Field | Example (`ssh-bf`) | Meaning |
|-------|--------------------|---------|
| `leakspeed` | `10s` | One drop leaks out every 10 seconds |
| `capacity` | `5` | More than 5 drops in a burst means **overflow, which raises an alert** |
| `blackhole` | `1m` | After an alert, stay quiet for this IP for 1 minute (no alert spam) |
| `reprocess` | `true` | Feed the overflow back in so other scenarios can react to it |

Slow attackers (say, one try every 15 s) never fill this bucket. That's what `ssh-slow-bf` is for: a bigger bucket that leaks much more slowly.

```bash
sudo cscli metrics show scenarios                 # how often each scenario fired
sudo cscli scenarios inspect crowdsecurity/ssh-bf # full description
sudo cscli scenarios list -a | less               # everything on the Hub
```

### 5. Test: SSH brute force

Start clean:

```bash
sudo cscli alerts delete --all
sudo cscli alerts list
```

On the server, watch two terminals:

```bash
sudo journalctl -u ssh -f            # on non-Ubuntu distros the unit may be "sshd"
```

```bash
sudo tail -f /var/log/crowdsec.log
```

From the **outside** machine, fail 12 SSH logins on purpose. The target must be your server's SSH as seen from the internet. If SSH isn't exposed (it shouldn't be!), forward a port temporarily for this test and remove it afterwards, or skip to the [web test](#8-expose-a-web-service).

```bash
for i in $(seq 1 12); do
  echo "SSH attempt $i"
  sshpass -p wrongpass ssh \
    -o StrictHostKeyChecking=accept-new \
    -o PubkeyAuthentication=no \
    -o PreferredAuthentications=password \
    -o NumberOfPasswordPrompts=1 \
    -o ConnectTimeout=5 \
    demo-attacker@<PUBLIC_IP> true
done
```

Back on the server:

```bash
sudo cscli metrics show scenarios    # the overflow counter went up
sudo cscli alerts list               # usually two alerts: ssh-bf and ssh-slow-bf
sudo cscli alerts inspect -d <ALERT_ID>
```

`inspect` shows the full story: attacker IP, scenario, every event that filled the bucket, plus the country and network owner added by enrichment.

### 6. Decisions

An alert says something bad happened. A **decision** is what actually gets enforced. Profiles create decisions from alerts.

```bash
sudo cscli decisions list
```

Your attacker is banned for 4 hours, with source `crowdsec` (your own engine). You can also add decisions by hand:

```bash
sudo cscli decisions add --ip 203.0.113.10 --type ban --duration 5m --reason "Manual test"
sudo cscli decisions list            # source: cscli
sudo cscli decisions delete --ip 203.0.113.10
```

> `203.0.113.0/24` is a documentation range (RFC 5737). It belongs to nobody, so it's safe for tests.

Decision types:
- **`ban`**: block the IP completely.
- **`captcha`**: show a challenge page instead. Only useful for websites, and only some bouncers support it (covered in Part 3).

**The crowd.** `decisions list` shows only your own decisions by default. See what the community blocklist gives you:

```bash
sudo cscli metrics show decisions
```

That's roughly 15,000 known-bad IPs (scanners, crawlers, brute forcers), already loaded. It works both ways: your engine anonymously reports the IPs it bans. One report alone doesn't ban an IP for everyone. Many independent machines have to agree, so nobody can poison the list.

### 7. Install a bouncer

```bash
sudo cscli bouncers list
```

Empty. **Right now CrowdSec blocks nothing.** Add the simplest bouncer: nftables on the host itself.

```bash
sudo apt install crowdsec-firewall-bouncer-nftables
sudo cscli bouncers list              # registered automatically
sudo nft list table ip crowdsec       # already full of community-blocklist IPs
```

Within about 10 seconds the bouncer pulls all decisions, including your SSH attacker from step 5.

> 🐳 **Docker note:** this bouncer hooks both the **input** and **forward** chains. Traffic to Docker containers goes through **forward**, not input, so a bouncer that only hooks input does **not** protect your containers.

**Prove the golden rule.** Stop the bouncer:

```bash
sudo systemctl stop crowdsec-firewall-bouncer
sudo nft list tables                  # the crowdsec table is gone
sudo cscli decisions list             # ...but every decision is still there
sudo systemctl start crowdsec-firewall-bouncer
```

The decision list says what to block. The bouncer does the blocking.

### 8. Expose a web service

Nobody should expose SSH to the internet anyway. So let's protect something that really is public: a web app.

#### DNS: plain A record, no proxy

Create an **A record** `whoami.example.com → <PUBLIC_IP>`. If you use Cloudflare, make sure the **proxy is OFF (grey cloud)**. With the orange cloud, your server only sees Cloudflare's IPs, never the attacker's real IP, and CrowdSec can't work. (Part 3 covers that case.)

```bash
host whoami.example.com
```

#### Router: forward TCP 443 to the host

Any router works. Forward **TCP 443** to the homelab. Only 443 is needed: the Caddyfile disables the HTTP-01 challenge, so Let's Encrypt validates over 443 (TLS-ALPN).

MikroTik example (adjust the interface and IP):

```routeros
/ip firewall nat add chain=dstnat in-interface=WAN protocol=tcp dst-port=443 action=dst-nat to-addresses=192.168.1.10 to-ports=443 comment="crowdsec-demo"
/ip firewall filter add chain=forward action=accept connection-nat-state=dstnat in-interface=WAN comment="crowdsec-demo"
```

Make sure the filter rule sits **above** your forward-chain drop rule.

#### The Docker stack

Edit the hostname in [`Caddyfile`](Caddyfile), then start the stack:

```bash
cd ~/crowdsec-demo
docker compose up -d
```

Why put Caddy in front of whoami? Three reasons:

1. **It writes a JSON access log**, which CrowdSec can parse.
2. **It returns 404 for every unknown path.** This matters: CrowdSec's web scenarios count 404s. `whoami` answers *every* path with 200 OK (as do many apps, or they redirect everything to a login page). Then a scanner looks exactly like a normal visitor. Returning 404 for junk paths gives CrowdSec a clear signal.
3. **Automatic HTTPS** from Let's Encrypt.

Test from the outside machine:

```bash
curl https://whoami.example.com
```

whoami prints the client IP it saw, which should be the attacker machine's real public IP. On the server:

```bash
sudo tail -n 3 ~/crowdsec-demo/logs/access.log | jq
```

> 🔎 **Why do scanners show up within minutes?** [Certificate Transparency](https://certificate.transparency.dev/). Every certificate is published in public logs, and scanners watch them. Want to keep subdomain names private? Use a **wildcard certificate**: only `*.example.com` shows up in the logs.

### 9. Teach CrowdSec to read Caddy logs

A **collection** bundles the parsers and scenarios for one service:

```bash
sudo cscli collections list -a
sudo cscli collections inspect crowdsecurity/caddy
sudo cscli collections install crowdsecurity/caddy
```

It includes the Caddy log parser and the web scenarios: probing, sensitive-file hunting, bad user agents, known exploits and more.

Now tell CrowdSec where the log is. Create `/etc/crowdsec/acquis.d/caddy-demo.yaml` (use your real home path):

```yaml
source: file
filenames:
  - /home/<you>/crowdsec-demo/logs/*.log
labels:
  type: caddy
```

Restart and check that lines are both **read** and **parsed**:

```bash
sudo systemctl restart crowdsec
curl https://whoami.example.com/      # from anywhere, to generate a log line
sudo cscli metrics show acquisition
```

> Lines read but **not parsed** almost always means a wrong `type` label or a log format that doesn't match the parser.

### 10. Attack: scan the site and watch the ban

On the server, watch the access log in a compact format (time, IP, path, status):

```bash
sudo tail -f -n 0 ~/crowdsec-demo/logs/access.log \
  | jq -r --unbuffered '[(.ts|floor|todate), .request.client_ip, .request.uri, .status] | @tsv'
```

On the **outside** machine, run `ffuf` against **your own** site:

```bash
ffuf -u https://whoami.example.com/FUZZ -w ~/wordlists/common.txt \
     -mc all -fc 404 -rate 20 -timeout 5 -maxtime 40
```

You'll see a wall of 404s... and then silence. ffuf keeps running, but every request now times out. Try the site again from the attacker:

```bash
curl --max-time 10 https://whoami.example.com/
```

No error page, no "connection refused". The packets are silently dropped. On the server:

```bash
sudo cscli alerts list                # e.g. http-crawl-non_statics, http-probing, http-sensitive-files
sudo cscli alerts inspect -d <ALERT_ID>
sudo cscli decisions list
sudo nft list set ip crowdsec crowdsec-blacklists-crowdsec
```

The attacker's IP is in the nftables set. CrowdSec read the log, caught the scan, and the bouncer dropped the IP, with no hand-written firewall rules.

Unban your test machine:

```bash
sudo cscli decisions delete --ip <ATTACKER_IP>
```

**Debug without banning anyone:** `cscli explain` replays a log through the parsers and scenarios and shows every step. It's very useful when you write your own parsers or scenarios.

```bash
sudo cscli explain --file ~/crowdsec-demo/logs/access.log --type caddy
```

### 11. Why some requests still got through

```bash
sudo jq -r 'select(.request.client_ip=="<ATTACKER_IP>") | .request.uri' ~/crowdsec-demo/logs/access.log | wc -l
```

Two delays add up:

1. **The bucket has to fill.** Detection happens *after* requests are logged.
2. **The bouncer polls on a timer.**

   ```bash
   sudo grep update_frequency /etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml   # default: 10s
   ```

The goal isn't to stop request #1. The goal is that the scanner doesn't get to try all ~4,000 paths. To stop request #1 you need a WAF (Part 4).

---

## Part 2: MikroTik bouncer, Jellyfin and blocklists

Part 1 works, but look at the path: the attacker's packets cross the router, enter the LAN, reach the server, and only *then* get dropped. Part 2 moves the wall to the router and makes three upgrades:

1. **Better detection.** To Caddy a wrong password is just a `401`. Jellyfin's own log knows the username and the real client IP, so CrowdSec reads **two logs**.
2. **A wall at the edge.** A second bouncer pushes the same decisions to a **MikroTik router** over the encrypted RouterOS API. Bans now protect every device behind the router, save bandwidth, and keep working even if the server-side bouncer is off.
3. **More threat intelligence.** Subscribe to a **curated blocklist** so known attackers are dropped before they ever knock.

```
          Internet: visitors, bots, scanners
                         │
                         ▼   jellyfin.example.com  →  A record  →  your public IP
┌──────────────────────── MikroTik router ─────────────────────────┐
│  RAW prerouting: drop src in "crowdsec-banned"  ◄───────┐        │
│  NAT: forward TCP 443 → homelab                         │        │
└────────────────────────┬────────────────────────────────┼────────┘
                         ▼                                │ api-ssl
┌────────────────────────── Homelab host ─────────────────┼────────┐
│                                                         │        │
│   nftables ◄── firewall bouncer ◄──┬──► RouterOS bouncer┘        │
│                                    │                             │
│                          CrowdSec Local API ◄── blocklists       │
│                                    ▲       (community + console) │
│   Caddy ──► logs/access.log ───────┤                             │
│     │                              │                             │
│     ▼                              │                             │
│   Jellyfin ──► config/log/*.log ───┘                             │
└──────────────────────────────────────────────────────────────────┘
```

Three layers: **two logs** for detection, **two firewalls** for enforcement, **two feeds** for intelligence.

### 12. Expose Jellyfin behind Caddy

Why Jellyfin? It's exactly the case for a [direct port forward](#what-to-expose-and-how): family watches on TVs and phones (a VPN is a pain), and Cloudflare's free plan doesn't allow video streaming.

Jellyfin runs in its own container on the same Docker network as Caddy, with **no public port**. Caddy is its only door. The [`Caddyfile`](Caddyfile) gets one new block that proxies to the `demo-jellyfin` container and writes to the **same access log**, so the Part 1 scanner detection covers Jellyfin too:

```caddyfile
jellyfin.example.com {
	log {
		output file /var/log/caddy/access.log
		format json
	}
	tls {
		issuer acme {
			disable_http_challenge
		}
	}
	reverse_proxy demo-jellyfin:8096
}
```

Edit the hostnames in [`Caddyfile`](Caddyfile) and in [`jellyfin/docker-compose.yaml`](jellyfin/docker-compose.yaml), then bring up Caddy first (it creates the shared `crowdsec_demo` network), then Jellyfin:

```bash
cd ~/crowdsec-demo
docker compose up -d

cd ~/crowdsec-demo/jellyfin
mkdir -p config cache media
docker compose up -d
```

Point DNS at your public IP (A record, no proxy) and make sure the router forwards TCP 443 to Caddy, as in [step 8](#8-expose-a-web-service):

```bash
host jellyfin.example.com
ssh <ROUTER_IP> '/ip firewall nat print where comment~"crowdsec-demo"'
```

Open `https://jellyfin.example.com` and finish the initial setup wizard.

### 13. The proxy trap: Known proxies

You'll fall into this every time you put an app behind a reverse proxy. Try one wrong password from the outside machine, then check Jellyfin's log:

```bash
grep denied ~/crowdsec-demo/jellyfin/config/log/log_*.log | tail -1
```

The IP is something like `172.16.x.x`. That's not the attacker, that's **Caddy** on the Docker network. The attacker talks to Caddy, Caddy talks to Jellyfin, so Jellyfin thinks Caddy is the client.

Caddy does pass the real IP in the `X-Forwarded-For` header, but Jellyfin ignores it unless it trusts the sender. Fix it in Jellyfin: **Dashboard → Networking → Known proxies**, add `demo-caddy` (the Caddy container name), save, then restart:

```bash
cd ~/crowdsec-demo/jellyfin
docker compose restart
```

Repeat the wrong password and check again: the log now shows the attacker's real public IP.

> ⚠️ This applies to almost every app behind a reverse proxy: Nextcloud, Immich, Home Assistant, Vaultwarden... **Always check that the app logs the real client IP**, otherwise CrowdSec bans your proxy (which is whitelisted as a private IP, so in practice: nothing gets banned at all).

### 14. Teach CrowdSec to read Jellyfin's log

There's a Hub collection for it, with a parser for Jellyfin's log and two scenarios: **brute-force logins** and **user enumeration**.

```bash
sudo cscli collections list -a | grep jellyfin
sudo cscli collections install LePresidente/jellyfin
sudo cscli collections list
```

Right now CrowdSec reads only Caddy's log:

```bash
sudo cscli metrics show acquisition
```

Add Jellyfin's log folder. Jellyfin starts a new file every day, so use a wildcard. Create `/etc/crowdsec/acquis.d/jellyfin-demo.yaml` (use your real home path):

```yaml
source: file
filenames:
  - /home/<you>/crowdsec-demo/jellyfin/config/log/log_*.log
labels:
  type: jellyfin
```

Restart, try one bad password, and the counter ticks:

```bash
sudo systemctl restart crowdsec
sudo cscli metrics show acquisition
```

Two logs, two witnesses. The recipe works for any app: put a reverse proxy in front (every request gets analysed), then add the app's collection (WordPress, Nextcloud, Grafana, GitLab, Home Assistant...) so CrowdSec also reads the application's own log.

### 15. Why block on the router, and why in RAW

The nftables bouncer from Part 1 already blocks the attacker on the server:

```bash
sudo cscli bouncers list
```

Blocking on the router as well gives you four things:

1. It protects **everything behind the router**: every device, every port, every protocol.
2. It **saves bandwidth**: attack traffic never enters the LAN.
3. **Defense in depth**: if the nftables bouncer is ever off, the router keeps blocking.
4. The **community blocklist** now protects the whole network, not just the homelab.

**Where** on the router matters (same on nftables, iptables or any Linux-based firewall). An incoming packet goes:

```
RAW (prerouting) ──► connection tracking ──► NAT (dst-nat / port forward) ──► filter
```

Drop a banned IP in **filter** and the router has already tracked the connection, done NAT and spent a conntrack entry on a packet it's about to throw away. With tens of thousands of banned IPs, that's a lot of junk. Drop it in **RAW** and it's gone at the first step: no tracking, no NAT, no table entry. The catch: RAW knows nothing about connection state, but for a ban that doesn't matter.

### 16. Prepare the router: encrypted API and a restricted user

The bouncer talks to the router through the **RouterOS API**. Enable it **encrypted only** (`api-ssl`), and give the bouncer a user that can do as little as possible. In the router terminal:

Create a small CA on the router and sign an API certificate with it:

```routeros
/certificate add name=crowdsec-ca common-name=crowdsec-ca key-usage=key-cert-sign,crl-sign days-valid=3650
/certificate sign crowdsec-ca
/certificate add name=crowdsec-api common-name=<ROUTER_IP> subject-alt-name=IP:<ROUTER_IP> key-usage=digital-signature,key-encipherment,tls-server days-valid=3650
/certificate sign crowdsec-api ca=crowdsec-ca
/certificate print where name~"crowdsec"
```

Enable `api-ssl`, reachable only from the homelab host:

```routeros
/ip service set api-ssl certificate=crowdsec-api available-from=192.168.1.10/32 disabled=no
```

> The bouncer compose file uses port `65432`. If you keep the default api-ssl port (`8729`), change `MIKROTIK_HOST` accordingly, or move the service with `port=65432`.

A user group that can only read, write and use the API. No SSH, no Winbox, no reboot, no password changes:

```routeros
/user group add name=crowdsec policy=read,write,api,!sensitive,!ftp,!local,!ssh,!reboot,!policy,!test,!password,!sniff,!romon,!rest-api comment="crowdsec-routeros-bouncer"
```

And a user in that group that can **only log in from the homelab**, so a stolen password is useless anywhere else:

```routeros
/user add name=crowdsec group=crowdsec password="<router-password>" address=192.168.1.10/32 comment="crowdsec-routeros-bouncer (home lab)"
```

### 17. Run the MikroTik bouncer

The bouncer is [cs-routeros-bouncer](https://github.com/jmrplens/cs-routeros-bouncer), running as a container on the homelab host. Register it with CrowdSec, which prints an API key:

```bash
sudo cscli bouncers add mikrotik-routeros
```

Put the key and the router password in `.env`:

```bash
cd ~/crowdsec-demo/routeros-bouncer
cp .env.example .env
vi .env            # MIKROTIK_PASS=...  CROWDSEC_BOUNCER_API_KEY=...
chmod 600 .env
```

Check [`routeros-bouncer/docker-compose.yaml`](routeros-bouncer/docker-compose.yaml) and set `MIKROTIK_HOST` to your router. Notes:

- **Host networking**, because the CrowdSec Local API only listens on `127.0.0.1:8080`.
- It connects over **TLS** as the `crowdsec` user.
- `MIKROTIK_TLS_INSECURE=true`: traffic is encrypted, but the router's self-signed certificate isn't verified. At home, with the API locked to one IP, that's acceptable. For full verification, make the bouncer trust your router's CA.

Start it and watch the log:

```bash
docker compose up -d
docker logs -f crowdsec-routeros-bouncer
```

It creates its firewall rules, collects the existing bans (community blocklist, ~16,000 IPs) and pushes them to the router in a few seconds. On the router:

```routeros
/ip firewall raw print stats where comment~"crowdsec-bouncer"
/ip firewall address-list print count-only where list=crowdsec-banned
/ipv6 firewall raw print stats where comment~"crowdsec-bouncer"
/ipv6 firewall address-list print count-only where list=crowdsec6-banned
```

Two rules at the top of RAW prerouting: the first only **counts** traffic (so the bouncer can report what it saw and dropped), the second is the actual **drop**. They match against an **address list**. That's not 16,000 rules, it's one rule checking a set: one lookup, same speed with 16 bans or 16,000. The bouncer also adds the same pair in the filter input chain as a second net, and handles IPv6 with a separate list.

Back in CrowdSec you now see two bouncers, with packets processed and dropped by the router:

```bash
sudo cscli metrics show bouncers
```

**How fast does a ban reach the router?** Add one by hand and watch the address list:

```bash
sudo cscli decisions add -i 203.0.113.51 -d 5m -R "routeros bouncer test"
```

```routeros
/ip firewall address-list print where list=crowdsec-banned address=203.0.113.51
```

The bouncer polls every ~10 seconds, so the IP shows up within that window. Remove it, and it's gone from the router too:

```bash
sudo cscli decisions delete -i 203.0.113.51
```

> On a clean stop (`docker compose stop`) the bouncer removes its rules, so the router blocks nothing until it starts again. Address-list entries keep their timeout and expire on their own. On start, and every 15 minutes, it reconciles the router's list with CrowdSec's decisions, so bans made while it was down (or entries deleted by hand on the router) are fixed automatically. That's why you keep the nftables bouncer too.

### 18. Subscribe to a curated blocklist

Those ~16,000 bans came from the **community blocklist**: every engine reports what it catches, CrowdSec de-duplicates it and hands it back to everyone. No account, on by default. But it's general purpose.

**Curated blocklists** target what's being exploited *right now* (Tor exit nodes, open proxies, botnets, individual CVE campaigns). And a blocklist isn't a new mechanism: it's just **decisions** (IP, reason, duration) landing in the same Local API, picked up by the same bouncers, dropped by the same single RAW rule.

In the [CrowdSec Console](https://app.crowdsec.net) (enrolled in [step 2](#2-optional-enroll-in-the-crowdsec-console)): **Blocklists → Add blocklists**, pick one and **Subscribe**. The video uses the **React2Shell** campaign list (~50,000 IPs caught sending that exact exploit, refreshed as the campaign moves). Your app doesn't need to be React: scanners spray the payload at everything.

- The free (community) plan allows **up to 3 blocklists**.
- A new subscription can take **up to 2 hours** to show up (refresh cycle). Just wait.

```bash
sudo cscli console status                              # authenticated with CAPI and enrolled
sudo cscli metrics show decisions                      # origins: "lists" (your subscriptions) and CAPI (community)
sudo cscli decisions list --origin lists --limit 10    # reason = list name
```

```routeros
/ip firewall address-list print count-only where list=crowdsec-banned
```

From ~16,000 to ~73,000 (a bit less than the sum, since lists overlap). Nobody touched RouterOS and the bouncer didn't restart: it polled, saw new decisions, and added them.

```bash
docker logs --tail 20 crowdsec-routeros-bouncer
```

> A blocklist is **not a patch**. If you run Next.js, update it. CrowdSec buys you time, it doesn't fix the bug.

### 19. Attack: brute-force Jellyfin and get banned at the edge

On the **outside** machine, note its public IP:

```bash
curl -4 ifconfig.me
```

On the server, make sure it isn't already banned (if it's on a blocklist you'd be testing intelligence, not detection; use a different VM):

```bash
sudo cscli decisions list -i <ATTACKER_IP>
```

In the attacker's browser, open `https://jellyfin.example.com`, user `admin`, and enter a wrong password **8 times, quickly**. The scenario is a leaky bucket, so slow attempts drain out and never trip it. Then the site stops responding: no error, the packets are just dropped.

```bash
sudo cscli alerts list --limit 3       # reason: the Jellyfin brute-force scenario, not Caddy
sudo cscli decisions list -i <ATTACKER_IP>
```

```routeros
/ip firewall address-list print where list=crowdsec-banned address=<ATTACKER_IP>
```

Under 10 seconds from decision to edge. Eight wrong passwords in one app, and the attacker is locked out of the **entire network**. Unban:

```bash
sudo cscli decisions delete -i <ATTACKER_IP>
```

---

## What to expose, and how

Rule #1 of self-hosting: **don't expose anything you don't need to.** When you do need remote access, pick a tier per service:

| Tier | How | Good for | Trade-offs |
|------|-----|----------|------------|
| **1. VPN only** | WireGuard (or Tailscale / ZeroTier) | SSH, router admin, databases, dashboards, Jupyter | Invisible to scanners: WireGuard doesn't answer without a valid key. Every device needs a client, so it's awkward for family and friends |
| **2. Cloudflare Tunnel** (+ optional Cloudflare Access) | No open ports, home IP hidden | Home Assistant, Open WebUI, SSO, ntfy, SearXNG | Cloudflare decrypts TLS at the edge. Free plan: 100 MB upload limit, no heavy video streaming, HTTP(S) only |
| **3. Direct port forward** | What this lab does | Big uploads (S3 / backups), video streaming (Jellyfin), non-HTTP services (game, mail) | Real client IP (CrowdSec works best), but your IP is public, gets scanned within minutes, and patching is all on you |

Think of it as an onion: the VPN is the hidden core, Cloudflare the protected middle, and direct forwarding the exposed outer layer. The further out a service sits, the more active defense (CrowdSec, fast patching) it needs.

---

## Troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| Attack from home never gets banned | Private ranges are whitelisted. Test from mobile data or a cloud VM |
| Decisions exist but nothing is blocked | No bouncer installed or running: `sudo cscli bouncers list`, `systemctl status crowdsec-firewall-bouncer` |
| Docker services not protected | Bouncer must hook the **forward** chain (the nftables bouncer does by default) |
| Acquisition shows lines read but not parsed | Wrong `type` label, or log format doesn't match. Use `cscli explain` |
| CrowdSec sees Cloudflare IPs, not real clients | DNS record is proxied (orange cloud). Use a grey cloud for this lab |
| Scanner never triggers web scenarios | App returns 200 for everything. Return 404 for unknown paths (see `Caddyfile`) |
| Console looks empty | Private IPs aren't sent; free plan keeps only 500 alerts |
| Caddy can't get a certificate | Port 443 not forwarded, or DNS doesn't point at your public IP yet |
| Jellyfin log shows `172.x.x.x` instead of the attacker IP | Caddy isn't a Known proxy in Jellyfin. Add `demo-caddy` under Dashboard → Networking, restart Jellyfin |
| Jellyfin brute force never triggers | Attempts too slow (leaky bucket drains), or the acquisition path / `type: jellyfin` is wrong: `sudo cscli metrics show acquisition` |
| RouterOS bouncer can't connect | `api-ssl` disabled, wrong port in `MIKROTIK_HOST`, or `available-from` / the user's `address=` doesn't match the homelab IP |
| Router rules vanished | The bouncer was stopped cleanly and removed them. `docker compose start` recreates them |
| New blocklist not showing up | Subscriptions can take up to 2 hours. Check `sudo cscli console status` |

---

## cscli cheat sheet

```bash
sudo cscli lapi status / capi status          # API health
sudo cscli console status                      # console enrollment
sudo cscli metrics show acquisition            # which logs are read and parsed
sudo cscli metrics show scenarios              # how often scenarios fired
sudo cscli metrics show decisions              # decisions by origin (incl. community list)
sudo cscli parsers|scenarios|collections list [-a]
sudo cscli collections install crowdsecurity/caddy
sudo cscli alerts list
sudo cscli alerts inspect -d <ID>
sudo cscli alerts delete --all
sudo cscli decisions list
sudo cscli decisions add --ip <IP> --type ban --duration 5m --reason "test"
sudo cscli decisions delete --ip <IP>
sudo cscli bouncers list
sudo cscli explain --file <log> --type <type>  # dry-run the pipeline
sudo cscli bouncers add <name>                 # register a bouncer, prints its API key
sudo cscli metrics show bouncers               # per-bouncer traffic (processed / dropped)
sudo cscli decisions list --origin lists       # decisions from blocklist subscriptions
```

RouterOS (Part 2):

```routeros
/ip firewall raw print stats where comment~"crowdsec-bouncer"
/ip firewall address-list print count-only where list=crowdsec-banned
/ipv6 firewall address-list print count-only where list=crowdsec6-banned
```

---

## Clean up

```bash
# Part 2 stacks (stopping the bouncer cleanly removes its router rules)
cd ~/crowdsec-demo/routeros-bouncer && docker compose down
cd ~/crowdsec-demo/jellyfin && docker compose down
sudo cscli bouncers delete mikrotik-routeros
sudo rm /etc/crowdsec/acquis.d/jellyfin-demo.yaml
sudo cscli collections remove LePresidente/jellyfin

# Demo stack (-v also removes the Caddy certificate volume)
cd ~/crowdsec-demo && docker compose down -v

# CrowdSec and the bouncer
sudo apt purge crowdsec-firewall-bouncer-nftables crowdsec
sudo rm -rf /etc/crowdsec /var/lib/crowdsec /var/log/crowdsec*.log
```

Remove the router port-forward (MikroTik example):

```routeros
/ip firewall nat remove [find comment="crowdsec-demo"]
/ip firewall filter remove [find comment="crowdsec-demo"]

# Part 2: bouncer user, API and certificates
/ip firewall address-list remove [find list=crowdsec-banned]
/ipv6 firewall address-list remove [find list=crowdsec6-banned]
/user remove [find name=crowdsec]
/user group remove [find name=crowdsec]
/ip service set api-ssl disabled=yes certificate=none
/certificate remove [find name~"crowdsec"]
```

Also delete the DNS records, unsubscribe from blocklists, and remove the engine in the CrowdSec Console if you enrolled one.

---

**Next up, Part 3:** apps exposed over a Cloudflare Tunnel, where the server no longer sees the real client IP, protected by the Cloudflare bouncer blocking traffic right at Cloudflare's edge.
