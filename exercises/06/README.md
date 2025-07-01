# Exercise 06 - debugging local and remote servers

The [Debugging] section of Capire tells us that with `cds debug` we can "_debug applications running locally or remotely on SAP BTP Cloud Foundry. Local applications will be started in debug mode, while (already running) remote applications are put into debug mode._".

The benefit of using the same procedure and in fact the same debugging tools regardless of whether the CAP server is local or remote is enormous. We can use our local development tools and the same techniques, and while connected to a remote server we can still remain "local" in our minds.

In this exercise we'll create a simple CAP project, run and debug it locally, then deploy it to Cloud Foundry, connect to it and debug it remotely.

## Initialize a new CAP Node.js project

For this topic we'll limit ourselves to a simple CAP service.

👉 In a new terminal session window, initialize a new CAP Node.js project "debugtest" in the workshop root directory:

```bash
cd /workspaces/cap-local-development-workshop/ \
  && cds init --add tiny-sample debugtest && cd $_
```

The "tiny-sample" facet causes a super small books based service with a couple of data records, and it relies on the built-in service implementation as there are no JavaScript files alongside the service definition, as we can see:

```bash
; tree
.
├── app
├── db
│   ├── data
│   │   └── my.bookshop-Books.csv
│   └── schema.cds
├── eslint.config.mjs
├── package.json
├── README.md
└── srv
    └── cat-service.cds
```

👉 So that we have something simple to which we can attach a breakpoint when debugging, add a `srv/cat-service.js` file with this content:

```javascript
const cds = require('@sap/cds')
module.exports = cds.service.impl(function() {
    this.after('each', 'Books', book => {
      console.log(book)
    })
})
```

When debugging, we'll set a breakpoint on the `console.log(book)` line shortly.

## Try out debugging locally

We're already all set - it's straightforward.

### Start the CAP server in debug mode

👉 Start debugging the service with `cds debug`, which as we'll see from the output is just shorthand for `cds watch --debug`:

```bash
cds debug
```

The output contains information relevant for our debugging session, but otherwise the CAP server is operating pretty much the same way as it does normally when started with `cds watch`:

```log
Starting 'cds watch --debug'

cds serve all --with-mocks --in-memory?
( live reload enabled for browsers )

        ___________________________

Debugger listening on ws://127.0.0.1:9229/2f95339d-33e9-4b40-9e24-461d1a75cc6c
For help, see: https://nodejs.org/en/docs/inspector
...

[cds] - serving CatalogService { impl: 'srv/cat-service.js', path: '/odata/v4/catalog' }

[cds] - server listening on { url: 'http://localhost:4004' }
```

The difference is that the Node.js process has been started with the `--inspect` option. See the link to the Node.js debugging content in the [Further reading](#further-reading) section.

Note the websocket address given: `ws://127.0.0.1:9229/<process-guid>`.

That port 9229 is the default for Node.js debugging connections; let's see for ourselves that it's listening for connections:

```bash
netstat -atn | grep LISTEN
```

This should elicit output like this:

```log
tcp        0      0 127.0.0.1:33791         0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:9229          0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:2222            0.0.0.0:*               LISTEN
tcp6       0      0 :::4004                 :::*                    LISTEN
tcp6       0      0 :::2222                 :::*                    LISTEN
tcp6       0      0 ::1:35729               :::*                    LISTEN
```

In this list we can see the `127.0.0.1:9229` socket, as well as the `<any-host-address>:4004` representing the socket which is ready to accept incoming connections to the OData service served by the CAP server.

> An IP address or hostname combined with a port number is commonly referred to as a "socket".

### Start a debugging client

There are various debugging clients generally but the "classic" for this type of Node.js debugging is the Chrome Developer Tools, specifically the "Inspector".

👉 In a new tab in your Chrome (or Chromium) browser, go to address `chrome://inspect` where you should see something like this:

![Chrome DevTools Inspector showing a remote process debug target](assets/devtools-remote-target-list.png)

This shows a single debugging target ready to be attached to and inspected. From the detail we can see it's our CAP server.

> Don't be thrown by the fact that this target is listed in the "Remote Target" section; from the DevTools Inspector point of view, all these websocket based targets are "remote".

### Attach the inspector to the target

👉 Select the "inspect" link next to the target.

You should see this in the CAP server log:

```log
Debugger attached.
```

You should also be presented with an Inspector window that, if you've used DevTools in Chrome before, should be familiar:

![Chrome DevTools Inspector attached](assets/devtools-inspector-attached.png)



![The inspector](assets/







---

## Further reading

- The [Debugging] topic in Capire
- [Node.js debugging]
- [Debugging JavaScript with Chrome DevTools]

[Debugging]: https://cap.cloud.sap/docs/tools/cds-cli#cds-debug
[Node.js debugging]: https://nodejs.org/en/learn/getting-started/debugging
[Debugging JavaScript with Chrome DevTools]: https://developer.chrome.com/docs/devtools/javascript
