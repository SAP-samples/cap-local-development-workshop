# Exercise 02 - configuration profiles, more on initial data, and the cds REPL

The [profile] concept is a great way to organize different collections of configuration. There are some built-in profiles named "production", "development" and (in one particular context) "hybrid" but we are free to use profiles in whatever way we choose. They can help us manage our local development in many ways; in this exercise we'll extend our look at initial data and use that to explore profiles. We'll also take a first look at the cds REPL.

> Throughout this exercise keep the `cds watch` process running and in its own terminal instance; if necessary, open a second terminal to run any other commands you need, so you've always got the `cds watch` process running and visible.

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

## Remove the sample data and switch back to in-memory

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

## Maintain a separate initial data collection

Sometimes it's useful to maintain and use different starting sets of initial data. You can manage this with the combination of convention (the mechanism looks for `data/` directories directly within the `db/`, `srv/` and `app/` directories and any other referenced locations) and the [profile] concept. Let's try this out.

OK. The name of the `data/` directory is special (see [footnote-6](#footnote-6)), and its relative location is also special; if we move it to somewhere else, the files containing the initial data won't get picked up automatically.

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

---

## Footnotes

<a name="footnote-1"></a>
### Footnote 1

They are already exposed but only in the `AdminService` which is annotated to protect it, with:

```cds
service AdminService @(requires:'admin') { ... }
```

(in `srv/admin-service.cds`). Yes, we can embrace the mock authentication:

```bash
curl -s -u 'alice:' localhost:4004/odata/v4/admin/Books
```

But to be honest there's another reason, which is that they're also annotated (in `app/admin-books/fiori-service.cds`) as being draft-enabled:

```cds
annotate sap.capire.bookshop.Books with @fiori.draft.enabled;
```

This adds a second key (`isActiveEntity`) to the entity, and we don't want to get into that at this early stage.

<a name="footnote-2"></a>
### Footnote 2

Note the `--in-memory?` option in the expanded version of `cds w` which is `cds serve all --with-mocks --in-memory?`. The meaning of the question mark is important here - this is what the help says for the option:

```text
    Automatically adds a transient in-memory database bootstrapped on
    each (re-)start in the same way cds deploy would do, based on defaults
    or configuration in package.json#cds.requires.db. Add a question
    mark to apply a more defensive variant which respects the configured
    database, if any, and only adds an in-memory database if no
    persistent one is configured.
```

<a name="footnote-3"></a>
### Footnote 3

There is [no particular strict convention for SQLite database filename extensions]; choosing `.db` or `.sqlite` are decent choices though.

<a name="footnote-4"></a>
### Footnote 4

An example of a one-shot command, i.e. a single `sqlite3` invocation at the shell prompt, is:

```bash
sqlite3 db.sqlite 'select count(*) from sap_capire_bookshop_Authors'
```

<a name="footnote-5"></a>
### Footnote 5

The development profile is the default; with `cds env requires.db --profile production` we get:

```text
undefined
```

<a name="footnote-6"></a>
### Footnote 6

The name can also be `csv/` which is also "special".

[provide initial data]: https://cap.cloud.sap/docs/guides/databases#providing-initial-data
[no particular strict convention for SQLite database filename extensions]: https://stackoverflow.com/questions/808499/does-it-matter-what-extension-is-used-for-sqlite-database-files
[profile]: https://cap.cloud.sap/docs/node.js/cds-env#profiles
[Hitchhiker's Guide To The Galaxy]: https://en.wikipedia.org/wiki/The_Hitchhiker%27s_Guide_to_the_Galaxy
[Deploy to a persistent file]: ../01/README.md#deploy-to-a-persistent-file
[assets/]: assets/
[CDL]: https://cap.cloud.sap/docs/cds/cdl
