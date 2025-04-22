CREATE FUNCTION get_scheduler_end_date(p_schedule_id BIGINT) 
    RETURNS DATE
    LANGUAGE plpgsql
AS
$$
DECLARE
    scheduler_end_date DATE;
BEGIN
    -- Retrieve the end date of the scheduler based on the provided schedule ID
    SELECT "endDate" INTO scheduler_end_date
    FROM "Scheduler"
    WHERE "id" = p_schedule_id;

    -- Return the retrieved end date
    RETURN scheduler_end_date;
END;
$$;

ALTER FUNCTION get_scheduler_end_date(BIGINT) OWNER TO neetprep_rw;
