<!-- MEX-SIGNED-REGION-START -->
Author: npub1_MARKDOWN_V2_AUTHOR_PLACEHOLDER (replace with the version-2 author's npub before sealing)
Base: markdown
Previous version: 1
Previous file_sha256: 3964c748f08e2df1c3c1cf8ccf7efe3a9d516f2ae9c2855675a570f084eecc4e
Previous author signature: PENDING_MARKDOWN1_SIGNATURE (the MEX-MARKDOWN-1 author's signature of the hash above)

# MEX-MARKDOWN-2

## Abstract
This document is the second feature increment of Markup Extensions (MEX) on the Markdown base. It inherits, in full, the architecture, four lineage laws, adoption governance, compliance model, file format, and Renderer safety rule defined in MEX-CORE; the Markdown base binding and External Base Lock defined in MEX-MARKDOWN-0; and the version-1 token dictionary defined in MEX-MARKDOWN-1. It adds one construct: the `{grid}` layout grid.

## What the grid is (and is not)
`{grid}` is a **layout** primitive: a two-dimensional arrangement of cells, where each cell holds arbitrary block content (paragraphs, images, lists, code blocks, a GFM table, MEX tokens, even a nested grid). It is the rich-content answer to the thing a GFM table cannot do: put real content side by side, with control over sizing.

It is **not** a data table. It has no header row and no column data semantics; a GFM table (Layer 1) remains the construct for tabular data. The division is deliberate: the grid arranges *layout*, a table arranges *data*. To place data inside a grid, nest a GFM table in a cell.

## Category
MEX-CORE names two kinds of extensions (static text decorations, dynamic value tokens). `{grid}` is a third kind: a **block container** that wraps nested block content. A future MEX-CORE revision SHOULD record this category; it is noted here so implementers treat the grid as a container, not as a token with a text payload.

## Syntax
The grid uses three block markers, each of which MUST appear alone on its own line (ignoring leading and trailing spaces):

- `{grid <attributes>}` opens a grid.
- `{cell}` opens a cell. The cell's content runs until the next `{cell}` or the matching `{/grid}`.
- `{/grid}` closes the grid.

Content between the `{grid ...}` line and the first `{cell}` is ignored (keep it empty). Every cell, including the first, is introduced by its own `{cell}`.

All three markers use only the brace family and plain keywords, so they satisfy the External Base Lock (MEX-MARKDOWN-0): no `*`, `_`, `` ` ``, `#`, `>`, `-`, `+`, `|`, `( )`, `< >`, or `[ ]` appears anywhere in grid syntax.

### Attributes
Attributes on the `{grid}` line are space-separated `key=value` pairs. Unknown keys are ignored.

- **`cols`** (REQUIRED): the number of columns, a positive integer. Cells **auto-flow**: they fill left to right and wrap to a new row every `cols` cells, so the number of rows is `ceil(cellCount / cols)`. A `{grid}` with no valid `cols` is left as literal text (degrades).
- **`widths`** (optional): the relative widths of the columns. Omitted, all columns are equal. See Size lists.
- **`heights`** (optional): the relative heights of the rows. Omitted, every row sizes to its content. See Size lists.

### Size lists
A size value (`widths` or `heights`) is either the keyword `equal` (all tracks equal) or a `:`-separated list of terms. Each term is one of:

- a positive number, a relative weight, e.g. `2`;
- a repeat `AxB`, meaning the number `A` repeated `B` times (`B` a positive integer). `x` is the lowercase letter, chosen because `*` is Markdown emphasis and barred by the External Base Lock. Example: `1x4` means `1:1:1:1`. The list is split on `:` first, then each term is expanded, so `2x2:4` means `2:2:4`.

Resolution:
- **`widths`**: the expanded list is matched to the columns in order. Fewer entries than `cols`, the remaining columns default to weight `1`. More entries than `cols`, the extras are ignored.
- **`heights`**: omitted, all rows are content-auto. Provided, the expanded list is matched to the rows in order; rows beyond the list default to weight `1`. `heights=equal` makes all rows equal height.

Weights are relative and self-normalizing: `2:2:2` renders the same as `1:1:1`. So "all tracks the same size" needs no list at all (omit `widths`), and the magnitude of a uniform value is irrelevant.

## Cell content and nesting
A cell's content is parsed as ordinary Layer 1 plus any MEX tokens, with no restriction: multiple paragraphs, lists, images, code blocks, blockquotes, GFM tables, and MEX-MARKDOWN-1 tokens are all valid. A cell MAY contain another `{grid}`; the parser counts `{grid}` and `{/grid}` so an inner `{/grid}` closes the innermost open grid. A nested GFM table needs no special handling, it is not a grid marker.

## Parsing rules
1. `{grid ...}`, `{cell}`, and `{/grid}` are recognized only when each is the sole content of its line and is **not inside a fenced or indented code block**. Layer 1 code fences are resolved first; a literal `{cell}` inside a code block is code, not a marker.
2. Cells are delimited by `{cell}` lines within the current grid depth; a cell ends at the next `{cell}` or the matching `{/grid}`.
3. Grid nesting is matched by depth-counting `{grid}` / `{/grid}`.
4. An unterminated grid (end of its containing block or document with no matching `{/grid}`) is closed implicitly at that boundary; a renderer MAY instead leave the unterminated `{grid}` and its content as literal text.

## Rendered output
A grid renders as a CSS grid container; each cell is a grid item holding that cell's rendered block content.

- `widths` maps to `grid-template-columns` as `fr` units (e.g. `widths=1:2` gives `1fr 2fr`); omitted or `equal` gives `repeat(cols, 1fr)`.
- `heights`, when provided, maps to `grid-template-rows` as `fr` units; `equal` gives `repeat(rows, 1fr)`; omitted leaves rows to size to content (`grid-auto-rows: auto`).
- Cells are placed in source order by the default row auto-flow.

Example skeleton:
```html
<div class="mex-grid" style="display:grid;grid-template-columns:1fr 2fr;grid-template-rows:2fr 2fr 4fr;">
  <div class="mex-grid-cell"><!-- cell 1 rendered content --></div>
  <!-- ... one div per cell ... -->
</div>
```

## Renderer safety
Per MEX-CORE (Renderer safety), attribute values are data: a renderer MUST validate and escape them. A malformed `cols`, `widths`, or `heights` is ignored or degrades the grid to literal text, never injected into output. Cell content is rendered through the same sanitizing Layer 1 plus MEX pipeline as any other content.

## Graceful degradation
A client that does not support `{grid}` (MEX-unaware, or a MEX client that does not implement version 2) MUST leave the `{grid ...}`, `{cell}`, and `{/grid}` lines as readable literal text and render each cell's content in source order, stacked vertically. Because every cell holds ordinary Layer 1 plus MEX content, the document stays readable; it loses only the side-by-side layout.

## Worked example
A 2-column, 3-row grid: left column weight 1, right weight 2; rows 2, 2, 4.

```
{grid cols=2 widths=1:2 heights=2x2:4}

{cell}
Just a normal paragraph of text.

{cell}
- [x] First task done
- [ ] Second task to do

{cell}
![a dog](dog.jpg)

{cell}
[Visit the site](https://example.com)

{cell}
| Q | Rev |
|---|-----|
| 1 | 10  |
| 2 | 14  |

{cell}
{>&&Approved and underlined&&>}

{/grid}
```

Cells auto-flow into three rows of two: (text, checklist), (image, link), (GFM table, right-aligned underline). The last cell nests MEX-MARKDOWN-1's underline `&&...&&` inside MEX-MARKDOWN-1's align-right `{>...>}`. An unsupporting client stacks the six cells vertically in that order.
<!-- MEX-SIGNED-REGION-END -->

<!-- MEX-SEAL
file_sha256: 937b71e24111372f9d95cbe045fdcb1a7d9e5ee02688755b2ef8ec564ee057f7
signature: PENDING_SIGNATURE
-->
