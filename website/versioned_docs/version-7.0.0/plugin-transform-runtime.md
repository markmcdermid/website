---
id: version-7.0.0-babel-plugin-transform-runtime
title: @babel/plugin-transform-runtime
sidebar_label: transform-runtime
original_id: babel-plugin-transform-runtime
---

A plugin that enables the re-use of Babel's injected helper code to save on codesize.

> NOTE: Instance methods such as `"foobar".includes("foo")` will not work since that would require modification of existing built-ins (you can use [`@babel/polyfill`](polyfill.md) for that).

## Installation

Install it as development dependency.

```sh
npm install --save-dev @babel/plugin-transform-runtime
```

and [`@babel/runtime`](runtime.md) as a production dependency (since it's for the "runtime").

```sh
npm install --save @babel/runtime
```

The transformation plugin is typically used only in development, but the runtime itself will be depended on by your deployed code. See the examples below for more details.

## Why?

Babel uses very small helpers for common functions such as `_extend`. By default this will be added to every file that requires it. This duplication is sometimes unnecessary, especially when your application is spread out over multiple files.

This is where the `@babel/plugin-transform-runtime` plugin comes in: all of the helpers will reference the module `@babel/runtime` to avoid duplication across your compiled output. The runtime will be compiled into your build.

Another purpose of this transformer is to create a sandboxed environment for your code. If you use [@babel/polyfill](polyfill.md) and the built-ins it provides such as `Promise`, `Set` and `Map`, those will pollute the global scope. While this might be ok for an app or a command line tool, it becomes a problem if your code is a library which you intend to publish for others to use or if you can't exactly control the environment in which your code will run.

The transformer will alias these built-ins to `core-js` so you can use them seamlessly without having to require the polyfill.

See the [technical details](#technical-details) section for more information on how this works and the types of transformations that occur.

## Usage

### With a configuration file (Recommended)

Without options:

```json
{
  "plugins": ["@babel/plugin-transform-runtime"]
}
```

With options (and their defaults):

```json
{
  "plugins": [
    [
      "@babel/plugin-transform-runtime",
      {
        "absoluteRuntime": false,
        "corejs": false,
        "helpers": true,
        "regenerator": true,
        "useESModules": false,
        "version": "7.0.0-beta.0"
      }
    ]
  ]
}
```

The plugin defaults to assuming that all polyfillable APIs will be provided by the user. Otherwise the [`corejs`](#corejs) option needs to be specified.

### Via CLI

```sh
babel --plugins @babel/plugin-transform-runtime script.js
```

### Via Node API

```javascript
require("@babel/core").transform("code", {
  plugins: ["@babel/plugin-transform-runtime"],
});
```

## Options

### `corejs`

`boolean` or `number` , defaults to `false`.

e.g. `['@babel/plugin-transform-runtime', { corejs: 2 }],`

Specifying a number will rewrite the helpers that need polyfillable APIs to reference `core-js` instead.

This requires changing the dependency used to be [`@babel/runtime-corejs2`](runtime-corejs2.md) instead of `@babel/runtime`.

```sh
npm install --save @babel/runtime-corejs2
```

### `helpers`

`boolean`, defaults to `true`.

Toggles whether or not inlined Babel helpers (`classCallCheck`, `extends`, etc.) are replaced with calls to `moduleName`.

For more information, see [Helper aliasing](#helper-aliasing).

### `polyfill`

> This option was removed in v7 by just making it the default.

### `regenerator`

`boolean`, defaults to `true`.

Toggles whether or not generator functions are transformed to use a regenerator runtime that does not pollute the global scope.

For more information, see [Regenerator aliasing](#regenerator-aliasing).

### `useBuiltIns`

> This option was removed in v7 by just making it the default.

### `useESModules`

`boolean`, defaults to `false`.

When enabled, the transform will use helpers that do not get run through
`@babel/plugin-transform-modules-commonjs`. This allows for smaller builds in module
systems like webpack, since it doesn't need to preserve commonjs semantics.

For example, here is the `classCallCheck` helper with `useESModules` disabled:

```js
exports.__esModule = true;

exports.default = function(instance, Constructor) {
  if (!(instance instanceof Constructor)) {
    throw new TypeError("Cannot call a class as a function");
  }
};
```

And, with it enabled:

```js
export default function(instance, Constructor) {
  if (!(instance instanceof Constructor)) {
    throw new TypeError("Cannot call a class as a function");
  }
}
```

### `absoluteRuntime`

`boolean` or `string`, defaults to `false`.

This allows users to run `transform-runtime` broadly across a whole project. By default, `transform-runtime` imports from `@babel/runtime/foo` directly, but that only works if `@babel/runtime` is in the `node_modules` of the file that is being compiled. This can be problematic for nested `node_modules`, npm-linked modules, or CLIs that reside outside the user's project, among other cases. To avoid worrying about how the runtime module's location is resolved, this allows users to resolve the runtime once up front, and then insert absolute paths to the runtime into the output code.

Using absolute paths is not desirable if files are compiled for use at a later time, but in contexts where a file is compiled and then immediately consumed, they can be quite helpful.

> You can read more about configuring plugin options [here](https://babeljs.io/docs/en/plugins#plugin-options)

### `version`

By default transform-runtime assumes that `@babel/runtime@7.0.0` is installed. If you have later versions of
`@babel/runtime` (or their corejs counterparts e.g. `@babel/runtime-corejs3`) installed or listed as a dependency, transform-runtime can use more advanced features.

## Technical details

The `transform-runtime` transformer plugin does three things:

- Automatically requires `@babel/runtime/regenerator` when you use generators/async functions (toggleable with the `regenerator` option).
- Can use `core-js` for helpers if necessary instead of assuming it will be polyfilled by the user (toggleable with the `corejs` option)
- Automatically removes the inline Babel helpers and uses the module `@babel/runtime/helpers` instead (toggleable with the `helpers` option).

What does this actually mean though? Basically, you can use built-ins such as `Promise`, `Set`, `Symbol`, etc., as well use all the Babel features that require a polyfill seamlessly, without global pollution, making it extremely suitable for libraries.

Make sure you include `@babel/runtime` as a dependency.

### Regenerator aliasing

Whenever you use a generator function or async function:

```javascript
function* foo() {}
```

the following is generated:

```javascript
"use strict";

var _marked = [foo].map(regeneratorRuntime.mark);

function foo() {
  return regeneratorRuntime.wrap(
    function foo$(_context) {
      while (1) {
        switch ((_context.prev = _context.next)) {
          case 0:
          case "end":
            return _context.stop();
        }
      }
    },
    _marked[0],
    this
  );
}
```

This isn't ideal since it relies on the regenerator runtime being included, which
pollutes the global scope.

With the `runtime` transformer, however, it is compiled to:

```javascript
"use strict";

var _regenerator = require("@babel/runtime/regenerator");

var _regenerator2 = _interopRequireDefault(_regenerator);

function _interopRequireDefault(obj) {
  return obj && obj.__esModule ? obj : { default: obj };
}

var _marked = [foo].map(_regenerator2.default.mark);

function foo() {
  return _regenerator2.default.wrap(
    function foo$(_context) {
      while (1) {
        switch ((_context.prev = _context.next)) {
          case 0:
          case "end":
            return _context.stop();
        }
      }
    },
    _marked[0],
    this
  );
}
```

This means that you can use the regenerator runtime without polluting your current environment.

### `core-js` aliasing

Sometimes you may want to use new built-ins such as `Map`, `Set`, `Promise` etc. Your only way
to use these is usually to include a globally polluting polyfill.

This is with the `corejs` option.

The plugin transforms the following:

```javascript
var sym = Symbol();

var promise = new Promise();

console.log(arr[Symbol.iterator]());
```

into the following:

```javascript
"use strict";

var _getIterator2 = require("@babel/runtime-corejs2/core-js/get-iterator");

var _getIterator3 = _interopRequireDefault(_getIterator2);

var _promise = require("@babel/runtime-corejs2/core-js/promise");

var _promise2 = _interopRequireDefault(_promise);

var _symbol = require("@babel/runtime-corejs2/core-js/symbol");

var _symbol2 = _interopRequireDefault(_symbol);

function _interopRequireDefault(obj) {
  return obj && obj.__esModule ? obj : { default: obj };
}

var sym = (0, _symbol2.default)();

var promise = new _promise2.default();

console.log((0, _getIterator3.default)(arr));
```

This means is that you can seamlessly use these native built-ins and static methods
without worrying about where they come from.

**NOTE:** Instance methods such as `"foobar".includes("foo")` will **not** work.

### Helper aliasing

Usually Babel will place helpers at the top of your file to do common tasks to avoid
duplicating the code around in the current file. Sometimes these helpers can get a
little bulky and add unnecessary duplication across files. The `runtime`
transformer replaces all the helper calls to a module.

That means that the following code:

```javascript
class Person {}
```

usually turns into:

```javascript
"use strict";

function _classCallCheck(instance, Constructor) {
  if (!(instance instanceof Constructor)) {
    throw new TypeError("Cannot call a class as a function");
  }
}

var Person = function Person() {
  _classCallCheck(this, Person);
};
```

the `runtime` transformer however turns this into:

```javascript
"use strict";

var _classCallCheck2 = require("@babel/runtime/helpers/classCallCheck");

var _classCallCheck3 = _interopRequireDefault(_classCallCheck2);

function _interopRequireDefault(obj) {
  return obj && obj.__esModule ? obj : { default: obj };
}

var Person = function Person() {
  (0, _classCallCheck3.default)(this, Person);
};
```

> You can read more about configuring plugin options [here](https://babeljs.io/docs/en/plugins#plugin-options)
