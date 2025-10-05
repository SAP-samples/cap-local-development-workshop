# Exercise 04 - a first look at local messaging and events

In [The Art and Science of CAP] series Daniel Hutzel shared with us many of the
influences that informed CAP's design, and explained in great detail some of
the core axioms. There are key philosophical truths that are inherent in what
CAP is, two of which are:

- [Everything is a service]
- [Everything is an event]

In this exercise we'll explore [events and messaging] in CAP, and in
particular, the facilities in this space that are available to us when
developing locally.

The general idea, as you might expect, is that CAP's eventing is agnostic at
the definition and API level; the actual mechanism used to manage the sending,
receiving, queueing and relaying of messages is an implementation and platform
context detail.

Whether the "message channel" facilities are provided by SAP Cloud Application
Event Hub, SAP Event Mesh, or another mechanism, is largely irrelevant from a
developer perspective, especially in a local context, where, in addition to an
in-process facility, [file-based messaging] is available and the main focus of
this exercise.

## Take a quick look at in-process eventing

Like [in-process mocking of required services], CAP supports in-process
eventing. It's worth taking a brief look here, not least to have the chance to
practice embracing the cds REPL, and to see how simple things are.

### Start the cds REPL

👉 Before you begin, stop any CAP server processes for now. Then launch the
REPL:

```bash
cds repl
```

👉 At the prompt, define a simple service with a handler for a
"widget-produced" event:

```javascript
(srv = new cds.Service).on('widget-produced', x => console.log('Received:', x))
```

This should emit the basic `Service` definition just created, which includes
the `on` phase handler:

```javascript
Service {
  name: 'Service',
  options: {},
  handlers: EventHandlers {
    _initial: [],
    before: [],
    on: [ { on: 'widget-produced', handler: [Function (anonymous)] } ],
    after: [],
    _error: []
  },
  definition: undefined
}
```

👉 Now emit a few events, with a simple data payload:

```javascript
['small', 'medium', 'large'].forEach(size => srv.emit('widget-produced', { size: size }))
```

Each of the three events emitted reach the recipient causing this to be shown:

```log
Received: EventMessage { event: 'widget-produced', data: { size: 'small' } }
Received: EventMessage { event: 'widget-produced', data: { size: 'medium' } }
Received: EventMessage { event: 'widget-produced', data: { size: 'large' } }
```

This is a simple example of in-process eventing - emission, transmission,
receipt and handling of messages happened in the same process. It can be as
simple as that.

👉 Exit the REPL session.

