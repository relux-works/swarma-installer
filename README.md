# swarma-installer

The installer of the Swarma platform: the release-set manifest, the verifier
and the install plan.

**Status: draft. Nothing here is installable yet.** The first content is a
design and contract package (Markdown and JSON); the installer program comes
after that contract is accepted.

## Layout

| Path | Content |
| --- | --- |
| `contract/installer-v0/01-book-mapping.md` | Every decision of the package, with its source in the Swarma books and specs and its mark (same, refines, conflicts, new). |
| `contract/installer-v0/02-release-manifest-v0.md` | The format `swarma release-manifest/0`: one object per release set. |
| `contract/installer-v0/release-manifest-v0.schema.json` | JSON Schema (2020-12) for the parsed manifest shape. |
| `contract/installer-v0/vectors/` | Valid and invalid manifest vectors, `index.json` with the expected verdict of each, and the unpack cases. |
| `contract/installer-v0/03-verifier-contract.md` | The verifier: modes, ordered checks, result codes, limits and the unpack contract. |
| `contract/installer-v0/04-install-plan-colony1.md` | The install plan for the first test machine, under a closed test profile of nine named deviations, with every step, its gate, its read-back and its removal. |
| `contract/installer-v0/05-open-decisions.md` | Decisions still open, with their owners, and the reference table of those already taken. |

## Tools

| Tool | Use | Command |
| --- | --- | --- |
| Python 3 with `jsonschema` (Draft 2020-12) | Check every vector against the schema | `python3 -c 'import json,jsonschema,pathlib; d=pathlib.Path("contract/installer-v0"); s=json.load(open(d/"release-manifest-v0.schema.json")); jsonschema.Draft202012Validator.check_schema(s); print("schema ok")'`, then validate each vector file and compare each result with the `pass`/`fail` lists of `schema_expectation` in `vectors/index.json` (not with the per-vector `expect`, which is the verifier's shape-mode result; eight invalid vectors pass the schema and are refused only by the verifier). |

Outputs are printed only; nothing is written.
