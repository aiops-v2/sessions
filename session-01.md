# Session 01 — Why Metrics, Pull vs Push, Exporters & Prometheus Config

Nov 16, 2026 · Sivakumar Reddy

## Session overview

By the end of this 90-minute session, students can explain why we collect metrics, why Prometheus pulls instead of waiting for pushes, what an exporter does, and how every line of the default `prometheus.yml` and a `relabel_configs` rule works.

**Prerequisite state:** Prometheus 3.x on the monitoring VM (9090), node_exporter on all 3 VMs (9100), mysqld_exporter (9104), nginx-prometheus-exporter (9113), backend prom-client (9101), blackbox (9115).

| Time | Block | Mode |
| --- | --- | --- |
| 08:00–08:15 | Why metrics — the human body analogy | Whiteboard talk |
| 08:15–08:35 | Push vs pull | Talk + 1 live break |
| 08:35–08:55 | Exporters — what, why, kinds | Live `curl /metrics` |
| 08:55–09:10 | Default `prometheus.yml` line by line | Screen share |
| 09:10–09:25 | `relabel_configs` approach | Live edit + reload |
| 09:25–09:30 | Recap + homework | Talk |

## 1. Why metrics — the human body analogy

A metric is a number measured over time, so you can tell "normal" from "sick" before the patient collapses. Doctors don't wait for a heart attack; they check BP, sugar and pulse regularly. Servers need the same.

**Opening question to the class:** "When you go for a health check-up, what does the doctor measure first, before asking anything?" Let them answer: BP, pulse, temperature, sugar, weight, SpO2.

### Body metric → server metric

| Body metric | Normal range | What it tells the doctor | Server equivalent | Example metric in our lab |
| --- | --- | --- | --- | --- |
| Pulse (beats/min) | 60–100 | How hard the heart is working right now | Request rate / CPU busy | `rate(http_requests_total[5m])`, `node_cpu_seconds_total` |
| Temperature (°F) | 97–99 | Something is wrong somewhere (symptom, not cause) | Load average | `node_load1` |
| Blood pressure (mmHg) | 120/80 | Pressure inside the pipes | Latency, queue, saturation | `http_request_duration_seconds` |
| Blood sugar (mg/dL) | 70–140 | Fuel available vs used | Memory available | `node_memory_MemAvailable_bytes` |
| SpO2 (%) | 95–100 | Oxygen headroom left | Free disk, free DB connections | `node_filesystem_avail_bytes`, `mysql_global_status_threads_connected` |
| Weight trend (kg/month) | stable | Slow problem building up | Disk growth over days | `predict_linear(node_filesystem_avail_bytes[6h], 4*3600)` |
| Heartbeat present? | yes | Alive or not | Is the target reachable | `up` |

### Five teaching points from the analogy

1. **One reading means nothing; the trend matters.** BP 140 once after climbing stairs is fine; 140 every day for a month is hypertension. That's why Prometheus stores a time series, not just the latest value.
2. **Normal range = threshold.** The doctor knows 120/80 is normal. Our alerts need the same: CPU above 90% for 5 minutes, disk above 90%.
3. **Labels = patient details.** "BP 140" is useless without patient name, ward and bed number. In Prometheus that's `job`, `instance`, `tier`.
4. **Metrics tell you WHAT, not WHY.** Fever says something is wrong; the blood test (logs) and scan (traces) tell you why. This sets up logs and traces later in the course.
5. **Two kinds of numbers.** Body temperature goes up and down: a **gauge**. Total steps walked today only goes up (resets at midnight): a **counter**. Same split in Prometheus: `node_load1` is a gauge, `http_requests_total` is a counter.

**Line to land:** "Without metrics, your first monitoring tool is the customer calling to say the site is down."

## 2. Push vs pull

Prometheus uses **pull**: it visits each target every `scrape_interval` and reads its `/metrics` page. Push systems wait for targets to send data. The key difference is what silence means.

### The hospital analogy

- **Pull = nurse rounds.** Every 15 minutes the nurse walks to each bed and checks vitals. If a patient doesn't respond, the nurse knows immediately and raises the alarm.
- **Push = patient calls the nurse.** The patient presses the bell when they feel unwell. But an unconscious patient can't press the bell. Silence could mean "healthy" or "collapsed" — the nurse can't tell.

