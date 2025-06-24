# Exercise 03 - mocking auth and required services

There's no avoiding the fact that if you want your CAP apps and services to be useful, they're going to have to make use of authentication mechanisms, and consume other APIs, in the cloud. In local development mode, however, these requirements can get in the way and hinder progress.

Fortunately the "developer centric" nature of CAP's local-first strategy provides various ways to not _ignore_ the reality, but to _suspend_ it until it's really needed. With the "mocking" approach, we can design and declare our domain model and adorn it with annotations relating to authentication, and rely on mocked authentication while still thinking about, defining and testing user role and attribute based access control. We can also have required services mocked for us so we can connect to them from our own.

In this exercise we'll explore both those topics.

## Explore the auth in play

You may have already noticed the mocked authentication approach declared in the CAP server's log lines.

👉 Check this by stopping any currently running CAP server (the `cds watch` process), restarting it with:

```bash
DEBUG=auth cds w --profile classics
```

and observe this output:

```log
[cds] - using auth strategy {
  kind: 'mocked',
  impl: 'node_modules/@sap/cds/lib/srv/middlewares/auth/basic-auth'
}
```

The "mocked" [authentication strategy] uses HTTP Basic Authentication (simple usernames and passwords) combined with [assignment of sample roles to pre-defined test users]. It's more or less the same as the "basic" authentication strategy, but with this user and role pre-configuration.

### Try accessing AdminService resources

The `AdminService` (currently made available at `/odata/v4/admin`) has a [@requires] annotation which declares that only users with the "admin" role have access:

```cds
using { sap.capire.bookshop as my } from '../db/schema';
service AdminService @(requires:'admin') {
  entity Books as projection on my.Books;
  entity Authors as projection on my.Authors;
}
```

#### Attempt an unauthenticated request

👉 Try to access the `Books` resource within that service without supplying any authentication detail:

```bash
curl -i localhost:4004/odata/v4/admin/Books
```

> The `--include` (or `-i`) option causes `curl` to emit the response headers.

This should result in an HTTP 401 response that will look like this (other response headers have been omitted for brevity):

```log
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="Users"
```

This will also have been logged by the CAP server due to the `DEBUG=auth` setting:

```log
[basic] - 401 > login required
```

As authorization checks (in play here) are predicated on verified identity claims, [authentication is a prerequisite] here.

#### Attempt a request authenticated as a user without the requisite role

👉 Try authenticating with the pre-defined user "yves", who has the role "internal-user":

```bash
curl -i -u 'yves:' localhost:4004/odata/v4/admin/Books
```

> The colon separates the username and password when supplying this detail via `curl`'s `--user` (or `-u`) option. None of the pre-defined users have passwords. If you omit the colon from the value supplied to `-u` here, `curl` will prompt you for a password (you can just hit Enter to send an empty password in this case).

Underlining the difference between HTTP [401] and HTTP [403] responses, this request should elicit:

```log
HTTP/1.1 403 Forbidden
```

with some extra info in the CAP server log output too:

```log
[basic] - authenticated: { user: 'yves', tenant: undefined, features: undefined }
[odata] - GET /odata/v4/admin/Books
[error] - 403 - Error: Forbidden
    at requires_check (/work/scratch/myproj/node_modules/@sap/cds/lib/srv/protocols/http.js:54:32)
    at http_log (/work/scratch/myproj/node_modules/@sap/cds/lib/srv/protocols/http.js:42:59) {
  code: '403',
  reason: "User 'yves' is lacking required roles: [admin]",
  user: User { id: 'yves', roles: { 'internal-user': 1 } },
  required: [ 'admin' ],
  '@Common.numericSeverity': 4
}
```

> Briefly:
>
> Response Code | Meaning | In short
> -|-|-
> 401|The request lacked valid authentication credentials|Can't verify who you are
> 403|The request did contain valid credentials (i.e. was properly authenticated) but the authenticated user does not have the requisite permissions|Your identify is verified but you don't have access

#### Make a request authenticated as a user with the requisite role

👉 Repeat the same resource request but this time as user "alice", who does have the "admin" role:

```bash
curl -s -u 'alice:' localhost:4004/odata/v4/admin/Books \
  | jq .value[].title
```

