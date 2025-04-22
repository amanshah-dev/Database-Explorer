create function refresh_materialized_view(view_name text)
    returns TABLE(refreshed_view_name text, refresh_time interval)
    language plpgsql
as
$$
DECLARE
    view_exists boolean;
    can_refresh_concurrently boolean;
    start_time timestamp with time zone;
    end_time timestamp with time zone;
BEGIN
    -- Record the start time
    start_time := clock_timestamp();

    -- Check if the materialized view exists
    SELECT EXISTS (
        SELECT 1
        FROM pg_matviews
        WHERE matviewname = view_name
    ) INTO view_exists;

    IF NOT view_exists THEN
        RAISE EXCEPTION 'Materialized view % does not exist', view_name;
    END IF;

    -- Check if the materialized view can be refreshed concurrently
    SELECT relhasindex
    FROM pg_class
    WHERE relname = view_name
    INTO can_refresh_concurrently;

    IF can_refresh_concurrently THEN
        EXECUTE format('REFRESH MATERIALIZED VIEW CONCURRENTLY %I', view_name);
    ELSE
        RAISE NOTICE 'Materialized view % cannot be refreshed concurrently. Refreshing non-concurrently.', view_name;
        EXECUTE format('REFRESH MATERIALIZED VIEW %I', view_name);
    END IF;

    -- Record the end time
    end_time := clock_timestamp();

    -- Prepare output
    refreshed_view_name := view_name;
    refresh_time        := end_time - start_time;

    -- Return the row using RETURN QUERY
    RETURN QUERY
    SELECT view_name, (end_time - start_time);
END;
$$;

alter function refresh_materialized_view(text) owner to learner;
