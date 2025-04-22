-- Create function to get the best scorers based on total marks
create function bestscorers(userids bigint[], toppercent integer) returns bigint[]
    language plpgsql
as
$$
DECLARE
    bestScorers BIGINT[] := ARRAY []::BIGINT[]; -- Array to store the result
    target      INT;
BEGIN
    target := 720 * (100 - topPercent) / 100; -- Calculate target based on top percentage
    -- Get the best scorers
    SELECT ARRAY_AGG("userId")
    INTO bestScorers
    FROM (SELECT ta."userId"
          FROM "TestAttempt" ta
                   JOIN "Test" t ON ta."testId" = t."id"
              AND t."numQuestions" = 200 -- Ensure the test has 200 questions
          WHERE ta."createdAt" >= '2024-05-05'
            AND ta."userId" = ANY (userIds) -- Handle the array of user IDs
          GROUP BY ta."userId" -- Group by user to aggregate total marks
          HAVING MAX((ta.result ->> 'totalMarks')::INT) >= target -- Users with total marks >= target
         ) subquery;

    -- Return the array of best scorers
    RETURN bestScorers;
END;
$$;

-- Alter function owner
alter function bestscorers(bigint[], integer) owner to neetprep_rw;
