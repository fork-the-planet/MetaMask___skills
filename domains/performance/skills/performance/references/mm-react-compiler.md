---
title: React Compiler (MetaMask)
impact: HIGH
tags: react-compiler, memoization, babel, incremental-adoption
---

# Skill: React Compiler in MetaMask

React Compiler auto-memoizes components, callbacks, and computed values at build time — removing most of the need for manual `React.memo`/`useMemo`/`useCallback`. MetaMask adopts it **incrementally**, path by path, via `babel.config.js`.

## Current state (verified)

- `babel-plugin-react-compiler`, `react-compiler-runtime`, and `eslint-plugin-react-compiler` are installed.
- ESLint: `react-compiler/react-compiler: 'warn'` is enabled.
- `babel.config.js`: `target: '18'`, plugin runs **first**, and only these paths are opted in:
  ```js
  // babel.config.js → plugins → ['react-compiler', { target: '18', sources }]
  const pathsToInclude = [
    'app/components/Nav',
    'app/components/UI/DeepLinkModal',
  ];
  return pathsToInclude.some((path) => filename.includes(path));
  ```
- `react-native-reanimated/plugin` must remain **last** in the plugin list (required for `'worklet'`).

## Opt a new feature in

1. **Health-check the code first:**
   ```bash
   npx react-compiler-healthcheck@latest
   ```
   It reports Rules-of-React violations that would make the compiler skip a component (silently — safe, but you miss the optimization).
2. **Add the path** to `pathsToInclude` in `babel.config.js` (a directory prefix or a specific file path; `filename.includes` matches substrings).
3. **Clear Metro's cache** — it caches compiled output aggressively:
   ```bash
   yarn watch:clean
   ```
4. **Verify:** in React DevTools, optimized components show a **`Memo ✨`** badge. You can also confirm `'use no memo'` isn't silently opting a component out.

## What it does / doesn't do

- **Does:** auto-memoize components and hook values that follow the Rules of React; reduce cascading re-renders.
- **Doesn't:** fix bad patterns. It optimizes *correct* code. A broken selector still returns new refs — the compiler can't save you. Fix [mm-selector-memoization.md](mm-selector-memoization.md) first.
- **Class components** are not optimized.

## Interaction with manual memoization

On opted-in paths you can gradually drop hand-written `useMemo`/`useCallback`/`React.memo` once the compiler is verified working — but do it deliberately and re-measure. Off opted-in paths, manual memoization still matters.

## What breaks compilation (it will skip the component)

- Mutating props or state during render.
- Side effects during render (e.g. incrementing a module variable).
- Other Rules-of-React violations flagged by the ESLint plugin / healthcheck.

Fix the ESLint `react-compiler` warnings on a path before/after opting it in.

## Don't

- Don't hardcode the current path list as if it's permanent — it grows over time; read `babel.config.js`.
- Don't use `target: '19'` blindly — the repo targets `'18'`; match it.
- Don't reorder the reanimated plugin away from last.

## Related

- [js-react-compiler.md](js-react-compiler.md) — upstream reference on how the compiler transforms code
- [mm-selector-memoization.md](mm-selector-memoization.md) — fix data-layer re-renders the compiler can't
