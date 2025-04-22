CREATE FUNCTION get_test_attempt_summary(p_test_attempt_row "TestAttempt")
    RETURNS TABLE(total_questions integer, correct_questions integer, incorrect_questions integer, accuracy_rate double precision)
    LANGUAGE plpgsql
AS
$$
BEGIN
    RETURN QUERY
    SELECT 
        t."numQuestions" AS total_questions,
        (p_test_attempt_row."result"->>'correctAnswerCount')::INT AS correct_questions,
        (p_test_attempt_row."result"->>'incorrectAnswerCount')::INT AS incorrect_questions,
        CASE 
            WHEN t."numQuestions" = 0 THEN 0::FLOAT
            ELSE ((p_test_attempt_row."result"->>'correctAnswerCount')::FLOAT / t."numQuestions"::FLOAT) * 100
        END AS accuracy_rate
    FROM 
        "Test" t
    WHERE 
        t."id" = p_test_attempt_row."testId";
END;
$$;

ALTER FUNCTION get_test_attempt_summary("TestAttempt") OWNER TO learner;
