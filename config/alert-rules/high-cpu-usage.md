# Alert rule: High CPU Usage

A Grafana managed alert rule. Grafana stores these in its own database rather
than in a file on disk, so this document is the source of truth for recreating
the rule from scratch.

## Definition

| Setting | Value |
|---|---|
| Rule name | `High CPU Usage` |
| Rule type | Grafana managed alert rule |
| Data source | `prometheus` (`http://localhost:9090`) |
| Folder | `Linux Servers` |
| Evaluation group | `cpu` |
| Evaluation interval | `10s` |
| Pending period | `30s` |
| Condition | Threshold, input `A` **is above** `80` |
| Labels | `alertname=High CPU Usage`, `grafana_folder=Linux Servers`, `instance=localhost:9100` |

## Query A

```promql
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)
```

`node_cpu_seconds_total` is a counter of seconds each CPU has spent in each
mode. `rate(...[1m])` converts the idle counter into a per second rate over a
one minute window, `avg by(instance)` collapses the individual cores into a
single value per host, and subtracting from 100 turns idle time into busy time
as a percentage.

## Recreating the rule

1. Grafana, **Alerting**, **Alert rules**, **New alert rule**.
2. Name it `High CPU Usage`.
3. Section 2, query `A`: select the `prometheus` data source and paste the
   PromQL above. Set the query range to `10m`.
4. Section 3, expression `C` (Threshold): input `A`, **is above**, `80`.
   Mark it as the alert condition.
5. Section 4: folder `Linux Servers`, new evaluation group `cpu` with a `10s`
   interval, pending period `30s`.
6. Save and exit.

## Behaviour under load

With a `10s` evaluation interval and a `30s` pending period, the rule needs
roughly three consecutive breaching evaluations before it moves from `Pending`
to `Firing`. The one minute `rate` window adds further smoothing, so a load
test should run for at least two minutes before the alert is expected to fire.

## Verification

```bash
# Generate load (two CPU workers)
stress --cpu 2 --timeout 900 &

# Confirm the workers are on CPU
top -b -n 1 | head -15

# Restore normal operation
pkill stress
```
