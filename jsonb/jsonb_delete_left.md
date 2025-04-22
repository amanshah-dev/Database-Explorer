CREATE FUNCTION jsonb_delete_left(a JSONB, b JSONB) RETURNS JSONB
IMMUTABLE
STRICT
LANGUAGE sql
AS
$$
    SELECT COALESCE(     
        (
            SELECT ('{' || string_agg(to_json(key) || ':' || value, ',') || '}')
            FROM jsonb_each(a)
            WHERE NOT ('{' || to_json(key) || ':' || value || '}')::jsonb <@ b
        )
    , '{}')::jsonb;
$$;

COMMENT ON FUNCTION jsonb_delete_left(JSONB, JSONB) IS 'delete matching pairs in second argument from first argument';

ALTER FUNCTION jsonb_delete_left(JSONB, JSONB) OWNER TO learner;
