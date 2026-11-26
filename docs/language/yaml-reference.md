---
title: Connector YAML reference
---

# Connector YAML reference

A connector is a YAML file that describes one data point of an item type: where the data comes from, how it is
converted, what is stored and when it is refreshed. This page lists every keyword a connector file can use. The
expressions inside the values are described in [Expression language](expressions.md), `include` and `requires` in
[Includes and requires](includes-and-requires.md), and the `refresh` grammar in [Refresh rules](refresh.md).

## File location and name

The path of a file is the identity of the connector:

```text
<client>/<item type group>/<item type>/<connector>.yaml

sitekick/hosting/domain/dns.yaml              ->  sitekick/hosting/domain/dns
your.online/commercial/incentive/ssl.yaml     ->  your.online/commercial/incentive/ssl
```

| Part | Meaning | Example |
|---|---|---|
| `client` | The client (brand) that owns the connector | `sitekick`, `your.online`, `argeweb` |
| `item type group` | The subject group of the item type | `hosting`, `commercial` |
| `item type` | The item type the connector belongs to | `domain`, `server`, `incentive` |
| `connector` | The file name without extension; also the last segment of the URL | `dns`, `ssl_https` |

- Files ending in `.yaml` or `.yml` are loaded. Files whose name starts with `_` are not connectors: `_entity.yaml`
  defines the item type itself (see [Entities and relations](../entities-and-relations.md)).
- Use lower case and underscores in connector names, as `hosting_by_dns` and `google_analytics` do.
- Each item type folder has an `_entity.yaml`. Its `key-names` list the names under which the item key is available to
  connectors, for example `domain` for domains, so a domain connector reads the domain name as `params.domain`.

### How a file becomes a connector

The platform operator loads the repository into a Sitekick platform. For every file, the loader takes the last three
folders and the file name as the connector name and posts it, with the unchanged YAML text, to the platform's
`/data-types` endpoint. The platform then:

1. parses the YAML, checks that the name has four parts and that the item type exists;
2. selects the data form from `type`;
3. checks that every `index` field has a type conversion and that the data form has a free column of that type;
4. maps each `index` field onto a column of the data form;
5. stores the definition as a new version if it differs from the stored one. The key order of the file is kept,
   because it is the order of evaluation.

A definition that fails these checks is rejected with status `422` and a message that names the problem.

A connector belongs to the client of its folder and is available to that client and to every client below it in the
client hierarchy. When a connector is requested for a client that has no file of that name, the connector of the
parent client is used, up to `sitekick` at the root. A brand overrides a connector by placing a file with the same name
in its own folder, next to its own `_entity.yaml`.

A connector of a domain is then available at a URL such as:

```http
GET https://api.example.com/yourhosting/hosting/domain/example.com/dns
```

## Structure of a definition

A definition is a YAML mapping. Its top-level keys are evaluated from top to bottom, and the result of each key is
available under its name to every key below it. A connector is therefore a small program in which each key is a named
step.

- **Keywords** are top-level keys with a defined meaning, listed below.
- **Intermediate keys** are all other top-level keys, such as `data`, `url` or `headers`. They hold values for later
  keys and are not stored.
- Only top-level keys act as keywords. Inside a mapping, `include` or `file` is an ordinary value.
- Values are evaluated recursively: strings with `<< ... >>` are expressions, other strings and numbers are literal
  values, and mappings and lists are evaluated element by element.

```yaml
type: StrInt                                   # keyword: data form

data: << functions.hosting.homepage(params.domain) >>  # intermediate key

requires: << data.final_url >>                  # keyword: stop when empty

index:                                          # keyword: indexed fields
    final_url: << data.final_url >> -> str
    https: << data.https >> -> int
```

## Keyword summary

