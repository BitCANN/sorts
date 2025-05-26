# Simple Op Return Text Standard (SORTS)

    Title: Simple Op Return Text Standard
    Type: Standards
    Layer: Applications
    Maintainer: Kuldeep Singh
    Status: Draft
    Initial Publication Date: 2025-05-25
    Latest Revision Date: 2025-05-26
    Version: 1.0.1

## Overview

This document defines the Simple Op Return Text Standard (SORTS) for use in on-chain identity and metadata systems. It is designed for append-only, immutable environments like BitCANN or similar smart contract systems, where data is in fixed-size or limited-size (e.g max 223 bytes in BCH, 2025). The SORTS enables flexible key-value data, namespacing, robust revocation, deduplication, and support for large values through fragmentation.
The goal of this standard is to provide a human-readable, stateless, and portable format that supports extensibility and simple parsing.


## 1. Key-Value Record Format

- Format: `key=value`
- Keys should use dot-separated namespaces (e.g. `social.github`, `bio.name`, `dns.txt`, `contract.fingerprint`)
- Parsers should treat unrecognized keys as opaque unless they follow the standard format

## 2. Reserved namespaces/keys

The standard reserves the following keys for internal use:

| Key         | Value                | Description |
|-------------|-------------------------------|-------------|
| revoked    | `<sha256-hash>`   | Used to signal revocation of a record. The value is always the SHA-256 hash of the full `key=value` string of the record being revoked. |
| meta       | `type:<array\|text\|object\|...>`            | Represents the behaviour of the record. Example: `meta=type:array`. For simple records, it can be omitted. |

## 3. Updating Records and Fragmentation

- A new value for the same key replaces the old one.
- An existing record should be revoked by hash if its invalidation is needed.
- A record with `.meta` tag attached to it, the parser should assume that the single key will have a complex nature, i.e it can be an array or a text or other complex types.
  ```
  <namespace>.meta=type:<array|text|...>
  ```
- Attachment of `.<n>` to a key indicates that the key has an ordered value
  ```
  <namespace>.meta.<n>=<value>
  ```

- Examples:
  ```
  // Unordered array
  bio.email=a@example.com
  bio.email=b@example.com
  bio.email.meta=type:array
  ```

  ```
  // Ordered array
  bio.email.0=a@example.com
  bio.email.1=b@example.com
  bio.email.meta=type:array
  ```

  ```
  // Ordered text
  profile.bio.0=This is the beginning ...
  profile.bio.1=... continued text ...
  profile.bio.meta=type:text
  ```

## 4. Record Invalidation / Revocation

- Since on-chain data is immutable, revocation is achieved through a dedicated `revoked=<hash>` record signaling indexers to exclude the record.
- The `<hash>` is always the SHA-256 of the full `key=value` string
  ```
  # example
  Record: social.github=example
  Revoked: revoked=8a2e7f4c...
  ```
- Indexers must exclude any key-value whose hash appears under the `revoked` namespace

- Revocation of records with `.meta` tag will result in removal of all the records connected to it. To remove the individual parts of the record with meta tag, only the part to be removed can be revoked

  ```
  # example
  bio.email=a@example.com
  bio.email=b@example.com
  bio.email.meta=type:array

  # Revocation of bio.email array
  revoked=8a2e7f4c...  # sha256(bio.email.meta=type:array)

  # Revocation of bio.email.1
  revoked=8a2e7f4c...  # sha256(bio.email.1)
  ```

## 5. Recommended Namespaces and Types