> The `--silent` (or `-s`) option turns on silent mode which means we don't get the typical progress info:
>
> ```log
>   % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
>                                  Dload  Upload   Total   Spent    Left  Speed
> 100  4572  100  4572    0     0   611k      0 --:--:-- --:--:-- --:--:--  637k

Success! Here's what the CAP server shows:

```log
[basic] - authenticated: { user: 'alice', tenant: undefined, features: undefined }
[odata] - GET /odata/v4/admin/Books
```

and here's what's returned:

```log
"Wuthering Heights"
"Jane Eyre"
"The Raven"
"Eleonora"
"Catweazle"
```

### Explore the @requires and @restrict annotations with the Ex01Service

In a previous exercise we [added a new service definition]; now we'll add some annotations to that to get a feel for how they work, but more importantly to see how the mocked strategy supports exactly what would be supported in production.

#### Set up restrictions

👉 Append annotation declarations to the `srv/ex01-service.cds` file so the entire content ends up looking like this:

```cds
using { sap.capire.bookshop as my } from '../db/schema';
@path: '/ex01' service Ex01Service {
  entity Books as projection on my.Books;
}

annotate Ex01Service with @requires: 'authenticated-user';
annotate Ex01Service.Books with @restrict: [
  { grant: 'READ' },
  { grant: 'WRITE', to: 'backoffice' }
]);
```

These annotations set up:

- a requirement that any request to the service be authenticated (i.e. made with a verified identity)
- a restriction on the `Books` entity in that any authenticated user can perform read operations, but only users with the "backoffice" role can perform write operations

#### Make an unauthenticated request

👉 Try to retrieve the details of the book with ID 207 ("Jane Eyre") without providing any authentication details:

```bash
curl -i localhost:4004/ex01/Books/207
```

As expected, we don't get very far with this; the output includes:

```log
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="Users"

Unauthorized
```

and there's this line in the CAP server log:

```log
[basic] - 401 > login required
```

#### Make authenticated requests with an administrative user

👉 Try that again, authenticating as the pre-defined administrative user "alice":

```bash
curl -i -u 'alice:' localhost:4004/ex01/Books/207
```

The conditions of the (rather general) `READ` grant, combined with the `authenticated-user` requirement, are both fulfilled, meaning success for Alice:

```log
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

{"@odata.context":"$metadata#Books/$entity","createdAt":"2025-06-23T13:26:48.247Z","createdBy":"anonymous","modifiedAt":"2025-06-23T13:26:48.247Z","modifiedBy":"anonymous","ID":207,"title":"Jane Eyre","descr":"..."}
```

But can Alice perform operations with the `WRITE` semantic?

👉 Try it:

```bash
curl -X DELETE -i -u 'alice:' localhost:4004/ex01/Books/207
```

No!

```log
HTTP/1.1 403 Forbidden

{"error":{"message":"Forbidden","code":"403","@Common.numericSeverity":4}}
```

#### Define a new office user

While there are pre-defined users we can make use of in the mocked authentication strategy, we can define our own too, which is especially helpful when we're iterating locally on building out the domain model and including the security considerations with that.

We could put this configuration in `package.json#cds` but for a change, let's use a [project-local .cdsrc.json] file.

👉 Create a `.cdsrc.json` file in the project root, with this content for Milton, the [stapler guy]:

```json
{
  "requires": {
    "auth": {
      "users": {
        "milton": {
          "password": "dontmovemydesk",
          "roles": [
            "stapler"
          ]
        }
      }
    }
  }
}
```

Once this is saved, the CAP server will restart.

Just for interest, check the effective environment, specifically for the auth details:

```bash
cds env requires.auth
```

This should emit:

```log
{
  restrict_all_services: false,
  kind: 'mocked',
  users: {
    alice: { tenant: 't1', roles: [ 'admin' ] },
    bob: { tenant: 't1', roles: [ 'cds.ExtensionDeveloper' ] },
    carol: { tenant: 't1', roles: [ 'admin', 'cds.ExtensionDeveloper' ] },
    dave: { tenant: 't1', roles: [ 'admin' ], features: [] },
    erin: { tenant: 't2', roles: [ 'admin', 'cds.ExtensionDeveloper' ] },
    fred: { tenant: 't2', features: [ 'isbn' ] },
    me: { tenant: 't1', features: [ '*' ] },
    yves: { roles: [ 'internal-user' ] },
    '*': true,
    milton: { password: 'dontmovemydesk', roles: [ 'stapler' ] }
  },
  tenants: { t1: { features: [ 'isbn' ] }, t2: { features: '*' } }
}
```

Alongside the pre-defined users we can see Milton.

> Adding the `--profile classics` option here is also possible, but the end result is the same in this case.

#### Authenticate a request with the new office user

👉 Let's try this new user, like this:

```bash
curl -X DELETE -i -u 'milton:dontmovemydesk' localhost:4004/ex01/Books/207
```

Not quite!

```log
HTTP/1.1 403 Forbidden

{"error":{"message":"Forbidden","code":"403","@Common.numericSeverity":4}}
```

