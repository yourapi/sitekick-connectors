---
title: Connector catalogue
---

# Connector catalogue

This catalogue lists the connector definitions in this repository. The detailed descriptions are on two pages:

- [Domain connectors](hosting-domain.md): the connectors of the item type `hosting/domain`, the largest set.
- [Other connectors](other.md): servers, settings, the incentive programme, subscriptions, products, customers, and the
  development and test connectors.

## Reading the catalogue

The path of a file is its identity. `sitekick/hosting/domain/dns.yaml` is the connector `dns` of the item type
`hosting/domain` of the client `sitekick`. A file named `_entity.yaml` defines the item type itself: its key names, its
own attributes and its relations (see [Entities and relations](../entities-and-relations.md)). Other files whose name
starts with `_` are not loaded.

Clients form a hierarchy with `sitekick` at the root, and a client inherits the connectors of the clients above it.
The domain connectors under `sitekick/` are therefore available to every brand, such as `yourhosting` or `argeweb`. In
the API you address a connector through the brand:

```http
GET https://api.example.com/yourhosting/hosting/domain/example.com/dns
```

The columns of the tables below:

| Column | Meaning |
|---|---|
| Path | Path of the YAML file, relative to the client folder |
| Connector | Name of the connector; `_entity` for an item type definition |
| Purpose | What the connector provides |
| Refresh | The refresh rule (see [Refresh rules](../language/refresh.md)); "default" means the platform's default interval, every six months at a random moment unless the platform is configured otherwise; "API" means the data is set through the API and not refreshed by a source |
| Status | See below |

| Status | Meaning |
|---|---|
| production | Part of the production set of connectors |
| planned | The definition describes an intended use: its purpose starts with "Designed for". It is not part of the production set |
| utility | Supports other connectors or requests; does not collect data about an item |
| test | Used to test the connector engine, in development and test environments; no business data |
| legacy | Kept for reference from an earlier use of the platform; not extended |

A purpose that starts with "Designed for" describes the intended use of the connector.

### Pseudo-connectors

