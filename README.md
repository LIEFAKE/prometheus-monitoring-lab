# Prometheus Monitoring Lab

A self-directed home lab where I built a small monitoring stack from scratch with Docker Compose. I wanted to learn the full path from a metric to an alert: collecting host metrics, writing PromQL, alert rules, routing alerts with Alertmanager and showing everything in Grafana.

> This is a home lab, not a production setup.

## Stack

| Service | Version | Port | What it does |
|---|---|---|---|
| Prometheus | 2.55.1 | 9090 | Scrapes metrics and evaluates alert rules |
| node_exporter | 1.8.2 | 9100 | Exposes host metrics (CPU, memory, disk, network). The host filesystem is mounted read-only |
| Alertmanager | 0.27.0 | 9093 | Groups alerts, applies the inhibit rule, sends notifications to Telegram |
| Grafana | 11.3.0 | 3030 | Dashboards. Data source and dashboards are provisioned as code |

## How it works

```
node_exporter --> Prometheus --> Alertmanager --> Telegram
                      |
                      +--> Grafana (dashboards)
```

- Prometheus scrapes itself and node_exporter every 15 seconds.
- Alert rules are evaluated every 15 seconds. When a rule is true for its `for` duration, the alert goes to Alertmanager.
- Alertmanager groups alerts by `alertname` and `instance` and sends them to a Telegram bot. The bot token is read from a mounted file, not written in the config.
- Grafana gets the Prometheus data source and the dashboards from files on start, so nothing is set up by hand in the UI.

## Alerts

| Alert | Condition | For | Severity |
|---|---|---|---|
| TargetDown | `up == 0` | 1m | critical |
| HighCpuUsage | CPU usage above 80% (from `node_cpu_seconds_total`, idle mode, `rate` over 5m) | 5m | warning |
| DiskAlmostFull | Less than 15% free space on `/` | 5m | warning |

**Inhibit rule:** if a critical alert is firing for an instance, warning alerts for the same instance are not sent. For example, when the target is down there is no point in also getting CPU or disk warnings for it.

## Quick start

```bash
git clone <this repo>
cd <repo>

cp .env.example .env                       # set the Grafana admin user and password
cp telegram_token.example telegram_token   # put your Telegram bot token here
# set your chat_id in alertmanager.yml

docker compose up -d
```

Then open:

- Prometheus: http://localhost:9090
- Alertmanager: http://localhost:9093
- Grafana: http://localhost:3030

## Dashboards

- **Node Exporter Full**: the standard community dashboard for host metrics.
- **My CPU dashboard**: a custom dashboard I made, CPU usage with an 80% threshold line, same as the alert.

Both are loaded from `grafana/dashboards/`. I checked that they are still there after the Grafana container is re-created.

## How I tested it

- Validated the configs before starting: `promtool check config`, `promtool check rules` and `amtool check-config`.
- **TargetDown:** stopped the node_exporter container. After about a minute the alert became firing and the message came to Telegram. After starting it again I got the resolved message.
- **HighCpuUsage:** loaded all CPU cores on the host with `yes`:
  ```bash
  for i in $(seq $(nproc)); do yes > /dev/null & done
  ```
  CPU went to ~100%, the alert stayed pending for 5 minutes, then became firing and came to Telegram. Then I stopped all `yes` processes.
- **Inhibit rule:** I could not just stop node_exporter for this test, because then there are no disk metrics and the warning never fires. So I temporarily changed TargetDown to fire on a live target and lowered the DiskAlmostFull threshold, both with `for: 30s`. Both alerts were firing in Prometheus, but in Alertmanager only TargetDown was visible; DiskAlmostFull showed up only with the **Inhibited** filter. Then I put the rules back.

## What I learned

- How Prometheus pulls metrics and why `rate()` is needed for counters like CPU seconds.
- Why alerts need a `for` duration, so short spikes do not wake anyone up.
- Grouping and inhibition in Alertmanager to avoid alert noise.
- Keeping secrets out of the repo: the bot token and Grafana password live in files listed in `.gitignore`, with `.example` files instead.
