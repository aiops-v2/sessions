# PromQL in 2 Sessions — Concept + Live Practice on the Expense App

Two classes of 90 min. Each topic follows the same rhythm:
**Concept (2–3 min) → Run the query live → Observe → Students try the exercise → One deliberate mistake.**

Environment: ALB → frontend VM → backend VM → MySQL VM, Prometheus on the monitoring VM (`:9090`).
Public URL: `https://expense.daws88s.online`

---

## 0. Before class (do this 15 min early)

### 0.1 Check the metric names on your VMs
Names below are the standard ones. Confirm yours once, and fix the sheet if anything differs:

```bash
curl -s http://backend.daws90s.shop:9101/metrics | grep -E '^# (TYPE|HELP)' | grep http
```
Look for: `http_requests_total`, `http_request_duration_seconds` (histogram), the summary name, the in-flight gauge name, and labels `method`, `route`, `status_code`.

### 0.2 Fix `route="unmatched"` first
If every series shows `route="unmatched"`, all RED demos will only show 404s. Real API traffic must appear as `route="/api/expenses"` (template, not raw URL).

### 0.3 Start load (k6)
Adjust endpoints/body to your backend's real API. Mix of valid + invalid so students see 2xx, 4xx and (during drills) 5xx.

```js
// load.js
import http from 'k6/http';
import { sleep } from 'k6';

const BASE = 'https://expense.daws88s.online';

export const options = {
  insecureSkipTLSVerify: true,
  stages: [
    { duration: '2m', target: 10 },
    { duration: '100m', target: 20 },   // hold through the whole class
    { duration: '1m', target: 0 },
  ],
};

export default function () {
  http.get(`${BASE}/api/expenses`);                                    // 200
  http.post(`${BASE}/api/expenses`,
    JSON.stringify({ description: 'tea', amount: 20 }),
    { headers: { 'Content-Type': 'application/json' } });             // 200/201
  if (Math.random() < 0.1) {
    http.post(`${BASE}/api/expenses`, JSON.stringify({}),
      { headers: { 'Content-Type': 'application/json' } });           // 400
  }
  if (Math.random() < 0.05) http.get(`${BASE}/api/does-not-exist`);  // 404
  sleep(1);
}
```
```bash
k6 run load.js
```
Wait ~5 min before class so `[5m]` windows have full data.

### 0.4 Break drills you'll use (keep terminals open)
| Drill | Command | Where |
|---|---|---|
| Counter reset | `sudo systemctl restart backend` | backend VM |
| 5xx errors | `sudo systemctl stop mysqld` (restore: `start`) | MySQL VM |
| Exporter down | `sudo systemctl stop mysqld_exporter` | MySQL VM |

---

# SESSION 1 — Selecting & Rates (90 min)

## 1.1 Data model (0–10 min)

**Concept:** A time series = metric name + unique set of labels. Each label combination is a *separate* series.
- **Instant vector** — one latest value per series ("what is it now?")
- **Range vector** — many samples per series over a window ("what happened in the last 1m?")

**Run (Table view):**
```promql
http_requests_total
```
Observe: many rows — one per method × route × status_code.

```promql
count(http_requests_total)
```
Observe: how many series this one metric creates.

```promql
http_requests_total[1m]
```
Observe: each series shows ~4 raw samples with timestamps (15s scrape).

**Deliberate mistake:** switch the range vector query to Graph view → error *"expected type instant vector… got range vector"*. Read the error first: graphs need instant vectors.

**Exercise:** How many series does `node_cpu_seconds_total` have on the backend VM? Why so many? (cpu × mode)

---

## 1.2 Label matchers (10–25 min)

**Concept:** `=` exact, `!=` not equal, `=~` regex, `!~` regex not. Regex is **fully anchored**.

**Run:**
```promql
http_requests_total{route="/api/expenses"}
http_requests_total{status_code=~"4.."}
http_requests_total{status_code=~"4..|5.."}
http_requests_total{route!="unmatched"}
node_load1{tier="mysql"}
{__name__=~"node_memory_Mem.*"}
```

