CREATE FUNCTION get_chapter_percentages(p_test_attempt_row "TestAttempt")
    RETURNS TABLE(topic_name CHARACTER VARYING, correct_percentage NUMERIC)
    LANGUAGE plpgsql
AS
$$
BEGIN
    RETURN QUERY
    WITH test_questions AS (
        -- Extract questions and corresponding chapter (topic) IDs from the test
        SELECT 
            tq."questionId",
            cq."chapterId" AS "topicId"
        FROM 
            "TestQuestion" tq
        LEFT JOIN 
            "ChapterQuestion" cq ON tq."questionId" = cq."questionId"
        WHERE 
            tq."testId" = p_test_attempt_row."testId"
    ),
    attempted_questions AS (
        -- Extract attempted questions and correctness
        SELECT 
            aq."key"::INTEGER AS "questionId",
            CASE
                WHEN aq."value"::INTEGER = q."correctOptionIndex" THEN 1
                ELSE 0
            END AS is_correct
        FROM 
            LATERAL json_each_text(p_test_attempt_row."userAnswers") AS aq
        JOIN 
            "Question" q ON aq."key"::INTEGER = q."id"
    ),
    all_questions AS (
        -- Combine test questions and attempted questions
        SELECT 
            tq."topicId",
            COALESCE(aq.is_correct, 0) AS is_correct
        FROM 
            test_questions tq
        LEFT JOIN 
            attempted_questions aq ON tq."questionId" = aq."questionId"
        WHERE 
            tq."topicId" IS NOT NULL  -- Ensure we only include questions with valid topics
    )
    SELECT 
        t."name" AS topic_name,
        CASE 
            WHEN COUNT(*) = 0 THEN 0
            ELSE (SUM(is_correct)::NUMERIC / COUNT(*)) * 100
        END AS correct_percentage
    FROM 
        all_questions aq
    JOIN 
        "Topic" t ON aq."topicId" = t."id"
    GROUP BY 
        t."name";
END;
$$;

ALTER FUNCTION get_chapter_percentages("TestAttempt") OWNER TO learner;
