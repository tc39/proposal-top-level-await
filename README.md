# ECMAScript proposal: Top-level `await`

## Status

This proposal is currently in stage 1 of [the TC39 process](https://tc39.github.io/process-document/).

## Background

[The `async` / `await` proposal](https://github.com/tc39/ecmascript-asyncawait) was originally brought to committee in [January of 2014](https://github.com/tc39/tc39-notes/blob/master/es6/2014-01/jan-30.md). In [April of 2014](https://github.com/tc39/tc39-notes/blob/master/es6/2014-04/apr-10.md) it was discussed that the keyword `await` should be reserved in the module goal for the purpose of top-level `await`. In [July of 2015](https://github.com/tc39/tc39-notes/blob/master/es7/2015-07/july-30.md) [the `async` / `await` proposal](https://github.com/tc39/ecmascript-asyncawait) advanced to Stage 2. During this meeting it was decided to punt on top-level `await` to not block the current proposal as top-level `await` would need to be "designed in concert with the loader".

Since the decision to delay standardizing top-level `await` it has come up in a handful of committee discussions, primarily to ensure that it would remain possible in the language.

## Motivation

### top-level main and IIAFEs

The current implementation of `async / await` only support the `await` keyword inside of `async` functions. As such many programs that are utilizing `await` now make use of top-level main function:

```mjs
import ...
async function main() {
  const dynamic = await import('./dynamic-thing.mjs');
  const data = await fetch(dynamic.url);
  console.log(data);
}
main();
export ...
```

This pattern is creating an abundance of boilerplate code in the ecosystem. If the pattern is repeated throughout the module graph it degrades the determinism of the execution.

This pattern can also be immediately invoked.

```mjs
import static1 from './static1.mjs';
import { readFile } from 'fs';

(async () => {
  const dynamic = await import('./dynamic' + process.env.something + '.js');
  const file = JSON.parse(await readFile('./config.json'));

  // main program goes here...
})();
```

###  completely dynamic modules

Another pattern that is beginning to surface is exporting async function and awaiting the results of imports, which drastically impacts our ability to do static analysis of the module graph.

```mjs
export default async () => {
  // import other modules like this one
  const import1 = await (await import('./import1.mjs')).default();
  const import2 = await (await import('./import2.mjs')).default();

  // compute some exports...
  return {
    export1: ...,
    export2: ...
  };
};
```

At the current time [a search](https://github.com/search?utf8=%E2%9C%93&q=%22export+async+function%22&type=Code) for "export async function" on github produces over 5000 unique code examples of exporting an async function.

## Proposed solutions

### Variant A: top-level `await` blocks tree execution

In this proposed solution a call to `top-level await` would block execution in the graph until it had resolved.

In this implementation you could consider the following

```mjs
import a from './a.mjs'
import b from './b.mjs'
import c from './c.mjs'

console.log(a, b, c);
```

If each of the modules above had a top level await present the loading would have similar execution order to

```mjs
(async () => {
  const a = await import('./a.mjs');
  const b = await import('./b.mjs');
  const c = await import('./c.mjs');
  
  console.log(a, b, c);
})();
```

Module a would need to finish executing before b or c could execute.

### Variant B: top-level `await` does not block sibling execution

In this proposed solution a call to `top-level await` would block execution of child nodes in the graph but would allow siblings to continue to execute. 

In this implementation you could consider the following

```mjs
import a from './a.mjs'
import b from './b.mjs'
import c from './c.mjs'

console.log(a, b, c);
```

If each of the modules above had a top level await present the loading would have similar execution order to

```mjs
(async () => {
  const [a, b, c] = await Promise.all([
    import('./a.mjs'),
    import('./b.mjs'),
    import('./c.mjs')
  ]);
  console.log(a, b, c);
})();
```

Modules a, b, and c would all execute in order up until the first await in each of them; we then wait on all of them to resume and finish evaluating before continuing.

### Optional Constraint: top-level `await` can only be used in modules without exports

Many of the use cases for top-level `await` are for bootstrapping an application. Enforcing that top-level `await` could only be used inside of a module without exports would allow individuals to use many of the patterns they would like to use in application bootstrapping while avoiding the edge cases of graph blocking or deadlocks.

With this constraint the implementation would still need to decide between Variant A or B for sibling execution behavior.

## Illustrative examples

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

### FAQ

#### Isn't top-level `await` a footgun?

If you have seen [the gist](https://gist.github.com/Rich-Harris/0b6f317657f5167663b493c722647221) you likely have heard this critique before. My hope is that as a committee we can weigh the pros / cons of the various approaches and determine if the benefits of the feature outweigh the risks.

#### Halting Progress

Variant A would halt progress in the module graph until resolved.

Variant B offers a unique approach to blocking, as it will not block siblings execution.

Variant C would alleviate concerns of halting process

##### Existing Ways to halt progress

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


#### What about deadlock?

The main problem space in designing top level await is to aid in detecting and preventing forms of deadlock that can occur. All examples below will use a cyclic `import()` to a graph of ``'a' -> 'b', 'b' -> 'a'`` with both using top level await to halt progress until the other finishes loading. For brevity the examples will only show one side of the graph, the other side is a mirror.

##### Guarding against

###### Implementing a TDZ

```mjs
// a
await import('b');

// implement a hoistable then()
export function then(f, r) {
  r('not finished');
};

// remove the rejection
then = null;
```

```mjs
// b
await import('a');
```

Having a `then` in the TDZ is a way to prevent cycles while a module is still 
evaluating. `try{}catch{}` can also be used as a recovery or notification
mechanism.

```mjs
// b
let a;
try {
  a = await import('a');
} catch {
  // do something
}
```

Given the amount of boilerplate code required to implement this behavior, one could argue that the language specification should codify this behavior by saying that `import()` should always reject if the imported module has not finished evaluating. However, that behavior would make `import()` a less predictable abstraction for importing asynchronous modules, since it would deprive modules using top-level `await` safely (that is, without creating dependency cycles) of the chance to finish async work, just because they were imported using `import()`. Given the likelihood that these "safe" modules will vastly outnumber modules using top-level `await` unsafely, rejection should not be a blanket policy, but instead an option for handling specific cases of circular imports.

Instead of `import()` rejecting if the imported module is suspended on an `await` expression, it might be useful if there was a way for dynamic `import()` to obtain a reference to the incomplete module namespace object, similar to how CommonJS `require` returns an incomplete `module.exports` object in the event of cycles. As long as the incomplete namespace object provides enough information, or the namespace object isn't used until later, this strategy would allow the application to keep making progress, rather than deadlocking. In contrast to CommonJS, ECMAScript modules can better enforce TDZ restrictions (preventing export use before initialization), and can throw more useful static errors. However, preventing any access to the incomplete namespace object would be a loss of functionality compared to CommonJS.

## Specification

* [Ecmarkup source](https://github.com/mylesborins/proposal-top-level-await/blob/master/spec.html)
* [HTML version](https://mylesborins.github.io/proposal-top-level-await/)

## Implementations

* none yet

## References

* https://github.com/bmeck/top-level-await-talking/
* https://gist.github.com/Rich-Harris/0b6f317657f5167663b493c722647221
