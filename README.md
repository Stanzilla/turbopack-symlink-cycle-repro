# Turbopack symlink-cycle repro

`next build` (Turbopack) crashes with `Too many levels of symbolic links (os error 62/40)` when:

1. `outputFileTracingIncludes` has at least one entry (any pattern), and
2. the pnpm workspace contains a package that is physically nested inside another workspace package and declares that ancestor package as a dependency or devDependency.

This layout is legal in pnpm workspaces. pnpm plants a `node_modules/umbrella -> ../..` symlink inside the nested package, which makes the directory tree self-embedding. Turbopack's include-glob walk (`get_glob_includes` in `crates/next-api/src/nft.rs` → `read_glob` in `turbopack/crates/turbo-tasks-fs/src/read_glob.rs`) walks the whole tracing root and follows the cycle until the OS returns `ELOOP`.

The cycle detection in `resolve_symlink_safely()` only catches a symlink whose target is an ancestor of the symlink's own virtual path. The walk here enters the cycle through a different symlink (`app/node_modules/shared` or the hoisted `.pnpm/node_modules/shared`), so no single link points at a virtual ancestor and the check never fires.

`outputFileTracingExcludes` cannot prevent this: excludes filter traced assets in `nft_json.rs`, they are not applied to the include-glob directory walk.

## Layout

```
pnpm-workspace.yaml         packages: [app, packages/*, packages/umbrella/*]
app/                        Next.js app, output: standalone, depends on `shared`
  next.config.mjs           outputFileTracingRoot: workspace root
                            outputFileTracingIncludes: { '/*': ['included-asset.txt'] }
packages/umbrella/          package `umbrella`
packages/umbrella/shared/   package `shared`, devDependencies: { umbrella: workspace:* }  <-- ancestor dep
packages/umbrella/leagues/  package `leagues`, dependencies: { shared: workspace:* }
```

After `pnpm install`, `packages/umbrella/shared/node_modules/umbrella` is a symlink to `../..`, so the tree contains the infinite path `.../shared/node_modules/umbrella/leagues/node_modules/shared/node_modules/umbrella/...`.

## Reproduce

```bash
pnpm install
pnpm --filter app exec next build
```

## Result

```
Error [TurbopackInternalError]: Failed to write app endpoint /page

Caused by:
- reading dir ".../app/node_modules/shared/node_modules/umbrella/leagues/node_modules/shared/node_modules/umbrella/leagues/.../node_modules/shared"
- Too many levels of symbolic links (os error 62)

Debug info:
- Execution of *NftJsonAsset::new failed
- Execution of trace_endpoint failed
- Execution of read_glob failed
- Execution of read_glob_inner failed (x many)
- Execution of <DiskFileSystem as FileSystem>::raw_read_dir failed
```

Reproduced with next 16.3.0, 16.3.1, and 16.3.1-canary.22 on macOS (arm64) with pnpm 11.22.0 and Node 24.19.0.

## Real-world trigger

`@sentry/nextjs` >= 10.70 unconditionally merges entries into `outputFileTracingIncludes` (getsentry/sentry-javascript#23055), which turns this from an opt-in config into something any pnpm monorepo with a nested-workspace-package layout hits on upgrade.
