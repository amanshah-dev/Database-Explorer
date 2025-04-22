CREATE FUNCTION get_test_section_difficulty_counts(_testid integer)
    RETURNS TABLE("sectionIndex" integer, section text, "difficultyLevel" text, "numQuestions" integer)
    LANGUAGE plpgsql
AS
$$
BEGIN
    RETURN QUERY
    WITH section_ranges AS (
        /* Extract sections and compute start and end numbers */
        SELECT 
            ordinality::INTEGER AS "sectionIndex",  -- Ensure integer type
            value->>0 AS "section",
            COALESCE((value->>1)::INTEGER, 1) AS "startNum",
            COALESCE(
                (LEAD((value->>1)) OVER (ORDER BY ordinality))::INTEGER, 
                (SELECT "Test"."numQuestions" FROM "Test" WHERE "Test"."id" = _testid) + 1  -- ✅ FIXED: Explicitly reference table
            ) - 1 AS "endNum"
        FROM "Test"
        CROSS JOIN LATERAL json_array_elements(
            CASE 
                WHEN json_typeof("Test"."sections") = 'array' THEN "Test"."sections"
                ELSE '[]'::json  -- Ensure valid JSON array
            END
        ) WITH ORDINALITY
        WHERE "Test"."id" = _testid
    ),
    ordered_questions AS (
        /* Assign a unique question number based on seqNum and questionId */
        SELECT
            "TestQuestion"."questionId",
            ROW_NUMBER() OVER (
                ORDER BY "TestQuestion"."seqNum" ASC, "TestQuestion"."questionId" ASC
            ) AS "questionNum"
        FROM "TestQuestion"
        WHERE "TestQuestion"."testId" = _testid
    ),
    question_sections AS (
        /* Assign each question to a section based on questionNum */
        SELECT 
            oq."questionId",
            sr."section",
            sr."sectionIndex"
        FROM ordered_questions oq
        JOIN section_ranges sr
            ON oq."questionNum" BETWEEN sr."startNum" AND sr."endNum"
    ),
    difficulty_counts AS (
        /* Count the number of questions per difficulty level */
        SELECT 
            qs."section",
            qs."sectionIndex",
            COALESCE(qa."difficultyLevel", 'undefined') AS "difficultyLevel",
            COUNT(qs."questionId")::INTEGER AS "numQuestions"  -- ✅ FIXED: Cast COUNT() to integer
        FROM question_sections qs
        LEFT JOIN "QuestionAnalytics" qa 
            ON qs."questionId" = qa."id"
        GROUP BY qs."section", qs."sectionIndex", qa."difficultyLevel"
    ),
    all_difficulties AS (
        /* Generate all difficulty levels for each section */
        SELECT 
            sr."sectionIndex",
            sr."section",
            dl."difficultyLevel"
        FROM section_ranges sr
        CROSS JOIN (VALUES ('easy'), ('medium'), ('difficult')) AS dl("difficultyLevel")  -- Ensure all levels exist
    )
    SELECT 
        ad."sectionIndex",
        ad."section",
        ad."difficultyLevel",
        COALESCE(dc."numQuestions", 0) AS "numQuestions"  -- Default to 0 if no questions exist
    FROM all_difficulties ad
    LEFT JOIN difficulty_counts dc
        ON ad."sectionIndex" = dc."sectionIndex" 
        AND ad."section" = dc."section"
        AND ad."difficultyLevel" = dc."difficultyLevel"
    ORDER BY ad."sectionIndex", 
             CASE ad."difficultyLevel"
                 WHEN 'easy' THEN 1
                 WHEN 'medium' THEN 2
                 WHEN 'difficult' THEN 3
             END;
END;
$$;

ALTER FUNCTION get_test_section_difficulty_counts(integer) OWNER TO learner;
