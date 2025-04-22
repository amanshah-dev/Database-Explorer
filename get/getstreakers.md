CREATE FUNCTION getstreakers(userids bigint[], questions integer, subject text)
    RETURNS bigint[]
    LANGUAGE plpgsql
AS
$$
DECLARE
    eligibleUsers bigint[] := ARRAY []::bigint[];
BEGIN
    IF subject = 'Biology' THEN
        SELECT array_agg(userId)
        INTO eligibleUsers
        FROM (
            SELECT tr.userId AS userId
            FROM getCorrectAnswers(userIds, '') tr
            LEFT JOIN getCorrectAnswers(userIds, 'Chemistry') cr ON cr.userId = tr.userId
            LEFT JOIN getCorrectAnswers(userIds, 'Physics') pr ON pr.userId = tr.userId
            WHERE COALESCE(tr.correctAnswers, 0) - COALESCE(cr.correctAnswers, 0) -
                  COALESCE(pr.correctAnswers, 0) >= questions
        ) AS subquery;
    ELSE
        SELECT array_agg(userId)
        INTO eligibleUsers
        FROM (
            SELECT tr.userId AS userId
            FROM getCorrectAnswers(userIds, subject) tr
            WHERE tr.correctAnswers >= questions
        ) AS subquery;
    END IF;
    
    RETURN eligibleUsers;
END;
$$;

ALTER FUNCTION getstreakers(bigint[], integer, text) OWNER TO neetprep_rw;
