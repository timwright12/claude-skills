# Grove repro recipes

Heavy, runnable reference for the "Runs correctly in a React/TypeScript consumer" and "Tree-shaking/`sideEffects`" pillars in `SKILL.md`. Run these instead of reasoning from memory — see that file's hard rule on verifying claims.

## TypeScript / React consumer check

Run this as **one shell execution** (don't split it across separate tool calls) — the variables it sets don't survive across fresh shells, and splitting it is what caused an earlier version of this recipe to fail with "No such file or directory" when the fallback ran in a different shell than the primary attempt.

**Use a real published tarball, not a symlink into the working directory, when you can.** A symlink into `packages/njwds` exposes the whole working tree — it will resolve `.d.ts` files that exist on disk but aren't actually listed in `package.json`'s `files` array (so the check passes even though the real published package would fail), and it resolves `lit`/other dependencies from this repo's own `node_modules` regardless of whether they're declared in `dependencies` or only `devDependencies`. `npm pack` respects `files` and a real `npm install` of the tarball respects `dependencies`. Fall back to symlinking only if `npm pack`/`npm install` can't run at all (network-restricted or cross-platform sandbox — `grove:build` depends on Rolldown, which ships one prebuilt binary per OS/architecture, so a `node_modules` built on a different platform won't have the right one there either).

```bash
REPO="$(pwd)"  # run from the repo root
SCRATCH="$(mktemp -d)"   # always fresh — no rm -rf needed (and no permission prompt for one)
echo "$SCRATCH" > /tmp/.grove-check-path   # fixed pointer so the sideEffects check can find it from a fresh shell
cd "$SCRATCH"
echo '{ "name": "grove-ts-check", "private": true }' > package.json

# Write the test files up front — both the primary path and the fallback need them.
cat > tsconfig.json <<'EOF'
{ "compilerOptions": { "target": "ES2022", "lib": ["ES2022","DOM"], "module": "ESNext",
  "moduleResolution": "bundler", "jsx": "react-jsx", "strict": true, "noEmit": true,
  "skipLibCheck": true, "types": [] }, "files": ["consumer.tsx"] }
EOF
cat > consumer.tsx <<'EOF'
import { GroveDivider } from "@newjersey/njwds";
const el: GroveDivider = document.createElement("grove-divider");
function App() { return <grove-divider />; }

// Negative controls — without these, a clean compile can mean nothing was
// actually checked (e.g. an ambient-types misconfiguration can make every
// tag resolve to `any`, controls included).
const ok = <div />;                    // must compile — if this errors, React's
                                        // own types aren't loading; stop and fix
                                        // that before trusting any other result.
// @ts-expect-error — misspelled tag must fail, or JSX checking isn't active
const bad = <grove-dividr />;
EOF

TSC="$SCRATCH/node_modules/.bin/tsc"
FILENAME="$(cd "$REPO/packages/njwds" && npm pack --pack-destination "$SCRATCH" | tail -1)"
# npm pack runs the package's own "prepare" script first, which runs grove:build —
# no separate manual build step needed. Don't suppress its stderr: if prepare fails,
# FILENAME ends up holding an unrelated line and everything after fails confusingly.
if [ -f "$SCRATCH/$FILENAME" ] && npm install --no-save "$SCRATCH/$FILENAME" typescript@5.9.3 react@19 @types/react@19; then
  echo "primary path: real tarball install"
  echo tarball > "$SCRATCH/.install-mode"
else
  echo "falling back to symlinks (network-restricted or cross-platform sandbox)"
  echo symlink > "$SCRATCH/.install-mode"
  # Confirm this is actually meaningful before trusting it: nothing rebuilds dist/ in this path.
  # Compare against the newest individual FILE under dist/web-components, not the directory
  # itself — a directory's own mtime doesn't update when an existing file inside it is
  # overwritten in place (confirmed: rebuilding grove-divider.js left the directory's own
  # mtime unchanged), so comparing against the directory would report "stale" every time,
  # even immediately after a correct rebuild.
  NEWEST_DIST_FILE="$(find "$REPO/packages/njwds/dist/web-components" -type f -print 2>/dev/null | xargs ls -t 2>/dev/null | head -1)"
  if [ -z "$NEWEST_DIST_FILE" ]; then
    echo "WARNING: dist/web-components/ missing or empty — nothing to test; build on a machine where grove:build works"
  elif [ -n "$(find "$REPO/packages/njwds/src" -type f -newer "$NEWEST_DIST_FILE" -print -quit)" ]; then
    echo "WARNING: src/ has files newer than dist/ — results below may not reflect the current diff"
  fi
  mkdir -p "$SCRATCH/node_modules/@newjersey" "$SCRATCH/node_modules/@types"
  ln -sfn "$REPO/packages/njwds" "$SCRATCH/node_modules/@newjersey/njwds"
  ln -sfn "$REPO/node_modules/react" "$SCRATCH/node_modules/react"
  ln -sfn "$REPO/node_modules/@types/react" "$SCRATCH/node_modules/@types/react"
  # Don't symlink typescript itself — that doesn't create node_modules/.bin/tsc.
  TSC="$REPO/node_modules/.bin/tsc"
  echo "NOTE: symlink fallback used — this skips the files-array and dependencies-declaration checks"
fi

"$TSC" -p tsconfig.json
```

