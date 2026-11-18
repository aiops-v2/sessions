# Prometheus Special Labels (`__address__`, `__param_*`, `__meta_*` …)

## The one rule

> **Starts with `__`** → a **scrape setting**. Prometheus uses it to build the request, then **throws it away**.
> **No `__`** → a **real label**, **stored on every metric** from that target.

Inside Prometheus, each target is just **a bag of labels**, and the scrape URL is assembled from the `__` ones:

```
__scheme__ :// __address__ __metrics_path__ ? __param_<name>=value
   http    :// localhost:9115    /probe      ? target=https://expense.daws88s.online & module=https_2xx
```

---

## The lifecycle of a target

```
1. DISCOVERY          2. relabel_configs       3. SCRAPE          4. metric_relabel_configs   5. STORE
targets: / ec2_sd  →  edit the label bag   →   build URL from  →  edit each metric        →   save with
creates the bag       (before scrape)          __ labels, call    (after scrape)              non-__ labels
                                               the exporter
```

---

## Group 1: labels that build the request

| Label | Controls | Default | Set in YAML by | Blackbox value |
|---|---|---|---|---|
| `__address__` | host:port to **connect** to | each entry in `targets:` | `targets:` | starts as `https://expense.daws88s.online`, rewritten to `localhost:9115` |
| `__scheme__` | `http` or `https` | `http` | `scheme:` | `http` (talking to the exporter) |
| `__metrics_path__` | URL path | `/metrics` | `metrics_path:` | `/probe` |
| `__param_<name>` | URL query parameter `?<name>=…` | none | `params:` | `__param_target`, `__param_module` |
| `__scrape_interval__` | how often this target is scraped | job/global | `scrape_interval:` | 15s |
| `__scrape_timeout__` | timeout for this target | job/global | `scrape_timeout:` | 10s |

**How the YAML maps to labels:**

```yaml
- job_name: blackbox_http
  metrics_path: /probe          # → __metrics_path__="/probe"
  params:
    module: [https_2xx]         # → __param_module="https_2xx"
  static_configs:
    - targets: [https://expense.daws88s.online]   # → __address__="https://expense.daws88s.online"
      labels: { probe: https }                     # → probe="https"   (normal label, kept)
```

`metrics_path:` and `params:` are just shortcuts for setting these labels. Relabeling can override them **per target**.

---

## Group 2: helper labels (only exist during relabeling)

| Label | Meaning |
|---|---|
| `__meta_*` | Metadata from **service discovery**, e.g. `__meta_ec2_private_ip`, `__meta_ec2_tag_Name`, `__meta_ec2_instance_state`. `static_configs` has none. |
| `__tmp_*` | Your own scratch space for multi-step rules. Prometheus never uses these. |

Both are **deleted after relabeling**. To keep a value (e.g. an EC2 `Name` tag), copy it into a normal label:

```yaml
- source_labels: [__meta_ec2_tag_Name]
  target_label: instance
```

---

## Group 3: labels Prometheus adds automatically (stored)

| Label | Value |
|---|---|
| `job` | the `job_name` |
| `instance` | `__address__` **after** relabeling, **unless you've already set `instance` yourself** |

This is why the blackbox rule (2) below exists. Without it, `instance` would be `localhost:9115` on every probe, because that's the final `__address__`.

---

## Group 4: only after the scrape

| Label | Where | Meaning |
|---|---|---|
| `__name__` | `metric_relabel_configs` | The **metric name**. `http_requests_total{…}` is really `{__name__="http_requests_total", …}` |

That's why this works:

```yaml
metric_relabel_configs:
  - source_labels: [__name__]
    regex: "go_gc_.*"
    action: drop
```

It also works in PromQL: `{__name__=~"probe_.*"}` returns every probe metric.

---

## Group 5: metrics Prometheus creates for every target

These aren't labels, but they come up in the same discussion:

| Metric | Meaning |
|---|---|
| `up` | 1 = scrape OK, 0 = failed |
| `scrape_duration_seconds` | how long the scrape took (for blackbox, roughly the probe time) |
| `scrape_samples_scraped` | how many samples came back |
| `scrape_series_added` | new series this scrape |

---

## The blackbox job, label by label

```yaml
relabel_configs:
  - source_labels: [__address__]      # (1) website → ?target=
    target_label: __param_target
  - source_labels: [__param_target]   # (2) readable instance label
    target_label: instance
  - target_label: __address__         # (3) connect to the exporter (must be LAST)
    replacement: localhost:9115
```

| Step | `__address__` | `__param_target` | `__param_module` | `instance` | `probe` |
|---|---|---|---|---|---|
| From YAML | `https://expense.daws88s.online` | – | `https_2xx` | – | `https` |
| rule (1) `__address__` → `__param_target` | same | `https://expense…` | `https_2xx` | – | `https` |
| rule (2) `__param_target` → `instance` | same | same | same | `https://expense…` | `https` |
| rule (3) set `__address__` | **`localhost:9115`** | same | same | same | `https` |
| **Stored?** | ✗ dropped | ✗ dropped | ✗ dropped | ✅ kept | ✅ kept |

**Scrape URL:** `http://localhost:9115/probe?target=https://expense.daws88s.online&module=https_2xx`

(+ `job="blackbox_http"` is added.) The result is:

```
probe_success{job="blackbox_http", instance="https://expense.daws88s.online", probe="https"} 1
```

---

## See it live

**Status → Service Discovery** → open a target → **Discovered labels** (before) vs **Target labels** (after).

Show a `node` target and a `blackbox` target side by side. That one screen explains the whole concept.

---

## Gotchas

| Gotcha | Result |
|---|---|
| Typo, e.g. `__schema__` or `__param_targt` | **No error.** It's silently ignored |
| Rule order | Rules run top to bottom. Overwrite `__address__` before copying it and the website name is lost |
| `__meta_*` not copied | The tag info disappears and isn't on your metrics |
| Regex is anchored | `regex: frontend` matches only exactly `frontend`. Use `frontend.*` |

---

**One line for students:** `__` labels **build the request** and then vanish. Normal labels **stay on the data**. Relabeling = **editing the bag before the scrape**.