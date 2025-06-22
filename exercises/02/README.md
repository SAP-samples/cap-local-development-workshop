# Exercise 02 - configuration profiles, more on initial data, and the cds REPL

The [profile] concept is a great way to organize different collections of configuration. There are some built-in profiles named "production", "development" and (in one particular context) "hybrid" but we are free to use profiles in whatever way we choose. They can help us manage our local development in many ways; in this exercise we'll extend our look at initial data and use that to explore profiles. We'll also take a first look at the cds REPL.

> Throughout this exercise keep the `cds watch` process running and in its own terminal instance; if necessary, open a second terminal to run any other commands you need, so you've always got the `cds watch` process running and visible.

## Modify the data organization

From the previous exercise, here's what we have. The initial and sample data looks like this:

```text
db
├── data
│   ├── sap.capire.bookshop-Authors.csv
│   ├── sap.capire.bookshop-Books.csv
│   ├── sap.capire.bookshop-Books_texts.csv
│   └── sap.capire.bookshop-Genres.csv
└── schema.cds

test/
└── data
    └── Ex01Service.Sales.csv
```

and the `package.json#cds.requires.db` section, reflecting the persistent file `db.sqlite`, looks like this:

```json
  "cds": {
    "requires": {
      "db": {
        "kind": "sqlite"
      }
    }
  }
```

### Remove the sample data and switch back to in-memory

👉 To keep things simple and keep "noise" to a minimum, remove the sample data entirely and re-deploy, as we don't need it any more:

```bash
rm -rf test/ \
  && cds deploy
```

Now we should switch back to in-memory, mostly so we can more comfortably and immediately see the effects of what we're going to do in this next section (if the database is in-memory then a deployment is done each and every time the server restarts, which makes sense when you think about it).

There are different ways we can switch back to in-memory. Here are a few:

We could just remove the current `package.json#cds.requires.db` configuration entirely.

We could add explicit `credentials` section to the current `package.json#cds.requires.db` configuration, reflecting explicitly the implicit default configuration:

```json
  "cds": {
    "requires": {
      "db": {
        "kind": "sqlite",
        "credentials": {
          "url": ":memory:"
        }
      }
    }
  }
```

> This is where having such an explicit section might have been helpful, as mentioned in the note in the [Deploy to a persistent file] section of the previous exercise.

We could invoke `cds serve all` using the `--in-memory` option without the trailing question mark.

👉 Let's go for the option of adding an explicit `credentials` section; edit the configuration in `package.json` so it looks like the sample just above.

This should cause the CAP server to emit some familiar log lines:

```log
[cds] - connect to db > sqlite { url: ':memory:' }
  > init from db/data/sap.capire.bookshop-Genres.csv
  > init from db/data/sap.capire.bookshop-Books_texts.csv
  > init from db/data/sap.capire.bookshop-Books.csv
  > init from db/data/sap.capire.bookshop-Authors.csv
/> successfully deployed to in-memory database.
```

This reminds us that data is being loaded from CSV files in `db/data/`, according to convention.

### Maintain a separate initial data collection

Sometimes it's useful to maintain and use different starting sets of initial data. You can manage this with the combination of convention (the mechanism looks for `data/` directories directly within the `db/`, `srv/` and `app/` directories and any other referenced locations) and the [profile] concept. Let's try this out.

