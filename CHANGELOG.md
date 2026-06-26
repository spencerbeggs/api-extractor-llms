# api-extractor-llms

## 0.2.0

### Features

* [`64601e6`](https://github.com/spencerbeggs/api-extractor-llms/commit/64601e6ec4236d50e3d968b9f1b47a66b02fb82d) ### `filter` option and `isEmittable` predicate

`renderPackage` now accepts an optional `filter` predicate on `RenderPackageOptions` that controls which top-level `ApiItem`s appear in the rendered markdown corpus. When `filter` is provided it fully replaces the built-in emit rule.

The built-in rule — now exported as `isEmittable(item)` — drops compiler-synthetic forgotten exports (items where `isExported === false`, such as the `*_base` classes TypeScript hoists for Effect class mixins) from the rendered output while leaving them untouched in the `.api.json` model.

```ts
import { renderPackage, isEmittable } from "api-extractor-llms";

// Compose the default rule with your own predicate:
const docs = await renderPackage(model, {
  filter: (item) => isEmittable(item) && !item.displayName.startsWith("_"),
});
```

**Default behavior change:** `renderPackage` now silently drops forgotten exports even when no `filter` is provided. For consumers on the canonical API Extractor pipeline this is the intended corpus-hygiene fix — synthetic base classes and re-exported implementation details no longer appear in LLM context. If you need every item rendered regardless of `isExported`, pass `filter: () => true` to opt out.

## 0.1.0

### Features

* [`b651c24`](https://github.com/spencerbeggs/api-extractor-llms/commit/b651c247a47b65a6390bdabaeaa7bd299cd2a069) ### Markdown output system with injectable seams

Renders Microsoft API Extractor `.api.json` models into LLM-lean markdown. Two injection points let each consumer control framing while sharing the body rendering logic:

* `FrontmatterRenderer` — receives item metadata and returns the frontmatter block (or an empty string to omit it)
* `RouteFormatter` — receives an item reference and returns the crosslink URL

```ts
import { renderPackage, loadApiModel } from "api-extractor-llms";

const model = loadApiModel("./docs/my-package.api.json");
const docs = renderPackage(model, {
  packageName: "my-package",
  frontmatter: (meta) => `---\ntitle: ${meta.name}\n---\n\n`,
  routeFor: (ref) => `/api/${ref.kind}/${ref.slug}`,
});
```

### Rendering API

* `renderPackage(model, options)` — renders all top-level items in an `ApiPackage` and returns one `RenderedDoc` per item
* `renderItem(item, options)` — renders a single `ApiItem` to a `RenderedDoc`

### Model loading

* `loadApiModel(path)` — loads an `.api.json` file produced by API Extractor and returns a typed `ApiModel`

### Cross-linking

* `CrossLinker` — matches known API item names in prose as whole-word strings and rewrites them to markdown links. Longest names are matched first to avoid partial matches. Does not resolve TSDoc `{@link}` tags.

### Type-signature formatting

* `TypeSignatureFormatter` — renders API Extractor type signatures as compact markdown-ready strings

### TSDoc extraction helpers

* `extractPlainText(docNode)` — strips TSDoc markup and returns plain text
* `getSummary(item)`, `getParams(item)`, `getReturns(item)`, `getExamples(item)` — extract named TSDoc sections
* `getDeprecation(item)`, `getReleaseTag(item)`, `hasModifierTag(item, tag)` — extract modifiers and release tags

### Public types

`ApiItemRef`, `DocMeta`, `RenderedDoc`, `RenderPackageOptions`, `RenderItemOptions`, `ItemKindSlug`, `FrontmatterRenderer`, `RouteFormatter`
