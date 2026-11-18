# Prometheus Metric Types — Analogies & Expense App Examples

**App:** ALB → frontend (nginx) → backend (Node, metrics on `:9101`) → MySQL
**Domain:** `daws90s.shop` · public URL `https://expense.daws88s.online`
**Where to run queries:** Prometheus UI → **Query** tab (**Table** = now, **Graph** = over time)

> Backend labels: `method`, `route`, `status_code` (not `status`).
> Real routes: `/api/expenses`, `/api/expenses/summary`, `/api/categories`, `/api/insights/month`, `/api/auth/signup`, `/api/auth/me`, plus `unmatched` (404s).

---

## 1. The 4 types at a glance

There are only **4 metric types**. The `# TYPE` line on `/metrics` tells you which one:

```bash
curl -s backend.daws90s.shop:9101/metrics | grep '^# TYPE'
```

| Type | 🚗 Car analogy | 🎓 Class analogy | Expense app example |
|---|---|---|---|
| **Counter** | Odometer / trip meter — only goes up | Total answer papers corrected so far | `http_requests_total` |
| **Gauge** | Speedometer, fuel gauge, engine temp — up and down | Students in the room right now | in-flight requests, free memory, `probe_success` |
| **Histogram** | Trip log: seconds driven in each speed band | Tally sheet of marks bands (≤40, ≤60, ≤80…) | `http_request_duration_seconds` |
| **Summary** | Car computer says "95% of your driving was under 90 km/h" | Each exam hall reports its own percentile | backend summary metric |

---

## 2. Counter — "how many so far"

### Analogy
🚗 **Odometer / trip meter.** "12,340 km" tells you nothing about how you're driving *now*.
- Read it now, read it 5 min later, divide by time → **speed** = `rate()`
- "How many km today?" → `increase()`
- Trip meter reset to 0 at the petrol bunk → **counter reset** (backend restart). `rate()` handles it — it doesn't think you drove −12,000 km.
- Speed from just the last 2 readings → `irate()` — jumpy, every brake shows.

### Raw on the expense app
```
http_requests_total{method="GET",  route="/api/expenses", status_code="200"} 1523
http_requests_total{method="POST", route="/api/expenses", status_code="201"} 210
http_requests_total{method="GET",  route="/api/auth/me",  status_code="401"} 34
http_requests_total{method="GET",  route="unmatched",     status_code="404"} 6
```
`1523` = "GET /api/expenses succeeded 1523 times since the backend started" — useless alone.

### Queries
| Question | Query |
|---|---|
| Backend requests/sec | `sum(rate(http_requests_total[5m]))` |
| Requests/sec per route | `sum by (route) (rate(http_requests_total[5m]))` |
| Requests/sec per status | `sum by (status_code) (rate(http_requests_total[5m]))` |
| Expenses added in last hour | `sum(increase(http_requests_total{method="POST", route="/api/expenses"}[1h]))` |
| Error % (5xx), safe at 0 errors | `100 * (sum(rate(http_requests_total{status_code=~"5.."}[5m])) or vector(0)) / sum(rate(http_requests_total[5m]))` |
| Unauthenticated `/api/auth/me` calls in last hour | `sum(increase(http_requests_total{route="/api/auth/me", status_code="401"}[1h]))` |
| nginx requests/sec | `rate(nginx_http_requests_total[5m])` |
| MySQL queries/sec | `rate(mysql_global_status_queries[5m])` |
| CPU busy % per VM (`node_cpu_seconds_total` is a counter!) | `100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])))` |

### Demo
```bash
sudo systemctl restart backend     # on the backend VM
```
Raw `http_requests_total` drops to 0 → `rate()` graph barely moves.

### Gotcha
A route called **only once** shows rate = 0. The series is *born* at 1 and never changes, so Prometheus never sees 0 → 1. You need **repeated** traffic.

---

## 3. Gauge — "what is it right now"

### Analogy
🚗 **Speedometer, fuel gauge, temperature needle.** Goes up and down; you read it directly.
- "Top speed in last 10 min?" → `max_over_time`
- "Distance to empty" display → `predict_linear` (disk filling up)
- `rate()` on the fuel gauge = nonsense (refills look like resets).

🎓 **Students in the room right now** — people come and go.

### Raw on the expense app
```
http_requests_in_flight 3                          ← backend (check exact name)
node_memory_MemAvailable_bytes 1.2e+09
node_filesystem_avail_bytes{mountpoint="/"} 1.4e+10
mysql_global_status_threads_connected 5
probe_success 1
```

