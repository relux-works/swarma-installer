# Release-set manifest v0

Status: design proposal, round 4. Written for the repository `relux-works/swarma-installer`. Nothing here is implemented. Identifiers: `D-..` rows of `01-book-mapping.md`, `OD-..` rows of `05-open-decisions.md`, `V-..` and `U-..` rows of `03-verifier-contract.md`, `TP-..` rows of the test-profile table in `04-install-plan-colony1.md`.

A release-set manifest is ONE closed JSON object for one release set. It names every product of the set and every artifact of every product, each with its digest. There are no per-product manifests, no dependencies between manifests and no signatures in the format (rulings OD-6 and OD-7, and the later ruling that no signing key is a trust root in v0). The round 1 shape with `dependencies` and `signatures` was never released and is refused (`invalid-23`).

The identity of a manifest is the SHA-256 of its complete file bytes, called the **set digest**. The installer trusts a manifest only when its set digest equals the digest that the owner approved in person (V-03). Nothing inside the manifest gives authority.

The machine-readable shape is `release-manifest-v0.schema.json`. The table below is the contract; the schema is a partial mechanical copy of it.

## The contract

| Member | Type | Bounds | Required | Who writes it | Who checks it |
| --- | --- | --- | --- | --- | --- |
| (the file) | bytes | At most 65536 bytes. The bytes are exactly the CCJ-1 canonical encoding of the object: member names sorted by Unicode scalar value, no whitespace outside strings, no byte-order mark, no trailing byte, array order preserved, integers in shortest decimal form. v0 narrows CCJ-1 further: every byte is ASCII `0x20` to `0x7e`, and no string contains a quote, a backslash or an escape. Container depth is at most 5. | yes | The release operator who assembles the set, after the two reads of each release have compared the built artifact digests with a second, independent build. That comparison produces the second-build comparison record that check V-20 consumes on the test profile (TP-6): the two reads produce it (each reports the digest it built, per artifact; the release operator assembles one record per set), its members are fixed in file 3, its producing reads are audit-only members, and its byte encoding is MISSING (owner: the installer specification; on colony1's first-step path: yes). MISSING: no tool writes this file today (owner: the release standard; on colony1's first-step path: yes). | V-02 (size), V-03 (set digest), V-04 (scan), V-05 (canonical bytes) |
| (any object) | object | Closed: a member name not in this table refuses. No member name appears twice in one object, at any depth. Every member of this table is required; there are no optional members and no defaults. | yes | same | V-04 (duplicate), V-07 (unknown), V-08 (missing) |
| `kind` | string | Exactly `swarma release-manifest/0`. The repository that owns the format is decided (`relux-works/swarma-installer`); the prefix stays `swarma` until the installer specification in that repository names the component part of the wire name. Changing it later is a new format version, not an edit of this one. | yes | same | V-06 |
| `set` | string | `[a-z0-9][a-z0-9-]{0,62}`. The name of the release set. It names no repository, no signer and no machine. | yes | same | V-10 |
| `version` | string | `(0\|[1-9][0-9]*)\.(0\|[1-9][0-9]*)\.(0\|[1-9][0-9]*)(-rc\.[1-9][0-9]*)?`, at most 64 bytes: the version of the set. No leading zero, no `rc.0`, no range, no wildcard, no `latest`. | yes | same | V-10 |
| `products` | array of product objects | 1 to 32 items. No two items have the same `product`. Exactly one item has `purpose` equal to `first-install`. | yes | same | V-10 (count), V-11 (`product_duplicate`), V-12 (`first_install_product_count`) |
| `products[].product` | string | Same pattern as `set`. Names the product, not a repository and not a signer. | yes | same | V-10, V-11 |
| `products[].version` | string | Same pattern as `version`. It equals the product's release tag without the leading `v`; the pattern is the release standard's tag admission pattern, so every value that passes can be an admissible tag. Whether it equals an actual tag is semantic and external: no verifier check and no schema clause can see it. | yes | same | V-10 |
| `products[].source_commit` | string | Exactly 40 lowercase hexadecimal digits: the commit the product's release tag points to. | yes | same | V-10 (shape only; the verifier has no repository and does not check that the commit exists) |
| `products[].purpose` | string | One of `first-install`, `owner-account`, `platform-root`. It decides who unpacks the product's artifact and where. `first-install`: the package that carries the installer; installed by the operating system's package installer from root's protected copy (plan S-01). `owner-account`: unpacked under the owner's unprivileged account into that account's files, where root never writes (plan S-08b); where the plan's placement table gives root a file of such a product, root takes it from its own protected copy (plan S-09). `platform-root`: unpacked by root into the platform root (plan S-09). | yes | same | V-10 (an unknown value is `manifest_value_invalid`), V-12 |
| `products[].artifacts` | array of artifact objects | 1 to 4 items. No two items of one product have the same `platform`. | yes | same | V-10 (count), V-11 (`artifact_platform_duplicate`) |
| `artifacts[].name` | string | `[A-Za-z0-9][A-Za-z0-9._+-]{0,127}`: the file name under which the release publishes the artifact. It starts with a letter or digit, so `.` and `..` are not names. Unique across the WHOLE set. The installer never uses this name as a path on disk: copies are stored under the artifact's digest (V-16). | yes | same | V-10, V-11 (`artifact_duplicate`) |
| `artifacts[].platform` | string | One of `linux/amd64`, `linux/arm64`, `darwin/amd64`, `darwin/arm64`. Any other string refuses the whole manifest. A value that is not a string is refused by the type rule. | yes | same | V-09 (`platform_unknown`), V-10 (type), V-13 and V-14 (host) |
| `artifacts[].format` | string | One of `deb`, `tar.gz`. A product with purpose `first-install` carries only `deb` artifacts and only for `linux/amd64` or `linux/arm64`. Every other product carries only `tar.gz` artifacts. | yes | same | V-10 (unknown value), V-12 (`artifact_format_not_allowed`) |
| `artifacts[].size` | integer | 1 to 9007199254740991: the exact length of the artifact in bytes. The operational caps are in the limits table of `03-verifier-contract.md`. | yes | same | V-10 (shape), V-15 (caps), V-17 (the copy has this length) |
| `artifacts[].sha256` | string | Exactly 64 lowercase hexadecimal digits: SHA-256 of the artifact bytes. Unique across the WHOLE set: one digest is one artifact of one product for one platform. | yes | same | V-10 (shape), V-11 (`artifact_digest_duplicate`), V-18 (the copy has this digest) |
| `artifacts[].unpacked_size` | integer | 1 to 9007199254740991: the total number of content bytes of all regular files the artifact unpacks to. It is the upper bound of what unpacking may write. | yes | same | V-10 (shape), V-15 (caps), U-11 (the unpack stops at this bound) |

