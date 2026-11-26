---
title: Storage of results
---

# Storage of results

The result of a connector for one item is a data point. This page describes where the parts of a result are stored and
how the platform keeps their history. Only three keywords store anything: `index`, `json` and `file` (or its short
form `simple-file`). All other keys of a connector are intermediate results, available to later keys but not stored.

| Keyword | Stored in | Use it for |
|---|---|---|
| `index` | Typed, indexed columns | A few fields to filter, sort and analyse across all items |
| `json` | One JSON column | Any number of extra fields, returned with the data point |
| `file`, `simple-file` | Object storage | Large or binary results: raw responses, reports, screenshots |

## Data forms

The `type` keyword selects the **data form** of a connector. Each data form is a pair of tables (current values and
history) with a fixed set of generic columns. Choose the most compact form that holds your indexed fields.

| Data form | Text | Small integer | Float | Datetime | JSON | Example connectors |
|---|---|---|---|---|---|---|
| `Empty` | | | | | | `screenshot`, `content`, `keywords`: files only |
| `StrInt` | 4 | 4 | | | | `dns`, `homepage`, `status` |
| `StrDateInt` | 4 | 4 | | 2 | | `certificate`, `whois` |
| `FloatInt` | | 4 | 4 | | | `timing`, `pagespeed`, `domainvalue` |
| `JsonStrInt` | 4 | 4 | | | 1 | `detailed_homepage`, `cms` |
| `Json` | | | | | 1 | `availability`, `subscription` |