| Keyword | Value | Evaluated | Purpose |
|---|---|---|---|
| [`type`](#type) | Data form name | Read as written | Storage form of the data point; default `Json` |
| [`description`](#description) | Text | Read as written | Description of the connector |
| [`params`](#params) | Mapping | Yes | Default parameters |
| [`include`](#include) | Connector name or list of names | Yes | Fetch other data points of the same item |
| [`requires`](#requires) | Expression | Yes | Stop the connector when the value is empty or false |
| [`code`](#code) | Python source | Executed | Define helpers for later keys |
| [`index`](#index) | Mapping of typed fields | Yes | Fields stored in indexed columns |
| [`json`](#json) | Any value | Yes | Value stored in the JSON column |
| [`file`](#file) | Mapping of file name to file definition | Yes | Files written to object storage |
| [`simple-file`](#simple-file) | Mapping of file name to content | Yes | Short form of `file` |
| [`refresh`](#refresh) | Refresh rule | Yes | Moment of the next scheduled refresh |
| [`properties`](#properties) | Mapping | Read as written | Limits for refreshes and timestamps |
| [`bff`](#bff) | Modifier name | Read as written | Default modifier of the data point |

"Read as written" means that the platform reads the value as it is written in the definition, not from the evaluated
result, so it must be a literal value.

## Keywords

### type

The data form: the table pair in which the data point is stored, and with it the indexed columns that are available
to `index`. See [Storage of results](../storage.md) for the columns of each form.

| Value | Typical use |
|---|---|
| `Empty` | Connectors that only store files, such as `screenshot` and `content` |
| `StrInt` | Text and small integers, such as `dns` and `homepage` |
| `StrDateInt` | Text, small integers and datetimes, such as `certificate` and `whois` |
| `FloatInt` | Measurements, such as `timing` and `pagespeed` |
| `JsonStrInt` | Text and small integers plus a JSON value, such as `detailed_homepage` |
| `Json` | A JSON value only, such as `availability`; the default when `type` is missing |

```yaml
type: StrDateInt
```

### description

Free text that describes the connector. It is used as the description of the connector in the OpenAPI documentation
of the API.

```yaml
description: The homepage of the domain, resolve the url based on the domain
```

### params

Default values for the parameters of the connector. A parameter supplied in the request (as a query parameter) takes
precedence over the default. From this key on, `params` holds the merged result.

- The values are evaluated before they are merged, so a default can be an expression. Inside the `params` key,
  `params` still refers to the supplied parameters.
- Supplied parameters arrive as text: convert them where needed, as in `float(params.seconds)`.
- Before `params` is evaluated, `params` already holds the supplied parameters and the key names of the item type,
  such as `params.domain`.
- A default value that is not a mapping is stored as `params.params`.

```yaml
params:
    tlds: "eu com be de me net nu org online"

json:
    availability: << functions.hosting.available_tlds(params.domain, params.tlds) >>
```

The current `params` are passed on to included connectors, so place `params` after `include` when its defaults are
meant for this connector only, as `screenshot` does.

### include

One connector name, or a list of names, of the same item type. The data points of those connectors for the same item
are fetched, in parallel for a list, before the next key is evaluated. The result is available as
`include.<name>`; use `include['<name>']` for names with a hyphen.

```yaml
include:
    - homepage
    - category
```

See [Includes and requires](includes-and-requires.md#include) for the result, waiting, refresh and failure behaviour.

### requires

A condition that must be true, or a value that must not be empty, for the connector to continue. When it is not met,
the connector stops and the platform stores a data point that holds only a status code instead of the data.

| Key | Status stored when not met |
|---|---|
| `requires` | 404 |
| `requires(<status>)`, for example `requires(424)` | The given status |
| `requires0` … `requires9`, also with a status, for example `requires1(424)` | 404, or the given status |

A YAML mapping cannot contain the same key twice, so use the numbered variants for more than one requirement:

```yaml
requires: << match >>
year1: << max(int((m.replace('-', ' ').strip()).rsplit()[-1]) for m in match) >>
year: << year1 if year1 > 1900 else year1 + 1900 if year1 > 80 else year1 + 2000 >>
requires1: << 1980 <= year <= datetime.date.today().year >>
```

The status codes and what is stored are described in
[Includes and requires](includes-and-requires.md#requires).

### code

A block of Python source, executed at its position in the definition. Functions and names defined or imported in it
are available to the expressions of later keys. The `pagespeed` connector defines a function to summarise a
Lighthouse report:

```yaml
code: |
    def lighthouse_summary(lighthouse: dict) -> dict:
        result = {'audits': {}, 'categories': {}}
        ...
        return result

simple-file:
    summary.json: << lighthouse_summary(data) >>
```

See [Expression language](expressions.md#the-code-block) for the names visible inside the block.

### index

A mapping of fields that are stored in indexed columns of the data form, for filtering, sorting and analysis. Every
field ends in a type conversion, `-> str`, `-> int`, `-> float` or `-> datetime`, and the data form must have a free
column of that type. Fields with an empty value are left out of the stored data point.

```yaml
index:
    a1: << dns.generic.a1 >> -> str
    ns1: << dns.generic.ns1 >> -> str
    mx1: << dns.generic.mx1 >> -> str
```

- A comment after a field, written as `description | example`, becomes the description and example of the field in the
  OpenAPI documentation:

    ```yaml
    final_domain: << data.final_domain >> -> str  # final domain, not necessarily the one the user typed in | example.com
    ```

- A field written as only a type, such as `ipv6: -> float` in `internet_nl`, declares a column without an expression.
  Its value is taken from the field with the same name in the JSON value of the data point, or from data written to
  the data point by the platform.
- `int` columns are small integers: values are limited to the range -32767 to 32767.
- After this key, `index` holds the evaluated fields, so a refresh rule can use `index.category`.

Column mapping, the response format and the change of a field's type are described in
[Storage of results](../storage.md).

### json

A value, usually a mapping, that is stored in the JSON column of the data point (data forms `JsonStrInt` and `Json`).
Its keys appear next to the `index` fields in the `attributes` of the response, so keep the names of both distinct.

```yaml
json: << data >>
```

After this key, the name `json` refers to this value instead of the Python `json` module. Call `json.loads` in a key
above the `json` key.

### file

Files that are written to object storage, with a reference per data point. Each entry maps a file name to a
definition:

| Field | Default | Meaning |
|---|---|---|
| `content` | | Bytes, text, a JSON-serialisable value, a file path or a file object; mappings and lists are stored as JSON |
| `content-type` | Derived from the file extension | MIME type of the file |
| `access` | `private` | `private`, `public` or `by-uuid` |

```yaml
file:
    report.json:
        content: << text >>
        access: private
```

The access levels and the URLs of stored files are described in [Storage of results](../storage.md).

### simple-file

The short form of `file`: a mapping of file name to content. The content type follows from the extension and the
access is `private`.

```yaml
simple-file:
    full.json: << dns >>
```

The files are written when the key is evaluated. Place `file` and `simple-file` after the `requires` keys, so that no
files are written for a connector that stops.

### refresh

The rule for the next scheduled refresh. It is evaluated like any other key, so it can depend on data computed above
it:

```yaml
refresh:
    every: << "3 ~months" if not index.a1 else "1 ~month" >>
```

The values `never` and `on-demand`, and the keys `every`, `crontab` and `at`, are described in
[Refresh rules](refresh.md). Without a `refresh` key the platform's default interval applies.

### properties

Settings that the platform reads from the stored definition. Write them as literal values; expressions are not
evaluated here.

| Property | Value | Meaning |
|---|---|---|
| `min_refresh_time` | Seconds | No refresh is started if the last refresh is more recent, also not for `refresh=now` |
| `max_timestamps` | Number | Maximum number of check timestamps kept for the data point |

```yaml
properties:
    min_refresh_time: 2592000  # 30 days
    max_timestamps: 10
```

### bff

The name of the modifier applied when the data point is requested without one. A modifier reshapes the data point for
a consumer (a backend for frontend); the available modifiers are part of the platform (see the Sitekick platform
documentation). The suffix `/_raw` in the URL returns the plain data point instead. An include requests the data point
by its plain name, so it receives the modified form as well.

```yaml
bff: extra
```

With this line, `.../domainvalue` returns the `extra` view and `.../domainvalue/_raw` the stored data point.

## Intermediate keys

Every other top-level key is an intermediate result. Name it after its content, so that later expressions read
naturally:

```yaml
include: homepage

final_url: << include.homepage.attributes.final_url >>
requires(424): << final_url >>

url: https://api.example.com/v1/urltoimage
```

A key hides a helper of the same name for the keys below it. Do not name keys `functions`, `f`, `get`, `regex`,
`datetime` or `arrow`; see [Expression language](expressions.md#names-available-in-expressions).

## A complete example

The `status` connector for domains combines four other connectors into one verdict:

```yaml
type: StrInt

include:
    - category
    - rtr
    - whois
    - dns

data: << functions.hosting.supercat(registered=include.rtr.attributes, whois=include.whois.attributes, dns=include.dns.attributes, category=include.category.attributes) >>

index:
    category: << data.category >> -> str
    subcategory: << data.subcategory >> -> str
    subscription: << self.attributes.subscription.category >> -> str
    slugs: << self.attributes.subscription.slugs >> -> str

refresh:
    every: << '~month' if index.category != 'active' else '6 ~months' >>
```

1. `type` selects a form with text and small-integer columns.
2. `include` fetches four data points of the same domain in parallel.
3. `data` calls a hosting function with the attributes of the included data points.
4. `index` stores four text fields; `self.attributes` are the domain's own attributes.
5. `refresh` uses `index.category`: inactive domains are checked monthly, active ones twice a year.

More worked examples are in [Recipes](../recipes.md) and the [catalogue](../catalogue/index.md).
