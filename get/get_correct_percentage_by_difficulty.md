CREATE FUNCTION get_correct_percentage_by_difficulty(p_test_attempt_row "TestAttempt")
    RETURNS TABLE(difficulty_level CHARACTER VARYING, correct_percentage NUMERIC)
    LANGUAGE plpgsql
AS
$$
BEGIN
    RETURN QUERY
    WITH test_questions AS (
        -- Extract all questions from the test
        SELECT 
            tq."questionId",
            qa."difficultyLevel"::VARCHAR
        FROM 
            "TestQuestion" tq
        JOIN 
            "QuestionAnalytics" qa ON tq."questionId" = qa."id"
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
            tq."questionId",
            tq."difficultyLevel",
            COALESCE(aq.is_correct, 0) AS is_correct
        FROM 
            test_questions tq
        LEFT JOIN 
            attempted_questions aq ON tq."questionId" = aq."questionId"
    )
    SELECT 
        "difficultyLevel",
        CASE 
            WHEN COUNT(*) = 0 THEN 0
            ELSE (SUM(is_correct)::NUMERIC / COUNT(*)) * 100
        END AS correct_percentage
    FROM 
        all_questions
    GROUP BY 
        "difficultyLevel";
END;
$$;

ALTER FUNCTION get_correct_percentage_by_difficulty("TestAttempt") OWNER TO learner;
