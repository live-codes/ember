# Ember × LiveCodes — proof of concept

A single HTML page that compiles and runs a modern Ember 5 app entirely in the browser. No build
step, no bundler, no server — the runtime and the whole toolchain come from a CDN, and everything
else happens in the page.

Modern means modern: Glimmer components with `<template>` tags, `@tracked`, `{{on}}`, `@args`,
ES modules with real Ember module specifiers, and route templates compiled the way
`ember-cli-htmlbars` compiles them — through `content-tag` and
`babel-plugin-ember-template-compilation`.

This is a standalone spike ahead of adding Ember as a language to LiveCodes.

## Run it

**Open `index.html` in a browser.** That's it — it works straight off `file://`, and the only
network access is to jsDelivr, unpkg and esm.sh. A static server works too, but nothing is
compiled server-side either way.

Edit any file on the left and press **Run** (or ⌘/Ctrl+Enter). Compile errors come back with the
file path and a code frame, and a failed compile leaves the previous preview on screen.

## What it does

The project is an ordinary Ember 5 app layout:

```
router.js                        modern Router class + Router.map
templates/application.hbs        nav with <LinkTo> + {{outlet}}
templates/index.hbs              a counter, plus two <CounterPanel> instances
templates/about.hbs              second route
controllers/index.js             Controller with @tracked state
components/counter-panel.gjs     Glimmer component with <template> and @tracked state
```

Pick the **Compiled output** tab to see what the toolchain produced for the selected file — the
real Ember wire format, the template's `scope`, and the `setComponentTemplate` call.

## Pipeline

```
app files (ES modules: .gjs, .js, .hbs)
   │
   │  content-tag                 <template> is not JavaScript; it is rewritten into
   │  (WASM, in the page)         `import { template } from "@ember/template-compiler"`
   │                              plus a `static { template(`...`, { component: this }) }` block
   ▼
JS with tagged/static template calls
   │
   │  @babel/standalone
   │  + babel-plugin-ember-template-compilation/browser
   │    (its `compiler` option = ember-source's `ember-template-compiler/minimal`)
   ▼
precompiled to Ember's wire format, co-located with
`setComponentTemplate(createTemplateFactory({...}), TheClass)` / `templateOnly()`
   │
   │  + proposal-decorators (legacy) + class-properties + class-static-block
   │  + transform-modules-commonjs
   ▼
CommonJS modules
   │  evaluated with a `require` that resolves relative files itself, serves
   │  `@glimmer/component` from a pre-bundled copy, and defers everything
   │  else to ember's own AMD module registry
   ▼
Ember.TEMPLATES + a resolver + `Ember.Router.dslCallbacks` → Application.create()
   ▼
DOM
```

## Findings

The modern pipeline is a *pipeline*, not a library — there is no single "compile Ember file"
entry point that a host can call. Each stage had a surprise.

### 1. The modern ESM distribution cannot compile templates; the legacy bundle can

`ember-source@5.12.0/dist/packages/@ember/template-compilation/index.js` is a **stub**. Its
`precompileTemplate` throws by design:

```js
precompileTemplate = () => {
  throw new Error('Attempted to call `precompileTemplate` at runtime, but this API is meant to be used at compile time. ...');
};
```

and `compileTemplate` delegates to a compiler that must be registered first — and the bundle's own
comment says that compiler is supplied by the build and "will **not** be present at runtime".

So `dist/packages/*` is a build artefact, not a browser entry point. But the *legacy* bundle it
publishes is exactly what the Babel plugin asks for. From its README:

> `compiler` — The ember-template-compiler.js module that ships within your ember-source version.
> In the browser implementation, this is **mandatory**.

And the compiler bundle exposes precisely the subset the plugin uses as its `minimal` module:

```js
require('ember-template-compiler/minimal')  // { _buildCompileOptions, _preprocess, _print, precompile }
```

So the modern pipeline uses `ember-template-compiler.js` as a *compiler*, never as a runtime — no
`Ember.HTMLBars`, no template registry. That is the one piece of the classic bundle the modern
toolchain genuinely still needs.

### 2. `<template>` is not JavaScript, so Babel needs a preprocessor