That is the most important point: **with pull, a dead target shows up as `up == 0`. With push, a dead target just goes quiet.**

### Comparison

| Question | Pull (Prometheus) | Push (StatsD, Graphite, InfluxDB line protocol, OTLP) |
| --- | --- | --- |
| Who starts the connection? | Prometheus server | The application/agent |
| Who decides how often? | Server (`scrape_interval`) | Each client |
| Target is down — how do we know? | Scrape fails → `up = 0` automatically | Data stops; needs separate "absence" logic |
| Where is the list of targets? | One place: `prometheus.yml` / service discovery | Spread across every client's config |
| Can I debug by hand? | Yes: `curl host:9100/metrics` | Harder; need to capture outgoing traffic |
| Server overload risk | Low — server controls the rate | A spike of clients can flood the server |
| Two Prometheus servers for HA? | Both just scrape the same target | Client must send to both |
| Works behind NAT/firewall? | Server must reach the target | Yes, client only needs outbound access |
| Short-lived jobs (run 10 seconds)? | May finish before the next scrape | Good fit |

### Why pull is useful (for us)

1. Free health check for every target: the `up` metric, with no extra code.
2. Central control: change the interval or add a target in one file, then reload.
3. Easy to debug: every exporter is just a web page you can `curl`.
4. The app stays simple: it doesn't need to know where Prometheus lives.

### When push is used

1. **Short-lived batch jobs** (a nightly DB backup cron, a CI job): they push to **Pushgateway**, and Prometheus scrapes the Pushgateway. Warning: metrics stay there until deleted, and `up` reflects the Pushgateway, not the job.
2. **Targets Prometheus can't reach** (behind NAT, edge sites, IoT devices): a local Prometheus in agent mode or an OTel Collector pushes using `remote_write` / OTLP.
3. **Events, logs and traces**: these are naturally pushed. In our own stack, Promtail pushes logs to Loki and the OTel SDK pushes traces to the Collector.
4. **Long-term storage**: Prometheus pushes to Mimir/Thanos via `remote_write`.

**Line to land:** "Pull for the long-running services you own; push only when the target can't be reached or doesn't live long enough to be scraped."

### Live break (2 minutes)

1. On the backend VM: `sudo systemctl stop node_exporter`
2. In the Prometheus UI, run `up{job="node"}` — backend drops to 0 within 15 seconds.
3. Open **Status → Target health** and read the error first: `connection refused`.
4. Ask: "With a push system, what would we have seen?" Answer: nothing at all.
5. Restore: `sudo systemctl start node_exporter`

## 3. Exporters

An exporter is a small translator program. It reads numbers from a system that doesn't speak Prometheus, such as the Linux kernel, MySQL or nginx, and shows them on an HTTP `/metrics` page in Prometheus text format.

### Why we need them

Linux keeps CPU numbers in `/proc/stat`. MySQL keeps them behind `SHOW GLOBAL STATUS`. nginx shows them on `stub_status`. Prometheus can't read any of these formats directly, and we can't change the code of MySQL or the kernel. So we run a translator next to each one.

**Analogy:** the patient speaks only Telugu and the doctor reads only English charts. The ward assistant (the exporter) takes the reading from the BP machine and writes it on the chart in the format the doctor understands. The assistant doesn't diagnose anything; it only reads and translates.

Point to stress: **an exporter is a pure reader.** node_exporter doesn't calculate CPU %; it copies the kernel's counters. The calculation happens later in PromQL.

### What `/metrics` looks like

```
# HELP node_load1 1m load average.
# TYPE node_load1 gauge
node_load1 0.21
# HELP node_cpu_seconds_total Seconds the CPUs spent in each mode.
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 18234.55
node_cpu_seconds_total{cpu="0",mode="user"} 412.08
```

Each line is: metric name, labels in `{}`, value. `HELP` describes it, `TYPE` says counter, gauge, histogram or summary.

### Kinds of exporters

