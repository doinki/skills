---
name: setup-hey-api
description: Interactively scaffold `@hey-api/openapi-ts` in a TypeScript project. Manual-trigger only — runs on `/setup-hey-api`, "set up hey-api", "configure openapi-ts", "scaffold OpenAPI/Swagger TypeScript SDK", "generate API client from swagger", or equivalents in any language. Collects one or more Swagger/OpenAPI URLs, then lets the user pick plugins across four sections — Core (multi-select), Client (single), Validator (single, optional), State Management (single, optional). Writes a single `openapi-ts.config.ts` that exports `defineConfig([...])` (one entry per spec), installs required npm packages with the detected package manager, and runs codegen. Never auto-triggers.
---

## Overview

This skill scaffolds the `@hey-api/openapi-ts` toolchain. It is **manual-trigger only**: invoke it only when the user explicitly asks. The interactive flow gathers Swagger/OpenAPI URLs and plugin choices, then produces a single `openapi-ts.config.ts` at the project root that drives generation for every spec the user entered.

## When to Use

Trigger only on explicit user request, such as:

- `/setup-hey-api`, `setup-hey-api`, `setup hey api`
- "set up hey-api", "configure openapi-ts", "init openapi-ts"
- "scaffold an OpenAPI/Swagger TypeScript SDK / client"
- "generate API SDK from swagger"
- Equivalent phrasings in any language

## When NOT to Use

- Do NOT auto-trigger from contextual signals (presence of `swagger.json`, opening `openapi-ts.config.ts`, importing a generated SDK, etc.).
- Do NOT use to hand-edit files inside an already-generated SDK directory — they are recreated on every run and your edits will be lost.
- Do NOT use for one-off `fetch` helpers or fully hand-written API clients.
- Skip if the working directory is not a Node/TypeScript project (no `package.json`).

## Prerequisites

Verify before starting the interactive flow:

1. A `package.json` exists in the project root.
2. A package manager can be detected from a lockfile:
   - `pnpm-lock.yaml` → `pnpm`
   - `yarn.lock` → `yarn`
   - `package-lock.json` → `npm`
   - `bun.lockb` / `bun.lock` → `bun`
   - If none found, ask the user.
3. Node ≥ 18 is generally available. Do not block; warn only if `engines.node` clearly forbids it.

## Workflow

Each step is interactive. Ask one focused question at a time, wait for the answer, and never silently assume.

### Step 1 — Confirm intent

State briefly what the skill will do (write `openapi-ts.config.ts`, install packages, run codegen) and confirm the user wants to proceed. Stop if they decline.

### Step 2 — Detect existing setup

Look in the project root for any of:

- `openapi-ts.config.ts` / `.js` / `.mjs` / `.cjs`
- Custom generator scripts (e.g. `scripts/codegen.js`, `scripts/openapi.js`)
- An obvious existing generated SDK directory (e.g. `src/client/`, `app/api/sdk/`)

If anything is found, ask the user one question with three choices:

- (a) Overwrite the existing file(s)
- (b) Abort the skill
- (c) Pick a different output directory and continue

Default behavior is **never** to overwrite without explicit confirmation.

### Step 3 — Collect Swagger/OpenAPI specs

Loop until the user signals "done":

1. "Enter the Swagger/OpenAPI URL (or an absolute file path)."
2. "Short name for this spec — lowercase kebab-case. It becomes the output sub-directory."
3. "Add another spec? (y/N)"

After the loop, ask once for the **base output directory** (default: `src/client`). Each spec's `output` becomes `<base>/<name>`.

If the host can make outbound HTTP requests, attempt a lightweight fetch of each URL and report HTTP status. On failure, let the user re-enter the URL, skip validation, or abort.

### Step 4 — Pick plugins (4 sections)

Present the full catalog in `## Plugin Catalog` below. Apply these rules:

| Section          | Selection rule       | Notes                                                                                                                                               |
| ---------------- | -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| Core             | **multi-select**     | `@hey-api/typescript` and `@hey-api/sdk` are mandatory (always included; user toggles only the extras `@hey-api/transformers`, `@hey-api/schemas`). |
| Client           | **single, required** | Default: `@hey-api/client-fetch`.                                                                                                                   |
| Validator        | **single, optional** | "None" is a valid choice.                                                                                                                           |
| State Management | **single, optional** | "None" is a valid choice.                                                                                                                           |

