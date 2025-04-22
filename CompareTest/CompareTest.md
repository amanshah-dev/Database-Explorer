-- Create function to compare two tests
create function CompareTest(testid1 integer, testid2 integer, checkorder boolean DEFAULT false)
    returns TABLE(test1_questionid integer, test2_questionid integer)
    language plpgsql
as
$$
BEGIN
    RETURN QUERY
    WITH
    /* Get questions from testId1 */
    test1_questions AS (
        SELECT "questionId", row_number() OVER (ORDER BY "questionId") AS row_num
        FROM "TestQuestion"
        WHERE "testId" = testId1
    ),
    /* Get questions from testId2 */
    test2_questions AS (
        SELECT "questionId", row_number() OVER (ORDER BY "questionId") AS row_num
        FROM "TestQuestion"
        WHERE "testId" = testId2
    )
    SELECT
        /* Questions present in test1 but not in test2 */
        test1."questionId" AS test1_questionId,
        NULL AS test2_questionId
    FROM test1_questions test1
    LEFT JOIN test2_questions test2 ON test1."questionId" = test2."questionId"
    WHERE checkOrder = FALSE AND test2."questionId" IS NULL

    UNION ALL

    SELECT
        /* Questions present in test2 but not in test1 */
        NULL AS test1_questionId,
        test2."questionId" AS test2_questionId
    FROM test2_questions test2
    LEFT JOIN test1_questions test1 ON test2."questionId" = test1."questionId"
    WHERE checkOrder = FALSE AND test1."questionId" IS NULL

    UNION ALL

    /* Check order if checkOrder is true */
    SELECT
        /* Mismatched sequence */
        test1."questionId" AS test1_questionId,
        test2."questionId" AS test2_questionId
    FROM test1_questions test1
    FULL OUTER JOIN test2_questions test2 ON test1.row_num = test2.row_num
    WHERE checkOrder = TRUE AND test1."questionId" IS DISTINCT FROM test2."questionId";
END;
$$;

-- Alter function owner
alter function "CompareTest"(integer, integer, boolean) owner to neetprep_rw;
