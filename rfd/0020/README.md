---
authors: Thomas Li
state: discussion
discussion: "[#21](https://github.com/zinc-sig/affairs/pull/21)"
labels: direction, platform
---

# [RFD] Instance Settings and Secrets

An instance owner has no way to change instance-wide policy without asking
ops to edit the deployment config and restart the processes. This RFD adds
**instance settings**: values an instance administrator changes through the
API and the console, stored one row per setting in the core schema, read by
the core and the extensions at the point of use, and applied without a
restart. It also sets the rules for **secrets** that an administrator
enters, such as an API key for a model provider, which do not belong in the
settings table.

The RFD's main content is a set of rules that decide where a value lives:
deployment config, an instance setting, a secret connection, a scoped
setting, domain data, or state. The storage and API follow from those rules.

## Background

Every process reads a YAML config file at startup. The core reads the full
file. Each extension reads a small local file (broker, logger, transport,
authority) and fetches the rest from the core once at boot over NATS, through
the `extensions.get_configuration` request: database DSN, object storage
keys, cache, OpenFGA, Temporal, DinD, registry, LiveKit, telemetry, and the
session cookie fields. The extension builds its long-lived clients from that
response and keeps it. A change to the core's config therefore reaches an
extension only when the extension restarts. In September 2026 this dependency
was implicit, and operators missed it. Since chart 0.3.6
([zinc-sig/cobe-deployment#9](https://github.com/zinc-sig/cobe-deployment/pull/9)),
the Helm chart rolls the extension pods whenever the core's rendered config
changes. The HKBU Compose deployment recreates the extensions in the same
case, using a per-config content hash.

That path suits connection settings, which ops owns and which are needed to
construct a client. It does not suit policy that an instance owner decides:
a site name, a login notice, a default for new activities, a budget for a
grading feature. Today each of those would be a config key, owned by ops, and
applied by a restart.

[RFD 0018](../0018/README.md) added settings scoped to one activity: an
envelope stored on the activity row, one key per extension, typed in a Go
leaf package (`internal/activitysettings`), with an absent key resolving to a
constant default. Nothing equivalent exists for the instance as a whole.

Two instance-level secrets exist, and each solved storage on its own terms:

- The Microsoft Entra directory connection is a singleton table in the core
  schema (`core.directory_connection`). Its token set is sealed with
  AES-256-GCM under a key derived from the authority key, bound to the
  connection by additional authenticated data, and never returned by the API.
- The proctoring extension holds a versioned key-encryption key map in its
  own config file. The map is not delivered over the extension config bus.

Agentic grading will need at least one more secret: an API key for a model
provider, entered by an administrator.

The trust boundaries between processes are weaker than the schema split
suggests, and the secret rules below depend on that:

- All processes share one database login. The schema boundary between the
  core and the extensions is enforced by convention and by `boundary-lint`,
  not by grants.
- The authority key is identical in every process, so a key derived from it
  is available to every process.
- The extension config bus answers any requester on the broker.

## Proposal

1. **Placement rules.** A fixed, ordered set of tests decides where a value
   lives. Only values that pass every test become instance settings.
2. **One row per setting** in `core.system_setting`, keyed by `owner` and
   `name`, with a JSONB value. A setting may be a scalar or an object; an
   object is used when its fields are decided and validated together.
3. **A typed contract in a Go leaf package**, `internal/systemsettings`, one
   file per owner, declaring each setting's type, constant default,
   validator, and visibility. An absent row resolves to the constant default.
4. **The core writes, every process reads.** Writes go through the core's
   API, gated on `system#admin`, with optional `If-Match`. The core and the
   extensions read the table directly and cache it for a few seconds; a
   change notification on core NATS shortens the wait. No process restarts.
5. **Secrets are connections, not settings.** A secret lives in a table in
   the owning service's schema, sealed, written through a write-only API, and
   decrypted only by the owning service. A secret is never passed to a
   process that runs or reads untrusted content.
6. **Files are assets.** An uploaded file is stored in object storage with a
   row in `core.instance_asset`, keyed by its content hash. A setting refers
   to an asset by its hash, and an administrator assigns it through the
   ordinary settings write.
7. **Branding is the first set**: the product name, the institution name,
   the logo in two themes, the favicon, and the support link, owned by the
   core and public. An institution may rename the product; the About dialog
   names the software whatever the product name is.

### Abandoned ideas

- **Instance settings over the extension config bus.** The bus is a
  one-time fetch at boot, which is the restart dependency described above.
  Adding policy to it would make every policy change a restart.
- **NATS JetStream key-value as the store.** Every extension already reads
  the core schema, so a second stateful store adds backups and a second
  source of truth without giving any process data it cannot reach. NATS is
  kept for the change notification only.
- **One row per owner holding one object.** This mirrors the activity
  envelope and makes cross-field validation atomic. It was rejected because
  it records who changed a group, not a setting, makes two administrators
  editing unrelated settings of one owner conflict, and freezes every field
  of the object at its default of the day once any field is set. Object
  values per setting keep the grouped case available where it is wanted.
- **One dotted key column.** Reading one owner's settings becomes a `LIKE`
  prefix match, and `_` is a wildcard in `LIKE`. Separate columns make it an
  equality on the leading column of the primary key; the dotted form is
  joined where one string is needed.
- **A secret type inside the settings table.** The settings table is read by
  every process and written by the core, and a secret needs the opposite:
  written and read by its owner only, and never returned. A secret also has
  a lifecycle (connected, failing, replaced) that a settings row does not
  model.
- **Extensions advertising their settings schema at runtime.** Deferred for
  the reasons recorded for activity settings: a default or validator must
  not depend on whether an extension is running, so a runtime schema would
  need a persisted registry. The static Go table leaves room for it.
- **Hot reload of connection settings.** Applying a new DSN to a running
  process means rebuilding its pools and workers while requests are in
  flight. A restart does the same with fewer failure modes, and the
  deployment tooling performs it.
- **Asset bytes in the setting's JSON value.** Every process caches the whole
  settings table, and the administrator read returns every value. A logo
  encoded into a row would be held by every process and copied into every
  administrator read and every database dump.
- **An upload route that also writes the setting.** It is a second write path
  beside `PUT /v1/settings/{owner}/{name}`, and it would need its own
  `If-Match`, `updated_by`, and change notification. The generic route would
  still accept an asset reference, so its validator would have to check the
  asset table anyway. Separating upload from assignment makes that validator
  the one check.
- **Presigned object storage URLs for asset reads.** The object store is not
  reachable from browsers in every deployment, and a presigned URL expires,
  which conflicts with an immutable cache lifetime.
- **An asset read route keyed by setting name**, with the hash as a cache
  parameter. A request with an outdated parameter would need its own
  handling. A route keyed by hash returns the same bytes for every request,
  so every response is immutable.
- **A periodic sweep that deletes unused assets after a retention period.**
  It needs a timestamp per asset, a background job, and locking between the
  job and the settings write. Branding assets are few and small, so deletion
  from the settings page is enough. A setting whose assets accumulate can
  add the sweep.
- **A favicon derived from the logo.** A logo may be up to four times as
  wide as it is high, and a favicon is square. Deriving one from the other
  would also change the tab icon whenever an administrator changes the
  logo, which prevents an instance from showing the CoBE mark in the top
  bar and the institution's crest in the tab.
- **An institution logo shown beside the institution name.** It adds a
  layout to the top bar and the pages without a session. The favicon can
  carry the institution's mark, so the first set names the institution in
  text, and an institution logo can be added later as one more asset
  setting.
- **One display string for the product and the institution**, such as
  "CoBE · HKUST CSE". The top bar sets the two names in different type, and
  page titles use the product name alone. A client cannot split one string
  reliably.
- **The product name inserted into feature copy.** Every string that names a
  feature after the product would become a template, and every string added
  later would have to follow the same rule. Copy that names the feature
  itself reads correctly under any product name.
- **The default artwork as a built-in asset in the core**, served by the
  asset read route. Every page would wait on a request to the core to show
  the default mark, and a page rendered while the core is unreachable would
  show none. The console and the exam client already bundle the artwork. A
  third client needs its own copy; the desktop application has one in its
  build profiles.

## Where a value belongs

Apply the tests in order. The first test that matches decides the home.

| Test | If yes, the value is | Home |
|---|---|---|
| 1. Is it a credential or token? | a secret | a connection table in the owner's schema |
| 2. Is it needed to construct a client, open a listener, establish process identity, or reach the database? | connection config | deployment config |
| 3. Does it vary by activity, course, or user? | a scoped setting | the activity envelope, a course column, or a user preference |
| 4. Can there be many of it, each with an identity and a lifecycle? | domain data | its own table |
| 5. Is it written by the system rather than decided by a person? | state | the owning table |
| 6. Does it define a trust boundary of the deployment, or can a wrong value prevent the administrator from correcting it? | a deployment control | deployment config |
| 7. Should an instance administrator change it without ops and without a release? | an instance setting | `core.system_setting` |

A value that fails test 7 is a code constant.

Test 6 covers values such as the OIDC issuer, the session cookie domain, the
allowed CORS origins, the OIDC redirect URI, and the allow-list of image
registries that environment builds may pull from. Each limits what the
deployment trusts. If an administrator account could widen one through the
console, a compromised administrator account could widen it too. A value in
this class moves into settings only by an explicit decision recorded in an
RFD.

### Requirements for an instance setting

A value that reaches test 7 is an instance setting only if all of the
following hold:

- **It has a constant default in code.** An instance with an empty settings
  table works.
- **It takes effect when read.** The consumer reads the value at the point
  of use and accepts that it may be a few seconds old, and that two
  processes may briefly see different values.
- **The core validates it alone.** The validator does not call the owning
  extension and does not depend on whether that extension is running.
- **It is safe for every process to read**, and safe to appear in a
  database dump and in the administrator's read.
- **Every value the validator accepts is safe on a live instance.**
- **It has one owner**: the core or one extension. Other processes read it
  through the typed accessor.
- **It is private unless its owner declares it public.**
- **It is not a read-time default for an activity setting.** RFD 0018
  requires an undecided activity key to resolve to a constant computed from
  the envelope alone. An instance-level default for new activities is
  applied when the activity is created, as a recorded decision.

### Settings bounded by capacity

Some values are policy that an administrator decides but that consume
infrastructure: timeouts, concurrency, resource limits, budgets. For these,
ops sets a ceiling in deployment config and the administrator chooses a value
at or below it. The core's validator reads the ceiling from the core's
config, because validation does not call the extension. The consumer also
clamps the value to its own ceiling at the point of use, because a ceiling
lowered after a value was stored would otherwise leave the stored value above
it. A capacity value with no sensible ceiling stays deployment config.

### Examples

| Value | Home | Test |
|---|---|---|
| Database DSN, object storage keys, Temporal address | deployment config | 1, 2 |
| OIDC issuer, session cookie domain and prefix | deployment config | 2, 6 |
| Allowed base image registries | deployment config | 6 |
| Telemetry sampling rate, linter image | deployment config | 2 |
| Model provider API key, Entra directory connection | secret connection | 1 |
| Default model, per-run token budget for agentic grading | instance setting, under a ceiling | 7 |
| Sandbox session timeout | instance setting, under a ceiling | 7 |
| Product name, institution name, support link, login notice | instance setting, public | 7 |
| An uploaded logo or favicon file | domain data, an asset | 4 |
| Which asset is the logo or the favicon | instance setting, public | 7 |
| Default score visibility for new activities | instance setting, applied at creation | 7 |
| Proctoring policy of an activity | activity settings | 3 |
| Academic terms | domain data | 4 |

## The settings table

```sql
CREATE TABLE core.system_setting (
    owner      TEXT NOT NULL,
    name       TEXT NOT NULL,
    value      JSONB NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_by INTEGER REFERENCES core."user"(id) ON DELETE SET NULL,
    PRIMARY KEY (owner, name)
);
```

- **`owner`** is `core` or an extension name as registered in the
  application's extension map. **`name`** is unique within the owner.
- **An absent row means undecided.** The setting resolves to its constant
  default. Deleting a row resets the setting.
- **A present row is a recorded decision.** Its value is the JSON encoding of
  the setting's Go type.
- **`updated_at`** is the revision token for `If-Match`, following the
  optimistic concurrency convention RFD 0018 introduced. A trigger sets it
  on every update to the later of the clock time and one microsecond after
  the previous value. The token therefore changes on every write, including
  two writes within one microsecond. **`updated_by`** records the
  administrator. A history table is not part of this RFD.
- The table follows the core schema's boundary rule: every process may read
  it, and the core alone writes it.

### Scalar and object settings

A setting is a scalar unless its fields form one decision. Use an object
when the fields are validated together, as with a minimum and a maximum, or
when one field selects a mode and the others are that mode's parameters.

An object is stored whole. Once an administrator sets it, every field holds
the stored value, and a later change to a field's code default does not
reach that instance. That is correct for one decision and wrong for a group
of independent switches, which stay separate settings.

## The typed contract

`internal/systemsettings` is a leaf package that imports nothing from the
extensions or the rest of the core, so every process can import it. Each
owner's settings live in one file, written by that owner's maintainers and
registered into a static table at package init, as in
`internal/activitysettings`. Each setting declares:

- its owner and name;
- its Go type, which defines the JSON shape;
- its constant default;
- its validator, which may read a ceiling from the core's config;
- whether it is public.

The package provides typed accessors. A consumer asks for a setting by its
declared handle and receives a value of the declared type with the default
applied; no code outside the package decodes the stored JSON. The core and
the extensions are one binary at one version, so every process holds the
same table.

## Writes

The core serves the write routes:

- `PUT /v1/settings/{owner}/{name}` stores a decision.
- `DELETE /v1/settings/{owner}/{name}` removes it, which restores the
  default.

Both require `system#admin`. Both accept an optional `If-Match` carrying the
row's `updated_at`; a stale token returns 412 with the fault
`request.precondition_failed`. The core rejects an unknown owner or name, a
value that does not decode into the declared type, and a value the validator
rejects, each with a 400.

The token uses the HTTP entity-tag form. For an undecided setting it is
`""`, so a write that sets a setting only while it is still undecided sends
`If-Match: ""`. An `If-Match` header with an empty value counts as the same
token, so that form cannot skip the check. A write's body is the setting's
JSON value. Resetting goes through `DELETE`, so a null value is rejected.
Deleting an undecided setting returns 204 and publishes no notification.

A write changes one setting. Fields that must change together form one
object setting, so the API has no batch write.

After a write commits, the core publishes a notification on core NATS on the
subject `logistics.events.system_settings.changed.<owner>.<name>`, carrying
the owner, the name, and `updated_at`. Delivery is at most once. The
notification lets a process drop its cache early; the cache expiry is what
guarantees that every process converges.

## Reads

- **Inside the core and the extensions**, a loader reads the whole table,
  resolves defaults, and caches the result for a short expiry (proposed: ten
  seconds). A process that receives a change notification drops its cache.
  The table holds tens of rows, so a full read is cheaper than tracking
  individual rows.
- **Administrators** read `GET /v1/settings` and `GET /v1/settings/{owner}`.
  Each entry carries the resolved value, the default, whether a decision is
  recorded, the ceiling where one applies, `updated_at`, and `updated_by`
  with the administrator's name. It also carries the setting's type
  (`string` or `asset`), whether it is public, and its constraints, so the
  console can choose an editor and check a value before sending it. A read
  for an undeclared owner returns 404.
- **Everyone, including unauthenticated clients,** reads the public subset
  through `GET /v1/instance`, which describes the instance to a client
  before login. It carries only settings their owner declared public,
  grouped by owner as `{"core": {...}}`, so that settings of different
  owners cannot collide by name. The response carries
  `Cache-Control: public, max-age=10`, the expiry of the core's cache.
- **A client server that renders pages** may cache `GET /v1/instance` for
  the same expiry as the core's loader, so that rendering a page does not
  add a request to the core.

## Assets

An **asset** is a file that an administrator uploads for a setting to refer
to, such as a logo. Under the Where a value belongs section, an asset is
domain data: there can be many, each with an identity and a lifecycle, so it
has its own table. Which asset the instance uses as its logo is a decision,
so it is a setting whose value refers to the asset. The bytes live in object
storage, and the settings table holds the reference alone.

### The asset table

```sql
CREATE TABLE core.instance_asset (
    sha256      TEXT PRIMARY KEY,
    media_type  TEXT NOT NULL,
    width       INTEGER NOT NULL,
    height      INTEGER NOT NULL,
    size        INTEGER NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by  INTEGER REFERENCES core."user"(id) ON DELETE SET NULL
);
```

- **`sha256`** is the hash of the stored bytes and the asset's identity. The
  object key is derived from it as `<sha256>.png`, so the row does not store
  the key. Uploading identical bytes twice yields one row.
- The objects live in a bucket of their own, named in the core's config with
  the default `instance-assets`, as recordings and submission exports have
  theirs.

### Upload

`POST /v1/settings/assets` takes the file as the request body and requires
`system#admin`. The core processes it in these steps:

1. It reads at most 1 MiB and rejects a larger body.
2. It decodes the body as PNG, JPEG, or WebP. The format is judged from the
   bytes, and the `Content-Type` header is ignored.
3. It rejects an image wider or taller than 2048 pixels.
4. It converts the decoded image to 8-bit RGBA and re-encodes it as PNG.
   The stored bytes are then a plain raster with no metadata, and the hash
   describes the bytes that are served.
5. It rejects an encoded image larger than 4 MiB. Re-encoding can make a
   compressed upload many times larger, and the stored image is served on
   every page.
6. It writes the object, then the row, and returns the asset descriptor:
   `sha256`, `media_type`, `width`, `height`, and `size`. The status is 201
   when the upload creates the asset and 200 when identical bytes are
   already stored.

The upload and the deletion of one asset both lock its row. A deletion
therefore cannot remove the object of an asset that an upload is storing
again.

The upload writes no setting. Vector formats are rejected because an SVG can
carry script, and the first settings do not need one.

### Asset references in settings

A setting that holds an asset declares the asset reference type in
`internal/systemsettings`. Its JSON value is `{"asset": "<sha256>"}`, and its
constant default is no asset. Each such setting declares constraints on the
asset, such as a square image of at least 180 pixels for a favicon.

An administrator assigns an asset through the ordinary settings write,
`PUT /v1/settings/{owner}/{name}`. `If-Match`, `updated_by`, the change
notification, and `DELETE` to restore the default apply unchanged. The
validator checks that the asset row exists and meets the setting's
constraints. It reads a core table and calls no extension, so the core still
validates the setting alone.

In the transaction that writes the settings row, the core takes a share lock
on the assigned asset row. A deletion of that asset then waits for the write
to commit and sees the reference.

### Asset reads

- **`GET /v1/instance`** carries, for each public asset setting, the asset
  descriptor and the path of the read route, or null when no asset is
  assigned. For null, a client shows its own built-in artwork. That artwork
  belongs to the client, and the core has no copy of it.
- **`GET /v1/instance/assets/{sha256}`** serves an asset whose row exists
  to anyone, including unauthenticated clients. The core streams the bytes
  from object storage, as it does for pipeline result artifacts and
  proctoring recordings. The URL names the content, so the response carries
  `Cache-Control: public, max-age=31536000, immutable`. It also carries
  `X-Content-Type-Options: nosniff` and the core's policy for untrusted
  content, `Content-Security-Policy: sandbox; default-src 'none'`, so that
  opening the URL directly runs nothing. If the row exists and the object
  does not, the core logs the inconsistency and returns 404, and the client
  shows its built-in artwork.

Every asset is public. A file for a private setting needs an access check on
the read route and is outside this RFD.

### Listing and deleting assets

An asset is kept until an administrator deletes it. Branding assets are few
and small, and they change rarely, so the storage they take stays small.

- **`GET /v1/settings/assets`** lists every asset with its descriptor, the
  path of the read route, `created_at`, `created_by` with the
  administrator's name, and the settings that refer to it. It requires
  `system#admin`. The settings page offers earlier uploads from this list,
  so an administrator can return to a previous logo without uploading it
  again.
- **`DELETE /v1/settings/assets/{sha256}`** deletes an asset that no setting
  refers to, and returns 409 for an asset that a setting refers to. It
  requires `system#admin`. The core deletes the row in a transaction that
  locks it and checks for references, then removes the object. If removing
  the object fails, the object remains with no row, and the read route does
  not serve it.

A page rendered before a change keeps working, because the asset it refers
to stays until an administrator deletes it.

## Branding settings

The first settings are the instance's branding. Their owner is `core`, and
all of them are public. The console and the exam client send a visitor
without a session directly to the identity provider, so neither renders a
sign-in page. Three console pages do render without a session: the
signed-out page, the sign-in error page, and the device code entry page. The
favicon loads on every page. Both cases need the settings before login.

| Name | Type | Default | Constraints |
|---|---|---|---|
| `product_name` | string | `CoBE` | 1 to 24 characters |
| `institution_name` | string | the empty string | at most 60 characters |
| `logo` | asset reference | no asset, shown as the CoBE mark | height at least 64 pixels, width at most four times the height |
| `logo_dark` | asset reference | no asset, shown as `logo` | the constraints of `logo` |
| `favicon` | asset reference | no asset, shown as the CoBE icon | square, at least 180 pixels |
| `support_url` | string | the empty string | the empty string, or an absolute `https:` or `mailto:` URL |

### Where a client shows them

- **The product name** is the wordmark in the top bar, the suffix of page
  titles, and the label of the About button.
- **The institution name** appears beside the product name in the top bar,
  in a smaller secondary style, and on the pages that render without a
  session. An instance that keeps the product name shows both names.
- **The logo** is the top-bar mark. It appears before the product name in
  the top bar, in the About dialog, and on the pages that render without a
  session. While the dark theme is active, the client shows `logo_dark`, or
  `logo` if no dark asset is assigned. The exam client's top bar shows the
  examination and carries no branding.
- **The favicon** is the browser tab icon and the touch icon on every page
  of both clients.
- **The support link** is the help link in the About dialog, and the clients
  offer it where their text tells a user to contact support. If it is the
  empty string, the clients show no help link.

Each image setting is named by where the image appears, and each has its own
default. Any combination is valid: an instance may keep the CoBE mark in the
top bar and show the institution's crest as the favicon, or replace both.
Setting one image never changes another, because a logo may be up to four
times as wide as it is high and a favicon is square. One asset may fill
several settings, so a crest uploaded once can serve as the favicon and
later as the logo. The institution is named in text.

The default artwork is the CoBE mark and icon, and each client bundles its
own copy. For an undecided `logo` or `favicon`, `GET /v1/instance` returns
null and the client shows its bundled artwork. A page then renders the
default with no request to the core, and the default still shows while the
core is unreachable. The console's settings page previews the default from
the same bundled copy.

### The product name

An institution may replace the product name. An institution that replaces
it usually replaces the logo too; otherwise the CoBE mark stands beside a
different name. The settings page previews the top bar, so an administrator
sees the combination before saving it. Two rules keep the interface
consistent across instances and keep a renamed instance traceable in
support:

- **Interface copy does not name a feature after the product.** Text such
  as "Answer in CoBE" names the answering mode after the product, and on a
  renamed instance it would show a name that appears nowhere else. Such text
  names the mode itself, and the product name appears only in the places
  listed above.
- **The About dialog names the software.** For any product name, it states
  that the instance runs CoBE and gives the version, the commit, and a link
  to the ZINC project, so a screenshot in a bug report identifies the build.
  This text is not a setting.

The desktop application's name is fixed when its installer is built. Each
desktop build profile gains an optional product name, so an institution that
renames the product builds an application with the same name. The profile
changes the name the application displays. The application identity that
updates rely on stays per release channel. The profile is part of the desktop
build and is not an instance setting.

Branding makes a small first set. It has no ceiling, every value the
validators accept is safe on a live instance, and no extension reads it. It
still uses the table, the typed contract, the public read, the asset routes,
and the console page, so each part of this RFD has a consumer.

## Secrets

A secret is a credential an administrator enters for the platform to use,
such as an API key for a model provider. Secrets follow these rules:

1. **A secret is a connection.** It lives in a table in the owning
   service's schema, one row per connection, with the sealed credential and
   the connection's state: who set it and when, a version, the last
   verification time, and the last error. `core.directory_connection` is the
   precedent.
2. **The owner writes and reads it.** The owning service serves the routes,
   and no other process queries the table. For a secret owned by an
   extension, `boundary-lint` rejects reads from sibling schemas. For a
   secret owned by the core, whose schema every process may read, the rule
   rests on review.
3. **The API is write-only.** A write stores or replaces the credential. A
   read returns the metadata and a short hint, such as the last four
   characters, and never the credential. A verify action makes a cheap call
   to the provider and records the result. Writes require `system#admin` and
   accept `If-Match` on the version.
4. **The credential is sealed at rest** with AES-256-GCM. The additional
   authenticated data binds the ciphertext to its connection name and
   version, so a ciphertext copied to another row does not open. The
   ciphertext starts with a key identifier so that the sealing key can
   change without a format change.
5. **The plaintext is held in a type that formats as redacted** in logs,
   JSON, and traces, as the Entra token set does.
6. **A secret is decrypted only in the owning service's process.** It is
   never passed to a grading container, a sandbox, or any process that runs
   or reads untrusted content. Such a process receives a short-lived
   credential scoped to one run, or none. For agentic grading, either the
   agent loop runs in the pipeline worker and the container only executes
   tools, or an agent in the container calls a gateway run by the pipeline
   extension with a per-run token, and the gateway adds the provider key.
7. **Non-secret parameters are instance settings.** The default model, the
   allowed models, and budgets are settings rows owned by the same owner.
8. **Replacing a secret needs no restart.** The owner caches the decrypted
   credential for a short expiry and drops it on its own change
   notification.

### The sealing key

Two keys are available:

- **A key derived from the authority key** with a label specific to the
  table. This is what the Entra connection uses, and it needs no extra
  deployment secret. It protects the credential against a leaked database
  dump or backup. It does not protect it against another process, because
  every process holds the authority key. If the authority key rotates, the
  stored credentials no longer open, and an administrator enters them again.
- **A key held only by the owning extension**, delivered in its local config
  file and kept off the config bus, like the proctoring key-encryption keys.
  It isolates the credential from the other processes and supports versioned
  rotation, at the cost of one more deployment secret.

The proposal is the authority-derived key. All processes share one database
login and one binary, so isolation between processes is not a boundary the
platform enforces today. The key identifier in the ciphertext allows a later
move to an owner-held key.

## Open questions

1. Test 6: do trust-boundary values stay in deployment config by default,
   with any exception decided in an RFD?
2. The ceiling rule: are capacity-related settings bounded by a ceiling in
   deployment config, or kept out of settings entirely?
3. The sealing key: authority-derived, or held by the owning extension?
4. Secret scope: one connection per provider for the instance, or also
   connections owned by a school or a course?
5. Agentic grading: does the agent loop run in the pipeline worker or in the
   grading container? This decides whether the pipeline extension runs a
   gateway, and belongs to the agentic grading design; rule 6 of the secrets
   section constrains both answers.

## Decision status

| Decision | Status |
|---|---|
| One row per setting, JSONB value, scalar or object | decided |
| Separate `owner` and `name` columns | decided |
| Placement rules and setting requirements | proposed |
| Postgres as the store, read directly, cached, notified over core NATS | proposed |
| Typed leaf package with constant defaults | proposed |
| Write routes, `system#admin`, `If-Match` | proposed |
| Secrets as owner-held connections with a write-only API | proposed |
| Secret never reaches a process that handles untrusted content | proposed |
| Authority-derived sealing key | proposed |
| Uploaded files as assets in `core.instance_asset`, referred to by content hash | proposed |
| Upload separate from assignment, assignment through the settings write | proposed |
| Public, immutable asset read route that the core streams | proposed |
| Raster formats, re-encoded as PNG | decided |
| Assets kept until an administrator deletes one, with no automatic sweep | decided |
| Branding (product name, institution name, logo, dark logo, favicon, support link) as the first settings | proposed |
| Institution shown beside the product name, which an institution may replace | decided |
| The About dialog names the software for any product name | proposed |
| Interface copy names features without the product name | decided |
| Image settings named by placement, each defaulting to the CoBE artwork that each client bundles | decided |
| No institution logo setting in the first set; the favicon may carry the institution's mark | decided |
| An empty support link hides the help link | decided |
| Desktop build profiles gain an optional product name for the displayed application name | decided |

## Implementation

1. A follow-up migration creates `core.system_setting` and
   `core.instance_asset`, with its down migration.
2. `internal/systemsettings` holds the static table, the typed accessors, the
   loader with its cache, and the notification subject and payload shared by
   publisher and subscribers.
3. The core's logistics module serves the write routes and the administrator
   reads, and publishes the notification. `GET /v1/instance` serves the
   public subset.
4. The core serves the asset upload, list, delete, and read routes, and
   declares the asset reference type with its share lock on assignment.
5. Each process registers the loader and subscribes to the notification at
   startup.
6. The SDK and `zincli` gain the settings and asset routes; the console gains
   an instance settings page per owner, starting with branding. The console
   and the exam client read the branding settings in their root loaders.
   Before the product name becomes configurable, the clients' copy that
   names a feature after the product is reworded; that change does not
   depend on the settings and can come first.
7. The desktop build profiles gain an optional product name, used for the
   window title, the notices, and the installer's displayed name.
8. The sealing code in `internal/entra` moves to a shared helper in
   `internal/crypto`, and the Entra connection uses it. The first secret
   connection beyond Entra is designed with agentic grading.

No setting moves out of deployment config as part of this RFD. Each move is
a separate change that applies the placement rules. The branding settings
have no counterpart in deployment config.

## UX

- **Administrators** get an instance settings page in the console, grouped
  by owner. Each entry shows the value, the default, whether it was changed,
  who changed it and when, and the ceiling where one applies. A reset action
  deletes the row.
- **Branding** is grouped by where each setting appears: names (product
  name and institution name), the top bar (logo and dark logo), the browser
  tab (favicon), and help (support link). Each group shows a preview of the
  place it changes.
- **Asset settings** show the assigned asset beside the default artwork, an
  upload control, the earlier uploads to choose from, and the reset action.
  An earlier upload that no setting uses has a delete action. The console previews a chosen file
  before it uploads it, and assigns the asset once the upload returns.
- **Clients before login** read `GET /v1/instance` for the public subset,
  including the branding settings, and load assets from the asset read
  route.
- **Secret connections** show their state, the hint, and the last
  verification, with replace, verify, and remove actions. The credential is
  never shown after entry.
- **Ops** keeps deployment config for every value the placement rules assign
  to it, and sets the ceilings for capacity settings.