| Kind | What it watches | Examples | Default port | In our lab? |
| --- | --- | --- | --- | --- |
| Host / OS | CPU, memory, disk, network | node_exporter, windows_exporter | 9100 / 9182 | Yes — all 3 VMs |
| Database | Connections, queries, replication | mysqld_exporter, postgres_exporter, redis_exporter | 9104 / 9187 / 9121 | Yes — mysqld_exporter |
| Web server / proxy | Requests, active connections | nginx-prometheus-exporter, apache_exporter | 9113 / 9117 | Yes — nginx |
| Probe (outside-in) | Is the URL/port/DNS answering, cert expiry | blackbox_exporter | 9115 | Yes |
| Containers | Per-container CPU/memory | cAdvisor | 8080 | Later (Docker phase) |
| Cloud | AWS service metrics from CloudWatch | cloudwatch_exporter, YACE | 9106 / 5000 | No |
| Runtime / JVM | Heap, GC, threads | jmx_exporter | varies | No |
| Messaging | Topic lag, consumers | kafka_exporter | 9308 | No |

Two cases that are **not** exporters but expose the same format:

- **Direct instrumentation:** our backend uses `prom-client` to expose its own metrics on 9101. The app is its own exporter. This is the only way to get business metrics like `expenses_created_total`.
- **Prometheus itself** exposes `/metrics` on 9090. That's why the default config scrapes `localhost:9090`.

### Inside-out vs outside-in

- **Inside-out** (node, mysqld, nginx exporters): the doctor checks vitals from inside the body. Tells you *why* something is slow.
- **Outside-in** (blackbox): someone rings the doorbell to see if anyone answers. Tells you *what the user sees*. You need both: all internals can look healthy while the ALB or DNS is broken.

### Live demo

Run from the monitoring VM:

```
curl -s http://frontend.daws90s.shop:9100/metrics | grep -E '^node_load'
curl -s http://mysql.daws90s.shop:9104/metrics | grep mysql_up
curl -s http://frontend.daws90s.shop:9113/metrics | grep nginx_connections_active
curl -s http://backend.daws90s.shop:9101/metrics | grep http_requests_total
```

Ask: "`mysql_up` is 1 and `up` for the mysql job is 1. Are these the same thing?" No: `up` means the exporter answered; `mysql_up` means the exporter could log into MySQL. **UP ≠ healthy.**

## 4. The default prometheus.yml

The file that ships in the Prometheus tarball has four top-level blocks: `global`, `alerting`, `rule_files` and `scrape_configs`. Out of the box it scrapes exactly one target: Prometheus itself.

First show which file is live: `systemctl cat prometheus | grep config.file`. Editing `/opt/prometheus/prometheus.yml` does nothing if the service reads `/etc/prometheus/prometheus.yml`.

```yaml
# my global config
global:
  scrape_interval: 15s     # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]
        # The label name is added as a label `label_name=<label_value>` to any timeseries scraped from this config.
        labels:
          app: "prometheus"
```

The `labels: app` block appears in the 3.x sample file; older 2.x files stop at `targets`. Open your own copy with `cat` and compare it during class.

### Line by line

| Setting | Value in file | Built-in default if removed | Meaning (hospital version) |
| --- | --- | --- | --- |
| `global.scrape_interval` | 15s | 1m | How often the nurse does rounds |
| `global.evaluation_interval` | 15s | 1m | How often the doctor reviews the chart rules (recording + alerting rules) |
| `global.scrape_timeout` | not set | 10s | How long the nurse waits at a bed before marking "no response"; must be ≤ scrape_interval |
| `alerting.alertmanagers` | target commented out | none | Who to phone when a rule fires. Empty = alerts fire in the UI but nobody is notified |
| `rule_files` | commented out | none | Files with recording and alerting rules |
| `scrape_configs[].job_name` | prometheus | required | A group of similar targets; becomes the `job` label |
| `metrics_path` | not set | /metrics | URL path on the target |
| `scheme` | not set | http | http or https |
| `static_configs[].targets` | localhost:9090 | required | host:port list; becomes `__address__`, then `instance` |
| `static_configs[].labels` | app: prometheus | none | Extra labels stuck on every series from these targets |

### Things to make students notice

1. **Per-job overrides.** Any job can set its own `scrape_interval`, `scrape_timeout`, `metrics_path` or `scheme`. The job value beats the global value.
2. **Where `job` and `instance` come from.** Run `up` in the UI. You'll see `{job="prometheus", instance="localhost:9090", app="prometheus"}`. Nobody wrote `instance` — Prometheus derived it from the target.
3. **Free metrics per target.** Prometheus adds `up`, `scrape_duration_seconds`, `scrape_samples_scraped` and `scrape_series_added` for every target, with no exporter.
4. **Validate, then reload.** `promtool check config /etc/prometheus/prometheus.yml`, then `curl -X POST http://localhost:9090/-/reload` (works because we started with `--web.enable-lifecycle`).
5. **YAML indentation breaks silently in vim paste.** Write config with a heredoc or `:set paste`.

