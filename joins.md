### Theory

> CROSS
- all left rows * all right rows
(cartesian product, or all possible combinations)

> FULL
- all left rows joined to matching (ON | USING) right rows
- not matching right rows

> LEFT, RIGHT 
- all left rows joined to matching (ON | USING) right rows

> INNER 
- some left rows (ON | USING)
- some right rows (ON | USING)

### LATERAL join

`LEFT JOIN` vs `LEFT JOIN LATERAL`:
- in a `LEFT JOIN` right results are limited by:  
    - comparing `val_left` vs `val_right` in `[ON | USING]`
- in a `LEFT JOIN LATERAL` right results are limited by:
    - ***not*** comparing `val_left` vs `val_right` in `[ON | USING]` (just do `on true`)
    - comparing `val_left` vs `val_right` ***inside*** subquery
    ???
    - profit: it's means `LIMIT`, `ORDER BY` can be done ***after*** comparing `val_left` vs `val_right`

Subqueries appearing in `FROM` can be preceded by the key word `LATERAL`. This allows them to reference columns provided by preceding `FROM` items. (Without `LATERAL`, each subquery is evaluated independently and so cannot cross-reference any other `FROM` item.)

Example 1: `LEFT JOIN LATERAL`
```sql
SELECT user_id, first_order_time, next_order_time, id FROM
    (SELECT 
        user_id, 
        min(created_at) AS first_order_time 
    FROM orders GROUP BY user_id) o1
    
    LEFT JOIN LATERAL
    -- lateral subquery
    (
        SELECT id, created_at AS next_order_time
        FROM orders
        WHERE user_id = o1.user_id 
            -- reference columns provided by preceding FROM items (o1.first_order_time)
            AND created_at > o1.first_order_time
        ORDER BY created_at ASC LIMIT 1
    ) o2 ON true;
```

Example 2: `CROSS JOIN LATERAL`
```sql
SELECT *
FROM table1 t1
CROSS JOIN LATERAL
    -- lateral subquery
    (
        SELECT *
        FROM t2
        -- t1.col1 is only allowed because of lateral
        WHERE   t1.col1 = t2.col1 
    ) sub
```