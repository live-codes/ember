# Ember × LiveCodes — proof of concept

A single HTML page that compiles and runs a multi-file Ember project entirely in the
browser. No build step, no bundler, no server — `ember.prod.js` and
`ember-template-compiler.js` come from a CDN and everything else happens in the page.

This is a standalone spike ahead of adding Ember as a language to LiveCodes.

## Run it

**Open `index.html` in a browser.** That's it — it works straight off `file://`, and the only
network access is to jsDelivr. A static server works too (and is closer to how LiveCodes would
host it), but nothing is compiled server-side either way.

Edit any file on the left and press **Run** (or ⌘/Ctrl+Enter). Compile errors and runtime
errors appear above the preview with Handlebars code frames.

Everything is pinned to `ember-source@5.12.0`.

## What it does

The starter project is four templates and one script, kept deliberately familiar:

```
application.hbs                  routes + <LinkTo> nav + {{outlet}}
index.hbs                        a counter, plus two <CounterPanel> instances
about.hbs                        second route
components/counter-panel.hbs     classic component template
app.js                           Router map, controllers, component class
```

Templates are compiled in the page, registered by name, and the application is built and booted
on top of them. The right-hand pane has three views:

- **Result** — the rendered app.
- **Compiled template** — the wire-format output of `Ember.HTMLBars.precompile` for the
  selected `.hbs` file, which is what actually gets registered.
- **App JS** — the script that runs.

## Pipeline

```
*.hbs files
   │  Ember.HTMLBars.precompile(src, { moduleName })     ← ember-template-compiler.js, in the page
   ▼
wire-format template string
   │  Ember.HTMLBars.compile(...) → template factory
   ▼
Ember.TEMPLATES["components/counter-panel"] = factory     ← shared registry
   │  Application.create({ ...staged classes, Resolver, location: 'hash' })
   ▼
DOM, rendered into the preview element
```

## Findings

Ember is the odd one out among the frameworks LiveCodes supports: it has no browser-capable
`file → JS` compiler that you can just call, and its globals are not what they used to be.

### 1. The modern ESM distribution cannot compile templates at runtime

`ember-source@5.12.0/dist/packages/@ember/template-compilation/index.js` is a **stub**:

```js
let precompileTemplate;
if (isDevelopingApp()) {
  precompileTemplate = () => {
    throw new Error('Attempted to call `precompileTemplate` at runtime, but this API is meant to be used at compile time. ...');
  };
}
```

and `compileTemplate` delegates to a compiler that must be registered first:

```js
const compileTemplate = (...args) => {
  if (!__emberTemplateCompiler) {
    throw new Error('Attempted to call `compileTemplate` without first loading the runtime template compiler.');
  }
  return __emberTemplateCompiler.compile(...args);
};
```

The bundle's own comment is explicit that this is supplied by the build, *for* the build, and
"will **not** be present at runtime". So `dist/packages/*` is a build artefact — it is not a
browser entry point, and the ESM tree also reaches for `@embroider/macros`. Modern Ember can
only compile templates ahead of time.

### 2. The classic AMD builds are still published — that is what makes this possible

`ember-source@5.12.0` still ships all three files the classic workflow needs:

| file | size | role |
| --- | --- | --- |
| `dist/ember.prod.js` | 1.83 MB | runtime |
| `dist/ember.debug.js` | — | runtime, with assertions |
| `dist/ember-template-compiler.js` | 673 KB | the template compiler |

Each is a self-contained bundle with its own tiny AMD `define`/`require` shim. This is the
"runtime template compilation" capability Ember removed from its modern distribution, still
shipping in a legacy form.

### 3. `dist/ember.prod.js` does **not** define `window.Ember`

This is the first surprise. Verified in the browser:

```
typeof window.Ember   → "undefined"
typeof window.require → "function"
typeof window.define  → "function"
```

The bundle publishes a global AMD loader instead of the namespace. Ember is reached as:

```js
const Ember = window.require('ember').default;
```

That `default` is a `Proxy` whose `get` trap fires a deprecation for every property read, which
matters for which build you use (finding 9).

`ember-template-compiler.js` notices the existing loader and reuses it, which is why **script
order matters**: runtime first, then compiler.

### 4. Loading the compiler bundle is what switches on `Ember.HTMLBars`

`ember-template-compiler.js` registers itself as the AMD module `ember-template-compiler/index`
(and `.../minimal`), then hands itself to the runtime:

```js
templateCompilation.__registerTemplateCompiler(emberTemplateCompilerLibPublicApi);
```

`Ember.HTMLBars` and `Ember.Handlebars` exist in the runtime as *lazy getters* that stay inert
until that registration happens; the getter is where `precompile` and `compile` get attached.
So "can I compile templates?" is answered by whether the compiler bundle has been evaluated,
not by any option.

The compiler bundle also registers a `domTemplates` application initializer that compiles
`<script type="text/x-handlebars">` tags found in the document into the registry. Handy to know
about, and worth *not* relying on — this PoC drives the compiler API directly instead.

