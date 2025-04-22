CREATE FUNCTION move_and_delete_answers(p_userid INTEGER, p_date DATE) RETURNS VOID
LANGUAGE plpgsql
AS
$$
BEGIN
    -- Insert answers before the specified date into CopyAnswer table
    INSERT INTO "CopyAnswer" ("questionId", "userId", "testAttemptId", "durationInSec", "userAnswer", "createdAt", "updatedAt")
    SELECT "questionId", "userId", "testAttemptId", "durationInSec", "userAnswer", "createdAt", "updatedAt"
    FROM "Answer"
    WHERE "userId" = p_userId
      AND "createdAt" < p_date::TIMESTAMP;  -- Cast date to timestamp to include all time values for that day

    -- Delete the answers that have been copied to the CopyAnswer table
    DELETE FROM "Answer"
    WHERE "userId" = p_userId
      AND "createdAt" < p_date::TIMESTAMP;  -- Cast date to timestamp to include all time values for that day
END;
$$;

ALTER FUNCTION move_and_delete_answers(INTEGER, DATE) OWNER TO neetprep_rw;
