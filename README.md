# sitekick-connectors

Connector definitions for the Sitekick hosting intelligence platform. A connector is a YAML file that tells the
platform how to obtain one data point about an item (for example the DNS records, the certificate or the CMS of a
domain), how to store the result and when to refresh it. The platform loads the definitions from this repository; the
Python functions that connectors call live in the platform.

## Documentation

The documentation is published at <https://yourapi.github.io/sitekick-connectors/> and is built from `docs/` with
MkDocs Material:

- **Getting started**: writing your first connector, and recipes for common tasks.
- **Concepts**: items, item types, connectors, data points, clients, includes, refresh, history and storage.
- **Connector language**: the YAML reference, the expression language, includes and requires, refresh rules, secrets.
- **Catalogue**: every connector in this repository.

To preview the site locally (the configuration file is `_mkdocs.yml`: the platform's connector loader skips YAML
files whose name starts with an underscore):

```bash
uvx --with-requirements requirements-docs.txt mkdocs serve --config-file _mkdocs.yml
```

## Repository layout

Each file is stored as `<client>/<item type group>/<item type>/<connector>.yaml`:

| Folder | Contents |
|---|---|
| `sitekick/` | Connectors of the root client, available to all clients: domain, server and network checks, settings, public data |
| `your.online/` | Connectors of the Your.Online group and its brands: incentives, reports, contacts, products, subscriptions |
| `argeweb/` | Connectors specific to the Argeweb brand |
| `parker/` | Legacy item types |
| `docs/` | The documentation site |
