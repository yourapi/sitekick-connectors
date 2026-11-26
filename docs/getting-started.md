---
title: Writing your first connector
---

# Writing your first connector

This page takes you through one connector from start to finish: you choose where the file goes, write a small
definition, load it into a development or test environment of the Sitekick platform, request its data point for
`example.com`, check what is stored, and propose the connector through a pull request. The terms are explained in
[Concepts](concepts.md).

## 1. Choose the folder and the file name

The path of a file is the identity of the connector:

```text
{client}/{item type group}/{item type}/{connector}.yaml
sitekick/hosting/domain/landing_page.yaml   ->  sitekick/hosting/domain/landing_page
```

- **Client.** Place a connector that every brand may use under `sitekick/`, the root client. A connector for one
  brand goes in that brand's folder, and is then available to that brand and the clients below it.
- **Item type.** The folder of the item type decides for which items the connector runs: `hosting/domain` for
  domains, `hosting/server` for servers. The folder's `_entity.yaml` lists the key names, so a domain connector reads
  the domain name as `params.domain`.
- **Connector name.** The file name without `.yaml` is the connector name and the last segment of its URL. Use lower
  case and underscores, and pick a name that no other file in the folder uses; a file with the name of an inherited
  connector overrides it for your client. Files whose name starts with `_` are not loaded as connectors.

See [File location and name](language/yaml-reference.md#file-location-and-name) for the details.

## 2. Write a minimal connector

The example below is a reduced form of the `homepage` connector. It resolves the homepage of the domain and stores
three fields:

```yaml
type: StrInt
description: Final address of the homepage, and whether it uses https

data: << functions.hosting.homepage(params.domain) >>

requires: << data.final_url >>   # no homepage: store status 404 and stop

index:
    final_url: << data.final_url >> -> str  # address after all redirects | https://www.example.com/
    https: << data.https >> -> int  # 1 if the final address uses https | 1
    redirects: << data.redirect_number >> -> int  # number of redirects followed | 1

refresh:
    every: ~month
```

The keys are evaluated from top to bottom:

1. `type` selects the data form `StrInt`, with four text and four small-integer columns (see
   [Data forms](storage.md#data-forms)).
2. `data` is an intermediate key: it calls the platform function `functions.hosting.homepage` and keeps the result for
   the keys below. It is not stored.
3. `requires` stops the connector when there is no final address; the platform then stores only the status 404.
4. `index` stores three typed fields. The cast after each expression (`-> str`, `-> int`) is required and selects the
   column. The comment after a field, written as `description | example`, becomes its description in the API
   documentation.
5. `refresh` asks for a new evaluation once a month, at a random moment.

Every keyword is described in the [Connector YAML reference](language/yaml-reference.md), and the expressions between
`<<` and `>>` in [Expression language](language/expressions.md).

## 3. Load it into a development or test environment

Load a new connector into a development or test environment of the platform first, never straight into production.
The platform's connector loader does this for every file in the repository: it takes the four parts of the path as the
connector name and posts that name, with the unchanged YAML text, to the platform's `/data-types` endpoint. Use the
loader, or post the single file in the same way, with a token that may create connectors. If you do not operate the
environment yourself, ask its operator to load your branch.

When the platform accepts the definition, it stores it as a new version of the connector if it differs from the
stored one. When it rejects it, it
answers with status `422` and a message that names the problem, for example an `index` field without a cast, or more
`-> str` fields than the data form has text columns. Correct the file and load it again.

## 4. Request the data point

Request the data point for `example.com` through a client that inherits the connector, such as `yourhosting`, and
force a fresh evaluation with `refresh=now`:

```http
GET /yourhosting/hosting/domain/example.com/landing_page?refresh=now HTTP/1.1
Host: api.example.com
Authorization: Bearer <token>
```

- Only a token with create rights on the path starts a refresh. If the domain item does not exist yet, such a token
  also creates it.
- The response holds the indexed fields under `attributes`, and the moment of the refresh under
  `meta.refreshed_at`.
- If the refresh takes longer than the platform waits, the answer is `202 Accepted`; request the data point again,
  without `refresh=now`, to read the result.
- If the requirement is not met, the answer has status 404 and the data point holds only that status. See
  [Includes and requires](language/includes-and-requires.md#requires).

## 5. Check the stored history

The `history` modifier returns every stored version of the data point:

```http
GET /yourhosting/hosting/domain/example.com/landing_page/history HTTP/1.1
Host: api.example.com
Authorization: Bearer <token>
```

Request the data point once more with `refresh=now` and look at the history again. When the stored fields have not
changed, there is still one version: the platform records only the moment of the check. A new version appears only
when a stored value changes. Keep volatile values out of `index` and `json`, or round them, so the history stays
compact (see [History](storage.md#history)).

Try a domain without a website as well, to see the status-only data point, and check that the refresh rule suits the
source you call.

## 6. Propose the change

Changes to this repository are proposed through pull requests:

1. Create a branch, add your file, and commit it. Never commit a secret in plain text; an API key goes into the file
   encrypted (see [Secrets in connectors](secrets.md)).
2. Open a pull request. Describe what the connector stores, which functions and external services it calls, its
   refresh rule, and the domains you tested it with.
3. The pull request is reviewed before it is merged; a reviewer can load the branch into a test environment to check
   the result.

## Next steps

- [Recipes](recipes.md): complete connectors for common tasks, such as DNS checks, files, includes and API keys.
- [Includes and requires](language/includes-and-requires.md): build on the data points of other connectors.
- [Refresh rules](language/refresh.md): control when a data point is computed again.
- [Connector catalogue](catalogue/index.md): the existing connectors, to reuse or to learn from.