Repeat with `strict: false` for the non-strict fallback, and swap `moduleResolution`/`module` to `node10`/`commonjs` or `nodenext`/`nodenext` to check other consumer configs. For the `jsx: preserve` variant specifically, **drop `"types": []` from tsconfig.json** (or add `import React from "react";` to `consumer.tsx`) before trusting the `ok` control — with `types: []` under `preserve`, React's own ambient JSX namespace never loads either, so `ok` fails for a reason that has nothing to do with Grove (confirmed: `TS7026: JSX element implicitly has type 'any' because no interface 'JSX.IntrinsicElements' exists`, on the plain `<div />` control itself). Record which TypeScript version and date you ran this with in the Evidence column — these are checked outputs to compare against, not settled facts that stay true across every future TypeScript release. Checked with TypeScript 5.9.3 on 2026-10-26, on a Mac with network access: primary path produces `TS7016` for the missing `.d.ts` and `TS2339: Property 'grove-divider' does not exist on type 'JSX.IntrinsicElements'` for the missing JSX augmentation, with both negative controls behaving correctly.

## Tree-shaking / `sideEffects` check

Test the real packed tarball, not throwaway files inside this repo — a throwaway test here will pick up the *monorepo root's* `package.json` (which has no `sideEffects` field) instead of the package being tested, which is a plausible reason this check has given confusing answers before. This needs `vite@8` from the registry plus a working Rolldown binary for your platform; if the symlink fallback above was needed, mark this row `unverified` rather than spending time trying to work around it — there's no symlink equivalent for this one.

Test the **real, unmodified package first**, before editing anything — editing `node_modules/@newjersey/njwds/package.json` for the control below is destructive to that installed copy; if you need to go back to the real value afterward, rerun the TypeScript recipe above (it creates a fresh `$SCRATCH`) rather than trying to undo the edit. This block is safe to run as its own shell call — it re-derives `$SCRATCH` from the pointer file and checks which install path the TypeScript recipe used:

```bash
SCRATCH="$(cat /tmp/.grove-check-path)"   # written by the TypeScript recipe above
MODE="$(cat "$SCRATCH/.install-mode" 2>/dev/null)"
if [ "$MODE" = symlink ]; then
  echo "sideEffects row: unverified — TypeScript recipe used the symlink fallback; this check needs the real tarball install"
  exit 0
elif [ "$MODE" != tarball ]; then
  echo "sideEffects row: not run — run the TypeScript recipe above first (it creates $SCRATCH)"
  exit 0
fi
cd "$SCRATCH"
# Re-specify the tarball here, not just vite — a bare `npm install vite@8` on its own
# prunes the njwds install from the step above as "extraneous" (confirmed: it goes
# from a real package to an empty directory). The tarball file itself isn't inside
# node_modules so it survives that prune; reinstall from it in the same command.
TARBALL="$(ls "$SCRATCH"/newjersey-njwds-*.tgz)"
npm install --no-save "$TARBALL" vite@8
cat > entry.js <<'EOF'
import { GroveDivider } from "@newjersey/njwds";   // also try a bare `import "@newjersey/njwds";`
console.log("named import, never referenced below");
EOF
cat > vite.config.js <<'EOF'
import { defineConfig } from "vite";
export default defineConfig({ build: { outDir: "dist-check", minify: false,
  rollupOptions: { input: "entry.js", output: { entryFileNames: "out.js" } } } });
EOF
node_modules/.bin/vite build
grep -o "grove-divider\|customElements" dist-check/out.js | sort -u   # blank = fully dropped
```

Only after recording that result, run the **control**: edit `node_modules/@newjersey/njwds/package.json` to set `"sideEffects": true` (the whole package, unconditionally), rebuild into a different `outDir`, and confirm `grove-divider`/`customElements` now *do* show up. If you also want a `"sideEffects": false` comparison, that's a second edit on top — track which edit is currently in place, since they overwrite each other on the same installed copy. The real value should match one of the two forced controls; if it matches `false` when the intent was to protect the registration, or doesn't match either cleanly, that's a finding about the specific PR under review, not something to resolve inside this recipe file — record the dated result in the review itself, not here, since the package's `sideEffects` value is expected to change over time and a result recorded in a reference file would go stale and mislead future reviews.
