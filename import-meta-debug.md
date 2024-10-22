# Debug Assertions in JavaScript

**TL;DR:** Wouldn't it be nice if all tools supported a consistent, simple, and relatively safe way to mark development-time only code?

And why not make it [`import.meta.DEBUG`](#importmetaDEBUG)?

## The Problem

### Debug Assertions?

```ts
DCHECK(typeof arg === 'string',
  `This function used to accept strings but no more!`)
```

The term is likely overly narrow but the pattern may be familiar:

* You want to provide high quality, actionable error messages to developers using your library.
* Relative to the unoptimized code used during development, the size and performance cost of these checks is acceptable.
* But doing the necessary checks and including verbose strings in an optimized bundle would degrade user experience.

The code in question may just be simple `typeof` checks on arguments. But some of these checks may require creating and keeping track of additional data structures.

The question is: How do you express these checks in JavaScript?

### Desired Behavior

#### Default: Disabled But Valid

If developers don't see these checks in local development, it's not the end of the world. They can adjust their setup and everything's fine.

End-users don't have that luxury. If these debug assertions reach them, they are stuck with a slower than necessary experience with no real upside. There's no adjusting build options when visting your website (hopefully).

At the same time, the code shouldn't be invalid just because it didn't run through a specific kind of build tool. As much as possible, the code should run and be performant in any standard JS environment.

#### Easy to Eliminate

In practice, we'd want to be able to completely strip this code from production bundles. End users shouldn't have to pay a size price for something they don't use.

Different minifiers have different thresholds for what's "easy to eliminate". The simpler the code is, including in the presence of down-leveling, the more likely that all tools can reliably strip it.

Down-leveling complicates things because defensive code can be transformed in a way that makes it harder to eliminate, e.g.:

```ts
// Before down-leveling:
globalThis.process?.env?.NODE_ENV
// After down-leveling:
globalThis.process != null
    ? (globalThis.process.env != null
       ? globalThis.process.env.NODE_ENV
       : undefined)
    : undefined
```

If the bundler is configured to just replace `process.env.NODE_ENV` as is often recommended, the checks for `process` and `process.env` may end up being retained. Which can completely break the guard.

#### (Optional) Pre-Publish Replacement

It's neat when these replacement can be done at the application level but it's also slightly wasteful. Especially if it's code in a library that's widely used.

Having a well documented path to "render out" files with and without debug code would allow libraries to publish already processed files if they prefer.

#### (Optional) Scoped

Today most solutions require that the "debug-ness" of code is set globally. Being able to opt into additional checks from just one or a few packages may be useful in specific situations. E.g. when the runtime performance of code under test is important but helpful errors from other code would be nice to have at the same time.

## Existing Solutions

*Or: It's not like nobody tried this before.*

Let's dig into what's offered today. Under each option includes a note on where it has:

* "Support": It fully works without further configuration, switching between on and off automatically.
* "Gracefull fallback": It will gracefully accept the code without explicit configuration but it won't have special semantics.

### `process.env.NODE_ENV`

**Support:** Parcel, Rspack, Rsbuild, Vite, Webpack.<br/>
**Graceful fallback:** Node.js, deno, bun.

Let's start with the elephant in the room. `NODE_ENV`. It's the hammer everybody reached for because it existed and seemed fine, at first glance.

But it's not actually fine.

1. It's easy to use wrong. Checking `NODE_ENV === 'production'` and then forgetting to set `NODE_ENV` was the root cause of many production issues. Both in Node.js but also in (early) bundlers.
2. It throws in a "vanilla" JS environment since `process` is not a global everywhere.
3. Where it exists, `process` references a global and cannot cleanly be scoped to just a few files.
4. If it's preserved, checking a property on `process.env` is anything but free in Node.js.
5. There's no standard for what exactly a "`NODE_ENV`" is. If you want to pre-process your library, you'd have to know all the random values that people might put into `NODE_ENV` and which would or wouldn't be considered "debug mode on".

It _is_ generaly easy to eliminate by a minifier. That's less true though if you try to code defensively against the absence of `process`.

### Package Exports `development`

**Support:** Parcel, Rspack, Rsbuild, Vite, Webpack.<br/>
**Graceful fallback:** esbuild, `@rollup/plugin-node-resolve`, Node.js, deno, bun.

```js
  "exports": {
    "development": "./index-debug.js",
    "default": "./index.js"
  }
```

It's great although [I might be biased][biased]. But for this particular use case, there's currently signficant overhead to using it. The options are roughly:

1. Manually keep two copies of the file, one with and one without debug code. Try your best to keep them in sync.
2. Move the debug code to a separate file. Pray that the minifier figures it out despite the added indirection.
3. Invent your own inline debug marker and generate debug/non-debug variants based on it.

Option 3 would be great if it didn't require that each package maintainer reinvents the wheel for their project. It also doesn't help application developers if they want to write code that's not tightly coupled to a specific build toolchain.

*Aside: Never use the production condition. It was a mistake to have a production condition, see `NODE_ENV`.*

[biased]: https://github.com/jkrems/proposal-pkg-exports?tab=readme-ov-file#configuring-conditions

### `import.meta.env.DEV`

**Support:** Vite, Rsbuild.<br/>
**Graceful fallback:** bun[1].

One of the spiritual successors of `NODE_ENV`. It does have two advantages:

1. `.DEV` is a simple boolean, removing some boilerplate.
2. By attaching to `import.meta`, the API generally allows for non-global scoping of the debug flag.

But there's also some akward aspects.

1. `.PROD` and `.MODE` muddy the waters and continue the trend of `NODE_ENV`: It's easy to get this wrong and accidentally default to shipping debug code to production.
2. Nesting under `import.meta.env` means that the easiest pattern to detect (`import.meta.env.DEV`) throws in a vanilla JS environment.
3. It sits in a bucket called `env` that otherwise contains "environment variables" but it's not one itself and isn't a string like an environment variable would be.

[1] Support in `bun` is tricky because it has `import.meta.env` but `.DEV` is just the value of the environment variable named "DEV".

### `ngDevMode` (Angular)

**Support:** *Only via plugins or custom configuration.*<br/>
**Graceful fallback:** Everywhere.

An implementation detail of Angular, so don't tell anybody. In Angular apps, this is used to both enable debug code and also keep some private debug state. The canonical usage looks like this:

```ts
typeof ngDevMode === 'undefined' || ngDevMode
```

Name aside, it's a global. Nice, we can access `typeof ngDevMode` all day long in a vanilla JS environment! It defaults to off as well which is exactly what we'd want.

Now for the bad news: The `typeof x || x` check is very verbose. And, maybe more importantly, a global with a _generic_ name would have a high propability of conflicting with existing code. It doesn't generalize well.

### `goog.DEBUG` (Closure Compiler)

**Support:** Closure Compiler<br/>
**Graceful fallback:** Everywhere.

Finally, an honorable mention for Closure. It actually predates `NODE_ENV`, for what that's worth. For our purposes, `goog.DEBUG` isn't much different from `ngDevMode`: It has a lower collision propability because it's namespaced (`goog.`). But because of that nesting, defensive code is harder to write in a way that optimizes well after down-leveling.

A more significant difference from `ngDevMode` is that it's defined as:

```ts
goog.DEBUG = goog.define('goog.DEBUG', true);
```

Which means that it defaults to `true` instead of being off by default.

*Aside: That snippet demonstrates a unique aspect of `goog.DEBUG`: It's not just a one-off solution; it uses a generic "compile time define" mechanism.*

## `import.meta.DEBUG`

**Support:** TBD.<br/>
**Graceful fallback:** Everywhere.

The heading is a bit of a spoiler - let's recap first. What makes a good guard for debug code?

1. Easy to use by developers with a "pit of success": It shouldn't break in any standard JS environment and err on the side of being turned off.
2. Easy to detect by tools with a low failure rate.
3. Supported by default across tools, with no manual setup required. This implies: It should be maximally compatible with existing code.
4. Cleanly maps to existing primitives so that libraries can choose to apply the guards ahead of time in their published artifacts.
5. Tools should be able (if they so desire) to offer finer granularity than "enabled globally".

What ticks all the boxes? `import.meta.DEBUG`. Well, it doesn't have wide support in tools but let's call that "no wide support _yet_".

The semantics are as follows:

```ts
interface ImportMeta {
  /**
   * import.meta.DEBUG
   * 
   * This value is true if and only if the development
   * condition is active for this file.
   * Otherwise it's either false or undefined.
   * 
   * Support for this field implies the following:
   *   - It SHOULD be set to `true` if the development
   *     condition is enabled.
   *   - It MUST be defined as `false` for dead code removal
   *     purposes if the production condition is enabled.
   *   - It MAY be preserved in the output if neither of
   *     development/production are enabled.
   */
  DEBUG?: boolean;
}
```

Usage example:

```ts
export function isPrime(n) {
  import.meta.DEBUG &&
      assert(typeof n === 'number', `Can't check non-numbers!`);
  return n === 7 ? 'yes' : 'maybe';
}
```

Assuming that this code exists as the root `exports` entry of a package, it should be fully equivalent to:

```ts
// In package.json
  "exports": {
    "development": "./is-prime.debug.js",
    "default": "./is-prime.default.js"
  }