## Identity across the set

| Rule | Why | Refusal |
| --- | --- | --- |
| One product appears once. | Two entries for one product leave nothing to say which version is installed. | `product_duplicate` |
| One artifact name appears once in the whole set. | A name is how a person finds the published file; two files under one name cannot be told apart by it. | `artifact_duplicate` |
| One digest appears once in the whole set. | One file of bytes belongs to exactly one product and platform. | `artifact_digest_duplicate` |
| One product has at most one artifact per platform. | Selection for a host can then never match more than one. | `artifact_platform_duplicate` |
| Exactly one product has purpose `first-install`. | Exactly one package carries the installer. | `first_install_product_count` |

## Artifact selection for a host

| Step | Rule | Refusal |
| --- | --- | --- |
| 1 | The host platform is `linux/amd64` or `linux/arm64`, as the operating system reports it. | `host_platform_unsupported` (V-13) |
| 2 | For EVERY product of the set, the selected artifact is the one whose `platform` equals the host platform. A set installs whole: no product is skipped. | n/a |
| 3 | Zero matches for any product refuses the set, with that product as the subject. | `artifact_missing_for_host` (V-14) |
| 4 | More than one match cannot happen in a manifest that passed V-11; the manifest itself was refused. | `artifact_platform_duplicate` (V-11) |
| 5 | Artifacts of other platforms are never opened, copied or hashed. | n/a |

## The set digest

```text
set digest = SHA-256(complete file bytes of the manifest)
text form  = 64 lowercase hexadecimal digits
```

