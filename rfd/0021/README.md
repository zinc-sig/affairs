---
authors: hoifanrd
state: discussion
discussion: "[#31](https://github.com/zinc-sig/affairs/pull/31)"
labels: direction, infrastructure, ux
---

# [RFD] Database Questions

An instructor cannot set a question whose answer is SQL. This RFD adds
**database questions** for MySQL, PostgreSQL, SQL Server and Oracle: the
student writes SQL, presses Run during the exam, and is graded on hidden
data. A database server runs in a container of its own beside the container
that runs the question's commands. The pipeline declares that it uses the
database with a `service "database"` block, the question's language names
the engine, the activity's image runs it, and every test case gets its own
empty database.

The RFD's main content is the contract between a pipeline and the platform:
what the pipeline declares, how a command learns where its test case's
database is, where data comes from, and what the platform guarantees between
runs. It amends [RFD 0013](../0013/README.md) (the pipeline language),
[RFD 0014](../0014/README.md) (the generator) and
[RFD 0015](../0015/README.md) (the runtime).

## Background

A coding question is set up in four places, and each owns one thing. The
course's image library holds the container images that code runs in; an
instructor builds one from a base image in a terminal. An activity selects
one image under Runtime. A question selects a language under Answer Options,
from a registry the core owns. The question's grading is a pipeline
([RFD 0013](../0013/README.md)), which the console's grading builder writes
from a form and which the generator assembles into the activity's config
([RFD 0014](../0014/README.md)). A student's Run executes that pipeline on a
warm executor of the sandbox pool, with no network. A graded run executes the
same pipeline in one container made for the run
([RFD 0015](../0015/README.md)).

A pipeline is plain commands over files. There is no shell: `command` is a
program in the image and `args` are passed as written. A test is another
command, usually `diff`, that compares a produced file with an expected one.
The language has no subject matter of its own. Its one subject, the exam,
enters as a label on a general block, `document "examination"`, and every
name an expression reads is declared by a block in the same file: `locals`,
`document`, and in the formula the empty `pipeline "<name>" {}` declaration.
When the platform owns a value that a pipeline needs, the pipeline reads it
by reference, as it reads `marking.expected_file`. Only `args` is an
expression today; `env` is a literal map.