### 5. The compile contract

```js
Ember.TEMPLATES[name] = Ember.HTMLBars.compile(source, { moduleName: name });
```

- `compile()` returns a **template factory function** (own keys: `__id`, `__meta`).
- `precompile()` returns the wire-format **string** — this is what the *Compiled template* pane
  shows, and it is what a real build would emit.
- `Ember.TEMPLATES` is not a plain object literal: it is an accessor property delegating to a
  shared registry with `getTemplate` / `setTemplate` / `hasTemplate`. Assigning into it works.
- Template names are free-form strings, but components have to be found under
  `components/<dasherized-name>` (finding 11).

### 6. Ember 5 has no resolver, but `Application.create()` requires one

`Ember.DefaultResolver` is gone from ember-source (it lives in the `ember-resolver` addon now),
and `Ember.Application.Resolver` is unset. But `Application#buildRegistry` still calls:

```js
function resolverFor(namespace) {
  let ResolverClass = namespace.Resolver;
  return ResolverClass.create({ namespace });
}
```

so `Application.create()` throws on `undefined.create` before it ever boots. A browser host
**must** supply a resolver. The contract is small — `resolve(fullName) → factory | undefined`
is the only method that must exist (`normalize`, `lookupDescription`, `makeToString`,
`knownForType` are all optional and guarded).

Two details make a minimal resolver enough:

- Class lookups are namespace-based — `controller:post` → `App.PostController`.
- `template:*` is registered with `instantiate: false`, so the resolver returns the bare
  template factory straight from `Ember.TEMPLATES` and the container hands it back unmodified.
  That is what makes component layouts resolve at all.

### 7. Routes must be registered *before* `Application.create()`

The nastiest one, because it fails silently in a production build.

- `setupEngineRegistry` does `registry.register('router:main', EmberRouter)` — the **base**
  Router class, not the application's.
- `Router#_setupRouter` reads the route map from `this.constructor.dslCallbacks`, an own
  *static* on that class.
- `_prepareForGlobalsMode()` runs during `init()` and sets `App.Router` to a **fresh subclass**,
  which is a different class from the one `router:main` resolves to.

So the idiomatic-looking

```js
const App = Ember.Application.create({ ... });
App.Router.map(function () { this.route('about'); });   // silently ignored
```

registers routes on a class nobody reads. The symptom is not an error: it is a router that only
knows the implicit routes, which surfaces as every link rendering the current route's `href`
and being marked `active`. In the prod build even `assert` is stripped, so the only way to see
it is to ask the router directly (`urlFor('about')` → *"There is no route named about"*).

The tempting fix — `autoboot: false`, then set things up, then boot — does not work either,
because `_globalsMode` is forced equal to `autoboot` and `didBecomeReady` only builds and starts
the instance when `autoboot` is truthy. (The modern `visit()` path is a different design.)

What works: collect the script's assignments first, register routes on `Ember.Router` (the class
`router:main` actually resolves to), clear `dslCallbacks` per run, and hand everything to
`create()` in one go. That staging object is also what makes `App.IndexController` resolvable.
This is the one piece of glue a host cannot avoid, and it has to come from the runtime shim
rather than from user code.

### 8. `{{#link-to "route"}}` no longer works

`component:link-to` is now `LinkTo = opaquify(_LinkTo, LinkToTemplate)` — a Glimmer component
taking **named** arguments. Curly invocation with positional params is silently dropped, so
`{{#link-to "about"}}` renders `href="#/"` and is marked as the active route. The working form is
angle-bracket:

```hbs
<LinkTo @route="about">About</LinkTo>
```

This is worth calling out for a starter template, because the old form is what every Ember
tutorial and blog post still uses.

### 9. Use `ember.prod.js`, not `ember.debug.js`