`<template>` tags are the content-tag proposal: valid only after a transform. That transform is
`content-tag`, a Rust/SWC-WASM preprocessor, and it ships a browser build
(`content-tag/pkg/standalone.js`) that works in the page. It rewrites

```js
export default class Greeting extends Component {
  <template><p>Hello {{this.name}}!</p></template>
}
```

into the RFC 931 form that the Babel plugin looks for:

```js
import { template as template_fd9b2463 } from "@ember/template-compiler";
export default class Greeting extends Component {
    static{
        template_fd9b2463(`<p>Hello {{this.name}}!</p>`, { component: this, eval () { return eval(arguments[0]); } });
    }
}
```

Note the injected import of `@ember/template-compiler`, and that the module specifier is mangled
per conversion (so two files never collide). The plugin's
`{ moduleName: '@ember/template-compiler', export: 'template', rfc931Support: 'polyfilled' }`
entry is what recognises it.

### 3. `@babel/standalone` cannot run the plugin — but a one-time `await` fixes it

`babel-plugin-ember-template-compilation`'s plugin factory is **async** (it awaits its options
loader). Babel supports async plugins only through `transformAsync`, and `@babel/standalone` has
no `transformAsync` — it exposes only `transform`, `transformFromAst`, `transformScriptTags`.

It also does not expose `types` / `traverse` / `template` on the global; those live under
`Babel.packages` as module namespaces.

Both problems dissolve if you resolve the plugin yourself. Let Babel invoke a trivial plugin to
hand you its own plugin API, await the real plugin once, and pass the resulting plain `PluginObj`
to the synchronous `transform`:

```js
let babelApi;
Babel.transform('', { plugins: [(api) => { babelApi = api; return { visitor: {} }; }] });

const pluginObj = await templatePlugin(babelApi, {
  compiler: window.require('ember-template-compiler/minimal'),
});

Babel.transform(source, { plugins: [pluginObj, ...] });   // synchronous, works
```

This keeps `@babel/standalone` (a single UMD script) rather than pulling in `@babel/core` — which
would not load from esm.sh here at all.

### 4. The wire output needs only three runtime imports, and they are all in the runtime

For a `<template>` in a class, the emitted output is:

```js
class CounterPanel extends Component {
  static {
    setComponentTemplate(createTemplateFactory({ "block": "[[[10,0],[14,0,\"panel\"], ... ]", "scope": () => [_modifier.on], "isStrictMode": true }), CounterPanel);
  }
}
```

— i.e. it imports `createTemplateFactory` from `@ember/template-factory`, `setComponentTemplate`
from `@ember/component`, and `templateOnly` from `@ember/component/template-only` when there is no
backing class. All three are registered in the AMD runtime, verified individually. Template-only
components therefore need no extra package at all.

### 5. `@glimmer/component` is the one missing piece, and it is small

It is not in ember-source; it is an addon. But `@glimmer/component@2.1.1` ships as a **single
21 KB file** whose imports are exactly five modules, every one present in the runtime:

```js
import { DEBUG } from '@glimmer/env';
import { capabilities, setComponentManager } from '@ember/component';
import { destroy } from '@ember/destroyable';
import { schedule } from '@ember/runloop';
import { setOwner } from '@ember/owner';
```

Its declared dependencies (`@embroider/addon-shim`, `@glimmer/env`) are not imported by the dist
build at runtime. So it can be fetched, run through the same CommonJS transform, and evaluated
against the AMD registry — which is what makes real Glimmer components possible here. Note it
calls `capabilities('3.13', {...})` and `setComponentManager(...)`, so it is coupled to the
runtime's internals, and has to stay on a version that matches.

### 6. Decorators: Ember's `@tracked` is a *legacy* decorator

This was the most expensive finding, and it presented as a bug in Ember rather than in the build:

```
Invalid value used as weak map key
  at setMeta → meta → descriptorForField → tracked(...) → <instance_members_initializer>
```

`@tracked` implements the **legacy** decorator protocol — `tracked(target, key, descriptor)`. With
Babel's default (stage 3) decorators a field decorator is called as `(value, context)`, so Ember
reads `target = undefined`, calls `descriptorForField`, and then `meta(undefined)` dies on
`WeakMap.set`. The fix is `['proposal-decorators', { legacy: true }]`.

That drags two more plugins in, and the order matters:

