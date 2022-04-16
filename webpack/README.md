# Webpack (from the wrong end)

This page will explain webpack as deeply as I can, and how module finding works on a lower level.

It will also explain how you can make the most of Cumcord's versatile searchers to find things as easily as possible.

I will start by getting some basic concepts down, then digging into the implementation,
key things to know about, and finally a tiny bit about how webpack works internally.

## The basic concepts

### Modules

![mod](../media/wp-module.svg ':size=55%')

Webpack modules are essentially analogous to ES Modules.

In most applications, including Discord, they group together things of similar purpose, include deps, can be a react component(s), or maybe a Flux store.

Each module has an ID, by which it is referenced internally, and an `exports` property, under which are the exported function(s), object(s), class(es), really anything the module exports.

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

wpRequire is designed to be used for code splitting, dynamic module loading, and the like, but is very valuable to us as it exposes the internal logic of webpack to us, and lets us do very ~~irresponsible~~ *fun* things.

This function is actually intended to be similar to CJS' `require()` function, but for use in webpack moudles.

It has some fun props of use:

- `c`: an object with all loaded modules - this is the most important one for us
- `m`: an object with functions to load all modules - useful for messing with lazy loading
- `e`: an async function to load more chunks: we can force lazy loads!
- `u`: gets the js filename for a chunk ID
- many more that arent very relevant to us

### Chunks

![chunk](../media/wp-chunk.svg ':size=55%')

Chunks are the core of code splitting and are how Webpack loads groups of modules. They're essentially an array with a specific structure. We actually load a fake chunk to get access to wpRequire in most injection methods!

In Discord, all chunks are stored in an array on window called `webpackChunkdiscord_app`.

The chunks don't actually contain the modules directly, they contain a set of functions that take many args, and then once done will produce the loaded module. These functions are lazily ran when required, and can depend on other modules.

These being functions will later allow us to patch them for lazy loading fun!

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
cumcord.modules.webpack.modules = wpRequire.c;
```

## How it works: find with filters

First, we get the exports of each module.
If the module has an `exports.default` and is an ES Module, we test the default export against the filter.

We then test the module exports against the filter.

This means that your filter will be called twice for most modules: once with `exports`, and once with `exports.default`.

Then, we can return either the first, or all of these modules.

*Almost* all webpack modules are built on top of these simple find filters.

### For example

```js
find(m => m.displayName === "Switch")
```

### findByProps

Check that module has all the props.

This is the most useful and performant find, you can often simply pass it a function with a name that sounds useful to you and have a chance at getting what you want back.

For example, many Flux stores have prototype functions that are very useful, and this will find them.

```js
find(m =>
  props.every(prop => m[prop] !== undefined)
);
```

findByPrototypes is as above but it searches on `m.prototype` not `m`. Useful for finding classes.

### findByDisplayName

Check that a module has the specified display name.

Very very useful for finding React components.

Cumcord's implementation has the `defaultExp` option, by default `true`, but when set false will get the module exports object instead of the component function itself. This is useful as getting the exports object allows you to patch the component (though for class components this is unnecessary as you need to patch `exports.default.prototype.render`).

```js
find(
  m => (defaultExp ? m.displayName : m.default?.displayName) === displayName
)
```

### findByStrings

`findByStrings` is more of an interesting case, as it is recursive. I won't try to explain a full implementation, but you can see one [here](https://git.sr.ht/~creatable/Cumcord/tree/d2cdd9aefffedbba766cf0fbfd7e73b9d9685d63/item/src/api/modules/webpackModules.js#L110).

But essentially, recursively `.toString()` through all props to see if any match the strings (use a tree searcher for this!)

### findByKeywordAll

Find by a set of keywords in the props, case insensitively, searching substrings. Very useful when you don't know the exact name of the prop you want, but know roughly what you need.

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

This function searches through the webpack modules for one that matches what we have currently and returns it.

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

A quick note here - this method produces a LOT of patches, but remember that the CC patcher is low overhead and our injected patches are for the most part too - so just keep in mind that **PATCHES ARE CHEAP** *(here)*.

There are many approaches to patching over lazy loads, but ours is as follows:

- Patch `push` on the webpack chunk store - we add a separate patch here for each async find requested
- Patch every module *loader* in the chunk (remember this happens for each find)
- Whenever one of these chunks is called:
  * test if the module finds by traditional technique after this module loads
  * if it does, resolve the promise with the module and unpatch everything
- If the API consumer manually "unpatches" early, then remove all our patches

This implementation lives [here](https://git.sr.ht/~creatable/Cumcord/tree/a721ca3d0841086ed597bb14b7720595ddb4f964/item/src/api/modules/webpack/findAsync.js).

Thus we use findAsync as so:

```js
const [modulePromise, unpatch] = findAsync(() => findByDisplayName("MessageContextMenu", false), false);

await modulePromise === {default: /* f() ... */};
```

However, I ***HIGHLY*** suggest you don't use `findAsync` directly in the majority of cases, rather use `findAndPatch`:

```js
const unpatch = findAndPatch(
  () => findByDisplayName("MessageContextMenu", false),
  (MessCtxtMenu) => after("default", MessCtxtMenu, () => {})
);
```



## For the curious

Here is what the `wpRequire` function looks like internally, cleaned up:

```js
function wpRequire(modId) {
  let maybeModule = modulesStore[modId];
  if (maybeModule !== undefined)
    return maybeModule.exports;

  let newModule = { id: modId, loaded: false, exports: {} };
  modulesStore[modId] = newModule;

  modFuncStore[modId].apply(newModule.exports, [newModule, newModule.exports, wpRequire]);
  newModule.loaded = true;
  return newModule.exports;
}
```

`modulesStore` is `wpRequire.c`, and `modFuncStore` is `wpRequire.m`: functions that build the `.c` modules.