// In is-prime.debug.js:
export function isPrime(n) {
  // import.meta.DEBUG=true inlined.
  assert(typeof n === 'number', `Can't check non-numbers!`);
  return n === 7 ? 'yes' : 'maybe';
}

// In is-prime.default.js:
export function isPrime(n) {
  // import.meta.DEBUG=false inlined.
  return n === 7 ? 'yes' : 'maybe';
}
```

There should never be an observable difference between these two, micro benchmarks aside.

### "If the development condition is active for this file"

These words are confusing - because this concept isn't a thing. Conditions currently apply either to an import site ("require"/"import") or globally. They aren't active for a specific file.

Which means, right now the answer is easy. The "development" condition is enabled globally by tools that support it so - it's just active for all files. For 100% of current adoption cases, that's all that matters.

---

**Anything that follows is crystal gazing, bordering on wild speculation. At best, it's describing "preserved design space".**

---

But in a world with more fine-grained "development" conditions, there may be a need to define "active for one specific file". The intended meaning in this context is: The condition is applied to the enclosing package.

Example:

```
/app.js
/node_modules/lib-a/package.json
/node_modules/lib-a/src/a.js
/node_modules/lib-b/package.json
/node_modules/lib-b/src/b.js
```

In this setup, `lib-a` is the "enclosing package" of `lib-a/src/a.js`. Running a hypothetical build tool with scoped development support might look like this:

```sh
$ npx future-tool build
  --mode="production"
  --development="lib-a"