The file has no signature member, so the file bytes never change after the set is assembled. The set digest is computed by whoever needs it from the bytes they hold; no member of the manifest states it.

## Rules the schema cannot express

JSON Schema validates a parsed value. These rules are checked on bytes or across items, by the verifier only:

- duplicate member names (V-04): a lenient parser silently keeps one of them;
- canonical bytes (V-05), including a number written as `5242880.0` or `1e3`, which the schema type `integer` accepts;
- uniqueness of `product`, of `name` and `sha256` across the whole set, and of `platform` inside one product (V-11);
- the file size cap (V-02) and the set digest (V-03);
- that a product `version` equals an actual release tag without its leading letter (semantic and external);
- any numeric comparison against the true safe-integer ceiling in a validator that reads numbers as binary floating point.

The schema does express: closed objects and required members, every pattern and bound, exactly one `first-install` product, and the format allowed for each purpose.

## What the format deliberately does not carry

- No signature, no key identifier, no signer list. In v0 no signing key is a trust root (TP-1). A signature on a release, where a repository publishes one, has no weight for the installer.
- No part signature per artifact (TP-2). The set manifest lists every artifact digest instead.
- No download address. Bytes may come from any mirror; an address proves nothing.
- No expiry, no version counter and no revocation list. v0 has no protection against the owner approving an old set; the approval is a human act and names one exact digest.
- No list of what is inside an archive, and no per-file digests. Unpack rules U-01 to U-14 bound what an archive may contain; the placement table of the plan names the files the installer takes from it. The plan therefore pins, per placed file, the ARTIFACT digest (from this manifest, through the verification record) and the member path inside that artifact — never a per-file digest, which nothing can know before root unpacks the verified copy at S-09. S-11 and S-29 compare each placed file with the file root itself unpacked at S-09 from that verified copy. (Adding a per-file digest list to this format is the alternative; v0 does not take it, because the artifact digest already binds every byte the unpack can produce under U-01 to U-14.)
- No profile pins. The install book lets the manifest pin the exact version of a role profile; profiles are outside v0. The profile bytes the plan installs come from an artifact of the approved set named in the placement table, or the step that needs them is not runnable (plan S-29).

## Vectors

All vectors are in `vectors/`, indexed by `vectors/index.json`. Every vector is shape-only: digests are synthetic, and the expected result is the result of the verifier's shape mode (V-04 to V-14), not of an installation.