| Namespace  | Key/Sub-Namespace | Type              | Description                                                                                                 |
|------------|-------------------|-------------------|-------------------------------------------------------------------------------------------------------------|
| bio        | `name`              | string            | Display name                                                                       |
|            | `avatar`            | string            | Avatar image URL                                                                       |
|            | `banner`             | string            | Banner image URL                                                                |
|            | `description`       | string/fragmented | A description   
|            | `email`             | string            | E-mail address                                                                |
|            | `mail`              | string            | Physical mailing address                                                                                    |
|            | `phone`             | string            | Phone number                                                                             |                                                   |
|            | `keywords`          | string            | Comma-separated keywords, ordered by most significant first           |
|            | `notice`            | string/fragmented | A notice regarding this name                                                   |
|            | `location`          | string            | Generic location (e.g. "Ladakh, India")                                                                   |
|            | `url`               | string            | Website URL                                                                                                 |
|            | *                   | string/fragmented | **Many additional keys are recommended as per [schema.org Person](https://schema.org/Person) (e.g., `birthDate`, `jobTitle`, `alumniOf`, etc.); see schema.org for full list.** |
| privacy    | `pgp`               | fragmented | PGP public key (ASCII-armored)                                                                              |
|            | `rpa`               | string/fragmented | RPA (Resuable Personal Address)                                                                              |
|            | *       | string/fragmented            | Any other privacy scheme                                                                              |
| `crypto`   | `BCH.address`                        | string            | Bitcoincash address
|  | `ETH.address`                        | string            | Ethereum address                                                                        |
|            | `BTC.address`                        | string            | Bitcoin address                        |
|            | `<TICKER>.address`                   | string            | Cryptocurrency address for specified ticker                                                                 |
|            | `<TICKER>.version.<VERSION>.address` | string            | Multi-chain currency address (e.g., USDT on ERC20, TRON, EOS, OMNI)                                         |
|            | *                                    | string            | **Cryptocurrency address format inspired by [Unstoppable Domains Records Reference](https://docs.unstoppabledomains.com/resolution/records-reference/)** |
| `token`    | `<FAMILY>.address`                   | string            | Blockchain family address (e.g., EVM)                                                                        |
|            | `<FAMILY>.<NETWORK>.address`         | string            | Blockchain network address (e.g., UTXO.BCH)                                                                   |
|            | `<FAMILY>.<NETWORK>.<TOKEN>.address` | string            | Token-specific address (e.g., UTXO.BCH.USDT)                                                                |
|            | *                                    | string/fragmented            | **Token format inspired by [Unstoppable Domains Records Reference](https://docs.unstoppabledomains.com/resolution/records-reference/)** |
| social     | `twitter`           | string            | Twitter handle; case-insensitive                                                                            |
|            | `github`            | string            | GitHub username; letters, numbers, hyphens only                                                             |
|            | `telegram`          | string            | Telegram username; letters, numbers, underscores                                                            |
|            | `discord`           | string            | Discord username; must include #                                                                            |
|            | *             | string/fragmented            | Other social platform handles                                                                               |
| dns        | `a`                 | string/fragmented | IPv4 address in dotted decimal format (e.g., 192.0.2.1)                                                     |
|            | `aaaa`              | string/fragmented | IPv6 address; lowercase recommended                                                                         |
|            | `cname`             | string/fragmented | Canonical name record; may end with a dot                                                                   |
|            | `mx`                | string/fragmented | Mail exchange record; priority and host                                                    |
|            | `txt`               | string/fragmented | TXT record                                                            |
|            | `ns`                | string/fragmented | Nameserver record optional                                                                     |
|            | *                 | string/fragmented | **Any DNS record type as defined by [IANA DNS Parameters](https://www.iana.org/assignments/dns-parameters/dns-parameters.xhtml#dns-parameters-4) is supported. Use the corresponding type name as the key (e.g., `dns.srv`, `dns.ptr`, `dns.soa`, etc.).** |
| msg        | `<string>`          | string/fragmented | generic messages                                                                |
| contract   | `fingerprint`          | string | **Contract fingerprint. See [smart-contract-fingerprinting](https://gitlab.com/0353F40E/smart-contract-fingerprinting) for details.** |
|            | `params`           | string/fragmented | Contract parameters                                                              |

## Examples

```
# Bio
bio.name=Vikram Sarabhai
bio.avatar=https://example.com/vikram.jpg
bio.email=vikram@example.com
bio.description.0=Founder of the Indian Space Research Organisation (ISRO).
bio.description.1=Spearheaded India's entry into the space age with the first satellite launch.
bio.description.meta=type:text

# Privacy
privacy.pgp=-----BEGIN PGP PUBLIC KEY BLOCK-----
privacy.rpa=example.rpa

# Revocation of bio.description
revoked=8a2e7f4c...  # sha256(bio.description.meta=type:text)

# Social
social.twitter=handle
social.telegram=handle

# Crypto addresses
crypto.BCH.address=bitcoincash:qpm2qsznhks23z7629mms6s4cwef74abcdabdaabdc
crypto.USDT.version.ERC20.address=0xabcdef1234567890abcdef1234567890abcdef12
token.EVM.ETH.address=0xabcdef1234567890abcdef1234567890abcdef12

# Basic DNS records
dns.a=192.168.1.1
dns.aaaa=2606:2800:220:1:248:1893:25c8:194

# Contract
contract.fingerprint=0x1234567890abcdef1234567890abcdef12345678
contract.params=1465077837 4 -789332029 e4
contract.meta=type:object
```

## References

1. [Unstoppable Domains Records Reference](https://docs.unstoppabledomains.com/resolution/records-reference/) - Documentation for Unstoppable Domains' record types and formats
2. [ENS Text Records](https://docs.ens.domains/web/records/) - Ethereum Name Service documentation for text records and metadata

## Changelog

- #### 1.0.1 - 2025-05-26
  - Added `privacy` namespace
  - Added `object` in meta type
  - Added `string/fragmented` type to wildcard keys (`*`)
- #### v1.0.0-draft – 2025-05-25 ([69066ab](https://github.com/BitCANN/sorts/commit/69066ab9d187efde26853b5a47dc6ebcd438ce91))
  - Initial publication


## Copyright

This document is placed in the public domain.