```

This would load everything with the "production" condition enabled - with the exception of any `imports` and `exports` fields in `lib-a/package.json`. And since the "development" condition is active in `lib-a/package.json`, `import.meta.DEBUG` would be set to true in `lib-a/src/a.js` as well. The condition would be considered enabled "for this file".

Scoping it at the package level is meaningful because it prevents zalgo when mixing inline conditionals (`import.meta.DEBUG`) with conditionals in `package.json` in the same package.

## References

* Bundler Docs
    * webpack
        * [Module Variables](https://webpack.js.org/api/module-variables/)
        * [Package Exports: Optimizations](https://webpack.js.org/guides/package-exports/#optimizations)
    * Rspack
        * [Mode](https://rspack.dev/config/mode#usage)
        * [Module Variables](https://rspack.dev/api/runtime-api/module-variables)
        * [Code: Package Exports conditions](https://github.com/web-infra-dev/rspack/blob/d01e85cb18a253d384d13e04eab8823d9960dc7f/packages/rspack/src/config/defaults.ts#L986)
    * Vite
        * [Env Variables and Modes](https://vite.dev/guide/env-and-mode)
        * [Package Exports: `resolve.conditions`](https://vite.dev/config/shared-options.html#resolve-conditions)
    * esbuild
        * [Platform: `process.env.NODE_ENV`](https://esbuild.github.io/api/#platform)
        * [Conditions - no development/production by default](https://esbuild.github.io/api/#conditions)
        * [Code: Final conditions set](https://github.com/evanw/esbuild/blob/d34e79e2a998c21bb71d57b92b0017ca11756912/internal/resolver/resolver.go#L282-L298)
    * Parcel
        * [Package exports](https://parceljs.org/features/dependency-resolution/#package-exports)
        * [`import.meta`](https://parceljs.org/languages/javascript/#import.meta)
        * [Environment Variables](https://parceljs.org/features/node-emulation/#environment-variables)
    * Rollup
        * [`plugin-node-resolve`: `exportConditions`](https://www.npmjs.com/package/@rollup/plugin-node-resolve#user-content-exportconditions)
        * [`plugin-replace`: NODE_ENV](https://github.com/rollup/plugins/tree/master/packages/replace)
* Testing Docs
    * Jest
        * [ECMAScript Modules / `import.meta`](https://jestjs.io/blog/2022/04/25/jest-28#ecmascript-modules)
        * [`import.meta.env` & Jest](https://dev.to/rubymuibi/jest-and-vite-cannot-use-importmeta-outside-a-module-24n3)
        * [`process.env.NODE_ENV = 'test'`](https://jestjs.io/docs/environment-variables)
* Server Runtime Docs
    * Bun
        * [`import.meta` w/ `.env.DEV`](https://bun.sh/docs/api/import-meta)
    * Deno
        * [`import.meta`](https://docs.deno.com/api/web/~/ImportMeta)
    * Node.js
        * [`import.meta`](https://nodejs.org/api/esm.html#importmeta)
    * WinterCG
        * [`import.meta` Registry](https://github.com/wintercg/import-meta-registry)
