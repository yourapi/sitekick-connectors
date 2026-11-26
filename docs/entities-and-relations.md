---
title: Entities and relations
---

# Entities and relations

Connectors compute data points for items. This page describes how item types are defined, how items are identified,
and how items are related to each other.

## Item types and `_entity.yaml`

Every item type folder contains an `_entity.yaml`. The loader creates the item type from it before it loads the
connectors of the folder, because a connector can only be stored for an existing item type. The file has three
sections, all optional:

```yaml
# sitekick/hosting/server/_entity.yaml (shortened)
key-names:            # synonyms for the item key, passed to connectors as parameters
    - ip
    - ip-address

attributes:           # the item's own fields, defined like a connector named `attributes`
    type: JsonStrInt
    index:
        name: -> str
        brand: -> str
        cpus: -> int
    refresh: never

relationships:        # relation types to other item types, see Relations below
    sitekick/hosting/platform:
        cardinality: ...  # see Cardinality below
        name:
            from: hosted-on
            to: hosts
```

| Section | Meaning |
|---|---|
| `key-names` | Names under which the item key is available in `params`. With `domain`, a domain connector writes `params.domain`; with `ip`, a server connector writes `params.ip`. A name with a hyphen is read as `params['ip-address']`. |
| `attributes` | The fields of the item itself, such as the brand of a domain or the dates of a subscription. They are stored as a data point named `attributes` and appear in the item's `attributes`. |
| `relationships` | The relation types from this item type to other item types, see [Relations](#relations). |

The attributes of an item are usually supplied by the system that creates the item, not computed. An index field that
has only a cast, such as `year: -> int`, takes its value from the supplied attribute of the same name (see
[Storage of results](storage.md#fields-without-an-expression)). The attributes data point is never refreshed by the
schedule unless its definition sets a refresh rule.

## Keys

An item is identified within its client and item type. It can be addressed in three ways:

| Identifier | Example | Notes |
|---|---|---|
| Natural key | `example.com` | The key that is natural to the type: a domain name, an IP address, a UUID from another system. Unique per client and item type. |
| Id | `12345` | The numeric id assigned by the platform. |
| UUID | `<uuid>` | Derived from the id by encryption, so it can be shared without revealing the id. |

A value of digits only is read as an id, and a value in UUID form as the platform UUID. Subscriptions have a natural key
in UUID form, taken from the subscription system; address one with a leading underscore in the URL, as
`.../hosting/subscription/_<uuid>`, so the key is not read as a platform UUID.

The same natural key can exist once for each client: `example.com` for `yourhosting` and `example.com` for `argeweb`
are two items, each with its own data points.

### Key normalisation

Before an item is created, the platform normalises its key, and an item type can check whether the key is valid. For
domains:

- a URL is reduced to its host name: `https://Example.com:443/shop?x=1` becomes `example.com`;
- any other key is converted to lower case and stripped of spaces;
- the key must be a valid domain name, below a public suffix: `example.com` is valid, `com` is not.

Other item types keep the key as it is supplied.

### The item in expressions

In a connector the item is available as `self` (also `item`). Its parts follow the json:api structure of the item:

| Expression | Content |
|---|---|
| `self.data.key` | The natural key |
| `self.data.id` | The numeric id |
| `self.data.type` | The item type path, such as `yourhosting/hosting/domain` |
| `self.attributes.<name>` | An attribute of the item, such as `self.attributes.domain` of an incentive |
| `self.meta.created_at` | When the item was created |
| `self.links.self` | The URL of the item in the API |
| `self.relationships.<name>` | The related items under a relation name, see below |
| `params.<key name>` | The key, under each name in `key-names` |

The `status` connector, for example, reads the subscription category from the domain's attributes with
`self.attributes.subscription.category`. The incentive connectors read the domain they check from
`self.attributes.domain`.

## Relations

Items can be related: a domain is hosted on a server, a subscription belongs to a domain, a server belongs to a
platform. A **relation type** links two item types and is declared in the `relationships` section of the
`_entity.yaml` of one of them, keyed by the path of the other item type. A relation declared once applies to both
sides; the other item type does not repeat it.

```yaml
# your.online/hosting/subscription/_entity.yaml
key-names:
    - uuid

relationships:
    sitekick/hosting/domain:
        cardinality:
            from: "1"        # one subscription is connected to one domain
            to: ">= 0"       # a domain has zero or more subscriptions
        name:
            from: domain
            to: subscriptions
```

| Key | Meaning |
|---|---|
| `cardinality.from` | How many related items an item of the declaring type has under the relation, here: one domain per subscription |
| `cardinality.to` | The count for the other side |
| `name.from` | The name under which an item of the declaring type lists its related items, here `domain` |
| `name.to` | The name under which the related item lists items of the declaring type, here `subscriptions` |

Without `name`, the names are derived from the item type names, in plural where more than one related item is
possible. Without `cardinality`, a side defaults to `(0, 1)`.

### Cardinality

A cardinality is written as the possible number of related items:

| Notation | Meaning |
|---|---|
| `"1"` or `"(1)"` | Exactly one |
| `"0, 1"` or `"(0, 1)"` | Optional: zero or one |
| `">= 0"` | Zero or more |
| `">= 1"`, `"> 0"` | One or more |
| `"<= 1"`, `"< 2"` | At most one |
| `"(1, 3)"`, `">= 1 & <= 3"` | A range |

From the two sides the platform derives whether each side is singular (at most one) or plural, and whether it is
optional (zero allowed). Together they express the usual relation kinds:

| Kind | Singular sides | Example in this repository |
|---|---|---|
| One-to-one (1:1) | Both | none at present |
| One-to-many (1:n) | One | A subscription has one domain; a domain has any number of subscriptions |
| Many-to-many (n:m) | Neither | Possible with `">= 0"` on both sides |

Each relation between two items is stored once per relation type. Relation names are unique per pair of item types.

### Creating relations

Relations are created when an item is posted with a `relationships` member, in json:api form. Each relation names the
item type path and the numeric id of the related item:

```json
{
  "item": {
    "data": {"key": "<subscription uuid>"},
    "attributes": {"product_slug": "hosting-basic", "contract_period": 12},
    "relationships": {
      "domain": {"data": {"type": "sitekick/hosting/domain", "id": "12345"}}
    }
  }
}
```

The platform looks up the relation type by the relation name and the two item types, in either direction. It checks
that a single related item is supplied for a singular side and a list for a plural side, and answers `422` otherwise.

Connectors create relations in the same way, by posting items through the platform's API from a function:

- The domain connector `subscription` (`sitekick/hosting/domain/subscription.yaml`) calls
  `functions.hosting.update_subscription`. That function reads subscription data pushed to the platform for the
  domain, and posts it as a `subscription` item with a `domain` relation to the current domain. The domain then lists
  the subscription under `subscriptions`.
- Incentives (`your.online/commercial/incentive`) are created with relations to their domain, topic and employee
  e-mail address. Their connectors check the domain and write the result back to the incentive's attributes.

### Reading relations

A single item lists its related items under `relationships`, keyed by relation name: one related item for a singular
side, a list for a plural side. Each related item has its `data`, `links` and `meta`.

```json
{
  "data": {"id": "67890", "type": "yourhosting/hosting/subscription", "key": "<subscription uuid>"},
  "relationships": {
    "domain": {"data": {"id": "12345", "type": "yourhosting/hosting/domain", "key": "example.com"}}
  }
}
```

In a connector the same structure is available as `self.relationships.<name>`, for example
`self.relationships.domain.data.key`. The design provides for querying and filtering on relations in lists of items.

## Item groups

Items can also be collected in an **item group**: a named set of items per client, either a fixed list or a query. A
group can hold its own properties and a list of connectors. Groups are managed through the API and are not defined in
this repository.
