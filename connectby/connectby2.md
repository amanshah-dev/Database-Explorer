WITH RECURSIVE connect_tree AS (
    SELECT id, parent_id, name
    FROM your_table
    WHERE parent_id IS NULL -- or base condition for recursion
    UNION ALL
    SELECT t.id, t.parent_id, t.name
    FROM your_table t
    JOIN connect_tree ct ON ct.id = t.parent_id
)
SELECT * FROM connect_tree;
