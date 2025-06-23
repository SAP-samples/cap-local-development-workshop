# Exercise 03 - mocking authentication and required services

There's no avoiding the fact that if you want your CAP apps and services to be useful, they're going to have to make use of authentication mechanisms, and consume other APIs, in the cloud. In local development mode, however, these requirements can get in the way and hinder progress.

Fortunately the "developer centric" nature of CAP's local-first strategy provides various ways to not _ignore_ the reality, but to _suspend_ it until it's really needed. With the "mocking" approach, we can design and declare our domain model and adorn it with annotations relating to authentication, and rely on mocked authentication while still thinking about, defining and testing user role and attribute based access control. We can also have required services mocked for us so we can connect to them from our own.

In this exercise we'll explore both those topics.

## Observe the current state of authentication

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

The "mocked" [authentication strategy] uses HTTP Basic Authentication (simple usernames and passwords) combined with [assignment of preconfigured roles to some test users]. It's more or less the same as the "basic" authentication strategy, but with this user and role pre-configuration.

### Try accessing AdminService resources

The `AdminService` (currently made available at `/odata/v4/admin`) has a [@requires] annotation which declares that only users with the `admin` role have access:

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
curl -i -s localhost:4004/odata/v4/admin/Books
```

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
curl -i -s -u 'yves:' localhost:4004/odata/v4/admin/Books
```

> The colon separates the username and password when supplying this detail via `curl`'s `--user` (or `-u`) option. None of the pre-defined users have passwords.

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

👉 Repeat the same resource request but this time as user "alice", who does have the "admin" role:

```bash
curl -s -u 'alice:' localhost:4004/odata/v4/admin/Books \
  | jq .value[].title
```

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

---

## Further reading

- The [Authentication] topic in Capire
- The [CDS-based Authorization] topic in Capire

## Questions

If you finish earlier than your fellow participants, you might like to ponder these questions. There isn't always a single correct answer and there are no prizes - they're just to give you something else to think about.

## Footnotes

<a name="footnote-1"></a>
### Footnote 1









[authentication strategy]: https://cap.cloud.sap/docs/node.js/authentication#strategies
[assignment of preconfigured roles to some test users]: https://cap.cloud.sap/docs/node.js/authentication#mock-users
[role]: https://cap.cloud.sap/docs/node.js/authentication#mock-users
[@requires]: https://cap.cloud.sap/docs/guides/security/authorization#requires
[Authentication]: https://cap.cloud.sap/docs/node.js/authentication
[CDS-based Authorization]: https://cap.cloud.sap/docs/guides/security/authorization
[authentication is a prerequisite]: https://cap.cloud.sap/docs/guides/security/authorization#prerequisite-authentication
[401]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/401
[403]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/403
