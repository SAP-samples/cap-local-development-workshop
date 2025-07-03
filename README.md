# CAP local development workshop

[![REUSE status](https://api.reuse.software/badge/github.com/SAP-samples/cap-local-development-workshop)](https://api.reuse.software/info/github.com/SAP-samples/cap-local-development-workshop)

## Description

The content of this repository is for use in a [reCAP] 2025 hands-on workshop:

Title: Stay cool, stay local.

Abstract: CAP has myriad features to help developers develop. And that means local first, in a tight feedback loop. In this session you'll learn about those features and tools at your disposal as a CAP developer (predominantly Node.js) and get the chance to try some of them out yourself.

## Prerequisites

To participate in this workshop, the following prerequisites are required:

Ideally:

- The following installed on your laptop:
  - [VS Code](https://code.visualstudio.com/download)
  - [Docker Desktop](https://www.docker.com/products/docker-desktop/) (or equivalent container runtime engine)
  - the [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) installed in VS Code
  - [git](https://git-scm.com/) (to be able to clone a repository from GitHub)

Alternatively:

Your own development environment for CAP Node.js already set up and working, with CAP Node.js [9.0.0+](https://cap.cloud.sap/docs/releases/may25) installed and a Bash-compatible shell (e.g. via WSL on Windows), plus [jq](https://jqlang.org/) and [curl](https://curl.se/) installed.

In both cases:

  - An active trial account on SAP Business Technology Platform, with a Cloud Foundry environment instance (see [this tutorial] for details) - this will be needed for the last exercise, and we may be able to supply temporary access for participants

## Exercises

Work through each of the following exercises one at a time, each of which cover one or more topics that are relevant for local development (and are shown in brackets following the exercise titles). When reading through the exercises, actions for you to take, things you have to do yourself, are indicated with the 👉 symbol.

- [00 - setting up and getting to a running server](exercises/00/) ([cds watch])
- [01 - cds watch, SQLite, initial data and sample data](exercises/01/) ([cds watch], [SQLite], [initial data])
- [02 - configuration profiles, more on initial data, and the cds REPL](exercises/02/) ([configuration profiles], [initial data], [cds REPL])
- [03 - mocking auth and required services](exercises/03) ([mock user authentication], [mocking of required services])
- [04 - a first look at local messaging and events](exercises/04) ([local messaging], [file-based messaging])
- [05 - workspaces, monorepos and more on messaging and events](exercises/05) ([workspaces and monorepos], [plugins], [file-based messaging])
- [06 - debugging local and remote servers](exercises/06) ([debugging remote applications])

## Other topics

There are other topics in Capire that also are relevant for local development, but not covered in this workshop (except in passing):

- [hybrid testing]
- [linting]
- [cds init]
- [cds add]
- [serving UIs]

## How to obtain support

Support for the content in this repository is available during the actual time of the workshop event for which this content has been designed.

## License

Copyright (c) 2025 SAP SE or an SAP affiliate company. All rights reserved. This project is licensed under the Apache Software License, version 2.0 except as noted otherwise in the [LICENSE](LICENSES/Apache-2.0.txt) file.

[reCAP]: https://recap-conf.dev/
[this tutorial]: https://developers.sap.com/tutorials/hcp-create-trial-account.html
[hybrid testing]: https://cap.cloud.sap/docs/advanced/hybrid-testing
[configuration profiles]: https://cap.cloud.sap/docs/node.js/cds-env#profiles
[SQLite]: https://cap.cloud.sap/docs/guides/databases-sqlite
[initial data]: https://cap.cloud.sap/docs/guides/databases#providing-initial-data
[cds watch]: https://cap.cloud.sap/docs/tools/cds-cli#cds-watch
[cds REPL]: https://cap.cloud.sap/docs/tools/cds-cli#cds-repl
[mock user authentication]: https://cap.cloud.sap/docs/guides/security/authorization#prerequisite-authentication
[debugging remote applications]: https://cap.cloud.sap/docs/tools/cds-cli#remote-applications
[mocking of required services]: https://cap.cloud.sap/docs/guides/using-services#mock-remote-service-as-odata-service-node-js
[plugins]: https://cap.cloud.sap/docs/plugins/#support-for-plugins
[workspaces and monorepos]: https://cap.cloud.sap/docs/guides/deployment/microservices#create-a-solution-monorepo
[linting]: https://cap.cloud.sap/docs/tools/cds-lint/#usage-lint-cli
[cds init]: https://cap.cloud.sap/docs/tools/cds-cli#cds-init
[cds add]: https://cap.cloud.sap/docs/tools/cds-cli#cds-add
[local messaging]: https://cap.cloud.sap/docs/node.js/messaging#local-messaging
[file-based messaging]: https://cap.cloud.sap/docs/node.js/messaging#file-based
[serving UIs]: https://cap.cloud.sap/docs/get-started/in-a-nutshell#uis
