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

```shell
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

```shell
curl -X DELETE localhost:4004/ex01/Books/251
```

and you can check with the previous `curl` invocation that it's really gone.

Move to the terminal where the CAP server is running and hit Enter, which will cause it to restart. Because the default mode for the use of SQLite at this point, with no explicit configuration, is in-memory (see [footnote 2](#footnote-2)), the deployment of the initial data to the in-memory SQLite database is redone and "The Raven" is back (check with the previous `curl` invocation again) ... no doubt to [continue repeating the word "Nevermore"].

### Deploy to a persistent file

We can also use a persistent database file, useful if we want the outcome of our OData requests to persist across CAP server restarts.

Use:

```shell
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
> What's _your_ preference?

And when you trigger a CAP server restart (with Enter) you should see something like this:

```log
[cds] - connect to db > sqlite { url: 'db.sqlite' }
```

Now when you make modifications to the data, the modifications persist within the database file, across CAP server restarts too of course.

### Work directly at the database layer with the SQLite CLI

Now that we have some data (and the schema into which it fits) that we can look at, let's do that. The database engine is local, the database is local, so everything is at hand.

The `sqlite3` executable is known as SQLite's "command line shell" as it offers a prompt-based environment where we can explore (but we can also fire off one-shot commands too - see [footnote-4](#footnote-4)).

Let's start out by looking at what we have.

```shell
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

```shell
; curl -s 'localhost:4004/ex01/Books/271?$select=title,stock' | jq .
{
  "@odata.context": "$metadata#Books/$entity",
  "title": "Catweazle",
  "stock": 4711,
  "ID": 271
}
```

## Footnotes

<a name="footnote-1"></a>
### Footnote 1

They are already exposed but only in the `AdminService` which is annotated to protect it, with:

```cds
service AdminService @(requires:'admin') { ... }
```

(in `srv/admin-service.cds`). Yes, we can embrace the mock authentication:

```shell
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

An example of a one-shot command, i.e. a single `sqlite3` invocation, is:

```shell
sqlite3 db.sqlite 'select count(*) from sap_capire_bookshop_Authors'
```

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
