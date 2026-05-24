## Problem

The static `spa-dist` build hangs the browser as soon as you focus a text input on the login page.

Root cause: the React chunk in the static build is **empty (1 byte)**:

```text
1        spa-dist/assets/react-l0sNRNKZ.js   ← empty
136925   spa-dist/assets/router-yyNn1TUU.js
224411   spa-dist/assets/index-DnEMtUT0.js
```

`spa-build/vite.config.ts` declares a `manualChunks` group that pulls `react`, `react-dom`, and `react/jsx-runtime` into their own chunk. Because the SPA entry (`spa-build/main.tsx`) does not import these directly and Rollup's tree-shaking sees no live exports referenced from that chunk, React ends up duplicated inside the other chunks that use it (router, index). Two React copies share no internal dispatcher state, so the first real event handler (focus / input) triggers a hooks invariant that re-renders forever and freezes the tab.

## Fix

Remove the broken `react` manual chunk group so React is bundled once into the entry chunk, then rebuild the static SPA and repackage the zip.

### Steps

1. Edit `spa-build/vite.config.ts`: delete the `react: ["react", "react-dom", "react/jsx-runtime"]` line from `build.rollupOptions.output.manualChunks`. Keep the other groups (router, supabase, charts, radix).
2. Rebuild the static SPA:
   ```bash
   bunx vite build --config spa-build/vite.config.ts
   ```
3. Verify the new build does not contain an empty `assets/react-*.js` and that React is bundled into the main entry (size > 100 KB).
4. Repackage `/mnt/documents/bfg-planner-static.zip` from `spa-dist/` (keeping `.htaccess`, `index.php`, `README-DEPLOY.txt`).
5. Deliver the refreshed zip via `<presentation-artifact>` so you can re-upload it to your host.

No source/runtime code changes are needed — only the bundler config and a fresh build.
