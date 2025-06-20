# Exercise 01 - SQLite, initial data and a first look at profiles

While not really appropriate for [productive use], SQLite shines in local development environments and allows for the tightest feedback loop. It's no second class database system either, as you'll see; via the modern `@cap-js/sqlite` database service implementation it provides full support for all kinds of CQL constructions such as path expressions. And with the [command line shell for SQLite], it's easy to interact with locally and natively. Along with with one of CAP's great features for local development and fast boostrapping - the ability to [provide initial data] - it's a combination that's hard to beat.

In this exercise you'll explore the facilities on offer in this space, using the sample project you created at the end of the previous exercise. The sample project is a "bookshop" style affair with authors, books and genres as the main players.

> Throughout this exercise keep the `cds watch` process running and in its own terminal instance; if necessary, open a second terminal to run any other commands you need, so you've always got the `cds watch` process running and visible.

## Add a new service definition

To illustrate the simple power of `cds watch` plus the ultimate [developer friendly version of no-code] (the code is in the framework, not anything you write or even generate as boilerplate), add a new service definition to expose the books in a straightforward (non-administrative) way to keep things simple (see [footnote 1](#footnote-1)).

Add the following to a new file `srv/ex01-service.cds`:

```cds
using { sap.capire.bookshop as my } from '../db/schema';
@path: '/ex01' service Ex01Service {
  entity Books as projection on my.Books;
}
```

When you save the file the CAP server process restarts automatically, and you should notice a couple of things. First, it is gathered up in the collection of CDS model sources:

```log
[cds] - loaded model from 10 file(s):

  srv/ex01-service.cds
  ...
```

Also, it is automatically made available via the (default) OData adapter too:

```log
[cds] - serving Ex01Service {
  impl: 'node_modules/@sap/cds/libx/_runtime/common/Service.js',
  path: '/ex01'
}
```

> It's worth pausing here to reflect on this; while not specifically a "local development" facility, the fact that we get CRUD+Q handled completely and automatically for us for any service like this, without writing a line of code, is [insanely great].

## Dig in to the SQLite storage

Inspect the books data like this:

```bash
curl -s localhost:4004/ex01/Books \
| jq -r '.value|map([.ID, .title])[]|@tsv'
```

This should emit:

```text
201     Wuthering Heights
207     Jane Eyre
251     The Raven
252     Eleonora
271     Catweazle
```

### Experience the default in-memory mode

Banish "The Raven":

```bash
curl -X DELETE localhost:4004/ex01/Books/251
```

and you can check with the previous `curl` invocation that it's really gone.

Move to the terminal where the CAP server is running and hit Enter, which will cause it to restart. Because the default mode for the use of SQLite at this point, with no explicit configuration, is in-memory (see [footnote 2](#footnote-2)), the deployment of the initial data to the in-memory SQLite database is redone and "The Raven" is back (check with the previous `curl` invocation again) ... no doubt to [continue repeating the word "Nevermore"].

We can check this default configuration with [cds env]:

```bash
cds env requires.db
```

which should return something like this, reflecting the implicit out-of-the-box default for development (see [footnote-5](#footnote-5)):

```text
{
  impl: '@cap-js/sqlite',
  credentials: { url: ':memory:' },
  kind: 'sqlite'
}
```

### Deploy to a persistent file

We can also use a persistent database file, useful if we want the outcome of our OData requests to persist across CAP server restarts.

Use:

```bash
cds deploy --to sqlite
```

to deploy the CDS model, and the initial data, to a file whose name defaults to `db.sqlite` (see [footnote-3](#footnote-3)).

Nudge the CAP server to restart as before (move to the terminal where it's running and hit Enter) ... and notice that nothing has changed:

```log
[cds] - connect to db > sqlite { url: ':memory:' }
```

That's because we need to explicitly configure this setup. So let's do that now, by adding this to `package.json`:

```json
"cds": {
  "requires": {
    "db": {
      "kind": "sqlite"
    }
  }
}
```

> We could of course have specified a custom name for the database file, such as `bookshop.db`, but why fight CAP's wonderful [convention over configuration]? Incidentally, we would have had to specify the custom name within a `credentials` section of what we've just added:
>
> ```json
> "cds": {
>   "requires": {
>     "db": {
>       "kind": "sqlite",
>       "credentials": {
>         "url": "bookshop.db"
>       }
>     }
>   }
> }
> ```
>
> That's not to say that we can't be explicit here even with the default filename:
>
> ```json
> "cds": {
>   "requires": {
>     "db": {
>       "kind": "sqlite",
>       "credentials": {
>         "url": "db.sqlite"
>       }
>     }
>   }
> }
> ```
>
> As we'll see later in this exercise, this comes in handy sometimes!

While we're here thinking about configuration and the [cds env], let's check that same node now:

```bash
cds env requires.db
```

The output should reflect what we've added, replacing the earlier default (note the value for `url` is different):

```text
{
  impl: '@cap-js/sqlite',
  credentials: { url: 'db.sqlite' },
  kind: 'sqlite'
}
```

And when you trigger a CAP server restart (with Enter) you should see something like this:

```log
[cds] - connect to db > sqlite { url: 'db.sqlite' }
```

Now when you make modifications to the data, the modifications persist within the database file, across CAP server restarts too of course.

### Work directly at the database layer with the SQLite CLI

Now that we have some data (and the schema into which it fits) that we can look at, let's do that. The database engine is local, the database is local, so everything is at hand.

The `sqlite3` executable is known as SQLite's "command line shell" as it offers a prompt-based environment where we can explore (but we can also fire off one-shot commands too - see [footnote-4](#footnote-4)).

Let's start out by looking at what we have.

```bash
sqlite3 db.sqlite
```

This should land us in the command line shell:

```text
SQLite version 3.40.1 2022-12-28 14:03:47
Enter ".help" for usage hints.
sqlite>
```

How about looking at what artifacts are in there:

```text
sqlite> .tables
AdminService_Authors                  cds_outbox_Messages
AdminService_Books                    localized_AdminService_Books
AdminService_Books_drafts             localized_AdminService_Currencies
AdminService_Books_texts              localized_AdminService_Genres
AdminService_Books_texts_drafts       localized_AdminService_Languages
AdminService_Currencies               localized_CatalogService_Books
AdminService_Currencies_texts         localized_CatalogService_Currencies
AdminService_DraftAdministrativeData  localized_CatalogService_Genres
AdminService_Genres                   localized_CatalogService_ListOfBooks
AdminService_Genres_texts             localized_Ex01Service_Books
AdminService_Languages                localized_Ex01Service_Currencies
AdminService_Languages_texts          localized_Ex01Service_Genres
CatalogService_Books                  localized_sap_capire_bookshop_Books
CatalogService_Books_texts            localized_sap_capire_bookshop_Genres
CatalogService_Currencies             localized_sap_common_Currencies
CatalogService_Currencies_texts       localized_sap_common_Languages
CatalogService_Genres                 sap_capire_bookshop_Authors
CatalogService_Genres_texts           sap_capire_bookshop_Books
CatalogService_ListOfBooks            sap_capire_bookshop_Books_texts
DRAFT_DraftAdministrativeData         sap_capire_bookshop_Genres
Ex01Service_Books                     sap_capire_bookshop_Genres_texts
Ex01Service_Books_texts               sap_common_Currencies
Ex01Service_Currencies                sap_common_Currencies_texts
Ex01Service_Currencies_texts          sap_common_Languages
Ex01Service_Genres                    sap_common_Languages_texts
Ex01Service_Genres_texts
```

Knowing that the CDS model, at the Data Definition Language ([DDL]) layer, consists predominantly of tables and views, we can dig in and see which is what with something we learned from [The Art and Science of CAP], in particular [in Episode 8] (output reduced for brevity):

```text
sqlite> select type,name from sqlite_schema order by type;
index|sqlite_autoindex_sap_common_Languages_1
index|sqlite_autoindex_sap_common_Currencies_1
index|sqlite_autoindex_sap_capire_bookshop_Books_texts_1
index|sqlite_autoindex_sap_capire_bookshop_Books_texts_2
...
table|sap_capire_bookshop_Books
table|sap_capire_bookshop_Authors
table|sap_capire_bookshop_Genres
table|cds_outbox_Messages
table|sap_common_Languages
table|sap_common_Currencies
table|sap_capire_bookshop_Books_texts
...
view|AdminService_Books
view|AdminService_Authors
view|CatalogService_Books
view|Ex01Service_Books
view|AdminService_Languages
view|AdminService_Genres
view|AdminService_Currencies
view|AdminService_Books_texts
view|CatalogService_Genres
view|CatalogService_Currencies
view|CatalogService_Books_texts
view|Ex01Service_Genres
view|Ex01Service_Currencies
...
```

What about the data?

```text
sqlite> select title,stock from sap_capire_bookshop_Books;
Wuthering Heights|12
Jane Eyre|11
The Raven|333
Eleonora|555
Catweazle|22
```

> The `sqlite3` shell has completion, you might want to try it out, it's triggered with the Tab key, and especially useful for long table names such as the one here.

Sometimes we will want to perhaps adjust or augment the data in the database directly, for testing purposes (to avoid having to modify the source initial data and then re-deploy and re-start the CAP server). That's easy because everything is local. Let's try that now:

```text
sqlite> update sap_capire_bookshop_Books set stock = 1000 where ID = 271;
```

Yep, that works:

```bash
; curl -s 'localhost:4004/ex01/Books/271?$select=title,stock' | jq .
{
  "@odata.context": "$metadata#Books/$entity",
  "title": "Catweazle",
  "stock": 4711,
  "ID": 271
}
```

## Manage your initial data

Did you notice these lines in the CAP server log output:

```log
  > init from db/data/sap.capire.bookshop-Genres.csv
  > init from db/data/sap.capire.bookshop-Books_texts.csv
  > init from db/data/sap.capire.bookshop-Books.csv
  > init from db/data/sap.capire.bookshop-Authors.csv
/> successfully deployed to in-memory database.
```

You will have also seen them in deploy output, for example `cds deploy --to sqlite` produces this:

```log
  > init from db/data/sap.capire.bookshop-Genres.csv
  > init from db/data/sap.capire.bookshop-Books_texts.csv
  > init from db/data/sap.capire.bookshop-Books.csv
  > init from db/data/sap.capire.bookshop-Authors.csv
/> successfully deployed to db.sqlite
```

Capire has details on how to [provide initial data] and the convention is to place CSV files with names corresponding to the fully qualified entity names in a `data/` directory adjacent to the definitions in the CDS model, typically (as implied in the log output here) directly within the `db/` directory.

### Maintain a separate initial data collection

Sometimes it's necessary to maintain and use different starting sets of initial data. You can manage this with the combination of convention (the mechanism looks for `data/` directories directly within the `db/`, `srv/` and `app/` directories) and the [profile] concept.

Let's try this out.

First, we'll switch back to the in-memory SQLite facility so we can more easily and immediately see the effect of our actions in the CAP server log (if the database is in-memory then a deployment is done each and every time the server restarts, which makes sense when you think about it).

Rather than remove the configuration we added earlier to `package.json`, we can adjust it to explicitly specify the `url` to be `:memory:`, so it looks like this:

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

Once a CAP server restart is triggered, there will be the familiar "init from ..." lines to see.

OK. The name of the `data/` directory is special (see [footnote-6](#footnote-6)), and its relative location is also special; if we move it to somewhere else, the files containing the initial data won't get picked up automatically. Let's try that now:

```bash
mkdir db/classics/ \
  && mv db/data/ db/classics/
```

At this point, the log output from the restarted CAP server doesn't show any "init from ..." lines, as no initial data was found (in the expected / default location(s)).

But we can tell the CAP server about this "classics" initial data collection and assign a name to it, in the form of a [profile].

Let's do that now, by adding a new node to `package.json#cds.requires` so that it looks like this:

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

This doesn't have any positive effect yet; we need a couple more things. First, we need to add an empty CDS model file in the form of `index.cds` to the new `db/classics/` directory; this is so the CDS model compiler acknowledges this new "classics" directory and treats it as part of the model, including any initial data loading requirements:

```bash
touch db/classics/index.cds
```

Now we need to actually go to the CAP server, stop it (with Ctrl-C) and restart it, specifying this new "classics" name as a profile:

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

To illustrate this technique more fully, let's add a second initial data collection in a similar way. There are a couple of CSV data files for the author Douglas Adams and the books in his (increasingly inaccurately named) [Hitchhiker's Guide To The Galaxy] trilogy.

Create a new "hitchhikers" directory and copy them in from this workshop repository's [attic/] directory, like this:

```bash
mkdir -p db/hitchhikers/data/ \
  && cp ../exercises/01/assets/data/json/* $_ \
  && touch db/hitchhikers/index.cds
```

> Note that this time, just to illustrate the possibility, the data files are JSON not CSV. This works too, but JSON files are only supported in development mode.

Add this to the effective configuration with a new section in `package.json#cds.requires` similar to the previous one, so that the entire `cds` stanza looks like this:

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

Now you can restart the CAP server using the "hitchhikers" profile:

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

### Combine the features

You can combine this feature with the ability to configure file based persistence in `cds.requires` too. Try this:

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

## Understand the difference between sample and initial data

Until now we've been managing and using initial data. There's also support for using _sample_ data locally. Briefly, sample data is exclusively for tests and demos, in other words for local development only, not for production.

The CAP server will look for and load sample data from `data/` directories inside a project root based `test/` directory parent.

To wrap up this exercise, let's explore.

First, we should stop the CAP server (with Ctrl-C), then restart it in "watch" mode but without a specific profile:

```bash
cds w
```

Next, let's create a symbolic link that will look like this:

```text
+----------+      +----------------------+
| db/data/ |----->| db/hitchhikers/data/ |
+----------+      +----------------------+
```

so that we can treat the "hitchhikers" data set as the default initial data and run without an explicit profile for now:

```bash
cd db \
  && ln -s hitchhikers/data . \
  && cd -
```

When the CAP server restarts it should show the "init from ..." log lines taking data from these "hitchhikers" JSON files.

Now add some sample data in a new directory `test/data/`, thus:

```bash
mkdir -p test/data/ \
  && cp ../exercises/01/assets/test/data/csv/* $_
```

As soon as you do this, on restarting, the CAP server will emit these log lines:

```log
  > init from test/data/sap.capire.bookshop-Books.csv
  > init from db/hitchhikers/data/sap.capire.bookshop-Books.json
  > init from db/hitchhikers/data/sap.capire.bookshop-Authors.json
```

showing that initial and sample data has been loaded.

However, as the Capire documentation on how to [provide initial data] points out, `test/data/` based sample data is for tests and demos only, and not for production.

<!-- TODO: SHOW CDS BUILD EXAMPLE -->


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

[productive use]: https://cap.cloud.sap/docs/guides/databases-sqlite#sqlite-in-production
[command line shell for SQLite]: https://sqlite.org/cli.html
[provide initial data]: https://cap.cloud.sap/docs/guides/databases#providing-initial-data
[developer friendly version of no-code]: https://qmacro.org/blog/posts/2024/11/07/five-reasons-to-use-cap/#1-the-code-is-in-the-framework-not-outside-of-it
[continue repeating the word "Nevermore"]: https://en.wikipedia.org/wiki/The_Raven#:~:text=the%20raven%20seems%20to%20further%20antagonize%20the%20protagonist%20with%20its%20repetition%20of%20the%20word%20%22nevermore%22
[insanely great]: https://www.inc.com/jason-aten/the-2-word-phrase-steve-jobs-used-to-inspire-his-team-to-make-worlds-most-iconic-products.html
[no particular strict convention for SQLite database filename extensions]: https://stackoverflow.com/questions/808499/does-it-matter-what-extension-is-used-for-sqlite-database-files
[convention over configuration]: https://qmacro.org/blog/posts/2019/11/06/cap-is-important-because-it's-not-important/#start-smart
[The Art and Science of CAP]: https://qmacro.org/blog/posts/2024/12/06/the-art-and-science-of-cap/
[in Episode 8]: https://qmacro.org/blog/posts/2025/02/14/tasc-notes-part-8/#exploring-in-sqlite
[DDL]: https://cap.cloud.sap/docs/guides/databases#rules-for-generated-ddl
[profile]: https://cap.cloud.sap/docs/node.js/cds-env#profiles
[cds env]: https://cap.cloud.sap/docs/tools/cds-cli#cds-env
[Hitchhiker's Guide To The Galaxy]: https://en.wikipedia.org/wiki/The_Hitchhiker%27s_Guide_to_the_Galaxy
[attic/]: https://github.com/SAP-samples/cap-local-development-workshop/tree/main/attic
