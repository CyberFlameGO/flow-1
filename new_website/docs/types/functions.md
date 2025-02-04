---
title: Function Types
slug: /types/functions
---

Functions have two places where types are applied: Parameters (input) and the
return value (output).

```js flow-check
// @flow
function concat(a: string, b: string): string {
  return a + b;
}

concat("foo", "bar"); // Works!
// $ExpectError
concat(true, false);  // Error!
```

Using inference, these types are often optional:

```js flow-check
// @flow
function concat(a, b) {
  return a + b;
}

concat("foo", "bar"); // Works!
// $ExpectError
concat(true, false);  // Error!
```

Sometimes Flow's inference will create types that are more permissive than you
want them to be.

```js flow-check
// @flow
function concat(a, b) {
  return a + b;
}

concat("foo", "bar"); // Works!
concat(1, 2);         // Works!
```

For that reason (and others), it's useful to write types for important
functions.

## Syntax of functions {#toc-syntax-of-functions}

There are three forms of functions that each have their own slightly different syntax.

### Function Declarations {#toc-function-declarations}

Here you can see the syntax for function declarations with and without types
added.

```js
function method(str, bool, ...nums) {
  // ...
}

function method(str: string, bool?: boolean, ...nums: Array<number>): void {
  // ...
}
```

### Arrow Functions {#toc-arrow-functions}

Here you can see the syntax for arrow functions with and without types added.

```js
let method = (str, bool, ...nums) => {
  // ...
};

let method = (str: string, bool?: boolean, ...nums: Array<number>): void => {
  // ...
};
```

### Function Types {#toc-function-types}

Here you can see the syntax for writing types that are functions.

```js
(str: string, bool?: boolean, ...nums: Array<number>) => void
```

You may also optionally leave out the parameter names.

```js
(string, boolean | void, Array<number>) => void
```

You might use these functions types for something like a callback.

```js
function method(callback: (error: Error | null, value: string | null) => void) {
  // ...
}
```

## Function Parameters {#toc-function-parameters}

Function parameters can have types by adding a colon `:` followed by the type
after the name of the parameter.

```js
function method(param1: string, param2: boolean) {
  // ...
}
```

## Optional Parameters {#toc-optional-parameters}

You can also have optional parameters by adding a question mark `?` after the
name of the parameter and before the colon `:`.

```js flow-check
function method(optionalValue?: string) {
  // ...
}
```

Optional parameters will accept missing, `undefined`, or matching types. But
they will not accept `null`.

```js flow-check
// @flow
function method(optionalValue?: string) {
  // ...
}

method();          // Works.
method(undefined); // Works.
method("string");  // Works.
// $ExpectError
method(null);      // Error!
```

### Rest Parameters {#toc-rest-parameters}

JavaScript also supports having rest parameters or parameters that collect an
array of arguments at the end of a list of parameters. These have an ellipsis
`...` before them.

You can also add type annotations for rest parameters using the same syntax but
with an `Array`.

```js flow-check
function method(...args: Array<number>) {
  // ...
}
```

You can pass as many arguments as you want into a rest parameter.

```js flow-check
// @flow
function method(...args: Array<number>) {
  // ...
}

method();        // Works.
method(1);       // Works.
method(1, 2);    // Works.
method(1, 2, 3); // Works.
```

> Note: If you add a type annotation to a rest parameter, it must always
> explicitly be an `Array` type.

### Function Returns {#toc-function-returns}

Function returns can also add a type using a colon `:` followed by the type
after the list of parameters.

```js flow-check
function method(): number {
  // ...
}
```

Return types ensure that every branch of your function returns the same type.
This prevents you from accidentally not returning a value under certain
conditions.

```js flow-check
// @flow
// $ExpectError
function method(): boolean {
  if (Math.random() > 0.5) {
    return true;
  }
}
```

Async functions implicitly return a promise, so the return type must always be a `Promise`.

```js flow-check
// @flow
async function method(): Promise<number> {
  return 123;
}
```

### Function `this` {#toc-function-this}

Every function in JavaScript can be called with a special context named `this`.
You can call a function with any context that you want. Flow allows you to annotate
the type for this context by adding a special parameter at the start of the function's parameter list:

