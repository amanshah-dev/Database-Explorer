-- Create function to get the best subject scorers based on the number of answered questions
create function bestsubjectscorers(userids bigint[], subjectname text, requiredquestions integer) returns bigint[]
    language plpgsql
as
$$
DECLARE
    eligibleUsers BIGINT[] := ARRAY []::BIGINT[]; -- Array to store the result
    testId        BIGINT; -- Variable to store the test ID
BEGIN
    -- Get Test ID using function
    SELECT get_test_id(subjectName) INTO testId;

    -- Return empty array if no test found
    IF testId IS NULL THEN
        RETURN ARRAY []::BIGINT[];
    END IF;

    -- Fetch users who answered enough questions
    SELECT ARRAY_AGG(subquery."userId")
    INTO eligibleUsers
    FROM (SELECT a."userId"
          FROM "Answer" AS a
                   JOIN "TestQuestion" AS tq ON a."questionId" = tq."questionId"
          WHERE a."userId" = ANY (userIds)
            AND tq."testId" = testId
          GROUP BY a."userId"
          HAVING COUNT(*) >= requiredQuestions) subquery;

    -- Return the array of eligible users
    RETURN eligibleUsers;
END;
$$;

-- Alter function owner
alter function bestsubjectscorers(bigint[], text, integer) owner to learner;