Because `require('ember').default` is a deprecation-firing `Proxy` (finding 3), the debug build
logs a `DEPRECATION: importing X from the 'ember' barrel file is deprecated` with a stack trace
for *every* `Ember.Something` access in a run — dozens per boot. The prod build strips
deprecations, so the console stays clean (verified: only Ember's own "install the Ember
Inspector" debug note).

The trade-off is real and worth stating: prod also strips `assert`, which is exactly what makes
finding 7 fail silently. During adaptor development, `ember.debug.js` is the better diagnostic
tool; for a playground, prod is the better default.

### 10. `location` matters for navigation

- `location: 'none'` — the router has no URL, so `LinkTo` cannot compute an `href` and clicking
  does nothing.
- `location: 'hash'` — works from `file://` and inside an iframe; keeps the host page's own URL
  intact.
- `location: 'history'` — needs a real origin and would rewrite the host's URL. Not appropriate
  for an embedded playground.

### 11. Classic components still work, and layouts resolve through the resolver

`Ember.Component.extend({...})` with a `components/<name>.hbs` template works: the layout is
looked up as `template:components/<dasherized-name>`, which the resolver from finding 6 serves
from `Ember.TEMPLATES`. Curly invocation with hash args maps to component properties, and
`{{action}}`, `Ember.computed`, `set` / `incrementProperty` and per-instance state all behave.
Two instances of the same component keep independent state, which is the thing worth confirming.

## Verified

Checked in headless Chromium via CDP, loaded from `file://`, `ember-source@5.12.0`:

- Four templates compile in-page and register on `Ember.TEMPLATES`; the app boots in ~5–15 ms.
- Two routes, with `<LinkTo>` rendering correct `href`s (`#/`, `#/about`) and correct `active`
  state; clicking transitions (`currentRouteName` → `about`) and renders the right template.
- Actions update state (`count` → `2`), `{{#if}}` / `Ember.computed` re-render reactively.
- Two `CounterPanel` instances from one component class, each with independent state.
- Carrying the router across a rebuild: two consecutive **Run**s both work, on a route other
  than `index`, without duplicate routes or leaked state.
- Live editing: changing a template's text and re-running shows the change.
- Error reporting: a malformed template surfaces a Handlebars code frame
  (`Parse error on line 4: ... Expecting 'OPEN_INVERSE_CHAIN', 'INVERSE', 'OPEN_ENDBLOCK', got 'EOF'`),
  and a throwing `app.js` surfaces its message. A template compile error leaves the previous
  preview intact.
- Console is clean (no errors, no deprecation spam).

## PoC limitations

- **Classic Ember only.** Globals-style API: `Ember.Application`, `Ember.Route`,
  `Ember.Controller`, `Ember.Component`, `{{action}}`. No `@glimmer/component`, no decorators,
  no `.gjs`/`.gts`, no `@tracked`.
- **Flat template registry.** No template imports and no module graph — templates are looked up
  by name at runtime, so there is no "import a component" story and nothing to bundle.
- **No compile-time checks.** Everything is runtime compilation, so there is no template
  type-checking and no build-time optimization; the wire format is evaluated with `new Function`.
- **The route map is global, per-run state.** `Ember.Router.dslCallbacks` is cleared between
  runs; two apps cannot coexist.
- **One application script.** No multi-file JS, no npm dependencies, no `import`.
- **Main-thread compilation**, and ~2.5 MB from the CDN on first load.
- **Specifier/wire format is version-coupled.** The compiled templates and the runtime internals
  must come from the same `ember-source` version.
- The editor is a plain `<textarea>`; there is no syntax highlighting or autocomplete.

## What this means for a LiveCodes adaptor

The hooks LiveCodes already has are close to sufficient, but Ember needs more from the *result
page* than Vue or Svelte do.

- **Two scripts in order, from a pinned version.** The result page has to load the runtime
  (`ember.prod.js`) and *then* the compiler (`ember-template-compiler.js`) — the compiler reuses
  the runtime's AMD loader, and only its evaluation turns `Ember.HTMLBars` into something
  callable. `compiler.scripts` with ordering, or `compiler.dependencies`, covers this. Version
  pinning is not cosmetic: the emitted wire format and the runtime internals must match.
- **No `window.Ember`.** The adaptor must not assume the classic global; the namespace is
  `require('ember').default`, and in a debug build that access is deprecation-instrumented.
- **Per-file output is trivial.** LiveCodes' one-file-one-module model maps cleanly here: a
  `.hbs` file compiles to `Ember.TEMPLATES["<name>"] = <wire format>`. There is no module graph
  to link, which makes this simpler than Qwik or Marko. `options.filename` should be forwarded
  as the compiler's `moduleName` so code frames name the right file.
- **The resolver and the route-map staging are the adaptor's job.** They cannot live in user
  code: `create()` throws without a `Resolver`, and routes must exist before `create()` while
  `App.Router.map()` afterwards is silently ignored. This wants to be emitted by the language's
  `inlineScript` / `inlineModule`, sized to `config.files` — and it has to stay in step with the
  pinned Ember version, since both are internal API.
- **Compile errors are already good.** `Ember.HTMLBars.compile` throws Handlebars parse errors
  with code frames, so the existing error pane has something useful to show.
- **Consider the modern route instead.** `babel-plugin-ember-template-compilation/browser` is
  documented as working in browsers without Node APIs, and takes `ember-template-compiler.js`
  as its mandatory `compiler` option — the same file this PoC uses. Paired with `@babel/standalone`
  and ember-source's ESM dist it would give `precompileTemplate` in the wire format that
  `setComponentTemplate` expects, which is the path to real Glimmer components and `.gjs`
  rather than the classic API. That is a bigger build, and the ESM tree's `@embroider/macros`
  dependency is the main unknown; it is the natural follow-up to this spike.

Sources: [ember-source@5.12.0 on jsDelivr](https://cdn.jsdelivr.net/npm/ember-source@5.12.0/),
[babel-plugin-ember-template-compilation](https://github.com/emberjs/babel-plugin-ember-template-compilation),
[Using AMD bundles deprecation](https://deprecations.emberjs.com/id/using-amd-bundles/).
