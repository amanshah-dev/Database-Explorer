CREATE FUNCTION generatetestintervals(startdate DATE, enddate DATE, userid INTEGER)
    RETURNS TABLE(startinterval DATE, endinterval DATE, testdate DATE, testscheduleid BIGINT)
    LANGUAGE plpgsql
AS
$$
DECLARE
    testDates DATE[];
BEGIN
    -- Generate test dates using the testDateGenFunc function
    testDates := testDateGenFunc(startDate, endDate);

    -- Retrieve test schedules while ensuring uniqueness and correct ordering
    RETURN QUERY 
    WITH userSchedules AS (
        SELECT "id" FROM "UserSchedule" WHERE "userId" = userid
    ),
    orderedTestSchedules AS (
        SELECT DISTINCT ON (ts."testDate") 
               ts."id", 
               ts."testDate"
        FROM "TestSchedule" ts
        JOIN userSchedules us ON ts."userScheduleId" = us."id"
        WHERE ts."testDate"::DATE = ANY(testDates)
        ORDER BY ts."testDate", ts."id"  -- Ensuring the first occurrence per test date
    )
    SELECT 
        (ots."testDate" - INTERVAL '7 days')::DATE AS startInterval, 
        (ots."testDate" + INTERVAL '7 days')::DATE AS endInterval,
        ots."testDate"::DATE, 
        ots."id"
    FROM orderedTestSchedules ots
    ORDER BY ots."testDate"; -- Ensure output is in the correct order
END;
$$;

ALTER FUNCTION generatetestintervals(DATE, DATE, INTEGER) OWNER TO learner;
