CREATE FUNCTION mul_sfunc(anyelement, anyelement) RETURNS anyelement
LANGUAGE sql
AS
$$
SELECT $1 * COALESCE($2, 1);
$$;

ALTER FUNCTION mul_sfunc(anyelement, anyelement) OWNER TO learner;