### Queries
| Question | Query |
|---|---|
| Requests being processed now | `http_requests_in_flight` |
| Peak in-flight, last 15 min | `max_over_time(http_requests_in_flight[15m])` |
| Memory used % per tier | `100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)` |
| Disk used % | `100 * (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})` |
| Open MySQL connections | `mysql_global_status_threads_connected` |
| Public site up? | `probe_success` |
| Availability % today | `100 * avg_over_time(probe_success[1d])` |
| Disk full within 24h? | `predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[6h], 24*3600) < 0` |
| Days to TLS cert expiry | `(probe_ssl_earliest_cert_expiry - time()) / 86400` |
| Backend restarts in 6h | `changes(process_start_time_seconds{job="backend"}[6h])` |

### Rule
**Never `rate()` a gauge.** Prometheus won't stop you — it just gives wrong numbers.

---

## 4. First: what is a percentile?

### 🎓 Exam analogy
100 students write an exam. Sort them by marks, lowest → highest.

```
Position:  1  2  3 ... 93  94 | 95 | 96  97  98  99  100
           └──── 94 below ───┘  YOU  └──── 5 above ────┘
```
- Position 50 = **p50 (median)** — the typical student
- Position 95 = **p95** — 94 below, **5 above**
- Position 99 = **p99** — only 1 above

"99 percentile in CAT" = you beat 99% of candidates. **Not** 99% marks.

| Percentage | Percentile |
|---|---|
| Your marks out of 100 | Your **position** compared to everyone |

**Bigger class:** 4,567 students → 1% = 45.67 → about **46 students** above the p99 student.

### Map to API requests
Sort requests **fastest → slowest**.

| Exam | API latency |
|---|---|
| Sort students by marks | Sort requests by response time |
| p50 student = typical | p50 = typical user's wait |
| p99 = only 1 scored **higher** | p99 = only 1 in 100 was **slower** |

⚠️ **The flip:** in exams, high = good (topper). In latency, high = bad — the p99 request is the **unluckiest user**.

### Why not the average?
- 🎓 Class average 70 marks → looks fine. But 5 students scored below 20 — hidden.
- 🌐 99 requests at 50ms + 1 at 5s → average ≈ 100ms looks fine; **p99 = 5s** shows the real pain.

> **Average tells you about the class. Percentile tells you about the student at a position.**

### How to read p95 correctly
`histogram_quantile(0.95, ...)` returns `0.233` means:

> ✅ "In the last 5 minutes, **95% of requests finished within 233ms**; the slowest 5% took longer."
>
> ❌ Not "the time 95% of requests took" (that sounds like a total or average).

p95 is a **cut-off line**, not a sum.

---

## 5. Histogram — "how long did requests take"

### Analogy
🚗 **Trip log by speed band:** tally of seconds driven ≤40, ≤60, ≤100 km/h, plus total km and total seconds.
🎓 **Marks tally sheet:** teacher doesn't write each student's exact marks — just puts a tally mark in each band.

A fleet manager can **add up logbooks from 10 cars** → that's why histograms combine across backend instances.

### Who does what
| Step | Who | What |
|---|---|---|
| 1 | **Backend app** (prom-client) | Times each request (e.g. 120ms) |
| 2 | **Backend app** | Adds +1 to **every bucket it fits** (≤0.2, ≤0.3, ≤0.5, … +Inf) and to `_count`; adds 0.12 to `_sum` |
| 3 | **Backend app** | Throws away the exact 120ms — keeps only counts |
| 4 | **Prometheus** | Scrapes `/metrics` every 15s, stores the counts |
| 5 | **Prometheus** | At query time, `histogram_quantile()` **estimates** the percentile from the buckets |

🎓 App = teacher making tally marks · Prometheus = exam office copying the tally sheet · `histogram_quantile` = office estimating "p90 student got ~80 marks" from the tally alone.

**Why this design:** 1 million requests = still ~13 numbers per route. Trade-off: percentiles are **estimates**.

### Your real buckets
`0.01, 0.025, 0.05, 0.1, 0.2, 0.3, 0.5, 1.0, 2.0, 5.0, +Inf` seconds

### Cumulative buckets — reading a real screenshot
`route="unmatched", status_code="404"`:

| `le` | Value | Meaning |
|---|---|---|
| 0.01 | 5 | 5 requests ≤ 10ms |
| 0.025 | 6 | 6 requests ≤ 25ms (the same 5 + 1 more) |
| 0.05 … +Inf | 6 | still the same 6 |

→ **6 requests total** (`+Inf` = total). 5 took < 10ms, 1 took 10–25ms (6 − 5).
It's **not** 5 + 6 + 6 + 6… — the same requests are counted in every bucket they fit.

`route="/api/auth/me", status_code="401"`: 0 up to `le=0.1`, 1 from `le=0.2` → **1 request, 100–200ms**.