Database questions were first asked for MySQL and PostgreSQL, and then for
SQL Server and Oracle as well
([zinc-sig/ui#803](https://github.com/zinc-sig/ui/pull/803)). Three drafts
cover the parts that do not depend on a server: the two language entries
([zinc-sig/core#822](https://github.com/zinc-sig/core/pull/822)), a syntax
check for both dialects in the linter
([zinc-sig/linter#7](https://github.com/zinc-sig/linter/pull/7)), and editor
highlighting ([zinc-sig/ui#803](https://github.com/zinc-sig/ui/pull/803)).
None of them can run an answer.

The first approach started the server inside the run container with a
script ([zinc-sig/core#808](https://github.com/zinc-sig/core/issues/808)).
It works for MySQL and PostgreSQL with changes to the sandbox. It cannot
work for the other two engines. Measured with the stock images, one CPU:

| | SQL Server 2022 | Oracle Free |
|---|---|---|
| Ready after start | 12 to 15 s | 25 to 33 s |
| Tasks | about 175 threads | 52 to 58 processes |
| Lowest memory limit that started | 1000 MiB | 2000 MiB |
| Fails inside the sandbox container on | the task cap, dropped capabilities, the anonymous user | the task cap, the read-only root, the anonymous user, the seccomp profile |

A server that takes 12 to 33 seconds to start cannot be started for each
command, and neither engine survives the sandbox's restrictions. So the
server has to live outside the run container and outlive a command.

The platform already runs one service on a question's behalf, the linter. It
is the platform's to run: the image comes from deployment config, the core
supervises it, and the question holds a switch and its language. A database
differs in two ways. The student's code talks to it directly, and an
instructor may need to change the server itself, for example to add an
extension.

One measured fact constrains how a command can be told where its database
is. All four stock clients accept the connection in their arguments. `psql`
and `sqlcmd` also accept it from environment variables. `mysql` cannot
select a database without an argument or an option file, and `sqlplus`
cannot take credentials without an argument or a login file.

## Proposal

1. **A database container beside each run, for every engine.** It shares
   the run container's network namespace and nothing else, so the server is
   at `127.0.0.1` and a Run still has no route anywhere.
2. **The pipeline declares the service.** A `service "database" {}` block
   under `pipeline` says that the pipeline's commands use a database. A run
   starts a database container only for a pipeline that declares it.
3. **The question's language names the engine.** The generator adds
   `engine` to the block from the language. Nothing is inferred from an
   image and nothing is chosen in a dialog.
4. **The activity's image runs the engine.** It is the image the instructor
   built, started through its own entrypoint, so its settings, its installed
   packages and its first-start scripts apply.
5. **Every scenario has its own database.** It is empty when the scenario's
   first stage starts and is the same database in the scenario's later
   stages.
6. **A command reads the connection by reference.** `service.database`
   has five values, `host`, `port`, `user`, `password` and `name`, readable
   in `args` and `env` and resolved per scenario at the parse phase. Nothing
   else is set for the command.
7. **Data is a test case's input.** A stage loads it through the client.
   What every test case shares may be prepared in the image by a first-start
   script.
8. **A warm server is reused only after a verified reset.** This applies to
   Runs on the sandbox pool. A graded run makes its database container for
   the job and removes it with the job.
9. **The console writes all of it.** A language preset writes the block,
   the commands and the connection; the instructor fills in test cases.

### Abandoned ideas

- **The server inside the run container for MySQL and PostgreSQL, and a
  service for the other two.** Two models for one feature. The in-container
  model also needs shared-memory system calls in the sandbox's seccomp
  profile and a non-root user in graded runs.
- **An engine image fixed in deployment config, as the linter's is.** The
  instructor could then change nothing in the server: no extension, no
  setting, no chosen version. The client in the instructor's image and the
  server could also differ in version.
- **Recognising the engine from the image's name.** It is a guess made from
  another layer, and no list of image names is complete.
- **Choosing the engine in the New image dialog.** The language already
  says which engine, so the choice would have two homes, and the image is
  made before any question exists.
- **The block written by the generator alone.** The need follows the
  language on every save, but the name that commands read is then declared
  nowhere in the text the author edits. RFD 0013 rejected an undeclared
  `pipeline.<name>` for that reason.
- **A `database` block and root name.** It puts a subject word into the
  grammar. `document "examination"` shows the alternative: a general block
  whose label is the kind.
- **The platform setting each client's own settings**, such as `PGHOST` or
  a MySQL option file, so that a bare `psql` works. The pipeline does not
  show it, it serves known clients only, and `mysql` and `sqlplus` need
  files written into the run container. The stored result would no longer
  show what ran.
- **Fixed, documented coordinates typed as literals.** The values are the
  platform's, and stored text would pin where the database is.
- **Environment variables with a `$(NAME)` rule in `args`, or the same
  variables through `sh -c`.** The first is a second substitution syntax
  beside HCL's, unchecked at save, and contradicts "each argument is passed
  as written". The second brings back the shell.
- **An instructor's script with two accounts and a reference query run at
  grading time.** The instructor writes the plumbing, and a second account
  that the student must not reach exists for no other question type. An
  expected output compared with `diff` is how every coding question is
  checked.
- **A `query` step that the core executes.** It is a new step kind in both
  runtimes and a second way to run something, where `exec` is the canonical
  primitive.
- **A new server for every job.** It costs 4 to 33 seconds per Run, and an
  Oracle start writes about 3 GB.

## What an instructor does

| Step | A Python question today | A database question |
|---|---|---|
| Course, Images, New image | Base image `python:3.13`; install packages in the terminal | Base image `postgres:17`; install what the course needs in the terminal |
| Activity, Runtime | Select the image | Select the image |
| Question, Answer Options | Language Python | Language PostgreSQL |
| Question, Grading, "Compare output" | The preset writes the pipeline | The preset writes the pipeline |
| Pipeline settings | Answer file | Answer file, and a Database line |
| A test case | Input and expected output | Data, an SQL script loaded before the answer runs, and expected output |
| The Run step | `python3 main.py` | `psql`, and a connection made of five inserted names |
| The student presses Run | Sees the program's output | Sees the query's result |

The five names come from an Insert menu, the way a score inserts a test
result today. Another program, such as `python3 check.py`, gets the same
five values wherever it wants them.

What "Show its HCL" shows:

```hcl
pipeline "q5" {
  submission { file = "answer.sql" }
  service "database" {}

  # Each test case loads its data, runs the answer, and compares the output.
  stage "load" {
    exec {
      command = "psql"
      args    = ["postgresql://${service.database.user}:${service.database.password}@${service.database.host}:${service.database.port}/${service.database.name}"]
      scenario "sample" { stdin_path = "shop.sql" }
      scenario "case-2" { stdin_path = "examination-assets/q5/shop-2.sql" }
    }
  }

  stage "run" {
    exec {
      command    = "psql"
      args       = ["postgresql://${service.database.user}:${service.database.password}@${service.database.host}:${service.database.port}/${service.database.name}"]
      stdin_path = "answer.sql"
      scenario "sample" { stdout_path = "sample.out" }
      scenario "case-2" { stdout_path = "case-2.out" }
    }
  }

  stage "check" {
    exec {
      command = "diff"
      scenario "sample" { args = ["sample.out", "sample.expected"] }
      scenario "case-2" { args = ["case-2.out", "examination-assets/q5/case-2.expected"] }
    }
  }
}
```

`shop-2.sql` and `case-2.expected` are assets hidden from Run. In a
student's Run the `load` of `case-2` fails for want of its file, and the
existing dependency gating skips that case's `run` and `check`, as it does
for any hidden test.

The presets for the four clients, where `d` stands for `service.database`:

| Engine | Command | Arguments |
|---|---|---|
| PostgreSQL | `psql` | `["postgresql://${d.user}:${d.password}@${d.host}:${d.port}/${d.name}"]` |
| MySQL | `mysql` | `["--host=${d.host}", "--port=${d.port}", "--user=${d.user}", d.name]`, with `env = { MYSQL_PWD = d.password }` |
| SQL Server | `sqlcmd` | `["-C", "-S", "${d.host},${d.port}", "-U", d.user, "-P", d.password, "-d", d.name]` |
| Oracle | `sqlplus` | `["-S", "-L", "${d.user}/${d.password}@//${d.host}:${d.port}/${d.name}"]` |

Each of these forms was run from a locked-down run container against a
server in a second container, with the data and then the answer on standard
input.

## The `service` block

### Syntax and placement

```hcl
pipeline "q5" {
  service "database" {}
}
```

- **Placement.** Under `pipeline`, beside `submission` and `resources`,
  which also say what the run needs from the platform. A pipeline has at
  most one.
- **Label.** The kind of service. Only `database` is defined.
- **Body.** One attribute, `engine`, the id of a database language in the
  core's registry.
- **In a coding question's fragment** the body is empty. `engine` is
  refused there with the words used for other generator-owned attributes:
  "Not allowed here; it comes from the question." The generator adds it to
  the assembled config from the question's language, as it adds `include`
  and `optional` to `submission`.
- **In a pipeline config written by hand** for an upload activity, which
  has no question and no language, the author writes `engine`.

### The reference

A pipeline that declares the block may read `service.database` in the
`args` and `env` of its `exec` and `scenario` blocks. It is an object of
five strings:

| Value | Meaning |
|---|---|
| `host` | Address of the server, from the run container |
| `port` | The engine's port |
| `user` | The scenario's account |
| `password` | The account's password |
| `name` | The scenario's database (the service name, on Oracle, where the account is the schema) |

- **The value is the current scenario's.** An `args` written on the `exec`
  resolves once for each scenario. The same scenario code has the same
  values in every stage. A stage with no scenario reads the values of
  `default`.
- **It is not readable where no scenario exists yet:** in `locals`, and in
  the `for_each` and `labels` of a `dynamic` block.
- **`env` becomes a deferred expression**, as `args` is. The merge rule
  does not change: the exec's variables first, the scenario's laid over
  them. The MySQL preset needs it, because the client prints a warning on
  every command when the password is in its arguments.
- **There is no ready-made connection string.** One string per client
  beside the five parts it is made of would be two ways to say one thing.

### Checks at save

| Diagnostic | Cause | Fix |
|---|---|---|
| `undeclared service reference` | The pipeline reads `service.database` and has no `service "database"` block | Add the block |
| Unsupported attribute | A value other than the five, such as `service.database.uri` | Use one of the five |
| `service not available for this language` | The block is on a question whose language is not a database | Remove the block, or choose a database language under Answer Options |
| `engine not allowed here` | `engine` in a question's fragment | Remove it; it comes from the question |
| `unknown engine` | A hand-written config names an engine the registry does not have | Use a registered id |
| `duplicate service block` | Two blocks of one kind | Keep one |

The check between the language and the block follows the existing one
between an answer modality and its marking scheme. Saving the grading
refuses a block the language does not support. Saving a language change
only warns, because the language and the pipeline are two writes and
refusing the first would leave no order in which to save both. Publishing a
question whose language and block disagree is refused.

### The parse phase

The parse phase takes one more input beside the document snapshot: the
engine's host and port, from the registry. It gives every scenario code an
account, and `service.database` resolves to it. The resolved pipeline
carries the engine and each scenario's account, so both runtimes create
exactly the databases that the arguments name, and neither looks anything
up again. A config that declares the block and receives no binding fails at
the parse phase, as a config that declares a document and receives no
snapshot does.

The account's user and database name are short identifiers of the
platform's own. A scenario code may hold a hyphen or start with a digit,
which no engine accepts as a plain identifier, so the code itself is not
used.

### Why a block, and why its body is empty

The block follows two existing declarations.

- **The formula's `pipeline "<name>" {}`** declares what the formula reads
  as `pipeline.<name>`. Its body is empty and the save-time check keeps it
  empty, reserved for later options. RFD 0013 records the trade: the block
  "feels like boilerplate for the simple case, but the consistency and
  validation wins outweigh the one-line cost". The same holds here: a
  misspelt or undeclared name is refused at save from the one file, and the
  body has room for a later option.
- **`document "examination"`** is a general block whose label is a type,
  with one type defined.

The block is not empty syntax in the sense RFD 0013 uses for the blocks it
dropped: it has two consumers, the runtime that starts the container and
the references that read it.

Two things are new. The block grounds a name from inside `pipeline` and not
from file scope, because a database belongs to one pipeline's run and a
question's fragment allows no other top-level block. And declaring it has an
effect, which `document` and the formula's declaration do not have: it
starts a container.

## The engine registry

A language registry entry says whether the language is a database. For a
database it also holds what the core needs to run the engine:

- **Start.** The command, the user, the settings the base image's
  entrypoint requires, and the engine's own start settings. MySQL starts
  without its binary log, because with it a test case's account may not
  create functions or triggers.
- **Limits.** Port, memory, task limit, data volume size.
- **Operations.** The statements for ready, make a scenario's account and
  database, reset, and fingerprint.
- **Replacement policy.** How many jobs and what age a warm server may
  reach.

The entries are built in and a deployment may override them in config, as
it may override the language list today. The engine ids need a home that the
pipeline package can read without importing the examination domain.

## The database container

The database container is the activity's image, started through the image's
entrypoint with the registry's start entry. A saved image's own command is
`/bin/bash` and its user is root, so neither is relied on. The image
therefore has to be built on the engine's own image. When it is not, the
container fails at start, for example with `exec: "postgres": executable
file not found in $PATH`, and the run ends with a message that points to
Runtime. Save cannot check what an image holds, so this is found at the
first run, as a missing interpreter is today.

- **Network.** The container joins the run container's network namespace.
  On Docker it is created in that namespace. On Kubernetes it is a second
  container of the pod; in a grading Job it is a native sidecar, which does
  not hold the Job open.
- **Posture.** Its own, not the sandbox's: the engine's user, a
  size-limited data volume, memory and task limits from the registry,
  no-new-privileges, capabilities dropped as far as the engine allows, and
  the runtime's default seccomp profile. Both heavy engines fail under the
  sandbox's posture.
- **Administrator access** stays in the database container. The core uses
  it through exec. No command of a pipeline holds it.
- **In a graded run** the run container has a network, because the agent
  needs Temporal and object storage, and the database container shares it.
  The server listens on loopback only, so another run's container cannot
  reach it. The database container receives none of the agent's
  credentials.
- **Oracle** refuses to start in a namespace whose only interface is
  loopback, which is what a Docker sandbox executor has. It started once a
  dummy interface with a private address and no route was added, which
  takes `NET_ADMIN` in a one-shot helper.
- **SQL Server** ships for amd64 only, and its binary carries file
  capabilities, so it fails to exec with every capability dropped.

## Scenario databases

Before a scenario's first stage, the core makes the account that the
resolved pipeline names and one empty database owned by it, on Oracle a
schema. In a Run the scenarios run one at a time and each is made just
before its first stage. In a graded run every scenario's database is made
once the server is ready, since the scenarios of a stage run at the same
time.

- **The database is made from the engine's default template**, so what the
  image prepared for every database is in it.
- **The account can do anything inside its database** (tables, views,
  procedures, functions, triggers, sequences) and nothing outside it. It
  cannot create a database, read another scenario's, or ask the server to
  read a file. On PostgreSQL this needs `CONNECT` revoked from `PUBLIC` on
  each database.
- **The unit is the scenario** because a `load` stage has to feed `run`,
  which rules out a database per command, and because two test cases load
  different rows into the same table names, which rules out one per run.

## Seeding

### Per test case

This is the way the platform promises, and it is the same on any image. The
`load` stage runs the client in the run container as the scenario's account,
with the test case's data on standard input: an SQL script, typed in the
test case or uploaded as an asset, visible or hidden. Every run seeds again.

The client can bulk-load a file from the workspace (`\copy` in `psql`,
`LOAD DATA LOCAL` in `mysql`). The server cannot read workspace files: it is
another container, and the account has no such right. A 100,000-row script
loaded in 1.1 s on PostgreSQL and 1.4 s on MySQL.

### Per image

The base images run the scripts in their first-start folder when a server
starts on an empty data directory, and the image builder keeps files put
there. A first-start script runs as the engine's administrator before any
test case exists.

| Prepared in the builder | Reaches a test case |
|---|---|
| An extension package alone (`pgvector`) | No. The scenario's account may not create it: "Must be superuser to create this extension" |
| The package, and a first-start script that creates it in `template1` | Yes, in every database made from the default template |
| A reference table made in `template1` by a first-start script | Yes, copied into every scenario's database |
| A server setting (PostgreSQL time zone, MySQL `local_infile`) | Yes |
| MySQL: a reference database made by a first-start script | No. The scenario's account may not read it |

This was tested on PostgreSQL and MySQL. Data loaded by hand in a builder
session is not kept, because those two base images keep their data
directory in a volume, which a save does not capture. A script or a setting
that breaks the server shows at the first run.

## Reuse between Runs

A sandbox executor is warm and is reused. Its database container is reused
with it, under one rule: only after a verified reset.

1. **Reset.** End the job's sessions, then drop its accounts and databases.
2. **Verify.** Compare a fingerprint of server-wide state with the one
   taken when the server was clean. It covers accounts, databases, server
   settings, open sessions, objects in shared system areas and jobs. The
   clean fingerprint includes whatever the image's first-start scripts made.
3. **Replace on any difference.** The executor and its database container
   are destroyed and the pool makes a new pair. This uses the existing
   dirty-executor path.
4. **Replace on schedule.** A pair is also replaced after a configured
   number of jobs or age. The fingerprint cannot see an engine bug that lets
   an account leave its database; the schedule is the answer to that.

| | PostgreSQL 17 | MySQL 8.4 | SQL Server 2022 | Oracle Free |
|---|---|---|---|---|
| Reset plus fingerprint check | 0.5 s | 0.6 s | 0.4 s (3.4 s once in five) | 0.8 s |
| Clean again after reset | 5 of 5 | 5 of 5 | 5 of 5 | 6 of 6 |
| An account planted behind the reset's back | noticed | noticed | noticed | noticed |
| Replacement (a fresh server ready) | 4 s | 15 s stock, about 1 s on a prepared data directory | 11 to 16 s | 25 to 33 s |

Each test run left behind an open session in the middle of a statement,
extra tables, temporary tables, and what the account may change about
itself. Two lessons for the fingerprint: take the clean reading once two
readings agree, because SQL Server answers queries before its start-up work
ends, and leave out the server's own cached temporary objects. Oracle's 0.8
s needs `ALTER SYSTEM DISCONNECT SESSION ... IMMEDIATE` and one connection
for the whole reset; its first reset after a start took 4.8 s, so one
warm-up cycle runs before the first job.

The engine is part of an executor's shape, so a pipeline that declares the
service gets a pair and any other pipeline of the paper a plain executor.
The pool prewarms one executor per shape the paper's pipelines declare.

A graded run does not reuse anything. RFD 0015 rejected shared containers
for grading, and that holds: the database container is made for the job and
removed with it.

## Security

- **What protects the database is reachability.** Only the run's own
  containers share the namespace. A Run has no route anywhere, and in a
  graded run the server listens on loopback only.
- **A scenario's password is not a secret.** The student's own SQL runs as
  that account. Resolved arguments are stored with every result, and a
  result shows students its command and arguments unless the stage hides its
  metadata, so the password is on screen.
- **The administrator never enters the run container.** Its credentials are
  made per server start and used by the core through exec.
- **A client's own commands** (`\!` in `psql`, `system` in `mysql`) run in
  the run container as the sandbox's user. A Python answer can already do
  the same there.
- **Between runs** the verified reset and the scheduled replacement stand
  between one student's Run and the next. A graded run shares nothing.
- **Hidden data in a graded run** is as exposed to the student's SQL as
  hidden files are to a student's program today. A restricted identity per
  command would change that for every question type and is not part of this
  RFD.

## Languages and linting

- **Language ids.** `mysql` and `postgresql` (zinc-sig/core#822), then
  `mssql` and `oracle`. There is no generic `sql` id, since it would give
  the editor and the linter no dialect.
- **Editor.** The SQL mode with the MySQL and PostgreSQL dialects
  (zinc-sig/ui#803), then MSSQL and PL/SQL.
- **Linter.** A syntax check for MySQL and PostgreSQL (zinc-sig/linter#7).
  PostgreSQL uses the server's own parser. MySQL uses a parser with guard
  rules, and reported no false error on 161 statements the real server
  accepts. A check for T-SQL or PL/SQL is added only when a parser meets the
  same bar.

## Measurements

Stock images on local Docker with one CPU: `postgres:17`, `mysql:8.4`,
`mcr.microsoft.com/mssql/server:2022-latest` and
`gvenzl/oracle-free:23-slim-faststart`. The run container side had no
network, a read-only root, an anonymous user and no capabilities.

Measured:

- Start time, tasks and memory of each engine, and each sandbox restriction
  applied alone to SQL Server and Oracle.
- All four engines as a second container in the run container's namespace.
- The four stock clients with the connection in their arguments.
- Reset, fingerprint and replacement on all four.
- One account refused another's data on all four.
- A builder-saved image in both roles, first-start scripts, the template,
  settings and bulk loading, on PostgreSQL and MySQL.
- A reference resolved per scenario in the parser: a prototype bound a
  namespace in the per-scenario context of `args` in 31 changed lines, with
  the package's tests passing and a misspelt attribute refused at save.

Not measured:

- The declared block, the `service` root and `env` as an expression.
- Kubernetes: the second container of an executor pod, and a native sidecar
  in a grading Job.
- Each engine listening on loopback only.
- SQL Server Express, Oracle's own image, a smaller Oracle SGA.
- The smallest capability set for SQL Server.
- What an instructor's image can prepare on SQL Server and Oracle. The SQL
  Server image has no first-start folder.
- Hostile attempts beyond the leftovers above, size and connection limits,
  and several scenarios at once.
- The console form.

## Changes to earlier RFDs

- **RFD 0013.** A `service` block under `pipeline`. A reserved
  `service.<kind>` namespace in the scope of `exec` and `scenario`,
  grounded by the block. `env` as a deferred expression. One more input to
  the parse phase. RFD 0013 lists external variable inputs as out of scope
  and says a pipeline is parsed against a fixed examination snapshot. That
  stays true for the author: no value is supplied by whoever starts a run.
  The added input is the platform's own.
- **RFD 0014.** The generator reads each coding question's language and
  adds `engine` to a declared block. `engine` joins the generator-owned
  attributes that a fragment may not set.
- **RFD 0015.** A pipeline run may have a companion container beside the
  grading container, made and removed with the job. The runner backend
  gains an exec into it. Decision 2's one job per run and the rejection of
  shared containers for grading are unchanged.

## Open questions

1. A scenario's password shows on the result screen. Is that acceptable as
   it is, or should results mask it?
2. Are the accounts' values new for each run, or fixed for a given config?
   New values keep anyone from typing them as literals. Fixed values keep a
   parse reproducible.
3. Should a stage with no scenario that reads the service get a warning at
   save, since it reads the database of `default`, which no test case
   shares?
4. The save check between the block and the language reads a version of
   the answer options. Is it the draft or the published one?
5. May Oracle Database Free be used for exams under its terms? Oracle's
   pages confirm its limits, not this use. SQL Server Developer is licensed
   for non-production use only, so the proposal is Express.
6. If T-SQL or PL/SQL gets no trustworthy parser, does the language ship
   without linting, and does the gate that holds registry languages against
   the linter's manifest change?
7. How many jobs and what age may a warm server reach before it is
   replaced?
8. A paper's pool holds five executors by default. Is that right when each
   carries a database container of 1.5 GB (SQL Server) or 2.5 GB (Oracle)?
9. Does the registry state which versions of an engine its statements are
   tested against?
10. The sandbox sets two Python variables for a Run that the pipeline does
    not write. They are the one existing case of a setting the pipeline
    does not show, and they make a Run differ from a graded run. Are they
    in scope to remove?

## Decision status

| Decision | Status |
|---|---|
| A database container beside the run container, for all four engines | decided |
| The pipeline declares `service "database"`; the block is what starts the container | decided |
| The body is empty in a fragment; the generator adds `engine` from the language | decided |
| The activity's image runs the engine, started through its entrypoint | decided |
| Commands read `service.database.host`, `port`, `user`, `password`, `name` | decided |
| One database per scenario, made from the default template | decided |
| Data is a test case's input; shared preparation by a first-start script | decided |
| Expected output compared with `diff`; no reference query | decided |
| A warm server is reused only after a verified reset; replaced on any difference | decided |
| `env` as a deferred expression | proposed |
| The parse phase assigns the accounts; the resolved pipeline carries them | proposed |
| MySQL started without its binary log | proposed |
| Scheduled replacement by job count and age | proposed |
| The engine in the executor's shape; prewarm per declared shape | proposed |
| Database container posture, per engine | proposed |
| A native sidecar in a grading Job | proposed |
| Language ids `mssql` and `oracle`; no generic `sql` | proposed |
| SQL Server Express and Oracle Database Free | proposed |

## Implementation

Checks by hand come first, on the items listed as not measured. A failing
check changes the design before product code is written.

1. **The registry and the engine operations** (core). The registry entry,
   one package per engine holding its statements, and tests that run each
   engine in a container and replay the leftovers and the planted account.
   The tests are heavy and get their own CI job.
2. **The pipeline language** (core). The `service` block and its rules,
   `env` as an expression, the scope, the stand-in at save, the binding and
   the per-scenario context at the parse phase, the engine and the accounts
   on the resolved pipeline, `engine` refused in a fragment, the generated
   reference and editor schema. The generator learns each question's
   language. The examination domain gains the save check and the warning.
   Every process must carry the new spec before any stored config uses the
   block.
3. **The sandbox** (core). The engine in the executor's shape. The Docker
   driver creates and removes the database container with the executor. The
   Kubernetes driver adds the container to the pod and takes a container
   name for exec, readiness and out-of-memory attribution. The orchestrator
   creates a scenario's account before its first stage. Reset verifies the
   server and marks the executor dirty on a difference.
4. **Graded runs** (core). The job spec gains the companion, behind a
   workflow version marker. Both runner backends create, adopt, clean up
   and sweep it. An activity after the container starts creates every
   scenario's account. The agent and its contract do not change: the
   arguments are plain strings before they reach it. On Kubernetes the
   pipeline worker needs `pods/exec` in the grading namespace.
5. **The console** (ui). The four presets, a test-case template with a load
   stage whose arguments can hold a reference, the Insert menu, the
   Database line, and the messages in place.
6. **SQL Server and Oracle as languages** (core, linter, ui).
7. **Deployment.** Memory for the database containers, the `pods/exec`
   rule, and the `NET_ADMIN` helper if Oracle keeps needing it.

Until step 3 ships, a database question can be written, highlighted and
linted, and has no supported way to run.

## UX

- **Instructors** set up a database question with the steps of any coding
  question. Choosing a database language and "Compare output" writes the
  block, the commands and the connection. Pipeline settings show a Database
  line: each test case runs against its own empty database, started from
  this activity's image.
- **A test case** has a name, its data, an expected output, points and a
  sample flag. Compare gains "Ignore row order".
- **The Run step** shows the command and the connection. The five values
  are inserted names and the other arguments are literal text, as today.
  This is the one place where the form shows a reference inside arguments.
- **Mistakes** are refused at save with the diagnostics above. When the run
  reports that the activity's image cannot start the engine, the question
  editor says so in place and points to Runtime.
- **The image builder** says that a saved image keeps what was installed
  and what is in the first-start folder, not a running server's data. The
  guide documents first-start scripts for an extension or shared reference
  data.
- **Students** see no difference from other coding questions: an editor
  with SQL highlighting, a Run button, the result of their query, and lint
  messages for syntax.
- **Ops** sizes the nodes for the database containers and grants the
  pipeline worker exec in the grading namespace.

## Out of scope

- **A database for a question whose language is not one**, such as a Python
  program that connects to PostgreSQL.
- **Reference data on MySQL that every test case reads without loading it.**
  The account would need a grant on a shared database.
- **The engine running beside the builder's terminal**, so an instructor
  can try a script there.
- **A restricted identity per command.**
- **Services that are not databases.** The block's label leaves room for
  them.