### From default to our lab

We already added these jobs to the default file:

| job | Target(s) | Port | Extra |
| --- | --- | --- | --- |
| prometheus | localhost | 9090 | default job |
| node | frontend / backend / mysql.daws90s.shop | 9100 | `tier` label per target |
| nginx | frontend.daws90s.shop | 9113 | — |
| backend | backend.daws90s.shop | 9101 | prom-client app metrics |
| mysql | mysql.daws90s.shop | 9104 | — |
| blackbox | ALB URL, DNS, TCP | 9115 | needs `relabel_configs` (next section) |

## 5. The relabel_configs approach

Prometheus treats every target as a bag of labels. `relabel_configs` is a list of rules that edit that bag *before* the scrape: decide whether to scrape it, where to connect, and which labels to keep.

**Analogy:** before the nurse's rounds, the reception desk edits each patient's file. They write the ward number, cross out patients discharged today, and correct the bed number. The nurse then simply follows the file. `metric_relabel_configs` is the clerk who filters the nurse's report *after* the readings are taken.

### The one rule

- Label name starts with `__` → a **scrape setting**. Used to build the URL, then thrown away.
- Label name without `__` → **stored** on every series from that target.

The URL is built from these reserved labels:

```
__scheme__ :// __address__ __metrics_path__ ? __param_<name>=<value>
   http    :// frontend.daws90s.shop:9100 /metrics
```

### What happens to one target, in order

1. Prometheus reads `targets:` and `labels:` into a label set: `__address__`, `__scheme__`, `__metrics_path__`, `job`, plus your labels. These are the *Discovered labels*.
2. Each `relabel_configs` rule runs, top to bottom. Each rule sees the result of the previous one.
3. If a `keep`/`drop` rule removes the target, it is never scraped.
4. If no rule set `instance`, Prometheus copies the final `__address__` into `instance`.
5. All `__` labels are deleted. What's left are the *Target labels*.
6. Prometheus scrapes the URL.
7. `metric_relabel_configs` runs on every returned sample, then samples are stored.

Show this on screen: **Status → Service discovery** shows Discovered labels on the left and Target labels on the right. This one page explains relabeling better than any slide.

### Fields of one rule

| Field | Default | Meaning |
| --- | --- | --- |
| `source_labels` | none | Labels to read; values joined with `;` |
| `regex` | `(.*)` | Pattern to match. Always anchored: must match the whole string |
| `target_label` | none | Label to write |
| `replacement` | `$1` | Value to write; `$1`, `$2` are regex groups |
| `action` | `replace` | `replace`, `keep`, `drop`, `labelmap`, `labeldrop`, `labelkeep`, `hashmod`, `lowercase`, `uppercase` |

### Example 1 — derive tier and a clean instance from the hostname

Today we hand-write `tier` per target. With relabeling, the hostname gives it to us.

```yaml
- job_name: node
  static_configs:
    - targets:
        - frontend.daws90s.shop:9100
        - backend.daws90s.shop:9100
        - mysql.daws90s.shop:9100
  relabel_configs:
    # frontend.daws90s.shop:9100  ->  tier="frontend"
    - source_labels: [__address__]
      regex: '([^.]+)\..*'
      target_label: tier
      replacement: '$1'
    # frontend.daws90s.shop:9100  ->  instance="frontend.daws90s.shop"
    - source_labels: [__address__]
      regex: '([^:]+):\d+'
      target_label: instance
      replacement: '$1'
```

| Label | Before relabeling | After relabeling |
| --- | --- | --- |
| `__address__` | frontend.daws90s.shop:9100 | (used to connect, then dropped) |
| `job` | node | node |
| `tier` | — | frontend |
| `instance` | — | frontend.daws90s.shop |

Automation note for later: inside a Terraform `templatefile()`, `$1` style is safe but `${1}` is interpreted by Terraform. Write `$${1}` or avoid braces.

### Example 2 — blackbox: point the scrape at the exporter

The target is a website, but Prometheus must connect to the blackbox exporter and pass the website as a URL parameter.

