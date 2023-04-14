# Example of Recursive call in PostgreSQL

For certain data models, such as family trees or organization charts, a recursive function may be necessary to traverse all relationships between data elements. For instance, we could use a recursive function to find all the descentors of a person on a family tree.

Performing these types of recursive calls directly on the back-end server can be quite expensive since it requires loading a large number of elements from the database to the server and then processing them. This can consume a significant amount of the server's RAM and CPU resources. On the other hand, databases are typically more scalable than servers. Therefore, if we can perform the recursive function as a database query, it would be more efficient.

## Here is an example solution

Assuming we have an employee table with a boss_id field that points to the same table to reflect a direct boss-employee relationship:
| employees  | 
| ------------- |
| id  |
| name  |
| boss_id  |
| company_id  |

| companies  | 
| ------------- |
| id  |
| name  |

Here is a query that can be used to retrieve a list of employee IDs corresponding to their respective bosses in the company's database.

```SQL
WITH RECURSIVE cte AS (
  WITH anchor AS (
    SELECT e.id AS id, e.boss_id AS boss_id
    FROM employees AS e
    WHERE e.boss_id IS NOT NULL AND e.company_id = {company_id}
  )
  SELECT * from anchor
  UNION
  SELECT cte.id, anchor.boss_id
  FROM cte
  JOIN anchor ON cte.boss_id = anchor.id AND cte.id != anchor.boss_id
)
SELECT boss_id as id, ARRAY_AGG(id) AS employee_ids
FROM cte
GROUP BY boss_id
```

The syntax for recursive SQL queries can be difficult to understand at first. However, we can think of it as a way to join multiple lists of rows using the `UNION` operation. The first list above the `UNION` keyword is the `anchor` member, while the subsequent lists below the `UNION` keyword are the `recursive` members. Essentially, each recursive member is the previous recursive member `JOIN`ed with the anchor. By performing a `UNION` operation on each pair of lists, we can construct the final result.

Let's apply the above instructions to a simple example to see how it works. We have an employee A who has B as a boss, B has C as a boss, D also has C as a boss, and E does not have a boss. All of these employees belong to the same company, which is the target company for the `anchor` query. We can refer to these employees by their IDs: A, B, C, D, and E.

The `anchor` member will look like this

| anchor ||
| ------------- | - |
| id  | boss_id |
| A | B |
| B | C |
| D | C |

In the first `UNION`, the `anchor` is joined to itself to create a `recursive` member. Let's take a closer look at how this `JOIN` works: for each row in the `cte` list, the query finds the corresponding boss in the `anchor` list and then adds the employee's `id` along with the boss's `boss_id`. This means that we are finding the boss of a boss of an employee.

| recursive_member ||
| ------------- | - |
| id  | boss_id |
| A | C |

There is only 1 row on this list because only boss B has his higher boss as C.
Joining the `anchor` and `recursive_member` give us the new `cte`.

| cte ||
| ------------- | - |
| id  | boss_id |
| A | B |
| B | C |
| D | C |
| A | C |

We can continue the recursion by doing another `UNION` between `cte` and the previous `recursive_member`. This will give us a new `recursive_member` that includes all the employees and their bosses up to two levels higher. Specifically, we will get a list of each employee and their boss, as well as the boss's boss, if applicable.

On the second `UNION`, the `recursive_member` list that resulted from the previous recursion is empty because there is no boss of a boss of a boss. This signals the recursive call to stop. And then we can use `GROUP BY` to create the final result by grouping the employees based on their boss's ID and make a list of employees in each group.

| cte ||
| ------------- | - |
| id  | employee_ids |
| C | [B, D, A] |
| B | [A] |

In the query I also add this condition when joining `cte` and `anchor`. It will prevent a infinite loop when `employee` has `boss_id` pointed to the record itself. (So the employee is the boss of hiself).

```
cte.id != anchor.boss_id
```

I also make use of `UNION` and not `UNION ALL` like normal recursive call in SQL. Because there is a case that we can have a loop on those boss-employee relationship. In the example above, boss C can have `boss_id = A`, and the `recursive_member` will always has member so the loop will never terminate.