| step | plugin | why |
| --- | --- | --- |
| 1 | `proposal-decorators` `{ legacy: true }` | what Ember's decorators expect |
| 2 | `transform-class-properties` | otherwise legacy decorators throw `_initializerWarningHelper` ("ensure that transform-class-properties is enabled and runs after the decorators transform") |
| 3 | `transform-class-static-block` | the template plugin emits `static { setComponentTemplate(...) }`, which the class-properties transform cannot parse |

plus `assumptions: { setPublicClassFields: true }`, so decorated fields are assigned rather than
defined — otherwise the initialiser a decorator installed is clobbered by the field definition.

This is precisely the mess `decorator-transforms` (what a real Ember 5 build uses) exists to
avoid: it lets the fields stay native and hands decorator application to its own runtime. That is
the natural upgrade here, and the reason this PoC pins the legacy combination instead is only that
it needs no additional runtime module.

### 7. Strict-mode `<template>` templates must import their built-ins

The first run of the `.gjs` component failed with:

```
Attempted to resolve a modifier in a strict mode template, but that value was not in scope: on
```

`<template>` compiles to `isStrictMode: true`, where nothing is resolved implicitly — so
`{{on "click" ...}}` requires `import { on } from '@ember/modifier'`, which then shows up in the
compiled output as `"scope": () => [_modifier.on]`. Route templates in `.hbs` use resolution mode
and do not need this, which is why the same `{{on}}` in `templates/index.hbs` works unimported.
Correct Ember behaviour, and worth knowing before writing a starter template.

### 8. `moduleName` has to be patched in

The plugin passes the file name to the compiler as `filename`, but the wire format's module name
comes from `moduleName`, which nothing sets. Every template therefore claims to be
`(unknown template module)` — including inside runtime error messages. A post-hoc
`"(unknown template module)" → <path>` replacement in the generated code is the difference between
a useful error and a useless one. (Compile-time errors are already prefixed by Babel with the real
path, so only the runtime side needs it.)

### 9. The host glue from the runtime still applies

These are unchanged from any other browser host, and are all still required here:

- **`dist/ember.prod.js` does not define `window.Ember`.** It publishes a global `define`/`require`
  AMD loader; the namespace is `require('ember').default`, and that access is
  deprecation-instrumented in a debug build (which is why prod is used — the console stays clean).
- **`Ember.DefaultResolver` is gone** from ember-source, but `Application#buildRegistry` still calls
  `namespace.Resolver.create(...)` unconditionally, so `Application.create()` throws without a
  resolver. `template:*` is registered with `instantiate: false`, so the resolver returns the bare
  template factory from `Ember.TEMPLATES`.
- **Routes must be registered before `Application.create()`.** `setupEngineRegistry` registers
  `router:main` as the *base* Router class, and `Router#_setupRouter` reads that class's own
  `dslCallbacks` — so `App.Router.map()` after `create()` is silently ignored in a prod build. This
  PoC supports the idiomatic modern `app/router.js` (`class Router extends EmberRouter` +
  `Router.map(...)`) and copies the subclass's `dslCallbacks` onto `Ember.Router` before
  `create()`, clearing it per run.
- **`location: 'hash'`**, because `none` leaves `LinkTo` unable to compute an `href` and `history`
  would rewrite the host page's URL.
- **`{{#link-to "route"}}` no longer works** — `component:link-to` is a Glimmer component with named
  args, so every link renders the current route's href and is marked active. `<LinkTo @route="...">`
  is required.

## Verified

Checked in headless Chromium via CDP, loaded from `file://`, ember-source 5.12.0:

- Clean load: no console errors, no warnings; compile + boot in ~30–90 ms.
- Six modules compiled and registered; `templates: application, index, about`,
  `components: counter-panel`.
- Route templates render; `<LinkTo>` produces correct `href`s (`#/`, `#/about`) and `active` state;
  clicking transitions and renders the right template.
- `@tracked` on a controller updates through `{{on "click"}}` (count 0 → 2).
- Two `<CounterPanel @start={{...}} />` instances from one `.gjs` component, each with independent
  `@tracked` state, state initialised from `@args`.
- Two consecutive **Run**s, while on a non-`index` route, with no duplicate routes or leaked state.
- A malformed template surfaces the path and a Handlebars code frame
  (`/templates/index.hbs: Parse error on line 4: ... Expecting 'OPEN_INVERSE_CHAIN', ...`), and
  leaves the previous preview intact.