```yaml
- job_name: blackbox-https
  metrics_path: /probe
  params:
    module: [https_2xx]
  static_configs:
    - targets: [https://expense.daws90s.shop]
  relabel_configs:
    - source_labels: [__address__]     # website -> ?target=
      target_label: __param_target
    - source_labels: [__param_target]  # readable instance label
      target_label: instance
    - target_label: __address__        # connect to the exporter (keep last)
      replacement: localhost:9115
```

Final URL: `http://localhost:9115/probe?module=https_2xx&target=https://expense.daws90s.shop`. Without rule 2, every probe would show `instance="localhost:9115"`.

### Example 3 — skip a target without deleting it

```yaml
  relabel_configs:
    - source_labels: [__address__]
      regex: 'mysql\..*'
      action: drop
```

### Example 4 — drop noisy metrics after the scrape

```yaml
- job_name: backend
  static_configs:
    - targets: [backend.daws90s.shop:9101]
  metric_relabel_configs:
    - source_labels: [__name__]
      regex: 'nodejs_.*|process_.*'
      action: drop
```

Compare `scrape_samples_scraped` with `scrape_samples_post_metric_relabeling` for the backend job to prove it worked.

### relabel_configs vs metric_relabel_configs

| | relabel_configs | metric_relabel_configs |
| --- | --- | --- |
| When | Before scrape | After scrape, before storing |
| Works on | Each target | Each sample |
| Can see | `__address__`, `__meta_*`, your labels | All metric labels incl. `__name__` |
| Typical use | Build URL, pick targets, set `instance`/`tier` | Drop metrics, drop high-cardinality labels |
| Cost | Once per target | Every sample, every scrape |

### Gotchas

- **Regex is anchored.** `regex: frontend` doesn't match `frontend.daws90s.shop`. Use `frontend.*`.
- **Typos are silent.** `__schema__` instead of `__scheme__` gives no error; it is just ignored.
- **Order matters.** In blackbox, setting `__address__` first would overwrite the website before you copy it.
- **`__meta_*` labels vanish.** When we move to EC2 service discovery, copy useful tags into normal labels.

Full reference: the *Prometheus Relabeling — Plain-English Cheat Sheet* in this project's docs.

## 6. Lab checklist and student exercises

### Before class (trainer)

- [ ] All targets green under Status → Target health
- [ ] `curl` commands from section 3 return data from the monitoring VM
- [ ] A backup of the working config: `sudo cp /etc/prometheus/prometheus.yml /etc/prometheus/prometheus.yml.bak`
- [ ] Service Discovery page open in a browser tab

### Student exercises

1. Run `up` in the Prometheus UI. List every `job` and `instance` you see. Which labels did nobody write in the config?
2. Stop node_exporter on one VM. Note how long until `up` turns 0, and copy the exact error from Target health. Restore it.
3. Change the global `scrape_interval` to 30s, validate with `promtool`, reload, and confirm under Status → Configuration. Change it back.
4. Replace the hand-written `tier` labels in the node job with Example 1. Before reloading, predict the Target labels; then check the Service discovery page.
5. Add Example 4 to the backend job. Compare `scrape_samples_scraped` and `scrape_samples_post_metric_relabeling` before and after.
6. Break it on purpose: change `__address__` to `__adress__` in the blackbox job. Does `promtool` catch it? What does Target health show?

## 7. Recap and homework

**Recap in five lines:**

1. Metrics are vitals over time: a single reading means little, the trend and the normal range matter.
2. Pull makes silence visible (`up == 0`); push is for short jobs, unreachable targets, and logs/traces.
3. Exporters are translators that only read; the app's own instrumentation (prom-client) is the only source of business metrics.
4. The default config scrapes only Prometheus itself; `job` comes from `job_name`, `instance` from the target address.
5. `relabel_configs` edits each target's label bag before the scrape; `__` labels build the URL and then disappear.

**Homework:**

- Write, from memory, the URL Prometheus builds for a blackbox target, and name the label behind each part.
- Find one exporter not covered today (for example redis_exporter). Write down its default port, and three metrics you'd alert on and why.
- Read: [Prometheus relabel_config reference](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config) and the [default port allocations list](https://github.com/prometheus/prometheus/wiki/Default-port-allocations).

**Next session:** PromQL basics — instant vs range vectors, `rate()` on counters, and why CPU % is always a ratio of two snapshots.