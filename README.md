# libpq — PostgreSQL for sysl

A binding of PostgreSQL's own client library, with **no C of its own**.

*An independent sysl package for PostgreSQL. It is not affiliated with or endorsed by PostgreSQL's authors or trademark holders.*

```sysl
import sh.sysl.postgres.*

var db = connect("host=127.0.0.1 dbname=shop user=app")?

val rows = db.query("select id, email from users where age > $1", [Param.Text("21")])?

for r in 0..<rows.rows()
    print(s"${rows.integer(r, 0).unwrap()} ${rows.text(r, 1).unwrap()}")
```

```
dependencies {
  postgres { git = "github.com/sysl-lang/libpq", version = "0.1.1" }
}
```

The repository is named for the library it binds and the module for the thing it talks to, which is
the split `hiredis`/`sh.sysl.redis` already makes.

## Every value goes out of band, and there is no other way to write a query

**`PQexec` is not bound.** It takes a command string and nothing else, which is the shape an
injection is written in, and offering it beside the safe call would make the unsafe one the
convenient one. What is here is `PQexecParams` and `PQsendQueryParams`: the SQL carries `$1`, `$2`,
`$3`, the values travel beside it, and the server never parses a value as syntax.

```sysl
val nasty = "'; drop table users; --"

db.query("select $1::text", [Param.Text(nasty)])?      // one string, with a semicolon in it
```

A parameter is `Param.Text`, `Param.Binary` or `Param.Null`. **`Null` is not `Text("")`** — the two
are different values and the server treats them differently, which is the whole reason the enum has
three cases rather than an empty string standing in.

**An identifier is the one thing a parameter cannot be.** `select * from $1` is not legal SQL, so a
program naming a table it was told about at run time uses `identifier()`, which is libpq's own
quoting and knows the server's encoding and its `standard_conforming_strings` setting:

```sysl
val name = db.identifier(whatever_the_user_said)?      // "odd""name"

db.query(s"select * from $name", [])?
```

Quoting makes `users; drop table x` a *table called* `users; drop table x`, which is safe and is not
what anybody meant. Deciding the program is willing to name that table is still the caller's.

## The asynchronous half is bound, because a server needs it

`PQexec` blocks the thread until the server answers, which in a program serving more than one
request at a time is the whole thread. The other road is a state machine over a file descriptor, and
it pairs with a `libuv` poll handle or a plain `select` loop:

```sysl
var db = connect_start("host=127.0.0.1 dbname=shop")?

var ready = false

while !ready
    db.poll() match                     // wait on db.socket() in the direction it names
        Poll.Ready -> ready = true
        Poll.Failed -> return Err(...)
        _ -> ()

db.send("select 1", [])?

db.consume()?                           // whenever the socket is readable

if !db.busy()
    while db.result()? match
        Some(rows) -> ...
        None -> false
```

**`result()` must be drained to `None`.** libpq answers one `PGresult` per command in the string and
then a null to say the command is finished; a connection whose last result was never collected
refuses the next command. That is libpq's rule rather than this package's, and it is the one thing
about the asynchronous path that is easy to get wrong — so it is a test here rather than a sentence.

`single_row_mode()` makes each result hold one row, for a query whose answer is larger than memory.

## What a value is

| | |
|---|---|
| `bytes(r, c)` | the cell's bytes, **borrowed from the result** |
| `text(r, c)` | the cell as a `string`, **copied**, so it outlives the row set |
| `integer` / `number` / `boolean` | parsed; `None` for NULL **or** for a column that does not hold one |
| `bytea(r, c)` | the `\x…` form decoded |
| `is_null(r, c)` | the only thing that tells NULL from an empty string |

**`None` means SQL NULL.** `PQgetvalue` answers an empty string for a NULL and for an empty string
alike, which is why `is_null` exists at all.

The borrow/copy split is deliberate. A row set can be large and the borrow is what makes reading one
free; a `string` that outlived its `Rows` would be a use-after-free that prints correctly until the
memory is reused. There is a test that reads a `text()` inside a function, lets the `Rows` die, and
checks the value afterwards with a hundred thousand bytes of churn in between — under
AddressSanitizer that would report a `heap-use-after-free` if `text()` had merely pointed.

