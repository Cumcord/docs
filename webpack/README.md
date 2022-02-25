# Webpack explained

This page will explain webpack as deeply as I can, and how module finding works on a lower level.

I will start by getting some basic concepts down, then digging into the implementation,
key things to know about, and finally a tiny bit about how webpack works internally.

## The basic concepts

### Modules

If you have ever worked with ES Modules (hint hint, import and export in your plugins),
then you will be used to the concept of every file being a separate module with imports,
and more crucially exports.

Webpack groups together modules in a similar way: each module has the following:
 - an ID, by which it is referenced internally
 - if the moudle is loaded or not
 - an `exports` property, under which are the exports of the ES Module the webpack moudle represents.

We care about the exports, as they contain useful things we can use in our plugins.

For example, as I write this, the module with `getGuild()` has an ID of `30098`,
so if we have all webpack modules, we could get the function as so:
```js
webpackModules[30098].exports.default.getGuild
```

These modules are the basis of our comfy module finding APIs,
which will check every module for a condition we specify.

### Webpack Require

Webpack require, from now on referred to as `wpRequire`, is a crucial part of webpack functioning.

Modules have imports, and to fullfil those imports, webpack has a function to "require" modules.

`wpRequire` can be called with a module id to load it,
but more interestingly has a larger set of props with fun™ uses:
 - c: an object with all loaded modules - this is the most important one for us
 - m: an object with functions to load all modules - has some interesting uses with lazy loading
 - e: an async function to load more chunks: we can force lazy loads!
 - u: gets the js file for a chunk ID
 - many more that arent very relevant to us

### Chunks

Webpack does not load moudles directly, but rather groups them into "chunks".
This is a key aspect of how code splitting works.

In Discord, all chunks are stored in an array on window called `webpackChunkdiscord_app`.
Hence we can add and remove chunks ourself manually.

Chunks contain the following:
 - an ID
 - an object with functions on it to construct modules - these go into `m` until they are needed
 - optionally a function to run on load, this is the most relevant to us


## How we get wpRequire

First, we push our own module onto the webpack chunk.
We use `Symbol()`, as it is a unique id and thus will always load.

We supply an empty object, as we don't actually want to export any modules (though we actually could!!!).

And finally we supply a function that just returns its only argument - **wpRequire**,
which the webpack chunk `push()` function passes thru as its return.

Finally we pop to remove our chunk. The full code looks like this:

```js
const wpRequire = webpackChunkdiscord_app.push([
  [Symbol()],
  {},
  (e) => e,
]);

webpackChunkdiscord_app.pop();
```

We then get all modules on `wpRequire.c`.

This is exported on `cumcord.modules.webpack.modules`.

## How it works: find with filters

First, we need to get all the modules in webpack, then we get the exports of each.
If the module has a default export and is an ES Module, we test the default export against
the filter.

We then test the modules exports against the filter.

This means that your filter will be called twice for most modules: once with `exports`, and once with `exports.default`.

Then, we can return either the first, or all of these modules.

*Almost* all webpack modules are built on top of these simple find filters.

### An example
This will be passed every module, and finds the root module for message components.
We use `> -1` instead of `!== -1` so that the value must be a number,
which is an extra constraint in finding the module.
```js
find(
  (m) => m.type?.toString().indexOf("MESSAGE_A11Y_ROLE_DESCRIPTION") > -1
);
```

### findByProps
Check that module has all the props:
```js
find(m =>
  props.every(prop => m[prop] !== undefined)
);
```

### findByPrototypes
Check that a module has all the props on its prototype:
```js
find(m =>
  m.prototype &&
  props.every(prop => m.prototype[prop] !== undefined)
);
```

### findByDisplayName
Check that a module has the specified display name:
```js
defaultExport
  ? find(
      m => m.displayName === displayName
    )
  : find(
      m => m?.default?.displayName === displayName
    )
```

### findByStrings

`findByStrings` is more of an interesting case, as it is recursive.
I won't try to explain a full implementation, but you can see one
[here](https://github.com/Cumcord/Cumcord/blob/2aa6eb7642e3a561d92b6a72ddccf7d351109ab5/src/api/modules/webpackModules.js#L110).

But essentially:
 - `find(m => )` through all your modules
 - recursively `.toString()` through all props to see if any match the strings
   * use a tree searcher for this

### findByKeywordAll
Find by a set of keywords
```js
find(
  m =>
    searches.every(
      s =>
        Object.keys(m).some(
          k => k.toLowerCase().includes(s.toLowerCase())
        )
    )
)
```

## getModule
Sometimes, we have the default export yet want the top level exports.
We can easily implement it like this:

```js
for (const id in modules) {
  const mod = modules[id]?.exports;
  // the ?.default is important here
  if (mod === module || mod?.default === module)
    return mod;
}
```

## How it works: lazy loading

When Discord wants to load a chunk,
it will be fetched from the web via `wpRequire.e()`, then pushed to the chunk.

At this point, all the modules in the chunk will be added to `wpRequire.m`,
and when they are loaded, will be built into `wpRequire.c`, which means we can find them.

### How we patch it

There are many approaches to patching over lazy loads, but ours is as follows:
 - Patch `push` for each module we find - the patcher is lightweight enough for this
 - Patch every module in the chunk (`Object.values(chunk[1])`).
 - Now that we have gotten patches into the chunk, remove this chunk patch
 - Whenever one of these chunks is called:
   * test if the module finds
   * if it does, resolve the promise with the module and unpatch stuff
 - If someone manually unpatches, remove all our patches early

This implementation lives at https://github.com/Cumcord/Cumcord/blob/2aa6eb7642e3a561d92b6a72ddccf7d351109ab5/src/api/modules/findAsync.js

Thus we use findAsync as so:
```js
const [modulePromise, unpatch] = findAsync(() => findByDisplayName("MessageContextMenu", false), false);

await modulePromise === {default: /* f() ... */};
```

## Key takeaways

- Modules are stored in a big object
- We can use getModule to get exports from exports.default to help with patching
- We can patch over chunks to allow easily dealing with lazy loading

## The internals of webpack

Here is what the wpRequire function looks like internally, cleaned up:

```js
function wpRequire(modId) {
  let maybeModule = modulesStore[modId];
  if (maybeModule !== undefined)
    return maybeModule.exports;

  let newModule = { id: modId, loaded: false, exports: {} };
  modulesStore[modId] = newModule;
  
  modFuncStore[modId].bind(newModule.exports)(newModule, newModule.exports, wpRequire);
  newModule.loaded = true;
  return newModule.exports;
}
```

`modulesStore` is `wpRequire.c`,
and `modFuncStore` is `wpRequire.m`: functions that build the `.c` modules.