```js flow-check
// @flow
function method<T>(this: { x: T }) : T {
  return this.x;
}

var num: number = method.call({x : 42});
var str: string = method.call({x : 42}); // error
```

This parameter has no effect at runtime, and is erased along with types when Flow is transformed into JavaScript.
When present, `this` parameters must always appear at the very beginning of the function's parameter list, and must
have an annotation. Additionally, [arrow functions](./#toc-arrow-functions) may not have a `this` parameter annotation, as
these functions bind their `this` parameter at the definition site, rather than the call site.

If an explicit `this` parameter is not provided, Flow will attempt to infer one based on usage. If `this` is not mentioned
in the body of the function, Flow will infer `mixed` for its `this` parameter.

### Predicate Functions {#toc-predicate-functions}

Sometimes you will want to move the condition from an `if` statement into a function:

```js
function concat(a: ?string, b: ?string): string {
  if (a && b) {
    return a + b;
  }
  return '';
}
```

However, Flow will flag an error in the code below:

```js
function truthy(a, b): boolean {
  return a && b;
}

function concat(a: ?string, b: ?string): string {
  if (truthy(a, b)) {
    // $ExpectError
    return a + b;
  }
  return '';
}
```

You can fix this by making `truthy` a *predicate function*, by using
the `%checks` annotation like so:

```js
function truthy(a, b): boolean %checks {
  return !!a && !!b;
}

function concat(a: ?string, b: ?string): string {
  if (truthy(a, b)) {
    return a + b;
  }
  return '';
}
```

#### Limitations of predicate functions {#toc-limitations-of-predicate-functions}

The body of these predicate functions need to be expressions (i.e. local variable declarations are not supported).
But it's possible to call other predicate functions inside a predicate function.
For example:

```js
function isString(y): %checks {
  return typeof y === "string";
}

function isNumber(y): %checks {
  return typeof y === "number";
}

function isNumberOrString(y): %checks {
  return isString(y) || isNumber(y);
}

function foo(x): string | number {
  if (isNumberOrString(x)) {
    return x + x;
  } else {
    return x.length; // no error, because Flow infers that x can only be an array
  }
}

foo('a');
foo(5);
foo([]);
```

Another limitation is on the range of predicates that can be encoded. The refinements
that are supported in a predicate function must refer directly to the value that
is passed in as an argument to the respective call.

For example, consider the *inlined* refinement

```js
declare var obj: { n?: number };

if (obj.n) {
  const n: number = obj.n;
}
```
Here, Flow will let you refine `obj.n` from `?number` to `number`. Note that the
refinement here is on the property `n` of `obj`, rather than `obj` itself.

If you tried to create a *predicate* function
```js
function bar(a): %checks {
  return a.n;
}
```
to encode the same condition, then the following refinement would fail
```js
if (bar(obj)) {
  // $ExpectError
  const n: number = obj.n;
}
```
This is because the only refinements supported through `bar` would be on `obj` itself.


### Callable Objects {#toc-callable-objects}

Callable objects can be typed, for example:

```js
type CallableObj = {
  (number, number): number,
  bar: string
};

function add(x, y) {
  return x + y;
}

// $ExpectError
(add: CallableObj);

add.bar = "hello world";

(add: CallableObj);
```

### `Function` Type {#toc-function-type}

> NOTE: For new code prefer `any` or `(...args: Array<any>) => any`. `Function` has become an alias to `any` and will be
> deprecated and removed in a future version of Flow.

Sometimes it is useful to write types that accept arbitrary functions, for
those you should write `() => mixed` like this:

```js
function method(func: () => mixed) {
  // ...
}
```

However, if you need to opt-out of the type checker, and don't want to go all
the way to `any`, you can instead use `(...args: Array<any>) => any`. (Note that [`any`](../any) is unsafe and
should be avoided). For historical reasons, the `Function` keyword is still available.

For example, the following code will not report any errors:

```js
function method(func: (...args: Array<any>) => any) {
  func(1, 2);     // Works.
  func("1", "2"); // Works.
  func({}, []);   // Works.
}

method(function(a: number, b: number) {
  // ...
});
```

Neither will this:

```js
function method(obj: Function) {
  obj = 10;
}

method(function(a: number, b: number) {
  // ...
});
```

> **You should follow [all the same rules](../any) as `any` when using `Function`.**
