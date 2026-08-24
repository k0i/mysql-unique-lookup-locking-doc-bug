# Report for Bug #121151

## Background

[Bug #121151](https://bugs.mysql.com/bug.php?id=121151) originally attributed an unexpected next-key lock on a page's supremum pseudo-record — despite the InnoDB Locking manual's [guarantee](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html#innodb-gap-locks) that unique-index equality searches need no gap lock — to `WHERE unique_col IN (...)` compiling into `range` access.

**That diagnosis was wrong.** `IN (...)` has nothing to do with it. The actual cause is that the matched row is the first record of a non-leftmost leaf page — a plain single-value `const`-access equality lookup (`SELECT * FROM child WHERE id = 100`, the manual's own example) hits the same lock whenever that's true of the matched row, regardless of how the query reached it.

## Repro

```sql
DROP DATABASE IF EXISTS innodb_locking_repro2;
CREATE DATABASE innodb_locking_repro2;
USE innodb_locking_repro2;

CREATE TABLE repro (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  filler VARCHAR(255) NOT NULL DEFAULT '',
  PRIMARY KEY (id)
) ENGINE=InnoDB;

-- A wider row means fewer rows fit per 16KB page, so a page boundary
-- is reached within a small number of rows.
SET SESSION cte_max_recursion_depth = 500;
INSERT INTO repro (filler)
WITH RECURSIVE seq(n) AS (SELECT 1 UNION ALL SELECT n + 1 FROM seq WHERE n < 500)
SELECT REPEAT('x', 190) FROM seq;

ANALYZE TABLE repro;
```

Find the real per-page record boundaries for this specific build (row-size- and version-dependent, don't assume a fixed id) using `innodb_space` from [jeremycole/innodb_ruby](https://github.com/jeremycole/innodb_ruby):

```
innodb_space -f repro.ibd space-index-pages-summary
```

This lists each leaf page's record count for the `PRIMARY` index. Taking a running total over the pages (in the order the tool lists them) gives the first id of each non-leftmost leaf page — that id is the one to test.

```sql
-- ============================================================
-- Session A: setup + the locking statement (leave transaction open)
-- ============================================================
START TRANSACTION;

-- <first_id_of_a_non_leftmost_leaf_page> from the innodb_space output above.
-- EXPLAIN confirms this is `type: const` -- a single exact value, unique index.
SELECT * FROM repro WHERE id = <first_id_of_a_non_leftmost_leaf_page> FOR UPDATE;

-- Do NOT COMMIT yet -- leave this session open and switch to Session B.

-- ============================================================
-- Session B: run this while Session A's transaction above is still open
-- ============================================================
SELECT LOCK_MODE, LOCK_STATUS, LOCK_DATA
FROM performance_schema.data_locks
WHERE OBJECT_SCHEMA = 'innodb_locking_repro2' AND OBJECT_NAME = 'repro';

-- Expected if the manual's stated guarantee held without qualification for
-- any const-access unique-index equality search:
--   1 row of X,REC_NOT_GAP (on the matched id) + 1 IX (table-level intent lock).
-- Actual:
--   X,REC_NOT_GAP on the matched id, PLUS one additional row:
--     LOCK_MODE=X, LOCK_DATA='supremum pseudo-record'

-- Back in Session A: ROLLBACK to release the locks and clean up.
```

For comparison, querying the id immediately *before* the boundary (the last record of the *previous* page, also `type: const`) takes only `X,REC_NOT_GAP` — no extra lock. This rules out "every row near a boundary" as the trigger; specifically, it's the first record of a page that's affected.

## Mechanism

Consistent with the original report's citation of `row_search_mvcc` (`storage/innobase/row/row0sel.cc`): a next-key lock on a matched record also locks the gap immediately *preceding* it, to guard against phantom inserts under REPEATABLE READ. When the matched record is the first row on its (non-leftmost) leaf page, there is no predecessor on that same page — the logical gap "before" it is represented on disk by the *previous* page's supremum pseudo-record. Taking a next-key lock on the matched record therefore locks that supremum, regardless of whether the record was reached via `range` or `const` access.

## Verification

Reproduced on MySQL 8.3.0 with the exact repro above (500 rows, `REPEAT('x', 190)` filler). `innodb_space -f repro.ibd space-index-pages-summary` confirmed the `PRIMARY` index's leaf pages hold, in order: 34, 69, 69, 69, 69, 69, 69, 52 records — so id 34 is the last record of the first leaf page and id 35 is the first record of the second.

`EXPLAIN SELECT * FROM repro WHERE id = 34` and `WHERE id = 35` both confirm `type: const`.

#### performance_schema.data_locks

Session A holds `SELECT * FROM repro WHERE id = 34 FOR UPDATE` open; session B's query returns:

```
LOCK_MODE       LOCK_STATUS  LOCK_DATA
IX              GRANTED      NULL
X,REC_NOT_GAP   GRANTED      34
```

Session A holds `SELECT * FROM repro WHERE id = 35 FOR UPDATE` open instead; the same query returns:

```
LOCK_MODE       LOCK_STATUS  LOCK_DATA
IX              GRANTED      NULL
X               GRANTED      supremum pseudo-record
X,REC_NOT_GAP   GRANTED      35
```

#### SHOW ENGINE INNODB STATUS

Same session A (`id = 35 FOR UPDATE`, still open), with `SET GLOBAL innodb_status_output_locks = ON` first:

```
RECORD LOCKS space id 10817 page no 5 n bits 136 index PRIMARY of table `innodb_locking_repro2`.`repro` trx id 26562979 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

RECORD LOCKS space id 10817 page no 6 n bits 136 index PRIMARY of table `innodb_locking_repro2`.`repro` trx id 26562979 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 8; hex 0000000000000023; asc        #;;
 ...
```

The matched row (`id = 35`, hex `0x23`) lives on **page 6**, correctly `locks rec but not gap`. The extra lock — on-disk bytes spelling `supremum` — is on **page 5**, the page before it.
