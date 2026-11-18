# sitekick-connectors

Connector definitions for the new sitekick platform. A connector defines the data source, processing and validation
rules for a specific type of data.

# Overview

The Sitekick-api is an easily accessible API on any combination of subject domains. The user can easily define object
types, their attributes and relations between objects.  
The main use of the Sitekick-platform is integration of (external) data in one place. Using a [connector](#connectors),
you can easily define the data to connect to.

# Entities

# Connectors

The connectors for website tests are defined in `sitekick/hosting/domain/`. Derived pass/fail checks, used for
commercial incentives, are in `your.online/commercial/incentive/`. The actual implementation of the checks
(`functions.hosting.*`) lives in the Sitekick platform, not in this repository.

## Overview of website tests

| Group                          | Connectors                                                               |
|--------------------------------|--------------------------------------------------------------------------|
| Reachability, redirects, HTTPS | `homepage`, `detailed_homepage`, `timing`, `links`, `screenshot`         |
| Certificates and SSL           | `certificate`, `certificate_www`, `ssl`                                  |
| Lighthouse / PageSpeed         | `pagespeed`                                                              |
| Content and technology         | `content`, `category`, `status`, `cms`, `copyright`, `google_analytics`, `seo` |
| DNS, domain and hosting        | `dns`, `whois`, `hosting_by_dns`, `internet_nl`, `rtr`, `availability`, `domdetailer`, `keywords`, `domainvalue`, `plesk` |
| Derived (incentives)           | `ssl`, `ssl_https`, `website`, `m365`, `domain` (in `your.online/commercial/incentive/`) |

The connectors `test`, `test3`, `test-req-1/2/3`, `sleep`, `multi` and `hubspot` are utility or test connectors and do
not test a website.

## List of connectors

### hosting|domain: reachability, redirects and HTTPS

#### homepage

Simple data on the homepage, basically homepage resolving based on a domain.

* url: first resolved url after domain entry, like _https://yourhosting.nl_
* final_url: final url after all redirects are followed, like _https://www.yourhosting.nl/_
* final_domain: the domain after possible redirects are followed, like _yourhosting.nl_
* https: does the website support https, 0 or 1
* http_redirected: filled from the number of redirects (see [known issues](#known-issues))
* redirects: first redirect status code, like _301_
* domain_redirected: is the final domain different from the original domain, 0 or 1

#### detailed_homepage

Info on redirect from http to https, redirect from no subdomain to www etc.

* final_url: final url after all redirects are followed
* https: does the website support https, 0 or 1
* http_redirected: is a request to http redirected to https, 0 or 1
* www_supported: does the `www.` variant of the domain respond, 0 or 1
* redirected_to_www: is the bare domain redirected to `www.`, 0 or 1

The complete result is stored as json.

#### timing

Time it takes to connect and get the first byte. The time is measured 10 times in a row, the first call is ignored to
allow for 'warming up' of the connection. Refreshed once a month.

* ttfb: total time to first byte, from request to first response, in seconds, like _0.0875_
* ttfb_ex_dns: time to first byte excluding dns lookup, in seconds, like _0.0868_
* connection_time: time to set up the connection, in seconds
* ttlb: total time until last byte is received, like _0.0876_. This time is NOT the time to first render.
* content_size: size of the homepage, in KB
* speed: download speed, in MB/s

#### links

Checks the links on the homepage.

* valid: number of valid links
* invalid: number of invalid (broken) links

The full list of checked links is stored in `full.json`.

#### screenshot

Full page screenshots of the homepage (via apiflash), without cookie banners, ads and tracking. Stored as files:

* desktop.webp: width 1920
* mobile.webp: width 384
* desktop-thumb.webp, mobile-thumb.webp: cropped thumbnails

### hosting|domain: certificates and SSL

#### certificate

Data for the certificate on the domain, if it is present. Contains these fields:

* issuer: the full name of the issuer, like `Sectigo RSA Extended Validation Secure Server CA`, `R10` or `R11`
* valid_from: the timestamp from which the certificate is valid. Client clock must not be before this timestamp, like
  `2025-04-22T00:00:00Z`
* valid_to: the timestamp until which the certificate is valid. Client clock must not be after this timestamp, like
  `2026-05-22T00:00:00Z`
* common_name: the name of the certificate, like `*.domain.name`
* invalid: the reason why the certificate is invalid, empty when valid

#### certificate_www

Identical to [certificate](#certificate), but for `www.` + the domain.

#### ssl

Combination of `status`, `homepage`, `detailed_homepage`, `certificate` and `certificate_www`.

* category: the status category of the website
* final_url: final url after all redirects are followed
* https: does the website support https, 0 or 1
* http_redirected: is a request to http redirected to https, 0 or 1
* www_consistent: does the website behave the same with and without `www.`, 0 or 1
* domain_redirected: is the final domain different from the original domain, 0 or 1
* issuer: the issuer of the certificate
* invalid: the reason why the certificate on the domain or on `www.` is invalid

### hosting|domain: Lighthouse / Google PageSpeed

#### pagespeed

Runs Google PageSpeed Insights (Lighthouse) on the final url of the homepage, for the categories Performance,
Accessibility, Best Practices and SEO, locale `nl-NL`. Only desktop is tested. The test is only executed when the
homepage resolves and the category of the website is `active`.

* performance: score 0-100
* accessibility: score 0-100
* best-practices: score 0-100
* seo: score 0-100

The individual Lighthouse audits are not indexed but stored as files:

* report.json: the complete raw Lighthouse report
* summary.json: per audit the `score`, `displayValue`, `numericValue` and `numericUnit`, per category the score and the
  weighted audit references, and the category groups

### hosting|domain: content and technology

#### content

Downloads the text and html of the homepage and stores it in `full.json`. Used by `category`, `copyright` and
`google_analytics`.

#### category

The category for the website, determined by keywords in the content, can be:

* parked: standard landing page for the website, no active website.
* initiated: the user has logged in and initiated building a website, but no further action was taken. User stopped
  after first step.
* installed: the user installed a CMS, but did not place any non-standard content. Example is the `hello world`
  website.
* outdated: active, but prehistoric in internet-time (like Microsoft Frontpage-edited)
* unreachable: no response from the server
* forsale: the user has placed a website for sale
* active: a standard website is present, the user has placed content.

A subcategory gives more detail, like `Hello world`, `Lorem ipsum`, `Moved out`, `DDoS protection`,
`pending validation` or `account suspended`.

#### status

The overall category for the website, combining `category`, `rtr`, `whois` and `dns`. Besides the values of
[category](#category), it can be:

* not_registered: the domain is not registered with a registrar.
* no_whois: the domain is registered, but no whois data is available.
* no_dns: the domain is registered, but no DNS data is provisioned.
* no_a_record: the domain is registered, but no A-record is present.
* no_nameserver: the domain is registered, but no nameserver entry is present in the DNS.
* no_youronline_nameserver: the nameserver is not for any of the your.online brands

Also contains the subcategory and the subscription category and slugs. Refreshed monthly, or every 6 months for
active websites.

#### cms

Info on the CMS, if it can be determined.

* description: _Zorg ervoor dat WordPress in elk geval veilig is door een upgrade naar 5.6.14 uit te voeren._
* ok: _false_
* score: 0-100
* title: _WordPress is onveilig_
* confidence: in the ascertainment of the cms, 0-100
* name: common name of the CMS, like _WordPress_
* status: status of the cms version, if known, like _insecure_
* version: version of the cms, if known, like _5.6.9_
* requiredVersion: (only for WordPress): upgrade to at least this version to ensure security, like _5.6.14_
* recommendedVersion: (only for WordPress): like _6.7.1_
* type: type of the CMS, if applicable, like _WordPress_

#### copyright

The most recent copyright year found in the content (like `© 2013-2025` → _2025_), an indication of how up to date
the website is. Only years from 1980 up to the current year are accepted.

* year: like _2025_

#### google_analytics

* has_analytics: is Google Analytics present, 0 or 1
* type: the type of analytics
* tracking_id: the tracking id

#### seo

Placeholder for [python-seo-analyzer](https://github.com/sethblack/python-seo-analyzer); not implemented yet.

### hosting|domain: DNS, domain and hosting

#### dns

DNS lookup for the domain. Refreshed every month, or every 3 months when no A-record is present.

* a1: the first A-record
* a_fqdn1: the first fqdn for the A-record
* ns1: the first nameserver
* mx1: the first MX-record

The full lookup is stored in `full.json`.

#### whois

Whois data for the domain, the content and details are dependent on the registry, from almost none for `.de` to quite
extensive for `.com`

* registrar
* name_server, name_servers
* creation_date, expiration_date

The full whois data is stored in `full.json`.

#### hosting_by_dns

Compares hosting data from whois and DNS.

* registrar
* ns_whois: nameservers according to whois
* status_whois: is the whois status `active`, 0 or 1
* ns_dns: nameservers according to DNS
* a_fqdn: fqdn's for the A-records

#### internet_nl

Runs the [internet.nl](https://internet.nl) test asynchronously (via a queue). Refreshed monthly.

* ipv6: score
* dnssec: score
* https: score
* total: total score

#### rtr

Real time registrar lookup, for availability of the domain.

* available: is the domain open for registering, 0 or 1
* premium: 0 or 1
* message: if no availability is known, contains the message, like _The TLD "bbb" is not supported_
* type: the type of message, like _UnsupportedTld_

#### availability

Availability of the domain name with other TLDs, on demand. The TLDs can be specified in the query parameter `tlds`,
separated by spaces; default `eu com be de me net nu org online`.

#### domdetailer

Backlink metrics from domdetailer: `majesticCF`, `majesticTF`, `majesticLinks`, `majesticRefDomains`,
`majesticRefSubnets`, `majesticIPs`, `mozLinks`, `mozRank`. The full result is stored in `full.json`.

#### keywords

Keyword data (search volume, cpc, trend) from Keywords Everywhere for keywords derived from the domain name, or for
the keywords in the parameters `keyword`/`keywords`. Stored in `full.json`.

#### domainvalue

The estimated financial value of the domain, based on popularity and backlinks, with a proprietary algorithm. Uses
`domdetailer`, `keywords` and `whois`. Returns:

* value: the estimated semi-precise value in euros, like _4540842_
* value_rounded: the estimated value, in two significant digits, like _4500000_
* searchvol: the search volume for the domain name
* cpc: the cost per click
* mozRank: the mozRank is an indication of the quality and quantity of the backlinks, a logarithmic scale from 0 to 10
  (stored multiplied by 10)

#### plesk

Data parsed from Plesk output: PHP version, server, number of mailboxes, provider and Plesk version, and for
WordPress the version, state, latest version, plugins and themes. The complete parsed data is stored in `plesk.json`.

### your.online|commercial|incentive

Derived checks with a pass/fail field `valid`, used to set incentive points. Refreshed daily while not valid, every 3
months once valid.

* ssl / ssl_https: the certificate is valid and the website uses https. Also gives `issuer`, `valid_from`, `valid_to`,
  `invalid`, `https`, `domain_redirected` and `paid_provider` (1 when the issuer is not Let's Encrypt).
* website: the category of the website is `active`.
* m365: the MX-records point to `*.mail.protection.outlook.com`.
* domain: which of the TLDs `nl be eu com online de net org email tel` are registered for the domain name.

## Known issues

* `homepage`: `http_redirected` is filled from `data.redirect_number` (the number of redirects) instead of a 0/1 flag
  whether http is redirected to https. Use `detailed_homepage.http_redirected` for the redirect test.
* `ssl`: refers to `detailed_homepage.attributes.https_supported` and `endpoints_are_identical`, which are not indexed
  by `detailed_homepage` (the field is called `https`), and to `include.detailed_homepage.redirected_to_www` without
  `.attributes`. As a result `https` and `www_consistent` are probably empty or wrong.
* `pagespeed`: only desktop is tested; the mobile strategy is commented out.

# Examples

For getting info, you first need a key, to be requested at Sitekick.
The base url is: `https://eu.sitekick.online`
The next parts are:

* client name (sitekick for default access, your shorthand name for your organisation). Clients can have a hierarchical
  relationship where an upper client is allowed more access than lower clients, like `yourhosting`, `argeweb`.
* subject area: general subject, like `hosting`, ...
* subject: specific subject, like `domain`, `server`
* key: the unique key for the item, can be a unique index like `3141592`, uuid or 'natural' key, like `yourhosting.nl`.

General data:
* [https://.../domain/yourhosting.nl](https://eu.sitekick.online/yourhosting/hosting/domain/yourhosting.nl)
Which is equivalent to using the database-id for this domain:
* [https://.../domain/119396](https://eu.sitekick.online/yourhosting/hosting/domain/119396)
Every item has a uuid to enable publication in a semi-secure way:
* [https://.../domain/b56f5dc3-912e-b14f-3612-2e260cd6a9ce](https://eu.sitekick.online/yourhosting/hosting/domain/b56f5dc3-912e-b14f-3612-2e260cd6a9ce)

Some specific data points:
* [https://.../domain/yourhosting.nl/timing](https://eu.sitekick.online/yourhosting/hosting/domain/yourhosting.nl/timing)
* [https://.../domain/yourhosting.nl/dns](https://eu.sitekick.online/yourhosting/hosting/domain/yourhosting.nl/dns)
* [https://.../domain/yourhosting.nl/domainvalue](https://eu.sitekick.online/yourhosting/hosting/domain/yourhosting.nl/domainvalue)
  When a data point has not been refreshed, it will be refreshed on first access.

Force refresh of the data:
* [https://.../domain/yourhosting.nl/timing?refresh=now](https://eu.sitekick.online/yourhosting/hosting/domain/yourhosting.nl/timing?refresh=now)
  As you can see, the data for this data point are refreshed on every request.
