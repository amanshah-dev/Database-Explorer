CREATE FUNCTION getcorrectanswers(userids bigint[], subject text)
    RETURNS TABLE(userid bigint, correctanswers bigint)
    LANGUAGE plpgsql
AS
$$
DECLARE
    sql_query TEXT;
BEGIN
    sql_query := 'SELECT "userId"::bigint, COALESCE(SUM("eventCount"), 0) as correctAnswers
        FROM "DailyUserEvent"
        WHERE "userId" = any ($1)
          AND "event" = ''' || subject ||'CorrectQuestion'' 
          AND "eventDate" = CURRENT_DATE
        GROUP BY "userId"
        ORDER BY "userId" DESC;';
    RETURN QUERY EXECUTE sql_query USING userIds;
END;
$$;

ALTER FUNCTION getcorrectanswers(bigint[], text) OWNER TO neetprep_rw;
