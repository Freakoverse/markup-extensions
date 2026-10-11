# MEX: Markup Extensions

MEX (Markup EXtensions) is an additive, versioned, adoption-governed formatting layer that sits on top of a base markup language (Layer 1). It adds new tokens a base does not have (underline, spoiler, highlight, localized timestamps, alignment, a layout grid, and so on) without ever modifying the base, and content degrades to readable plain text on any client that does not support it.

The base language is a parameter, not a fixed choice: Markdown today, djot or another lightweight markup later. Each base is bound by its own profile that descends from a single, shared, base-agnostic core.

## Files

| File | Role |
|------|------|
| [MEX-CORE.md](MEX-CORE.md) | Base-agnostic core (genesis). Architecture, the four lineage laws, governance, file format, and the document declaration. Names no base, defines no tokens. The root of the chain. |
| [MEX-MARKDOWN-0.md](MEX-MARKDOWN-0.md) | Markdown profile genesis. Binds `base = markdown` to CommonMark 0.31.2 + GFM 0.29, states the Markdown External Base Lock, and holds the Layer 1 reference. Defines no tokens. Descends from MEX-CORE. |
| [MEX-MARKDOWN-1.md](MEX-MARKDOWN-1.md) | Markdown version-1 token dictionary (underline, spoiler, highlight, timestamp, alignment). Descends from MEX-MARKDOWN-0. |
| [MEX-MARKDOWN-2.md](MEX-MARKDOWN-2.md) | Markdown version-2: the `{grid}` layout grid. Descends from MEX-MARKDOWN-1. |
| [mex-hash.mjs](mex-hash.mjs) | Reference canonicalizer / hasher. Computes a file's `file_sha256` over its canonicalized signed region. |
| [../NIP-MEX.md](../NIP-MEX.md) | Nostr transport binding: how an event advertises its MEX baseline via the `mex` tag. |

Each base profile is named `MEX-<BASE>-N`: the genesis is `-0`, token dictionaries are `-1`, `-2`, and so on. A future base would be added as a sibling lineage, for example `MEX-DJOT-0.md` (binding) and `MEX-DJOT-1.md` (dictionary), also descending from MEX-CORE. Nothing in the core or the Markdown branch changes.

## How a document declares MEX

Five required fields, carried in the host format's native comment/metadata facility or, on Nostr, in a tag:

```
["mex", "<base>", "<version>", "<lineage>", "<hash>", "<signature>"]
```

- `base` dispatch discriminator (e.g. `markdown`); unsupported base degrades immediately.
- `version` the MEX version within that base's lineage.
- `lineage` the lineage author's npub (disambiguates forks).
- `hash` the targeted spec file's `file_sha256`; this is what selects the exact dictionary.
- `signature` the lineage author's signature of that hash (carried for redundancy).

See MEX-CORE "Declaring MEX in a document" for the per-file-type forms, and NIP-MEX for the Nostr tag.

## The hash chain (self-verifying spec files)

Each spec file carries its own hash and signature inside the file, excluded from the region they cover (the same idea as a Nostr event's `id`/`sig`). The signed region is the content strictly between `<!-- MEX-SIGNED-REGION-START -->` and `<!-- MEX-SIGNED-REGION-END -->`; the `MEX-SEAL` block after the end marker holds `file_sha256` + `signature` and is excluded from the hash.

The chain is a tree rooted at the core:

```
MEX-CORE
   └─► MEX-MARKDOWN-0 (markdown genesis)
          └─► MEX-MARKDOWN-1 (markdown v1 tokens)
                 └─► MEX-MARKDOWN-2 (markdown v2: grid)
```

Each child's `Previous file_sha256` is its parent's `file_sha256`.

### Recomputing the seals

Compute one file's hash:

```bash
node docs/MEX/mex-hash.mjs docs/MEX/MEX-CORE.md
```

Because a change to a parent changes its hash (and therefore every child's `Previous file_sha256`), reseal in order, parent before child: MEX-CORE, then MEX-MARKDOWN-0 (set its `Previous` to CORE's new hash, then recompute), then MEX-MARKDOWN-1, then MEX-MARKDOWN-2. Writing the seal never changes the hash, since the seal sits outside the signed region.

## Status

- Markdown is the only base profile so far; djot and others are intentionally deferred (djot is not yet stable).
- Author npubs and signatures are `PENDING` placeholders. Before sealing for real: insert each file's author npub into its signed region, recompute the chain in order, then sign each `file_sha256` and fill the `signature` (and each child's predecessor-signature) fields.

This README is documentation only; it is not part of the signed chain.
