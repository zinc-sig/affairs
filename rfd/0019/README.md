---
authors: Thomas Li
state: discussion
discussion: https://github.com/stdx-space/zinc-affairs/pull/1
labels: direction, interop
---

# [RFD] Pagination of List Endpoints: Response Headers, Offset and Cursor

List endpoints in the core and the extensions page their results with query
parameters and return a bare JSON array. No response tells a client whether
more rows exist, so every client decides that a list has ended when a page is
shorter than the limit it asked for, and several clients read a truncated
list as the whole list. This RFD defines one response contract for paginated
lists. The body stays a bare array. The page metadata moves into response
headers: `Link` ([RFC 8288](https://www.rfc-editor.org/rfc/rfc8288)) with the
`self` and `next` relations, and `X-Total-Count` on the endpoints whose
screens show a total. The contract has two variants, offset and cursor, which
a client consumes the same way. Page-size defaults and the maximum move out
of SQL into one Go package.

The RFD records the decisions, their reasons, and the rejected alternatives.
It covers the server contract, how zinc-sig/ui and the Go clients adopt it,
and the order of the rollout across repositories.

## Background

### How lists are paginated

Verified against the core repository at the time of writing:

- **Request.** The struct `override.PaginationOptions`, with `limit`,
  `offset`, and `page`, is embedded in 21 list query types. `page` is bound
  and documented on `/activities` and `/courses` and is read nowhere.
- **Defaults in SQL.** Each paginated query ends in
  `LIMIT COALESCE(sqlc.narg('limit'), N) OFFSET COALESCE(sqlc.narg('offset'), 0)`.
  N is 10 or 20 depending on the query, and 100 for the sandbox attempt
  counts. A caller that passes no limit receives N rows.
- **Service code.** The services hold seventeen copies of the same two
  conditionals that copy a positive limit and offset into the query
  parameters. Two packages have a local helper; the pipeline one clamps the
  limit to 500. No other endpoint has a maximum.
- **Response.** A bare JSON array. No response carries a total, a flag for
  more rows, or a reference to the next page.
- **One token-paged endpoint.** `GET /pipeline/workflows` takes `page_size`
  and `next_page_token` and returns an envelope with the next token, because
  Temporal's visibility API has no offset.
- **Unpaginated lists.** 24 GET endpoints return an array and take no paging
  parameters.

The core repository's written conventions, the API package guide and the
query conventions document, cover the request struct and the SQL clause.
They do not define a response.

### Consequences

Every client ends a list on a short page: `useFetchAll` in zinc-sig/ui,
`listAllPages` in zinc-mcp, and the exam bundle reader. The rule fails in two
ways. A server can return fewer rows than asked for before the end: the
pipeline run list drops superseded runs after the SQL limit is applied. A
caller can send no limit and receive the SQL default as if it were the whole
list.

The second failure is visible to users:

- The console's grading panel reads an activity's scores with no limit,
  receives the 10 most recently updated, and searches them for the delivery
  on screen. For other students the score breakdown does not render.
- The console home page loads 10 terms and 10 courses.
- The attempt timeline shows the 10 newest pipeline runs of a delivery.
- In the Go SDK, `CollectionsClient.List` and `DeliveriesClient.List` take no
  limit and are documented as returning all rows; they return 10. Most
  zincli list commands print the server's default first page.
- Internal callers that need every row either pass a large number to escape
  the default (`allRows = 100000` in the grading roster, `Limit: 1000` when
  resolving a committed delivery) or pass nothing and receive 10 (the run
  read of the grading snapshot).

Other defects in the same area:

- `/documents` accepts `limit` and `offset` and ignores them. A client that
  pages it with a page size at or below the number of documents never
  terminates, because every page is the full list.
- `/formulas` returns at most 10 rows because of its query, while its API
  description says it returns every formula and declares no paging
  parameters.
- The sort override on the logistics lists replaces the ORDER BY with one
  column and drops the unique tie-breaker that the query conventions
  require. Offset pages over a sort by name or status can skip or repeat
  rows.

## Proposal

- **Headers carry the page metadata; the body stays a bare array.** Every
  paginated response has `Link` with `rel="self"`, and with `rel="next"` when
  another page exists. Four endpoints also send `X-Total-Count`.
- **Two variants, one response contract.** The offset and cursor variants
  differ in the position parameter a request carries. The response headers
  and the end-of-list rule are identical.
- **A client ends a list only when `next` is absent.** Page length carries no
  meaning.
- **Defaults and the maximum live in Go.** The default page size is 50 and
  the maximum is 1000. An endpoint may have a lower maximum only when its
  data source requires it, with the reason in a comment. SQL takes a required
  limit; a read that needs every row uses its own unpaginated query.
- **Out-of-range input is handled uniformly.** An omitted or zero limit
  selects the default, a limit above the maximum is clamped, and a negative
  limit or offset is rejected with 400.
- **Each client reads the headers in one place.** In zinc-sig/ui a
  `fetchPage` helper wraps the generated client functions. In Go a shared SDK
  transport returns the parsed page, and list methods return a page or
  iterate over every page.
- **A caller that presents a list as complete reads every page.** This holds
  for the console, the exam client, zincli, zinc-mcp, and the exam bundle
  reader.

### Abandoned ideas

- **A response envelope** (`{items, limit, offset, has_more, total}`). It
  changes the body of every list response, so the core, zinc-sig/ui, the Go
  SDK, zincli, and zinc-mcp would have to release together, and zincli
  binaries built before the change would fail on list commands. With headers
  the body does not change: the core releases first and each client adopts
  the headers one endpoint at a time. The envelope's advantage, a total typed
  in the generated TypeScript, costs one parsing helper per client to
  replace.
- **Paging parameters in request headers**, either `Range: items=0-49` or a
  custom header. The React Query keys that kubb generates include path and
  query parameters and omit request headers, so two pages of one list would
  share one cache entry. A custom request header also makes every paginated
  GET a preflighted CORS request; the core sends no `Access-Control-Max-Age`,
  so browsers cache each preflight for five seconds. `Range` brings semantics
  that do not fit a list: RFC 9110 lets a server ignore it, and defines
  `Content-Range` only for 206 and 416 responses.
- **One variant for every endpoint.** Offset cannot express Temporal's
  listing, which has no offset. Cursors everywhere remove random page access
  from tables that jump to a page or change the page size, and a cursor over
  a caller-chosen sort column needs a composite key.
- **A page-number parameter.** It duplicates `offset`; `page` was accepted
  and ignored.
- **Defaults in SQL.** A caller that forgets the limit receives the default
  as if it were the whole list, the defaults differ between queries, and
  internal callers pass large numbers to escape them.
- **A shared unbounded constant passed through paginated queries** for
  internal reads. The next caller can forget it, which reproduces the silent
  default. A dedicated query states in its comment why it is unbounded.
- **A default and maximum chosen per endpoint.** No endpoint needs a
  different default. The one lower maximum, on workflows, is expressed as a
  documented exception.
- **A separate count query per endpoint.** The filter would exist in two
  queries that can drift apart. A window function keeps one filter.
- **Rejecting a limit above the maximum with 400.** A client that asks for
  more than the maximum would fail where a smaller page serves it. Because
  `self` and `next` carry the applied limit, clamping loses no information.
- **The `prev`, `first`, and `last` relations.** Offset clients compute them
  from `self`. A cursor source cannot supply `prev`: Temporal pages forward
  only.
- **Path-absolute or absolute link targets.** The core would need to know
  the external host, the scheme, and any path prefix a proxy adds. A
  query-only reference resolves against the request URL
  ([RFC 3986, section 5.4.1](https://www.rfc-editor.org/rfc/rfc3986#section-5.4.1)).
- **Reading the headers through kubb's `dataReturnType: "full"`.** In kubb 4
  the full result is the raw axios response. Server loaders dehydrate query
  data into the page, and the response's `config.headers` carries the session
  cookie, so the option needs a custom client module, selected through
  `importPath`, to strip it. Kubb 5 removes both `dataReturnType` and
  `importPath`, so the mechanism would be rebuilt at that upgrade, and about
  200 call sites would change type in between.
- **Upgrading zinc-sig/ui to Kubb 5 before adopting the headers.** Kubb 5
  returns the response from every call, but the upgrade is unrelated work,
  and pagination would wait for it.
- **Converting only the paging call sites in zinc-sig/ui.** The call sites
  that treat one response as the whole list would still truncate, at 50 rows
  in place of 10 or 20.
- **A lint rule in zinc-sig/ui against direct calls to list operations.** The
  rule would have to name the list operations, a set that changes with the
  API. Review enforces the rule.
- **Keeping the Go list signatures and adding `ListAll` methods, or offering
  iterators only.** Either way a single-page caller cannot see `next` or the
  total, so it cannot report that more rows exist.
- **Header capture in each of the seven SDK HTTP helpers.** Seven copies of
  the same parsing. A shared transport also gives every SDK package the same
  error type.
- **Folding the Go client work into the migration change.** The transport
  refactor and the SDK signature changes touch different files from the
  server contract and are reviewed on their own.
- **One page by default in zincli**, with an `--all` flag or a note on
  standard error. A command named `list` would return part of a list by
  default, while scripts and the end-to-end scenarios read its output as the
  whole list.

## The response contract

### Rules for both variants

1. The response body is a JSON array of the page's rows.
2. `limit` selects the page size. An omitted or zero value selects the
   default of 50. A value above the endpoint's maximum is clamped to it. A
   negative value is rejected with 400, as is a value that is not an
   integer.
3. Every response carries `Link` with `rel="self"`. Its target holds the
   limit that was applied and the position of the page.
4. A response carries `Link` with `rel="next"` if and only if another page
   exists.
5. A client treats only the absence of `next` as the end of the list. A
   short or empty page that carries `next` is not the end.
6. A link target is a query-only relative reference, for example
   `<?limit=50&offset=100>`. The server builds it from the request's query
   string, replaces the position parameter and `limit`, and keeps every other
   parameter, so filters and sort carry over. Keys are sorted, so identical
   requests produce identical links.
7. The server emits one `Link` field with comma-separated values, as RFC
   8288 allows.
8. The core sends `Access-Control-Expose-Headers: Link, X-Total-Count`. Both
   frontends call the core cross-origin, and without the exposure a browser
   client cannot read either header.

### Offset variant

- The position parameter is `offset`, an integer of zero or more. A negative
  offset is rejected with 400.
- `self` carries `limit` and `offset` explicitly, including an offset of
  zero.
- The server requests `limit + 1` rows from the query. The extra row
  decides `next` and is not returned.
- An offset past the end returns `200` with an empty array, `self`, and no
  `next`.
- A client may build any offset, which allows a table to jump to a page or
  change its page size. A client that computes offsets itself uses the
  `limit` from `self`, not the one it requested, so a clamped limit does not
  make it skip rows.
- `X-Total-Count` is sent only on the endpoints listed below. It is the row
  count of the filtered list, computed in the same query with
  `COUNT(*) OVER ()`. It is omitted when the page is empty, because the
  window function has no row to report on.
- The query's ORDER BY ends in a unique tie-breaker. The query conventions
  already require this; the sort override follows it too.

### Cursor variant

- The position parameter is `cursor`, an opaque string. A client obtains it
  only from a `next` link and never builds or edits it.
- A cursor is valid only with the other parameters of the link that carried
  it. A client that changes a filter starts again from the first page.
- The server encodes the source's token as base64url without padding, so the
  value needs no percent-encoding in a query string.
- An undecodable cursor, or one the source rejects, is rejected with 400.
- `self` carries the cursor the request used, and has none on the first
  page.
- A cursor endpoint sends no `X-Total-Count`.

### Examples

```
GET /v1/users?limit=50&offset=100&sort=name

200 OK
Link: <?limit=50&offset=100&sort=name>; rel="self", <?limit=50&offset=150&sort=name>; rel="next"
X-Total-Count: 437

GET /v1/deliveries?collection_id=7&limit=5000

200 OK
Link: <?collection_id=7&limit=1000&offset=0>; rel="self", <?collection_id=7&limit=1000&offset=1000>; rel="next"

GET /v1/pipeline/workflows?limit=50&status=running

200 OK
Link: <?limit=50&status=running>; rel="self", <?cursor=CgZhYmNkZWY&limit=50&status=running>; rel="next"
```

## Endpoint assignment

Every endpoint uses the default of 50. The maximum is 1000 except where
listed.

| Endpoint | Variant | `X-Total-Count` | Notes |
|---|---|---|---|
| `/users` | offset | yes | |
| `/terms` | offset | yes | |
| `/courses` | offset | yes | |
| `/activities` | offset | yes | |
| `/courses/{course_id}/students` | offset | no | |
| `/courses/{course_id}/staff` | offset | no | |
| `/courses/{course_id}/groups` | offset | no | |
| `/collections` | offset | no | |
| `/deliveries` | offset | no | |
| `/prescreens` | offset | no | |
| `/activities/{activity_id}/scores` | offset | no | |
| `/documents` | offset | no | paginated by this RFD |
| `/documents/{document_id}/questions` | offset | no | |
| `/documents/{document_id}/context` | offset | no | |
| `/documents/{document_id}/context/relation` | offset | no | |
| `/documents/{document_id}/appendix` | offset | no | |
| `/environment-templates` | offset | no | |
| `/formulas` | offset | no | parameters documented by this RFD |
| `/pipeline/configs` | offset | no | |
| `/pipeline/runs` | offset | no | |
| `/pipeline/runs/config/{config_id}` | offset | no | |
| `/pipeline/workflows` | cursor | no | maximum 100 |

The totals go to the four endpoints whose console tables are paged on the
server and show a count. Other lists are either read in full by the client,
which counts the rows itself, or show no count.

The workflows maximum stays at 100 because each workflow row runs an extra
query for its linked-run count.

The run logs, `/pipeline/runs` and `/pipeline/runs/config/{config_id}`, are
append-only and ordered by creation time, which suits a keyset cursor: under
offsets, a run inserted while a reader pages shifts the later rows. They stay
on offset because no console table pages over them, and a client that
follows `next` does not change when an endpoint changes variant.

## Server implementation

### The listing package

One package in the core writes the headers and resolves the request. The
sketch below gives the shape; names are implementation choices.

```go
// Bounds are an endpoint's default and maximum page size.
type Bounds struct{ Default, Max int }

// Standard is the bound for every endpoint without a documented exception.
var Standard = Bounds{Default: 50, Max: 1000}

type OffsetQuery struct {
	Limit  int `query:"limit"`
	Offset int `query:"offset"`
}

type CursorQuery struct {
	Limit  int    `query:"limit"`
	Cursor string `query:"cursor"`
}

// Window is a resolved offset request. Fetch is Limit+1: the row count to
// request so that the existence of a next page is known.
type Window struct{ Limit, Offset, Fetch int }

func (q OffsetQuery) Resolve(b Bounds) (Window, error)
func (q CursorQuery) Resolve(b Bounds) (limit int, err error)

// Offset trims rows to w.Limit and writes the self and next links.
func Offset[T any](c *echo.Context, w Window, rows []T) []T

// Cursor writes the self link, and the next link when next is non-empty.
func Cursor(c *echo.Context, limit int, used, next string)

// Total writes X-Total-Count.
func Total(c *echo.Context, n int64)
```

`OffsetQuery` and `CursorQuery` replace `override.PaginationOptions` in the
list query types. The two local helpers and the seventeen copied conditionals
are removed.

### SQL

A paginated query takes a required limit and offset and has no default:

```sql
-- name: ListUsers :many
SELECT *, COUNT(*) OVER () AS total_count
FROM core.user
ORDER BY id DESC
LIMIT sqlc.arg('limit')::INT
OFFSET sqlc.arg('offset')::INT;
```

The generated parameters become `int`. A caller that omits the limit
requests zero rows, which fails any test that reads the list, where a
default of 10 would pass a test with fewer than 10 rows. The
`total_count` column appears only in the four queries that back a total. The
hand-written sorted variants in the ORM scan it where it exists, and the sort
override appends the id tie-breaker in the requested direction.

For `/pipeline/runs`, the filter that drops superseded runs moves from Go
into the query, so a page is full whenever more rows exist.

### Internal reads of every row

Each internal caller that reads a paginated query in full gets its own
unpaginated query, with the comment the query conventions require for an
unbounded `:many`:

| Caller | Rows | At the time of writing | Replacement |
|---|---|---|---|
| grading roster (report) | runs of one collection | limit of 100000 | unpaginated query, bounded by the collection |
| grading roster (report) | current scores of one activity | limit of 100000 | unpaginated query, bounded by the activity |
| grading snapshot (pipeline) | runs of one delivery | no limit, receives 10 | current runs of the delivery, filtered on `superseded_at IS NULL` in SQL |
| committed-delivery resolution (submission) | committed deliveries of one user in one collection | limit of 1000 | unpaginated query |
| submission export (examination) | contexts of one document | limit equal to the layout row count | unpaginated query |

Two further queries depend on a SQL default and are resolved the same way:

- **The sandbox attempt counts.** The attempt-count list for one question is
  an unpaginated endpoint with one row per student who ran code on the
  question. Its query is capped at 100 rows by its SQL default and ordered by
  last attempt time alone. It becomes an unpaginated query, bounded by the
  students of the course and ordered by `last_attempt_at DESC, user_id`.
- **Queries with no caller.** `ListCourseStudyingUsers`,
  `ListRunAttemptCountsByDocument`, and `ListPoolConfigs` are referenced only
  by their generated code. They are deleted.

### Swagger annotations

Each paginated route declares its parameters and response headers. swag
attaches a `@Header` line only to a status already declared, so it follows
`@Success`.

```go
// @Param   limit  query int    false "Page size (default 50, max 1000)"
// @Param   offset query int    false "Rows to skip"
// @Success 200 {array} User
// @Header  200 {string}  Link          "RFC 8288: self, and next when more rows exist"
// @Header  200 {integer} X-Total-Count "Rows matching the filters"
```

A cursor route declares `cursor` in place of `offset` and no total.

## Tests and enforcement

1. **Unit tests for the listing package.** Defaults, clamping, and the 400
   cases; `self` and `next` keep the filters and sort with sorted keys;
   `next` is absent on the last page; a cursor passes through unchanged; the
   total is omitted on an empty page.
2. **A test over the generated swagger document**, which needs no running
   stack: every GET that takes `limit` declares `Link` and takes exactly one
   of `offset` and `cursor`.
3. **A page walk in each endpoint's integration test.** Seed five rows, read
   with `limit=2` by following `next`, and assert that no row repeats, none
   is missing, and the last page has no `next`. The four total endpoints also
   assert `X-Total-Count`.
4. **Targeted integration tests.** `/pipeline/runs` returns full pages when
   superseded rows exist. Each unpaginated internal query returns more rows
   than the old default.
5. **A query lint rule.** `just boundary-lint` rejects
   `COALESCE(sqlc.narg('limit')`, so no query reintroduces a default in SQL.

Tests that encode the old behaviour change with the endpoints they cover:
the zinc-mcp short-page test, the report and pipeline unit tests that pin a
nil limit and the 500 clamp, the environment integration test that expects
10 of 10 templates from the default, and the exam bundle integration reader,
which does not page and relies on the default of 20.

## Clients

### zinc-sig/ui

The generated kubb clients return `res.data` and discard the headers. Each
generated client function accepts a per-call `client` option, and the helper
below uses it to see the response. The kubb configuration and the generated
code do not change.

- **`fetchPage` in `packages/sdk`.** It calls a generated client function
  with a per-call client that delegates to kubb's axios client and parses
  `Link` and `X-Total-Count` from the response. It returns a plain object,
  `{ items, page }`, where `page` holds the parsed `self`, `next`, and
  `total`. The plain object survives the dehydration of loader data into the
  page, and it carries neither the request configuration nor the session
  cookie that server loaders send in it.
- **`useFetchAll`** takes a fetch function that returns `{ items, page }` and
  requests the page in `page.next` until `next` is absent. Its 38 console call
  sites change their fetch function.
- **The server-paged tables**, which are the four admin lists, the course
  page's activity list, and the workflows admin page, query through
  `fetchPage` under a key that extends the generated query key, so
  invalidating the generated key still reaches them. They read `self`,
  `next`, and `total`.
- **Every call site that treats one response as the whole list reads every
  page**: through `useFetchAll` in the console, and through an equivalent
  helper in the exam client. This covers the requests with `limit: 100` or
  `limit: 1000` and the requests with no limit, about 35 call sites. The exam
  client's check that fails at 1000 rows is removed.
- **Kubb 5.** The upgrade replaces `res.data` with a result that carries the
  response. Only the body of `fetchPage` changes at that upgrade.

### Go clients

- **One SDK transport.** A single transport inside the SDK replaces the HTTP
  helpers of the seven packages. It decodes the body, parses the page from
  the headers, and builds errors with `ParseAPIError`. The pipeline and
  report packages build plain formatted errors at the time of writing; they
  return `*sdk.APIError` like the other packages, which changes how zincli
  prints their errors.
- **List methods return a page, or iterate over every page.** Names in the
  sketch are implementation choices.

```go
// Position is where a page starts: an offset, or a cursor from a next link.
type Position struct {
	Limit  int
	Offset int
	Cursor string
}

type Page[T any] struct {
	Items []T
	Next  *Position // nil on the last page
	Total *int64    // set only by endpoints that send X-Total-Count
}

func (c *UsersClient) List(ctx context.Context, q UsersQuery) (Page[User], error)
func (c *UsersClient) All(ctx context.Context, q UsersQuery) iter.Seq2[User, error]
```

The `List(ctx, limit, offset)` signatures are replaced, and the four list
methods that take no limit take the same shape. Every consumer is inside the
core repository and changes in the same change.

- **zinc-mcp and the exam bundle reader** iterate with `All` in place of
  their short-page loops.
- **zincli list commands** read every page through `All` and print one
  output in the selected format. `--limit` and `--offset` request a single
  page; when that page has a `next` link, zincli writes the next offset to
  standard error.

## Delivery

1. Review of this RFD.
2. **The console defects listed in the background**, fixed in separate
   changes. They do not depend on the contract.
3. **Core plumbing change:** the listing package, the CORS exposure, and the
   edits to the API package guide and the query conventions document.
4. **Core migration change:** every endpoint in the assignment table, the
   SQL shape, the unpaginated and deleted queries, the fixes listed under
   Server implementation, the removal of `page`, the swagger test, and the
   lint rule. It merges together with the zinc-sig/ui change to the
   workflows admin page.
5. **Core Go client change:** the shared SDK transport, the list method
   shape, zincli, zinc-mcp, and the exam bundle reader.
6. **zinc-sig/ui adoption:** `fetchPage`, `useFetchAll`, the server-paged
   tables, and the call sites that read a whole list. It merges after the
   migration change is deployed, because the browser needs the headers and
   their CORS exposure.

Steps 5 and 6 do not depend on each other. Clients have no fallback for a
response without `Link`: under the contract a missing `next` ends the list,
so a client that meets an older core reads one page. The order above
prevents that combination.

The migration change is compatible with every client except one. Bodies do
not change. Defaults rise from 10 or 20 to 50, so a caller that sends no
limit receives more rows. The maximum of 1000 is at or above every page size
a client requests. The console's `/documents` callers filter by activity,
which has one or two documents. The exception is `/pipeline/workflows`,
whose body and parameters change; its only consumer is the console's
workflows admin page, which changes with it.

Every decision in this RFD is settled for review. The items below are
outside its scope.

### Deferred

- **The authorization ceiling on global lists.** `/activities`, `/courses`,
  `/collections`, `/deliveries`, `/documents`, and `/formulas` first ask
  OpenFGA which objects the caller can read. ListObjects returns at most 1000
  objects by default and does not indicate that it stopped early, and no
  deployment raises the limit. No page can reach a row past that set, and
  the totals on `/courses` and `/activities` are bounded by it. The remedy
  needs its own design.
- **The unpaginated array endpoints.** They stay as they are; an audit
  decides which need paging.
- **A keyset cursor for the run logs.**
- **Paged terminal output for zincli.** When the output format is a table and
  standard output is a terminal, zincli pipes the table through `$PAGER`, in
  the way git does, with `less -FRX` as the default and `--no-pager` to turn
  it off. `-F` exits at once when the output fits one screen, and `-X` leaves
  the rows in the terminal's scrollback. An interactive table built on
  bubbletea with lazy page fetching was considered and set aside: it adds a
  second renderer for every list command, its alternate screen clears the
  rows on exit, and lazy fetching saves little when a list takes one to three
  requests at a page size of 1000. It becomes worth building if list commands
  need actions on rows.
- **`Content-Disposition` exposure.** The console reads it to name a
  submission export download. It is not in the exposed headers, so a browser
  client cannot read it and the download falls back to a generated name.
  This is inferred from the code and was not observed on a deployment.

## UX

Browser clients read `Link` and `X-Total-Count` through the CORS exposure.
The console's server-paged tables show the range and total on the four total
endpoints and enable the next-page control from `next`. Every console and
exam client view that shows a whole list reads every page.

zincli `list` commands print every row by default, in the table, JSON, or
YAML format selected. `--limit` and `--offset` print one page and write the
next offset to standard error when more rows exist. SDK errors from the
pipeline and report packages print in the same form as the other packages.
zinc-mcp's list tools return whole lists.

For other API consumers the body of every list response is unchanged, except
`/pipeline/workflows`, which returns a bare array and takes `limit` and
`cursor` in place of `page_size` and `next_page_token`.
