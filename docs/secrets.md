---
title: Secrets in connectors
---

# Secrets in connectors

Many connectors call external services that need an API key or another credential. This repository is public, so a
secret is never written in a connector as plain text. It is written in encrypted form, and the platform decrypts it
when it loads the connector.

## Encrypted values

An encrypted value is placed between the markers `<<<--` and `-->>>`:

```yaml
headers:
    Authorization: "Bearer <<<--<encrypted>-->>>"
    Accept: application/json
```

- The encrypted value can stand alone, as the whole value of a key, or inside other text, as in the
  `Authorization` header above. Quote the string when other text surrounds it.
- A string can contain more than one encrypted value.
- Whitespace and line breaks inside the encrypted value are ignored, so a long value can be split over lines.
- Encrypted values can be used anywhere in a definition, for example in a header or in an intermediate key that is
  used to build a URL.

## How the platform handles them

1. The definition is stored in the platform as you wrote it, with the encrypted values.
2. When a definition is loaded into the in-memory cache of a platform process, every encrypted value is decrypted with
   the platform's key. The decrypted value exists only in memory.
3. When the connector is evaluated, expressions see the decrypted value, for example in `headers.Authorization`.

The decryption key belongs to the platform installation and is never part of this repository. Without it, the values
in this repository cannot be decrypted.

## Producing an encrypted value

Encrypted values are produced with the platform's encryption tool, with the key of the platform that will load the
connector. Ask the operator of your Sitekick installation to encrypt the value, or, if you operate the platform
yourself, use its encryption helper as described in the Sitekick platform documentation. The result is the complete
value, markers included; paste it into the definition unchanged.

## Rules

- **Never commit a secret in plain text**, not even temporarily: the history of a public repository keeps it.
- **Never copy an encrypted value** from one connector or installation to documentation, issues or examples. Write
  `<<<--<encrypted>-->>>` instead.
- **Keep secrets in intermediate keys.** Put an API key in a header or a URL that is built in an intermediate key, as
  `pagespeed` does with `key` and `url`. Intermediate keys are not stored, while `index`, `json` and files are stored
  and returned to API clients.
- **Do not store responses that echo the credential.** If a service returns the key in its response, remove it before
  the response goes into `json` or a file.
- **Put secrets in headers or intermediate keys**, not in `params`: `params` are the defaults of the request
  parameters.
- **Replace a leaked secret** at the service that issued it; encrypting it again does not make a leaked value safe.
