# ECMAScript proposal: Top-level `await`

Champions: Myles Borins, Yulia Startsev.

Authors: Myles Borins, Yulia Startsev, Daniel Ehrenberg, Guy Bedford, Ms2ger, and others.

Status: Stage 4

## Synopsis

Top-level `await` enables modules to act as big async functions: With top-level `await`, ECMAScript Modules (ESM) can `await` resources, causing other modules who `import` them to wait before they start evaluating their body.

## Motivation

### Limitations on IIAFEs

With `await` only available within `async` functions, a module can include an `await` in the code that executes at startup by factoring that code into an `async` function:

```mjs
// awaiting.mjs
import { process } from "./some-module.mjs";
let output;
async function main() {
  const dynamic = await import(computedModuleSpecifier);
  const data = await fetch(url);
  output = process(dynamic.default, data);
}
main();
export { output };
```

This pattern can also be immediately invoked. You could call this an Immediately Invoked Async Function Expression (IIAFE), as a play on [IIFE](https://developer.mozilla.org/en-US/docs/Glossary/IIFE) idiom.

```mjs
// awaiting.mjs
import { process } from "./some-module.mjs";
let output;
(async () => {
  const dynamic = await import(computedModuleSpecifier);
  const data = await fetch(url);
  output = process(dynamic.default, data);
})();
export { output };
```

This pattern is appropriate for situations where loading a module is intended to schedule work that will happen some time later. However, the exports from this module may be accessed before this async function completes: If another module imports this one, it may see `output` as `undefined`, or it may see it after it's initialized to the return value of `process`, depending on when the access occurs! For example:

```mjs
// usage.mjs
import { output } from "./awaiting.mjs";
export function outputPlusValue(value) { return output + value; }

console.log(outputPlusValue(100));
setTimeout(() => console.log(outputPlusValue(100), 1000);
```

### Workaround: Export a Promise to represent initialization

In the absence of this feature, it's possible to export a Promise from a module, and wait on that to know when its exports are ready. For example, the above module could be written as:

```mjs
// awaiting.mjs
import { process } from "./some-module.mjs";
let output;
export default (async () => {
  const dynamic = await import(computedModuleSpecifier);
  const data = await fetch(url);
  output = process(dynamic.default, data);
})();
export { output };
```

Then, the module could be used as:

```mjs
// usage.mjs
import promise, { output } from "./awaiting.mjs";
export function outputPlusValue(value) { return output + value }

promise.then(() => {
  console.log(outputPlusValue(100));
  setTimeout(() => console.log(outputPlusValue(100), 1000);
});
```

However, this leaves us with a number of problems still:
- Everyone has to learn about a particular protocol to find the right Promise to wait on the module being loaded
- If you forget to apply the protocol, things might "just work" some of the time (due to the race being won in a certain way)
- In a deep module hierarchy, the Promise needs to be explicitly threaded through each step of the chain.

For example, here, we waited on the promise from `"./awaiting.mjs"` properly, but we forgot to re-export it, so modules that use our module may still run into the original race condition.

### Avoiding the race through significant additional dynamism

To avoid the hazard of forgetting to wait for the exported Promise before accessing exports, a module could instead export a Promise which resolves to an object which contains exports

```mjs
// awaiting.mjs
import { process } from "./some-module.mjs";
export default (async () => {
  const dynamic = await import(computedModuleSpecifier);
  const data = await fetch(url);
  const output = process(dynamic.default, data);
  return { output };
})();
```

```mjs
// usage.mjs
import promise from "./awaiting.mjs";

export default promise.then(({output}) => {
  function outputPlusValue(value) { return output + value }

  console.log(outputPlusValue(100));
  setTimeout(() => console.log(outputPlusValue(100), 1000);

  return { outputPlusValue };
});
```

It's unclear whether this pattern has caught on, but it's sometimes [recommended in StackOverflow](https://stackoverflow.com/questions/42958334/how-can-i-export-promise-result/42958644#42958644) to people who face these sorts of issues.

However, this pattern has the undesirable effect of requiring a broad reorganization of the related source into more dynamic patterns, and placing much of the module body inside the `.then()` callback in order to use the dynamically available imports. This represents a significant regression in terms of static analyzability, testability, ergonomics and more, compared to ES2015 modules. And when you run into a deep dependency which needs to `await`, you need to reorganize all dependent modules to use this pattern.

### Solution: Top-level `await`

Top-level `await` lets us rely on the module system itself to handle all of these promises, and make sure that things are well-coordinated. The above example could be simply written and used as follows:

```mjs
// awaiting.mjs
import { process } from "./some-module.mjs";
const dynamic = import(computedModuleSpecifier);
const data = fetch(url);
export const output = process((await dynamic).default, await data);
```

```mjs
// usage.mjs
import { output } from "./awaiting.mjs";
export function outputPlusValue(value) { return output + value }

console.log(outputPlusValue(100));
setTimeout(() => console.log(outputPlusValue(100), 1000);
```

None of the statements in `usage.mjs` will execute until the `await`s in `awaiting.mjs` have had their Promises resolved, so the race condition is avoided by design. This is an extension of how, if `awaiting.mjs` didn't use top-level `await`, none of the statements in `usage.mjs` will execute until `awaiting.mjs` is loaded and all of its statements have executed.

## Use cases

When would it make sense to have a module which waits on an asynchronous operation to load? This section gives some examples.

### Dynamic dependency pathing

```mjs
const strings = await import(`/i18n/${navigator.language}`);
```

This allows for Modules to use runtime values in order to determine
dependencies. This is useful for things like development/production splits,
internationalization, environment splits, etc.

### Resource initialization

```mjs
const connection = await dbConnector();
```

This allows Modules to represent resources and also to produce errors in 
cases where the Module will never be able to be used.

### Dependency fallbacks

```mjs
let jQuery;
try {
  jQuery = await import('https://cdn-a.com/jQuery');
} catch {
  jQuery = await import('https://cdn-b.com/jQuery');
}
```

### WebAssembly Modules

WebAssembly Modules are "compiled" and "instantiated" in a logically asynchronous way, based on their imports: Some WebAssembly implementations do nontrivial work at either phase, which is important to be able to shunt off into another thread. To integrate with the JavaScript module system, they will need to do the equivalent of a top-level await. See the [WebAssembly ESM integration proposal](https://github.com/webassembly/esm-integration) for more details.

## Semantics as desugaring

Currently, a module waits for all of its dependencies to execute all of their statements before the import is considered finished, and the module's code can run. This proposal maintains this property when introducing `await`, : dependencies still execute through to the end, even if you need to wait for that execution to finish asynchronously. One way to think of this is as if each module exported a Promise, and after all the `import` statements, but before the rest of the module, the Promises are all `await`ed:

```mjs
import { a } from './a.mjs';
import { b } from './b.mjs';
import { c } from './c.mjs';

console.log(a, b, c);
```

would be roughly equivalent to

```mjs
import { promise as aPromise, a } from './a.mjs';
import { promise as bPromise, b } from './b.mjs';
import { promise as cPromise, c } from './c.mjs';

export const promise = Promise.all([aPromise, bPromise, cPromise]).then(() => {

console.log(a, b, c);

});
```

Modules `a.mjs`, `b.mjs`, and `c.mjs` would all execute in order up until the first await in each of them; we then wait on all of them to resume and finish evaluating before continuing.

### FAQ

#### Isn't top-level `await` a footgun?

If you have seen [the gist](https://gist.github.com/Rich-Harris/0b6f317657f5167663b493c722647221) you likely have heard this critique before.

Some responses to some of the top concerns here:

##### Will top-level `await` cause developers to make their code block longer than it should?

It's true that top-level `await` gives developers a new tool to make their code wait. Our hope is that proper developer education can ensure that the semantics of top-level `await` are well-understood, so that people know to use it just when they intend that importers should block on it.

We've seen this work well in the past. For example, it's easy to write code with async/await that serializes two tasks that could be done in parallel, but a deliberate developer education effort has popularized the use of `Promise.all` to avoid this hazard.

##### Will top-level `await` encourage developers to use `import()` unnecessarily, which is less optimizable?

Many JavaScript developers are learning about `import()` specifically as a tool for code splitting. People are becoming aware of the relationship between bundling and multiple requests, and learning how to combine them for good application performance. Top-level `await` doesn't really change the calculus--using `import()` from a top-level `await` will have similar performance effects to using it from a function. As long as we can tie top-level `await`'s educational materials into the existing knowledge of that performance tradeoff, we hope to be able to avoid counterproductive increases in the use of `import()`.

#### What exactly is blocked by a top-level `await`?

When one module imports another one, the importing module will only start executing its module body once the dependency's body has finished executing. If the dependency reaches a top-level await, that will have to complete before the importing module's body starts executing.

#### Why doesn't top-level `await` block the import of an adjacent module?

If one module wants to declare itself dependent on another module, for the purposes of waiting for that other module to complete its top-level `await` statements before the module body executes, it can declare that other module as an import.

In a case such as the following, the printed order will be `"X1"`, `"Y"`, `"X2"`, because importing one module "before" another does not create an implicit dependency.

```mjs
// x.mjs
console.log("X1");
await new Promise(r => setTimeout(r, 1000));
console.log("X2");
```

```mjs
// y.mjs
console.log("Y");
```

```mjs
// z.mjs
import "./x.mjs";
import "./y.mjs";
```

Dependencies are required to be explicitly noted in order to boost the potential for parallelism: Most setup work that will be blocking due to a top-level await (for example, all of the case studies above) can be done in parallel with other setup work from unrelated modules. When some of this work may be highly parallelizable (e.g., network fetches), it's important to get as many of these queued up close to the start of execution as possible.

#### What is guaranteed about code execution order?

Modules maintain the same ordering as in ES2015 for when they start executing. If a module reaches an `await`, it will yield control and let other modules initialize themselves in the same well-specified order.

To be specific: Regardless of whether top-level `await` is used, modules always initially start running in the same post-order traversal established in ES2015: execution of module bodies starts with the deepest imports, in the order that the import statements for them are reached. After a top-level `await` is reached, control is passed to start the next module in this traversal order, or to other asynchronously scheduled code.

#### Do these guarantees meet the needs of polyfills?

Currently (in a world without top-level `await`), polyfills are synchronous. So, the idiom of importing a polyfill (which modifies the global object) and then importing a module which should be affected by the polyfill will still work if top-level `await` is added. However, if a polyfill includes a top-level `await`, it will need to be imported by modules that depend on it in order to reliably take effect.

#### Does the `Promise.all` happen even if none of the imported modules have a top-level `await`?

If the module's execution is deterministically synchronous (that is, if it and its dependencies each contain no top-level `await`), there will be no entry in the `Promise.all` for that module. In this case, it will run synchronously.

These semantics preserve the current behavior of ES Modules, where, when top-level `await` is not used, the Evaluate phase is entirely synchronous. The semantics are a bit in contrast with uses of Promises elsewhere. For a concrete example and further discussion, see [issue #43](https://github.com/tc39/proposal-top-level-await/issues/43) and [#47](https://github.com/tc39/proposal-top-level-await/issues/47).

#### How exactly are dependencies waited on? Does it really use `Promise.all`?

The semantics of modules without top-level `await` is synchronous: the whole tree executes in postorder, with a module running just after its dependencies have run. The same semantics apply to modules with top-level `await`: Once the module which contains top-level `await` has executed, it will trigger the synchronous execution of dependent modules whose dependencies have all executed. If a module does contain top-level `await`, even if the `await` is not dynamically reached, the whole module will be treated as "asynchronous", as if it were a big async function. Therefore, anything that runs when it's done is in a Promise reaction. However, from there, if there are multiple modules which are dependent on it, and these do not contain top-level await, then they will run synchronously, *without* any Promise work between them.

#### Does top-level `await` increase the risk of deadlocks?

Top-level `await` creates a new mechanism for deadlocks, but the champions of this proposal consider the risk to be worth it, because:
- There are many existing ways that modules can create deadlocks or otherwise halt progress, and developer tools can help in debugging them
- All deterministic deadlock prevention strategies considered would be overly broad and block appropriate, realistic, useful patterns

##### Existing Ways to block progress

###### Infinite Loops

```mjs
for (const n of primes()) {
  console.log(`${n} is prime}`);
}
```

Infinite series or lack of base condition means static control structures
are vulnerable to infinite looping.

###### Infinite Recursion

```mjs
const fibb = n => (n ? fibb(n - 1) : 1);
fibb(Infinity);
```

Proper tail calls allow for recursion to never overflow the stack. This makes
it vulnerable to infinite recursion.

###### Atomics.wait

```mjs
Atomics.wait(shared_array_buffer, 0, 0);
```

Atomics allow blocking forward progress by waiting on an index that never changes.

###### `export function then`

```mjs
// a
export function then(f, r) {}
```

```mjs
async function start() {
  const a = await import('a');
  console.log(a);
}
```

Exporting a `then` function allows blocking `import()`.

###### Conclusion: Ensuring continued progress is a larger problem

##### Rejected deadlock prevention mechanisms

A potential problem space to solve for in designing top level await is to aid in detecting and preventing forms of deadlock that can occur. For example awaiting on a cyclical dynamic import could introduce deadlock into the module graph execution.

The following sections about deadlock prevention will be based on this code example:

```jsx
// file.html
<script type=module src="a.mjs"></script>

// a.mjs
await import("./b.mjs");

// b.mjs
await import("./a.mjs");
```

###### Alternative: Return a partially filled module record

In `b.mjs`, resolve the Promise immediately, even though `a.mjs` has not yet completed, to avoid a deadlock.

###### Alternative: Throw an exception on use of in-progress modules

In `b.mjs`, reject the Promise when importing `a.mjs` because that module hasn't completed yet, to prevent a deadlock.

###### Case study: Race to `import()` a module

Both of these strategies fall over when considering that multiple pieces of code may want to dynamically import the same module. Such multiple imports would not ordinarily be any sort of race or deadlock to worry about. However, neither of the above mechanisms would handle the situation well: One would reject the Promise, and the other would fail to wait for the imported module to be initialized.

###### Conclusion: No feasible strategy for deadlock avoidance

#### Will top-level await work in transpilers?

To the greatest extent possible. The widely deployed CommonJS (CJS) module system does not directly support top-level await, so any transpilation strategy targeting it will need adjustments. However, within this context, we've made several adjustments to the semantics of top-level await based on the feedback and experience of the authors of several JavaScript module systems, including transpiler authors. This proposal aims to be implementable in such contexts.

#### Without this proposal, module graph execution is synchronous. Does this proposal maintain developer expectations that such loading be synchronous?

To the greatest extent possible. When a module includes a top-level `await` (even if that `await` is not dynamically reached), this is not synchronous, and at the very least takes one trip through the Promise job queue. However, module subgraphs which do not use top-level await continue to run synchronously in exactly the same way as without this proposal. And if several modules which do not use top-level `await` depend on a module which does use it, then those modules will all run when the async module is ready, without yielding to any other work (neither the Promise job queue/microtask queue, nor the host's event loop, etc.). See [#74](https://github.com/tc39/proposal-top-level-await/pull/74) for details on the logic used.

#### Should module loading include microtask checkpoints between modules, or yielding to the event loop after modules load?

Maybe! These module loading questions are part of an exciting research area in loading performance, as well as an interesting discussion on the invariants surrounding microtask checkpoints. This proposal doesn't take an opinion on these questions, leaving the asynchronous behavior for separate proposals. Host environments may wrap modules in a way which does these things, and the top-level await specification machinery can be used to coordinate things. A future proposal either in TC39 or in a host environment could add additional microtask checkpoints. For related discussion, see [whatwg/html#4400](https://github.com/whatwg/html/issues/4400).

#### Would top-level await work in web pages?

Yes. The details of the integration into the HTML specification is proposed at [whatwg/html#4352](https://github.com/whatwg/html/pull/4352).

## History

[The `async` / `await` proposal](https://github.com/tc39/ecmascript-asyncawait) was originally brought to committee in [January of 2014](https://github.com/tc39/tc39-notes/blob/master/meetings/2014-01/jan-30.md#asyncawait). In [April of 2014](https://github.com/tc39/tc39-notes/blob/master/meetings/2014-04/apr-10.md#preview-of-asnycawait) it was discussed that the keyword `await` should be reserved in the module goal for the purpose of top-level `await`. In [July of 2015](https://github.com/tc39/tc39-notes/blob/master/meetings/2015-07/july-30.md#64-advance-async-functions-to-stage-2) [the `async` / `await` proposal](https://github.com/tc39/ecmascript-asyncawait) advanced to Stage 2. During this meeting it was decided to punt on top-level `await` to not block the current proposal as top-level `await` would need to be "designed in concert with the loader".

Since the decision to delay standardizing top-level `await` it has come up in a handful of committee discussions, primarily to ensure that it would remain possible in the language.

In May 2018, this proposal reached Stage 2 in TC39's process, with many design decisions (in particular, whether to block "sibling" execution) left open to be discussed during Stage 2.

## Specification

* [Ecmarkup source](https://github.com/tc39/proposal-top-level-await/blob/HEAD/spec.html)
* [HTML version](https://tc39.github.io/proposal-top-level-await/)

## Implementations

* [V8 v8.9](https://v8.dev/blog/v8-release-89)
* SpiderMonkey via [`javascript.options.experimental.top_level_await`](https://bugzilla.mozilla.org/show_bug.cgi?id=1519100) flag
* [JavaScriptCore](https://bugs.webkit.org/show_bug.cgi?id=202484)
* [webpack 5.0.0](https://webpack.js.org/blog/2020-10-10-webpack-5-release/)

## References

* https://github.com/bmeck/top-level-await-talking/
* https://gist.github.com/Rich-Harris/0b6f317657f5167663b493c722647221

[defer]: https://jakearchibald.com/2017/es-modules-in-browsers/#defer-by-default
