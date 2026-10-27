# Webpack HMR topology reproductions

Eight small, independent repositories for discussion with webpack maintainers. They isolate **five families of behavior**, not eight claims of unrelated bugs. These were reduced from investigation of [Rspack PR #15955](https://github.com/web-infra-dev/rspack/pull/15955), then reproduced with **unmodified webpack**.

## Verified baseline

- Date: 2026-10-28.
- webpack: **5.112.2**, npm package with lockfiles in every repository.
- Node.js: **22.23.4**. JS cases run real emitted Node/Worker runtimes through webpack's public Node API.
- CSS: **mini-css-extract-plugin 2.10.2**, css-loader 7.1.2, Playwright 1.64.1, Chrome 156.
- No runtime patching, React, Rsbuild guard, custom reload suppression, lazy-compilation throttle, or app code is required.
- Each primary test asserts the desired result and currently exits **1**. Each success control exits **0**. See [validation.json](validation.json).

The installed HMR compiler and JavaScript HMR runtime match main commit [bd8eddff945733f65ada91ad441ce8b613f1544a](https://github.com/webpack/webpack/commit/bd8eddff945733f65ada91ad441ce8b613f1544a) byte-for-byte. This comparison covers these two files, not an execution of every target from main:

| File | Git blob SHA |
| --- | --- |
| lib/hmr/HotModuleReplacementPlugin.js | 83972860b3caddb17666c76546dca8fe25e33003 |
| lib/hmr/JavascriptHotModuleReplacement.runtime.js | 9d779240688bc8bf3c24b91c9c23232992ff454f |

## Repositories

| # | Scenario | Observed result | Successful control |
| --- | --- | --- | --- |
| 1 | [New initial JS dependency](https://github.com/JSerFeng/webpack-hmr-new-initial-chunk-repro) | `Cannot find module './dep.js'`; HMR status `fail` | Disable splitChunks |
| 2 | [Already loaded async module gains unchanged dependency](https://github.com/JSerFeng/webpack-hmr-loaded-async-dependency-repro) | Same missing factory without calling import again | Disable splitChunks |
| 3 | [Import caller changes only because chunk topology changes](https://github.com/JSerFeng/webpack-hmr-chunk-topology-propagation-repro) | HMR aborts at unchanged `bridge.js` | Disable splitChunks |
| 4 | [Single-runtime module ownership is lost](https://github.com/JSerFeng/webpack-hmr-single-runtime-ownership-repro) | HMR is idle with `updated: []`; later import returns 2 instead of 3 | Load the new owning chunk before editing dep |
| 5 | [Retained cached module gains a new synchronous dependency](https://github.com/JSerFeng/webpack-hmr-retained-module-dependency-repro) | Edited dep arrives, but its self-accept cannot require n | Load the other branch first |
| 6 | [Intermediate HMR imports an overwritten ordinary chunk](https://github.com/JSerFeng/webpack-hmr-unversioned-chunk-race-repro) | V1 accept executes V2 page.js using V1's chunk list; dep is missing | Contenthash filenames with old files retained |
| 7 | [Running Worker gains an initial dependency](https://github.com/JSerFeng/webpack-hmr-worker-initial-chunk-repro) | Worker HMR fails on missing dep | Disable splitChunks |
| 8 | [Extracted stylesheet migrates to a new shared chunk](https://github.com/JSerFeng/webpack-hmr-extracted-css-migration-repro) | Hot page stays red while a fresh page is blue | Disable CSS splitting |

For the seven Node cases:

```sh
npm ci
npm test                 # expected to fail on the pinned affected version
npm run test:control     # expected to pass
```

The CSS repo additionally needs `npx playwright install chromium`, or `BROWSER_CHANNEL=chrome` when Chrome is already installed.

## Five questions for maintainers

1. **Dependency completeness** (1, 2, 5, 7): what guarantees that a changed, cached module has its new synchronous dependencies before self-accept? New initial chunks, already-existing but unloaded shared chunks, and async Worker entries expose the same missing guarantee in different ways.
2. **Topology-only invalidation** (3): should changing the chunk list encoded in a dynamic-import caller propagate HMR even when that caller's source is unchanged? The current abort is consistent with changed generated factory code; whether to isolate the loading plan is a design question.
3. **Module ownership** (4): how should installed factories remain eligible for updates after moving to an unloaded chunk while the old chunk stays installed? This repro deliberately does not remove the old chunk from its runtime.
4. **Version coherence** (6): what is the intended behavior when an intermediate update executes an import against a newer unversioned ordinary asset? A self-consistent hot-update alone does not make the subsequently loaded asset self-consistent.
5. **CSS resource membership** (8): how should HMR attach a newly required stylesheet when the old stylesheet name is no longer the current owner? This is a webpack/mini-css-extract-plugin integration report, not an attribution to webpack core alone.

## Why #21131 does not cover scenario 4

[webpack #21131](https://github.com/webpack/webpack/pull/21131) handles an old chunk leaving a runtime, using manifest `f` to force-load a surviving owner. We verified that branch as a successful contrast.

In scenario 4 there is only one runtime. `shared` stays in it because it still contains `keeper`. Only dep moves to `moved`. The migration manifest has `r: []`, and later changes to dep are silently missed. The relevant boundary is chunk/runtime membership, not just the count of runtimes.

[webpack #19494](https://github.com/webpack/webpack/pull/19494) also does not make every new dependency available: preserving an updated module from an old chunk and supplying a previously unavailable dependency are separate requirements.

## What is not claimed

- These Node cases demonstrate errors/aborts/stale values, **not measured browser reload counts**. A dev-server client may turn an error into reload; it is deliberately absent here.
- No claim is made that Turbopack or Rspack currently passes all these cases.
- The CSS case retains old ordinary assets, so it demonstrates stale CSS rather than a 404. A fresh page using the new entrypoint files is verified blue.
- The versioned-file control for scenario 6 is not a recommendation to add historical snapshot retention to Rspack.
- Historical implementation defects and design tradeoffs in the Rspack PR are listed separately in [PR-ONLY-FINDINGS.md](PR-ONLY-FINDINGS.md). They are not attributed to stock webpack.

## 中文索引

这 8 个 demo 归并为 5 类问题：新增同步依赖的完整性、chunk 结构变化引起的 HMR 传播、module ownership 丢失、跨编译版本加载混用、CSS 资源归属变化。

其中第 4 项直接验证了：**同一个 runtime 内，旧 chunk 没有删除，module 搬迁后仍然会漏掉更新**。第 5 项独立验证了：即使 changed factory 已经收到，缓存 module 新增的同步依赖仍可能缺失。两者不能混为一个下载选择问题。

每个仓库包含起始源码、分步编辑、自动复现、成功对照和实际 JSON 结果。仅用于准备维护者讨论，尚未向 webpack 提交 issue 或评论。