- The **Compiled output** pane shows the real wire format, the strict-mode `scope`, and the
  `setComponentTemplate` call.

## PoC limitations

- **Decorators go through the legacy pipeline**, with three plugins in a fixed order (finding 6).
  `decorator-transforms` would be the faithful Ember 5 setup.
- **No TypeScript.** `.gts` would need `@babel/plugin-transform-typescript` on top; the pipeline
  supports it, but it is not wired up.
- **No npm imports.** Modules can import Ember module specifiers (`@ember/*`, `@glimmer/*`, `ember/*`)
  and relative files, and `@glimmer/component` is pre-bundled — but nothing else resolves.
- **No services.** The resolver maps `template:`, `component:`, `controller:` and `route:` only, so
  `@service` injections do not resolve.
- **Route templates are `.hbs` only** — a `.gjs` route template would need explicit registration.
- **The route map is global, per-run state** (`Ember.Router.dslCallbacks`), so two apps cannot
  coexist.
- **Main-thread compilation**, and the first load is heavy — roughly 9.5 MB uncompressed before the
  plugin: `ember.prod.js` (1.83 MB), `ember-template-compiler.js` (673 KB), `@babel/standalone`
  (5.1 MB) and content-tag's WASM (1.88 MB). Moving the pipeline into a worker would not reduce
  that, only unblock the UI while it loads.
- **No language services** — the editor is a plain `<textarea>`; there is no highlighting, no
  template linting, no type checking (all of which need a different answer than a browser Babel).

## What this means for a LiveCodes adaptor

The important structural point: **the compiler is a pipeline, and it belongs in the worker.**
`content-tag` (WASM), `@babel/standalone` and the template plugin all run happily without a DOM, so
LiveCodes' `compiler.factory` can host the whole thing and hand the result page a finished module
graph — which is the same shape as the Vue/Svelte workers, just with more stages.

- **Pinning is load-bearing.** The wire format is coupled to ember-source, and `@glimmer/component`
  calls runtime internals. Pin ember-source 5.12.0 (runtime *and* compiler),
  `babel-plugin-ember-template-compilation` 4.0.0, `content-tag` 4.2.0, `@glimmer/component` 2.1.1.
- **Scripts, in order.** `compiler.scripts` (or `dependencies`) — `ember.prod.js`, then
  `ember-template-compiler.js` (it reuses the runtime's loader), then `@babel/standalone`. The
  content-tag WASM and the Babel plugin load as ES modules.
- **`compiler.imports` is not needed.** No bare specifier survives into the result page: the
  compiler resolves them all and emits CommonJS. What the wrapped modules need at execution time is
  the AMD registry, which the runtime already provides.
- **The transform configuration is part of the compiler**, not a detail: plugin order for
  decorators (finding 6), `assumptions.setPublicClassFields`, and the `moduleName` patch (finding 8).
- **The resolver and the route-map bridge stay the host's job** (finding 9) — they cannot live in
  user code, and they must be emitted by the language's `inlineScript`/`inlineModule`, sized to
  `config.files`. Forward `options.filename` as the module path so `moduleName` and error locations
  are right.
- **Errors are good enough to ship.** Babel's `buildCodeFrameError` gives file-prefixed code frames
  for both template and JS problems; the template compiler's own errors carry Handlebars code frames.
- **Worth considering:** a `.gjs`-only language plus the modern `<template>`-in-`.hbs` story would
  let LiveCodes skip the resolution-mode path entirely — strict mode is the simpler model, and it is
  the one that surfaces `scope` explicitly.

Sources: [ember-source@5.12.0 on jsDelivr](https://cdn.jsdelivr.net/npm/ember-source@5.12.0/),
[babel-plugin-ember-template-compilation](https://github.com/emberjs/babel-plugin-ember-template-compilation),
[content-tag](https://github.com/embroider-build/content-tag),
[@glimmer/component](https://cdn.jsdelivr.net/npm/@glimmer/component@2.1.1/),
[RFC 496 (strict mode / `precompileTemplate`)](https://github.com/emberjs/rfcs/blob/master/text/0496-handlebars-strict-mode.md).
