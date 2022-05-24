# The Cumcord Websocket

[Websockets](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)
allow symmetric two way communication between a client and server,
as opposed to the traditional request-response model.

Discord uses a websocket on desktop for rich presence,
however Cumcord injects into this and allows adding custom actions to the socket.

## Talking to the websocket

The websocket accepts requests on the `/cumcord` route, e.g. `ws://127.0.0.1:6463/cumcord`,
and then works with objects in json format only.

There should be an `action` key to choose which handler to invoke, and a `uuid` key,
which is bounced back in responses to help tracking in the client.

Responses will fall into three categories:

- Raw: Any object, but guaranteed to have the uuid, and `name: "CUMCORD_WEBSOCKET"`
- Ok: Name and uuid like raw, and `{ status: "OK", message?: any }`
- Err: Name and uuid like raw, and `{ status: "ERROR", message?: any }`

You should use [cummunicate](https://npmjs.com/@cumjar/cummunicate) to talk to the WS,
if you are connecting from a website.

## Registering websocket handlers

You may add a handler for a WS action with `cumcord.websocket.addHandler`

It takes two props: the name of the action, and a handler function.

If the action is taken, it will throw.

Like patches, the function to remove the handler is returned.

```ts
function addHandler(action: string, handler: HandlerFunc): () => void;
```

### The handler function

It takes two args:

- `msg`: The message object received by CC. It is an object, not a JSON string.
- `{ raw, ok, error }`: A set of functions to send replies back, see above.

**YOU _MUST_ USE `cumcord.ui.modals.showConfirmationModal` FOR USER CONFIRMATION OF ACTIONS.**

## An end-to-end example

This is a sample WS handler and invoker for a theme loader plugin to install a theme.

Note: this is just a sample, but you can see a real-world example of a
[invoker](https://github.com/yellowsink/stain-send/blob/bb6237a6046f7773db12344edcc38213112c8a89/index.html#L104)
and
[handler](https://github.com/yellowsink/cc-plugins/blob/master/plugins/cumstain/patches/exposeWs.js)
at these links.

```js
// CC PLUGIN
import { addTheme, themeIsInstalled } from "./util.js";
import { addHandler } from "@cumcord/websocket";
import { showConfirmationModal } from "@cumcord/ui/modals";

export default () =>
  addHandler("install_theme", (msg, { ok, error }) => {
    if (!msg?.url) error("You must pass a theme url");
    else if (themeIsInstalled(msg.url)) error("That theme is installed");
    else
      showConfirmationModal(
        {
          header: "Add theme?",
          content: `A website wants to add the theme \`${msg.url}\`. Allow?`,
        },
        (res) => {
          // installTheme is async
          if (res) installTheme(msg.url).then(ok, error);
          else error("User declined");
        }
      );
  });
```

```js
// BROWSER
import cummunicate from "https://cdn.esm.sh/@cumjar/cummunicate";

console.log(await cummunicate("install_theme", { url: "https://path/to/theme.css" }));
/* 
[
    [6463, { name: "CUMCORD_WEBSOCKET", uuid: "<random>", status: "OK" }],
    [6464, { name: "CUMCORD_WEBSOCKET", uuid: "<random>", status: "ERROR", message: "User declined" }],
    [6465, { name: "CUMCORD_WEBSOCKET", uuid: "<random>", status: "ERROR", message: "That theme is installed" }],
]
where the discord on port 6463 installed fine,
the user of the discord on port 6464 declined
and the discord on port 6465 already had the theme
no other discord instances were found.
*/
```

## Usage on web and custom handler triggers

On web, the addHandler API works as usual.

The handler can be invoked as if a "ws" message was received with
`cumcord.websocket.triggerHandler`, with two args:
 - The message, MUST BE JSON ENCODED
 - A callback function, which is passed any "sent" messages, json encoded.

The browser extension to inject CC, CumLoad, is currently experimenting with its own
IPC implementation that websites can use to trigger the websocket handlers in instances
of Cumcord running in other tabs in the same browser.
Once this is ready, it will be supported by Cummunicate.