## Failures carry SQLSTATE

```sysl
db.query("insert into users (email) values ($1)", [Param.Text(email)]) match
    Ok(_) -> ()
    Err(e) -> e match
        Error.Failed(state, _) if state == unique_violation -> already_have_that_one()
        Error.Failed(_, why) -> return Err(why)
        _ -> return Err("the database went away")
```

`Failed` is the server refusing a command; everything else is the connection. That is the division a
caller acts on — a `Failed` is answered by fixing the statement or the data, and the rest by
reconnecting. The five-character SQLSTATE is what to match on: the message text is localised and its
wording moves between versions. The codes worth naming are constants here — `unique_violation`,
`foreign_key_violation`, `not_null_violation`, `check_violation`, `serialization_failure`,
`deadlock_detected`, `undefined_table`, `undefined_column`, `syntax_error`,
`insufficient_privilege` — and they are strings rather than an enum because the set is open.

## `LISTEN`/`NOTIFY`, cancelling, and health

`notification()` answers PostgreSQL's own pub/sub, with the channel, the notifying backend's pid and
the payload, all copied out of libpq's storage. **Notifications are collected by `consume()` and by
any query**, so a program that only listens still has to call `consume()` for anything to arrive.

`cancel()` asks the server to abandon whatever the connection is running. It is a request rather
than a guarantee: `Ok` means the cancel was delivered, and the command in flight still has to be
drained — it may succeed, or fail with `57014`, which is what a cancel that landed looks like.

`health()` is `PQping`, which is what `pg_isready` runs. It costs no session on the server, so it is
the call for a health check.

## No shim, and the survey that established it

Every name in `sh.sysl.postgres.c` is a symbol in `libpq.dylib`, and there is no C anywhere in this
package. The three shapes only C can reach are a caller-allocated opaque type, a macro standing in
for a symbol, and a shape with no sysl spelling; libpq's client surface has none of them:

- `PGconn`, `PGresult` and `PGcancel` are allocated **by the library**, so nothing here has to know
  how big one is.
- `libpq-fe.h` declares **no `static inline` at all**, and its only two function-like macros are
  `PQsetdb` (a deprecated spelling of `PQsetdbLogin`) and `PQfreeNotify` (a spelling of `PQfreemem`).
  Neither is needed.
- Parameters go in as `char * const *` and values come back as `char *`, which are `**u8` and `*u8`.

`PGnotify` is the one struct read through, and the header's own comment says it would be opaque if
it were not so simple. It is declared field for field, with `sizeof` and three `offsetof`s asserted
against the header — so a libpq that reordered or grew it fails a test rather than reading the wrong
field at run time.

**Binary-format results are one integer rather than a second code path**, which is why they are
offered at all: `query(..., Format.Binary)` and `bytes()` has the raw wire bytes. It is
all-or-nothing per result, which is libpq's rule; `bytea()` covers the ordinary reason to want it
without turning the other columns binary.

## The parameter buffer, and why it is built the way it is

Every parameter's bytes go into one `Buf` and the pointers are computed **only after the last of
them has been written**. A `Buf` moves its storage when it grows, so a pointer taken during the loop
is dangling by the end of it — and the failure is silent, because the freed bytes are usually still
the ones that were there.

`many_parameters_survive_the_buffer_growing` is the test that sees it: sixty-four parameters of
growing length force several reallocations. Breaking `bind` on purpose reddens it while
`select $1::int + $2::int` still passes, which is the whole reason a wide test is there beside the
narrow ones.

## Handles are boxed, so their destructors run

`connect` hands back a `&Connection` and `query` a `&Rows`, and that is load-bearing rather than
stylistic. **A destructor runs when a box's strong count reaches zero and for nothing else** — a
plain struct value is copied rather than owned, so a `Drop` on a type nobody boxes is never called.
A type annotation at the call site does not fix it either: reading a value into a `&Connection`
binding boxes a *copy*.

