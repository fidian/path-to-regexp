# Path-to-RegExp

> Turn a path string such as `/user/:name` into a regular expression.

[![NPM version][npm-image]][npm-url]
[![Build status][travis-image]][travis-url]
[![Test coverage][coveralls-image]][coveralls-url]
[![Dependency Status][david-image]][david-url]
[![License][license-image]][license-url]
[![Downloads][downloads-image]][downloads-url]

## Installation

```
npm install path-to-regexp --save
```

## Usage

```javascript
const { pathToRegexp, match, parse, compile } = require("path-to-regexp");

// pathToRegexp(path, keys?, options?)
// match(path)
// parse(path)
// compile(path)
```

- **path** A string, array of strings, or a regular expression.
- **keys** An array to populate with keys found in the path.
- **options**
  - **sensitive** When `true` the regexp will be case sensitive. (default: `false`)
  - **strict** When `true` the regexp allows an optional trailing delimiter to match. (default: `false`)
  - **end** When `true` the regexp will match to the end of the string. (default: `true`)
  - **start** When `true` the regexp will match from the beginning of the string. (default: `true`)
  - **delimiter** The default delimiter for segments, e.g. `[^/]` for `:named` patterns. (default: `'/'`)
  - **endsWith** Optional character, or list of characters, to treat as "end" characters.
  - **encode** A function to encode strings before inserting into `RegExp`. (default: `x => x`)
  - **prefixes** List of characters to automatically consider prefixes when parsing. (default: `./`)

```javascript
const keys = [];
const regexp = pathToRegexp("/foo/:bar", keys);
// regexp = /^\/foo\/([^\/]+?)\/?$/i
// keys = [{ name: 'bar', prefix: '/', delimiter: '/', optional: false, repeat: false, pattern: '[^\\/]+?' }]
```

**Please note:** The `RegExp` returned by `path-to-regexp` is intended for ordered data (e.g. pathnames, hostnames). It can not handle arbitrarily ordered data (e.g. query strings, URL fragments, JSON, etc).

### Parameters

The path argument is used to define parameters and populate keys.

#### Named Parameters

Named parameters are defined by prefixing a colon to the parameter name (`:foo`).

```js
const regexp = pathToRegexp("/:foo/:bar");
// keys = [{ name: 'foo', prefix: '/', ... }, { name: 'bar', prefix: '/', ... }]

regexp.exec("/test/route");
//=> [ '/test/route', 'test', 'route', index: 0, input: '/test/route', groups: undefined ]
```

**Please note:** Parameter names must use "word characters" (`[A-Za-z0-9_]`).

##### Custom Matching Parameters

Parameters can have a custom regexp, which overrides the default match (`[^/]+`). For example, you can match digits or names in a path:

```js
const regexpNumbers = pathToRegexp("/icon-:foo(\\d+).png");
// keys = [{ name: 'foo', ... }]

regexpNumbers.exec("/icon-123.png");
//=> ['/icon-123.png', '123']

regexpNumbers.exec("/icon-abc.png");
//=> null

const regexpWord = pathToRegexp("/(user|u)");
// keys = [{ name: 0, ... }]

regexpWord.exec("/u");
//=> ['/u', 'u']

regexpWord.exec("/users");
//=> null
```

**Tip:** Backslashes need to be escaped with another backslash in JavaScript strings.

##### Custom Prefix and Suffix

Parameters can be wrapped in `{}` to create custom prefixes or suffixes for your segment:

```js
const regexp = pathToRegexp("/:attr1?{-:attr2}?{-:attr3}?");

regexp.exec("/test");
// => ['/test', 'test', undefined, undefined]

regexp.exec("/test-test");
// => ['/test', 'test', 'test', undefined]
```

#### Unnamed Parameters

It is possible to write an unnamed parameter that only consists of a regexp. It works the same the named parameter, except it will be numerically indexed:

```js
const regexp = pathToRegexp("/:foo/(.*)");
// keys = [{ name: 'foo', ... }, { name: 0, ... }]

regexp.exec("/test/route");
//=> [ '/test/route', 'test', 'route', index: 0, input: '/test/route', groups: undefined ]
```

#### Modifiers

Modifiers must be placed after the parameter (e.g. `/:foo?`, `/(test)?`, or `/:foo(test)?`).

##### Optional

Parameters can be suffixed with a question mark (`?`) to make the parameter optional.

```js
const regexp = pathToRegexp("/:foo/:bar?");
// keys = [{ name: 'foo', ... }, { name: 'bar', delimiter: '/', optional: true, repeat: false }]

regexp.exec("/test");
//=> [ '/test', 'test', undefined, index: 0, input: '/test', groups: undefined ]

regexp.exec("/test/route");
//=> [ '/test/route', 'test', 'route', index: 0, input: '/test/route', groups: undefined ]
```

**Tip:** The prefix is also optional, escape the prefix `\/` to make it required.

##### Zero or more

Parameters can be suffixed with an asterisk (`*`) to denote a zero or more parameter matches.

```js
const regexp = pathToRegexp("/:foo*");
// keys = [{ name: 'foo', delimiter: '/', optional: true, repeat: true }]

regexp.exec("/");
//=> [ '/', undefined, index: 0, input: '/', groups: undefined ]

regexp.exec("/bar/baz");
//=> [ '/bar/baz', 'bar/baz', index: 0, input: '/bar/baz', groups: undefined ]
```

##### One or more

Parameters can be suffixed with a plus sign (`+`) to denote a one or more parameter matches.

```js
const regexp = pathToRegexp("/:foo+");
// keys = [{ name: 'foo', delimiter: '/', optional: false, repeat: true }]

regexp.exec("/");
//=> null

regexp.exec("/bar/baz");
//=> [ '/bar/baz','bar/baz', index: 0, input: '/bar/baz', groups: undefined ]
```

### Match

The `match` function will return a function for transforming paths into parameters:

```js
// Make sure you consistently `decode` segments.
const match = match("/user/:id", { decode: decodeURIComponent });

match("/user/123"); //=> { path: '/user/123', index: 0, params: { id: '123' } }
match("/invalid"); //=> false
match("/user/caf%C3%A9"); //=> { path: '/user/caf%C3%A9', index: 0, params: { id: 'café' } }
```

#### Process Pathname

You should make sure variations of the same path match the expected `path`. Here's one possible solution using `encode`:

```js
const match = match("/café", { encode: encodeURI, decode: decodeURIComponent });

match("/user/caf%C3%A9"); //=> { path: '/user/caf%C3%A9', index: 0, params: { id: 'café' } }
```