| File | Expected | Rule shown | Valid neighbour |
| --- | --- | --- | --- |
| `valid-01-minimal-set.json` | `shape_ok` | two products, one host artifact each | n/a |
| `valid-02-multi-platform.json` | `shape_ok` | release candidates, several platforms | n/a |
| `valid-03-three-purposes.json` | `shape_ok` | all three purposes | n/a |
| `invalid-01-unknown-member.json` | `manifest_unknown_member` | unknown member | valid-01 |
| `invalid-02-duplicate-member.json` | `manifest_duplicate_member` | duplicate member | valid-01 |
| `invalid-03-missing-member.json` | `manifest_missing_member` | missing member | valid-01 |
| `invalid-04-wrong-digest-length.json` | `manifest_value_invalid` | wrong digest length | valid-01 |
| `invalid-05-kind-unknown.json` | `manifest_kind_unknown` | unknown format version | valid-01 |
| `invalid-06-platform-unknown.json` | `platform_unknown` | a platform the installer does not know | valid-01 |
| `invalid-07-not-canonical.json` | `manifest_not_canonical` | non-canonical bytes | valid-01 |
| `invalid-08-product-listed-twice.json` | `product_duplicate` | a product listed twice | valid-03 |
| `invalid-09-artifact-name-twice-in-set.json` | `artifact_duplicate` | one artifact name under two products | valid-03 |
| `invalid-10-two-artifacts-one-platform.json` | `artifact_platform_duplicate` | more than one artifact would match a host | valid-02 |
| `invalid-11-one-digest-two-names.json` | `artifact_digest_duplicate` | one digest listed twice | valid-03 |
| `invalid-12-no-first-install-product.json` | `first_install_product_count` | no first-install product | valid-01 |
| `invalid-13-two-first-install-products.json` | `first_install_product_count` | two first-install products | valid-01 |
| `invalid-14-format-not-allowed.json` | `artifact_format_not_allowed` | a format the purpose does not allow | valid-01 |
| `invalid-15-version-leading-zero.json` | `manifest_value_invalid` | a version no release tag can have | valid-01 |
| `invalid-16-version-rc-zero.json` | `manifest_value_invalid` | `rc.0` | valid-02 |
| `invalid-17-name-only-dots.json` | `manifest_value_invalid` | a dot name | valid-01 |
| `invalid-18-size-zero.json` | `manifest_value_invalid` | size 0 | valid-01 |
| `invalid-19-uppercase-digest.json` | `manifest_value_invalid` | uppercase hexadecimal | valid-01 |
| `invalid-20-no-products.json` | `manifest_value_invalid` | empty set | valid-01 |
| `invalid-21-number-not-shortest-integer.json` | `manifest_not_canonical` | a number that is not a shortest decimal integer | valid-01 |
| `invalid-22-no-artifact-for-host.json` | `artifact_missing_for_host` on a `linux/amd64` host | zero artifacts match the host for one product | valid-01 |
| `invalid-23-round1-members.json` | `manifest_unknown_member` | the removed `dependencies` member | valid-01 |
| `invalid-24-first-install-not-linux.json` | `artifact_format_not_allowed` | a first-install artifact for a non-Linux platform | valid-02 |
| `invalid-25-purpose-unknown.json` | `manifest_value_invalid` | a purpose outside the closed set | valid-01 |
| `invalid-26-format-unknown.json` | `manifest_value_invalid` | a format outside the closed set (round 4: exactly this one member of valid-01 varies) | valid-01 |
| `invalid-27-source-commit-malformed.json` | `manifest_value_invalid` | a source commit of 39 digits | valid-01 |
| `invalid-28-set-label-malformed.json` | `manifest_value_invalid` | a set label outside the pattern | valid-01 |
| `invalid-29-no-artifacts.json` | `manifest_value_invalid` | an empty artifact list | valid-01 |
| `invalid-30-nested-unknown-member.json` | `manifest_unknown_member` | an unknown member in a nested object | valid-01 |
| `invalid-31-nested-missing-member.json` | `manifest_missing_member` | a missing member in a nested object | valid-01 |
| `invalid-32-platform-wrong-type.json` | `manifest_value_invalid` | a platform that is not a string | valid-01 |
| `invalid-33-set-version-malformed.json` | `manifest_value_invalid` | a set-level version defect | valid-01 |
| `invalid-34-size-negative.json` | `manifest_value_invalid` | a negative size in canonical shortest form | valid-01 |

Rules without a file vector, and why, are listed under `not_shipped_as_files` in the index. Every refusal code of every mode has a file there or a harness rule there, `first_install_mismatch` (V-19) and the second-build codes (V-20) included.

## Open points of the format

| Point | Why it is open | Where it is tracked |
| --- | --- | --- |
| The final signing scheme and trust root | Decided by the owner: the product's trust root is the owner's offline publisher key, with its fingerprint in the install command. The install book describes a different final scheme (offline threshold keys under an update root, part signatures, a log); the book text must follow the owner's decision. Until the offline key exists, the owner-approved set digest is the interim trust root of the test profile (TP-1). When the key exists, the format gains signed metadata and this file changes. | file 1, D-M11; file 5, reference table (OD-2) |
| The `kind` prefix | The repository is decided. The component part of the wire name is not named by any source; the installer specification in that repository names it. | the installer specification (MISSING) |
| The values of the operational caps | Accepted for the test machine and recorded as provisional. The install book assigns them to a platform release specification that does not exist. | file 5, reference table (OD-17) |
| Further members of the book's part record (declared effects, service-state compatibility, encodings) | They belong to the platform release specification; v0 carries only name, version, platform, purpose, size, digest and unpacked size. | with the final scheme, above |
| The exact CCJ-1 text | This package cites CCJ-1 as the dispatcher specification describes it. The registrar specification that defines it was not among the references. | HANDOFF |
