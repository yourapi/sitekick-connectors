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

#### status
The category for the website, can be:
* not_registered: the domain is ont registered with a registrar.
* no_whois: the domain is registered, but no whois data is available.
* no_dns: the domain is registered, but no DNS data is provisioned.
* no_a_record: the domain is registered, but no A-record is present.
* no_nameserver: the domain is registered, but no nameserver entry is present in the DNS.
* no_youronline_nameserver: the nameserver is not for any of the your.online brands
* parked: standard landing page for the website, no active website.
* initiated: the user has logged in and initiated building a website, but no further action was taken. User stopped after first step.
* installed: the user installed a CMS, but did not place any non-standard content. Example is the `hello world` website.
* outdated: active, but prehistoric in internet-time (like Microsoft Frontpage-edited)
* unreachable: no response from the server
* forsale: the user has placed a website for sale
* active: a standard website is present, the user has placed content.

* #### category
* parked: standard landing page for the website, no active website.
* initiated: the user has logged in and initiated building a website, but no further action was taken. User stopped after first step.
* installed: the user installed a CMS, but did not place any non-standard content. Example is the `hello world` website.
* outdated: active, but prehistoric in internet-time (like Microsoft Frontpage-edited)
* unreachable: no response from the server
* forsale: the user has placed a website for sale
* active: a standard website is present, the user has placed content.

#### certificate
Data for the certificate, 
{"issuer":"R11","ip":"172.104.255.188","valid_to":"2025-04-06T20:01:17Z","hostname":"sitekick.eu","common_name":"
sitekick.eu","valid_from":"2025-01-06T20:01:18Z"}

# Examples


