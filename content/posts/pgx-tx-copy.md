---
title: "pgx, database/sql transactions, and COPY"
date: 2025-02-02T14:01:32-04:00
---

At [work](https://www.grax.com/), we've just about completed a migration from the [lib/pq](https://pkg.go.dev/github.com/lib/pq) Go PostgreSQL driver to [pgx](https://pkg.go.dev/github.com/jackc/pgx/v5).

We use the pgx [database/sql compatibility layer](https://pkg.go.dev/github.com/jackc/pgx/v5/stdlib) so our code doesn't need to care which driver is used, for the most part.

As we neared the end of the migration, one unsolved issue remained: using [`COPY`](https://www.postgresql.org/docs/current/sql-copy.html) inside transactions.

We use `COPY` in a few places to bulk load data. With lib/pq, that looks something [like this](https://pkg.go.dev/github.com/lib/pq#hdr-Bulk_imports):

``` go
// db is a *database/sql.DB
tx, err := db.Begin()
if err != nil { // ...
defer tx.Rollback()

stmt, err := tx.Prepare(pq.CopyIn("users", "name", "age"))
if err != nil { // ...

for _, user := range users {
	_, err = stmt.Exec(user.Name, int64(user.Age))
	if err != nil { // ...
}

_, err = stmt.Exec()
if err != nil { // ...

err = stmt.Close()
if err != nil { // ...

err = tx.Commit()
if err != nil { // ...
```

`pq.CopyIn("users", "name", "age")` produces a statement that looks like:

``` sql
COPY "users" ("name", "age") FROM STDIN
```

This statement is detected by some [behind the scenes support](https://github.com/lib/pq/blob/b7ffbd3b47da4290a4af2ccd253c74c2c22bfabf/conn.go#L860) lib/pq has as part of `tx.Prepare`. It then handles the special interactions needed for `COPY` as part of `stmt.Exec`.

pgx [has support for `COPY`](https://pkg.go.dev/github.com/jackc/pgx/v5#hdr-Copy_Protocol).
However, its database/sql compatibility layer does not have the same behind the scenes support as lib/pq.

The documentation gives an example of using `pgx.Conn.CopyFrom` from a pgx-backed database/sql.DB but doesn't give an example of how to do it as part of a transaction.

That's what we had to figure out to complete our migration.

We have code that looks much like the lib/pq example above, where we also use the database/sql.Tx for other things.
That includes passing it around a bit.

After some testing, we arrived at something not too far off the pgx example that meets our needs:

``` go
// First, explicitly check out a database/sql.Conn from the DB.
conn, err := db.Conn(ctx)
if err != nil { // ...
defer conn.Close()

// Then, begin a transaction on that connection.
tx, err := conn.BeginTx(ctx, nil)
if err != nil { // ...
defer tx.Rollback()

// Use tx...

// Do a COPY inside the transaction:
err = conn.Raw(func(driverConn any) error {
	// Do not use tx in here!

	// stdlib is github.com/jackc/pgx/v5/stdlib
	stdlibConn, ok := driverConn.(*stdlib.Conn)
	if !ok { // error ...
	pgxConn := stdlibConn.Conn() // The underlying *pgx.Conn
	_, err := pgxConn.CopyFrom(
		ctx,
		pgx.Identifier{table},
		columns,
		pgx.CopyFromRows(rows),
	)
	if err != nil { // ...
	return nil
})
if err != nil { // ...

// Eventually commit the transaction:
err = tx.Commit()
if err != nil { // ...
```

Since both `BeginTx` and `Raw` are called on the same explicitly checked-out connection anything that happens in `Raw`'s func, such as the use of `pgxConn.CopyFrom`, happens inside the transaction.

One major caveat: in this example, `tx` must not be used inside `Raw`'s func.
This is because at a low level things like `tx.QueryContext` and `Raw` take the same lock.

Getting this sorted let us convert the few places we were using `pq.CopyIn` to this setup and continue our migration.