Two paths need more than the box, because on them nothing owns the handle at all: a **failed
connect** still returns a `PGconn` (which is how libpq reports the failure) and a **failed command**
still returns a `PGresult` (which is where the SQLSTATE is). Both are read and then closed where
they stand.

`connections_are_closed_when_they_go` is the regression test, and it works because PostgreSQL's
`max_connections` defaults to 100: a hundred and fifty connections that are actually closed is fine,
and a hundred and fifty that are not is refused by the server. The other three paths are memory
only, so they are checked by measuring — `/usr/bin/time -l` over a loop:

| loop, 20 000–30 000 iterations | before | after |
|---|---|---|
| `query` returning rows | 179 MB | 18.4 MB |
| a command the server refuses | 85 MB | 18.4 MB |
| a connect to a port with nothing on it | 738 MB | 18.6 MB |

Each figure is flat against the same program at a tenth the iterations.

## Building

libpq has to be installed, and so do its headers. **Nothing is vendored**, and the reason is the
org's rule rather than the size of the download: vendoring is a claim that the code can run
somewhere a package manager cannot reach, and there is no microcontroller that wants a PostgreSQL
client.

```
brew install libpq              # macOS, or the whole of postgresql@16
sudo apt install libpq-dev      # Debian / Ubuntu
sudo pacman -S postgresql-libs  # Arch
```

The build needs no flags — `pkg-config` is asked where the library is. On macOS the keg-only `libpq`
formula is not on the default search path, so a `brew install libpq` alone wants

```
export PKG_CONFIG_PATH=/opt/homebrew/opt/libpq/lib/pkgconfig
```

which `postgresql@16` does not need.

**`requires { heap, posix }`.** Every handle is one libpq allocated; and a database client is a
socket and a pile of files — libpq reads `~/.pgpass`, `~/.pg_service.conf`, `PGHOST` and the rest of
the environment, and reaches the machine's certificate store when the connection is TLS.

## Running the suite

Most of these tests need a server, and that is deliberate: a libpq binding is a wire protocol, an
authentication handshake and a type system that all live on the other side of a socket, so a suite
that mocked them would be checking that this package agrees with itself. **An absent server reddens
the suite rather than skipping it**, since a green run over nothing is the one result worth less
than a red one.

Four commands bring one up, on port 5433 so that it cannot be confused with a server the machine is
already running:

```
initdb -D /tmp/sysl-pg -U sysl --auth=trust
mkdir -p /tmp/pgsock
pg_ctl -D /tmp/sysl-pg -l /tmp/sysl-pg.log -o "-p 5433 -k /tmp/pgsock" start
sysl test .
```

`SYSL_PG_TEST` overrides the connection string; it defaults to
`host=127.0.0.1 port=5433 dbname=postgres user=sysl`. Stop it with
`pg_ctl -D /tmp/sysl-pg stop`.

**51 tests**, against PostgreSQL 16.14 and libpq 16.14, covering connect, parameters, NULL, the
result accessors, SQLSTATE, `bytea` in both formats, identifier quoting, transaction status, the
whole asynchronous path, single-row mode, notifications, cancelling, and the guards.

**AddressSanitizer clean**, suite and a live program, with `nm -u` showing 21 `__asan_*` symbols
including `__asan_memcpy` — the instrumentation confirmed rather than assumed:

```
SYSL_EXTRA_CFLAGS="-fsanitize=address -g" sysl test .
```

**It covers the sysl half and not libpq.** The objects came from a package manager months ago and no
flag of ours instruments them; what the sanitizer is watching here is the raw-pointer paths on this
side of the boundary.

## What is missing

`COPY` — `PQputCopyData`, `PQgetCopyData` — which is how a bulk load is done and is a code path of
its own. Pipeline mode. `PQprepare`/`PQexecPrepared`, which is worth having when one statement runs
many times. Notice and warning callbacks. Building a connection from a `PQconninfoOption` array
rather than a string. None has been asked for.
