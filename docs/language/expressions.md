---
title: Expression language
---

# Expression language

The values in a connector are literal YAML values or Python expressions between `<<` and `>>`. This page describes how
expressions are evaluated, which names they can use, how values are converted and how errors are handled. The
keywords themselves are listed in the [Connector YAML reference](yaml-reference.md).

## Expressions

A string that contains `<< ... >>` is an expression. The text between the markers is evaluated as a Python
expression; a string without the markers is a literal value.

```yaml
url: https://api.example.com/v1/urltoimage                     # literal string
data: << functions.hosting.homepage(params.domain) >>          # expression: a mapping
https: << 1 if data.final_url.startswith('https') else 0 >>    # expression: an integer
```

| Form | Result |
|---|---|
| `<< expr >>` | The value of the expression, with its own type: a number, text, list, mapping or response object |
| `<< expr >> -> type` | The value converted to `type` (see [Type conversion](#type-conversion)) |
| `text << expr >> text` | A string: the text with the value of the expression filled in |
| `text << a >> text << b >>` | A string with several expressions filled in |

```yaml
url: << f"https://www.googleapis.com/pagespeedonline/v5/runPagespeed?url={include.homepage.attributes.final_url}" >>
all: host is << test1b.headers.Host >> and UA is << test1c.headers['User-Agent'] >> !
```

- **Multi-line expressions.** An expression can span several lines. Use a YAML block scalar for long expressions:

    ```yaml
    result: |
        << functions.hosting.domain_value(
            params.domain,
            include.whois.attributes,
            get(include.keywords.links.files['full.json']).json(),
            include.domdetailer.attributes) >>
    ```

- **YAML quoting.** Quote the whole value when the expression contains `: ` or ` #`, or would otherwise be read as YAML
  syntax, as `category` does: `content: "<< get(...) if include.content.links.files else {} >>"`.
- **The closing marker.** The expression ends at the first `>>`, so an expression cannot contain `>>` itself.
- **Nested values.** Mappings and lists are evaluated element by element, at any depth. A mapping that results from an
  expression supports attribute access: `data.final_url` is the same as `data['final_url']`.

## Type conversion

`-> type` after an expression converts its value. The conversion applies when the value consists of only the
expression and the conversion, without text before `<<`.

| Type | Conversion | Use in `index` |
|---|---|---|
| `str` | `str(value)` | Text column |
| `int` | `int(value)`, limited to -32767 … 32767 | Small-integer column |
| `float` | `float(value)` | Float column |
| `datetime` | Parsed as a datetime: a `datetime` object or an ISO 8601 string | Datetime column |
| `date` | Parsed as a date | Not available |
| `time` | Parsed as a time | Not available |
| `bool` | `bool(value)` | Not available |

- If `str`, `int`, `float` or `bool` cannot convert the value, the result is empty, and the field is left out.
- `datetime`, `date` and `time` require a parseable value; any other value raises an error.
- `int` drops the decimals of a float. Use `round()` where that matters, as `timing` does with
  `min(round(data.content_size / 1024), 32767)`.
- In the `index` section the conversion also selects the column type, and it is required for every field.

## Empty values

An expression whose value is `None` gives an empty value. Empty values are left out of the stored result, so an index
field with an empty value is simply absent from the data point.

Missing data is empty as well, and stays empty when you access it further:

- a key that does not exist in a mapping, such as `include.dns.attributes.a1` when `dns` has no `a1`;
- an attribute or item of an empty value, such as `data.redirects[0]` when `redirects` is missing;
- an index beyond the end of a list.

An empty value is false, has length 0, iterates as an empty list, converts to `''`, `0` or `0.0`, and compares as
`0` or `''`. Calling a method on it gives an empty value as well. Expressions such as
`','.join(include.whois.attributes.name_servers)`, `include.a.attributes.a1 >= 1` or
`data.creation_date.date()` therefore work without a guard when the data is missing. Use `@suppress` for data that is
present but has an unexpected form.

When the first expression in a string with surrounding text is empty, the whole value is empty, not a string with a
gap.

## Suppressing errors

`@suppress` at the start of an expression turns an exception during evaluation into an empty value:

```yaml
index:
    name_server: << @suppress data.name_servers_generic[0] >> -> str
    creation_date: << @suppress data.creation_date.date() >> -> datetime
    spf_all: << @suppress(TypeError) regex.search(r'[-~?+]all', spf)[0] >> -> str
```

| Form | Suppresses |
|---|---|
| `@suppress expr` | Every exception (`Exception`) |
| `@suppress(KeyError) expr` | Only the named exception |
| `@suppress(KeyError, TypeError) expr` | Any of the named exceptions |

`@suppress` hides real errors as well. Name the expected exception, and use it only where a missing value is normal.
The type conversion after `->` takes place after the suppression and is not covered by it.

## Names available in expressions

| Name | Content |
|---|---|
| `params` | The parameters of the request, the key names of the item type (such as `params.domain`), and from the `params` key on, the defaults of the connector |
| `self`, `item` | The item the connector is evaluated for (`this` and `parent` are synonyms) |
| `include` | The included data points, from the `include` key on (see [Includes and requires](includes-and-requires.md)) |
| Earlier keys | The result of every top-level key above the current one, under its name, such as `data`, `index` or `json` |
| `functions`, `f` | The library of hosting functions of the platform |
| `get` | Asynchronous HTTP GET; returns a response object |
| `regex` | The `regex` module, a superset of Python's `re` |
| `json` | Python's `json` module, until a key named `json` is evaluated |
| `datetime` | Python's `datetime` module, for example `datetime.date.today()` |
| `arrow` | The `arrow` date and time library, for example `arrow.now()` |
| Builtins | Python's built-in functions, such as `len`, `min`, `round`, `any`, `next`, `int` and `str` |

### The item

`self` (also `item`, `this` and `parent`) holds the item in the JSON:API form of the platform:

| Expression | Content |
|---|---|
| `self.data.key` | The natural key, such as `example.com` |
| `self.data.id` | The numeric id |
| `self.data.type` | The item type path, such as `yourhosting/hosting/domain` |
| `self.attributes.<name>` | An attribute of the item, such as `self.attributes.subscription.category` in `status`, or `self.attributes.domain` in the incentive connectors |
| `self.meta.created_at` | When the item was created |
| `self.links.self` | The URL of the item in the API |
| `self.relationships.<name>` | The related items under a relation name, see [Entities and relations](../entities-and-relations.md#reading-relations) |
| `params.<key name>` | The key, under each name in `key-names` of the item type |

For the key of the item, prefer the key name: `params.domain` for a domain.

### Parameters

`params` merges, from low to high precedence: the defaults from the `params` key, the key names of the item type,
and the query parameters of the request. Query parameters arrive as text:

```yaml
params:
    seconds: 10

code: |
    from asyncio import sleep

sleep: << await sleep(float(params.seconds)) >>
```

A request with `?seconds=2` waits two seconds; without it, ten.

### The functions library

The hosting functions of the platform are available by their dotted name, without an import, as
`functions.<package>.<function>` or the shorthand `f.<package>.<function>`:

```yaml
dns: << functions.hosting.nslookup(params.domain, params.dns_type, substitutions) >>
data: << f.http.request.get(url, headers=header).json() >>
```

The functions themselves are part of the Sitekick platform and are documented there.

### HTTP requests

`get(url, params=..., headers=...)` sends a GET request and returns the response, with `.json()`, `.text`, `.content`
and `.status_code`. The other methods are in `f.http.request`: `post`, `put`, `patch`, `delete`, `head` and `options`.

```yaml
data_raw: << f.http.request.post(url, data=body, headers=headers).json() >>
```

`get` also reads the files of included data points, through their link:
`<< get(include.content.links.files['full.json']).json() >>`.

### Names of keys

A key hides a helper of the same name for the keys below it. After a `json` key, `json` is that value, so call
`json.loads` above it. The names to keep free are listed in
[Reserved names](yaml-reference.md#reserved-names).

## Asynchronous calls

Many functions, such as `get` and the HTTP functions, are asynchronous. The evaluator detects calls to asynchronous
functions of the library and awaits them, so you write plain expressions:

```yaml
text: << get(url).text >>
```

- **One asynchronous call per expression.** Give each network call its own key, as `keywords` does with `data_raw`
  before converting it in `data`. To run calls in parallel, use a hosting function or a list of includes.
- **Explicit `await`.** You can write `await` yourself. It is needed for asynchronous functions that the evaluator
  does not know, such as functions imported or defined in a `code` block:

    ```yaml
    code: |
        from asyncio import sleep

    sleep: << await sleep(float(params.seconds)) >>
    ```

## The code block

A `code` key holds Python source, executed when the key is reached. It is typically used to import modules and to
define helper functions:

```yaml
code: |
    month_map = {datetime.date(2000, i, 1).strftime('%B'): i for i in range(1, 13)}

    def convert_month(d):
        if isinstance(d, dict):
            return {k: convert_month(v) for k, v in d.items()}
        elif isinstance(d, list):
            return [convert_month(e) for e in d]
        elif isinstance(d, str):
            return month_map.get(d, d)
        else:
            return d

data: << convert_month(functions.convert.convert_type(data_raw, int, float)) >>
```

- Names defined or imported in the block are available to the expressions of later keys.
- The block runs in the global namespace of the expressions: it sees the helpers (`functions`, `f`, `get`, `regex`,
  `json`, `datetime`, `arrow`) and earlier `code` blocks, but not the results of earlier keys, `params` or `self`.
  Pass those to your functions as arguments, as `lighthouse_summary(data)` in `pagespeed` does.
- An import in the block makes a module available, for example `from urllib.parse import quote` in `screenshot`.

## Scope in comprehensions and lambdas

Expressions are evaluated with two namespaces: the helpers are global, while `params`, `self`, `include` and earlier
keys are local. Python resolves names inside comprehensions, generator expressions and lambdas in the global namespace,
except for the iterable of the first `for` clause. Use earlier keys only there:

```yaml
links: << functions.hosting.link_check(include.homepage.attributes.final_url) >>

index:
    valid: << len([link for link in links if link.get('valid')]) >> -> int   # links: first for clause
```

Compute any other value that a comprehension needs in a key of its own, and iterate over that key.

## Errors

An exception in an expression, that is not suppressed, stops the evaluation and the refresh fails. The stored values
of the data point are not changed. The platform records the exception with the refresh and retries it later with
increasing intervals, so a corrected connector is picked up on a later attempt. The response to a request that waited
for the refresh reports that the refresh failed; the details are available to administrators of the platform.

An unmet `requires` is not an error: the connector stops and stores a status, as described in
[Includes and requires](includes-and-requires.md#requires).

## Summary of rules

- Order keys by dependency: a key can only use the keys above it.
- Make one asynchronous call per expression; write `await` for functions from a `code` block.
- Use `@suppress` with a named exception, only where a missing value is normal.
- Give every `index` field a type conversion that fits a free column of the data form.
- Keep the [reserved names](yaml-reference.md#reserved-names) free; mind that a `json` key replaces the `json`
  module.
- In comprehensions and lambdas, use earlier keys only in the first `for` clause.
- Never put secrets in plain text; see [Secrets in connectors](../secrets.md).
