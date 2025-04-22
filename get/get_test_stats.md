CREATE FUNCTION get_test_stats(_userid integer, _testid integer)
    RETURNS TABLE(section text, sectionindex integer, totalquestions integer, totaluserattempts integer, correctanswers integer)
    LANGUAGE plpgsql
AS
$$
BEGIN
    RETURN QUERY
    WITH section_ranges AS (
        /* Extract sections and compute start and end numbers */
        SELECT
            ordinality AS "sectionIndex",
            value->>0 AS "section",
            COALESCE((value->>1)::integer, 1) AS "startNum",
            COALESCE(
                (LEAD((value->>1)) OVER (ORDER BY ordinality))::integer,
                "Test"."numQuestions" + 1
            ) - 1 AS "endNum"
        FROM "Test"
        LEFT JOIN LATERAL json_array_elements(COALESCE("Test"."sections", '[]'::json))
            WITH ORDINALITY
            ON TRUE
        WHERE "Test"."id" = _testid
    ),
    ordered_questions AS (
        /* Assign a unique question number based on seqNum and questionId */
        SELECT
            "TestQuestion"."questionId",
            "Question"."correctOptionIndex",
            ROW_NUMBER() OVER (
                ORDER BY "TestQuestion"."seqNum" ASC, "TestQuestion"."questionId" ASC
            ) AS "questionNum"
        FROM "TestQuestion"
        JOIN "Question"
            ON "Question"."id" = "TestQuestion"."questionId"
        WHERE "TestQuestion"."testId" = _testid
    ),
    question_sections AS (
        /* Assign each question to a section based on questionNum */
        SELECT
            oq."questionId",
            oq."correctOptionIndex",
            sr."section",
            sr."sectionIndex"
        FROM ordered_questions oq
        JOIN section_ranges sr
            ON oq."questionNum" BETWEEN sr."startNum" AND sr."endNum"
    ),
    latest_attempts AS (
        /* Find the latest attempt for each question by the user where testAttemptId is NULL */
        SELECT
            a."questionId",
            a."userAnswer",
            RANK() OVER (PARTITION BY a."questionId" ORDER BY a."createdAt" DESC) AS "attemptRank"
        FROM "Answer" a
        WHERE a."userId" = _userid
          AND a."testAttemptId" IS NULL  -- Only consider answers where testAttemptId is NULL
    ),
    filtered_attempts AS (
        /* Only consider the most recent attempt (RANK() = 1) */
        SELECT
            la."questionId",
            la."userAnswer"
        FROM latest_attempts la
        WHERE la."attemptRank" = 1
    ),
    aggregated_answers AS (
        /* Aggregate the filtered attempts and count correct answers */
        SELECT
            fa."questionId",
            COUNT(fa."userAnswer")::INT AS "totalUserAttempts",  -- Count only the latest attempt
            SUM(CASE WHEN fa."userAnswer" = qs."correctOptionIndex" THEN 1 ELSE 0 END)::INT AS "correctAnswers"
        FROM filtered_attempts fa
        RIGHT JOIN question_sections qs
            ON fa."questionId" = qs."questionId"
        GROUP BY fa."questionId"
    )
    SELECT
        qs."section",
        qs."sectionIndex"::integer,
        COUNT(qs."questionId")::INT AS "totalQuestions",  -- Total unique questions in the section
        COALESCE(SUM(agg."totalUserAttempts"), 0)::INT AS "totalUserAttempts",  -- Total attempts, only considering the latest
        COALESCE(SUM(agg."correctAnswers"), 0)::INT AS "correctAnswers"  -- Correct answers
    FROM question_sections qs
    LEFT JOIN aggregated_answers agg
        ON qs."questionId" = agg."questionId"
    GROUP BY qs."section", qs."sectionIndex"
    ORDER BY qs."sectionIndex";
END;
$$;

ALTER FUNCTION get_test_stats(integer, integer) OWNER TO learner;