For each selected plugin that has options worth surfacing (see catalog), ask the user; otherwise apply documented defaults silently.

Cross-plugin wiring (ask the user, default = Yes when both are present):

- Validator + `@hey-api/sdk` → set `validator: true` on the SDK plugin so payloads are validated at call sites.
- `@hey-api/transformers` + `@hey-api/sdk` → set `transformer: true` on the SDK plugin.

The chosen plugin set is applied **uniformly to every spec** collected in Step 3.

### Step 5 — Optional runtime-config file

Ask: "Generate a runtime-config stub so you can set `baseUrl`, auth headers, or interceptors in one place? (Y/n)"

If yes:

1. Ask for the path (default: `<base output dir>/runtime-config.ts`).
2. Write a stub matching the chosen client (see `## Runtime Config Stubs`).
3. Set `runtimeConfigPath` on the client plugin entry in `openapi-ts.config.ts` to that path.

If no, skip and emit the client plugin without `runtimeConfigPath`.

### Step 6 — Confirm and write `openapi-ts.config.ts`

Render the final config to the user as a code block and ask: "Write `openapi-ts.config.ts`? (Y/n)"

If yes, write the file with this shape:

```ts
import { defineConfig } from '@hey-api/openapi-ts';

export default defineConfig([
  {
    input: '<spec-1-url>',
    output: '<base>/<spec-1-name>',
    plugins: [
      /* shared selection */
    ],
  },
  {
    input: '<spec-2-url>',
    output: '<base>/<spec-2-name>',
    plugins: [
      /* shared selection */
    ],
  },
]);
```

Per plugin: emit the object form `{ name: '<id>', ...options }` only when at least one option is set; otherwise emit the bare string `'<id>'`.

### Step 7 — Install dependencies

Resolve the dependency list:

- Always: `@hey-api/openapi-ts` (devDependency).
- Client runtime package — install both the Hey API plugin package and any peer HTTP library not bundled by it (see `## Installation Map`).
- Validator: the chosen library (`valibot`, `zod`, ...) as a regular dependency.
- State manager: the framework-specific package (`@tanstack/react-query`, `@pinia/colada`, ...).

Show the resolved install command(s) and run them with the detected package manager. Do NOT pass `--force`, `--legacy-peer-deps`, or skip-lockfile flags unless install fails and the user opts in afterwards. On failure, surface the exact stdout/stderr and stop.

### Step 8 — Run codegen

If `package.json` has no `scripts.codegen`, add it:

```json
{ "scripts": { "codegen": "openapi-ts" } }
```

If one already exists, ask for a new name (suggested default: `codegen:hey-api`).

Then execute `<pm> exec openapi-ts` (or `<pm> run <script-name>`). Stream output to the user. On non-zero exit, surface the error verbatim and stop — do not retry blindly.

### Step 9 — Summarize

Print a short summary listing:

- Files written (config, runtime-config stub, package.json script changes)
- Packages installed (dev vs runtime)
- Generated SDK directories
- The command to regenerate later (`<pm> run <script-name>`)

Stop. Do not take any further action.

## Plugin Catalog

Identifiers below are the exact strings accepted by `@hey-api/openapi-ts`'s `plugins` array. Use the object form `{ name: '<id>', ... }` whenever options are set.

### Core (multi-select; `typescript` and `sdk` are mandatory)

| ID                      | Purpose                                                              | Notable options                                                                                                                                                                                                                                                                                                 |
| ----------------------- | -------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `@hey-api/typescript`   | TypeScript types in `types.gen.ts`. **Always include.**              | `enums`: `false` (default) \| `'javascript'` \| `'typescript'`; `comments`: `true` (default); plus per-artifact `.name` / `.case` customization.                                                                                                                                                                |
| `@hey-api/sdk`          | Generated request functions / class. **Always include.**             | `operations.strategy`: `'flat'` (default, tree-shakeable) \| `'single'` (class-based); `operations.containerName`; `operations.nesting`; `auth: boolean` (built-in `bearer`/`basic` handling); `validator: true \| '<validator-id>'`; `transformer: true`; `responseStyle`: `'data'` \| `'fields'`; `examples`. |
| `@hey-api/transformers` | Runtime transformers for special types.                              | `dates: boolean` (default `false`, parses date strings to `Date`); `bigInt: boolean` (default `false`, forces `BigInt` numerics).                                                                                                                                                                               |
| `@hey-api/schemas`      | Runtime schemas (JSON Schema) generated from `#/components/schemas`. | `type`: `'json'` (default) \| `'form'`.                                                                                                                                                                                                                                                                         |

