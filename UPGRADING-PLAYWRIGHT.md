# Upgrading Playwright — the wrapper package

> Audience: a future Claude (or human) moving `@electrovir/rebrowser-playwright` to a newer
> Playwright version.
>
> **This package contains no stealth code.** All the rebrowser + custom stealth patches live in
> `@electrovir/rebrowser-playwright-core`. The authoritative guide is that repo's
> [`UPGRADING-PLAYWRIGHT.md`](../rebrowser-playwright-core/UPGRADING-PLAYWRIGHT.md) — read it first.
> This file only covers what's specific to the wrapper.

---

## 1. What this package is

`@electrovir/rebrowser-playwright` is the fork's drop-in replacement for the upstream `playwright`
package (the one that bundles the test runner, CLI, transforms, etc. on top of `playwright-core`).

It is a **published build artifact** (no `src/`, no build step, single `init` commit), and it is
**stock Playwright** except for **one** fork-specific change:

```jsonc
// package.json
"dependencies": {
  // the ONLY thing that makes this the stealth fork: playwright-core resolves to our patched core
  "playwright-core": "npm:@electrovir/rebrowser-playwright-core@~<version>"
}
```

Verification that it carries no stealth logic itself:

```bash
grep -rEl "__re__|REBROWSER_PATCHES" lib/   # expect: no matches
```

Everything stealthy (isolated-world execution, the locator/utility-world fixes, the
`alwaysIsolated` default) comes transitively through that aliased `playwright-core`. So upgrading
this package is mostly mechanical; the real work is in the core repo.

---

## 2. Upgrade procedure (wrapper)

Do the **core** upgrade first (see its guide) and publish it, because this package depends on the
new core version existing on the registry.

1. **Start from the target upstream `playwright@<target>` build.** Copy its published contents
   (or however the fork artifact is produced) into this repo, replacing `lib/`, `index.*`, `cli.js`,
   `types/`, etc.

2. **Re-apply the fork's identity in `package.json`** (these are the bits upstream won't have):
   - `name`: `@electrovir/rebrowser-playwright`
   - `version`: match the Playwright version (keep wrapper version == core version == Playwright
     version, e.g. all `1.62.0`).
   - `dependencies.playwright-core`:
     `"npm:@electrovir/rebrowser-playwright-core@~<target>"` ← **the critical line.** Without this
     alias the wrapper would pull stock `playwright-core` and lose all stealth.
   - If publishing under the `@electrovir` scope, ensure `"publishConfig": { "access": "public" }`.

3. **Confirm no stray stealth assumptions.** This package shouldn't reference `__re__*` or
   `REBROWSER_PATCHES_*`. If a grep finds any, upstream changed something and you should consult the
   core guide — but historically the wrapper is clean.

---

## 3. Verification

This package has no test suite of its own — the behavior it exposes is entirely the core's, which is
covered by `@electrovir/rebrowser-playwright-core`'s `npm run test:all` (features + bot-detector).
Run that in the core repo as the real gate.

For the wrapper specifically, the only thing to confirm is that it **resolves to the patched core**
and loads:

```bash
# from a project that has @electrovir/rebrowser-playwright installed:
node -e "console.log(require('playwright-core/package.json').name)"
#   expect: @electrovir/rebrowser-playwright-core   (NOT plain 'playwright-core')

node -e "console.log(typeof require('@electrovir/rebrowser-playwright').chromium)"
#   expect: object
```

A consuming monorepo can encode this as an automated guard that asserts `playwright`,
`@playwright/test`, and `rebrowser-playwright` all resolve to the `@electrovir` fork and that no
stock `playwright-core` remains anywhere in the dependency tree. Run such a guard after re-pinning
consumers.

---

## 4. Publish order & consumers

1. Publish `@electrovir/rebrowser-playwright-core@<version>` first.
2. Bump this wrapper's `playwright-core` alias to `~<version>`, then publish
   `@electrovir/rebrowser-playwright@<version>`.
3. Re-pin and re-test downstream consumers, including any package-resolution guard that enforces the
   fork is used everywhere in the tree.

---

## 5. Checklist

- [ ] Core upgraded, patched, tested (`npm run test:all` green there), and published first.
- [ ] Wrapper `name` = `@electrovir/rebrowser-playwright`, `version` matches Playwright.
- [ ] `dependencies.playwright-core` = `npm:@electrovir/rebrowser-playwright-core@~<version>`.
- [ ] `publishConfig.access: public` if needed.
- [ ] `grep -rE "__re__|REBROWSER_PATCHES" lib/` → no matches (wrapper stays stealth-free).
- [ ] Resolution check: `playwright-core` resolves to the `@electrovir` core.
- [ ] Consumers re-pinned and re-tested (incl. any fork-resolution guard).
