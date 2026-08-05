# SQL Injection Prevention

It's 2026 and SQL injection is *still* on the OWASP Top 10 — over 25 years after it was first documented. That's not because developers are stupid. It's because the vulnerability isn't in SQL itself, it's in how we *build* SQL. One string concatenation, one missing parameter, one "temporary" hotfix, and your `users` table belongs to whoever found the right input field.

Here's the scary part: SQL injection isn't hard to exploit. A single quote in a login form is enough to test the waters, and automated scanners find these holes by the millions. This article covers how the attack actually works, why parameterized queries kill it dead, and where the "obvious" fixes still leave you exposed.

## What's actually happening

SQL injection happens when your application mixes *code* (the query structure) with *data* (user input) and can't tell them apart anymore.

```
// ❌ String concatenation: user input becomes part of the query structure
const query = "SELECT * FROM users WHERE email = '" + email + "' AND password = '" + password + "'";

// What happens when email is:  ' OR '1'='1
// The query becomes:
// SELECT * FROM users WHERE email = '' OR '1'='1' AND password = ''
// '1'='1' is always true → you're logged in as... whoever matches first
```

The database sees a perfectly valid query. It has no idea that part of it used to be user input. That's the whole game — the attacker isn't breaking SQL, they're *writing* SQL using your string concatenation as the keyboard.

## The attack family

Not all injections look the same. Once you understand the mechanics, you start recognizing the patterns:

| Attack type | What it does | Classic payload |
|-------------|--------------|-----------------|
| Classic / tautology | Bypasses auth by making the WHERE clause always true | `' OR '1'='1' --` |
| Union-based | Merges the result set with another table's data | `' UNION SELECT username, password FROM users --` |
| Boolean-based blind | Asks the DB yes/no questions to extract data bit by bit | `' AND (SELECT SUBSTRING(password,1,1) FROM users) = 'a' --` |
| Time-based blind | Uses delays as a yes/no channel when no output is visible | `' AND SLEEP(5) --` |
| Stacked queries | Runs a second, independent statement | `'; DROP TABLE users; --` |

The blind variants matter because most modern defenses (and most modern apps) don't echo database output back to the screen. So attackers adapt: instead of reading the data, they *ask* the database questions and measure the response time. `SLEEP(5)` — five second delay means the guess was right. Slow, tedious, and completely automatable.

## The fix: parameterized queries

The solution has existed since the 90s and it's embarrassingly simple: **separate the query structure from the data**. You send the database the query with placeholders, then send the values separately. The database engine treats the values as *values* — strings, numbers, whatever — never as SQL syntax. Even if someone passes `' OR '1'='1' --` as a value, it's just a weird string now.

```
// ✅ Parameterized query (Node.js + pg)
const result = await db.query(
  "SELECT * FROM users WHERE email = $1 AND password = $2",
  [email, password]   // values travel separately, never concatenated
);

// ✅ Parameterized query (Python + psycopg2)
cursor.execute(
  "SELECT * FROM users WHERE email = %s AND password = %s",
  (email, password)
);

// ✅ Parameterized query (Java + JDBC)
PreparedStatement stmt = conn.prepareStatement(
  "SELECT * FROM users WHERE email = ? AND password = ?"
);
stmt.setString(1, email);
stmt.setString(2, password);
```

Notice what's *not* in there: no string interpolation, no `+`, no `.format()`, no template literal building the WHERE clause. That's the whole fix. It's not a clever trick — it's simply refusing to mix code and data.

## ORMs help, until they don't

"If I use an ORM I'm safe, right?" Mostly — but only if you stay on the rails. ORMs parameterize everything they generate. The danger zone is every escape hatch they give you:

```
# ❌ Raw SQL through the ORM's escape hatch — full injection risk returns
User.objects.raw("SELECT * FROM users WHERE email = '%s'" % email)

# ✅ Same raw query, but parameterized properly
User.objects.raw("SELECT * FROM users WHERE email = %s", [email])

