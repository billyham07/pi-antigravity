# Dynamic Antigravity model discovery

## Goal

Make Antigravity the source of truth for the Pi model catalog so newly released models become selectable without publishing a new extension release just to add IDs.

Keep Pi as the only harness. This extension should remain a native Pi provider and must not shell out to `agy` or delegate the agent loop to another CLI.

## Current state

`src/index.ts` registers `ANTIGRAVITY_MODELS` directly with `pi.registerProvider()`. `ANTIGRAVITY_MODELS` and `ANTIGRAVITY_ROUTING` are maintained statically in `src/models/models.ts`.

The extension already has Antigravity OAuth, Cloud Code Assist transport, streaming, diagnostics, usage/quota fetching, and `/antigravity.models`, so this work should reuse those pieces rather than introduce a second transport.

## Target architecture

1. Fetch the live Antigravity model catalog with the same authenticated backend source used by `/antigravity.models` / usage discovery.
2. Normalize runtime model IDs into public Pi models.
3. Group runtime variants such as `*-low`, `*-medium`, and `*-high` into one public model with an appropriate `thinkingLevelMap`.
4. Preserve runtime IDs in a routing table used by `getAntigravityRequestModelId()`.
5. Expose the normalized catalog through Pi's provider model-refresh mechanism when available.
6. Maintain a last-known-good cache so startup does not depend on network access and transient discovery failures do not remove working models.
7. Keep a small static seed catalog for cold start and discoverability. Seed IDs must request their own runtime IDs; they must not rewrite the selected model to another generation.

## Design constraints

- No `agy` subprocess and no second agent loop.
- OAuth and request streaming remain unchanged unless required by the provider API.
- Unknown new Gemini model IDs should be usable without a code release when backend metadata is sufficient.
- Do not guess exact capabilities when the backend does not advertise them. Use conservative family defaults and clearly isolate fallback inference.
- Existing pinned runtime overrides (`ANTIGRAVITY_RUNTIME_MODEL`) must continue to work.
- Existing routing workarounds such as `gemini-pro-agent` should remain expressible as explicit overrides on top of discovery.
- Existing `/antigravity.models`, `/antigravity.usage`, diagnostics, image generation, and model-routing tests must keep working.
- Backend model visibility is scoped to the current OAuth/auth surface. Dynamic discovery does not by itself enable models the token cannot see.
- **No silent runtime downgrade.** If the selected model 404s (for example Gemini 3.8 Flash on this auth surface), surface the error. Never remap it to 3.7 or another model.
- Investigating a separate Antigravity CLI OAuth client / entitlement field is an auth-layer task, not part of model discovery.

## Suggested implementation

### `src/models/discovery.ts`

Add functions roughly along these lines:

```ts
export async function discoverAntigravityModels(
  apiKey: string,
  signal?: AbortSignal,
): Promise<{
  models: ProviderModelConfig[];
  routing: Record<string, AntigravityRouting>;
}>;
```

Responsibilities:

- fetch live model metadata;
- identify model families / public IDs;
- group thinking variants;
- create `ProviderModelConfig` entries;
- produce runtime routing data.

### `src/models/cache.ts`

Persist the normalized last-known-good catalog. Follow Pi's existing conventions if there is already a model cache helper in the current dependency version.

The cache must be replace-on-success only. A failed or empty discovery response must not erase a valid cached catalog.

### `src/models/models.ts`

Refactor static data into three layers:

- conservative fallback model definitions;
- explicit routing/capability overrides for exceptional backend IDs;
- helpers that consume the latest discovered catalog/routing snapshot.

`getAntigravityRequestModelId()` should resolve against discovered routing first, then explicit compatibility overrides, then fall back to the public model ID.

### `src/index.ts`

Register the provider with the fallback/cached model list, then wire live refresh using Pi's provider refresh API supported by the current `@earendil-works/pi-coding-agent` version.

Important: verify the exact current provider API/types from the installed Pi dependency rather than assuming a `refreshModels` signature.

## Runtime grouping rules

Do not hard-code only Gemini 3.8. The parser should handle future generations where possible.

Examples:

```text
gemini-3.8-flash-low
gemini-3.8-flash-medium
gemini-3.8-flash-high
```

becomes:

```text
public id: gemini-3.8-flash
levels: low, medium, high
routing.low    -> gemini-3.8-flash-low
routing.medium -> gemini-3.8-flash-medium
routing.high   -> gemini-3.8-flash-high
```

Runtime names that do not match a safe grouping pattern should be exposed conservatively as individual selectable models rather than silently discarded.

## Tests / acceptance criteria

Add deterministic unit tests using fixture model catalogs; tests must not require real Google credentials.

Minimum cases:

- discovers a new `gemini-3.8-*` family not present in the static fallback;
- groups low/medium/high variants correctly;
- handles a single runtime model with no thinking suffix;
- preserves Claude/GPT models;
- explicit legacy routing overrides still win where required;
- discovery failure falls back to cached/static models;
- empty backend response does not wipe the last-known-good cache;
- `ANTIGRAVITY_RUNTIME_MODEL` override remains functional;
- `bun run check` passes.

## Definition of done

After signing in, a model newly enabled by Antigravity should appear in Pi's model picker after provider refresh without editing `ANTIGRAVITY_MODELS` or publishing another release solely for a model ID update.
