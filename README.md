# SQL Window Functions & CTEs — Complete Tutorial Notes

Companion notes for the YouTube tutorial. Use this guide to follow the examples, revise each concept, and practise the queries yourself.

You will learn how to rank players, compare consecutive innings, calculate running totals and moving averages, understand window frames, and build readable queries with CTEs.

**SQL dialect:** MySQL 8.0+  
**Example:** Cricket analytics using eight players and 24 innings  
**Prerequisites:** Basic `SELECT`, `WHERE`, `GROUP BY`, and `ORDER BY`

The scores are fictional teaching data. Every example in these notes uses the same `demo_innings` table, which you can create below.

## Contents

- [Set up the practice data](#set-up-the-practice-data)
- [1. Window functions and OVER](#1-window-functions-and-over)
- [2. Ranking and distribution functions](#2-ranking-and-distribution-functions)
- [3. LAG and LEAD](#3-lag-and-lead)
- [4. Window frames, running totals, and moving averages](#4-window-frames-running-totals-and-moving-averages)
- [5. FIRST_VALUE, LAST_VALUE, and NTH_VALUE](#5-first_value-last_value-and-nth_value)
- [6. CTEs and the WITH clause](#6-ctes-and-the-with-clause)
- [7. Percentage contributions](#7-percentage-contributions)
- [Common mistakes](#common-mistakes)
- [Quick revision sheet](#quick-revision-sheet)
- [Practice questions](#practice-questions)

## How to use these notes

1. Create the practice table once.
2. Read the question before each query and predict its result.
3. Run the complete query in your MySQL editor.
4. Compare your output with the result shown, then change one part of the query to explore its effect.

## Before you begin

You should be comfortable with `SELECT`, `WHERE`, `ORDER BY`, `GROUP BY`, aggregate functions, and basic joins. Use a MySQL connection in your preferred SQL editor or the MySQL command-line client.

Run the demo setup once in an empty practice database. It creates a table and inserts 24 rows. If you already have the same `demo_innings` data, skip the setup and run the lesson queries. Repeating the setup against an existing table will fail rather than silently replace your work.

## Set up the practice data

Create and select a practice database, or select an existing empty database if your account cannot create one:

```sql
CREATE DATABASE sql_window_tutorial CHARACTER SET utf8mb4;
USE sql_window_tutorial;

CREATE TABLE demo_innings (
    player_name VARCHAR(50) NOT NULL,
    team_name   VARCHAR(10) NOT NULL,
    match_no    INT NOT NULL,
    runs        INT NOT NULL,
    PRIMARY KEY (player_name, match_no)
);

INSERT INTO demo_innings (player_name, team_name, match_no, runs) VALUES
('Arjun',  'CSK', 1, 88), ('Arjun',  'CSK', 2, 42), ('Arjun',  'CSK', 3, 70),
('Dev',    'CSK', 1, 75), ('Dev',    'CSK', 2, 65), ('Dev',    'CSK', 3, 60),
('Kiran',  'CSK', 1, 40), ('Kiran',  'CSK', 2, 50), ('Kiran',  'CSK', 3, 60),
('Rohit',  'MI',  1, 95), ('Rohit',  'MI',  2, 80), ('Rohit',  'MI',  3, 75),
('Sam',    'MI',  1, 60), ('Sam',    'MI',  2, 70), ('Sam',    'MI',  3, 70),
('Neel',   'MI',  1, 50), ('Neel',   'MI',  2, 45), ('Neel',   'MI',  3, 55),
('Ishaan', 'RCB', 1, 80), ('Ishaan', 'RCB', 2, 75), ('Ishaan', 'RCB', 3, 65),
('Kabir',  'RCB', 1, 45), ('Kabir',  'RCB', 2, 55), ('Kabir',  'RCB', 3, 50);

SELECT COUNT(*) AS innings_count,
       COUNT(DISTINCT player_name) AS player_count,
       SUM(runs) AS total_runs
FROM demo_innings;
```

Expected: **24 innings, 8 players, 1,520 runs**. Each player has three innings. `match_no` represents that player's ordered innings in this simplified example.

| Player | Team | Match 1 | Match 2 | Match 3 | Total |
| --- | --- | ---: | ---: | ---: | ---: |
| Rohit | MI | 95 | 80 | 75 | 250 |
| Ishaan | RCB | 80 | 75 | 65 | 220 |
| Arjun | CSK | 88 | 42 | 70 | 200 |
| Dev | CSK | 75 | 65 | 60 | 200 |
| Sam | MI | 60 | 70 | 70 | 200 |
| Kabir | RCB | 45 | 55 | 50 | 150 |
| Kiran | CSK | 40 | 50 | 60 | 150 |
| Neel | MI | 50 | 45 | 55 | 150 |

## 1. Window functions and OVER

**Question:** Can we display every innings alongside the player's total runs?

An aggregate query with `GROUP BY` returns one row for each group:

```sql
SELECT player_name, SUM(runs) AS player_total
FROM demo_innings
GROUP BY player_name
ORDER BY player_name;
```

This returns eight rows, one per player. A window function calculates across related rows while retaining each innings:


```sql
SELECT player_name, match_no, runs,
       SUM(runs) OVER (PARTITION BY player_name) AS player_total,
       MAX(runs) OVER () AS highest_innings
FROM demo_innings
ORDER BY player_name, match_no;
```

This returns all 24 innings. Arjun's three rows each show `player_total = 200`; every row shows `highest_innings = 95`.

`PARTITION BY` groups rows for a window calculation. An `ORDER BY` inside `OVER` controls the calculation's sequence; the final query's `ORDER BY` controls the displayed order. See [MySQL's window concepts](https://dev.mysql.com/doc/refman/8.0/en/window-functions-usage.html).

## 2. Ranking and distribution functions

**Question:** How can we compare players by total runs while handling ties correctly?

First, add up each player's runs. The `WITH season_totals AS (...)` block below names that intermediate result; CTEs are explained in section 6.

| Function | Meaning | Remember |
| --- | --- | --- |
| `ROW_NUMBER()` | A unique position for each row | Add a tie-breaker for consistent numbering |
| `RANK()` | Equal scores share a rank | The next rank can have a gap |
| `DENSE_RANK()` | Equal scores share a rank | The next distinct score gets the next rank |
| `NTILE(4)` | Divide ordered players into four groups | Groups are based on row counts |
| `CUME_DIST()` | Cumulative share through the current peer group | Read the result in the chosen sort direction |
| `PERCENT_RANK()` | Relative position of the current rank | Different from the cumulative share |

Run this comparison query:


```sql
WITH season_totals AS (
    SELECT player_name, SUM(runs) AS total_runs
    FROM demo_innings
    GROUP BY player_name
)
SELECT player_name, total_runs,
       ROW_NUMBER() OVER (
           ORDER BY total_runs DESC, player_name
       ) AS row_num,
       RANK() OVER (ORDER BY total_runs DESC) AS player_rank,
       DENSE_RANK() OVER (ORDER BY total_runs DESC) AS dense_rank_value,
       NTILE(4) OVER (
           ORDER BY total_runs DESC, player_name
       ) AS performance_group,
       ROUND(CUME_DIST() OVER (
           ORDER BY total_runs DESC
       ) * 100, 2) AS cumulative_pct,
       ROUND(PERCENT_RANK() OVER (
           ORDER BY total_runs DESC
       ) * 100, 2) AS percent_rank_pct
FROM season_totals
ORDER BY total_runs DESC, player_name;
```

| Player | Total | Row number | Rank | Dense rank | NTILE(4) | Cumulative % | Percent rank % |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Rohit | 250 | 1 | 1 | 1 | 1 | 12.50 | 0.00 |
| Ishaan | 220 | 2 | 2 | 2 | 1 | 25.00 | 14.29 |
| Arjun | 200 | 3 | 3 | 3 | 2 | 62.50 | 28.57 |
| Dev | 200 | 4 | 3 | 3 | 2 | 62.50 | 28.57 |
| Sam | 200 | 5 | 3 | 3 | 3 | 62.50 | 28.57 |
| Kabir | 150 | 6 | 6 | 4 | 3 | 100.00 | 71.43 |
| Kiran | 150 | 7 | 6 | 4 | 4 | 100.00 | 71.43 |
| Neel | 150 | 8 | 6 | 4 | 4 | 100.00 | 71.43 |

`ROW_NUMBER` assigns unique positions; `RANK` leaves gaps after ties; `DENSE_RANK` does not. `NTILE` distributes rows into groups and may split ties. With descending scores, `CUME_DIST` includes all players at or above the current score. `PERCENT_RANK` uses `(rank - 1) / (row_count - 1)` for partitions with more than one row; a single-row partition returns zero. The percentages above are numeric values multiplied by 100. See [MySQL's function definitions](https://dev.mysql.com/doc/refman/8.0/en/window-function-descriptions.html).

The name tie-breaker is intentional only for `ROW_NUMBER` and `NTILE`. Adding it to the other four windows would make equal scores cease to be peers.

### Understand the percentages

For a player with 200 total runs, five of the eight players scored 200 or more:

```text
Cumulative percentage = 5 / 8 × 100 = 62.50%
```

Those players occupy rank 3 in a leaderboard with eight rows:

```text
Percent rank = (3 - 1) / (8 - 1) × 100 = 28.57%
```

All three 200-run players receive the same values. The bottom players share rank 6, so their percent rank is `(6 - 1) / 7 × 100 = 71.43%`.

For `NTILE(4)`, eight players form four groups of two. Arjun and Dev go into group 2, while Sam goes into group 3 despite having the same total. In general, bucket sizes differ by at most one, with larger buckets first.

## 3. LAG and LEAD

**Question:** What did Arjun score immediately before and after each innings?


```sql
SELECT player_name, match_no, runs,
       LAG(runs) OVER (
           PARTITION BY player_name ORDER BY match_no
       ) AS previous_runs,
       LEAD(runs) OVER (
           PARTITION BY player_name ORDER BY match_no
       ) AS next_runs
FROM demo_innings
WHERE player_name = 'Arjun'
ORDER BY match_no;
```

| Player | Match | Runs | Previous runs | Next runs |
| --- | ---: | ---: | ---: | ---: |
| Arjun | 1 | 88 | NULL | 42 |
| Arjun | 2 | 42 | 88 | 70 |
| Arjun | 3 | 70 | 42 | NULL |

Both functions default to an offset of one row and return `NULL` when that neighboring row does not exist. `LEAD` reads recorded data; it does not predict a score. See [MySQL's LAG and LEAD reference](https://dev.mysql.com/doc/refman/8.0/en/window-function-descriptions.html#function_lag).

Read the result row by row:

- Match 1 has no previous innings, so its previous value is `NULL`. Its next value is match 2's score, 42.
- Match 2 looks back to 88 and ahead to 70.
- Match 3 looks back to 42. There is no next innings, so its next value is `NULL`.

To calculate improvement, subtract `previous_runs` from `runs` in an outer query. Arjun's changes are `NULL`, `-46`, and `28`.

## 4. Window frames, running totals, and moving averages

**Question:** How many runs had Arjun accumulated after each innings, and what was his average over the latest two innings?

A partition defines the group. Ordering arranges that group. A frame selects the rows used for a frame-sensitive calculation at the current row.

| Frame boundary | Meaning |
| --- | --- |
| `UNBOUNDED PRECEDING` | Start of the partition |
| `1 PRECEDING` | One ordered row before the current row when using `ROWS` |
| `CURRENT ROW` | Current row; `RANGE` also considers its peers |
| `1 FOLLOWING` | One ordered row after the current row when using `ROWS` |
| `UNBOUNDED FOLLOWING` | End of the partition |


```sql
SELECT player_name, match_no, runs,
       SUM(runs) OVER (
           PARTITION BY player_name ORDER BY match_no
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS running_total,
       ROUND(AVG(runs) OVER (
           PARTITION BY player_name ORDER BY match_no
           ROWS BETWEEN 1 PRECEDING AND CURRENT ROW
       ), 2) AS last_two_match_avg
FROM demo_innings
WHERE player_name = 'Arjun'
ORDER BY match_no;
```

| Match | Runs | Running total | Last-two average |
| --- | ---: | ---: | ---: |
| 1 | 88 | 88 | 88.00 |
| 2 | 42 | 130 | 65.00 |
| 3 | 70 | 200 | 56.00 |

The first average contains one available row. At match 3, the moving average includes 42 and 70, excluding 88.

`ROWS` uses positions in the ordered rows. `RANGE` uses ordering values and peer groups. For frame-sensitive functions, the default with window ordering is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, including current-row peers. Use an explicit frame to make running or moving calculations clear. See [MySQL's frame specification](https://dev.mysql.com/doc/refman/8.0/en/window-functions-frames.html).

## 5. FIRST_VALUE, LAST_VALUE, and NTH_VALUE

**Question:** What are Arjun's first, last, and second innings scores?

`WINDOW w AS (...)` gives a window definition a name so these three functions can reuse it:


```sql
SELECT player_name, match_no, runs,
       FIRST_VALUE(runs) OVER w AS first_score,
       LAST_VALUE(runs) OVER w AS last_score,
       NTH_VALUE(runs, 2) OVER w AS second_score
FROM demo_innings
WHERE player_name = 'Arjun'
WINDOW w AS (
    PARTITION BY player_name ORDER BY match_no
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
ORDER BY match_no;
```

All three output rows show `first_score = 88`, `last_score = 70`, and `second_score = 42`. The explicit full-partition frame makes every innings available at every row.

For the second-highest innings rather than the second chronological innings, use this separate query:

```sql
SELECT player_name, match_no, runs,
       NTH_VALUE(runs, 2) OVER (
           PARTITION BY player_name ORDER BY runs DESC, match_no
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS second_best_innings
FROM demo_innings
WHERE player_name = 'Arjun'
ORDER BY runs DESC, match_no;
```

The result order is match 1 (88), match 3 (70), match 2 (42); `second_best_innings` is 70 in every row. `NTH_VALUE` reads a row position, which need not be a distinct score. See [MySQL's value-function reference](https://dev.mysql.com/doc/refman/8.0/en/window-function-descriptions.html#function_nth-value).

If the full frame is removed from this second query, its first row returns `NULL` for `NTH_VALUE`, because the second ordered row is outside that row's default frame. Similarly, default-frame `LAST_VALUE` can return the current ordering value rather than the partition's final value. See [MySQL's frame examples](https://dev.mysql.com/doc/refman/8.0/en/window-functions-frames.html).

## 6. CTEs and the WITH clause

A **common table expression**, or CTE, gives a query result a name that you can use within the same statement. It helps you read an analysis one step at a time.

### One CTE: players with at least 200 runs

```sql
WITH season_totals AS (
    SELECT player_name, SUM(runs) AS total_runs
    FROM demo_innings
    GROUP BY player_name
)
SELECT player_name, total_runs
FROM season_totals
WHERE total_runs >= 200
ORDER BY total_runs DESC, player_name;
```

Expected players: Rohit (250), Ishaan (220), Arjun (200), Dev (200), and Sam (200).

### Multiple CTEs: each team's leaders

**Question:** Who is the highest scorer in each team, including tied winners?


```sql
WITH season_totals AS (
    SELECT team_name, player_name, SUM(runs) AS total_runs
    FROM demo_innings
    GROUP BY team_name, player_name
),
ranked_players AS (
    SELECT team_name, player_name, total_runs,
           RANK() OVER (
               PARTITION BY team_name ORDER BY total_runs DESC
           ) AS team_rank
    FROM season_totals
)
SELECT team_name, player_name, total_runs, team_rank
FROM ranked_players
WHERE team_rank = 1
ORDER BY team_name, player_name;
```

| Team | Player | Total runs | Team rank |
| --- | --- | ---: | ---: |
| CSK | Arjun | 200 | 1 |
| CSK | Dev | 200 | 1 |
| MI | Rohit | 250 | 1 |
| RCB | Ishaan | 220 | 1 |

The flow is **aggregate → rank within each team → filter**. A CTE names a result used within one statement; it does not create a permanent table or guarantee faster execution. Multiple CTEs follow a single `WITH`, separated by commas. See [MySQL's WITH reference](https://dev.mysql.com/doc/refman/8.0/en/with.html).

## 7. Percentage contributions

**Question:** What percentage of a player's total came from each innings?

```sql
SELECT player_name, match_no, runs,
       SUM(runs) OVER (PARTITION BY player_name) AS player_total,
       ROUND(
           100.0 * runs / NULLIF(
               SUM(runs) OVER (PARTITION BY player_name), 0
           ), 2
       ) AS contribution_pct
FROM demo_innings
WHERE player_name = 'Arjun'
ORDER BY match_no;
```

| Match | Runs | Player total | Contribution % |
| --- | ---: | ---: | ---: |
| 1 | 88 | 200 | 44.00 |
| 2 | 42 | 200 | 21.00 |
| 3 | 70 | 200 | 35.00 |

`NULLIF(total, 0)` makes a zero denominator `NULL`. Here, the three contributions add up to 100%.

## Common mistakes

- **Filtering too early:** To ask what happened after an innings of 50 or more, compute `LEAD` across all relevant innings in a CTE, then apply `runs >= 50` outside it. Otherwise the window sees only qualifying innings.
- **Filtering a window alias in the same SELECT's WHERE:** Compute the window in a CTE or derived table, then filter in the outer query. See [MySQL's window evaluation rules](https://dev.mysql.com/doc/refman/8.0/en/window-functions-usage.html).
- **Breaking meaningful ties:** Keep player names or IDs out of rank/distribution window ordering when equal scores should share a result.
- **Confusing missing data with zero:** An absent neighboring row produces `NULL`; a recorded duck is zero.
- **Reading percent rank as percentage outperformed:** In this descending leaderboard, smaller percent ranks represent better positions. Tied bottom players need not reach 100%.
- **Forgetting display order:** Use an outer `ORDER BY` for reproducible result tables.
- **Using cricket overs as decimal numbers:** For economy calculations, use `6.0 * runs_conceded / balls_bowled`; 3.4 overs means 22 legal balls.

## Quick revision sheet

| Goal | Use |
| --- | --- |
| Keep detail rows alongside group calculations | An aggregate with `OVER (...)` |
| Create separate groups for a window calculation | `PARTITION BY` |
| Give every row a unique position | `ROW_NUMBER()` |
| Rank ties with gaps | `RANK()` |
| Rank ties without gaps | `DENSE_RANK()` |
| Retrieve a previous / next value | `LAG()` / `LEAD()` |
| Calculate a running total | `SUM()` with `ROWS ... UNBOUNDED PRECEDING ... CURRENT ROW` |
| Calculate a two-row moving average | `AVG()` with `ROWS BETWEEN 1 PRECEDING AND CURRENT ROW` |
| Read first / last / nth frame values | `FIRST_VALUE()` / `LAST_VALUE()` / `NTH_VALUE()` |
| Split rows into approximately equal groups | `NTILE(n)` |
| Find cumulative share including ties | `CUME_DIST()` |
| Find a relative rank position | `PERCENT_RANK()` |
| Name query steps | `WITH ... AS (...)` |
| Filter a calculated window result | An outer query over a CTE or derived table |

## Practice questions

1. Calculate the difference between each innings and the preceding innings. For Arjun, expect `NULL`, `-46`, and `28`.
2. Return the top two distinct player-total levels, including ties. Expect Rohit and Ishaan.
3. Return every team's tied leaders. Expect four rows: Arjun, Dev, Rohit, and Ishaan.
4. Change the two-innings moving average to three innings. Arjun's last average becomes `66.67`.
5. Reverse the distribution ordering to ascending and explain how the interpretation changes.
6. Remove the full frame from the second-highest-innings example and explain the first `NULL`.
7. Calculate each player's percentage of their team's total runs. For Arjun, check `200 / 550 × 100 = 36.36%`.

For each exercise, state the partition, ordering, frame if relevant, and expected result before running the query.
