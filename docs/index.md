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

## List of connectors

### hosting|domain

#### category

The category for the website, can be:

* parked
* initiated
* installed
* outdated
* unreachable
* forsale
* active

#### certificate

Data for the certificate, if it is present. Contains these fields:

* issuer: the full name of the issuer, like `Sectigo RSA Extended Validation Secure Server CA`, `R10` or `R11`
* valid_from: the timestamp from which the certificate is valid. Client clock must not be before this timestamp, like
  `2025-04-22T00:00:00Z`
* valid_to: the timestamp until which the certificate is valid. Client clock must not be after this timestamp, like
  `2026-05-22T00:00:00Z`
* hostname: the published name for the website, like `www.domain.name`
* common_name: the name of the certificate, like `*.domain.name`
* ip: the ip address by which the host is reachable, like `31.7.0.224`

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

#### detailed_homepage

Info on redirect from http to https, redirect from no subdomain to www etc.

#### dns

#### domdetailer

#### homepage

Simple data on the homepage, basically homepage resolving based on a domain.

* https: does the website support https, 0 or 1
* final_domain: the domain after possible redirects are followed, like _yourhosting.nl_
* domain_redirected: is the final domain different from the original domain, 0 or 1
* url: first resolved url after domain entry, like _https://yourhosting.nl_
* redirects: first redirect status code, like _301_
* final_url: final url after alal redirects are followed, like _https://www.yourhosting.nl/_
* http_redirected: is a request to http redirected to https, 0 or 1

#### rtr

Real time registrar lookup, for availability of the domain.

* available: is the domain open for registering, 0 or 1
* premium: 0 or 1
* message: if no availability is known, contains the message, like _The TLD "bbb" is not supported_
* type: the type of message, like _UnsupportedTld_

#### screenshot

#### timing

Time it takes to connect and get the first byte.

* ttfb_ex_dns: excluding dns lookup, in seconds, like _0.0867899_
* ttfb: total time to first byte, from request to first response, like _0.08757759999999999_
* ttlb: total time until last byte is received, like _0.08763800000000001_. This time is NOT the time to first render.
  The time is measured 10 times in a row, the first call is ignored to allow for 'warming up' of the connection.

#### whois

Whois data for the domain, the content and details are dependent on the registry, from almost none for `.de` to quite
extensive for `.com`

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
https://eu.sitekick.online/yourhosting/hosting/domain/yourhosting.nl
https://eu.sitekick.online/yourhosting/hosting/domain/119396

One data area:
https://eu.sitekick.online/yourhosting/hosting/domain/yourhosting.nl/timing

Force refresh of the data:
https://eu.sitekick.online/yourhosting/hosting/domain/yourhosting.nl/dns?refresh=now