**Why 404 is fast and 401 slow:** 404 = route not found, reply immediately. 401 = auth check (cookie/token, maybe MySQL lookup) — real work.

### Worked example — 200 requests to `GET /api/expenses`

| `le` | Cumulative | In this band |
|---|---|---|
| 0.01 | 10 | 10 took ≤ 10ms |
| 0.025 | 50 | 40 took 10–25ms |
| 0.05 | 120 | 70 took 25–50ms |
| 0.1 | 170 | 50 took 50–100ms |
| 0.2 | 188 | 18 took 100–200ms |
| 0.3 | 194 | 6 took 200–300ms |
| 0.5 | 197 | 3 took 300–500ms |
| 1.0 | 199 | 2 took 0.5–1s |
| 2.0 | 200 | 1 took 1–2s |
| 5.0 / +Inf | 200 | **total = 200** |

`_sum` = 15.5s · `_count` = 200

| Question | Working | Answer |
|---|---|---|
| Average | 15.5 / 200 | **78ms** |
| % under 100ms | 170 / 200 | **85%** |
| % within 500ms SLO | 197 / 200 | **98.5%** |
| % slower than 1s | (200 − 199) / 200 | **0.5%** |
| Requests 100–500ms | 197 − 170 | **27** |
| p50 (rank 100) | 0.025 + (100−50)/(120−50) × 0.025 | **≈ 43ms** |
| p90 (rank 180) | 0.1 + (180−170)/(188−170) × 0.1 | **≈ 156ms** |
| p95 (rank 190) | 0.2 + (190−188)/(194−188) × 0.1 | **≈ 233ms** |
| p99 (rank 198) | 0.5 + (198−197)/(199−197) × 0.5 | **≈ 750ms** |

**Percentile by hand:**
1. Rank = percentile × total
2. Find the first bucket ≥ rank
3. Assume even spread inside that bucket → interpolate

That is exactly what `histogram_quantile()` does.

### Same route, MySQL slow (+300ms `tc netem`)

| `le` | Normal | MySQL slow |
|---|---|---|
| 0.01 → 0.3 | 10 … 194 | **0** everywhere |
| 0.5 | 197 | 150 |
| 1.0 | 199 | 185 |
| 2.0 | 200 | 197 |
| 5.0 / +Inf | 200 | 200 |

| | Normal | MySQL slow |
|---|---|---|
| Average | 78ms | ≈ 574ms |
| p50 | 43ms | ≈ 433ms |
| p95 | 233ms | ≈ 1.42s |
| p99 | 750ms | ≈ 3s |
| SLO (≤ 500ms) | 98.5% | **75%** |

Request count is still 200 → a counter sees nothing wrong. Only the histogram shows the distribution moved right.

### Live queries
```promql
# p95 overall (real traffic only)
histogram_quantile(0.95,
  sum by (le) (rate(http_request_duration_seconds_bucket{route!="unmatched"}[5m])))

# p95 per route — which API is slowest?
histogram_quantile(0.95,
  sum by (le, route) (rate(http_request_duration_seconds_bucket{route!="unmatched"}[5m])))

# Latency per status code (401 vs 404 vs 200)
histogram_quantile(0.95,
  sum by (le, status_code) (rate(http_request_duration_seconds_bucket[5m])))

# SLO: % of requests under 500ms
100 * sum(rate(http_request_duration_seconds_bucket{le="0.5", route!="unmatched"}[5m]))
    / sum(rate(http_request_duration_seconds_count{route!="unmatched"}[5m]))

# % slower than 1s   (le="1.0", not "1")
100 * (1 - sum(rate(http_request_duration_seconds_bucket{le="1.0"}[5m]))
         / sum(rate(http_request_duration_seconds_count[5m])))

# Requests/sec that took 100–500ms (bucket subtraction)
sum(rate(http_request_duration_seconds_bucket{le="0.5"}[5m]))
  - sum(rate(http_request_duration_seconds_bucket{le="0.1"}[5m]))

# Average latency per route
sum by (route) (rate(http_request_duration_seconds_sum[5m]))
  / sum by (route) (rate(http_request_duration_seconds_count[5m]))

# 3 slowest routes at p99
topk(3, histogram_quantile(0.99,
  sum by (le, route) (rate(http_request_duration_seconds_bucket[5m]))))
```

`rate(...[5m])` turns "counts since start" into "what happened in the **last 5 minutes**".

