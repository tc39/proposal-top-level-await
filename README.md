# ECMAScript proposal: Top-level `await`

## Status

This proposal is currently being considered to enter stage 0 of [the TC39 process](https://tc39.github.io/process-document/).

## Background

[The `async` / `await` proposal](https://github.com/tc39/ecmascript-asyncawait) was originally brought to committee in [January of 2014](https://github.com/tc39/tc39-notes/blob/master/es6/2014-01/jan-30.md). In [April of 2014](https://github.com/tc39/tc39-notes/blob/master/es6/2014-04/apr-10.md) it was discussed that the keyword `await` should be reserved in the module goal for the purpose of top-level `await`. In [July of 2015](https://github.com/tc39/tc39-notes/blob/master/es7/2015-07/july-30.md) [the `async` / `await` proposal](https://github.com/tc39/ecmascript-asyncawait) advanced to Stage 2. During this meeting it was decided to punt on top-level `await` to not block the current proposal as top-level `await` would need to be "designed in concert with the loader".

Since the decision to delay standardizing top-level `await` it has come up in a handful of committee discussions, primarily to ensure that it would remain possible in the language.

## Motivation

The current implementation of `async / await` only support the `await` keyword inside of `async` functions. As such we are beginning to see the following pattern emerge:

```js
import ...
async function main() {
  const dynamic = await import('./dynamic-thing.mjs');
}
main();
export ...
```

This pattern is reminiscent of the classic pattern of wrapping all of your code in a self executing function `(fn(){}())`. This type of "magic" does not benefit new developers and creates excessive repeated code throughout the ecosystem.

Another risk with the above pattern is that it makes the body of the function asynchronus. If people are utilizing this pattern throughout their graph there will no longer be a deterministic execution order.

Another pattern that is begining to surface is exporting async function and awaiting the results of imports, which drasticly impacts our ability to do static analysis of the module graph.

```js
export default async function (url) {
  if (!fetch) {
    const fetch = await import('./fetch-polyfill.mjs');
  }
  const data = await fetch(url);
  return data;
}
```

## Proposed solutions

### Variant A: top-level `await` blocks tree execution

In this proposed solution a call to `top-level await` would block execution in the graph until it had resolved.

### Variant B: top-level `await` does not block sibling execution

In this proposed solution a call to `top-level await` would block execution of child nodes in the graph but would allow siblings to continue to execute. 

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

If you have seen [the gist](https://gist.github.com/Rich-Harris/0b6f317657f5167663b493c722647221) you likely have heard this critique before. My hope is that as a committee we can weigh the pros / cons of the various approaches and determine if the feature is in fact a foot gun

#### Halting Progress

Variant A would halt progress in the module graph until resolved.

Variant B offers a unique approach to blocking, as it will not block siblings execution.

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

While it is tempting to codify this behavior by specifying that `import()` should reject if the imported module has not finished evaluating, that behavior would make `import()` a poor abstraction for importing asynchronous modules, since it would deprive well-behaved modules of the chance to finish async work, just because they were imported using `import()`.

Instead of `import()` rejecting if the imported module is suspended on an `await` expression, it might be more useful if there was a way for dynamic `import()` to obtain a reference to the incomplete module namespace object, similar to how CommonJS `require` returns an incomplete `module.exports` object in the event of cycles. As long as the incomplete namespace object provides enough information, or the namespace object isn't used until later, this strategy would allow the application to keep making progress, rather than deadlocking.

At the current time [a search](https://github.com/search?utf8=%E2%9C%93&q=%22export+async+function%22&type=Code) for "export async function" on github produces over 5000 unique code examples of exporting an async function.

## Specification

* [Ecmarkup source](https://github.com/mylesborins/proposal-top-level-await/blob/master/spec.html)
* [HTML version](https://mylesborins.github.io/proposal-top-level-await/)

## Implementations

* none yet

## References

* https://github.com/bmeck/top-level-await-talking/
* https://gist.github.com/Rich-Harris/0b6f317657f5167663b493c722647221
