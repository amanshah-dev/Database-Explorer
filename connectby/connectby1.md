WITH RECURSIVE connect_tree AS (
    SELECT id, parent_id, data
    FROM your_table
    WHERE parent_id IS NULL -- or some base condition
    UNION ALL
    SELECT t.id, t.parent_id, t.data
    FROM your_table t
    INNER JOIN connect_tree ct ON ct.id = t.parent_id
)
SELECT * FROM connect_tree;