### Histogram gotchas
| You see | Meaning |
|---|---|
| **NaN** | No requests in the window (0 ÷ 0). Generate repeated traffic. Grafana shows "No data". |
| **Empty result** | Series doesn't exist — `le` missing from `by()`, wrong label (`status` vs `status_code`), wrong `le` (`"1"` vs `"1.0"`) |
| p99 = exactly **5** | Real latency > 5s (highest bucket) — Prometheus can't see beyond it |
| p99 = 750ms but real was 510ms | It's an **estimate** inside the 0.5–1.0 bucket. Add buckets where precision matters. |
| Can't measure a 400ms SLO exactly | No `le="0.4"` bucket. **Design buckets around your SLO.** |
| `[500m]` | 500 **minutes**, not milliseconds. `ms` = milliseconds, `s`, `m` = minutes, `h` |

---

## 6. Summary — "the app already worked out the percentile"

### Analogy
🚗 Car's computer: "95% of your driving was under 90 km/h." Exact — for that one car.
Fleet manager: car A says p95 = 90, car B says p95 = 60 → averaging to 75 is **wrong**; the raw data was thrown away.

🎓 Each exam hall reports its own p99 — you can't average two halls' percentiles to get the whole exam's.

### Raw (one backend process)
```
<summary_name>{quantile="0.5"}   0.061
<summary_name>{quantile="0.95"}  0.452
<summary_name>{quantile="0.99"}  0.880
<summary_name>_sum               11.8
<summary_name>_count             100
```
Read directly → p99 = 880ms **for this one process**.

### The trap
Add a second backend VM: backend-1 p99 = 880ms, backend-2 p99 = 120ms.
```promql
avg(<summary_name>{quantile="0.99"})    # ❌ 500ms — meaningless
```
Histogram: `sum by (le)` across both → true combined p99. ✅

| | Histogram | Summary |
|---|---|---|
| Percentile calculated by | Prometheus (query time) | App (fixed in code) |
| Combine across instances | ✅ | ❌ |
| New percentile later | ✅ any | ❌ fixed |
| Accuracy | Estimate (bucket-dependent) | Exact |
| **Default choice** | ✅ | Rarely |

---

## 7. One incident, all four types

**Break drill:** 300ms network delay on the MySQL VM while students use the app.

```bash
IF=$(ip route | awk '/default/{print $5}')
sudo tc qdisc add dev $IF root netem delay 300ms
# ... watch ...
sudo tc qdisc del dev $IF root
```

| Type | Metric | What happens |
|---|---|---|
| Counter | `rate(http_requests_total[5m])` | ≈ same — users still clicking |
| Gauge | `http_requests_in_flight` | ⬆ requests pile up |
| Gauge | `mysql_global_status_threads_connected` | ⬆ connections held open |
| Histogram | p50 | ⬆ rises somewhat |
| Histogram | p99 | ⬆⬆ past 500ms |
| Histogram | SLO % | ⬇ ~98% → ~75% |
| Gauge | `probe_success` | still 1 — "up" but slow |

> **Up ≠ healthy.** Only the latency histogram shows users are suffering.

---

## 8. Class exercises (use the §5 worked example)

1. How many requests took 25–50ms?
2. What % finished within 50ms?
3. Estimate p75.
4. SLO changes to 200ms. New SLO %?
5. Can you calculate the exact SLO % for a 400ms target? Why?
6. With 4,567 requests, how many are slower than p95?
7. From the 404 screenshot: how many 404s, and how many took 10–25ms?

<details>
<summary>Answers</summary>

1. 120 − 50 = **70**
2. 120 / 200 = **60%**
3. Rank 150 → between 0.05 (120) and 0.1 (170): 0.05 + 30/50 × 0.05 = **80ms**
4. 188 / 200 = **94%**
5. **No** — no `le="0.4"` bucket; only 0.3 and 0.5. Design buckets around the SLO.
6. 5% × 4,567 = 228.35 → **~228**
7. `+Inf` row = **6**; 6 − 5 = **1**

</details>

---

## 9. Rules for the board

```
1. Counter   → rate() / increase()      never read the raw value
2. Gauge     → read directly / *_over_time / predict_linear      never rate()
3. Histogram → histogram_quantile(q, sum by (le) (rate(x_bucket[5m])))
               avg = rate(_sum) / rate(_count)
4. Summary   → read directly, per instance, never average
5. rate first, then sum
6. keep `le` in by()
7. window ≥ 4 × scrape interval   (15s → [1m] min, [5m] normal)
8. no irate in alerts
9. guard empty results with `or vector(0)`
10. our labels: status_code (not status), le="1.0" (not "1")
11. NaN = no traffic in window → generate repeated requests
```

## 10. Before class
- Start traffic **≥ 5 min** before percentile demos (students using the app, or a looped "Copy as cURL" from DevTools).
- Have everyone hit the **same few pages** so each route gets many requests.
- Run the MySQL delay drill **while** students are clicking — they feel it and see it.