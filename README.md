# Memoizee
## Complete memoize/cache solution for JavaScript

_Originally derived from [es5-ext](https://github.com/medikoo/es5-ext) package._

Memoization is best technique to save on memory or CPU cycles when we deal with repeated operations. For detailed insight see: http://en.wikipedia.org/wiki/Memoization

### Features

* Works with any type of function arguments – __no serialization is needed__
* Works with [__any length of function arguments__](#arguments-length). Length can be set as fixed or dynamic.
* One of the [__fastest__](#benchmarks) available solutions.
* Support for [__promises__](#promise-returning-functions) and [__asynchronous functions__](#nodejs-callback-style-functions)
* [__Primitive mode__](#primitive-mode) which assures fast performance when arguments are conversible to strings.
* [__WeakMap based mode__](#weakmap-based-configurations) for garbage collection friendly configuration
* Can be configured [__for methods__](#memoizing-a-method) (when `this` counts in)
* Cache [__can be cleared manually__](#manual-clean-up) or [__after specified timeout__](#expire-cache-after-given-period-of-time)
* Cache size can be __[limited on LRU basis](#limiting-cache-size)__
* Optionally [__accepts resolvers__](#resolvers) that normalize function arguments before passing them to underlying function.
* Optional [__reference counter mode__](#reference-counter), that allows more sophisticated cache management
* [__Profile tool__](#profiling--statistics) that provides valuable usage statistics
* Covered by [__over 500 unit tests__](#tests-)

### Installation

In your project path — __note the two `e`'s in `memoizee`:__

	$ npm install memoizee

_`memoize` name was already taken, therefore project is published as `memoizee` on NPM._

To port it to Browser or any other (non CJS) environment, use your favorite CJS bundler. No favorite yet? Try: [Browserify](http://browserify.org/), [Webmake](https://github.com/medikoo/modules-webmake) or [Webpack](http://webpack.github.io/)

### Usage

```javascript
var memoize = require('memoizee');

var fn = function (one, two, three) { /* ... */ };

memoized = memoize(fn);

memoized('foo', 3, 'bar');
memoized('foo', 3, 'bar'); // Cache hit
```

### Configuration

All below options can be applied in any combination

#### Arguments length

By default fixed number of arguments that function take is assumed (it's read from function's  `length` property) this can be overridden:

```javascript
memoized = memoize(fn, { length: 2 });

memoized('foo');            // Assumed: 'foo', undefined
memoized('foo', undefined); // Cache hit

memoized('foo', 3, {}); // Third argument is ignored (but passed to underlying function)
memoized('foo', 3, 13); // Cache hit
```

Dynamic _length_ behavior can be forced by setting _length_ to `false`, that means memoize will work with any number of arguments.

```javascript
memoized = memoize(fn, { length: false });

memoized('foo');
memoized('foo'); // Cache hit
memoized('foo', undefined);
memoized('foo', undefined); // Cache hit

memoized('foo', 3, {});
memoized('foo', 3, 13);
memoized('foo', 3, 13); // Cache hit
```

#### Primitive mode

If we work with large result sets, or memoize hot functions, default mode may not perform as fast as we expect. In that case it's good to run memoization in _primitive_ mode. To provide fast access, results are saved in hash instead of an array. Generated hash ids are result of arguments to string convertion. __Mind that this mode will work correctly only if stringified arguments produce unique strings.__

```javascript
memoized = memoize(fn, { primitive: true });

memoized('/path/one');
memoized('/path/one'); // Cache hit
```

#### Cache id resolution (normalization)

By default cache id for given call is resolved either by:
- Direct Comparison of values passed in arguments as they are. In such case two different objects, even if their characteristics is exactly same (e.g. `var a = { foo: 'bar' }, b = { foo: 'bar' }`) will be treated as two different values.
- Comparison of stringified values of given arguments (`primitive`  mode), which serves well, when arguments are expected to be primitive values, or objects that stringify naturally do unique values (e.g. arrays)

Still above two methods do not serve all cases, e.g. if we want to memoize function where arguments are hash objects which we do not want to compare by instance but by its content.

##### Writing custom cache id normalizers

There's a `normalizer` option through which we can pass custom cache id normalization function  
e.g. if we want to memoize a function where argument is a hash object which we do not want to compare by instance but by its content, then we can achieve it as following:

```javascript
var mfn = memoize(function (hash) {
	// body of memoized function
}, { normalizer: function (args) {
	// args is arguments object as accessible in memoized function
	return JSON.stringify(args[0]);
} });

mfn({ foo: 'bar' });
mfn({ foo: 'bar' }); // Cache hit
```

#### Argument resolvers

When we're expecting arguments of certain type it's good to coerce them before doing memoization. We can do that by passing additional resolvers array:

```javascript
memoized = memoize(fn, { length: 2, resolvers: [String, Boolean] });

memoized(12, [1,2,3].length);
memoized("12", true); // Cache hit
memoized({ toString: function () { return "12"; } }, {}); // Cache hit
```

__Note. If your arguments are collections (arrays or hashes) that you want to memoize by content (not by self objects), you need to cast them to strings__, for it's best to just use [primitive mode](#primitive-mode). Arrays have standard string representation and work with primitive mode out of a box, for hashes you need to define `toString` method, that will produce unique string descriptions, or rely on `JSON.stringify`.

Similarly __if you want to memoize functions by their code representation not by their objects, you should use primitive mode__.

#### Memoizing asynchronous functions

##### Promise returning functions

With _promise_ option we indicate that we memoize a function that returns promise.

The difference from natural behavior is that in case when promise was rejected with exception,
the result is immediately removed from memoize cache, and not kept as further reusable result.

```javascript
var afn = function (a, b) {
	return new Promise(function (res) { res(a + b); });
};
memoized = memoize(afn, { promise: true });

memoized(3, 7);
memoized(3, 7); // Cache hit
```

###### Important notice on internal promises handling

To avoid error swallowing and registration of error handlers, `done` and `finally` (if implemented) are preferred over `then`

Still relying on `done` & `finally` pair, may cause trouble if implementation that's used throws rejection reasons when `done` is called with no _onFail_ callback, even though error handler might have been registered through other `then` or `done` call.

If that's the case for you, you can force to not use `finally` or `done` (even if implemented) by providing following value to `promise` option:
- `'done'`  - If `done` is implemented, it will purely try use `done` to register internal callbacks and not `finally` (even if it's implemented). If `done` is not implemented, this setting has no effect and callbacks are registered via `then`.  
_This mode comes with side effect of silencing eventual 'Unhandled errors' on returned promise_
- `'then'` - No matter if `done` and `finally` are implemented, internal callbacks will be registered via `then`.  
_This mode comes with side effect of silencing eventual 'Unhandled errors' on returned promise_

```javascript
memoized = memoize(afn, { promise: 'then' });
```

##### Node.js callback style functions

With _async_ option we indicate that we memoize asynchronous (Node.js style) function
Operations that result with an error are not cached.

```javascript
afn = function (a, b, cb) {
  setTimeout(function () {
    cb(null, a + b);
  }, 200);
};
memoized = memoize(afn, { async: true });

memoized(3, 7, function (err, res) {
  memoized(3, 7, function (err, res) {
    // Cache hit
  });
});

memoized(3, 7, function (err, res) {
  // Cache hit
});
```

#### Memoizing methods

When we are defining a prototype, we may want to define a method that will memoize it's results in relation to each instance. A basic way to obtain that would be:

```javascript
var Foo = function () {
  this.bar = memoize(this.bar.bind(this), { someOption: true });
  // ... constructor logic
};
Foo.prototype.bar = function () {
  // ... method logic
};
```

There's a lazy methods descriptor generator provided:

```javascript
var d = require('d');
var memoizeMethods = require('memoizee/methods');

var Foo = function () {
  // ... constructor logic
};
Object.defineProperties(Foo.prototype, memoizeMethods({
  bar: d(function () {
    // ... method logic
  }, { someOption: true })
}));
```

#### WeakMap based configurations

In this case memoization cache is not bound to memoized function (which we may want to keep forever), but to objects for which given results were generated.

This mode works only for functions of which first argument is expected to be an object.  
It can be combined with other options mentioned across documentation. However due to WeakMap specificity global clear is not possible.

```javascript
var memoize = require('memoizee/weak');

var memoized = memoize(function (obj) { return Object.keys(obj); });

var obj = { foo: true, bar: false };
memoized(obj);
memoized(obj); // Cache hit
```

#### Cache handling

##### Manual clean up:

Delete data for particular call.

```javascript
memoized.delete('foo', true);
```

Arguments passed to `delete` are treated with same rules as input arguments passed to function

Clear all cached data:

```javascript
memoized.clear();
```

##### Expire cache after given period of time

With _maxAge_ option we can ensure that cache for given call is cleared after predefined period of time (in milliseconds)

```javascript
memoized = memoize(fn, { maxAge: 1000 }); // 1 second

memoized('foo', 3);
memoized('foo', 3); // Cache hit
setTimeout(function () {
  memoized('foo', 3); // No longer in cache, re-executed
  memoized('foo', 3); // Cache hit
}, 2000);
```

Additionally we may ask to _pre-fetch_ in a background a value that is about to expire. _Pre-fetch_ is invoked only if value is accessed close to its expiry date. By default it needs to be within at least 33% of _maxAge_ timespan before expire:

```javascript
memoized = memoize(fn, { maxAge: 1000, preFetch: true }); // Defaults to 0.33

memoized('foo', 3);
memoized('foo', 3); // Cache hit

setTimeout(function () {
  memoized('foo', 3); // Cache hit
}, 500);

setTimeout(function () {
  memoized('foo', 3); // Cache hit, silently pre-fetched in next tick
}, 800);

setTimeout(function () {
  memoized('foo', 3); // Cache hit
}, 1300);
```

_Pre-fetch_ timespan can be customized:

```javascript
memoized = memoize(fn, { maxAge: 1000, preFetch: 0.6 });

memoized('foo', 3);
memoized('foo', 3); // Cache hit

setTimeout(function () {
  memoized('foo', 3); // Cache hit, silently pre-fetched in next tick
}, 500);

setTimeout(function () {
  memoized('foo', 3); // Cache hit
}, 1300);
```

_Thanks [@puzrin](https://github.com/puzrin) for helpful suggestions concerning this functionality_

##### Reference counter

We can track number of references returned from cache, and manually delete them. When the last reference is cleared, the cache is purged automatically:

```javascript
memoized = memoize(fn, { refCounter: true });

memoized('foo', 3);           // refs: 1
memoized('foo', 3);           // Cache hit, refs: 2
memoized('foo', 3);           // Cache hit, refs: 3
memoized.deleteRef('foo', 3); // refs: 2
memoized.deleteRef('foo', 3); // refs: 1
memoized.deleteRef('foo', 3); // refs: 0, Cache purged for 'foo', 3
memoized('foo', 3);           // Re-executed, refs: 1
```

##### Limiting cache size

With _max_ option you can limit cache size, it's backed with [LRU algorithm](http://en.wikipedia.org/wiki/Cache_algorithms#Least_Recently_Used), provided by low-level [lru-queue](https://github.com/medikoo/lru-queue) utility.

The _size_ relates purely to count of results we want to keep in cache, it doesn't relate to memory cost assiociated with cache value (but such feature is likely to be introduced with next version of memoizee).


```javascript
memoized = memoize(fn, { max: 2 });

memoized('foo', 3);
memoized('bar', 7);
memoized('foo', 3);    // Cache hit
memoized('bar', 7);    // Cache hit
memoized('lorem', 11); // Cache cleared for 'foo', 3
memoized('bar', 7);    // Cache hit
memoized('foo', 3);    // Re-executed, Cache cleared for 'lorem', 11
memoized('lorem', 11); // Re-executed, Cache cleared for 'bar', 7
memoized('foo', 3);    // Cache hit
memoized('bar', 7);    // Re-executed, Cache cleared for 'lorem', 11
```

##### Registering dispose callback

You can register a callback to be called on each value removed from the cache:

```javascript
memoized = memoize(fn, { dispose: function (value) { /*…*/ } });

var foo3 = memoized('foo', 3);
var bar7 = memoized('bar', 7);
memoized.clear('foo', 3); // Dispose called with foo3 value
memoized.clear('bar', 7); // Dispose called with bar7 value
```

### Benchmarks

Simple benchmark tests can be found in _benchmark_ folder. Currently it's just plain simple calculation of fibonacci sequences. To run it you need to install other test candidates:

	$ npm install underscore lodash lru-cache secondary-cache

Example output taken under Node v0.10.35 on 2011 MBP Pro:

```
Fibonacci 3000 x10:

1:    15ms  Memoizee (primitive mode)
2:    15ms  Underscore
3:    18ms  lru-cache                 LRU (max: 1000)
4:    21ms  secondary-cache           LRU (max: 1000)
5:    37ms  Lo-dash
6:    62ms  Memoizee (primitive mode) LRU (max: 1000)
7:   163ms  Memoizee (object mode)    LRU (max: 1000)
8:   195ms  Memoizee (object mode)
```

### Profiling & Statistics

If you want to make sure how much you benefit from memoization or just check if memoization works as expected, loading profile module will give access to all valuable information.

__Module needs to be imported before any memoization (that we want to track) is configured. Mind also that running profile module affects performance, it's best not to use it in production environment__

```javascript
var memProfile = require('memoizee/profile');
```

Access statistics at any time:

```javascript
memProfile.statistics;         // Statistcs accessible for programmatical use
console.log(memProfile.log()); // Output statistics data in readable form
```

Example console output:

```
------------------------------------------------------------
Memoize statistics:

 Init  Cache  %Cache  Source location
11604  35682   75.46  (all)
 2112  19901   90.41  at /Users/medikoo/Projects/_packages/next/lib/fs/is-ignored.js:276:12
 2108   9087   81.17  at /Users/medikoo/Projects/_packages/next/lib/fs/is-ignored.js:293:10
 6687   2772   29.31  at /Users/medikoo/Projects/_packages/next/lib/fs/watch.js:125:9
  697   3922   84.91  at /Users/medikoo/Projects/_packages/next/lib/fs/is-ignored.js:277:15
------------------------------------------------------------
```

* _Init_ – Initial hits
* _Cache_ – Cache hits
* _%Cache_ – What's the percentage of cache hits (of all function calls)
* _Source location_ – Where in the source code given memoization was initialized

### Tests [![Build Status](https://travis-ci.org/medikoo/memoizee.svg)](https://travis-ci.org/medikoo/memoizee)

	$ npm test

### Contributors

* [@puzrin](https://github.com/puzrin) (Vitaly Puzrin)
  * Proposal and help with coining right _pre-fetch_ logic for [_maxAge_](https://github.com/medikoo/memoize#expire-cache-after-given-period-of-time) variant
