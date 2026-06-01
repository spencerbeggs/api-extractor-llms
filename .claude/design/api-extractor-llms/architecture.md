---
status: current
module: api-extractor-llms
category: architecture
created: 2026-06-01
updated: 2026-06-01
last-synced: 2026-06-01
completeness: 90
related: []
dependencies: []
---

# api-extractor-llms architecture

A standalone library that renders Microsoft API Extractor `.api.json` models into
LLM-lean markdown. Originally developed inside `savvy-web/systems` and consumed by
`@savvy-web/mcp`; extracted into this repository so it can be published and reused
independently — by `@savvy-web/systems` and by `rspress-plugin-api-extractor`, which
will adopt it in place of its own emit logic.

## Table of Contents

- [Overview](#overview)
- [Current State](#current-state)
- [The Load-Bearing Design Decision](#the-load-bearing-design-decision)
- [Modules](#modules)
- [Data Flow](#data-flow)
- [Provenance and Porting](#provenance-and-porting)
- [Boundaries](#boundaries)
- [Rationale](#rationale)

## Overview

`api-extractor-llms` turns a package's API Extractor model into plain markdown docs
suitable for LLM consumption. The body of every rendered doc — item H1, optional
deprecation notice, summary, fenced `ts` signature, params table, returns, container
members, examples and cross-links — is identical regardless of the downstream
consumer. The only things that differ across consumers are the **frontmatter block**
prepended to each doc and the **crosslink URL scheme** used for intra-package links.
Both are injected services supplied by the caller.

**Package:** `api-extractor-llms` on npm (unscoped, top-level); `@spencerbeggs/api-extractor-llms`
on GitHub Packages. The scoped name exists only because GitHub Packages requires an owner scope;
the `rslib.config.ts` `transform()` rewrites `pkg.name` to the scoped form for the
`npm.pkg.github.com` target only. npm is the canonical, unscoped name.
**Repository:** `spencerbeggs/api-extractor-llms` (source in `src/`)
**Build:** ESM-only via `@savvy-web/rslib-builder` (`NodeLibraryBuilder`)

## Current State

A standalone published package. The pure logic was first written as a net-new library
in Phase B of the MCP Resource Layer work inside `savvy-web/systems`, where
`@savvy-web/mcp` consumes it as a `devDependency` (a build-time generator, not a server
runtime dependency). It has since been extracted into this repository so it can be
versioned and published on its own. The `rspress-plugin-api-extractor` adoption — having
the plugin replace its own emit logic with this shared output system — is a documented
follow-up (Phase C), gated on publishing the library.

## The Load-Bearing Design Decision

One shared markdown output system with two injectable services.

The body rendering is identical across consumers. The mcp generator and the rspress
plugin differ in exactly two ways: the frontmatter block at the top of each doc (silk
YAML vs RSPress front-matter) and the crosslink URL scheme (`silk://packages/<pkg>/api/…`
vs `/packages/<pkg>/api/…`). Duplicating body rendering to accommodate these differences
would produce two diverging forks that must be kept in sync. Instead, `render.ts` exposes
two injection points:

- **`FrontmatterRenderer`** — a function `(meta: DocMeta) => string` that produces the
  frontmatter block (including trailing blank line) for one doc, or `""` for no
  frontmatter. `DocMeta` carries `name`, `kind`, `slug`, `summary` and `packageName`.
- **`RouteFormatter`** — a function `(ref: ApiItemRef) => string` that turns an item
  reference into a crosslink URL. `ApiItemRef` carries `name`, `kind` and `slug`.

Both are optional on `RenderPackageOptions`. With neither injected, `renderPackage` emits
bare bodies — useful for tests and for a plain dump without consumer-specific wiring.

The `CrossLinker` receives the `RouteFormatter` at construction and uses it whenever it
inlines a link. This is the only place the URL scheme appears; the scanner (whole-word
matching, skip code spans and existing links, longest-name-first) is fully shared.

## Modules

See `src/` for implementation.

**`model-loader.ts`** — `loadApiModel(path)` resolves and loads a `.api.json` file,
returning the package's `ApiPackage`. Throws a descriptive error when the file is absent.

**`tsdoc.ts`** — pure TSDoc extraction helpers ported from
`rspress-plugin-api-extractor`'s `ApiParser` statics: `extractPlainText`, `getSummary`,
`getParams`, `getReturns`, `getExamples`, `getDeprecation`, `getReleaseTag`,
`hasModifierTag`. Each reads documentation off an `ApiItem` and returns plain data — no
rendering, no I/O.

**`formatter.ts`** — `TypeSignatureFormatter` ported from `rspress-plugin-api-extractor`.
Takes an API Extractor `Excerpt` (its `spannedTokens`) and produces a clean one-or-few-line
signature, stripping `export`/`declare` from the leading token. The line-wrap heuristic
splits at `|`/`&` operators when a line would exceed `maxLineLength`.

**`cross-linker.ts`** — `CrossLinker` ported from `rspress-plugin-api-extractor`. Scans
prose for known item names and replaces whole-word matches with markdown links, skipping
text inside backtick code spans or existing markdown links. The link destination comes
from the injected `RouteFormatter`. Longest names match first to prevent shorter prefixes
from shadowing longer names.

**`render.ts`** — the output system. `renderItem(item, opts)` produces the shared markdown
body for one API item. `renderPackage(apiPackage, opts)` walks the first entry point, builds
a `CrossLinker` from the injected `RouteFormatter` (if any) and assembles each doc as
injected frontmatter plus cross-linked body, returning `RenderedDoc[]`.

**`types.ts`** — the public type surface: `ItemKindSlug`, `ApiItemRef`, `DocMeta`,
`RouteFormatter`, `FrontmatterRenderer`, `RenderedDoc`, `RenderPackageOptions`.

**`index.ts`** — barrel re-exporting the full public surface (including
`RenderItemOptions` from `render.ts`).

## Data Flow

```text
.api.json file
      ↓
  loadApiModel → ApiPackage
      ↓
  renderPackage(pkg, { packageName, routeFor, frontmatter })
      ↓  (per entry-point member)
  renderItem → body string
      ↑ uses tsdoc.ts helpers, TypeSignatureFormatter, CrossLinker(routeFor)
      ↓
  frontmatter(meta) → frontmatter block string
      ↓
  RenderedDoc { name, kind, slug, summary, packageName, markdown }
```

The caller receives `RenderedDoc[]` and is responsible for all I/O (writing files,
generating paths, determining output directory layout).

## Provenance and Porting

The pure logic in `model-loader.ts`, `tsdoc.ts`, `formatter.ts` and `cross-linker.ts` is a
faithful port from `spencerbeggs/rspress-plugin-api-extractor`. The port is intentionally
conservative — biome-ignore comments are carried through where dynamic property access
against TSDoc/API Extractor node internals is unavoidable. The `render.ts` output system and
the injection seams (`FrontmatterRenderer`, `RouteFormatter`) are net-new. This code first
lived in `savvy-web/systems` as `@savvy-web/api-extractor-llms` before being extracted into
this standalone repository.

## Boundaries

`api-extractor-llms` imports only `@microsoft/api-extractor-model` and `@microsoft/tsdoc`.
It depends on no consumer — neither `@savvy-web/mcp` nor `rspress-plugin-api-extractor` — so
it sits strictly below its consumers in their dependency graphs. Inside `savvy-web/systems`
this was a hard requirement to keep the turbo pipeline acyclic (mcp's build depends on
models from library packages; those library packages cannot depend back on mcp or on
api-extractor-llms). As a standalone package the same one-way relationship holds: consumers
depend on this library, never the reverse.

## Rationale

### Why one shared library rather than a fork per consumer

Two consumers (mcp generator, rspress plugin) would independently maintain body rendering,
TSDoc extraction and cross-linking logic. Any improvement — better param formatting, a new
TSDoc tag — would need to be applied in two places. The shared library means one improvement
reaches both consumers. Publishing it as a standalone package makes that sharing work across
repositories, not just within one monorepo. The injection seams keep the two real differences
(frontmatter, routes) from bleeding into the shared layer.

### Why the ported code stays conservative

The extraction helpers use dynamic property access against TSDoc/API Extractor node internals
in exactly the same way as the upstream source. Rewriting them to stricter TypeScript would
require either casting differently or wrapping each access in a type guard, producing more
code with the same runtime behavior. Faithfulness to the port is the safer choice when the
upstream has proven the approach.
