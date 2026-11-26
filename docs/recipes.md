---
title: Recipes
---

# Recipes

Each recipe is a complete connector for a common task, modelled on connectors in this repository. Copy one into the
folder of its item type, for example `sitekick/hosting/domain/<name>.yaml`, and adapt it. The keywords are described in
the [YAML reference](language/yaml-reference.md), the expressions in the [expression language](language/expressions.md).

All recipes are domain connectors, so the domain is available as `params.domain`.

## A simple HTTP-based check

Request the homepage and store the status code, the final URL and the server software.

```yaml
type: StrInt
description: HTTP status, final URL and server software of the homepage

response: << @suppress get('https://' + params.domain + '/', timeout=20) >>
requires: << response >>

index:
    status_code: << response.status_code >> -> int  # HTTP status of the final response | 200
    final_url: << str(response.url) >> -> str  # URL after all redirects | https://www.example.com/
    server: << response.headers.get('server') >> -> str  # value of the Server header | nginx
    redirected: << 1 if response.history else 0 >> -> int  # 1 if one or more redirects were followed | 1

refresh:
    every: ~month
```

`get` is the platform's asynchronous HTTP client; it follows redirects, and the evaluator awaits the call itself. If
the website cannot be reached, the request raises an exception; `@suppress` turns that into an empty value, and
`requires` then stores status 404 instead of failing the refresh. The comment after each index field, written as
`description | example`, ends up in the API description of the field. For real websites, the `homepage` connector
does this work with more care, through `functions.hosting.homepage`.

## A DNS-based check

Determine whether a domain receives its mail through Google Workspace, from its MX records.

```yaml
type: StrInt
description: Whether the mail of the domain is handled by Google Workspace

dns: << functions.hosting.nslookup(params.domain, 'mx') >>

requires: << dns.generic.mx >>   # no MX records: 404

index:
    mx1: << dns.generic.mx1 >> -> str  # first mail server | aspmx.l.google.com
    mx1_short: << dns.generic.mx1_short >> -> str
    google: << any(mx.endswith(('.google.com', '.googlemail.com')) for mx in dns.generic.mx) >> -> int

refresh:
    every: << '~week' if not index.google else '3 ~months' >>
```