**Deliberate mistakes:**
- `http_requests_total{status_code=~"4"}` → empty. Anchored regex means "exactly 4", not "starts with 4".
- `{job=~".*"}` → error *"must contain at least one non-empty matcher"*.

**Exercise:**
1. All series for POST requests that were not 2xx.
2. `up` for every target except Prometheus itself.

---

## 1.3 Counters: `rate`, `increase`, `irate` (25–50 min) — most important topic

**Concept:** A counter only goes up (and resets to 0 on restart). The raw value is useless; the *speed* of change is what matters.
- `rate(x[5m])` — average per-second rate over 5 min
- `increase(x[5m])` — total increase over 5 min (= rate × 300)
- `irate(x[5m])` — rate from the last 2 samples only (spiky)

**Run (Graph view):**
```promql
http_requests_total{route="/api/expenses", method="GET", status_code="200"}
```
Observe: straight line going up. Ask: "is traffic high or low?" — can't tell.

```promql
rate(http_requests_total{route="/api/expenses", method="GET", status_code="200"}[5m])
```
Observe: flat line ≈ requests/sec from k6. Compare with k6 output.

```promql
increase(http_requests_total{route="/api/expenses", method="GET", status_code="200"}[5m])
```
Observe: requests in last 5 min. Not an integer — Prometheus extrapolates to the window edges.

```promql
irate(http_requests_total{route="/api/expenses", method="GET", status_code="200"}[5m])
```
Observe: spiky vs smooth `rate`. Use `irate` for zoomed-in debugging, `rate` for alerts.

**Break drill — counter reset:** restart the backend.
- Raw counter → drops to 0.
- `rate(...)` → continues almost unaffected. `rate` detects resets.
```promql
resets(http_requests_total{route="/api/expenses"}[15m])
```
Observe: shows 1 after the restart.

**Deliberate mistake:** `rate(http_requests_total[15s])` → empty. Window must hold ≥ 2 samples; rule of thumb ≥ 4 × scrape interval (use `[1m]` minimum, `[5m]` default).

**Exercise:**
1. Requests/sec for 404 responses.
2. How many POST requests in the last 10 minutes?
3. Network bytes received/sec on the frontend VM: `node_network_receive_bytes_total`.

---

## 1.4 Aggregation (50–70 min)

**Concept:** `sum`, `avg`, `min`, `max`, `count`, `topk`. `by (labels)` keeps only those labels; `without (labels)` drops only those.
**Golden rule: `rate()` first, then `sum`.**

**Run:**
```promql
sum(rate(http_requests_total[5m]))                         # total RPS
sum by (route) (rate(http_requests_total[5m]))             # RPS per route
sum by (status_code) (rate(http_requests_total[5m]))       # RPS per status
sum by (method, route) (rate(http_requests_total[5m]))
sum without (instance, job) (rate(http_requests_total[5m]))
topk(3, sum by (route) (rate(http_requests_total[5m])))
count by (tier) (up{job="node"})
```

**Deliberate mistake — sum before rate:**
```promql
rate(sum(http_requests_total)[5m:])      # WRONG (subquery)
sum(rate(http_requests_total[5m]))       # RIGHT
```
Restart the backend and graph both. The wrong one shows a fake dip/spike at the restart — once counters are summed, `rate` can no longer see each series' reset.

**Exercise:**
1. RPS per status_code, only for `/api/expenses`.
2. Which tier has the highest 1-min load? (`topk(1, node_load1)`)
3. CPU % per VM: `100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)` — explain every part.

---

## 1.5 Gauges (70–85 min)

**Concept:** A gauge goes up and down (memory, load, in-flight requests). Use the value directly or `*_over_time`. **Never `rate()` a gauge.**

