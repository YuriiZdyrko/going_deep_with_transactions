```sql
SELECT
        grouping_element,
        ...
        aggregate_function(non_grouping_column)
    FROM ...
    [WHERE ...]
    GROUP BY
         grouping_element,
         ...
    HAVING boolean_expression
```

```sql
-- and grouping_element can be one of:

( )
expression
( expression [, ...] )
ROLLUP ( { expression | ( expression [, ...] ) } [, ...] )
CUBE ( { expression | ( expression [, ...] ) } [, ...] )
GROUPING SETS ( grouping_element [, ...] )
```

`HAVING` allows to filter based on aggregate value:
```sql
SELECT 
   column_1, 
   column_2,
   aggregate_function(column_3)
FROM 
   table_name
GROUP BY 
   column_1,
   column_2
HAVING aggregate_function(column_3) => 5;

```

`GROUP BY` can group by result of an expression,
not just by raw column value
```sql
SELECT 
   DATE(payment_date) paid_date, 
   SUM(amount) sum
FROM 
   payment
GROUP BY
   DATE(payment_date);
```

`GROUPING SETS` is same as having multiple `GROUP_BY` joined together using `UNION_ALL` ([as explained here](https://www.sqltutorial.org/sql-grouping-sets/))

```sql
brand | size | sales
-------+------+-------
 Foo   | L    |  10
 Foo   | M    |  20
 Bar   | M    |  15
 Bar   | L    |  5

=> SELECT brand, size, sum(sales) FROM items_sold 
    GROUP BY GROUPING SETS ((brand), (size), ());

 brand | size | sum
-------+------+-----
 Foo   |      |  30
 Bar   |      |  20
       | L    |  15
       | M    |  35
       |      |  50
(5 rows)
```

`ROLLUP` - subtotals (totals by groups), and also grand total (last row)
```sql
ROLLUP ( e1, e2, e3, ... )
-- is equivalent to
GROUPING SETS (
    ( e1, e2, e3, ... ),
    ...
    ( e1, e2 ),
    ( e1 ),
    ( )
)

ROLLUP ( a, (b, c), d )
-- is equivalent to
GROUPING SETS (
    ( a, b, c, d ),
    ( a, b, c    ),
    ( a          ),
    (            ) -- nothing, nothing, nothing -> grand total
)
```

`CUBE` - group by all possible combinations of grouping items
```sql
CUBE ( e1, e2, ... )
--- is equivalent to
GROUPING SETS (
    ( a, b, c ),
    ( a, b    ),
    ( a,    c ),
    ( a       ),
    (    b, c ),
    (    b    ),
    (       c ),
    (         )

CUBE ( (a, b), (c, d) )
-- is equivalent to
GROUPING SETS (
    ( a, b, c, d ),
    ( a, b       ),
    (       c, d ),
    (            )
)
```

Brain fuck
```sql
GROUP BY a, CUBE (b, c), GROUPING SETS ((d), (e))
-- is equivalent to
GROUP BY GROUPING SETS (
    (a, b, c, d), (a, b, c, e),
    (a, b, d),    (a, b, e),
    (a, c, d),    (a, c, e),
    (a, d),       (a, e)
)
```