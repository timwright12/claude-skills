---
name: reviewing-grove-contributions
description: Use when reviewing a pull request, branch, or code change in this repository (Grove / @newjersey/njwds design system) that touches web components (src/web-components), package distribution (package.json, build-scripts/, vite.config.grove.ts), or anything published to npm/CDN consumers. Not needed for Storybook-story-only, Sass/token-only, or snapshot-only changes.
---

# Reviewing Grove Contributions

## Overview

Grove ships as an installed npm package consumed by other teams' React/TypeScript apps, and eventually via `<script>`-tag/CDN delivery (usually a separate pipeline). Every review here checks ordinary code quality AND whether the change holds up once it leaves this repo and runs inside someone else's build. Web-component and package-architecture decisions are hard to walk back once a second or third component copies the pattern — review the first instance of a pattern harder than a routine change.

**Hard rule, stated once up front and enforced below: verify, don't assert.** Claims about bundling/tree-shaking, SSR behavior, and TypeScript module resolution sound right from general web-platform knowledge and are frequently wrong in practice — this has already happened in review comments on this repo, more than once. The urge to write "`sideEffects` is correctly configured," "tree-shaking is handled properly," or "Shadow DOM isolates this correctly" without having run anything is itself the signal to stop and go verify first.

## General code quality (always check)

- Tests cover new/changed logic, not just a Storybook story with no `play()` — a story with no `play()` only proves "doesn't throw" (see CLAUDE.md).
- Run `npm run test:coverage` and check the report table — confirm changed files are actually attributed coverage, not silently absent from it.
- No dead code shipped in the npm tarball: anything new that lands in `package.json`'s `files` array ships even if nothing imports it.
- Unrelated changes (CI workflow triggers, dependency bumps) aren't bundled into a feature PR without being called out.
- Playwright accessibility/visual specs: confirm they were actually executed, not just present — either run them yourself (`npm run storybook` + `npm run test:accessibility`/`test:visual`) or find the CI job log that ran them for this PR. A spec file existing in the tree is not evidence it passed.

## Web component / package architecture

This repo's own `CLAUDE.md` currently states the project doesn't use `LitElement`/`@customElement` or register custom elements/Shadow DOM "anywhere in `src/`" — that was true before the first web component landed and may not have been updated since. If the diff being reviewed adds `customElements`/`@customElement`/`LitElement`, the pillars below apply regardless of what `CLAUDE.md` currently says; don't mark these rows `n/a` on the strength of that line alone.

**Scale — will this still make sense at component #5, not just #1?**
- Is a new component auto-discovered by the build (directory convention), or does adding one require hand-editing shared config?
- How is a component exposed to consumers — a shared barrel re-exporting everything, or per-component entry points? If `sideEffects` covers the whole tree, name the actual risk, both directions: too narrow/wrong and a bundler may tree-shake out the `customElements.define()` call entirely (the component silently never registers, no error, nothing renders); too broad and every consumer may pay for every component regardless of which one they imported. Don't assert either outcome from the field's presence alone — see repro-recipes.md.
- Does `customElements.define()` — directly, or hidden inside `@customElement(...)` — guard against double registration? `@customElement(...)` has **no built-in guard**; the guard has to be added either by not using the decorator or by wrapping it. A bare `customElements.get(tag)` check that no-ops silently is itself a hazard: if two different versions of the package load, the first one registered silently wins and the second's changes never take effect. Prefer a guard that at least warns when the already-registered class differs from the one being defined.

**Performance**
- Is the component's runtime dependency (e.g. `lit`) externalized from the bundle instead of inlined, so it isn't duplicated per component? If so, is it declared in `dependencies` or `peerDependencies` (not `devDependencies`) — consumers need it present at runtime since it isn't bundled.
- Externalizing that dependency trades off against `<script>`-tag/CDN delivery below: an externalized bare `import "lit"` only resolves inside a bundler or with an import map; a consumer loading the component via a plain `<script type="module">` tag has neither. These two checks pull in opposite directions — note explicitly which one this change prioritizes.
- If the component uses Shadow DOM, is there any supported way for a consumer to theme it (CSS custom properties, which cross the shadow boundary, or `::part()`)? Decide the theming contract on the first component — retrofitting it later across many components is expensive.

**Accessibility**
- Native semantics survive Shadow DOM for non-interactive elements (`<hr>`, etc.); anything interactive needs real keyboard/ARIA verification, not an assumption that Shadow DOM "just works."
- `:host` has an explicit `display` set — custom elements default to `display: inline` and break layout without it. If `:host { display: block }` (or similar) is unconditional, also check for `:host([hidden]) { display: none }` — an unconditional `:host` display rule, being part of the component's own shadow stylesheet, overrides the native `hidden` attribute's default, so `<grove-divider hidden>` silently stops hiding anything.
- `aria-labelledby`/`aria-describedby` cannot reference an element on the other side of a shadow boundary — the single most common Shadow DOM accessibility bug. Flag any ARIA attribute that points at an ID, and check whether the referenced element is in the same root. Form-like components need `ElementInternals`/form-association, not just ARIA, to participate in a surrounding `<form>`.

**Semver / breaking-change surface**
- For a published package, renaming a custom-element tag, a CSS custom property, or a `::part()` name is a breaking change for every existing consumer, same as renaming an export. Adding an `exports` map is also breaking if it removes a deep-import path (e.g. `@newjersey/njwds/dist/...`) that consumers already rely on. Call this out explicitly rather than reviewing it like an internal rename.