# ✅ Normal ORM usage — query builder parameterizes for you
User.objects.filter(email=email)
```

Same story in every ORM: Sequelize's `query()`, TypeORM's `createQueryBuilder().where(...)`, Rails' `find_by_sql`, Hibernate's native queries. The escape hatches exist for a reason — complex reporting, legacy schemas, performance-critical queries — but every raw query you write is a place where the discipline has to be reapplied manually. A good rule: **any raw SQL in your codebase gets a code review, full stop.**

## The fixes that don't actually fix it

You'll still find old advice floating around. Here's why each one is weaker than it looks:

**Escaping input.** Functions like `mysql_real_escape_string()` or `addslashes()` mangle the input so quotes can't break out. The problem: escaping is context-dependent and easy to get wrong, encoding tricks (`%bf%27`, Unicode normalization) have historically slipped through, and you have to remember to apply it *everywhere*. One missed call and you're back to square one. Parameterization doesn't require remembering anything — the boundary is structural, not behavioral.

**Stored procedures.** "We use stored procedures, so we're safe." You're safe *if* the procedure itself uses parameters:

```
-- ❌ The procedure concatenates internally — injection just moved one layer down
CREATE PROCEDURE get_user(IN email VARCHAR(255))
BEGIN
  SET @sql = CONCAT('SELECT * FROM users WHERE email = ''', email, '''');
  PREPARE stmt FROM @sql;
  EXECUTE stmt;
END;

-- ✅ Parameters inside the procedure — data stays data
CREATE PROCEDURE get_user(IN email VARCHAR(255))
BEGIN
  SELECT * FROM users WHERE email = email;
END;
```

Stored procedures are fine as a layer of control, but they're not a security boundary on their own.

**Input validation.** Blocking `'`, `--`, and `;` in inputs *reduces* the attack surface but is a blacklist — the exact pattern that fails eventually (see: every injection ever found despite input filters). Validate for data-quality reasons, but don't treat it as the injection defense. The injection defense is parameterization.

## The tricky part: dynamic SQL you can't parameterize

Here's the honest section. Sometimes you *can't* parameterize everything, because parameters can only stand in for *values*, not for *identifiers*. You can't do `ORDER BY $1` in most databases — the sort column has to be part of the query text.

So when you build dynamic queries for sortable tables, filterable search, or schema-driven tooling:

```
// ❌ Whitelisting the *input* — an attacker just needs one value you forgot
const allowed = ["name", "email", "created_at"];
const column = req.query.sort;
if (allowed.includes(column)) {           // "name" passes...
  orderBy(column);                        // ...but so does anything in the list
}

// ✅ Whitelist the *mapping* — input is an index, not SQL
const COLUMNS = {
  name: "name",
  email: "email",
  created: "created_at",
  newest: "created_at DESC",
};
const column = COLUMNS[req.query.sort] ?? "created_at DESC";
// req.query.sort is never used as SQL — it's a lookup key
```

The pattern: never let user input reach the query text in *any* form. Map it through a fixed lookup table first. If the input doesn't match a key, use the default. Same approach for table names, schema names, and anything else that can't be parameterized.

## Defense in depth — because you will slip up

Even with parameterization everywhere, treat the database as if it's one mistake away from being queried by strangers:

- **Least privilege**: your app's DB user shouldn't be able to `DROP TABLE` or `SELECT` from every schema. If the login service only needs `SELECT` on `users` and `INSERT` on `sessions`, grant exactly that. Injection damage is capped by what the account can do.
- **Never leak query errors**: that stack trace with the full SQL statement is a gift to attackers refining their payloads. Log the details server-side, return a generic error to the client.
- **Hide the database**: no direct DB exposure from the internet, ever. Connections through the app layer only.
- **WAFs and scanners**: a WAF can block obvious payloads and DAST scanners will find the injections you missed before attackers do. They're a safety net, not the primary defense — a WAF is pattern-matching, and patterns can be bypassed.
- **Test for it**: automated tests that deliberately send `' OR 1=1 --` through every input field are cheap and catch regressions the moment a raw query sneaks back in.

## The trade-off worth naming

Parameterized queries have a real cost: you lose some flexibility. Building complex dynamic queries gets clunkier, some databases handle prepared statements less efficiently in specific edge cases (plan caching quirks with wildly different parameter values), and ORM-generated SQL is sometimes less optimal than hand-tuned queries.

But here's the thing — those are *performance and ergonomics* problems with known workarounds. A SQL injection is a *security incident* with data loss, legal exposure, and a 3am incident call. The asymmetry isn't even close. Optimize the query after you've secured it, not instead of.

## Actionable takeaways

- **Parameterize every query that touches user input** — values go as parameters, never as concatenated text. This is non-negotiable.
- **Treat ORM raw-query escape hatches as code-review magnets** — the ORM keeps you safe until you reach for `query()` or `raw()`.
- **Never rely on escaping, stored procedures, or input filtering as the injection defense** — they're layers, not the fix.
- **For dynamic SQL (ORDER BY, table names), map input through a fixed lookup table** — user input becomes a key, never SQL text.
- **Run the DB with least-privilege credentials** — the damage an injection can do is bounded by the account it runs under.
- **Write a regression test that sends `' OR 1=1 --` through every input** — it's the fastest way to catch a slipped raw query.

SQL injection isn't a clever attack. It's the predictable result of mixing code with data, and it's been fully solved for decades. The only way it survives is through individual moments of carelessness — which is exactly why the fix has to be structural, not behavioral.