**Run:**
```promql
node_memory_MemAvailable_bytes / 1024 / 1024              # MB free
max_over_time(node_load1[30m])                            # peak load in 30m
avg_over_time(node_load1[30m])
http_requests_in_flight                                   # use your gauge name from 0.1
max_over_time(http_requests_in_flight[10m])
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[1h], 4*3600)   # free bytes in 4h
```

**Deliberate mistake:** `rate(node_memory_MemAvailable_bytes[5m])` → returns numbers, but they're meaningless (and drops are treated as resets). Prometheus won't stop you — *the metric name lies to you unless you verify it yourself*; check `# TYPE` in /metrics.

**Exercise:** Increase k6 to 50 VUs for 2 min. What was the max in-flight requests and max load on the backend VM during that spike?

## Session 1 homework (5 min)
1. Total RPS hitting nginx: `sum(rate(nginx_http_requests_total[5m]))`. Does it match backend RPS? Why/why not?
2. Requests per minute for POST `/api/expenses`.
3. Top 2 routes by request count in the last 15 minutes.
4. Lowest free memory (MB) on the MySQL VM in the last hour.
5. Why does `increase()` return 299.7 instead of 300?

---

# SESSION 2 — Combining & Real Use (90 min)

## 2.1 Binary operators (0–15 min)

**Concept:** Arithmetic `+ - * / %`, comparison `> < == !=`, logical `and or unless`. Between two vectors, Prometheus matches series with **identical labels**.

**Run:**
```promql
# Memory used % — both sides have the same labels, so they match
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Error ratio (5xx / total)
sum(rate(http_requests_total{status_code=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```
Observe: the error ratio is **empty** — there are no 5xx series yet. Remember this; it comes back in 2.4.

**Break drill:** stop MySQL. Wait 1 min. Error ratio now shows a value. Students watch it climb. Restore MySQL.

```promql
sum by (route) (rate(http_requests_total[5m])) > 1          # filter: keeps only >1
sum by (route) (rate(http_requests_total[5m])) > bool 1     # returns 1 / 0
```

**Exercise:** 4xx percentage per route.

---

## 2.2 Histogram queries (15–30 min)

**Concept:** (from the metric-types class) buckets are cumulative counts `≤ le`; `histogram_quantile` estimates the percentile by interpolating inside a bucket.

**Run:**
```promql
# p50 / p95 / p99 overall
histogram_quantile(0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket{route!="unmatched"}[5m])))

# p99 per route
histogram_quantile(0.99,
  sum by (le, route) (rate(http_request_duration_seconds_bucket{route!="unmatched"}[5m])))

# Average latency
sum(rate(http_request_duration_seconds_sum[5m]))
/ sum(rate(http_request_duration_seconds_count[5m]))

# SLO: % of requests ≤ 500ms (exact, no estimation)
sum(rate(http_request_duration_seconds_bucket{le="0.5", route!="unmatched"}[5m]))
/ sum(rate(http_request_duration_seconds_count{route!="unmatched"}[5m])) * 100
```
Compare: histogram p99 vs the summary's `{quantile="0.99"}` value.

**Live experiment:** raise k6 to 50–100 VUs. Watch p99 climb on the t3.micro while p50 barely moves.

**Deliberate mistakes:**
- `sum(rate(..._bucket[5m]))` without `by (le)` → NaN/empty.
- Include `unmatched` 404s → p99 looks better than reality.

**Exercise:** p95 latency of POST requests only.

---

## 2.3 Vector matching (30–45 min) — one solid example

**Concept:** When labels don't match exactly, tell Prometheus how to match:
- `on(labels)` — match only on these; `ignoring(labels)` — match ignoring these
- `group_left(extra)` — many-to-one; copy `extra` labels from the right side

**Run — add hostname to CPU %:**
```promql
(100 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
* on(instance) group_left(nodename)
node_uname_info
```
`node_uname_info` always = 1, so multiplying keeps the CPU value and adds `nodename`.