`multi` and `hubspot` are pseudo-connectors. They do not collect data, but act as prefix modifiers: pseudo-connectors
that precede the connector name in the URL, while most modifiers follow it (see
[Modifier](../concepts.md#modifier)):

```text
/yourhosting/hosting/domain/example.com/multi/dns,whois,ssl
/yourhosting/hosting/domain/example.com/hubspot/pagespeed
```

`multi` requests the listed connectors in parallel and returns their results in one object, keyed by the connector
names as written in the URL, each with its modifier, if any.
`hubspot` requests the connector within the timeout set in its definition, and answers with the fallback status of its
definition when the connector takes longer. See [multi](hosting-domain.md#multi) and
[hubspot](hosting-domain.md#hubspot).

## Client `sitekick`

### `hosting/domain`

| Path | Connector | Purpose | Refresh | Status |
|---|---|---|---|---|
| `hosting/domain/_entity.yaml` | [`_entity`](hosting-domain.md#_entity) | Item type domain: key name `domain`, attributes `tld` and `sld`, relations to platform and server | never | production |
| `hosting/domain/homepage.yaml` | [`homepage`](hosting-domain.md#homepage) | Resolves the domain like a browser: first responding URL, final URL, redirects, https | default | production |
| `hosting/domain/detailed_homepage.yaml` | [`detailed_homepage`](hosting-domain.md#detailed_homepage) | Tries `http`/`https` with and without `www.`: https support, redirects, `www.` set-up | default | production |
| `hosting/domain/timing.yaml` | [`timing`](hosting-domain.md#timing) | Response times of the homepage: time to first and last byte, connection time, size, speed | `~month` | production |
| `hosting/domain/links.yaml` | [`links`](hosting-domain.md#links) | Number of valid and invalid links of the website, with the full link list | default | production |
| `hosting/domain/screenshot.yaml` | [`screenshot`](hosting-domain.md#screenshot) | Full-page screenshots and thumbnails, desktop and mobile | default | production |
| `hosting/domain/certificate.yaml` | [`certificate`](hosting-domain.md#certificate) | TLS certificate of the domain: issuer, validity period, common name, reason when invalid | default | production |
| `hosting/domain/certificate_www.yaml` | [`certificate_www`](hosting-domain.md#certificate_www) | As `certificate`, for the `www.` host name | default | production |
| `hosting/domain/ssl.yaml` | [`ssl`](hosting-domain.md#ssl) | Combined https and certificate view of the domain | default | production |
| `hosting/domain/pagespeed.yaml` | [`pagespeed`](hosting-domain.md#pagespeed) | Google PageSpeed Insights (Lighthouse) scores, with the full report and a summary | default | production |
| `hosting/domain/content.yaml` | [`content`](hosting-domain.md#content) | Content of the homepage with text analysis, stored as a file | default | production |
| `hosting/domain/category.yaml` | [`category`](hosting-domain.md#category) | Category of the website (`active`, `parked`, `forsale`, ...) from its content | default | production |
| `hosting/domain/status.yaml` | [`status`](hosting-domain.md#status) | Overall status: registration, whois, DNS and website category, with the subscription | `~month`, `6 ~months` when active | production |
| `hosting/domain/cms.yaml` | [`cms`](hosting-domain.md#cms) | CMS of the website, its version and security status | default | production |
| `hosting/domain/copyright.yaml` | [`copyright`](hosting-domain.md#copyright) | Most recent copyright year on the homepage | default | production |
| `hosting/domain/google_analytics.yaml` | [`google_analytics`](hosting-domain.md#google_analytics) | Google Analytics or Tag Manager on the homepage, with type and tracking id | default | production |
| `hosting/domain/seo.yaml` | [`seo`](hosting-domain.md#seo) | Designed for an SEO analysis of the homepage | default | planned |
| `hosting/domain/dns.yaml` | [`dns`](hosting-domain.md#dns) | DNS records: first A record and its host name, name server, mail server; full lookup as a file | `1 ~month`, `3 ~months` without A record | production |
| `hosting/domain/whois.yaml` | [`whois`](hosting-domain.md#whois) | Registrar, name servers, creation and expiration date | default | production |
| `hosting/domain/hosting_by_dns.yaml` | [`hosting_by_dns`](hosting-domain.md#hosting_by_dns) | Compares the name servers in whois with those in DNS | default | production |
| `hosting/domain/internet_nl.yaml` | [`internet_nl`](hosting-domain.md#internet_nl) | internet.nl scores for IPv6, DNSSEC and https, with a total | `~month` | production |
| `hosting/domain/rtr.yaml` | [`rtr`](hosting-domain.md#rtr) | Real-time registry check: is the domain available for registration | default | production |
| `hosting/domain/availability.yaml` | [`availability`](hosting-domain.md#availability) | Availability of the domain name under other TLDs | on-demand | production |
| `hosting/domain/domdetailer.yaml` | [`domdetailer`](hosting-domain.md#domdetailer) | Backlink and popularity metrics (Majestic, Moz) | default | production |
| `hosting/domain/keywords.yaml` | [`keywords`](hosting-domain.md#keywords) | Search volume, cost per click and trend of the keywords in the domain name | default | production |
| `hosting/domain/domainvalue.yaml` | [`domainvalue`](hosting-domain.md#domainvalue) | Estimated financial value of the domain name | default | production |
| `hosting/domain/plesk.yaml` | [`plesk`](hosting-domain.md#plesk) | Hosting data of the domain from its Plesk server: PHP, mailboxes, WordPress | never | production |
| `hosting/domain/subscription.yaml` | [`subscription`](hosting-domain.md#subscription) | Links a pushed subscription to the domain | never | production |
| `hosting/domain/multi.yaml` | [`multi`](hosting-domain.md#multi) | Pseudo-connector: several connectors in one request | on-demand | utility |
| `hosting/domain/hubspot.yaml` | [`hubspot`](hosting-domain.md#hubspot) | Pseudo-connector: a connector with a time limit, for HubSpot | on-demand | utility |
| `hosting/domain/sleep.yaml` | [`sleep`](hosting-domain.md#sleep) | Waits a number of seconds | on-demand | test |
| `hosting/domain/test.yaml` | [`test`](hosting-domain.md#test-and-test3) | Exercises expressions, HTTP calls and nested values | default | test |
| `hosting/domain/test3.yaml` | [`test3`](hosting-domain.md#test-and-test3) | As `test`, with interpolated strings | default | test |
| `hosting/domain/test-req-1.yaml` | [`test-req-1`](hosting-domain.md#test-req-1-test-req-2-and-test-req-3) | A requirement driven by a parameter | default | test |
| `hosting/domain/test-req-2.yaml` | [`test-req-2`](hosting-domain.md#test-req-1-test-req-2-and-test-req-3) | Includes `test-req-1` | default | test |
| `hosting/domain/test-req-3.yaml` | [`test-req-3`](hosting-domain.md#test-req-1-test-req-2-and-test-req-3) | Includes `test-req-1` and `test-req-2` in parallel | default | test |

### Other item types of `sitekick`

| Path | Connector | Purpose | Refresh | Status |
|---|---|---|---|---|
| `hosting/server/_entity.yaml` | [`_entity`](other.md#servers) | Item type server: identified by IP address; name, brand, CPUs, memory, disks; relation to platform | never | production |
| `hosting/server/ipapi.yaml` | [`ipapi`](other.md#servers) | Designed for network and location data for the server's IP address | default | planned |
| `hosting/server/info.yaml` | [`info`](other.md#servers) | Designed for general server information | default | planned |
| `hosting/server/plesk.yaml` | [`plesk`](other.md#servers) | Designed for Plesk data pushed by the server: host name, IP and MAC address | never | planned |
| `hosting/server/server.yaml` | [`server`](other.md#servers) | Designed for uptime, memory and CPU data pushed by the server | default | planned |
| `hosting/network/_entity.yaml` | [`_entity`](other.md#networks-and-platforms) | Item type network, without attributes of its own | — | production |
| `hosting/platform/_entity.yaml` | [`_entity`](other.md#networks-and-platforms) | Item type hosting platform; its relations are declared by domains and servers | — | production |
| `global/setting/_entity.yaml` | [`_entity`](other.md#global-settings) | Item type for global settings, such as the state of a synchronisation | never | production |
| `public/get/_entity.yaml` | [`_entity`](other.md#public-item-types) | Item type in the group `public`, with free JSON attributes | never | production |
| `public/put/_entity.yaml` | [`_entity`](other.md#public-item-types) | Item type in the group `public`, with free JSON attributes | never | production |
| `dev/exception/_entity.yaml` | [`_entity`](other.md#development-and-test-connectors) | Test item type for error handling | never | test |
| `dev/exception/error.yaml` | [`error`](other.md#development-and-test-connectors) | Designed to test error handling: sample values or a chosen exception | default | test |
| `dev/require/_entity.yaml` | [`_entity`](other.md#development-and-test-connectors) | Test item type for includes and requirements | never | test |
| `dev/require/a.yaml` | [`a`](other.md#development-and-test-connectors) | Stores two parameters | expression | test |
| `dev/require/b.yaml` | [`b`](other.md#development-and-test-connectors) | Includes `a` and requires its value | expression | test |
| `dev/require/c.yaml` | [`c`](other.md#development-and-test-connectors) | Includes `a` and `b` | expression | test |
| `_future_entity.yaml` (top level) | — | Template for a default item type definition; not loaded | — | utility |

## Client `argeweb`

`argeweb` is a brand below `yourhosting`. It inherits all domain connectors and adds its own domain item type.

| Path | Connector | Purpose | Refresh | Status |
|---|---|---|---|---|
| `hosting/domain/_entity.yaml` | [`_entity`](hosting-domain.md#argeweb-domain-item-type) | Item type domain for `argeweb`: key name `domain`, attributes `tld` and `sld` | never | production |
| `hosting/domain/crm.yaml` | [`crm`](hosting-domain.md#crm) | Designed for customer and domain data from the Argeweb customer administration | default | legacy |

## Client `your.online`

`your.online` is the parent client of the brand `yourhosting`, so `yourhosting` and the brands below it inherit these
item types and connectors.

| Path | Connector | Purpose | Refresh | Status |
|---|---|---|---|---|
| `commercial/action/_entity.yaml` | [`_entity`](other.md#incentive-programme) | Item type action: one action of an employee for an incentive | never | production |
| `commercial/action/aggregate.yaml` | [`aggregate`](other.md#incentive-programme) | Designed for counting the unique actions | default | planned |
| `commercial/incentive/_entity.yaml` | [`_entity`](other.md#incentive-programme) | Item type incentive: employee, domain and topic, with date fields for reporting | never | production |
| `commercial/incentive/domain.yaml` | [`domain`](other.md#incentive-programme) | Topic: extra TLDs registered for the domain name | designed to check daily during the first three months after the incentive was created | production |
| `commercial/incentive/m365.yaml` | [`m365`](other.md#incentive-programme) | Topic: e-mail handled by Microsoft 365 | daily until valid, then `3 months` | production |
| `commercial/incentive/ssl.yaml` | [`ssl`](other.md#incentive-programme) | Topic: valid certificate and https | daily until valid, then `3 months` | production |
| `commercial/incentive/ssl_https.yaml` | [`ssl_https`](other.md#incentive-programme) | Topic: valid certificate and https (same check as `ssl`) | daily until valid, then `3 months` | production |
| `commercial/incentive/website.yaml` | [`website`](other.md#incentive-programme) | Topic: the website category is `active` | daily until valid, then `3 months` | production |
| `commercial/report/_entity.yaml` | [`_entity`](other.md#incentive-programme) | Item type report for the incentive programme | `night` (daily) | production |
| `commercial/topic/_entity.yaml` | [`_entity`](other.md#incentive-programme) | Item type topic of an incentive | never | production |
| `contact/email/_entity.yaml` | [`_entity`](other.md#incentive-programme) | Item type employee, identified by e-mail address | never | production |
| `crm/template/_entity.yaml` | [`_entity`](other.md#crm-templates) | Item type CRM template, with free JSON attributes | never | production |
| `hosting/subscription/_entity.yaml` | [`_entity`](other.md#subscriptions) | Item type subscription, identified by UUID; belongs to one domain | — | production |
| `hosting/subscription/attributes.yaml` | [`attributes`](other.md#subscriptions) | Attributes of the item type, in a separate file: dates, product and periods of the subscription | never (API) | production |
| `hosting/subscription/retrieve.yaml` | [`retrieve`](other.md#subscriptions) | Designed for registering a pushed subscription on the subscription itself | never | planned |
| `product/domain/_entity.yaml` | [`_entity`](other.md#products) | Item type domain product | never | production |
| `product/email/_entity.yaml` | [`_entity`](other.md#products) | Item type e-mail product | never | production |
| `product/hosting/_entity.yaml` | [`_entity`](other.md#products) | Item type hosting product | never | production |
| `product/subscription/_entity.yaml` | [`_entity`](other.md#products) | Item type subscription product | never | production |
| `sandwave/customer/_entity.yaml` | [`_entity`](other.md#sandwave-customers) | Item type customer in Sandwave, with its subscriptions and domains | — | production |
| `sandwave/customer/attributes.yaml` | [`attributes`](other.md#sandwave-customers) | Attributes of the item type, in a separate file: dates, product and periods of the customer | never (API) | production |

## Client `parker`

| Path | Connector | Purpose | Refresh | Status |
|---|---|---|---|---|
| `analytical/device/_entity.yaml` | [`_entity`](other.md#parker-legacy) | Item type device, with its measurements | never | legacy |
| `analytical/measurement/_entity.yaml` | [`_entity`](other.md#parker-legacy) | Item type measurement of a device | never | legacy |