**Runs correctly in a React/TypeScript consumer** — verify with the recipes in repro-recipes.md; don't reason about it from memory:
- Does the build emit a real `.d.ts` (`vite-plugin-dts`, `tsc --emitDeclarationOnly`, etc.) and does `package.json` point at it via `"types"`/`exports`? Without one, the failure mode is config-dependent: under `strict`/`noImplicitAny`, expect `TS7016: Could not find a declaration file for module ... implicitly has an 'any' type` (a real build-breaking error). Without `strict`, it silently compiles with `any` and no error at all. Checked with TypeScript 5.9.3 on 2026-10-26 across `bundler`, `node10`, and `nodenext` resolution — all three gave `TS7016`, not `TS2307`, for this case. `TS2307: Cannot find module` is a real, separate error class (genuine resolution failure — wrong path, a typo, an `exports` condition that truly excludes the import), but it's rarer than it sounds and shouldn't be assumed as the default failure mode for a plain missing-`.d.ts` case; don't state which error a given change will produce without running the recipes in repro-recipes.md against that change's actual shape.
- Even once a `.d.ts` ships: `HTMLElementTagNameMap` augmentation covers `document.createElement`/`querySelector` but **not** JSX. Use `declare module "react" { namespace JSX { interface IntrinsicElements {...} } }` — checked with TypeScript 5.9.3 on 2026-10-26 and it compiles clean against both `@types/react@18.3.31` and `@types/react@19.3.0`, so prefer it over the older `declare global { namespace JSX {...} }` form, which still works on 18.3.31 but fails outright on 19.x with the same error as no augmentation at all (the global `JSX` namespace was removed). Also check `types/declarations.d.ts` in this repo — it already hand-rolls this exact augmentation for `feedback-widget`, a sibling dependency with the same gap, so this isn't hypothetical for this codebase. But that file uses the old `declare global { namespace JSX {...} }` form, which — per the line above — no longer works against the repo's installed `@types/react@19.3.0`. Treat it as a live example of the bug, not a pattern to copy; it's very likely just unexercised (nothing in `src/` currently writes `<feedback-widget>` as a JSX expression, since this repo's components are `lit-html` string templates, not `.tsx`), which is why it hasn't visibly broken anything yet.
- Does importing the component crash under SSR (Next.js, etc.)? Lit's `node` package-export condition shims a no-op `customElements` registry so it won't throw — but that shim only applies to whichever copy of `lit`/`@lit/reactive-element` actually gets resolved at the consumer's runtime. If this package ever inlines Lit instead of externalizing it (see Performance above), the browser-condition Lit code gets baked in permanently at *this* repo's build time (a browser-target Vite build), and that inlined copy will throw `ReferenceError: HTMLElement is not defined` under SSR regardless of what the consumer's own `lit` would have done. These two checks interact — don't review them independently.

**Runs correctly via `<script>`-tag/CDN delivery** — CDN infrastructure is a separate pipeline, but this check is not optional just because it is:
- Is the published build ESM-only? A plain `<script src=...>` include with no bundler and no `type="module"` can't consume a bare ESM export. If the runtime dependency is externalized (see Performance above), a `<script type="module">` include still can't resolve a bare `import "lit"` without an import map. Confirm whether CDN delivery expects a separate UMD/IIFE bundle with dependencies inlined, and whether this change produced one.
- The same double-registration risk above applies across independently-loaded `<script>` includes on one page.

## When to actually run a repro

Run the recipes in `repro-recipes.md` (same directory as this file) when the PR touches `package.json`, build config (`vite.config.grove.ts`, `build-scripts/`), or a component's public surface (new/changed props, attributes, events, or exported class). A docs-only or Storybook-story-only change doesn't need it — mark those rows `n/a` with the reason ("no public-surface change") rather than running the full recipe or silently skipping the row. That file has a runnable TypeScript/React consumer check (with a network-restricted-sandbox fallback) and a tree-shaking/`sideEffects` check with a real control — read it before claiming either pillar passed or failed.

## Required output shape

Report one line per pillar below, with an **Evidence** column containing both the command you ran *and* the relevant output line or error code — a command alone doesn't show what happened. A pillar silently missing from the review is a bigger failure than a pillar marked unchecked. Verdict must be one of: `pass`, `fail`, `unverified`, `n/a`, `not checked` — no other wording. `pass` requires evidence; a `pass` with no evidence is itself a defect in the review. If Playwright can't actually launch a browser in your environment, the honest verdict for that row is `unverified` with a pointer to the CI job log — "couldn't launch" is not evidence of a pass.

| Pillar | Verdict | Evidence |
|---|---|---|
| General code quality | | |
| Scale (auto-discovery, entry-point shape, double-registration guard) | | |
| Performance (externalized deps, theming contract, dependency declared correctly) | | |
| Accessibility (native semantics, `:host` display + `hidden`, ARIA across shadow boundary) | | |
| Semver / breaking-change surface | | |
| React/TypeScript — `.d.ts` emitted + `"types"`/`exports` wired up | | |
| React/TypeScript — `JSX.IntrinsicElements` augmentation (separate row — do not merge into the `.d.ts` row above) | | |
| React/TypeScript — SSR/Next.js import behavior | | |
| `<script>`-tag/CDN delivery (module format, double-registration) | | |
