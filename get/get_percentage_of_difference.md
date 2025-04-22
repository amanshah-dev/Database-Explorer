CREATE FUNCTION get_percentage_of_difference(
    start_date TIMESTAMP WITH TIME ZONE, 
    end_date TIMESTAMP WITH TIME ZONE, 
    user_id INTEGER
) 
RETURNS NUMERIC
LANGUAGE plpgsql
AS
$$
DECLARE
    count_result INTEGER;
    total_count INTEGER;
    percentage DECIMAL;
    start_date_gmt TIMESTAMP;
    end_date_gmt TIMESTAMP;
BEGIN
    -- Convert start_date and end_date to GMT timezone (assuming IST is +05:30)
    start_date_gmt := start_date AT TIME ZONE 'IST' AT TIME ZONE 'GMT';
    end_date_gmt := end_date AT TIME ZONE 'IST' AT TIME ZONE 'GMT';

    -- Get the count of rows where the difference is less than 10 seconds and testAttemptId is null and userAnswer is not null
    SELECT count(*) FILTER (WHERE "time_diff" < INTERVAL '10 seconds'), count(*) 
    INTO count_result, total_count 
    FROM (
        SELECT 
            CASE 
                WHEN LEAD("questionId") OVER (ORDER BY "createdAt" ASC) = "questionId" THEN 
                    LEAD("createdAt", 2) OVER (ORDER BY "createdAt" ASC) - "createdAt"
                ELSE 
                    LEAD("createdAt") OVER (ORDER BY "createdAt" ASC) - "createdAt"
            END AS "time_diff"
        FROM "Answer"
        WHERE "userId" = user_id
            AND "createdAt" >= start_date_gmt
            AND "createdAt" <= end_date_gmt
            AND "testAttemptId" IS NULL
            AND "userAnswer" IS NOT NULL
    ) t;

    -- Calculate the percentage
    IF total_count > 0 THEN
        percentage := (count_result::DECIMAL / total_count) * 100;
    ELSE
        percentage := 0;
    END IF;

    RETURN percentage;
END;
$$;

ALTER FUNCTION get_percentage_of_difference(
    TIMESTAMP WITH TIME ZONE, 
    TIMESTAMP WITH TIME ZONE, 
    INTEGER
) OWNER TO learner;
