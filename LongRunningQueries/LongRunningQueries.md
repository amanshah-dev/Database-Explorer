CREATE FUNCTION "LongRunningQueries"(minutes INTEGER DEFAULT 1)
RETURNS TABLE(processid INTEGER, dur INTERVAL, q TEXT, s TEXT)
LANGUAGE plpgsql
AS
$$
  BEGIN
    RETURN QUERY 
    SELECT
      pid,
      now() - pg_stat_activity.query_start AS duration,
      query,
      state
    FROM pg_stat_activity
    WHERE (now() - pg_stat_activity.query_start) > (minutes || 'minutes')::INTERVAL;
  END;
$$;

ALTER FUNCTION "LongRunningQueries"(INTEGER) OWNER TO learner;
