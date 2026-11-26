---
title: Refresh rules
---

# Refresh rules

A refresh evaluates a connector again for one item and stores the new data point. Refreshes happen in two ways:

- **Scheduled.** After every refresh, the platform computes the moment of the next one from the connector's `refresh`
  rule. A scheduler picks up data points whose moment has passed and refreshes them in the background.
- **On access.** A request for a data point can start a refresh: when the data point does not exist yet, when the
  request asks for it, or when the connector is `on-demand`.

This page describes the grammar of the `refresh` keyword and the request parameters that control refreshes.

## The refresh keyword

```yaml
refresh:
    every: ~month
```

| Rule | Meaning |
|---|---|
| `refresh: never` | Not refreshed by the schedule; refreshed on access only |
| `refresh: on-demand` | Refreshed on every request; the request waits for the result. Not scheduled |
| `every: <interval>` | Refreshed after an interval (see [Intervals](#intervals)) |
| `crontab: "<expression>"` | Refreshed at the moments of a cron expression |
| `at: <moment>` | Refreshed at a fixed moment |
| No `refresh` key | The platform's default interval, `6 ~months` unless the operator configured otherwise |

- `every`, `crontab` and `at` are keys of a mapping. If more than one is given, the earliest moment wins. A mapping
  without any of them uses the default interval.
- `never` and `on-demand` are plain values. Any other plain value is not a schedule: write an interval under `every`.
- The `attributes` connector of an item type, defined in `_entity.yaml`, is never scheduled unless it has a rule.

### Rules with expressions

`refresh` is evaluated like any other key, so a rule can depend on the data computed above it. Place it after the keys
it reads:

```yaml
index:
    a1: << dns.generic.a1 >> -> str

refresh:
    every: << "3 ~months" if not index.a1 else "1 ~month" >>
```

The design also provides for refreshes tied to the data itself, for example an `at` rule that schedules a certificate
check around the expiry date in `index.valid_to`.

## Intervals

An interval is a number, an optional `~`, a unit and optional hours:

```text
[number] [~]unit[@start-end]

week        2 weeks     10 days     1 ~month     3 ~months@11-23
```

| Part | Meaning |
|---|---|
| `number` | Number of units; default 1. A space between number and unit is optional (`2weeks`) |
| `~` | A random moment within the whole period, for `week`, `month`, `quarter` and `year` |
| `unit` | One of the units below |
| `@start-end` | For intervals of a day or longer: the refresh takes place between these hours, such as `@11-23` |

| Unit | Accepted names |
|---|---|
| Seconds | `s`, `sec`, `second`, `seconds` |
| Minutes | `m`, `min`, `minute`, `minutes` |
| Hours | `h`, `hr`, `hour`, `hours` |
| Days | `d`, `day`, `days`, `night`, `nights` |
| Weeks | `w`, `week`, `weeks` |
| Months | `mon`, `month`, `months` |
| Quarters | `q`, `quarter`, `quarters` |
| Years | `y`, `year`, `years` |

Note that `m` means minutes; write `mon` or `month` for months.

### How the moment is placed

- **Fixed intervals** are counted from the first refresh of the data point, not from the last one, so the schedule does
  not drift. The next moment is the first one after the last refresh.
- **Whole periods (`~`)** pick a random day: in the week (Monday to Sunday), in the month (up to the day before its
  last day), in the quarter or in the year. Use `~` to spread the load of many items over the period, as most
  connectors do.
- **Intervals of a day or longer** are moved to a random time outside working hours, by default between 20:30 and
  06:30 in the platform's time zone, so heavy recurring work runs at night. With `@start-end` the time falls between
  those hours instead.
- **Shorter intervals and cron rules** are moved slightly earlier by a small random fraction of the interval, never
  later, to spread peaks.

## Cron expressions

`crontab` takes a five-field cron expression: minute, hour, day of the month, month and day of the week.

```yaml
refresh:
    crontab: "30 12 * */2 sun"   # 12:30 on Sundays, every second month
```

Write days of the week as names (`mon` … `sun`) rather than numbers.

## Fixed moments

`at` takes a moment, as an ISO 8601 string or as a datetime from an expression. Quote literal moments, and include a
time zone; a moment without one is read as UTC.

```yaml
refresh:
    at: "2027-01-01T03:00:00+01:00"
```

Use a moment in the future: a moment in the past makes the data point due at once.

## Examples from the catalogue

| Connector | Rule | Effect |
|---|---|---|
| `timing`, `internet_nl` | `every: ~month` | Once a month, on a random day, at night |
| `dns` | `every: << "3 ~months" if not index.a1 else "1 ~month" >>` | Monthly, or quarterly for domains without an A record |
| `status` | `every: << '~month' if index.category != 'active' else '6 ~months' >>` | Monthly until the website is active, then twice a year |
| `ssl`, `website`, `m365` (incentives) | `every: << "day" if not index.valid else "3 months" >>` | Daily until the check passes, then every three months |
| `availability`, `sleep`, `multi` | `refresh: on-demand` | Evaluated on every request |
| `plesk` (server), `subscription` | `refresh: never` | Only on access |
| `pagespeed`, `whois` | No rule | The default interval |

## Status data points

When a connector stops on a `requires`, it stores a data point with only a status (see
[Includes and requires](includes-and-requires.md#requires)). The connector has stopped, so its `refresh` key is not
evaluated: the platform schedules the status data point with the rule as it is written in the file.

- A literal rule, such as `every: ~month`, applies as usual.
- Without a rule, the default interval applies.

Status data points are scheduled with the rule as written in the file. Give a connector a literal rule when its status
data points must be retried on a schedule.

## Refresh on access

A `GET` on a data point returns the stored value. These query parameters, and the connector's `on-demand` rule and
`min_refresh_time` property, decide whether a refresh is started first:

| Parameter | Effect |
|---|---|
| None | A refresh only when the data point does not exist yet, or when the connector is `on-demand` |
| `refresh=now` | Refresh now |
| `refresh=<interval>` | Refresh if the last refresh is older than the interval, for example `refresh=1d` or `refresh=12h` |
| `refresh=no`, `refresh=false` | No refresh, except for `on-demand` connectors |
| `wait=false` | Do not wait for the result: the API answers `202 Accepted` at once and the refresh runs in the background |
| `max-refresh-level=<n>` | Refresh only connectors up to include level *n*; `0` refreshes the requested data point but not its includes |

```http
GET https://api.example.com/yourhosting/hosting/domain/example.com/timing?refresh=now
```

- **Waiting.** Without `wait=false`, the request waits for the refresh, up to a time limit of the platform (60 seconds
  by default). If the refresh takes longer, the API answers `202 Accepted`; request the data point again later to
  read the result.
- **Rights.** Only a token with create rights on the path starts a refresh. Other tokens receive the stored value.
- **Minimum age.** `properties.min_refresh_time` (in seconds) suppresses every refresh, including `refresh=now` and
  `on-demand`, while the last refresh is more recent:

    ```yaml
    properties:
        min_refresh_time: 2592000  # 30 days
    ```

- **Includes.** The parameters of the request, including `refresh` and `max-refresh-level`, are passed on to included
  connectors, so `refresh=now` also refreshes the included data points (see
  [Includes and requires](includes-and-requires.md#how-an-include-is-fetched)).
- **Identical requests** for the same data point and parameters that arrive while a refresh is running wait for that
  refresh instead of starting another.

### Priorities

Refreshes are queued with a priority. A request that waits for its result is served before background work. Requests
with `wait=false` use the lower background priority, so they use the capacity that interactive callers leave free.

## Scheduled refresh

The scheduler runs every minute by default and requests each data point whose refresh moment has passed. A scheduled refresh also refreshes the directly included data points that are older than an
hour; deeper includes are read as stored. Each data point therefore follows its own rule, and a connector that is
mostly used through includes still needs a sensible rule of its own.
