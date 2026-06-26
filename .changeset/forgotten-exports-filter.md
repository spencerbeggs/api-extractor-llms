---
"api-extractor-llms": minor
---

## Features

### `filter` option and `isEmittable` predicate

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
