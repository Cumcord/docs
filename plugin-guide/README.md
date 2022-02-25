# Creating Plugins
Cumcord plugins are in their most basic form, a JavaScript file that tells Cumcord what to do when it is loaded and unloaded.

This guide will explain technical details of what is involved in plugin creation.

## Getting Started
To get started, you'll need `sperm`, Cumcord's plugin build tool.  
`sperm` can be installed from [npm](https://www.npmjs.com/package/sperm '50%').

Creating a new plugin is as simple as making a new directory and running `sperm init` to initialize your plugin.

![init](../media/sperm-init.png ':size=55%')

## Creating a Plugin
`sperm` should've asked you for a main file path, so if you haven't already, create a JS (or JSX) file at the path you put in.

Let's start with an example plugin:
```js
export default (data) => {
  return {
    // This onLoad function will be called when your plugin loads, as the name implies.
    onLoad() {},
    // This onUnload function will be called when your plugin unloads, as the name *also* implies.
    onUnload() {}
  }
}
```

Plugins are functions that take in a `data` argument and return an object containing `onLoad` and `onUnload` functions.

Cumcord will manage your plugin by calling your `onLoad` and `onUnload` functions.

The `data` argument contains the following properties:
- `persist`: A [Nest](https://github.com/Kyza/nests) that persists your plugin's data
- `id`: The plugin's ID (The URL it loaded from)


## Using persistent data
Cumcord uses [Nests](https://github.com/Kyza/nests) to provide your plugin with a `persist` Nest that persists data between sessions.
The simplest way of using persistent data is to use the `persist.store` properties of the `data` argument.

```js
export default (data) => {
  return {
    onLoad() {
      const store = data.persist.store;
      // Refer to data.persist.store to store data persistently
      store.my.data.goes.here = "This data will persist between reloads!";
    },
    onUnload() {}
  }
}
```

`data.persist` is full Nest object, so I suggest you read the [Nests documentation](https://github.com/Kyza/nests) to learn more about how to use it.

The maximum size of your persistent data is the size of the user's disk.

## Building a Plugin
Plugins need to be built into single-file bundles before they can be used. `sperm build` will automatically do this for you and create a `dist/` directory with the built plugin. Do not edit the built plugin, as it will be overwritten. Instead, make changes to the plugin's source files and run `sperm build` again.

## Testing a Plugin locally
Testing a plugin locally can be done with `sperm dev` on Discord desktop. This will connect to Cumcord's development websocket and reload your plugin when you make changes.

Please note that you must enable developer mode with `cumcord.dev.toggleDevMode()` before you can use `sperm dev`.

If you want to access your plugin's settings modal while developing, you can use `cumcord.dev.showSettings()`.

You can access your plugin's persist nest from `cumcord.dev.storage`.

`sperm dev` does not automatically rebuild your plugin on disk, so you'll need to run `sperm build` again once you're done testing.

## Distributing a Plugin
Distributing plugins is as simple as statically hosting your plugin's `dist/` directory and publishing a link to it. Updating plugins is as simple as building a new version and hosting the new `dist/` directory at the same URL.

Static hosting platforms I recommend:
- GitHub Pages
- Netlify
- Vercel (The site this guide is hosted on!)

If you'd like to get your plugin on the [Cumdump](https://dump.cumcord.com) you can join the Cumcord Discord and ask a Cumdump manager to put it there.

## Cumcord Websocket API
Cumcord has a built in websocket API that allows you to trigger plugin installation from the web.

Cumcord's websocket API **only works on Discord desktop** and hooks into Discord's RPC websocket. As such, Cumcord's websocket runs on the same port as Discord's websocket. Discord's RPC websocket port range is 6463 - 6472, however if this changes in the future and these docs aren't updated you can see their current port range [here](https://discord.com/developers/docs/topics/rpc#rpc-server-ports).

Accessing the websocket API is as simple as connecting to `ws://127.0.0.1:[PORT]/cumcord`. If you forget the `/cumcord`, you will end up connecting to Discord's RPC websocket.

### Using the websocket API <!-- {docsify-ignore} -->
The websocket API is a simple JSON-based protocol.
You must send a JSON object with the following properties:
- `action`: The action to perform.
- `uuid`: A unique ID for the request to allow you to match the response to the request.

Cumcord will respond with a JSON object with the following properties:
- `uuid`: The same ID as the request.
- `status`: The status of the request, either `OK` or `ERROR`.
- `message`: A message describing the error if `status` is `ERROR`.

### install_plugin <!-- {docsify-ignore} -->
`install_plugin` will prompt Cumcord to install a plugin.
`install_plugin` takes the following properties:
- `url`: The URL of the plugin to install.

### update_plugin_dev <!-- {docsify-ignore} -->
`update_plugin_dev` will prompt Cumcord to load a dev plugin from http://127.0.0.1:42069.

`update_plugin_dev` will not work until development mode is toggled via `cumcord.dev.toggleDevMode()`.

## Importing Cumcord APIs
Cumcord plugins can import Cumcord APIs through the `import` keyword with the alias `@cumcord`.

For example, the following plugin imports Cumcord's logger and prints a message to the console:
```js
import { log } from '@cumcord/utils/logger';

export default (data) => {
  return {
    onLoad() {
      log("I've been loaded!");
    },
    onUnload() {
      log("I've been unloaded!");
    }
  }
}
```

These are always equal to the same apis on the cumcord global
```js
import { findByProps } from "@cumcord/modules/webpack";
findByProps === cumcord.modules.webpack.findByProps;
import patcher from "@cumcord/patcher"
patcher === cumcord.patcher;
```

You can also import the nest from your plugin specific data:
```js
import { persist } from "@cumcord/pluginData";
```

Imports like this are heavily preferred over accessing the cumcord global,
but we will use cumcord.* in these docs for the sake of conciseness.

### Static file imports <!-- {docsify-ignore} -->
Cumcord plugins can import files statically for use by appending `:static` to a file's path.

```js
import fileContent from "./file.txt:static";
```

## Interfacing with Discord
Cumcord provides an API to get Discord's internal modules.
- `cumcord.modules.webpack` provides functions for searching through Discord's internal modules.
- `cumcord.modules.common` provides a list of common Discord modules such as React, ReactDOM, etc.
- `cumcord.modules.internal` provides Cumcord's npm dependencies such as `idbKeyval` and `nests`.

Typically, you'll use `cumcord.modules.webpack.findByProps` to find basic Discord internal functions such as `getUser()`, and `cumcord.modules.webpack.findByDisplayName` to find React components such as `Markdown`.

Internal functions typically use camelCase (likeThis), and React components typically use PascalCase (LikeThis).

## Patching
Cumcord provides an API for patching things at `cumcord.patcher`.

### cumcord.patcher.injectCSS <!-- {docsify-ignore} -->
`cumcord.patcher.injectCSS` injects CSS styles to the DOM and returns a function for modifying them. Calling this function with a string will replace the current CSS styles with the new styles, and calling it with `null` will remove the current styles.

```js
let injectedCSS;

// Add styles
injectedCSS = cumcord.patcher.injectCSS(`
  .vizality {
    display: none;
  }
`);

// modify styles to something different
// replaces old styles
injectedCSS(`
  .vizality > * {
    color: red;
    background-color: red;
  }
`);

// Remove styles
injectedCSS();
```

Importing a CSS, SASS, or SCSS file will return a function that injects the css in that file.
```js
import cssInject from "./styles.css";

let uninjectCss;

// Inject styles
uninjectCss = cssInject();

// Remove styles
uninjectCss();
```

### cumcord.patcher.before <!-- {docsify-ignore} -->
`cumcord.patcher.before` injects into a function before it is finished running and passes an array with the patched function's arguments to a callback function.

```js
let patched;

function exampleFunction(arg1, arg2) { console.log(arg2) };

// Currently, this function will log "there" to the console. Let's patch it so it also logs "hi"!
exampleFunction("hi", "there");

/*
  The first argument is the name of the function to patch as a string, the second is the parent object, and the third
  is a function to be called before the function has finished running.
*/
patched = cumcord.patcher.before("exampleFunction", window, (arguments) => { console.log(arguments[0]) });

/*
  This function will now log the following:
  hi
  there
*/
exampleFunction("hi", "there");

// This removes the patch.
patched();

// This now logs "there" again.
exampleFunction("hi", "there");
```

If you return an array in your callback function, Cumcord will use the array's contents as the arguments to the patched function.


### cumcord.patcher.after <!-- {docsify-ignore} -->
`cumcord.patcher.after` injects into a function after it is finished running and passes an array with the patched function's arguments to a callback function as well as the function's return value.

```js
let patched;

function exampleFunction() { return "hi" };

// Currently, this logs "hi" to the console. Let's patch it so it logs "hi there :)" instead.
console.log(exampleFunction());

patched = cumcord.patcher.after("exampleFunction", window, (arguments, returnValue) => { return `${returnValue} there :)` });

// This now logs "hi there :)"
console.log(exampleFunction());

// This removes the patch.
patched();

// This now logs "hi" again.
console.log(exampleFunction());
```

Returning anything in your callback function will override the return value of the patched function.

### cumcord.patcher.instead <!-- {docsify-ignore} -->
`cumcord.patcher.instead` replaces a function with a new function and passes an array with the patched function's arguments to a callback function as well as the original function itself.

```js
let patched;

function exampleFunction() { console.log("hello"); };

// Currently, this function logs "hello" to the console. Let's patch it so it logs "goodbye" instead.
exampleFunction();

patched = cumcord.patcher.instead("exampleFunction", window, (arguments, originalFunction) => { console.log("goodbye") });

// This now logs "goodbye".
exampleFunction();

// This removes the patch.
patched();

// This now logs "hello" again.
exampleFunction();
```

### cumcord.patcher.findAndPatch <!-- {docsify-ignore} -->
`cumcord.patcher.findAndPatch` solves the problem of lazy-loaded components.

This provides an easy method to patch a component as soon as it is available:
```js
// traditional way
const MessageContextMenu = cumcord.webpack.findByDisplayName("MessageContextMenu", false)
const unpatch = cumcord.patcher.after("default", MessageContextMenu, () => {});
// ERROR!!!! - MessageContextMenu may be `undefined`

// new way
const unpatch = cumcord.patcher.findAndPatch(
  () => cumcord.webpack.findByDisplayName("MessageContextMenu", false),
  (MessageContextMenu) => cumcord.patcher.after("default", MessageContextMenu, () => {});
); // works!
```


All of these patcher functions are powerful and will likely be used frequently in your plugins.

## UI Elements
Cumcord provides multiple APIs for managing UI elements, but under the hood they all use React.

Cumcord plugins can use .jsx files to create React components.

JSX files get the `React` object in them automatically for free,
and sperm will replace imports from `"react"` to make npm packages work,
but it is preferred to explicitly import from cumcord if you need react apis, as follows:

```js
import { React } from "@cumcord/modules/common";
```

### cumcord.ui.toasts.showToast <!-- {docsify-ignore} -->
`cumcord.ui.toasts.showToast` shows a toast message, removes it after a set period of time, and returns a function that removes it immediately.

The function takes an object with the following properties:
- `content`: The content of the toast. This can be a React component.
- `title`: The title of the toast. This can *also* be a React component.
- `duration`: The duration to show the toast in milliseconds.
- `class`: An html class to use for the toast.
- `onClick`: A function to call when the toast is clicked.

```js
// The duration is in milliseconds to allow fine control of your toasts.
cumcord.ui.toasts.showToast({
  title: "Hello!",
  content: "Hello there!",
  duration: 3000
});
```

### cumcord.ui.modals.showConfirmationModal <!-- {docsify-ignore} -->
`cumcord.ui.modals.showConfirmationModal` shows a confirmation modal, and returns a promise that resolves to a boolean.

This function takes an object and a callback function. The object contains the following properties:
- `header`: A header for the modal.
- `content`: A description for the modal.
- `confirmText`: The text for the confirm button.
- `cancelText`: The text for the cancel button.
- `type`: The type for the confirm button. Can be `danger`, `confirm`, or `neutral`.

```js
let confirmed = await cumcord.ui.modals.showConfirmationModal({
  header: "Are you sure?",
  content: "This will delete your account.",
  confirmText: "Delete",
  type: "danger"
});
```

## Commands
Sometimes, you'll want to add a command to Discord. Cumcord provides a command API for this.

### cumcord.commands.addCommand <!-- {docsify-ignore} -->
`cumcord.commands.addCommand` adds a client-side commmand to Discord and returns a function that removes the command.

```js
const removeCommand = cumcord.commands.addCommand({
  name: "example",
  description: "An example command.",
  args: [
    {
      name: "arg1",
      description: "An example argument.",
      type: "string" // This can be string, bool, user, channel, or role. If you don't specify, it defaults to string.
      required: false // You can specify whether an arg is required or not. If you don't specify, it defaults to true.
    }
  ],
  handler: (ctx, send) => { // This is the function that gets called when the command is run. It can be an async function.
    // ctx is an object that contains 3 properties, args, guild, and channel.
    ctx.args.arg1; // The value of the first argument. If the arg is not provided, it will be undefined.

    ctx.guild; // The guild the command was run in. This is a standard Discord guild object, and you can find out more about it by console.logging it.

    ctx.channel; // The channel the command was run in. This is a standard Discord channel object, and you can find out more about it by console.logging it.

    send("Hello world!") // This sends a client-side message to the channel the command was run in.

    return "Hello world!" // Returning a string will send that string as a message to the channel the command was run in. Returning nothing will not send anything.
  }
})

removeCommand() // This removes the command.
```

## Plugin settings
Cumcord plugins can export a React component to be used as a settings panel.

```jsx
export default () => {
  return {
    onLoad(),
    onUnload(),
    settings: YourComponent
  }
}
```

The settings prop can be one of the following:
 - A react component (`settings: YourComponent`)
 - A react element (`settings: <YourComponent />`)
 - An array of react **ELEMENTS** (`settings: [<YourComponent />, <YourOtherComponent />]`)

A component is the preferred way to use settings.

## Finding internal Discord functions
Usually, finding internal Discord functions is fairly difficult, given that Discord's internal modules are not documented.

Cumcord currently provides many useful APIs for finding Discord internal functions, under `cumcord.modules.webpack`.

### Webpack searching: findByProps  <!-- {docsify-ignore} -->
`cumcord.modules.webpack.findByProps` finds a module based on the props exported on it.

For example, to find the component with the `getGuild` function:
```js
const { getGuild } = cumcord.modules.webpack.findByProps("getGuild")
getGuild("824921608560181258") // the Cumcord guild!
```

You can also pass multiple props that must exist: `findByProps("getGuilds", "getGuildCount")`.

You can get all modules that match instead of the first one with `findByPropsAll`.

### Webpack searching: findByDisplayName  <!-- {docsify-ignore} -->
`cumcord.modules.webpack.findByDisplayName` finds a module based on the display name of its default export.

This is most useful for finding react components:
```js
const FormText = cumcord.modules.webpack.findByDisplayName("FormText");
<FormText />
```

You can also pass `false` as the second argument to get the parent for patching with
```js
const FormText = cumcord.modules.webpack.findByDisplayName("FormText", false);
const unpatch = cumcord.patcher.after("default", FormText, () => {});
```

You can find all modules with that display name instead of only the first with `findByDisplayNameAll`,
though that does not support finding the parent of the module for patching.

### Webpack searching: findByKeywordAll  <!-- {docsify-ignore} -->
`cumcord.modules.webpack.findByKeywordAll` can be helpful in finding modules that you don't know the exact props of.

It searches for any props in the module that case insensitively match a set of keywords.
For example, for all modules mentioning the user, you can:
```js
const matches = cumcord.modules.webpack.findByKeywordAll("user");
matches.length === 219 // as of the time of writing :)
```

**findByKeywordAll is not efficient, do NOT use this in a finished plugin, its for development use**

### Webpack searching: findByStrings  <!-- {docsify-ignore} -->
`cumcord.modules.webpack.findByStrings` finds a module based on strings it may contain.
This function is VERY inefficient but almost guaranteed to find a match of your string.

It should never be needed except for development, and where it may be needed (notable example: zustand),
you should instead write a much more performant custom find filter

Custom find filters will be explained more in the [webpack](/webpack/) page.

**THIS IS ACTUALLY REALLY SLOW, NEVER EVER EVER USE THIS IN A PLUGIN**