# Exercise 01 - cds watch, SQLite, initial data and sample data

While not really appropriate for [productive use], SQLite shines in local development environments and allows for the tightest feedback loop. It's no second class database system either, as you'll see; via the modern `@cap-js/sqlite` database service implementation it provides full support for all kinds of CQL constructions such as path expressions (see the [Further reading](#further-reading) section for more info). And with the [command line shell for SQLite], it's easy to interact with locally and natively. Along with with one of CAP's great features for local development and fast boostrapping - the ability to [provide initial data] - it's a combination that's hard to beat.

In this exercise you'll explore the facilities on offer in this space, using the sample project you created at the end of the previous exercise. The sample project is a "bookshop" style affair with authors, books and genres as the main players.

> Throughout this exercise keep the `cds watch` process from the previous exercise running and in its own terminal instance; if necessary, open a second terminal and move to the `myproj/` project root directory to run any other commands you need, so you've always got the CAP server running and the log output visible.

## Add a new service definition

To illustrate the simple power of `cds watch` plus the ultimate [developer friendly version of no-code] (the code is in the framework, not anything you write or even generate as boilerplate), add a new service definition to expose the books in a straightforward (non-administrative) way to keep things simple (see [footnote 1](#footnote-1)).

👉 Add the following to a new file `srv/ex01-service.cds`:

```cds
using { sap.capire.bookshop as my } from '../db/schema';
@path: '/ex01' service Ex01Service {
  entity Books as projection on my.Books;
}
```

When you save the file the CAP server process restarts automatically, and you should notice a few things. The service definition is gathered up in the collection of CDS model sources:

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

In addition, there is some loading of data into an in-memory database:

```log
[cds] - connect to db > sqlite { url: ':memory:' }
  > init from db/data/sap.capire.bookshop-Genres.csv
  > init from db/data/sap.capire.bookshop-Books_texts.csv
  > init from db/data/sap.capire.bookshop-Books.csv
  > init from db/data/sap.capire.bookshop-Authors.csv
/> successfully deployed to in-memory database.
```

## Dig in to the SQLite storage

👉 Inspect the books data via a QUERY operation on the corresponding entityset in the new OData service that is now available, like this:

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

From the CAP server log output we observed that there's a SQLite powered in-memory database in play. Let's see how that affects things.

👉 Banish "The Raven":

```bash
curl -X DELETE localhost:4004/ex01/Books/251
```

and you can check with the previous `curl` invocation that it's really gone.

👉 Move to the terminal where the CAP server is running and hit Enter, which will cause it to restart.

As the default mode for the use of SQLite, with no explicit configuration, is in-memory (see [footnote 2](#footnote-2)), deployment of the initial data to the in-memory SQLite database is redone:

```log
[cds] - connect to db > sqlite { url: ':memory:' }
  > init from db/data/sap.capire.bookshop-Genres.csv
  > init from db/data/sap.capire.bookshop-Books_texts.csv
  > init from db/data/sap.capire.bookshop-Books.csv
  > init from db/data/sap.capire.bookshop-Authors.csv
/> successfully deployed to in-memory database.
```

and "The Raven" is back (check with the previous `curl` invocation again) ... no doubt to [continue repeating the word "Nevermore"].

👉 Check this default configuration with [cds env]:

```bash
cds env requires.db
```

which should return something like this, reflecting the implicit out-of-the-box default for development (see [footnote 5](#footnote-5)):

```text
{
  impl: '@cap-js/sqlite',
  credentials: { url: ':memory:' },
  kind: 'sqlite'
}
```

Note that there is no `cds` section within `package.json` at this point; this really is a built-in default.

### Deploy to a persistent file

We can also use a persistent database file, useful if we want the outcome of our OData requests to persist across CAP server restarts.

👉 Use:

```bash
cds deploy --to sqlite
```

to deploy the CDS model, and the initial data, to a file whose name defaults to `db.sqlite` (see [footnote 3](#footnote-3)).

👉 Nudge the CAP server to restart as before (move to the terminal where it's running and hit Enter) ... and notice that nothing has changed:

```log
[cds] - connect to db > sqlite { url: ':memory:' }
```

That's because we need to explicitly configure this setup.

👉 So let's do that now, by adding this to `package.json`:

```json
"cds": {
  "requires": {
    "db": {
      "kind": "sqlite"
    }
  }
}
```

> We could have specified a custom name for the database file, such as `bookshop.db`, but why fight CAP's wonderful [convention over configuration]? Incidentally, we would have had to specify the custom name within a `credentials` section of what we've just added:
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
> As we'll see in the next exercise, this comes in handy sometimes!

👉 While we're here thinking about configuration and the [cds env], let's check that same node now:

```bash
cds env requires.db
```

The output should reflect what we've added, replacing the earlier default. Specifically the value for `url` is different, going from `:memory:` to `db.sqlite`:

```text
{
  impl: '@cap-js/sqlite',
  credentials: { url: 'db.sqlite' },
  kind: 'sqlite'
}
```

And when the CAP server restarts you should see something like this:

```log
[cds] - connect to db > sqlite { url: 'db.sqlite' }
```

Notice too what you _don't_ see - there are no "init from ..." log lines now as there is no deployment (which is the mechanism to which this data loading belongs) to be done - the CAP server will not try and deploy to something that's persistent like our database file, without our say so.

> That's not to say there isn't anything in `db.sqlite` yet - did you notice the "init from ..." lines when you executed the `cds deploy` command just now?

Now when you make modifications to the data, the modifications persist within the database file, across CAP server restarts too of course.

### Work directly at the database layer with the SQLite CLI

Now that we have some data (and the schema into which it fits) that we can look at, let's do that. The database engine is local, the database is local, so everything is at hand.

The `sqlite3` executable is known as SQLite's "command line shell" as it offers a prompt-based environment where we can explore (see [footnote 4](#footnote-4)).

👉 Invoke the executable, specifying our database file:

```bash
sqlite3 db.sqlite
```

This should land us in the command line shell:

```text
SQLite version 3.40.1 2022-12-28 14:03:47
Enter ".help" for usage hints.
sqlite>
```

How about looking at what artifacts are in there?

👉 Try the `.tables` command:

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

Knowing that - at the Data Definition Language ([DDL]) layer - the CDS model consists predominantly of tables and views, we can dig in and see the artifacts and their types with something we learned from [The Art and Science of CAP], in particular [in Episode 8] where we looked at the `sqlite_schema`.

👉 In the SQLite shell, where you'll execute this and the next few commands, try this:

```sql
select type,name from sqlite_schema order by type;
```

which should emit a long list, similar to this (output reduced for brevity):

```text
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

👉 Try this:

```sql
select title,stock from sap_capire_bookshop_Books;
```

which should show:

```text
Wuthering Heights|12
Jane Eyre|11
The Raven|333
Eleonora|555
Catweazle|22
```

> The `sqlite3` shell has completion; you might want to try it out, it's triggered with the Tab key, and especially useful for long table names such as the one here.

Sometimes we will want to perhaps adjust or augment the data in the database directly, for testing purposes (to avoid having to modify the source initial data and then re-deploy and re-start the CAP server). That's easy because everything is local.

👉 Let's try that now:

```sql
update sap_capire_bookshop_Books set stock = 1000 where ID = 271;
```

👉 Now, exit the SQLite shell, then perform an OData READ operation to see if that has taken effect:

```bash
curl -s 'localhost:4004/ex01/Books/271?$select=title,stock' | jq .
```

Yep, looks like it did:

```json
{
  "@odata.context": "$metadata#Books/$entity",
  "title": "Catweazle",
  "stock": 4711,
  "ID": 271
}
```

## Understand the difference between initial and sample data

Thus far we've been managing and using _initial_ data. There's also support for using _sample_ data locally.

Briefly, sample data is exclusively for tests and demos, in other words for local development only, not for production.

The CAP server will look for and load sample data from `data/` directories _not_ within the standard `db/`, `srv/` and `app/` directories, but inside a project root based `test/` directory parent:

```text
.
├── db
│   └── data  <-- current initial data location
├── srv
└── test
    └── data  <-- sample data location
```

Let's explore this sample data concept now.

### Add a temporary Sales entity to the service

For the sake of keeping things simple, let's assume we want to think of our authors, books and genres as initial "master" data as ultimately destined for production, and explore the sample data concept with some "transactional" data in the form of some basic sales records, sample data that we only want while we're developing locally.

👉 In a new file called `srv/ex01-sales.cds` add this:

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

### Generate some sample sales data

👉 Next, use `cds add` with the `data` facet to create a CSV file with sample data for this new entity:

```bash
cds add data \
  --filter Sales \
  --records 3 \
  --out test/data/ \
  --force
```

> The `--force` option here isn't stricly necessary but useful in case we want to re-run this invocation later.

You should see something like this:

```log
using '--force' ... existing files will be overwritten
adding data
  creating test/data/Ex01Service.Sales.csv

successfully added features to your project
```

### Re-deploy to db.sqlite

The CAP server will have restarted, but we'll need to re-deploy, because currently our persistent storage (the SQLite `db.sqlite` file) contains neither DDL statements for the new `Sales` entity nor the sales records themselves, as we can see:

```bash
; sqlite3 db.sqlite
SQLite version 3.40.1 2022-12-28 14:03:47
Enter ".help" for usage hints.
sqlite> select type,name from sqlite_schema where name like '%Sales%';
sqlite>
```

So let's re-deploy. Now that we have the persistent (and default `db.sqlite` filename based) configuration defined in `package.json#cds.requires.db` we can simply invoke `cds deploy` without any options, as it will look at `cds.requires.db` to work out what to do.

👉 Make the deployment:

```bash
cds deploy
```

This should show:

```log
  > init from test/data/Ex01Service.Sales.csv
  > init from db/data/sap.capire.bookshop-Genres.csv
  > init from db/data/sap.capire.bookshop-Books_texts.csv
  > init from db/data/sap.capire.bookshop-Books.csv
  > init from db/data/sap.capire.bookshop-Authors.csv
/> successfully deployed to db.sqlite
```

Hey, look at that - the sales data from `test/data/Ex01Service.Sales.csv` is included!

👉 Let's check that (first making sure the CAP server has restarted - give it a nudge with Enter if needed):

```bash
curl -s localhost:4004/ex01/Sales | jq .
```

This should emit something similar to this:

```json
{
  "@odata.context": "$metadata#Sales",
  "value": [
    {
      "ID": "26243430-a307-4ba4-a72e-c4ce44653fa2",
      "date": "2003-09-29",
      "book_ID": 271,
      "quantity": 26
    },
    {
      "ID": "26243431-d479-4891-9c09-b2c88f7a43c2",
      "date": "2011-05-03",
      "book_ID": 207,
      "quantity": 29
    },
    {
      "ID": "26243432-68ce-4a10-b1e1-678e40f51346",
      "date": "2009-08-24",
      "book_ID": 207,
      "quantity": 69
    }
  ]
}
```

Great!

> Did you notice that the values for the `book_ID` property ... are not just random?

But let's make sure these sales records really are considered just local sample data.

### Perform a build

With the cds [build] command we can prepare a deployment for the cloud. Let's do that.

👉 Using `DEBUG=build` to see everything that happens, including all the files that are taken into account, run `build`:

```bash
DEBUG=build cds build --for hana
```

This produces a lot of log output, much of which has been omitted here for brevity:

```log
[cli] - determining build tasks for project [/work/scratch/myproj].
...
[cli] - model: db/schema.cds, srv/admin-service.cds, srv/cat-service.cds, srv/ex01-sales.cds, srv/ex01-service.cds, app/common.cds, app/services.cds, node_modules/@sap/cds/srv/outbox.cds
[cli] - compile.to.hana returned
done > wrote output to:
   gen/db/package.json
   gen/db/src/gen/.hdiconfig
   ...
   gen/db/src/gen/AdminService.Authors.hdbview
   gen/db/src/gen/AdminService.Books.hdbview
   ...
   gen/db/src/gen/CatalogService.Books.hdbview
   gen/db/src/gen/CatalogService.Books_texts.hdbview
   ...
   gen/db/src/gen/data/sap.capire.bookshop-Authors.csv
   gen/db/src/gen/data/sap.capire.bookshop-Authors.hdbtabledata
   ...
   gen/db/src/gen/sap.capire.bookshop.Authors.hdbtable
   gen/db/src/gen/sap.capire.bookshop.Books.hdbtable
   ...

build completed in 411 ms
```

The data files containing the _initial_ data (files in `db/data/`) are included, but the sample data file (in `test/data/`) is not:

```bash
; DEBUG=build cds build --for hana | grep -E '\/data\/'
   gen/db/src/gen/data/sap.capire.bookshop-Authors.csv
   gen/db/src/gen/data/sap.capire.bookshop-Authors.hdbtabledata
   gen/db/src/gen/data/sap.capire.bookshop-Books.csv
   gen/db/src/gen/data/sap.capire.bookshop-Books.hdbtabledata
   gen/db/src/gen/data/sap.capire.bookshop-Books_texts.csv
   gen/db/src/gen/data/sap.capire.bookshop-Books_texts.hdbtabledata
   gen/db/src/gen/data/sap.capire.bookshop-Genres.csv
   gen/db/src/gen/data/sap.capire.bookshop-Genres.hdbtabledata
```

We can see that initial data, in `test/`, is only for local sample and demo purposes.

That's the end of this exercise!

---

## Further reading

- [SQLite features]

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

We can also invoke one-shot commands too. An example of a one-shot command, i.e. a single `sqlite3` invocation at the shell prompt, is:

```bash
sqlite3 db.sqlite 'select count(*) from sap_capire_bookshop_Authors;'
```

<a name="footnote-5"></a>
### Footnote 5

The development profile is the default; with `cds env requires.db --profile production` we get:

```text
undefined
```

See the next exercise for more on profiles.

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
[cds env]: https://cap.cloud.sap/docs/tools/cds-cli#cds-env
[build]: https://cap.cloud.sap/docs/guides/deployment/custom-builds#build-task-properties
[SQLite features]: https://cap.cloud.sap/docs/guides/databases-sqlite#features