OK. The name of the `data/` directory is special (see [footnote-1](#footnote-1)), and its relative location is also special; if we move it to somewhere else, the files containing the initial data won't get picked up automatically.

👉 Let's try that now:

```bash
mkdir db/classics/ \
  && mv db/data/ db/classics/
```

At this point, the log output from the restarted CAP server doesn't show any "init from ..." lines, as no initial data was found ... in the expected / default location(s).

But we can tell the CAP server about this "classics" initial data collection and assign a name to it, in the form of a [profile].

👉 Let's do that now, by adding a new node (note the square brackets round "classics") to `package.json#cds.requires` so that it looks like this:

```json
  "cds": {
    "requires": {
      "db": {
        "kind": "sqlite",
        "credentials": {
          "url": ":memory:"
        }
      },
      "[classics]": {
        "initdata": {
          "model": "db/classics/"
        }
      }
    }
  }
```

This doesn't have any positive effect yet; we need a couple more things.

👉 First, we need to add an empty CDS model file in the form of `index.cds` to the new `db/classics/` directory; this is so the CDS model compiler acknowledges this new "classics" directory and treats it as part of the model, including any initial data loading requirements:

```bash
touch db/classics/index.cds
```

👉 Now we need to actually go to the CAP server, stop it (with Ctrl-C) and restart it, specifying this new "classics" name as a profile:

```bash
cds w --profile classics
```

Lo and behold, the initial data in the CSV files in `db/classics/data/` is now loaded!

```log
  > init from db/classics/data/sap.capire.bookshop-Genres.csv
  > init from db/classics/data/sap.capire.bookshop-Books_texts.csv
  > init from db/classics/data/sap.capire.bookshop-Books.csv
  > init from db/classics/data/sap.capire.bookshop-Authors.csv
```

### Add a second initial data collection

To illustrate this technique more fully, let's add a second initial data collection in a similar way. There are a couple of data files for the author Douglas Adams and the books in his (increasingly inaccurately named) [Hitchhiker's Guide To The Galaxy] trilogy.

👉 Create a new "hitchhikers" directory, copy the data in (from this exercise's [assets/] directory) and create an empty `index.cds` file in there too, like this:

```bash
mkdir -p db/hitchhikers/data/ \
  && cp ../exercises/01/assets/data/json/* "$_" \
  && touch db/hitchhikers/index.cds
```

> Note that this time, just to illustrate the possibility, the data files are JSON not CSV. This works too, but JSON files are only supported in development mode.

At this point the `db/` directory should look like this:

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

It's now time to define a further stanza in `package.json#cds.requires` to "require" (effectively _include_) this new "hitchhikers" model too. Like the "classics" model earlier, there isn't actually any more [CDL] to load, but the very fact that there's an `index.cds` there, even an empty one, will cause the data loading mechanism to scoop up anything in any conventional relative `data/` locations.

👉 Add this stanza next to the "classics" one so that the entire `cds` stanza looks like this:

```json
  "cds": {
    "requires": {
      "db": {
        "kind": "sqlite",
        "credentials": {
          "url": ":memory:"
        }
      },
      "[classics]": {
        "initdata": {
          "model": "db/classics/"
        }
      },
      "[hitchhikers]": {
        "initdata": {
          "model": "db/hitchhikers/"
        }
      }
    }
  }
```

👉 Now restart the CAP server using the "hitchhikers" profile:

```bash
cds w --profile hitchhikers
```

and enjoy a different initial data set:

```log
  > init from db/hitchhikers/data/sap.capire.bookshop-Books.json
  > init from db/hitchhikers/data/sap.capire.bookshop-Authors.json
```

Naturally this also works the same way when deploying, for example:

```bash
cds deploy --to sqlite:hitchhikers.db --profile hitchhikers
```

which results in:

```log
  > init from db/hitchhikers/data/sap.capire.bookshop-Books.json
  > init from db/hitchhikers/data/sap.capire.bookshop-Authors.json
/> successfully deployed to hitchhikers.db
```

At this point, only the novels in Douglas Adams' "trilogy" are in that `hitchhikers.db` file:

```log
; sqlite3 hitchkikers.db 'select title from sap_capire_bookshop_Books;'
The Hitchhiker's Guide to the Galaxy
The Restaurant at the End of the Universe
Life, the Universe and Everything
So Long, and Thanks for All the Fish
Mostly Harmless
And Another Thing... (novel)
```

### Combine the features

You can combine this feature with the ability to configure file based persistence in `cds.requires` too.

👉 Add the SQLite persistence info to the "hitchhikers" configuration:

```json
      "[hitchhikers]": {
        "initdata": {
          "model": "db/hitchhikers/"
        },
        "db": {
          "kind": "sqlite",
          "credentials": {
            "url": "hitchhikers.db"
          }
        }
      }
```

Having that `db` entry within the `[hitchhikers]` profile entry will override the "profile-independent" values (whether they're implicit or explicit) when the "hitchhikers" profile is specified via the `--profile` option on relevant `cds` commands.

Feel free to explore this combination if you have time!

<!-- TODO: use cds REPL to dig into the data, using some of the https://cap.cloud.sap/docs/guides/databases-sqlite#path-expressions-filters features -->

## Use the cds REPL to explore path expression features with SQLite

Using SQLite for local development doesn't mean sacrificing database features. The new database services, including the one for SQLite, offer a common set of [features] including all kinds of [path expressions & filters]. This is a good opportunity to try some of these out, directly, interactively, with the [cds REPL].

Before we continue, we need to add the [@cap-js/cds-test] package, which will make it easy for us to have the REPL start CAP servers for us. Add it now as a local development dependency:

```bash
npm add -D @cap-js/cds-test
```

👉 Start up the REPL in its basic form, specifying the "classics" profile (so the initial data in `db/classics/data/` is loaded):

```bash
cds repl --profile classics
```

This should present a simple prompt that looks like this:

```text
Welcome to cds repl v 9.0.4
>
```

👉 At the prompt, ask for help with `.help`, which should show:

```log
.break     Sometimes you get stuck, this gets you out
.clear     Alias for .break
.editor    Enter editor mode
.exit      Exit the REPL
.help      Print this help message
.inspect   Sets options for util.inspect, e.g. `.inspect .depth=1`.
.load      Load JS from a file into the REPL session
.run       Runs a cds server from a given CAP project folder, or module name like @capire/bookshop.
.save      Save all evaluated commands in this REPL session to a file

Press Ctrl+C to abort current expression, Ctrl+D to exit the REPL
```

👉 Get a taste of what's possible and available by using the cds specific command `.inspect` to look at the entire CDS facade at a high level, noting that much of it is [lazily loaded]:

```text
.inspect cds .depth=0
```

This should show the current top level properties of the facade, like this:

```log
cds: cds_facade {
  _events: [Object: null prototype],
  _eventsCount: 2,
  _maxListeners: undefined,
  model: undefined,
  db: undefined,
  cli: [Object],
  root: '/work/scratch/myproj',
  services: {},
  extend: [Function (anonymous)],
  version: '9.0.4',
  builtin: [Object],
  service: [Function],
  log: [Function],
  parse: [Function],
  home: '/work/scratch/myproj/node_modules/@sap/cds',
  env: [Config],
  requires: {},
  Symbol(shapeMode): false,
  Symbol(kCapture): false
}
```

Certain properties in the facade such as `model` and `db` are still `undefined`, because there is no model loaded, no CAP server running and providing services.

👉 Let's change that situation now, and use the `.run` command to comfortably start a CAP server in the context of the REPL, for the current project (pointed to by the `.` symbol, the normal representation for "current directory"):

```text
.run .
```

> You can also use the `--run` option when you invoke `cds repl` to have a CAP server started up in the REPL context immediately.

This should emit the usual CAP server startup log lines, plus something like this:

```log
Following variables are made available in your repl's global context:

from cds.entities: {
  Books,
  Authors,
  Genres,
}

from cds.services: {
  db,
  AdminService,
  CatalogService,
  Ex01Service,
}

Simply type e.g. Ex01Service in the prompt to use the respective objects.
```

We [defined our `Ex01Service` in the simplest way], but we can see in the REPL that there are already plenty of handlers attached.

👉 Have a look at them with `.inspect Ex01Service.handlers`:

```text
.inspect Ex01Service.handlers
```

This should show handlers, grouped by phase, for the service; it's the ones in the `on` phase that provide the built-in handling for CRUD operations:

```text
Ex01Service.handlers: EventHandlers {
  _initial: [
    {
      before: '*',
      handler: [Function: check_service_level_restrictions]
    },
    { before: '*', handler: [Function: check_auth_privileges] },
    { before: '*', handler: [Function: check_readonly] },
    { before: '*', handler: [Function: check_insertonly] },
    { before: '*', handler: [Function: check_odata_constraints] },
    { before: '*', handler: [Function: check_autoexposed] },
    { before: '*', handler: [AsyncFunction: enforce_auth] },
    { before: 'READ', handler: [Function: restrict_expand] },
    { before: 'CREATE', handler: [AsyncFunction: validate_input] },
    { before: 'UPDATE', handler: [AsyncFunction: validate_input] },
    { before: 'NEW', handler: [AsyncFunction: validate_input] },
    { before: 'READ', handler: [Function: handle_paging] },
    { before: 'READ', handler: [Function: handle_sorting] }
  ],
  before: [],
  on: [
    { on: 'CREATE', handler: [AsyncFunction: handle_crud_requests] },
    { on: 'READ', handler: [AsyncFunction: handle_crud_requests] },
    { on: 'UPDATE', handler: [AsyncFunction: handle_crud_requests] },
    { on: 'DELETE', handler: [AsyncFunction: handle_crud_requests] },
    { on: 'UPSERT', handler: [AsyncFunction: handle_crud_requests] }
  ],
  after: [],
  _error: []
}
```

Now it's time to try one of those [CQL] path expressions with infix filters that are supported by all the new database services (including SQLite, HANA and Postgres).

At the REPL prompt, declare a query like this:

```text
yorkshireBooks = SELECT `from ${Books}:author[placeOfBirth like '%Yorkshire%'] {placeOfBirth, books.title as book, name as author }`
```

This should emit the query object that results:

```text
cds.ql {
  SELECT: {
    from: {
      ref: [
        'sap.capire.bookshop.Books',
        {
          id: 'author',
          where: [
            { ref: [ 'placeOfBirth' ] },
            'like',
            { val: '%Yorkshire%' }
          ]
        }
      ]
    },
    columns: [
      { ref: [ 'placeOfBirth' ] },
      { ref: [ 'books', 'title' ], as: 'book' },
      { ref: [ 'name' ], as: 'author' }
    ]
  }
}
```

As well as [stare at] it for a bit, we can also execute it (technically: send it to the default database service). Do that now:

```text
await yorkshireBooks
```

It should return the books from the two Brontë sisters:

```text
[
  {
    placeOfBirth: 'Thornton, Yorkshire',
    book: 'Wuthering Heights',
    author: 'Emily Brontë'
  },
  {
    placeOfBirth: 'Thornton, Yorkshire',
    book: 'Jane Eyre',
    author: 'Charlotte Brontë'
  }
]
```

There's plenty more to explore - see the links in the [Further reading](#further-reading) section below.

---

## Further reading

- [Level up your CAP skills by learning how to use the cds REPL](https://qmacro.org/blog/posts/2025/03/21/level-up-your-cap-skills-by-learning-how-to-use-the-cds-repl/)
- [The Art and Science of CAP](https://qmacro.org/blog/posts/2024/12/06/the-art-and-science-of-cap/)

## Footnotes

<a name="footnote-1"></a>
### Footnote 1

The name can also be `csv/` which is also "special".

[profile]: https://cap.cloud.sap/docs/node.js/cds-env#profiles
[Hitchhiker's Guide To The Galaxy]: https://en.wikipedia.org/wiki/The_Hitchhiker%27s_Guide_to_the_Galaxy
[Deploy to a persistent file]: ../01/README.md#deploy-to-a-persistent-file
[assets/]: assets/
[CDL]: https://cap.cloud.sap/docs/cds/cdl
[features]: https://cap.cloud.sap/docs/guides/databases-sqlite#features
[path expressions & filters]: https://cap.cloud.sap/docs/guides/databases-sqlite#path-expressions-filters
[cds REPL]: https://cap.cloud.sap/docs/tools/cds-cli#cds-repl
[@cap-js/cds-test]: https://github.com/cap-js/cds-test
[lazily loaded]: https://qmacro.org/blog/posts/2024/12/10/tasc-notes-part-4/#lazy-loading-of-the-cds-facades-many-features
[defined our `Ex01Service` in the simplest way]: ../01/README.md#add-a-new-service-definition
[CQL]: https://cap.cloud.sap/docs/cds/cql
[stare at]: https://qmacro.org/blog/posts/2017/02/19/the-beauty-of-recursion-and-list-machinery/#initial-recognition