> For more info on using the cds REPL, see the [Further
> reading](#further-reading) section below.

## Explore file-based messaging

This is what our Ex01Service definition looks like right now, from
`srv/ex01-service.cds`:

```cds
using { sap.capire.bookshop as my } from '../db/schema';
@path: '/ex01' service Ex01Service {
  entity Books as projection on my.Books;
}

annotate Ex01Service with @requires: 'authenticated-user';
annotate Ex01Service.Books with @restrict: [
  { grant: 'READ' },
  { grant: 'WRITE', to: 'backoffice' }
];
```

and `srv/ex01-sales.cds`:

```cds
using { cuid } from '@sap/cds/common';
using { Ex01Service } from './ex01-service';

extend service Ex01Service with {
  entity Sales : cuid {
    date: Date;
    book: Association to Ex01Service.Books;
    quantity: Integer;
  }
}
```

Let's define an event, and emit it.

### Switch the classics data back to the default development profile

Before we start, and to keep things simple, let's switch back the authors,
books and genres data from within the classics profile back to the default,
i.e. let's move:

```text
db
├── classics
│   ├── data
│   │   ├── sap.capire.bookshop-Authors.csv
│   │   ├── sap.capire.bookshop-Books.csv
│   │   ├── sap.capire.bookshop-Books_texts.csv
│   │   └── sap.capire.bookshop-Genres.csv
│   └── index.cds
├── hitchhikers
│   ├── data
│   │   ├── sap.capire.bookshop-Authors.json
│   │   └── sap.capire.bookshop-Books.json
│   └── index.cds
└── schema.cds
```

to be:

```text
db
├── data
│   ├── sap.capire.bookshop-Authors.csv
│   ├── sap.capire.bookshop-Books.csv
│   ├── sap.capire.bookshop-Books_texts.csv
│   └── sap.capire.bookshop-Genres.csv
├── hitchhikers
│   ├── data
│   │   ├── sap.capire.bookshop-Authors.json
│   │   └── sap.capire.bookshop-Books.json
│   └── index.cds
└── schema.cds
```

👉 Do this now:

```bash
mv db/classics/data/ db/ && rm -rf db/classics/
```

👉 Also, to keep things clean, remove the corresponding entry in
`package.json#cds.requires`, and stay in the file as we'll be adding something
in the next part:

```text
  "cds": {
    "requires": {
      "db": {
        "kind": "sqlite",
        "credentials": {
          "url": ":memory:"
        }
      },
      "[classics]": {                     -+
        "initdata": {                      |
          "model": "db/classics/"          | remove this
        }                                  |
      },                                  -+
      "[hitchhikers]": {
        "initdata": {
          "model": "db/hitchhikers/"
        }
      },
      "northbreeze": {
        "kind": "odata",
        "model": "srv/external/northbreeze"
      }
    }
  }
```

### Declare a requirement for file-based messaging

OK, the first thing we need to do is define a requirement for messaging. And
for our local development scenario, we should use file-based messaging, which
is the default.

👉 Add a `messaging` section within `package.json#cds.requires`, so it looks
like this:

```json
  "cds": {
    "requires": {
      "db": {
        "kind": "sqlite",
        "credentials": {
          "url": ":memory:"
        }
      },
      "[hitchhikers]": {
        "initdata": {
          "model": "db/hitchhikers/"
        }
      },
      "northbreeze": {
        "kind": "odata",
        "model": "srv/external/northbreeze"
      },
      "messaging": true
    }
  }
```

> We could also have been more explicit, specifying the value for `messaging` like this:
>
> ```json
> {
>   "messaging": {
>     "kind": "file-based-messaging"
>   }
> }
> ```

### Define an event

In the previous exercise we created the "milton" user and gave them the
"backoffice" role which allowed them to perform `WRITE` semantic operations on
books. Let's define an event that should be emitted when a book is deleted.

👉 First, declare that by adding a [custom event definition] "bookremoved" to
the service in `srv/ex01-service.cds`:

```cds
...
@path: '/ex01' service Ex01Service {
  entity Books as projection on my.Books;
  event bookremoved: { ID: Books:ID; }     // <---
}
...
```

The type structure (`{ ... }`) is deliberately as compact as possible for this
example, designed to convey just the ID of the book that was removed.

### Add handler code to emit the event

Now it's time to define when and how that event should be emitted. Let's start
simple, with a temporary `console.log` statement.

👉 Create `srv/ex01-service.js` with the following content:

```javascript
const cds = require('@sap/cds')

class Ex01Service extends cds.ApplicationService { init() {

  this.after (['DELETE'], 'Books', (_, req) => {
    console.log('bookremoved', req.data)
  })

  return super.init()

}}

module.exports = Ex01Service
```

This defines an "after" phase handler for DELETE events (yes, let's use the
word "event" here too) relating to the `Books` entity. The signature of an
[after handler] is such that the first parameter is the data relating to the
event, and the second parameter is the request object. In the request object
there's the data; let's have a look at what that is in this context.

👉 Start up the CAP server again, like this:

```bash
cds w
```

Just out of interest, notice in the log output that instead of the built-in
service implementation:

```log
[cds] - serving Ex01Service {
  impl: 'node_modules/@sap/cds/libx/_runtime/common/Service.js',
  path: '/ex01'
}
```

our new custom implementation is now in play for the Ex01Service:

```log
[cds] - serving Ex01Service { impl: 'srv/ex01-service.js', path: '/ex01' }
```

While we're looking at the log output from the CAP server, we can also see that
the "file-based-messaging" channel is active, owing to the `messaging` entry we
added to `package.json#cds.requires`:

```log
[cds] - connect to messaging > file-based-messaging
```

👉 Now, in another terminal session, delete "The Raven":

```bash
curl -X DELETE -u milton:dontmovemydesk localhost:4004/ex01/Books/251
```

In the CAP server log, we should see:

```log
[odata] - DELETE /ex01/Books/251
bookremoved { ID: 251 }
```

Great - the `req.data` is exactly what we need for the event payload.

👉 Now adjust the code in the "after" phase handler, changing:

```javascript
console.log('bookremoved', req.data)
```

to:

```javascript
this.emit('bookremoved', req.data)
```

👉 At this point, the CAP server should have restarted; if not, give it a nudge
by heading over to the terminal session in which it's running and pressing
Enter.

### Start monitoring the message channel

The message channel we're using here for this local context is [file-based
messaging]. Event messages are stored and queued in a file. Where is that file?
It's called `.cds-msg-box` and sits alongside another local development related
file (`.cds-services.json`) in your home directory.

> If a CAP file is in your home directory, it's a big clue that it's for local
> development only.

👉 In a separate terminal session, ensure the file exists and start monitoring
the contents:

```bash
touch ~/.cds-msg-box \
  && tail -f $_
```

### Delete a book

The moment of truth is upon us!

👉 In the terminal session where you recently deleted "The Raven", bring up the
command again and resend the request:

> Because the CAP server is running with an in-memory SQLite database, we
> benefit from the book data being restored on each restart.

```bash
curl -X DELETE -u milton:dontmovemydesk localhost:4004/ex01/Books/251
```

### Observe the event message

While the CAP server log emits the usual:

```log
[odata] - DELETE /ex01/Books/251
```

we also now see that something has been written to our file-based messaging
store:

```log
Ex01Service.bookremoved {"data":{"ID":251},"headers":{"x-correlation-id":"c72e2a47-2faf-4bcb-9f54-37ebb2ca88a6"}}
```

But what happens now? How is such a message subsequently received? We'll find
out with a more comprehensive example in the next exercise where we also learn
how to manage a larger scale CAP project, with independent services, locally.

---

## Further reading

- [Level up your CAP skills by learning how to use the cds REPL]

---

[Next exercise](../05)

---

[The Art and Science of CAP]: https://qmacro.org/blog/posts/2024/12/06/the-art-and-science-of-cap/
[Everything is a service]: https://qmacro.org/blog/posts/2024/12/10/tasc-notes-part-4/#everything-is-a-service
[Everything is an event]: https://qmacro.org/blog/posts/2024/11/07/five-reasons-to-use-cap/#:~:text=Everything%20is%20an%20event
[events and messaging]: https://cap.cloud.sap/docs/guides/messaging/
[in-process mocking of required services]: ../03/README.md#footnote-1
[file-based messaging]: https://cap.cloud.sap/docs/guides/messaging/#_1-use-file-based-messaging-in-development
[custom event definition]: https://cap.cloud.sap/docs/cds/cdl#events
[after handler]: https://cap.cloud.sap/docs/node.js/core-services#srv-after-request
[Level up your CAP skills by learning how to use the cds REPL]: https://qmacro.org/blog/posts/2025/03/21/level-up-your-cap-skills-by-learning-how-to-use-the-cds-repl/