`functions.hosting.nslookup` returns the records per type, and a section `generic` with, among others, `mx`: the list
of mail servers in order of priority, `mx1`: the first of them, and `mx1_short`: its generic name. These are the same
fields that the `dns` connector stores in its file `full.json` (see [dns](catalogue/hosting-domain.md#dns)). Missing
values propagate as empty values, so `dns.generic.mx` is empty for a domain without MX records, or one that does not
exist. The `m365` incentive connector uses the same pattern for
Microsoft 365. Earlier keys are used only in the first `for` clause of the generator expression, as the expression
language requires.

## A derived pass/fail check

Combine two existing connectors into one verdict, without fetching anything new.

```yaml
type: StrInt
description: Pass/fail check, the website uses https with a valid certificate

include:
    - homepage
    - certificate

requires(424): << include.homepage.attributes.final_url >>

index:
    https: << include.homepage.attributes.https >> -> int
    issuer: << include.certificate.attributes.issuer >> -> str
    invalid: << include.certificate.attributes.invalid >> -> str
    valid: << 1 if include.homepage.attributes.https and include.certificate.attributes.common_name and not include.certificate.attributes.invalid else 0 >> -> int

refresh:
    every: << 'day' if not index.valid else '3 months' >>
```

`include` fetches the `homepage` and `certificate` data points of the same domain in parallel, computing them first if
needed. Without a homepage there is nothing to judge, so the connector stores 424 (failed dependency). A domain
without a certificate has an empty `include.certificate.attributes`, and `valid` becomes 0. The refresh rule checks a
failing domain daily and a passing one every three months. The incentive connectors
`your.online/commercial/incentive/ssl`, `website` and `m365` follow the same pass/fail pattern with a `valid` field.
They run on incentive items rather than domains, so they compute their checks directly instead of including domain
connectors.

## A connector that stores files

Check the links on the homepage, index the counts, and keep the full result as files.

```yaml
type: StrInt
description: Valid and broken links on the homepage

include: homepage

requires(424): << include.homepage.attributes.final_url >>

links: << functions.hosting.link_check(include.homepage.attributes.final_url) >>

index:
    valid: << len([link for link in links if link.get('valid')]) >> -> int
    invalid: << len([link for link in links if not link.get('valid')]) >> -> int

file:
    full.json:
        content: << links >>
    broken.json:
        content: << [link for link in links if not link.get('valid')] >>
        content-type: application/json
        access: by-uuid
```

`file` writes each entry to object storage. A list or mapping is stored as JSON; the content type follows from the
suffix, so `content-type` is only needed when the suffix does not tell. `by-uuid` stores the file under the item's
UUID instead of its id. The data point lists the files under `links.files`, and
`.../example.com/<connector>/file/full.json` returns one. When all files are private and need no explicit content type,
the short form `simple-file`, a mapping of file name to content, is enough; the `links` connector uses it.

## A scheduled refresh

Control when a data point is computed again, how often a refresh may be forced, and how many check moments are kept.

```yaml
type: StrDateInt
description: Registrar and expiry date of the domain

data: << functions.hosting.whois(params.domain) >>

requires: << data.registrar >>

index:
    registrar: << data.registrar >> -> str
    expiration_date: << @suppress data.expiration_date.date() >> -> datetime

refresh:
    every: << '~week' if not index.expiration_date else '1 ~month@11-23' >>

properties:
    min_refresh_time: 86400   # seconds: a data point younger than a day is not refreshed again
    max_timestamps: 20        # keep the 20 most recent check moments
```

`every: 1 ~month@11-23` picks a random moment within the next month, between 11:00 and 23:00, which spreads the load
over the month and the day. The rule is an expression that uses `index.expiration_date`, computed one key earlier:
without an expiry date the domain is checked weekly. `min_refresh_time` also applies to `refresh=now`, which protects
rate-limited sources. `properties` are read from the stored definition, so they hold literal values. Other rules are
`crontab: "0 3 * * mon"` (every Monday at 03:00), `at` (a fixed moment), `never` and `on-demand`; see
[Refresh rules](language/refresh.md).

## A connector that requires another

Build on data that another connector stores as a file, and stop with a clear status when it is missing.

```yaml
type: StrInt
description: Title of the homepage, from the stored homepage content

include:
    - homepage
    - content

requires(424): << include.homepage.attributes.final_url and include.content.links.files >>

content: << get(include.content.links.files['full.json']).json() >>

title: << @suppress(TypeError) regex.search(r'<title[^>]*>(.*?)</title>', content.html, regex.I | regex.DOTALL)[1] >>

requires1: << title >>   # no title in the page: 404

index:
    title: << title.strip()[:200] >> -> str
    length: << len(title.strip()) >> -> int
```

The `content` connector stores the text and HTML of the homepage in its file `full.json`; this connector reads that
file through the link in the included data point. `requires(424)` stops with "failed dependency" when the homepage or
its content is missing, and `requires1`, a second requirement, stops with 404 when the page has no title. A YAML
mapping cannot hold the same key twice, hence the numbered variants `requires0` to `requires9`. `@suppress(TypeError)`
turns the failed lookup on a missing match into an empty value. The `copyright` and `google_analytics` connectors work
in the same way.

## A connector with a default view

Name a view (a backend for frontend, BFF) that is applied when the data point is requested without a modifier.

```yaml
type: FloatInt
description: Estimated value of the domain

include:
    - domdetailer
    - keywords
    - whois

result: |
    << functions.hosting.domain_value(
        params.domain,
        include.whois.attributes,
        get(include.keywords.links.files['full.json']).json(),
        include.domdetailer.attributes) >>

index:
    value: << result.domainvalue_num >> -> float
    value_rounded: << result.domainvalue_num_rounded >> -> float

bff: extra
```

This is a shortened `domainvalue`. With `bff: extra`, a request for `.../example.com/domainvalue` returns the `extra`
view; `.../example.com/domainvalue/_raw` returns the plain data point, and `.../example.com/domainvalue/youradvice`
another view. Views are implemented in the platform: the view `<modifier>` of connector `<connector>` is the method
`<connector>_<modifier>` of the platform's modifier class. A connector can also exist only to configure a view, as
`hubspot` does:

```yaml
type: Empty

params:
    timeout: 8
    status: 503

refresh: on-demand
```

`.../example.com/hubspot/<connector>` requests the named connector and waits at most `params.timeout` seconds; on a
timeout it answers with `params.status`. This suits consumers that cannot wait long.

## A connector with an API key

Call an external API that needs a key, without exposing the key.

```yaml
type: StrInt
description: Reputation of the domain at an external reputation service

url: << f"https://api.example.com/v1/domains/{params.domain}/reputation" >>

headers:
    Authorization: "Bearer <<<--<encrypted>-->>>"
    Accept: application/json

data: << get(url, headers=headers).json() >>

requires: << data >>

index:
    listed: << 1 if data.listed else 0 >> -> int  # 1 if the domain is on a blocklist | 0
    score: << data.score >> -> int  # reputation score, 0-100 | 87
    category: << data.category >> -> str  # category assigned by the service | business

refresh:
    every: ~month
```

The key is encrypted between `<<<--` and `-->>>`; the platform decrypts it in memory when it loads the definition.
`headers` is an intermediate key, so the key is used in the request but never stored. Only the fields in `index` are
stored. Replace `<<<--<encrypted>-->>>` with the value produced by the platform's encryption tool; see
[Secrets in connectors](secrets.md). The `rtr` connector calls its registry API in the same way.

## A connector with request parameters

Let the caller choose what to check, and compute the result on every request.

```yaml
type: Json
description: >-
    Availability of the domain name with other top-level domains. The TLDs to check can be given in the
    query parameter `tlds`, separated by spaces.

params:
    tlds: "eu com be de me net nu org online"

json:
    availability: << functions.hosting.available_tlds(params.domain, params.tlds) >>

refresh: on-demand
```

This is the `availability` connector. `params` holds the defaults; a request such as
`.../example.com/availability?tlds=com%20org` overrides them. With `refresh: on-demand` the data point is computed on
every request, and the request waits for the result. The result is stored in `json`; the data form `Json` has no
indexed columns.