### Client (single, required)

| ID                                | Required runtime peers                     | Notable options                                                                                                                        |
| --------------------------------- | ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------- |
| `@hey-api/client-fetch` (default) | none (native `fetch`)                      | `baseUrl`; `auth` (string or `() => string`); `fetch` (custom instance); `runtimeConfigPath`.                                          |
| `@hey-api/client-ky`              | `ky`                                       | `baseUrl`; `auth`; `ky` (custom instance); `runtimeConfigPath`.                                                                        |
| `@hey-api/client-axios`           | `axios`                                    | `baseURL` (note capital URL); `auth`; `axios` (custom instance); `runtimeConfigPath`.                                                  |
| `@hey-api/client-nuxt`            | `nuxt`/`ofetch` (provided by Nuxt project) | `baseURL`; `auth`; `$fetch` (custom instance); `onRequest`; `runtimeConfigPath`.                                                       |
| `@hey-api/client-ofetch`          | `ofetch`                                   | `baseUrl`; `auth`; `ofetch` (custom instance); `onRequest` / `onResponse` / `onRequestError` / `onResponseError`; `runtimeConfigPath`. |

Other clients listed by Hey API (Angular, Next.js, Effect, Got) may be in preview or planned status. Do not offer them unprompted. If the user explicitly names one, configure best-effort and warn that support may be partial.

### Validator (single, optional)

| ID        | npm dep   | Notable options                                                                                                              |
| --------- | --------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `valibot` | `valibot` | `requests: boolean`; `responses: boolean`; `definitions: boolean`; `metadata: boolean \| (op) => object`; `.name` / `.case`. |
| `zod`     | `zod`     | Symmetrical to `valibot` (request/response/definition schemas + naming).                                                     |

Other validators referenced by Hey API (`ajv`, `arktype`, `joi`, `typebox`, `yup`) may be planned/not yet generally available. Do not offer them unprompted; on explicit request, configure best-effort and warn.

When a validator is selected and `@hey-api/sdk` is present, ask whether to set `validator: true` on the SDK plugin (default Yes).

### State Management (single, optional)

| ID                                     | npm dep                                |
| -------------------------------------- | -------------------------------------- |
| `@tanstack/react-query`                | `@tanstack/react-query`                |
| `@tanstack/vue-query`                  | `@tanstack/vue-query`                  |
| `@tanstack/svelte-query`               | `@tanstack/svelte-query`               |
| `@tanstack/solid-query`                | `@tanstack/solid-query`                |
| `@tanstack/angular-query-experimental` | `@tanstack/angular-query-experimental` |
| `@tanstack/preact-query`               | `@tanstack/preact-query`               |
| `@pinia/colada`                        | `@pinia/colada`                        |

Shared options across these plugins:

- `queryOptions: boolean` — function suffix `Options`
- `queryKeys: boolean \| { tags: boolean }` — function suffix `QueryKey`
- `infiniteQueryOptions: boolean` — suffix `InfiniteOptions`
- `infiniteQueryKeys: boolean` — suffix `InfiniteQueryKey`
- `mutationOptions: boolean \| { meta: (op) => object }` — suffix `Mutation`
- `mutationKeys: boolean`
- `meta`, `tags: boolean`, `.name`, `.case`

If a state manager is picked, ask which of `queryOptions` / `infiniteQueryOptions` / `mutationOptions` to generate (default: all three on).

## Installation Map

For the user-confirmed install step, compute the package list as follows:

- DevDependencies (always): `@hey-api/openapi-ts`
- Client → install the plugin package and the underlying HTTP lib (when not bundled):
  - `@hey-api/client-fetch` → `@hey-api/client-fetch`
  - `@hey-api/client-ky` → `@hey-api/client-ky`, `ky`
  - `@hey-api/client-axios` → `@hey-api/client-axios`, `axios`
  - `@hey-api/client-ofetch` → `@hey-api/client-ofetch`, `ofetch`
  - `@hey-api/client-nuxt` → `@hey-api/client-nuxt` (Nuxt provides `ofetch`)
