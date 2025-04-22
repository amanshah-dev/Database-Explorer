CREATE FUNCTION "NextDayPopulateDailyQuestionEvent"() RETURNS VOID
LANGUAGE plpgsql
AS
$$
DECLARE
    maxDate DATE;
BEGIN
    SELECT max("eventDate") + interval '1 day' 
    FROM "DailyQuestionEvent" 
    INTO maxDate;
    
    PERFORM "PopulateDailyQuestionEvent"(maxDate);
END
$$;

ALTER FUNCTION "NextDayPopulateDailyQuestionEvent"() OWNER TO learner;