**Deliberate mistakes:**
- Without `on(instance) group_left` → empty (label sets differ).
- Joining backend app metrics with node metrics `on(instance)` → empty, because `backend:9101` ≠ `backend:9100`. The `instance` label includes the port. (Fix needs `label_replace` — notes only.)

**Exercise:** Memory used % with `nodename` attached.

---

## 2.4 Missing data (45–55 min)

**Concept:** No series ≠ zero. Empty results silently break math and alerts.

**Run:**
```promql
# Fix the empty error ratio from 2.1
(sum(rate(http_requests_total{status_code=~"5.."}[5m])) or vector(0))
/
sum(rate(http_requests_total[5m]))

up{job="mysql"}
mysql_up
absent(mysql_up)
```

**Break drill A — stop MySQL (not the exporter):**
`up{job="mysql"}` = 1, `mysql_up` = 0. **UP ≠ healthy.** The exporter is fine; the database is down.

**Break drill B — stop mysqld_exporter:**
`up{job="mysql"}` = 0, `mysql_up` disappears, `absent(mysql_up)` = 1.

**Exercise:** Write one query that returns a series when *either* the exporter or MySQL is down.
(answer: `up{job="mysql"} == 0 or mysql_up == 0`)

---

## 2.5 USE + RED — real queries (55–80 min)

Students build each one, then you show it. These become the dashboards and alerts.

### USE — node_exporter (per VM)
| Signal | Query |
|---|---|
| CPU % | `100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)` |
| Memory % | `(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100` |
| Disk % | `(1 - node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"}) * 100` |
| Saturation | `node_load1` (t3.micro = 2 vCPU → >2 is saturated) |
| Network | `rate(node_network_receive_bytes_total{device!="lo"}[5m]) * 8` (bits/s) |

### RED — backend
| Signal | Query |
|---|---|
| Rate | `sum by (route) (rate(http_requests_total{route!="unmatched"}[5m]))` |
| Errors % | `(sum(rate(http_requests_total{status_code=~"5.."}[5m])) or vector(0)) / sum(rate(http_requests_total[5m])) * 100` |
| Duration p99 | `histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{route!="unmatched"}[5m])))` |

### Edge / outside-in
| Signal | Query |
|---|---|
| nginx RPS | `sum(rate(nginx_http_requests_total[5m]))` |
| Active connections | `nginx_connections_active` |
| Site up (blackbox) | `probe_success` |
| Cert days left | `(probe_ssl_earliest_cert_expiry - time()) / 86400` |

**Final drill:** stop MySQL with the whole table open. Ask students which signals move (error %, `mysql_up`, maybe p99 drops because errors are fast!) and which don't (CPU, `probe_success` if the homepage is static). Inside-out vs outside-in.

---

## 2.6 Preview: recording rules & cardinality (80–90 min)

**Recording rule** = save an expensive query as a new metric:
```yaml
groups:
  - name: backend_recording_rules
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))
```
Naming: `level:metric:operation`.

**Cardinality:**
```promql
topk(10, count by (__name__) ({__name__=~".+"}))
count(http_request_duration_seconds_bucket)
```
Observe: the histogram is one of the biggest metrics — buckets × routes × methods × status codes. That's why `route` must be a template, never a raw URL.

---

## Left for notes (not taught live)
`offset` / `@`, `label_replace` / `label_join`, subqueries in depth, `count_values`, `quantile` operator, `delta` / `deriv`, `group_right`, burn-rate math (SLO class).

## Homework answers (Session 1)
1. Usually nginx RPS > backend RPS — nginx also serves static frontend files that never reach the backend.
2. `sum(rate(http_requests_total{method="POST", route="/api/expenses"}[5m])) * 60`
3. `topk(2, sum by (route) (increase(http_requests_total[15m])))`
4. `min_over_time(node_memory_MemAvailable_bytes{tier="mysql"}[1h]) / 1024 / 1024`
5. `increase` extrapolates from the samples inside the window to the window edges; samples rarely land exactly on the edges.