Without `type`, a connector uses `Json`. The `Empty` form stores no values of its own; it holds the status code of a
data point that has no data (see [Status-only data points](#status-only-data-points)), and suits connectors that only
store files or only act.

## Indexed fields

Each field in `index` is stored in a column of the data form, so it can be used in filters and sorting:

```text
/yourhosting/hosting/domains?filter[dns.mx1]=_endswith_.example.com
```

Every indexed field needs a cast after its expression. The cast converts the value and selects the column type:

| Cast | Column | Conversion |
|---|---|---|
| `-> str` | Text | `str(value)` |
| `-> int` | Small integer | `int(value)`, limited to the range −32767 to 32767 |
| `-> float` | Float | `float(value)` |
| `-> datetime` | Datetime (timezone-aware) | Parsed from an ISO 8601 string, a date or a datetime |

An empty value leaves the column empty, and so does a value that cannot be converted to `str`, `int` or `float`. A
value that cannot be parsed as a datetime raises an error; guard it with `@suppress` where it may be malformed, as
`whois` does for its dates. Keep integers within the small-integer range yourself where the exact value matters, as `timing` does with `min(round(data.speed / 1_048_576), 32767)`.
Store larger numbers as `float`, as `domainvalue` does with `value`.

Pass/fail values are stored as `-> int`, so `True` becomes `1` and `False` becomes `0`.

### Column mapping

When a connector is loaded, the platform maps each indexed field onto a free column of its type and records the mapping
in a **key translation table**: the field name (`long_name`) against the column (`short_name`), per connector. For
`certificate`, `issuer` maps to a text column such as `value_str_0` and `valid_to` to a datetime column such as
`value_datetime_1`. Filters on a field name are translated through this table into conditions on the column.

The platform validates the `index` section when the connector is loaded, and rejects the definition with `422` if:

- a field has no cast;
- the data form has no free column left of that type, for example a fifth `-> str` field, or a `-> datetime` field in
  `StrInt`.

The mapping is kept across changes of the connector:

- a field keeps its column as long as its name stays the same;
- a new field gets a free column of its type;
- a removed field loses its mapping, and its column becomes free.

Give a field a new name when you change its type, so it is mapped to a column of the new type.

### Fields without an expression

An indexed field can consist of a cast only:

```yaml
index:
    year: -> int
    action: -> str
```

Such a field has no expression; its value is taken from the data written to the data point under the same name. This
is how the attributes of an item type (in `_entity.yaml`) are stored: the attributes posted with an item fill the
indexed columns of the same name, and all posted attributes are stored in the JSON column if the data form has one.
The `internet_nl` connector uses the same mechanism: it hands the item to an asynchronous, rate-limited checker,
and the platform writes the scores to the data point when they arrive.

## JSON

The value of `json` is stored in the JSON column of the data form, so it requires `JsonStrInt` or `Json`. It can be a
mapping with named fields or any JSON-serialisable value:

```yaml
json:
    availability: << functions.hosting.available_tlds(params.domain, params.tlds) >>
```

```yaml
json: << data >>      # the complete result
```

Dates and datetimes are stored as ISO 8601 strings. JSON fields appear next to the indexed fields under `attributes`
in the response, so keep the names of both distinct. They can be filtered on equality only; use an indexed field for
anything else.

The key translation table is also designed for JSON keys: it can map each long JSON key to a short name of one to three
characters (`A`, `B`, ..., `v0`, ..., `w00`), which saves space when millions of rows share the same keys. JSON fields are
returned under their own names.

## Files

`file` and `simple-file` write content to S3-compatible object storage and record a reference per item, connector and
file name. Use files for anything large, binary or not needed in lists: complete API responses (`full.json`), reports
(`report.json`), screenshots (`desktop.webp`).

`simple-file` maps file names to content, with private access:

```yaml
simple-file:
    full.json: << dns >>
```

`file` gives each file its own settings:

```yaml
file:
    report.json:
        content: << text >>
    desktop.webp:
        content: << content.desktop >>
        content-type: image/webp
        access: by-uuid
```

| Setting | Meaning |
|---|---|
| `content` | Bytes as they are; text in UTF-8; a mapping or list as JSON; `None` as an empty file |
| `content-type` | Optional. Without it the type follows from the file suffix: `json`, `txt`, `csv`, `xml`, `pdf`, `html`, `css`, `js`, `svg`, `png`, `jpg`, `gif`, `webp` and others; unknown suffixes get `application/octet-stream` |
| `access` | `private` (default), `public` or `by-uuid`. Each access level has its own storage bucket. With `by-uuid`, the storage path contains the item's UUID instead of its id. |

A file is written on every refresh and replaces the previous file of the same name; files have no history. The data
point lists its files under `links.files`, and the `file/<name>` modifier serves one through the API:

```text
/yourhosting/hosting/domain/example.com/dns/file/full.json
```

Another connector can read a stored file through its include, as `category` does with the content of the homepage:

```yaml
include: content
content: << get(include.content.links.files['full.json']).json() if include.content.links.files else {} >>
```

## Status-only data points

When a requirement is not met, the connector stops and the platform stores a data point that holds only a status
code, in the `Empty` form. A request for it returns that code as its HTTP status:

| Status | Written as | Meaning |
|---|---|---|
| 404 | `requires` | No information available for this item, for example no DNS records |
| 424 | `requires(424)` | Failed dependency: data from another connector is missing |
| 403 | `requires(403)` | Data rejected, for example pushed data that fails validation |
| 410 | `requires(410)` | Gone: no refresh moment, timestamp or history version is recorded |

The status-only data point is scheduled with the refresh rule as written in the file. See
[Includes and requires](language/includes-and-requires.md).

## History

Each data form has a **current** table, with one row per item and connector, and a **timeline** table, with one row
per distinct value. After every refresh the platform compares the new value with the latest timeline row:

| Situation | Effect on the timeline |
|---|---|
| First result | A first row with sequence number 1 |
| Value changed | The latest row is closed at the moment of the refresh; a new row starts at the same moment |
| Value unchanged | No change |

The comparison covers the indexed columns and the JSON value; JSON is compared with sorted keys, so a different key
order is not a change. Files are not part of the comparison. Consecutive rows meet exactly, so the value at any moment
is the row whose interval contains that moment.

Every refresh also adds its moment to the list of check moments of the current row, so unchanged checks are recorded
without a new timeline row. `properties.max_timestamps` limits that list to the most recent moments:

```yaml
properties:
    max_timestamps: 10   # keep the last 10 check moments
```

The `history` modifier returns all versions of a data point, in order:

```text
/yourhosting/hosting/domain/example.com/dns/history
```

Because history is written only on change, a connector produces a compact history when its stored fields are stable.
Keep volatile values, such as response times measured to the millisecond, out of a connector whose changes you want to
follow, or round them, as `timing` does for content size and speed. The design provides for consolidating older
history, for example after a year.
