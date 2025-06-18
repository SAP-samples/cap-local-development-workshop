# Exercise 01 - SQLite, initial data and a first look at profiles

While not really appropriate for [productive use], SQLite shines in local development environments and allows for the tightest feedback loop. It's no second class database system either, as you'll see; via the modern `@cap-js/sqlite` database service implementation it provides full support for all kinds of CQL constructions such as path expressions. And with the [command line shell for SQLite], it's easy to interact with locally and natively. Along with with one of CAP's great features for local development and fast boostrapping - the ability to [provide initial data] - it's a combination that's hard to beat.

In this exercise you'll explore the facilities on offer in this space, using the sample project you created at the end of the previous exercise. The sample project is a "bookshop" style affair with authors, books and genres as the main players.

> Throughout this exercise keep the `cds watch` process running and in its own terminal instance; if necessary, open a second terminal to run any other commands you need, so you've always got the `cds watch` process running and visible.

## Add a new service definition Ex01Service

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

It's worth pausing here to reflect on this; while not specifically a "local development" facility, the fact that we get CRUD+Q handled completely and automatically for us for any service like this, without writing a line of code, is "bonkers good".

Inspect the books data like this:

```shell
curl -s localhost:4004/ex01/Books \
| jq -r '.value|map([.ID, .title])[]|@tsv'
```

This should emit something like this:

```text
201     Wuthering Heights
207     Jane Eyre
251     The Raven
252     Eleonora
271     Catweazle
```

Delete "The Raven":

```shell
curl -X DELETE localhost:4004/ex01/Books/251
```

and you can check with the previous `curl` invocation that it's really gone.

Move to the terminal where the CAP server is running and hit Enter, which will cause it to restart. Because the default mode for the use of SQLite at this point, with no explicit configuration, is in-memory (note the `--in-memory?` option in the expanded version of `cds w` which is `cds serve all --with-mocks --in-memory?`, and look up the meaning of the `?` in that last option specified).

The deployment of the initial data to the in-memory SQLite database is redone and "The Raven" is back (check with the previous `curl` invocation again) ... no doubt to [continue repeating the word "Nevermore"].o

<!-- TODO: NEXT, DEPLOY TO DB.SQLITE FILE -->

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

[productive use]: https://cap.cloud.sap/docs/guides/databases-sqlite#sqlite-in-production
[command line shell for SQLite]: https://sqlite.org/cli.html
[provide initial data]: https://cap.cloud.sap/docs/guides/databases#providing-initial-data
[developer friendly version of no-code]: https://qmacro.org/blog/posts/2024/11/07/five-reasons-to-use-cap/#1-the-code-is-in-the-framework-not-outside-of-it
[continue repeating the word "Nevermore"]: https://en.wikipedia.org/wiki/The_Raven#:~:text=the%20raven%20seems%20to%20further%20antagonize%20the%20protagonist%20with%20its%20repetition%20of%20the%20word%20%22nevermore%22