Of course, we need to give him the "backoffice" role.

👉 Do that now by adding it to the `[ ... ]` list of roles in `.cdsrc.json` so that it looks like this:

```json
"roles": [
  "stapler",
  "backoffice"
]
```

Now try again:

```bash
curl -X DELETE -i -u 'milton:dontmovemydesk' localhost:4004/ex01/Books/207
```

Success!

```log
HTTP/1.1 204 No Content
```

This just scratches the surface of what's possible; remember that the power of all of the abstracted authentication and authorisation layers (including users) is available to all authentication strategies, even (or "especially") the ones designed for local development. And there's no change when one moves to production, at that level.

See the [Further reading](#further-reading) section for more information.

## Mock an external service

Working locally doesn't mean that we need to avoid development that involves remote services. A remote service API definition can be downloaded and imported, so that it becomes known to the CAP server (as a "required" service, rather than a "provided" service), and via the translation of the API definition to an internal model representation in Core Schema Notation [CSN] it can also be given active behavior and even test data.

Everyone loves Northwind (don't they?) so let's use a cut-down version of Northwind, called Northbreeze, which is available at <https://developer-challenge.cfapps.eu10.hana.ondemand.com/odata/v4/northbreeze>.

### Import the API definition

The Northbreeze service is an OData v4 service and the API definition for that is essentially the EDMX available in the service's metadata document.

👉 Retrieve the metadata document resource and store the representation in a file:

```bash
curl -s \
  --url 'https://developer-challenge.cfapps.eu10.hana.ondemand.com/odata/v4/northbreeze/$metadata' \
  > northbreeze.edmx
```

👉 Now use the `cds import` command to import the API definition (in its EDMX form) and convert it to CSN:

```bash
cds import northbreeze.edmx
```

You should see something like this:

```log
[cds] - imported API to srv/external/northbreeze
> use it in your CDS models through the like of:

using { northbreeze as external } from './external/northbreeze'
```

👉 Let's have a look at where the imported CSN is, in relation to other content in `srv/`:

```bash
tree srv
```

We can see that the default location that `cds import` uses makes a lot of sense, in that it's a service, but not part of our own overall CDS model:

```log
srv
├── admin-service.cds
├── admin-service.js
├── cat-service.cds
├── cat-service.js
├── ex01-service.cds
└── external
    ├── northbreeze.csn
    └── northbreeze.edmx
```

Moreover, a reference to this as a "required" service has been added to the `package.json#cds` based configuration, which we can perhaps better observe by looking at the effective `requires` configuration.

👉 Do that now:

```bash
cds env requires
```

As well as the sections for `middlewares`, `queue`, `auth` and `db` (which have been reduced for brevity), we now have `northbreeze` listed, an "external" "OData" resource whose "model" is known and which is "implemented" by a built-in remote service module:

```log
{
  middlewares: true,
  queue: {
    model: '@sap/cds/srv/outbox',
    ...
    kind: 'persistent-queue'
  },
  auth: {
    restrict_all_services: false,
    kind: 'mocked',
    users: {
      alice: { tenant: 't1', roles: [ 'admin' ] },
      ...
      milton: {
        password: 'dontmovemydesk',
        roles: [ 'stapler', 'backoffice' ]
      }
    },
    tenants: { t1: { features: [ 'isbn' ] }, t2: { features: '*' } }
  },
  db: {
    impl: '@cap-js/sqlite',
    credentials: { url: ':memory:' },
    kind: 'sqlite'
  },
  northbreeze: {
    impl: '@sap/cds/libx/_runtime/remote/Service.js',
    external: true,
    kind: 'odata',
    model: 'srv/external/northbreeze'
  }
}
```

### Have the service mocked

👉 Start mocking this service:

```bash
cds mock northbreeze --port 5005
```

> Normally, without the `--port` option, a random port will be chosen, but for the sake of this workshop and consistency of instructions, we'll use a specific port.

This will start a CAP server just for this service:

```log
...
[cds] - connect using bindings from: { registry: '~/.cds-services.json' }
...
[cds] - mocking northbreeze {
  impl: 'node_modules/@sap/cds/libx/_runtime/common/Service.js',
  path: '/odata/v4/northbreeze'
}
[cds] - server listening on { url: 'http://localhost:5005' }
...
```

But there's no data right now, as illustrated with simple request like this:

```bash
; curl 'localhost:5005/odata/v4/northbreeze/Suppliers/$count'
0
```

### Add some data

The sensible place to put data is "next to" the model definition, which means here, in this case:

```text
srv/
├── admin-service.cds
├── admin-service.js
├── cat-service.cds
├── cat-service.js
├── ex01-service.cds
└── external
    ├── data
    │   └── [data files go here]
    ├── northbreeze.csn
    └── northbreeze.edmx
```

#### Use generated data

👉 So, after stopping the server, create a `data/` directory in `srv/external/`, and then use the "data" facet with `cds add` to generate a few records of mock data for the `Suppliers` entity:

```bash
mkdir srv/external/data/ \
  && cds add data \
    --filter Suppliers \
    --records 5 \
    --out srv/external/data/
```

This results in:

```log
adding data
  creating srv/external/data/northbreeze.Suppliers.csv

successfully added features to your project
```

👉 That data is pretty useful already - check with:

```bash
curl -s localhost:5005/odata/v4/northbreeze/Suppliers | jq .value
```

which should show output similar to this (massively reduced here for brevity):

```json
[
  {
    "SupplierID": 3865327,
    "CompanyName": "CompanyName-3865327",
    "ContactName": "ContactName-3865327",
    "ContactTitle": "ContactTitle-3865327",
    "Address": "Address-3865327",
    "City": "City-3865327",
    "Region": "Region-3865327",
    "PostalCode": "PostalCode-3865327",
    "Country": "Country-3865327",
    "Phone": "Phone-3865327",
    "Fax": "Fax-3865327",
    "HomePage": "HomePage-3865327"
  },
  {
    "SupplierID": 3865328,
    "CompanyName": "...",
  }
]
```

#### Retrieve, store and use data from the real service

But we can do better. Why not grab and store some "real" data from the actual service, and use it when we mock?

👉 First, remove the CSV file we just generated:

```bash
rm srv/external/data/northbreeze-Suppliers.csv
```

👉 Now retrieve the entitysets and put the data into files in that `srv/external/data/` directory:

```bash
for entity in Products Suppliers Categories; do
  echo -n "$entity: "
  curl \
    --silent \
    --url "https://developer-challenge.cfapps.eu10.hana.ondemand.com/odata/v4/northbreeze/$entity" \
    | jq .value \
    | tee "srv/external/data/northbreeze-$entity.json" \
    | jq length
done
```

This should result in the retrieval of a number of records for each entityset:

```log
Products: 77
Suppliers: 29
Categories: 8
```

and the creation of the corresponding files, this time in JSON format:

```
srv/
├── admin-service.cds
├── admin-service.js
├── cat-service.cds
├── cat-service.js
├── ex01-service.cds
└── external
    ├── data
    │   ├── northbreeze-Categories.json
    │   ├── northbreeze-Products.json
    │   └── northbreeze-Suppliers.json
    ├── northbreeze.csn
    └── northbreeze.edmx
```

👉 Now restart the mock service:

```bash
cds mock northbreeze --port 5005
```

👉 and re-request the same entityset:

```bash
curl -s localhost:5005/odata/v4/northbreeze/Suppliers | jq .value
```

This time, the data is more realistic, as it's the actual data we fetched from the real service (again, heavily reduced here for brevity):

```json
[
  {
    "SupplierID": 1,
    "CompanyName": "Exotic Liquids",
    "ContactName": "Charlotte Cooper",
    "ContactTitle": "Purchasing Manager",
    "Address": "49 Gilbert St.",
    "City": "London",
    "Region": "NULL",
    "PostalCode": "EC1 4SD",
    "Country": "UK",
    "Phone": "(171) 555-2222",
    "Fax": "NULL",
    "HomePage": "NULL"
  },
  {
    "SupplierID": 2,
    "CompanyName": "..."
  }
]
```

Great! Now we have a fully mocked external service complete with real data.

### Access the mocked remote service from the cds REPL (bonus)

If you have time, you can build on your knowledge of and confidence with the cds REPL by connecting to this mocked remote service from within the cds REPL.

👉 First, let's have a look at the "wiring" for this mocked remote service in our local development mode context; take a peek in the `.cds-services.json` file in your home directory:

```bash
jq . ~/.cds-services.json
```

This is a file that the CAP server runtime uses in local development mode to declare and detail which services are (being) provided, and where. It will look something like this (the server IDs are process IDs so will be different for you):

```json
{
  "cds": {
    "provides": {
      "northbreeze": {
        "kind": "odata",
        "credentials": {
          "url": "http://localhost:5005/odata/v4/northbreeze"
        },
        "server": 124021
      }
    },
    "servers": {
      "124021": {
        "root": "file:///work/scratch/myproj",
        "url": "http://localhost:5005"
      }
    }
  }
}
```

👉 Now start the cds REPL:

```bash
cds repl
```

👉 and in the prompt, [connect to the remote service] and store that connection in a variable:

```text
nb = await cds.connect.to("northbreeze")
```

This will emit the internal representation of this connection, which you can get a summary of using the `.inspect` command which you learned about [in a previous exercise], like this:

```bash
> .inspect nb .depth=0
nb: RemoteService {
  name: 'northbreeze',
  options: [Object],
  kind: 'odata',
  model: [LinkedCSN],
  handlers: [EventHandlers],
  definition: [service],
  namespace: 'northbreeze',
  actions: [LinkedDefinitions],
  selectProduct: [Function: northbreeze.selectProduct],
  entities: [LinkedDefinitions],
  _source: '/work/scratch/myproj/node_modules/@sap/cds/libx/_runtime/remote/Service.js',
  datasource: undefined,
  destinationOptions: undefined,
  destination: [Object],
  path: undefined,
  requestTimeout: 60000,
  csrf: undefined,
  csrfInBatch: undefined,
  middlewares: [Object]
}
```

👉 Now, still at the cds REPL prompt, construct a query on the fly and send it across the connection to your locally mocked version of the remote Northbreeze service:

```text
await nb.run(SELECT `CompanyName` .from(`Suppliers`))
```

> Remember that pretty much everything in this context is going to be asynchronous, i.e. in a Promise wrapper, so `await` is needed here to realise the call.

This results in:

- an actual OData call from the cds REPL context across to the mocked Northbreeze service on port 5005
- the retrieval of the Suppliers entityset, specifically the `CompanyName` property for each entity

```text
[
  { CompanyName: 'Exotic Liquids', SupplierID: 1 },
  { CompanyName: 'New Orleans Cajun Delights', SupplierID: 2 },
  { CompanyName: "Grandma Kelly's Homestead", SupplierID: 3 },
  { CompanyName: 'Tokyo Traders', SupplierID: 4 },
  { CompanyName: "Cooperativa de Quesos 'Las Cabras'", SupplierID: 5 },
  { CompanyName: "Mayumi's", SupplierID: 6 },
  { CompanyName: 'Pavlova Ltd.', SupplierID: 7 },
  ...
  { CompanyName: 'Gai pâturage', SupplierID: 28 },
  { CompanyName: "Forêts d'érables", SupplierID: 29 }
]
```

Excellent!

---

## Further reading

- The [Authentication] topic in Capire
- The [CDS-based Authorization] topic in Capire
- The contents of the [Service integration with SAP Cloud Application Programming Model] CodeJam
- [Part 4 - digging deeper] of [Level up your CAP skills by learning to use the cds REPL]

## Questions

If you finish earlier than your fellow participants, you might like to ponder these questions. There isn't always a single correct answer and there are no prizes - they're just to give you something else to think about.

## Footnotes

<a name="footnote-1"></a>
### Footnote 1

[authentication strategy]: https://cap.cloud.sap/docs/node.js/authentication#strategies
[assignment of sample roles to pre-defined test users]: https://cap.cloud.sap/docs/node.js/authentication#mock-users
[@requires]: https://cap.cloud.sap/docs/guides/security/authorization#requires
[Authentication]: https://cap.cloud.sap/docs/node.js/authentication
[CDS-based Authorization]: https://cap.cloud.sap/docs/guides/security/authorization
[authentication is a prerequisite]: https://cap.cloud.sap/docs/guides/security/authorization#prerequisite-authentication
[401]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/401
[403]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/403
[added a new service definition]: ../01/README.md#add-a-new-service-definition
[project-local .cdsrc.json]: https://cap.cloud.sap/docs/node.js/cds-env#in-cdsrc-json
[CSN]: https://cap.cloud.sap/docs/cds/csn
[Service integration with SAP Cloud Application Programming Model]: https://github.com/SAP-samples/cap-service-integration-codejam/
[connect to the remote service]: https://cap.cloud.sap/docs/node.js/cds-connect#cds-connect-to-1
[in a previous exercise]: ../02/README.md#use-the-cds-repl-to-explore-path-expression-features-with-sqlite
[Part 4 of Level up your CAP skills by learning to use the cds REPL]: https://qmacro.org/blog/posts/2025/03/21/level-up-your-cap-skills-by-learning-how-to-use-the-cds-repl/#part-4-digging-deeper
[Part 4 - digging deeper]: https://qmacro.org/blog/posts/2025/03/21/level-up-your-cap-skills-by-learning-how-to-use-the-cds-repl/#part-4-digging-deeper
[Level up your CAP skills by learning to use the cds REPL]: https://qmacro.org/blog/posts/2025/03/21/level-up-your-cap-skills-by-learning-how-to-use-the-cds-repl/