**Note:** [`URL`](https://developer.mozilla.org/en-US/docs/Web/API/URL) automatically encodes pathnames for you.

##### Alternative Using Normalize

Sometimes you won't have an already normalized pathname. You can normalize it yourself before processing:

```js
/**
 * Normalize a pathname for matching, replaces multiple slashes with a single
 * slash and normalizes unicode characters to "NFC". When using this method,
 * `decode` should be an identity function so you don't decode strings twice.
 */
function normalizePathname(pathname: string) {
  return (
    decodeURI(pathname)
      // Replaces repeated slashes in the URL.
      .replace(/\/+/g, "/")
      // Reference: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/normalize
      // Note: Missing native IE support, may want to skip this step.
      .normalize()
  );
}

// Two possible ways of writing `/café`:
const re = pathToRegexp("/caf\u00E9");
const input = encodeURI("/cafe\u0301");

re.test(input); //=> false
re.test(normalizePathname(input)); //=> true
```

### Parse

The `parse` function will return a list of strings and keys from a path string:

```js
const tokens = parse("/route/:foo/(.*)");

console.log(tokens[0]);
//=> "/route"

console.log(tokens[1]);
//=> { name: 'foo', prefix: '/', delimiter: '/', optional: false, repeat: false, pattern: '[^\\/]+?' }

console.log(tokens[2]);
//=> { name: 0, prefix: '/', delimiter: '/', optional: false, repeat: false, pattern: '.*' }
```

**Note:** This method only works with strings.

### Compile ("Reverse" Path-To-RegExp)

The `compile` function will return a function for transforming parameters into a valid path:

```js
// Make sure you encode your path segments consistently.
const toPath = compile("/user/:id", { encode: encodeURIComponent });

toPath({ id: 123 }); //=> "/user/123"
toPath({ id: "café" }); //=> "/user/caf%C3%A9"
toPath({ id: "/" }); //=> "/user/%2F"

toPath({ id: ":/" }); //=> "/user/%3A%2F"

// Without `encode`, you need to make sure inputs are encoded correctly.
const toPathRaw = compile("/user/:id");

toPathRaw({ id: "%3A%2F" }); //=> "/user/%3A%2F"
toPathRaw({ id: ":/" }, { validate: false }); //=> "/user/:/"

const toPathRepeated = compile("/:segment+");

toPathRepeated({ segment: "foo" }); //=> "/foo"
toPathRepeated({ segment: ["a", "b", "c"] }); //=> "/a/b/c"

const toPathRegexp = compile("/user/:id(\\d+)");

toPathRegexp({ id: 123 }); //=> "/user/123"
toPathRegexp({ id: "123" }); //=> "/user/123"
toPathRegexp({ id: "abc" }); //=> Throws `TypeError`.
toPathRegexp({ id: "abc" }, { validate: false }); //=> "/user/abc"
```

**Note:** The generated function will throw on invalid input.

### Working with Tokens

Path-To-RegExp exposes the two functions used internally that accept an array of tokens:

- `tokensToRegexp(tokens, keys?, options?)` Transform an array of tokens into a matching regular expression.
- `tokensToFunction(tokens)` Transform an array of tokens into a path generator function.

#### Token Information

- `name` The name of the token (`string` for named or `number` for unnamed index)
- `prefix` The prefix string for the segment (e.g. `"/"`)
- `suffix` The suffix string for the segment (e.g. `""`)
- `pattern` The RegExp used to match this token (`string`)
- `modifier` The modifier character used for the segment (e.g. `?`)

## Compatibility with Express <= 4.x

Path-To-RegExp breaks compatibility with Express <= `4.x`:

- RegExp special characters can only be used in a parameter
  - Express.js 4.x supported `RegExp` special characters regardless of position - this is considered a bug
- Parameters have suffixes that augment meaning - `*`, `+` and `?`. E.g. `/:user*`
- No wildcard asterisk (`*`) - use parameters instead (`(.*)` or `:splat*`)

## Live Demo

You can see a live demo of this library in use at [express-route-tester](http://forbeslindesay.github.com/express-route-tester/).

## License

MIT

[npm-image]: https://img.shields.io/npm/v/path-to-regexp.svg?style=flat
[npm-url]: https://npmjs.org/package/path-to-regexp
[travis-image]: https://img.shields.io/travis/pillarjs/path-to-regexp.svg?style=flat
[travis-url]: https://travis-ci.org/pillarjs/path-to-regexp
[coveralls-image]: https://img.shields.io/coveralls/pillarjs/path-to-regexp.svg?style=flat
[coveralls-url]: https://coveralls.io/r/pillarjs/path-to-regexp?branch=master
[david-image]: http://img.shields.io/david/pillarjs/path-to-regexp.svg?style=flat
[david-url]: https://david-dm.org/pillarjs/path-to-regexp
[license-image]: http://img.shields.io/npm/l/path-to-regexp.svg?style=flat
[license-url]: LICENSE.md
[downloads-image]: http://img.shields.io/npm/dm/path-to-regexp.svg?style=flat
[downloads-url]: https://npmjs.org/package/path-to-regexp