- Validator → the chosen library only (`valibot`, `zod`, ...).
- State manager → the chosen package only (`@tanstack/react-query`, `@pinia/colada`, ...).

## Runtime Config Stubs

Substitute the actual `output` path of the first spec for `<path-to-generated-client>` (`client.gen.ts` lives inside it). Note the property name differs between clients.

### Fetch / Ky / OFetch (uses `baseUrl`)

```ts
import type { CreateClientConfig } from '<path-to-generated-client>/client/client.gen';

export const createClientConfig: CreateClientConfig = (config) => ({
  ...config,
  baseUrl: 'https://example.com',
});
```

### Axios / Nuxt (uses `baseURL`)

```ts
import type { CreateClientConfig } from '<path-to-generated-client>/client/client.gen';

export const createClientConfig: CreateClientConfig = (config) => ({
  ...config,
  baseURL: 'https://example.com',
});
```

## Examples

### Minimal — Fetch only, single spec

```ts
import { defineConfig } from '@hey-api/openapi-ts';

export default defineConfig([
  {
    input: 'https://petstore3.swagger.io/api/v3/openapi.json',
    output: 'src/client/petstore',
    plugins: ['@hey-api/client-fetch', '@hey-api/typescript', '@hey-api/sdk'],
  },
]);
```

### Two specs, Ky + Valibot + TanStack React Query

```ts
import { defineConfig } from '@hey-api/openapi-ts';

export default defineConfig([
  {
    input: 'https://api.example.com/openapi/storefront',
    output: 'src/client/storefront',
    plugins: [
      { name: '@hey-api/client-ky', runtimeConfigPath: './src/client/runtime-config' },
      { name: '@hey-api/typescript', enums: 'javascript' },
      { name: '@hey-api/sdk', responseStyle: 'data', validator: true },
      'valibot',
      { name: '@tanstack/react-query', queryOptions: true, mutationOptions: true },
    ],
  },
  {
    input: 'https://api.example.com/openapi/public',
    output: 'src/client/public',
    plugins: [
      { name: '@hey-api/client-ky', runtimeConfigPath: './src/client/runtime-config' },
      { name: '@hey-api/typescript', enums: 'javascript' },
      { name: '@hey-api/sdk', responseStyle: 'data', validator: true },
      'valibot',
      { name: '@tanstack/react-query', queryOptions: true, mutationOptions: true },
    ],
  },
]);
```

## Capabilities Required

The host agent must be able to:

- Read files in the project root (detect `package.json`, lockfiles, existing config, generated SDK).
- Write text files (`openapi-ts.config.ts`, optional runtime-config stub, edit `package.json` to add a script).
- Ask the user free-form and multiple-choice questions and wait for replies.
- Execute shell commands in the project root with the detected package manager (`pnpm` / `npm` / `yarn` / `bun`).
- (Recommended) Make a lightweight outbound HTTP request to validate Swagger URLs. If unavailable, fall back to user confirmation.

If any required capability is missing, surface the gap and stop.

## Constraints

- **Manual trigger only.** Never auto-invoke from context.
- Do not modify files outside the project root.
- Do not overwrite existing files without explicit user confirmation.
- Do not delete `node_modules/`, lockfiles, or any pre-existing SDK directory.
- Do not switch lockfiles — match the detected package manager.
- Do not use `--force`, `--legacy-peer-deps`, or skip-lockfile flags unless the user opts in after a failure.
- Emit only options the user actually configured or that are strictly required to make the chosen plugin work. Avoid speculative defaults.
- Do not invent plugin IDs or option names. If a user-named plugin is not in the catalog, configure it best-effort and warn that support may be partial.
- Apply the same plugin selection to every spec — never silently diverge.

## Output Format

The skill produces, and only produces:

1. `openapi-ts.config.ts` at the project root, exporting `defineConfig([...])` with one entry per spec.
2. (Optional) A runtime-config stub at the user-confirmed path.
3. A `codegen` (or user-renamed) script in `package.json`.
4. Installed dependencies via the detected package manager.
5. Generated SDK files under each spec's `output` directory (the result of running `openapi-ts`).

A short one-screen summary at the end lists every file written, every package added, and the command to regenerate later.
