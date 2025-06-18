# Exercise 01 - SQLite, initial data and an introduction to the REPL

While not really appropriate for [productive use], SQLite shines in local development environments and allows for the tightest feedback loop. It's no second class database system either, as you'll see; via the modern `@cap-js/sqlite` database service implementation it provides full support for all kinds of CQL constructions such as path expressions. And with the [command line shell for SQLite], it's easy to interact with locally and natively. Along with with one of CAP's great features for local development and fast boostrapping - the ability to [provide initial data] - it's a combination that's hard to beat.

[productive use]: https://cap.cloud.sap/docs/guides/databases-sqlite#sqlite-in-production
[command line shell for SQLite]: https://sqlite.org/cli.html
[provide initial data]: https://cap.cloud.sap/docs/guides/databases#providing-initial-data
