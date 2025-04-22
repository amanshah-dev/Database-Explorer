CREATE FUNCTION jsonb_delete_left(a JSONB, b TEXT) RETURNS JSONB
IMMUTABLE
STRICT
LANGUAGE sql
AS
$$
    SELECT COALESCE(     
        (
            SELECT ('{' || string_agg(to_json(key) || ':' || value, ',') || '}')
            FROM jsonb_each(a)
            WHERE key <> b
        )
    , '{}')::jsonb;
$$;

COMMENT ON FUNCTION jsonb_delete_left(JSONB, TEXT) IS 'delete key in second argument from first argument';

ALTER FUNCTION jsonb_delete_left(JSONB, TEXT) OWNER TO learner;
